# 11. DataPipe — No-Code ETL Pipeline Builder

## Implementation Plan

**MVP Scope:** Visual pipeline builder with React Flow DAG editor, 10 source connectors (Stripe, Shopify, HubSpot, Google Ads, GA4, PostgreSQL, MySQL, Google Sheets, CSV upload, REST API), 3 destination loaders (BigQuery, PostgreSQL, Google Sheets), DuckDB-powered transform engine with 5 transform types (filter, rename, computed column, aggregate, deduplicate), BullMQ-based scheduling (daily/hourly), 3-stage ETL execution on AWS ECS Fargate with S3 parquet staging, pipeline monitoring with run history and row counts, email alerts on failure via Resend, AG Grid data preview at each pipeline step, Stripe billing with four tiers (Free / Pro $49/mo / Team $149/mo / Enterprise custom).

---

## Tech Decisions

| Layer | Technology | Notes |
|-------|-----------|-------|
| Framework | Next.js 14+ App Router | `src/` directory, TypeScript strict |
| API | tRPC v11 | superjson transformer, Clerk auth context |
| Database | Neon PostgreSQL | Pipeline configs, run history, connection metadata |
| ORM | Drizzle ORM | Push-based migrations, typed schema |
| Auth | Clerk | Organization-scoped, `orgId` from session |
| Billing | Stripe | Checkout Sessions + Customer Portal + Webhooks |
| Email | Resend | Alert notifications on pipeline failures |
| Pipeline Editor | React Flow | Custom nodes for sources, transforms, destinations |
| Data Preview | AG Grid | Virtualized table for large row-set previews |
| Transform Engine | DuckDB (via `duckdb-async`) | In-process SQL engine for all transform operations |
| Queue / Scheduling | BullMQ + Redis (Upstash) | Cron-based pipeline scheduling, job orchestration |
| Workers | AWS ECS Fargate | Long-running extract/transform/load tasks |
| Staging | AWS S3 | Parquet files for intermediate data between ETL stages |
| UI | Tailwind CSS, Radix UI, Recharts | |
| Hosting | Vercel (app), AWS ECS (workers) | |
| Monitoring | Sentry, Vercel Analytics | |

### Key Architecture Decisions

1. **DuckDB over Pandas/Spark for transforms**: DuckDB provides a full SQL engine that runs in-process with zero infrastructure. Transform DAGs are compiled into a sequence of SQL statements operating on parquet files loaded from S3. This gives sub-second transforms for datasets up to 10M rows while keeping worker memory under 2GB. No cluster management, no JVM, no Python runtime needed.

2. **S3 parquet staging between ETL stages**: Each pipeline run creates a staging prefix in S3 (`s3://datapipe-staging/{orgId}/{runId}/`). The extractor writes raw data as parquet, the transformer reads and writes transformed parquet, and the loader reads the final parquet to write to the destination. This decouples the three stages, enables retrying individual stages, and provides a data audit trail.

3. **React Flow for the pipeline DAG editor**: React Flow provides a mature, performant graph editor with built-in zoom/pan, custom node rendering, edge routing, and minimap. Each transform type gets a custom node component with inline config. The DAG is serialized as a JSON graph (nodes + edges) stored in `pipelines.transformConfig`.

4. **BullMQ cron scheduling over Temporal**: For MVP scope (daily/hourly schedules, no complex dependency chains), BullMQ's built-in cron repeatable jobs are sufficient and avoid the operational complexity of running a Temporal server. Each active pipeline gets a repeatable BullMQ job that triggers the 3-stage ETL workflow. Migration to Temporal can happen post-MVP when pipeline dependencies are needed.

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
  bigint,
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
  plan: text("plan").default("free").notNull(), // free | pro | team | enterprise
  stripeCustomerId: text("stripe_customer_id"),
  stripeSubscriptionId: text("stripe_subscription_id"),
  settings: jsonb("settings").default({}).$type<{
    defaultTimezone?: string;
    alertEmail?: string;
    slackWebhookUrl?: string;
    maxConcurrentRuns?: number;
  }>(),
  usageRowsSynced: bigint("usage_rows_synced", { mode: "number" }).default(0).notNull(),
  usageResetAt: timestamp("usage_reset_at", { withTimezone: true }).defaultNow().notNull(),
  createdAt: timestamp("created_at", { withTimezone: true }).defaultNow().notNull(),
  updatedAt: timestamp("updated_at", { withTimezone: true }).defaultNow().notNull(),
});

// ---------------------------------------------------------------------------
// Members (org membership tracking for role-based access)
// ---------------------------------------------------------------------------
export const members = pgTable(
  "members",
  {
    id: uuid("id").primaryKey().defaultRandom(),
    orgId: uuid("org_id")
      .references(() => organizations.id, { onDelete: "cascade" })
      .notNull(),
    clerkUserId: text("clerk_user_id").notNull(),
    email: text("email").notNull(),
    name: text("name"),
    role: text("role").default("editor").notNull(), // admin | editor | viewer
    createdAt: timestamp("created_at", { withTimezone: true }).defaultNow().notNull(),
  },
  (t) => [
    uniqueIndex("members_org_user_idx").on(t.orgId, t.clerkUserId),
    index("members_org_idx").on(t.orgId),
  ]
);

// ---------------------------------------------------------------------------
// Connections (source and destination credentials)
// ---------------------------------------------------------------------------
export const connections = pgTable(
  "connections",
  {
    id: uuid("id").primaryKey().defaultRandom(),
    orgId: uuid("org_id")
      .references(() => organizations.id, { onDelete: "cascade" })
      .notNull(),
    name: text("name").notNull(), // "Production Stripe", "Analytics BigQuery"
    type: text("type").notNull(), // source | destination
    connectorType: text("connector_type").notNull(),
    // stripe | shopify | hubspot | google_ads | ga4 | postgres | mysql
    // google_sheets | csv_upload | rest_api | bigquery
    credentials: jsonb("credentials").default({}).$type<{
      // All values AES-256-GCM encrypted
      apiKey?: string;
      accessToken?: string;
      refreshToken?: string;
      clientId?: string;
      clientSecret?: string;
      // Database connections
      host?: string;
      port?: number;
      database?: string;
      username?: string;
      password?: string;
      sslMode?: string;
      // BigQuery
      projectId?: string;
      datasetId?: string;
      serviceAccountJson?: string;
      // Google Sheets
      spreadsheetId?: string;
      sheetName?: string;
      // REST API
      baseUrl?: string;
      headers?: Record<string, string>;
      authType?: "bearer" | "api_key" | "basic" | "none";
    }>(),
    schemaCache: jsonb("schema_cache").default(null).$type<{
      tables: Array<{
        name: string;
        columns: Array<{
          name: string;
          type: string;
          nullable: boolean;
        }>;
        rowCount?: number;
      }>;
      cachedAt: string;
    } | null>(),
    status: text("status").default("connected").notNull(), // connected | error | testing
    errorMessage: text("error_message"),
    lastTestedAt: timestamp("last_tested_at", { withTimezone: true }),
    createdAt: timestamp("created_at", { withTimezone: true }).defaultNow().notNull(),
    updatedAt: timestamp("updated_at", { withTimezone: true }).defaultNow().notNull(),
  },
  (t) => [
    index("connections_org_idx").on(t.orgId),
    index("connections_org_type_idx").on(t.orgId, t.type),
  ]
);

// ---------------------------------------------------------------------------
// Pipelines (ETL pipeline definitions)
// ---------------------------------------------------------------------------
export const pipelines = pgTable(
  "pipelines",
  {
    id: uuid("id").primaryKey().defaultRandom(),
    orgId: uuid("org_id")
      .references(() => organizations.id, { onDelete: "cascade" })
      .notNull(),
    name: text("name").notNull(),
    description: text("description"),
    sourceConnectionId: uuid("source_connection_id")
      .references(() => connections.id, { onDelete: "set null" }),
    destinationConnectionId: uuid("destination_connection_id")
      .references(() => connections.id, { onDelete: "set null" }),
    sourceConfig: jsonb("source_config").default({}).$type<{
      tables: string[];                    // Selected source tables
      filters?: Record<string, unknown>;   // Pre-extraction filters
      incrementalKey?: string;             // Column for incremental sync
      syncMode: "full" | "incremental";
    }>(),
    transformConfig: jsonb("transform_config").default({ nodes: [], edges: [] }).$type<{
      nodes: Array<{
        id: string;
        type: "source" | "filter" | "rename" | "computed" | "aggregate" | "deduplicate" | "destination";
        position: { x: number; y: number };
        data: Record<string, unknown>;
      }>;
      edges: Array<{
        id: string;
        source: string;
        target: string;
      }>;
    }>(),
    destinationConfig: jsonb("destination_config").default({}).$type<{
      targetTable: string;
      loadMode: "append" | "upsert" | "replace";
      upsertKey?: string;                 // Column for upsert merge key
      createTableIfMissing?: boolean;
    }>(),
    schedule: text("schedule"),            // Cron expression: "0 * * * *" (hourly)
    scheduleEnabled: boolean("schedule_enabled").default(false).notNull(),
    status: text("status").default("draft").notNull(), // draft | active | paused | error
    version: integer("version").default(1).notNull(),
    lastRunAt: timestamp("last_run_at", { withTimezone: true }),
    nextRunAt: timestamp("next_run_at", { withTimezone: true }),
    createdAt: timestamp("created_at", { withTimezone: true }).defaultNow().notNull(),
    updatedAt: timestamp("updated_at", { withTimezone: true }).defaultNow().notNull(),
  },
  (t) => [
    index("pipelines_org_idx").on(t.orgId),
    index("pipelines_org_status_idx").on(t.orgId, t.status),
    index("pipelines_next_run_idx").on(t.nextRunAt),
  ]
);

