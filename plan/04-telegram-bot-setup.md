# Telegram Bot Setup & Integration Guide

## Overview
Complete guide for creating and configuring a Telegram bot integrated with n8n workflows for the AI Meal Planning Assistant.

## Table of Contents
1. [Bot Creation with BotFather](#bot-creation-with-botfather)
2. [n8n Telegram Integration](#n8n-telegram-integration)
3. [Message Handling Patterns](#message-handling-patterns)
4. [Interactive Features](#interactive-features)
5. [Advanced Features](#advanced-features)
6. [Security & Best Practices](#security--best-practices)
7. [Testing & Debugging](#testing--debugging)

---

## Bot Creation with BotFather

### Step 1: Create Bot (10 minutes)

1. Open Telegram and search for `@BotFather`
2. Start conversation with `/start`
3. Create new bot with `/newbot`
4. Provide bot name: `OMAD Meal Planner` (display name)
5. Provide username: `omad_meal_planner_bot` (must end in 'bot')
6. **Save the API token**: `1234567890:ABCdefGHIjklMNOpqrsTUVwxyz` (example)

### Step 2: Configure Bot Settings

```bash
# Set bot description (shown in chat header)
/setdescription
Your AI-powered OMAD meal planning assistant. Get personalized meals, weekly plans, and grocery lists.

# Set about text (shown in profile)
/setabouttext
AI Meal Planning Assistant
🍽️ Custom OMAD meals
📅 Weekly meal plans
🛒 Auto grocery lists
⚖️ Nutrition tracking

# Set bot profile photo
/setuserpic
[Upload 512x512 image]

# Set commands for autocomplete
/setcommands

start - Start using the meal planner
meal - Generate a single meal
week - Create 7-day meal plan
grocery - View/generate grocery list
history - View meal history
settings - Update your preferences
help - Show help & commands
```

### Step 3: Enable Inline Mode (Optional)

```bash
/setinline
@omad_meal_planner_bot

/setinlinefeedback
100  # Request feedback after 100 inline queries
```

---

## n8n Telegram Integration

### Webhook Setup

#### Option 1: Telegram Webhook Trigger (Recommended)

```json
{
  "nodes": [
    {
      "name": "Telegram Webhook",
      "type": "n8n-nodes-base.telegramTrigger",
      "typeVersion": 1,
      "position": [250, 300],
      "webhookId": "auto-generated-uuid",
      "parameters": {
        "authentication": "telegramApi",
        "updates": [
          "message",
          "callback_query",
          "inline_query"
        ]
      },
      "credentials": {
        "telegramApi": {
          "id": "1",
          "name": "Telegram OMAD Bot"
        }
      }
    }
  ]
}
```

**Setup Steps**:
1. Add Telegram Trigger node to workflow
2. Create Telegram API credential:
   - Name: `Telegram OMAD Bot`
   - Access Token: `[your bot token]`
3. Activate workflow (auto-registers webhook)
4. Test by sending message to bot

#### Option 2: Webhook Node + Manual Registration

```json
{
  "name": "Webhook",
  "type": "n8n-nodes-base.webhook",
  "parameters": {
    "path": "telegram-omad-bot",
    "responseMode": "responseNode",
    "responseData": "allEntries",
    "options": {}
  }
}
```

**Manual webhook registration**:
```bash
# Set webhook URL
curl -X POST "https://api.telegram.org/bot<TOKEN>/setWebhook" \
  -H "Content-Type: application/json" \
  -d '{
    "url": "https://your-n8n.com/webhook/telegram-omad-bot",
    "max_connections": 100,
    "allowed_updates": ["message", "callback_query"]
  }'

# Verify webhook
curl "https://api.telegram.org/bot<TOKEN>/getWebhookInfo"
```

### Credentials Configuration

**In n8n:**
1. Go to Credentials → New Credential
2. Select "Telegram API"
3. Fill in:
   - **Credential Name**: Telegram OMAD Bot
   - **Access Token**: [your bot token from BotFather]
4. Test connection
5. Save

---

## Message Handling Patterns

### Extract User Data (Code Node)

```javascript
// Node: Extract Message Data
const input = $input.first().json;
const message = input.message || input.callback_query?.message;
const from = input.message?.from || input.callback_query?.from;

// Handle different update types
let messageText = '';
let callbackData = null;

if (input.message) {
    messageText = input.message.text || input.message.caption || '';
} else if (input.callback_query) {
    messageText = input.callback_query.message.text || '';
    callbackData = input.callback_query.data;
}

// Extract user info
const userData = {
    telegram_user_id: from.id,
    telegram_username: from.username || null,
    telegram_first_name: from.first_name || null,
    telegram_last_name: from.last_name || null,
    chat_id: message.chat.id,
    message_id: message.message_id,

    // Message details
    message_text: messageText,
    callback_data: callbackData,
    is_command: messageText.startsWith('/'),
    command: messageText.startsWith('/') ? messageText.split(' ')[0].substring(1) : null,
    command_args: messageText.startsWith('/') ? messageText.split(' ').slice(1).join(' ') : null,

    // Message type
    message_type: input.message ? 'message' : 'callback_query',

    // Timestamp
    timestamp: new Date().toISOString()
};

return [{ json: userData }];
```

### Command Router (Switch Node)

```json
{
  "name": "Command Router",
  "type": "n8n-nodes-base.switch",
  "parameters": {
    "mode": "rules",
    "rules": {
      "rules": [
        {
          "operation": "equal",
          "value1": "={{$json.command}}",
          "value2": "start",
          "output": 0
        },
        {
          "operation": "equal",
          "value1": "={{$json.command}}",
          "value2": "meal",
          "output": 1
        },
        {
          "operation": "equal",
          "value1": "={{$json.command}}",
          "value2": "week",
          "output": 2
        },
        {
          "operation": "equal",
          "value1": "={{$json.command}}",
          "value2": "grocery",
          "output": 3
        },
        {
          "operation": "equal",
          "value1": "={{$json.command}}",
          "value2": "settings",
          "output": 4
        },
        {
          "operation": "equal",
          "value1": "={{$json.command}}",
          "value2": "help",
          "output": 5
        }
      ]
    },
    "fallbackOutput": 6
  }
}
```

---

## Interactive Features

### 1. Inline Keyboards (Buttons)

```javascript
// Code Node: Create Inline Keyboard
function createMealTypeKeyboard() {
    return {
        inline_keyboard: [
            [
                { text: "🍲 Cook4me", callback_data: "method:cook4me" },
                { text: "👨‍🍳 Traditional", callback_data: "method:traditional" }
            ],
            [
                { text: "📅 Weekly Plan", callback_data: "action:weekly_plan" }
            ],
            [
                { text: "⚙️ Settings", callback_data: "action:settings" }
            ]
        ]
    };
}

return [{
    json: {
        chat_id: $json.chat_id,
        text: "How would you like your meal prepared?",
        reply_markup: JSON.stringify(createMealTypeKeyboard())
    }
}];
```

**Send with Telegram Node:**
```json
{
  "operation": "sendMessage",
  "chatId": "={{$json.chat_id}}",
  "text": "={{$json.text}}",
  "additionalFields": {
    "reply_markup": "={{$json.reply_markup}}"
  }
}
```

### 2. Typing Indicator

```javascript
// Code Node: Show Typing
return [{
    json: {
        method: "sendChatAction",
        chat_id: $json.chat_id,
        action: "typing"
    }
}];
```

**HTTP Request Node:**
```json
{
  "method": "POST",
  "url": "=https://api.telegram.org/bot{{$credentials.telegramApi.accessToken}}/sendChatAction",
  "bodyParameters": {
    "parameters": [
      {
        "name": "chat_id",
        "value": "={{$json.chat_id}}"
      },
      {
        "name": "action",
        "value": "typing"
      }
    ]
  }
}
```

### 3. Formatted Messages (MarkdownV2)

```javascript
// Code Node: Format Recipe Message
function formatRecipeMessage(recipe) {
    // MarkdownV2 requires escaping special characters
    function escape(text) {
        return text.replace(/[_*[\]()~`>#+=|{}.!-]/g, '\\$&');
    }

    const message = `
🍽️ *${escape(recipe.meal_name)}*

${escape(recipe.description)}

⏱️ *Prep*: ${recipe.prep_time_minutes} min \\| *Cook*: ${recipe.cook_time_minutes} min

📊 *Nutrition*:
• Calories: ${recipe.nutritional_summary.total_calories} kcal
• Protein: ${recipe.nutritional_summary.total_protein_g}g
• Carbs: ${recipe.nutritional_summary.total_carbs_g}g
• Fat: ${recipe.nutritional_summary.total_fat_g}g

👨‍🍳 *${recipe.cooking_method === 'cook4me' ? 'Cook4me' : 'Traditional'}* cooking method

_Tap "View Recipe" for full instructions_
`.trim();

    return message;
}

return [{
    json: {
        chat_id: $json.chat_id,
        text: formatRecipeMessage($json.recipe),
        parse_mode: "MarkdownV2",
        reply_markup: JSON.stringify({
            inline_keyboard: [
                [{ text: "📖 View Full Recipe", callback_data: `recipe:${$json.recipe.id}` }],
                [
                    { text: "⭐ Rate", callback_data: `rate:${$json.recipe.id}` },
                    { text: "🔄 New Meal", callback_data: "action:new_meal" }
                ]
            ]
        })
    }
}];
```

### 4. Progress Messages

```javascript
// Code Node: Update Message with Progress
async function updateProgress(chatId, messageId, progress, total, task) {
    const percentage = Math.round((progress / total) * 100);
    const progressBar = '█'.repeat(Math.floor(percentage / 10)) +
                       '░'.repeat(10 - Math.floor(percentage / 10));

    return {
        method: "editMessageText",
        chat_id: chatId,
        message_id: messageId,
        text: `⏳ ${task}\n\n${progressBar} ${percentage}%\n\n${progress}/${total} meals generated`,
        parse_mode: "Markdown"
    };
}

// Usage
return [{
    json: await updateProgress($json.chat_id, $json.message_id, 3, 7, "Generating weekly meal plan")
}];
```

---

## Advanced Features

### 1. Conversation State Management

```javascript
// Code Node: Get Conversation Context
async function getConversationContext(userId) {
    // Fetch from Redis or Supabase
    const context = await $redis.get(`conversation:${userId}`);

    if (context) {
        return JSON.parse(context);
    }

    // Initialize new context
    return {
        state: 'idle',
        last_intent: null,
        pending_action: null,
        data: {}
    };
}

// Code Node: Update Conversation Context
async function updateContext(userId, updates) {
    const context = await getConversationContext(userId);
    const updated = { ...context, ...updates, updated_at: new Date().toISOString() };

    await $redis.setex(
        `conversation:${userId}`,
        86400, // 24 hour expiry
        JSON.stringify(updated)
    );

    return updated;
}

// Example: Multi-step preferences update
const context = await getConversationContext($json.telegram_user_id);

if (context.state === 'awaiting_calorie_target') {
    // User just sent calorie target
    await updateContext($json.telegram_user_id, {
        state: 'awaiting_protein_target',
        data: { ...context.data, calories: parseInt($json.message_text) }
    });

    return [{
        json: {
            text: "Great! Now, what's your daily protein target in grams? (e.g., 130)"
        }
    }];
}
```

### 2. File Uploads (Recipe Images)

```javascript
// Handle photo uploads
if ($input.first().json.message?.photo) {
    const photos = $input.first().json.message.photo;
    const largestPhoto = photos[photos.length - 1]; // Highest resolution

    return [{
        json: {
            file_id: largestPhoto.file_id,
            file_unique_id: largestPhoto.file_unique_id,
            user_message: "User uploaded meal photo",
            action: "save_meal_photo"
        }
    }];
}

// Download photo
// HTTP Request Node:
{
    "method": "GET",
    "url": "=https://api.telegram.org/bot{{$credentials.telegramApi.accessToken}}/getFile?file_id={{$json.file_id}}",
    "responseFormat": "json"
}

// Then download file:
{
    "method": "GET",
    "url": "=https://api.telegram.org/file/bot{{$credentials.telegramApi.accessToken}}/{{$json.result.file_path}}",
    "responseFormat": "file"
}
```

### 3. Scheduled Messages

```javascript
// Code Node: Schedule Daily Meal Reminder
async function scheduleMealReminder(userId, preferredTime) {
    // Store in database for cron processing
    await $supabase.from('scheduled_messages').insert({
        user_id: userId,
        message_type: 'meal_reminder',
        scheduled_time: preferredTime,
        message_data: {
            text: "🍽️ Time to start preparing your OMAD! Would you like today's meal suggestion?"
        },
        is_active: true
    });
}

// Separate workflow triggered by cron
// Cron Trigger: 0 */1 * * * (every hour)
// Check for due messages and send
```

---

## Security & Best Practices

### 1. Input Validation

```javascript
// Code Node: Sanitize User Input
function sanitizeInput(text) {
    // Remove potentially dangerous content
    const cleaned = text
        .replace(/<script[^>]*>.*?<\/script>/gi, '')
        .replace(/javascript:/gi, '')
        .replace(/on\w+\s*=/gi, '')
        .trim();

    // Limit length
    if (cleaned.length > 4000) {
        return cleaned.substring(0, 4000);
    }

    return cleaned;
}

const sanitized = sanitizeInput($json.message_text);
```

### 2. Rate Limiting

```javascript
// Code Node: Check Rate Limit
async function checkRateLimit(userId) {
    const key = `rate_limit:${userId}`;
    const limit = 20; // 20 messages per hour
    const window = 3600; // 1 hour

    const current = await $redis.incr(key);

    if (current === 1) {
        await $redis.expire(key, window);
    }

    if (current > limit) {
        return {
            limited: true,
            retry_after: await $redis.ttl(key),
            message: "⚠️ Too many requests. Please wait before sending more messages."
        };
    }

    return { limited: false };
}

const rateCheck = await checkRateLimit($json.telegram_user_id);

if (rateCheck.limited) {
    return [{
        json: {
            chat_id: $json.chat_id,
            text: rateCheck.message
        }
    }];
}
```

### 3. Error Handling

```javascript
// Code Node: User-Friendly Error Messages
function getUserFriendlyError(error) {
    const errorMap = {
        'RATE_LIMITED': '⏱️ High demand right now. Please try again in a moment.',
        'AI_SERVICE_DOWN': '🔧 Our AI service is temporarily unavailable. Switching to backup...',
        'DATABASE_ERROR': '💾 Database issue. Your request is saved and will process shortly.',
        'INVALID_INPUT': '❌ I couldn\'t understand that. Try /help for available commands.',
        'TIMEOUT': '⏳ Request took too long. Trying with a simpler approach...'
    };

    return errorMap[error.code] || '❌ Something went wrong. Our team has been notified.';
}

// Usage in error catch
try {
    // ... workflow logic
} catch (error) {
    const friendlyMessage = getUserFriendlyError(error);

    return [{
        json: {
            chat_id: $json.chat_id,
            text: friendlyMessage
        }
    }];
}
```

### 4. Webhook Security

```bash
# Set webhook with secret token (Telegram validates requests)
curl -X POST "https://api.telegram.org/bot<TOKEN>/setWebhook" \
  -H "Content-Type: application/json" \
  -d '{
    "url": "https://your-n8n.com/webhook/telegram-omad-bot",
    "secret_token": "your-random-secret-token-min-32-chars",
    "max_connections": 40,
    "drop_pending_updates": false
  }'
```

```javascript
// Code Node: Validate Webhook Secret
const receivedToken = $input.first().headers['x-telegram-bot-api-secret-token'];
const expectedToken = $env.TELEGRAM_WEBHOOK_SECRET;

if (receivedToken !== expectedToken) {
    throw new Error('Invalid webhook secret token');
}

// Continue processing...
```

---

## Testing & Debugging

### 1. Test Bot Locally

```bash
# Use ngrok for local testing
ngrok http 5678  # n8n default port

# Set webhook to ngrok URL
curl -X POST "https://api.telegram.org/bot<TOKEN>/setWebhook" \
  -d "url=https://your-ngrok-url.ngrok.io/webhook/telegram-omad-bot"
```

### 2. Debug Commands

```javascript
// Add /debug command for testing (only for admin users)
const ADMIN_USER_IDS = [123456789]; // Your Telegram user ID

if ($json.command === 'debug' && ADMIN_USER_IDS.includes($json.telegram_user_id)) {
    const debugInfo = {
        workflow_execution_id: $workflow.id,
        user_data: $json,
        environment: $env.NODE_ENV,
        timestamp: new Date().toISOString(),
        active_workflows: await $workflows.getAll()
    };

    return [{
        json: {
            chat_id: $json.chat_id,
            text: `\`\`\`json\n${JSON.stringify(debugInfo, null, 2)}\n\`\`\``,
            parse_mode: "Markdown"
        }
    }];
}
```

### 3. Logging

```javascript
// Code Node: Structured Logging
function logTelegramInteraction(data) {
    const logEntry = {
        timestamp: new Date().toISOString(),
        level: 'info',
        event: 'telegram_interaction',
        user_id: data.telegram_user_id,
        chat_id: data.chat_id,
        message_type: data.message_type,
        command: data.command,
        workflow_id: $workflow.id,
        execution_id: $execution.id
    };

    console.log(JSON.stringify(logEntry));

    // Also save to database
    $supabase.from('interaction_logs').insert(logEntry);
}
```

---

## Best Practices Checklist

- ✅ Always acknowledge user messages (even if processing takes time)
- ✅ Use typing indicators for operations > 2 seconds
- ✅ Implement rate limiting (prevent spam)
- ✅ Sanitize all user inputs
- ✅ Provide helpful error messages
- ✅ Use inline keyboards for common actions
- ✅ Keep messages concise (< 4096 chars)
- ✅ Handle callback queries (answer within 30 seconds)
- ✅ Implement conversation state for multi-step flows
- ✅ Log all interactions for debugging
- ✅ Test with multiple users concurrently
- ✅ Monitor webhook health

---

## Next Steps

1. ✅ Create bot with BotFather
2. 🔗 Configure n8n Telegram credentials
3. 🧪 Test message handling
4. 🎨 Design inline keyboard flows
5. 🚀 Proceed to [API Integrations](./05-api-integrations.md)

---

## Version History

- **v1.0** (2025-01-15): Initial Telegram bot setup guide
- **v1.1** (Pending): Add inline query support
- **v1.2** (Pending): Add payment integration for premium features
