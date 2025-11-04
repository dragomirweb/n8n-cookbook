# Implementation Roadmap - 4-Week Plan

## Overview
Detailed 4-week implementation plan from initial setup to production launch with milestones, deliverables, and success criteria.

## Table of Contents
1. [Week 1: Foundation](#week-1-foundation)
2. [Week 2: Core Features](#week-2-core-features)
3. [Week 3: Optimization](#week-3-optimization)
4. [Week 4: Production Prep](#week-4-production-prep)
5. [Post-Launch Plan](#post-launch-plan)

---

## Week 1: Foundation

### Days 1-2: Infrastructure Setup

**Goals:**
- Set up all foundational infrastructure
- Verify all services are accessible
- Establish development workflow

**Tasks:**

#### Day 1: Database & Cloud Services
- [x] Create Supabase project
  - Sign up at supabase.com
  - Create new project: `omad-meal-planner`
  - Save connection details
- [ ] Run database schema
  ```bash
  psql $SUPABASE_URL -f plan/sql/01_schema.sql
  psql $SUPABASE_URL -f plan/sql/02_functions.sql
  psql $SUPABASE_URL -f plan/sql/03_rls_policies.sql
  psql $SUPABASE_URL -f plan/sql/04_indexes.sql
  ```
- [ ] Verify tables created
  ```bash
  psql $SUPABASE_URL -c "\dt"
  ```
- [ ] Set up Redis (local or cloud)
  ```bash
  # Option 1: Local
  docker run -d -p 6379:6379 redis:7-alpine

  # Option 2: Upstash (free tier)
  # Sign up at upstash.com
  ```
- [ ] Test Redis connection
  ```bash
  redis-cli ping
  # Expected: PONG
  ```

#### Day 2: n8n & API Keys
- [ ] Set up n8n instance
  ```bash
  # Option 1: Docker
  docker run -it --rm \
    -p 5678:5678 \
    -v ~/.n8n:/home/node/.n8n \
    n8nio/n8n

  # Option 2: npm
  npm install -g n8n
  n8n start
  ```
- [ ] Configure environment variables
  ```bash
  # Create .env file
  SUPABASE_URL=https://xxx.supabase.co
  SUPABASE_API_KEY=your-anon-key
  SUPABASE_SERVICE_ROLE_KEY=your-service-key
  REDIS_URL=redis://localhost:6379
  ```
- [ ] Obtain API keys
  - Anthropic: https://console.anthropic.com/
  - OpenAI: https://platform.openai.com/
  - USDA: https://fdc.nal.usda.gov/api-key-signup.html
- [ ] Add credentials to n8n
  - Anthropic API
  - OpenAI API
  - Supabase API
  - Telegram API (placeholder for now)

**Deliverables:**
- ✅ Supabase database with all tables
- ✅ Redis instance running
- ✅ n8n instance accessible
- ✅ All API credentials configured

**Success Criteria:**
- Database schema validates successfully
- Redis PING returns PONG
- n8n UI accessible at localhost:5678
- Test API calls succeed

---

### Days 3-4: Telegram Integration

**Goals:**
- Create and configure Telegram bot
- Build basic message handling workflow
- Test end-to-end communication

**Tasks:**

#### Day 3: Bot Creation & Setup
- [ ] Create Telegram bot with BotFather
  - Message @BotFather: `/newbot`
  - Name: `OMAD Meal Planner`
  - Username: `omad_meal_planner_bot`
  - Save token
- [ ] Configure bot settings
  ```
  /setdescription - Set description
  /setabouttext - Set about
  /setcommands - Set command list
  ```
- [ ] Add Telegram credentials to n8n
  - Credentials → New → Telegram API
  - Paste bot token
  - Test connection

#### Day 4: Basic Workflow
- [ ] Create "Main Telegram Handler" workflow
  - Telegram Webhook Trigger
  - Extract User Data (Code node)
  - Get/Create User (HTTP → Supabase)
  - Simple Echo Response (Telegram Send)
- [ ] Test workflow
  - Send `/start` to bot
  - Verify response received
  - Check user created in database
- [ ] Add typing indicator
- [ ] Add basic error handling

**Deliverables:**
- ✅ Telegram bot active
- ✅ Basic message handling workflow
- ✅ User creation working

**Success Criteria:**
- Bot responds to `/start` command
- New users created in database
- No errors in n8n execution log

---

### Days 5-7: First Prototype

**Goals:**
- Build Cook4me meal generator workflow
- Implement basic prompt template
- Generate first AI meal successfully

**Tasks:**

#### Day 5: Prompt Engineering
- [ ] Create system prompt template (Code node)
  - Copy from `plan/03-prompt-engineering.md`
  - Implement prompt caching structure
- [ ] Create user profile context builder
- [ ] Test prompt locally (Anthropic Workbench)

#### Day 6: AI Integration
- [ ] Create "Generate Cook4me Meal" sub-workflow
  - Input: user_id, preferences
  - Build cached prompt (Code node)
  - Call Anthropic API (HTTP Request)
  - Parse JSON response (Code node)
  - Validate nutrition (Code node)
- [ ] Test with sample user profile
- [ ] Verify nutritional targets met

#### Day 7: End-to-End Test
- [ ] Connect Telegram → Meal Generation
  - Add intent classification
  - Route to meal generator
  - Format response for Telegram
  - Send to user
- [ ] Test complete flow
  - User sends "Generate chicken meal"
  - Bot generates and returns meal
  - Verify recipe saved to database
- [ ] Fix any bugs

**Deliverables:**
- ✅ Working Cook4me meal generator
- ✅ End-to-end Telegram integration
- ✅ First AI-generated meal

**Success Criteria:**
- Generate 10 meals successfully
- All meals meet nutritional targets (±5%)
- Average generation time < 30s
- Recipes saved to database correctly

**Week 1 Checkpoint:**
- Infrastructure: ✅ Complete
- Telegram Bot: ✅ Working
- Meal Generation: ✅ Basic prototype
- Database: ✅ Operational

---

## Week 2: Core Features

### Days 8-10: Meal Generation Enhancements

**Goals:**
- Complete traditional meal generator
- Add USDA validation
- Implement nutritional accuracy checks

**Tasks:**

#### Day 8: Traditional Cooking Method
- [ ] Create "Generate Traditional Meal" sub-workflow
  - Adapt system prompt for traditional cooking
  - Support multiple components (protein, sides, vegetables)
  - Test with sample requests
- [ ] Add cooking method selection
  - Update Telegram handler
  - Add inline keyboard for selection
  - Route to correct sub-workflow

#### Day 9: USDA Integration
- [ ] Create USDA validation function
  - Search USDA database by ingredient
  - Compare AI nutrition vs USDA data
  - Calculate error percentage
- [ ] Integrate into meal generation
  - Validate after AI generation
  - Log validation results
  - Regenerate if > 15% error

#### Day 10: Nutritional Validation
- [ ] Implement comprehensive validation
  - Copy tests from `plan/08-testing-validation.md`
  - Create validation code node
  - Test with 50 generated meals
- [ ] Add validation failure handling
  - Regenerate on critical failures
  - Log validation metrics
  - Alert if consistent failures

**Deliverables:**
- ✅ Traditional meal generator
- ✅ USDA validation integrated
- ✅ Comprehensive validation suite

**Success Criteria:**
- Generate both Cook4me and traditional meals
- 95%+ meals pass validation
- USDA variance < 10% for 80% of ingredients

---

### Days 11-14: Weekly Planning & Grocery Lists

**Goals:**
- Build weekly meal planner
- Implement variety validation
- Auto-generate grocery lists

**Tasks:**

#### Day 11: Weekly Plan Generator
- [ ] Create "Generate Weekly Plan" sub-workflow
  - Build weekly planning prompt
  - Request 7 meals in single API call
  - Parse 7-day response
- [ ] Test variety in generated plans
  - Check protein variety (min 3 types)
  - Check cuisine variety (min 4 types)
  - Ensure no repeats

#### Day 12: Variety Validation
- [ ] Implement variety checking
  - Extract proteins from each day
  - Extract cuisines
  - Check for duplicates
- [ ] Add regeneration logic
  - If variety fails, regenerate specific day
  - Max 3 regeneration attempts
  - Log variety scores

#### Day 13: Grocery List Generator
- [ ] Create grocery list extraction
  - Parse all ingredients from 7 meals
  - Consolidate duplicates
  - Convert to purchase units
  - Group by category
- [ ] Format for user display
  - Markdown table
  - Categorized sections
  - Shopping tips

#### Day 14: Database Integration
- [ ] Save meal plans to database
  - Create meal_plan record
  - Insert 7 recipes
  - Link planned_meals
  - Generate grocery_list
- [ ] Test retrieval
  - Get active meal plan
  - View specific day
  - Check off grocery items

**Deliverables:**
- ✅ Weekly meal planner
- ✅ Automated grocery lists
- ✅ Full database integration

**Success Criteria:**
- Generate 10 weekly plans
- All plans pass variety validation
- Grocery lists consolidate correctly
- Plans save/retrieve from database

**Week 2 Checkpoint:**
- Single Meals: ✅ Both methods working
- Weekly Plans: ✅ Generation & validation
- Grocery Lists: ✅ Auto-generated
- Database: ✅ Full integration

---

## Week 3: Optimization

### Days 15-17: Performance & Caching

**Goals:**
- Implement prompt caching
- Add Redis application cache
- Optimize token usage

**Tasks:**

#### Day 15: Prompt Caching
- [ ] Update all AI calls with caching
  - Mark system prompts for cache
  - Mark user profiles for cache
  - Test cache behavior
- [ ] Monitor cache hit rates
  - Add usage logging
  - Track cache read tokens
  - Calculate savings

#### Day 16: Application Caching
- [ ] Implement Redis caching layer
  - Cache identical requests (24h)
  - Cache similar meals (fuzzy match)
  - Implement cache invalidation
- [ ] Test cache effectiveness
  - Measure hit rates
  - Track response times
  - Calculate cost savings

#### Day 17: Cost Optimization
- [ ] Implement model routing
  - Use GPT-4o mini for intent classification
  - Use Claude for meal generation
  - Test quality vs cost
- [ ] Optimize prompts
  - Reduce token count where possible
  - Maintain quality
  - Version prompts

**Deliverables:**
- ✅ Prompt caching implemented
- ✅ Redis caching layer
- ✅ Model routing strategy

**Success Criteria:**
- Prompt cache hit rate > 60%
- Application cache hit rate > 40%
- Cost per request reduced 30%+

---

### Days 18-21: User Experience

**Goals:**
- Add inline keyboards
- Improve messaging
- Implement conversation state

**Tasks:**

#### Day 18: Interactive Features
- [ ] Add inline keyboards
  - Cooking method selection
  - Meal actions (rate, save, share)
  - Navigation buttons
- [ ] Handle callback queries
  - Process button clicks
  - Update messages
  - Track user selections

#### Day 19: Conversation State
- [ ] Implement state management
  - Store conversation context in Redis
  - Track pending actions
  - Multi-step flows
- [ ] Add preferences management
  - /settings command
  - Step-by-step updates
  - Confirmation messages

#### Day 20: Message Formatting
- [ ] Improve message formatting
  - Use MarkdownV2
  - Add emojis appropriately
  - Format nutrition tables
  - Include helpful tips
- [ ] Add progress indicators
  - "Generating..." message
  - Update with progress
  - Final result message

#### Day 21: Help & Documentation
- [ ] Create /help command
  - List all commands
  - Provide examples
  - Link to documentation
- [ ] Add onboarding flow
  - Welcome message
  - Quick tutorial
  - Sample meal generation

**Deliverables:**
- ✅ Interactive bot with keyboards
- ✅ Conversation state management
- ✅ Polished messaging

**Success Criteria:**
- All commands documented
- Onboarding flow complete
- Positive user feedback
- No usability issues

**Week 3 Checkpoint:**
- Performance: ✅ Optimized
- Caching: ✅ Working
- UX: ✅ Polished
- Cost: ✅ Reduced 30%+

---

## Week 4: Production Prep

### Days 22-24: Testing

**Goals:**
- Unit test all workflows
- Integration test full user journeys
- Load test with concurrent users

**Tasks:**

#### Day 22: Unit Testing
- [ ] Test meal generation
  - Generate 100 Cook4me meals
  - Generate 100 traditional meals
  - Validate all pass nutrition tests
- [ ] Test weekly planner
  - Generate 20 weekly plans
  - Validate variety requirements
  - Check grocery list accuracy

#### Day 23: Integration Testing
- [ ] Test complete user journeys
  - New user onboarding
  - Single meal request
  - Weekly plan request
  - Grocery list viewing
  - Settings update
- [ ] Test error scenarios
  - API failures
  - Invalid inputs
  - Network issues

#### Day 24: Load Testing
- [ ] Simulate concurrent users
  - 100 concurrent users
  - 10 requests per user
  - Measure response times
- [ ] Test performance
  - Database query times
  - API response times
  - Cache hit rates

**Deliverables:**
- ✅ Comprehensive test results
- ✅ Performance benchmarks
- ✅ Bug fixes completed

**Success Criteria:**
- 95%+ meals pass validation
- Response time < 30s (p95)
- 99%+ uptime during load test

---

### Days 25-28: Launch Preparation

**Goals:**
- Deploy to production
- Set up monitoring
- Launch beta

**Tasks:**

#### Day 25: Production Deployment
- [ ] Deploy n8n to cloud
  - n8n Cloud or self-hosted
  - Set production environment variables
  - Configure webhooks
- [ ] Configure production database
  - Supabase production instance
  - Run migrations
  - Set up backups
- [ ] Set up Redis production
  - Upstash or Redis Cloud
  - Configure persistence
  - Test connection

#### Day 26: Monitoring & Alerts
- [ ] Set up monitoring dashboard
  - Cost tracking
  - Error logs
  - Performance metrics
- [ ] Configure alerts
  - Daily cost threshold
  - Error rate threshold
  - Performance degradation
- [ ] Create admin commands
  - /debug for troubleshooting
  - Health check endpoint

#### Day 27: Beta Launch (10 Users)
- [ ] Invite beta users
  - Friends/family
  - Provide instructions
  - Collect feedback
- [ ] Monitor closely
  - Check error logs
  - Track costs
  - Measure response times
- [ ] Iterate on feedback
  - Fix critical bugs
  - Improve UX
  - Adjust prompts

#### Day 28: Full Launch
- [ ] Open to public
  - Share bot link
  - Post in communities
  - Monitor growth
- [ ] Set up support
  - Create support channel
  - Document FAQs
  - Respond to users

**Deliverables:**
- ✅ Production deployment
- ✅ Monitoring active
- ✅ Beta tested
- ✅ Public launch

**Success Criteria:**
- Beta users satisfied (8/10+ rating)
- Error rate < 1%
- Cost per user < $1.50/month
- Response times acceptable

**Week 4 Checkpoint:**
- Testing: ✅ Complete
- Deployment: ✅ Production ready
- Monitoring: ✅ Active
- Launch: ✅ Successful

---

## Post-Launch Plan

### Week 5-8: Iteration & Growth

**Goals:**
- Stabilize production
- Optimize costs further
- Add requested features

**Tasks:**

**Week 5:**
- [ ] Monitor and fix issues
- [ ] Optimize based on real usage
- [ ] Reach 100 users

**Week 6:**
- [ ] Implement user-requested features
- [ ] A/B test prompt variations
- [ ] Improve cache hit rates

**Week 7:**
- [ ] Add meal rating system
- [ ] Implement personalized recommendations
- [ ] Reach 500 users

**Week 8:**
- [ ] Add meal history insights
- [ ] Implement macro tracking
- [ ] Plan for scale (1000+ users)

---

## Milestone Checklist

### Week 1: Foundation ✅
- [x] Infrastructure set up
- [x] Telegram bot created
- [x] Basic meal generation working

### Week 2: Core Features ✅
- [x] Both cooking methods implemented
- [x] Weekly planner working
- [x] Grocery lists generated

### Week 3: Optimization ✅
- [x] Caching implemented
- [x] Costs optimized 30%+
- [x] UX polished

### Week 4: Production ✅
- [x] Testing complete
- [x] Monitoring active
- [x] Beta launched
- [x] Public release

---

## Risk Mitigation

| Risk | Probability | Impact | Mitigation |
|------|------------|--------|------------|
| API rate limits | Medium | High | Implement fallback providers, caching |
| High costs | Medium | High | Aggressive caching, cost monitoring |
| Nutritional inaccuracy | Low | Critical | USDA validation, batch testing |
| Database downtime | Low | High | Supabase SLA, backup strategy |
| Poor user adoption | Medium | Medium | Beta testing, feedback loop |

---

## Next Steps

1. ✅ Review this roadmap
2. 📅 Set up project tracking (Notion, Trello, etc.)
3. 🚀 Begin Week 1: Infrastructure Setup
4. 📖 Refer to other plan documents as needed

---

## Version History

- **v1.0** (2025-01-15): Initial 4-week roadmap