// ---------------------------------------------------------------------------
// Pipeline Runs (execution history)
// ---------------------------------------------------------------------------
export const pipelineRuns = pgTable(
  "pipeline_runs",
  {
    id: uuid("id").primaryKey().defaultRandom(),
    pipelineId: uuid("pipeline_id")
      .references(() => pipelines.id, { onDelete: "cascade" })
      .notNull(),
    orgId: uuid("org_id")
      .references(() => organizations.id, { onDelete: "cascade" })
      .notNull(),
    version: integer("version").notNull(),
    status: text("status").default("queued").notNull(),
    // queued | extracting | transforming | loading | success | failed | cancelled
    trigger: text("trigger").default("manual").notNull(), // manual | schedule | api
    stagingPrefix: text("staging_prefix"),   // s3://datapipe-staging/{orgId}/{runId}/
    rowsExtracted: integer("rows_extracted").default(0).notNull(),
    rowsTransformed: integer("rows_transformed").default(0).notNull(),
    rowsLoaded: integer("rows_loaded").default(0).notNull(),
    bytesProcessed: bigint("bytes_processed", { mode: "number" }).default(0).notNull(),
    extractStartedAt: timestamp("extract_started_at", { withTimezone: true }),
    extractCompletedAt: timestamp("extract_completed_at", { withTimezone: true }),
    transformStartedAt: timestamp("transform_started_at", { withTimezone: true }),
    transformCompletedAt: timestamp("transform_completed_at", { withTimezone: true }),
    loadStartedAt: timestamp("load_started_at", { withTimezone: true }),
    loadCompletedAt: timestamp("load_completed_at", { withTimezone: true }),
    errorMessage: text("error_message"),
    errorDetails: jsonb("error_details").default(null).$type<{
      stage: "extract" | "transform" | "load";
      failedAtRow?: number;
      stackTrace?: string;
      retryCount?: number;
    } | null>(),
    startedAt: timestamp("started_at", { withTimezone: true }),
    completedAt: timestamp("completed_at", { withTimezone: true }),
    durationMs: integer("duration_ms"),
    createdAt: timestamp("created_at", { withTimezone: true }).defaultNow().notNull(),
  },
  (t) => [
    index("pipeline_runs_pipeline_idx").on(t.pipelineId),
    index("pipeline_runs_org_idx").on(t.orgId),
    index("pipeline_runs_status_idx").on(t.pipelineId, t.status),
    index("pipeline_runs_created_at_idx").on(t.pipelineId, t.createdAt),
  ]
);

// ---------------------------------------------------------------------------
// Pipeline Versions (version history for rollback)
// ---------------------------------------------------------------------------
export const pipelineVersions = pgTable(
  "pipeline_versions",
  {
    id: uuid("id").primaryKey().defaultRandom(),
    pipelineId: uuid("pipeline_id")
      .references(() => pipelines.id, { onDelete: "cascade" })
      .notNull(),
    orgId: uuid("org_id")
      .references(() => organizations.id, { onDelete: "cascade" })
      .notNull(),
    version: integer("version").notNull(),
    sourceConfig: jsonb("source_config").notNull(),
    transformConfig: jsonb("transform_config").notNull(),
    destinationConfig: jsonb("destination_config").notNull(),
    changeNote: text("change_note"),
    createdByUserId: text("created_by_user_id"),
    createdAt: timestamp("created_at", { withTimezone: true }).defaultNow().notNull(),
  },
  (t) => [
    uniqueIndex("pipeline_version_unique_idx").on(t.pipelineId, t.version),
    index("pipeline_versions_pipeline_idx").on(t.pipelineId),
  ]
);

// ---------------------------------------------------------------------------
// Alert Configs (notification rules)
// ---------------------------------------------------------------------------
export const alertConfigs = pgTable(
  "alert_configs",
  {
    id: uuid("id").primaryKey().defaultRandom(),
    orgId: uuid("org_id")
      .references(() => organizations.id, { onDelete: "cascade" })
      .notNull(),
    pipelineId: uuid("pipeline_id")
      .references(() => pipelines.id, { onDelete: "cascade" }),
    // null pipelineId = global alert for all pipelines
    name: text("name").notNull(),
    type: text("type").notNull(), // failure | slow_run | zero_rows | schema_change
    channel: text("channel").default("email").notNull(), // email | slack | webhook
    config: jsonb("config").default({}).$type<{
      recipients?: string[];           // Email addresses
      slackWebhookUrl?: string;
      webhookUrl?: string;
      slowRunThresholdMs?: number;     // Alert if run exceeds this duration
      consecutiveFailures?: number;    // Alert after N consecutive failures
    }>(),
    enabled: boolean("enabled").default(true).notNull(),
    createdAt: timestamp("created_at", { withTimezone: true }).defaultNow().notNull(),
    updatedAt: timestamp("updated_at", { withTimezone: true }).defaultNow().notNull(),
  },
  (t) => [
    index("alert_configs_org_idx").on(t.orgId),
    index("alert_configs_pipeline_idx").on(t.pipelineId),
  ]
);

// ---------------------------------------------------------------------------
// Alert Log (sent alerts history)
// ---------------------------------------------------------------------------
export const alertLog = pgTable(
  "alert_log",
  {
    id: uuid("id").primaryKey().defaultRandom(),
    orgId: uuid("org_id")
      .references(() => organizations.id, { onDelete: "cascade" })
      .notNull(),
    alertConfigId: uuid("alert_config_id")
      .references(() => alertConfigs.id, { onDelete: "set null" }),
    pipelineId: uuid("pipeline_id")
      .references(() => pipelines.id, { onDelete: "set null" }),
    pipelineRunId: uuid("pipeline_run_id")
      .references(() => pipelineRuns.id, { onDelete: "set null" }),
    type: text("type").notNull(), // failure | slow_run | zero_rows | schema_change
    channel: text("channel").notNull(), // email | slack | webhook
    status: text("status").default("sent").notNull(), // sent | failed
    message: text("message").notNull(),
    metadata: jsonb("metadata").default({}).$type<{
      pipelineName?: string;
      runDurationMs?: number;
      errorMessage?: string;
      recipientEmail?: string;
      resendMessageId?: string;
    }>(),
    sentAt: timestamp("sent_at", { withTimezone: true }).defaultNow().notNull(),
  },
  (t) => [
    index("alert_log_org_idx").on(t.orgId),
    index("alert_log_pipeline_idx").on(t.pipelineId),
    index("alert_log_sent_at_idx").on(t.sentAt),
  ]
);

// ---------------------------------------------------------------------------
// Connector Registry (available connector definitions)
// ---------------------------------------------------------------------------
export const connectorRegistry = pgTable("connector_registry", {
  id: uuid("id").primaryKey().defaultRandom(),
  slug: text("slug").unique().notNull(),     // stripe | shopify | bigquery | ...
  name: text("name").notNull(),              // "Stripe", "Shopify"
  type: text("type").notNull(),              // source | destination | both
  category: text("category").notNull(),      // payment | crm | marketing | analytics | ecommerce | database | file | api
  iconUrl: text("icon_url"),
  authType: text("auth_type").notNull(),     // oauth | api_key | connection_string | file_upload | none
  configSchema: jsonb("config_schema").notNull().$type<{
    fields: Array<{
      key: string;
      label: string;
      type: "text" | "password" | "number" | "select" | "boolean";
      required: boolean;
      placeholder?: string;
      options?: Array<{ label: string; value: string }>;
    }>;
  }>(),
  supportedSyncModes: jsonb("supported_sync_modes")
    .default(["full"])
    .$type<Array<"full" | "incremental">>(),
  isActive: boolean("is_active").default(true).notNull(),
  createdAt: timestamp("created_at", { withTimezone: true }).defaultNow().notNull(),
});

// ---------------------------------------------------------------------------
// Scheduler State (BullMQ job tracking)
// ---------------------------------------------------------------------------
export const schedulerState = pgTable(
  "scheduler_state",
  {
    id: uuid("id").primaryKey().defaultRandom(),
    pipelineId: uuid("pipeline_id")
      .references(() => pipelines.id, { onDelete: "cascade" })
      .notNull(),
    orgId: uuid("org_id")
      .references(() => organizations.id, { onDelete: "cascade" })
      .notNull(),
    bullmqJobId: text("bullmq_job_id"),
    cronExpression: text("cron_expression").notNull(),
    status: text("status").default("active").notNull(), // active | paused | removed
    lastTriggeredAt: timestamp("last_triggered_at", { withTimezone: true }),
    nextTriggerAt: timestamp("next_trigger_at", { withTimezone: true }),
    createdAt: timestamp("created_at", { withTimezone: true }).defaultNow().notNull(),
    updatedAt: timestamp("updated_at", { withTimezone: true }).defaultNow().notNull(),
  },
  (t) => [
    uniqueIndex("scheduler_state_pipeline_idx").on(t.pipelineId),
    index("scheduler_state_org_idx").on(t.orgId),
    index("scheduler_state_next_trigger_idx").on(t.nextTriggerAt),
  ]
);

// ---------------------------------------------------------------------------
// Relations
// ---------------------------------------------------------------------------
export const organizationsRelations = relations(organizations, ({ many }) => ({
  members: many(members),
  connections: many(connections),
  pipelines: many(pipelines),
  pipelineRuns: many(pipelineRuns),
  alertConfigs: many(alertConfigs),
  alertLog: many(alertLog),
  schedulerState: many(schedulerState),
}));

export const membersRelations = relations(members, ({ one }) => ({
  organization: one(organizations, {
    fields: [members.orgId],
    references: [organizations.id],
  }),
}));

export const connectionsRelations = relations(connections, ({ one, many }) => ({
  organization: one(organizations, {
    fields: [connections.orgId],
    references: [organizations.id],
  }),
  sourcePipelines: many(pipelines, { relationName: "sourceConnection" }),
  destinationPipelines: many(pipelines, { relationName: "destinationConnection" }),
}));

export const pipelinesRelations = relations(pipelines, ({ one, many }) => ({
  organization: one(organizations, {
    fields: [pipelines.orgId],
    references: [organizations.id],
  }),
  sourceConnection: one(connections, {
    fields: [pipelines.sourceConnectionId],
    references: [connections.id],
    relationName: "sourceConnection",
  }),
  destinationConnection: one(connections, {
    fields: [pipelines.destinationConnectionId],
    references: [connections.id],
    relationName: "destinationConnection",
  }),
  runs: many(pipelineRuns),
  versions: many(pipelineVersions),
  alertConfigs: many(alertConfigs),
  schedulerState: many(schedulerState),
}));

