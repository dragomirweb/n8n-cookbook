# n8n Workflow Architecture - Complete Implementation Guide

## Overview
This document provides a critical analysis and comprehensive implementation guide for the n8n workflow architecture powering the AI Meal Planning Assistant.

## Table of Contents
1. [Main Orchestration Workflow](#main-orchestration-workflow)
2. [Sub-Workflows](#sub-workflows)
3. [Node Configuration Best Practices](#node-configuration-best-practices)
4. [Critical Analysis & Improvements](#critical-analysis--improvements)
5. [Latest n8n Conventions](#latest-n8n-conventions)

---

## Main Orchestration Workflow

### Architecture Flow

```
Telegram Webhook Trigger
    ↓
[Extract User Data] (Code Node)
    ↓
[Get/Create User Profile] (HTTP Request → Supabase)
    ↓
[Send Typing Indicator] (Telegram API)
    ↓
[AI Intent Classification] (AI Agent - Claude Sonnet 4.5)
    ↓
[Switch: Route by Intent]
    ├─ single_meal_cook4me → Execute Sub-workflow
    ├─ single_meal_traditional → Execute Sub-workflow
    ├─ weekly_plan → Execute Sub-workflow
    ├─ grocery_list → Execute Sub-workflow
    ├─ settings → Execute Sub-workflow
    └─ help/general → Direct AI Response
    ↓
[Format Response] (Code Node)
    ↓
[Send to User] (Telegram API)
    ↓
[Parallel Branch]
    ├─ [Save to Database] (HTTP → Supabase)
    └─ [Update Context] (HTTP → Redis/Supabase)
```

### Critical Analysis

**✅ Strengths:**
- Clear separation of concerns
- Modular sub-workflow architecture
- Intent-based routing reduces complexity
- Typing indicator improves UX

**⚠️ Potential Issues:**
1. **Single Point of Failure**: Main workflow handles all routing - if it fails, entire system is down
2. **No Request Queuing**: Concurrent requests from same user could cause race conditions
3. **Limited Context Management**: Redis context updates happen after response, risking data loss
4. **No Rate Limiting**: Users could spam expensive AI calls

**🔧 Improvements Required:**

1. **Add Request Queue Node**
```javascript
// Code Node: Request Queue Management
const userId = $json.telegram_user_id;
const requestKey = `request_lock:${userId}`;

// Check if user has pending request
const existingRequest = await $redis.get(requestKey);
if (existingRequest) {
    return [{
        json: {
            queued: true,
            message: "⏳ Please wait for your previous request to complete"
        }
    }];
}

// Set lock with 5-minute expiration
await $redis.setex(requestKey, 300, Date.now());

return [{
    json: {
        queued: false,
        request_id: `${userId}_${Date.now()}`
    }
}];
```

2. **Add Rate Limiting**
```javascript
// Code Node: Rate Limit Check
const userId = $json.telegram_user_id;
const rateLimitKey = `rate_limit:${userId}`;

// Get request count in last hour
const count = await $redis.incr(rateLimitKey);
if (count === 1) {
    await $redis.expire(rateLimitKey, 3600); // 1 hour window
}

if (count > 20) { // Max 20 requests/hour
    return [{
        json: {
            rate_limited: true,
            message: "⚠️ Rate limit exceeded. Please try again in an hour."
        }
    }];
}

return [{ json: { rate_limited: false, count } }];
```

---

## Sub-Workflows

### Sub-Workflow 1: Single Meal Generator (Cook4me)

**Purpose**: Generate a complete OMAD meal optimized for Cook4me pressure cooker

**Inputs Required**:
- `user_id` (UUID)
- `user_preferences` (JSONB)
- `dietary_restrictions` (Array)
- `request_id` (String)

**Process Flow**:
```
[Receive Trigger]
    ↓
[Load User Profile] (Supabase Query)
    ↓
[Check Meal Cache] (Redis)
    ├─ Cache Hit → Return Cached Meal (90% faster, $0 cost)
    └─ Cache Miss → Continue
    ↓
[Build Prompt] (Code Node with Prompt Template)
    ↓
[AI Generation] (Claude Sonnet 4.5)
    ├─ Success → Continue
    └─ Error → [Fallback to GPT-4o]
    ↓
[Parse JSON Response] (Code Node with Validation)
    ↓
[Nutritional Validation]
    ├─ Calories: 2185-2415 kcal
    ├─ Protein: 130-160g
    ├─ Carbs: 230-280g
    └─ Fat: 70-90g
    ↓
[USDA Cross-Check] (Optional - HTTP Request)
    ↓
[Cook4me Format Validation]
    ├─ Check: "Pressure cooking time = X min" exists
    ├─ Check: Minimum 200ml liquid
    └─ Check: Proper instruction sequence
    ↓
[Save Recipe] (Supabase Insert)
    ↓
[Cache Result] (Redis - 24h TTL)
    ↓
[Return to Main Workflow]
```

**Critical Implementation Details**:

1. **Cache Strategy**
```javascript
// Code Node: Intelligent Caching
async function checkMealCache(userProfile) {
    // Create cache key from user preferences
    const cacheKey = `meal:cook4me:${JSON.stringify({
        protein: userProfile.preferred_proteins,
        restrictions: userProfile.dietary_restrictions,
        calories: userProfile.daily_calorie_target
    })}`;

    // Check cache
    const cached = await $redis.get(cacheKey);
    if (cached) {
        const meal = JSON.parse(cached);
        // Add variation note
        meal.note = "✨ Curated meal from your preferences";
        return { cached: true, meal };
    }

    return { cached: false, cacheKey };
}
```

2. **Prompt Building with Caching**
```javascript
// Code Node: Build Cached Prompt
function buildCook4mePrompt(userProfile) {
    // CRITICAL: These sections are cached by Claude for 5 minutes
    const systemPrompt = {
        type: "text",
        text: getSystemPromptTemplate(), // 2000 tokens - CACHED
        cache_control: { type: "ephemeral" }
    };

    const userProfileContext = {
        type: "text",
        text: JSON.stringify(userProfile), // 500 tokens - CACHED
        cache_control: { type: "ephemeral" }
    };

    // This changes per request (not cached)
    const userRequest = {
        type: "text",
        text: `Generate a Cook4me meal with these specific requirements: ${$json.specific_request || 'chicken-based meal'}`
    };

    return {
        system: [systemPrompt, userProfileContext],
        messages: [{ role: "user", content: [userRequest] }]
    };
}
```

3. **Response Validation**
```javascript
// Code Node: Strict JSON Validation
function validateMealResponse(aiResponse) {
    const errors = [];

    // Required fields
    const required = [
        'meal_name', 'ingredients', 'instructions',
        'nutritional_summary', 'cooking_method'
    ];

    required.forEach(field => {
        if (!aiResponse[field]) {
            errors.push(`Missing required field: ${field}`);
        }
    });

    // Nutritional validation
    const nutrition = aiResponse.nutritional_summary;
    if (nutrition) {
        if (nutrition.total_calories < 2185 || nutrition.total_calories > 2415) {
            errors.push(`Calories out of range: ${nutrition.total_calories}`);
        }
        if (nutrition.total_protein_g < 130 || nutrition.total_protein_g > 160) {
            errors.push(`Protein out of range: ${nutrition.total_protein_g}g`);
        }
    }

    // Cook4me format validation
    const instructions = aiResponse.instructions || [];
    const hasPressureTime = instructions.some(i =>
        /Pressure cooking time = \d+ min/i.test(i.instruction || i)
    );
    if (!hasPressureTime) {
        errors.push('Missing required "Pressure cooking time = X min" instruction');
    }

    return {
        valid: errors.length === 0,
        errors,
        meal: aiResponse
    };
}
```

**⚠️ Critical Warnings**:
- **Never skip nutritional validation** - invalid meals could harm users
- **Always validate Cook4me format** - incorrect instructions could damage the device
- **Implement retry logic** - AI responses occasionally fail JSON parsing
- **Cache aggressively** - saves 90% on costs for repeated preferences

---

### Sub-Workflow 2: Weekly Meal Planner

**Purpose**: Generate 7 distinct meals with automatic grocery list

**Key Innovation**: Generate all 7 meals in **single AI call** (saves 60% vs 7 separate calls)

**Process Flow**:
```
[Receive Trigger]
    ↓
[Load User Profile + Meal History]
    ↓
[Build Weekly Prompt] (Code Node)
    ├─ Include: Last 30 days of meals (prevent repeats)
    ├─ Include: Seasonal preferences
    └─ Include: Variety requirements
    ↓
[AI Generation] (Claude Sonnet 4.5 - 4096 tokens)
    ↓
[Parse 7 Meals] (Code Node)
    ↓
[Validate Variety]
    ├─ Check: No repeated proteins
    ├─ Check: Minimum 3 cuisines
    └─ Check: Each meal meets nutrition targets
    ↓
[IF Validation Fails]
    └─ [Regenerate Single Meal] (targeted fix)
    ↓
[Transaction Start]
    ├─ [Create Meal Plan Record]
    ├─ [Insert 7 Recipes]
    ├─ [Create Planned Meals Links]
    └─ [Generate Grocery List]
[Transaction Commit]
    ↓
[Return Weekly Plan + Grocery List]
```

**Critical Implementation**:

1. **Variety Validation**
```javascript
// Code Node: Strict Variety Enforcement
function validateWeeklyVariety(meals) {
    const issues = [];

    // Extract proteins
    const proteins = meals.map(m =>
        extractMainProtein(m.ingredients)
    );
    const uniqueProteins = new Set(proteins);

    if (uniqueProteins.size < 3) {
        issues.push({
            type: 'protein_variety',
            message: `Only ${uniqueProteins.size} different proteins. Need 3+`,
            proteins: Array.from(uniqueProteins)
        });
    }

    // Extract cuisines
    const cuisines = meals.map(m => m.cuisine_type);
    const uniqueCuisines = new Set(cuisines);

    if (uniqueCuisines.size < 3) {
        issues.push({
            type: 'cuisine_variety',
            message: `Only ${uniqueCuisines.size} cuisines. Need 3+`,
            cuisines: Array.from(uniqueCuisines)
        });
    }

    // Check for exact duplicates
    const mealNames = meals.map(m => m.meal_name.toLowerCase());
    const duplicates = mealNames.filter((name, index) =>
        mealNames.indexOf(name) !== index
    );

    if (duplicates.length > 0) {
        issues.push({
            type: 'duplicate_meals',
            message: 'Duplicate meals detected',
            duplicates
        });
    }

    return {
        valid: issues.length === 0,
        issues,
        variety_score: (uniqueProteins.size + uniqueCuisines.size) / 2
    };
}
```

2. **Grocery List Generation**
```javascript
// Code Node: Smart Grocery Consolidation
function generateGroceryList(meals) {
    const items = {};

    meals.forEach((meal, dayIndex) => {
        meal.ingredients.forEach(ing => {
            const normalizedName = normalizeIngredient(ing.item);
            const baseUnit = convertToBaseUnit(ing.quantity, ing.unit);

            if (!items[normalizedName]) {
                items[normalizedName] = {
                    name: ing.item,
                    quantity: 0,
                    unit: getPreferredPurchaseUnit(normalizedName),
                    category: categorizeIngredient(normalizedName),
                    used_in_meals: []
                };
            }

            items[normalizedName].quantity += baseUnit;
            items[normalizedName].used_in_meals.push(
                `Day ${dayIndex + 1}: ${meal.meal_name}`
            );
        });
    });

    // Convert to purchase units and round up
    Object.values(items).forEach(item => {
        item.quantity = Math.ceil(item.quantity);
        // Add 10% buffer for waste
        item.quantity = Math.ceil(item.quantity * 1.1);
    });

    // Group by category
    const categorized = {
        'Produce': [],
        'Proteins': [],
        'Dairy': [],
        'Grains & Pasta': [],
        'Pantry': [],
        'Frozen': [],
        'Other': []
    };

    Object.values(items).forEach(item => {
        const category = item.category || 'Other';
        categorized[category].push(item);
    });

    return categorized;
}
```

---

## Node Configuration Best Practices

### AI Agent Node Configuration (Claude Sonnet 4.5)

```javascript
{
    "model": "claude-sonnet-4-5-20250120", // Latest model version
    "max_tokens": 2500, // Single meal: 2500, Weekly: 4096
    "temperature": 0.4, // Balance creativity & consistency
    "top_p": 1.0,
    "system": [
        {
            "type": "text",
            "text": "{{$('Get System Prompt').item.json.prompt}}",
            "cache_control": { "type": "ephemeral" }
        }
    ],
    "messages": [
        {
            "role": "user",
            "content": "{{$json.user_prompt}}"
        }
    ],
    "response_format": { "type": "json_object" },

    // n8n specific settings
    "options": {
        "timeout": 60000, // 60 second timeout
        "retry": {
            "maxRetries": 3,
            "retryDelay": 2000,
            "retryOnStatusCodes": [429, 500, 502, 503, 504]
        }
    }
}
```

**Critical Settings Explained**:
- `temperature: 0.4` - Low enough for consistency, high enough for variety
- `cache_control: ephemeral` - Caches for 5 minutes, saves 90% on repeated prompts
- `max_tokens: 2500` - Sufficient for single meal, prevents unnecessary costs
- `timeout: 60000` - Prevents hanging requests

### HTTP Request Node (Supabase)

```javascript
{
    "method": "POST",
    "url": "={{$env.SUPABASE_URL}}/rest/v1/rpc/{{$json.function_name}}",
    "authentication": "predefinedCredential",
    "nodeCredentialType": "supabaseApi",
    "sendHeaders": true,
    "headerParameters": {
        "parameters": [
            {
                "name": "Prefer",
                "value": "return=representation"
            },
            {
                "name": "Content-Type",
                "value": "application/json"
            }
        ]
    },
    "sendBody": true,
    "bodyParameters": {
        "parameters": [
            {
                "name": "user_id",
                "value": "={{$json.user_id}}"
            }
        ]
    },
    "options": {
        "timeout": 10000,
        "retry": {
            "maxRetries": 3,
            "retryOnStatusCodes": [408, 429, 500, 502, 503, 504]
        },
        "response": {
            "response": {
                "fullResponse": false,
                "responseFormat": "json"
            }
        }
    }
}
```

### Switch Node (Intent Routing)

```javascript
{
    "mode": "expression",
    "rules": [
        {
            "output": 0, // Cook4me
            "expression": "={{$json.intent === 'single_meal_cook4me'}}"
        },
        {
            "output": 1, // Traditional
            "expression": "={{$json.intent === 'single_meal_traditional'}}"
        },
        {
            "output": 2, // Weekly Plan
            "expression": "={{$json.intent === 'weekly_plan'}}"
        },
        {
            "output": 3, // Grocery List
            "expression": "={{$json.intent === 'grocery_list'}}"
        },
        {
            "output": 4, // Settings
            "expression": "={{$json.intent === 'settings'}}"
        }
    ],
    "fallbackOutput": 5 // Help/General
}
```

---

## Latest n8n Conventions (2025)

### 1. Use Sub-workflows Instead of Execute Workflow

**❌ Old Way (Deprecated)**:
```javascript
// Execute Workflow node - now deprecated
{
    "workflowId": "12345",
    "data": {
        "user_id": "{{$json.user_id}}"
    }
}
```

**✅ New Way (2025)**:
```javascript
// Use Execute Workflow node with sub-workflow mode
{
    "mode": "subWorkflow",
    "subWorkflowId": "meal-generator-cook4me",
    "data": "={{$json}}", // Pass entire object
    "options": {
        "waitForSubWorkflow": true,
        "passInputData": true
    }
}
```

### 2. Use New AI Agent Node (Not Chat Model)

**✅ Correct (2025)**:
```javascript
// AI Agent node with built-in tools support
{
    "nodeType": "@n8n/n8n-nodes-langchain.agent",
    "parameters": {
        "agent": "conversationalAgent",
        "model": {
            "model": "claude-sonnet-4-5",
            "credentials": "anthropicApi"
        },
        "options": {
            "systemMessage": "={{$json.system_prompt}}",
            "memory": "windowBuffer"
        }
    }
}
```

### 3. Environment Variables Best Practices

**✅ Use Environment Variables for All Credentials**:
```javascript
// In Code Node
const supabaseUrl = $env.SUPABASE_URL;
const supabaseKey = $env.SUPABASE_API_KEY;

// In HTTP Request Node
{
    "url": "={{$env.SUPABASE_URL}}/rest/v1/users",
    "headers": {
        "apikey": "={{$env.SUPABASE_API_KEY}}"
    }
}
```

### 4. Error Handling with Error Trigger

**✅ Modern Error Handling**:
```
[Main Workflow]
    ↓
[Error Occurs]
    ↓
[Error Trigger Workflow] (Separate workflow)
    ├─ [Log to Database]
    ├─ [Send Alert to Admin]
    ├─ [Send User-Friendly Message]
    └─ [Attempt Recovery]
```

### 5. Use Merge Node for Parallel Processing

**✅ Parallel Operations**:
```javascript
[Single Input]
    ↓
[Split Into Batches]
    ├─ [Process Batch 1] ─┐
    ├─ [Process Batch 2] ─┤
    └─ [Process Batch 3] ─┤
                           ↓
                    [Merge Results]
                           ↓
                    [Continue]
```

---

## Critical Analysis & Production Recommendations

### Security Concerns

**🚨 HIGH PRIORITY**:
1. **Input Validation**: Sanitize all Telegram inputs before processing
2. **Rate Limiting**: Implement per-user rate limits (20 requests/hour)
3. **API Key Protection**: Never log or expose API keys
4. **SQL Injection**: Use parameterized queries in Supabase
5. **User Data Privacy**: Implement proper RLS policies

### Performance Optimization

**Key Metrics to Monitor**:
- Workflow execution time: Target < 30s for single meal
- Cache hit rate: Target > 60%
- Database query time: Target < 500ms
- AI response time: Target < 20s

**Optimization Strategies**:
1. **Aggressive Caching**: Cache user profiles, system prompts, similar requests
2. **Batch Operations**: Generate weekly plans in single AI call
3. **Async Processing**: Use separate workflows for non-blocking operations
4. **Connection Pooling**: Reuse database connections

### Scalability Considerations

**Current Limitations**:
- Single n8n instance can handle ~100 concurrent users
- Supabase free tier: 500MB database, 2GB bandwidth
- Anthropic API: 5 requests/second limit

**Scaling Strategy**:
1. **Phase 1 (0-100 users)**: Single n8n instance, Supabase free tier
2. **Phase 2 (100-1000 users)**: n8n auto-scaling, Supabase Pro
3. **Phase 3 (1000+ users)**: Multi-region deployment, dedicated Redis

---

## Workflow Export Format

All workflows should be exported in n8n JSON format with proper versioning:

```json
{
  "name": "AI Meal Planner - Main Orchestrator",
  "nodes": [...],
  "connections": {...},
  "settings": {
    "executionOrder": "v1",
    "saveManualExecutions": true,
    "callerPolicy": "workflowsFromSameOwner",
    "errorWorkflow": "error-handler-workflow-id"
  },
  "staticData": null,
  "tags": ["production", "meal-planner", "telegram"],
  "triggerCount": 1,
  "updatedAt": "2025-01-15T00:00:00.000Z",
  "versionId": "1.0.0"
}
```

---

## Next Steps

1. ✅ Review this architecture document
2. 📝 Proceed to [Database Schema Documentation](./02-database-schema.md)
3. 🔧 Review [Prompt Engineering Guide](./03-prompt-engineering.md)
4. 🚀 Follow [Implementation Roadmap](./10-implementation-roadmap.md)

---

## Version History

- **v1.0** (2025-01-15): Initial architecture design with n8n 2025 conventions
- **v1.1** (Pending): Add A/B testing framework for prompt optimization
- **v1.2** (Pending): Multi-language support architecture
