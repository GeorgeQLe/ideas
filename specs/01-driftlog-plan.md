# 1. DriftLog — Automated Changelog Generator

## Implementation Plan

**MVP Scope:** GitHub App integration with OAuth 2.0 and webhook-based PR/release event processing, AI changelog generation from pull request titles + descriptions + code diffs using GPT-4o, hosted changelog page with basic branding (logo, colors) at `{slug}.driftlog.io`, editorial workflow with rich text editor for editing AI-generated entries, email notifications to subscribers via Resend, manual release creation, category auto-classification (feature/improvement/bugfix/breaking/performance/security), Stripe billing with three tiers (Free / Pro $19/mo / Team $49/mo).

---

## Tech Decisions

| Layer | Technology | Notes |
|-------|-----------|-------|
| Framework | Next.js 14+ App Router | `src/` directory, TypeScript strict |
| API | tRPC v11 | superjson transformer, Clerk auth context |
| Database | Neon PostgreSQL | Releases, entries, subscribers, GitHub metadata |
| ORM | Drizzle ORM | Push-based migrations, typed schema |
| Auth | Clerk | Organization-scoped, `orgId` from session |
| Billing | Stripe | Checkout Sessions + Customer Portal + Webhooks |
| AI — Generation | OpenAI GPT-4o | Temperature 0.7 for creative yet accurate summaries |
| AI — Classification | OpenAI GPT-4o-mini | JSON mode, temperature 0.1 for categorization |
| Queue | BullMQ + Redis (Upstash) | Async webhook processing + AI generation |
| GitHub | GitHub App (OAuth 2.0 + webhooks) | Installation-based access, PR + Release events |
| Email | Resend | Subscriber notifications, digest emails |
| Storage | Cloudflare R2 | Logos, OG images, custom assets |
| Changelog Hosting | Vercel (ISR) | Incremental Static Regeneration for public pages |
| Rich Text | Tiptap (ProseMirror) | Changelog entry editor with markdown output |
| UI | Tailwind CSS, Radix UI, Recharts | |
| Hosting | Vercel | |
| Monitoring | Sentry, Vercel Analytics | |

### Key Architecture Decisions

1. **GitHub App over personal access tokens**: GitHub Apps provide installation-based access with granular permissions, webhook delivery with verified signatures, and rate limits per installation. This is more secure and scalable than PATs. Each org installs the DriftLog GitHub App, granting read access to PRs, commits, and releases.

2. **AI generation from PR metadata + selective diffs**: Rather than feeding entire diffs to GPT-4o (expensive, noisy), we concatenate PR title + description + file change summary (files changed, insertions, deletions) + first 200 lines of diff for the most significant files. This provides enough context for accurate summaries while keeping token costs under $0.05 per release.

3. **ISR for public changelog pages**: Changelog pages are statically generated at build time and revalidated on-demand when a release is published. This gives near-instant page loads while keeping content fresh. The public changelog route uses Next.js ISR with `revalidateTag('changelog-{orgSlug}')`.

4. **BullMQ for webhook + AI processing**: GitHub webhooks return 200 immediately while a BullMQ worker processes the event asynchronously. This handles webhook timeout requirements (10 seconds) and manages OpenAI rate limits with job-level concurrency control.

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
  plan: text("plan").default("free").notNull(), // free | pro | team
  stripeCustomerId: text("stripe_customer_id"),
  stripeSubscriptionId: text("stripe_subscription_id"),
  branding: jsonb("branding").default({}).$type<{
    logoUrl?: string;
    primaryColor?: string;
    accentColor?: string;
    fontFamily?: string;
    faviconUrl?: string;
    customDomain?: string;
  }>(),
  settings: jsonb("settings").default({}).$type<{
    defaultTone?: "professional" | "casual" | "technical" | "marketing";
    defaultAudience?: "end-users" | "developers" | "internal";
    autoPublish?: boolean;
    notificationFromName?: string;
    notificationFromEmail?: string;
  }>(),
  createdAt: timestamp("created_at", { withTimezone: true }).defaultNow().notNull(),
  updatedAt: timestamp("updated_at", { withTimezone: true }).defaultNow().notNull(),
});

// ---------------------------------------------------------------------------
// Repositories (connected GitHub repos)
// ---------------------------------------------------------------------------
export const repositories = pgTable(
  "repositories",
  {
    id: uuid("id").primaryKey().defaultRandom(),
    orgId: uuid("org_id")
      .references(() => organizations.id, { onDelete: "cascade" })
      .notNull(),
    provider: text("provider").default("github").notNull(), // github | gitlab (v2)
    providerRepoId: text("provider_repo_id").notNull(), // GitHub repo ID (numeric string)
    name: text("name").notNull(), // e.g. "owner/repo"
    url: text("url").notNull(), // https://github.com/owner/repo
    defaultBranch: text("default_branch").default("main").notNull(),
    installationId: text("installation_id").notNull(), // GitHub App installation ID
    config: jsonb("config").default({}).$type<{
      tone?: "professional" | "casual" | "technical" | "marketing";
      audience?: "end-users" | "developers" | "internal";
      ignoredPaths?: string[]; // glob patterns: ["*.lock", "ci/**"]
      triggerBranches?: string[]; // branches that trigger generation
      triggerOnRelease?: boolean;
      triggerOnTag?: boolean;
      autoPublish?: boolean;
      includeDependencyBumps?: boolean;
    }>(),
    isActive: boolean("is_active").default(true).notNull(),
    lastWebhookAt: timestamp("last_webhook_at", { withTimezone: true }),
    createdAt: timestamp("created_at", { withTimezone: true }).defaultNow().notNull(),
  },
  (t) => [
    uniqueIndex("repo_provider_idx").on(t.provider, t.providerRepoId),
    index("repo_org_idx").on(t.orgId),
  ],
);

// ---------------------------------------------------------------------------
// Releases
// ---------------------------------------------------------------------------
export const releases = pgTable(
  "releases",
  {
    id: uuid("id").primaryKey().defaultRandom(),
    orgId: uuid("org_id")
      .references(() => organizations.id, { onDelete: "cascade" })
      .notNull(),
    repoId: uuid("repo_id")
      .references(() => repositories.id, { onDelete: "cascade" }),
    version: text("version"), // e.g. "v2.1.0"
    title: text("title").notNull(),
    status: text("status").default("draft").notNull(),
    // draft | review | published | scheduled
    source: text("source").default("auto").notNull(), // auto | manual
    githubReleaseId: text("github_release_id"), // GitHub release ID
    githubReleaseUrl: text("github_release_url"),
    tagName: text("tag_name"),
    publishedAt: timestamp("published_at", { withTimezone: true }),
    scheduledAt: timestamp("scheduled_at", { withTimezone: true }),
    createdAt: timestamp("created_at", { withTimezone: true }).defaultNow().notNull(),
    updatedAt: timestamp("updated_at", { withTimezone: true }).defaultNow().notNull(),
  },
  (t) => [
    index("release_org_idx").on(t.orgId),
    index("release_status_idx").on(t.status),
    index("release_published_idx").on(t.publishedAt),
  ],
);

