# 7. FeedLoop — Customer Feedback Aggregator

## Implementation Plan

**MVP Scope:** Intercom + Zendesk integration (OAuth), manual feedback entry, AI categorization and tagging (GPT-4o), basic deduplication via embedding similarity (pgvector + text-embedding-3-small), feature request board (kanban + ranked list), priority scoring (frequency + recency), email notifications on status changes, Stripe billing with three tiers.

---

## Tech Decisions

| Layer | Technology | Notes |
|-------|-----------|-------|
| Framework | Next.js 14+ App Router | `src/` directory, TypeScript strict |
| API | tRPC v11 | superjson transformer, Clerk auth context |
| Database | Neon PostgreSQL + pgvector | `vector(1536)` columns for embedding similarity |
| ORM | Drizzle ORM | Push-based migrations, typed schema |
| Auth | Clerk | Organization-scoped, `orgId` from session |
| Billing | Stripe | Checkout Sessions + Customer Portal + Webhooks |
| Email | Resend | Transactional status-change notifications |
| AI — Categorization | OpenAI GPT-4o | JSON mode, temperature 0.1 for classification |
| AI — Embeddings | text-embedding-3-small | 1536 dimensions, cosine similarity |
| Queue | BullMQ + Redis (Upstash) | Async AI processing for incoming feedback |
| Integrations | Intercom API v2 (OAuth 2.0), Zendesk API v2 (OAuth 2.0) | Webhook receivers + polling sync |
| UI | Tailwind CSS, Radix UI, @hello-pangea/dnd (kanban), Recharts | |
| Hosting | Vercel (app), Upstash (Redis) | Serverless-compatible BullMQ via `@upstash/qstash` fallback |
| Monitoring | Sentry, Vercel Analytics | |

### Key Architecture Decisions

1. **Embedding-based deduplication over keyword matching**: New feedback items are embedded on ingestion. A pgvector cosine similarity query finds existing FeatureRequests above a 0.82 threshold. Matches are surfaced as "AI suggested match" in the inbox — the user confirms or dismisses. This avoids false auto-merges while reducing manual triage.

2. **BullMQ for async AI processing**: Webhook receivers and manual submissions return 200 immediately, enqueue the item, and a worker processes categorization + embedding + dedup matching. This keeps webhook response times under 500ms and handles rate limits gracefully with exponential backoff.

3. **Priority scoring as a materialized computation**: Priority scores are recomputed on-demand (when viewing the board) using a SQL query with configurable weights stored in `organizations.scoring_config`. No cron job needed — the score is a derived value.

4. **OAuth token storage**: Intercom/Zendesk OAuth tokens are stored AES-256-GCM encrypted in the `feedback_sources.config` JSONB column. The encryption key is a separate env var (`INTEGRATION_ENCRYPTION_KEY`).

---

## Database Schema (Drizzle)

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
  plan: text("plan").default("starter").notNull(), // starter | growth | enterprise
  stripeCustomerId: text("stripe_customer_id"),
  stripeSubscriptionId: text("stripe_subscription_id"),
  scoringConfig: jsonb("scoring_config").default({
    frequencyWeight: 0.35,
    recencyWeight: 0.30,
    sentimentWeight: 0.20,
    arrWeight: 0.15,
    recencyHalfLifeDays: 30,
  }).$type<{
    frequencyWeight: number;
    recencyWeight: number;
    sentimentWeight: number;
    arrWeight: number;
    recencyHalfLifeDays: number;
  }>(),
  settings: jsonb("settings").default({}).$type<{
    notificationFromName?: string;
    notificationFromEmail?: string;
    brandColor?: string;
  }>(),
  createdAt: timestamp("created_at", { withTimezone: true }).defaultNow().notNull(),
  updatedAt: timestamp("updated_at", { withTimezone: true }).defaultNow().notNull(),
});

// ---------------------------------------------------------------------------
// Feedback Sources (Intercom, Zendesk, Manual)
// ---------------------------------------------------------------------------
export const feedbackSources = pgTable("feedback_sources", {
  id: uuid("id").primaryKey().defaultRandom(),
  orgId: uuid("org_id")
    .references(() => organizations.id, { onDelete: "cascade" })
    .notNull(),
  type: text("type").notNull(), // intercom | zendesk | manual | api
  name: text("name").notNull(), // "Intercom — Production", "Zendesk Support"
  config: jsonb("config").default({}).$type<{
    // Intercom OAuth
    accessToken?: string;       // AES-256-GCM encrypted
    intercomAppId?: string;
    webhookSecret?: string;
    syncTags?: string[];        // Only import conversations with these tags
    // Zendesk OAuth
    zendeskSubdomain?: string;
    zendeskAccessToken?: string; // AES-256-GCM encrypted
    zendeskWebhookToken?: string;
    syncTicketTags?: string[];
    syncTicketFieldId?: string;
  }>(),
  status: text("status").default("active").notNull(), // active | paused | error
  errorMessage: text("error_message"),
  lastSyncedAt: timestamp("last_synced_at", { withTimezone: true }),
  createdAt: timestamp("created_at", { withTimezone: true }).defaultNow().notNull(),
});

// ---------------------------------------------------------------------------
// Customers
// ---------------------------------------------------------------------------
export const customers = pgTable(
  "customers",
  {
    id: uuid("id").primaryKey().defaultRandom(),
    orgId: uuid("org_id")
      .references(() => organizations.id, { onDelete: "cascade" })
      .notNull(),
    name: text("name"),
    email: text("email"),
    company: text("company"),
    arr: real("arr").default(0), // Annual Recurring Revenue in dollars
    planTier: text("plan_tier"),
    healthScore: real("health_score"), // 0-100, nullable until computed
    totalFeedbackCount: integer("total_feedback_count").default(0).notNull(),
    externalIds: jsonb("external_ids").default({}).$type<{
      intercomContactId?: string;
      intercomCompanyId?: string;
      zendeskUserId?: string;
      zendeskOrgId?: string;
    }>(),
    createdAt: timestamp("created_at", { withTimezone: true }).defaultNow().notNull(),
    updatedAt: timestamp("updated_at", { withTimezone: true }).defaultNow().notNull(),
  },
  (table) => [
    index("customers_org_id_idx").on(table.orgId),
    index("customers_email_idx").on(table.orgId, table.email),
  ]
);

// ---------------------------------------------------------------------------
// Feedback Items
// ---------------------------------------------------------------------------
export const feedbackItems = pgTable(
  "feedback_items",
  {
    id: uuid("id").primaryKey().defaultRandom(),
    orgId: uuid("org_id")
      .references(() => organizations.id, { onDelete: "cascade" })
      .notNull(),
    sourceId: uuid("source_id")
      .references(() => feedbackSources.id, { onDelete: "set null" }),
    customerId: uuid("customer_id")
      .references(() => customers.id, { onDelete: "set null" }),
    rawText: text("raw_text").notNull(),
    extractedAsk: text("extracted_ask"), // AI-extracted core request
    category: text("category"), // feature_request | bug | praise | complaint | question
    sentiment: text("sentiment"), // positive | neutral | negative
    sentimentScore: real("sentiment_score"), // -1.0 to 1.0
    tags: jsonb("tags").default([]).$type<string[]>(), // ["onboarding", "billing", "api"]
    featureRequestId: uuid("feature_request_id")
      .references(() => featureRequests.id, { onDelete: "set null" }),
    sourceUrl: text("source_url"), // Link back to original Intercom/Zendesk conversation
    sourceExternalId: text("source_external_id"), // intercom_conversation_id or zendesk_ticket_id
    sourceMetadata: jsonb("source_metadata").default({}).$type<{
      intercomConversationId?: string;
      zendeskTicketId?: string;
      submittedByName?: string;
    }>(),
    submittedByUserId: text("submitted_by_user_id"), // Clerk userId of team member who logged it
    aiProcessed: boolean("ai_processed").default(false).notNull(),
    aiConfidence: real("ai_confidence"), // 0-1 confidence of categorization
    dismissed: boolean("dismissed").default(false).notNull(),
    // embedding vector(1536) — added via raw SQL migration
    createdAt: timestamp("created_at", { withTimezone: true }).defaultNow().notNull(),
    updatedAt: timestamp("updated_at", { withTimezone: true }).defaultNow().notNull(),
  },
  (table) => [
    index("feedback_items_org_id_idx").on(table.orgId),
    index("feedback_items_source_id_idx").on(table.sourceId),
    index("feedback_items_customer_id_idx").on(table.customerId),
    index("feedback_items_feature_request_id_idx").on(table.featureRequestId),
    index("feedback_items_category_idx").on(table.orgId, table.category),
    index("feedback_items_created_at_idx").on(table.createdAt),
  ]
);

// ---------------------------------------------------------------------------
// Feature Requests (clustered feedback)
// ---------------------------------------------------------------------------
export const featureRequests = pgTable(
  "feature_requests",
  {
    id: uuid("id").primaryKey().defaultRandom(),
    orgId: uuid("org_id")
      .references(() => organizations.id, { onDelete: "cascade" })
      .notNull(),
    title: text("title").notNull(),
    description: text("description"),
    status: text("status").default("new").notNull(),
    // new | under_review | planned | in_progress | shipped | wont_do
    priorityScore: real("priority_score").default(0).notNull(),
    manualEffortEstimate: text("manual_effort_estimate"), // xs | s | m | l | xl
    requestCount: integer("request_count").default(0).notNull(),
    totalArrRequesting: real("total_arr_requesting").default(0).notNull(),
    kanbanOrder: integer("kanban_order").default(0).notNull(),
    shippedAt: timestamp("shipped_at", { withTimezone: true }),
    closedAt: timestamp("closed_at", { withTimezone: true }),
    createdAt: timestamp("created_at", { withTimezone: true }).defaultNow().notNull(),
    updatedAt: timestamp("updated_at", { withTimezone: true }).defaultNow().notNull(),
  },
  (table) => [
    index("feature_requests_org_id_idx").on(table.orgId),
    index("feature_requests_status_idx").on(table.orgId, table.status),
    index("feature_requests_priority_idx").on(table.orgId, table.priorityScore),
  ]
);

// ---------------------------------------------------------------------------
// Feature Request Activity (status changes, comments)
// ---------------------------------------------------------------------------
export const featureRequestActivity = pgTable("feature_request_activity", {
  id: uuid("id").primaryKey().defaultRandom(),
  featureRequestId: uuid("feature_request_id")
    .references(() => featureRequests.id, { onDelete: "cascade" })
    .notNull(),
  orgId: uuid("org_id")
    .references(() => organizations.id, { onDelete: "cascade" })
    .notNull(),
  type: text("type").notNull(), // status_change | comment | linked_feedback | unlinked_feedback
  userId: text("user_id"), // Clerk userId
  fromStatus: text("from_status"),
  toStatus: text("to_status"),
  comment: text("comment"),
  metadata: jsonb("metadata").default({}).$type<{
    feedbackItemId?: string;
    customerName?: string;
  }>(),
  createdAt: timestamp("created_at", { withTimezone: true }).defaultNow().notNull(),
});

