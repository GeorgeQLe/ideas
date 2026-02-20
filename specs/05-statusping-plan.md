# 5. StatusPing — Uptime Monitoring with AI Incident Reports: Implementation Plan

**MVP Scope:** HTTP/HTTPS monitoring from 5 global regions (1-min minimum interval), email + Slack alerts, hosted status page with basic branding, auto-incident creation on downtime, AI-drafted incident updates with human approval, basic response time charts, and Stripe billing (Free / Pro $29/mo / Business $79/mo).

---

## Tech Decisions

- **Monitoring workers: Node.js on Railway (not Go on Fly.io).** Go would be ideal for raw performance, but for a solo/small team MVP, keeping the entire stack in TypeScript reduces cognitive overhead. Node.js workers deployed as separate Railway services in 5 regions (us-east, us-west, eu-west, eu-central, ap-southeast) are sufficient for HTTP checks at 1-min intervals. Each worker runs a simple polling loop that reads its assigned monitors from Redis and POSTs results back to the central API. We can migrate to Go later if per-check cost becomes a concern (unlikely below 10K monitors).
- **Time-series data: Regular PostgreSQL with monthly partitioned tables (not TimescaleDB).** TimescaleDB adds operational complexity on Neon/Supabase. For MVP scale (<25K monitors), a native PostgreSQL partitioned `checks` table with monthly partitions and a composite index on `(monitor_id, checked_at DESC)` is performant and simple. We add a cron job to create partitions 2 months ahead and drop partitions beyond the plan's retention window. This handles up to ~50M rows/month before needing to reconsider.
- **Status pages: ISR on Vercel (not Cloudflare Workers+KV).** Keeps the entire app in one deployment. Status pages use Next.js ISR with 10-second revalidation. The public status page route (`/s/[slug]`) reads from a Redis cache that the API updates on every status change, so pages reflect changes within ~10 seconds without a full database query on every request. Custom domains can be added later via Vercel's domain API.
- **Real-time: Upstash Redis pub/sub for live status updates.** The dashboard and public status pages use Server-Sent Events (SSE) backed by Upstash Redis pub/sub channels (one channel per organization: `org:{orgId}:events`). When a check result changes a monitor's status, the API publishes to the channel, and connected SSE clients receive the update instantly.
- **AI: OpenAI GPT-4o for incident drafting.** Used to generate incident updates (investigating, identified, monitoring, resolved) from structured check failure data. Prompt includes: monitor name, URL, failing regions, error codes, failure duration, and prior incident updates. All AI drafts require one-click approval before publishing to the status page. Estimated cost: ~$0.01-0.03 per incident draft.
- **Queue: Upstash QStash for reliable check result processing.** Workers POST check results to a QStash-backed endpoint that handles alert evaluation, incident creation, and status page cache invalidation. QStash provides at-least-once delivery and automatic retries, eliminating the need for a self-hosted BullMQ/Redis queue.
- **Auth: Clerk with organization support.** Clerk organizations map 1:1 to StatusPing organizations. Clerk handles invite flows, roles (admin/member), and SSO for future Enterprise tier.
- **Database: Neon PostgreSQL.** Serverless Postgres with branching for dev/preview environments. Connection pooling via Neon's built-in pgbouncer proxy.

---

## Database Schema (Drizzle)

**Tables:** organizations, monitors, checks, components, status_pages, incidents, incident_updates, alert_channels, monitor_alert_channels, subscribers, maintenance_windows

### Full Schema Definition

