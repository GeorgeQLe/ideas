# 17. BugBounce — Smart Bug Report Manager

## Implementation Plan

**MVP Scope:** In-app JavaScript widget (Preact + Shadow DOM, <15KB gzipped) with screenshot capture (html2canvas), console log interception, and browser info auto-collection, bug report dashboard with CRUD and filtering, AI deduplication via embedding similarity (pgvector + text-embedding-3-small, 0.82 threshold), auto-priority scoring (frequency + severity + recency), Jira integration (create issues, one-way sync), email intake channel (forward-to address), Stripe billing with three tiers (Free / Pro $29/mo / Team $79/mo).

---

## Tech Decisions

| Layer | Technology | Notes |
|-------|-----------|-------|
| Framework | Next.js 14+ App Router | `src/` directory, TypeScript strict |
| API | tRPC v11 | superjson transformer, Clerk auth context |
| Database | Neon PostgreSQL + pgvector | `vector(1536)` for dedup embeddings |
| ORM | Drizzle ORM | Push-based migrations, typed schema |
| Auth | Clerk | Organization-scoped, `orgId` from session |
| Billing | Stripe | Checkout Sessions + Customer Portal + Webhooks |
| AI — Categorization | OpenAI GPT-4o-mini | JSON mode, temperature 0.1 for classification |
| AI — Embeddings | text-embedding-3-small | 1536 dimensions, cosine similarity for dedup |
| Queue | BullMQ + Redis (Upstash) | Async AI processing for incoming reports |
| Storage | Cloudflare R2 | Screenshots, file attachments |
| Widget | Preact 10 + Shadow DOM + html2canvas | <15KB (excl. html2canvas lazy-loaded) |
| Widget CDN | Cloudflare R2 + CDN | Edge-cached widget bundle |
| Email Intake | Resend (inbound) or Cloudflare Email Workers | Parse forwarded bug reports |
| Issue Tracker | Jira Cloud REST API (OAuth 2.0) | Create issues, field mapping |
| UI | Tailwind CSS, Radix UI, Recharts | |
| Hosting | Vercel (app), Cloudflare (widget + R2) | |
| Monitoring | Sentry, Vercel Analytics | |

### Key Architecture Decisions

1. **Two-tier bug model (BugReport + BugSubmission)**: A `BugReport` represents a unique bug. Each individual report from any channel (widget, email, API) creates a `BugSubmission`. When AI detects a duplicate, the new submission is linked to the existing BugReport and the `report_count` is incremented. This cleanly separates deduplication from data collection.

2. **Embedding-based deduplication over keyword matching**: Each BugSubmission is embedded using the concatenation of title + description + error messages. A pgvector cosine similarity query finds existing BugReports above a configurable threshold (default 0.82). Matches are surfaced as "AI suggested match" for human confirmation. Above 0.95 threshold, auto-merge is enabled (configurable).

3. **BullMQ for async AI processing**: Widget submissions and email intake return 200 immediately. A BullMQ worker processes: (1) AI categorization, (2) embedding generation, (3) dedup matching, (4) priority scoring. This keeps submission response times under 300ms.

4. **html2canvas lazy-loaded on demand**: The core widget bundle (<15KB) doesn't include html2canvas (~40KB). It's loaded dynamically only when the user clicks "Take Screenshot". This keeps initial load fast while still providing screenshot capability.

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
  vector,
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
  scoringConfig: jsonb("scoring_config").default({
    frequencyWeight: 0.35,
    severityWeight: 0.30,
    recencyWeight: 0.20,
    userImpactWeight: 0.15,
    recencyHalfLifeDays: 14,
  }).$type<{
    frequencyWeight: number;
    severityWeight: number;
    recencyWeight: number;
    userImpactWeight: number;
    recencyHalfLifeDays: number;
  }>(),
  dedupConfig: jsonb("dedup_config").default({
    autoMergeThreshold: 0.95,
    suggestThreshold: 0.82,
  }).$type<{
    autoMergeThreshold: number;
    suggestThreshold: number;
  }>(),
  inboundEmailAddress: text("inbound_email_address"), // bugs@slug.bugbounce.io
  settings: jsonb("settings").default({}).$type<{
    brandColor?: string;
    notificationFromName?: string;
  }>(),
  createdAt: timestamp("created_at", { withTimezone: true }).defaultNow().notNull(),
  updatedAt: timestamp("updated_at", { withTimezone: true }).defaultNow().notNull(),
});

// ---------------------------------------------------------------------------
// Bug Reports (unique bugs, after deduplication)
// ---------------------------------------------------------------------------
export const bugReports = pgTable(
  "bug_reports",
  {
    id: uuid("id").primaryKey().defaultRandom(),
    orgId: uuid("org_id")
      .references(() => organizations.id, { onDelete: "cascade" })
      .notNull(),
    title: text("title").notNull(),
    description: text("description").notNull(),
    status: text("status").default("new").notNull(),
    // new | triaged | in_progress | resolved | closed | wont_fix
    priority: text("priority").default("medium").notNull(),
    // critical | high | medium | low
    priorityScore: real("priority_score").default(0).notNull(),
    category: text("category").default("other").notNull(),
    // crash | functional | ui | performance | security | other
    reportCount: integer("report_count").default(1).notNull(),
    affectedUserCount: integer("affected_user_count").default(1).notNull(),
    assignedToUserId: text("assigned_to_user_id"), // Clerk user ID
    externalIssueId: text("external_issue_id"), // Jira issue key
    externalIssueUrl: text("external_issue_url"),
    embedding: vector("embedding", { dimensions: 1536 }),
    firstResponseAt: timestamp("first_response_at", { withTimezone: true }),
    resolvedAt: timestamp("resolved_at", { withTimezone: true }),
    slaTier: text("sla_tier").default("standard").notNull(),
    // standard (72h response, 14d resolution) | priority (24h response, 7d resolution) |
    // critical (4h response, 48h resolution)
    slaResponseDueAt: timestamp("sla_response_due_at", { withTimezone: true }),
    slaResolutionDueAt: timestamp("sla_resolution_due_at", { withTimezone: true }),
    slaBreached: boolean("sla_breached").default(false).notNull(),
    createdAt: timestamp("created_at", { withTimezone: true }).defaultNow().notNull(),
    updatedAt: timestamp("updated_at", { withTimezone: true }).defaultNow().notNull(),
  },
  (t) => [
    index("bug_report_org_idx").on(t.orgId),
    index("bug_report_status_idx").on(t.status),
    index("bug_report_priority_idx").on(t.priority),
    index("bug_report_category_idx").on(t.category),
    index("bug_report_created_idx").on(t.createdAt),
    // HNSW index for embedding similarity search
    index("bug_report_embedding_idx").using(
      "hnsw",
      t.embedding.op("vector_cosine_ops"),
    ),
  ],
);

// ---------------------------------------------------------------------------
// Bug Submissions (individual reports, many-to-one with BugReport)
// ---------------------------------------------------------------------------
export const bugSubmissions = pgTable(
  "bug_submissions",
  {
    id: uuid("id").primaryKey().defaultRandom(),
    bugReportId: uuid("bug_report_id")
      .references(() => bugReports.id, { onDelete: "cascade" })
      .notNull(),
    orgId: uuid("org_id")
      .references(() => organizations.id, { onDelete: "cascade" })
      .notNull(),
    source: text("source").default("widget").notNull(),
    // widget | email | slack | api | web_form
    reporterName: text("reporter_name"),
    reporterEmail: text("reporter_email"),
    reporterUserId: text("reporter_user_id"), // end-user ID from widget
    description: text("description").notNull(),
    browserInfo: jsonb("browser_info").$type<{
      browser?: string;
      browserVersion?: string;
      os?: string;
      osVersion?: string;
      screenWidth?: number;
      screenHeight?: number;
      url?: string;
      userAgent?: string;
      locale?: string;
    }>(),
    consoleLogs: jsonb("console_logs").default([]).$type<
      Array<{
        level: "error" | "warn" | "info" | "log";
        message: string;
        timestamp: string;
        stack?: string;
      }>
    >(),
    networkErrors: jsonb("network_errors").default([]).$type<
      Array<{
        url: string;
        method: string;
        status: number;
        statusText: string;
        timestamp: string;
      }>
    >(),
    screenshotUrls: jsonb("screenshot_urls").default([]).$type<string[]>(),
    attachmentUrls: jsonb("attachment_urls").default([]).$type<string[]>(),
    customFields: jsonb("custom_fields").default({}).$type<
      Record<string, string | number | boolean>
    >(),
    userContext: jsonb("user_context").default({}).$type<{
      plan?: string;
      role?: string;
      accountName?: string;
      accountId?: string;
    }>(),
    embedding: vector("embedding", { dimensions: 1536 }),
    submittedAt: timestamp("submitted_at", { withTimezone: true }).defaultNow().notNull(),
  },
  (t) => [
    index("bug_sub_report_idx").on(t.bugReportId),
    index("bug_sub_org_idx").on(t.orgId),
    index("bug_sub_source_idx").on(t.source),
    index("bug_sub_submitted_idx").on(t.submittedAt),
  ],
);

// ---------------------------------------------------------------------------
// Bug Comments
// ---------------------------------------------------------------------------
export const bugComments = pgTable(
  "bug_comments",
  {
    id: uuid("id").primaryKey().defaultRandom(),
    bugReportId: uuid("bug_report_id")
      .references(() => bugReports.id, { onDelete: "cascade" })
      .notNull(),
    userId: text("user_id").notNull(), // Clerk user ID
    text: text("text").notNull(),
    isInternal: boolean("is_internal").default(true).notNull(),
    createdAt: timestamp("created_at", { withTimezone: true }).defaultNow().notNull(),
  },
  (t) => [
    index("bug_comment_report_idx").on(t.bugReportId),
  ],
);