export const pipelineRunsRelations = relations(pipelineRuns, ({ one }) => ({
  pipeline: one(pipelines, {
    fields: [pipelineRuns.pipelineId],
    references: [pipelines.id],
  }),
  organization: one(organizations, {
    fields: [pipelineRuns.orgId],
    references: [organizations.id],
  }),
}));

export const pipelineVersionsRelations = relations(pipelineVersions, ({ one }) => ({
  pipeline: one(pipelines, {
    fields: [pipelineVersions.pipelineId],
    references: [pipelines.id],
  }),
  organization: one(organizations, {
    fields: [pipelineVersions.orgId],
    references: [organizations.id],
  }),
}));

export const alertConfigsRelations = relations(alertConfigs, ({ one, many }) => ({
  organization: one(organizations, {
    fields: [alertConfigs.orgId],
    references: [organizations.id],
  }),
  pipeline: one(pipelines, {
    fields: [alertConfigs.pipelineId],
    references: [pipelines.id],
  }),
  logs: many(alertLog),
}));

export const alertLogRelations = relations(alertLog, ({ one }) => ({
  organization: one(organizations, {
    fields: [alertLog.orgId],
    references: [organizations.id],
  }),
  alertConfig: one(alertConfigs, {
    fields: [alertLog.alertConfigId],
    references: [alertConfigs.id],
  }),
  pipeline: one(pipelines, {
    fields: [alertLog.pipelineId],
    references: [pipelines.id],
  }),
  pipelineRun: one(pipelineRuns, {
    fields: [alertLog.pipelineRunId],
    references: [pipelineRuns.id],
  }),
}));

export const connectorRegistryRelations = relations(connectorRegistry, () => ({}));

export const schedulerStateRelations = relations(schedulerState, ({ one }) => ({
  pipeline: one(pipelines, {
    fields: [schedulerState.pipelineId],
    references: [pipelines.id],
  }),
  organization: one(organizations, {
    fields: [schedulerState.orgId],
    references: [organizations.id],
  }),
}));

// ---------------------------------------------------------------------------
// Type Exports
// ---------------------------------------------------------------------------
export type Organization = typeof organizations.$inferSelect;
export type NewOrganization = typeof organizations.$inferInsert;
export type Member = typeof members.$inferSelect;
export type NewMember = typeof members.$inferInsert;
export type Connection = typeof connections.$inferSelect;
export type NewConnection = typeof connections.$inferInsert;
export type Pipeline = typeof pipelines.$inferSelect;
export type NewPipeline = typeof pipelines.$inferInsert;
export type PipelineRun = typeof pipelineRuns.$inferSelect;
export type NewPipelineRun = typeof pipelineRuns.$inferInsert;
export type PipelineVersion = typeof pipelineVersions.$inferSelect;
export type NewPipelineVersion = typeof pipelineVersions.$inferInsert;
export type AlertConfig = typeof alertConfigs.$inferSelect;
export type NewAlertConfig = typeof alertConfigs.$inferInsert;
export type AlertLogEntry = typeof alertLog.$inferSelect;
export type NewAlertLogEntry = typeof alertLog.$inferInsert;
export type ConnectorRegistryEntry = typeof connectorRegistry.$inferSelect;
export type SchedulerStateEntry = typeof schedulerState.$inferSelect;
```

### Initial Migration SQL

```sql
-- src/server/db/migrations/0000_init.sql

-- Enable required extensions
CREATE EXTENSION IF NOT EXISTS pg_trgm;

-- (All tables created by Drizzle push)

-- Trigram index for searching pipeline names
CREATE INDEX IF NOT EXISTS pipelines_name_trgm_idx
  ON pipelines USING gin (name gin_trgm_ops);

-- Trigram index for searching connection names
CREATE INDEX IF NOT EXISTS connections_name_trgm_idx
  ON connections USING gin (name gin_trgm_ops);

-- Partial index for active scheduled pipelines (used by scheduler)
CREATE INDEX IF NOT EXISTS pipelines_active_scheduled_idx
  ON pipelines (next_run_at)
  WHERE status = 'active' AND schedule_enabled = true;

-- Partial index for running pipeline runs (used by monitoring)
CREATE INDEX IF NOT EXISTS pipeline_runs_running_idx
  ON pipeline_runs (started_at)
  WHERE status IN ('queued', 'extracting', 'transforming', 'loading');

-- Seed connector registry with initial connectors
INSERT INTO connector_registry (slug, name, type, category, auth_type, config_schema, supported_sync_modes) VALUES
  ('stripe', 'Stripe', 'source', 'payment', 'api_key',
   '{"fields": [{"key": "apiKey", "label": "Secret Key", "type": "password", "required": true, "placeholder": "sk_live_..."}]}',
   '["full", "incremental"]'),
  ('shopify', 'Shopify', 'source', 'ecommerce', 'api_key',
   '{"fields": [{"key": "shopDomain", "label": "Shop Domain", "type": "text", "required": true, "placeholder": "mystore.myshopify.com"}, {"key": "accessToken", "label": "Access Token", "type": "password", "required": true}]}',
   '["full", "incremental"]'),
  ('hubspot', 'HubSpot', 'source', 'crm', 'oauth',
   '{"fields": [{"key": "accessToken", "label": "Access Token", "type": "password", "required": true}]}',
   '["full", "incremental"]'),
  ('google_ads', 'Google Ads', 'source', 'marketing', 'oauth',
   '{"fields": [{"key": "customerId", "label": "Customer ID", "type": "text", "required": true}, {"key": "refreshToken", "label": "Refresh Token", "type": "password", "required": true}]}',
   '["full"]'),
  ('ga4', 'Google Analytics 4', 'source', 'analytics', 'oauth',
   '{"fields": [{"key": "propertyId", "label": "Property ID", "type": "text", "required": true}, {"key": "serviceAccountJson", "label": "Service Account JSON", "type": "password", "required": true}]}',
   '["full"]'),
  ('postgres', 'PostgreSQL', 'both', 'database', 'connection_string',
   '{"fields": [{"key": "host", "label": "Host", "type": "text", "required": true}, {"key": "port", "label": "Port", "type": "number", "required": true, "placeholder": "5432"}, {"key": "database", "label": "Database", "type": "text", "required": true}, {"key": "username", "label": "Username", "type": "text", "required": true}, {"key": "password", "label": "Password", "type": "password", "required": true}, {"key": "sslMode", "label": "SSL Mode", "type": "select", "required": false, "options": [{"label": "Require", "value": "require"}, {"label": "Prefer", "value": "prefer"}, {"label": "Disable", "value": "disable"}]}]}',
   '["full", "incremental"]'),
  ('mysql', 'MySQL', 'source', 'database', 'connection_string',
   '{"fields": [{"key": "host", "label": "Host", "type": "text", "required": true}, {"key": "port", "label": "Port", "type": "number", "required": true, "placeholder": "3306"}, {"key": "database", "label": "Database", "type": "text", "required": true}, {"key": "username", "label": "Username", "type": "text", "required": true}, {"key": "password", "label": "Password", "type": "password", "required": true}]}',
   '["full", "incremental"]'),
  ('google_sheets', 'Google Sheets', 'both', 'file', 'oauth',
   '{"fields": [{"key": "spreadsheetId", "label": "Spreadsheet ID", "type": "text", "required": true}, {"key": "sheetName", "label": "Sheet Name", "type": "text", "required": false, "placeholder": "Sheet1"}]}',
   '["full"]'),
  ('csv_upload', 'CSV Upload', 'source', 'file', 'file_upload',
   '{"fields": []}',
   '["full"]'),
  ('rest_api', 'REST API', 'source', 'api', 'api_key',
   '{"fields": [{"key": "baseUrl", "label": "Base URL", "type": "text", "required": true, "placeholder": "https://api.example.com"}, {"key": "authType", "label": "Auth Type", "type": "select", "required": true, "options": [{"label": "Bearer Token", "value": "bearer"}, {"label": "API Key Header", "value": "api_key"}, {"label": "Basic Auth", "value": "basic"}, {"label": "None", "value": "none"}]}, {"key": "apiKey", "label": "API Key / Token", "type": "password", "required": false}]}',
   '["full"]'),
  ('bigquery', 'BigQuery', 'destination', 'database', 'api_key',
   '{"fields": [{"key": "projectId", "label": "Project ID", "type": "text", "required": true}, {"key": "datasetId", "label": "Dataset ID", "type": "text", "required": true}, {"key": "serviceAccountJson", "label": "Service Account JSON", "type": "password", "required": true}]}',
   '["full"]');
```

---

## Architecture Deep-Dives

### Deep-Dive 1: Source Connector Pattern + Stripe Connector Example

```typescript
// src/server/connectors/base-connector.ts

import { S3Client, PutObjectCommand } from "@aws-sdk/client-s3";
import * as parquet from "@duckdb/node-api";
import { Connection } from "../db/schema";
import { decrypt } from "../services/encryption";

/**
 * Schema discovery result — describes available tables and columns
 * for a source connector. Used to populate the pipeline builder UI.
 */
export interface DiscoveredSchema {
  tables: Array<{
    name: string;
    columns: Array<{
      name: string;
      type: "string" | "number" | "boolean" | "date" | "datetime" | "json";
      nullable: boolean;
    }>;
    rowCount?: number;
    supportsIncremental: boolean;
    incrementalKeys?: string[];
  }>;
}

/**
 * Extract result — returned after each batch of data is extracted.
 * Connectors yield batches to support streaming large datasets.
 */
export interface ExtractBatch {
  tableName: string;
  rows: Record<string, unknown>[];
  schema: Array<{ name: string; type: string }>;
  isLast: boolean;
  cursor?: string; // Pagination cursor for incremental/paginated extraction
}

/**
 * Extract options — passed to the connector's extract method.
 */
