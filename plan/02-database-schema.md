# Supabase Database Schema - Production-Ready Design

## Overview
Comprehensive PostgreSQL database schema for the AI Meal Planning Assistant, optimized for Supabase with Row Level Security (RLS), proper indexing, and scalability.

## Table of Contents
1. [Schema Design Principles](#schema-design-principles)
2. [Complete SQL Schema](#complete-sql-schema)
3. [Table Relationships](#table-relationships)
4. [Indexes & Performance](#indexes--performance)
5. [Row Level Security (RLS)](#row-level-security-rls)
6. [Database Functions](#database-functions)
7. [Critical Analysis](#critical-analysis)
8. [Migrations Strategy](#migrations-strategy)

---

## Schema Design Principles

### Core Principles
1. **Normalization**: 3NF (Third Normal Form) to prevent data redundancy
2. **Foreign Key Constraints**: Enforce referential integrity
3. **Soft Deletes**: Use `deleted_at` for important records (not hard deletes)
4. **Audit Trail**: Track `created_at`, `updated_at` for all tables
5. **JSONB for Flexibility**: Use for semi-structured data (dietary preferences, ingredients)
6. **UUIDs**: Use for primary keys (better for distributed systems)
7. **Cascading**: Carefully choose ON DELETE behaviors

### Performance Principles
1. **Index Strategy**: Index all foreign keys and frequently queried columns
2. **GIN Indexes**: For JSONB and full-text search
3. **Partial Indexes**: For common filtered queries
4. **Materialized Views**: For expensive aggregations

---

## Complete SQL Schema

### 1. Users & Authentication

```sql
-- =====================================================
-- USERS TABLE
-- =====================================================
-- Stores core user identity linked to Telegram
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    telegram_user_id BIGINT UNIQUE NOT NULL,
    telegram_username TEXT,
    telegram_first_name TEXT,
    telegram_last_name TEXT,
    is_active BOOLEAN DEFAULT true,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW(),
    last_seen_at TIMESTAMPTZ DEFAULT NOW()
);

-- Indexes
CREATE INDEX idx_users_telegram_id ON users(telegram_user_id);
CREATE INDEX idx_users_active ON users(is_active) WHERE is_active = true;
CREATE INDEX idx_users_last_seen ON users(last_seen_at DESC);

-- Comments
COMMENT ON TABLE users IS 'Core user identity table linked to Telegram accounts';
COMMENT ON COLUMN users.telegram_user_id IS 'Telegram unique user ID (not username)';
COMMENT ON COLUMN users.last_seen_at IS 'Last interaction timestamp for activity tracking';

-- =====================================================
-- USER PROFILES TABLE
-- =====================================================
-- Stores detailed nutritional preferences and targets
CREATE TABLE user_profiles (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,

    -- Physical Metrics
    age INTEGER CHECK (age BETWEEN 18 AND 120),
    weight_kg DECIMAL(5,2) CHECK (weight_kg > 0),
    height_cm INTEGER CHECK (height_cm BETWEEN 100 AND 250),
    gender TEXT CHECK (gender IN ('male', 'female', 'other', 'prefer_not_to_say')),
    activity_level TEXT CHECK (activity_level IN ('sedentary', 'light', 'moderate', 'active', 'very_active')) DEFAULT 'moderate',

    -- Nutritional Targets (Daily)
    daily_calorie_target INTEGER DEFAULT 2300 CHECK (daily_calorie_target BETWEEN 1200 AND 5000),
    daily_protein_target_g DECIMAL(6,2) DEFAULT 130.0 CHECK (daily_protein_target_g BETWEEN 50 AND 400),
    daily_carbs_target_g DECIMAL(6,2) DEFAULT 250.0 CHECK (daily_carbs_target_g BETWEEN 50 AND 600),
    daily_fat_target_g DECIMAL(6,2) DEFAULT 75.0 CHECK (daily_fat_target_g BETWEEN 20 AND 200),
    daily_fiber_target_g DECIMAL(5,2) DEFAULT 30.0 CHECK (daily_fiber_target_g BETWEEN 10 AND 100),

    -- Preferences (JSONB for flexibility)
    dietary_restrictions JSONB DEFAULT '[]'::jsonb,
    -- Example: ["vegetarian", "gluten-free", "low-sodium"]

    preferred_proteins JSONB DEFAULT '["chicken"]'::jsonb,
    -- Example: ["chicken", "fish", "tofu"]

    allergens JSONB DEFAULT '[]'::jsonb,
    -- Example: ["peanuts", "shellfish", "dairy"]

    disliked_foods JSONB DEFAULT '[]'::jsonb,
    -- Example: ["mushrooms", "cilantro"]

    cuisine_preferences JSONB DEFAULT '[]'::jsonb,
    -- Example: ["italian", "asian", "mediterranean"]

    -- Cooking Preferences
    default_cooking_method TEXT DEFAULT 'cook4me' CHECK (default_cooking_method IN ('cook4me', 'traditional', 'both')),
    max_prep_time_minutes INTEGER DEFAULT 30 CHECK (max_prep_time_minutes BETWEEN 5 AND 180),
    max_cook_time_minutes INTEGER DEFAULT 60 CHECK (max_cook_time_minutes BETWEEN 10 AND 300),

    -- Meal Timing
    meal_type_preference TEXT DEFAULT 'dinner' CHECK (meal_type_preference IN ('breakfast', 'lunch', 'dinner', 'any')),
    preferred_meal_time TIME DEFAULT '18:00:00',

    -- Settings
    language_code TEXT DEFAULT 'en' CHECK (length(language_code) = 2),
    timezone TEXT DEFAULT 'UTC',
    notifications_enabled BOOLEAN DEFAULT true,

    -- Audit
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW(),

    UNIQUE(user_id)
);

-- Indexes
CREATE INDEX idx_user_profiles_user_id ON user_profiles(user_id);
CREATE INDEX idx_user_profiles_cooking_method ON user_profiles(default_cooking_method);

-- GIN indexes for JSONB columns
CREATE INDEX idx_user_profiles_dietary ON user_profiles USING GIN (dietary_restrictions);
CREATE INDEX idx_user_profiles_allergens ON user_profiles USING GIN (allergens);
CREATE INDEX idx_user_profiles_proteins ON user_profiles USING GIN (preferred_proteins);

-- Comments
COMMENT ON TABLE user_profiles IS 'Detailed user nutritional preferences and targets';
COMMENT ON COLUMN user_profiles.dietary_restrictions IS 'JSONB array of dietary restrictions (vegetarian, vegan, keto, etc.)';
COMMENT ON COLUMN user_profiles.allergens IS 'JSONB array of allergens to strictly avoid';

-- Trigger for updated_at
CREATE OR REPLACE FUNCTION update_updated_at_column()
RETURNS TRIGGER AS $$
BEGIN
    NEW.updated_at = NOW();
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER update_user_profiles_updated_at
    BEFORE UPDATE ON user_profiles
    FOR EACH ROW
    EXECUTE FUNCTION update_updated_at_column();
```

### 2. Recipes & Ingredients

```sql
-- =====================================================
-- RECIPES TABLE
-- =====================================================
CREATE TABLE recipes (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),

    -- Basic Info
    name TEXT NOT NULL,
    description TEXT,
    servings INTEGER DEFAULT 1 CHECK (servings BETWEEN 1 AND 12),

    -- Timing
    prep_time_minutes INTEGER CHECK (prep_time_minutes >= 0),
    cook_time_minutes INTEGER CHECK (cook_time_minutes >= 0),
    total_time_minutes INTEGER GENERATED ALWAYS AS (prep_time_minutes + cook_time_minutes) STORED,

    -- Recipe Content (JSONB for flexibility)
    ingredients JSONB NOT NULL,
    -- Structure: [{"item": "chicken breast", "quantity": 300, "unit": "g", "preparation": "diced", "calories": 495, "protein_g": 93, "carbs_g": 0, "fat_g": 11}]

    instructions JSONB NOT NULL,
    -- Structure: [{"step": 1, "instruction": "Preheat...", "timing": "2 min", "image_url": null}]

    -- Cooking Method
    cooking_method TEXT NOT NULL CHECK (cooking_method IN ('cook4me', 'traditional', 'mixed')),
    cooking_equipment JSONB DEFAULT '[]'::jsonb,
    -- Example: ["cook4me", "knife", "cutting board"]

    -- Nutrition (per serving)
    calories_per_serving INTEGER NOT NULL CHECK (calories_per_serving > 0),
    protein_per_serving_g DECIMAL(6,2),
    carbs_per_serving_g DECIMAL(6,2),
    fat_per_serving_g DECIMAL(6,2),
    fiber_per_serving_g DECIMAL(5,2),
    sodium_per_serving_mg DECIMAL(7,2),
    sugar_per_serving_g DECIMAL(6,2),

    -- Total Nutrition (for OMAD - servings=1)
    total_calories INTEGER GENERATED ALWAYS AS (calories_per_serving * servings) STORED,
    total_protein_g DECIMAL(6,2) GENERATED ALWAYS AS (protein_per_serving_g * servings) STORED,
    total_carbs_g DECIMAL(6,2) GENERATED ALWAYS AS (carbs_per_serving_g * servings) STORED,
    total_fat_g DECIMAL(6,2) GENERATED ALWAYS AS (fat_per_serving_g * servings) STORED,

    -- Categorization
    cuisine_type TEXT,
    meal_type TEXT CHECK (meal_type IN ('breakfast', 'lunch', 'dinner', 'snack', 'any')),
    difficulty_level TEXT CHECK (difficulty_level IN ('easy', 'medium', 'hard')) DEFAULT 'medium',
    tags JSONB DEFAULT '[]'::jsonb,
    -- Example: ["high-protein", "quick", "one-pot"]

    -- AI Generation Metadata
    generated_by_ai BOOLEAN DEFAULT true,
    ai_model_used TEXT,
    ai_prompt_version TEXT,
    generation_cost_usd DECIMAL(10,6),

    -- Validation
    nutritional_accuracy_verified BOOLEAN DEFAULT false,
    verified_at TIMESTAMPTZ,
    verified_by UUID REFERENCES users(id),

    -- User Engagement
    times_generated INTEGER DEFAULT 1,
    avg_user_rating DECIMAL(3,2) CHECK (avg_user_rating BETWEEN 1 AND 5),
    total_ratings INTEGER DEFAULT 0,

    -- Audit
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW()
);

-- Indexes
CREATE INDEX idx_recipes_cooking_method ON recipes(cooking_method);
CREATE INDEX idx_recipes_cuisine_type ON recipes(cuisine_type);
CREATE INDEX idx_recipes_calories ON recipes(total_calories);
CREATE INDEX idx_recipes_protein ON recipes(total_protein_g);
CREATE INDEX idx_recipes_created ON recipes(created_at DESC);
CREATE INDEX idx_recipes_rating ON recipes(avg_user_rating DESC NULLS LAST);

-- GIN indexes for JSONB
CREATE INDEX idx_recipes_ingredients ON recipes USING GIN (ingredients);
CREATE INDEX idx_recipes_instructions ON recipes USING GIN (instructions);
CREATE INDEX idx_recipes_tags ON recipes USING GIN (tags);

-- Full-text search on recipe names and descriptions
CREATE INDEX idx_recipes_search ON recipes USING GIN (
    to_tsvector('english', COALESCE(name, '') || ' ' || COALESCE(description, ''))
);

-- Comments
COMMENT ON TABLE recipes IS 'Complete recipe information including AI-generated meals';
COMMENT ON COLUMN recipes.ingredients IS 'JSONB array of ingredients with detailed nutritional info';
COMMENT ON COLUMN recipes.instructions IS 'JSONB array of step-by-step instructions';
COMMENT ON COLUMN recipes.times_generated IS 'Tracks popularity of AI-generated recipes';

-- Trigger for updated_at
CREATE TRIGGER update_recipes_updated_at
    BEFORE UPDATE ON recipes
    FOR EACH ROW
    EXECUTE FUNCTION update_updated_at_column();
```

### 3. Meal Plans

```sql
-- =====================================================
-- MEAL PLANS TABLE
-- =====================================================
CREATE TABLE meal_plans (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,

    -- Plan Details
    name TEXT NOT NULL,
    description TEXT,
    start_date DATE NOT NULL,
    end_date DATE NOT NULL,

    -- Validation
    CONSTRAINT valid_date_range CHECK (end_date >= start_date),

    -- Status
    is_active BOOLEAN DEFAULT true,
    completed BOOLEAN DEFAULT false,
    completion_percentage DECIMAL(5,2) DEFAULT 0.00 CHECK (completion_percentage BETWEEN 0 AND 100),

    -- Metadata
    total_planned_meals INTEGER DEFAULT 0,
    total_estimated_cost_usd DECIMAL(10,2),

    -- Audit
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW()
);

-- Indexes
CREATE INDEX idx_meal_plans_user_id ON meal_plans(user_id);
CREATE INDEX idx_meal_plans_active ON meal_plans(user_id, is_active) WHERE is_active = true;
CREATE INDEX idx_meal_plans_date_range ON meal_plans(start_date, end_date);

-- Comments
COMMENT ON TABLE meal_plans IS 'Weekly or monthly meal planning containers';
COMMENT ON COLUMN meal_plans.completion_percentage IS 'Percentage of planned meals actually consumed';

-- =====================================================
-- PLANNED MEALS TABLE
-- =====================================================
CREATE TABLE planned_meals (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    meal_plan_id UUID NOT NULL REFERENCES meal_plans(id) ON DELETE CASCADE,
    recipe_id UUID NOT NULL REFERENCES recipes(id) ON DELETE RESTRICT,

    -- Scheduling
    scheduled_date DATE NOT NULL,
    scheduled_time TIME DEFAULT '18:00:00',
    meal_type TEXT DEFAULT 'dinner' CHECK (meal_type IN ('breakfast', 'lunch', 'dinner', 'snack')),

    -- Serving Info
    servings DECIMAL(4,2) DEFAULT 1 CHECK (servings > 0),

    -- Status
    completed BOOLEAN DEFAULT false,
    completed_at TIMESTAMPTZ,
    skipped BOOLEAN DEFAULT false,
    skip_reason TEXT,

    -- Audit
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW(),

    -- Prevent duplicate meals on same day
    UNIQUE(meal_plan_id, scheduled_date)
);

-- Indexes
CREATE INDEX idx_planned_meals_plan_id ON planned_meals(meal_plan_id);
CREATE INDEX idx_planned_meals_recipe_id ON planned_meals(recipe_id);
CREATE INDEX idx_planned_meals_date ON planned_meals(scheduled_date);
CREATE INDEX idx_planned_meals_pending ON planned_meals(meal_plan_id) WHERE completed = false AND skipped = false;

-- Comments
COMMENT ON TABLE planned_meals IS 'Individual meals within a meal plan';
COMMENT ON COLUMN planned_meals.skip_reason IS 'User-provided reason for skipping meal';

-- Trigger to update meal plan stats
CREATE OR REPLACE FUNCTION update_meal_plan_stats()
RETURNS TRIGGER AS $$
BEGIN
    UPDATE meal_plans
    SET
        total_planned_meals = (
            SELECT COUNT(*)
            FROM planned_meals
            WHERE meal_plan_id = COALESCE(NEW.meal_plan_id, OLD.meal_plan_id)
        ),
        completion_percentage = (
            SELECT CASE
                WHEN COUNT(*) = 0 THEN 0
                ELSE (COUNT(*) FILTER (WHERE completed = true) * 100.0 / COUNT(*))
            END
            FROM planned_meals
            WHERE meal_plan_id = COALESCE(NEW.meal_plan_id, OLD.meal_plan_id)
        )
    WHERE id = COALESCE(NEW.meal_plan_id, OLD.meal_plan_id);
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER update_meal_plan_stats_trigger
    AFTER INSERT OR UPDATE OR DELETE ON planned_meals
    FOR EACH ROW
    EXECUTE FUNCTION update_meal_plan_stats();
```

### 4. Meal History & Tracking

```sql
-- =====================================================
-- MEAL HISTORY TABLE
-- =====================================================
CREATE TABLE meal_history (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    recipe_id UUID REFERENCES recipes(id) ON DELETE SET NULL,
    planned_meal_id UUID REFERENCES planned_meals(id) ON DELETE SET NULL,

    -- Consumption Details
    consumed_at TIMESTAMPTZ DEFAULT NOW(),
    servings DECIMAL(4,2) DEFAULT 1 CHECK (servings > 0),

    -- Actual Nutrition (may differ from planned if servings differ)
    actual_calories INTEGER,
    actual_protein_g DECIMAL(6,2),
    actual_carbs_g DECIMAL(6,2),
    actual_fat_g DECIMAL(6,2),

    -- User Feedback
    rating INTEGER CHECK (rating BETWEEN 1 AND 5),
    notes TEXT,
    would_eat_again BOOLEAN,
    taste_rating INTEGER CHECK (taste_rating BETWEEN 1 AND 5),
    ease_rating INTEGER CHECK (ease_rating BETWEEN 1 AND 5),

    -- Photos
    photo_url TEXT,

    -- Audit
    created_at TIMESTAMPTZ DEFAULT NOW()
);

-- Indexes
CREATE INDEX idx_meal_history_user_consumed ON meal_history(user_id, consumed_at DESC);
CREATE INDEX idx_meal_history_recipe ON meal_history(recipe_id);
CREATE INDEX idx_meal_history_rating ON meal_history(rating DESC) WHERE rating IS NOT NULL;
CREATE INDEX idx_meal_history_date ON meal_history(consumed_at::date);

-- Partitioning by date (for scalability)
-- Future: Consider partitioning by month for large datasets

-- Comments
COMMENT ON TABLE meal_history IS 'Complete history of all meals consumed by users';
COMMENT ON COLUMN meal_history.would_eat_again IS 'Boolean flag for quick preference tracking';

-- Trigger to update recipe ratings
CREATE OR REPLACE FUNCTION update_recipe_ratings()
RETURNS TRIGGER AS $$
BEGIN
    IF NEW.rating IS NOT NULL THEN
        UPDATE recipes
        SET
            total_ratings = total_ratings + 1,
            avg_user_rating = (
                SELECT AVG(rating)
                FROM meal_history
                WHERE recipe_id = NEW.recipe_id AND rating IS NOT NULL
            )
        WHERE id = NEW.recipe_id;
    END IF;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER update_recipe_ratings_trigger
    AFTER INSERT ON meal_history
    FOR EACH ROW
    EXECUTE FUNCTION update_recipe_ratings();
```

### 5. Grocery Lists

```sql
-- =====================================================
-- GROCERY LISTS TABLE
-- =====================================================
CREATE TABLE grocery_lists (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    meal_plan_id UUID REFERENCES meal_plans(id) ON DELETE SET NULL,

    -- List Details
    name TEXT NOT NULL,
    description TEXT,

    -- Status
    status TEXT DEFAULT 'active' CHECK (status IN ('active', 'shopping', 'completed', 'archived')),
    shopping_started_at TIMESTAMPTZ,
    shopping_completed_at TIMESTAMPTZ,

    -- Costs
    estimated_total_cost_usd DECIMAL(10,2),
    actual_total_cost_usd DECIMAL(10,2),

    -- Metadata
    total_items INTEGER DEFAULT 0,
    purchased_items INTEGER DEFAULT 0,

    -- Audit
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW()
);

-- Indexes
CREATE INDEX idx_grocery_lists_user_id ON grocery_lists(user_id);
CREATE INDEX idx_grocery_lists_status ON grocery_lists(user_id, status);
CREATE INDEX idx_grocery_lists_meal_plan ON grocery_lists(meal_plan_id);

-- Comments
COMMENT ON TABLE grocery_lists IS 'Auto-generated and manual grocery shopping lists';

-- =====================================================
-- GROCERY LIST ITEMS TABLE
-- =====================================================
CREATE TABLE grocery_list_items (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    grocery_list_id UUID NOT NULL REFERENCES grocery_lists(id) ON DELETE CASCADE,

    -- Item Details
    item_name TEXT NOT NULL,
    quantity DECIMAL(10,2) NOT NULL CHECK (quantity > 0),
    unit TEXT NOT NULL,

    -- Organization
    category TEXT CHECK (category IN (
        'Produce', 'Proteins', 'Dairy', 'Grains & Pasta',
        'Pantry', 'Frozen', 'Beverages', 'Snacks', 'Other'
    )),
    aisle_number INTEGER,

    -- Purchase Status
    is_purchased BOOLEAN DEFAULT false,
    purchased_at TIMESTAMPTZ,

    -- Costs
    estimated_cost_usd DECIMAL(8,2),
    actual_cost_usd DECIMAL(8,2),

    -- Metadata
    used_in_meals JSONB DEFAULT '[]'::jsonb,
    -- Example: ["Monday: Chicken Stir-Fry", "Wednesday: Grilled Chicken"]

    notes TEXT,
    brand_preference TEXT,

    -- Audit
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW()
);

-- Indexes
CREATE INDEX idx_grocery_items_list_id ON grocery_list_items(grocery_list_id);
CREATE INDEX idx_grocery_items_category ON grocery_list_items(category);
CREATE INDEX idx_grocery_items_purchased ON grocery_list_items(is_purchased);

-- GIN index for meal references
CREATE INDEX idx_grocery_items_meals ON grocery_list_items USING GIN (used_in_meals);

-- Comments
COMMENT ON TABLE grocery_list_items IS 'Individual items in grocery lists with purchase tracking';

-- Trigger to update grocery list stats
CREATE OR REPLACE FUNCTION update_grocery_list_stats()
RETURNS TRIGGER AS $$
BEGIN
    UPDATE grocery_lists
    SET
        total_items = (
            SELECT COUNT(*)
            FROM grocery_list_items
            WHERE grocery_list_id = COALESCE(NEW.grocery_list_id, OLD.grocery_list_id)
        ),
        purchased_items = (
            SELECT COUNT(*)
            FROM grocery_list_items
            WHERE grocery_list_id = COALESCE(NEW.grocery_list_id, OLD.grocery_list_id)
              AND is_purchased = true
        ),
        estimated_total_cost_usd = (
            SELECT SUM(estimated_cost_usd)
            FROM grocery_list_items
            WHERE grocery_list_id = COALESCE(NEW.grocery_list_id, OLD.grocery_list_id)
        ),
        actual_total_cost_usd = (
            SELECT SUM(actual_cost_usd)
            FROM grocery_list_items
            WHERE grocery_list_id = COALESCE(NEW.grocery_list_id, OLD.grocery_list_id)
              AND actual_cost_usd IS NOT NULL
        )
    WHERE id = COALESCE(NEW.grocery_list_id, OLD.grocery_list_id);
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER update_grocery_list_stats_trigger
    AFTER INSERT OR UPDATE OR DELETE ON grocery_list_items
    FOR EACH ROW
    EXECUTE FUNCTION update_grocery_list_stats();
```

### 6. Conversation Context

```sql
-- =====================================================
-- CONVERSATION SESSIONS TABLE
-- =====================================================
CREATE TABLE conversation_sessions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,

    -- Session Timing
    started_at TIMESTAMPTZ DEFAULT NOW(),
    last_activity_at TIMESTAMPTZ DEFAULT NOW(),
    ended_at TIMESTAMPTZ,

    -- Status
    is_active BOOLEAN DEFAULT true,

    -- Context Data
    context JSONB DEFAULT '{}'::jsonb,
    -- Structure: {"last_intent": "single_meal", "preferences": {...}, "conversation_history": []}

    message_count INTEGER DEFAULT 0,
    last_message_text TEXT,

    -- Metadata
    platform TEXT DEFAULT 'telegram',
    session_metadata JSONB DEFAULT '{}'::jsonb,

    -- Audit
    created_at TIMESTAMPTZ DEFAULT NOW()
);

-- Indexes
CREATE INDEX idx_conv_sessions_user_id ON conversation_sessions(user_id);
CREATE INDEX idx_conv_sessions_active ON conversation_sessions(user_id, is_active) WHERE is_active = true;
CREATE INDEX idx_conv_sessions_activity ON conversation_sessions(last_activity_at DESC);

-- GIN index for context
CREATE INDEX idx_conv_sessions_context ON conversation_sessions USING GIN (context);

-- Comments
COMMENT ON TABLE conversation_sessions IS 'Tracks conversation state for context-aware responses';

-- Auto-expire sessions after 24 hours of inactivity
CREATE OR REPLACE FUNCTION expire_inactive_sessions()
RETURNS void AS $$
BEGIN
    UPDATE conversation_sessions
    SET is_active = false, ended_at = NOW()
    WHERE is_active = true
      AND last_activity_at < NOW() - INTERVAL '24 hours';
END;
$$ LANGUAGE plpgsql;

-- Schedule via pg_cron or external scheduler
```

### 7. System Tables (Logging & Analytics)

```sql
-- =====================================================
-- API USAGE LOGS TABLE
-- =====================================================
CREATE TABLE api_usage_logs (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID REFERENCES users(id) ON DELETE SET NULL,

    -- API Details
    provider TEXT NOT NULL CHECK (provider IN ('openai', 'anthropic', 'usda', 'telegram', 'supabase')),
    endpoint TEXT,
    method TEXT,

    -- Usage Metrics
    input_tokens INTEGER,
    output_tokens INTEGER,
    cache_read_tokens INTEGER DEFAULT 0,
    cache_write_tokens INTEGER DEFAULT 0,

    -- Costs
    cost_usd DECIMAL(10,6),

    -- Performance
    response_time_ms INTEGER,
    success BOOLEAN DEFAULT true,
    error_message TEXT,

    -- Audit
    created_at TIMESTAMPTZ DEFAULT NOW()
);

-- Indexes
CREATE INDEX idx_api_logs_created ON api_usage_logs(created_at DESC);
CREATE INDEX idx_api_logs_user ON api_usage_logs(user_id, created_at DESC);
CREATE INDEX idx_api_logs_provider ON api_usage_logs(provider, created_at DESC);
CREATE INDEX idx_api_logs_date ON api_usage_logs((created_at::date));

-- Partition by month for scalability
CREATE TABLE api_usage_logs_2025_01 PARTITION OF api_usage_logs
    FOR VALUES FROM ('2025-01-01') TO ('2025-02-01');

-- Comments
COMMENT ON TABLE api_usage_logs IS 'Tracks all API usage for cost monitoring and analytics';

-- =====================================================
-- ERROR LOGS TABLE
-- =====================================================
CREATE TABLE error_logs (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID REFERENCES users(id) ON DELETE SET NULL,

    -- Error Details
    error_type TEXT NOT NULL,
    error_message TEXT NOT NULL,
    stack_trace TEXT,

    -- Context
    workflow_name TEXT,
    node_name TEXT,
    request_data JSONB,

    -- Severity
    severity TEXT CHECK (severity IN ('low', 'medium', 'high', 'critical')) DEFAULT 'medium',

    -- Resolution
    resolved BOOLEAN DEFAULT false,
    resolved_at TIMESTAMPTZ,
    resolution_notes TEXT,

    -- Audit
    created_at TIMESTAMPTZ DEFAULT NOW()
);

-- Indexes
CREATE INDEX idx_error_logs_created ON error_logs(created_at DESC);
CREATE INDEX idx_error_logs_type ON error_logs(error_type);
CREATE INDEX idx_error_logs_severity ON error_logs(severity, resolved);
CREATE INDEX idx_error_logs_unresolved ON error_logs(created_at DESC) WHERE resolved = false;

-- Comments
COMMENT ON TABLE error_logs IS 'Centralized error logging for debugging and monitoring';
```

---

## Table Relationships

```
users (1) ─────< (M) user_profiles
  │
  ├──< (M) meal_plans ─────< (M) planned_meals >───── recipes (M)
  │                                │
  ├──< (M) meal_history ───────────┘
  │
  ├──< (M) grocery_lists ─────< (M) grocery_list_items
  │
  ├──< (M) conversation_sessions
  │
  ├──< (M) api_usage_logs
  │
  └──< (M) error_logs

recipes (1) ─────< (M) planned_meals
recipes (1) ─────< (M) meal_history
```

**Key Relationships**:
1. **User → User Profile**: 1:1 (via UNIQUE constraint)
2. **User → Meal Plans**: 1:M
3. **Meal Plan → Planned Meals**: 1:M
4. **Recipe → Planned Meals**: 1:M
5. **Recipe → Meal History**: 1:M (soft reference)
6. **Meal Plan → Grocery List**: 1:1 or 1:M

---

## Row Level Security (RLS)

### Enable RLS on All User Tables

```sql
-- Enable RLS
ALTER TABLE user_profiles ENABLE ROW LEVEL SECURITY;
ALTER TABLE meal_plans ENABLE ROW LEVEL SECURITY;
ALTER TABLE planned_meals ENABLE ROW LEVEL SECURITY;
ALTER TABLE meal_history ENABLE ROW LEVEL SECURITY;
ALTER TABLE grocery_lists ENABLE ROW LEVEL SECURITY;
ALTER TABLE grocery_list_items ENABLE ROW LEVEL SECURITY;
ALTER TABLE conversation_sessions ENABLE ROW LEVEL SECURITY;

-- =====================================================
-- RLS POLICIES
-- =====================================================

-- User Profiles: Users can only see/edit their own profile
CREATE POLICY user_profiles_select_own ON user_profiles
    FOR SELECT
    USING (user_id = auth.uid());

CREATE POLICY user_profiles_update_own ON user_profiles
    FOR UPDATE
    USING (user_id = auth.uid());

CREATE POLICY user_profiles_insert_own ON user_profiles
    FOR INSERT
    WITH CHECK (user_id = auth.uid());

-- Meal Plans: Users can only see/manage their own meal plans
CREATE POLICY meal_plans_select_own ON meal_plans
    FOR SELECT
    USING (user_id = auth.uid());

CREATE POLICY meal_plans_all_own ON meal_plans
    FOR ALL
    USING (user_id = auth.uid());

-- Planned Meals: Users can only see meals in their meal plans
CREATE POLICY planned_meals_select_own ON planned_meals
    FOR SELECT
    USING (
        meal_plan_id IN (
            SELECT id FROM meal_plans WHERE user_id = auth.uid()
        )
    );

CREATE POLICY planned_meals_all_own ON planned_meals
    FOR ALL
    USING (
        meal_plan_id IN (
            SELECT id FROM meal_plans WHERE user_id = auth.uid()
        )
    );

-- Meal History: Users can only see their own history
CREATE POLICY meal_history_select_own ON meal_history
    FOR SELECT
    USING (user_id = auth.uid());

CREATE POLICY meal_history_insert_own ON meal_history
    FOR INSERT
    WITH CHECK (user_id = auth.uid());

-- Grocery Lists: Users can only see their own lists
CREATE POLICY grocery_lists_select_own ON grocery_lists
    FOR SELECT
    USING (user_id = auth.uid());

CREATE POLICY grocery_lists_all_own ON grocery_lists
    FOR ALL
    USING (user_id = auth.uid());

-- Grocery List Items: Users can only see items in their lists
CREATE POLICY grocery_items_select_own ON grocery_list_items
    FOR SELECT
    USING (
        grocery_list_id IN (
            SELECT id FROM grocery_lists WHERE user_id = auth.uid()
        )
    );

CREATE POLICY grocery_items_all_own ON grocery_list_items
    FOR ALL
    USING (
        grocery_list_id IN (
            SELECT id FROM grocery_lists WHERE user_id = auth.uid()
        )
    );

-- Conversation Sessions: Users can only see their own sessions
CREATE POLICY conv_sessions_select_own ON conversation_sessions
    FOR SELECT
    USING (user_id = auth.uid());

CREATE POLICY conv_sessions_all_own ON conversation_sessions
    FOR ALL
    USING (user_id = auth.uid());

-- Recipes: Everyone can read, only system can write
CREATE POLICY recipes_select_all ON recipes
    FOR SELECT
    USING (true); -- Public read access

-- Note: For n8n integration, use service role key which bypasses RLS
```

---

## Database Functions

### 1. Get or Create User

```sql
CREATE OR REPLACE FUNCTION get_or_create_user(
    p_telegram_user_id BIGINT,
    p_telegram_username TEXT DEFAULT NULL,
    p_telegram_first_name TEXT DEFAULT NULL,
    p_telegram_last_name TEXT DEFAULT NULL
)
RETURNS TABLE (
    user_id UUID,
    is_new_user BOOLEAN,
    profile JSONB
) AS $$
DECLARE
    v_user_id UUID;
    v_is_new BOOLEAN := false;
BEGIN
    -- Try to find existing user
    SELECT id INTO v_user_id
    FROM users
    WHERE telegram_user_id = p_telegram_user_id;

    -- Create if doesn't exist
    IF v_user_id IS NULL THEN
        INSERT INTO users (telegram_user_id, telegram_username, telegram_first_name, telegram_last_name)
        VALUES (p_telegram_user_id, p_telegram_username, p_telegram_first_name, p_telegram_last_name)
        RETURNING id INTO v_user_id;

        -- Create default profile
        INSERT INTO user_profiles (user_id)
        VALUES (v_user_id);

        v_is_new := true;
    ELSE
        -- Update last seen
        UPDATE users
        SET last_seen_at = NOW(),
            telegram_username = COALESCE(p_telegram_username, telegram_username)
        WHERE id = v_user_id;
    END IF;

    -- Return user and profile
    RETURN QUERY
    SELECT
        v_user_id,
        v_is_new,
        row_to_json(up.*)::jsonb
    FROM user_profiles up
    WHERE up.user_id = v_user_id;
END;
$$ LANGUAGE plpgsql;

-- Usage in n8n:
-- SELECT * FROM get_or_create_user(123456789, 'john_doe', 'John', 'Doe');
```

### 2. Get User's Recent Meals (Prevent Repeats)

```sql
CREATE OR REPLACE FUNCTION get_recent_meals(
    p_user_id UUID,
    p_days INTEGER DEFAULT 30
)
RETURNS TABLE (
    recipe_id UUID,
    recipe_name TEXT,
    consumed_at TIMESTAMPTZ,
    cuisine_type TEXT
) AS $$
BEGIN
    RETURN QUERY
    SELECT
        r.id,
        r.name,
        mh.consumed_at,
        r.cuisine_type
    FROM meal_history mh
    JOIN recipes r ON r.id = mh.recipe_id
    WHERE mh.user_id = p_user_id
      AND mh.consumed_at > NOW() - (p_days || ' days')::INTERVAL
    ORDER BY mh.consumed_at DESC;
END;
$$ LANGUAGE plpgsql;
```

### 3. Calculate Daily Nutrition Summary

```sql
CREATE OR REPLACE FUNCTION get_daily_nutrition(
    p_user_id UUID,
    p_date DATE DEFAULT CURRENT_DATE
)
RETURNS TABLE (
    total_calories INTEGER,
    total_protein_g DECIMAL,
    total_carbs_g DECIMAL,
    total_fat_g DECIMAL,
    meal_count INTEGER
) AS $$
BEGIN
    RETURN QUERY
    SELECT
        SUM(actual_calories)::INTEGER,
        SUM(actual_protein_g),
        SUM(actual_carbs_g),
        SUM(actual_fat_g),
        COUNT(*)::INTEGER
    FROM meal_history
    WHERE user_id = p_user_id
      AND consumed_at::date = p_date;
END;
$$ LANGUAGE plpgsql;
```

### 4. Get Popular Recipes

```sql
CREATE OR REPLACE FUNCTION get_popular_recipes(
    p_cooking_method TEXT DEFAULT NULL,
    p_limit INTEGER DEFAULT 10
)
RETURNS TABLE (
    recipe_id UUID,
    recipe_name TEXT,
    avg_rating DECIMAL,
    times_made INTEGER,
    cuisine_type TEXT
) AS $$
BEGIN
    RETURN QUERY
    SELECT
        r.id,
        r.name,
        r.avg_user_rating,
        r.times_generated,
        r.cuisine_type
    FROM recipes r
    WHERE (p_cooking_method IS NULL OR r.cooking_method = p_cooking_method)
      AND r.avg_user_rating IS NOT NULL
    ORDER BY r.avg_user_rating DESC, r.times_generated DESC
    LIMIT p_limit;
END;
$$ LANGUAGE plpgsql;
```

---

## Critical Analysis

### ✅ Strengths

1. **Proper Normalization**: Avoids data redundancy
2. **Referential Integrity**: Foreign keys enforce relationships
3. **Audit Trail**: All tables have timestamps
4. **Flexible Data**: JSONB for semi-structured data
5. **Security**: RLS policies protect user data
6. **Performance**: Strategic indexing on common queries
7. **Scalability**: Partitioning strategy for logs

### ⚠️ Potential Issues & Solutions

#### Issue 1: JSONB Query Performance
**Problem**: Deep queries on JSONB can be slow
**Solution**: Add GIN indexes + consider materialized views for common aggregations

#### Issue 2: Unbounded Table Growth
**Problem**: `meal_history` and `api_usage_logs` will grow indefinitely
**Solution**: Implement partitioning by month + archival strategy

```sql
-- Partition meal_history by month
CREATE TABLE meal_history (
    id UUID NOT NULL,
    -- ... other columns
    consumed_at TIMESTAMPTZ NOT NULL,
    PRIMARY KEY (id, consumed_at)
) PARTITION BY RANGE (consumed_at);

-- Create monthly partitions
CREATE TABLE meal_history_2025_01 PARTITION OF meal_history
    FOR VALUES FROM ('2025-01-01') TO ('2025-02-01');

-- Automate partition creation via cron
```

#### Issue 3: Race Conditions on Recipe Ratings
**Problem**: Concurrent updates to `avg_user_rating` could be inaccurate
**Solution**: Use trigger with SELECT FOR UPDATE

```sql
CREATE OR REPLACE FUNCTION update_recipe_ratings_safe()
RETURNS TRIGGER AS $$
DECLARE
    v_recipe RECORD;
BEGIN
    IF NEW.rating IS NOT NULL THEN
        SELECT * INTO v_recipe
        FROM recipes
        WHERE id = NEW.recipe_id
        FOR UPDATE; -- Lock row

        UPDATE recipes
        SET
            total_ratings = total_ratings + 1,
            avg_user_rating = (
                SELECT AVG(rating)
                FROM meal_history
                WHERE recipe_id = NEW.recipe_id AND rating IS NOT NULL
            )
        WHERE id = NEW.recipe_id;
    END IF;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;
```

#### Issue 4: Soft Delete vs Hard Delete
**Problem**: Cascading deletes might remove important history
**Solution**: Add `deleted_at` for soft deletes on critical tables

```sql
ALTER TABLE recipes ADD COLUMN deleted_at TIMESTAMPTZ;

-- Update RLS policies
CREATE POLICY recipes_select_active ON recipes
    FOR SELECT
    USING (deleted_at IS NULL);
```

---

## Migrations Strategy

### Migration File Structure

```
migrations/
├── 001_initial_schema.sql
├── 002_add_rls_policies.sql
├── 003_add_database_functions.sql
├── 004_add_indexes.sql
├── 005_add_partitioning.sql
└── 006_seed_data.sql
```

### Migration Best Practices

1. **Always Use Transactions**
```sql
BEGIN;
    -- migration code
COMMIT;
```

2. **Make Migrations Reversible**
```sql
-- Up migration
ALTER TABLE recipes ADD COLUMN difficulty_level TEXT;

-- Down migration (separate file)
ALTER TABLE recipes DROP COLUMN difficulty_level;
```

3. **Test on Staging First**
4. **Backup Before Migration**
```bash
pg_dump $DATABASE_URL > backup_$(date +%Y%m%d).sql
```

---

## Performance Monitoring Queries

### 1. Find Slow Queries
```sql
SELECT
    query,
    calls,
    total_exec_time,
    mean_exec_time,
    max_exec_time
FROM pg_stat_statements
ORDER BY mean_exec_time DESC
LIMIT 10;
```

### 2. Find Missing Indexes
```sql
SELECT
    schemaname,
    tablename,
    attname,
    n_distinct,
    correlation
FROM pg_stats
WHERE schemaname = 'public'
  AND n_distinct > 100
  AND correlation < 0.1;
```

### 3. Table Sizes
```sql
SELECT
    schemaname,
    tablename,
    pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename)) AS size
FROM pg_tables
WHERE schemaname = 'public'
ORDER BY pg_total_relation_size(schemaname||'.'||tablename) DESC;
```

---

## Next Steps

1. ✅ Review this schema documentation
2. 📝 Run initial schema migration
3. 🧪 Test database functions
4. 🔒 Verify RLS policies
5. 📊 Set up monitoring
6. 🚀 Proceed to [Prompt Engineering](./03-prompt-engineering.md)

---

## Version History

- **v1.0** (2025-01-15): Initial schema with RLS, functions, and partitioning
- **v1.1** (Pending): Add full-text search optimization
- **v1.2** (Pending): Add materialized views for analytics
