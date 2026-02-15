# 8. ShipFlag — Feature Flag Management

**MVP Scope:** Boolean + string flags with percentage rollout (sticky via murmur3 hash), user ID targeting, 3 environments (dev/staging/prod), JavaScript + React + Node.js SDKs, dashboard with flag management, basic audit log, real-time updates via SSE, and Stripe billing.

---

## Tech Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Framework | Next.js 14+ App Router | Consistent with DriftLog/SnipVault stack; server components for dashboard, API routes for SDK endpoints |
| API layer | tRPC (dashboard) + REST (SDK endpoints) | tRPC for type-safe dashboard mutations; plain REST for SDK consumption (SDKs are language-agnostic) |
| ORM | Drizzle ORM + Neon Postgres | Same as DriftLog; serverless-compatible, type-safe schema |
| Auth (dashboard) | Clerk | Organization support built-in, role management, same pattern as DriftLog |
| Auth (SDK) | API key in `Authorization` header | Separate client/server keys per environment; keys hashed with SHA-256 before storage |
| Cache | Upstash Redis | Flag rule cache for sub-10ms API responses; invalidated on every flag change |
| Real-time | SSE (Server-Sent Events) | Simpler than WebSockets, works through CDNs/proxies, sufficient for one-way flag-push |
| Percentage hashing | murmur3(flagKey + userId) % 100 | Deterministic, fast, uniform distribution, sticky per user per flag |
| Billing | Stripe Checkout + Customer Portal | Same pattern as DriftLog; usage metered by MAU count |
| Email | Resend | Team invitations, billing receipts |
| Hosting | Vercel | Edge-compatible API routes for SSE, same as DriftLog |

### Evaluation Engine Design

The flag evaluation engine is the core of the product. Two modes exist:

**Client-side (evaluated on server, results pushed to SDK):**
```
Client SDK ---GET /api/sdk/flags---> API (evaluates all flags for context) ---> returns evaluated results
Client SDK <---SSE stream---------- API pushes updates when flags change
```

**Server-side (rules downloaded, evaluated locally in-process):**
```
Server SDK ---GET /api/sdk/rules---> API returns raw rules for environment
Server SDK evaluates locally per flag check (zero network latency)
Server SDK <---SSE stream----------- API pushes rule updates when flags change
```

### Percentage Rollout Algorithm (Pseudocode)

```typescript
import { murmurhash3_32_gc } from "murmurhash";

function evaluatePercentageRollout(
  flagKey: string,
  userId: string,
  percentage: number // 0-100
): boolean {
  // Concatenate flagKey + userId for per-flag stickiness
  const hashInput = `${flagKey}:${userId}`;
  // murmur3 produces a 32-bit unsigned integer
  const hash = murmurhash3_32_gc(hashInput, 0); // seed = 0
  // Map to 0-99 bucket
  const bucket = hash % 100;
  // User is "in" if their bucket is below the percentage threshold
  return bucket < percentage;
}
```

This guarantees:
- Same user always gets the same result for the same flag (sticky)
- Different flags produce different bucketing (changing one flag's percentage does not affect another)
- Uniform distribution across the 0-99 range

### Flag Rule Evaluation Order

```typescript
function evaluateFlag(
  flag: Flag,
  envConfig: FlagEnvironmentConfig,
  context: { userId?: string; attributes?: Record<string, unknown> }
): Variation {
  // 1. Kill switch: if environment-level toggle is OFF, return default
  if (!envConfig.enabled) {
    return flag.variations.find(v => v.key === envConfig.defaultVariationKey);
  }

  // 2. Walk rules in order (first match wins)
  for (const rule of envConfig.rules) {
    // 2a. User ID targeting: exact match
    if (rule.userIds?.length > 0) {
      if (context.userId && rule.userIds.includes(context.userId)) {
        return flag.variations.find(v => v.key === rule.variationKey);
      }
      // If rule is user-ID-only and user not in list, skip to next rule
      if (!rule.percentage && !rule.conditions?.length) continue;
    }

    // 2b. Attribute conditions (AND logic within a rule)
    if (rule.conditions?.length > 0) {
      const allMatch = rule.conditions.every(cond =>
        evaluateCondition(cond, context.attributes)
      );
      if (!allMatch) continue;
    }

    // 2c. Percentage rollout
    if (rule.percentage !== undefined && rule.percentage !== null) {
      if (!context.userId) continue; // Can't hash without userId
      const inRollout = evaluatePercentageRollout(
        flag.key,
        context.userId,
        rule.percentage
      );
      if (inRollout) {
        return flag.variations.find(v => v.key === rule.variationKey);
      }
      continue; // Not in percentage, skip to next rule
    }

    // 2d. If rule has no percentage and conditions passed, it's a match
    return flag.variations.find(v => v.key === rule.variationKey);
  }

  // 3. No rules matched: return default variation
  return flag.variations.find(v => v.key === envConfig.defaultVariationKey);
}
```

### SSE Streaming Implementation

```typescript
// src/app/api/sdk/stream/route.ts
export async function GET(req: Request) {
  const apiKey = req.headers.get("authorization")?.replace("Bearer ", "");
  const env = await authenticateApiKey(apiKey); // validates & returns environment
  if (!env) return new Response("Unauthorized", { status: 401 });

  const encoder = new TextEncoder();
  const stream = new ReadableStream({
    start(controller) {
      // Send initial heartbeat
      controller.enqueue(encoder.encode(": heartbeat\n\n"));

      // Subscribe to Redis pub/sub channel for this environment
      const channel = `flags:${env.id}`;
      const subscriber = createRedisSubscriber();
      subscriber.subscribe(channel, (message: string) => {
        // message is JSON: { type: "flag.updated", flagKey: "...", timestamp: "..." }
        controller.enqueue(
          encoder.encode(`event: flag.updated\ndata: ${message}\n\n`)
        );
      });

      // Heartbeat every 30s to keep connection alive
      const heartbeat = setInterval(() => {
        controller.enqueue(encoder.encode(": heartbeat\n\n"));
      }, 30_000);

      // Cleanup on close
      req.signal.addEventListener("abort", () => {
        clearInterval(heartbeat);
        subscriber.unsubscribe(channel);
        subscriber.disconnect();
        controller.close();
      });
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

### API Key Authentication Middleware

```typescript
// src/server/auth/api-key.ts
import { createHash } from "crypto";
import { eq } from "drizzle-orm";
import { db } from "@/server/db";
import { environments } from "@/server/db/schema";

export function hashApiKey(rawKey: string): string {
  return createHash("sha256").update(rawKey).digest("hex");
}

export function generateApiKey(prefix: "sf_client" | "sf_server"): {
  raw: string;
  hashed: string;
} {
  const random = crypto.randomUUID().replace(/-/g, "");
  const raw = `${prefix}_${random}`;
  return { raw, hashed: hashApiKey(raw) };
}

export async function authenticateApiKey(
  rawKey: string | undefined | null
): Promise<{ env: Environment; type: "client" | "server" } | null> {
  if (!rawKey) return null;

  const hashed = hashApiKey(rawKey);
  const isClient = rawKey.startsWith("sf_client_");
  const isServer = rawKey.startsWith("sf_server_");

  if (!isClient && !isServer) return null;

  const column = isClient
    ? environments.apiKeyClientHash
    : environments.apiKeyServerHash;

  const [env] = await db
    .select()
    .from(environments)
    .where(eq(column, hashed))
    .limit(1);

  if (!env) return null;

  return { env, type: isClient ? "client" : "server" };
}
```

### SDK API Design

**JavaScript SDK (Client):**
```typescript
import { ShipFlag } from "@shipflag/js";

const client = ShipFlag.init({
  clientKey: "sf_client_abc123...",
  context: { userId: "user-42", attributes: { plan: "pro" } },
});

// Get evaluated flag value (pre-evaluated by server)
const isEnabled = client.isEnabled("new-dashboard"); // boolean
const value = client.getString("banner-text", "default"); // string with fallback

// Listen for real-time changes
client.on("change", (flagKey, newValue) => { /* re-render */ });

// Update context (triggers re-evaluation)
client.identify({ userId: "user-42", attributes: { plan: "enterprise" } });

// Cleanup
client.destroy();
```

**React SDK:**
```typescript
import { ShipFlagProvider, useFlag, useStringFlag } from "@shipflag/react";

// Provider wraps app
<ShipFlagProvider clientKey="sf_client_..." userId="user-42" attributes={{ plan: "pro" }}>
  <App />
</ShipFlagProvider>

// Hook usage in components
function Dashboard() {
  const showNewDashboard = useFlag("new-dashboard", false);
  const bannerText = useStringFlag("banner-text", "Welcome!");

  if (showNewDashboard) return <NewDashboard banner={bannerText} />;
  return <OldDashboard />;
}
```

**Node.js SDK (Server):**
```typescript
import { ShipFlagServer } from "@shipflag/node";

// Initialize once at startup (downloads rules, starts SSE listener)
const sf = await ShipFlagServer.init({
  serverKey: "sf_server_xyz789...",
});

// Evaluate locally (zero network latency)
const enabled = sf.isEnabled("new-dashboard", { userId: "user-42" });
const value = sf.getString("banner-text", { userId: "user-42" }, "default");

// Graceful shutdown
await sf.close();
```

---

## Database Schema (Drizzle)

```typescript
// src/server/db/schema.ts
import { relations } from "drizzle-orm";
import {
  pgTable,
  uuid,
  text,
  boolean,
  integer,
  timestamp,
  jsonb,
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
  plan: text("plan").default("free").notNull(), // free | pro | scale
  stripeCustomerId: text("stripe_customer_id"),
  stripeSubscriptionId: text("stripe_subscription_id"),
  createdAt: timestamp("created_at").defaultNow().notNull(),
  updatedAt: timestamp("updated_at").defaultNow().notNull(),
});