// ---------------------------------------------------------------------------
// Notifications
// ---------------------------------------------------------------------------
export const notifications = pgTable(
  "notifications",
  {
    id: uuid("id").primaryKey().defaultRandom(),
    orgId: uuid("org_id")
      .references(() => organizations.id, { onDelete: "cascade" })
      .notNull(),
    featureRequestId: uuid("feature_request_id")
      .references(() => featureRequests.id, { onDelete: "cascade" })
      .notNull(),
    customerId: uuid("customer_id")
      .references(() => customers.id, { onDelete: "cascade" }),
    recipientEmail: text("recipient_email").notNull(),
    type: text("type").notNull(), // status_change | shipped
    channel: text("channel").default("email").notNull(), // email
    status: text("status").default("pending").notNull(), // pending | sent | failed
    sentAt: timestamp("sent_at", { withTimezone: true }),
    errorMessage: text("error_message"),
    metadata: jsonb("metadata").default({}).$type<{
      featureTitle?: string;
      oldStatus?: string;
      newStatus?: string;
      resendMessageId?: string;
    }>(),
    createdAt: timestamp("created_at", { withTimezone: true }).defaultNow().notNull(),
  },
  (table) => [
    index("notifications_org_id_idx").on(table.orgId),
    index("notifications_feature_request_id_idx").on(table.featureRequestId),
    index("notifications_status_idx").on(table.status),
  ]
);

// ---------------------------------------------------------------------------
// Relations
// ---------------------------------------------------------------------------
export const organizationsRelations = relations(organizations, ({ many }) => ({
  feedbackSources: many(feedbackSources),
  customers: many(customers),
  feedbackItems: many(feedbackItems),
  featureRequests: many(featureRequests),
  notifications: many(notifications),
}));

export const feedbackSourcesRelations = relations(feedbackSources, ({ one, many }) => ({
  organization: one(organizations, {
    fields: [feedbackSources.orgId],
    references: [organizations.id],
  }),
  feedbackItems: many(feedbackItems),
}));

export const customersRelations = relations(customers, ({ one, many }) => ({
  organization: one(organizations, {
    fields: [customers.orgId],
    references: [organizations.id],
  }),
  feedbackItems: many(feedbackItems),
  notifications: many(notifications),
}));

export const feedbackItemsRelations = relations(feedbackItems, ({ one }) => ({
  organization: one(organizations, {
    fields: [feedbackItems.orgId],
    references: [organizations.id],
  }),
  source: one(feedbackSources, {
    fields: [feedbackItems.sourceId],
    references: [feedbackSources.id],
  }),
  customer: one(customers, {
    fields: [feedbackItems.customerId],
    references: [customers.id],
  }),
  featureRequest: one(featureRequests, {
    fields: [feedbackItems.featureRequestId],
    references: [featureRequests.id],
  }),
}));

export const featureRequestsRelations = relations(featureRequests, ({ one, many }) => ({
  organization: one(organizations, {
    fields: [featureRequests.orgId],
    references: [organizations.id],
  }),
  feedbackItems: many(feedbackItems),
  activity: many(featureRequestActivity),
  notifications: many(notifications),
}));

export const featureRequestActivityRelations = relations(
  featureRequestActivity,
  ({ one }) => ({
    featureRequest: one(featureRequests, {
      fields: [featureRequestActivity.featureRequestId],
      references: [featureRequests.id],
    }),
    organization: one(organizations, {
      fields: [featureRequestActivity.orgId],
      references: [organizations.id],
    }),
  })
);

export const notificationsRelations = relations(notifications, ({ one }) => ({
  organization: one(organizations, {
    fields: [notifications.orgId],
    references: [organizations.id],
  }),
  featureRequest: one(featureRequests, {
    fields: [notifications.featureRequestId],
    references: [featureRequests.id],
  }),
  customer: one(customers, {
    fields: [notifications.customerId],
    references: [customers.id],
  }),
}));

// ---------------------------------------------------------------------------
// Type Exports
// ---------------------------------------------------------------------------
export type Organization = typeof organizations.$inferSelect;
export type NewOrganization = typeof organizations.$inferInsert;
export type FeedbackSource = typeof feedbackSources.$inferSelect;
export type NewFeedbackSource = typeof feedbackSources.$inferInsert;
export type Customer = typeof customers.$inferSelect;
export type NewCustomer = typeof customers.$inferInsert;
export type FeedbackItem = typeof feedbackItems.$inferSelect;
export type NewFeedbackItem = typeof feedbackItems.$inferInsert;
export type FeatureRequest = typeof featureRequests.$inferSelect;
export type NewFeatureRequest = typeof featureRequests.$inferInsert;
export type Notification = typeof notifications.$inferSelect;
export type NewNotification = typeof notifications.$inferInsert;
export type FeatureRequestActivityRow = typeof featureRequestActivity.$inferSelect;
```

### Initial Migration SQL

```sql
-- src/server/db/migrations/0000_init.sql

-- Enable required extensions
CREATE EXTENSION IF NOT EXISTS vector;
CREATE EXTENSION IF NOT EXISTS pg_trgm;

-- (All tables created by Drizzle push, then manually add vector column:)
ALTER TABLE feedback_items ADD COLUMN IF NOT EXISTS embedding vector(1536);

-- HNSW index for fast cosine similarity search on feedback embeddings
CREATE INDEX IF NOT EXISTS feedback_items_embedding_idx
  ON feedback_items USING hnsw (embedding vector_cosine_ops)
  WITH (m = 16, ef_construction = 64);

-- Trigram index for fuzzy text search on raw feedback
CREATE INDEX IF NOT EXISTS feedback_items_raw_text_trgm_idx
  ON feedback_items USING gin (raw_text gin_trgm_ops);

-- Trigram index for searching feature request titles
CREATE INDEX IF NOT EXISTS feature_requests_title_trgm_idx
  ON feature_requests USING gin (title gin_trgm_ops);