// ---------------------------------------------------------------------------
// Bug Status Changes (audit trail)
// ---------------------------------------------------------------------------
export const bugStatusChanges = pgTable(
  "bug_status_changes",
  {
    id: uuid("id").primaryKey().defaultRandom(),
    bugReportId: uuid("bug_report_id")
      .references(() => bugReports.id, { onDelete: "cascade" })
      .notNull(),
    fromStatus: text("from_status"),
    toStatus: text("to_status").notNull(),
    changedBy: text("changed_by").notNull(), // Clerk user ID or "system"
    note: text("note"),
    changedAt: timestamp("changed_at", { withTimezone: true }).defaultNow().notNull(),
  },
  (t) => [
    index("bug_status_report_idx").on(t.bugReportId),
  ],
);

// ---------------------------------------------------------------------------
// Dedup Suggestions (pending human review)
// ---------------------------------------------------------------------------
export const dedupSuggestions = pgTable(
  "dedup_suggestions",
  {
    id: uuid("id").primaryKey().defaultRandom(),
    orgId: uuid("org_id")
      .references(() => organizations.id, { onDelete: "cascade" })
      .notNull(),
    submissionId: uuid("submission_id")
      .references(() => bugSubmissions.id, { onDelete: "cascade" })
      .notNull(),
    matchedBugReportId: uuid("matched_bug_report_id")
      .references(() => bugReports.id, { onDelete: "cascade" })
      .notNull(),
    similarityScore: real("similarity_score").notNull(),
    status: text("status").default("pending").notNull(), // pending | accepted | rejected
    reviewedBy: text("reviewed_by"), // Clerk user ID
    reviewedAt: timestamp("reviewed_at", { withTimezone: true }),
    createdAt: timestamp("created_at", { withTimezone: true }).defaultNow().notNull(),
  },
  (t) => [
    index("dedup_org_idx").on(t.orgId),
    index("dedup_status_idx").on(t.status),
  ],
);

// ---------------------------------------------------------------------------
// Widget Configuration
// ---------------------------------------------------------------------------
export const widgetConfigs = pgTable("widget_configs", {
  id: uuid("id").primaryKey().defaultRandom(),
  orgId: uuid("org_id")
    .references(() => organizations.id, { onDelete: "cascade" })
    .unique()
    .notNull(),
  theme: jsonb("theme").default({}).$type<{
    primaryColor?: string;
    backgroundColor?: string;
    textColor?: string;
    position?: "bottom-right" | "bottom-left" | "top-right" | "top-left";
    borderRadius?: number;
  }>(),
  customFields: jsonb("custom_fields").default([]).$type<
    Array<{
      id: string;
      type: "text" | "select" | "checkbox";
      label: string;
      required: boolean;
      options?: string[];
    }>
  >(),
  triggerType: text("trigger_type").default("button").notNull(),
  // button | keyboard | programmatic
  keyboardShortcut: text("keyboard_shortcut").default("ctrl+shift+b"),
  screenshotEnabled: boolean("screenshot_enabled").default(true).notNull(),
  consoleCaptureEnabled: boolean("console_capture_enabled").default(true).notNull(),
  networkCaptureEnabled: boolean("network_capture_enabled").default(false).notNull(),
  showPoweredBy: boolean("show_powered_by").default(true).notNull(),
  createdAt: timestamp("created_at", { withTimezone: true }).defaultNow().notNull(),
});

// ---------------------------------------------------------------------------
// Integration Configurations
// ---------------------------------------------------------------------------
export const integrationConfigs = pgTable(
  "integration_configs",
  {
    id: uuid("id").primaryKey().defaultRandom(),
    orgId: uuid("org_id")
      .references(() => organizations.id, { onDelete: "cascade" })
      .notNull(),
    type: text("type").notNull(), // jira | linear | github
    credentials: text("credentials").notNull(), // AES-256-GCM encrypted JSON
    config: jsonb("config").default({}).$type<{
      projectKey?: string; // Jira project key
      issueTypeName?: string; // Jira issue type
      defaultLabels?: string[];
      fieldMapping?: Record<string, string>; // BugBounce field → Jira field
      autoSync?: boolean;
    }>(),
    isActive: boolean("is_active").default(true).notNull(),
    lastSyncedAt: timestamp("last_synced_at", { withTimezone: true }),
    createdAt: timestamp("created_at", { withTimezone: true }).defaultNow().notNull(),
  },
  (t) => [
    uniqueIndex("integration_org_type_idx").on(t.orgId, t.type),
  ],
);

// ---------------------------------------------------------------------------
// Inbound Emails (raw storage before processing)
// ---------------------------------------------------------------------------
export const inboundEmails = pgTable(
  "inbound_emails",
  {
    id: uuid("id").primaryKey().defaultRandom(),
    orgId: uuid("org_id")
      .references(() => organizations.id, { onDelete: "cascade" })
      .notNull(),
    fromAddress: text("from_address").notNull(),
    subject: text("subject"),
    bodyText: text("body_text"),
    bodyHtml: text("body_html"),
    attachmentUrls: jsonb("attachment_urls").default([]).$type<string[]>(),
    processedAt: timestamp("processed_at", { withTimezone: true }),
    bugSubmissionId: uuid("bug_submission_id"),
    status: text("status").default("pending").notNull(), // pending | processed | failed
    receivedAt: timestamp("received_at", { withTimezone: true }).defaultNow().notNull(),
  },
  (t) => [
    index("inbound_email_org_idx").on(t.orgId),
  ],
);

// ---------------------------------------------------------------------------
// Reporter Notifications (track what we've notified reporters about)
// ---------------------------------------------------------------------------
export const reporterNotifications = pgTable(
  "reporter_notifications",
  {
    id: uuid("id").primaryKey().defaultRandom(),
    bugReportId: uuid("bug_report_id")
      .references(() => bugReports.id, { onDelete: "cascade" })
      .notNull(),
    reporterEmail: text("reporter_email").notNull(),
    type: text("type").notNull(), // acknowledged | in_progress | resolved
    sentAt: timestamp("sent_at", { withTimezone: true }).defaultNow().notNull(),
  },
  (t) => [
    index("reporter_notif_report_idx").on(t.bugReportId),
  ],
);

// ---------------------------------------------------------------------------
// Relations
// ---------------------------------------------------------------------------
export const organizationsRelations = relations(organizations, ({ one, many }) => ({
  bugReports: many(bugReports),
  bugSubmissions: many(bugSubmissions),
  widgetConfig: one(widgetConfigs),
  integrationConfigs: many(integrationConfigs),
  inboundEmails: many(inboundEmails),
  dedupSuggestions: many(dedupSuggestions),
}));

export const bugReportsRelations = relations(bugReports, ({ one, many }) => ({
  organization: one(organizations, {
    fields: [bugReports.orgId],
    references: [organizations.id],
  }),
  submissions: many(bugSubmissions),
  comments: many(bugComments),
  statusChanges: many(bugStatusChanges),
  dedupSuggestions: many(dedupSuggestions),
  reporterNotifications: many(reporterNotifications),
}));

export const bugSubmissionsRelations = relations(bugSubmissions, ({ one }) => ({
  bugReport: one(bugReports, {
    fields: [bugSubmissions.bugReportId],
    references: [bugReports.id],
  }),
  organization: one(organizations, {
    fields: [bugSubmissions.orgId],
    references: [organizations.id],
  }),
}));

export const bugCommentsRelations = relations(bugComments, ({ one }) => ({
  bugReport: one(bugReports, {
    fields: [bugComments.bugReportId],
    references: [bugReports.id],
  }),
}));

export const bugStatusChangesRelations = relations(bugStatusChanges, ({ one }) => ({
  bugReport: one(bugReports, {
    fields: [bugStatusChanges.bugReportId],
    references: [bugReports.id],
  }),
}));

export const dedupSuggestionsRelations = relations(dedupSuggestions, ({ one }) => ({
  organization: one(organizations, {
    fields: [dedupSuggestions.orgId],
    references: [organizations.id],
  }),
  submission: one(bugSubmissions, {
    fields: [dedupSuggestions.submissionId],
    references: [bugSubmissions.id],
  }),
  matchedBugReport: one(bugReports, {
    fields: [dedupSuggestions.matchedBugReportId],
    references: [bugReports.id],
  }),
}));

export const widgetConfigsRelations = relations(widgetConfigs, ({ one }) => ({
  organization: one(organizations, {
    fields: [widgetConfigs.orgId],
    references: [organizations.id],
  }),
}));

export const integrationConfigsRelations = relations(integrationConfigs, ({ one }) => ({
  organization: one(organizations, {
    fields: [integrationConfigs.orgId],
    references: [organizations.id],
  }),
}));

export const inboundEmailsRelations = relations(inboundEmails, ({ one }) => ({
  organization: one(organizations, {
    fields: [inboundEmails.orgId],
    references: [organizations.id],
  }),
}));

export const reporterNotificationsRelations = relations(reporterNotifications, ({ one }) => ({
  bugReport: one(bugReports, {
    fields: [reporterNotifications.bugReportId],
    references: [bugReports.id],
  }),
}));
```

---

## Initial Migration SQL

```sql
-- Enable required extensions
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
CREATE EXTENSION IF NOT EXISTS "vector";

