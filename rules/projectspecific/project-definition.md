# AsyncTaskFlow - Unified Project Specification

## Project Overview

AsyncTaskFlow is a production-ready distributed task processing system built with FastAPI, Redis, and Celery. It implements a robust architecture for handling long-running text summarization tasks via OpenRouter API, featuring comprehensive error handling, circuit breaker patterns, retry mechanisms, and dead letter queue management.

## Core Architecture

### System Components

```
┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐
│                 │     │                 │     │                 │
│  Web Frontend   │────▶│   FastAPI App   │────▶│  Redis Broker   │
│  (TypeScript)   │     │   (REST API)    │     │   (Queues)      │
│                 │     │                 │     │                 │
└─────────────────┘     └─────────────────┘     └────────┬────────┘
                                                          │
                        ┌─────────────────┐               │
                        │                 │               │
                        │ Celery Workers  │◀──────────────┘
                        │ (Task Execution)│
                        │                 │
                        └────────┬────────┘
                                 │
                        ┌────────▼────────┐     ┌─────────────────┐
                        │                 │     │                 │
                        │ OpenRouter API  │     │ Circuit Breaker │
                        │ (Summarization) │◀────│   (Protection)  │
                        │                 │     │                 │
                        └─────────────────┘     └─────────────────┘
```

## Task State Model

### Core States

1. **PENDING** - Task queued, awaiting worker pickup
2. **ACTIVE** - Currently being processed by a worker
3. **COMPLETED** - Successfully completed with result stored
4. **FAILED** - Execution failed, eligible for retry evaluation
5. **DLQ** - Moved to dead letter queue (exceeded retries or permanent error)

### Task Metadata Structure

```json
{
    "task_id": "550e8400-e29b-41d4-a716-446655440000",
    "state": "PENDING",
    "content": "Text to summarize...",
    "retry_count": 0,
    "max_retries": 3,
    "last_error": null,
    "error_type": null,
    "retry_after": null,
    "created_at": "2024-01-10T15:00:00Z",
    "updated_at": "2024-01-10T15:00:00Z",
    "completed_at": null,
    "result": null,
    "error_history": []
}
```

## Queue Architecture

### Dual-Queue System

The system uses two distinct source queues to prevent retry storms from blocking new work:

1. **Primary Queue** (`tasks:pending:primary`)
   - Contains all newly submitted tasks
   - FIFO processing order
   - No retry tasks ever enter this queue

2. **Retry Queue** (`tasks:pending:retry`)
   - Contains only tasks that failed and are eligible for retry
   - Isolated from new tasks to ensure fair processing
   - Prevents cascading failures from monopolizing workers

3. **Scheduled Queue** (`tasks:scheduled`)
   - Redis sorted set for delayed retries
   - Score = Unix timestamp when task should be moved to retry queue
   - Processed by scheduler service

### Worker Consumption Strategy

Workers consume from both queues using a configurable ratio (default 70/30):

```python
async def get_next_task(redis, retry_ratio=0.3):
    """
    Consume tasks with configurable split between primary/retry queues.
    Default: 70% from primary queue, 30% from retry queue.
    """
    if random.random() > retry_ratio:
        # Try primary queue first
        task_id = await redis.lpop("tasks:pending:primary")
        if not task_id:
            task_id = await redis.lpop("tasks:pending:retry")
    else:
        # Try retry queue first
        task_id = await redis.lpop("tasks:pending:retry")
        if not task_id:
            task_id = await redis.lpop("tasks:pending:primary")
    
    return task_id
```

The retry ratio dynamically adjusts based on queue pressure:
- Normal (< 1000 retry tasks): 30% retry consumption
- Warning (1000-5000 retry tasks): 20% retry consumption  
- Critical (> 5000 retry tasks): 10% retry consumption

## State Transitions

### Valid State Transitions

```
PENDING → ACTIVE → COMPLETED
   ↓        ↓
   DLQ    FAILED → PENDING (retry queue)
            ↓
           DLQ
```

### Transition Rules

**PENDING → ACTIVE**
- Worker picks up task
- Circuit breaker is closed/half-open
- Task age < max_task_age

**ACTIVE → COMPLETED**
- Successful execution
- Result stored in Redis

**ACTIVE → FAILED**
- Any error during execution
- Triggers retry evaluation

**FAILED → PENDING (retry queue)**
- Transient error type
- retry_count < max_retries
- Task age < max_task_age

**FAILED → DLQ**
- Permanent error type OR
- retry_count >= max_retries OR
- Task age >= max_task_age

## Error Classification and Retry Logic

### Error Categories

**Transient Errors** (will retry):
- Network timeouts
- Rate limiting (429)
- Service unavailable (503)
- Server errors (500, 502, 504)

**Permanent Errors** (straight to DLQ):
- Invalid input (400)
- Authentication failure (401)
- Permission denied (403)
- Not found (404)

### Retry Strategy