// ---------------------------------------------------------------------------
// Changelog Entries (individual items within a release)
// ---------------------------------------------------------------------------
export const changelogEntries = pgTable(
  "changelog_entries",
  {
    id: uuid("id").primaryKey().defaultRandom(),
    releaseId: uuid("release_id")
      .references(() => releases.id, { onDelete: "cascade" })
      .notNull(),
    orgId: uuid("org_id")
      .references(() => organizations.id, { onDelete: "cascade" })
      .notNull(),
    category: text("category").default("improvement").notNull(),
    // feature | improvement | bugfix | breaking | performance | security | deprecation
    title: text("title").notNull(),
    body: text("body").notNull(), // rich text / markdown
    aiGeneratedBody: text("ai_generated_body"), // original AI output
    humanEdited: boolean("human_edited").default(false).notNull(),
    sourcePrNumber: integer("source_pr_number"),
    sourcePrUrl: text("source_pr_url"),
    sourcePrTitle: text("source_pr_title"),
    sourceCommitShas: jsonb("source_commit_shas").default([]).$type<string[]>(),
    sortOrder: integer("sort_order").default(0).notNull(),
    createdAt: timestamp("created_at", { withTimezone: true }).defaultNow().notNull(),
  },
  (t) => [
    index("entry_release_idx").on(t.releaseId),
    index("entry_org_idx").on(t.orgId),
  ],
);

// ---------------------------------------------------------------------------
// Webhook Events (raw GitHub webhook log)
// ---------------------------------------------------------------------------
export const webhookEvents = pgTable(
  "webhook_events",
  {
    id: uuid("id").primaryKey().defaultRandom(),
    repoId: uuid("repo_id")
      .references(() => repositories.id, { onDelete: "cascade" }),
    eventType: text("event_type").notNull(), // pull_request | release | push
    action: text("action"), // opened | closed | merged | published
    payload: jsonb("payload").notNull(),
    processedAt: timestamp("processed_at", { withTimezone: true }),
    status: text("status").default("pending").notNull(), // pending | processing | completed | failed
    errorMessage: text("error_message"),
    receivedAt: timestamp("received_at", { withTimezone: true }).defaultNow().notNull(),
  },
  (t) => [
    index("webhook_repo_idx").on(t.repoId),
    index("webhook_status_idx").on(t.status),
  ],
);

// ---------------------------------------------------------------------------
// Subscribers (email notification list)
// ---------------------------------------------------------------------------
export const subscribers = pgTable(
  "subscribers",
  {
    id: uuid("id").primaryKey().defaultRandom(),
    orgId: uuid("org_id")
      .references(() => organizations.id, { onDelete: "cascade" })
      .notNull(),
    email: text("email").notNull(),
    name: text("name"),
    preferences: jsonb("preferences").default({}).$type<{
      categories?: string[]; // only notify for these categories
      frequency?: "instant" | "weekly";
    }>(),
    lastSeenReleaseId: uuid("last_seen_release_id"),
    isActive: boolean("is_active").default(true).notNull(),
    unsubscribeToken: text("unsubscribe_token").notNull(),
    subscribedAt: timestamp("subscribed_at", { withTimezone: true }).defaultNow().notNull(),
  },
  (t) => [
    uniqueIndex("subscriber_org_email_idx").on(t.orgId, t.email),
    index("subscriber_org_idx").on(t.orgId),
  ],
);

// ---------------------------------------------------------------------------
// Notification Sends (track what we sent)
// ---------------------------------------------------------------------------
export const notificationSends = pgTable(
  "notification_sends",
  {
    id: uuid("id").primaryKey().defaultRandom(),
    subscriberId: uuid("subscriber_id")
      .references(() => subscribers.id, { onDelete: "cascade" })
      .notNull(),
    releaseId: uuid("release_id")
      .references(() => releases.id, { onDelete: "cascade" })
      .notNull(),
    sentAt: timestamp("sent_at", { withTimezone: true }).defaultNow().notNull(),
    openedAt: timestamp("opened_at", { withTimezone: true }),
    clickedAt: timestamp("clicked_at", { withTimezone: true }),
  },
  (t) => [
    index("notif_send_subscriber_idx").on(t.subscriberId),
    index("notif_send_release_idx").on(t.releaseId),
  ],
);

// ---------------------------------------------------------------------------
// Page Views (changelog page analytics)
// ---------------------------------------------------------------------------
export const pageViews = pgTable(
  "page_views",
  {
    id: uuid("id").primaryKey().defaultRandom(),
    orgId: uuid("org_id")
      .references(() => organizations.id, { onDelete: "cascade" })
      .notNull(),
    releaseId: uuid("release_id"),
    entryId: uuid("entry_id"),
    path: text("path"),
    referrer: text("referrer"),
    country: text("country"),
    device: text("device"), // desktop | mobile | tablet
    viewedAt: timestamp("viewed_at", { withTimezone: true }).defaultNow().notNull(),
  },
  (t) => [
    index("pv_org_idx").on(t.orgId),
    index("pv_viewed_idx").on(t.viewedAt),
  ],
);

// ---------------------------------------------------------------------------
// Widget Configs
// ---------------------------------------------------------------------------
export const widgetConfigs = pgTable("widget_configs", {
  id: uuid("id").primaryKey().defaultRandom(),
  orgId: uuid("org_id")
    .references(() => organizations.id, { onDelete: "cascade" })
    .unique()
    .notNull(),
  position: text("position").default("bottom-right").notNull(),
  theme: jsonb("theme").default({}).$type<{
    primaryColor?: string;
    backgroundColor?: string;
    textColor?: string;
    borderRadius?: number;
  }>(),
  triggerSelector: text("trigger_selector"), // CSS selector for custom trigger
  showBadge: boolean("show_badge").default(true).notNull(),
  maxEntries: integer("max_entries").default(10).notNull(),
  showPoweredBy: boolean("show_powered_by").default(true).notNull(),
  createdAt: timestamp("created_at", { withTimezone: true }).defaultNow().notNull(),
});

// ---------------------------------------------------------------------------
// Relations
// ---------------------------------------------------------------------------
export const organizationsRelations = relations(organizations, ({ many, one }) => ({
  repositories: many(repositories),
  releases: many(releases),
  changelogEntries: many(changelogEntries),
  subscribers: many(subscribers),
  pageViews: many(pageViews),
  widgetConfig: one(widgetConfigs),
}));

export const repositoriesRelations = relations(repositories, ({ one, many }) => ({
  organization: one(organizations, {
    fields: [repositories.orgId],
    references: [organizations.id],
  }),
  releases: many(releases),
  webhookEvents: many(webhookEvents),
}));

export const releasesRelations = relations(releases, ({ one, many }) => ({
  organization: one(organizations, {
    fields: [releases.orgId],
    references: [organizations.id],
  }),
  repository: one(repositories, {
    fields: [releases.repoId],
    references: [repositories.id],
  }),
  entries: many(changelogEntries),
  notificationSends: many(notificationSends),
}));

export const changelogEntriesRelations = relations(changelogEntries, ({ one }) => ({
  release: one(releases, {
    fields: [changelogEntries.releaseId],
    references: [releases.id],
  }),
  organization: one(organizations, {
    fields: [changelogEntries.orgId],
    references: [organizations.id],
  }),
}));

export const webhookEventsRelations = relations(webhookEvents, ({ one }) => ({
  repository: one(repositories, {
    fields: [webhookEvents.repoId],
    references: [repositories.id],
  }),
}));

export const subscribersRelations = relations(subscribers, ({ one, many }) => ({
  organization: one(organizations, {
    fields: [subscribers.orgId],
    references: [organizations.id],
  }),
  notificationSends: many(notificationSends),
}));

export const notificationSendsRelations = relations(notificationSends, ({ one }) => ({
  subscriber: one(subscribers, {
    fields: [notificationSends.subscriberId],
    references: [subscribers.id],
  }),
  release: one(releases, {
    fields: [notificationSends.releaseId],
    references: [releases.id],
  }),
}));

