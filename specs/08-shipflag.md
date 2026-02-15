# ShipFlag — Feature Flag Management

## Executive Summary

ShipFlag is a simple, fast, affordable feature flag service built for startup engineering teams. It provides percentage-based rollouts, user targeting, environment management, and edge-powered evaluation in under 10ms — without the enterprise price tag of LaunchDarkly or the complexity of self-hosting.

---

## Problem Statement

**The pain:**
- Shipping features behind flags is a best practice, but the tooling landscape is split between expensive enterprise tools and complex self-hosted solutions
- LaunchDarkly starts at $10/seat/month and quickly gets expensive for growing teams
- Self-hosting (Unleash, Flagsmith) requires DevOps setup, maintenance, and monitoring
- Many teams resort to config files or environment variables, which don't support gradual rollouts or targeting
- Without proper flag management, teams either ship to everyone at once (risky) or maintain long-lived feature branches (painful)

**Market gap:**
- LaunchDarkly: powerful but expensive ($10+/seat/mo, enterprise-focused)
- Unleash: open-source but requires self-hosting and maintenance
- Split.io: complex, enterprise-oriented
- Flagsmith: good but less focus on developer experience
- No tool optimizes specifically for startup teams (2-30 devs) with simple pricing and fast setup

---

## Target Users

### Primary Personas

**1. Ben — Startup CTO**
- 12-person engineering team
- Wants to ship safely without slowing down velocity
- Tried LaunchDarkly but couldn't justify the cost
- Needs: affordable, fast setup, solid SDK support

**2. Kira — Senior Frontend Engineer**
- Implementing flags in a Next.js app
- Wants instant flag evaluation (no loading spinners)
- Needs: React SDK with SSR support, simple API

**3. Leo — Backend Engineer**
- Building a Node.js/Python API
- Needs to gate features by user plan, region, or percentage
- Needs: server SDK with local evaluation, no network latency per check

---

## Core Features

### F1: Flag Types
- **Boolean flags**: on/off toggle
- **String flags**: return different string values per variation
- **Number flags**: numeric variations
- **JSON flags**: complex configuration objects
- Default value fallback for all types
- Flag descriptions and tags for organization

### F2: Targeting Rules
- **Percentage rollout**: enable for X% of users (sticky by user ID hash)
- **User targeting**: enable/disable for specific user IDs
- **Segment targeting**: define user segments by attributes
  - User attributes: plan, role, country, signup_date, custom properties
  - Operators: equals, not equals, contains, starts with, ends with, in list, regex, greater than, less than, before, after
  - Combine rules with AND/OR logic
- **Default rule**: what happens when no targeting rules match
- **Kill switch**: instantly disable for everyone
- Rule evaluation order (first match wins)
- Preview: "What would user X see?" test evaluation

### F3: Environments
- Pre-configured: Development, Staging, Production
- Custom environments (unlimited)
- Per-environment flag configurations
- Environment-specific API keys
- Promote flag config from one environment to another
- Environment-level kill switch

### F4: SDKs (Client-Side)
- **JavaScript/TypeScript**: vanilla JS, works everywhere
- **React**: hooks-based (`useFlag`, `useFlags`), SSR/RSC support, context provider
- **React Native**: mobile SDK
- **Vue**: composable-based
- **Next.js**: dedicated SDK with App Router + Pages Router support, middleware integration
- Client SDKs:
  - Initial load via API (single request for all flags)
  - Real-time updates via SSE/WebSocket (no polling)
  - Local storage caching for offline/instant start
  - Bundle size < 5KB gzipped

### F5: SDKs (Server-Side)
- **Node.js**: with streaming updates
- **Python**: sync and async variants
- **Go**: goroutine-safe evaluation
- **Ruby**: thread-safe
- **PHP**: Laravel integration
- **Java/Kotlin**: Spring Boot integration (v2)
- Server SDKs:
  - Local evaluation (rules downloaded, evaluated in-process)
  - Background polling for rule updates (configurable interval)
  - Zero network latency per flag check
  - Graceful degradation: use cached rules if API is unreachable