export interface ExtractOptions {
  tables: string[];
  syncMode: "full" | "incremental";
  incrementalKey?: string;
  lastSyncCursor?: string;  // Resume from last successful sync
  maxRows?: number;          // Plan-based row limit
}

/**
 * Base class for all source connectors.
 * Each connector implements schema discovery and data extraction.
 */
export abstract class BaseSourceConnector {
  protected connection: Connection;
  protected decryptedCredentials: Record<string, unknown>;

  constructor(connection: Connection) {
    this.connection = connection;
    // Decrypt all credential values on construction
    this.decryptedCredentials = this.decryptCredentials(connection.credentials ?? {});
  }

  private decryptCredentials(creds: Record<string, unknown>): Record<string, unknown> {
    const decrypted: Record<string, unknown> = {};
    for (const [key, value] of Object.entries(creds)) {
      if (typeof value === "string" && value.includes(":")) {
        try {
          decrypted[key] = decrypt(value);
        } catch {
          decrypted[key] = value; // Not encrypted, pass through
        }
      } else {
        decrypted[key] = value;
      }
    }
    return decrypted;
  }

  /** Test the connection credentials. Throws on failure. */
  abstract testConnection(): Promise<{ ok: boolean; message: string }>;

  /** Discover available tables and their schemas. */
  abstract discoverSchema(): Promise<DiscoveredSchema>;

  /** Extract data in batches. Yields ExtractBatch for streaming. */
  abstract extract(options: ExtractOptions): AsyncGenerator<ExtractBatch>;

  /** Preview sample data from a table (limited rows for UI preview). */
  async previewTable(tableName: string, limit: number = 100): Promise<ExtractBatch> {
    const gen = this.extract({
      tables: [tableName],
      syncMode: "full",
      maxRows: limit,
    });
    const first = await gen.next();
    return first.value;
  }
}

// ---------------------------------------------------------------------------
// Connector factory — resolves connector type to implementation
// ---------------------------------------------------------------------------
export function createSourceConnector(connection: Connection): BaseSourceConnector {
  switch (connection.connectorType) {
    case "stripe":
      return new StripeConnector(connection);
    case "shopify":
      return new ShopifyConnector(connection);
    case "hubspot":
      return new HubSpotConnector(connection);
    case "postgres":
      return new PostgresSourceConnector(connection);
    case "mysql":
      return new MySQLSourceConnector(connection);
    case "google_sheets":
      return new GoogleSheetsSourceConnector(connection);
    case "csv_upload":
      return new CsvUploadConnector(connection);
    case "rest_api":
      return new RestApiConnector(connection);
    case "google_ads":
      return new GoogleAdsConnector(connection);
    case "ga4":
      return new GA4Connector(connection);
    default:
      throw new Error(`Unknown connector type: ${connection.connectorType}`);
  }
}
```

```typescript
// src/server/connectors/stripe-connector.ts

import Stripe from "stripe";
import { BaseSourceConnector, DiscoveredSchema, ExtractBatch, ExtractOptions } from "./base-connector";

/**
 * Stripe connector — extracts data from Stripe API.
 *
 * Supported tables: charges, customers, invoices, subscriptions,
 * payment_intents, products, prices, disputes, refunds, balance_transactions.
 *
 * Incremental sync uses `created` timestamp for list endpoints that support it.
 */

const STRIPE_TABLES = [
  { name: "charges", listMethod: "charges", incrementalKey: "created" },
  { name: "customers", listMethod: "customers", incrementalKey: "created" },
  { name: "invoices", listMethod: "invoices", incrementalKey: "created" },
  { name: "subscriptions", listMethod: "subscriptions", incrementalKey: "created" },
  { name: "payment_intents", listMethod: "paymentIntents", incrementalKey: "created" },
  { name: "products", listMethod: "products", incrementalKey: "created" },
  { name: "prices", listMethod: "prices", incrementalKey: "created" },
  { name: "disputes", listMethod: "disputes", incrementalKey: "created" },
  { name: "refunds", listMethod: "refunds", incrementalKey: "created" },
  { name: "balance_transactions", listMethod: "balanceTransactions", incrementalKey: "created" },
] as const;

const STRIPE_SCHEMA_MAP: Record<string, Array<{ name: string; type: string; nullable: boolean }>> = {
  charges: [
    { name: "id", type: "string", nullable: false },
    { name: "amount", type: "number", nullable: false },
    { name: "currency", type: "string", nullable: false },
    { name: "status", type: "string", nullable: false },
    { name: "customer", type: "string", nullable: true },
    { name: "description", type: "string", nullable: true },
    { name: "payment_method", type: "string", nullable: true },
    { name: "created", type: "datetime", nullable: false },
    { name: "metadata", type: "json", nullable: true },
  ],
  customers: [
    { name: "id", type: "string", nullable: false },
    { name: "email", type: "string", nullable: true },
    { name: "name", type: "string", nullable: true },
    { name: "phone", type: "string", nullable: true },
    { name: "currency", type: "string", nullable: true },
    { name: "created", type: "datetime", nullable: false },
    { name: "metadata", type: "json", nullable: true },
  ],
  // ... additional table schemas follow same pattern
};

export class StripeConnector extends BaseSourceConnector {
  private client: Stripe;

  constructor(connection: any) {
    super(connection);
    this.client = new Stripe(this.decryptedCredentials.apiKey as string, {
      apiVersion: "2024-06-20",
      maxNetworkRetries: 3,
    });
  }

  async testConnection(): Promise<{ ok: boolean; message: string }> {
    try {
      const account = await this.client.accounts.retrieve();
      return {
        ok: true,
        message: `Connected to Stripe account: ${account.settings?.dashboard?.display_name ?? account.id}`,
      };
    } catch (err: any) {
      return { ok: false, message: `Stripe connection failed: ${err.message}` };
    }
  }

  async discoverSchema(): Promise<DiscoveredSchema> {
    return {
      tables: STRIPE_TABLES.map((t) => ({
        name: t.name,
        columns: (STRIPE_SCHEMA_MAP[t.name] ?? []).map((c) => ({
          name: c.name,
          type: c.type as any,
          nullable: c.nullable,
        })),
        supportsIncremental: true,
        incrementalKeys: ["created"],
      })),
    };
  }

  async *extract(options: ExtractOptions): AsyncGenerator<ExtractBatch> {
    for (const tableName of options.tables) {
      const tableConfig = STRIPE_TABLES.find((t) => t.name === tableName);
      if (!tableConfig) continue;

      const listMethod = tableConfig.listMethod as string;
      const params: Stripe.PaginationParams & { created?: Stripe.RangeQueryParam } = {
        limit: 100,
      };

      // Incremental: only fetch records created after last sync cursor
      if (options.syncMode === "incremental" && options.lastSyncCursor) {
        params.created = { gt: parseInt(options.lastSyncCursor, 10) };
      }

      let hasMore = true;
      let totalRows = 0;
      let lastCursor: string | undefined;

      while (hasMore) {
        // Use Stripe auto-pagination via list method
        const resource = (this.client as any)[listMethod] as any;
        const response = await resource.list(params);

        const rows = response.data.map((item: any) => this.flattenStripeObject(item));
        totalRows += rows.length;

        // Track cursor for incremental sync (latest `created` timestamp)
        if (rows.length > 0) {
          lastCursor = String(response.data[0].created);
        }

        const isLast = !response.has_more || (options.maxRows && totalRows >= options.maxRows);

        yield {
          tableName,
          rows: options.maxRows ? rows.slice(0, options.maxRows - (totalRows - rows.length)) : rows,
          schema: STRIPE_SCHEMA_MAP[tableName] ?? [],
          isLast: isLast ?? false,
          cursor: lastCursor,
        };

        hasMore = response.has_more && !(options.maxRows && totalRows >= options.maxRows);
        if (hasMore && response.data.length > 0) {
          params.starting_after = response.data[response.data.length - 1].id;
        }
      }
    }
  }

  /**
   * Flatten a Stripe object into a simple key-value record.
   * Nested objects (metadata, address) are serialized as JSON strings.
   */
  private flattenStripeObject(obj: Record<string, any>): Record<string, unknown> {
    const flat: Record<string, unknown> = {};
    for (const [key, value] of Object.entries(obj)) {
      if (value === null || value === undefined) {
        flat[key] = null;
      } else if (typeof value === "object" && !Array.isArray(value) && key !== "metadata") {
        // Expand one-level nested objects (e.g., customer as string ID)
        flat[key] = typeof value.id === "string" ? value.id : JSON.stringify(value);
      } else if (typeof value === "object") {
        flat[key] = JSON.stringify(value);
      } else {
        flat[key] = value;
      }
    }
    // Convert Unix timestamps to ISO strings
    if (typeof flat.created === "number") {
      flat.created = new Date((flat.created as number) * 1000).toISOString();
    }
    return flat;
  }
}
```

### Deep-Dive 2: DuckDB Transform Engine (DAG Compiler + SQL Generation)

```typescript
// src/server/transforms/duckdb-engine.ts

import { Database } from "duckdb-async";
import {
  S3Client,
  GetObjectCommand,
  PutObjectCommand,
  ListObjectsV2Command,
} from "@aws-sdk/client-s3";
import { Readable } from "stream";
import { pipeline as streamPipeline } from "stream/promises";
import * as fs from "fs";
import * as path from "path";
import * as os from "os";

/**
 * Transform node types and their configuration shapes.
 * These correspond to the visual nodes in the React Flow editor.
 */
export interface FilterNodeData {
  type: "filter";
  column: string;
  operator: "eq" | "neq" | "gt" | "gte" | "lt" | "lte" | "contains" | "not_contains" | "is_null" | "is_not_null" | "in";
  value: unknown;
}

export interface RenameNodeData {
  type: "rename";
  mappings: Array<{ from: string; to: string }>;
  dropUnmapped: boolean;
}

export interface ComputedNodeData {
  type: "computed";
  columns: Array<{
    name: string;
    expression: string;   // SQL expression: "amount / 100.0", "CONCAT(first_name, ' ', last_name)"
    resultType: string;   // DOUBLE, VARCHAR, BOOLEAN, TIMESTAMP
  }>;
}