export const pageViewsRelations = relations(pageViews, ({ one }) => ({
  organization: one(organizations, {
    fields: [pageViews.orgId],
    references: [organizations.id],
  }),
}));

export const widgetConfigsRelations = relations(widgetConfigs, ({ one }) => ({
  organization: one(organizations, {
    fields: [widgetConfigs.orgId],
    references: [organizations.id],
  }),
}));
```

---

## Initial Migration SQL

```sql
-- Enable required extensions
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";

-- Organizations
CREATE TABLE organizations (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  clerk_org_id TEXT UNIQUE NOT NULL,
  name TEXT NOT NULL,
  slug TEXT UNIQUE NOT NULL,
  plan TEXT NOT NULL DEFAULT 'free',
  stripe_customer_id TEXT,
  stripe_subscription_id TEXT,
  branding JSONB DEFAULT '{}',
  settings JSONB DEFAULT '{}',
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Repositories
CREATE TABLE repositories (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  org_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
  provider TEXT NOT NULL DEFAULT 'github',
  provider_repo_id TEXT NOT NULL,
  name TEXT NOT NULL,
  url TEXT NOT NULL,
  default_branch TEXT NOT NULL DEFAULT 'main',
  installation_id TEXT NOT NULL,
  config JSONB DEFAULT '{}',
  is_active BOOLEAN NOT NULL DEFAULT true,
  last_webhook_at TIMESTAMPTZ,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE UNIQUE INDEX repo_provider_idx ON repositories(provider, provider_repo_id);
CREATE INDEX repo_org_idx ON repositories(org_id);

-- Releases
CREATE TABLE releases (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  org_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
  repo_id UUID REFERENCES repositories(id) ON DELETE CASCADE,
  version TEXT,
  title TEXT NOT NULL,
  status TEXT NOT NULL DEFAULT 'draft',
  source TEXT NOT NULL DEFAULT 'auto',
  github_release_id TEXT,
  github_release_url TEXT,
  tag_name TEXT,
  published_at TIMESTAMPTZ,
  scheduled_at TIMESTAMPTZ,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX release_org_idx ON releases(org_id);
CREATE INDEX release_status_idx ON releases(status);
CREATE INDEX release_published_idx ON releases(published_at);

-- Changelog Entries
CREATE TABLE changelog_entries (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  release_id UUID NOT NULL REFERENCES releases(id) ON DELETE CASCADE,
  org_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
  category TEXT NOT NULL DEFAULT 'improvement',
  title TEXT NOT NULL,
  body TEXT NOT NULL,
  ai_generated_body TEXT,
  human_edited BOOLEAN NOT NULL DEFAULT false,
  source_pr_number INTEGER,
  source_pr_url TEXT,
  source_pr_title TEXT,
  source_commit_shas JSONB DEFAULT '[]',
  sort_order INTEGER NOT NULL DEFAULT 0,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX entry_release_idx ON changelog_entries(release_id);
CREATE INDEX entry_org_idx ON changelog_entries(org_id);

-- Webhook Events
CREATE TABLE webhook_events (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  repo_id UUID REFERENCES repositories(id) ON DELETE CASCADE,
  event_type TEXT NOT NULL,
  action TEXT,
  payload JSONB NOT NULL,
  processed_at TIMESTAMPTZ,
  status TEXT NOT NULL DEFAULT 'pending',
  error_message TEXT,
  received_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX webhook_repo_idx ON webhook_events(repo_id);
CREATE INDEX webhook_status_idx ON webhook_events(status);

-- Subscribers
CREATE TABLE subscribers (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  org_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
  email TEXT NOT NULL,
  name TEXT,
  preferences JSONB DEFAULT '{}',
  last_seen_release_id UUID,
  is_active BOOLEAN NOT NULL DEFAULT true,
  unsubscribe_token TEXT NOT NULL,
  subscribed_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE UNIQUE INDEX subscriber_org_email_idx ON subscribers(org_id, email);
CREATE INDEX subscriber_org_idx ON subscribers(org_id);

-- Notification Sends
CREATE TABLE notification_sends (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  subscriber_id UUID NOT NULL REFERENCES subscribers(id) ON DELETE CASCADE,
  release_id UUID NOT NULL REFERENCES releases(id) ON DELETE CASCADE,
  sent_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  opened_at TIMESTAMPTZ,
  clicked_at TIMESTAMPTZ
);
CREATE INDEX notif_send_subscriber_idx ON notification_sends(subscriber_id);
CREATE INDEX notif_send_release_idx ON notification_sends(release_id);

-- Page Views
CREATE TABLE page_views (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  org_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
  release_id UUID,
  entry_id UUID,
  path TEXT,
  referrer TEXT,
  country TEXT,
  device TEXT,
  viewed_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX pv_org_idx ON page_views(org_id);
CREATE INDEX pv_viewed_idx ON page_views(viewed_at);

-- Widget Configs
CREATE TABLE widget_configs (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  org_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE UNIQUE,
  position TEXT NOT NULL DEFAULT 'bottom-right',
  theme JSONB DEFAULT '{}',
  trigger_selector TEXT,
  show_badge BOOLEAN NOT NULL DEFAULT true,
  max_entries INTEGER NOT NULL DEFAULT 10,
  show_powered_by BOOLEAN NOT NULL DEFAULT true,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

---

## Architecture Deep-Dives

### 1. GitHub App Webhook Processing Pipeline

When GitHub sends a webhook (PR merged, release published), we validate the signature, store the raw event, and enqueue it for async processing. The worker fetches PR details and diffs from the GitHub API, then generates a changelog entry via AI.

```typescript
// src/server/workers/webhook-processor.ts

import { Worker, Queue } from "bullmq";
import { Octokit } from "@octokit/rest";
import { createAppAuth } from "@octokit/auth-app";
import { eq, and } from "drizzle-orm";
import { db } from "../db";
import {
  repositories,
  releases,
  changelogEntries,
  webhookEvents,
} from "../db/schema";
import { generateChangelogEntry } from "../services/ai-generation";

interface WebhookJob {
  webhookEventId: string;
}

export const webhookQueue = new Queue<WebhookJob>("webhook-processing", {
  connection: { url: process.env.UPSTASH_REDIS_URL },
});

const worker = new Worker<WebhookJob>(
  "webhook-processing",
  async (job) => {
    const event = await db.query.webhookEvents.findFirst({
      where: eq(webhookEvents.id, job.data.webhookEventId),
    });
    if (!event) return;

    await db
      .update(webhookEvents)
      .set({ status: "processing" })
      .where(eq(webhookEvents.id, event.id));

    try {
      if (event.eventType === "pull_request" && event.action === "closed") {
        await processMergedPR(event);
      } else if (event.eventType === "release" && event.action === "published") {
        await processRelease(event);
      }

      await db
        .update(webhookEvents)
        .set({ status: "completed", processedAt: new Date() })
        .where(eq(webhookEvents.id, event.id));
    } catch (err) {
      await db
        .update(webhookEvents)
        .set({
          status: "failed",
          errorMessage: err instanceof Error ? err.message : String(err),
        })
        .where(eq(webhookEvents.id, event.id));
      throw err; // Let BullMQ handle retry
    }
  },
  {
    connection: { url: process.env.UPSTASH_REDIS_URL },
    concurrency: 3,
    limiter: { max: 10, duration: 60_000 },
  },
);

async function processMergedPR(event: typeof webhookEvents.$inferSelect) {
  const payload = event.payload as any;
  const pr = payload.pull_request;

  // Only process merged PRs (not just closed)
  if (!pr.merged) return;

  const repo = await db.query.repositories.findFirst({
    where: eq(repositories.providerRepoId, String(payload.repository.id)),
    with: { organization: true },
  });
  if (!repo || !repo.isActive) return;

  const config = repo.config || {};

  // Check if this should be ignored (CI changes, dependency bumps, etc.)
  if (shouldIgnorePR(pr, config)) return;

  // Get Octokit for this installation
  const octokit = await getInstallationOctokit(repo.installationId);

  // Fetch PR diff (limited to avoid huge diffs)
  const [owner, repoName] = repo.name.split("/");
  const { data: diffData } = await octokit.pulls.get({
    owner,
    repo: repoName,
    pull_number: pr.number,
    mediaType: { format: "diff" },
  });

  // Fetch changed files summary
  const { data: files } = await octokit.pulls.listFiles({
    owner,
    repo: repoName,
    pull_number: pr.number,
    per_page: 30,
  });

  const fileSummary = files
    .filter((f) => !isIgnoredPath(f.filename, config.ignoredPaths || []))
    .map((f) => `${f.status}: ${f.filename} (+${f.additions}/-${f.deletions})`)
    .join("\n");

  // Truncate diff for AI processing (keep first 4000 chars)
  const truncatedDiff = typeof diffData === "string"
    ? diffData.slice(0, 4000)
    : "";

  // Find or create a draft release for this PR
  let release = await db.query.releases.findFirst({
    where: and(
      eq(releases.orgId, repo.orgId),
      eq(releases.repoId, repo.id),
      eq(releases.status, "draft"),
    ),
  });

  if (!release) {
    const [newRelease] = await db
      .insert(releases)
      .values({
        orgId: repo.orgId,
        repoId: repo.id,
        title: `Release from ${new Date().toLocaleDateString()}`,
        status: "draft",
        source: "auto",
      })
      .returning();
    release = newRelease;
  }

  // Generate changelog entry via AI
  const generated = await generateChangelogEntry({
    prTitle: pr.title,
    prDescription: pr.body || "",
    fileSummary,
    diff: truncatedDiff,
    tone: config.tone || repo.organization.settings?.defaultTone || "professional",
    audience: config.audience || repo.organization.settings?.defaultAudience || "end-users",
  });

  // Save entry
  const maxOrder = await db
    .select({ max: sql`COALESCE(MAX(sort_order), 0)` })
    .from(changelogEntries)
    .where(eq(changelogEntries.releaseId, release.id));

  await db.insert(changelogEntries).values({
    releaseId: release.id,
    orgId: repo.orgId,
    category: generated.category,
    title: generated.title,
    body: generated.body,
    aiGeneratedBody: generated.body,
    humanEdited: false,
    sourcePrNumber: pr.number,
    sourcePrUrl: pr.html_url,
    sourcePrTitle: pr.title,
    sourceCommitShas: [pr.merge_commit_sha].filter(Boolean),
    sortOrder: (maxOrder[0]?.max as number) + 1,
  });

  // Auto-publish if configured
  if (config.autoPublish) {
    await db
      .update(releases)
      .set({ status: "published", publishedAt: new Date() })
      .where(eq(releases.id, release.id));
  }
}

function shouldIgnorePR(pr: any, config: any): boolean {
  const title = pr.title.toLowerCase();
  const ignoredPatterns = [
    /^bump /i,
    /^chore\(deps\)/i,
    /^ci:/i,
    /^docs:/i,
    /dependabot/i,
    /renovate/i,
  ];
  if (!config.includeDependencyBumps) {
    return ignoredPatterns.some((p) => p.test(title));
  }
  return false;
}

function isIgnoredPath(filename: string, patterns: string[]): boolean {
  return patterns.some((pattern) => {
    const regex = new RegExp(
      pattern.replace(/\*/g, ".*").replace(/\?/g, "."),
    );
    return regex.test(filename);
  });
}

async function getInstallationOctokit(installationId: string): Promise<Octokit> {
  const auth = createAppAuth({
    appId: process.env.GITHUB_APP_ID!,
    privateKey: process.env.GITHUB_PRIVATE_KEY!,
    installationId: Number(installationId),
  });
  const { token } = await auth({ type: "installation" });
  return new Octokit({ auth: token });
}
```

### 2. AI Changelog Generation Engine

The generation engine takes PR metadata and diff context, then produces a user-facing changelog entry with category classification. Two-step process: classify the change type first, then generate the summary in the configured tone/audience.

```typescript
// src/server/services/ai-generation.ts

import { OpenAI } from "openai";

const openai = new OpenAI();

interface GenerationInput {
  prTitle: string;
  prDescription: string;
  fileSummary: string;
  diff: string;
  tone: "professional" | "casual" | "technical" | "marketing";
  audience: "end-users" | "developers" | "internal";
}

interface GeneratedEntry {
  category: string;
  title: string;
  body: string;
}

const TONE_INSTRUCTIONS: Record<string, string> = {
  professional: "Write in a clear, professional tone. Be concise and factual.",
  casual: "Write in a friendly, approachable tone. Use simple language.",
  technical: "Include technical details relevant to developers. Reference APIs and components.",
  marketing: "Emphasize the benefit to users. Use excitement-building language.",
};

const AUDIENCE_INSTRUCTIONS: Record<string, string> = {
  "end-users": "Write for non-technical users. Focus on what changed for them, not how it was implemented. Avoid jargon.",
  developers: "Write for developers. You can reference APIs, libraries, and technical concepts.",
  internal: "Write for the internal team. Include implementation context and architectural decisions.",
};

export async function generateChangelogEntry(
  input: GenerationInput,
): Promise<GeneratedEntry> {
  // Step 1: Classify the change type
  const classifyResult = await openai.chat.completions.create({
    model: "gpt-4o-mini",
    temperature: 0.1,
    response_format: { type: "json_object" },
    messages: [
      {
        role: "system",
        content: `Classify this code change into exactly one category. Return JSON:
{ "category": "feature" | "improvement" | "bugfix" | "breaking" | "performance" | "security" | "deprecation" }`,
      },
      {
        role: "user",
        content: `PR Title: ${input.prTitle}
PR Description: ${input.prDescription}
Files Changed:\n${input.fileSummary}`,
      },
    ],
  });

  const classification = JSON.parse(
    classifyResult.choices[0].message.content || '{"category":"improvement"}',
  );

  // Step 2: Generate user-facing summary
  const generateResult = await openai.chat.completions.create({
    model: "gpt-4o",
    temperature: 0.7,
    messages: [
      {
        role: "system",
        content: `You are writing a changelog entry for a software product.

${TONE_INSTRUCTIONS[input.tone]}
${AUDIENCE_INSTRUCTIONS[input.audience]}

Return JSON with:
{
  "title": "A concise one-line title (max 80 chars)",
  "body": "A 1-3 sentence description of the change and its impact. Use markdown formatting."
}

Important:
- Focus on WHAT changed and WHY it matters, not HOW it was implemented
- Don't reference PR numbers, commit hashes, or file names
- Don't start with "We" or "This PR"
- Make it scannable and valuable for the reader`,
      },
      {
        role: "user",
        content: `Change type: ${classification.category}
PR Title: ${input.prTitle}
PR Description: ${input.prDescription}
Files Changed:\n${input.fileSummary}
Diff excerpt:\n${input.diff.slice(0, 2000)}`,
      },
    ],
    response_format: { type: "json_object" },
  });

  const generated = JSON.parse(
    generateResult.choices[0].message.content || "{}",
  );

  return {
    category: classification.category,
    title: generated.title || input.prTitle,
    body: generated.body || input.prDescription,
  };
}

// Regenerate with different parameters (user action)
export async function regenerateEntry(
  input: GenerationInput & { feedback?: string },
): Promise<GeneratedEntry> {
  const result = await openai.chat.completions.create({
    model: "gpt-4o",
    temperature: 0.8, // slightly higher for variety
    response_format: { type: "json_object" },
    messages: [
      {
        role: "system",
        content: `You are rewriting a changelog entry. Generate a fresh version.

${TONE_INSTRUCTIONS[input.tone]}
${AUDIENCE_INSTRUCTIONS[input.audience]}

${input.feedback ? `User feedback on previous version: ${input.feedback}` : ""}

Return JSON: { "category": "...", "title": "...", "body": "..." }`,
      },
      {
        role: "user",
        content: `PR Title: ${input.prTitle}
PR Description: ${input.prDescription}
Files Changed:\n${input.fileSummary}
Diff excerpt:\n${input.diff.slice(0, 2000)}`,
      },
    ],
  });

  return JSON.parse(result.choices[0].message.content || "{}");
}
```

### 3. Public Changelog Page with ISR

The public changelog page uses Next.js Incremental Static Regeneration to serve static HTML that's revalidated when a release is published. This provides instant page loads with fresh content.

```typescript
// src/app/(public)/changelog/[orgSlug]/page.tsx

import { notFound } from "next/navigation";
import { eq, and, desc } from "drizzle-orm";
import { db } from "@/server/db";
import { organizations, releases, changelogEntries } from "@/server/db/schema";
import { ChangelogPageClient } from "./ChangelogPageClient";

interface Props {
  params: { orgSlug: string };
  searchParams: { category?: string; page?: string };
}

export async function generateMetadata({ params }: Props) {
  const org = await db.query.organizations.findFirst({
    where: eq(organizations.slug, params.orgSlug),
  });
  if (!org) return {};

  return {
    title: `${org.name} Changelog — What's New`,
    description: `See the latest updates, features, and fixes from ${org.name}`,
    openGraph: {
      title: `${org.name} Changelog`,
      description: `See the latest updates from ${org.name}`,
      images: org.branding?.logoUrl ? [{ url: org.branding.logoUrl }] : [],
    },
  };
}

export default async function ChangelogPage({ params, searchParams }: Props) {
  const org = await db.query.organizations.findFirst({
    where: eq(organizations.slug, params.orgSlug),
  });
  if (!org) notFound();

  const page = parseInt(searchParams.page || "1", 10);
  const perPage = 10;

  const publishedReleases = await db.query.releases.findMany({
    where: and(
      eq(releases.orgId, org.id),
      eq(releases.status, "published"),
    ),
    with: {
      entries: {
        orderBy: [desc(changelogEntries.sortOrder)],
      },
      repository: true,
    },
    orderBy: [desc(releases.publishedAt)],
    limit: perPage,
    offset: (page - 1) * perPage,
  });

  // Apply category filter if present
  const filteredReleases = searchParams.category
    ? publishedReleases.map((r) => ({
        ...r,
        entries: r.entries.filter((e) => e.category === searchParams.category),
      })).filter((r) => r.entries.length > 0)
    : publishedReleases;

  return (
    <ChangelogPageClient
      org={org}
      releases={filteredReleases}
      page={page}
      activeCategory={searchParams.category}
    />
  );
}

// Revalidate every 60 seconds, or on-demand via revalidateTag
export const revalidate = 60;

// src/app/api/revalidate/route.ts — called on release publish
import { revalidateTag } from "next/cache";
import { NextRequest, NextResponse } from "next/server";

export async function POST(req: NextRequest) {
  const { orgSlug, secret } = await req.json();
  if (secret !== process.env.REVALIDATION_SECRET) {
    return NextResponse.json({ error: "Unauthorized" }, { status: 401 });
  }

  revalidateTag(`changelog-${orgSlug}`);
  return NextResponse.json({ revalidated: true });
}
```

### 4. Subscriber Email Notification System

When a release is published, notifications are sent to all active subscribers via Resend. Uses React Email templates and tracks opens/clicks.

```typescript
// src/server/services/notifications.ts

import { Resend } from "resend";
import { eq, and } from "drizzle-orm";
import { nanoid } from "nanoid";
import { db } from "../db";
import {
  subscribers,
  notificationSends,
  releases,
  changelogEntries,
  organizations,
} from "../db/schema";
import { ReleaseNotificationEmail } from "./email-templates/release-notification";
import { render } from "@react-email/render";

const resend = new Resend(process.env.RESEND_API_KEY);

export async function notifySubscribers(releaseId: string): Promise<void> {
  const release = await db.query.releases.findFirst({
    where: eq(releases.id, releaseId),
    with: {
      entries: true,
      organization: true,
    },
  });
  if (!release || release.status !== "published") return;

  const org = release.organization;
  const activeSubscribers = await db.query.subscribers.findMany({
    where: and(
      eq(subscribers.orgId, org.id),
      eq(subscribers.isActive, true),
    ),
  });

  const fromName = org.settings?.notificationFromName || org.name;
  const fromEmail = org.settings?.notificationFromEmail || "changelog@driftlog.io";

  // Group entries by category for the email
  const entriesByCategory = new Map<string, typeof release.entries>();
  for (const entry of release.entries) {
    const existing = entriesByCategory.get(entry.category) || [];
    existing.push(entry);
    entriesByCategory.set(entry.category, existing);
  }

  // Send to each subscriber (batched via Resend)
  const batch = activeSubscribers.map((subscriber) => {
    // Filter entries by subscriber preferences
    const prefs = subscriber.preferences || {};
    const relevantEntries = prefs.categories
      ? release.entries.filter((e) => prefs.categories!.includes(e.category))
      : release.entries;

    if (relevantEntries.length === 0) return null;

    return {
      from: `${fromName} <${fromEmail}>`,
      to: subscriber.email,
      subject: `${org.name}: ${release.title}`,
      html: render(
        ReleaseNotificationEmail({
          orgName: org.name,
          orgSlug: org.slug,
          releaseTitle: release.title,
          releaseVersion: release.version,
          entries: relevantEntries,
          changelogUrl: `https://${org.slug}.driftlog.io`,
          unsubscribeUrl: `${process.env.NEXT_PUBLIC_APP_URL}/unsubscribe/${subscriber.unsubscribeToken}`,
          branding: org.branding || {},
        }),
      ),
      tags: [
        { name: "release_id", value: releaseId },
        { name: "org_id", value: org.id },
      ],
    };
  }).filter(Boolean);

  // Send in batches of 100
  for (let i = 0; i < batch.length; i += 100) {
    const chunk = batch.slice(i, i + 100);
    await resend.batch.send(chunk as any);
  }

  // Record sends
  for (const subscriber of activeSubscribers) {
    await db.insert(notificationSends).values({
      subscriberId: subscriber.id,
      releaseId,
    });
  }
}
```

---

## Phase Breakdown

### Phase 1 — Project Scaffold + Auth + DB (Days 1–4)

**Day 1: Repository and framework setup**
```bash
npx create-next-app@latest driftlog --ts --tailwind --app --src-dir
cd driftlog
npm i drizzle-orm @neondatabase/serverless
npm i -D drizzle-kit
npm i @trpc/server @trpc/client @trpc/next @trpc/react-query superjson zod
npm i @clerk/nextjs
```
- `src/app/layout.tsx` — Clerk provider, tRPC provider
- `src/app/(dashboard)/layout.tsx` — authenticated shell with sidebar
- `src/server/db/index.ts` — Neon connection
- `drizzle.config.ts`
- `.env.local`

**Day 2: Full database schema**
- `src/server/db/schema.ts` — all 10 tables: organizations, repositories, releases, changelogEntries, webhookEvents, subscribers, notificationSends, pageViews, widgetConfigs
- All `relations()` blocks
- Run `npx drizzle-kit push`

**Day 3: tRPC setup and org initialization**
- `src/server/trpc.ts` — context, auth middleware, org-scoped procedure
- `src/server/routers/_app.ts` — root router
- `src/server/routers/org.ts` — getOrCreate, update settings, update branding

**Day 4: Auth flows**
- `src/app/(auth)/sign-in/[[...sign-in]]/page.tsx`
- `src/app/(auth)/sign-up/[[...sign-up]]/page.tsx`
- `src/server/lib/org-seed.ts` — create org + widget config on first login

### Phase 2 — GitHub App Integration (Days 5–10)

**Day 5: GitHub App setup**
- Register GitHub App in GitHub Developer Settings
- Configure permissions: read access to PRs, commits, releases, repository metadata
- Configure webhook events: pull_request (closed), release (published)
- `src/server/services/github.ts` — Octokit setup with App auth

**Day 6: GitHub OAuth installation flow**
- `src/app/api/auth/github/route.ts` — redirect to GitHub App installation
- `src/app/api/auth/github/callback/route.ts` — handle installation callback, save installation ID
- `src/server/routers/repository.ts` — list repos from installation, connect/disconnect repo

**Day 7: Repository management UI**
- `src/app/(dashboard)/repos/page.tsx` — connected repos list with "Connect GitHub" button
- `src/app/(dashboard)/repos/[id]/page.tsx` — repo settings: tone, audience, ignored paths, auto-publish toggle
- `src/components/repos/RepoCard.tsx` — repo card with status indicator
- `src/components/repos/RepoConfigForm.tsx` — per-repo config form

**Day 8: Webhook receiver and signature verification**
- `src/app/api/webhooks/github/route.ts` — verify HMAC signature, parse event type/action, store in webhook_events table, enqueue to BullMQ
- `src/server/lib/github-webhook-verify.ts` — HMAC-SHA256 signature validation
- Test: trigger webhook from GitHub → event stored in DB

**Day 9: Webhook processor — PR merged**
- `src/server/workers/webhook-processor.ts` — BullMQ worker for `pull_request.closed` (merged)
- Fetch PR diff and file list via Octokit
- Apply ignored paths filter
- Find or create draft release for the repo
- Enqueue AI generation

**Day 10: Webhook processor — GitHub Release published**
- Handle `release.published` events
- Create DriftLog release from GitHub release metadata (version, tag, title)
- Fetch all PRs merged since last release (compare commits)
- Generate entries for each PR
- Link entries to the release

### Phase 3 — AI Generation Pipeline (Days 11–14)

**Day 11: AI generation service**
```bash
npm i openai bullmq
```
- `src/server/services/ai-generation.ts` — two-step generation: classify + generate
- `src/server/lib/queue.ts` — BullMQ queue config with Upstash Redis
- Configurable tone and audience per repo

**Day 12: AI quality tuning**
- Test generation across 20 diverse PRs (feature, bugfix, refactor, dependency bump)
- Tune prompts for each category
- Add ignored patterns: CI config, lock files, test-only changes
- Handle edge cases: empty PR descriptions, massive diffs, non-English content

**Day 13: Regeneration and feedback**
- `src/server/services/ai-generation.ts` — `regenerateEntry` function with user feedback
- `src/server/routers/entry.ts` — `regenerate` mutation
- "Regenerate" button in UI with optional feedback text

**Day 14: Entry management**
- `src/server/routers/entry.ts` — CRUD for changelog entries
- `src/server/routers/release.ts` — CRUD for releases, status transitions (draft→review→published)
- Sort order management (drag to reorder)

### Phase 4 — Dashboard UI + Editor (Days 15–21)

**Day 15: Dashboard home**
- `src/app/(dashboard)/page.tsx` — overview cards: total releases, entries this month, subscribers
- Recent releases list
- Quick action buttons

**Day 16: Release list and management**
- `src/app/(dashboard)/releases/page.tsx` — release list with status tabs (all, draft, review, published)
- `src/server/routers/release.ts` — list with pagination, get with entries
- `src/components/releases/ReleaseCard.tsx` — release card with version, title, status, entry count, date

**Day 17: Release editor — entry list**
- `src/app/(dashboard)/releases/[id]/page.tsx` — release editor
- Left panel: draggable entry list (reorderable)
- Entry cards showing: category badge, title, source PR link
- Add manual entry button

**Day 18: Release editor — rich text**
```bash
npm i @tiptap/react @tiptap/starter-kit @tiptap/extension-placeholder
```
- Center panel: Tiptap rich text editor for selected entry
- Markdown ↔ rich text toggle
- AI draft vs. human-edited diff indicator
- "Regenerate" button per entry

**Day 19: Release publishing**
- Publish button with confirmation dialog
- Preview mode: see changelog page as users will see it
- Schedule publication: date/time picker
- `src/server/cron/publish-scheduled.ts` — publish releases at scheduled time
- `vercel.json` — cron job every 5 minutes

**Day 20: Manual release creation**
- `src/app/(dashboard)/releases/new/page.tsx` — create release form (title, version, repo)
- Add entries manually or select from draft entries
- Import PRs: list recent merged PRs, select which to generate entries for

**Day 21: Branding settings**
- `src/app/(dashboard)/settings/branding/page.tsx` — logo upload, color picker, font selector
- `src/components/settings/BrandingPreview.tsx` — live preview of changelog page with branding
- R2 upload for logo and favicon

### Phase 5 — Public Changelog Page + Subscribers (Days 22–27)

**Day 22: Public changelog page**
- `src/app/(public)/changelog/[orgSlug]/page.tsx` — ISR-rendered changelog
- `src/components/changelog/ChangelogTimeline.tsx` — release timeline with entries
- `src/components/changelog/EntryCard.tsx` — entry with category badge, title, body
- Responsive layout, dark mode toggle

**Day 23: Changelog page features**
- Category filter chips (feature, bugfix, improvement, etc.)
- Search bar (full-text across entries)
- Pagination (infinite scroll or page numbers)
- RSS feed: `src/app/(public)/changelog/[orgSlug]/rss/route.ts`
- SEO meta tags and structured data

**Day 24: Subscribe flow**
- Subscribe form on changelog page (email input)
- `src/app/api/public/subscribe/route.ts` — create subscriber with unsubscribe token
- `src/app/unsubscribe/[token]/page.tsx` — one-click unsubscribe page
- Email preference page: choose categories, frequency

**Day 25: Notification system**
- `src/server/services/notifications.ts` — send release notification to subscribers
- React Email template: release title, entries grouped by category, changelog link
- Resend batch sending (100 per batch)
- Track notification sends

**Day 26: Notification triggering**
- On release publish: trigger notification send
- `src/server/routers/release.ts` — publish mutation calls `notifySubscribers()`
- Notification history in dashboard
- `src/app/(dashboard)/subscribers/page.tsx` — subscriber list with count, unsubscribe rate

**Day 27: Email open/click tracking**
- Resend webhook for email events (opened, clicked)
- `src/app/api/webhooks/resend/route.ts` — update notification_sends with opened_at, clicked_at
- Display open/click rates in subscriber dashboard

### Phase 6 — Billing + Plan Enforcement (Days 28–31)

**Day 28: Stripe integration**
- `src/server/services/stripe.ts` — Stripe client
- `src/server/routers/billing.ts` — createCheckoutSession, createPortalSession
- `src/app/api/webhooks/stripe/route.ts` — subscription events
- Plan mapping: free (1 repo, 10 entries/mo, branded), pro ($19/mo: unlimited repos/entries, custom domain, widget, analytics), team ($49/mo: approval workflow, 10 members, Slack, API)

**Day 29: Plan enforcement**
- `src/server/lib/plan-limits.ts` — check limits:
  - Free: 1 repo, 10 entries/month, DriftLog branding, no widget, no analytics
  - Pro: unlimited repos/entries, remove branding, widget, custom domain, analytics
  - Team: all pro + approval workflow, team members, Slack integration
- Middleware enforcement on mutations

**Day 30: Billing UI**
- `src/app/(dashboard)/settings/billing/page.tsx`
- Plan comparison, usage meter, upgrade buttons

**Day 31: Plan-gated features**
- Widget config hidden on free plan
- Custom domain hidden on free plan
- Branding removal on paid plans
- Approval workflow hidden on free/pro

### Phase 7 — Analytics + Polish (Days 32–35)

**Day 32: Page view tracking**
- `src/app/api/public/track/route.ts` — beacon endpoint for changelog views
- Lightweight JS tracker on public changelog page
- Track: page, referrer, device, country

**Day 33: Analytics dashboard**
- `src/app/(dashboard)/analytics/page.tsx` — changelog analytics
- `src/components/analytics/ViewsChart.tsx` — page views over time
- `src/components/analytics/TopEntries.tsx` — most viewed entries
- `src/components/analytics/SubscriberGrowth.tsx` — subscriber count over time
- `src/components/analytics/NotificationMetrics.tsx` — open/click rates

**Day 34: Polish and edge cases**
- Webhook retry logic (exponential backoff, 3 retries)
- Handle GitHub API rate limits gracefully
- Large PR handling (>100 files: summarize top 30)
- Empty release handling (no entries → don't show)
- Duplicate webhook detection (idempotency key)

**Day 35: Launch preparation**
- GitHub Marketplace listing preparation
- End-to-end test: install app → PR merge → AI generates → edit → publish → subscriber notified
- Cross-browser changelog page testing
- Performance audit: changelog page load <1s
- Deploy to production

---

## Critical Files

```
driftlog/
├── src/
│   ├── app/
│   │   ├── layout.tsx                          # Root layout
│   │   ├── (auth)/
│   │   │   ├── sign-in/[[...sign-in]]/page.tsx
│   │   │   └── sign-up/[[...sign-up]]/page.tsx
│   │   ├── (dashboard)/
│   │   │   ├── layout.tsx                      # Dashboard shell
│   │   │   ├── page.tsx                        # Dashboard home
│   │   │   ├── repos/
│   │   │   │   ├── page.tsx                    # Connected repos list
│   │   │   │   └── [id]/page.tsx               # Repo settings
│   │   │   ├── releases/
│   │   │   │   ├── page.tsx                    # Release list
│   │   │   │   ├── new/page.tsx                # Create release
│   │   │   │   └── [id]/page.tsx               # Release editor
│   │   │   ├── subscribers/page.tsx            # Subscriber management
│   │   │   ├── analytics/page.tsx              # Analytics dashboard
│   │   │   └── settings/
│   │   │       ├── page.tsx                    # Org settings
│   │   │       ├── branding/page.tsx           # Logo, colors, fonts
│   │   │       └── billing/page.tsx            # Plan management
│   │   ├── (public)/
│   │   │   └── changelog/
│   │   │       └── [orgSlug]/
│   │   │           ├── page.tsx                # Public changelog (ISR)
│   │   │           └── rss/route.ts            # RSS feed
│   │   ├── unsubscribe/[token]/page.tsx        # Unsubscribe page
│   │   └── api/
│   │       ├── trpc/[trpc]/route.ts
│   │       ├── auth/github/
│   │       │   ├── route.ts                    # GitHub App install redirect
│   │       │   └── callback/route.ts           # Installation callback
│   │       ├── public/
│   │       │   ├── subscribe/route.ts          # Email subscribe
│   │       │   └── track/route.ts              # Page view tracking
│   │       ├── revalidate/route.ts             # ISR revalidation trigger
│   │       └── webhooks/
│   │           ├── github/route.ts             # GitHub webhook receiver
│   │           ├── stripe/route.ts             # Stripe billing
│   │           └── resend/route.ts             # Email open/click tracking
│   ├── server/
│   │   ├── db/
│   │   │   ├── index.ts                        # Neon connection
│   │   │   └── schema.ts                       # 10 tables + relations
│   │   ├── trpc.ts
│   │   ├── routers/
│   │   │   ├── _app.ts
│   │   │   ├── org.ts
│   │   │   ├── repository.ts                   # Repo CRUD + GitHub install
│   │   │   ├── release.ts                      # Release CRUD + publish
│   │   │   ├── entry.ts                        # Changelog entry CRUD + regenerate
│   │   │   ├── subscriber.ts                   # Subscriber management
│   │   │   ├── analytics.ts                    # Analytics queries
│   │   │   └── billing.ts                      # Stripe checkout/portal
│   │   ├── services/
│   │   │   ├── github.ts                       # GitHub App auth + API helpers
│   │   │   ├── ai-generation.ts                # Changelog generation engine
│   │   │   ├── notifications.ts                # Subscriber email notifications
│   │   │   ├── stripe.ts                       # Stripe client
│   │   │   ├── email.ts                        # Resend client
│   │   │   ├── r2.ts                           # R2 storage for assets
│   │   │   └── email-templates/
│   │   │       └── release-notification.tsx     # React Email template
│   │   ├── workers/
│   │   │   └── webhook-processor.ts            # BullMQ: process GitHub events
│   │   ├── lib/
│   │   │   ├── org-seed.ts
│   │   │   ├── plan-limits.ts
│   │   │   ├── queue.ts                        # BullMQ config
│   │   │   └── github-webhook-verify.ts        # HMAC signature validation
│   │   └── cron/
│   │       └── publish-scheduled.ts            # Publish scheduled releases
│   └── components/
│       ├── repos/
│       │   ├── RepoCard.tsx
│       │   └── RepoConfigForm.tsx
│       ├── releases/
│       │   ├── ReleaseCard.tsx
│       │   ├── EntryEditor.tsx                 # Tiptap rich text editor
│       │   └── EntryList.tsx                   # Draggable entry list
│       ├── changelog/
│       │   ├── ChangelogTimeline.tsx            # Public changelog layout
│       │   └── EntryCard.tsx                    # Public entry display
│       ├── analytics/
│       │   ├── ViewsChart.tsx
│       │   ├── TopEntries.tsx
│       │   ├── SubscriberGrowth.tsx
│       │   └── NotificationMetrics.tsx
│       ├── settings/
│       │   └── BrandingPreview.tsx
│       └── billing/
│           └── PlanCard.tsx
├── drizzle.config.ts
├── vercel.json                                 # Cron config
├── .env.local
└── package.json
```

---

## Verification

### Manual Testing Checklist

1. **GitHub App installation** — install app on test repo, verify repos appear in dashboard
2. **Repo configuration** — set tone, audience, ignored paths, verify saved
3. **PR merge → webhook** — merge PR on test repo, verify webhook received and event stored
4. **AI generation** — verify changelog entry generated with correct category and user-facing language
5. **Ignored patterns** — merge a CI-only PR (`.github/workflows/`), verify it's filtered out
6. **Release editor** — edit AI-generated entry, verify `human_edited` flag set, original AI text preserved
7. **Entry regeneration** — click "Regenerate", verify new text generated, old text replaced
8. **Drag-and-drop reorder** — reorder entries in release editor, verify sort order persists
9. **Release publishing** — publish release, verify changelog page updates within 60s (ISR)
10. **Public changelog page** — visit `/{orgSlug}`, verify entries display with correct branding
11. **Category filter** — filter by "bugfix", verify only bugfix entries shown
12. **RSS feed** — subscribe to RSS feed, verify entries appear in reader
13. **Email subscribe** — enter email on changelog page, verify welcome email received
14. **Release notification** — publish release with subscribers, verify notification emails sent
15. **Unsubscribe** — click unsubscribe link, verify subscriber deactivated
16. **Plan limits** — on free plan: try connecting second repo, verify blocked with upgrade prompt

### SQL Verification Queries

```sql
-- 1. Webhook processing health
SELECT status, COUNT(*) as count,
  AVG(EXTRACT(EPOCH FROM (processed_at - received_at)))::numeric(10,1) as avg_process_seconds
FROM webhook_events
WHERE received_at >= NOW() - INTERVAL '7 days'
GROUP BY status;

-- 2. AI generation quality (human edit rate)
SELECT
  COUNT(*) as total_entries,
  SUM(CASE WHEN human_edited THEN 1 ELSE 0 END) as edited_count,
  ROUND(100.0 * SUM(CASE WHEN human_edited THEN 1 ELSE 0 END) / COUNT(*), 1) as edit_rate_pct
FROM changelog_entries;

-- 3. Category distribution
SELECT category, COUNT(*) as count
FROM changelog_entries
GROUP BY category ORDER BY count DESC;

-- 4. Subscriber engagement
SELECT
  COUNT(DISTINCT ns.subscriber_id) as notified,
  SUM(CASE WHEN ns.opened_at IS NOT NULL THEN 1 ELSE 0 END) as opened,
  ROUND(100.0 * SUM(CASE WHEN ns.opened_at IS NOT NULL THEN 1 ELSE 0 END) /
    NULLIF(COUNT(*), 0), 1) as open_rate_pct
FROM notification_sends ns
WHERE ns.sent_at >= NOW() - INTERVAL '30 days';

-- 5. Releases per org
SELECT o.name, COUNT(r.id) as release_count,
  COUNT(CASE WHEN r.status = 'published' THEN 1 END) as published_count
FROM organizations o
LEFT JOIN releases r ON r.org_id = o.id
GROUP BY o.name ORDER BY release_count DESC;
```

### Performance Benchmarks

| Operation | Target | Method |
|-----------|--------|--------|
| Webhook response time | <200ms | Immediate 200, async processing |
| AI generation per entry | <5s | GPT-4o classify + generate |
| Full release generation (10 PRs) | <45s | Parallel AI calls, 3 concurrent |
| Changelog page load (ISR) | <500ms | Static HTML, edge-cached |
| Changelog page ISR revalidation | <3s | On-demand revalidation |
| Release editor load | <1s | Fetch release + entries |
| Email notification batch (100) | <5s | Resend batch API |
| Dashboard page load | <1.5s | Server components + tRPC |
| GitHub API calls per webhook | 2-3 | PR diff + files list |
| Webhook processing (end-to-end) | <30s | Receive → process → entry created |

---

## Public API Endpoints

### RSS Feed

`GET /changelog/{orgSlug}/rss`

Returns an RSS 2.0 XML feed of published releases for the given organization. Supports `If-Modified-Since` and `ETag` headers for cache efficiency. Each `<item>` includes the release title, publication date, and all changelog entries concatenated as the description. The feed is regenerated on ISR revalidation.

### JSON Changelog API

`GET /api/public/changelog/{orgSlug}`

Returns a JSON representation of published changelog releases. Supports optional query parameters:

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `page` | integer | 1 | Page number |
| `limit` | integer | 10 | Items per page (max 50) |
| `category` | string | all | Filter by category (feature, bugfix, improvement, etc.) |
| `since` | ISO 8601 date | none | Only return releases published after this date |

Response format:

```json
{
  "releases": [
    {
      "id": "uuid",
      "version": "v2.1.0",
      "title": "Release Title",
      "publishedAt": "2026-02-01T00:00:00Z",
      "entries": [
        {
          "category": "feature",
          "title": "Entry Title",
          "body": "Markdown description"
        }
      ]
    }
  ],
  "pagination": {
    "page": 1,
    "limit": 10,
    "totalPages": 5,
    "totalCount": 48
  }
}
```

---

## Implementation Progress

| Phase | Step | Description | Status |
|-------|------|-------------|--------|
| 1 | Day 1 | Project scaffold + environment | Done |
| 1 | Day 2 | Database schema + Clerk auth | Done |
| 1 | Day 3 | tRPC setup + org initialization | Done |
| 1 | Day 4 | Auth flows | Done |
| 2 | Day 5 | GitHub App setup | Done |
| 2 | Day 6 | GitHub OAuth installation flow | Done |
| 2 | Day 7 | Repository management UI + settings page | Done |
| 2 | Day 8 | Webhook receiver + signature verification | Not started |
| 2 | Day 9 | Webhook processor — PR merged | Not started |
| 2 | Day 10 | Webhook processor — GitHub Release published | Not started |
| 3 | Day 11 | AI generation service | Not started |
| 3 | Day 12 | AI quality tuning | Not started |
| 3 | Day 13 | Regeneration and feedback | Not started |
| 3 | Day 14 | Entry management | Not started |
| 4 | Day 15 | Dashboard home | Not started |
| 4 | Day 16 | Release list and management | Not started |
| 4 | Day 17 | Release editor — entry list | Not started |
| 4 | Day 18 | Release editor — rich text | Not started |
| 4 | Day 19 | Release publishing | Not started |
| 4 | Day 20 | Manual release creation | Not started |
| 4 | Day 21 | Branding settings | Not started |
| 5 | Day 22 | Public changelog page | Not started |
| 5 | Day 23 | Changelog page features | Not started |
| 5 | Day 24 | Subscribe flow | Not started |
| 5 | Day 25 | Notification system | Not started |
| 5 | Day 26 | Notification triggering | Not started |
| 5 | Day 27 | Email open/click tracking | Not started |
| 6 | Day 28 | Stripe integration | Not started |
| 6 | Day 29 | Plan enforcement | Not started |
| 6 | Day 30 | Billing UI | Not started |
| 6 | Day 31 | Plan-gated features | Not started |
| 7 | Day 32 | Page view tracking | Not started |
| 7 | Day 33 | Analytics dashboard | Not started |
| 7 | Day 34 | Polish and edge cases | Not started |
| 7 | Day 35 | Launch preparation | Not started |

**Last updated:** 2026-02-27 — Phase 2 Day 7 (repository settings page with tone, audience, auto-publish, ignored paths configuration)

---

## Post-MVP Roadmap

| Feature | Description | Priority |
|---------|-------------|----------|
| GitLab Integration | Support GitLab webhooks and MR events as a changelog source, mirroring the GitHub App model | High |
| Bitbucket Support | Bitbucket Cloud webhook integration for PR-based changelog generation | Medium |
| Monorepo Per-Package Changelogs | Detect package boundaries in monorepos and generate separate changelogs per package using path-based filtering and scoped release management | High |
| Multi-Language Support | AI changelog generation in non-English languages (French, German, Japanese, etc.) with language detection from repo settings and per-org language preferences | Medium |