```typescript
// src/server/db/schema.ts

import { relations, sql } from "drizzle-orm";
import {
  pgTable,
  uuid,
  text,
  boolean,
  integer,
  timestamp,
  jsonb,
  real,
  index,
  uniqueIndex,
} from "drizzle-orm/pg-core";

// ---------------------------------------------------------------------------
// Organizations
// ---------------------------------------------------------------------------
export const organizations = pgTable("organizations", {
  id: uuid("id").primaryKey().defaultRandom(),
  clerkOrgId: text("clerk_org_id").unique().notNull(),
  name: text("name").notNull(),
  slug: text("slug").unique().notNull(),
  plan: text("plan").default("free").notNull(), // free | pro | business
  stripeCustomerId: text("stripe_customer_id"),
  stripeSubscriptionId: text("stripe_subscription_id"),
  settings: jsonb("settings").default({}).$type<{
    timezone?: string;
    defaultAlertChannels?: string[];
  }>(),
  createdAt: timestamp("created_at").defaultNow().notNull(),
  updatedAt: timestamp("updated_at").defaultNow().notNull(),
});

// ---------------------------------------------------------------------------
// Monitors
// ---------------------------------------------------------------------------
export const monitors = pgTable(
  "monitors",
  {
    id: uuid("id").primaryKey().defaultRandom(),
    orgId: uuid("org_id")
      .references(() => organizations.id, { onDelete: "cascade" })
      .notNull(),
    name: text("name").notNull(),
    url: text("url").notNull(),
    method: text("method").default("GET").notNull(), // GET | POST | PUT | HEAD
    headers: jsonb("headers").default({}).$type<Record<string, string>>(),
    body: text("body"),
    expectedStatusCodes: jsonb("expected_status_codes")
      .default([200])
      .$type<number[]>(),
    timeoutMs: integer("timeout_ms").default(30000).notNull(),
    intervalSeconds: integer("interval_seconds").default(300).notNull(), // 60 | 120 | 300 | 600
    regions: jsonb("regions")
      .default(["us-east"])
      .$type<string[]>(),
    status: text("status").default("unknown").notNull(), // up | down | degraded | paused | unknown
    isPaused: boolean("is_paused").default(false).notNull(),
    componentId: uuid("component_id").references(() => components.id, {
      onDelete: "set null",
    }),
    lastCheckedAt: timestamp("last_checked_at"),
    lastStatusChangeAt: timestamp("last_status_change_at"),
    consecutiveFailures: integer("consecutive_failures").default(0).notNull(),
    confirmationsRequired: integer("confirmations_required").default(2).notNull(), // regions that must fail
    createdAt: timestamp("created_at").defaultNow().notNull(),
    updatedAt: timestamp("updated_at").defaultNow().notNull(),
  },
  (table) => ({
    orgIdIdx: index("monitors_org_id_idx").on(table.orgId),
    statusIdx: index("monitors_status_idx").on(table.status),
  })
);

// ---------------------------------------------------------------------------
// Checks (time-series, will be partitioned by month via raw SQL migration)
// ---------------------------------------------------------------------------
export const checks = pgTable(
  "checks",
  {
    id: uuid("id").defaultRandom().notNull(),
    monitorId: uuid("monitor_id").notNull(), // no FK for partitioned tables
    region: text("region").notNull(), // us-east | us-west | eu-west | eu-central | ap-southeast
    status: text("status").notNull(), // up | down | degraded
    statusCode: integer("status_code"),
    responseTimeMs: integer("response_time_ms"),
    errorMessage: text("error_message"),
    headers: jsonb("headers").$type<Record<string, string>>(),
    checkedAt: timestamp("checked_at").defaultNow().notNull(),
  },
  (table) => ({
    monitorCheckedAtIdx: index("checks_monitor_checked_at_idx").on(
      table.monitorId,
      table.checkedAt
    ),
    checkedAtIdx: index("checks_checked_at_idx").on(table.checkedAt),
  })
);

// ---------------------------------------------------------------------------
// Components (status page grouping)
// ---------------------------------------------------------------------------
export const components = pgTable("components", {
  id: uuid("id").primaryKey().defaultRandom(),
  orgId: uuid("org_id")
    .references(() => organizations.id, { onDelete: "cascade" })
    .notNull(),
  statusPageId: uuid("status_page_id")
    .references(() => statusPages.id, { onDelete: "cascade" })
    .notNull(),
  name: text("name").notNull(),
  description: text("description"),
  groupName: text("group_name"), // optional grouping header
  status: text("status").default("operational").notNull(), // operational | degraded | partial_outage | major_outage | maintenance
  sortOrder: integer("sort_order").default(0).notNull(),
  createdAt: timestamp("created_at").defaultNow().notNull(),
});

// ---------------------------------------------------------------------------
// Status Pages
// ---------------------------------------------------------------------------
export const statusPages = pgTable(
  "status_pages",
  {
    id: uuid("id").primaryKey().defaultRandom(),
    orgId: uuid("org_id")
      .references(() => organizations.id, { onDelete: "cascade" })
      .notNull(),
    slug: text("slug").notNull(), // unique across all orgs for subdomain routing
    title: text("title").notNull(),
    description: text("description"),
    logoUrl: text("logo_url"),
    faviconUrl: text("favicon_url"),
    branding: jsonb("branding").default({}).$type<{
      primaryColor?: string;
      backgroundColor?: string;
      textColor?: string;
      fontFamily?: string;
      customCss?: string;
    }>(),
    customDomain: text("custom_domain"),
    isPublic: boolean("is_public").default(true).notNull(),
    createdAt: timestamp("created_at").defaultNow().notNull(),
    updatedAt: timestamp("updated_at").defaultNow().notNull(),
  },
  (table) => ({
    slugIdx: uniqueIndex("status_pages_slug_idx").on(table.slug),
  })
);

// ---------------------------------------------------------------------------
// Incidents
// ---------------------------------------------------------------------------
export const incidents = pgTable(
  "incidents",
  {
    id: uuid("id").primaryKey().defaultRandom(),
    orgId: uuid("org_id")
      .references(() => organizations.id, { onDelete: "cascade" })
      .notNull(),
    title: text("title").notNull(),
    status: text("status").default("investigating").notNull(), // investigating | identified | monitoring | resolved
    severity: text("severity").default("major").notNull(), // minor | major | critical
    startedAt: timestamp("started_at").defaultNow().notNull(),
    resolvedAt: timestamp("resolved_at"),
    autoGenerated: boolean("auto_generated").default(false).notNull(),
    affectedComponentIds: jsonb("affected_component_ids")
      .default([])
      .$type<string[]>(),
    createdAt: timestamp("created_at").defaultNow().notNull(),
    updatedAt: timestamp("updated_at").defaultNow().notNull(),
  },
  (table) => ({
    orgIdStatusIdx: index("incidents_org_status_idx").on(
      table.orgId,
      table.status
    ),
  })
);

// ---------------------------------------------------------------------------
// Incident Updates
// ---------------------------------------------------------------------------
export const incidentUpdates = pgTable(
  "incident_updates",
  {
    id: uuid("id").primaryKey().defaultRandom(),
    incidentId: uuid("incident_id")
      .references(() => incidents.id, { onDelete: "cascade" })
      .notNull(),
    status: text("status").notNull(), // investigating | identified | monitoring | resolved
    message: text("message").notNull(), // the published message
    aiDraft: text("ai_draft"), // original AI-generated text before human edit
    isAiGenerated: boolean("is_ai_generated").default(false).notNull(),
    isInternal: boolean("is_internal").default(false).notNull(), // if true, not shown on status page
    postedBy: text("posted_by"), // Clerk user ID, null if auto-generated
    approved: boolean("approved").default(false).notNull(), // must be true to show on status page
    postedAt: timestamp("posted_at").defaultNow().notNull(),
  },
  (table) => ({
    incidentIdIdx: index("incident_updates_incident_id_idx").on(
      table.incidentId
    ),
  })
);

// ---------------------------------------------------------------------------
// Alert Channels
// ---------------------------------------------------------------------------
export const alertChannels = pgTable("alert_channels", {
  id: uuid("id").primaryKey().defaultRandom(),
  orgId: uuid("org_id")
    .references(() => organizations.id, { onDelete: "cascade" })
    .notNull(),
  name: text("name").notNull(),
  type: text("type").notNull(), // email | slack | webhook
  config: jsonb("config").notNull().$type<
    | { type: "email"; addresses: string[] }
    | { type: "slack"; webhookUrl: string; channel?: string }
    | { type: "webhook"; url: string; secret?: string }
  >(),
  enabled: boolean("enabled").default(true).notNull(),
  createdAt: timestamp("created_at").defaultNow().notNull(),
});

// ---------------------------------------------------------------------------
// Monitor <-> AlertChannel junction
// ---------------------------------------------------------------------------
export const monitorAlertChannels = pgTable(
  "monitor_alert_channels",
  {
    monitorId: uuid("monitor_id")
      .references(() => monitors.id, { onDelete: "cascade" })
      .notNull(),
    alertChannelId: uuid("alert_channel_id")
      .references(() => alertChannels.id, { onDelete: "cascade" })
      .notNull(),
  },
  (table) => ({
    pk: uniqueIndex("monitor_alert_channels_pk").on(
      table.monitorId,
      table.alertChannelId
    ),
  })
);

// ---------------------------------------------------------------------------
// Subscribers (public status page subscribers)
// ---------------------------------------------------------------------------
export const subscribers = pgTable(
  "subscribers",
  {
    id: uuid("id").primaryKey().defaultRandom(),
    statusPageId: uuid("status_page_id")
      .references(() => statusPages.id, { onDelete: "cascade" })
      .notNull(),
    email: text("email").notNull(),
    confirmed: boolean("confirmed").default(false).notNull(),
    confirmationToken: text("confirmation_token"),
    unsubscribeToken: text("unsubscribe_token").notNull(),
    subscribedAt: timestamp("subscribed_at").defaultNow().notNull(),
  },
  (table) => ({
    emailPageIdx: uniqueIndex("subscribers_email_page_idx").on(
      table.statusPageId,
      table.email
    ),
  })
);

// ---------------------------------------------------------------------------
// Maintenance Windows (out of MVP but schema placeholder)
// ---------------------------------------------------------------------------
export const maintenanceWindows = pgTable("maintenance_windows", {
  id: uuid("id").primaryKey().defaultRandom(),
  orgId: uuid("org_id")
    .references(() => organizations.id, { onDelete: "cascade" })
    .notNull(),
  title: text("title").notNull(),
  description: text("description"),
  startsAt: timestamp("starts_at").notNull(),
  endsAt: timestamp("ends_at").notNull(),
  affectedComponentIds: jsonb("affected_component_ids")
    .default([])
    .$type<string[]>(),
  createdAt: timestamp("created_at").defaultNow().notNull(),
});
```

### Partition Migration (raw SQL, run after Drizzle push)

```sql
-- migrations/0001_create_checks_partitioned.sql
-- Drop the Drizzle-created checks table and recreate as partitioned
DROP TABLE IF EXISTS checks;

CREATE TABLE checks (
  id         UUID DEFAULT gen_random_uuid() NOT NULL,
  monitor_id UUID NOT NULL,
  region     TEXT NOT NULL,
  status     TEXT NOT NULL,
  status_code INTEGER,
  response_time_ms INTEGER,
  error_message TEXT,
  headers    JSONB,
  checked_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
) PARTITION BY RANGE (checked_at);

CREATE INDEX checks_monitor_checked_at_idx ON checks (monitor_id, checked_at DESC);
CREATE INDEX checks_checked_at_idx ON checks (checked_at);

-- Create initial partitions (current month + next 2 months)
CREATE TABLE checks_2026_02 PARTITION OF checks
  FOR VALUES FROM ('2026-02-01') TO ('2026-03-01');
CREATE TABLE checks_2026_03 PARTITION OF checks
  FOR VALUES FROM ('2026-03-01') TO ('2026-04-01');
CREATE TABLE checks_2026_04 PARTITION OF checks
  FOR VALUES FROM ('2026-04-01') TO ('2026-05-01');
```

### Partition Maintenance Cron (runs monthly)

```sql
-- Called by src/app/api/cron/partitions/route.ts (Vercel Cron)
-- Dynamically creates next month's partition and drops expired ones

-- Create partition 2 months ahead:
-- CREATE TABLE checks_YYYY_MM PARTITION OF checks
--   FOR VALUES FROM ('YYYY-MM-01') TO ('YYYY-{MM+1}-01');

-- Drop partitions older than retention window:
-- Free plan: 7 days (drop anything > 1 month old)
-- Pro plan: 90 days (drop > 4 months old)
-- Business plan: 1 year (drop > 13 months old)
```

### Key Constraints and Indexes

- `organizations.slug` -- UNIQUE, used for org lookup from Clerk
- `status_pages.slug` -- UNIQUE across all orgs, used for public URL `/s/[slug]`
- `monitors.org_id` -- indexed, all monitor queries are org-scoped
- `checks(monitor_id, checked_at DESC)` -- composite index for response time charts (queries: "last 24h of checks for monitor X")
- `incidents(org_id, status)` -- composite index for "active incidents for this org"
- `subscribers(status_page_id, email)` -- UNIQUE, prevent duplicate subscriptions
- No foreign key from `checks.monitor_id` to `monitors.id` because PostgreSQL does not support FK references from partitioned tables to non-partitioned tables efficiently; enforce at application level

---

## Relations (Drizzle)

