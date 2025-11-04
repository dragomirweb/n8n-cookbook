# Error Handling & Resilience - Production Guide

## Overview
Comprehensive error handling strategy for production-ready n8n workflows with graceful degradation, retry logic, and user-friendly error messages.

## Table of Contents
1. [Error Classification](#error-classification)
2. [Global Error Handler Workflow](#global-error-handler-workflow)
3. [Retry Strategies](#retry-strategies)
4. [Circuit Breaker Pattern](#circuit-breaker-pattern)
5. [User-Friendly Error Messages](#user-friendly-error-messages)
6. [Logging & Monitoring](#logging--monitoring)
7. [Recovery Mechanisms](#recovery-mechanisms)

---

## Error Classification

### Error Types & Handling

| Error Type | Severity | Retry? | User Action Required? | Example |
|-----------|----------|--------|----------------------|---------|
| **Rate Limit** | Medium | Yes (with backoff) | No | API returns 429 |
| **Timeout** | Medium | Yes (3x max) | No | Request > 60s |
| **Validation** | High | No | Sometimes | Invalid JSON from AI |
| **Auth Error** | High | No | Yes | Invalid API key |
| **Network** | Medium | Yes (5x max) | No | Connection refused |
| **Data Error** | High | No | Yes | Missing user profile |
| **System** | Critical | No | Yes (admin) | Out of memory |

### Code Node: Error Classifier

```javascript
// Code Node: Classify Error
function classifyError(error) {
    const errorPatterns = {
        rate_limit: {
            pattern: /429|rate limit|too many requests/i,
            severity: 'medium',
            retryable: true,
            user_message: '⏱️ High demand. Retrying in a moment...'
        },
        timeout: {
            pattern: /timeout|timed out/i,
            severity: 'medium',
            retryable: true,
            user_message: '⏳ Request took too long. Trying again...'
        },
        validation: {
            pattern: /validation|invalid|schema/i,
            severity: 'high',
            retryable: false,
            user_message: '❌ Invalid data received. Regenerating...'
        },
        auth: {
            pattern: /unauthorized|forbidden|invalid.*key/i,
            severity: 'high',
            retryable: false,
            user_message: '🔐 Authentication issue. Admin notified.'
        },
        network: {
            pattern: /network|connection|ECONNREFUSED/i,
            severity: 'medium',
            retryable: true,
            user_message: '🌐 Network issue. Retrying...'
        },
        provider_down: {
            pattern: /503|service unavailable/i,
            severity: 'high',
            retryable: true,
            user_message: '🔧 Service temporarily down. Switching to backup...'
        }
    };

    const errorMessage = error.message || error.toString();

    for (const [type, config] of Object.entries(errorPatterns)) {
        if (config.pattern.test(errorMessage)) {
            return {
                type,
                severity: config.severity,
                retryable: config.retryable,
                user_message: config.user_message,
                original_error: errorMessage,
                timestamp: new Date().toISOString()
            };
        }
    }

    // Unknown error
    return {
        type: 'unknown',
        severity: 'high',
        retryable: false,
        user_message: '❌ Unexpected error. Our team has been notified.',
        original_error: errorMessage,
        timestamp: new Date().toISOString()
    };
}

// Usage
const errorInfo = classifyError($input.first().json.error);
return [{ json: errorInfo }];
```

---

## Global Error Handler Workflow

### Error Trigger Workflow

```json
{
  "name": "Global Error Handler",
  "nodes": [
    {
      "name": "Error Trigger",
      "type": "n8n-nodes-base.errorTrigger",
      "position": [250, 300]
    },
    {
      "name": "Classify Error",
      "type": "n8n-nodes-base.code",
      "position": [450, 300],
      "parameters": {
        "jsCode": "// Classify error code here"
      }
    },
    {
      "name": "Switch: Error Type",
      "type": "n8n-nodes-base.switch",
      "position": [650, 300],
      "parameters": {
        "mode": "rules",
        "rules": [
          {
            "operation": "equal",
            "value1": "={{$json.type}}",
            "value2": "rate_limit",
            "output": 0
          },
          {
            "operation": "equal",
            "value1": "={{$json.type}}",
            "value2": "provider_down",
            "output": 1
          },
          {
            "operation": "equal",
            "value1": "={{$json.type}}",
            "value2": "validation",
            "output": 2
          }
        ],
        "fallbackOutput": 3
      }
    }
  ]
}
```

### Error Handler Logic

```javascript
// Code Node: Handle Error
const errorInfo = $json;
const executionData = $execution;

// Log error
await $supabase.from('error_logs').insert({
    user_id: errorInfo.user_id || null,
    error_type: errorInfo.type,
    error_message: errorInfo.original_error,
    workflow_name: executionData.workflow.name,
    node_name: executionData.lastNodeExecuted,
    severity: errorInfo.severity,
    resolved: false
});

// Determine action
let action = {
    retry: false,
    fallback: null,
    notify_user: true,
    notify_admin: false
};

switch (errorInfo.type) {
    case 'rate_limit':
        action.retry = true;
        action.retry_after_ms = 60000; // 1 minute
        break;

    case 'provider_down':
        action.fallback = 'switch_provider';
        break;

    case 'validation':
        action.fallback = 'regenerate';
        break;

    case 'auth':
        action.notify_admin = true;
        action.notify_user = true;
        break;

    default:
        action.notify_admin = errorInfo.severity === 'critical';
        break;
}

return [{
    json: {
        error: errorInfo,
        action,
        user_message: errorInfo.user_message
    }
}];
```

---

## Retry Strategies

### 1. Exponential Backoff

```javascript
// Code Node: Exponential Backoff Retry
async function retryWithExponentialBackoff(fn, maxRetries = 5, baseDelayMs = 1000) {
    const errors = [];

    for (let attempt = 0; attempt < maxRetries; attempt++) {
        try {
            const result = await fn();
            return {
                success: true,
                result,
                attempts: attempt + 1
            };

        } catch (error) {
            errors.push({
                attempt: attempt + 1,
                error: error.message,
                timestamp: new Date().toISOString()
            });

            // Check if error is retryable
            const errorInfo = classifyError(error);
            if (!errorInfo.retryable || attempt === maxRetries - 1) {
                return {
                    success: false,
                    errors,
                    final_error: error.message
                };
            }

            // Calculate backoff with jitter
            const exponentialDelay = Math.pow(2, attempt) * baseDelayMs;
            const jitter = Math.random() * 0.3 * exponentialDelay; // ±30% jitter
            const delayMs = Math.min(exponentialDelay + jitter, 30000); // Max 30s

            console.log(`Retry ${attempt + 1}/${maxRetries} after ${Math.round(delayMs)}ms`);

            await new Promise(resolve => setTimeout(resolve, delayMs));
        }
    }
}

// Usage
const result = await retryWithExponentialBackoff(async () => {
    return await callAnthropicAPI($json.request);
}, 5, 2000);

if (!result.success) {
    throw new Error(`All retries failed: ${result.final_error}`);
}

return [{ json: result.result }];
```

### 2. Retry with Fallback

```javascript
// Code Node: Retry with Provider Fallback
async function retryWithFallback(request) {
    const providers = [
        { name: 'anthropic', fn: callAnthropic, retries: 3 },
        { name: 'openai', fn: callOpenAI, retries: 3 }
    ];

    for (const provider of providers) {
        const result = await retryWithExponentialBackoff(
            () => provider.fn(request),
            provider.retries
        );

        if (result.success) {
            return {
                ...result.result,
                provider_used: provider.name,
                total_attempts: result.attempts
            };
        }

        console.log(`${provider.name} exhausted retries, trying next provider...`);
    }

    throw new Error('All providers and retries exhausted');
}

const result = await retryWithFallback($json.request);
return [{ json: result }];
```

---

## Circuit Breaker Pattern

### Implementation

```javascript
// Code Node: Circuit Breaker
class CircuitBreaker {
    constructor(threshold = 5, timeout = 60000) {
        this.threshold = threshold; // Failures before opening
        this.timeout = timeout; // Time before retry (ms)
        this.failureCount = 0;
        this.state = 'CLOSED'; // CLOSED, OPEN, HALF_OPEN
        this.nextAttempt = Date.now();
    }

    async execute(fn) {
        if (this.state === 'OPEN') {
            if (Date.now() < this.nextAttempt) {
                throw new Error('Circuit breaker is OPEN');
            }
            this.state = 'HALF_OPEN';
        }

        try {
            const result = await fn();
            this.onSuccess();
            return result;

        } catch (error) {
            this.onFailure();
            throw error;
        }
    }

    onSuccess() {
        this.failureCount = 0;
        this.state = 'CLOSED';
    }

    onFailure() {
        this.failureCount++;

        if (this.failureCount >= this.threshold) {
            this.state = 'OPEN';
            this.nextAttempt = Date.now() + this.timeout;
            console.log(`Circuit breaker OPEN for ${this.timeout}ms`);
        }
    }

    getState() {
        return {
            state: this.state,
            failures: this.failureCount,
            next_attempt: new Date(this.nextAttempt).toISOString()
        };
    }
}

// Global circuit breakers (stored in workflow static data)
const breakers = {
    anthropic: new CircuitBreaker(5, 60000),
    openai: new CircuitBreaker(5, 60000)
};

// Usage
try {
    const result = await breakers.anthropic.execute(async () => {
        return await callAnthropicAPI($json.request);
    });

    return [{ json: { result, breaker_state: breakers.anthropic.getState() } }];

} catch (error) {
    if (error.message === 'Circuit breaker is OPEN') {
        // Try fallback provider
        return await breakers.openai.execute(async () => {
            return await callOpenAI($json.request);
        });
    }
    throw error;
}
```

---

## User-Friendly Error Messages

### Error Message Templates

```javascript
// Code Node: Generate User Message
function getUserFriendlyMessage(errorType, context = {}) {
    const messages = {
        rate_limit: {
            emoji: '⏱️',
            title: 'High Demand',
            body: 'Our AI is busy helping others. Retrying your request in a moment...',
            action: null
        },
        timeout: {
            emoji: '⏳',
            title: 'Taking Longer Than Expected',
            body: 'Your request is taking longer than usual. Optimizing and trying again...',
            action: null
        },
        provider_down: {
            emoji: '🔧',
            title: 'Service Issue',
            body: 'Primary service temporarily unavailable. Switching to backup AI...',
            action: 'Please wait 10-15 seconds'
        },
        validation: {
            emoji: '🔄',
            title: 'Regenerating',
            body: 'The AI response had an issue. Generating a new meal for you...',
            action: null
        },
        auth: {
            emoji: '🔐',
            title: 'System Error',
            body: 'Technical issue detected. Our team has been notified.',
            action: 'Please try again in a few minutes'
        },
        unknown: {
            emoji: '❌',
            title: 'Unexpected Error',
            body: 'Something unexpected happened. Our team is investigating.',
            action: 'Try /help for available commands'
        }
    };

    const template = messages[errorType] || messages.unknown;

    let message = `${template.emoji} *${template.title}*\n\n${template.body}`;

    if (template.action) {
        message += `\n\n_${template.action}_`;
    }

    if (context.request_id) {
        message += `\n\nRequest ID: \`${context.request_id}\``;
    }

    return {
        text: message,
        parse_mode: 'Markdown',
        inline_keyboard: [
            [
                { text: '🔄 Try Again', callback_data: 'action:retry' },
                { text: '📞 Contact Support', callback_data: 'action:support' }
            ]
        ]
    };
}

// Usage
const userMessage = getUserFriendlyMessage($json.error_type, {
    request_id: $json.request_id
});

return [{ json: userMessage }];
```

---

## Logging & Monitoring

### Structured Logging

```javascript
// Code Node: Structured Error Logging
function logError(error, context) {
    const logEntry = {
        timestamp: new Date().toISOString(),
        level: 'error',
        error_type: error.type,
        error_message: error.original_error,
        severity: error.severity,

        // Context
        user_id: context.user_id,
        chat_id: context.chat_id,
        workflow_id: $workflow.id,
        workflow_name: $workflow.name,
        execution_id: $execution.id,
        node_name: context.node_name,

        // Request details
        request_id: context.request_id,
        request_type: context.request_type,

        // Stack trace (if available)
        stack_trace: error.stack || null,

        // Metadata
        environment: $env.NODE_ENV || 'production',
        version: $env.APP_VERSION || '1.0.0'
    };

    // Console log (JSON format for log aggregation tools)
    console.error(JSON.stringify(logEntry));

    // Save to database
    $supabase.from('error_logs').insert(logEntry);

    // Send to monitoring service (optional)
    if (error.severity === 'critical') {
        sendToMonitoring(logEntry);
    }

    return logEntry;
}

// Usage
const logEntry = logError($json.error, {
    user_id: $json.user_id,
    chat_id: $json.chat_id,
    node_name: 'AI Generation',
    request_id: $json.request_id,
    request_type: 'single_meal'
});

return [{ json: logEntry }];
```

### Admin Alerts

```javascript
// Code Node: Send Admin Alert
async function sendAdminAlert(errorLog) {
    const ADMIN_CHAT_IDS = [123456789]; // Telegram admin user IDs

    if (errorLog.severity !== 'critical' && errorLog.severity !== 'high') {
        return; // Only alert for serious errors
    }

    const alertMessage = `
🚨 *${errorLog.severity.toUpperCase()} ERROR*

*Type*: ${errorLog.error_type}
*Workflow*: ${errorLog.workflow_name}
*Node*: ${errorLog.node_name}

*Message*:
\`${errorLog.error_message}\`

*User*: ${errorLog.user_id || 'N/A'}
*Time*: ${errorLog.timestamp}
*Request ID*: ${errorLog.request_id}

*Environment*: ${errorLog.environment}
`.trim();

    // Send to all admins
    for (const adminId of ADMIN_CHAT_IDS) {
        await sendTelegramMessage(adminId, alertMessage, 'Markdown');
    }

    // Also send to Slack/Discord if configured
    if ($env.SLACK_WEBHOOK_URL) {
        await sendSlackAlert(errorLog);
    }
}

await sendAdminAlert($json.error_log);
```

---

## Recovery Mechanisms

### 1. Automatic Recovery

```javascript
// Code Node: Attempt Recovery
async function attemptRecovery(error, originalRequest) {
    const recoveryStrategies = {
        rate_limit: async () => {
            await new Promise(resolve => setTimeout(resolve, 60000));
            return await callAPI(originalRequest);
        },

        validation: async () => {
            // Regenerate with stricter validation
            const strictRequest = {
                ...originalRequest,
                temperature: 0.2, // More deterministic
                max_retries: 2
            };
            return await callAPI(strictRequest);
        },

        provider_down: async () => {
            // Switch to fallback provider
            return await callFallbackAPI(originalRequest);
        },

        timeout: async () => {
            // Reduce complexity
            const simplifiedRequest = {
                ...originalRequest,
                max_tokens: Math.floor(originalRequest.max_tokens * 0.7)
            };
            return await callAPI(simplifiedRequest);
        }
    };

    const strategy = recoveryStrategies[error.type];

    if (strategy) {
        console.log(`Attempting recovery for ${error.type}...`);
        return await strategy();
    }

    throw new Error(`No recovery strategy for ${error.type}`);
}

// Usage
try {
    const result = await attemptRecovery($json.error, $json.original_request);
    return [{ json: { recovered: true, result } }];

} catch (recoveryError) {
    // Recovery failed, use fallback response
    return [{
        json: {
            recovered: false,
            fallback_used: true,
            message: 'Using cached similar meal...'
        }
    }];
}
```

### 2. Graceful Degradation

```javascript
// Code Node: Graceful Degradation
async function degradeGracefully(error, context) {
    const degradationLevels = [
        {
            name: 'cached_similar',
            fn: async () => {
                // Return similar cached meal
                const cached = await getCachedSimilarMeal(context.user_preferences);
                if (cached) {
                    return {
                        success: true,
                        meal: cached,
                        note: '✨ Curated meal from your previous preferences'
                    };
                }
                throw new Error('No cached meals available');
            }
        },
        {
            name: 'template_based',
            fn: async () => {
                // Generate from template (no AI)
                const meal = generateTemplateBasedMeal(context.user_preferences);
                return {
                    success: true,
                    meal,
                    note: '📋 Template-based meal matching your preferences'
                };
            }
        },
        {
            name: 'manual_request',
            fn: async () => {
                // Queue for manual generation
                await queueManualRequest(context);
                return {
                    success: true,
                    message: '📝 Your request has been queued. We\'ll notify you when ready (usually < 1 hour).'
                };
            }
        }
    ];

    for (const level of degradationLevels) {
        try {
            console.log(`Trying degradation level: ${level.name}`);
            return await level.fn();

        } catch (err) {
            console.log(`${level.name} failed, trying next level...`);
            continue;
        }
    }

    // Ultimate fallback
    return {
        success: false,
        message: '❌ All systems temporarily unavailable. Please try again in 10 minutes.'
    };
}

const fallback = await degradeGracefully($json.error, {
    user_id: $json.user_id,
    user_preferences: $json.user_profile
});

return [{ json: fallback }];
```

---

## Error Prevention

### 1. Input Validation

```javascript
// Code Node: Validate Inputs
function validateInputs(data) {
    const errors = [];

    // Required fields
    const required = ['user_id', 'chat_id', 'message_text'];
    required.forEach(field => {
        if (!data[field]) {
            errors.push(`Missing required field: ${field}`);
        }
    });

    // Data type validation
    if (data.user_id && typeof data.user_id !== 'string') {
        errors.push('user_id must be a string');
    }

    // Length validation
    if (data.message_text && data.message_text.length > 4000) {
        errors.push('message_text exceeds maximum length (4000 chars)');
    }

    // Sanitization
    if (data.message_text) {
        data.message_text = data.message_text.trim();
    }

    if (errors.length > 0) {
        throw new Error(`Validation failed: ${errors.join(', ')}`);
    }

    return data;
}

const validated = validateInputs($json);
return [{ json: validated }];
```

### 2. Health Checks

```javascript
// Separate Workflow: Health Check (runs every 5 minutes)
async function healthCheck() {
    const checks = {
        anthropic: false,
        openai: false,
        supabase: false,
        redis: false
    };

    // Check Anthropic
    try {
        await callAnthropicAPI({ test: true, max_tokens: 10 });
        checks.anthropic = true;
    } catch (err) {
        console.error('Anthropic health check failed:', err.message);
    }

    // Check OpenAI
    try {
        await callOpenAI({ test: true, max_tokens: 10 });
        checks.openai = true;
    } catch (err) {
        console.error('OpenAI health check failed:', err.message);
    }

    // Check Supabase
    try {
        await $supabase.from('users').select('id').limit(1);
        checks.supabase = true;
    } catch (err) {
        console.error('Supabase health check failed:', err.message);
    }

    // Check Redis
    try {
        await $redis.ping();
        checks.redis = true;
    } catch (err) {
        console.error('Redis health check failed:', err.message);
    }

    // Alert if critical services down
    const criticalDown = !checks.supabase || (!checks.anthropic && !checks.openai);

    if (criticalDown) {
        await sendAdminAlert({
            severity: 'critical',
            error_type: 'health_check_failed',
            error_message: `Critical services down: ${JSON.stringify(checks)}`,
            timestamp: new Date().toISOString()
        });
    }

    return checks;
}

const status = await healthCheck();
return [{ json: { health: status, timestamp: new Date().toISOString() } }];
```

---

## Next Steps

1. ✅ Implement global error handler workflow
2. 🧪 Test all error scenarios
3. 📊 Set up monitoring dashboard
4. 🚨 Configure admin alerts
5. 🚀 Proceed to [Cost Optimization](./07-cost-optimization.md)

---

## Version History

- **v1.0** (2025-01-15): Initial error handling guide
- **v1.1** (Pending): Add distributed tracing
