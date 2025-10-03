# Console Package Architecture

**Package**: `packages/console`
**Purpose**: Management console for opencode cloud services
**Framework**: Solid Start
**Deployment**: SST

---

## Overview

The `console` package provides a **web-based management console** for opencode cloud services. It's a separate application from the main `app` package, focused on administrative and account management features.

Features:
- User account management
- Billing and subscriptions
- Usage analytics
- API key management
- Team management
- Audit logs

---

## Package Structure

```
packages/console/
├── app/              # Solid Start application
├── core/             # Business logic
├── function/         # Serverless functions
├── resource/         # SST resources
└── scripts/          # Build scripts
```

---

## Technology Stack

- **Solid Start**: Full-stack Solid.js framework
- **SST**: Infrastructure as code
- **TailwindCSS**: Styling
- **tRPC**: Type-safe API layer
- **Postgres**: Database (via SST)

---

## Key Features

### 1. Dashboard
- Usage overview (sessions, tokens, cost)
- Recent activity
- Quick actions

### 2. Account Settings
- Profile management
- Email preferences
- Notification settings

### 3. Billing
- Subscription plans
- Payment methods
- Invoices and receipts
- Usage-based billing

### 4. API Keys
- Create/revoke API keys
- Key permissions and scopes
- Usage tracking per key

### 5. Team Management
- Invite members
- Role-based access control
- Audit logs

### 6. Analytics
- Session metrics
- Token usage trends
- Cost breakdown by model
- Error rates

---

## Architecture

### Solid Start

Full-stack framework with:
- Server-side rendering (SSR)
- API routes
- File-based routing
- Data loading

### Database Schema

```sql
-- Users
CREATE TABLE users (
  id UUID PRIMARY KEY,
  email TEXT UNIQUE,
  name TEXT,
  created_at TIMESTAMP
);

-- API Keys
CREATE TABLE api_keys (
  id UUID PRIMARY KEY,
  user_id UUID REFERENCES users(id),
  key TEXT UNIQUE,
  name TEXT,
  created_at TIMESTAMP
);

-- Usage
CREATE TABLE usage (
  id UUID PRIMARY KEY,
  user_id UUID REFERENCES users(id),
  session_id TEXT,
  tokens_input INT,
  tokens_output INT,
  cost DECIMAL,
  timestamp TIMESTAMP
);
```

---

## Deployment

Deployed via SST to AWS/Cloudflare:

```bash
cd packages/console
sst deploy --stage production
```

Available at: `https://console.opencode.ai`

---

## Future Features

- Multi-region deployment
- Custom domains for API endpoints
- Webhooks for events
- Export data (GDPR compliance)
- SSO integration (SAML, OIDC)