```typescript
export const organizationsRelations = relations(organizations, ({ many }) => ({
  monitors: many(monitors),
  components: many(components),
  statusPages: many(statusPages),
  incidents: many(incidents),
  alertChannels: many(alertChannels),
}));

export const monitorsRelations = relations(monitors, ({ one, many }) => ({
  organization: one(organizations, {
    fields: [monitors.orgId],
    references: [organizations.id],
  }),
  component: one(components, {
    fields: [monitors.componentId],
    references: [components.id],
  }),
  monitorAlertChannels: many(monitorAlertChannels),
}));

export const incidentsRelations = relations(incidents, ({ one, many }) => ({
  organization: one(organizations, {
    fields: [incidents.orgId],
    references: [organizations.id],
  }),
  updates: many(incidentUpdates),
}));

export const incidentUpdatesRelations = relations(incidentUpdates, ({ one }) => ({
  incident: one(incidents, {
    fields: [incidentUpdates.incidentId],
    references: [incidents.id],
  }),
}));

export const statusPagesRelations = relations(statusPages, ({ one, many }) => ({
  organization: one(organizations, {
    fields: [statusPages.orgId],
    references: [organizations.id],
  }),
  components: many(components),
  subscribers: many(subscribers),
}));

export const componentsRelations = relations(components, ({ one }) => ({
  organization: one(organizations, {
    fields: [components.orgId],
    references: [organizations.id],
  }),
  statusPage: one(statusPages, {
    fields: [components.statusPageId],
    references: [statusPages.id],
  }),
}));

export const subscribersRelations = relations(subscribers, ({ one }) => ({
  statusPage: one(statusPages, {
    fields: [subscribers.statusPageId],
    references: [statusPages.id],
  }),
}));
```

---

## Worker Architecture: Multi-Region Monitoring

### Overview

```
                    ┌─────────────────────────────┐
                    │    Vercel (Next.js App)       │
                    │  - Dashboard UI               │
                    │  - tRPC API                   │
                    │  - Check result ingestion     │
                    │  - SSE endpoint               │
                    │  - Cron jobs                  │
                    └──────────┬──────────┬─────────┘
                               │          │
                     writes     │          │ reads
                               │          │
              ┌────────────────▼──────────▼────────────────┐
              │           Neon PostgreSQL                     │
              │  organizations, monitors, checks (partitioned)│
              │  incidents, status_pages, etc.                 │
              └───────────────────────────────────────────────┘
                               │
                    ┌──────────▼──────────┐
                    │   Upstash Redis      │
                    │  - Monitor assignments│
                    │  - Status page cache  │
                    │  - Pub/sub channels   │
                    │  - Rate limiting      │
                    └──────────┬──────────┘
                               │
              ┌────────────────┼────────────────┐
              │                │                │
     ┌────────▼──┐   ┌────────▼──┐   ┌────────▼──┐
     │ Worker     │   │ Worker     │   │ Worker     │  ... (5 regions)
     │ us-east    │   │ eu-west    │   │ ap-southeast│
     │ (Railway)  │   │ (Railway)  │   │ (Railway)  │
     └────────────┘   └────────────┘   └────────────┘
```

### Worker Health Monitoring

Each regional worker implements a heartbeat mechanism to detect stale or crashed workers:

- **Heartbeat reporting**: Every worker POSTs a heartbeat to Redis every 30 seconds: `SET worker:{region}:heartbeat {timestamp} EX 120`. The key includes the worker region and auto-expires after 120 seconds if not refreshed.
- **Stale worker detection**: A Vercel cron job (`/api/cron/worker-health`) runs every 2 minutes. For each expected region, it checks `GET worker:{region}:heartbeat`. If the key is missing or the timestamp is older than 2 minutes, the worker is considered stale.
- **Alerting on stale workers**: When a stale worker is detected, the system fires an internal alert to the ops team via email (Resend) and Slack (webhook to an internal channel). The alert includes: region name, last heartbeat timestamp, number of monitors assigned to that region.
- **Automatic failover**: If a worker in region X is stale for more than 5 minutes, its assigned monitors are temporarily redistributed to the two nearest healthy regions (`SUNIONSTORE worker:{fallbackRegion}:monitors worker:{fallbackRegion}:monitors worker:{staleRegion}:monitors`). When the stale worker recovers and resumes heartbeats, its monitors are reassigned back. A `worker_failover_log` Redis list tracks all failover events for debugging.

### Worker Process (Node.js)

Each worker is a standalone Node.js process (not a Next.js app) deployed on Railway in a specific region. It:

1. **Polls Redis every 5 seconds** for its assigned monitor list: `SMEMBERS worker:{region}:monitors` returns a set of monitor IDs.
2. **Fetches monitor configs** from Redis hash: `HGETALL monitor:{monitorId}` (cached by the API whenever a monitor is created/updated).
3. **Maintains a local schedule** using `node-cron` or a simple interval map. For each monitor, it fires an HTTP check at the configured interval.
4. **Executes HTTP check**: uses `undici` (Node.js native HTTP client) for low-overhead requests. Records: status code, response time (via `performance.now()`), response headers, and any error.
5. **POSTs result to the central API** via QStash: `POST /api/webhooks/check-result` with payload `{ monitorId, region, status, statusCode, responseTimeMs, errorMessage, checkedAt }`. QStash signs the request with HMAC so the API can verify authenticity.

### Worker Source Code Location

```
packages/
  worker/
    src/
      index.ts          # Entry point: poll Redis, manage check schedules
      checker.ts         # HTTP check execution (undici-based)
      scheduler.ts       # Interval management per monitor
      reporter.ts        # POST results to QStash
    package.json         # Standalone package, not part of Next.js
    Dockerfile           # For Railway deployment
    railway.toml         # Region configuration
```

### Monitor Assignment Strategy

When a monitor is created or updated via the API:

1. The tRPC mutation writes the monitor to PostgreSQL.
2. It serializes the monitor config into Redis: `HSET monitor:{id} url "..." method "GET" interval 60 ...`
3. For each region in the monitor's `regions` array, it adds the monitor ID to the worker's set: `SADD worker:{region}:monitors {monitorId}`
4. When a monitor is deleted or paused, the API removes it from all worker sets: `SREM worker:{region}:monitors {monitorId}` and deletes the Redis hash.

### Check Result Processing Pipeline

When the API receives a check result at `POST /api/webhooks/check-result`:

```
Check result received
  │
  ├─ 1. Insert into `checks` table (time-series)
  │
  ├─ 2. Update `monitors` table:
  │     - last_checked_at = now()
  │     - If status changed → update status, last_status_change_at
  │     - If down → increment consecutive_failures
  │     - If up → reset consecutive_failures to 0
  │
  ├─ 3. Evaluate alert conditions:
  │     - If consecutive_failures >= confirmations_required
  │       AND no active incident for this monitor:
  │       │
  │       ├─ Create incident (auto_generated = true)
  │       ├─ Update affected component status
  │       ├─ Generate AI draft for initial update
  │       ├─ Fire alerts (email, Slack) to linked alert channels
  │       ├─ Notify status page subscribers
  │       └─ Publish to Redis pub/sub: org:{orgId}:events
  │     │
  │     - If monitor was down AND now up:
  │       │
  │       ├─ Resolve active incident
  │       ├─ Generate AI resolution draft
  │       ├─ Update component status to operational
  │       ├─ Fire recovery alerts
  │       └─ Publish to Redis pub/sub
  │
  └─ 4. Update Redis status page cache:
        HSET status_page:{pageId}:components {componentId} "operational"
```

---

## Phase Breakdown (8 Weeks)

### Week 1: Project Scaffolding + Database + Auth

**Day 1-2: Project Setup**
- Initialize Next.js 14+ project with App Router, TypeScript, Tailwind CSS, and ESLint:
  ```bash
  npx create-next-app@latest statusping --typescript --tailwind --app --src-dir
  ```
- Install core dependencies:
  ```bash
  npm i drizzle-orm @neondatabase/serverless @trpc/server @trpc/client @trpc/next @trpc/react-query @tanstack/react-query zod @clerk/nextjs stripe resend @upstash/redis @upstash/qstash openai
  npm i -D drizzle-kit @types/node
  ```
- Configure `src/env.ts` with Zod validation:
  ```typescript
  // Required env vars:
  // DATABASE_URL, CLERK_SECRET_KEY, NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY,
  // STRIPE_SECRET_KEY, STRIPE_WEBHOOK_SECRET, NEXT_PUBLIC_STRIPE_PUBLISHABLE_KEY,
  // RESEND_API_KEY, OPENAI_API_KEY, UPSTASH_REDIS_URL, UPSTASH_REDIS_TOKEN,
  // QSTASH_URL, QSTASH_TOKEN, QSTASH_CURRENT_SIGNING_KEY, QSTASH_NEXT_SIGNING_KEY,
  // NEXT_PUBLIC_APP_URL, WORKER_API_SECRET
  ```
- Set up `drizzle.config.ts` pointing to Neon:
  ```typescript
  import { defineConfig } from "drizzle-kit";
  export default defineConfig({
    schema: "./src/server/db/schema.ts",
    out: "./drizzle",
    dialect: "postgresql",
    dbCredentials: { url: process.env.DATABASE_URL! },
  });
  ```
