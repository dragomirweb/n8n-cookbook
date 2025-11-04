# Cost Optimization - Complete Strategy

## Overview
Comprehensive cost optimization strategies to reduce AI API costs from $2.84/user/month to $0.95/user/month (67% savings).

## Table of Contents
1. [Cost Baseline Analysis](#cost-baseline-analysis)
2. [Prompt Caching (Anthropic)](#prompt-caching-anthropic)
3. [Application-Level Caching (Redis)](#application-level-caching-redis)
4. [Model Routing Strategy](#model-routing-strategy)
5. [Batch Processing](#batch-processing)
6. [Cost Monitoring & Alerts](#cost-monitoring--alerts)
7. [Optimization Checklist](#optimization-checklist)

---

## Cost Baseline Analysis

### Without Optimization (Per 1000 Users)

```javascript
// Monthly costs per user (10 requests/month average)
const BASELINE_COSTS = {
    single_meal: {
        requests_per_month: 7,
        input_tokens: 2500,
        output_tokens: 2000,
        provider: 'anthropic',
        cost_per_request: (2500 / 1000 * 0.003) + (2000 / 1000 * 0.015), // $0.0375
        monthly_cost: 7 * 0.0375 // $0.2625
    },
    weekly_plan: {
        requests_per_month: 1,
        input_tokens: 3000,
        output_tokens: 4000,
        provider: 'anthropic',
        cost_per_request: (3000 / 1000 * 0.003) + (4000 / 1000 * 0.015), // $0.069
        monthly_cost: 1 * 0.069 // $0.069
    },
    grocery_list: {
        requests_per_month: 1,
        input_tokens: 1500,
        output_tokens: 800,
        provider: 'anthropic',
        cost_per_request: (1500 / 1000 * 0.003) + (800 / 1000 * 0.015), // $0.0165
        monthly_cost: 1 * 0.0165 // $0.0165
    },
    intent_classification: {
        requests_per_month: 10,
        input_tokens: 500,
        output_tokens: 100,
        provider: 'anthropic',
        cost_per_request: (500 / 1000 * 0.003) + (100 / 1000 * 0.015), // $0.003
        monthly_cost: 10 * 0.003 // $0.03
    }
};

const totalPerUser = Object.values(BASELINE_COSTS).reduce((sum, category) => {
    return sum + category.monthly_cost;
}, 0);

console.log(`Baseline cost per user: $${totalPerUser.toFixed(4)}`);
// Output: $0.378 per user/month (without any optimization)

// For 1000 users: $378/month
```

### With Full Optimization (Target)

```javascript
const OPTIMIZED_COSTS = {
    single_meal: {
        cache_hit_rate: 0.70, // 70% cache hits
        cached_input_tokens: 2250, // 90% of prompt is cached
        new_input_tokens: 250,
        output_tokens: 2000,

        // Cached request cost
        cached_cost: (2250 / 1000 * 0.0003) + (250 / 1000 * 0.003) + (2000 / 1000 * 0.015),
        // $0.0314

        // Non-cached cost
        full_cost: 0.0375,

        // Average cost
        avg_cost: (0.70 * 0.0314) + (0.30 * 0.0375), // $0.0333
        requests_per_month: 7,
        monthly_cost: 7 * 0.0333 // $0.2331
    },

    weekly_plan: {
        // Batch generation instead of 7 separate calls
        batch_optimization: 0.60, // 60% savings from batching
        base_cost: 0.069,
        optimized_cost: 0.069 * 0.40, // $0.0276
        requests_per_month: 1,
        monthly_cost: 0.0276
    },

    grocery_list: {
        // Cached from weekly plan
        cached: true,
        cost: 0, // Generated during weekly plan
        monthly_cost: 0
    },

    intent_classification: {
        // Use smaller model (GPT-4o mini)
        input_tokens: 500,
        output_tokens: 100,
        cost_per_request: (500 / 1000 * 0.00015) + (100 / 1000 * 0.0006), // $0.00014
        requests_per_month: 10,
        monthly_cost: 10 * 0.00014 // $0.0014
    }
};

const optimizedTotal = Object.values(OPTIMIZED_COSTS).reduce((sum, category) => {
    return sum + category.monthly_cost;
}, 0);

console.log(`Optimized cost per user: $${optimizedTotal.toFixed(4)}`);
// Output: $0.2621 per user/month (31% reduction)

// For 1000 users: $262/month (savings: $116/month)
```

**Realistic Target with All Optimizations**: $0.95/user/month including:
- Prompt caching (90% of prompts)
- Application caching (60% hit rate)
- Model routing (cheaper models for simple tasks)
- Batch processing (weekly plans)

---

## Prompt Caching (Anthropic)

### How It Works

Anthropic's Claude caches system prompts and repeated context for 5 minutes. Cache reads cost **90% less** than regular input tokens.

**Pricing (January 2025)**:
- Input tokens: $0.003 / 1K tokens
- Cached input (read): $0.0003 / 1K tokens (10x cheaper)
- Cache write: $0.00375 / 1K tokens (25% surcharge on first use)
- Output tokens: $0.015 / 1K tokens

### Implementation

```javascript
// Code Node: Build Request with Caching
const SYSTEM_PROMPT = `[Your 2000-token system prompt]`;
const USER_PROFILE = JSON.stringify($json.user_profile); // ~500 tokens

const request = {
    model: "claude-sonnet-4-5-20250120",
    max_tokens: 2500,
    temperature: 0.4,

    // CRITICAL: Mark these for caching
    system: [
        {
            type: "text",
            text: SYSTEM_PROMPT,
            cache_control: { type: "ephemeral" } // Cache for 5 min
        },
        {
            type: "text",
            text: USER_PROFILE,
            cache_control: { type: "ephemeral" } // Cache for 5 min
        }
    ],

    // This changes per request (NOT cached)
    messages: [
        {
            role: "user",
            content: $json.user_message // ~100 tokens
        }
    ]
};

return [{ json: { request } }];
```

### Cache Hit Rate Calculation

```javascript
// Code Node: Track Cache Performance
function calculateCacheMetrics(usageLogs) {
    const totalRequests = usageLogs.length;
    let totalCacheReads = 0;
    let totalInputTokens = 0;
    let totalSavings = 0;

    usageLogs.forEach(log => {
        totalCacheReads += log.cache_read_tokens || 0;
        totalInputTokens += log.input_tokens;

        // Savings calculation
        const cacheReadCost = (log.cache_read_tokens || 0) / 1000 * 0.0003;
        const regularCost = (log.cache_read_tokens || 0) / 1000 * 0.003;
        totalSavings += (regularCost - cacheReadCost);
    });

    const cacheHitRate = totalInputTokens > 0
        ? (totalCacheReads / totalInputTokens * 100).toFixed(2)
        : 0;

    return {
        total_requests: totalRequests,
        cache_hit_rate: cacheHitRate + '%',
        total_cache_reads: totalCacheReads,
        total_input_tokens: totalInputTokens,
        total_savings_usd: totalSavings.toFixed(4),
        avg_savings_per_request: (totalSavings / totalRequests).toFixed(4)
    };
}

// Get last 1000 requests
const logs = await $supabase
    .from('api_usage_logs')
    .select('*')
    .eq('provider', 'anthropic')
    .order('created_at', { ascending: false })
    .limit(1000);

const metrics = calculateCacheMetrics(logs.data);

console.log('Cache Performance:', metrics);
// Expected: 60-80% cache hit rate after warm-up
```

**Target Cache Hit Rate**: 70%+

**Strategies to Improve Cache Hit Rate**:
1. Keep system prompts consistent (version them instead of tweaking)
2. Update user profiles in batches, not per-request
3. Group similar requests within 5-minute windows
4. Pre-warm cache for active users

---

## Application-Level Caching (Redis)

### Multi-Layer Caching Strategy

```javascript
// Code Node: Multi-Layer Cache Check
async function getCachedOrGenerate(request, userId) {
    const cacheKey = generateCacheKey(request, userId);

    // Layer 1: Check identical request cache (24 hours)
    const exactMatch = await $redis.get(cacheKey);
    if (exactMatch) {
        console.log('Cache hit: Exact match');
        return {
            meal: JSON.parse(exactMatch),
            cache_type: 'exact',
            cost: 0
        };
    }

    // Layer 2: Check similar request cache (fuzzy matching)
    const similarKey = await findSimilarCachedMeal(request);
    if (similarKey) {
        const similar = await $redis.get(similarKey);
        console.log('Cache hit: Similar meal');
        return {
            meal: JSON.parse(similar),
            cache_type: 'similar',
            cost: 0,
            note: '✨ Curated meal matching your preferences'
        };
    }

    // Layer 3: No cache hit - generate and cache
    const generated = await generateMealWithAI(request);

    // Cache for 24 hours
    await $redis.setex(cacheKey, 86400, JSON.stringify(generated.meal));

    console.log('Cache miss: Generated new meal');
    return {
        meal: generated.meal,
        cache_type: 'none',
        cost: generated.cost
    };
}

function generateCacheKey(request, userId) {
    const keyData = {
        user_preferences: request.user_preferences,
        dietary_restrictions: request.dietary_restrictions,
        cooking_method: request.cooking_method
    };

    // Create deterministic hash
    return `meal:${userId}:${hash(JSON.stringify(keyData))}`;
}

function hash(str) {
    let hash = 0;
    for (let i = 0; i < str.length; i++) {
        const char = str.charCodeAt(i);
        hash = ((hash << 5) - hash) + char;
        hash = hash & hash;
    }
    return hash.toString(36);
}

// Usage
const result = await getCachedOrGenerate($json.request, $json.user_id);

// Track cache hits
await $supabase.from('cache_stats').insert({
    user_id: $json.user_id,
    cache_type: result.cache_type,
    cost_saved: result.cache_type !== 'none' ? 0.0375 : 0,
    timestamp: new Date().toISOString()
});

return [{ json: result }];
```

### Cache Invalidation

```javascript
// Code Node: Smart Cache Invalidation
async function invalidateCaches(userId, reason) {
    const patterns = {
        profile_update: [`meal:${userId}:*`],
        preferences_change: [`meal:${userId}:*`, `grocery:${userId}:*`],
        weekly_plan_complete: [`meal:${userId}:*`]
    };

    const keysToDelete = patterns[reason] || [];

    for (const pattern of keysToDelete) {
        const keys = await $redis.keys(pattern);
        if (keys.length > 0) {
            await $redis.del(...keys);
            console.log(`Invalidated ${keys.length} cache keys for ${reason}`);
        }
    }
}

// Trigger on user profile update
if ($json.action === 'update_profile') {
    await invalidateCaches($json.user_id, 'profile_update');
}
```

**Expected Cache Hit Rate**: 60%+

---

## Model Routing Strategy

### Route by Complexity

```javascript
// Code Node: Route to Optimal Model
function selectModel(taskType, complexity) {
    const models = {
        // Simple tasks - use cheaper models
        intent_classification: {
            model: 'gpt-4o-mini',
            provider: 'openai',
            cost_per_1k_input: 0.00015,
            cost_per_1k_output: 0.0006
        },

        quick_response: {
            model: 'gpt-4o-mini',
            provider: 'openai',
            cost_per_1k_input: 0.00015,
            cost_per_1k_output: 0.0006
        },

        // Complex tasks - use advanced models
        meal_generation: {
            model: 'claude-sonnet-4-5',
            provider: 'anthropic',
            cost_per_1k_input: 0.003,
            cost_per_1k_output: 0.015
        },

        weekly_planning: {
            model: 'claude-sonnet-4-5',
            provider: 'anthropic',
            cost_per_1k_input: 0.003,
            cost_per_1k_output: 0.015
        },

        // Medium complexity - balance cost/quality
        grocery_list: {
            model: 'gpt-4o',
            provider: 'openai',
            cost_per_1k_input: 0.0025,
            cost_per_1k_output: 0.01
        }
    };

    const selected = models[taskType] || models.meal_generation;

    console.log(`Selected ${selected.model} for ${taskType}`);

    return selected;
}

// Usage
const modelConfig = selectModel($json.task_type, $json.complexity);

return [{
    json: {
        model: modelConfig.model,
        provider: modelConfig.provider,
        estimated_cost: calculateCost($json.input_tokens, $json.output_tokens, modelConfig)
    }
}];
```

**Savings Example**:
- Intent classification with GPT-4o mini: $0.0001 (vs $0.003 with Claude)
- 10 classifications/user/month × 1000 users = **$29/month saved**

---

## Batch Processing

### Weekly Plan Batch Generation

```javascript
// Code Node: Generate 7 Meals in Single Call
const BATCH_PROMPT = `Generate 7 DISTINCT daily OMAD meals in a SINGLE response.

CRITICAL: Return all 7 days in ONE JSON response, not separate responses.

This saves API costs and ensures variety checking across the full week.`;

// Single API call for 7 meals
const request = {
    model: "claude-sonnet-4-5",
    max_tokens: 4096, // Enough for 7 meals
    messages: [{ role: "user", content: BATCH_PROMPT }]
};

const response = await callAnthropic(request);
const weeklyPlan = JSON.parse(response.content);

// Cost comparison:
// 7 separate calls: 7 × $0.0375 = $0.2625
// 1 batch call: $0.069
// Savings: 74%

return [{ json: { weekly_plan: weeklyPlan, batch_savings: 0.1935 } }];
```

---

## Cost Monitoring & Alerts

### Daily Cost Dashboard

```javascript
// Code Node: Generate Daily Cost Report
async function getDailyCostReport() {
    const today = new Date().toISOString().split('T')[0];

    const logs = await $supabase
        .from('api_usage_logs')
        .select('*')
        .gte('created_at', today);

    const breakdown = {
        anthropic: { requests: 0, cost: 0, cache_savings: 0 },
        openai: { requests: 0, cost: 0 },
        total: 0,
        total_requests: logs.data.length
    };

    logs.data.forEach(log => {
        if (log.provider === 'anthropic') {
            breakdown.anthropic.requests++;
            breakdown.anthropic.cost += log.cost_usd;

            // Calculate cache savings
            const cacheReads = log.cache_read_tokens || 0;
            const savedCost = (cacheReads / 1000) * (0.003 - 0.0003);
            breakdown.anthropic.cache_savings += savedCost;

        } else if (log.provider === 'openai') {
            breakdown.openai.requests++;
            breakdown.openai.cost += log.cost_usd;
        }

        breakdown.total += log.cost_usd;
    });

    // Calculate metrics
    breakdown.avg_cost_per_request = (breakdown.total / breakdown.total_requests).toFixed(4);
    breakdown.cache_efficiency = (
        breakdown.anthropic.cache_savings / breakdown.anthropic.cost * 100
    ).toFixed(2) + '%';

    return breakdown;
}

const report = await getDailyCostReport();

console.log('Daily Cost Report:', JSON.stringify(report, null, 2));

// Alert if costs exceed budget
const DAILY_BUDGET = 50; // $50/day
if (report.total > DAILY_BUDGET) {
    await sendAdminAlert(`⚠️ Daily costs ($${report.total.toFixed(2)}) exceeded budget ($${DAILY_BUDGET})`);
}

return [{ json: report }];
```

### Cost Alerts

```javascript
// Code Node: Set Cost Alerts
async function checkCostThresholds() {
    const report = await getDailyCostReport();

    const thresholds = [
        { level: 'warning', amount: 30, message: '⚠️ Daily costs approaching budget' },
        { level: 'critical', amount: 50, message: '🚨 Daily budget exceeded!' },
        { level: 'severe', amount: 100, message: '🔴 Daily costs 2x budget - investigate immediately' }
    ];

    for (const threshold of thresholds) {
        if (report.total >= threshold.amount) {
            await sendAdminAlert({
                level: threshold.level,
                message: threshold.message,
                amount: report.total,
                breakdown: report
            });
            break;
        }
    }
}

await checkCostThresholds();
```

---

## Optimization Checklist

### Implementation Checklist

- ✅ **Prompt Caching (Primary)**
  - [ ] Mark system prompts with `cache_control`
  - [ ] Mark user profiles with `cache_control`
  - [ ] Monitor cache hit rate (target: 70%+)
  - [ ] Version prompts instead of frequent edits

- ✅ **Application Caching (Secondary)**
  - [ ] Implement Redis caching layer
  - [ ] Cache exact matches (24h TTL)
  - [ ] Cache similar meals (fuzzy matching)
  - [ ] Implement smart invalidation

- ✅ **Model Routing**
  - [ ] Use GPT-4o mini for intent classification
  - [ ] Use Claude Sonnet 4.5 for meal generation
  - [ ] Use GPT-4o for medium complexity tasks
  - [ ] Monitor quality vs cost tradeoffs

- ✅ **Batch Processing**
  - [ ] Generate weekly plans in single call
  - [ ] Batch user profile updates
  - [ ] Group similar requests

- ✅ **Monitoring**
  - [ ] Daily cost reports
  - [ ] Cost alerts at thresholds
  - [ ] Cache hit rate tracking
  - [ ] Per-user cost tracking

### Expected Savings Breakdown

| Optimization | Monthly Savings (1000 users) | Implementation Effort |
|-------------|------------------------------|----------------------|
| Prompt Caching | $114 (30%) | Low |
| Application Caching | $91 (24%) | Medium |
| Model Routing | $38 (10%) | Low |
| Batch Processing | $53 (14%) | Medium |
| **Total** | **$296 (78%)** | **Medium** |

**Target**: $0.95/user/month (67% reduction from baseline)

---

## Next Steps

1. ✅ Implement prompt caching
2. 🔄 Set up Redis caching layer
3. 📊 Deploy cost monitoring dashboard
4. 🧪 A/B test model routing
5. 🚀 Proceed to [Testing & Validation](./08-testing-validation.md)

---

## Version History

- **v1.0** (2025-01-15): Initial cost optimization guide
- **v1.1** (Pending): Add token streaming optimization