### F6: Edge Evaluation
- Cloudflare Workers-powered edge SDK
- Flag evaluation at the edge in <10ms globally
- Perfect for A/B testing and personalization
- No cold start penalties
- Automatic rule synchronization

### F7: Audit Log
- Every flag change logged: who, when, what changed
- Diff view showing before/after state
- Filter by user, flag, environment, date
- Exportable for compliance
- Webhook on flag changes (for CI/CD integration)

### F8: Flag Lifecycle Management
- Flag age tracking
- Stale flag detection: flags that haven't changed in 30+ days and are 100% rolled out
- Cleanup recommendations: "These 5 flags are fully rolled out and can be removed from code"
- Code references: search for flag usage across repositories (GitHub integration)
- Archive flags (soft delete)

### F9: A/B Testing Integration (v2)
- Define experiment: flag variations as experiment arms
- Track events (conversion, click, signup) via SDK
- Statistical significance calculation
- Results dashboard with confidence intervals
- Auto-rollout winner when significance reached

### F10: Analytics
- Flag evaluation volume (per flag, per environment)
- Variation distribution (how many users see each variation)
- Change frequency per flag
- SDK usage by platform
- Error rates (evaluation failures, timeout)

---

## Technical Architecture

### Tech Stack

| Layer | Technology |
|-------|-----------|
| Dashboard | Next.js 14+, Tailwind CSS, Radix UI |
| API | Next.js API Routes, tRPC |
| Database | PostgreSQL (Neon) |
| Cache | Redis (Upstash) — flag rule cache |
| Real-time | Cloudflare Durable Objects or Ably for SSE streaming |
| Edge Evaluation | Cloudflare Workers + KV |
| SDKs | TypeScript (client), various languages (server) |
| Auth | Clerk |
| Hosting | Vercel (dashboard), Cloudflare (edge/API) |

### System Flow

```
SDK (client/server)
    │
    ├── Initial: GET /api/flags/:env-key → all flag rules
    │
    ├── Evaluate locally (server SDK) or receive evaluated result (client SDK)
    │
    ├── Real-time updates: SSE stream → /api/flags/:env-key/stream
    │
    └── Track events: POST /api/events (for A/B testing)

Dashboard → API → PostgreSQL
    │
    └── On flag change:
        ├── Invalidate Redis cache
        ├── Push update to all connected SSE streams
        └── Update Cloudflare KV (edge cache)
```

### Data Model

```
Organization
├── id, name, slug, plan, created_at
│
├── Environment[]
│   ├── id, org_id, name, key (dev/staging/prod/custom)
│   ├── api_key_client, api_key_server (hashed)
│   ├── color (for UI)
│   └── sort_order
│
├── Flag[]
│   ├── id, org_id, key (unique string, e.g., "new-dashboard")
│   ├── name, description, tags[]
│   ├── type (boolean/string/number/json)
│   ├── variations (JSON[]: [{key: "on", value: true}, {key: "off", value: false}])
│   ├── created_at, updated_at
│   ├── is_archived
│   │
│   └── FlagEnvironmentConfig[]
│       ├── id, flag_id, environment_id
│       ├── enabled (boolean — kill switch)
│       ├── default_variation_key
│       ├── rules (JSON[]):
│       │   [{
│       │     variation_key: "on",
│       │     percentage: 50,         // optional
│       │     segments: ["beta-users"], // optional
│       │     user_ids: ["user-123"], // optional
│       │     conditions: [{attribute, operator, value}]
│       │   }]
│       └── updated_at
│
├── Segment[]
│   ├── id, org_id, name, key
│   ├── rules (JSON): [{attribute, operator, value, logic}]
│   └── description
│
├── AuditLog[]
│   ├── id, org_id, user_id
│   ├── flag_id, environment_id
│   ├── action (created/updated/archived/toggled)
│   ├── previous_state (JSON), new_state (JSON)
│   └── created_at
│
├── Event[] (for A/B testing, time-series)
│   ├── id, org_id, flag_id, environment_id
│   ├── user_id, variation_key
│   ├── event_name, event_value
│   └── created_at
│
└── Member[]
    ├── id, org_id, user_id, role (owner/admin/member/viewer)
    └── invited_at, joined_at
```