// ---------------------------------------------------------------------------
// Members
// ---------------------------------------------------------------------------
export const members = pgTable(
  "members",
  {
    id: uuid("id").primaryKey().defaultRandom(),
    orgId: uuid("org_id")
      .references(() => organizations.id, { onDelete: "cascade" })
      .notNull(),
    clerkUserId: text("clerk_user_id").notNull(),
    role: text("role").default("member").notNull(), // owner | admin | member | viewer
    email: text("email"),
    name: text("name"),
    invitedAt: timestamp("invited_at").defaultNow().notNull(),
    joinedAt: timestamp("joined_at"),
  },
  (table) => ({
    orgUserIdx: uniqueIndex("members_org_user_idx").on(
      table.orgId,
      table.clerkUserId
    ),
  })
);

// ---------------------------------------------------------------------------
// Environments
// ---------------------------------------------------------------------------
export const environments = pgTable(
  "environments",
  {
    id: uuid("id").primaryKey().defaultRandom(),
    orgId: uuid("org_id")
      .references(() => organizations.id, { onDelete: "cascade" })
      .notNull(),
    name: text("name").notNull(), // "Development", "Staging", "Production"
    key: text("key").notNull(), // "dev", "staging", "prod"
    color: text("color").notNull(), // "#10B981", "#F59E0B", "#EF4444"
    apiKeyClientHash: text("api_key_client_hash").unique(), // SHA-256 hash
    apiKeyServerHash: text("api_key_server_hash").unique(), // SHA-256 hash
    apiKeyClientPrefix: text("api_key_client_prefix"), // "sf_client_a1b2" (first 16 chars for display)
    apiKeyServerPrefix: text("api_key_server_prefix"), // "sf_server_c3d4" (first 16 chars for display)
    sortOrder: integer("sort_order").default(0).notNull(),
    createdAt: timestamp("created_at").defaultNow().notNull(),
  },
  (table) => ({
    orgKeyIdx: uniqueIndex("environments_org_key_idx").on(
      table.orgId,
      table.key
    ),
  })
);

// ---------------------------------------------------------------------------
// Flags
// ---------------------------------------------------------------------------
export const flags = pgTable(
  "flags",
  {
    id: uuid("id").primaryKey().defaultRandom(),
    orgId: uuid("org_id")
      .references(() => organizations.id, { onDelete: "cascade" })
      .notNull(),
    key: text("key").notNull(), // "new-dashboard", "banner-text"
    name: text("name").notNull(), // "New Dashboard", "Banner Text"
    description: text("description"),
    type: text("type").notNull(), // "boolean" | "string"
    variations: jsonb("variations")
      .notNull()
      .$type<
        Array<{
          key: string; // "on", "off", "variant-a", "control"
          value: boolean | string;
          name?: string;
        }>
      >(), // e.g. [{ key: "on", value: true }, { key: "off", value: false }]
    tags: jsonb("tags").default([]).$type<string[]>(),
    isArchived: boolean("is_archived").default(false).notNull(),
    createdAt: timestamp("created_at").defaultNow().notNull(),
    updatedAt: timestamp("updated_at").defaultNow().notNull(),
  },
  (table) => ({
    orgKeyIdx: uniqueIndex("flags_org_key_idx").on(table.orgId, table.key),
    orgArchivedIdx: index("flags_org_archived_idx").on(
      table.orgId,
      table.isArchived
    ),
  })
);

// ---------------------------------------------------------------------------
// Flag Environment Configs (per-env settings for each flag)
// ---------------------------------------------------------------------------
export const flagEnvironmentConfigs = pgTable(
  "flag_environment_configs",
  {
    id: uuid("id").primaryKey().defaultRandom(),
    flagId: uuid("flag_id")
      .references(() => flags.id, { onDelete: "cascade" })
      .notNull(),
    environmentId: uuid("environment_id")
      .references(() => environments.id, { onDelete: "cascade" })
      .notNull(),
    enabled: boolean("enabled").default(false).notNull(), // kill switch
    defaultVariationKey: text("default_variation_key").notNull(), // "off" or "control"
    rules: jsonb("rules")
      .default([])
      .$type<
        Array<{
          id: string; // nanoid for stable identity in UI
          variationKey: string; // which variation to serve
          percentage?: number; // 0-100, optional
          userIds?: string[]; // specific user IDs, optional
          conditions?: Array<{
            attribute: string; // "plan", "country", etc.
            operator: string; // "eq" | "neq" | "contains" | "startsWith" | "in"
            value: string | string[];
          }>;
          description?: string; // human-readable label for the rule
        }>
      >(),
    updatedAt: timestamp("updated_at").defaultNow().notNull(),
  },
  (table) => ({
    flagEnvIdx: uniqueIndex("flag_env_configs_flag_env_idx").on(
      table.flagId,
      table.environmentId
    ),
  })
);

// ---------------------------------------------------------------------------
// Audit Log
// ---------------------------------------------------------------------------
export const auditLogs = pgTable(
  "audit_logs",
  {
    id: uuid("id").primaryKey().defaultRandom(),
    orgId: uuid("org_id")
      .references(() => organizations.id, { onDelete: "cascade" })
      .notNull(),
    flagId: uuid("flag_id").references(() => flags.id, {
      onDelete: "set null",
    }),
    environmentId: uuid("environment_id").references(() => environments.id, {
      onDelete: "set null",
    }),
    clerkUserId: text("clerk_user_id").notNull(),
    userName: text("user_name"),
    action: text("action").notNull(), // "flag.created" | "flag.updated" | "flag.toggled" | "flag.archived" | "flag.rules_updated" | "env.created" | "env.key_rotated"
    previousState: jsonb("previous_state").$type<Record<string, unknown>>(),
    newState: jsonb("new_state").$type<Record<string, unknown>>(),
    createdAt: timestamp("created_at").defaultNow().notNull(),
  },
  (table) => ({
    orgCreatedIdx: index("audit_logs_org_created_idx").on(
      table.orgId,
      table.createdAt
    ),
    flagIdx: index("audit_logs_flag_idx").on(table.flagId),
  })
);

// ---------------------------------------------------------------------------
// Relations
// ---------------------------------------------------------------------------
export const organizationsRelations = relations(organizations, ({ many }) => ({
  members: many(members),
  environments: many(environments),
  flags: many(flags),
  auditLogs: many(auditLogs),
}));

export const membersRelations = relations(members, ({ one }) => ({
  organization: one(organizations, {
    fields: [members.orgId],
    references: [organizations.id],
  }),
}));

export const environmentsRelations = relations(
  environments,
  ({ one, many }) => ({
    organization: one(organizations, {
      fields: [environments.orgId],
      references: [organizations.id],
    }),
    flagConfigs: many(flagEnvironmentConfigs),
  })
);

export const flagsRelations = relations(flags, ({ one, many }) => ({
  organization: one(organizations, {
    fields: [flags.orgId],
    references: [organizations.id],
  }),
  environmentConfigs: many(flagEnvironmentConfigs),
}));

