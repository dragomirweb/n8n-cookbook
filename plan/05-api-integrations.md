# API Integrations - Complete Implementation Guide

## Overview
Comprehensive guide for integrating all external APIs (Anthropic Claude, OpenAI, USDA, Supabase, Redis) with proper error handling, retry logic, and cost optimization.

## Table of Contents
1. [Anthropic Claude Integration](#anthropic-claude-integration)
2. [OpenAI Integration (Fallback)](#openai-integration-fallback)
3. [USDA FoodData Central](#usda-fooddata-central)
4. [Supabase Integration](#supabase-integration)
5. [Redis Caching](#redis-caching)
6. [Multi-Provider Fallback Strategy](#multi-provider-fallback-strategy)
7. [Rate Limiting & Retry Logic](#rate-limiting--retry-logic)

---

## Anthropic Claude Integration

### Setup

**Environment Variables:**
```bash
ANTHROPIC_API_KEY=sk-ant-api03-...
ANTHROPIC_MODEL=claude-sonnet-4-5-20250120
ANTHROPIC_API_VERSION=2023-06-01
```

### HTTP Request Node Configuration

```json
{
  "name": "Claude API Call",
  "type": "n8n-nodes-base.httpRequest",
  "parameters": {
    "method": "POST",
    "url": "https://api.anthropic.com/v1/messages",
    "authentication": "predefinedCredential",
    "nodeCredentialType": "anthropicApi",
    "sendHeaders": true,
    "headerParameters": {
      "parameters": [
        {
          "name": "anthropic-version",
          "value": "2023-06-01"
        },
        {
          "name": "Content-Type",
          "value": "application/json"
        }
      ]
    },
    "sendBody": true,
    "specifyBody": "json",
    "jsonBody": "={{$json.request}}",
    "options": {
      "timeout": 60000,
      "retry": {
        "maxRetries": 3,
        "retryOnStatusCodes": [429, 500, 502, 503, 504]
      }
    }
  }
}
```

### Code Node: Build Request with Prompt Caching

```javascript
// Code Node: Build Claude Request
const COOK4ME_SYSTEM_PROMPT = `[Your comprehensive system prompt here]`;
const userProfile = $json.user_profile;

const request = {
    model: $env.ANTHROPIC_MODEL || "claude-sonnet-4-5-20250120",
    max_tokens: 2500,
    temperature: 0.4,
    system: [
        {
            type: "text",
            text: COOK4ME_SYSTEM_PROMPT,
            cache_control: { type: "ephemeral" }  // Cache for 5 minutes
        },
        {
            type: "text",
            text: JSON.stringify(userProfile, null, 2),
            cache_control: { type: "ephemeral" }  // Cache for 5 minutes
        }
    ],
    messages: [
        {
            role: "user",
            content: $json.user_message
        }
    ]
};

return [{ json: { request } }];
```

### Parse Response & Track Usage

```javascript
// Code Node: Parse Claude Response
const response = $input.first().json;

// Extract content
const content = response.content[0].text;
let parsedMeal;

try {
    parsedMeal = JSON.parse(content);
} catch (error) {
    throw new Error(`Failed to parse AI response as JSON: ${error.message}`);
}

// Track usage and costs
const usage = {
    provider: 'anthropic',
    model: response.model,
    input_tokens: response.usage.input_tokens,
    output_tokens: response.usage.output_tokens,
    cache_read_tokens: response.usage.cache_read_input_tokens || 0,
    cache_creation_tokens: response.usage.cache_creation_input_tokens || 0,

    // Calculate costs (as of January 2025)
    input_cost_usd: (response.usage.input_tokens / 1000) * 0.003,
    output_cost_usd: (response.usage.output_tokens / 1000) * 0.015,
    cache_read_cost_usd: ((response.usage.cache_read_input_tokens || 0) / 1000) * 0.0003,
    cache_write_cost_usd: ((response.usage.cache_creation_input_tokens || 0) / 1000) * 0.00375,

    total_cost_usd: 0
};

usage.total_cost_usd =
    usage.input_cost_usd +
    usage.output_cost_usd +
    usage.cache_read_cost_usd +
    usage.cache_write_cost_usd;

// Cache hit rate
const cache_hit_rate = usage.cache_read_tokens > 0
    ? (usage.cache_read_tokens / (usage.input_tokens + usage.cache_read_tokens) * 100).toFixed(2)
    : 0;

return [{
    json: {
        meal: parsedMeal,
        usage,
        cache_hit_rate,
        timestamp: new Date().toISOString()
    }
}];
```

---

## OpenAI Integration (Fallback)

### Setup

**Environment Variables:**
```bash
OPENAI_API_KEY=sk-...
OPENAI_MODEL=gpt-4o
```

### HTTP Request Node

```json
{
  "name": "OpenAI API Call",
  "type": "n8n-nodes-base.httpRequest",
  "parameters": {
    "method": "POST",
    "url": "https://api.openai.com/v1/chat/completions",
    "authentication": "predefinedCredential",
    "nodeCredentialType": "openAiApi",
    "sendHeaders": true,
    "headerParameters": {
      "parameters": [
        {
          "name": "Content-Type",
          "value": "application/json"
        }
      ]
    },
    "sendBody": true,
    "specifyBody": "json",
    "jsonBody": "={{$json.request}}",
    "options": {
      "timeout": 60000,
      "response": {
        "response": {
          "responseFormat": "json"
        }
      }
    }
  }
}
```

### Code Node: Build OpenAI Request

```javascript
// Code Node: Build OpenAI Request
const request = {
    model: $env.OPENAI_MODEL || "gpt-4o",
    messages: [
        {
            role: "system",
            content: $json.system_prompt
        },
        {
            role: "user",
            content: $json.user_message
        }
    ],
    response_format: { type: "json_object" },
    temperature: 0.4,
    max_tokens: 2500
};

return [{ json: { request } }];
```

### Parse OpenAI Response

```javascript
// Code Node: Parse OpenAI Response
const response = $input.first().json;

const content = response.choices[0].message.content;
const parsedMeal = JSON.parse(content);

// Track usage
const usage = {
    provider: 'openai',
    model: response.model,
    input_tokens: response.usage.prompt_tokens,
    output_tokens: response.usage.completion_tokens,

    // GPT-4o pricing (as of January 2025)
    input_cost_usd: (response.usage.prompt_tokens / 1000) * 0.0025,
    output_cost_usd: (response.usage.completion_tokens / 1000) * 0.01,
    total_cost_usd: 0
};

usage.total_cost_usd = usage.input_cost_usd + usage.output_cost_usd;

return [{
    json: {
        meal: parsedMeal,
        usage,
        timestamp: new Date().toISOString()
    }
}];
```

---

## USDA FoodData Central

### Setup

**Get API Key:**
1. Visit: https://fdc.nal.usda.gov/api-key-signup.html
2. Sign up for free API key
3. Add to environment: `USDA_API_KEY=your-key`

### Search for Food

```javascript
// Code Node: Search USDA Database
async function searchUSDAFood(query) {
    const url = 'https://api.nal.usda.gov/fdc/v1/foods/search';

    const params = new URLSearchParams({
        query: query,
        pageSize: 5,
        api_key: $env.USDA_API_KEY
    });

    const response = await fetch(`${url}?${params}`);
    const data = await response.json();

    if (!data.foods || data.foods.length === 0) {
        return null;
    }

    return data.foods.map(food => ({
        fdc_id: food.fdcId,
        description: food.description,
        brand: food.brandName || food.brandOwner,
        nutrients: extractNutrients(food.foodNutrients)
    }));
}

function extractNutrients(foodNutrients) {
    const nutrients = {};

    foodNutrients.forEach(n => {
        if (n.nutrientName === 'Energy') {
            nutrients.calories = n.value;
        } else if (n.nutrientName === 'Protein') {
            nutrients.protein_g = n.value;
        } else if (n.nutrientName === 'Carbohydrate, by difference') {
            nutrients.carbs_g = n.value;
        } else if (n.nutrientName === 'Total lipid (fat)') {
            nutrients.fat_g = n.value;
        } else if (n.nutrientName === 'Fiber, total dietary') {
            nutrients.fiber_g = n.value;
        }
    });

    return nutrients;
}

// Usage
const chickenData = await searchUSDAFood('chicken breast raw');
return [{ json: { usda_data: chickenData } }];
```

### Validate Recipe Nutrition

```javascript
// Code Node: Validate Against USDA
async function validateRecipeNutrition(recipe) {
    const errors = [];

    for (const ingredient of recipe.ingredients) {
        const usdaData = await searchUSDAFood(ingredient.item);

        if (!usdaData || usdaData.length === 0) {
            errors.push({
                ingredient: ingredient.item,
                error: 'No USDA data found',
                severity: 'warning'
            });
            continue;
        }

        const usdaNutrition = usdaData[0].nutrients;

        // Compare calories (allow 10% variance)
        const calorieError = Math.abs(ingredient.calories - usdaNutrition.calories) / usdaNutrition.calories * 100;

        if (calorieError > 10) {
            errors.push({
                ingredient: ingredient.item,
                error: `Calorie variance ${calorieError.toFixed(1)}% (AI: ${ingredient.calories}, USDA: ${usdaNutrition.calories})`,
                severity: calorieError > 20 ? 'error' : 'warning'
            });
        }
    }

    return {
        valid: errors.filter(e => e.severity === 'error').length === 0,
        errors
    };
}

const validation = await validateRecipeNutrition($json.recipe);

return [{
    json: {
        recipe: $json.recipe,
        validation_result: validation
    }
}];
```

---

## Supabase Integration

### Setup

**Environment Variables:**
```bash
SUPABASE_URL=https://your-project.supabase.co
SUPABASE_API_KEY=your-anon-key
SUPABASE_SERVICE_ROLE_KEY=your-service-role-key  # For bypassing RLS
```

### HTTP Request Node Configuration

```json
{
  "name": "Supabase Query",
  "type": "n8n-nodes-base.httpRequest",
  "parameters": {
    "method": "POST",
    "url": "={{$env.SUPABASE_URL}}/rest/v1/rpc/{{$json.function_name}}",
    "authentication": "headerAuth",
    "sendHeaders": true,
    "headerParameters": {
      "parameters": [
        {
          "name": "apikey",
          "value": "={{$env.SUPABASE_API_KEY}}"
        },
        {
          "name": "Authorization",
          "value": "=Bearer {{$env.SUPABASE_API_KEY}}"
        },
        {
          "name": "Content-Type",
          "value": "application/json"
        },
        {
          "name": "Prefer",
          "value": "return=representation"
        }
      ]
    },
    "sendBody": true,
    "specifyBody": "json",
    "jsonBody": "={{$json.params}}"
  }
}
```

### Common Operations

#### 1. Get or Create User

```javascript
// Code Node: Call get_or_create_user Function
return [{
    json: {
        function_name: 'get_or_create_user',
        params: {
            p_telegram_user_id: $json.telegram_user_id,
            p_telegram_username: $json.telegram_username,
            p_telegram_first_name: $json.telegram_first_name,
            p_telegram_last_name: $json.telegram_last_name
        }
    }
}];

// Parse response
const result = $input.first().json;
const userData = result[0];  // Returns array with single object

return [{
    json: {
        user_id: userData.user_id,
        is_new_user: userData.is_new_user,
        profile: userData.profile
    }
}];
```

#### 2. Save Recipe

```javascript
// Code Node: Insert Recipe
return [{
    json: {
        function_name: 'insert_recipe',
        params: {
            p_recipe_data: {
                name: $json.meal.meal_name,
                description: $json.meal.description,
                servings: $json.meal.servings,
                prep_time_minutes: $json.meal.prep_time_minutes,
                cook_time_minutes: $json.meal.cook_time_minutes,
                ingredients: $json.meal.ingredients,
                instructions: $json.meal.instructions,
                cooking_method: $json.meal.cooking_method,
                cuisine_type: $json.meal.cuisine_type,
                calories_per_serving: $json.meal.nutritional_summary.total_calories,
                protein_per_serving_g: $json.meal.nutritional_summary.total_protein_g,
                carbs_per_serving_g: $json.meal.nutritional_summary.total_carbs_g,
                fat_per_serving_g: $json.meal.nutritional_summary.total_fat_g,
                fiber_per_serving_g: $json.meal.nutritional_summary.total_fiber_g,
                generated_by_ai: true,
                ai_model_used: $json.usage.model,
                generation_cost_usd: $json.usage.total_cost_usd
            }
        }
    }
}];
```

#### 3. Log API Usage

```javascript
// Code Node: Log API Call
return [{
    json: {
        function_name: 'log_api_usage',
        params: {
            p_user_id: $json.user_id,
            p_provider: $json.usage.provider,
            p_endpoint: 'chat/completions',
            p_method: 'POST',
            p_input_tokens: $json.usage.input_tokens,
            p_output_tokens: $json.usage.output_tokens,
            p_cache_read_tokens: $json.usage.cache_read_tokens || 0,
            p_cost_usd: $json.usage.total_cost_usd,
            p_response_time_ms: $json.response_time_ms,
            p_success: true
        }
    }
}];
```

---

## Redis Caching

### Setup

**Environment Variables:**
```bash
REDIS_URL=redis://localhost:6379
REDIS_PASSWORD=your-password  # If applicable
REDIS_DB=0
```

### n8n Redis Node (if available)

```json
{
  "name": "Redis Get",
  "type": "n8n-nodes-base.redis",
  "parameters": {
    "operation": "get",
    "key": "={{$json.cache_key}}"
  },
  "credentials": {
    "redis": {
      "id": "1",
      "name": "Redis Localhost"
    }
  }
}
```

### HTTP Request to Redis (Alternative)

```javascript
// Code Node: Redis Operations via ioredis
const Redis = require('ioredis');
const redis = new Redis($env.REDIS_URL);

// GET
async function getCache(key) {
    const data = await redis.get(key);
    return data ? JSON.parse(data) : null;
}

// SET with expiration
async function setCache(key, value, ttlSeconds = 3600) {
    await redis.setex(key, ttlSeconds, JSON.stringify(value));
}

// DELETE
async function deleteCache(key) {
    await redis.del(key);
}

// INCR (for rate limiting)
async function incrementCounter(key, ttlSeconds = 3600) {
    const count = await redis.incr(key);
    if (count === 1) {
        await redis.expire(key, ttlSeconds);
    }
    return count;
}

// Usage: Cache meal generation
const cacheKey = `meal:${JSON.stringify($json.user_preferences)}`;
const cached = await getCache(cacheKey);

if (cached) {
    return [{ json: { meal: cached, from_cache: true } }];
}

// If not cached, generate and cache
const meal = await generateMeal($json);
await setCache(cacheKey, meal, 86400); // 24 hours

return [{ json: { meal, from_cache: false } }];
```

### Cache Invalidation Strategy

```javascript
// Code Node: Smart Cache Invalidation
async function invalidateUserCache(userId) {
    // Invalidate user profile cache
    await redis.del(`user:${userId}:profile`);

    // Invalidate meal caches for this user
    const mealKeys = await redis.keys(`meal:${userId}:*`);
    if (mealKeys.length > 0) {
        await redis.del(...mealKeys);
    }

    // Invalidate grocery list cache
    await redis.del(`grocery:${userId}:active`);
}

// Trigger invalidation when user updates preferences
if ($json.action === 'update_preferences') {
    await invalidateUserCache($json.user_id);
}
```

---

## Multi-Provider Fallback Strategy

### Code Node: Multi-Provider Handler

```javascript
// Code Node: Call AI with Fallback
async function callAIWithFallback(prompt, userProfile) {
    const providers = [
        {
            name: 'anthropic',
            fn: callAnthropic,
            cost_per_request: 0.034,
            timeout: 60000
        },
        {
            name: 'openai',
            fn: callOpenAI,
            cost_per_request: 0.053,
            timeout: 45000
        }
    ];

    const errors = [];

    for (let i = 0; i < providers.length; i++) {
        const provider = providers[i];

        try {
            console.log(`Attempting ${provider.name}...`);

            const result = await provider.fn(prompt, userProfile);

            if (result && result.meal) {
                return {
                    success: true,
                    provider: provider.name,
                    provider_index: i,
                    cost: provider.cost_per_request,
                    meal: result.meal,
                    usage: result.usage
                };
            }

        } catch (error) {
            console.log(`${provider.name} failed: ${error.message}`);
            errors.push({
                provider: provider.name,
                error: error.message
            });

            // If not last provider, continue to next
            if (i < providers.length - 1) {
                await new Promise(resolve => setTimeout(resolve, 2000)); // 2s delay
                continue;
            }
        }
    }

    // All providers failed
    throw new Error(`All AI providers failed: ${JSON.stringify(errors)}`);
}

// Usage
const result = await callAIWithFallback($json.prompt, $json.user_profile);

return [{ json: result }];
```

---

## Rate Limiting & Retry Logic

### Exponential Backoff Retry

```javascript
// Code Node: Retry with Exponential Backoff
async function retryWithBackoff(fn, maxRetries = 5) {
    for (let attempt = 0; attempt < maxRetries; attempt++) {
        try {
            return await fn();

        } catch (error) {
            // Check if error is retryable
            const isRetryable =
                error.message?.includes('429') ||  // Rate limit
                error.message?.includes('503') ||  // Service unavailable
                error.message?.includes('timeout');

            if (!isRetryable || attempt === maxRetries - 1) {
                throw error;
            }

            // Calculate backoff: 2^attempt * 1000ms (max 30s)
            const backoffMs = Math.min(Math.pow(2, attempt) * 1000, 30000);

            console.log(`Retry attempt ${attempt + 1}/${maxRetries} after ${backoffMs}ms`);

            await new Promise(resolve => setTimeout(resolve, backoffMs));
        }
    }
}

// Usage
const result = await retryWithBackoff(async () => {
    return await callAnthropicAPI($json.request);
});

return [{ json: result }];
```

---

## Cost Monitoring Dashboard

```javascript
// Code Node: Daily Cost Report
async function getDailyCostReport() {
    const today = new Date().toISOString().split('T')[0];

    const usage = await $supabase
        .from('api_usage_logs')
        .select('*')
        .gte('created_at', today);

    const costs = {
        anthropic: {
            requests: 0,
            input_tokens: 0,
            output_tokens: 0,
            cache_read_tokens: 0,
            total_cost: 0
        },
        openai: {
            requests: 0,
            input_tokens: 0,
            output_tokens: 0,
            total_cost: 0
        },
        total: 0
    };

    usage.data.forEach(log => {
        if (log.provider === 'anthropic') {
            costs.anthropic.requests++;
            costs.anthropic.input_tokens += log.input_tokens;
            costs.anthropic.output_tokens += log.output_tokens;
            costs.anthropic.cache_read_tokens += log.cache_read_tokens || 0;
            costs.anthropic.total_cost += log.cost_usd;
        } else if (log.provider === 'openai') {
            costs.openai.requests++;
            costs.openai.input_tokens += log.input_tokens;
            costs.openai.output_tokens += log.output_tokens;
            costs.openai.total_cost += log.cost_usd;
        }
    });

    costs.total = costs.anthropic.total_cost + costs.openai.total_cost;

    // Cache hit rate
    const cache_hit_rate = costs.anthropic.input_tokens > 0
        ? (costs.anthropic.cache_read_tokens / costs.anthropic.input_tokens * 100).toFixed(2)
        : 0;

    return {
        date: today,
        costs,
        cache_hit_rate,
        avg_cost_per_request: (costs.total / (costs.anthropic.requests + costs.openai.requests)).toFixed(4)
    };
}

const report = await getDailyCostReport();

// Alert if costs exceed threshold
if (report.costs.total > 50) {
    await sendAdminAlert(`⚠️ Daily costs: $${report.costs.total.toFixed(2)}`);
}

return [{ json: report }];
```

---

## Next Steps

1. ✅ Set up all API credentials
2. 🧪 Test each integration individually
3. 🔄 Implement fallback logic
4. 📊 Set up cost monitoring
5. 🚀 Proceed to [Error Handling](./06-error-handling.md)

---

## Version History

- **v1.0** (2025-01-15): Initial API integrations guide
- **v1.1** (Pending): Add webhook support for real-time updates