---

## UI/UX — Key Screens

### 1. Flag List
- Table: flag name, key, type badge, environments (mini toggles for each)
- Quick toggle: enable/disable per environment directly from list
- Tags for filtering
- Search by name or key
- "Stale" badge on flags that should be cleaned up
- Create flag button

### 2. Flag Detail
- Header: name, key, description, type
- Environment tabs (Dev | Staging | Production)
- Per environment:
  - Enable/disable toggle
  - Targeting rules builder
  - Percentage rollout slider
  - User ID targeting input
  - Segment selector
  - "Test evaluation" panel: enter user attributes, see result
- Variations configuration
- Audit log tab
- Usage/analytics tab

### 3. Targeting Rule Builder
- Visual builder with sentence structure
- "If user [attribute] [operator] [value], serve [variation]"
- Add condition, add rule group (AND/OR)
- Percentage slider for gradual rollout
- Preview panel showing evaluation logic

### 4. Segments
- List of defined segments
- Segment builder: attribute conditions
- Preview: "X users match this segment" (if user data is available)
- Which flags use this segment

### 5. Audit Log
- Chronological list of all changes
- Filter by flag, user, environment, action type
- Diff view: click entry to see before/after JSON diff
- Export button

### 6. Settings
- Environments: add/remove/rename
- API keys: rotate, copy
- Team members: invite, role management
- Integrations: GitHub (code references), webhooks
- Billing

---

## Monetization

### Free Tier
- 5 flags
- 1,000 MAU (monthly active users evaluated)
- 2 environments (dev, production)
- JavaScript + React SDKs
- Community support

### Pro — $25/month
- Unlimited flags
- 10,000 MAU
- Unlimited environments
- All SDKs
- Segments and advanced targeting
- Audit log
- Flag lifecycle management
- Email support

### Scale — $79/month
- Everything in Pro
- 100,000 MAU
- A/B testing and experiments
- Edge evaluation
- Webhooks
- API access
- Priority support

### Enterprise — Custom
- Unlimited MAU
- SSO/SAML
- SLA
- Custom SDK support
- Dedicated infrastructure
- Compliance (SOC 2)

---

## Go-to-Market Strategy

- Position as "LaunchDarkly for startups" — 80% of features at 20% of the price
- Launch on Hacker News and Product Hunt with pricing comparison
- Blog: "Feature flags done right for small teams"
- Free tier is genuinely useful (not crippled)
- SDK quality as differentiator (great DX, excellent docs)
- Developer-focused content: tutorials for Next.js, React, Node.js
- Open-source the client SDKs (build trust)
- Comparison landing pages for SEO

---

## Success Metrics

| Metric | Target (Month 6) | Target (Month 12) |
|--------|-------------------|---------------------|
| Organizations | 300 | 1,500 |
| Active flags | 3,000 | 20,000 |
| Flag evaluations/day | 5M | 50M |
| SDK downloads (npm) | 10K/mo | 50K/mo |
| Paying customers | 50 | 200 |
| MRR | $2,500 | $12,000 |

---

## MVP Scope (v1.0)

### In Scope
- Boolean and string flags
- Percentage rollout
- User ID targeting
- 3 environments (dev, staging, prod)
- JavaScript + React + Node.js SDKs
- Dashboard with flag management
- Basic audit log
- Real-time updates (SSE)

### Out of Scope (v1.1+)
- JSON/number flags
- Segments
- Edge evaluation
- A/B testing
- Flag lifecycle management
- Go/Python/Ruby SDKs
- Advanced analytics

### MVP Timeline: 6-7 weeks
- Week 1: Data model, flag CRUD API, environment system
- Week 2: Evaluation engine (targeting rules, percentage hashing)
- Week 3: JavaScript + React SDKs with real-time updates
- Week 4: Node.js server SDK with local evaluation
- Week 5: Dashboard UI (flag list, detail, targeting builder)
- Week 6: Audit log, API key management, settings
- Week 7: Billing, docs site, polish, launch