- Create `src/server/db/index.ts` with Neon serverless driver + Drizzle:
  ```typescript
  import { neon } from "@neondatabase/serverless";
  import { drizzle } from "drizzle-orm/neon-http";
  import * as schema from "./schema";
  export const db = drizzle(neon(process.env.DATABASE_URL!), { schema });
  ```

**Day 3: Database Schema + Migrations**
- Write full schema in `src/server/db/schema.ts` (as defined above).
- Run `npx drizzle-kit push` to create tables.
- Execute raw SQL migration for partitioned `checks` table (`drizzle/0001_create_checks_partitioned.sql`).
- Verify all tables, indexes, and constraints are created correctly.

**Day 4: Clerk Auth + Organization Sync**
- Configure Clerk middleware in `src/middleware.ts`:
  ```typescript
  import { clerkMiddleware, createRouteMatcher } from "@clerk/nextjs/server";
  const isPublicRoute = createRouteMatcher([
    "/", "/sign-in(.*)", "/sign-up(.*)", "/s/(.*)", "/api/webhooks/(.*)",
    "/api/public/(.*)", "/api/cron/(.*)",
  ]);
  export default clerkMiddleware(async (auth, req) => {
    if (!isPublicRoute(req)) await auth.protect();
  });
  ```
- Create Clerk webhook handler at `src/app/api/webhooks/clerk/route.ts` to sync organization creation/updates to the `organizations` table:
  ```typescript
  // On "organization.created" → insert into organizations
  // On "organization.updated" → update organizations
  // On "organization.deleted" → delete (cascades to all child records)
  ```
- Create `src/server/auth/index.ts` helper to extract org context from Clerk session.

**Day 5: tRPC Setup + Organization Router**
- Set up tRPC with Clerk context in `src/server/trpc/trpc.ts`:
  ```typescript
  import { initTRPC, TRPCError } from "@trpc/server";
  import { auth } from "@clerk/nextjs/server";
  // Create context that resolves the current org from Clerk
  // orgProtectedProcedure: ensures user belongs to an org, attaches orgId
  ```
- Create `src/server/trpc/context.ts` with `createContext` function.
- Create `src/server/trpc/routers/_app.ts` with root router.
- Create `src/server/trpc/routers/organization.ts`:
  - `getOrg` -- fetch current org profile, plan, settings
  - `updateSettings` -- update org timezone, default alert channels
- Wire up `src/app/api/trpc/[trpc]/route.ts`.
- Create tRPC client hooks in `src/lib/trpc/client.ts` and server caller in `src/lib/trpc/server.ts`.
- Create `src/app/providers.tsx` with TRPCProvider + ClerkProvider + QueryClientProvider.

---

### Week 2: Monitor CRUD + Worker System

**Day 1-2: Monitor tRPC Router**
- Create `src/server/trpc/routers/monitor.ts`:
  ```typescript
  // monitor.list -- paginated list with status, last_checked_at, sparkline data
  //   SELECT m.*, (
  //     SELECT json_agg(json_build_object('t', c.checked_at, 'ms', c.response_time_ms))
  //     FROM checks c WHERE c.monitor_id = m.id
  //       AND c.checked_at > NOW() - INTERVAL '24 hours'
  //     ORDER BY c.checked_at
  //   ) as sparkline_data FROM monitors m WHERE m.org_id = ?
  //
  // monitor.getById -- full detail + recent checks + response time chart data
  // monitor.create -- validate plan limits (free: 5 monitors, pro: 50, business: unlimited)
  //   - Insert monitor into PG
  //   - Sync to Redis: HSET monitor:{id} + SADD worker:{region}:monitors
  // monitor.update -- update config, re-sync Redis
  // monitor.delete -- remove from PG + Redis
  // monitor.pause -- set is_paused=true, remove from Redis worker sets
  // monitor.resume -- set is_paused=false, add back to Redis worker sets
  ```
- Create `src/server/services/monitor-sync.ts` to handle Redis sync logic:
  ```typescript
  export async function syncMonitorToRedis(monitor: Monitor) {
    const redis = Redis.fromEnv();
    // Store monitor config as Redis hash
    await redis.hset(`monitor:${monitor.id}`, {
      url: monitor.url,
      method: monitor.method,
      headers: JSON.stringify(monitor.headers),
      body: monitor.body || "",
      expectedStatusCodes: JSON.stringify(monitor.expectedStatusCodes),
      timeoutMs: monitor.timeoutMs,
      intervalSeconds: monitor.intervalSeconds,
    });
    // Add to each region's worker set
    for (const region of monitor.regions) {
      await redis.sadd(`worker:${region}:monitors`, monitor.id);
    }
  }
  export async function removeMonitorFromRedis(monitorId: string, regions: string[]) {
    const redis = Redis.fromEnv();
    await redis.del(`monitor:${monitorId}`);
    for (const region of regions) {
      await redis.srem(`worker:${region}:monitors`, monitorId);
    }
  }
  ```

**Day 3-4: Worker Package**
- Create `packages/worker/` as a standalone Node.js package.
- `packages/worker/src/index.ts` -- main entry:
  ```typescript
  const REGION = process.env.WORKER_REGION; // e.g., "us-east"
  const POLL_INTERVAL = 5000; // poll Redis every 5s for monitor list changes
  const schedules = new Map<string, NodeJS.Timeout>();

  async function pollAndSchedule() {
    const monitorIds = await redis.smembers(`worker:${REGION}:monitors`);
    // Add new monitors, remove deleted ones
    for (const id of monitorIds) {
      if (!schedules.has(id)) {
        const config = await redis.hgetall(`monitor:${id}`);
        const interval = parseInt(config.intervalSeconds) * 1000;
        // Execute immediately, then schedule
        executeCheck(id, config);
        schedules.set(id, setInterval(() => executeCheck(id, config), interval));
      }
    }
    // Remove monitors no longer in the set
    for (const [id, timer] of schedules) {
      if (!monitorIds.includes(id)) {
        clearInterval(timer);
        schedules.delete(id);
      }
    }
  }
  setInterval(pollAndSchedule, POLL_INTERVAL);
  ```
- `packages/worker/src/checker.ts` -- HTTP check execution:
  ```typescript
  import { request } from "undici";
  export async function executeHttpCheck(config: MonitorConfig): Promise<CheckResult> {
    const start = performance.now();
    try {
      const response = await request(config.url, {
        method: config.method,
        headers: config.headers,
        body: config.body || undefined,
        headersTimeout: config.timeoutMs,
        bodyTimeout: config.timeoutMs,
        signal: AbortSignal.timeout(config.timeoutMs),
      });
      const elapsed = Math.round(performance.now() - start);
      const isUp = config.expectedStatusCodes.includes(response.statusCode);
      return {
        status: isUp ? "up" : "down",
        statusCode: response.statusCode,
        responseTimeMs: elapsed,
        headers: Object.fromEntries(Object.entries(response.headers).filter(([_, v]) => v !== undefined)),
      };
    } catch (error) {
      const elapsed = Math.round(performance.now() - start);
      return {
        status: "down",
        responseTimeMs: elapsed,
        errorMessage: error instanceof Error ? error.message : "Unknown error",
      };
    }
  }
  ```
- `packages/worker/src/reporter.ts` -- POST results via QStash:
  ```typescript
  import { Client } from "@upstash/qstash";
  const qstash = new Client({ token: process.env.QSTASH_TOKEN! });
  export async function reportCheckResult(monitorId: string, region: string, result: CheckResult) {
    await qstash.publishJSON({
      url: `${process.env.API_URL}/api/webhooks/check-result`,
      body: { monitorId, region, ...result, checkedAt: new Date().toISOString() },
    });
  }
  ```
- Create `packages/worker/Dockerfile`:
  ```dockerfile
  FROM node:20-alpine
  WORKDIR /app
  COPY package*.json ./
  RUN npm ci --production
  COPY src/ ./src/
  CMD ["node", "--import", "tsx", "src/index.ts"]
  ```

**Day 5: Check Result Ingestion Endpoint**
- Create `src/app/api/webhooks/check-result/route.ts`:
  ```typescript
  import { Receiver } from "@upstash/qstash";
  // 1. Verify QStash signature
  // 2. Insert check into partitioned checks table
  // 3. Call monitorStatusEvaluator(monitorId, result)
  ```
- Create `src/server/services/monitor-evaluator.ts`:
  ```typescript
  // Fetches current monitor state, evaluates:
  // - Update consecutive_failures
  // - Determine if status changed (up→down or down→up)
  // - If status changed to down AND consecutive >= confirmationsRequired:
  //     → trigger createAutoIncident()
  //     → trigger fireAlerts()
  //     → update component status
  //     → publish to Redis pub/sub
  // - If status changed to up AND had active incident:
  //     → trigger resolveAutoIncident()
  //     → fire recovery alerts
  //     → update component status
  ```

