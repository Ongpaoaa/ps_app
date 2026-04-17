# EAS Hosting Deployment Guide

This guide covers deploying the API routes to **EAS Hosting** (Cloudflare Workers).

## Prerequisites

```bash
# Ensure you have the latest EAS CLI
npm install -g eas-cli@latest

# Login to EAS
eas login
```

## Environment Variables Setup

Before deploying, set up the required environment variables:

### Production Environment

```bash
eas env:create --name EXPO_PUBLIC_SUPABASE_URL \
  --value "https://iikrvgjfkuijcpvdwzvv.supabase.co/" \
  --environment production

eas env:create --name EXPO_PUBLIC_SUPABASE_PUBLISHABLE_KEY \
  --value "sb_publishable_3BzLdHOi5zinIv2WFRxVYQ_8gJq3nc_" \
  --environment production
```

### Preview Environment

```bash
eas env:create --name EXPO_PUBLIC_SUPABASE_URL \
  --value "https://iikrvgjfkuijcpvdwzvv.supabase.co/" \
  --environment preview

eas env:create --name EXPO_PUBLIC_SUPABASE_PUBLISHABLE_KEY \
  --value "sb_publishable_3BzLdHOi5zinIv2WFRxVYQ_8gJq3nc_" \
  --environment preview
```

### Verify Environment Variables

```bash
eas env:list
```

## Deployment

### First Deployment

```bash
# Deploy to EAS Hosting
eas deploy

# This will:
# 1. Build the API routes for Cloudflare Workers
# 2. Deploy to a unique URL
# 3. Show the deployment URL
```

### Subsequent Deployments

```bash
# Deploy updates
eas deploy

# Or with a specific channel
eas deploy --channel production
```

## API Endpoints After Deployment

Once deployed, your API routes will be available at:

```
https://<your-deployment-url>/api/hackathon/home-bundle
https://<your-deployment-url>/api/hackathon/phase/:phaseId
https://<your-deployment-url>/api/hackathon/team/:teamId
https://<your-deployment-url>/api/hackathon/webhook/cache-invalidate
```

## Mobile App Configuration

After deploying, update your mobile app to use the hosted API:

### Option 1: Environment Variable

Set `EXPO_PUBLIC_API_BASE_URL` in your EAS build profile:

```json
// eas.json
{
  "build": {
    "production": {
      "env": {
        "EXPO_PUBLIC_API_BASE_URL": "https://your-deployment-url"
      }
    }
  }
}
```

Then use it in your hooks:

```typescript
const API_BASE = process.env.EXPO_PUBLIC_API_BASE_URL || '';
const response = await fetch(`${API_BASE}/api/hackathon/home-bundle?participant_id=${id}`);
```

### Option 2: Same-Origin (Web Only)

If using EAS Hosting for both web and API:

```typescript
// Relative URLs work for web
const response = await fetch('/api/hackathon/home-bundle?participant_id=${id}');
```

### Option 3: Direct Supabase (Fallback)

For native builds without API, fall back to direct Supabase:

```typescript
const API_BASE = process.env.EXPO_PUBLIC_API_BASE_URL;
if (API_BASE) {
  // Use API routes
  const response = await fetch(`${API_BASE}/api/hackathon/home-bundle`);
} else {
  // Fallback to direct Supabase
  const { data } = await supabase.from('hackathon_team_members').select('*');
}
```

## Custom Domain (Optional)

Configure a custom domain in the Expo dashboard or via CLI:

```bash
eas domain:create --domain api.passionseed.org
```

## Troubleshooting

### Check Deployment Status

```bash
eas deploy:list
```

### View Logs

```bash
eas logs --channel production
```

### Local Testing

Test API routes locally before deploying:

```bash
npx expo serve
```

Then test with curl:

```bash
curl http://localhost:8081/api/hackathon/home-bundle?participant_id=test-id
```

## Important Notes

### Cloudflare Workers Runtime

EAS Hosting runs on Cloudflare Workers, which means:

- ✅ **Works:** fetch, Response, Request, crypto.subtle, Web APIs
- ❌ **Doesn't work:** Node.js fs, native Node modules, localStorage

### Supabase Auth on Server

Since API routes are serverless, they don't maintain user sessions. To authenticate:

1. Pass the user's JWT token from the client
2. Verify it server-side
3. Or use service role key for admin operations (be careful!)

Example with JWT:

```typescript
// In API route
const authHeader = request.headers.get('Authorization');
const token = authHeader?.replace('Bearer ', '');

// Create Supabase client with user's token
const supabase = createClient(url, key, {
  global: {
    headers: {
      Authorization: `Bearer ${token}`,
    },
  },
});
```

## Cache Headers

The API routes already include cache headers:

```typescript
'Cache-Control': 'public, max-age=300, stale-while-revalidate=600'
```

This means:
- **5 minutes:** Fresh cache
- **10 minutes:** Stale-while-revalidate window

## Webhook Configuration

After deploying, configure Supabase webhooks:

1. Go to Supabase Dashboard → Database → Webhooks
2. Create webhook pointing to:
   ```
   https://your-deployment-url/api/hackathon/webhook/cache-invalidate
   ```
3. Select tables: `hackathon_teams`, `hackathon_program_phases`, `hackathon_team_members`
4. Choose events: INSERT, UPDATE, DELETE