export const flagEnvironmentConfigsRelations = relations(
  flagEnvironmentConfigs,
  ({ one }) => ({
    flag: one(flags, {
      fields: [flagEnvironmentConfigs.flagId],
      references: [flags.id],
    }),
    environment: one(environments, {
      fields: [flagEnvironmentConfigs.environmentId],
      references: [environments.id],
    }),
  })
);

export const auditLogsRelations = relations(auditLogs, ({ one }) => ({
  organization: one(organizations, {
    fields: [auditLogs.orgId],
    references: [organizations.id],
  }),
  flag: one(flags, {
    fields: [auditLogs.flagId],
    references: [flags.id],
  }),
  environment: one(environments, {
    fields: [auditLogs.environmentId],
    references: [environments.id],
  }),
}));

// ---------------------------------------------------------------------------
// Type helpers
// ---------------------------------------------------------------------------
export type Organization = typeof organizations.$inferSelect;
export type NewOrganization = typeof organizations.$inferInsert;
export type Member = typeof members.$inferSelect;
export type Environment = typeof environments.$inferSelect;
export type Flag = typeof flags.$inferSelect;
export type NewFlag = typeof flags.$inferInsert;
export type FlagEnvironmentConfig = typeof flagEnvironmentConfigs.$inferSelect;
export type AuditLog = typeof auditLogs.$inferSelect;
```

### Initial Migration SQL

```sql
-- 0000_init.sql
CREATE TABLE organizations (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  clerk_org_id TEXT UNIQUE NOT NULL,
  name TEXT NOT NULL,
  slug TEXT UNIQUE NOT NULL,
  plan TEXT NOT NULL DEFAULT 'free',
  stripe_customer_id TEXT,
  stripe_subscription_id TEXT,
  created_at TIMESTAMP NOT NULL DEFAULT now(),
  updated_at TIMESTAMP NOT NULL DEFAULT now()
);

CREATE TABLE members (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  org_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
  clerk_user_id TEXT NOT NULL,
  role TEXT NOT NULL DEFAULT 'member',
  email TEXT,
  name TEXT,
  invited_at TIMESTAMP NOT NULL DEFAULT now(),
  joined_at TIMESTAMP
);
CREATE UNIQUE INDEX members_org_user_idx ON members(org_id, clerk_user_id);

CREATE TABLE environments (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  org_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
  name TEXT NOT NULL,
  key TEXT NOT NULL,
  color TEXT NOT NULL,
  api_key_client_hash TEXT UNIQUE,
  api_key_server_hash TEXT UNIQUE,
  api_key_client_prefix TEXT,
  api_key_server_prefix TEXT,
  sort_order INTEGER NOT NULL DEFAULT 0,
  created_at TIMESTAMP NOT NULL DEFAULT now()
);
CREATE UNIQUE INDEX environments_org_key_idx ON environments(org_id, key);

CREATE TABLE flags (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  org_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
  key TEXT NOT NULL,
  name TEXT NOT NULL,
  description TEXT,
  type TEXT NOT NULL,
  variations JSONB NOT NULL,
  tags JSONB DEFAULT '[]',
  is_archived BOOLEAN NOT NULL DEFAULT false,
  created_at TIMESTAMP NOT NULL DEFAULT now(),
  updated_at TIMESTAMP NOT NULL DEFAULT now()
);
CREATE UNIQUE INDEX flags_org_key_idx ON flags(org_id, key);
CREATE INDEX flags_org_archived_idx ON flags(org_id, is_archived);

CREATE TABLE flag_environment_configs (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  flag_id UUID NOT NULL REFERENCES flags(id) ON DELETE CASCADE,
  environment_id UUID NOT NULL REFERENCES environments(id) ON DELETE CASCADE,
  enabled BOOLEAN NOT NULL DEFAULT false,
  default_variation_key TEXT NOT NULL,
  rules JSONB DEFAULT '[]',
  updated_at TIMESTAMP NOT NULL DEFAULT now()
);
CREATE UNIQUE INDEX flag_env_configs_flag_env_idx ON flag_environment_configs(flag_id, environment_id);