Exponential backoff with jitter, customized by error type:

```python
RETRY_SCHEDULES = {
    "RateLimitError": [60, 120, 300, 600],      # Start with 1 minute
    "ServiceUnavailable": [5, 10, 30, 60, 120], # Quick initial retry
    "NetworkTimeout": [2, 5, 10, 30, 60],       # Standard backoff
    "Default": [5, 15, 60, 300]                 # Generic schedule
}

def calculate_retry_delay(retry_count, error_type):
    schedule = RETRY_SCHEDULES.get(error_type, RETRY_SCHEDULES["Default"])
    base_delay = schedule[min(retry_count, len(schedule) - 1)]
    jitter = random.uniform(0, base_delay * 0.1)
    return base_delay + jitter
```

## Circuit Breaker Pattern

### States
- **CLOSED**: Normal operation
- **OPEN**: Service down, fail fast
- **HALF_OPEN**: Testing recovery

### Implementation Approach

Instead of storing circuit breaker state in Redis, we recommend using Celery's built-in monitoring:

```python
class CircuitBreaker:
    def __init__(self, failure_threshold=0.5, volume_threshold=10, timeout=60):
        self.failure_threshold = failure_threshold
        self.volume_threshold = volume_threshold
        self.timeout = timeout
        self.failure_counts = defaultdict(int)
        self.success_counts = defaultdict(int)
        self.last_failure_time = {}
        self.state = defaultdict(lambda: "CLOSED")
    
    def record_success(self, service):
        self.success_counts[service] += 1
        if self.state[service] == "HALF_OPEN":
            self.state[service] = "CLOSED"
            self.reset_counts(service)
    
    def record_failure(self, service):
        self.failure_counts[service] += 1
        self.last_failure_time[service] = time.time()
        
        total = self.failure_counts[service] + self.success_counts[service]
        if total >= self.volume_threshold:
            failure_rate = self.failure_counts[service] / total
            if failure_rate >= self.failure_threshold:
                self.state[service] = "OPEN"
```

## Simplified API Design

### Task Management Endpoints

```python
# POST /api/v1/tasks/summarize/
# Submit a new summarization task
async def create_task(content: str) -> TaskResponse:
    task_id = str(uuid4())
    task_data = {
        "task_id": task_id,
        "content": content,
        "state": "PENDING",
        "created_at": datetime.utcnow().isoformat(),
        "retry_count": 0,
        "max_retries": 3
    }
    
    # Store task metadata
    await redis.hset(f"task:{task_id}", mapping=task_data)
    
    # Queue in primary queue
    await redis.lpush("tasks:pending:primary", task_id)
    
    # Trigger Celery task
    summarize_task.delay(task_id)
    
    return TaskResponse(task_id=task_id, state="PENDING")

# GET /api/v1/tasks/{task_id}
# Get task status and result (generic, works for any task type)
async def get_task(task_id: str) -> TaskDetail:
    task_data = await redis.hgetall(f"task:{task_id}")
    if not task_data:
        raise HTTPException(404, "Task not found")
    return TaskDetail(**task_data)

# POST /api/v1/tasks/{task_id}/retry
# Manually retry a failed task (generic, works for any task type)
async def retry_task(task_id: str) -> TaskResponse:
    task_data = await redis.hgetall(f"task:{task_id}")
    if not task_data or task_data["state"] not in ["FAILED", "DLQ"]:
        raise HTTPException(400, "Task cannot be retried")
    
    # Reset retry count and queue in retry queue
    await redis.hset(f"task:{task_id}", mapping={
        "state": "PENDING",
        "retry_count": 0,
        "updated_at": datetime.utcnow().isoformat()
    })
    await redis.lpush("tasks:pending:retry", task_id)
    
    return TaskResponse(task_id=task_id, state="PENDING")

# Additional generic task management endpoints:
# GET /api/v1/tasks/?status={TaskState}&limit={number}
# POST /api/v1/tasks/requeue-orphaned
# DELETE /api/v1/tasks/{task_id}
```

### Queue Monitoring Endpoints

```python
# GET /api/v1/queues/status
# Single endpoint for all queue statistics
async def get_queue_status() -> QueueStatus:
    primary_depth = await redis.llen("tasks:pending:primary")
    retry_depth = await redis.llen("tasks:pending:retry")
    scheduled_count = await redis.zcard("tasks:scheduled")
    dlq_depth = await redis.llen("dlq:tasks")
    
    # Get task counts by state
    states = defaultdict(int)
    async for key in redis.scan_iter("task:*"):
        state = await redis.hget(key, "state")
        states[state] += 1
    
    return QueueStatus(
        queues={
            "primary": primary_depth,
            "retry": retry_depth,
            "scheduled": scheduled_count,
            "dlq": dlq_depth
        },
        states=dict(states),
        retry_ratio=calculate_adaptive_retry_ratio(retry_depth)
    )

# GET /api/v1/queues/dlq
# List dead letter queue contents
async def get_dlq_tasks(limit: int = 100) -> List[TaskDetail]:
    task_ids = await redis.lrange("dlq:tasks", 0, limit - 1)
    tasks = []
    for task_id in task_ids:
        task_data = await redis.hgetall(f"dlq:task:{task_id}")
        if task_data:
            tasks.append(TaskDetail(**task_data))
    return tasks
```

