# Production Deployment Guide

## Overview
Step-by-step guide for deploying the AI Meal Planning Assistant to production with best practices for security, scalability, and reliability.

## Table of Contents
1. [Prerequisites](#prerequisites)
2. [Environment Setup](#environment-setup)
3. [Database Deployment](#database-deployment)
4. [n8n Deployment Options](#n8n-deployment-options)
5. [Telegram Bot Configuration](#telegram-bot-configuration)
6. [Monitoring & Logging](#monitoring--logging)
7. [Backup & Recovery](#backup--recovery)
8. [Scaling Strategy](#scaling-strategy)

---

## Prerequisites

### Required Accounts
- ✅ Supabase account (database)
- ✅ Upstash/Redis Cloud account (caching)
- ✅ Anthropic API key (AI)
- ✅ OpenAI API key (fallback AI)
- ✅ USDA API key (nutrition validation)
- ✅ Telegram Bot Token (chat interface)

### Tools Needed
```bash
# Install required CLI tools
npm install -g n8n
brew install postgresql  # For database management
brew install redis       # For local testing
```

---

## Environment Setup

### Production Environment Variables

Create `.env.production` file:

```bash
# =====================================================
# APPLICATION
# =====================================================
NODE_ENV=production
APP_VERSION=1.0.0
LOG_LEVEL=info

# =====================================================
# SUPABASE (Database)
# =====================================================
SUPABASE_URL=https://your-project.supabase.co
SUPABASE_API_KEY=your-anon-key-here
SUPABASE_SERVICE_ROLE_KEY=your-service-role-key-here

# =====================================================
# REDIS (Caching)
# =====================================================
# Option 1: Upstash
REDIS_URL=rediss://default:password@host:port

# Option 2: Redis Cloud
# REDIS_URL=redis://user:password@host:port/0

REDIS_TTL_DEFAULT=3600  # 1 hour default TTL

# =====================================================
# AI PROVIDERS
# =====================================================
# Anthropic Claude
ANTHROPIC_API_KEY=sk-ant-api03-...
ANTHROPIC_MODEL=claude-sonnet-4-5-20250120
ANTHROPIC_MAX_TOKENS=2500

# OpenAI (Fallback)
OPENAI_API_KEY=sk-...
OPENAI_MODEL=gpt-4o
OPENAI_MAX_TOKENS=2500

# =====================================================
# EXTERNAL APIs
# =====================================================
USDA_API_KEY=your-usda-key-here

# =====================================================
# TELEGRAM
# =====================================================
TELEGRAM_BOT_TOKEN=123456789:ABCdef...
TELEGRAM_WEBHOOK_SECRET=your-random-secret-32-chars-minimum

# =====================================================
# WEBHOOKS
# =====================================================
N8N_WEBHOOK_URL=https://your-n8n-instance.com

# =====================================================
# MONITORING
# =====================================================
# Optional: Sentry for error tracking
SENTRY_DSN=https://...@sentry.io/...

# Optional: Slack for admin alerts
SLACK_WEBHOOK_URL=https://hooks.slack.com/services/...

# =====================================================
# COST LIMITS
# =====================================================
DAILY_COST_BUDGET=50.00
MONTHLY_COST_BUDGET=1000.00

# =====================================================
# RATE LIMITING
# =====================================================
RATE_LIMIT_REQUESTS_PER_HOUR=20
RATE_LIMIT_REQUESTS_PER_DAY=100

# =====================================================
# SECURITY
# =====================================================
ADMIN_TELEGRAM_IDS=123456789,987654321  # Comma-separated admin user IDs
```

### Secure Environment Variables

**DO NOT** commit `.env` files to git!

```bash
# Add to .gitignore
echo ".env*" >> .gitignore
echo "!.env.example" >> .gitignore
```

**Best Practice:** Use secrets management:
- **n8n Cloud:** Built-in environment variables
- **Docker:** Docker secrets or environment files
- **Kubernetes:** Kubernetes secrets
- **Other:** HashiCorp Vault, AWS Secrets Manager

---

## Database Deployment

### Step 1: Create Supabase Project

1. Go to https://supabase.com
2. Click "New Project"
3. Fill in:
   - Name: `omad-meal-planner-prod`
   - Database Password: (generate strong password)
   - Region: (closest to your users)
4. Wait for provisioning (~2 minutes)

### Step 2: Run Database Migrations

```bash
# Save connection string
export DATABASE_URL="postgresql://postgres:[password]@db.[project].supabase.co:5432/postgres"

# Run migrations
psql $DATABASE_URL -f plan/sql/01_schema.sql
psql $DATABASE_URL -f plan/sql/02_functions.sql
psql $DATABASE_URL -f plan/sql/03_rls_policies.sql
psql $DATABASE_URL -f plan/sql/04_indexes.sql

# Verify
psql $DATABASE_URL -c "\dt"
```

### Step 3: Configure Row Level Security

```sql
-- Verify RLS is enabled
SELECT tablename, rowsecurity
FROM pg_tables
WHERE schemaname = 'public';

-- All tables should have rowsecurity = true
```

### Step 4: Create Database Functions

Already included in migration files, but verify:

```sql
-- Test get_or_create_user function
SELECT * FROM get_or_create_user(123456789, 'test_user', 'Test', 'User');
```

### Step 5: Set Up Database Backups

**Supabase Pro+:** Automatic daily backups
**Free Tier:** Manual backups

```bash
# Manual backup
pg_dump $DATABASE_URL > backup_$(date +%Y%m%d).sql

# Automated backup (cron job)
crontab -e
# Add:
0 3 * * * pg_dump $DATABASE_URL > /backups/backup_$(date +\%Y\%m\%d).sql
```

---

## n8n Deployment Options

### Option 1: n8n Cloud (Recommended)

**Pros:**
- Managed hosting
- Auto-scaling
- Built-in monitoring
- Easy setup

**Cons:**
- Monthly cost ($50-200/month)

**Setup:**
1. Go to https://n8n.io/cloud
2. Sign up for account
3. Create new instance
4. Import workflows
5. Configure environment variables

### Option 2: Self-Hosted (Docker)

**Pros:**
- Full control
- Lower cost (server only)

**Cons:**
- Manual management
- Requires devops knowledge

**Setup:**

```bash
# Create docker-compose.yml
cat > docker-compose.yml <<EOF
version: '3.8'

services:
  n8n:
    image: n8nio/n8n:latest
    restart: unless-stopped
    ports:
      - "5678:5678"
    environment:
      - N8N_HOST=your-domain.com
      - N8N_PORT=5678
      - N8N_PROTOCOL=https
      - NODE_ENV=production
      - WEBHOOK_URL=https://your-domain.com/
      - GENERIC_TIMEZONE=America/New_York
    volumes:
      - n8n_data:/home/node/.n8n
      - ./env-production.env:/home/node/.env
    networks:
      - n8n-network

  nginx:
    image: nginx:alpine
    restart: unless-stopped
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf
      - ./ssl:/etc/nginx/ssl
    depends_on:
      - n8n
    networks:
      - n8n-network

volumes:
  n8n_data:

networks:
  n8n-network:
    driver: bridge
EOF

# Start services
docker-compose up -d

# Check logs
docker-compose logs -f n8n
```

**Nginx Configuration:**

```nginx
# nginx.conf
events {
    worker_connections 1024;
}

http {
    server {
        listen 80;
        server_name your-domain.com;

        # Redirect to HTTPS
        return 301 https://$server_name$request_uri;
    }

    server {
        listen 443 ssl http2;
        server_name your-domain.com;

        ssl_certificate /etc/nginx/ssl/cert.pem;
        ssl_certificate_key /etc/nginx/ssl/key.pem;

        location / {
            proxy_pass http://n8n:5678;
            proxy_http_version 1.1;
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection 'upgrade';
            proxy_set_header Host $host;
            proxy_cache_bypass $http_upgrade;
        }
    }
}
```

### Option 3: Railway / Render / Fly.io

**Pros:**
- Easy deployment
- Automatic HTTPS
- Good free tiers

**Example: Railway**

```bash
# Install Railway CLI
npm i -g @railway/cli

# Login
railway login

# Create new project
railway init

# Add environment variables
railway variables set SUPABASE_URL=...
railway variables set ANTHROPIC_API_KEY=...
# ... add all variables

# Deploy
railway up
```

---

## Telegram Bot Configuration

### Production Webhook Setup

```bash
# Set webhook with secret token
curl -X POST "https://api.telegram.org/bot${TELEGRAM_BOT_TOKEN}/setWebhook" \
  -H "Content-Type: application/json" \
  -d '{
    "url": "https://your-n8n-instance.com/webhook/telegram-omad-bot",
    "secret_token": "'"${TELEGRAM_WEBHOOK_SECRET}"'",
    "max_connections": 40,
    "allowed_updates": ["message", "callback_query"],
    "drop_pending_updates": false
  }'

# Verify webhook
curl "https://api.telegram.org/bot${TELEGRAM_BOT_TOKEN}/getWebhookInfo"
```

**Expected Response:**
```json
{
  "ok": true,
  "result": {
    "url": "https://your-n8n-instance.com/webhook/telegram-omad-bot",
    "has_custom_certificate": false,
    "pending_update_count": 0,
    "max_connections": 40,
    "allowed_updates": ["message", "callback_query"]
  }
}
```

### Bot Commands Setup

```bash
# Set commands for autocomplete
curl -X POST "https://api.telegram.org/bot${TELEGRAM_BOT_TOKEN}/setMyCommands" \
  -H "Content-Type: application/json" \
  -d '{
    "commands": [
      {"command": "start", "description": "Start using the meal planner"},
      {"command": "meal", "description": "Generate a single meal"},
      {"command": "week", "description": "Create 7-day meal plan"},
      {"command": "grocery", "description": "View grocery list"},
      {"command": "history", "description": "View meal history"},
      {"command": "settings", "description": "Update preferences"},
      {"command": "help", "description": "Show help"}
    ]
  }'
```

---

## Monitoring & Logging

### Supabase Monitoring

Enable logging in Supabase dashboard:
1. Go to Project Settings → Logging
2. Enable query logs
3. Set retention period (7 days minimum)

### n8n Monitoring

**Error Workflow:**
Create separate "Error Handler" workflow that triggers on any error:

```json
{
  "nodes": [
    {
      "name": "Error Trigger",
      "type": "n8n-nodes-base.errorTrigger"
    },
    {
      "name": "Log to Database",
      "type": "n8n-nodes-base.httpRequest"
    },
    {
      "name": "Send Admin Alert",
      "type": "n8n-nodes-base.telegram"
    }
  ]
}
```

### Cost Monitoring

**Daily Cost Report Workflow (Cron: Daily at 9 AM):**

```javascript
// Code Node: Generate Daily Report
const report = await getDailyCostReport();

// Send to admins via Telegram
const message = `
📊 *Daily Cost Report*

*Total*: $${report.total.toFixed(2)}
*Anthropic*: $${report.anthropic.cost.toFixed(2)}
*OpenAI*: $${report.openai.cost.toFixed(2)}

*Requests*: ${report.total_requests}
*Cache Hit Rate*: ${report.cache_hit_rate}%

*Budget*: $${process.env.DAILY_COST_BUDGET}
*Remaining*: $${(process.env.DAILY_COST_BUDGET - report.total).toFixed(2)}
`;

return [{ json: { message } }];
```

### Health Check Endpoint

Create "Health Check" workflow (HTTP Webhook):

```javascript
// Code Node: Health Check
const health = {
    status: 'ok',
    timestamp: new Date().toISOString(),
    version: process.env.APP_VERSION,
    checks: {
        database: await checkDatabase(),
        redis: await checkRedis(),
        anthropic: await checkAnthropic(),
        telegram: await checkTelegram()
    }
};

const allHealthy = Object.values(health.checks).every(c => c.status === 'ok');
health.status = allHealthy ? 'ok' : 'degraded';

return [{ json: health }];
```

**Monitor with UptimeRobot or similar:**
- URL: `https://your-n8n-instance.com/webhook/health`
- Interval: 5 minutes
- Alert on status !== 'ok'

---

## Backup & Recovery

### Database Backups

**Automated Backup Script:**

```bash
#!/bin/bash
# backup-database.sh

DATE=$(date +%Y%m%d_%H%M%S)
BACKUP_DIR="/backups/database"
BACKUP_FILE="$BACKUP_DIR/omad_backup_$DATE.sql"

mkdir -p $BACKUP_DIR

# Backup
pg_dump $DATABASE_URL > $BACKUP_FILE

# Compress
gzip $BACKUP_FILE

# Upload to S3 (optional)
aws s3 cp "$BACKUP_FILE.gz" s3://your-bucket/backups/

# Delete old backups (keep last 30 days)
find $BACKUP_DIR -name "*.sql.gz" -mtime +30 -delete

echo "Backup completed: $BACKUP_FILE.gz"
```

**Cron Job:**
```bash
# Daily at 3 AM
0 3 * * * /path/to/backup-database.sh >> /var/log/backup.log 2>&1
```

### n8n Workflow Backups

```bash
# Export all workflows
n8n export:workflow --all --output=/backups/workflows/

# Backup to git
cd /backups/workflows
git init
git add .
git commit -m "Workflow backup $(date +%Y%m%d)"
git push origin main
```

### Disaster Recovery Plan

**RTO (Recovery Time Objective):** 1 hour
**RPO (Recovery Point Objective):** 24 hours

**Recovery Steps:**
1. Restore Supabase from backup
2. Deploy fresh n8n instance
3. Import workflows from git
4. Configure environment variables
5. Test health check
6. Resume Telegram webhook

---

## Scaling Strategy

### Phase 1: 0-100 Users
**Infrastructure:**
- Single n8n instance
- Supabase Free Tier
- Upstash Free Tier (Redis)

**Estimated Cost:** $0-20/month

### Phase 2: 100-1,000 Users
**Infrastructure:**
- Single n8n instance (or n8n Cloud starter)
- Supabase Pro ($25/month)
- Upstash Pro ($10/month)

**Estimated Cost:** $35-100/month

### Phase 3: 1,000-10,000 Users
**Infrastructure:**
- n8n Cloud with auto-scaling
- Supabase Team ($599/month)
- Redis Cloud ($50/month)

**Estimated Cost:** $700-1,500/month

### Scaling Checklist

**Database Scaling:**
- [ ] Enable connection pooling (PgBouncer)
- [ ] Add read replicas for queries
- [ ] Partition large tables (meal_history)
- [ ] Optimize indexes

**Application Scaling:**
- [ ] Enable n8n auto-scaling
- [ ] Implement request queuing
- [ ] Add CDN for static content
- [ ] Load balance multiple instances

**Caching Scaling:**
- [ ] Increase Redis memory
- [ ] Add Redis replicas
- [ ] Implement cache warming
- [ ] Optimize cache keys

---

## Security Checklist

### Pre-Launch Security Audit

- [ ] All API keys in environment variables (not code)
- [ ] Database RLS policies active
- [ ] HTTPS enabled for all endpoints
- [ ] Webhook secret tokens configured
- [ ] Rate limiting implemented
- [ ] Input validation on all user inputs
- [ ] SQL injection protection (parameterized queries)
- [ ] XSS protection (sanitize outputs)
- [ ] CORS configured correctly
- [ ] Admin endpoints restricted to admin IDs
- [ ] Error messages don't leak sensitive data
- [ ] Logs don't contain secrets
- [ ] Regular security updates scheduled

---

## Post-Deployment Checklist

### Day 1: Launch Day
- [ ] Deploy to production
- [ ] Verify webhook working
- [ ] Test all commands
- [ ] Monitor error logs
- [ ] Check cost dashboard

### Week 1: Stabilization
- [ ] Review all error logs
- [ ] Optimize slow queries
- [ ] Adjust cache TTLs
- [ ] Collect user feedback
- [ ] Fix critical bugs

### Week 2-4: Optimization
- [ ] Analyze cost reports
- [ ] Improve cache hit rates
- [ ] A/B test prompts
- [ ] Add requested features
- [ ] Scale infrastructure if needed

---

## Troubleshooting

### Common Issues

**Issue:** Webhook not receiving messages
```bash
# Check webhook status
curl "https://api.telegram.org/bot${TELEGRAM_BOT_TOKEN}/getWebhookInfo"

# Delete and re-register
curl -X POST "https://api.telegram.org/bot${TELEGRAM_BOT_TOKEN}/deleteWebhook"
# Then re-run setWebhook command
```

**Issue:** Database connection errors
```bash
# Check connection
psql $DATABASE_URL -c "SELECT 1"

# Check connection limit
psql $DATABASE_URL -c "SHOW max_connections"

# Enable connection pooling in Supabase
```

**Issue:** High costs
```bash
# Check today's usage
node scripts/check-costs.js

# Review cache hit rates
# Reduce max_tokens if possible
# Switch to cheaper models for simple tasks
```

---

## Next Steps

1. ✅ Complete this deployment checklist
2. 📊 Set up monitoring dashboards
3. 🧪 Run final production tests
4. 🚀 Launch to users!
5. 📈 Monitor and iterate

---

## Version History

- **v1.0** (2025-01-15): Initial deployment guide