CREATE TABLE audit_logs (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  org_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
  flag_id UUID REFERENCES flags(id) ON DELETE SET NULL,
  environment_id UUID REFERENCES environments(id) ON DELETE SET NULL,
  clerk_user_id TEXT NOT NULL,
  user_name TEXT,
  action TEXT NOT NULL,
  previous_state JSONB,
  new_state JSONB,
  created_at TIMESTAMP NOT NULL DEFAULT now()
);
CREATE INDEX audit_logs_org_created_idx ON audit_logs(org_id, created_at);
CREATE INDEX audit_logs_flag_idx ON audit_logs(flag_id);
```

---

## Phase Breakdown (7 weeks)

### Phase 1 — Data Model, Auth & Flag CRUD (Days 1-5)

**Day 1: Project scaffold + database**
- `npx create-next-app@latest shipflag --app --tailwind --typescript --eslint`
- Install dependencies: `@clerk/nextjs`, `drizzle-orm`, `@neondatabase/serverless`, `@trpc/server`, `@trpc/client`, `@trpc/react-query`, `@tanstack/react-query`, `zod`, `superjson`, `nanoid`, `stripe`, `resend`, `@upstash/redis`
- Create Neon database, configure `DATABASE_URL`
- Write `src/env.ts` (Zod schema for env vars: `DATABASE_URL`, `CLERK_SECRET_KEY`, `NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY`, `STRIPE_SECRET_KEY`, `STRIPE_WEBHOOK_SECRET`, `NEXT_PUBLIC_STRIPE_PUBLISHABLE_KEY`, `RESEND_API_KEY`, `UPSTASH_REDIS_REST_URL`, `UPSTASH_REDIS_REST_TOKEN`, `NEXT_PUBLIC_APP_URL`)
- Write `src/server/db/index.ts` (Neon + Drizzle setup)
- Write `src/server/db/schema.ts` (all 6 tables: organizations, members, environments, flags, flagEnvironmentConfigs, auditLogs)
- Write `drizzle.config.ts`, run `npx drizzle-kit push`
- Write `.env.example`

**Day 2: Clerk auth + organization setup**
- Configure Clerk: create application, enable organizations
- Write `src/middleware.ts` (Clerk middleware, public routes: `/`, `/sign-in(.*)`, `/sign-up(.*)`, `/api/webhooks/(.*)`, `/api/sdk/(.*)`)
- Write `src/app/layout.tsx` (ClerkProvider wrapping app)
- Write `src/app/(auth)/sign-in/[[...sign-in]]/page.tsx`
- Write `src/app/(auth)/sign-up/[[...sign-up]]/page.tsx`
- Write `src/server/trpc/context.ts` (Clerk auth + db injection)
- Write `src/server/trpc/trpc.ts`:
  - `publicProcedure` (no auth)
  - `protectedProcedure` (requires Clerk userId)
  - `orgProcedure` (resolves org from Clerk orgId, loads from DB)
  - `adminProcedure` (requires owner/admin role from members table)
  - Plan limit middleware (max flags based on plan)
- Write `src/server/trpc/routers/_app.ts` (combine all routers)
- Write `src/app/api/trpc/[trpc]/route.ts`
- Write `src/lib/trpc/client.ts` and `src/lib/trpc/server.ts`

**Day 3: Organization provisioning + environments**
- Write `src/server/trpc/routers/organization.ts`:
  - `getOrCreate` mutation: on first dashboard load, upsert org from Clerk orgId, auto-create 3 default environments (dev/staging/prod) with generated API keys
  - `get` query: return org with environments
  - `updateName` mutation
- Write `src/server/auth/api-key.ts`:
  - `generateApiKey(prefix)`: returns `{ raw, hashed }` using `crypto.randomUUID()`
  - `hashApiKey(raw)`: SHA-256
  - `authenticateApiKey(raw)`: lookup by hash, return environment + type (client/server)
- Write `src/server/trpc/routers/environment.ts`:
  - `list` query: all envs for org, ordered by sortOrder
  - `rotateKey` mutation: generate new key, hash & store, return raw key ONCE
  - `create` mutation (Pro+ only): add custom environment
  - `delete` mutation: only custom envs (not dev/staging/prod)

**Day 4: Flag CRUD**
- Write `src/server/trpc/routers/flag.ts`:
  - `list` query: all non-archived flags for org, with environment configs eagerly loaded; supports `search` text filter on key/name
  - `getByKey` query: single flag by key, with all environment configs
  - `getById` query: single flag by id, with all environment configs and audit log entries
  - `create` mutation: validates key uniqueness within org; creates flag + N flagEnvironmentConfig rows (one per env, all disabled, default variation = first variation); writes audit log; invalidates Redis cache; subject to plan flag limit
  - `update` mutation: update name, description, tags; audit log
  - `archive` mutation: soft-delete, audit log
  - `unarchive` mutation
  - `delete` mutation: hard delete (only archived flags)
- Write `src/server/lib/audit.ts`:
  - `createAuditEntry({ db, orgId, flagId?, envId?, userId, userName, action, prev, next })`: inserts into audit_logs
- Plan limit enforcement: Free = 5 flags, Pro = unlimited, Scale = unlimited

**Day 5: Flag environment config + toggle**
- Write `src/server/trpc/routers/flag-config.ts`:
  - `toggle` mutation: flip `enabled` boolean for flag+env pair; audit log with diff; invalidate Redis; publish SSE event
  - `updateDefaultVariation` mutation: change default variation key; audit log
  - `updateRules` mutation: replace entire rules array for flag+env; validate rule structure with Zod; audit log with previous/new state diff; invalidate Redis; publish SSE event
  - `promoteConfig` mutation: copy config from one env to another (for promoting staging -> prod)
- Write `src/server/cache/redis.ts`:
  - `getOrSetFlagRules(envId)`: check Redis, if miss load from DB, set with 5-min TTL
  - `invalidateFlagRules(envId)`: delete Redis key, publish to `flags:{envId}` channel
- Verify: create a flag via tRPC, toggle it, confirm audit log row exists, confirm Redis is invalidated

### Phase 2 — Evaluation Engine + SDK API (Days 6-10)

**Day 6: Evaluation engine core**
- Write `src/server/evaluation/murmur3.ts`: pure TypeScript murmur3 32-bit hash implementation (no native dependencies for Vercel compatibility)
- Write `src/server/evaluation/evaluate.ts`:
  - `evaluateFlag(flag, envConfig, context)`: implements the full rule evaluation order (kill switch -> rule walk -> default)
  - `evaluateAllFlags(flags, envConfigs, context)`: batch evaluate, returns `Record<string, { key: string; value: boolean | string }>`
  - `evaluateCondition(condition, attributes)`: handles eq, neq, contains, startsWith, endsWith, in operators
- Write unit tests: `src/server/evaluation/__tests__/evaluate.test.ts`
  - Test kill switch returns default
  - Test user ID targeting overrides percentage
  - Test percentage rollout stickiness (same user+flag = same result across 1000 calls)
  - Test percentage distribution (10,000 users at 50% should yield ~5,000 true)
  - Test rule priority (first match wins)
  - Test condition operators
  - Test empty rules returns default

**Day 7: SDK REST API endpoints**
- Write `src/app/api/sdk/flags/route.ts` (GET):
  - Authenticate via `Authorization: Bearer sf_client_...` header
  - Parse `X-ShipFlag-UserId` and `X-ShipFlag-Attributes` (JSON) headers
  - Load all non-archived flags + configs for the authenticated environment from Redis (or DB fallback)
  - Evaluate all flags against the provided context
  - Return `{ flags: { "flag-key": { key: "on", value: true }, ... }, _meta: { envId, evaluatedAt } }`
  - Set `Cache-Control: no-store` (must be fresh)
- Write `src/app/api/sdk/rules/route.ts` (GET):
  - Authenticate via `Authorization: Bearer sf_server_...` header (server keys only)
  - Load all non-archived flags + configs for the authenticated environment
  - Return raw rules: `{ flags: [{ key, type, variations, config: { enabled, defaultVariationKey, rules } }], _meta: { envId, generatedAt } }`
  - This is for server SDKs that evaluate locally
- Write `src/server/auth/sdk-middleware.ts`:
  - Rate limiting via Upstash: 100 req/s per API key for flag endpoints, 10 req/s for rules endpoint
  - CORS headers for client SDK requests
  - Request counting for MAU tracking

**Day 8: SSE streaming endpoint**
- Write `src/app/api/sdk/stream/route.ts` (GET):
  - Authenticate API key (client or server)
  - Return SSE stream using `ReadableStream` + `TextEncoder`
  - Subscribe to Upstash Redis pub/sub channel `flags:{envId}`
  - On message: for client keys, re-evaluate all flags and send evaluated results; for server keys, send raw rule update
  - Heartbeat every 30s (`: heartbeat\n\n`)
  - Cleanup subscriber on `AbortSignal`
  - Event types: `flag.updated` (single flag change), `flags.bulk` (multiple flags changed), `heartbeat`
- Write `src/server/cache/publish.ts`:
  - `publishFlagUpdate(envId, flagKey)`: publish to Redis channel, called by every flag mutation
- Write connection tracking for MAU metering:
  - On SSE connect: extract userId from headers, add to Redis HyperLogLog set `mau:{orgId}:{YYYY-MM}`
  - On SDK flag evaluation: same HyperLogLog tracking

**Day 9: Test evaluation panel API**
- Write `src/server/trpc/routers/test-evaluation.ts`:
  - `evaluate` query: takes flagId, environmentId, context (userId, attributes), returns evaluated variation with explanation of which rule matched and why
  - This powers the "Test evaluation" panel in the dashboard
- Write `src/app/api/sdk/identify/route.ts` (POST):
  - Client SDKs call this when context changes (new userId or attributes)
  - Re-evaluates all flags for new context
  - Returns fresh evaluated results
  - Used by `client.identify()` in JS SDK

**Day 10: SDK REST API hardening**
- Add comprehensive error responses: `{ error: "INVALID_API_KEY", message: "..." }` with proper HTTP status codes (401, 403, 429, 500)
- Add `X-Request-Id` header to all SDK responses
- Add request logging: endpoint, API key prefix, response time, status code
- Write integration tests for all SDK endpoints:
  - Client key can access /sdk/flags but not /sdk/rules
  - Server key can access /sdk/rules
  - Invalid key returns 401
  - Rate limiting returns 429
  - SSE stream delivers updates within 1s of flag change

### Phase 3 — JavaScript + React SDKs (Days 11-15)

**Day 11: JavaScript SDK core**
- Create `packages/sdk-js/` directory (monorepo-lite with npm workspaces or standalone repo)
- Write `packages/sdk-js/src/index.ts`:
  ```typescript
  export class ShipFlag {
    private flags: Map<string, EvaluatedFlag>;
    private eventSource: EventSource | null;
    private listeners: Map<string, Set<Function>>;

    static async init(config: {
      clientKey: string;
      context?: { userId?: string; attributes?: Record<string, unknown> };
      apiUrl?: string; // defaults to "https://api.shipflag.dev"
    }): Promise<ShipFlag>;

    isEnabled(flagKey: string, defaultValue?: boolean): boolean;
    getString(flagKey: string, defaultValue?: string): string;
    getAll(): Record<string, unknown>;

    on(event: "change" | "error" | "ready", listener: Function): void;
    off(event: string, listener: Function): void;

    async identify(context: { userId?: string; attributes?: Record<string, unknown> }): Promise<void>;
    destroy(): void;
  }
  ```
- Implement `init()`: fetch GET /sdk/flags with context headers, populate Map, start SSE connection
- Implement SSE listener: on `flag.updated` event, update Map, emit "change" event
- Implement `identify()`: POST /sdk/identify with new context, replace Map
- Implement `destroy()`: close EventSource, clear listeners
- Local storage caching: on every flag fetch, persist to `localStorage` with key `shipflag:{clientKey}:flags`; on `init()`, hydrate from localStorage immediately for instant start, then fetch fresh flags in background

**Day 12: JavaScript SDK polish**
- Error handling: retry SSE on disconnect (exponential backoff: 1s, 2s, 4s, 8s, max 30s)
- Offline support: if fetch fails, use localStorage cache, emit "error" event but still function
- TypeScript types: export `ShipFlagConfig`, `EvaluatedFlag`, `ShipFlagContext`, `ShipFlagEvent`
- Bundle with `tsup` (ESM + CJS + types)
- Target: <5KB gzipped
- Write `packages/sdk-js/package.json`: name `@shipflag/js`, main/module/types fields
- Write `packages/sdk-js/README.md`: installation, quickstart, API reference

**Day 13: React SDK**
- Create `packages/sdk-react/` directory
- Write `packages/sdk-react/src/provider.tsx`:
  ```typescript
  export function ShipFlagProvider({
    clientKey,
    userId,
    attributes,
    children,
  }: {
    clientKey: string;
    userId?: string;
    attributes?: Record<string, unknown>;
    children: React.ReactNode;
  }): JSX.Element;
  ```
  - Initializes ShipFlag JS client in useEffect
  - Stores client + flags in React context
  - Re-renders children on any flag change via SSE
  - Handles cleanup on unmount
- Write `packages/sdk-react/src/hooks.ts`:
  - `useFlag(key: string, defaultValue?: boolean): boolean`
  - `useStringFlag(key: string, defaultValue?: string): string`
  - `useFlags(): Record<string, unknown>` (all flags)
  - `useShipFlag(): ShipFlag` (access raw client)
- Memoization: hooks return stable references when flag value hasn't changed (avoid re-renders)
- SSR support: on server, hooks return default values (no flag evaluation); hydrate on client
- Write `packages/sdk-react/src/index.ts`: re-export everything
- Bundle with `tsup`, peerDependencies: react ^18 || ^19

**Day 14: React SDK SSR + testing**
- Server component support: `getFlags(serverKey, context)` async function for RSC (calls /sdk/flags from server-side, no SSE)
- Write `packages/sdk-react/src/server.ts`:
  ```typescript
  export async function getFlags(config: {
    serverKey: string;
    userId?: string;
    attributes?: Record<string, unknown>;
    apiUrl?: string;
  }): Promise<Record<string, unknown>>;
  ```
- Write unit tests for all hooks (React Testing Library)
- Write integration test: provider -> hook -> flag value -> SSE update -> re-render
- Test SSR: hooks return defaults on server, hydrate on client without flicker

**Day 15: SDK documentation + npm publish prep**
- Write SDK installation guides (for docs site, built later):
  - JavaScript: `npm install @shipflag/js` + quickstart
  - React: `npm install @shipflag/react` + provider setup + hooks
- Prepare `package.json` for both packages with proper fields (main, module, types, exports, files, repository, keywords)
- Verify tree-shaking works (import only `useFlag` doesn't pull entire SDK)
- Create `packages/sdk-js/CHANGELOG.md` and `packages/sdk-react/CHANGELOG.md`

### Phase 4 — Node.js Server SDK (Days 16-20)

**Day 16: Node.js SDK core**
- Create `packages/sdk-node/` directory
- Write `packages/sdk-node/src/index.ts`:
  ```typescript
  export class ShipFlagServer {
    private rules: FlagRuleSet;
    private eventSource: EventSource; // via 'eventsource' polyfill for Node
    private murmur3: (input: string) => number;

    static async init(config: {
      serverKey: string;
      apiUrl?: string;
      pollingIntervalMs?: number; // fallback polling, default 60_000
    }): Promise<ShipFlagServer>;

    isEnabled(flagKey: string, context?: EvalContext, defaultValue?: boolean): boolean;
    getString(flagKey: string, context?: EvalContext, defaultValue?: string): string;
    getAllFlags(context?: EvalContext): Record<string, unknown>;

    async close(): Promise<void>;
  }

  interface EvalContext {
    userId?: string;
    attributes?: Record<string, unknown>;
  }
  ```
- Implement `init()`: fetch GET /sdk/rules with server key, store raw rules in memory, start SSE connection
- Embed the same murmur3 + evaluation logic from `src/server/evaluation/` (shared as a pure function)
- Implement `isEnabled()` and `getString()`: run `evaluateFlag()` locally using in-memory rules (zero latency)

**Day 17: Node.js SDK resilience**
- SSE reconnection with exponential backoff (same as JS SDK)
- Fallback polling: if SSE fails for >60s, fall back to polling GET /sdk/rules every N seconds
- In-memory rule cache: never throw on evaluation; if rules haven't loaded yet, return default value
- Graceful degradation: if API is completely unreachable, persist last-known rules to disk (`/tmp/shipflag-rules-{envId}.json`) and load on restart
- Thread safety: rules are replaced atomically (single reference swap, no mutations)

**Day 18: Shared evaluation package**
- Extract evaluation logic into `packages/evaluation/`:
  - `murmur3.ts` (pure TS, no dependencies)
  - `evaluate.ts` (evaluateFlag, evaluateAllFlags, evaluateCondition)
  - `types.ts` (shared types between server and SDK)
- Both the main Next.js app and the Node.js SDK depend on `@shipflag/evaluation`
- This ensures client and server evaluate identically (no drift)
- Write comprehensive tests in `packages/evaluation/__tests__/`

**Day 19: Node.js SDK tests + polish**
- Unit tests: evaluation with mocked rules (100% coverage on evaluate.ts)
- Integration test: init SDK against real API, evaluate flags, verify SSE updates arrive
- Performance benchmark: 1M evaluations in <1s (murmur3 hashing speed)
- Write `packages/sdk-node/README.md`
- Bundle with `tsup` (CJS + ESM), target Node 18+
- peerDependencies: none (fully standalone)

**Day 20: MAU tracking + usage metering**
- Write `src/server/billing/usage.ts`:
  - On every SDK flag request: extract userId from headers, add to Redis HyperLogLog `mau:{orgId}:{YYYY-MM}`
  - `getMAUCount(orgId, month)`: PFCOUNT on HyperLogLog
  - Cron job (daily): check each org's MAU against plan limit, send warning email at 80% and 100%
- Write `src/app/api/cron/mau-check/route.ts`:
  - Vercel Cron (daily at 00:00 UTC)
  - For each org: compare MAU count to plan limit (Free: 1K, Pro: 10K, Scale: 100K)
  - At 100%: set `org.mauExceeded = true`, SDK endpoints return `X-ShipFlag-MAU-Exceeded: true` header but continue serving (grace period)
  - At 120%: SDK endpoints return 402 with error (hard cutoff after 7-day grace)
- Plan limit constants: `src/server/billing/plans.ts`

### Phase 5 — Dashboard UI (Days 21-28)

**Day 21: App shell + navigation**
- Write `src/app/(dashboard)/layout.tsx`: sidebar navigation (Flags, Audit Log, Settings), org switcher (Clerk `<OrganizationSwitcher />`), user menu
- Write `src/components/ui/sidebar.tsx`: collapsible sidebar with icons (Flag icon, Clock/AuditLog, Gear/Settings)
- Write `src/components/ui/badge.tsx`, `button.tsx`, `card.tsx`, `input.tsx`, `dialog.tsx`, `select.tsx`, `dropdown-menu.tsx`, `tooltip.tsx`, `switch.tsx`, `slider.tsx`, `tabs.tsx` (Radix UI primitives + Tailwind)
- Write `src/app/(dashboard)/page.tsx`: redirect to `/flags`
- Write `src/components/providers.tsx`: tRPC provider + React Query provider + ClerkProvider

**Day 22: Flag list page**
- Write `src/app/(dashboard)/flags/page.tsx`:
  - Search bar (filters flags by key or name, debounced 300ms)
  - "Create Flag" button (opens dialog)
  - Flag table with columns: Name/Key, Type badge (boolean=blue, string=purple), Environment toggles (colored dots: green=on, gray=off, for each env), Tags, Updated date, Stale badge (if unchanged for 30+ days and 100% enabled in prod)
  - Each row is clickable (navigates to flag detail)
  - Quick toggle: click env dot to toggle flag on/off inline (fires `flag-config.toggle` mutation, optimistic update)
  - Empty state: illustration + "Create your first flag" CTA
- Write `src/components/flags/flag-table.tsx`: the table component with sortable columns
- Write `src/components/flags/environment-toggle.tsx`: colored circle indicators with click-to-toggle
- Write `src/components/flags/stale-badge.tsx`: yellow badge with clock icon

**Day 23: Create flag dialog**
- Write `src/components/flags/create-flag-dialog.tsx`:
  - Step 1: Key input (auto-generated from name, kebab-case, editable), Name input, Description textarea, Type selector (boolean/string)
  - Step 2 (if string): define variations (key + value pairs), add/remove rows, minimum 2
  - For boolean: auto-create variations `[{ key: "on", value: true }, { key: "off", value: false }]`
  - Key validation: must be lowercase alphanumeric + hyphens, unique within org (checked via tRPC query on blur)
  - Submit: calls `flag.create` mutation
  - On success: navigate to flag detail page, show toast "Flag created"

**Day 24: Flag detail page - overview + env tabs**
- Write `src/app/(dashboard)/flags/[flagId]/page.tsx`:
  - Header: flag name (editable inline), key (copyable), type badge, description (editable), tags
  - Environment tabs: Dev | Staging | Production (color-coded underline matching env color)
  - Per-environment panel:
    - Enable/disable toggle (large, prominent) with confirmation dialog for production
    - Default variation selector dropdown
    - "Promote from [env]" button (copies config from previous env)
  - Tab bar at bottom: Targeting Rules | Test Evaluation | Audit Log
- Write `src/components/flags/flag-header.tsx`
- Write `src/components/flags/environment-tabs.tsx`
- Write `src/components/flags/enable-toggle.tsx` (with prod confirmation)

**Day 25: Targeting rules builder**
- Write `src/components/flags/rules-builder.tsx`:
  - Ordered list of rules (drag to reorder via `@dnd-kit`)
  - Each rule card contains:
    - Serve variation: dropdown selecting which variation
    - Rule type sections (all optional, combine for AND logic):
      - **User IDs**: text input with tag-style chips, paste comma-separated
      - **Percentage**: slider 0-100 with numeric input
      - **Conditions**: "If user [attribute] [operator] [value]" rows, add/remove
    - Description: optional label for the rule
    - Delete rule button (with confirmation)
  - "Add Rule" button at bottom
  - "Save Changes" button: sends entire rules array to `flag-config.updateRules`
  - Dirty state tracking: warn if navigating away with unsaved changes
- Write `src/components/flags/condition-row.tsx`: attribute input, operator dropdown (equals, not equals, contains, starts with, in list), value input
- Write `src/components/flags/percentage-slider.tsx`: custom slider with percentage display and bucket visualization

**Day 26: Test evaluation panel**
- Write `src/components/flags/test-evaluation.tsx`:
  - User ID input field
  - Attributes key-value editor (add rows of attribute name + value)
  - "Evaluate" button
  - Result display: variation key, variation value, and explanation text:
    - "Rule 2 matched: user ID 'user-42' is in the targeting list -> serving 'on' (true)"
    - "Percentage rollout: hash bucket 37 < threshold 50 -> serving 'on' (true)"
    - "No rules matched -> serving default 'off' (false)"
  - Calls `test-evaluation.evaluate` tRPC query
- Write `src/server/evaluation/explain.ts`: same as evaluate but returns step-by-step explanation of the decision path

**Day 27: Flag detail - audit log tab**
- Write `src/components/flags/flag-audit-log.tsx`:
  - Chronological list of changes for this flag
  - Each entry: user avatar + name, action description ("toggled Production ON", "updated targeting rules in Staging"), timestamp (relative + absolute on hover)
  - Expand entry to see JSON diff: side-by-side previous_state vs new_state with highlighted changes
  - Uses `src/components/ui/json-diff.tsx`: visual JSON diff component (green for additions, red for removals)
  - Pagination (load more)

**Day 28: Global audit log page**
- Write `src/app/(dashboard)/audit-log/page.tsx`:
  - Full audit log for the organization
  - Filters: flag selector, environment selector, user selector, action type, date range
  - Same entry format as flag-level audit log
  - Export to CSV button
- Write `src/server/trpc/routers/audit-log.ts`:
  - `list` query: paginated, with filters (flagId, envId, userId, action, dateFrom, dateTo)
  - `getById` query: single entry with full diff

### Phase 6 — Settings, API Keys & Billing (Days 29-35)

**Day 29: Settings - environments page**
- Write `src/app/(dashboard)/settings/page.tsx`: settings layout with sub-navigation (Environments, API Keys, Team, Billing)
- Write `src/app/(dashboard)/settings/environments/page.tsx`:
  - List of environments with name, key, color swatch, flag count
  - Reorder via drag-and-drop
  - "Add Environment" button (Pro+ only, shows upgrade prompt on Free)
  - Edit environment: change name, color
  - Delete environment: only custom ones, confirmation dialog warns about losing all flag configs

**Day 30: Settings - API keys page**
- Write `src/app/(dashboard)/settings/api-keys/page.tsx`:
  - Table grouped by environment
  - Each environment shows: Client Key (masked: `sf_client_a1b2...****`), Server Key (masked)
  - "Reveal" button (requires confirmation, shows full key for 30s, then re-masks)
  - "Rotate" button (confirmation dialog: "This will invalidate the current key. All connected SDKs will lose access."), calls `environment.rotateKey`
  - "Copy" button for each key
  - SDK quickstart snippets: show code examples with the key pre-filled

**Day 31: Settings - team management**
- Write `src/app/(dashboard)/settings/team/page.tsx`:
  - Current members list: name, email, role badge, joined date
  - Role dropdown per member (owner can change anyone, admin can change member/viewer)
  - "Invite Member" button: email input + role selector, sends invite via Clerk
  - Remove member (with confirmation)
  - Pending invitations list with "Resend" and "Revoke" actions
- Wire up Clerk's `<OrganizationProfile />` for heavy lifting, with custom UI wrapper for consistency

**Day 32: Billing - Stripe integration**
- Write `src/server/billing/stripe.ts`:
  - `PLANS` constant: `{ free: { name, price, maxFlags: 5, maxMAU: 1000, maxEnvs: 2 }, pro: { name, price: 25, maxFlags: Infinity, maxMAU: 10000, maxEnvs: Infinity }, scale: { name, price: 79, maxFlags: Infinity, maxMAU: 100000, maxEnvs: Infinity } }`
  - `createCheckoutSession({ orgId, plan })`: Stripe Checkout for pro/scale
  - `createPortalSession(stripeCustomerId)`: billing portal for plan changes
  - `getSubscriptionDetails(subscriptionId)`: current plan, status, renewal date
- Write `src/app/api/webhooks/stripe/route.ts`:
  - `checkout.session.completed`: update org plan + stripeCustomerId + stripeSubscriptionId
  - `customer.subscription.updated`: update org plan if changed
  - `customer.subscription.deleted`: downgrade org to free
  - `invoice.payment_failed`: send email notification via Resend
- Write `src/app/(dashboard)/settings/billing/page.tsx`:
  - Current plan card with feature list
  - Usage meters: flags used (X/5 or X/unlimited), MAU this month (X/1,000), environments (X/2)
  - Upgrade/downgrade buttons
  - Billing history (from Stripe portal link)

**Day 33: Plan enforcement**
- Write `src/server/billing/enforce.ts`:
  - Flag limit: on `flag.create`, check count. Free = 5 flags, Pro/Scale = unlimited
  - Environment limit: on `environment.create`, check count. Free = 2 envs, Pro/Scale = unlimited
  - MAU limit: enforced at SDK endpoint level (see Day 20)
  - SDK restriction: Free plan only allows client keys (no server SDK). On server key auth, check org plan; if free, return 403 with upgrade message
- Update all relevant mutations to call enforcement checks
- Write `src/components/billing/upgrade-prompt.tsx`: reusable component shown when hitting limits, with pricing comparison and CTA

**Day 34: Email notifications**
- Write `src/server/email/templates/`:
  - `team-invite.tsx`: "You've been invited to [org] on ShipFlag" (React Email component)
  - `mau-warning.tsx`: "You've reached 80% of your MAU limit" with usage stats + upgrade CTA
  - `mau-exceeded.tsx`: "Your MAU limit has been exceeded — upgrade to continue"
  - `flag-changed-prod.tsx`: optional notification when production flag is toggled (configurable per org)
- Write `src/server/email/send.ts`:
  - `sendEmail({ to, subject, template })` wrapper using Resend
  - Rate limiting: max 10 emails per org per hour

**Day 35: Onboarding flow**
- Write `src/app/(dashboard)/onboarding/page.tsx`:
  - Step 1: "Create your organization" (if none exists via Clerk)
  - Step 2: "Here are your environments" (show the 3 auto-created envs with keys)
  - Step 3: "Create your first flag" (inline creation form)
  - Step 4: "Install the SDK" (show code snippets for JS/React/Node with their actual API key)
  - Step 5: "Toggle it!" (live toggle with real-time verification: SDK connected indicator)
  - Skip button on each step, progress indicator
- Write `src/server/trpc/routers/onboarding.ts`:
  - `getStatus` query: returns which steps are completed (has org, has flag, has SDK connection)
  - `markComplete` mutation: stores onboarding completion in org settings

### Phase 7 — Polish, Docs & Launch (Days 36-42+)

**Day 36: Landing page**
- Write `src/app/(marketing)/page.tsx`: public landing page
  - Hero: "Ship features fearlessly" + "Feature flags that just work. $0 to start."
  - Feature grid: percentage rollouts, user targeting, real-time updates, audit log, 3 SDKs
  - Pricing table: Free / Pro ($25/mo) / Scale ($79/mo)
  - Code snippet preview: show React hook usage
  - Social proof section (placeholder)
  - "Get Started Free" CTA -> /sign-up
- Write `src/app/(marketing)/pricing/page.tsx`: detailed pricing comparison table
- Write `src/app/(marketing)/layout.tsx`: marketing header + footer

**Day 37: Dashboard polish + responsive**
- Mobile-responsive sidebar (hamburger menu)
- Loading states: skeleton loaders for flag list, flag detail, audit log
- Error boundaries: `src/components/error-boundary.tsx` for graceful error display
- Toast notifications: use Sonner for all mutations (flag created, toggle success, key rotated, etc.)
- Optimistic updates: toggle flag updates UI immediately, reverts on error
- Keyboard shortcuts: `/` to focus search, `c` to create flag, `Esc` to close dialogs

**Day 38: Real-time dashboard updates**
- Dashboard itself subscribes to SSE for the current environment: when another team member changes a flag, the list updates live
- Write `src/hooks/use-flag-updates.ts`: custom hook that subscribes to SSE endpoint using the dashboard session (not SDK key)
- Write `src/app/api/dashboard/stream/route.ts`: SSE endpoint authenticated via Clerk session, publishes all flag changes for the org
- Invalidate tRPC queries on SSE events (flag list, flag detail, audit log)

**Day 39: Documentation site structure**
- Write documentation pages under `src/app/(docs)/docs/`:
  - `/docs/quickstart`: 5-minute setup guide
  - `/docs/concepts/flags`: flag types, variations, environments
  - `/docs/concepts/targeting`: rules, percentage rollout, user targeting
  - `/docs/sdk/javascript`: JS SDK API reference
  - `/docs/sdk/react`: React SDK API reference + SSR guide
  - `/docs/sdk/node`: Node.js SDK API reference
  - `/docs/api/rest`: SDK REST API reference (for building custom SDKs)
  - `/docs/api/authentication`: API key management
- Use MDX for documentation pages: `@next/mdx` + Tailwind typography

**Day 40: Testing + hardening**
- End-to-end tests (Playwright):
  - Sign up -> create org -> create flag -> toggle -> verify audit log
  - Percentage rollout: create flag at 50%, hit SDK endpoint 100 times, verify ~50% true
  - SSE: toggle flag in dashboard, verify SDK receives update within 2s
  - Billing: upgrade to Pro, verify flag limit increases
- Load testing the SDK endpoint:
  - Target: 1000 req/s sustained on /sdk/flags with <50ms p99 latency
  - Redis cache hit rate should be >95%
- Security audit:
  - API keys cannot be retrieved from DB (only hashes stored)
  - Client keys cannot access /sdk/rules (server-only)
  - Rate limiting prevents abuse
  - CORS restricts client SDK to configured origins

**Day 41: Performance optimization**
- Redis cache warming: on deploy, pre-populate cache for all active environments
- SDK endpoint optimization: single DB query with JOINs (flags + configs + env), not N+1
- Flag list query optimization: ensure proper index usage, add DB-level pagination (cursor-based)
- SSE connection pooling: Upstash Redis pub/sub subscriber reuse across connections
- Dashboard bundle size: verify <200KB initial JS, lazy-load flag detail/rules builder/audit log

**Day 42: Launch prep**
- Publish SDKs to npm: `@shipflag/js`, `@shipflag/react`, `@shipflag/node`
- Set up Sentry for error monitoring (dashboard + SDK endpoints)
- Set up Vercel Analytics for dashboard usage metrics
- Set up Upstash Redis monitoring dashboard
- Create Stripe products and prices for Pro ($25/mo) and Scale ($79/mo)
- DNS + Vercel domain setup
- Final smoke test: full flow from sign-up to SDK integration
- Product Hunt / Hacker News launch post draft

---

## Critical Files

```
shipflag/
├── .env.example
├── drizzle.config.ts
├── package.json
├── tsconfig.json
├── next.config.ts
├── postcss.config.mjs
│
├── src/
│   ├── env.ts                              # Zod env validation
│   ├── middleware.ts                        # Clerk middleware + public routes
│   │
│   ├── app/
│   │   ├── layout.tsx                      # Root layout (ClerkProvider)
│   │   ├── page.tsx                        # Landing / redirect
│   │   │
│   │   ├── (auth)/
│   │   │   ├── sign-in/[[...sign-in]]/page.tsx
│   │   │   └── sign-up/[[...sign-up]]/page.tsx
│   │   │
│   │   ├── (marketing)/
│   │   │   ├── layout.tsx                  # Marketing header/footer
│   │   │   ├── page.tsx                    # Landing page
│   │   │   └── pricing/page.tsx            # Pricing page
│   │   │
│   │   ├── (dashboard)/
│   │   │   ├── layout.tsx                  # Sidebar + org switcher
│   │   │   ├── page.tsx                    # Redirect to /flags
│   │   │   │
│   │   │   ├── flags/
│   │   │   │   ├── page.tsx                # Flag list (table + search + create)
│   │   │   │   └── [flagId]/
│   │   │   │       └── page.tsx            # Flag detail (env tabs + rules + audit)
│   │   │   │
│   │   │   ├── audit-log/
│   │   │   │   └── page.tsx                # Global audit log with filters
│   │   │   │
│   │   │   ├── settings/
│   │   │   │   ├── page.tsx                # Settings layout
│   │   │   │   ├── environments/page.tsx   # Manage environments
│   │   │   │   ├── api-keys/page.tsx       # View/rotate API keys
│   │   │   │   ├── team/page.tsx           # Team members + invites
│   │   │   │   └── billing/page.tsx        # Plan + usage + upgrade
│   │   │   │
│   │   │   └── onboarding/
│   │   │       └── page.tsx                # Onboarding wizard
│   │   │
│   │   ├── (docs)/
│   │   │   └── docs/
│   │   │       ├── quickstart/page.mdx
│   │   │       ├── concepts/
│   │   │       │   ├── flags/page.mdx
│   │   │       │   └── targeting/page.mdx
│   │   │       └── sdk/
│   │   │           ├── javascript/page.mdx
│   │   │           ├── react/page.mdx
│   │   │           └── node/page.mdx
│   │   │
│   │   └── api/
│   │       ├── trpc/[trpc]/route.ts        # tRPC handler
│   │       │
│   │       ├── sdk/                         # SDK REST endpoints (API key auth)
│   │       │   ├── flags/route.ts           # GET: evaluated flags (client SDK)
│   │       │   ├── rules/route.ts           # GET: raw rules (server SDK)
│   │       │   ├── stream/route.ts          # GET: SSE stream (real-time updates)
│   │       │   └── identify/route.ts        # POST: re-evaluate for new context
│   │       │
│   │       ├── dashboard/
│   │       │   └── stream/route.ts          # GET: SSE for dashboard real-time
│   │       │
│   │       ├── webhooks/
│   │       │   └── stripe/route.ts          # Stripe webhook handler
│   │       │
│   │       └── cron/
│   │           └── mau-check/route.ts       # Daily MAU enforcement cron
│   │
│   ├── server/
│   │   ├── db/
│   │   │   ├── index.ts                    # Neon + Drizzle setup
│   │   │   ├── schema.ts                   # All tables + relations + types
│   │   │   └── migrations/
│   │   │       └── 0000_init.sql
│   │   │
│   │   ├── trpc/
│   │   │   ├── context.ts                  # Clerk auth + DB context
│   │   │   ├── trpc.ts                     # Procedures + middleware (auth, org, plan)
│   │   │   └── routers/
│   │   │       ├── _app.ts                 # Root router
│   │   │       ├── organization.ts         # Org CRUD + provisioning
│   │   │       ├── environment.ts          # Env list + key rotation
│   │   │       ├── flag.ts                 # Flag CRUD + archive
│   │   │       ├── flag-config.ts          # Toggle + rules + promote
│   │   │       ├── audit-log.ts            # Audit log queries
│   │   │       ├── test-evaluation.ts      # Test evaluation with explanation
│   │   │       ├── billing.ts              # Checkout + portal + usage
│   │   │       └── onboarding.ts           # Onboarding status
│   │   │
│   │   ├── evaluation/
│   │   │   ├── murmur3.ts                  # Pure TS murmur3 hash
│   │   │   ├── evaluate.ts                 # Flag evaluation engine
│   │   │   ├── explain.ts                  # Evaluation with decision explanation
│   │   │   └── __tests__/
│   │   │       └── evaluate.test.ts        # Unit tests for evaluation
│   │   │
│   │   ├── auth/
│   │   │   ├── api-key.ts                  # Generate, hash, authenticate API keys
│   │   │   └── sdk-middleware.ts            # Rate limiting + CORS + MAU tracking
│   │   │
│   │   ├── cache/
│   │   │   ├── redis.ts                    # Upstash Redis client + flag cache
│   │   │   └── publish.ts                  # Pub/sub for SSE notifications
│   │   │
│   │   ├── billing/
│   │   │   ├── stripe.ts                   # Stripe helpers (checkout, portal)
│   │   │   ├── plans.ts                    # Plan definitions + limits
│   │   │   ├── enforce.ts                  # Plan enforcement logic
│   │   │   └── usage.ts                    # MAU tracking (HyperLogLog)
│   │   │
│   │   ├── email/
│   │   │   ├── send.ts                     # Resend wrapper
│   │   │   └── templates/
│   │   │       ├── team-invite.tsx
│   │   │       ├── mau-warning.tsx
│   │   │       ├── mau-exceeded.tsx
│   │   │       └── flag-changed-prod.tsx
│   │   │
│   │   └── lib/
│   │       └── audit.ts                    # Audit log insertion helper
│   │
│   ├── components/
│   │   ├── providers.tsx                   # tRPC + React Query + Clerk providers
│   │   │
│   │   ├── ui/                             # Radix UI + Tailwind primitives
│   │   │   ├── badge.tsx
│   │   │   ├── button.tsx
│   │   │   ├── card.tsx
│   │   │   ├── dialog.tsx
│   │   │   ├── dropdown-menu.tsx
│   │   │   ├── input.tsx
│   │   │   ├── select.tsx
│   │   │   ├── sidebar.tsx
│   │   │   ├── slider.tsx
│   │   │   ├── switch.tsx
│   │   │   ├── tabs.tsx
│   │   │   ├── textarea.tsx
│   │   │   ├── toast.tsx
│   │   │   ├── tooltip.tsx
│   │   │   └── json-diff.tsx               # Visual JSON diff viewer
│   │   │
│   │   ├── flags/
│   │   │   ├── flag-table.tsx              # Flag list table
│   │   │   ├── environment-toggle.tsx      # Colored env toggle dots
│   │   │   ├── stale-badge.tsx             # Stale flag indicator
│   │   │   ├── create-flag-dialog.tsx      # Create flag modal
│   │   │   ├── flag-header.tsx             # Flag detail header
│   │   │   ├── environment-tabs.tsx        # Env tab switcher
│   │   │   ├── enable-toggle.tsx           # Enable/disable with prod confirm
│   │   │   ├── rules-builder.tsx           # Drag-and-drop targeting rules
│   │   │   ├── condition-row.tsx           # Single condition (attr, op, val)
│   │   │   ├── percentage-slider.tsx       # Percentage rollout slider
│   │   │   ├── test-evaluation.tsx         # Test evaluation panel
│   │   │   └── flag-audit-log.tsx          # Per-flag audit log
│   │   │
│   │   ├── billing/
│   │   │   └── upgrade-prompt.tsx          # Plan limit upgrade CTA
│   │   │
│   │   └── error-boundary.tsx              # Global error boundary
│   │
│   ├── hooks/
│   │   └── use-flag-updates.ts             # SSE hook for dashboard real-time
│   │
│   └── lib/
│       └── trpc/
│           ├── client.ts                   # tRPC React client
│           └── server.ts                   # tRPC server caller
│
├── packages/
│   ├── evaluation/                          # Shared evaluation logic
│   │   ├── package.json
│   │   ├── tsconfig.json
│   │   └── src/
│   │       ├── index.ts
│   │       ├── murmur3.ts
│   │       ├── evaluate.ts
│   │       ├── types.ts
│   │       └── __tests__/
│   │           ├── murmur3.test.ts
│   │           └── evaluate.test.ts
│   │
│   ├── sdk-js/                              # JavaScript client SDK
│   │   ├── package.json                     # @shipflag/js
│   │   ├── tsconfig.json
│   │   ├── tsup.config.ts
│   │   ├── README.md
│   │   └── src/
│   │       ├── index.ts                     # ShipFlag class
│   │       ├── types.ts
│   │       ├── storage.ts                   # localStorage cache
│   │       └── sse.ts                       # EventSource wrapper with reconnect
│   │
│   ├── sdk-react/                           # React SDK
│   │   ├── package.json                     # @shipflag/react
│   │   ├── tsconfig.json
│   │   ├── tsup.config.ts
│   │   ├── README.md
│   │   └── src/
│   │       ├── index.ts                     # Re-exports
│   │       ├── provider.tsx                 # ShipFlagProvider
│   │       ├── hooks.ts                     # useFlag, useStringFlag, useFlags
│   │       ├── context.ts                   # React context
│   │       └── server.ts                    # getFlags() for RSC
│   │
│   └── sdk-node/                            # Node.js server SDK
│       ├── package.json                     # @shipflag/node
│       ├── tsconfig.json
│       ├── tsup.config.ts
│       ├── README.md
│       └── src/
│           ├── index.ts                     # ShipFlagServer class
│           ├── types.ts
│           ├── polling.ts                   # Fallback polling logic
│           └── persistence.ts              # Disk cache for rules
│
└── drizzle/                                 # Generated migrations
    └── 0000_init.sql
