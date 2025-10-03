# Function Package Architecture

**Package**: `packages/function`
**Purpose**: Cloudflare Workers functions for cloud services
**Runtime**: Cloudflare Workers
**Framework**: Hono

---

## Overview

The `function` package contains **serverless functions** deployed to Cloudflare Workers for cloud integrations:

- GitHub App authentication
- OAuth flows
- Webhooks
- API proxies

These functions complement the core opencode server by handling cloud-specific operations that require external endpoints.

---

## Technology Stack

- **Cloudflare Workers**: Edge compute platform
- **Hono**: Lightweight HTTP framework
- **@octokit/rest**: GitHub API client
- **jose**: JWT/JWE library for auth

---

## Package Structure

```
packages/function/
├── src/
│   ├── github.ts       # GitHub App integration
│   ├── oauth.ts        # OAuth handlers
│   └── webhook.ts      # Webhook receivers
├── package.json
└── tsconfig.json
```

---

## Functions

### 1. GitHub App Authentication

Handles GitHub App installation and authentication:

```typescript
// src/github.ts
app.post("/github/install", async (c) => {
  const { code } = await c.req.json()

  // Exchange code for installation token
  const token = await octokit.apps.createInstallationAccessToken({
    installation_id: installationId
  })

  return c.json({ token: token.data.token })
})
```

### 2. OAuth Flows

Generic OAuth callback handler:

```typescript
// src/oauth.ts
app.get("/oauth/callback", async (c) => {
  const { code, state } = c.req.query()

  // Exchange code for access token
  const tokens = await exchangeCode(code)

  return c.redirect(`/success?token=${tokens.access_token}`)
})
```

### 3. Webhooks

Receives webhooks from external services:

```typescript
// src/webhook.ts
app.post("/webhook/github", async (c) => {
  const signature = c.req.header("x-hub-signature-256")
  const body = await c.req.text()

  // Verify signature
  if (!verifySignature(body, signature)) {
    return c.text("Invalid signature", 401)
  }

  // Process webhook
  await handleGitHubWebhook(JSON.parse(body))

  return c.text("OK")
})
```

---

## Deployment

Deployed to Cloudflare Workers via SST:

```bash
# In infra/
sst deploy --stage production
```

Functions available at:
- `https://api.opencode.ai/github/*`
- `https://api.opencode.ai/oauth/*`

---

## Environment Variables

```
GITHUB_APP_ID=123456
GITHUB_APP_PRIVATE_KEY="..."
GITHUB_WEBHOOK_SECRET="..."
OAUTH_CLIENT_ID="..."
OAUTH_CLIENT_SECRET="..."
```

---

## Future Functions

- GitLab integration
- Slack notifications
- Discord bot
- Email webhooks