export interface AggregateNodeData {
  type: "aggregate";
  groupBy: string[];
  measures: Array<{
    column: string;
    function: "sum" | "count" | "avg" | "min" | "max" | "count_distinct";
    alias: string;
  }>;
}

export interface DeduplicateNodeData {
  type: "deduplicate";
  columns: string[];          // Columns to check for duplicates
  keepStrategy: "first" | "last";
  orderBy?: string;           // Column to determine first/last
}

export type TransformNodeData =
  | FilterNodeData
  | RenameNodeData
  | ComputedNodeData
  | AggregateNodeData
  | DeduplicateNodeData;

/**
 * DAG node as stored in pipeline.transformConfig.
 */
export interface DagNode {
  id: string;
  type: "source" | "filter" | "rename" | "computed" | "aggregate" | "deduplicate" | "destination";
  position: { x: number; y: number };
  data: Record<string, unknown>;
}

export interface DagEdge {
  id: string;
  source: string;
  target: string;
}

/**
 * Compile a transform DAG into a topologically-sorted sequence of SQL statements.
 *
 * Strategy:
 * 1. Parse the DAG graph (nodes + edges)
 * 2. Topological sort to determine execution order
 * 3. For each non-source/non-destination node, generate a SQL CTE or temp view
 * 4. Return the final SQL that produces the output dataset
 */
export function compileDagToSql(
  nodes: DagNode[],
  edges: DagEdge[],
  sourceTableName: string
): { sql: string; executionOrder: string[] } {
  // Build adjacency list and in-degree map
  const adjacency = new Map<string, string[]>();
  const inDegree = new Map<string, number>();

  for (const node of nodes) {
    adjacency.set(node.id, []);
    inDegree.set(node.id, 0);
  }

  for (const edge of edges) {
    adjacency.get(edge.source)?.push(edge.target);
    inDegree.set(edge.target, (inDegree.get(edge.target) ?? 0) + 1);
  }

  // Topological sort (Kahn's algorithm)
  const queue: string[] = [];
  for (const [nodeId, degree] of inDegree) {
    if (degree === 0) queue.push(nodeId);
  }

  const executionOrder: string[] = [];
  while (queue.length > 0) {
    const current = queue.shift()!;
    executionOrder.push(current);
    for (const neighbor of adjacency.get(current) ?? []) {
      const newDegree = (inDegree.get(neighbor) ?? 1) - 1;
      inDegree.set(neighbor, newDegree);
      if (newDegree === 0) queue.push(neighbor);
    }
  }

  if (executionOrder.length !== nodes.length) {
    throw new Error("Transform DAG contains a cycle — cannot compile");
  }

  // Generate SQL CTEs for each transform node
  const nodeMap = new Map(nodes.map((n) => [n.id, n]));
  const cteParts: string[] = [];
  const nodeOutputNames = new Map<string, string>();

  for (const nodeId of executionOrder) {
    const node = nodeMap.get(nodeId)!;

    if (node.type === "source") {
      nodeOutputNames.set(nodeId, sourceTableName);
      continue;
    }

    if (node.type === "destination") {
      // Destination node just references its input
      const inputEdge = edges.find((e) => e.target === nodeId);
      if (inputEdge) nodeOutputNames.set(nodeId, nodeOutputNames.get(inputEdge.source)!);
      continue;
    }

    // Find this node's input(s)
    const inputEdges = edges.filter((e) => e.target === nodeId);
    const inputName = nodeOutputNames.get(inputEdges[0]?.source ?? "") ?? sourceTableName;
    const outputName = `step_${nodeId.replace(/[^a-zA-Z0-9]/g, "_")}`;

    const sql = generateNodeSql(node, inputName);
    cteParts.push(`${outputName} AS (\n  ${sql}\n)`);
    nodeOutputNames.set(nodeId, outputName);
  }

  // Find the final output (destination node's input or last transform)
  const destNode = nodes.find((n) => n.type === "destination");
  const finalOutput = destNode
    ? nodeOutputNames.get(destNode.id)
    : nodeOutputNames.get(executionOrder[executionOrder.length - 1]);

  const fullSql = cteParts.length > 0
    ? `WITH\n${cteParts.join(",\n\n")}\n\nSELECT * FROM ${finalOutput}`
    : `SELECT * FROM ${sourceTableName}`;

  return { sql: fullSql, executionOrder };
}

/**
 * Generate SQL for a single transform node based on its type and config.
 */
function generateNodeSql(node: DagNode, inputTable: string): string {
  const data = node.data as TransformNodeData;

  switch (data.type) {
    case "filter":
      return generateFilterSql(data, inputTable);
    case "rename":
      return generateRenameSql(data, inputTable);
    case "computed":
      return generateComputedSql(data, inputTable);
    case "aggregate":
      return generateAggregateSql(data, inputTable);
    case "deduplicate":
      return generateDeduplicateSql(data, inputTable);
    default:
      return `SELECT * FROM ${inputTable}`;
  }
}

function generateFilterSql(data: FilterNodeData, input: string): string {
  const col = quoteIdentifier(data.column);
  const operatorMap: Record<string, string> = {
    eq: `${col} = ${quoteLiteral(data.value)}`,
    neq: `${col} != ${quoteLiteral(data.value)}`,
    gt: `${col} > ${quoteLiteral(data.value)}`,
    gte: `${col} >= ${quoteLiteral(data.value)}`,
    lt: `${col} < ${quoteLiteral(data.value)}`,
    lte: `${col} <= ${quoteLiteral(data.value)}`,
    contains: `${col} ILIKE '%' || ${quoteLiteral(data.value)} || '%'`,
    not_contains: `${col} NOT ILIKE '%' || ${quoteLiteral(data.value)} || '%'`,
    is_null: `${col} IS NULL`,
    is_not_null: `${col} IS NOT NULL`,
    in: `${col} IN (${(data.value as unknown[]).map(quoteLiteral).join(", ")})`,
  };
  return `SELECT * FROM ${input} WHERE ${operatorMap[data.operator] ?? "TRUE"}`;
}

function generateRenameSql(data: RenameNodeData, input: string): string {
  if (data.dropUnmapped) {
    const selects = data.mappings.map(
      (m) => `${quoteIdentifier(m.from)} AS ${quoteIdentifier(m.to)}`
    );
    return `SELECT ${selects.join(", ")} FROM ${input}`;
  }
  // Rename mapped columns, keep all others
  const renames = new Map(data.mappings.map((m) => [m.from, m.to]));
  // Use DuckDB's RENAME/COLUMNS — or dynamic SELECT
  const renameExprs = data.mappings.map(
    (m) => `${quoteIdentifier(m.from)} AS ${quoteIdentifier(m.to)}`
  );
  const excludes = data.mappings.map((m) => m.from);
  return `SELECT * EXCLUDE (${excludes.map(quoteIdentifier).join(", ")}), ${renameExprs.join(", ")} FROM ${input}`;
}

function generateComputedSql(data: ComputedNodeData, input: string): string {
  const computedCols = data.columns.map(
    (c) => `CAST(${c.expression} AS ${c.resultType}) AS ${quoteIdentifier(c.name)}`
  );
  return `SELECT *, ${computedCols.join(", ")} FROM ${input}`;
}

function generateAggregateSql(data: AggregateNodeData, input: string): string {
  const groupByCols = data.groupBy.map(quoteIdentifier).join(", ");
  const measureExprs = data.measures.map((m) => {
    const col = quoteIdentifier(m.column);
    const fnMap: Record<string, string> = {
      sum: `SUM(${col})`,
      count: `COUNT(${col})`,
      avg: `AVG(${col})`,
      min: `MIN(${col})`,
      max: `MAX(${col})`,
      count_distinct: `COUNT(DISTINCT ${col})`,
    };
    return `${fnMap[m.function]} AS ${quoteIdentifier(m.alias)}`;
  });
  return `SELECT ${groupByCols}, ${measureExprs.join(", ")} FROM ${input} GROUP BY ${groupByCols}`;
}

function generateDeduplicateSql(data: DeduplicateNodeData, input: string): string {
  const partitionCols = data.columns.map(quoteIdentifier).join(", ");
  const orderCol = data.orderBy ? quoteIdentifier(data.orderBy) : "rowid";
  const orderDir = data.keepStrategy === "last" ? "DESC" : "ASC";
  return `SELECT * FROM (
    SELECT *, ROW_NUMBER() OVER (PARTITION BY ${partitionCols} ORDER BY ${orderCol} ${orderDir}) AS _rn
    FROM ${input}
  ) WHERE _rn = 1`;
}

function quoteIdentifier(name: string): string {
  return `"${name.replace(/"/g, '""')}"`;
}

function quoteLiteral(value: unknown): string {
  if (value === null || value === undefined) return "NULL";
  if (typeof value === "number") return String(value);
  if (typeof value === "boolean") return value ? "TRUE" : "FALSE";
  return `'${String(value).replace(/'/g, "''")}'`;
}
```

```typescript
// src/server/transforms/transform-executor.ts

import { Database } from "duckdb-async";
import { S3Client, GetObjectCommand, PutObjectCommand } from "@aws-sdk/client-s3";
import * as fs from "fs";
import * as path from "path";
import * as os from "os";
import { compileDagToSql, DagNode, DagEdge } from "./duckdb-engine";

const s3 = new S3Client({ region: process.env.AWS_REGION ?? "us-east-1" });
const STAGING_BUCKET = process.env.S3_STAGING_BUCKET ?? "datapipe-staging";

export interface TransformResult {
  outputS3Key: string;
  rowCount: number;
  columnCount: number;
  durationMs: number;
  compiledSql: string;
}

/**
 * Execute the transform stage of a pipeline run.
 *
 * 1. Download extracted parquet from S3 staging
 * 2. Load into DuckDB in-memory database
 * 3. Compile transform DAG into SQL
 * 4. Execute the compiled SQL
 * 5. Write result as parquet back to S3
 */
