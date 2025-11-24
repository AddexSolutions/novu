# Novu Railway Deployment Guide

This guide will walk you through deploying Novu on Railway with a working UI.

## Architecture Overview

Novu requires these services on Railway:
- **MongoDB** (database plugin)
- **Redis** (cache plugin)
- **API** (backend REST API)
- **Worker** (background job processor)
- **WebSocket** (real-time updates)
- **Dashboard** (frontend UI)

## Step 1: Create a New Railway Project

1. Go to [railway.app](https://railway.app) and sign in
2. Click **"New Project"**
3. Select **"Empty Project"**
4. Name it **"novu"**

## Step 2: Add Database Services

### Add MongoDB

1. Click **"+ New"** → **"Database"** → **"Add MongoDB"**
2. Railway will automatically provision MongoDB
3. Note the connection string (available in the MongoDB service variables as `MONGO_URL`)

### Add Redis

1. Click **"+ New"** → **"Database"** → **"Add Redis"**
2. Railway will automatically provision Redis
3. Note the connection details (available as `REDIS_URL`)

## Step 3: Deploy Backend Services

### 3.1 Deploy API Service

1. Click **"+ New"** → **"GitHub Repo"** (or **"Empty Service"** if deploying manually)
2. Select your Novu repository
3. Name the service: **"novu-api"**
4. In **Settings**:
   - **Root Directory**: Leave as `/` (monorepo root)
   - **Dockerfile Path**: `apps/api/Dockerfile`
   - **Custom Start Command**: `pm2-runtime start dist/main.js -i max`
   - Or link to `railway-api.json`: Set **"Config File Path"** to `railway-api.json`

5. In **Variables** tab, add these environment variables:

```bash
# === REQUIRED SECRETS (GENERATE THESE) ===
JWT_SECRET=<generate-random-64-char-string>
STORE_ENCRYPTION_KEY=<generate-random-32-char-string>
NOVU_SECRET_KEY=<generate-random-secret>

# === NODE & APP CONFIG ===
NODE_ENV=production
HOST_NAME=https://${{RAILWAY_PUBLIC_DOMAIN}}
API_PORT=3000
PORT=3000

# === DATABASE (from MongoDB plugin) ===
# IMPORTANT: Use MONGO_PRIVATE_URL to avoid egress fees!
# MONGO_URL connects over public internet (costs money)
# MONGO_PRIVATE_URL connects over Railway's private network (FREE)
MONGO_URL=${{MongoDB.MONGO_PRIVATE_URL}}
MONGO_MIN_POOL_SIZE=50
MONGO_MAX_POOL_SIZE=500

# === REDIS (from Redis plugin) ===
REDIS_HOST=${{Redis.REDIS_HOST}}
REDIS_PORT=${{Redis.REDIS_PORT}}
REDIS_PASSWORD=${{Redis.REDIS_PASSWORD}}
REDIS_DB_INDEX=2
REDIS_CACHE_SERVICE_HOST=${{Redis.REDIS_HOST}}
REDIS_CACHE_SERVICE_PORT=${{Redis.REDIS_PORT}}

# === URLS (will configure after dashboard deployment) ===
API_ROOT_URL=https://${{RAILWAY_PUBLIC_DOMAIN}}
FRONT_BASE_URL=<DASHBOARD_URL_HERE>
WIDGET_EMBED_PATH=https://web.novu.co/embed.umd.min.js
WIDGET_URL=https://web.novu.co

# === S3 STORAGE (use Railway volumes or external S3) ===
S3_BUCKET_NAME=novu-storage
S3_REGION=us-east-1
AWS_ACCESS_KEY_ID=<your-aws-key>
AWS_SECRET_ACCESS_KEY=<your-aws-secret>

# === WEBSOCKET (will configure after WS deployment) ===
REACT_APP_WS_URL=<WS_SERVICE_URL_HERE>

# === FEATURE FLAGS ===
DISABLE_USER_REGISTRATION=false
IS_API_IDEMPOTENCY_ENABLED=true
IS_API_RATE_LIMITING_ENABLED=true
IS_NEW_MESSAGES_API_RESPONSE_ENABLED=true
IS_V2_ENABLED=true

# === MONITORING (optional) ===
SENTRY_DSN=
NEW_RELIC_ENABLED=false
```

6. Click **"Deploy"**
7. Once deployed, go to **Settings** → **Networking** → **Generate Domain** to get public URL

### 3.2 Deploy Worker Service

1. Click **"+ New"** → **"GitHub Repo"**
2. Select your Novu repository
3. Name the service: **"novu-worker"**
4. In **Settings**:
   - **Dockerfile Path**: `apps/worker/Dockerfile`
   - Or **Config File Path**: `railway-worker.json`

5. In **Variables** tab, copy ALL environment variables from API service (they share the same config)

6. **Important**: Worker doesn't need a public domain (it's an internal service)