---

### Week 3: Alert System + Incident Auto-Creation

**Day 1-2: Alert Channels CRUD**
- Create `src/server/trpc/routers/alert-channel.ts`:
  - `alertChannel.list` -- list all channels for org
  - `alertChannel.create` -- validate config by type (email addresses, Slack webhook URL)
  - `alertChannel.update` -- update name, config, enabled
  - `alertChannel.delete`
  - `alertChannel.test` -- send a test alert to verify the channel works
- Create `src/server/trpc/routers/monitor-alert.ts`:
  - `monitorAlert.link` -- link a monitor to an alert channel (insert into junction table)
  - `monitorAlert.unlink` -- remove link
  - `monitorAlert.listForMonitor` -- list channels linked to a monitor

**Day 2-3: Alert Dispatch**
- Create `src/server/services/alert-dispatcher.ts`:
  ```typescript
  export async function dispatchAlerts(
    monitor: Monitor,
    incident: Incident,
    type: "down" | "recovered"
  ) {
    // Fetch linked alert channels for this monitor
    const channels = await db.select()
      .from(monitorAlertChannels)
      .innerJoin(alertChannels, eq(alertChannels.id, monitorAlertChannels.alertChannelId))
      .where(eq(monitorAlertChannels.monitorId, monitor.id));

    for (const channel of channels) {
      switch (channel.alert_channels.type) {
        case "email":
          await sendEmailAlert(channel.alert_channels.config, monitor, incident, type);
          break;
        case "slack":
          await sendSlackAlert(channel.alert_channels.config, monitor, incident, type);
          break;
        case "webhook":
          await sendWebhookAlert(channel.alert_channels.config, monitor, incident, type);
          break;
      }
    }
  }
  ```
- Create `src/server/alerts/email.ts` -- Resend-based email alert:
  ```typescript
  import { Resend } from "resend";
  // Subject: "[StatusPing] Monitor DOWN: {monitor.name}"
  // Body: HTML email with monitor details, failure time, regions affected
  ```
- Create `src/server/alerts/slack.ts` -- Slack webhook:
  ```typescript
  // POST to Slack webhook URL with Block Kit message:
  // - Color-coded attachment (red for down, green for recovery)
  // - Monitor name, URL, failure regions, duration
  // - Link to incident in StatusPing dashboard
  ```

**Day 3-4: Auto-Incident Creation**
- Create `src/server/services/incident-manager.ts`:
  ```typescript
  export async function createAutoIncident(monitor: Monitor, failedRegions: string[]) {
    // 1. Check if there's already an active incident for this monitor's component
    const existingIncident = await db.select().from(incidents)
      .where(and(
        eq(incidents.orgId, monitor.orgId),
        eq(incidents.status, sql`ANY(ARRAY['investigating','identified','monitoring'])`),
        // Check if affected components overlap
      ));
    if (existingIncident.length > 0) return existingIncident[0];

    // 2. Create new incident
    const incident = await db.insert(incidents).values({
      orgId: monitor.orgId,
      title: `${monitor.name} is experiencing issues`,
      status: "investigating",
      severity: "major",
      autoGenerated: true,
      affectedComponentIds: monitor.componentId ? [monitor.componentId] : [],
    }).returning();

    // 3. Update component status
    if (monitor.componentId) {
      await db.update(components)
        .set({ status: "major_outage" })
        .where(eq(components.id, monitor.componentId));
    }

    // 4. Generate AI draft for initial update
    const aiDraft = await generateIncidentDraft({
      monitorName: monitor.name,
      monitorUrl: monitor.url,
      failedRegions,
      failureSince: new Date(),
      type: "investigating",
    });

    // 5. Insert incident update (unapproved, AI-generated)
    await db.insert(incidentUpdates).values({
      incidentId: incident[0].id,
      status: "investigating",
      message: aiDraft,
      aiDraft: aiDraft,
      isAiGenerated: true,
      approved: false,
    });

    return incident[0];
  }

  export async function resolveAutoIncident(monitor: Monitor) {
    // Find active auto-generated incidents for this monitor's component
    // Set status to "resolved", resolvedAt = now()
    // Generate AI resolution draft
    // Update component status to "operational"
  }
  ```

**Day 5: Subscriber Notifications**
- Create `src/server/trpc/routers/subscriber.ts`:
  - `subscriber.subscribe` -- public endpoint, no auth, validates email, sends confirmation
  - `subscriber.confirm` -- confirm via token
  - `subscriber.unsubscribe` -- unsubscribe via token
  - `subscriber.list` -- org-protected, list subscribers for a status page
- Create `src/server/services/subscriber-notifier.ts`:
  ```typescript
  export async function notifySubscribers(statusPageId: string, incident: Incident, update: IncidentUpdate) {
    const subs = await db.select().from(subscribers)
      .where(and(eq(subscribers.statusPageId, statusPageId), eq(subscribers.confirmed, true)));
    // Batch send via Resend (up to 100 per batch)
    // Email includes: incident title, status, update message, link to status page
  }
  ```

---

### Week 4: Dashboard UI + Monitor Management

**Day 1: Layout + Navigation**
- Create `src/app/(dashboard)/layout.tsx` -- authenticated layout with:
  - Sidebar: Monitors, Incidents, Status Pages, Alert Channels, Settings
  - Top bar: org name, Clerk UserButton, plan badge
  - Active incident banner (red bar at top if any active incidents)
- Create `src/app/(dashboard)/page.tsx` -- redirect to `/monitors`
- Install UI dependencies: `npm i lucide-react @radix-ui/react-dialog @radix-ui/react-dropdown-menu @radix-ui/react-tabs recharts date-fns`
- Set up `src/components/ui/` with common components: Button, Card, Badge, Input, Select, Dialog, Toast, Skeleton

**Day 2-3: Monitor Dashboard Page**
- Create `src/app/(dashboard)/monitors/page.tsx`:
  ```typescript
  // Grid view of monitors
  // Each card shows:
  //   - Status dot (green/yellow/red/gray)
  //   - Monitor name + URL (truncated)
  //   - Sparkline (last 24h response time, using Recharts AreaChart)
  //   - Uptime % for last 30 days
  //   - Last check time (relative: "2m ago")
  //   - Actions dropdown: Edit, Pause, Delete
  // Filters: status (up/down/paused/all), search by name
  // "Add Monitor" button → dialog or separate page
  ```
- Create `src/components/monitors/monitor-card.tsx` -- individual monitor card with sparkline
- Create `src/components/monitors/sparkline.tsx` -- lightweight Recharts AreaChart (no axes, just the line/fill)
- Create `src/components/monitors/uptime-bar.tsx` -- 30-day uptime bar (thin horizontal bar, green/red segments)

**Day 3-4: Create/Edit Monitor Forms**
- Create `src/app/(dashboard)/monitors/new/page.tsx`:
  - Form fields: name, URL, method (GET/POST/PUT/HEAD), headers (key-value pairs), request body (for POST/PUT), expected status codes (multi-select), timeout, check interval (dropdown: 1m/2m/5m/10m with plan limits), regions (multi-checkbox: 5 regions), component assignment (optional dropdown)
  - Plan limit validation: free plan shows 5-min minimum interval as disabled with upgrade prompt
  - Alert channel selection: link to existing channels or create new ones
  - "Test Now" button: fires a single check from all selected regions and shows results inline
- Create `src/app/(dashboard)/monitors/[id]/page.tsx` -- monitor detail:
  - Large response time chart (Recharts LineChart) with time range selector: 1h, 24h, 7d, 30d
  - Uptime percentage with decimal precision (e.g., 99.94%)
  - Check log table: last 50 checks with columns: Time, Region, Status, Response Time, Status Code
  - Configuration summary panel
  - Linked incidents list
  - "Edit" and "Pause/Resume" buttons
- Create `src/app/(dashboard)/monitors/[id]/edit/page.tsx` -- edit form (same as create, pre-filled)

**Day 5: Response Time Chart Queries**
- Create optimized tRPC procedures for chart data:
  ```typescript
  // monitor.getResponseTimeChart
  // For 1h: raw checks (60 points max at 1-min interval)
  // For 24h: bucket into 5-min averages (288 points)
  //   SELECT date_trunc('minute', checked_at) - (EXTRACT(MINUTE FROM checked_at)::int % 5 || ' minutes')::interval as bucket,
  //          AVG(response_time_ms)::int as avg_ms, MIN(response_time_ms) as min_ms, MAX(response_time_ms) as max_ms
  //   FROM checks WHERE monitor_id = ? AND checked_at > NOW() - INTERVAL '24 hours'
  //   GROUP BY bucket ORDER BY bucket
  // For 7d: bucket into 1-hour averages (168 points)
  // For 30d: bucket into 4-hour averages (180 points)
  ```
