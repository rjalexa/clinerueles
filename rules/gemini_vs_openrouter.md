# OpenRouter vs Google Gemini API: Complete Error Codes, Rate Limits, and Monitoring Comparison

## Table of Contents
1. [Error Codes Comparison](#error-codes-comparison)
2. [Rate Limiting Comparison](#rate-limiting-comparison)
3. [Monitoring and Visibility Endpoints](#monitoring-and-visibility-endpoints)
4. [Key Differences](#key-differences)
5. [Migration Considerations](#migration-considerations)
6. [Implementation Examples](#implementation-examples)

## Error Codes Comparison

### HTTP Status Codes

| Error Type | OpenRouter | Google Gemini API | Notes |
|------------|------------|-------------------|-------|
| **Bad Request** | 400 | 400 | Both use standard HTTP codes |
| **Unauthorized** | 401 | 401 | Authentication failures |
| **Payment Required** | 402 | N/A | OpenRouter specific for credit issues |
| **Forbidden** | 403 | 403 | Permission/access denied |
| **Not Found** | 404 | 404 | Resource not found |
| **Rate Limited** | 429 | 429 | Both use 429 for rate limits |
| **Client Cancelled** | N/A | 499 | Gemini specific |
| **Internal Error** | 500 | 500 | Server errors |
| **Service Unavailable** | 503 | 503 | Temporary unavailability |
| **Timeout** | N/A | 504 | Gemini uses 504 for deadline exceeded |

### Error Response Formats

#### OpenRouter Error Response
```json
{
  "error": {
    "code": 400,
    "message": "Bad Request: Invalid parameters",
    "type": "invalid_request_error",
    "metadata": {
      // Additional context based on error type
    }
  }
}
```

#### Gemini Error Response
```json
{
  "error": {
    "code": 400,
    "message": "Request contains an invalid argument.",
    "status": "INVALID_ARGUMENT",
    "details": [
      {
        "@type": "type.googleapis.com/google.rpc.ErrorInfo",
        "reason": "SPECIFIC_REASON",
        "domain": "googleapis.com",
        "metadata": {
          "method": "google.cloud.aiplatform.v1beta1.PredictionService.GenerateContent",
          "service": "aiplatform.googleapis.com"
        }
      }
    ]
  }
}
```

## Detailed Error Classifications

### OpenRouter Error Categories

1. **Request Errors (4xx)**
   - 400: Invalid or missing parameters, CORS issues
   - 401: OAuth session expired, disabled/invalid API key
   - 402: Insufficient credits (unique to OpenRouter)
   - 403: Model requires moderation and input was flagged

2. **Server Errors (2xx/5xx)**
   - 200 with error in body: LLM processing errors
   - 5xx: Provider errors, fallback triggered

3. **Special Error Metadata**
   - Moderation errors include flagging details
   - Provider errors include provider-specific information

### Gemini Error Categories

1. **Client Errors (4xx)**
   - 400 INVALID_ARGUMENT: Request validation failed
   - 400 FAILED_PRECONDITION: Model requires allowlisting
   - 401 UNAUTHENTICATED: API key issues, OAuth required
   - 403 FORBIDDEN: Access denied by organization policy
   - 429 RESOURCE_EXHAUSTED: Rate limit exceeded

2. **Server Errors (5xx)**
   - 500 UNKNOWN/INTERNAL: Server overload or dependency failure
   - 503 UNAVAILABLE: Service temporarily unavailable
   - 504 DEADLINE_EXCEEDED: Request timeout

## Rate Limiting Comparison

### OpenRouter Rate Limits

**Free Tier:**
- 20 requests per minute for free models
- Daily limits:
  - < 10 credits purchased: 50 free requests/day
  - ≥ 10 credits purchased: 1000 free requests/day

**Paid Tier:**
- No fixed rate limits mentioned
- Based on account credits
- DDoS protection via Cloudflare

### Gemini Rate Limits

**Free Tier (Tier 1):**
- Varies by model (e.g., Gemini 1.5 Pro: 2 RPM)
- Tokens per minute (TPM) limits
- Requests per day (RPD) limits

**Paid Tiers:**
- **Tier 2**: Based on cumulative Google Cloud spending
- **Tier 3**: Higher spending thresholds
- Dynamic Shared Quota (DSQ) for newer models

**Rate Limit Dimensions:**
- Requests per minute (RPM)
- Tokens per minute (TPM)
- Tokens per day (TPD)
- Images per minute (IPM) for image models

## Monitoring and Visibility Endpoints

### OpenRouter Monitoring Capabilities

#### 1. **API Key Status Endpoint**
```bash
GET https://openrouter.ai/api/v1/auth/key
```
Returns:
- Current credit balance
- Rate limit status
- API key validity

#### 2. **Generation Details Endpoint**
```bash
GET https://openrouter.ai/api/v1/generation/{id}
```
Returns:
- Native token counts (not normalized)
- Actual cost calculation
- Model-specific token details
- Works for both streaming and non-streaming requests

#### 3. **Usage Accounting (In-Response)**
Include in request:
```json
{
  "usage": {
    "include": true
  }
}
```
Returns in response:
```json
{
  "usage": {
    "prompt_tokens": 123,
    "completion_tokens": 456,
    "total_tokens": 579,
    "cached_tokens": 0,
    "native_tokens": {
      "prompt": 125,
      "completion": 458,
      "total": 583
    }
  }
}
```

#### 4. **Available Models Endpoint**
```bash
GET https://openrouter.ai/api/v1/models
```
Returns available models with pricing and capabilities

#### 5. **Model Endpoints**
```bash
GET https://openrouter.ai/api/v1/models/{author}/{slug}/endpoints
```
Returns available providers for a specific model

### Gemini Monitoring Capabilities

#### 1. **Count Tokens API**
```bash
POST https://{LOCATION}-aiplatform.googleapis.com/v1/projects/{PROJECT_ID}/locations/{LOCATION}/publishers/google/models/{MODEL_ID}:countTokens
```
Returns:
```json
{
  "totalTokens": 31,
  "totalBillableCharacters": 96,
  "promptTokensDetails": [
    {
      "modality": "TEXT",
      "tokenCount": 31
    }
  ]
}
```
- No charge for counting tokens
- 3000 RPM limit for countTokens API
- Supports multimodal content (text, images, video)

#### 2. **Usage Metadata (In-Response)**
Automatically included in all responses:
```json
{
  "usageMetadata": {
    "promptTokenCount": 123,
    "candidatesTokenCount": 456,
    "totalTokenCount": 579,
    "cachedContentTokenCount": 0
  }
}
```

#### 3. **Google Cloud Monitoring**
- **API Dashboard**: Basic metrics visualization
- **Metrics Explorer**: Advanced filtering and aggregation
- Available metrics:
  - Request counts
  - Error rates
  - Latencies (total and backend)
  - Request/response sizes
  - Custom metrics via Cloud Logging

#### 4. **Cloud Logging Integration**
- Detailed request/response logging (opt-in)
- Custom metric creation from logs
- BigQuery export for analysis
- Log-based alerting

#### 5. **Vertex AI Monitoring Dashboard**
Automatic dashboards showing:
- Daily/weekly/monthly active users
- Response counts by method
- Token usage by modality
- Error distribution
- Latency percentiles

### Monitoring Features Comparison

| Feature | OpenRouter | Gemini API |
|---------|------------|------------|
| **Token Counting** | Via generation endpoint or inline | Dedicated countTokens API + inline |
| **Cost Calculation** | Native token-based pricing | Token + character-based billing |
| **Real-time Metrics** | Limited to request response | Full Cloud Monitoring suite |
| **Historical Analysis** | Not built-in | Cloud Logging + BigQuery |
| **Custom Dashboards** | No | Yes (Cloud Monitoring) |
| **Alerting** | No | Yes (Cloud Monitoring) |
| **Distributed Tracing** | No | Yes (Cloud Trace) |
| **Log Export** | No | Multiple destinations |

## Key Differences

### 1. **Payment/Credit System**
- **OpenRouter**: Uses credit system with 402 errors for insufficient credits
- **Gemini**: Uses Google Cloud Billing, no specific payment error code

### 2. **Rate Limit Granularity**
- **OpenRouter**: Simple RPM + daily limits based on credit tier
- **Gemini**: Complex multi-dimensional limits (RPM, TPM, TPD, IPM)

### 3. **Monitoring Depth**
- **OpenRouter**: Basic usage tracking and cost calculation
- **Gemini**: Comprehensive monitoring with Cloud Monitoring, Logging, and BigQuery integration

### 4. **Error Detail Level**
- **OpenRouter**: Simpler error structure with metadata
- **Gemini**: Detailed error structure with canonical codes and type information

### 5. **Fallback Behavior**
- **OpenRouter**: Automatic provider fallback on 5xx errors
- **Gemini**: No automatic fallback, manual retry required

### 6. **Visibility Tools**
- **OpenRouter**: API-based monitoring only
- **Gemini**: Full observability stack (dashboards, logs, traces, metrics)

## Migration Considerations

### Error Handling Updates
```javascript
// OpenRouter to Gemini error mapping
const mapOpenRouterToGeminiError = (openRouterError) => {
  const errorMap = {
    402: { code: 429, status: 'RESOURCE_EXHAUSTED', reason: 'INSUFFICIENT_CREDITS' },
    // Add other mappings
  };
  
  return errorMap[openRouterError.code] || {
    code: openRouterError.code,
    status: 'UNKNOWN',
    reason: 'UNMAPPED_ERROR'
  };
};
```

### Rate Limit Management
```javascript
// Multi-dimensional rate limit tracker for Gemini
class GeminiRateLimiter {
  constructor() {
    this.limits = {
      rpm: { count: 0, limit: 60, resetTime: null },
      tpm: { count: 0, limit: 60000, resetTime: null },
      tpd: { count: 0, limit: 1000000, resetTime: null }
    };
  }
  
  async checkLimits(tokenCount) {
    const now = Date.now();
    
    // Reset counters if needed
    Object.keys(this.limits).forEach(key => {
      const limit = this.limits[key];
      const resetInterval = key === 'tpd' ? 86400000 : 60000; // day vs minute
      
      if (!limit.resetTime || now > limit.resetTime) {
        limit.count = 0;
        limit.resetTime = now + resetInterval;
      }
    });
    
    // Check all limits
    if (this.limits.rpm.count >= this.limits.rpm.limit) {
      throw new Error('RPM limit exceeded');
    }
    if (this.limits.tpm.count + tokenCount > this.limits.tpm.limit) {
      throw new Error('TPM limit exceeded');
    }
    if (this.limits.tpd.count + tokenCount > this.limits.tpd.limit) {
      throw new Error('TPD limit exceeded');
    }
    
    // Update counters
    this.limits.rpm.count++;
    this.limits.tpm.count += tokenCount;
    this.limits.tpd.count += tokenCount;
  }
}
```

### Monitoring Integration
```javascript
// Gemini monitoring setup
const setupGeminiMonitoring = async () => {
  // 1. Enable Cloud Logging for detailed request tracking
  const logging = new Logging();
  const log = logging.log('gemini-api-requests');
  
  // 2. Create custom metrics
  const monitoring = new Monitoring();
  const metricDescriptor = {
    type: 'custom.googleapis.com/gemini/token_usage',
    metricKind: 'GAUGE',
    valueType: 'INT64',
    description: 'Token usage per request'
  };
  
  // 3. Set up alerts
  const alertPolicy = {
    displayName: 'High Token Usage Alert',
    conditions: [{
      displayName: 'Token usage exceeds threshold',
      conditionThreshold: {
        filter: 'metric.type="custom.googleapis.com/gemini/token_usage"',
        comparison: 'COMPARISON_GT',
        thresholdValue: 10000,
        duration: '60s'
      }
    }]
  };
  
  return { log, monitoring, alertPolicy };
};
```

## Implementation Examples

### Complete Error Handler
```javascript
class GeminiErrorHandler {
  constructor() {
    this.retryDelays = {
      429: 60000,  // Rate limit: 1 minute
      500: 5000,   // Server error: 5 seconds
      503: 10000,  // Unavailable: 10 seconds
      504: 2000    // Timeout: 2 seconds
    };
    this.maxRetries = 2;
  }
  
  async handleRequest(requestFn, retryCount = 0) {
    try {
      // Count tokens before request
      const tokenCount = await this.countTokens(requestFn.prompt);
      
      // Make request with monitoring
      const startTime = Date.now();
      const response = await requestFn();
      
      // Log successful request
      await this.logRequest({
        success: true,
        latency: Date.now() - startTime,
        tokens: response.usageMetadata,
        model: requestFn.model
      });
      
      return response;
      
    } catch (error) {
      const errorCode = error.error?.code;
      const shouldRetry = this.shouldRetry(errorCode, retryCount);
      
      // Log error
      await this.logRequest({
        success: false,
        error: error.error,
        retryCount,
        willRetry: shouldRetry
      });
      
      if (shouldRetry) {
        const delay = this.retryDelays[errorCode] || 1000;
        await new Promise(resolve => setTimeout(resolve, delay));
        return this.handleRequest(requestFn, retryCount + 1);
      }
      
      throw error;
    }
  }
  
  shouldRetry(errorCode, retryCount) {
    const retryableCodes = [429, 500, 503, 504];
    return retryableCodes.includes(errorCode) && retryCount < this.maxRetries;
  }
  
  async countTokens(prompt) {
    // Implementation of token counting
  }
  
  async logRequest(data) {
    // Log to Cloud Logging or custom monitoring system
  }
}
```

### Usage Tracking System
```javascript
class UsageTracker {
  constructor(projectId) {
    this.projectId = projectId;
    this.monitoring = new Monitoring.MetricServiceClient();
  }
  
  async trackUsage(request, response) {
    const dataPoint = {
      interval: {
        endTime: {
          seconds: Date.now() / 1000
        }
      },
      value: {
        int64Value: response.usageMetadata.totalTokenCount
      }
    };
    
    const timeSeriesData = {
      metric: {
        type: 'custom.googleapis.com/gemini/token_usage',
        labels: {
          model: request.model,
          method: 'generateContent'
        }
      },
      resource: {
        type: 'global',
        labels: {
          project_id: this.projectId
        }
      },
      points: [dataPoint]
    };
    
    await this.monitoring.createTimeSeries({
      name: this.monitoring.projectPath(this.projectId),
      timeSeries: [timeSeriesData]
    });
  }
}
```

## Recommendations

1. **Implement comprehensive error mapping** between OpenRouter and Gemini error codes
2. **Add token counting** to prevent hitting TPM limits before RPM
3. **Set up Cloud Monitoring** dashboards for real-time visibility
4. **Use Cloud Logging** for detailed request/response tracking
5. **Implement circuit breakers** for 503 errors
6. **Create custom metrics** for business-specific KPIs
7. **Set up alerting** for abnormal usage patterns
8. **Use BigQuery** for long-term usage analysis
9. **Consider Provisioned Throughput** for production workloads
10. **Implement proper retry logic** with exponential backoff