```

---

## Phase Breakdown (8 weeks) — Day-by-Day

### Phase 1: Foundation (Week 1) — Project Setup, Auth, Database, Core Models

**Day 1 — Project scaffold + environment**
- `npx create-next-app@latest feedloop --typescript --tailwind --app --src-dir`
- Install dependencies: `drizzle-orm @neondatabase/serverless @clerk/nextjs @trpc/server @trpc/client @trpc/next superjson zod openai stripe resend`
- Install dev dependencies: `drizzle-kit @types/node`
- Configure `src/env.ts` with Zod validation:
  ```typescript
  const envSchema = z.object({
    DATABASE_URL: z.string().min(1),
    CLERK_SECRET_KEY: z.string().min(1),
    NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY: z.string().min(1),
    STRIPE_SECRET_KEY: z.string().min(1),
    STRIPE_WEBHOOK_SECRET: z.string().min(1),
    NEXT_PUBLIC_STRIPE_PUBLISHABLE_KEY: z.string().min(1),
    RESEND_API_KEY: z.string().min(1),
    OPENAI_API_KEY: z.string().min(1),
    UPSTASH_REDIS_URL: z.string().url(),
    UPSTASH_REDIS_TOKEN: z.string().min(1),
    INTERCOM_CLIENT_ID: z.string().min(1),
    INTERCOM_CLIENT_SECRET: z.string().min(1),
    ZENDESK_CLIENT_ID: z.string().min(1),
    ZENDESK_CLIENT_SECRET: z.string().min(1),
    INTEGRATION_ENCRYPTION_KEY: z.string().length(64), // 32-byte hex
    NEXT_PUBLIC_APP_URL: z.string().url(),
  });
  ```
- Create Neon database, enable pgvector extension
- Configure `drizzle.config.ts` with Neon connection string

**Day 2 — Database schema + Clerk auth**
- Write full Drizzle schema (as above) in `src/server/db/schema.ts`
- Create `src/server/db/index.ts` with Neon serverless driver
- Run `drizzle-kit push` to apply schema
- Run migration SQL for vector column + indexes
- Set up Clerk organization: `src/middleware.ts` with `clerkMiddleware()`, protect `/dashboard/*` routes
- Create `src/app/layout.tsx` with `<ClerkProvider>`
- Test: verify Clerk sign-in works, org creation works

**Day 3 — tRPC foundation + org resolution**
- Create `src/server/trpc/trpc.ts`: init tRPC with superjson, auth middleware, org middleware (identical pattern to DriftLog)
- Create `src/server/trpc/context.ts`: pull `auth()` from Clerk, inject `db`
- Create `src/app/api/trpc/[trpc]/route.ts`: Next.js route handler
- Create `src/lib/trpc/client.ts`: React Query client with superjson
- Create `src/lib/trpc/server.ts`: server-side caller
- Build plan-limit middleware:
  ```typescript
  const PLAN_LIMITS = {
    starter: { maxSources: 3, maxItemsPerMonth: 500, maxUsers: 1 },
    growth: { maxSources: Infinity, maxItemsPerMonth: 5000, maxUsers: 5 },
    enterprise: { maxSources: Infinity, maxItemsPerMonth: Infinity, maxUsers: Infinity },
  } as const;
  ```
- Create `src/server/trpc/routers/_app.ts`: merge routers

**Day 4 — Organization + FeedbackSource tRPC routers**
- `src/server/trpc/routers/organization.ts`:
  - `getOrCreate`: find by clerkOrgId or create new
  - `update`: name, settings, scoringConfig
  - `getScoringConfig`: return current weights
  - `updateScoringConfig`: validate weights sum to 1.0
- `src/server/trpc/routers/feedbackSource.ts`:
  - `list`: all sources for org
  - `create`: type + name + config
  - `update`: config, status
  - `delete`: cascade cleanup
  - `testConnection`: verify OAuth token still valid
- Create `src/server/integrations/encryption.ts`:
  ```typescript
  import crypto from "crypto";

  const ALGORITHM = "aes-256-gcm";
  const IV_LENGTH = 16;
  const AUTH_TAG_LENGTH = 16;

  export function encrypt(plaintext: string): string {
    const key = Buffer.from(process.env.INTEGRATION_ENCRYPTION_KEY!, "hex");
    const iv = crypto.randomBytes(IV_LENGTH);
    const cipher = crypto.createCipheriv(ALGORITHM, key, iv);
    let encrypted = cipher.update(plaintext, "utf8", "hex");
    encrypted += cipher.final("hex");
    const authTag = cipher.getAuthTag();
    return iv.toString("hex") + ":" + authTag.toString("hex") + ":" + encrypted;
  }

  export function decrypt(ciphertext: string): string {
    const key = Buffer.from(process.env.INTEGRATION_ENCRYPTION_KEY!, "hex");
    const [ivHex, authTagHex, encryptedHex] = ciphertext.split(":");
    const iv = Buffer.from(ivHex, "hex");
    const authTag = Buffer.from(authTagHex, "hex");
    const decipher = crypto.createDecipheriv(ALGORITHM, key, iv);
    decipher.setAuthTag(authTag);
    let decrypted = decipher.update(encryptedHex, "hex", "utf8");
    decrypted += decipher.final("utf8");
    return decrypted;
  }
  ```

**Day 5 — Base UI: layout, navigation, dashboard shell**
- Install UI deps: `@radix-ui/react-dialog @radix-ui/react-dropdown-menu @radix-ui/react-tabs @radix-ui/react-select @radix-ui/react-tooltip @radix-ui/react-popover class-variance-authority clsx tailwind-merge lucide-react`
- Create shared UI components in `src/components/ui/`: button, card, badge, input, textarea, select, dialog, dropdown-menu, tabs, toast, skeleton
- Build app shell layout: `src/app/dashboard/layout.tsx`
  - Left sidebar: logo, nav links (Inbox, Feature Requests, Analytics, Settings), org switcher, user menu
  - Main content area with header breadcrumb
- Create stub pages:
  - `src/app/dashboard/page.tsx` (overview)
  - `src/app/dashboard/inbox/page.tsx`
  - `src/app/dashboard/requests/page.tsx`
  - `src/app/dashboard/analytics/page.tsx`
  - `src/app/dashboard/settings/page.tsx`
  - `src/app/dashboard/settings/sources/page.tsx`
  - `src/app/dashboard/settings/billing/page.tsx`

---

### Phase 2: Integrations (Week 2) — Intercom + Zendesk OAuth, Webhook Receivers, Manual Entry

**Day 6 — Intercom OAuth flow**
- Create `src/app/api/auth/intercom/route.ts`:
  ```typescript
  // GET — redirect to Intercom OAuth authorization
  export async function GET(req: NextRequest) {
    const orgId = req.nextUrl.searchParams.get("orgId");
    const state = crypto.randomBytes(16).toString("hex");
    // Store state in Redis with orgId for CSRF verification
    await redis.set(`intercom_oauth:${state}`, orgId, { ex: 600 });

    const url = new URL("https://app.intercom.com/oauth");
    url.searchParams.set("client_id", process.env.INTERCOM_CLIENT_ID!);
    url.searchParams.set("redirect_uri", `${process.env.NEXT_PUBLIC_APP_URL}/api/auth/intercom/callback`);
    url.searchParams.set("state", state);

    return NextResponse.redirect(url.toString());
  }
  ```
- Create `src/app/api/auth/intercom/callback/route.ts`:
  ```typescript
  // GET — exchange code for access token
  export async function GET(req: NextRequest) {
    const code = req.nextUrl.searchParams.get("code");
    const state = req.nextUrl.searchParams.get("state");

    const orgId = await redis.get(`intercom_oauth:${state}`);
    if (!orgId) return NextResponse.redirect("/dashboard/settings/sources?error=invalid_state");

    const tokenRes = await fetch("https://api.intercom.io/auth/eagle/token", {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({
        client_id: process.env.INTERCOM_CLIENT_ID,
        client_secret: process.env.INTERCOM_CLIENT_SECRET,
        code,
      }),
    });
    const { token: accessToken } = await tokenRes.json();

    // Fetch Intercom app info to get app_id
    const meRes = await fetch("https://api.intercom.io/me", {
      headers: { Authorization: `Bearer ${accessToken}` },
    });
    const meData = await meRes.json();

    // Store encrypted token as a new FeedbackSource
    await db.insert(feedbackSources).values({
      orgId,
      type: "intercom",
      name: `Intercom — ${meData.app?.name ?? "Connected"}`,
      config: {
        accessToken: encrypt(accessToken),
        intercomAppId: meData.app?.id_code,
      },
      status: "active",
    });

    return NextResponse.redirect("/dashboard/settings/sources?success=intercom");
  }
  ```

**Day 7 — Zendesk OAuth flow**
- Create `src/app/api/auth/zendesk/route.ts`:
  ```typescript
  // GET — redirect to Zendesk OAuth authorization
  export async function GET(req: NextRequest) {
    const orgId = req.nextUrl.searchParams.get("orgId");
    const subdomain = req.nextUrl.searchParams.get("subdomain");
    const state = crypto.randomBytes(16).toString("hex");
    await redis.set(`zendesk_oauth:${state}`, JSON.stringify({ orgId, subdomain }), { ex: 600 });

    const url = new URL(`https://${subdomain}.zendesk.com/oauth/authorizations/new`);
    url.searchParams.set("response_type", "code");
    url.searchParams.set("client_id", process.env.ZENDESK_CLIENT_ID!);
    url.searchParams.set("redirect_uri", `${process.env.NEXT_PUBLIC_APP_URL}/api/auth/zendesk/callback`);
    url.searchParams.set("scope", "read tickets:read users:read organizations:read");
    url.searchParams.set("state", state);

    return NextResponse.redirect(url.toString());
  }
  ```
- Create `src/app/api/auth/zendesk/callback/route.ts`: exchange code → token, store encrypted in `feedback_sources`
- Zendesk token exchange uses `https://{subdomain}.zendesk.com/oauth/tokens` with `grant_type=authorization_code`

**Day 8 — Intercom webhook receiver**
- Register webhook subscription via Intercom API: `POST https://api.intercom.io/subscriptions` for topics `conversation.closed`, `conversation_part.tag.created`
- Create `src/app/api/webhooks/intercom/route.ts`:
  ```typescript
  export async function POST(req: NextRequest) {
    const payload = await req.text();
    const signature = req.headers.get("x-hub-signature");

    // Verify HMAC-SHA1 signature
    // Intercom uses the client secret as the HMAC key
    const expected = "sha1=" + crypto
      .createHmac("sha1", process.env.INTERCOM_CLIENT_SECRET!)
      .update(payload)
      .digest("hex");

    if (!signature || !crypto.timingSafeEqual(
      Buffer.from(expected), Buffer.from(signature)
    )) {
      return NextResponse.json({ error: "Invalid signature" }, { status: 401 });
    }

    const body = JSON.parse(payload);
    const topic = body.topic;

    switch (topic) {
      case "conversation.closed":
      case "conversation_part.tag.created":
        await enqueueIntercomFeedback(body);
        break;
    }

    return NextResponse.json({ ok: true });
  }

  async function enqueueIntercomFeedback(body: IntercomWebhookPayload) {
    const conversationId = body.data?.item?.id;
    const appId = body.app_id;

    // Find the feedback source by intercom app ID
    const [source] = await db
      .select()
      .from(feedbackSources)
      .where(
        and(
          eq(feedbackSources.type, "intercom"),
          sql`${feedbackSources.config}->>'intercomAppId' = ${appId}`
        )
      )
      .limit(1);

    if (!source || source.status !== "active") return;

    // Fetch full conversation text via Intercom API
    const accessToken = decrypt(source.config!.accessToken!);
    const convRes = await fetch(
      `https://api.intercom.io/conversations/${conversationId}`,
      { headers: { Authorization: `Bearer ${accessToken}`, Accept: "application/json" } }
    );
    const conv = await convRes.json();

    // Extract customer-authored parts (skip admin/bot messages)
    const customerParts = conv.conversation_parts?.conversation_parts
      ?.filter((p: any) => p.author?.type === "user" || p.author?.type === "lead")
      ?.map((p: any) => p.body?.replace(/<[^>]+>/g, "") ?? "")
      ?.join("\n\n") ?? "";

    const rawText = [conv.source?.body?.replace(/<[^>]+>/g, ""), customerParts]
      .filter(Boolean).join("\n\n");

    if (!rawText.trim()) return;

    // Enqueue for async AI processing
    await feedbackQueue.add("process-feedback", {
      orgId: source.orgId,
      sourceId: source.id,
      sourceType: "intercom",
      rawText,
      sourceExternalId: conversationId,
      sourceUrl: `https://app.intercom.com/a/apps/${appId}/inbox/conversation/${conversationId}`,
      customerExternalId: conv.source?.author?.id,
      customerEmail: conv.source?.author?.email,
      customerName: conv.source?.author?.name,
    });
  }
  ```

**Day 9 — Zendesk webhook receiver**
- Create `src/app/api/webhooks/zendesk/route.ts`:
  ```typescript
  export async function POST(req: NextRequest) {
    const payload = await req.text();
    const body = JSON.parse(payload);

    // Zendesk webhook verification: check Bearer token in Authorization header
    // (set when configuring the webhook in Zendesk admin)
    const authHeader = req.headers.get("authorization");
    const token = authHeader?.replace("Bearer ", "");

    // Find matching source by webhook token
    const sources = await db
      .select()
      .from(feedbackSources)
      .where(eq(feedbackSources.type, "zendesk"));

    const source = sources.find((s) => {
      const config = s.config as any;
      return config?.zendeskWebhookToken === token;
    });

    if (!source) {
      return NextResponse.json({ error: "Invalid token" }, { status: 401 });
    }

    // Zendesk sends ticket data; extract latest public comment
    const ticketId = body.ticket_id ?? body.id;
    const accessToken = decrypt(source.config!.zendeskAccessToken!);
    const subdomain = source.config!.zendeskSubdomain!;

    // Fetch ticket with comments
    const ticketRes = await fetch(
      `https://${subdomain}.zendesk.com/api/v2/tickets/${ticketId}/comments`,
      { headers: { Authorization: `Bearer ${accessToken}` } }
    );
    const ticketData = await ticketRes.json();

    // Collect end-user comments (not agent responses)
    const userComments = ticketData.comments
      ?.filter((c: any) => c.author_id && !c.via?.source?.from?.address?.includes("@"))
      ?.map((c: any) => c.plain_body ?? c.body?.replace(/<[^>]+>/g, ""))
      ?.filter(Boolean)
      ?.join("\n\n") ?? "";

    if (!userComments.trim()) return NextResponse.json({ ok: true });

    // Fetch requester info
    const requesterId = body.requester_id;
    let customerEmail = "";
    let customerName = "";
    if (requesterId) {
      const userRes = await fetch(
        `https://${subdomain}.zendesk.com/api/v2/users/${requesterId}`,
        { headers: { Authorization: `Bearer ${accessToken}` } }
      );
      const userData = await userRes.json();
      customerEmail = userData.user?.email ?? "";
      customerName = userData.user?.name ?? "";
    }

    await feedbackQueue.add("process-feedback", {
      orgId: source.orgId,
      sourceId: source.id,
      sourceType: "zendesk",
      rawText: userComments,
      sourceExternalId: String(ticketId),
      sourceUrl: `https://${subdomain}.zendesk.com/agent/tickets/${ticketId}`,
      customerExternalId: String(requesterId),
      customerEmail,
      customerName,
    });

    return NextResponse.json({ ok: true });
  }
  ```

**Day 10 — Manual feedback entry + Customer resolution**
- `src/server/trpc/routers/feedback.ts`:
  - `submitManual`: accepts `{ rawText, customerEmail?, customerName?, customerCompany? }`, creates FeedbackItem, enqueues for AI processing
  - `list`: paginated inbox with filters (source, category, sentiment, date range, linked/unlinked)
  - `getById`: single item with customer + source + featureRequest relations
  - `linkToRequest`: set `featureRequestId`, update request counts
  - `unlinkFromRequest`: clear link, update request counts
  - `dismiss`: soft-dismiss from inbox
  - `bulkLink`: link multiple items to a request
- Create `src/server/services/customer-resolver.ts`:
  ```typescript
  /**
   * Find-or-create a customer by email + externalIds.
   * Merges external IDs from multiple sources onto the same customer record.
   */
  export async function resolveCustomer(opts: {
    orgId: string;
    email?: string;
    name?: string;
    company?: string;
    sourceType: "intercom" | "zendesk" | "manual";
    externalId?: string;
  }): Promise<string | null> {
    if (!opts.email && !opts.externalId) return null;

    // Try to find by email first
    if (opts.email) {
      const [existing] = await db
        .select()
        .from(customers)
        .where(and(eq(customers.orgId, opts.orgId), eq(customers.email, opts.email)))
        .limit(1);

      if (existing) {
        // Merge external IDs
        const ids = existing.externalIds ?? {};
        if (opts.sourceType === "intercom" && opts.externalId) {
          ids.intercomContactId = opts.externalId;
        } else if (opts.sourceType === "zendesk" && opts.externalId) {
          ids.zendeskUserId = opts.externalId;
        }
        await db.update(customers)
          .set({ externalIds: ids, name: opts.name ?? existing.name, updatedAt: new Date() })
          .where(eq(customers.id, existing.id));
        return existing.id;
      }
    }

    // Create new customer
    const externalIds: Record<string, string> = {};
    if (opts.sourceType === "intercom" && opts.externalId) {
      externalIds.intercomContactId = opts.externalId;
    } else if (opts.sourceType === "zendesk" && opts.externalId) {
      externalIds.zendeskUserId = opts.externalId;
    }

    const [created] = await db.insert(customers).values({
      orgId: opts.orgId,
      email: opts.email ?? null,
      name: opts.name ?? null,
      company: opts.company ?? null,
      externalIds,
    }).returning();

    return created.id;
  }
  ```

---

### Phase 3: AI Pipeline (Week 3) — Categorization, Embedding, Deduplication

**Day 11 — BullMQ queue setup + worker skeleton**
- Create `src/server/queue/connection.ts`:
  ```typescript
  import { Redis } from "ioredis";
  export const redis = new Redis(process.env.UPSTASH_REDIS_URL!, {
    maxRetriesPerRequest: null, // required by BullMQ
  });
  ```
- Create `src/server/queue/feedback-queue.ts`:
  ```typescript
  import { Queue, Worker, Job } from "bullmq";
  import { redis } from "./connection";

  export interface FeedbackJobData {
    orgId: string;
    sourceId: string | null;
    sourceType: "intercom" | "zendesk" | "manual" | "api";
    rawText: string;
    sourceExternalId?: string;
    sourceUrl?: string;
    customerExternalId?: string;
    customerEmail?: string;
    customerName?: string;
    customerCompany?: string;
  }

  export const feedbackQueue = new Queue<FeedbackJobData>("feedback-processing", {
    connection: redis,
    defaultJobOptions: {
      attempts: 3,
      backoff: { type: "exponential", delay: 2000 },
      removeOnComplete: { age: 86400 }, // 24h
      removeOnFail: { age: 604800 },    // 7d
    },
  });
  ```
- Create `src/server/queue/feedback-worker.ts` (runs as a separate process or in a long-running API route for dev):
  ```typescript
  const worker = new Worker<FeedbackJobData>(
    "feedback-processing",
    async (job: Job<FeedbackJobData>) => {
      const data = job.data;

      // Step 1: Resolve customer
      const customerId = await resolveCustomer({ ... });

      // Step 2: AI categorize + extract
      const aiResult = await categorizeFeedback(data.rawText);

      // Step 3: Generate embedding
      const embedding = await generateFeedbackEmbedding(data.rawText, aiResult.extractedAsk);

      // Step 4: Create feedback item
      const [item] = await db.insert(feedbackItems).values({ ... }).returning();

      // Step 5: Store embedding via raw SQL
      await storeEmbedding(item.id, embedding);

      // Step 6: Find dedup matches
      const matches = await findSimilarRequests(data.orgId, embedding);

      // Step 7: If high-confidence match (>0.88), auto-link. If medium (0.82-0.88), set as suggestion.
      if (matches.length > 0 && matches[0].similarity >= 0.88) {
        await linkFeedbackToRequest(item.id, matches[0].featureRequestId);
      }
      // Medium matches are stored in source_metadata for UI display

      // Step 8: Update customer feedback count
      if (customerId) {
        await db.update(customers)
          .set({ totalFeedbackCount: sql`total_feedback_count + 1` })
          .where(eq(customers.id, customerId));
      }
    },
    { connection: redis, concurrency: 5 }
  );
  ```

**Day 12 — AI categorization prompt + implementation**
- Create `src/server/ai/categorize-feedback.ts`:
  ```typescript
  import OpenAI from "openai";

  const openai = new OpenAI({ apiKey: process.env.OPENAI_API_KEY });

  export interface CategorizationResult {
    category: "feature_request" | "bug" | "praise" | "complaint" | "question";
    extractedAsk: string;          // The core request in 1-2 sentences
    sentiment: "positive" | "neutral" | "negative";
    sentimentScore: number;         // -1.0 to 1.0
    tags: string[];                // Product area tags
    confidence: number;             // 0-1 overall confidence
  }

  const CATEGORIZATION_PROMPT = `You are a customer feedback analysis system for a B2B SaaS product. Analyze the following customer feedback and return a structured JSON response.

  RULES:
  - "category" must be exactly one of: "feature_request", "bug", "praise", "complaint", "question"
  - "extracted_ask" must be a concise 1-2 sentence summary of what the customer is asking for or reporting. If it's praise, summarize what they liked. If it's a bug, summarize the issue.
  - "sentiment" must be "positive", "neutral", or "negative"
  - "sentiment_score" is a float from -1.0 (extremely negative) to 1.0 (extremely positive)
  - "tags" is an array of 1-4 product area tags (e.g., "onboarding", "billing", "api", "mobile", "integrations", "reporting", "authentication", "performance", "ui", "documentation"). Use lowercase, single-word or hyphenated tags.
  - "confidence" is how confident you are in the categorization, from 0.0 to 1.0

  EXAMPLES:
  - "I wish you guys had a dark mode, my eyes hurt at night" → feature_request, "Add dark mode theme option", negative, -0.3, ["ui"], 0.95
  - "The login keeps failing when I use SSO" → bug, "SSO login failures", negative, -0.6, ["authentication"], 0.92
  - "Love the new dashboard redesign!" → praise, "Positive feedback on dashboard redesign", positive, 0.8, ["ui", "reporting"], 0.97

  CUSTOMER FEEDBACK:
  ---
  {feedback_text}
  ---

  Return ONLY valid JSON: {"category": "...", "extracted_ask": "...", "sentiment": "...", "sentiment_score": 0.0, "tags": [...], "confidence": 0.0}`;

  export async function categorizeFeedback(rawText: string): Promise<CategorizationResult> {
    const truncated = rawText.slice(0, 4000);
    const prompt = CATEGORIZATION_PROMPT.replace("{feedback_text}", truncated);

    const response = await openai.chat.completions.create({
      model: "gpt-4o",
      messages: [{ role: "user", content: prompt }],
      temperature: 0.1,
      max_tokens: 300,
      response_format: { type: "json_object" },
    });

    const content = response.choices[0]?.message?.content ?? "{}";
    let parsed: Record<string, unknown>;
    try {
      parsed = JSON.parse(content);
    } catch {
      return {
        category: "question",
        extractedAsk: rawText.slice(0, 200),
        sentiment: "neutral",
        sentimentScore: 0,
        tags: [],
        confidence: 0,
      };
    }

    const validCategories = ["feature_request", "bug", "praise", "complaint", "question"];
    const validSentiments = ["positive", "neutral", "negative"];

    return {
      category: validCategories.includes(parsed.category as string)
        ? (parsed.category as CategorizationResult["category"])
        : "question",
      extractedAsk: typeof parsed.extracted_ask === "string"
        ? parsed.extracted_ask
        : rawText.slice(0, 200),
      sentiment: validSentiments.includes(parsed.sentiment as string)
        ? (parsed.sentiment as CategorizationResult["sentiment"])
        : "neutral",
      sentimentScore: typeof parsed.sentiment_score === "number"
        ? Math.max(-1, Math.min(1, parsed.sentiment_score))
        : 0,
      tags: Array.isArray(parsed.tags)
        ? parsed.tags.filter((t: unknown) => typeof t === "string").slice(0, 4)
        : [],
      confidence: typeof parsed.confidence === "number"
        ? Math.max(0, Math.min(1, parsed.confidence))
        : 0.5,
    };
  }
  ```

**Day 13 — Embedding generation + vector storage**
- Create `src/server/ai/embeddings.ts`:
  ```typescript
  import OpenAI from "openai";
  import { neon } from "@neondatabase/serverless";

  const openai = new OpenAI({ apiKey: process.env.OPENAI_API_KEY });
  const MAX_INPUT_CHARS = 8000;

  /**
   * Build embedding input from raw feedback + extracted ask.
   * The extracted ask is weighted by appearing first.
   */
  function buildEmbeddingInput(rawText: string, extractedAsk: string | null): string {
    const parts: string[] = [];
    if (extractedAsk) parts.push(extractedAsk);
    parts.push(rawText);
    return parts.join("\n\n").slice(0, MAX_INPUT_CHARS);
  }

  export async function generateFeedbackEmbedding(
    rawText: string,
    extractedAsk: string | null
  ): Promise<number[]> {
    const input = buildEmbeddingInput(rawText, extractedAsk);
    const response = await openai.embeddings.create({
      model: "text-embedding-3-small",
      input,
      dimensions: 1536,
    });
    return response.data[0].embedding;
  }

  export async function storeEmbedding(
    feedbackItemId: string,
    embedding: number[]
  ): Promise<void> {
    const sql = neon(process.env.DATABASE_URL!);
    const vectorStr = `[${embedding.join(",")}]`;
    await sql`UPDATE feedback_items SET embedding = ${vectorStr}::vector WHERE id = ${feedbackItemId}`;
  }
  ```

**Day 14 — Deduplication matching via pgvector**
- Create `src/server/ai/deduplication.ts`:
  ```typescript
  import { neon } from "@neondatabase/serverless";

  /**
   * Similarity thresholds:
   * - >= 0.88: Auto-link (high confidence duplicate)
   * - 0.82 - 0.87: Suggest match in UI (medium confidence)
   * - < 0.82: No match
   */
  const AUTO_LINK_THRESHOLD = 0.88;
  const SUGGEST_THRESHOLD = 0.82;

  export interface DedupMatch {
    featureRequestId: string;
    featureRequestTitle: string;
    similarity: number;
    requestCount: number;
  }

  /**
   * Find feature requests with feedback items that are semantically similar
   * to the given embedding. Returns top 5 matches above the suggestion threshold.
   *
   * Strategy: We compute average similarity across all feedback embeddings
   * linked to each feature request, then rank by best match.
   */
  export async function findSimilarRequests(
    orgId: string,
    embedding: number[]
  ): Promise<DedupMatch[]> {
    const sql = neon(process.env.DATABASE_URL!);
    const vectorStr = `[${embedding.join(",")}]`;

    // Find the most similar existing feedback items that are linked to feature requests,
    // then aggregate by feature request
    const results = await sql`
      WITH similar_items AS (
        SELECT
          fi.feature_request_id,
          1 - (fi.embedding <=> ${vectorStr}::vector) AS similarity
        FROM feedback_items fi
        WHERE fi.org_id = ${orgId}
          AND fi.feature_request_id IS NOT NULL
          AND fi.embedding IS NOT NULL
        ORDER BY fi.embedding <=> ${vectorStr}::vector
        LIMIT 50
      )
      SELECT
        si.feature_request_id,
        fr.title AS feature_request_title,
        MAX(si.similarity) AS similarity,
        fr.request_count
      FROM similar_items si
      JOIN feature_requests fr ON fr.id = si.feature_request_id
      WHERE si.similarity >= ${SUGGEST_THRESHOLD}
      GROUP BY si.feature_request_id, fr.title, fr.request_count
      ORDER BY MAX(si.similarity) DESC
      LIMIT 5
    `;

    return results.map((r: any) => ({
      featureRequestId: r.feature_request_id,
      featureRequestTitle: r.feature_request_title,
      similarity: parseFloat(r.similarity),
      requestCount: parseInt(r.request_count),
    }));
  }

  export { AUTO_LINK_THRESHOLD, SUGGEST_THRESHOLD };
  ```

**Day 15 — Wire up full worker pipeline + test end-to-end**
- Complete the `feedback-worker.ts` implementation connecting all stages
- Create `src/server/services/link-feedback.ts`:
  ```typescript
  /**
   * Link a feedback item to a feature request.
   * Updates request_count and total_arr_requesting on the feature request.
   */
  export async function linkFeedbackToRequest(
    feedbackItemId: string,
    featureRequestId: string
  ): Promise<void> {
    const [item] = await db
      .select()
      .from(feedbackItems)
      .where(eq(feedbackItems.id, feedbackItemId))
      .limit(1);

    if (!item) return;

    // Update the feedback item
    await db.update(feedbackItems)
      .set({ featureRequestId, updatedAt: new Date() })
      .where(eq(feedbackItems.id, feedbackItemId));

    // Recount linked items for this feature request
    const [countResult] = await db
      .select({
        count: sql<number>`count(*)`,
        totalArr: sql<number>`COALESCE(SUM(c.arr), 0)`,
      })
      .from(feedbackItems)
      .leftJoin(customers, eq(feedbackItems.customerId, customers.id))
      .where(eq(feedbackItems.featureRequestId, featureRequestId));

    await db.update(featureRequests).set({
      requestCount: Number(countResult.count),
      totalArrRequesting: Number(countResult.totalArr),
      updatedAt: new Date(),
    }).where(eq(featureRequests.id, featureRequestId));
  }
  ```
- Test with manual feedback submission: verify AI categorization, embedding storage, and dedup matching
- Test with Intercom webhook payload (use ngrok for local development)

---

### Phase 4: Feature Request Board (Week 4) — Kanban, List View, Priority Scoring

**Day 16 — Feature request tRPC router**
- `src/server/trpc/routers/featureRequest.ts`:
  - `list`: paginated, filterable by status, sortable by priority_score or request_count or created_at
  - `getById`: full detail with linked feedback items, customer list, activity log
  - `create`: from inbox (with initial linked feedback item) or blank
  - `update`: title, description, status, manual_effort_estimate
  - `updateStatus`: with state machine validation + activity log + notification trigger
  - `reorderKanban`: update `kanban_order` for drag-and-drop
  - `getBoard`: grouped by status for kanban view
  - `delete`: soft delete or hard delete with unlink

**Day 17 — Kanban state machine + status change logic**
- Create `src/server/services/status-machine.ts`:
  ```typescript
  /**
   * Valid status transitions for feature requests.
   * Enforced on every status change.
   */
  const VALID_TRANSITIONS: Record<string, string[]> = {
    new:          ["under_review", "planned", "wont_do"],
    under_review: ["new", "planned", "in_progress", "wont_do"],
    planned:      ["under_review", "in_progress", "wont_do"],
    in_progress:  ["planned", "shipped", "wont_do"],
    shipped:      ["in_progress"],  // Can reopen if premature
    wont_do:      ["new", "under_review"],  // Can reconsider
  };

  export function isValidTransition(from: string, to: string): boolean {
    return VALID_TRANSITIONS[from]?.includes(to) ?? false;
  }

  export function getAvailableTransitions(currentStatus: string): string[] {
    return VALID_TRANSITIONS[currentStatus] ?? [];
  }

  export const STATUS_LABELS: Record<string, string> = {
    new: "New",
    under_review: "Under Review",
    planned: "Planned",
    in_progress: "In Progress",
    shipped: "Shipped",
    wont_do: "Won't Do",
  };

  export const STATUS_COLORS: Record<string, string> = {
    new: "gray",
    under_review: "yellow",
    planned: "blue",
    in_progress: "purple",
    shipped: "green",
    wont_do: "red",
  };
  ```
- Implement `updateStatus` in tRPC router:
  ```typescript
  updateStatus: orgProcedure
    .input(z.object({
      id: z.string().uuid(),
      status: z.enum(["new", "under_review", "planned", "in_progress", "shipped", "wont_do"]),
    }))
    .mutation(async ({ ctx, input }) => {
      const [request] = await ctx.db.select()
        .from(featureRequests)
        .where(and(eq(featureRequests.id, input.id), eq(featureRequests.orgId, ctx.org.id)))
        .limit(1);

      if (!request) throw new TRPCError({ code: "NOT_FOUND" });
      if (!isValidTransition(request.status, input.status)) {
        throw new TRPCError({
          code: "BAD_REQUEST",
          message: `Cannot transition from ${request.status} to ${input.status}`,
        });
      }

      const updates: Record<string, unknown> = {
        status: input.status,
        updatedAt: new Date(),
      };
      if (input.status === "shipped") updates.shippedAt = new Date();
      if (input.status === "wont_do") updates.closedAt = new Date();

      await ctx.db.update(featureRequests).set(updates).where(eq(featureRequests.id, input.id));

      // Log activity
      await ctx.db.insert(featureRequestActivity).values({
        featureRequestId: input.id,
        orgId: ctx.org.id,
        type: "status_change",
        userId: ctx.auth.userId,
        fromStatus: request.status,
        toStatus: input.status,
      });

      // Trigger notifications if status is a customer-facing milestone
      if (["planned", "in_progress", "shipped"].includes(input.status)) {
        await enqueueStatusNotifications(ctx.org.id, input.id, request.status, input.status);
      }

      return { success: true };
    }),
  ```

**Day 18 — Priority scoring SQL**
- Create `src/server/services/priority-scoring.ts`:
  ```typescript
  /**
   * Priority Score Formula:
   *
   * score = W_freq * normalized_frequency
   *       + W_recency * recency_factor
   *       + W_sentiment * sentiment_urgency
   *       + W_arr * normalized_arr
   *
   * Where:
   * - normalized_frequency = request_count / max(request_count) across all requests in org
   * - recency_factor = avg(exp(-days_since_feedback / half_life)) across linked feedback items
   * - sentiment_urgency = avg(1 - (sentiment_score + 1) / 2) — inverted so negative = higher urgency
   * - normalized_arr = total_arr_requesting / max(total_arr_requesting) across all requests in org
   *
   * All factors are 0-1 normalized. Final score is 0-100.
   */

  export async function computePriorityScores(orgId: string): Promise<void> {
    const sql = neon(process.env.DATABASE_URL!);

    await sql`
      WITH org_config AS (
        SELECT
          COALESCE((scoring_config->>'frequencyWeight')::float, 0.35) AS w_freq,
          COALESCE((scoring_config->>'recencyWeight')::float, 0.30) AS w_recency,
          COALESCE((scoring_config->>'sentimentWeight')::float, 0.20) AS w_sentiment,
          COALESCE((scoring_config->>'arrWeight')::float, 0.15) AS w_arr,
          COALESCE((scoring_config->>'recencyHalfLifeDays')::float, 30) AS half_life
        FROM organizations
        WHERE id = ${orgId}
      ),
      request_stats AS (
        SELECT
          fr.id,
          fr.request_count,
          fr.total_arr_requesting,
          -- Recency: average exponential decay across linked feedback items
          COALESCE(AVG(
            EXP(-EXTRACT(EPOCH FROM (now() - fi.created_at)) / (86400 * oc.half_life))
          ), 0) AS recency_factor,
          -- Sentiment urgency: average of inverted sentiment (negative feedback = higher urgency)
          COALESCE(AVG(
            1.0 - (COALESCE(fi.sentiment_score, 0) + 1.0) / 2.0
          ), 0.5) AS sentiment_urgency
        FROM feature_requests fr
        CROSS JOIN org_config oc
        LEFT JOIN feedback_items fi ON fi.feature_request_id = fr.id
        WHERE fr.org_id = ${orgId}
          AND fr.status NOT IN ('shipped', 'wont_do')
        GROUP BY fr.id, fr.request_count, fr.total_arr_requesting
      ),
      max_vals AS (
        SELECT
          GREATEST(MAX(request_count), 1) AS max_count,
          GREATEST(MAX(total_arr_requesting), 1) AS max_arr
        FROM request_stats
      )
      UPDATE feature_requests fr
      SET
        priority_score = ROUND((
          oc.w_freq * (rs.request_count::float / mv.max_count)
          + oc.w_recency * rs.recency_factor
          + oc.w_sentiment * rs.sentiment_urgency
          + oc.w_arr * (rs.total_arr_requesting::float / mv.max_arr)
        ) * 100, 2),
        updated_at = now()
      FROM request_stats rs, max_vals mv, org_config oc
      WHERE fr.id = rs.id
    `;
  }
  ```
- Add a `computeScores` endpoint to the featureRequest router that triggers recomputation
- Priority scores are recomputed on: board page load (cached 5min), manual trigger, after feedback linking

**Day 19 — Kanban board UI**
- Install: `@hello-pangea/dnd` (maintained fork of react-beautiful-dnd)
- Create `src/components/board/kanban-board.tsx`:
  - `DragDropContext` with columns for each status (excluding `wont_do` — separate section)
  - Each column renders `Droppable` > list of `Draggable` cards
  - Card shows: title, request count badge, total ARR, top 3 customer avatars, priority score indicator
  - On drag end: call `featureRequest.reorderKanban` mutation for same-column reorder, or `featureRequest.updateStatus` for cross-column move
- Create `src/components/board/request-card.tsx`: compact card with hover actions
- Create `src/components/board/board-toolbar.tsx`: toggle between kanban/list view, search, filters

**Day 20 — List view + detail panel**
- Create `src/components/board/list-view.tsx`:
  - Table with columns: Title, Status (badge), Priority Score, Requests (#), ARR, Last Activity
  - Sortable by clicking column headers
  - Row click opens side panel or navigates to detail page
- Create `src/app/dashboard/requests/[id]/page.tsx`:
  - Three tabs: Evidence, Customers, Activity
  - **Evidence tab**: list of linked feedback items with source icon, verbatim quote, category badge, date
  - **Customers tab**: list of requesting customers with email, company, ARR, plan tier
  - **Activity tab**: timeline of status changes, comments, linked/unlinked feedback
  - Status dropdown with valid transitions
  - "Link feedback" button (opens search dialog)
  - "Create from feedback" action in inbox links back here

---

### Phase 5: Feedback Inbox UI (Week 5) — Stream, Triage, Linking Workflow

**Day 21 — Feedback inbox page**
- Create `src/app/dashboard/inbox/page.tsx`:
  - Infinite scroll list of feedback items, newest first
  - Each item card shows:
    - Source icon (Intercom logo, Zendesk logo, pencil for manual)
    - Customer name + company (if available)
    - Raw text (truncated to 3 lines, expandable)
    - AI category badge (color-coded): Feature Request (blue), Bug (red), Praise (green), Complaint (orange), Question (gray)
    - Sentiment indicator (colored dot: green/yellow/red)
    - AI extracted ask (italic, below raw text)
    - Tags as small pills
    - "AI Suggested Match" banner if dedup match found (>= 0.82 similarity):
      ```
      🔗 Similar to: "Dark mode support" (47 requests) — Link | Dismiss
      ```
    - Quick actions: Link to Request, Create New Request, Dismiss
  - Filter sidebar: Source dropdown, Category checkboxes, Sentiment radio, Date range, Linked/Unlinked toggle
  - Search bar with text search across raw_text and extracted_ask

**Day 22 — Link-to-request dialog + create-from-feedback flow**
- Create `src/components/inbox/link-dialog.tsx`:
  - Search existing feature requests by title (fuzzy match via pg_trgm)
  - Show AI-suggested matches at top (pre-populated from dedup results)
  - Each result shows: title, request count, status badge
  - "Create New Request" button at bottom if no match
  - On select: call `feedback.linkToRequest`, update request counts, close dialog
- Create `src/components/inbox/create-request-dialog.tsx`:
  - Pre-populate title from `extracted_ask`
  - Pre-populate description from `raw_text`
  - Status defaults to "new"
  - On submit: create feature request, link current feedback item, redirect to request detail
- Create `src/components/inbox/feedback-card.tsx`: reusable card component used in inbox and request detail evidence tab

**Day 23 — Bulk actions + inbox refinements**
- Add multi-select checkboxes to inbox items
- Bulk actions toolbar: "Link all to..." (opens link dialog), "Dismiss selected", "Categorize as..."
- Add "AI confidence" indicator: low confidence items (<0.7) show a subtle warning
- Add keyboard shortcuts: `j`/`k` to navigate, `l` to open link dialog, `d` to dismiss, `n` to create new request
- Implement text search using pg_trgm:
  ```sql
  SELECT * FROM feedback_items
  WHERE org_id = $1
    AND (raw_text % $2 OR extracted_ask % $2)
  ORDER BY similarity(raw_text, $2) DESC
  LIMIT 50;
  ```

**Day 24 — Manual feedback entry form**
- Create `src/app/dashboard/inbox/new/page.tsx`:
  - Textarea for feedback text (required)
  - Customer lookup: type-ahead search by email/name, or create new
  - Source defaults to "manual"
  - Optional: link to existing request on creation
  - Submit enqueues for AI processing, then redirects to inbox with new item highlighted
- Create `src/components/inbox/customer-autocomplete.tsx`:
  - Searches customers by name or email
  - Shows matching customers with company + ARR
  - "Create new customer" option at bottom

**Day 25 — Sources settings page**
- Create `src/app/dashboard/settings/sources/page.tsx`:
  - List of connected sources with status indicators (green dot = active, red = error, gray = paused)
  - For each source: type icon, name, last synced, item count, actions (pause/resume, edit, delete)
  - "Connect Intercom" button → OAuth flow
  - "Connect Zendesk" button → asks for subdomain first, then OAuth flow
  - Connection status test button
  - Error messages displayed inline when a source is in error state
- Wire up source CRUD to `feedbackSource` tRPC router

---

### Phase 6: Notifications + Analytics (Week 6)

**Day 26 — Email notification system**
- Create `src/server/services/notifications.ts`:
  ```typescript
  import { Resend } from "resend";
  const resend = new Resend(process.env.RESEND_API_KEY);

  export async function sendStatusChangeNotification(opts: {
    orgId: string;
    featureRequestId: string;
    featureTitle: string;
    oldStatus: string;
    newStatus: string;
  }): Promise<void> {
    // Find all customers linked to this feature request
    const linkedCustomers = await db
      .select({ email: customers.email, name: customers.name })
      .from(feedbackItems)
      .innerJoin(customers, eq(feedbackItems.customerId, customers.id))
      .where(
        and(
          eq(feedbackItems.featureRequestId, opts.featureRequestId),
          sql`${customers.email} IS NOT NULL`
        )
      )
      .groupBy(customers.id, customers.email, customers.name);

    const [org] = await db.select().from(organizations)
      .where(eq(organizations.id, opts.orgId)).limit(1);

    for (const customer of linkedCustomers) {
      if (!customer.email) continue;

      // Create notification record
      const [notification] = await db.insert(notifications).values({
        orgId: opts.orgId,
        featureRequestId: opts.featureRequestId,
        recipientEmail: customer.email,
        type: "status_change",
        channel: "email",
        status: "pending",
        metadata: {
          featureTitle: opts.featureTitle,
          oldStatus: opts.oldStatus,
          newStatus: opts.newStatus,
        },
      }).returning();

      try {
        const result = await resend.emails.send({
          from: `${org?.settings?.notificationFromName ?? org?.name ?? "FeedLoop"} <notifications@feedloop.io>`,
          to: customer.email,
          subject: getSubjectLine(opts.featureTitle, opts.newStatus),
          html: buildStatusChangeEmail({
            customerName: customer.name ?? "there",
            featureTitle: opts.featureTitle,
            newStatus: opts.newStatus,
            orgName: org?.name ?? "Our Team",
            brandColor: org?.settings?.brandColor ?? "#6366f1",
          }),
        });

        await db.update(notifications).set({
          status: "sent",
          sentAt: new Date(),
          metadata: { ...notification.metadata, resendMessageId: result.data?.id },
        }).where(eq(notifications.id, notification.id));
      } catch (error) {
        await db.update(notifications).set({
          status: "failed",
          errorMessage: error instanceof Error ? error.message : "Unknown error",
        }).where(eq(notifications.id, notification.id));
      }
    }
  }

  function getSubjectLine(featureTitle: string, newStatus: string): string {
    switch (newStatus) {
      case "planned": return `📋 "${featureTitle}" is now planned`;
      case "in_progress": return `🚧 "${featureTitle}" is now being built`;
      case "shipped": return `🚀 "${featureTitle}" has shipped!`;
      default: return `Update on "${featureTitle}"`;
    }
  }

  function buildStatusChangeEmail(opts: {
    customerName: string;
    featureTitle: string;
    newStatus: string;
    orgName: string;
    brandColor: string;
  }): string {
    const statusMessages: Record<string, string> = {
      planned: "We've added this to our roadmap and plan to build it.",
      in_progress: "Our team is actively working on this right now.",
      shipped: "Great news — this feature is now live! Check it out.",
    };

    return `
      <div style="font-family: -apple-system, system-ui, sans-serif; max-width: 560px; margin: 0 auto;">
        <div style="background: ${opts.brandColor}; padding: 24px; border-radius: 8px 8px 0 0;">
          <h1 style="color: white; margin: 0; font-size: 18px;">${opts.orgName}</h1>
        </div>
        <div style="padding: 24px; border: 1px solid #e5e7eb; border-top: none; border-radius: 0 0 8px 8px;">
          <p>Hi ${opts.customerName},</p>
          <p>You asked us about: <strong>${opts.featureTitle}</strong></p>
          <p>${statusMessages[opts.newStatus] ?? `The status has been updated to: ${opts.newStatus}`}</p>
          <p>Thank you for your feedback — it helps us build a better product.</p>
          <p style="color: #6b7280; font-size: 14px;">— The ${opts.orgName} team</p>
        </div>
      </div>
    `;
  }
  ```

**Day 27 — Notification tRPC router + UI**
- `src/server/trpc/routers/notification.ts`:
  - `list`: paginated notifications for org, filterable by type, status
  - `getStats`: total sent, failed, pending counts
- Create `src/app/dashboard/requests/[id]/notifications-tab.tsx`:
  - List of notifications sent for this request
  - Status badges: Pending (yellow), Sent (green), Failed (red)
  - "Resend failed" button
  - "Send update" manual trigger button
- Wire status change notifications into `featureRequest.updateStatus` mutation

**Day 28 — Analytics dashboard**
- Install: `recharts`
- Create `src/server/trpc/routers/analytics.ts`:
  - `feedbackVolume`: feedback items per day/week for the last 30/90 days, grouped by source
  - `topRequests`: top 10 feature requests by priority score
  - `sentimentTrend`: average sentiment score per week
  - `categoryBreakdown`: count of items by category
  - `sourceBreakdown`: count of items by source type
  - `overviewStats`: total feedback items, total requests, avg priority score, items this month
- Create `src/app/dashboard/analytics/page.tsx`:
  - Overview cards row: Total Feedback (this month), Open Requests, Avg Priority Score, Top Request name
  - Feedback volume line chart (Recharts `AreaChart` with source colors)
  - Top 10 requests bar chart (horizontal, sorted by score)
  - Sentiment trend line chart
  - Source breakdown pie chart
  - Category breakdown donut chart

**Day 29 — Dashboard overview page**
- Create `src/app/dashboard/page.tsx`:
  - Welcome header with org name
  - Quick stats row: Unprocessed items, Feature requests in progress, Items this week, Notifications sent
  - "Recent Feedback" — last 5 items with quick-link actions
  - "Needs Attention" — items with AI-suggested matches waiting for triage
  - "Top Requested" — top 5 by priority score with sparkline trends
  - Quick action buttons: "Add Feedback", "Connect Source"

---

### Phase 7: Billing + Settings (Week 7)

**Day 30 — Stripe billing setup**
- Create `src/server/billing/stripe.ts`:
  ```typescript
  import Stripe from "stripe";

  const stripe = new Stripe(process.env.STRIPE_SECRET_KEY!, {
    apiVersion: "2025-04-30.basil",
  });

  const PRICE_IDS: Record<string, string> = {
    starter: process.env.STRIPE_STARTER_PRICE_ID ?? "price_starter_placeholder",
    growth: process.env.STRIPE_GROWTH_PRICE_ID ?? "price_growth_placeholder",
    enterprise: process.env.STRIPE_ENTERPRISE_PRICE_ID ?? "price_enterprise_placeholder",
  };

  export const PLANS = {
    starter: {
      name: "Starter",
      price: 49,
      maxSources: 3,
      maxItemsPerMonth: 500,
      maxUsers: 1,
      features: [
        "3 feedback sources",
        "500 feedback items/month",
        "AI categorization & tagging",
        "Basic deduplication",
        "Feature request board",
        "Priority scoring",
        "Email notifications",
      ],
    },
    growth: {
      name: "Growth",
      price: 99,
      maxSources: Infinity,
      maxItemsPerMonth: 5000,
      maxUsers: 5,
      features: [
        "Unlimited feedback sources",
        "5,000 feedback items/month",
        "Advanced deduplication & clustering",
        "Priority scoring with custom weights",
        "Customer voting board",
        "Jira/Linear integration",
        "5 team members",
        "Email notifications",
      ],
    },
    enterprise: {
      name: "Enterprise",
      price: 249,
      maxSources: Infinity,
      maxItemsPerMonth: Infinity,
      maxUsers: Infinity,
      features: [
        "Unlimited everything",
        "CRM integration (revenue scoring)",
        "Advanced analytics",
        "API access",
        "SSO",
        "Unlimited team members",
        "Dedicated support",
      ],
    },
  } as const;

  export async function createCheckoutSession(opts: {
    orgId: string;
    clerkOrgId: string;
    stripeCustomerId?: string;
    plan: "starter" | "growth" | "enterprise";
  }): Promise<string> {
    const appUrl = process.env.NEXT_PUBLIC_APP_URL!;

    const sessionParams: Stripe.Checkout.SessionCreateParams = {
      mode: "subscription",
      payment_method_types: ["card"],
      line_items: [{ price: PRICE_IDS[opts.plan], quantity: 1 }],
      success_url: `${appUrl}/dashboard/settings/billing?success=true`,
      cancel_url: `${appUrl}/dashboard/settings/billing?canceled=true`,
      metadata: { orgId: opts.orgId, clerkOrgId: opts.clerkOrgId, plan: opts.plan },
      subscription_data: {
        metadata: { orgId: opts.orgId, clerkOrgId: opts.clerkOrgId, plan: opts.plan },
      },
    };

    if (opts.stripeCustomerId) {
      sessionParams.customer = opts.stripeCustomerId;
    }

    const session = await stripe.checkout.sessions.create(sessionParams);
    return session.url!;
  }

  export async function createPortalSession(stripeCustomerId: string): Promise<string> {
    const session = await stripe.billingPortal.sessions.create({
      customer: stripeCustomerId,
      return_url: `${process.env.NEXT_PUBLIC_APP_URL}/dashboard/settings/billing`,
    });
    return session.url;
  }
  ```

**Day 31 — Stripe webhook handler**
- Create `src/app/api/webhooks/stripe/route.ts`:
  ```typescript
  export async function POST(req: NextRequest) {
    const payload = await req.text();
    const signature = req.headers.get("stripe-signature")!;

    let event: Stripe.Event;
    try {
      event = stripe.webhooks.constructEvent(
        payload,
        signature,
        process.env.STRIPE_WEBHOOK_SECRET!
      );
    } catch {
      return NextResponse.json({ error: "Invalid signature" }, { status: 400 });
    }

    switch (event.type) {
      case "checkout.session.completed": {
        const session = event.data.object as Stripe.Checkout.Session;
        const { orgId, plan } = session.metadata!;
        await db.update(organizations).set({
          plan,
          stripeCustomerId: session.customer as string,
          stripeSubscriptionId: session.subscription as string,
          updatedAt: new Date(),
        }).where(eq(organizations.id, orgId));
        break;
      }
      case "customer.subscription.updated": {
        const sub = event.data.object as Stripe.Subscription;
        const orgId = sub.metadata?.orgId;
        const plan = sub.metadata?.plan;
        if (orgId && plan) {
          await db.update(organizations).set({
            plan,
            updatedAt: new Date(),
          }).where(eq(organizations.id, orgId));
        }
        break;
      }
      case "customer.subscription.deleted": {
        const sub = event.data.object as Stripe.Subscription;
        const orgId = sub.metadata?.orgId;
        if (orgId) {
          await db.update(organizations).set({
            plan: "starter",
            stripeSubscriptionId: null,
            updatedAt: new Date(),
          }).where(eq(organizations.id, orgId));
        }
        break;
      }
    }

    return NextResponse.json({ ok: true });
  }
  ```

**Day 32 — Billing tRPC router + UI**
- `src/server/trpc/routers/billing.ts`:
  - `getCurrentPlan`: return org plan + limits + usage (items this month, sources count, users count)
  - `createCheckout`: generate Stripe checkout URL for plan upgrade
  - `createPortal`: generate Stripe portal URL for managing subscription
- Create `src/app/dashboard/settings/billing/page.tsx`:
  - Current plan card with usage meters (items used / limit, sources used / limit)
  - Three plan cards: Starter, Growth, Enterprise with feature lists
  - "Upgrade" or "Manage Subscription" buttons
  - Success/canceled URL parameter handling with toast messages

**Day 33 — Settings pages**
- Create `src/app/dashboard/settings/page.tsx`:
  - Organization name and slug (editable)
  - Brand color picker
  - Notification from name/email
- Create `src/app/dashboard/settings/scoring/page.tsx`:
  - Four sliders for scoring weights (frequency, recency, sentiment, ARR)
  - Visual constraint: weights must sum to 1.0 (auto-adjust others when one changes)
  - Half-life days input for recency decay
  - "Preview" button: show current top 10 requests with new weights applied
  - "Save" button: persist to `organizations.scoring_config`
- Create `src/app/dashboard/settings/members/page.tsx` (stub for Growth+ plans):
  - Team member list from Clerk organization
  - Invite form
  - Role badges (admin, member)

---

### Phase 8: Polish + Launch (Week 8)

**Day 34 — Plan enforcement + rate limiting**
- Implement `itemLimitMiddleware` in tRPC:
  ```typescript
  export const itemLimitMiddleware = middleware(async ({ ctx, next }) => {
    const { org, planLimits } = ctx as {
      org: Organization;
      planLimits: typeof PLAN_LIMITS[keyof typeof PLAN_LIMITS];
    };

    if (planLimits.maxItemsPerMonth === Infinity) return next();

    const startOfMonth = new Date();
    startOfMonth.setDate(1);
    startOfMonth.setHours(0, 0, 0, 0);

    const [result] = await ctx.db
      .select({ count: sql<number>`count(*)` })
      .from(feedbackItems)
      .where(
        and(
          eq(feedbackItems.orgId, org.id),
          sql`${feedbackItems.createdAt} >= ${startOfMonth.toISOString()}`
        )
      );

    if (Number(result.count) >= planLimits.maxItemsPerMonth) {
      throw new TRPCError({
        code: "FORBIDDEN",
        message: `Your ${org.plan} plan allows ${planLimits.maxItemsPerMonth} feedback items per month. Please upgrade.`,
      });
    }

    return next();
  });
  ```
- Apply limit middleware to feedback submission and webhook processing
- Source count enforcement on `feedbackSource.create`
- Add usage banner in dashboard when approaching limits (>80%)

**Day 35 — Onboarding flow**
- Create `src/app/onboarding/page.tsx`:
  - Step 1: Create/name organization
  - Step 2: Connect first source (Intercom, Zendesk, or "Start with manual entry")
  - Step 3: Submit first feedback item (pre-filled example or paste real feedback)
  - Step 4: See AI categorization result → "Your inbox is ready!"
  - Progress indicator (4 steps)
  - Skip option to go straight to dashboard
- Store onboarding completion in `organizations.settings.onboardingCompleted`
- Redirect new orgs to onboarding if not completed

**Day 36 — Loading states, error handling, empty states**
- Add Suspense boundaries with skeleton loaders for all data-fetching pages
- Create empty state illustrations:
  - Inbox empty: "No feedback yet — connect a source or add feedback manually"
  - Board empty: "No feature requests — triage your inbox to create requests"
  - Analytics empty: "Need more data — analytics appear after 10+ feedback items"
- Toast notifications for all mutations (success/error)
- Global error boundary with retry
- 404 page for invalid routes
- tRPC error handling: map error codes to user-friendly messages

**Day 37 — Responsive design + accessibility**
- Mobile-responsive sidebar: collapsible on small screens, bottom nav on mobile
- Kanban board: horizontal scroll on mobile, or switch to list view automatically
- Inbox: single-column layout on mobile
- All interactive elements: focus rings, keyboard navigation, ARIA labels
- Color contrast verification for all status badges and category badges
- Screen reader text for icon-only buttons

**Day 38 — Performance optimization + SEO**
- Add `loading.tsx` files for route-level streaming
- Optimize database queries: ensure all list queries use indexed columns
- Add React Query `staleTime` and `gcTime` settings:
  - Board data: staleTime 30s
  - Inbox: staleTime 10s (near real-time)
  - Analytics: staleTime 5min
  - Scoring config: staleTime Infinity (manual invalidation)
- Server-side rendering for dashboard pages using tRPC server caller
- Landing page SEO: meta tags, Open Graph, structured data
- Vercel Analytics integration

**Day 39 — Landing page + documentation**
- Create `src/app/page.tsx`:
  - Hero: "Hear what your customers actually want" with CTA
  - How it works: 3-step visual (Connect → AI analyzes → Prioritize)
  - Features grid: Integrations, AI Categorization, Deduplication, Priority Scoring, Notifications
  - Pricing table with three tiers
  - Testimonial placeholder
  - Footer with links
- Create `src/app/pricing/page.tsx`: detailed pricing comparison table

**Day 40 — Final testing + deployment**
- End-to-end test flow:
  1. Sign up → Create org → Onboarding
  2. Connect Intercom (test with sandbox) or add manual feedback
  3. Verify AI categorization, embedding generation, dedup matching
  4. Create feature request from inbox → move through kanban statuses
  5. Verify email notification sent on "shipped"
  6. Check analytics dashboard populates
  7. Upgrade plan via Stripe (test mode)
  8. Verify plan limits enforced
- Set up Vercel project + environment variables
- Configure custom domain
- Set up Sentry for error tracking
- Deploy to production
- Stripe webhook endpoint registered in Stripe dashboard
- Intercom/Zendesk OAuth redirect URIs updated to production URLs

---

## Critical Files

```
feedloop/
├── src/
│   ├── env.ts                                    # Zod-validated environment variables
│   ├── middleware.ts                              # Clerk auth middleware, route protection
│   │
│   ├── app/
│   │   ├── layout.tsx                            # Root layout with ClerkProvider
│   │   ├── page.tsx                              # Landing page
│   │   ├── pricing/page.tsx                      # Pricing page
│   │   ├── onboarding/page.tsx                   # 4-step onboarding wizard
│   │   │
│   │   ├── dashboard/
│   │   │   ├── layout.tsx                        # App shell: sidebar + header
│   │   │   ├── page.tsx                          # Dashboard overview
│   │   │   ├── inbox/
│   │   │   │   ├── page.tsx                      # Feedback inbox (stream + filters)
│   │   │   │   └── new/page.tsx                  # Manual feedback entry form
│   │   │   ├── requests/
│   │   │   │   ├── page.tsx                      # Feature request board (kanban + list)
│   │   │   │   └── [id]/page.tsx                 # Feature request detail (evidence, customers, activity)
│   │   │   ├── analytics/page.tsx                # Analytics dashboard (charts)
│   │   │   └── settings/
│   │   │       ├── page.tsx                      # Organization settings
│   │   │       ├── sources/page.tsx              # Feedback source management
│   │   │       ├── scoring/page.tsx              # Priority scoring weight config
│   │   │       ├── billing/page.tsx              # Plan + usage + upgrade
│   │   │       └── members/page.tsx              # Team member management
│   │   │
│   │   └── api/
│   │       ├── trpc/[trpc]/route.ts              # tRPC HTTP handler
│   │       ├── auth/
│   │       │   ├── intercom/route.ts             # Intercom OAuth start
│   │       │   ├── intercom/callback/route.ts    # Intercom OAuth callback
│   │       │   ├── zendesk/route.ts              # Zendesk OAuth start
│   │       │   └── zendesk/callback/route.ts     # Zendesk OAuth callback
│   │       └── webhooks/
│   │           ├── intercom/route.ts             # Intercom webhook receiver
│   │           ├── zendesk/route.ts              # Zendesk webhook receiver
│   │           └── stripe/route.ts               # Stripe webhook handler
│   │
│   ├── server/
│   │   ├── db/
│   │   │   ├── schema.ts                        # Drizzle schema (all tables + relations)
│   │   │   ├── index.ts                         # Neon database client
│   │   │   └── migrations/
│   │   │       └── 0000_init.sql                # pgvector extension + vector columns + indexes
│   │   │
│   │   ├── trpc/
│   │   │   ├── trpc.ts                          # tRPC init, auth/org/plan middleware
│   │   │   ├── context.ts                       # Request context (auth + db)
│   │   │   └── routers/
│   │   │       ├── _app.ts                      # Merged app router
│   │   │       ├── organization.ts              # Org CRUD + scoring config
│   │   │       ├── feedbackSource.ts            # Source CRUD + connection test
│   │   │       ├── feedback.ts                  # Feedback item CRUD + linking + search
│   │   │       ├── featureRequest.ts            # Request CRUD + board + status transitions
│   │   │       ├── customer.ts                  # Customer list + search
│   │   │       ├── notification.ts              # Notification list + stats
│   │   │       ├── analytics.ts                 # Dashboard data queries
│   │   │       └── billing.ts                   # Plan info + Stripe checkout/portal
│   │   │
│   │   ├── ai/
│   │   │   ├── categorize-feedback.ts           # GPT-4o categorization prompt + parsing
│   │   │   ├── embeddings.ts                    # text-embedding-3-small generation + storage
│   │   │   └── deduplication.ts                 # pgvector similarity search + thresholds
│   │   │
│   │   ├── queue/
│   │   │   ├── connection.ts                    # Redis/ioredis connection
│   │   │   ├── feedback-queue.ts                # BullMQ queue definition
│   │   │   └── feedback-worker.ts               # Worker: categorize → embed → dedup → store
│   │   │
│   │   ├── services/
│   │   │   ├── customer-resolver.ts             # Find-or-create customer by email/external ID
│   │   │   ├── link-feedback.ts                 # Link/unlink feedback ↔ feature request
│   │   │   ├── priority-scoring.ts              # SQL-based score computation
│   │   │   ├── status-machine.ts                # Valid status transitions + labels + colors
│   │   │   └── notifications.ts                 # Resend email dispatch for status changes
│   │   │
│   │   ├── integrations/
│   │   │   └── encryption.ts                    # AES-256-GCM encrypt/decrypt for OAuth tokens
│   │   │
│   │   ├── billing/
│   │   │   └── stripe.ts                        # Plan definitions, checkout, portal
│   │   │
│   │   └── email/
│   │       └── templates.ts                     # HTML email templates for notifications
│   │
│   ├── components/
│   │   ├── ui/                                  # Shared Radix UI primitives
│   │   │   ├── button.tsx
│   │   │   ├── card.tsx
│   │   │   ├── badge.tsx
│   │   │   ├── input.tsx
│   │   │   ├── textarea.tsx
│   │   │   ├── select.tsx
│   │   │   ├── dialog.tsx
│   │   │   ├── dropdown-menu.tsx
│   │   │   ├── tabs.tsx
│   │   │   ├── toast.tsx
│   │   │   ├── skeleton.tsx
│   │   │   ├── tooltip.tsx
│   │   │   └── popover.tsx
│   │   │
│   │   ├── layout/
│   │   │   ├── sidebar.tsx                      # Navigation sidebar
│   │   │   ├── header.tsx                       # Page header with breadcrumbs
│   │   │   └── org-switcher.tsx                 # Clerk org switcher wrapper
│   │   │
│   │   ├── inbox/
│   │   │   ├── feedback-card.tsx                # Reusable feedback item card
│   │   │   ├── feedback-list.tsx                # Infinite scroll list
│   │   │   ├── link-dialog.tsx                  # Link-to-request search dialog
│   │   │   ├── create-request-dialog.tsx        # Create feature request from feedback
│   │   │   ├── customer-autocomplete.tsx        # Customer search typeahead
│   │   │   └── inbox-filters.tsx                # Filter sidebar
│   │   │
│   │   ├── board/
│   │   │   ├── kanban-board.tsx                 # Drag-and-drop kanban
│   │   │   ├── request-card.tsx                 # Kanban card
│   │   │   ├── list-view.tsx                    # Table/list view
│   │   │   └── board-toolbar.tsx                # View toggle + search + filters
│   │   │
│   │   ├── request-detail/
│   │   │   ├── evidence-tab.tsx                 # Linked feedback items
│   │   │   ├── customers-tab.tsx                # Requesting customers list
│   │   │   ├── activity-tab.tsx                 # Status change timeline
│   │   │   └── status-selector.tsx              # Status transition dropdown
│   │   │
│   │   └── analytics/
│   │       ├── volume-chart.tsx                 # Recharts area chart
│   │       ├── top-requests-chart.tsx           # Horizontal bar chart
│   │       ├── sentiment-chart.tsx              # Line chart
│   │       ├── source-breakdown.tsx             # Pie chart
│   │       └── stats-cards.tsx                  # Overview metric cards
│   │
│   └── lib/
│       ├── trpc/
│       │   ├── client.ts                        # React Query tRPC client
│       │   └── server.ts                        # Server-side tRPC caller
│       ├── utils.ts                             # cn() helper, date formatting
│       └── constants.ts                         # Category labels, sentiment labels, source icons
│
├── drizzle.config.ts
├── tailwind.config.ts
├── tsconfig.json
├── next.config.ts
├── package.json
└── .env.local
```

---

## Verification

### Manual Testing Checklist

1. **Auth flow**: Sign up with Clerk → create organization → verify org record created in DB
2. **Intercom OAuth**: Click "Connect Intercom" → complete OAuth → verify encrypted token stored in `feedback_sources`
3. **Zendesk OAuth**: Enter subdomain → complete OAuth → verify source created
4. **Intercom webhook**: Send test webhook payload → verify feedback item created with AI categorization
5. **Zendesk webhook**: Send test webhook payload → verify feedback item created
6. **Manual entry**: Submit feedback via form → verify AI processing enqueued and completed
7. **AI categorization**: Verify category, extracted_ask, sentiment, tags populated correctly
8. **Embedding storage**: Verify `embedding` column populated (non-null) on feedback items
9. **Deduplication**: Submit similar feedback → verify "AI Suggested Match" appears in inbox
10. **Auto-link**: Submit very similar feedback (>0.88 similarity) → verify auto-linked to existing request
11. **Feature request creation**: Create from inbox → verify feedback linked, request_count = 1
12. **Kanban drag-and-drop**: Move card between columns → verify status updated + activity logged
13. **Status machine**: Attempt invalid transition (e.g., new → shipped) → verify error returned
14. **Priority scoring**: Recompute → verify scores reflect frequency, recency, sentiment, ARR weights
15. **Email notifications**: Change status to "shipped" → verify emails sent to linked customers
16. **Analytics**: Verify charts populate with real data after 10+ feedback items
17. **Plan limits**: On Starter plan, attempt to add 4th source → verify error
18. **Billing upgrade**: Complete Stripe checkout → verify plan updated in DB
19. **Stripe webhook**: Simulate subscription deletion → verify plan reverts to starter

### Key SQL Queries for Verification

```sql
-- Check embedding was stored
SELECT id, raw_text, extracted_ask,
       embedding IS NOT NULL AS has_embedding,
       1 - (embedding <=> (SELECT embedding FROM feedback_items WHERE id = 'SOME_ID')) AS similarity