- Create `src/server/trpc/routers/check.ts`:
  - `check.getResponseTimeData` -- bucketed response time series
  - `check.getRecentChecks` -- paginated check log for detail page
  - `check.getUptimePercentage` -- calculate uptime % over time range:
    ```sql
    SELECT
      COUNT(*) FILTER (WHERE status = 'up') * 100.0 / COUNT(*) as uptime_pct
    FROM checks
    WHERE monitor_id = ? AND checked_at > NOW() - INTERVAL '30 days'
    ```

---

### Week 5: Status Page + Editor

**Day 1-2: Status Page CRUD + Editor**
- Create `src/server/trpc/routers/status-page.ts`:
  - `statusPage.get` -- get status page with components
  - `statusPage.create` -- validate slug uniqueness, plan limits (free: 1 page)
  - `statusPage.update` -- update title, description, branding
  - `statusPage.delete`
- Create `src/server/trpc/routers/component.ts`:
  - `component.list` -- list components for a status page, ordered by sort_order
  - `component.create` -- create component, assign to status page
  - `component.update` -- update name, description, group, sort_order
  - `component.reorder` -- batch update sort_order
  - `component.delete`
- Create `src/app/(dashboard)/status-pages/page.tsx` -- list of status pages
- Create `src/app/(dashboard)/status-pages/[id]/edit/page.tsx` -- status page editor:
  - Split view: editor on left, live preview on right
  - Editor sections:
    - General: title, description, slug
    - Branding: primary color (color picker), background color, text color, logo upload, favicon upload
    - Components: drag-to-reorder list with inline edit, group headers, add/remove
    - Subscribers: count of confirmed subscribers, download list
  - Preview: real-time rendering of what the public page will look like
  - "View Live Page" link

**Day 3-4: Public Status Page Rendering**
- Create `src/app/s/[slug]/page.tsx` -- public status page (ISR, revalidate: 10):
  ```typescript
  export const revalidate = 10; // ISR: regenerate every 10 seconds

  export default async function StatusPagePublic({ params }: { params: { slug: string } }) {
    // 1. Fetch status page config from Redis cache (fallback to DB)
    // 2. Fetch component statuses from Redis cache
    // 3. Fetch active incidents for this org
    // 4. Fetch last 90 days of uptime data per component
    // 5. Render page
  }
  ```
- Layout structure:
  - Header: logo, title, description
  - Overall status banner: "All Systems Operational" (green) or "Partial System Outage" (yellow/red)
  - Component list: each component shows name, current status badge, 90-day uptime bar
  - Active incidents section: chronological timeline of active incidents with updates
  - Past incidents: last 14 days of resolved incidents
  - Footer: "Subscribe to Updates" button, "Powered by StatusPing" link
- Create `src/components/status-page/uptime-history-bar.tsx` -- 90-day bar chart where each day is a thin vertical segment (green=100%, yellow=partial, red=outage)
- Create `src/components/status-page/incident-timeline.tsx` -- chronological incident updates
- Create `src/components/status-page/subscribe-form.tsx` -- email subscription form

**Day 4-5: Redis Cache Layer for Status Pages**
- Create `src/server/services/status-page-cache.ts`:
  ```typescript
  export async function updateStatusPageCache(statusPageId: string) {
    const redis = Redis.fromEnv();
    const page = await db.query.statusPages.findFirst({
      where: eq(statusPages.id, statusPageId),
      with: { components: true, organization: true },
    });
    // Cache full page config
    await redis.set(`status_page:${page.slug}`, JSON.stringify(page), { ex: 300 });
    // Cache individual component statuses
    for (const component of page.components) {
      await redis.hset(`status_page:${page.slug}:components`, {
        [component.id]: component.status,
      });
    }
  }
  ```
- Invalidate cache on: component status change, incident create/update/resolve, status page edit
- Create `src/server/services/uptime-calculator.ts`:
  ```typescript
  export async function getUptimeHistory(monitorId: string, days: number): Promise<DayUptime[]> {
    // Returns array of { date, uptimePercent, hasIncident } for each day
    const result = await db.execute(sql`
      SELECT DATE(checked_at) as day,
             COUNT(*) FILTER (WHERE status = 'up') * 100.0 / COUNT(*) as uptime_pct,
             COUNT(*) FILTER (WHERE status = 'down') > 0 as had_downtime
      FROM checks
      WHERE monitor_id = ${monitorId}
        AND checked_at > NOW() - INTERVAL '${days} days'
      GROUP BY day
      ORDER BY day
    `);
    return result;
  }
  ```

---

### Week 6: AI Incident Drafting + Incident Management UI

**Day 1-2: AI Incident Draft Pipeline**
- Create `src/server/ai/generate-incident-draft.ts`:
  ```typescript
  import OpenAI from "openai";
  const openai = new OpenAI();

  type IncidentDraftInput = {
    monitorName: string;
    monitorUrl: string;
    failedRegions: string[];
    statusCode?: number;
    errorMessage?: string;
    failureDurationMinutes: number;
    incidentStatus: "investigating" | "identified" | "monitoring" | "resolved";
    previousUpdates?: { status: string; message: string; postedAt: string }[];
    affectedComponents?: string[];
  };

  export async function generateIncidentDraft(input: IncidentDraftInput): Promise<string> {
    const systemPrompt = `You are a professional incident communication writer for a SaaS status page.
Write clear, concise, empathetic incident updates for customers.
Use professional but human tone. Never speculate about root cause unless explicitly told.
Do not use overly technical language. Focus on impact and next steps.
Keep updates to 2-3 sentences maximum.`;

    const userPrompt = buildPromptFromInput(input);
    // For "investigating":
    //   "We are investigating reports of issues with {monitorName}. HTTP checks from {regions}
    //    are returning {statusCode/error} since {duration} ago. We will provide an update shortly."
    // For "resolved":
    //   "The issue affecting {monitorName} has been resolved. Services were impacted for approximately
    //    {duration}. All monitoring checks are now returning healthy responses."

    const response = await openai.chat.completions.create({
      model: "gpt-4o",
      messages: [
        { role: "system", content: systemPrompt },
        { role: "user", content: userPrompt },
      ],
      max_tokens: 300,
      temperature: 0.3, // Low temperature for consistent, professional output
    });
    return response.choices[0].message.content || "";
  }
  ```
- Create `src/server/ai/prompts.ts` -- prompt templates for different incident statuses
- Test with sample data: verify tone, length, accuracy for each status type
- Add rate limiting: max 10 AI drafts per incident, max 50 per org per day

**Day 2-3: Incident Management tRPC Router**
- Create `src/server/trpc/routers/incident.ts`:
  ```typescript
  // incident.listActive -- active incidents for current org
  // incident.listAll -- paginated, with filters (status, severity, date range)
  // incident.getById -- full incident with all updates
  // incident.create -- manual incident creation (not auto-generated)
  //   Input: title, severity, affected components, initial message (optional)
  // incident.update -- update title, severity, affected components
  // incident.addUpdate -- manually add an update
  //   Input: status, message, isInternal
  // incident.approveAiDraft -- approve AI-generated update (sets approved=true)
  // incident.editAiDraft -- edit AI draft text, then approve
  // incident.regenerateDraft -- re-generate AI draft for a specific update
  //   Calls generateIncidentDraft with current context
  // incident.resolve -- set status to resolved, trigger AI resolution draft
  ```

**Day 4-5: Incident UI**
- Create `src/app/(dashboard)/incidents/page.tsx` -- incident list:
  - Tabs: Active | Resolved | All
  - Each row: severity badge, title, status, duration, affected components, created time
  - "Create Incident" button
- Create `src/app/(dashboard)/incidents/[id]/page.tsx` -- incident detail:
  - Left panel (2/3 width): incident timeline
    - Chronological list of updates, each showing: timestamp, status badge, message, author
    - Internal notes styled differently (gray background, "Internal" badge)
    - Add update form at the bottom: status dropdown, message textarea, internal toggle
  - Right panel (1/3 width): AI Draft Panel
    - Shows pending AI-generated drafts with "Approve", "Edit & Approve", "Regenerate" buttons
    - Draft preview with diff highlighting if edited
    - "Generate New Draft" button to create an AI draft for the current incident state
  - Top bar: incident title (editable), severity selector, affected components (multi-select)
  - Resolution section: "Mark as Resolved" button, auto-triggers AI resolution draft
- Create `src/components/incidents/ai-draft-panel.tsx` -- AI draft approval/edit component
- Create `src/components/incidents/incident-timeline.tsx` -- chronological timeline
- Create `src/components/incidents/create-incident-dialog.tsx` -- manual incident creation modal

