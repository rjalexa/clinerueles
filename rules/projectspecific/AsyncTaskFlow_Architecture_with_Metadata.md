# AsyncTaskFlow Architecture

This document describes the high-level architecture of the AsyncTaskFlow system.

## System Overview

AsyncTaskFlow is a production-ready distributed task processing system built with FastAPI, Redis, and Celery. It implements a robust architecture for handling long-running text summarization tasks via OpenRouter API.

## Components

### API Service (`src/api/`)
- FastAPI application serving REST endpoints
- Task management and queue monitoring
- Health checks and system status

### Worker Service (`src/worker/`)
- Celery workers for task execution
- Circuit breaker pattern implementation
- Retry logic and error handling
- **Token bucket rate limiting for API usage compliance**

### Frontend (`frontend/`)
- React/TypeScript web interface
- Real-time task monitoring
- Queue management dashboard

### Infrastructure
- Redis for message brokering and caching
- Docker containers for all services
- Nginx for frontend serving and API proxying

## Data Flow

1. User submits task via frontend
2. API creates task record in Redis
3. Task queued for worker processing
4. Worker executes task with retry logic and rate limiting
5. Results stored and status updated
6. Frontend displays real-time progress

## Rate Limiting Architecture (Token Bucket)

To prevent parallel Celery workers from breaching the rate limits of the external OpenRouter API, AsyncTaskFlow implements a centralized token bucket mechanism.

### How It Works

- A shared token bucket is maintained in Redis.
- The bucket refills at a fixed interval according to the API’s maximum allowed request rate.
- Before making an API request, each Celery worker must:
  1. Atomically acquire a token from the bucket.
  2. If a token is available, proceed with the request.
  3. If no token is available, wait or requeue with backoff.

### Advantages

- Global coordination across distributed workers.
- Prevents accidental bursting or overconsumption.
- Easily configurable for different API quotas (e.g., 60 req/min or 1 req/sec).
- Compatible with retry strategies and circuit breaker patterns already in place.

### Implementation Notes

- Token bucket logic resides in `src/worker/ratelimit.py`.
- Uses Redis LUA scripting or atomic pipeline operations to ensure thread safety.
- Integrated with the task retry and backoff strategy to avoid hard failures.

## Schema for PDF Extraction Tasks

Example JSON Output:
```json
{
  "filename": "giornale_1950-05-12_p1.png",
  "issue_date": "1950-05-12",
  "pages": [
    {
      "page_number": 1,
      "status": "processed",
      "reason": "",
      "articles": [
        {
          "title": "Titolo dell'articolo",
          "subtitle": "Sottotitolo",
          "author": "Mario Rossi",
          "body": "Testo completo dell'articolo senza interruzioni.",
          "topics": ["politica", "Italia", "democrazia"],
          "summary": "Breve riassunto del contenuto dell’articolo."
        }
      ]
    },
    {
      "page_number": 4,
      "status": "skipped",
      "reason": "conversion failed",
      "articles": []
    }
  ]
}
```

### Pydantic Model:
```python
from pydantic import BaseModel
from typing import List

class Article(BaseModel):
    title: str = ""
    subtitle: str = ""
    author: str = ""
    body: str
    topics: List[str] = []
    summary: str = ""

class Page(BaseModel):
    page_number: int
    status: str = "processed"  # "processed" or "skipped"
    reason: str = ""
    articles: List[Article]

class NewspaperPage(BaseModel):
    filename: str
    issue_date: str  # ISO 8601 format
    pages: List[Page]
```

## Key Features

- Dual-queue system (primary/retry)
- Circuit breaker protection
- Dead letter queue management
- Exponential backoff retry strategy
- Token bucket rate limiting mechanism
- Per-page processing metadata and fault tolerance
- Comprehensive monitoring and health checks

For detailed implementation information, see the project specification in `.clinerules/project-definition.md`.
