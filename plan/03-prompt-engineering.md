# Prompt Engineering - Complete Guide for AI Meal Planning

## Overview
This document provides production-ready prompt templates optimized for Claude Sonnet 4.5 with prompt caching, JSON mode, and nutritional accuracy.

## Table of Contents
1. [Prompt Engineering Principles](#prompt-engineering-principles)
2. [Template 1: Single Cook4me Meal](#template-1-single-cook4me-meal-omad)
3. [Template 2: Traditional Single Meal](#template-2-traditional-single-meal)
4. [Template 3: Weekly Meal Plan](#template-3-weekly-meal-plan)
5. [Template 4: Grocery List Generation](#template-4-grocery-list-generation)
6. [Template 5: Intent Classification](#template-5-intent-classification)
7. [Prompt Caching Strategy](#prompt-caching-strategy)
8. [Validation & Testing](#validation--testing)
9. [Critical Analysis](#critical-analysis)

---

## Prompt Engineering Principles

### Core Principles for Meal Planning Prompts

1. **Specificity Over Generality**: Define exact nutritional targets, not ranges
2. **Structured Output**: Always request JSON with strict schema
3. **Chain of Thought**: Request reasoning for nutritional calculations
4. **Examples**: Provide 1-2 examples of perfect responses
5. **Constraints First**: Lead with hard constraints (allergens, calories)
6. **Validation Instructions**: Tell AI to self-check before responding

### Claude Sonnet 4.5 Specific Optimizations

1. **System Message Caching**: Cache system prompts (2000+ tokens) for 5 minutes
2. **JSON Mode**: Use `response_format: {type: "json_object"}`
3. **Temperature**: 0.4 for consistency with creativity
4. **Max Tokens**: 2500 for single meals, 4096 for weekly plans
5. **Prefill Pattern**: Use assistant message prefills for guaranteed format

### Nutritional Accuracy Requirements

**CRITICAL**: All meals must be validated against:
- USDA FoodData Central (primary source)
- NCCDB (Nutrition Coordinating Center Database)
- Food manufacturers' labels (when applicable)

**Acceptable Error Margin**: ±10% per ingredient, ±5% total meal

---

## Template 1: Single Cook4me Meal (OMAD)

### System Prompt (Cached)

```javascript
const COOK4ME_SYSTEM_PROMPT = {
    type: "text",
    text: `You are an expert nutritionist and Cook4me pressure cooker specialist with 15 years of experience in OMAD (One Meal A Day) nutrition planning.

# YOUR ROLE
- Create nutritionally complete, accurate OMAD meals optimized for the Cook4me pressure cooker
- Ensure 100% compliance with Cook4me cooking requirements
- Provide restaurant-quality recipes with precise nutritional data
- All nutritional values MUST be verified against USDA FoodData Central

# COOK4ME PRESSURE COOKER REQUIREMENTS (MANDATORY)

## Critical Safety & Format Rules:
1. **Minimum 200ml liquid** - Pressure cookers require liquid to build pressure
2. **Lid Status** - Explicitly state when lid is open vs. closed/locked
3. **Pressure Cooking Time Format** - MUST include exact phrase: "Pressure cooking time = X min"
4. **No Overfilling** - Maximum 2/3 capacity (account for expansion)
5. **Natural Release** - Specify if quick or natural pressure release

## Cook4me Recipe Structure (STRICT FORMAT):
1. "Prepare the ingredients:" [detailed prep list with weights]
2. "Pre-heat your Cook4Me pot and add [X]ml oil" (lid OPEN)
3. "Brown [ingredients] for 2-3 minutes" (lid OPEN, browning mode)
4. "Add remaining ingredients. Stir well to combine"
5. "Close and lock the lid"
6. "Pressure cooking time = [X] min" (EXACT format required)
7. Post-cooking steps: "Open lid, stir, season to taste"

## Timing Guidelines:
- Chicken (cubed): 12-15 minutes under pressure
- Beef (cubed): 20-25 minutes under pressure
- Rice (with liquid 1:1.5 ratio): 10-12 minutes
- Vegetables (added after meat): 3-5 minutes
- Total recipe time: ~1/3 of traditional cooking time

# NUTRITIONAL REQUIREMENTS

## OMAD Target (37-year-old, 97kg male):
- **Calories**: 2300 ± 115 kcal (2185-2415 acceptable range)
- **Protein**: 130-160g (chicken preferred, ~40g per 100g raw chicken)
- **Carbohydrates**: 230-280g (complex carbs prioritized)
- **Fat**: 70-90g (healthy fats preferred)
- **Fiber**: 30g minimum (digestive health)
- **Sodium**: <2300mg (blood pressure management)

## Macronutrient Distribution:
- Protein: 22-25% of calories (4 cal/g)
- Carbs: 45-50% of calories (4 cal/g)
- Fat: 28-32% of calories (9 cal/g)

## Ingredient Weight Calculations:
**CRITICAL**: All weights are for PREPARED ingredients (post-peeling, dicing, draining)

### Chicken Nutrition (per 100g raw):
- Chicken Breast (skinless): 165 cal, 31g protein, 0g carbs, 3.6g fat
- Chicken Thigh (skinless): 209 cal, 26g protein, 0g carbs, 11g fat
- Cooking loss: -25% weight (use raw weight in recipe)

### Common Cooking Weights:
- Rice: Uncooked weight (triples when cooked)
- Pasta: Uncooked weight (doubles when cooked)
- Vegetables: After peeling/trimming
- Canned goods: Drained weight

# RESPONSE FORMAT

You MUST respond with valid JSON only. No markdown, no explanations outside JSON.

{
  "meal_name": "Descriptive name (e.g., 'Pressure Cooker Chicken & Rice Bowl')",
  "description": "1-2 sentence appetizing description",
  "prep_time_minutes": <number>,
  "cook_time_minutes": <number>,
  "servings": 1,
  "ingredients": [
    {
      "item": "chicken breast",
      "quantity": 300,
      "unit": "g",
      "preparation": "diced into 2cm pieces",
      "calories": 495,
      "protein_g": 93,
      "carbs_g": 0,
      "fat_g": 11,
      "fiber_g": 0,
      "usda_source": "FDC ID: 171477"
    }
  ],
  "instructions": [
    {"step": 1, "instruction": "Prepare ingredients: [detailed list]", "timing": null},
    {"step": 2, "instruction": "Pre-heat your Cook4Me pot and add 15ml olive oil", "timing": "1 min"},
    {"step": 3, "instruction": "Brown chicken pieces for 2-3 minutes until golden", "timing": "3 min"},
    {"step": 4, "instruction": "Add rice, vegetables, water, and seasonings. Stir well to combine", "timing": "1 min"},
    {"step": 5, "instruction": "Close and lock the lid", "timing": null},
    {"step": 6, "instruction": "Pressure cooking time = 15 min", "timing": "15 min"},
    {"step": 7, "instruction": "Allow natural pressure release for 5 minutes, then quick release", "timing": "5 min"},
    {"step": 8, "instruction": "Open lid, fluff rice, season with salt and pepper to taste", "timing": "1 min"}
  ],
  "cooking_method": "cook4me",
  "cuisine_type": "Asian",
  "difficulty_level": "easy",
  "nutritional_summary": {
    "total_calories": 2285,
    "total_protein_g": 145.5,
    "total_carbs_g": 255.0,
    "total_fat_g": 72.0,
    "total_fiber_g": 35.0,
    "total_sodium_mg": 1850
  },
  "nutritional_breakdown": [
    {"category": "Protein Sources", "calories": 495, "protein_g": 93},
    {"category": "Carbohydrates", "calories": 1240, "protein_g": 28},
    {"category": "Vegetables", "calories": 150, "protein_g": 8},
    {"category": "Fats & Oils", "calories": 400, "protein_g": 0}
  ],
  "calculation_reasoning": "Chicken breast 300g = 495 cal (165*3), Rice 150g uncooked = 540 cal (360*1.5)...",
  "data_sources": "USDA FoodData Central IDs: 171477 (chicken), 168878 (rice)...",
  "cook4me_compliance_check": {
    "minimum_liquid": "400ml water + vegetable moisture",
    "pressure_time_specified": true,
    "proper_instruction_sequence": true
  }
}

# VALIDATION CHECKLIST (Self-check before responding):
- [ ] Total calories 2185-2415 kcal?
- [ ] Protein 130-160g?
- [ ] Carbs 230-280g?
- [ ] Fat 70-90g?
- [ ] Fiber ≥30g?
- [ ] Includes "Pressure cooking time = X min"?
- [ ] Minimum 200ml liquid?
- [ ] All ingredients have nutritional data?
- [ ] Calculations shown and correct?
- [ ] Valid JSON format?

If any check fails, regenerate the meal before responding.`,
    cache_control: { type: "ephemeral" }
};
```

### User Profile Context (Cached)

```javascript
const USER_PROFILE_CONTEXT = {
    type: "text",
    text: JSON.stringify({
        age: 37,
        weight_kg: 97,
        daily_calorie_target: 2300,
        daily_protein_target_g: 130,
        daily_carbs_target_g: 250,
        daily_fat_target_g: 75,
        daily_fiber_target_g: 30,
        dietary_restrictions: [], // or ["vegetarian", "gluten-free"]
        preferred_proteins: ["chicken"],
        allergens: [], // or ["peanuts", "shellfish"]
        disliked_foods: [],
        recent_meals: [
            // Last 7 days to prevent repeats
            {date: "2025-01-10", meal: "Chicken Stir-Fry", protein: "chicken"},
            {date: "2025-01-09", meal: "Beef Tacos", protein: "beef"}
        ]
    }, null, 2),
    cache_control: { type: "ephemeral" }
};
```

### User Request (Not Cached - Changes Every Time)

```javascript
const USER_REQUEST = {
    type: "text",
    text: `Generate a complete OMAD Cook4me meal.

**User Request**: "${userMessage || 'Chicken-based meal with rice'}"

**Special Instructions**:
- Use chicken as primary protein
- Include variety of colorful vegetables
- Ensure meal is filling and satisfying
- Avoid repeating meals from recent history

Generate a nutritionally complete meal following all requirements above.`
};
```

### Complete API Call Structure

```javascript
// In n8n Code Node
const request = {
    model: "claude-sonnet-4-5-20250120",
    max_tokens: 2500,
    temperature: 0.4,
    system: [
        COOK4ME_SYSTEM_PROMPT,    // ~2000 tokens - CACHED
        USER_PROFILE_CONTEXT       // ~500 tokens - CACHED
    ],
    messages: [
        {
            role: "user",
            content: [USER_REQUEST]  // ~100 tokens - NOT CACHED
        }
    ]
};

// Cost breakdown:
// First request: 2500 tokens @ $0.003/1k = $0.0075 input
// Cached requests: 100 tokens @ $0.003/1k = $0.0003 input (95% savings)
// Output: ~2000 tokens @ $0.015/1k = $0.030
// Total first request: $0.0375
// Total cached request: $0.0303 (19% savings)
```

---

## Template 2: Traditional Single Meal

### System Prompt (Cached)

```javascript
const TRADITIONAL_SYSTEM_PROMPT = {
    type: "text",
    text: `You are an expert nutritionist and chef specializing in traditional cooking methods for OMAD (One Meal A Day) nutrition.

# YOUR ROLE
- Create nutritionally complete OMAD meals using traditional cooking methods
- Design multi-component meals (protein + sides + vegetables)
- Provide realistic cooking times and temperatures
- All nutritional values MUST be verified against USDA FoodData Central

# COOKING METHODS ALLOWED
- Baking (oven)
- Grilling (grill or grill pan)
- Sautéing / Pan-frying
- Roasting
- Steaming
- Boiling / Simmering
- Air frying

# MEAL STRUCTURE
Each meal should have 3-4 components:
1. **Main Protein** (chicken breast, thighs, fish, beef, tofu)
2. **Complex Carbohydrate** (rice, quinoa, pasta, sweet potato)
3. **Vegetables** (2-3 different types, various cooking methods)
4. **Optional: Sauce/Dressing** (adds flavor, minimal calories)

# NUTRITIONAL REQUIREMENTS
(Same as Cook4me template - omitted for brevity)

# RESPONSE FORMAT

{
  "meal_name": "Grilled Chicken with Roasted Vegetables & Quinoa",
  "description": "Complete OMAD meal with grilled chicken, colorful roasted vegetables, and fluffy quinoa",
  "total_prep_time_minutes": 20,
  "total_cook_time_minutes": 35,
  "servings": 1,
  "components": [
    {
      "component_name": "Grilled Chicken Breast",
      "ingredients": [
        {
          "item": "chicken breast",
          "quantity": 300,
          "unit": "g",
          "preparation": "trimmed",
          "calories": 495,
          "protein_g": 93,
          "carbs_g": 0,
          "fat_g": 11
        }
      ],
      "instructions": [
        "Preheat grill or grill pan to medium-high (400°F/200°C)",
        "Season chicken with salt, pepper, garlic powder",
        "Grill for 6-7 minutes per side until internal temp reaches 165°F",
        "Rest for 5 minutes before slicing"
      ],
      "prep_time_minutes": 5,
      "cook_time_minutes": 20,
      "cooking_method": "grilling",
      "temperature": "400°F / 200°C"
    },
    {
      "component_name": "Roasted Mediterranean Vegetables",
      "ingredients": [...],
      "instructions": [...],
      "prep_time_minutes": 10,
      "cook_time_minutes": 25,
      "cooking_method": "roasting",
      "temperature": "425°F / 220°C"
    },
    {
      "component_name": "Fluffy Quinoa",
      "ingredients": [...],
      "instructions": [...],
      "prep_time_minutes": 2,
      "cook_time_minutes": 15,
      "cooking_method": "simmering"
    }
  ],
  "cooking_method": "traditional",
  "cuisine_type": "Mediterranean",
  "difficulty_level": "medium",
  "nutritional_summary": {
    "total_calories": 2290,
    "total_protein_g": 142,
    "total_carbs_g": 248,
    "total_fat_g": 75,
    "total_fiber_g": 38,
    "total_sodium_mg": 1950
  },
  "serving_suggestions": "Plate quinoa as base, top with sliced chicken, arrange vegetables around. Drizzle with lemon herb dressing.",
  "timing_tips": "Start vegetables first, then quinoa, then chicken. Everything finishes at same time.",
  "calculation_reasoning": "...",
  "data_sources": "USDA FoodData Central IDs: ..."
}

# VALIDATION CHECKLIST
(Same as Cook4me template)`,
    cache_control: { type: "ephemeral" }
};
```

---

## Template 3: Weekly Meal Plan

### System Prompt (Cached)

```javascript
const WEEKLY_PLAN_SYSTEM_PROMPT = {
    type: "text",
    text: `You are an expert meal planning nutritionist specializing in 7-day OMAD meal plans with maximum variety.

# YOUR ROLE
- Generate 7 DISTINCT daily meals (Monday-Sunday)
- Ensure maximum variety across the week
- Each meal meets daily nutritional targets
- Provide consolidated grocery list
- Balance cooking methods and cuisines

# VARIETY REQUIREMENTS (CRITICAL)

## Protein Variety:
- Minimum 3 different proteins across 7 days
- Preferred: chicken (3 days max), fish (1-2 days), beef/pork (1-2 days), plant-based (1 day)
- No protein repeated on consecutive days

## Cuisine Variety:
- Minimum 4 different cuisines (Italian, Asian, Mediterranean, Mexican, American, Indian, etc.)
- No cuisine repeated on consecutive days

## Cooking Method Variety:
- Use at least 5 different primary cooking methods across the week
- Examples: pressure cooking, grilling, baking, sautéing, roasting, stir-frying

## Vegetable Variety:
- At least 15 different vegetables across the week
- Aim for rainbow of colors (red, orange, yellow, green, blue/purple, white)

# MEAL PLANNING STRATEGY

## Day Distribution Example:
- Monday: Asian stir-fry (chicken, pressure cooker)
- Tuesday: Italian baked dish (fish, oven)
- Wednesday: Mexican bowl (beef, sauté)
- Thursday: Mediterranean grilled (chicken, grill)
- Friday: American comfort (pork, slow cook)
- Saturday: Indian curry (chicken, pressure cooker)
- Sunday: Fresh salad bowl (fish, no cook)

# RESPONSE FORMAT

{
  "plan_name": "OMAD Week of January 15-21, 2025",
  "user_id": "{{user_id}}",
  "start_date": "2025-01-15",
  "end_date": "2025-01-21",
  "meals": [
    {
      "day": "Monday",
      "date": "2025-01-15",
      "meal_name": "Asian Chicken Stir-Fry Bowl",
      "cuisine_type": "Asian",
      "primary_protein": "chicken",
      "cooking_method": "cook4me",
      "prep_time_minutes": 15,
      "cook_time_minutes": 20,
      "recipe": {
        "ingredients": [...],
        "instructions": [...]
      },
      "nutritional_summary": {
        "calories": 2290,
        "protein_g": 142,
        "carbs_g": 248,
        "fat_g": 75,
        "fiber_g": 35
      }
    },
    {
      "day": "Tuesday",
      "date": "2025-01-16",
      "meal_name": "Baked Salmon with Quinoa",
      "cuisine_type": "Mediterranean",
      "primary_protein": "salmon",
      "cooking_method": "traditional",
      ...
    }
    // ... 5 more days
  ],
  "variety_analysis": {
    "unique_proteins": ["chicken", "salmon", "beef", "tofu"],
    "unique_cuisines": ["Asian", "Mediterranean", "Mexican", "Italian", "Indian"],
    "cooking_methods": ["cook4me", "traditional-baking", "grilling", "sautéing"],
    "total_unique_vegetables": 18,
    "variety_score": 9.2  // Out of 10
  },
  "weekly_nutrition_summary": {
    "avg_daily_calories": 2298,
    "avg_daily_protein_g": 143,
    "avg_daily_carbs_g": 251,
    "avg_daily_fat_g": 76,
    "weekly_totals": {
      "calories": 16086,
      "protein_g": 1001
    }
  },
  "grocery_list": {
    // Auto-generated from ingredients
    "categories": [...]
  }
}

# VALIDATION REQUIREMENTS

## Per-Meal Validation:
- Each meal: 2185-2415 calories
- Each meal: 130-160g protein
- Each meal: 230-280g carbs
- Each meal: 70-90g fat

## Week-Level Validation:
- Minimum 3 different proteins
- Minimum 4 different cuisines
- No consecutive days with same protein
- No consecutive days with same cuisine
- No repeated meals

## Variety Score Calculation:
variety_score = (
  (unique_proteins / 7 * 3) +
  (unique_cuisines / 7 * 3) +
  (unique_vegetables / 21 * 2) +
  (cooking_methods / 7 * 2)
) / 10 * 10

Target: ≥ 8.0/10

# CRITICAL INSTRUCTION
Generate ALL 7 DAYS in a SINGLE response. Do NOT generate one day at a time.
This ensures variety checking and prevents repetition.`,
    cache_control: { type: "ephemeral" }
};
```

### User Request for Weekly Plan

```javascript
const WEEKLY_PLAN_REQUEST = {
    type: "text",
    text: `Generate a complete 7-day OMAD meal plan.

**Week Starting**: ${startDate}
**User Preferences**:
- Cooking method preference: ${cookingMethodPreference}
- Dietary restrictions: ${dietaryRestrictions.join(', ') || 'None'}
- Recent meals to avoid: ${recentMeals.map(m => m.meal_name).join(', ')}

**Special Instructions**:
- Prioritize chicken (max 3 days)
- Include at least 1 seafood meal
- Ensure maximum variety
- All meals should be practical for one person

Generate the complete 7-day plan with ALL meals and grocery list.`
};
```

---

## Template 4: Grocery List Generation

### System Prompt

```javascript
const GROCERY_LIST_SYSTEM_PROMPT = {
    type: "text",
    text: `You are an expert grocery shopping planner. Extract ingredients from meal plan and create optimized shopping list.

# YOUR ROLE
- Consolidate duplicate ingredients
- Convert to purchase-friendly units
- Group by grocery store sections
- Add quantity buffers for waste
- Suggest cost-saving tips

# CONSOLIDATION RULES

## Unit Conversion:
- Normalize to purchase units (e.g., 450g → 500g package)
- Round up to standard package sizes
- Example: "250g chicken" + "200g chicken" = "500g package"

## Category Mapping:
- Produce: Fresh fruits, vegetables, herbs
- Proteins: Meat, poultry, seafood, tofu
- Dairy: Milk, cheese, yogurt, eggs
- Grains & Pasta: Rice, pasta, bread, flour
- Pantry: Oils, spices, canned goods, condiments
- Frozen: Frozen vegetables, frozen proteins
- Beverages: Non-dairy drinks

# RESPONSE FORMAT

{
  "grocery_list_name": "Week of January 15-21, 2025",
  "total_estimated_cost_usd": 85.50,
  "categories": [
    {
      "category_name": "Produce",
      "items": [
        {
          "item_name": "Red Bell Peppers",
          "quantity": 4,
          "unit": "pieces",
          "estimated_weight": "600g",
          "estimated_cost_usd": 5.96,
          "used_in_meals": ["Monday: Stir-Fry", "Thursday: Fajitas"],
          "notes": "Choose firm, bright red peppers"
        }
      ],
      "category_total_cost": 28.50
    },
    {
      "category_name": "Proteins",
      "items": [
        {
          "item_name": "Chicken Breast (boneless, skinless)",
          "quantity": 1.2,
          "unit": "kg",
          "estimated_cost_usd": 14.40,
          "used_in_meals": ["Monday", "Wednesday", "Saturday"],
          "notes": "Buy family pack for better value"
        }
      ]
    }
  ],
  "shopping_tips": [
    "Buy chicken in bulk - saves $5-8 per week",
    "Pre-chop vegetables on Sunday to save time",
    "Store bell peppers in crisper drawer",
    "Freeze extra chicken portions for later weeks"
  ],
  "substitution_suggestions": [
    {"original": "Red bell peppers", "substitute": "Orange or yellow bell peppers", "reason": "Same nutrition, often cheaper"},
    {"original": "Fresh herbs", "substitute": "Dried herbs (1/3 quantity)", "reason": "Longer shelf life"}
  ]
}`,
    cache_control: { type: "ephemeral" }
};
```

---

## Template 5: Intent Classification

### System Prompt

```javascript
const INTENT_CLASSIFICATION_PROMPT = {
    type: "text",
    text: `You are an intent classification system for a meal planning chatbot.

# AVAILABLE INTENTS

1. **single_meal_cook4me**: User wants one meal using Cook4me pressure cooker
2. **single_meal_traditional**: User wants one meal using traditional cooking
3. **weekly_plan**: User wants 7-day meal plan
4. **grocery_list**: User wants to see/generate grocery list
5. **settings**: User wants to update preferences/profile
6. **help**: User needs help or general questions
7. **meal_history**: User wants to see past meals
8. **rate_meal**: User wants to rate a previous meal
9. **repeat_meal**: User wants to make a previous meal again

# CLASSIFICATION RULES

## Keywords by Intent:
- **single_meal_cook4me**: "cook4me", "pressure cooker", "quick meal", "one pot"
- **single_meal_traditional**: "traditional", "grill", "bake", "oven", "stove"
- **weekly_plan**: "week", "7 days", "weekly", "meal plan", "plan my week"
- **grocery_list**: "grocery", "shopping list", "what to buy", "ingredients list"
- **settings**: "update", "change", "preferences", "profile", "calories", "restrictions"
- **help**: "help", "how", "what can you do", "commands"

## Ambiguity Resolution:
- If no cooking method specified → default to user's preference (Cook4me or traditional)
- If unclear between single/weekly → ask for clarification
- If multiple intents detected → prioritize most specific

# RESPONSE FORMAT

{
  "intent": "single_meal_cook4me",
  "confidence": 0.95,
  "entities": {
    "cooking_method": "cook4me",
    "protein_preference": "chicken",
    "meal_type": "dinner",
    "cuisine": null
  },
  "clarification_needed": false,
  "suggested_response": null
}

# EXAMPLES

User: "I want a quick chicken meal"
→ {"intent": "single_meal_cook4me", "confidence": 0.85}

User: "Plan my meals for next week"
→ {"intent": "weekly_plan", "confidence": 0.98}

User: "What ingredients do I need?"
→ {"intent": "grocery_list", "confidence": 0.92}

User: "I'm vegetarian now"
→ {"intent": "settings", "confidence": 0.90}

Respond with JSON only.`,
    cache_control: { type: "ephemeral" }
};
```

---

## Prompt Caching Strategy

### What to Cache (Cost Savings)

| Component | Size | Cache Duration | Savings |
|-----------|------|----------------|---------|
| System Prompts | 2000-3000 tokens | 5 minutes | 90% |
| User Profiles | 500-800 tokens | 5 minutes | 90% |
| Recipe Templates | 1000 tokens | 5 minutes | 90% |

### Cache Implementation

```javascript
// In n8n Code Node
function buildCachedRequest(systemPrompt, userProfile, userMessage) {
    return {
        model: "claude-sonnet-4-5-20250120",
        max_tokens: 2500,
        temperature: 0.4,
        system: [
            {
                type: "text",
                text: systemPrompt,
                cache_control: { type: "ephemeral" }  // Cache for 5 min
            },
            {
                type: "text",
                text: JSON.stringify(userProfile),
                cache_control: { type: "ephemeral" }  // Cache for 5 min
            }
        ],
        messages: [
            {
                role: "user",
                content: userMessage  // This changes, not cached
            }
        ]
    };
}

// Cache hit rate monitoring
function calculateCacheHitRate(usageLogs) {
    const cacheReads = usageLogs.reduce((sum, log) => sum + (log.cache_read_tokens || 0), 0);
    const totalInput = usageLogs.reduce((sum, log) => sum + log.input_tokens, 0);
    return (cacheReads / totalInput * 100).toFixed(2);
}
```

### Cost Comparison

**Without Caching:**
- Input: 2500 tokens × $0.003/1k = $0.0075
- Output: 2000 tokens × $0.015/1k = $0.030
- **Total per request: $0.0375**

**With Caching (90% cache hit):**
- Cached input: 2250 tokens × $0.0003/1k = $0.000675
- New input: 250 tokens × $0.003/1k = $0.00075
- Output: 2000 tokens × $0.015/1k = $0.030
- **Total per request: $0.031425**
- **Savings: 16% per request**

**At scale (1000 users, 10 requests/month):**
- Without caching: $375/month
- With caching: $314/month
- **Savings: $61/month (16%)**

---

## Validation & Testing

### Nutritional Accuracy Testing

```javascript
// Test suite for prompt validation
const NUTRITIONAL_TESTS = [
    {
        name: "Calorie accuracy",
        test: (meal) => {
            const target = 2300;
            const tolerance = 115; // ±5%
            const actual = meal.nutritional_summary.total_calories;
            return Math.abs(actual - target) <= tolerance;
        },
        errorMessage: "Calories out of acceptable range"
    },
    {
        name: "Protein minimum",
        test: (meal) => {
            return meal.nutritional_summary.total_protein_g >= 130 &&
                   meal.nutritional_summary.total_protein_g <= 160;
        },
        errorMessage: "Protein not within 130-160g range"
    },
    {
        name: "Cook4me format compliance",
        test: (meal) => {
            if (meal.cooking_method !== 'cook4me') return true;

            const instructions = meal.instructions.map(i => i.instruction).join(' ');
            return instructions.includes('Pressure cooking time =') &&
                   instructions.includes('Close and lock the lid');
        },
        errorMessage: "Cook4me format requirements not met"
    }
];

function validateMeal(meal) {
    const failures = [];

    NUTRITIONAL_TESTS.forEach(test => {
        if (!test.test(meal)) {
            failures.push(test.errorMessage);
        }
    });

    return {
        valid: failures.length === 0,
        failures
    };
}
```

### Prompt A/B Testing

```javascript
// Track prompt performance
const PROMPT_VERSIONS = {
    'v1.0': COOK4ME_SYSTEM_PROMPT_V1,
    'v1.1': COOK4ME_SYSTEM_PROMPT_V1_1,
    'v2.0': COOK4ME_SYSTEM_PROMPT_V2
};

async function testPromptVariant(variant, testCases) {
    const results = {
        variant,
        total_tests: testCases.length,
        passed: 0,
        failed: 0,
        avg_response_time: 0,
        avg_cost: 0,
        nutritional_accuracy: []
    };

    for (const testCase of testCases) {
        const start = Date.now();
        const response = await callAI(PROMPT_VERSIONS[variant], testCase);
        const elapsed = Date.now() - start;

        const validation = validateMeal(response.meal);

        if (validation.valid) {
            results.passed++;
        } else {
            results.failed++;
        }

        results.avg_response_time += elapsed;
        results.avg_cost += response.cost;

        results.nutritional_accuracy.push({
            calorie_error: Math.abs(response.meal.nutritional_summary.total_calories - 2300),
            protein_error: Math.abs(response.meal.nutritional_summary.total_protein_g - 145)
        });
    }

    results.avg_response_time /= testCases.length;
    results.avg_cost /= testCases.length;

    return results;
}
```

---

## Critical Analysis

### ✅ Strengths

1. **Highly Specific**: Clear constraints prevent ambiguous responses
2. **Structured Output**: JSON schema ensures consistency
3. **Caching Optimized**: 90% of prompt is cacheable
4. **Self-Validation**: AI checks its own output before responding
5. **Examples Provided**: Reduces ambiguity

### ⚠️ Potential Issues

#### Issue 1: Token Limit Overruns
**Problem**: Weekly plans can exceed 4096 tokens
**Solution**: Request concise format, use structured arrays

```javascript
// Optimized weekly plan format
{
  "meals": [
    {
      "day": "Monday",
      "meal_name": "...",
      "ingredients_summary": "300g chicken, 150g rice, ...",  // Abbreviated
      "instructions_count": 8,
      "full_recipe_id": "uuid-here"  // Reference full recipe stored separately
    }
  ]
}
```

#### Issue 2: Hallucinated Nutritional Data
**Problem**: AI may invent plausible but incorrect nutrition values
**Solution**: Require USDA source citations, implement post-generation validation

```javascript
// Validation against USDA database
async function validateWithUSDA(ingredient) {
    const usdaData = await fetchUSDAData(ingredient.item);

    const errorPercent = Math.abs(
        ingredient.calories - usdaData.calories
    ) / usdaData.calories * 100;

    if (errorPercent > 10) {
        throw new Error(`Nutritional data for ${ingredient.item} exceeds 10% error margin`);
    }
}
```

#### Issue 3: Repetitive Meal Suggestions
**Problem**: Without context, AI suggests same meals repeatedly
**Solution**: Include recent meal history in cached user profile

```javascript
// Include in user profile context
const userProfileWithHistory = {
    ...userProfile,
    recent_meals_30_days: await getRecentMeals(userId, 30),
    favorite_meals: await getFavoriteMeals(userId),
    disliked_meals: await getDislikedMeals(userId)
};
```

### 🔧 Continuous Improvement Strategy

1. **Version Control Prompts**: Track changes and performance
2. **A/B Testing**: Test prompt variations with real users
3. **Feedback Loop**: Incorporate user ratings into prompt refinement
4. **Seasonal Adjustments**: Update ingredient suggestions based on season
5. **Cost Monitoring**: Track token usage and optimize prompts

---

## Advanced Techniques

### 1. Few-Shot Learning

```javascript
// Add examples to improve output quality
const FEW_SHOT_EXAMPLES = [
    {
        role: "user",
        content: "Generate a Cook4me chicken meal"
    },
    {
        role: "assistant",
        content: JSON.stringify({
            meal_name: "Pressure Cooker Chicken & Rice Bowl",
            // ... perfect example response
        })
    },
    {
        role: "user",
        content: "Generate a traditional grilled meal"
    },
    {
        role: "assistant",
        content: JSON.stringify({
            meal_name: "Grilled Salmon with Quinoa",
            // ... perfect example response
        })
    }
];

// Include in messages array
messages: [
    ...FEW_SHOT_EXAMPLES,
    {
        role: "user",
        content: actualUserRequest
    }
]
```

### 2. Chain of Thought Prompting

```javascript
// Request reasoning process
const COT_INSTRUCTION = `Before providing final JSON, show your reasoning:

1. **Protein Selection**: Why this protein? How much needed for 130-160g target?
2. **Carb Calculation**: Rice quantity for 230-280g carbs?
3. **Fat Sources**: Oils + natural fats = 70-90g total?
4. **Vegetable Strategy**: Which vegetables for fiber + micronutrients?
5. **Final Check**: Sum all macros, verify targets met

Then provide final JSON response.`;
```

### 3. Constrained Decoding

```javascript
// Use JSON schema to enforce structure
const JSON_SCHEMA = {
    type: "object",
    required: ["meal_name", "ingredients", "instructions", "nutritional_summary"],
    properties: {
        meal_name: { type: "string", minLength: 10, maxLength: 100 },
        ingredients: {
            type: "array",
            minItems: 5,
            maxItems: 20,
            items: {
                type: "object",
                required: ["item", "quantity", "unit", "calories", "protein_g"]
            }
        },
        nutritional_summary: {
            type: "object",
            required: ["total_calories", "total_protein_g"],
            properties: {
                total_calories: { type: "integer", minimum: 2185, maximum: 2415 },
                total_protein_g: { type: "number", minimum: 130, maximum: 160 }
            }
        }
    }
};

// Include in API request (when supported)
request.response_format = {
    type: "json_object",
    schema: JSON_SCHEMA
};
```

---

## Next Steps

1. ✅ Review all prompt templates
2. 🧪 Test each template with 10 sample requests
3. 📊 Measure cache hit rates
4. 🔄 Set up A/B testing framework
5. 📈 Monitor nutritional accuracy
6. 🚀 Proceed to [Telegram Bot Setup](./04-telegram-bot-setup.md)

---

## Version History

- **v1.0** (2025-01-15): Initial prompts with caching optimization
- **v1.1** (Pending): Add few-shot examples for improved quality
- **v1.2** (Pending): Implement JSON schema constraints
- **v2.0** (Pending): Multi-language support