---

### Week 7: Real-Time SSE + Billing + Settings

**Day 1: Server-Sent Events for Live Updates**
- Create `src/app/api/sse/[orgId]/route.ts` -- SSE endpoint:
  ```typescript
  export async function GET(req: Request, { params }: { params: { orgId: string } }) {
    const encoder = new TextEncoder();
    const stream = new ReadableStream({
      async start(controller) {
        const redis = Redis.fromEnv();
        // Subscribe to org events channel
        // On message: controller.enqueue(encoder.encode(`data: ${JSON.stringify(event)}\n\n`))
        // Event types: monitor_status_changed, incident_created, incident_updated, incident_resolved
      },
    });
    return new Response(stream, {
      headers: {
        "Content-Type": "text/event-stream",
        "Cache-Control": "no-cache",
        Connection: "keep-alive",
      },
    });
  }
  ```
- **SSE Connection Lifecycle Details**:
  - Connections have a **30-minute server-side timeout** — after 30 minutes the server closes the stream and the client must reconnect
  - Client reconnects use **exponential backoff**: initial delay 1s, multiplied by 2 on each retry, capped at 30s, with +/-25% jitter to prevent thundering herd
  - The SSE endpoint sends a `ping` event every 15 seconds as a keepalive; if the client receives no data for 45 seconds it considers the connection dead and reconnects
  - **Buffer overflow handling**: if the Redis pub/sub message queue for a channel exceeds 100 pending messages (e.g., during a burst of check results), the oldest events are dropped and a `sync` event is sent to tell the client to refetch full state via the REST API instead of relying on incremental updates
  - The `Last-Event-ID` header is respected on reconnect — the server replays missed events from a short-lived Redis list (last 50 events, 5-minute TTL)
- Create `src/hooks/use-realtime.ts` -- React hook for SSE subscription:
  ```typescript
  export function useRealtime(orgId: string) {
    // Connect to SSE endpoint
    // On "monitor_status_changed" → invalidate monitor list query
    // On "incident_created" → invalidate incidents query, show toast
    // On "incident_updated" → invalidate specific incident query
    // On reconnect: use exponential backoff (1s base, 30s max, jitter)
    // On "sync" event: refetch all data via tRPC instead of incremental
    // Return: connected status, latest event
  }
  ```
- Wire into dashboard layout: `useRealtime` in the layout component to provide live updates across all pages

**Day 2-3: Stripe Billing**
- Create Stripe products and prices:
  ```
  Product: StatusPing Pro   → Price: $29/month (price_pro_monthly)
  Product: StatusPing Business → Price: $79/month (price_business_monthly)
  ```
- Create `src/server/billing/stripe.ts`:
  ```typescript
  export async function createCheckoutSession(orgId: string, priceId: string) {
    // Create Stripe checkout session with org metadata
    // success_url: /settings/billing?success=true
    // cancel_url: /settings/billing?canceled=true
  }
  export async function createPortalSession(customerId: string) {
    // Create Stripe billing portal session for managing subscription
  }
  ```
- Create `src/server/trpc/routers/billing.ts`:
  - `billing.getSubscription` -- current plan details, next billing date, usage counts
  - `billing.createCheckoutSession` -- initiate upgrade flow
  - `billing.createPortalSession` -- open Stripe customer portal
- Create `src/app/api/webhooks/stripe/route.ts`:
  ```typescript
  // Handle events:
  // checkout.session.completed → update org plan + stripeCustomerId + stripeSubscriptionId
  // customer.subscription.updated → update org plan based on price ID
  // customer.subscription.deleted → downgrade org to free plan
  // invoice.payment_failed → send email warning, don't immediately downgrade
  ```
- Create `src/server/services/plan-limits.ts`:
  ```typescript
  export const PLAN_LIMITS = {
    free:     { monitors: 5,  minInterval: 300, statusPages: 1, dataRetentionDays: 7,   aiDrafts: false, slackAlerts: false },
    pro:      { monitors: 50, minInterval: 30,  statusPages: 1, dataRetentionDays: 90,  aiDrafts: true,  slackAlerts: true },
    business: { monitors: -1, minInterval: 30,  statusPages: 5, dataRetentionDays: 365, aiDrafts: true,  slackAlerts: true },
  } as const;
  export function checkPlanLimit(org: Organization, resource: string, currentCount: number): boolean { ... }
  ```
- Enforce limits in monitor.create, statusPage.create, and incident AI draft generation

**Day 3-4: Settings Pages**
- Create `src/app/(dashboard)/settings/page.tsx` -- redirect to `/settings/general`
- Create `src/app/(dashboard)/settings/general/page.tsx`:
  - Organization name, slug, timezone
  - Delete organization (with confirmation)
- Create `src/app/(dashboard)/settings/billing/page.tsx`:
  - Current plan card with usage stats (monitors used / limit, data retention period)
  - Plan comparison table (Free / Pro / Business)
  - Upgrade button → Stripe checkout
  - "Manage Subscription" → Stripe billing portal
- Create `src/app/(dashboard)/settings/alert-channels/page.tsx`:
  - List of configured alert channels
  - Add channel: type selector → config form (email addresses / Slack webhook URL)
  - Test button per channel
  - Enable/disable toggle

**Day 5: Cron Jobs**
- Create `src/app/api/cron/partitions/route.ts` -- monthly: create next partition, drop expired:
  ```typescript
  // Vercel Cron: schedule: "0 0 1 * *" (1st of each month at midnight)
  export async function GET() {
    // Create partition for month+2
    // Drop partitions older than oldest active plan retention
    // Log results
  }
  ```
- Create `src/app/api/cron/data-retention/route.ts` -- daily: delete checks beyond plan retention:
  ```typescript
  // Vercel Cron: schedule: "0 2 * * *" (daily at 2am)
  // For each org, delete checks older than plan's dataRetentionDays
  // DELETE FROM checks WHERE monitor_id IN (SELECT id FROM monitors WHERE org_id = ?)
  //   AND checked_at < NOW() - INTERVAL '? days'
  ```
- Create `src/app/api/cron/monitor-health/route.ts` -- every 5 min: detect stale monitors:
  ```typescript
  // Find monitors where last_checked_at is >3x their interval
  // This means the worker isn't checking them → alert the StatusPing team
  ```
- Configure Vercel crons in `vercel.json`:
  ```json
  {
    "crons": [
      { "path": "/api/cron/partitions", "schedule": "0 0 1 * *" },
      { "path": "/api/cron/data-retention", "schedule": "0 2 * * *" },
      { "path": "/api/cron/monitor-health", "schedule": "*/5 * * * *" }
    ]
  }
  ```

---

### Week 8: Polish, Testing, Deploy, Launch

**Day 1: Onboarding Flow**
- Create `src/app/(dashboard)/onboarding/page.tsx` -- shown after first sign-up:
  - Step 1: Create your first monitor (URL input, we auto-detect name from page title)
  - Step 2: Set up an alert channel (email pre-filled from Clerk, or Slack webhook)
  - Step 3: Create your status page (auto-generate slug from org name)
  - Step 4: Done! Link to dashboard with confetti animation
- Create `src/server/services/onboarding.ts` -- track onboarding completion state in org settings

**Day 2: UI Polish + Loading States**
- Add loading skeletons to all pages (monitor list, incident list, status page editor)
- Add empty states with illustrations: "No monitors yet", "No incidents -- that's a good thing!"
- Toast notifications for mutations (monitor created, alert channel tested, incident resolved)
- Error boundaries with retry buttons
- Mobile responsive adjustments for dashboard sidebar (collapsible on mobile)
- Dark mode support (Tailwind `dark:` classes, system preference detection)

**Day 3: Edge Cases + Error Handling**
- Handle worker connectivity loss: if a worker stops reporting, don't mark monitors as down
- Handle Slack webhook failures: retry 3x with exponential backoff, then disable channel and alert org admin
- Handle OpenAI API failures: fall back to template-based incident drafts:
  ```typescript
  const FALLBACK_TEMPLATES = {
    investigating: "We are currently investigating issues with {monitorName}. We will provide updates as we have more information.",
    identified: "The issue with {monitorName} has been identified. Our team is working on a fix.",
    monitoring: "A fix has been applied for {monitorName}. We are monitoring the results.",
    resolved: "The issue with {monitorName} has been resolved. All systems are operational.",
  };
  ```
