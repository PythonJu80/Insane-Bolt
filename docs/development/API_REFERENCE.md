# API Reference

## Core Endpoints

### Model Management

#### Submit Task
```http
POST /api/task
Content-Type: application/json

{
    "model": "string",
    "input": "string",
    "priority": number,
    "options": {
        "quality": number,
        "timeout": number,
        "dependencies": string[],
        "customPriority": {
            "level": 1-10,
            "reason": string,
            "deadline": string
        }
    }
}
```

#### Batch Task Submission
```http
POST /api/tasks/batch
Content-Type: application/json

{
    "tasks": [
        {
            "model": "string",
            "input": "string",
            "priority": number
        }
    ],
    "batchOptions": {
        "parallel": boolean,
        "maxConcurrent": number,
        "timeoutPerTask": number
    }
}
```

#### Get Task Status
```http
GET /api/task/:taskId
```

#### Batch Status Retrieval
```http
GET /api/tasks/statuses?ids=taskId1,taskId2,taskId3
```

### Resource Management

#### GPU Status
```http
GET /api/resources/gpu

Response:
{
    "utilization": number,
    "memory": {
        "total": number,
        "used": number,
        "free": number
    },
    "temperature": number,
    "powerUsage": number,
    "advanced": {
        "averageLoad": number,
        "uptime": number,
        "processes": [
            {
                "pid": number,
                "memory": number,
                "runtime": number
            }
        ],
        "performance": {
            "throughput": number,
            "latency": number,
            "efficiency": number
        }
    }
}
```

### Audit and Logging

#### System Logs
```http
GET /api/audit/logs
Query Parameters:
- startTime: ISO timestamp
- endTime: ISO timestamp
- type: 'task'|'api'|'system'
- level: 'info'|'warning'|'error'

Response:
{
    "logs": [
        {
            "timestamp": string,
            "type": string,
            "level": string,
            "message": string,
            "details": object
        }
    ],
    "metadata": {
        "total": number,
        "filtered": number,
        "timeRange": string
    }
}
```

#### Error Logs
```http
GET /api/errors/log
Query Parameters:
- taskId: string
- severity: 'low'|'medium'|'high'
- timeframe: string

Response:
{
    "errors": [
        {
            "id": string,
            "timestamp": string,
            "context": object,
            "stack": string,
            "resolution": string
        }
    ]
}
```

## WebSocket Events

### Connection Management
```typescript
interface WebSocketConfig {
    heartbeat: {
        interval: number,
        timeout: number
    },
    reconnect: {
        maxAttempts: number,
        backoff: {
            initial: number,
            multiplier: number,
            maxDelay: number
        }
    }
}

// Heartbeat Example
client.on('connect', () => {
    setInterval(() => {
        client.ping();
    }, config.heartbeat.interval);
});

client.on('pong', () => {
    lastPong = Date.now();
});
```

### Task Progress
```typescript
interface TaskProgress {
    taskId: string;
    progress: number;
    status: 'running' | 'complete' | 'error';
    metrics: {
        quality: number;
        latency: number;
        resourceUsage: number;
        prediction: {
            timeRemaining: number,
            resourceNeeded: number
        }
    }
}
```

## Authentication

### OAuth2 Integration
```http
POST /oauth/token
Content-Type: application/x-www-form-urlencoded

grant_type=authorization_code
&code=<code>
&redirect_uri=<redirect_uri>
&client_id=<client_id>
&client_secret=<client_secret>

Response:
{
    "access_token": string,
    "token_type": "Bearer",
    "expires_in": number,
    "refresh_token": string,
    "scope": string
}
```

### API Key Management
```http
POST /api/keys/generate
Authorization: Bearer <oauth_token>

{
    "name": string,
    "permissions": string[],
    "expiry": string
}
```

## Rate Limiting

### Headers
```http
X-RateLimit-Limit: number
X-RateLimit-Remaining: number
X-RateLimit-Reset: number
X-RateLimit-Retry-After: number
```

### Rate Limit Response
```json
{
    "error": "rate_limit_exceeded",
    "message": "Rate limit exceeded",
    "retryAfter": 3600,
    "limits": {
        "user": {
            "limit": 1000,
            "remaining": 0,
            "reset": "2024-01-01T00:00:00Z"
        },
        "ip": {
            "limit": 10000,
            "remaining": 500,
            "reset": "2024-01-01T00:00:00Z"
        }
    }
}
```

## Webhooks

### Configuration
```http
POST /api/webhooks/configure
Content-Type: application/json

{
    "url": string,
    "events": string[],
    "secret": string,
    "security": {
        "signatureAlgorithm": "sha256",
        "headers": {
            "X-Signature": string,
            "X-Timestamp": string
        }
    },
    "retries": {
        "maxAttempts": number,
        "backoffSeconds": number
    }
}
```

### Webhook Verification
```typescript
const crypto = require('crypto');

function verifyWebhook(payload: string, signature: string, secret: string): boolean {
    const hmac = crypto.createHmac('sha256', secret);
    const expectedSignature = hmac.update(payload).digest('hex');
    return crypto.timingSafeEqual(
        Buffer.from(signature),
        Buffer.from(expectedSignature)
    );
}
```

## SDK Examples

### TypeScript/JavaScript
```typescript
import { BoltClient } from '@bolt/sdk';

const client = new BoltClient({
    apiKey: 'your-api-key',
    options: {
        retry: {
            maxAttempts: 3,
            backoff: {
                initial: 1000,
                multiplier: 2,
                maxDelay: 10000
            }
        },
        timeout: 30000,
        quality: 0.95
    }
});

// Error handling
try {
    const task = await client.submitTask({
        model: 'gpt-4',
        input: 'Hello, world!'
    });
    
    task.on('error', (error) => {
        console.error('Task error:', error);
        // Implement retry logic
    });
    
    task.on('progress', (progress) => {
        console.log(`Progress: ${progress}%`);
    });
    
    const result = await task.wait();
} catch (error) {
    if (error.code === 'RATE_LIMIT_EXCEEDED') {
        const retryAfter = error.retryAfter;
        // Implement retry after delay
    }
}
```

### CLI Tool
```bash
# Install CLI
npm install -g @bolt/cli

# Configure
bolt configure --api-key=<your-api-key>

# Submit task
bolt task submit --model=gpt-4 --input="Hello, world!"

# Monitor task
bolt task monitor --id=<task-id>

# View GPU status
bolt resources gpu

# Test webhook
bolt webhook test --url=<webhook-url> --event=task.complete