7. Click **"Deploy"**

### 3.3 Deploy WebSocket Service

1. Click **"+ New"** → **"GitHub Repo"**
2. Select your Novu repository
3. Name the service: **"novu-ws"**
4. In **Settings**:
   - **Dockerfile Path**: `apps/ws/Dockerfile`
   - Or **Config File Path**: `railway-ws.json`
   - **Port**: `3002`

5. In **Variables** tab, copy environment variables from API, plus add:

```bash
PORT=3002
WS_PORT=3002
```

6. Go to **Settings** → **Networking** → **Generate Domain**
7. Copy the WebSocket URL (you'll need this for other services)

8. Click **"Deploy"**

## Step 4: Deploy Dashboard (Frontend UI)

1. Click **"+ New"** → **"GitHub Repo"**
2. Select your Novu repository
3. Name the service: **"novu-dashboard"**
4. In **Settings**:
   - **Dockerfile Path**: `apps/dashboard/dockerfile`
   - Or **Config File Path**: `railway-dashboard.json`
   - **Port**: `4000`

5. In **Variables** tab, add:

```bash
# === BUILD-TIME VARIABLES (Vite) ===
VITE_API_HOSTNAME=<API_SERVICE_URL_WITHOUT_HTTPS>
VITE_WEBSOCKET_HOSTNAME=<WS_SERVICE_URL_WITHOUT_HTTPS>

# Example:
# If API URL is https://novu-api-production.up.railway.app
# Then: VITE_API_HOSTNAME=novu-api-production.up.railway.app
# If WS URL is https://novu-ws-production.up.railway.app
# Then: VITE_WEBSOCKET_HOSTNAME=novu-ws-production.up.railway.app

PORT=4000
NODE_ENV=production
```

6. Go to **Settings** → **Networking** → **Generate Domain**
7. This is your Dashboard URL - copy it!

8. Click **"Deploy"**

## Step 5: Update Cross-Service URLs

Now that all services are deployed, update the environment variables:

### Update API Service

Go back to **novu-api** → **Variables** and update:

```bash
FRONT_BASE_URL=https://<dashboard-domain>
REACT_APP_WS_URL=https://<ws-domain>
```

### Update WebSocket Service

Go to **novu-ws** → **Variables** and ensure:

```bash
CLIENT_SUCCESS_AUTH_REDIRECT=https://<dashboard-domain>
```

After updating, redeploy the services (Railway will auto-redeploy on variable changes).

## Step 6: Generate Secrets

Generate secure random secrets for your deployment:

```bash
# JWT_SECRET (64 characters)
openssl rand -hex 32

# STORE_ENCRYPTION_KEY (32 characters)
openssl rand -hex 16

# NOVU_SECRET_KEY (any length)
openssl rand -hex 32
```

Replace the placeholder secrets in your API and Worker services.

## Step 7: Access Your Novu Dashboard

1. Open your dashboard URL: `https://<dashboard-domain>`
2. You should see the Novu login/registration page
3. Create an account or sign in
4. Start using Novu!

## Troubleshooting

### Dashboard shows "Cannot connect to API"

**Check:**
1. API service is running (check logs in Railway)
2. `VITE_API_HOSTNAME` is set correctly (without `https://`)
3. API service has a public domain generated
4. CORS is configured correctly (should work by default)

**Fix:**
- Rebuild dashboard with correct `VITE_API_HOSTNAME`
- Check API logs for errors

### WebSocket connection fails

**Check:**
1. WebSocket service is running
2. `VITE_WEBSOCKET_HOSTNAME` is set correctly
3. WebSocket service has public domain
4. Port 3002 is exposed

**Fix:**
- Redeploy WS service
- Update `REACT_APP_WS_URL` in API service

### 500 errors from API

**Check:**
1. MongoDB connection is working (`MONGO_URL`)
2. Redis connection is working (`REDIS_HOST`, `REDIS_PORT`)
3. All required secrets are set (`JWT_SECRET`, `STORE_ENCRYPTION_KEY`)
4. Check API service logs in Railway

**Fix:**
- Verify all environment variables are set
- Check MongoDB and Redis plugins are running
- Review API logs for specific errors

### Worker not processing jobs

**Check:**
1. Worker service is running (check logs)
2. Worker has same environment variables as API
3. Redis connection is working
4. MongoDB connection is working

**Fix:**
- Ensure all env vars are copied from API
- Restart worker service
- Check worker logs for errors

### MongoDB resource exhaustion / Thread creation failures

**Symptoms:**
```
pthread_create failed: Resource temporarily unavailable
Failed to create service entry worker thread
connection <monitor> to X.X.X.X:27017 closed
```

**Root Cause:**
MongoDB's thread pool is exhausted or corrupted. This can happen with:
- Too many concurrent connections
- Memory pressure on the MongoDB container
- Kernel-level resource limits hit

**Fix:**
1. **Redeploy MongoDB** (not just restart):
   - Railway Dashboard → MongoDB service
   - Settings → Click "Redeploy"
   - This rebuilds the container from scratch, clearing thread pool corruption

2. **Increase MongoDB Resources** (if issue persists):
   - Railway Dashboard → MongoDB service
   - Settings → Resources
   - Increase Memory to at least 1GB
   - Increase CPU if available

3. **Reduce Connection Pool Sizes** (if frequently occurring):
   - Update API/Worker/WS services:
   ```bash
   MONGO_MIN_POOL_SIZE=10  # Down from 50
   MONGO_MAX_POOL_SIZE=100 # Down from 500
   ```

**Note:** Restart alone won't fix this issue - you need a full redeploy to rebuild the container.

## Scaling

Railway allows you to scale services:

1. Go to service **Settings** → **Resources**
2. Adjust:
   - **CPU**: Increase for better performance
   - **Memory**: API/Worker need at least 512MB-1GB
   - **Replicas**: Add more instances (horizontal scaling)

Recommended scaling:
- **API**: 2-4 replicas, 1GB RAM each
- **Worker**: 2-3 replicas, 1GB RAM each
- **WebSocket**: 1-2 replicas, 512MB RAM each
- **Dashboard**: 1 replica, 256MB RAM

## Cost Optimization

**Free Tier:**
Railway provides $5 free credits per month. For production use, you'll likely need a paid plan.

### Use Watch Patterns (Critical for Cost Savings!)

**The configurations in `railway-*.json` files already include watch patterns that prevent unnecessary rebuilds.**

Without watch patterns, **every single Git commit triggers ALL services to rebuild**, even if you only changed one file. This dramatically increases costs.

**How it works:**
- Each service only rebuilds when files in its watch patterns change
- Example: Editing `apps/dashboard/src/components/Button.tsx` only triggers Dashboard rebuild
- Other services (API, Worker, WS) don't rebuild, saving time and money

**Watch Pattern Configuration:**
```json
{
  "build": {
    "watchPatterns": [
      "apps/api/**",        // Only rebuild when API code changes
      "libs/**",            // Or shared library code changes
      "packages/**",        // Or package code changes
      "pnpm-lock.yaml",    // Or dependencies change
      "package.json",
      ".npmrc"
    ]
  }
}
```

**Cost Impact:**
- **Before**: 4 services × every commit = 4× build cost per commit
- **After**: Only services with changed files rebuild = 1× build cost per commit (typically)
- **Savings**: 75% reduction in build costs for most commits

**Additional Cost Optimization Tips:**

1. **Use Private Network Endpoints (Critical!)** 🚨
   - Railway charges egress fees for data leaving their network
   - **Always use `MONGO_PRIVATE_URL` instead of `MONGO_URL`**
   - **Always use `REDIS_PRIVATE_URL` instead of `REDIS_URL`**
   - Private endpoints use Railway's internal network (FREE)
   - Public endpoints route through internet (COSTS MONEY)

   **How to check:**
   - Railway Dashboard → Each service → Variables tab
   - Look for warnings: "This variable references a public endpoint"
   - Replace `${{MongoDB.MONGO_URL}}` with `${{MongoDB.MONGO_PRIVATE_URL}}`
   - Replace `${{Redis.REDIS_URL}}` with `${{Redis.REDIS_PRIVATE_URL}}`

   **Cost Impact:**
   - API/Worker/WS constantly query MongoDB/Redis
   - Using public endpoints = $$ per GB of data transfer
   - Using private endpoints = FREE
   - **Savings: Can be 50%+ of your total Railway bill**

2. Use Railway's MongoDB/Redis plugins (optimized pricing)
3. **Commit these railway-*.json files to enable watch patterns** ✅
4. Scale down non-production environments
5. Use external S3-compatible storage (cheaper than Railway volumes)
6. Monitor resource usage in Railway dashboard
7. Group related changes in single commits to minimize builds

## Advanced: Using External Services

### External MongoDB

If you have an existing MongoDB (MongoDB Atlas, etc.):

```bash
MONGO_URL=mongodb+srv://<user>:<password>@<cluster>.mongodb.net/novu?retryWrites=true&w=majority
```

### External Redis

If you have an existing Redis (Upstash, Redis Cloud, etc.):

```bash
REDIS_HOST=<your-redis-host>
REDIS_PORT=<your-redis-port>
REDIS_PASSWORD=<your-redis-password>
REDIS_TLS=true  # if using TLS
```

## Production Checklist

Before going to production:

- [ ] All secrets are strong and unique (JWT_SECRET, STORE_ENCRYPTION_KEY)
- [ ] MongoDB backups are configured
- [ ] Redis persistence is enabled
- [ ] S3 storage is configured (not using local storage)
- [ ] Rate limiting is enabled (`IS_API_RATE_LIMITING_ENABLED=true`)
- [ ] API idempotency is enabled (`IS_API_IDEMPOTENCY_ENABLED=true`)
- [ ] User registration is controlled (`DISABLE_USER_REGISTRATION=true` if needed)
- [ ] Monitoring is set up (Sentry, New Relic, or Railway metrics)
- [ ] Custom domains are configured (optional)
- [ ] SSL certificates are valid
- [ ] All services are scaled appropriately
- [ ] Health checks are configured
- [ ] Logs are being monitored

## Support

For issues specific to:
- **Novu**: Check [Novu GitHub Issues](https://github.com/novuhq/novu/issues)
- **Railway**: Check [Railway Discord](https://discord.gg/railway)

## Next Steps

After successful deployment:

1. **Configure notification providers** (SendGrid, Twilio, FCM, etc.)
2. **Create notification workflows** in the dashboard
3. **Integrate with your application** using Novu SDKs
4. **Set up monitoring and alerts**
5. **Configure custom domains** for production use

Happy building with Novu! 🚀