- **Partition creation error recovery strategy**:
  1. The cron job (`/api/cron/partitions/route.ts`) attempts to create the next month's partition via `CREATE TABLE IF NOT EXISTS checks_YYYY_MM PARTITION OF checks ...`
  2. If the `CREATE TABLE` fails (e.g., name collision, permission error, disk quota), catch the error, log it to Sentry with full context (partition name, date range, error message), and send an urgent alert to the ops Slack channel
  3. **Fallback**: if the partition for the current month does not exist when a check result arrives, the check result ingestion endpoint catches the "no partition for value" error, creates the missing partition inline with a `CREATE TABLE IF NOT EXISTS` statement inside a PG advisory lock (`pg_advisory_xact_lock(hash('create_partition_YYYY_MM'))`) to prevent concurrent creation attempts, then retries the insert
  4. A daily health-check query verifies partitions exist for the current month and the next 2 months; missing partitions are auto-created and logged as warnings
- Handle partition creation failures: alert via logging, fall back to default partition
- Rate limit public endpoints: subscriber.subscribe (5/min per IP), status page (100/min per IP)

**Day 4: Testing**
- Unit tests for critical business logic:
  - `monitor-evaluator.test.ts` -- consecutive failure counting, status transitions
  - `plan-limits.test.ts` -- plan limit enforcement
  - `incident-manager.test.ts` -- auto-incident creation, deduplication, resolution
  - `uptime-calculator.test.ts` -- uptime percentage calculation accuracy
  - `generate-incident-draft.test.ts` -- prompt construction, fallback templates
- Integration tests:
  - `check-result-pipeline.test.ts` -- end-to-end: check result → DB insert → status update → alert dispatch
  - `billing-webhook.test.ts` -- Stripe webhook → plan change → limit enforcement
- Worker tests:
  - `checker.test.ts` -- HTTP check with various responses (200, 500, timeout, DNS failure, SSL error)
  - `scheduler.test.ts` -- interval management, add/remove monitors

**Day 5: Deployment + Launch Prep**
- Deploy Next.js app to Vercel:
  - Set all environment variables
  - Configure custom domain (if any)
  - Enable Vercel Crons
  - Set up Vercel Analytics
- Deploy workers to Railway:
  - Create 5 Railway services (one per region: us-east, us-west, eu-west, eu-central, ap-southeast)
  - Set WORKER_REGION, API_URL, QSTASH_TOKEN, UPSTASH_REDIS_URL, UPSTASH_REDIS_TOKEN per service
  - Configure Railway region selection for each service
  - Verify workers are polling Redis and executing checks
- Set up Neon database:
  - Create production database
  - Run migrations + partition creation
  - Enable connection pooling
- Set up Sentry error tracking for both Next.js app and workers
- Set up Upstash Redis and QStash in production
- Smoke test: create a monitor, watch it check, trigger a downtime (point at an intentionally broken endpoint), verify full pipeline: check → alert → incident → AI draft → status page
- Verify billing flow end-to-end with Stripe test mode

---

## Critical Files

| File | Description |
|------|-------------|
| `src/server/db/schema.ts` | Complete Drizzle schema: organizations, monitors, checks (partitioned), components, status_pages, incidents, incident_updates, alert_channels, subscribers. All types, relations, and indexes. |
| `src/server/services/monitor-evaluator.ts` | Core business logic: processes each check result, manages consecutive failure counting, determines status transitions, triggers auto-incident creation and resolution, fires alerts, and invalidates caches. Every check result flows through this file. |
| `packages/worker/src/checker.ts` | HTTP check execution engine used by all 5 regional workers. Uses `undici` for low-overhead requests, handles timeouts, DNS failures, SSL errors, and status code matching. Reports results via QStash. |
| `src/server/ai/generate-incident-draft.ts` | OpenAI GPT-4o integration for generating incident updates. Builds context-aware prompts from monitor data, failure patterns, and prior updates. Includes fallback templates for API failures. |
| `src/app/s/[slug]/page.tsx` | Public status page rendered via ISR. Reads from Redis cache for fast global delivery. Shows component statuses, 90-day uptime bars, active incidents with timeline, and subscriber form. The external-facing page that customers see during outages. |
| `src/server/services/alert-dispatcher.ts` | Multi-channel alert routing: determines which channels to fire for a given monitor event (down/recovered), sends email via Resend, Slack via webhook, and generic webhooks. Handles retry logic and channel disable on persistent failures. |

---

## Verification

### Unit Tests

| Test File | What It Covers |
|-----------|---------------|
| `src/server/services/__tests__/monitor-evaluator.test.ts` | Consecutive failure counting (reset on success, increment on failure). Status transition logic (unknown→up, up→down, down→up). Deduplication of incidents when multiple regions fail. Threshold behavior (fire only when failures >= confirmationsRequired). |
| `src/server/services/__tests__/plan-limits.test.ts` | Free plan: max 5 monitors, 5-min minimum interval, no AI drafts. Pro plan: max 50 monitors, 30-sec interval, AI drafts enabled. Business plan: unlimited monitors. Upgrade scenarios: limit increase applies immediately. Downgrade scenarios: existing monitors kept but new ones blocked. |
| `src/server/services/__tests__/incident-manager.test.ts` | Auto-incident creation with correct title, severity, affected components. Deduplication: does not create duplicate incidents for same component. Auto-resolution when monitor recovers. Multiple monitors for same component: single incident, multiple affected. |
| `src/server/ai/__tests__/generate-incident-draft.test.ts` | Prompt construction includes all required fields (monitor name, URL, regions, error codes). Output is under 300 tokens. Fallback template used on OpenAI API error. Different prompts for different statuses (investigating vs resolved). |
| `packages/worker/src/__tests__/checker.test.ts` | 200 response → status "up". 500 response → status "down". Timeout → status "down" with error message. DNS failure → status "down". Custom expected status codes (e.g., 301 is "up"). Response time measurement accuracy (within 50ms tolerance). |

### Integration Tests

| Test File | What It Covers |
|-----------|---------------|
| `src/server/services/__tests__/check-result-pipeline.integration.test.ts` | End-to-end: POST check result → inserted in checks table → monitor status updated → alert dispatched (mock email/Slack) → incident created → component status updated → Redis cache invalidated → SSE event published. Uses test database. |
| `src/app/api/webhooks/__tests__/stripe.integration.test.ts` | Simulate Stripe webhooks: checkout.session.completed → org plan updated to "pro". subscription.deleted → org downgraded to "free". Verify plan limits enforced after downgrade. |
| `src/server/trpc/routers/__tests__/monitor.integration.test.ts` | Create monitor → verify plan limits → verify Redis sync (monitor config hash + worker set). Delete monitor → verify Redis cleanup. Pause/resume → verify worker set membership. |

### E2E Tests (Playwright)

| Test File | What It Covers |
|-----------|---------------|
| `e2e/onboarding.spec.ts` | Sign up → create first monitor → add email alert → create status page → verify monitor appears on dashboard with "unknown" status. |
| `e2e/monitor-lifecycle.spec.ts` | Create monitor → wait for checks to appear → view response time chart → pause monitor → verify no new checks → resume → verify checks restart. |
| `e2e/incident-flow.spec.ts` | (Uses mock worker) Trigger downtime → verify incident auto-created → verify AI draft appears in incident view → approve draft → verify update appears on public status page → trigger recovery → verify incident resolved. |
| `e2e/status-page.spec.ts` | Create status page → add components → verify public page renders at /s/[slug] → subscribe with email → verify confirmation email sent (mock). |
| `e2e/billing.spec.ts` | Free user tries to create 6th monitor → sees upgrade prompt → completes Stripe checkout (test mode) → verifies plan updated → can now create 50 monitors. |

### Manual QA Checklist

- [ ] Create an account, go through onboarding, create first monitor pointing to a known-good URL (e.g., https://httpstat.us/200)
- [ ] Verify monitor shows "up" status within 2 minutes, sparkline begins populating
- [ ] Create a monitor pointing to https://httpstat.us/500 -- verify it transitions to "down" after confirmationsRequired checks
- [ ] Verify email alert is received when monitor goes down
- [ ] Verify Slack alert is received (if configured) with correct formatting
- [ ] Verify auto-incident is created with AI-drafted investigating update
- [ ] Approve the AI draft, verify it appears on the public status page
- [ ] Change the monitor URL to https://httpstat.us/200 -- verify recovery alert and auto-resolution
- [ ] Test the status page in mobile viewport: verify responsive layout
- [ ] Test upgrade from Free to Pro via Stripe: verify monitor limit increases, AI drafts become available
- [ ] Verify data retention: free plan data older than 7 days is cleaned up (check after cron runs)
- [ ] Test concurrent failures: create 3 monitors for same component, all pointing to a failing endpoint. Verify only 1 incident is created (not 3)
- [ ] Test manual incident creation: create incident, add manual updates, resolve. Verify timeline on status page.
- [ ] Load test: create 50 monitors (Pro plan) and verify all are being checked at the correct interval across all regions
- [ ] Verify SSE: open dashboard in 2 browser tabs. Trigger a status change. Verify both tabs update without manual refresh.