FROM feedback_items
WHERE org_id = 'ORG_ID'
ORDER BY embedding <=> (SELECT embedding FROM feedback_items WHERE id = 'SOME_ID')
LIMIT 5;

-- Check priority scores
SELECT title, status, request_count, total_arr_requesting, priority_score
FROM feature_requests
WHERE org_id = 'ORG_ID'
ORDER BY priority_score DESC;

-- Check notification delivery
SELECT n.recipient_email, n.status, n.sent_at, n.metadata->>'featureTitle' AS feature
FROM notifications n
WHERE n.org_id = 'ORG_ID'
ORDER BY n.created_at DESC;

-- Monthly item count (for plan enforcement)
SELECT COUNT(*) FROM feedback_items
WHERE org_id = 'ORG_ID'
  AND created_at >= date_trunc('month', now());
```

### Performance Benchmarks

| Operation | Target | Method |
|-----------|--------|--------|
| Webhook receive → 200 response | < 500ms | Enqueue only, process async |
| AI categorization (single item) | < 3s | GPT-4o with 300 max_tokens |
| Embedding generation | < 1s | text-embedding-3-small |
| Dedup similarity search (1000 items) | < 100ms | HNSW index on pgvector |
| Inbox page load (50 items) | < 500ms | Indexed queries + React Query cache |
| Kanban board load (100 requests) | < 400ms | Single grouped query |
| Priority score recomputation (500 requests) | < 2s | Single SQL CTE query |
| Email notification dispatch | < 2s per email | Resend API |