### Health Check Endpoint

```python
# GET /health
# Comprehensive health check
async def health_check() -> HealthStatus:
    try:
        # Check Redis
        await redis.ping()
        redis_ok = True
    except:
        redis_ok = False
    
    # Check Celery workers
    active_workers = celery_app.control.inspect().active_queues()
    workers_ok = bool(active_workers)
    
    # Get circuit breaker status (from in-memory or Celery backend)
    circuit_status = get_circuit_breaker_status()
    
    return HealthStatus(
        status="healthy" if redis_ok and workers_ok else "unhealthy",
        components={
            "redis": redis_ok,
            "workers": workers_ok,
            "circuit_breaker": circuit_status
        },
        timestamp=datetime.utcnow().isoformat()
    )
```

## Service Tasks Architecture

For circuit breaker and scheduler services, we recommend:

1. **Circuit Breaker**: Implement as in-memory state within each worker process, synchronized via Celery's result backend or events. No need for long-term Redis storage.

2. **Scheduler Service**: Run as a single Celery beat process that:
   - Checks the scheduled queue every second
   - Moves due tasks to retry queue
   - Monitors task age limits

```python
@celery_app.task
def process_scheduled_tasks():
    """Celery periodic task to process scheduled retries"""
    now = time.time()
    
    # Get due tasks (up to 100 at a time)
    due_tasks = redis.zrangebyscore("tasks:scheduled", 0, now, start=0, num=100)
    
    for task_id in due_tasks:
        # Move to retry queue
        redis.lpush("tasks:pending:retry", task_id)
        redis.zrem("tasks:scheduled", task_id)
        
        # Update state
        redis.hset(f"task:{task_id}", "state", "PENDING")

# Configure Celery beat schedule
celery_app.conf.beat_schedule = {
    'process-scheduled-tasks': {
        'task': 'process_scheduled_tasks',
        'schedule': 1.0,  # Every second
    },
}
```

## Redis Key Structure (Single Source of Truth)

```
# Task data (single source for all task information)
task:{task_id}                 → Hash containing all task metadata

# Queue structures
tasks:pending:primary          → List of task IDs (new tasks)
tasks:pending:retry            → List of task IDs (retry tasks)
tasks:scheduled                → Sorted set (score=retry_time, member=task_id)

# Dead letter queue
dlq:tasks                      → List of task IDs
dlq:task:{task_id}             → Hash containing failed task details

# Metrics (optional, for monitoring)
metrics:queue:depths           → Hash of queue depths
metrics:tasks:by_state         → Hash of task counts by state
```

## Configuration

```python
# config.py
class Settings:
    # Redis
    REDIS_URL = "redis://localhost:6379/0"
    
    # Celery
    CELERY_BROKER_URL = REDIS_URL
    CELERY_RESULT_BACKEND = REDIS_URL
    
    # Task Processing
    DEFAULT_RETRY_RATIO = 0.3
    MAX_RETRIES = 3
    MAX_TASK_AGE = 3600  # 1 hour
    
    # Queue Pressure Thresholds
    RETRY_QUEUE_WARNING = 1000
    RETRY_QUEUE_CRITICAL = 5000
    
    # Circuit Breaker
    CIRCUIT_FAILURE_THRESHOLD = 0.5
    CIRCUIT_VOLUME_THRESHOLD = 10
    CIRCUIT_TIMEOUT = 60
    
    # OpenRouter
    OPENROUTER_API_KEY = os.getenv("OPENROUTER_API_KEY")
    OPENROUTER_BASE_URL = "https://openrouter.ai/api/v1"
    OPENROUTER_MODEL = "meta-llama/llama-3.2-90b-text-preview"
    OPENROUTER_TIMEOUT = 30
```

## Key Design Decisions

1. **Dual-Queue System**: Separates new tasks from retries to prevent retry storms from blocking new work while ensuring both get processed fairly.

2. **Single Source of Truth**: All task information is stored in `task:{task_id}` hash. Queues only contain task IDs, not duplicated data.

3. **Circuit Breaker**: Implemented in-memory within workers rather than Redis, reducing complexity while maintaining effectiveness.

4. **Simplified APIs**: Consolidated multiple overlapping endpoints into a minimal set that provides all necessary functionality.

5. **Celery-Native Services**: Scheduler runs as Celery beat task, leveraging existing infrastructure rather than separate services.

This architecture provides a robust, scalable system for distributed task processing with comprehensive error handling and monitoring capabilities.
