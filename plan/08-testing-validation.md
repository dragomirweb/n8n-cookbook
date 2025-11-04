# Testing & Validation - Complete Guide

## Overview
Comprehensive testing strategy ensuring 95%+ nutritional accuracy, 99% uptime, and production-ready quality.

## Table of Contents
1. [Testing Strategy](#testing-strategy)
2. [Nutritional Accuracy Testing](#nutritional-accuracy-testing)
3. [Workflow Testing](#workflow-testing)
4. [Load & Performance Testing](#load--performance-testing)
5. [Integration Testing](#integration-testing)
6. [User Acceptance Testing](#user-acceptance-testing)
7. [Continuous Testing](#continuous-testing)

---

## Testing Strategy

### Test Pyramid

```
           /\
          /  \         E2E Tests (10%)
         /    \        - Full user journeys
        /------\       - Real integrations
       /        \
      /          \     Integration Tests (30%)
     /            \    - API integrations
    /--------------\   - Database operations
   /                \
  /                  \ Unit Tests (60%)
 /____________________\ - Individual nodes
                        - Data transformations
```

### Testing Phases

1. **Development Testing** (Continuous)
   - Unit tests for each node
   - Immediate feedback

2. **Integration Testing** (Pre-deployment)
   - Test full workflows
   - Validate API integrations

3. **Staging Testing** (Pre-production)
   - Load testing
   - Performance benchmarks

4. **Production Monitoring** (Post-deployment)
   - Real-time validation
   - User feedback loop

---

## Nutritional Accuracy Testing

### Test Suite for Meal Validation

```javascript
// Code Node: Comprehensive Nutritional Tests
const NUTRITIONAL_TESTS = [
    {
        name: "Calorie Target Accuracy",
        test: (meal) => {
            const target = 2300;
            const tolerance = 115; // ±5%
            const actual = meal.nutritional_summary.total_calories;

            return {
                passed: Math.abs(actual - target) <= tolerance,
                expected: `${target - tolerance} - ${target + tolerance}`,
                actual: actual,
                error_percent: Math.abs((actual - target) / target * 100).toFixed(2)
            };
        },
        critical: true
    },
    {
        name: "Protein Range",
        test: (meal) => {
            const min = 130;
            const max = 160;
            const actual = meal.nutritional_summary.total_protein_g;

            return {
                passed: actual >= min && actual <= max,
                expected: `${min}g - ${max}g`,
                actual: `${actual}g`
            };
        },
        critical: true
    },
    {
        name: "Carbohydrate Range",
        test: (meal) => {
            const min = 230;
            const max = 280;
            const actual = meal.nutritional_summary.total_carbs_g;

            return {
                passed: actual >= min && actual <= max,
                expected: `${min}g - ${max}g`,
                actual: `${actual}g`
            };
        },
        critical: true
    },
    {
        name: "Fat Range",
        test: (meal) => {
            const min = 70;
            const max = 90;
            const actual = meal.nutritional_summary.total_fat_g;

            return {
                passed: actual >= min && actual <= max,
                expected: `${min}g - ${max}g`,
                actual: `${actual}g`
            };
        },
        critical: true
    },
    {
        name: "Minimum Fiber",
        test: (meal) => {
            const min = 30;
            const actual = meal.nutritional_summary.total_fiber_g;

            return {
                passed: actual >= min,
                expected: `≥${min}g`,
                actual: `${actual}g`
            };
        },
        critical: false
    },
    {
        name: "Ingredient Nutrition Sum",
        test: (meal) => {
            const summedCalories = meal.ingredients.reduce((sum, ing) => sum + ing.calories, 0);
            const reported = meal.nutritional_summary.total_calories;
            const diff = Math.abs(summedCalories - reported);
            const tolerance = reported * 0.05; // 5% tolerance

            return {
                passed: diff <= tolerance,
                expected: `±${tolerance.toFixed(0)} cal difference`,
                actual: `${diff.toFixed(0)} cal difference`,
                summed: summedCalories,
                reported: reported
            };
        },
        critical: true
    },
    {
        name: "Cook4me Format Compliance",
        test: (meal) => {
            if (meal.cooking_method !== 'cook4me') {
                return { passed: true, skipped: true };
            }

            const instructions = meal.instructions.map(i => i.instruction || i).join(' ');
            const checks = {
                has_pressure_time: /Pressure cooking time = \d+ min/i.test(instructions),
                has_close_lid: /close.*lid/i.test(instructions),
                has_liquid: meal.ingredients.some(i => i.unit === 'ml' && i.quantity >= 200)
            };

            const allPassed = Object.values(checks).every(c => c);

            return {
                passed: allPassed,
                expected: 'All Cook4me requirements met',
                actual: checks,
                details: !allPassed ? 'Missing required Cook4me instructions' : 'OK'
            };
        },
        critical: true
    }
];

async function validateMeal(meal) {
    const results = {
        meal_name: meal.meal_name,
        timestamp: new Date().toISOString(),
        tests: [],
        summary: {
            total: NUTRITIONAL_TESTS.length,
            passed: 0,
            failed: 0,
            critical_failures: 0
        }
    };

    for (const testDef of NUTRITIONAL_TESTS) {
        const result = testDef.test(meal);

        results.tests.push({
            name: testDef.name,
            passed: result.passed,
            skipped: result.skipped || false,
            critical: testDef.critical,
            expected: result.expected,
            actual: result.actual,
            details: result.details || null
        });

        if (result.skipped) continue;

        if (result.passed) {
            results.summary.passed++;
        } else {
            results.summary.failed++;
            if (testDef.critical) {
                results.summary.critical_failures++;
            }
        }
    }

    results.summary.pass_rate = (results.summary.passed / results.summary.total * 100).toFixed(2);
    results.overall_passed = results.summary.critical_failures === 0;

    return results;
}

// Usage
const validation = await validateMeal($json.meal);

// Log results
console.log('Validation Results:', JSON.stringify(validation, null, 2));

// Save to database
await $supabase.from('meal_validations').insert(validation);

// If critical failures, regenerate meal
if (!validation.overall_passed) {
    throw new Error(`Meal validation failed: ${validation.summary.critical_failures} critical failures`);
}

return [{ json: { meal: $json.meal, validation } }];
```

### USDA Cross-Validation

```javascript
// Code Node: Validate Against USDA Database
async function validateWithUSDA(meal) {
    const validationResults = [];

    for (const ingredient of meal.ingredients) {
        // Search USDA database
        const usdaData = await searchUSDAFood(ingredient.item);

        if (!usdaData || usdaData.length === 0) {
            validationResults.push({
                ingredient: ingredient.item,
                status: 'no_data',
                message: 'No USDA data found',
                severity: 'warning'
            });
            continue;
        }

        const usdaNutrients = usdaData[0].nutrients;

        // Compare nutritional values
        const comparisons = {
            calories: {
                ai: ingredient.calories,
                usda: usdaNutrients.calories,
                error: Math.abs(ingredient.calories - usdaNutrients.calories) / usdaNutrients.calories * 100
            },
            protein: {
                ai: ingredient.protein_g,
                usda: usdaNutrients.protein_g,
                error: Math.abs(ingredient.protein_g - usdaNutrients.protein_g) / usdaNutrients.protein_g * 100
            }
        };

        // Check if errors exceed threshold (10%)
        const errors = Object.entries(comparisons).filter(([key, data]) => data.error > 10);

        if (errors.length > 0) {
            validationResults.push({
                ingredient: ingredient.item,
                status: 'error_high',
                severity: errors.some(([k, d]) => d.error > 20) ? 'error' : 'warning',
                comparisons,
                message: `High variance detected: ${errors.map(([k]) => k).join(', ')}`
            });
        } else {
            validationResults.push({
                ingredient: ingredient.item,
                status: 'ok',
                severity: 'info',
                comparisons
            });
        }
    }

    // Overall validation
    const errorCount = validationResults.filter(r => r.severity === 'error').length;
    const warningCount = validationResults.filter(r => r.severity === 'warning').length;

    return {
        valid: errorCount === 0,
        total_ingredients: meal.ingredients.length,
        errors: errorCount,
        warnings: warningCount,
        details: validationResults
    };
}

const usdaValidation = await validateWithUSDA($json.meal);

if (!usdaValidation.valid) {
    console.warn('USDA validation failed:', usdaValidation.details);
    // Optionally regenerate or adjust nutrition values
}

return [{ json: { meal: $json.meal, usda_validation: usdaValidation } }];
```

### Batch Testing (100 Meals)

```javascript
// Code Node: Generate and Test 100 Meals
async function batchTestMeals(count = 100) {
    const results = {
        total: count,
        passed: 0,
        failed: 0,
        validations: [],
        start_time: new Date().toISOString()
    };

    for (let i = 0; i < count; i++) {
        try {
            // Generate meal
            const meal = await generateMeal({
                user_preferences: getRandomPreferences()
            });

            // Validate
            const validation = await validateMeal(meal);

            results.validations.push(validation);

            if (validation.overall_passed) {
                results.passed++;
            } else {
                results.failed++;
            }

            // Progress update
            if ((i + 1) % 10 === 0) {
                console.log(`Progress: ${i + 1}/${count} meals tested`);
            }

        } catch (error) {
            results.failed++;
            results.validations.push({
                error: error.message,
                passed: false
            });
        }
    }

    results.end_time = new Date().toISOString();
    results.pass_rate = (results.passed / results.total * 100).toFixed(2) + '%';

    // Aggregate statistics
    results.stats = {
        avg_calories: calculateAvg(results.validations, 'meal.nutritional_summary.total_calories'),
        avg_protein: calculateAvg(results.validations, 'meal.nutritional_summary.total_protein_g'),
        common_failures: findCommonFailures(results.validations)
    };

    return results;
}

// Run batch test
const testResults = await batchTestMeals(100);

console.log('Batch Test Results:', JSON.stringify(testResults, null, 2));

// Target: 95%+ pass rate
if (parseFloat(testResults.pass_rate) < 95) {
    await sendAdminAlert(`⚠️ Batch test pass rate below target: ${testResults.pass_rate}`);
}

return [{ json: testResults }];
```

---

## Workflow Testing

### Test Individual Workflow

```javascript
// Code Node: Test Workflow Execution
async function testWorkflow(workflowName, testCases) {
    const results = {
        workflow: workflowName,
        total_cases: testCases.length,
        passed: 0,
        failed: 0,
        cases: []
    };

    for (const testCase of testCases) {
        const startTime = Date.now();

        try {
            // Execute workflow
            const result = await $execution.executeWorkflow(workflowName, testCase.input);

            const duration = Date.now() - startTime;

            // Validate output
            const validation = testCase.validate(result);

            results.cases.push({
                name: testCase.name,
                passed: validation.passed,
                duration_ms: duration,
                expected: testCase.expected,
                actual: result,
                errors: validation.errors || []
            });

            if (validation.passed) {
                results.passed++;
            } else {
                results.failed++;
            }

        } catch (error) {
            results.failed++;
            results.cases.push({
                name: testCase.name,
                passed: false,
                error: error.message
            });
        }
    }

    results.pass_rate = (results.passed / results.total_cases * 100).toFixed(2) + '%';

    return results;
}

// Test cases for single meal workflow
const singleMealTests = [
    {
        name: 'Cook4me meal generation',
        input: {
            user_id: 'test-user-1',
            cooking_method: 'cook4me',
            message: 'Generate chicken meal'
        },
        expected: {
            cooking_method: 'cook4me',
            calories_in_range: true
        },
        validate: (result) => {
            const meal = result.meal;
            return {
                passed: meal.cooking_method === 'cook4me' &&
                       meal.nutritional_summary.total_calories >= 2185 &&
                       meal.nutritional_summary.total_calories <= 2415,
                errors: []
            };
        }
    },
    {
        name: 'Traditional meal generation',
        input: {
            user_id: 'test-user-1',
            cooking_method: 'traditional',
            message: 'Generate grilled meal'
        },
        expected: {
            cooking_method: 'traditional',
            calories_in_range: true
        },
        validate: (result) => {
            const meal = result.meal;
            return {
                passed: meal.cooking_method === 'traditional' &&
                       meal.nutritional_summary.total_calories >= 2185 &&
                       meal.nutritional_summary.total_calories <= 2415,
                errors: []
            };
        }
    }
];

const workflowResults = await testWorkflow('Single Meal Generator', singleMealTests);

console.log('Workflow Test Results:', JSON.stringify(workflowResults, null, 2));

return [{ json: workflowResults }];
```

---

## Load & Performance Testing

### Concurrent User Simulation

```javascript
// Code Node: Load Test
async function loadTest(concurrentUsers, requestsPerUser) {
    const startTime = Date.now();

    const results = {
        concurrent_users: concurrentUsers,
        requests_per_user: requestsPerUser,
        total_requests: concurrentUsers * requestsPerUser,
        successful: 0,
        failed: 0,
        response_times: [],
        errors: []
    };

    // Create user pools
    const userPromises = [];

    for (let user = 0; user < concurrentUsers; user++) {
        const userRequests = [];

        for (let req = 0; req < requestsPerUser; req++) {
            userRequests.push(async () => {
                const reqStart = Date.now();

                try {
                    await simulateUserRequest(`test-user-${user}`);

                    const duration = Date.now() - reqStart;
                    results.response_times.push(duration);
                    results.successful++;

                } catch (error) {
                    results.failed++;
                    results.errors.push({
                        user: `test-user-${user}`,
                        error: error.message
                    });
                }
            });
        }

        // Execute all requests for this user
        userPromises.push(Promise.all(userRequests.map(fn => fn())));
    }

    // Execute all users concurrently
    await Promise.all(userPromises);

    const totalDuration = Date.now() - startTime;

    // Calculate statistics
    results.duration_ms = totalDuration;
    results.requests_per_second = (results.total_requests / (totalDuration / 1000)).toFixed(2);
    results.avg_response_time = (results.response_times.reduce((a, b) => a + b, 0) / results.response_times.length).toFixed(2);
    results.p95_response_time = percentile(results.response_times, 95);
    results.p99_response_time = percentile(results.response_times, 99);
    results.success_rate = (results.successful / results.total_requests * 100).toFixed(2) + '%';

    return results;
}

function percentile(arr, p) {
    const sorted = arr.slice().sort((a, b) => a - b);
    const index = Math.ceil(sorted.length * (p / 100)) - 1;
    return sorted[index];
}

// Run load test: 100 concurrent users, 10 requests each
const loadResults = await loadTest(100, 10);

console.log('Load Test Results:', JSON.stringify(loadResults, null, 2));

// Performance targets
const targets = {
    success_rate: 99,
    avg_response_time: 30000, // 30 seconds
    p95_response_time: 45000  // 45 seconds
};

const meetsTargets =
    parseFloat(loadResults.success_rate) >= targets.success_rate &&
    parseFloat(loadResults.avg_response_time) <= targets.avg_response_time &&
    loadResults.p95_response_time <= targets.p95_response_time;

if (!meetsTargets) {
    await sendAdminAlert('⚠️ Load test failed to meet performance targets');
}

return [{ json: { results: loadResults, meets_targets: meetsTargets } }];
```

### Performance Benchmarks

**Target Metrics:**
- Single meal generation: < 30s (p95)
- Weekly plan generation: < 90s (p95)
- Database queries: < 500ms
- Cache hits: < 50ms
- API calls: < 20s (p95)
- Overall uptime: 99%+

---

## Integration Testing

### Test API Integrations

```javascript
// Code Node: Test All APIs
async function testAllIntegrations() {
    const tests = {
        anthropic: testAnthropicAPI(),
        openai: testOpenAI(),
        supabase: testSupabase(),
        redis: testRedis(),
        usda: testUSDA(),
        telegram: testTelegram()
    };

    const results = {};

    for (const [name, testFn] of Object.entries(tests)) {
        try {
            const result = await testFn;
            results[name] = {
                status: 'ok',
                response_time: result.duration,
                details: result.details
            };
        } catch (error) {
            results[name] = {
                status: 'error',
                error: error.message
            };
        }
    }

    return results;
}

async function testAnthropicAPI() {
    const start = Date.now();

    const response = await callAnthropicAPI({
        model: "claude-sonnet-4-5",
        max_tokens: 50,
        messages: [{ role: "user", content: "Say 'test ok'" }]
    });

    return {
        duration: Date.now() - start,
        details: { model: response.model, tokens: response.usage.output_tokens }
    };
}

// Similar test functions for other APIs...

const integrationResults = await testAllIntegrations();

console.log('Integration Test Results:', JSON.stringify(integrationResults, null, 2));

const allPassed = Object.values(integrationResults).every(r => r.status === 'ok');

return [{ json: { results: integrationResults, all_passed: allPassed } }];
```

---

## User Acceptance Testing

### Beta Testing Checklist

**Phase 1: Alpha (5 users, 1 week)**
- [ ] All core features work
- [ ] No critical bugs
- [ ] Response times acceptable
- [ ] Meals are accurate
- [ ] User feedback collected

**Phase 2: Beta (50 users, 2 weeks)**
- [ ] Load testing passed
- [ ] Cost per user within budget
- [ ] 95%+ user satisfaction
- [ ] Error rate < 1%
- [ ] All feedback addressed

**Phase 3: Limited Launch (500 users, 1 month)**
- [ ] Monitoring dashboard active
- [ ] Auto-scaling working
- [ ] Cost optimization effective
- [ ] Support system in place

---

## Continuous Testing

### Automated Daily Tests

```javascript
// Cron Workflow: Run Daily at 3 AM
async function dailyHealthCheck() {
    const tests = {
        meal_generation: await testMealGeneration(10),
        nutritional_accuracy: await testNutritionalAccuracy(20),
        api_integrations: await testAllIntegrations(),
        performance: await testPerformance(),
        cost_efficiency: await checkCostEfficiency()
    };

    const report = generateDailyReport(tests);

    // Send to admin
    await sendDailyReport(report);

    // Log to database
    await $supabase.from('daily_test_reports').insert(report);

    return report;
}

await dailyHealthCheck();
```

---

## Next Steps

1. ✅ Implement test suite
2. 🧪 Run batch nutritional tests (100 meals)
3. 📊 Perform load testing
4. 🔍 Conduct beta testing
5. 🚀 Proceed to [Implementation Roadmap](./09-implementation-roadmap.md)

---

## Version History

- **v1.0** (2025-01-15): Initial testing & validation guide