export async function executeTransforms(opts: {
  runId: string;
  orgId: string;
  stagingPrefix: string;    // s3://datapipe-staging/{orgId}/{runId}/
  nodes: DagNode[];
  edges: DagEdge[];
  sourceTableName: string;
}): Promise<TransformResult> {
  const startTime = Date.now();
  const tmpDir = path.join(os.tmpdir(), `datapipe-${opts.runId}`);
  fs.mkdirSync(tmpDir, { recursive: true });

  let db: Database | null = null;

  try {
    // Step 1: Download extracted parquet from S3
    const extractedKey = `${opts.orgId}/${opts.runId}/extracted/${opts.sourceTableName}.parquet`;
    const localParquetPath = path.join(tmpDir, "input.parquet");

    const s3Response = await s3.send(new GetObjectCommand({
      Bucket: STAGING_BUCKET,
      Key: extractedKey,
    }));

    const bodyStream = s3Response.Body as NodeJS.ReadableStream;
    const writeStream = fs.createWriteStream(localParquetPath);
    await new Promise<void>((resolve, reject) => {
      bodyStream.pipe(writeStream).on("finish", resolve).on("error", reject);
    });

    // Step 2: Initialize DuckDB and load parquet
    db = await Database.create(":memory:");
    await db.run("INSTALL parquet; LOAD parquet;");
    await db.run(
      `CREATE TABLE ${opts.sourceTableName} AS SELECT * FROM read_parquet('${localParquetPath}')`
    );

    // Step 3: Compile DAG to SQL
    const { sql: compiledSql, executionOrder } = compileDagToSql(
      opts.nodes,
      opts.edges,
      opts.sourceTableName
    );

    console.log(`[Transform] Run ${opts.runId} — compiled SQL:\n${compiledSql}`);
    console.log(`[Transform] Execution order: ${executionOrder.join(" → ")}`);

    // Step 4: Execute transform SQL and write to output table
    await db.run(`CREATE TABLE transform_output AS ${compiledSql}`);

    // Get row count
    const countResult = await db.all("SELECT COUNT(*) AS cnt FROM transform_output");
    const rowCount = Number(countResult[0].cnt);

    // Get column count
    const colResult = await db.all(
      "SELECT COUNT(*) AS cnt FROM information_schema.columns WHERE table_name = 'transform_output'"
    );
    const columnCount = Number(colResult[0].cnt);

    // Step 5: Write result as parquet to S3
    const outputParquetPath = path.join(tmpDir, "output.parquet");
    await db.run(`COPY transform_output TO '${outputParquetPath}' (FORMAT PARQUET, COMPRESSION ZSTD)`);

    const outputKey = `${opts.orgId}/${opts.runId}/transformed/${opts.sourceTableName}.parquet`;
    const fileBuffer = fs.readFileSync(outputParquetPath);
    await s3.send(new PutObjectCommand({
      Bucket: STAGING_BUCKET,
      Key: outputKey,
      Body: fileBuffer,
      ContentType: "application/octet-stream",
    }));

    return {
      outputS3Key: outputKey,
      rowCount,
      columnCount,
      durationMs: Date.now() - startTime,
      compiledSql,
    };
  } finally {
    // Cleanup
    if (db) await db.close();
    fs.rmSync(tmpDir, { recursive: true, force: true });
  }
}
```

### Deep-Dive 3: Pipeline Execution Orchestrator (3-Stage ETL with S3 Staging)

```typescript
// src/server/orchestrator/pipeline-runner.ts

import { eq, and, sql } from "drizzle-orm";
import { db } from "../db";
import {
  pipelines,
  pipelineRuns,
  connections,
  organizations,
  alertConfigs,
} from "../db/schema";
import { createSourceConnector, ExtractBatch } from "../connectors/base-connector";
import { createDestinationLoader } from "../loaders/base-loader";
import { executeTransforms } from "../transforms/transform-executor";
import { S3Client, PutObjectCommand } from "@aws-sdk/client-s3";
import { sendAlertNotifications } from "../services/alert-service";

const s3 = new S3Client({ region: process.env.AWS_REGION ?? "us-east-1" });
const STAGING_BUCKET = process.env.S3_STAGING_BUCKET ?? "datapipe-staging";

const PLAN_ROW_LIMITS: Record<string, number> = {
  free: 10_000,
  pro: 500_000,
  team: 5_000_000,
  enterprise: Infinity,
};

/**
 * Execute a full pipeline run: Extract → Transform → Load.
 *
 * This function is called by the BullMQ worker when a pipeline run is triggered
 * (either by schedule or manual trigger). It orchestrates all three stages
 * sequentially with error handling, status updates, and alert dispatch.
 */
export async function executePipelineRun(pipelineRunId: string): Promise<void> {
  // Load the run with all related data
  const [run] = await db
    .select()
    .from(pipelineRuns)
    .where(eq(pipelineRuns.id, pipelineRunId))
    .limit(1);

  if (!run) throw new Error(`Pipeline run not found: ${pipelineRunId}`);

  const [pipe] = await db
    .select()
    .from(pipelines)
    .where(eq(pipelines.id, run.pipelineId))
    .limit(1);

  if (!pipe) throw new Error(`Pipeline not found: ${run.pipelineId}`);

  const [org] = await db
    .select()
    .from(organizations)
    .where(eq(organizations.id, pipe.orgId))
    .limit(1);

  const rowLimit = PLAN_ROW_LIMITS[org?.plan ?? "free"];

  const [sourceConn] = await db
    .select()
    .from(connections)
    .where(eq(connections.id, pipe.sourceConnectionId!))
    .limit(1);

  const [destConn] = await db
    .select()
    .from(connections)
    .where(eq(connections.id, pipe.destinationConnectionId!))
    .limit(1);

  const stagingPrefix = `${pipe.orgId}/${pipelineRunId}`;

  // Update run with staging prefix and started timestamp
  await db
    .update(pipelineRuns)
    .set({
      stagingPrefix: `s3://${STAGING_BUCKET}/${stagingPrefix}/`,
      status: "extracting",
      startedAt: new Date(),
      extractStartedAt: new Date(),
    })
    .where(eq(pipelineRuns.id, pipelineRunId));

  try {
    // =========== STAGE 1: EXTRACT ===========
    const connector = createSourceConnector(sourceConn);
    const sourceConfig = pipe.sourceConfig as any;

    let totalExtracted = 0;
    const allRows: Record<string, unknown>[] = [];

    for await (const batch of connector.extract({
      tables: sourceConfig.tables ?? [],
      syncMode: sourceConfig.syncMode ?? "full",
      incrementalKey: sourceConfig.incrementalKey,
      maxRows: rowLimit,
    })) {
      allRows.push(...batch.rows);
      totalExtracted += batch.rows.length;

      // Update progress
      await db
        .update(pipelineRuns)
        .set({ rowsExtracted: totalExtracted })
        .where(eq(pipelineRuns.id, pipelineRunId));

      if (totalExtracted >= rowLimit) break;
    }

    // Write extracted data to S3 as JSON lines (DuckDB can read this)
    const extractedKey = `${stagingPrefix}/extracted/${sourceConfig.tables?.[0] ?? "data"}.parquet`;
    await writeRowsToParquet(allRows, STAGING_BUCKET, extractedKey);

    await db
      .update(pipelineRuns)
      .set({
        rowsExtracted: totalExtracted,
        extractCompletedAt: new Date(),
        status: "transforming",
        transformStartedAt: new Date(),
      })
      .where(eq(pipelineRuns.id, pipelineRunId));

    // =========== STAGE 2: TRANSFORM ===========
    const transformConfig = pipe.transformConfig as any;
    const hasTransforms =
      transformConfig?.nodes?.filter((n: any) =>
        !["source", "destination"].includes(n.type)
      ).length > 0;

    let transformedRowCount = totalExtracted;
    let transformedKey = extractedKey; // Pass-through if no transforms

    if (hasTransforms) {
      const transformResult = await executeTransforms({
        runId: pipelineRunId,
        orgId: pipe.orgId,
        stagingPrefix,
        nodes: transformConfig.nodes,
        edges: transformConfig.edges,
        sourceTableName: sourceConfig.tables?.[0] ?? "data",
      });

      transformedRowCount = transformResult.rowCount;
      transformedKey = transformResult.outputS3Key;
    }

    await db
      .update(pipelineRuns)
      .set({
        rowsTransformed: transformedRowCount,
        transformCompletedAt: new Date(),
        status: "loading",
        loadStartedAt: new Date(),
      })
      .where(eq(pipelineRuns.id, pipelineRunId));

    // =========== STAGE 3: LOAD ===========
    const loader = createDestinationLoader(destConn);
    const destConfig = pipe.destinationConfig as any;

    const loadResult = await loader.load({
      s3Bucket: STAGING_BUCKET,
      s3Key: transformedKey,
      targetTable: destConfig.targetTable,
      loadMode: destConfig.loadMode ?? "append",
      upsertKey: destConfig.upsertKey,
      createTableIfMissing: destConfig.createTableIfMissing ?? true,
    });

    // =========== COMPLETE ===========
    const completedAt = new Date();
    const durationMs = completedAt.getTime() - (run.startedAt?.getTime() ?? completedAt.getTime());

    await db
      .update(pipelineRuns)
      .set({
        rowsLoaded: loadResult.rowsLoaded,
        bytesProcessed: loadResult.bytesWritten,
        loadCompletedAt: completedAt,
        completedAt,
        durationMs,
        status: "success",
      })
      .where(eq(pipelineRuns.id, pipelineRunId));

    // Update pipeline last run timestamp
    await db
      .update(pipelines)
      .set({ lastRunAt: completedAt, updatedAt: completedAt })
      .where(eq(pipelines.id, pipe.id));

    // Update org usage counters
    await db
      .update(organizations)
      .set({
        usageRowsSynced: sql`usage_rows_synced + ${loadResult.rowsLoaded}`,
      })
      .where(eq(organizations.id, pipe.orgId));

    // Check for zero-rows alert
    if (loadResult.rowsLoaded === 0) {
      await checkAndFireAlerts(pipe.id, pipe.orgId, pipelineRunId, "zero_rows");
    }
  } catch (error: any) {
    // =========== FAILURE HANDLING ===========
    const failedAt = new Date();
    const durationMs = failedAt.getTime() - (run.startedAt?.getTime() ?? failedAt.getTime());

    // Determine which stage failed based on current status
    const [currentRun] = await db
      .select({ status: pipelineRuns.status })
      .from(pipelineRuns)
      .where(eq(pipelineRuns.id, pipelineRunId))
      .limit(1);

    const failedStage = currentRun?.status === "extracting"
      ? "extract"
      : currentRun?.status === "transforming"
        ? "transform"
        : "load";

    await db
      .update(pipelineRuns)
      .set({
        status: "failed",
        completedAt: failedAt,
        durationMs,
        errorMessage: error.message?.slice(0, 2000),
        errorDetails: {
          stage: failedStage,
          stackTrace: error.stack?.slice(0, 5000),
          retryCount: 0,
        },
      })
      .where(eq(pipelineRuns.id, pipelineRunId));

    // Update pipeline status to error
    await db
      .update(pipelines)
      .set({ status: "error", lastRunAt: failedAt, updatedAt: failedAt })
      .where(eq(pipelines.id, pipe.id));

    // Fire failure alerts
    await checkAndFireAlerts(pipe.id, pipe.orgId, pipelineRunId, "failure");

    throw error; // Re-throw for BullMQ retry logic
  }
}