```

---

## Verification

### Automated Tests

| Test | Command | Validates |
|------|---------|-----------|
| Evaluation unit tests | `npx vitest run src/server/evaluation/` | murmur3 correctness, rule evaluation order, percentage distribution, condition operators, kill switch, default fallback |
| Evaluation package tests | `cd packages/evaluation && npx vitest run` | Shared evaluation logic matches server-side |
| SDK endpoint integration | `npx vitest run src/app/api/sdk/` | Client/server key auth, CORS, rate limiting, response format |
| tRPC router tests | `npx vitest run src/server/trpc/` | Flag CRUD, toggle, rules update, audit log creation, plan enforcement |
| React SDK tests | `cd packages/sdk-react && npx vitest run` | Provider renders, hooks return flag values, SSE updates trigger re-render |
| E2E (Playwright) | `npx playwright test` | Full flow: sign up -> create flag -> toggle -> SDK receives update |

### Manual Verification Checklist

- [ ] Create account via Clerk, org auto-provisions with 3 environments and 6 API keys
- [ ] Create a boolean flag "test-flag", verify it appears in list with all env toggles OFF
- [ ] Toggle "test-flag" ON in Development, verify audit log entry created
- [ ] Hit `GET /api/sdk/flags` with dev client key + userId header, verify `{ "test-flag": { key: "on", value: true } }`
- [ ] Hit `GET /api/sdk/flags` with staging client key, verify `{ "test-flag": { key: "off", value: false } }` (still off in staging)
- [ ] Set percentage rollout to 50% in dev, hit SDK endpoint with 100 different userIds, verify approximately 50% get `true`
- [ ] Add user ID targeting rule for "user-42", verify that user always gets `true` regardless of percentage
- [ ] Open SSE stream, toggle flag in dashboard, verify SSE event arrives within 1 second
- [ ] Rotate API key, verify old key returns 401, new key works
- [ ] Test evaluation panel: enter userId "user-42", verify explanation shows "User ID targeting matched"
- [ ] Test evaluation panel: enter userId "user-99" with 0% rollout, verify explanation shows "No rules matched, returning default"
- [ ] Create 5 flags on Free plan, verify 6th flag creation is blocked with upgrade prompt
- [ ] Upgrade to Pro via Stripe, verify flag limit removed
- [ ] Install `@shipflag/react` in a test Next.js app, verify `useFlag` returns correct value and updates in real-time
- [ ] Install `@shipflag/node` in a test Express app, verify `isEnabled` evaluates locally (no network per call)
- [ ] Server SDK: disconnect network, verify cached rules still evaluate correctly
- [ ] Client SDK: disconnect network, verify localStorage flags still work
- [ ] Verify murmur3 stickiness: same userId+flagKey produces identical result across JS SDK, React SDK, Node SDK, and server evaluation
- [ ] Promote config from staging to production, verify both envs show identical rules
- [ ] Archive a flag, verify it disappears from list (but exists in audit log)
- [ ] Production toggle shows confirmation dialog before allowing change
- [ ] Stale badge appears on flags that are 100% enabled in prod and unchanged for 30+ days