-- Organizations
CREATE TABLE organizations (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  clerk_org_id TEXT UNIQUE NOT NULL,
  name TEXT NOT NULL,
  slug TEXT UNIQUE NOT NULL,
  plan TEXT NOT NULL DEFAULT 'free',
  stripe_customer_id TEXT,
  stripe_subscription_id TEXT,
  scoring_config JSONB DEFAULT '{"frequencyWeight":0.35,"severityWeight":0.30,"recencyWeight":0.20,"userImpactWeight":0.15,"recencyHalfLifeDays":14}',
  dedup_config JSONB DEFAULT '{"autoMergeThreshold":0.95,"suggestThreshold":0.82}',
  inbound_email_address TEXT,
  settings JSONB DEFAULT '{}',
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Bug Reports
CREATE TABLE bug_reports (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  org_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
  title TEXT NOT NULL,
  description TEXT NOT NULL,
  status TEXT NOT NULL DEFAULT 'new',
  priority TEXT NOT NULL DEFAULT 'medium',
  priority_score REAL NOT NULL DEFAULT 0,
  category TEXT NOT NULL DEFAULT 'other',
  report_count INTEGER NOT NULL DEFAULT 1,
  affected_user_count INTEGER NOT NULL DEFAULT 1,
  assigned_to_user_id TEXT,
  external_issue_id TEXT,
  external_issue_url TEXT,
  embedding vector(1536),
  first_response_at TIMESTAMPTZ,
  resolved_at TIMESTAMPTZ,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX bug_report_org_idx ON bug_reports(org_id);
CREATE INDEX bug_report_status_idx ON bug_reports(status);
CREATE INDEX bug_report_priority_idx ON bug_reports(priority);
CREATE INDEX bug_report_category_idx ON bug_reports(category);
CREATE INDEX bug_report_created_idx ON bug_reports(created_at);
CREATE INDEX bug_report_embedding_idx ON bug_reports
  USING hnsw (embedding vector_cosine_ops);

-- Bug Submissions
CREATE TABLE bug_submissions (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  bug_report_id UUID NOT NULL REFERENCES bug_reports(id) ON DELETE CASCADE,
  org_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
  source TEXT NOT NULL DEFAULT 'widget',
  reporter_name TEXT,
  reporter_email TEXT,
  reporter_user_id TEXT,
  description TEXT NOT NULL,
  browser_info JSONB,
  console_logs JSONB DEFAULT '[]',
  network_errors JSONB DEFAULT '[]',
  screenshot_urls JSONB DEFAULT '[]',
  attachment_urls JSONB DEFAULT '[]',
  custom_fields JSONB DEFAULT '{}',
  user_context JSONB DEFAULT '{}',
  embedding vector(1536),
  submitted_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX bug_sub_report_idx ON bug_submissions(bug_report_id);
CREATE INDEX bug_sub_org_idx ON bug_submissions(org_id);
CREATE INDEX bug_sub_source_idx ON bug_submissions(source);
CREATE INDEX bug_sub_submitted_idx ON bug_submissions(submitted_at);

-- Bug Comments
CREATE TABLE bug_comments (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  bug_report_id UUID NOT NULL REFERENCES bug_reports(id) ON DELETE CASCADE,
  user_id TEXT NOT NULL,
  text TEXT NOT NULL,
  is_internal BOOLEAN NOT NULL DEFAULT true,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX bug_comment_report_idx ON bug_comments(bug_report_id);

-- Bug Status Changes
CREATE TABLE bug_status_changes (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  bug_report_id UUID NOT NULL REFERENCES bug_reports(id) ON DELETE CASCADE,
  from_status TEXT,
  to_status TEXT NOT NULL,
  changed_by TEXT NOT NULL,
  note TEXT,
  changed_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX bug_status_report_idx ON bug_status_changes(bug_report_id);

-- Dedup Suggestions
CREATE TABLE dedup_suggestions (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  org_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
  submission_id UUID NOT NULL REFERENCES bug_submissions(id) ON DELETE CASCADE,
  matched_bug_report_id UUID NOT NULL REFERENCES bug_reports(id) ON DELETE CASCADE,
  similarity_score REAL NOT NULL,
  status TEXT NOT NULL DEFAULT 'pending',
  reviewed_by TEXT,
  reviewed_at TIMESTAMPTZ,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX dedup_org_idx ON dedup_suggestions(org_id);
CREATE INDEX dedup_status_idx ON dedup_suggestions(status);

-- Widget Configuration
CREATE TABLE widget_configs (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  org_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE UNIQUE,
  theme JSONB DEFAULT '{}',
  custom_fields JSONB DEFAULT '[]',
  trigger_type TEXT NOT NULL DEFAULT 'button',
  keyboard_shortcut TEXT DEFAULT 'ctrl+shift+b',
  screenshot_enabled BOOLEAN NOT NULL DEFAULT true,
  console_capture_enabled BOOLEAN NOT NULL DEFAULT true,
  network_capture_enabled BOOLEAN NOT NULL DEFAULT false,
  show_powered_by BOOLEAN NOT NULL DEFAULT true,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Integration Configurations
CREATE TABLE integration_configs (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  org_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
  type TEXT NOT NULL,
  credentials TEXT NOT NULL,
  config JSONB DEFAULT '{}',
  is_active BOOLEAN NOT NULL DEFAULT true,
  last_synced_at TIMESTAMPTZ,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE UNIQUE INDEX integration_org_type_idx ON integration_configs(org_id, type);

-- Inbound Emails
CREATE TABLE inbound_emails (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  org_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
  from_address TEXT NOT NULL,
  subject TEXT,
  body_text TEXT,
  body_html TEXT,
  attachment_urls JSONB DEFAULT '[]',
  processed_at TIMESTAMPTZ,
  bug_submission_id UUID,
  status TEXT NOT NULL DEFAULT 'pending',
  received_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX inbound_email_org_idx ON inbound_emails(org_id);

-- Reporter Notifications
CREATE TABLE reporter_notifications (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  bug_report_id UUID NOT NULL REFERENCES bug_reports(id) ON DELETE CASCADE,
  reporter_email TEXT NOT NULL,
  type TEXT NOT NULL,
  sent_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX reporter_notif_report_idx ON reporter_notifications(bug_report_id);
```

---

## Architecture Deep-Dives

### 1. Bug Submission Processing Pipeline (BullMQ Worker)

When a new bug submission arrives from any channel (widget, email, API), it's enqueued for async processing. The worker handles AI categorization, embedding generation, deduplication matching, and priority score computation.

```typescript
// src/server/workers/bug-processor.ts

import { Worker, Queue } from "bullmq";
import { OpenAI } from "openai";
import { and, eq, sql, desc } from "drizzle-orm";
import { db } from "../db";
import { bugReports, bugSubmissions, dedupSuggestions } from "../db/schema";
import { computePriorityScore } from "../services/priority";

const openai = new OpenAI();

interface BugProcessingJob {
  submissionId: string;
  bugReportId: string;
  orgId: string;
}

export const bugProcessingQueue = new Queue<BugProcessingJob>("bug-processing", {
  connection: { url: process.env.UPSTASH_REDIS_URL },
});

const worker = new Worker<BugProcessingJob>(
  "bug-processing",
  async (job) => {
    const { submissionId, bugReportId, orgId } = job.data;

    const submission = await db.query.bugSubmissions.findFirst({
      where: eq(bugSubmissions.id, submissionId),
    });
    if (!submission) return;

    const org = await db.query.organizations.findFirst({
      where: eq(organizations.id, orgId),
    });
    if (!org) return;

    // Step 1: AI categorization
    const categoryResult = await openai.chat.completions.create({
      model: "gpt-4o-mini",
      temperature: 0.1,
      response_format: { type: "json_object" },
      messages: [
        {
          role: "system",
          content: `You are a bug report classifier. Analyze the bug report and return JSON:
{
  "category": "crash" | "functional" | "ui" | "performance" | "security" | "other",
  "severity": "critical" | "high" | "medium" | "low",
  "title": "concise title for this bug (max 80 chars)",
  "summary": "one-sentence technical summary"
}`,
        },
        {
          role: "user",
          content: `Description: ${submission.description}
Console errors: ${JSON.stringify(submission.consoleLogs?.slice(0, 5))}
URL: ${submission.browserInfo?.url || "unknown"}
Browser: ${submission.browserInfo?.browser || "unknown"}`,
        },
      ],
    });

    const classification = JSON.parse(
      categoryResult.choices[0].message.content || "{}",
    );

    // Step 2: Generate embedding for deduplication
    const textToEmbed = [
      classification.title || "",
      submission.description,
      ...(submission.consoleLogs || [])
        .filter((l) => l.level === "error")
        .map((l) => l.message)
        .slice(0, 3),
    ].join(" ");

    const embeddingResult = await openai.embeddings.create({
      model: "text-embedding-3-small",
      input: textToEmbed,
    });
    const embedding = embeddingResult.data[0].embedding;

    // Update submission with embedding
    await db
      .update(bugSubmissions)
      .set({ embedding })
      .where(eq(bugSubmissions.id, submissionId));

    // Step 3: Find similar existing bugs (deduplication)
    const dedupThresholds = org.dedupConfig || {
      autoMergeThreshold: 0.95,
      suggestThreshold: 0.82,
    };

    const similarBugs = await db.execute<{
      id: string;
      title: string;
      similarity: number;
    }>(sql`
      SELECT id, title,
        1 - (embedding <=> ${sql.raw(`'[${embedding.join(",")}]'::vector`)}) as similarity
      FROM bug_reports
      WHERE org_id = ${orgId}
        AND id != ${bugReportId}
        AND status NOT IN ('closed', 'wont_fix')
        AND embedding IS NOT NULL
      ORDER BY embedding <=> ${sql.raw(`'[${embedding.join(",")}]'::vector`)}
      LIMIT 5
    `);

    const matches = similarBugs.rows.filter(
      (r) => r.similarity >= dedupThresholds.suggestThreshold,
    );

    if (matches.length > 0) {
      const topMatch = matches[0];

      if (topMatch.similarity >= dedupThresholds.autoMergeThreshold) {
        // Auto-merge: move submission to existing bug report
        await db
          .update(bugSubmissions)
          .set({ bugReportId: topMatch.id })
          .where(eq(bugSubmissions.id, submissionId));

        // Increment report count on target
        await db
          .update(bugReports)
          .set({
            reportCount: sql`report_count + 1`,
            affectedUserCount: sql`affected_user_count + 1`,
            updatedAt: new Date(),
          })
          .where(eq(bugReports.id, topMatch.id));

        // Delete the auto-created BugReport (now orphaned)
        await db
          .delete(bugReports)
          .where(eq(bugReports.id, bugReportId));

        // Recompute priority for the merged target
        await computePriorityScore(db, topMatch.id);
        return;
      }

      // Below auto-merge but above suggest threshold: create suggestion
      for (const match of matches) {
        await db.insert(dedupSuggestions).values({
          orgId,
          submissionId,
          matchedBugReportId: match.id,
          similarityScore: match.similarity,
          status: "pending",
        });
      }
    }

    // Step 4: Update the BugReport with AI results
    await db
      .update(bugReports)
      .set({
        title: classification.title || submission.description.slice(0, 80),
        category: classification.category || "other",
        priority: classification.severity || "medium",
        embedding,
        updatedAt: new Date(),
      })
      .where(eq(bugReports.id, bugReportId));

    // Step 5: Compute priority score
    await computePriorityScore(db, bugReportId);
  },
  {
    connection: { url: process.env.UPSTASH_REDIS_URL },
    concurrency: 5,
    limiter: {
      max: 20,
      duration: 60_000, // 20 jobs per minute (OpenAI rate limits)
    },
  },
);
```

### 2. Priority Scoring Algorithm

Priority scores are computed as a weighted combination of frequency, severity, recency, and user impact. Scores are recomputed whenever a new submission is merged or when manually triggered.

```typescript
// src/server/services/priority.ts

import { eq, sql, and, gte } from "drizzle-orm";
import { subDays } from "date-fns";
import { bugReports, bugSubmissions, organizations } from "../db/schema";

const SEVERITY_SCORES: Record<string, number> = {
  critical: 1.0,
  high: 0.75,
  medium: 0.5,
  low: 0.25,
};

const CATEGORY_SCORES: Record<string, number> = {
  crash: 1.0,
  security: 0.9,
  functional: 0.7,
  performance: 0.5,
  ui: 0.3,
  other: 0.2,
};

export async function computePriorityScore(
  db: DrizzleDB,
  bugReportId: string,
): Promise<number> {
  const bugReport = await db.query.bugReports.findFirst({
    where: eq(bugReports.id, bugReportId),
    with: { organization: true },
  });
  if (!bugReport) return 0;

  const config = bugReport.organization.scoringConfig || {
    frequencyWeight: 0.35,
    severityWeight: 0.30,
    recencyWeight: 0.20,
    userImpactWeight: 0.15,
    recencyHalfLifeDays: 14,
  };

  // 1. Frequency score: normalized report count
  // log scale to prevent very popular bugs from dominating
  const frequencyScore = Math.min(
    Math.log10(bugReport.reportCount + 1) / Math.log10(100),
    1.0,
  );

  // 2. Severity score: from AI classification + category bonus
  const severityBase = SEVERITY_SCORES[bugReport.priority] || 0.5;
  const categoryBonus = CATEGORY_SCORES[bugReport.category] || 0.2;
  const severityScore = Math.min(severityBase * 0.7 + categoryBonus * 0.3, 1.0);

  // 3. Recency score: exponential decay based on most recent submission
  const submissions = await db.query.bugSubmissions.findMany({
    where: eq(bugSubmissions.bugReportId, bugReportId),
    orderBy: [sql`submitted_at DESC`],
    limit: 1,
  });
  const lastSubmittedAt = submissions[0]?.submittedAt || bugReport.createdAt;
  const daysSinceLastReport =
    (Date.now() - lastSubmittedAt.getTime()) / (1000 * 60 * 60 * 24);
  const recencyScore = Math.exp(
    (-Math.LN2 * daysSinceLastReport) / config.recencyHalfLifeDays,
  );

  // 4. User impact score: affected users + their context
  // Higher score if enterprise/paid users are affected
  const recentSubmissions = await db.query.bugSubmissions.findMany({
    where: and(
      eq(bugSubmissions.bugReportId, bugReportId),
      gte(bugSubmissions.submittedAt, subDays(new Date(), 30)),
    ),
  });

  let userImpactScore = Math.min(bugReport.affectedUserCount / 50, 1.0);
  // Boost if high-value users are affected
  const hasEnterpriseUsers = recentSubmissions.some(
    (s) => s.userContext?.plan === "enterprise" || s.userContext?.plan === "team",
  );
  if (hasEnterpriseUsers) {
    userImpactScore = Math.min(userImpactScore * 1.5, 1.0);
  }

  // Weighted composite score (0-100 scale)
  const compositeScore =
    (frequencyScore * config.frequencyWeight +
      severityScore * config.severityWeight +
      recencyScore * config.recencyWeight +
      userImpactScore * config.userImpactWeight) *
    100;

  // Derive priority label from score
  let priorityLabel: string;
  if (compositeScore >= 75) priorityLabel = "critical";
  else if (compositeScore >= 50) priorityLabel = "high";
  else if (compositeScore >= 25) priorityLabel = "medium";
  else priorityLabel = "low";

  // Update bug report
  await db
    .update(bugReports)
    .set({
      priorityScore: Math.round(compositeScore * 100) / 100,
      priority: priorityLabel,
      updatedAt: new Date(),
    })
    .where(eq(bugReports.id, bugReportId));

  return compositeScore;
}
```

### 3. In-App Widget (Preact + Shadow DOM + html2canvas)

The widget is a lightweight Preact application that captures screenshot, browser info, and console logs, then submits to the BugBounce API. html2canvas is lazy-loaded only when the user requests a screenshot.

```typescript
// widget/src/index.ts

import { h, render } from "preact";
import { BugReportWidget } from "./BugReportWidget";
import type { WidgetConfig } from "./types";

const DEFAULT_CONFIG: WidgetConfig = {
  orgSlug: "",
  apiBaseUrl: "https://app.bugbounce.io/api/public",
  position: "bottom-right",
  triggerType: "button",
  keyboardShortcut: "ctrl+shift+b",
  screenshotEnabled: true,
  consoleCaptureEnabled: true,
  theme: {},
  userContext: {},
};

let consoleBuffer: Array<{
  level: string;
  message: string;
  timestamp: string;
  stack?: string;
}> = [];

// Intercept console errors/warnings before widget loads
function startConsoleCapture() {
  const originalError = console.error;
  const originalWarn = console.warn;

  console.error = (...args: unknown[]) => {
    consoleBuffer.push({
      level: "error",
      message: args.map(String).join(" "),
      timestamp: new Date().toISOString(),
      stack: new Error().stack,
    });
    // Keep only last 50 entries
    if (consoleBuffer.length > 50) consoleBuffer = consoleBuffer.slice(-50);
    originalError.apply(console, args);
  };

  console.warn = (...args: unknown[]) => {
    consoleBuffer.push({
      level: "warn",
      message: args.map(String).join(" "),
      timestamp: new Date().toISOString(),
    });
    if (consoleBuffer.length > 50) consoleBuffer = consoleBuffer.slice(-50);
    originalWarn.apply(console, args);
  };

  // Capture unhandled errors
  window.addEventListener("error", (event) => {
    consoleBuffer.push({
      level: "error",
      message: event.message,
      timestamp: new Date().toISOString(),
      stack: event.error?.stack,
    });
  });

  // Capture unhandled promise rejections
  window.addEventListener("unhandledrejection", (event) => {
    consoleBuffer.push({
      level: "error",
      message: `Unhandled rejection: ${event.reason}`,
      timestamp: new Date().toISOString(),
      stack: event.reason?.stack,
    });
  });
}

function mount(userConfig: Partial<WidgetConfig>) {
  const config = { ...DEFAULT_CONFIG, ...userConfig };
  if (!config.orgSlug) {
    console.error("[BugBounce] orgSlug is required");
    return;
  }

  if (config.consoleCaptureEnabled) {
    startConsoleCapture();
  }

  // Create Shadow DOM host
  const host = document.createElement("div");
  host.id = "bugbounce-widget";
  document.body.appendChild(host);

  const shadow = host.attachShadow({ mode: "open" });

  // Inject widget styles
  const style = document.createElement("style");
  style.textContent = getWidgetCSS(config);
  shadow.appendChild(style);

  const mountPoint = document.createElement("div");
  mountPoint.className = "bb-root";
  shadow.appendChild(mountPoint);

  render(
    h(BugReportWidget, {
      config,
      getConsoleLogs: () => [...consoleBuffer],
      getBrowserInfo: () => ({
        browser: detectBrowser(),
        os: detectOS(),
        screenWidth: window.innerWidth,
        screenHeight: window.innerHeight,
        url: window.location.href,
        userAgent: navigator.userAgent,
        locale: navigator.language,
      }),
    }),
    mountPoint,
  );

  // Keyboard shortcut
  if (config.triggerType === "keyboard" || config.keyboardShortcut) {
    document.addEventListener("keydown", (e) => {
      const shortcut = config.keyboardShortcut || "ctrl+shift+b";
      const parts = shortcut.toLowerCase().split("+");
      const needCtrl = parts.includes("ctrl");
      const needShift = parts.includes("shift");
      const key = parts[parts.length - 1];

      if (
        e.ctrlKey === needCtrl &&
        e.shiftKey === needShift &&
        e.key.toLowerCase() === key
      ) {
        e.preventDefault();
        // Dispatch custom event to widget
        shadow.dispatchEvent(new CustomEvent("bugbounce:open"));
      }
    });
  }
}

// Lazy-load html2canvas for screenshot capture
export async function captureScreenshot(): Promise<string | null> {
  try {
    const { default: html2canvas } = await import(
      /* webpackChunkName: "html2canvas" */ "html2canvas"
    );
    const canvas = await html2canvas(document.body, {
      logging: false,
      useCORS: true,
      allowTaint: true,
      scale: 1, // 1x for smaller file size
      ignoreElements: (el) => el.id === "bugbounce-widget",
    });
    return canvas.toDataURL("image/png", 0.8);
  } catch (err) {
    console.warn("[BugBounce] Screenshot capture failed:", err);
    return null;
  }
}

function detectBrowser(): string {
  const ua = navigator.userAgent;
  if (ua.includes("Firefox")) return "Firefox";
  if (ua.includes("Edg")) return "Edge";
  if (ua.includes("Chrome")) return "Chrome";
  if (ua.includes("Safari")) return "Safari";
  return "Unknown";
}

function detectOS(): string {
  const platform = navigator.platform;
  if (platform.includes("Win")) return "Windows";
  if (platform.includes("Mac")) return "macOS";
  if (platform.includes("Linux")) return "Linux";
  if (/iPhone|iPad/.test(navigator.userAgent)) return "iOS";
  if (/Android/.test(navigator.userAgent)) return "Android";
  return "Unknown";
}

function getWidgetCSS(config: WidgetConfig): string {
  const theme = config.theme || {};
  const pos = config.position || "bottom-right";
  const [vertical, horizontal] = pos.split("-");
  return `
    :host { all: initial; }
    .bb-root {
      font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', sans-serif;
      position: fixed;
      ${vertical}: 20px;
      ${horizontal}: 20px;
      z-index: 2147483647;
      --bb-primary: ${theme.primaryColor || "#6366f1"};
      --bb-bg: ${theme.backgroundColor || "#ffffff"};
      --bb-text: ${theme.textColor || "#1f2937"};
      --bb-radius: ${theme.borderRadius || 12}px;
    }
    /* ... widget CSS ... */
  `;
}

// Auto-init from script tag
function autoInit() {
  const scripts = document.querySelectorAll("script[data-bugbounce]");
  scripts.forEach((script) => {
    const el = script as HTMLScriptElement;
    mount({
      orgSlug: el.dataset.org || "",
      position: el.dataset.position as any,
      theme: el.dataset.theme ? JSON.parse(el.dataset.theme) : undefined,
    });
  });
}

(window as any).BugBounce = { mount, captureScreenshot };

if (document.readyState === "loading") {
  document.addEventListener("DOMContentLoaded", autoInit);
} else {
  autoInit();
}
```

### 4. Jira Integration (OAuth 2.0 + Issue Creation)

The Jira integration uses OAuth 2.0 (3LO) for authorization and creates issues with mapped fields from BugBounce bug reports. Screenshots are attached as files.

```typescript
// src/server/services/jira.ts

import { eq } from "drizzle-orm";
import { encrypt, decrypt } from "../lib/encryption";
import { integrationConfigs, bugReports, bugSubmissions } from "../db/schema";

interface JiraCredentials {
  accessToken: string;
  refreshToken: string;
  cloudId: string;
  expiresAt: number;
}

const JIRA_AUTH_URL = "https://auth.atlassian.com/authorize";
const JIRA_TOKEN_URL = "https://auth.atlassian.com/oauth/token";
const JIRA_API_BASE = "https://api.atlassian.com/ex/jira";

// Step 1: Generate OAuth authorization URL
export function getJiraAuthUrl(state: string): string {
  const params = new URLSearchParams({
    audience: "api.atlassian.com",
    client_id: process.env.JIRA_CLIENT_ID!,
    scope: "read:jira-work write:jira-work read:jira-user offline_access",
    redirect_uri: `${process.env.NEXT_PUBLIC_APP_URL}/api/auth/jira/callback`,
    state,
    response_type: "code",
    prompt: "consent",
  });
  return `${JIRA_AUTH_URL}?${params}`;
}

// Step 2: Exchange code for tokens
export async function handleJiraCallback(
  db: DrizzleDB,
  orgId: string,
  code: string,
): Promise<void> {
  const tokenRes = await fetch(JIRA_TOKEN_URL, {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({
      grant_type: "authorization_code",
      client_id: process.env.JIRA_CLIENT_ID,
      client_secret: process.env.JIRA_CLIENT_SECRET,
      code,
      redirect_uri: `${process.env.NEXT_PUBLIC_APP_URL}/api/auth/jira/callback`,
    }),
  });
  const tokens = await tokenRes.json();

  // Get accessible resources (cloud ID)
  const resourcesRes = await fetch(
    "https://api.atlassian.com/oauth/token/accessible-resources",
    { headers: { Authorization: `Bearer ${tokens.access_token}` } },
  );
  const resources = await resourcesRes.json();
  const cloudId = resources[0]?.id;
  if (!cloudId) throw new Error("No Jira cloud instance found");

  const credentials: JiraCredentials = {
    accessToken: tokens.access_token,
    refreshToken: tokens.refresh_token,
    cloudId,
    expiresAt: Date.now() + tokens.expires_in * 1000,
  };

  await db
    .insert(integrationConfigs)
    .values({
      orgId,
      type: "jira",
      credentials: encrypt(JSON.stringify(credentials)),
      isActive: true,
    })
    .onConflictDoUpdate({
      target: [integrationConfigs.orgId, integrationConfigs.type],
      set: {
        credentials: encrypt(JSON.stringify(credentials)),
        isActive: true,
      },
    });
}

// Step 3: Create Jira issue from BugBounce report
export async function createJiraIssue(
  db: DrizzleDB,
  orgId: string,
  bugReportId: string,
): Promise<{ issueKey: string; issueUrl: string }> {
  const integration = await db.query.integrationConfigs.findFirst({
    where: and(
      eq(integrationConfigs.orgId, orgId),
      eq(integrationConfigs.type, "jira"),
      eq(integrationConfigs.isActive, true),
    ),
  });
  if (!integration) throw new Error("Jira integration not configured");

  const creds = JSON.parse(decrypt(integration.credentials)) as JiraCredentials;

  // Refresh token if expired
  if (Date.now() >= creds.expiresAt) {
    await refreshJiraToken(db, integration.id, creds);
  }

  const bugReport = await db.query.bugReports.findFirst({
    where: eq(bugReports.id, bugReportId),
    with: { submissions: { limit: 1, orderBy: [sql`submitted_at ASC`] } },
  });
  if (!bugReport) throw new Error("Bug report not found");

  const firstSubmission = bugReport.submissions[0];
  const config = integration.config || {};

  // Build Jira issue body
  const description = buildJiraDescription(bugReport, firstSubmission);

  const issueRes = await fetch(
    `${JIRA_API_BASE}/${creds.cloudId}/rest/api/3/issue`,
    {
      method: "POST",
      headers: {
        Authorization: `Bearer ${creds.accessToken}`,
        "Content-Type": "application/json",
      },
      body: JSON.stringify({
        fields: {
          project: { key: config.projectKey || "BUG" },
          summary: bugReport.title,
          description: {
            type: "doc",
            version: 1,
            content: [
              {
                type: "paragraph",
                content: [{ type: "text", text: description }],
              },
            ],
          },
          issuetype: { name: config.issueTypeName || "Bug" },
          priority: {
            name: mapPriority(bugReport.priority),
          },
          labels: [
            `bugbounce`,
            `category:${bugReport.category}`,
            ...(config.defaultLabels || []),
          ],
        },
      }),
    },
  );

  const issue = await issueRes.json();
  const issueKey = issue.key;
  const issueUrl = `https://${creds.cloudId}.atlassian.net/browse/${issueKey}`;

  // Update bug report with external issue reference
  await db
    .update(bugReports)
    .set({
      externalIssueId: issueKey,
      externalIssueUrl: issueUrl,
      updatedAt: new Date(),
    })
    .where(eq(bugReports.id, bugReportId));

  return { issueKey, issueUrl };
}

function mapPriority(bbPriority: string): string {
  const map: Record<string, string> = {
    critical: "Highest",
    high: "High",
    medium: "Medium",
    low: "Low",
  };
  return map[bbPriority] || "Medium";
}

function buildJiraDescription(
  bugReport: any,
  submission: any,
): string {
  const parts: string[] = [
    `**Bug Report from BugBounce**`,
    ``,
    bugReport.description,
    ``,
    `---`,
    `**Report Count:** ${bugReport.reportCount}`,
    `**Category:** ${bugReport.category}`,
    `**Priority Score:** ${bugReport.priorityScore}`,
  ];

  if (submission?.browserInfo) {
    const bi = submission.browserInfo;
    parts.push(
      ``,
      `**Environment:**`,
      `- Browser: ${bi.browser || "Unknown"} ${bi.browserVersion || ""}`,
      `- OS: ${bi.os || "Unknown"}`,
      `- Screen: ${bi.screenWidth}x${bi.screenHeight}`,
      `- URL: ${bi.url || "N/A"}`,
    );
  }

  if (submission?.consoleLogs?.length > 0) {
    const errors = submission.consoleLogs
      .filter((l: any) => l.level === "error")
      .slice(0, 5);
    if (errors.length > 0) {
      parts.push(``, `**Console Errors:**`);
      errors.forEach((e: any) => parts.push(`- ${e.message}`));
    }
  }

  return parts.join("\n");
}

async function refreshJiraToken(
  db: DrizzleDB,
  integrationId: string,
  creds: JiraCredentials,
): Promise<void> {
  const res = await fetch(JIRA_TOKEN_URL, {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({
      grant_type: "refresh_token",
      client_id: process.env.JIRA_CLIENT_ID,
      client_secret: process.env.JIRA_CLIENT_SECRET,
      refresh_token: creds.refreshToken,
    }),
  });
  const tokens = await res.json();
  creds.accessToken = tokens.access_token;
  creds.expiresAt = Date.now() + tokens.expires_in * 1000;
  if (tokens.refresh_token) creds.refreshToken = tokens.refresh_token;

  await db
    .update(integrationConfigs)
    .set({ credentials: encrypt(JSON.stringify(creds)) })
    .where(eq(integrationConfigs.id, integrationId));
}
```

---

## Phase Breakdown

### Phase 1 — Project Scaffold + Auth + DB (Days 1–4)

**Day 1: Repository and framework setup**
```bash
npx create-next-app@latest bugbounce --ts --tailwind --app --src-dir
cd bugbounce
npm i drizzle-orm @neondatabase/serverless
npm i -D drizzle-kit
npm i @trpc/server @trpc/client @trpc/next @trpc/react-query superjson zod
npm i @clerk/nextjs
```
- `src/app/layout.tsx` — Clerk provider, tRPC provider, global layout
- `src/app/(dashboard)/layout.tsx` — authenticated shell with sidebar
- `src/server/db/index.ts` — Neon connection pool with pgvector extension
- `src/server/db/schema.ts` — organizations table
- `drizzle.config.ts` — Neon connection string
- `.env.local` — `DATABASE_URL`, `CLERK_*`, `NEXT_PUBLIC_CLERK_*`

**Day 2: Full database schema**
- `src/server/db/schema.ts` — all 12 tables: organizations, bugReports, bugSubmissions, bugComments, bugStatusChanges, dedupSuggestions, widgetConfigs, integrationConfigs, inboundEmails, reporterNotifications
- All `relations()` blocks
- HNSW index on `bug_reports.embedding`
- Run `npx drizzle-kit push` to apply schema

**Day 3: tRPC setup and org initialization**
- `src/server/trpc.ts` — base context, auth middleware, org-scoped procedure
- `src/server/routers/_app.ts` — root router
- `src/server/routers/org.ts` — getOrCreate org, update settings, update scoring config
- `src/app/api/trpc/[trpc]/route.ts` — tRPC handler

**Day 4: Auth flows and org seed**
- `src/app/(auth)/sign-in/[[...sign-in]]/page.tsx` — Clerk sign-in
- `src/app/(auth)/sign-up/[[...sign-up]]/page.tsx` — Clerk sign-up
- `src/server/lib/org-seed.ts` — on first login: create org, create default widget config, generate inbound email address (`bugs@{slug}.bugbounce.io`)
- Verify: sign up → org + widget config created

### Phase 2 — Bug Report CRUD + Dashboard (Days 5–9)

**Day 5: Bug report core CRUD**
- `src/server/routers/bug-report.ts` — list (with pagination, filters: status, priority, category, assignee, date range), get by ID (with submissions, comments, status changes), update (status, priority, assignee, category), delete
- `src/server/routers/bug-submission.ts` — list submissions for a bug, get submission detail

**Day 6: Bug inbox UI**
- `src/app/(dashboard)/bugs/page.tsx` — bug inbox list with status tabs (all, new, triaged, in_progress, resolved)
- `src/components/bugs/BugInbox.tsx` — sortable table: title, priority badge, report count, status, category, assignee, created date
- `src/components/bugs/BugFilters.tsx` — filter bar: status, priority, category, assignee, date range
- Bulk actions toolbar: assign, change priority, change status

**Day 7: Bug detail view**
- `src/app/(dashboard)/bugs/[id]/page.tsx` — full bug detail with tabs
- `src/components/bugs/BugDetail.tsx` — header (title, priority, status, report count, assignee)
- `src/components/bugs/BugSubmissions.tsx` — submissions tab: expandable cards with browser info, console logs, screenshots
- `src/components/bugs/BugActivity.tsx` — activity tab: comments + status changes interleaved chronologically

**Day 8: Bug actions and comments**
- `src/server/routers/bug-comment.ts` — add comment (internal/external), list comments
- `src/components/bugs/CommentInput.tsx` — comment form with internal/external toggle
- Status change actions with note: triage, start work, resolve, close, won't fix
- `src/server/lib/status-machine.ts` — valid status transitions
- `src/components/bugs/StatusChangeDialog.tsx` — status change modal with note

**Day 9: Manual bug submission and assignment**
- `src/app/(dashboard)/bugs/new/page.tsx` — manual bug creation form (title, description, severity, category)
- `src/components/bugs/AssigneeSelector.tsx` — team member dropdown for assignment
- `src/server/routers/bug-report.ts` — assign mutation (updates assignee, creates status change record)
- Priority badge colors and category icons throughout dashboard

### Phase 3 — Widget + Public Submission API (Days 10–16)

**Day 10: Public submission API**
- `src/app/api/public/submit/route.ts` — POST endpoint for widget submissions
- Input: orgSlug, description, browserInfo, consoleLogs, screenshotDataUrls, customFields, userContext
- Flow: validate → upload screenshots to R2 → create BugReport + BugSubmission → enqueue for AI processing
- Rate limiting: 10 submissions per IP per hour
- CORS headers for cross-origin widget requests

**Day 11: R2 screenshot upload**
- `src/server/services/r2.ts` — Cloudflare R2 client (S3-compatible)
- `src/app/api/public/upload/route.ts` — presigned URL generation for direct uploads
- Accept base64 data URLs from widget, convert and upload
- Image size limit: 5MB per screenshot
- Signed URLs for screenshot retrieval (1 hour expiry)

**Day 12: Widget project setup**
- `widget/` directory at repo root
- `widget/package.json` — preact, preact-hooks, esbuild
- `widget/tsconfig.json` — preact JSX config
- `widget/esbuild.config.ts` — single-file bundle, minify, external html2canvas
- `widget/src/index.ts` — auto-init, Shadow DOM, console interceptor

**Day 13: Widget — report form**
- `widget/src/BugReportWidget.tsx` — main component: floating trigger button, expandable form panel
- `widget/src/components/ReportForm.tsx` — description textarea, severity selector, email input
- `widget/src/components/ScreenshotCapture.tsx` — "Take Screenshot" button, lazy-loads html2canvas, shows preview with annotation overlay
- `widget/src/api.ts` — submit report to public API

**Day 14: Widget — context capture**
- `widget/src/capture/console.ts` — console.error/warn interceptor with circular buffer (50 entries)
- `widget/src/capture/network.ts` — XMLHttpRequest/fetch interceptor for failed requests
- `widget/src/capture/browser.ts` — browser detection, OS, screen size, URL, locale
- Auto-attach captured context to submission

**Day 15: Widget — screenshot annotation**
- `widget/src/components/AnnotationCanvas.tsx` — canvas overlay on screenshot for drawing (freehand, rectangle, arrow) and highlighting
- Color picker: red (default), yellow, blue
- Undo/redo support
- Export annotated screenshot as data URL

**Day 16: Widget — build and CDN**
- `widget/esbuild.config.ts` — production build: tree-shake, minify
- Output: `widget/dist/bugbounce.js` — core <15KB gzipped (html2canvas loaded on demand ~40KB)
- Version tagging: `bugbounce.1.0.0.js`
- `scripts/deploy-widget.ts` — upload to Cloudflare R2 with cache headers
- Widget config UI in dashboard: `src/app/(dashboard)/settings/widget/page.tsx`
- Embed code generator: `<script src="https://cdn.bugbounce.io/v1/bugbounce.js" data-bugbounce data-org="your-slug"></script>`

### Phase 4 — AI Pipeline: Deduplication + Priority (Days 17–22)

**Day 17: BullMQ worker setup**
```bash
npm i bullmq openai
```
- `src/server/workers/bug-processor.ts` — BullMQ worker with 5 concurrent jobs, rate limiter (20/min for OpenAI)
- `src/server/lib/queue.ts` — queue initialization, connection config
- Worker processes: categorization → embedding → dedup → priority scoring
- Error handling: exponential backoff, dead letter queue after 3 retries

**Day 18: AI categorization**
- `src/server/workers/bug-processor.ts` — GPT-4o-mini categorization step
- Prompt: classify category (crash/functional/ui/performance/security/other), severity, generate concise title
- JSON mode with temperature 0.1 for deterministic classification
- Update BugReport with AI-generated title and category

**Day 19: Embedding generation + dedup matching**
- `src/server/workers/bug-processor.ts` — text-embedding-3-small embedding step
- Embed concatenation of: title + description + top 3 error messages
- pgvector cosine similarity query against existing BugReports
- `dedupConfig.suggestThreshold` (0.82): create DedupSuggestion for human review
- `dedupConfig.autoMergeThreshold` (0.95): auto-merge submission into existing report

**Day 20: Dedup review UI**
- `src/app/(dashboard)/bugs/dedup/page.tsx` — pending dedup suggestions queue
- `src/components/bugs/DedupSuggestion.tsx` — side-by-side comparison: new submission vs matched bug
- Similarity percentage badge, one-click merge or "Not a Duplicate"
- On accept: move submission to matched BugReport, increment count, recompute priority
- On reject: mark suggestion as rejected, keep submission on original BugReport

**Day 21: Priority scoring**
- `src/server/services/priority.ts` — composite scoring algorithm
- Frequency (log-scaled report count) × weight
- Severity (AI-classified + category bonus) × weight
- Recency (exponential decay with configurable half-life) × weight
- User impact (affected user count + enterprise user boost) × weight
- Score → priority label mapping: ≥75 critical, ≥50 high, ≥25 medium, <25 low

**Day 22: Priority tuning UI**
- `src/app/(dashboard)/settings/scoring/page.tsx` — scoring weight sliders
- `src/components/settings/ScoringConfig.tsx` — 4 weight sliders (must sum to 1.0), recency half-life slider, dedup thresholds
- Live preview: show current bugs re-ranked with adjusted weights
- "Apply" saves to `organizations.scoring_config` and recomputes all open bug priorities

### Phase 5 — Email Intake + Jira Integration (Days 23–28)

**Day 23: Email intake — inbound processing**
- `src/app/api/webhooks/inbound-email/route.ts` — receive forwarded emails (Resend inbound or Cloudflare Email Workers)
- Parse email: extract subject as title, body as description, attachments as files
- Match to org by inbound address (`bugs@{slug}.bugbounce.io`)
- Store in `inbound_emails` table, create BugReport + BugSubmission, enqueue for AI

**Day 24: Email intake — parsing and attachments**
- `src/server/services/email-parser.ts` — parse email body (HTML → plain text), extract inline images
- Upload attachments to R2
- Handle reply chains: extract only new content (strip quoted replies)
- Sender email → `reporter_email` on submission

**Day 25: Jira OAuth flow**
- `src/server/services/jira.ts` — OAuth 2.0 (3LO) URL generation, callback handler, token storage
- `src/app/api/auth/jira/route.ts` — redirect to Atlassian OAuth
- `src/app/api/auth/jira/callback/route.ts` — exchange code, save to integration_configs
- `src/server/lib/encryption.ts` — AES-256-GCM encrypt/decrypt for credentials

**Day 26: Jira issue creation**
- `src/server/services/jira.ts` — `createJiraIssue` function
- Map BugBounce fields to Jira fields: title → summary, description → description (ADF format), priority → priority, category → label
- Attach screenshots as Jira attachments
- Include browser info, console errors, and report count in Jira description
- Update BugReport with `external_issue_id` and `external_issue_url`

**Day 27: Jira configuration UI**
- `src/app/(dashboard)/integrations/jira/page.tsx` — Jira connection status, project selection, issue type selection
- `src/components/integrations/JiraConfig.tsx` — project key dropdown (fetched from Jira API), issue type selector, default labels, field mapping
- "Send to Jira" button on bug detail page
- Bulk "Send selected to Jira" from inbox

**Day 28: Jira field mapping and sync**
- `src/components/integrations/FieldMapper.tsx` — drag-and-drop field mapping: BugBounce fields → Jira custom fields
- Auto-sync toggle: automatically create Jira issue on new bugs above priority threshold
- Link display: on bug detail page, show Jira issue key as clickable link
- Token refresh: handle expired tokens with automatic refresh

### Phase 6 — Billing + Plan Enforcement (Days 29–32)

**Day 29: Stripe integration**
- `src/server/services/stripe.ts` — Stripe client, product/price IDs
- `src/server/routers/billing.ts` — createCheckoutSession, createPortalSession, getCurrentPlan
- `src/app/api/webhooks/stripe/route.ts` — handle subscription events
- Plan mapping: free (50 reports/mo, no AI), pro ($29/mo: 500 reports, AI dedup, console capture, Jira), team ($79/mo: unlimited, all integrations, analytics, 10 members)

**Day 30: Plan limit enforcement**
- `src/server/lib/plan-limits.ts` — check limits:
  - Free: 50 reports/month, basic widget (screenshot only), no AI features, no integrations, powered-by badge
  - Pro: 500 reports/month, AI dedup + categorization, console/browser capture, Jira integration, no badge
  - Team: unlimited, all integrations, analytics, public bug tracker, 10 team members
- `src/server/middleware/plan-check.ts` — tRPC middleware for limit validation
- Widget returns 402 when submission limit reached

**Day 31: Billing UI**
- `src/app/(dashboard)/settings/billing/page.tsx` — current plan, usage meter, upgrade buttons
- `src/components/billing/PlanComparison.tsx` — feature comparison table
- `src/components/billing/UsageMeter.tsx` — reports used / limit this month

**Day 32: Gated features**
- Disable AI features on free plan (no dedup suggestions, no auto-categorization)
- Disable Jira integration on free plan
- Powered-by badge enforcement on free plan widget
- "Upgrade to Pro" callouts on restricted features
- Test plan transitions

### Phase 7 — Reporter Notifications + Polish (Days 33–36)

**Day 33: Reporter notification system**
- `src/server/services/reporter-notify.ts` — send email to reporters on status changes
- Template: "Your bug report '{title}' has been {acknowledged/fixed}"
- Track sent notifications to avoid duplicates
- Opt-out link in notification emails
- Trigger on status change: new→triaged (acknowledged), triaged→in_progress, in_progress→resolved (fixed)

**Day 34: Dashboard polish**
- Needs Triage section at top of inbox (new bugs, unreviewed dedup suggestions)
- Quick stats bar: open bugs, critical bugs, avg time to resolve, reports this week
- Keyboard shortcuts: j/k to navigate bugs, s to change status, a to assign
- Sort options: priority score, report count, newest, oldest

**Day 35: Widget configuration dashboard**
- `src/app/(dashboard)/settings/widget/page.tsx` — full widget config: theme, position, trigger, fields, capture settings
- `src/components/settings/WidgetPreview.tsx` — live preview of widget with current settings
- Custom fields builder: add text/select/checkbox fields
- Embed code with all configuration baked in

**Day 36: Error handling and edge cases**
- Widget offline mode: queue submissions when API is unreachable, retry on reconnect
- Screenshot capture failure: graceful fallback (submit without screenshot)
- Email parsing edge cases: multi-part MIME, base64-encoded bodies, non-English content
- AI processing retries: handle OpenAI rate limits and timeouts
- Race condition in dedup: handle concurrent submissions for same bug

### Phase 8 — Launch Prep (Days 37–39)

**Day 37: Monitoring and logging**
- Sentry integration for error tracking (dashboard + widget)
- Structured logging for: submission intake, AI processing, dedup matches, Jira sync
- Alerting: AI processing queue depth, failed submissions, Jira sync failures
- Widget error boundary: capture and report errors without crashing host page

**Day 38: Performance optimization**
- Widget bundle verification: core <15KB gzipped
- Public submission API: response <300ms (immediate return, async processing)
- Bug inbox: pagination (50 per page), indexed queries
- Dedup query: HNSW index performance with 10K+ bug reports
- Dashboard page load: <1.5s with skeleton loading

**Day 39: End-to-end testing and launch**
- Full flow: embed widget → submit bug → AI categorizes + dedup checks → appears in dashboard → send to Jira → status update → reporter notified
- Email intake flow: forward email → parsed → appears as bug
- Cross-browser widget testing: Chrome, Firefox, Safari, Edge
- Mobile widget testing: iOS Safari, Chrome Android
- Screenshot capture testing across different page types
- Deploy to production, verify Stripe webhooks

---

## Critical Files

```
bugbounce/
├── src/
│   ├── app/
│   │   ├── layout.tsx                            # Root layout: Clerk + tRPC providers
│   │   ├── (auth)/
│   │   │   ├── sign-in/[[...sign-in]]/page.tsx
│   │   │   └── sign-up/[[...sign-up]]/page.tsx
│   │   ├── (dashboard)/
│   │   │   ├── layout.tsx                        # Dashboard shell with sidebar
│   │   │   ├── bugs/
│   │   │   │   ├── page.tsx                      # Bug inbox with filters + tabs
│   │   │   │   ├── [id]/page.tsx                 # Bug detail with tabs
│   │   │   │   ├── new/page.tsx                  # Manual bug creation
│   │   │   │   └── dedup/page.tsx                # Dedup suggestion review queue
│   │   │   ├── integrations/
│   │   │   │   ├── page.tsx                      # Integration list
│   │   │   │   └── jira/page.tsx                 # Jira config
│   │   │   └── settings/
│   │   │       ├── page.tsx                      # Org settings
│   │   │       ├── widget/page.tsx               # Widget customizer
│   │   │       ├── scoring/page.tsx              # Priority scoring config
│   │   │       └── billing/page.tsx              # Plan management
│   │   └── api/
│   │       ├── trpc/[trpc]/route.ts              # tRPC handler
│   │       ├── auth/
│   │       │   └── jira/
│   │       │       ├── route.ts                  # Jira OAuth redirect
│   │       │       └── callback/route.ts         # Jira OAuth callback
│   │       ├── public/
│   │       │   ├── submit/route.ts               # Widget bug submission
│   │       │   └── upload/route.ts               # Screenshot upload (presigned URL)
│   │       └── webhooks/
│   │           ├── inbound-email/route.ts        # Email intake webhook
│   │           └── stripe/route.ts               # Stripe billing webhooks
│   ├── server/
│   │   ├── db/
│   │   │   ├── index.ts                          # Neon + pgvector connection
│   │   │   └── schema.ts                         # 12 tables + relations
│   │   ├── trpc.ts                               # tRPC base config + middleware
│   │   ├── routers/
│   │   │   ├── _app.ts                           # Root router
│   │   │   ├── org.ts                            # Organization settings
│   │   │   ├── bug-report.ts                     # Bug report CRUD + filters
│   │   │   ├── bug-submission.ts                 # Submission queries
│   │   │   ├── bug-comment.ts                    # Comments CRUD
│   │   │   ├── dedup.ts                          # Dedup suggestion review
│   │   │   ├── widget-config.ts                  # Widget config CRUD
│   │   │   ├── integration.ts                    # Integration management
│   │   │   └── billing.ts                        # Stripe checkout + portal
│   │   ├── services/
│   │   │   ├── priority.ts                       # Priority scoring algorithm
│   │   │   ├── jira.ts                           # Jira OAuth + issue creation
│   │   │   ├── r2.ts                             # Cloudflare R2 storage
│   │   │   ├── email-parser.ts                   # Inbound email parsing
│   │   │   ├── reporter-notify.ts                # Reporter notification emails
│   │   │   ├── email.ts                          # Resend client
│   │   │   └── stripe.ts                         # Stripe client
│   │   ├── workers/
│   │   │   └── bug-processor.ts                  # BullMQ: AI categorize + embed + dedup
│   │   ├── lib/
│   │   │   ├── encryption.ts                     # AES-256-GCM for integration tokens
│   │   │   ├── org-seed.ts                       # First-login initialization
│   │   │   ├── plan-limits.ts                    # Plan limit definitions
│   │   │   ├── status-machine.ts                 # Valid status transitions
│   │   │   └── queue.ts                          # BullMQ queue config
│   │   └── middleware/
│   │       └── plan-check.ts                     # tRPC plan enforcement
│   └── components/
│       ├── bugs/
│       │   ├── BugInbox.tsx                      # Inbox table
│       │   ├── BugFilters.tsx                    # Filter bar
│       │   ├── BugDetail.tsx                     # Detail view header
│       │   ├── BugSubmissions.tsx                # Submissions tab
│       │   ├── BugActivity.tsx                   # Activity timeline
│       │   ├── CommentInput.tsx                  # Comment form
│       │   ├── StatusChangeDialog.tsx            # Status change modal
│       │   ├── AssigneeSelector.tsx              # Member dropdown
│       │   └── DedupSuggestion.tsx               # Side-by-side dedup comparison
│       ├── integrations/
│       │   ├── JiraConfig.tsx                    # Jira project/type config
│       │   └── FieldMapper.tsx                   # Field mapping editor
│       ├── settings/
│       │   ├── ScoringConfig.tsx                 # Priority weight sliders
│       │   └── WidgetPreview.tsx                 # Live widget preview
│       └── billing/
│           ├── PlanComparison.tsx                # Feature comparison
│           └── UsageMeter.tsx                    # Usage progress bar
├── widget/
│   ├── package.json                              # Preact, html2canvas, esbuild
│   ├── tsconfig.json                             # Preact JSX config
│   ├── esbuild.config.ts                         # Build config
│   └── src/
│       ├── index.ts                              # Entry: auto-init, Shadow DOM, console capture
│       ├── BugReportWidget.tsx                   # Main widget component
│       ├── types.ts                              # WidgetConfig type
│       ├── api.ts                                # Public API client
│       └── components/
│           ├── ReportForm.tsx                    # Description + severity form
│           ├── ScreenshotCapture.tsx             # html2canvas trigger + preview
│           └── AnnotationCanvas.tsx              # Screenshot annotation overlay
│       └── capture/
│           ├── console.ts                        # Console interceptor
│           ├── network.ts                        # XHR/fetch interceptor
│           └── browser.ts                        # Browser/OS detection
├── scripts/
│   └── deploy-widget.ts                          # R2 upload script
├── drizzle.config.ts
├── vercel.json                                   # Cron jobs (if needed)
├── .env.local
└── package.json
```

---

## Verification

### Manual Testing Checklist

1. **Sign up → org + widget config created** — default widget config and inbound email address generated
2. **Manual bug creation** — create bug via dashboard form, appears in inbox with "new" status
3. **Widget embed** — paste embed code into test page, floating button appears, style-isolated
4. **Widget screenshot** — click "Take Screenshot", html2canvas loads, preview shows, can annotate
5. **Widget submission** — fill form, submit, see success state, bug appears in dashboard within 5 seconds
6. **Console capture** — trigger `console.error()` on test page, submit bug, verify console logs attached to submission
7. **AI categorization** — submit bug, wait for processing, verify AI-generated title, category, and priority
8. **Dedup detection** — submit two similar bugs, second shows dedup suggestion in dashboard
9. **Dedup auto-merge** — submit nearly identical bug (>0.95 similarity), verify auto-merged (report count incremented)
10. **Dedup review** — review pending suggestion: accept merges submissions, reject keeps separate
11. **Priority scoring** — submit 5 reports for same bug, verify priority score increases and label changes
12. **Priority tuning** — adjust scoring weights, verify bug rankings update
13. **Email intake** — forward email to `bugs@{slug}.bugbounce.io`, verify parsed and appears as bug
14. **Jira integration** — connect Jira, send bug to Jira, verify issue created with correct fields
15. **Status workflow** — move bug through: new → triaged → in_progress → resolved → closed
16. **Plan limits** — on free plan: submit 51st bug, verify rejected with upgrade prompt
17. **Widget powered-by** — free plan shows badge, pro plan hides it

### SQL Verification Queries

```sql
-- 1. Bug report volume and dedup effectiveness
SELECT
  COUNT(*) as total_reports,
  SUM(report_count) as total_submissions,
  ROUND(1.0 - COUNT(*)::numeric / NULLIF(SUM(report_count), 0), 2) as dedup_rate
FROM bug_reports
WHERE org_id = '<org_id>';

-- 2. Priority distribution
SELECT priority, COUNT(*) as count,
  ROUND(AVG(priority_score)::numeric, 1) as avg_score
FROM bug_reports
WHERE org_id = '<org_id>' AND status NOT IN ('closed', 'wont_fix')
GROUP BY priority ORDER BY avg_score DESC;

-- 3. Submission sources breakdown
SELECT source, COUNT(*) as count
FROM bug_submissions
WHERE org_id = '<org_id>'
  AND submitted_at >= NOW() - INTERVAL '30 days'
GROUP BY source ORDER BY count DESC;

-- 4. Average time to first response (triage)
SELECT
  ROUND(AVG(EXTRACT(EPOCH FROM (first_response_at - created_at)) / 3600)::numeric, 1)
    as avg_hours_to_triage
FROM bug_reports
WHERE org_id = '<org_id>' AND first_response_at IS NOT NULL;

-- 5. Dedup suggestion accuracy
SELECT status, COUNT(*) as count
FROM dedup_suggestions
WHERE org_id = '<org_id>'
GROUP BY status;
```

### Performance Benchmarks

| Operation | Target | Method |
|-----------|--------|--------|
| Widget initial load | <150ms | Lighthouse, <15KB core bundle |
| Screenshot capture (html2canvas) | <1.5s | Average page, 1x scale |
| Bug submission API response | <300ms | Immediate return, async processing |
| AI categorization (worker) | <3s | GPT-4o-mini classification |
| Embedding generation (worker) | <1s | text-embedding-3-small |
| Dedup query (pgvector HNSW) | <50ms | 10K bug reports, top-5 results |
| Dedup check at scale (50K+ reports) | <200ms | HNSW index with ef_search=40, top-5 cosine similarity |
| Full processing pipeline | <8s | Submit → AI categorize → embed → dedup → priority |
| Bug inbox page load | <1s | 50 bugs per page with filters |
| Bug detail page load | <800ms | Bug + 20 submissions + 50 comments |
| Jira issue creation | <2s | API call including attachment upload |

---

## Post-MVP Roadmap

| ID | Feature | Description | Priority |
|----|---------|-------------|----------|
| F1 | Session Replay (rrweb) | Integrate rrweb for DOM-level session recording attached to bug submissions. Replay user actions leading up to the bug in a visual player. Lazy-loaded recording (~50KB), 30-second buffer before submission. | High |
| F2 | Bi-Directional Jira Sync | Two-way sync between BugBounce and Jira: status changes in Jira reflect in BugBounce and vice versa. Webhook-based real-time sync with conflict resolution (last-write-wins with audit trail). | High |
| F3 | SLA Tracking & Alerts | SLA tier enforcement with automated alerts. Schema fields (`slaTier`, `slaResponseDueAt`, `slaResolutionDueAt`, `slaBreached`) already added. Configurable SLA tiers per priority: standard (72h/14d), priority (24h/7d), critical (4h/48h). Dashboard SLA compliance widget. | High |
| F4 | Advanced Analytics Dashboard | Bug trend charts (volume over time, resolution time trends, category distribution), team performance metrics (avg response time per assignee), dedup effectiveness tracking, and SLA compliance rates. | Medium |
| F5 | Public Bug Tracker | Customer-facing bug tracker portal showing acknowledged bugs, their status, and resolution timeline. Voters can upvote bugs to influence priority. Configurable visibility per bug. | Medium |
| F6 | Additional Integrations | Linear (GraphQL API), GitHub Issues (REST + webhooks), and Asana (REST API) as alternative issue tracker integrations. Same field mapping UI pattern as Jira. Integration selector during setup. | Medium |
| F7 | Reporter Communication | In-app messaging between team and bug reporters. Email thread integration — reply to reporter notification to add comment. Reporter portal for checking bug status without login. | Low |
| F8 | User Impact Segments | Tag submissions with user segments (enterprise, free, trial) for prioritization. Auto-enrich from CRM data (HubSpot, Salesforce). Priority boost for high-value customer segments. | Low |
| F9 | AI Root Cause Clustering | ML-based clustering of related bugs by stack trace and error pattern similarity. Auto-suggest root cause labels. "Related bugs" sidebar on bug detail page. | Low |