/**
 * Write an array of row objects to S3 as a parquet file via DuckDB.
 */
async function writeRowsToParquet(
  rows: Record<string, unknown>[],
  bucket: string,
  key: string
): Promise<void> {
  if (rows.length === 0) return;

  const { Database } = await import("duckdb-async");
  const tmpPath = path.join(os.tmpdir(), `extract-${Date.now()}.parquet`);
  const db = await Database.create(":memory:");

  try {
    // Write rows as JSON then import into DuckDB
    const jsonPath = tmpPath.replace(".parquet", ".json");
    fs.writeFileSync(jsonPath, JSON.stringify(rows));

    await db.run("INSTALL parquet; LOAD parquet;");
    await db.run(`CREATE TABLE data AS SELECT * FROM read_json_auto('${jsonPath}')`);
    await db.run(`COPY data TO '${tmpPath}' (FORMAT PARQUET, COMPRESSION ZSTD)`);

    // Upload to S3
    const fileBuffer = fs.readFileSync(tmpPath);
    await s3.send(new PutObjectCommand({
      Bucket: bucket,
      Key: key,
      Body: fileBuffer,
      ContentType: "application/octet-stream",
    }));

    // Cleanup
    fs.unlinkSync(jsonPath);
    fs.unlinkSync(tmpPath);
  } finally {
    await db.close();
  }
}

/**
 * Check alert configs for a pipeline and fire matching alerts.
 */
async function checkAndFireAlerts(
  pipelineId: string,
  orgId: string,
  runId: string,
  alertType: "failure" | "slow_run" | "zero_rows" | "schema_change"
): Promise<void> {
  const configs = await db
    .select()
    .from(alertConfigs)
    .where(
      and(
        eq(alertConfigs.orgId, orgId),
        eq(alertConfigs.enabled, true),
        sql`(${alertConfigs.pipelineId} = ${pipelineId} OR ${alertConfigs.pipelineId} IS NULL)`,
        eq(alertConfigs.type, alertType)
      )
    );

  for (const config of configs) {
    await sendAlertNotifications(config, pipelineId, runId);
  }
}

import * as path from "path";
import * as os from "os";
```

### Deep-Dive 4: React Flow Pipeline Editor (Custom Nodes, Config Panels, Data Preview)

```typescript
// src/components/pipeline-editor/pipeline-editor.tsx
"use client";

import { useCallback, useMemo, useState } from "react";
import ReactFlow, {
  ReactFlowProvider,
  Background,
  Controls,
  MiniMap,
  Panel,
  useNodesState,
  useEdgesState,
  addEdge,
  Connection,
  Node,
  Edge,
  NodeTypes,
  type OnConnect,
} from "reactflow";
import "reactflow/dist/style.css";

import { SourceNode } from "./nodes/source-node";
import { FilterNode } from "./nodes/filter-node";
import { RenameNode } from "./nodes/rename-node";
import { ComputedNode } from "./nodes/computed-node";
import { AggregateNode } from "./nodes/aggregate-node";
import { DeduplicateNode } from "./nodes/deduplicate-node";
import { DestinationNode } from "./nodes/destination-node";
import { NodeConfigPanel } from "./panels/node-config-panel";
import { DataPreviewPanel } from "./panels/data-preview-panel";
import { TransformToolbar } from "./toolbar/transform-toolbar";
import { trpc } from "@/lib/trpc/client";
import { Pipeline } from "@/server/db/schema";

/** Register all custom node types for React Flow */
const nodeTypes: NodeTypes = {
  source: SourceNode,
  filter: FilterNode,
  rename: RenameNode,
  computed: ComputedNode,
  aggregate: AggregateNode,
  deduplicate: DeduplicateNode,
  destination: DestinationNode,
};

interface PipelineEditorProps {
  pipeline: Pipeline;
  onSave: (config: { nodes: Node[]; edges: Edge[] }) => void;
}

export function PipelineEditor({ pipeline, onSave }: PipelineEditorProps) {
  const transformConfig = pipeline.transformConfig as any;
  const [nodes, setNodes, onNodesChange] = useNodesState(transformConfig?.nodes ?? []);
  const [edges, setEdges, onEdgesChange] = useEdgesState(transformConfig?.edges ?? []);
  const [selectedNodeId, setSelectedNodeId] = useState<string | null>(null);
  const [previewNodeId, setPreviewNodeId] = useState<string | null>(null);

  /** Handle new edge connections between nodes */
  const onConnect: OnConnect = useCallback(
    (params: Connection) => {
      // Validate connection: prevent cycles, enforce single input per node
      const targetNode = nodes.find((n) => n.id === params.target);
      const existingInput = edges.find((e) => e.target === params.target);

      if (existingInput && targetNode?.type !== "destination") {
        // Most transform nodes accept only one input (except join in v2)
        return;
      }

      setEdges((eds) =>
        addEdge(
          {
            ...params,
            animated: true,
            style: { stroke: "#6366f1", strokeWidth: 2 },
          },
          eds
        )
      );
    },
    [nodes, edges, setEdges]
  );

  /** Add a new transform node to the canvas */
  const addTransformNode = useCallback(
    (type: string) => {
      const id = `${type}-${Date.now()}`;
      const newNode: Node = {
        id,
        type,
        position: { x: 300, y: 100 + nodes.length * 120 },
        data: getDefaultNodeData(type),
      };
      setNodes((nds) => [...nds, newNode]);
      setSelectedNodeId(id);
    },
    [nodes, setNodes]
  );

  /** Update data for a specific node (from the config panel) */
  const updateNodeData = useCallback(
    (nodeId: string, data: Record<string, unknown>) => {
      setNodes((nds) =>
        nds.map((n) => (n.id === nodeId ? { ...n, data: { ...n.data, ...data } } : n))
      );
    },
    [setNodes]
  );

  /** Delete selected node and its connected edges */
  const deleteNode = useCallback(
    (nodeId: string) => {
      setNodes((nds) => nds.filter((n) => n.id !== nodeId));
      setEdges((eds) => eds.filter((e) => e.source !== nodeId && e.target !== nodeId));
      if (selectedNodeId === nodeId) setSelectedNodeId(null);
      if (previewNodeId === nodeId) setPreviewNodeId(null);
    },
    [selectedNodeId, previewNodeId, setNodes, setEdges]
  );

  /** Save the pipeline DAG configuration */
  const handleSave = useCallback(() => {
    onSave({ nodes, edges });
  }, [nodes, edges, onSave]);

  const selectedNode = useMemo(
    () => nodes.find((n) => n.id === selectedNodeId),
    [nodes, selectedNodeId]
  );

  return (
    <div className="flex h-[calc(100vh-64px)]">
      {/* Transform toolbar — left sidebar */}
      <TransformToolbar onAddNode={addTransformNode} />

      {/* React Flow canvas — center */}
      <div className="flex-1 relative">
        <ReactFlow
          nodes={nodes}
          edges={edges}
          nodeTypes={nodeTypes}
          onNodesChange={onNodesChange}
          onEdgesChange={onEdgesChange}
          onConnect={onConnect}
          onNodeClick={(_, node) => setSelectedNodeId(node.id)}
          onNodeDoubleClick={(_, node) => setPreviewNodeId(node.id)}
          onPaneClick={() => setSelectedNodeId(null)}
          fitView
          snapToGrid
          snapGrid={[16, 16]}
          defaultEdgeOptions={{
            animated: true,
            style: { stroke: "#6366f1", strokeWidth: 2 },
          }}
        >
          <Background gap={16} size={1} color="#e5e7eb" />
          <Controls />
          <MiniMap
            nodeStrokeColor="#6366f1"
            nodeColor="#f1f5f9"
            nodeBorderRadius={8}
          />
          <Panel position="top-right">
            <button
              onClick={handleSave}
              className="rounded-lg bg-indigo-600 px-4 py-2 text-sm font-medium text-white
                         hover:bg-indigo-700 shadow-sm"
            >
              Save Pipeline
            </button>
          </Panel>
        </ReactFlow>
      </div>

      {/* Config panel — right sidebar */}
      {selectedNode && (
        <NodeConfigPanel
          node={selectedNode}
          onUpdate={(data) => updateNodeData(selectedNode.id, data)}
          onDelete={() => deleteNode(selectedNode.id)}
          onPreview={() => setPreviewNodeId(selectedNode.id)}
        />
      )}

      {/* Data preview — bottom panel */}
      {previewNodeId && (
        <DataPreviewPanel
          pipelineId={pipeline.id}
          nodeId={previewNodeId}
          nodes={nodes}
          edges={edges}
          onClose={() => setPreviewNodeId(null)}
        />
      )}
    </div>
  );
}

/** Default data configuration for each node type */
function getDefaultNodeData(type: string): Record<string, unknown> {
  switch (type) {
    case "filter":
      return { type: "filter", column: "", operator: "eq", value: "" };
    case "rename":
      return { type: "rename", mappings: [], dropUnmapped: false };
    case "computed":
      return { type: "computed", columns: [] };
    case "aggregate":
      return { type: "aggregate", groupBy: [], measures: [] };
    case "deduplicate":
      return { type: "deduplicate", columns: [], keepStrategy: "first" };
    default:
      return {};
  }
}
```

```typescript
// src/components/pipeline-editor/nodes/filter-node.tsx
"use client";

import { memo } from "react";
import { Handle, Position, type NodeProps } from "reactflow";
import { Filter } from "lucide-react";

interface FilterNodeData {
  type: "filter";
  column: string;
  operator: string;
  value: unknown;
}

/**
 * Custom React Flow node for the Filter transform.
 * Displays a compact summary of the filter condition.
 */
export const FilterNode = memo(function FilterNode({ data, selected }: NodeProps<FilterNodeData>) {
  const operatorLabels: Record<string, string> = {
    eq: "=", neq: "!=", gt: ">", gte: ">=", lt: "<", lte: "<=",
    contains: "contains", not_contains: "!contains",
    is_null: "is null", is_not_null: "is not null", in: "in",
  };

  const hasConfig = data.column && data.operator;
  const label = hasConfig
    ? `${data.column} ${operatorLabels[data.operator] ?? data.operator} ${data.value ?? ""}`
    : "Configure filter...";

  return (
    <div
      className={`rounded-lg border-2 bg-white px-4 py-3 shadow-sm min-w-[200px]
        ${selected ? "border-indigo-500 ring-2 ring-indigo-200" : "border-amber-300"}`}
    >
      <Handle type="target" position={Position.Top} className="!bg-indigo-500 !w-3 !h-3" />
      <div className="flex items-center gap-2 mb-1">
        <div className="rounded bg-amber-100 p-1">
          <Filter className="h-4 w-4 text-amber-600" />
        </div>
        <span className="text-xs font-semibold uppercase tracking-wide text-amber-600">
          Filter
        </span>
      </div>
      <p className="text-sm text-gray-700 truncate">{label}</p>
      <Handle type="source" position={Position.Bottom} className="!bg-indigo-500 !w-3 !h-3" />
    </div>
  );
});
```

```typescript
// src/components/pipeline-editor/panels/data-preview-panel.tsx
"use client";

import { useMemo } from "react";
import { AgGridReact } from "ag-grid-react";
import { type ColDef } from "ag-grid-community";
import "ag-grid-community/styles/ag-grid.css";
import "ag-grid-community/styles/ag-theme-alpine.css";
import { trpc } from "@/lib/trpc/client";
import { Node, Edge } from "reactflow";
import { X, Loader2 } from "lucide-react";

interface DataPreviewPanelProps {
  pipelineId: string;
  nodeId: string;
  nodes: Node[];
  edges: Edge[];
  onClose: () => void;
}

/**
 * Bottom panel that shows a live data preview for any node in the pipeline.
 *
 * Calls the tRPC `pipeline.previewNodeData` endpoint which:
 * 1. Extracts a sample (100 rows) from the source
 * 2. Compiles the transform DAG up to the selected node
 * 3. Executes in DuckDB
 * 4. Returns the result set
 *
 * Uses AG Grid for virtualized rendering of potentially wide schemas.
 */
export function DataPreviewPanel({
  pipelineId,
  nodeId,
  nodes,
  edges,
  onClose,
}: DataPreviewPanelProps) {
  const preview = trpc.pipeline.previewNodeData.useQuery(
    {
      pipelineId,
      nodeId,
      nodes: nodes.map((n) => ({ id: n.id, type: n.type!, position: n.position, data: n.data })),
      edges: edges.map((e) => ({ id: e.id, source: e.source, target: e.target })),
    },
    {
      staleTime: 30_000, // Cache preview for 30 seconds
      refetchOnWindowFocus: false,
    }
  );

  /** Auto-generate AG Grid column definitions from the first row */
  const columnDefs = useMemo<ColDef[]>(() => {
    if (!preview.data?.rows?.length) return [];
    const firstRow = preview.data.rows[0];
    return Object.keys(firstRow).map((key) => ({
      field: key,
      headerName: key,
      sortable: true,
      filter: true,
      resizable: true,
      minWidth: 120,
      maxWidth: 400,
      // Auto-detect cell renderer based on value type
      cellRenderer: (params: any) => {
        const value = params.value;
        if (value === null || value === undefined) {
          return '<span class="text-gray-400 italic">null</span>';
        }
        if (typeof value === "object") {
          return `<span class="text-xs font-mono">${JSON.stringify(value)}</span>`;
        }
        return String(value);
      },
    }));
  }, [preview.data?.rows]);

  return (
    <div className="absolute bottom-0 left-0 right-0 h-[320px] border-t bg-white shadow-lg z-50">
      {/* Header */}
      <div className="flex items-center justify-between border-b px-4 py-2">
        <div className="flex items-center gap-3">
          <h3 className="text-sm font-semibold text-gray-800">Data Preview</h3>
          {preview.data && (
            <span className="rounded-full bg-gray-100 px-2 py-0.5 text-xs text-gray-600">
              {preview.data.rows.length} rows x {preview.data.columns.length} columns
            </span>
          )}
        </div>
        <button onClick={onClose} className="rounded p-1 hover:bg-gray-100">
          <X className="h-4 w-4 text-gray-500" />
        </button>
      </div>

      {/* Content */}
      <div className="h-[calc(100%-40px)]">
        {preview.isLoading && (
          <div className="flex h-full items-center justify-center gap-2 text-gray-500">
            <Loader2 className="h-5 w-5 animate-spin" />
            <span className="text-sm">Running preview...</span>
          </div>
        )}

        {preview.isError && (
          <div className="flex h-full items-center justify-center">
            <div className="text-center">
              <p className="text-sm font-medium text-red-600">Preview failed</p>
              <p className="mt-1 text-xs text-gray-500">{preview.error.message}</p>
            </div>
          </div>
        )}

        {preview.data && preview.data.rows.length > 0 && (
          <div className="ag-theme-alpine h-full w-full">
            <AgGridReact
              rowData={preview.data.rows}
              columnDefs={columnDefs}
              defaultColDef={{
                sortable: true,
                filter: true,
                resizable: true,
              }}
              animateRows={false}
              rowHeight={32}
              headerHeight={36}
              suppressMovableColumns
            />
          </div>
        )}

        {preview.data && preview.data.rows.length === 0 && (
          <div className="flex h-full items-center justify-center text-gray-500">
            <p className="text-sm">No rows returned. Check your transform configuration.</p>
          </div>
        )}
      </div>
    </div>
  );
}
```

---

## Error Recovery for Failed Pipeline Stages

Each pipeline run follows a 3-stage ETL process (Extract -> Transform -> Load). Error recovery is handled at the stage level:

- **Per-stage retry**: Each stage retries up to 3 times with exponential backoff (5s, 30s, 120s). Retry state is tracked in `pipelineRuns.errorDetails.retryCount`. If the extract stage fails, transform and load are never attempted.
- **Partial rollback on critical failure**: If the load stage fails mid-write (e.g., BigQuery insertion error after 50% of rows), the pipeline run records the `failedAtRow` marker in `errorDetails`. On retry, the loader resumes from the failed row using the staged parquet file in S3, avoiding re-extraction and re-transformation.
- **Dead letter queue**: Rows that fail transformation (e.g., type coercion error, null constraint violation) are written to a separate `{stagingPrefix}/dlq/` parquet file in S3 instead of halting the entire pipeline. The run summary includes a `dlqRowCount` field. Users can download the DLQ file from the run detail page for manual review.
- **Stage-level error reporting in UI**: The pipeline run detail page shows per-stage status (success/failed/skipped), timing, row counts, and error messages. Failed stages display the stack trace (truncated) and a "Retry from this stage" button that re-enqueues only the failed stage.

---

## Pipeline Version Tracking: MVP vs Post-MVP

The `pipelineVersions` table exists in the MVP schema to provide basic version snapshotting:

**MVP usage**: Every time a pipeline's configuration is saved (source, transform, or destination config changes), the `pipelines.version` counter increments and a snapshot row is inserted into `pipelineVersions` with the full config at that point. Each `pipelineRuns` row records which `version` it executed against. This enables debugging ("which config did this run use?") and comparing run results across config changes.

**Post-MVP usage (spec F7 — version control)**: The version tracking table becomes the foundation for full version control with features like: named versions (tagging), diff view between versions, rollback to a previous version, and branch-and-merge for team collaboration on pipeline definitions.

---

## Post-MVP Features

The following capabilities are referenced in the spec but deferred beyond MVP:

- **Version control for pipeline definitions (spec F7)**: Full Git-like version control with branching, diffing, and rollback. MVP provides basic version snapshotting (see above); post-MVP adds named releases, visual diff between versions, and one-click rollback.
- **Templates and recipes marketplace (spec F8)**: Pre-built pipeline templates for common use cases (e.g., "Stripe -> BigQuery revenue dashboard", "HubSpot -> Sheets weekly leads report"). Community-contributed templates with one-click import. Requires a template registry, sharing permissions, and a discovery UI.
- **Real-time streaming support**: Extend beyond batch ETL to support CDC (Change Data Capture) streams from PostgreSQL (logical replication), MySQL (binlog), and webhook-based sources. Requires a persistent streaming worker architecture (likely Kafka or Redpanda) that differs significantly from the batch ECS Fargate model.
- **Advanced scheduling (cron expressions)**: MVP supports daily/hourly presets. Post-MVP adds full cron expression support (e.g., "every 15 minutes on weekdays", "first Monday of each month at 6am"), timezone-aware scheduling, and pipeline dependency chains (trigger pipeline B after pipeline A completes successfully).
