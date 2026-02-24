# 16. PermitFlow — Access Control Dashboard

## Implementation Plan

**MVP Scope:** 4 SaaS adapter integrations in MVP — Google Workspace, Slack, GitHub, and AWS IAM (OAuth + API tokens), with adapter pattern designed for path to 15+ connectors post-MVP. Unified access view with per-user and per-app drill-downs (AG Grid), access anomaly detection (unused accounts, stale access), access review campaigns with reviewer assignment and approve/revoke/flag workflow, role templates for provisioning checklists, audit log with full action history, AES-256-GCM encrypted credential storage, BullMQ-based scheduled sync orchestrator with diff detection, email notifications via Resend, Stripe billing with three tiers (Starter $199/mo, Business $499/mo, Enterprise custom).

> **Scope Note — SCIM Provisioning:** SCIM 2.0 provisioning is excluded from MVP. While the `connectionType` field supports `"scim"` as a value, MVP adapters use OAuth and API token flows only. SCIM provisioning requires implementing a full SCIM server (RFC 7644) with user/group resource endpoints, which adds 2-3 weeks of scope. The adapter pattern supports adding SCIM as a connection type post-MVP without schema changes.

> **Scope Note — HRIS Integration:** Direct HRIS integration (BambooHR, Workday, Rippling) is excluded from MVP. The `managedUsers` table includes an `hrisId` field for future HRIS sync, and the `metadata` JSONB field can store provider-specific identifiers. MVP relies on adapter-discovered users and manual CSV import for employee data. HRIS sync will be added as a dedicated adapter type post-MVP.

---

## Tech Decisions

| Layer | Technology | Notes |
|-------|-----------|-------|
| Framework | Next.js 14+ App Router | `src/` directory, TypeScript strict |
| API | tRPC v11 | superjson transformer, Clerk auth context |
| Database | Neon PostgreSQL | Access records, review campaigns, audit log |
| ORM | Drizzle ORM | Push-based migrations, typed schema |
| Auth | Clerk (with SAML) | Organization-scoped, `orgId` from session, SAML for enterprise SSO |
| Billing | Stripe | Checkout Sessions + Customer Portal + Webhooks |
| Email | Resend | Review reminders, access request notifications, offboarding alerts |
| Queue | BullMQ + Upstash Redis | Scheduled sync jobs, review reminder cron, async revocation |
| Integrations | Google Admin SDK, Slack Web API, Octokit, AWS SDK (IAM) | App adapter pattern with encrypted credentials |
| Encryption | AES-256-GCM | OAuth tokens + API keys stored encrypted in `apps.credentials` |
| Data Grid | AG Grid (Community) | Unified access view with virtual scrolling, column filtering, export |
| UI | Tailwind CSS, Radix UI, Recharts | |
| Hosting | Vercel (app), Upstash (Redis) | Serverless-compatible BullMQ |
| Monitoring | Sentry, Vercel Analytics | |

### Key Architecture Decisions

1. **App Adapter Pattern over monolithic integration code**: Each SaaS integration implements a typed `AppAdapter` interface with methods `listUsers()`, `getUserAccess()`, `revokeAccess()`, and `grantAccess()`. New integrations are added by creating a single adapter file and registering it in the adapter registry. This isolates API-specific logic, simplifies testing with mock adapters, and enables parallel development of new connectors.

> **MVP connector scope:** 4 adapters are fully implemented (Google Workspace, Slack, GitHub, AWS IAM). The path to 15+ connectors: v1.1 adds Microsoft 365, Okta, Azure AD, Jira, and Salesforce. v2 adds remaining connectors from the spec's 100+ target list. Each new adapter follows the `AppAdapter` interface pattern and requires ~2-3 days of development.

2. **BullMQ scheduled sync with diff detection over real-time webhooks**: While webhooks are ideal, most SaaS apps have unreliable or limited webhook support for user/access changes. Instead, each connected app syncs on a configurable schedule (hourly/daily) via BullMQ repeatable jobs. The sync worker fetches current state, diffs against the last known state, and emits change events. This provides consistent behavior across all adapters and a single audit trail for all changes.

3. **Access Review as a state machine**: Review campaigns and individual review items follow strict state machines with enforced transitions. Campaign states: `draft` -> `active` -> `completed` | `cancelled`. Item states: `pending` -> `approved` | `revoked` | `flagged`. A "revoked" decision triggers the adapter's `revokeAccess()` method with confirmation logging. This ensures every access decision is auditable and no invalid state transitions occur.

4. **AG Grid for unified access view over custom table components**: The access matrix (users x apps x roles) can reach thousands of rows. AG Grid provides virtual scrolling, server-side row model, built-in column filtering, CSV/Excel export, and cell rendering customization without building a custom virtualized table. The Community edition is sufficient for MVP.

> **SCIM exclusion:** The spec lists SCIM as a connection method, but MVP defers SCIM provisioning. Rationale: SCIM requires implementing a SCIM server endpoint that identity providers push to, adding significant complexity (schema mapping, conflict resolution, partial updates). For MVP, the pull-based adapter pattern (scheduled API polling) provides equivalent functionality for access visibility. SCIM will be added in v2 for real-time provisioning/deprovisioning with Okta and Azure AD.

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
  plan: text("plan").default("starter").notNull(), // starter | business | enterprise
  stripeCustomerId: text("stripe_customer_id"),
  stripeSubscriptionId: text("stripe_subscription_id"),
  settings: jsonb("settings").default({}).$type<{
    defaultSyncFrequency?: "hourly" | "daily" | "weekly";
    reviewReminderDays?: number[];        // days before due to send reminders
    autoRevokeOnReviewDecision?: boolean; // auto-execute revocation on "revoke" decision
    notificationFromName?: string;
    notificationFromEmail?: string;
    securityAlertEmail?: string;          // email for anomaly/offboarding alerts
    staleAccessThresholdDays?: number;    // days without login to flag as stale (default 90)
  }>(),
  createdAt: timestamp("created_at", { withTimezone: true }).defaultNow().notNull(),
  updatedAt: timestamp("updated_at", { withTimezone: true }).defaultNow().notNull(),
});

// ---------------------------------------------------------------------------
// Members (org members who log into PermitFlow itself)
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
    name: text("name").notNull(),
    role: text("role").default("member").notNull(), // owner | admin | member | auditor
    createdAt: timestamp("created_at", { withTimezone: true }).defaultNow().notNull(),
    updatedAt: timestamp("updated_at", { withTimezone: true }).defaultNow().notNull(),
  },
  (t) => [
    uniqueIndex("members_clerk_user_idx").on(t.clerkUserId, t.orgId),
    index("members_org_idx").on(t.orgId),
  ],
);

// ---------------------------------------------------------------------------
// Managed Users (employees whose access is being tracked)
// ---------------------------------------------------------------------------
export const managedUsers = pgTable(
  "managed_users",
  {
    id: uuid("id").primaryKey().defaultRandom(),
    orgId: uuid("org_id")
      .references(() => organizations.id, { onDelete: "cascade" })
      .notNull(),
    email: text("email").notNull(),
    name: text("name").notNull(),
    department: text("department"),                          // Engineering, Sales, etc.
    title: text("title"),                                    // Software Engineer, etc.
    managerId: uuid("manager_id"),                           // self-ref to managedUsers
    employmentStatus: text("employment_status").default("active").notNull(),
    // active | offboarding | offboarded
    hrisId: text("hris_id"),                                 // external HRIS identifier
    startDate: timestamp("start_date", { withTimezone: true }),
    endDate: timestamp("end_date", { withTimezone: true }),
    lastSyncedAt: timestamp("last_synced_at", { withTimezone: true }),
    metadata: jsonb("metadata").default({}).$type<{
      googleUserId?: string;
      slackUserId?: string;
      githubUsername?: string;
      awsArn?: string;
      avatarUrl?: string;
    }>(),
    createdAt: timestamp("created_at", { withTimezone: true }).defaultNow().notNull(),
    updatedAt: timestamp("updated_at", { withTimezone: true }).defaultNow().notNull(),
  },
  (t) => [
    uniqueIndex("managed_users_org_email_idx").on(t.orgId, t.email),
    index("managed_users_org_idx").on(t.orgId),
    index("managed_users_department_idx").on(t.orgId, t.department),
    index("managed_users_status_idx").on(t.orgId, t.employmentStatus),
  ],
);

// ---------------------------------------------------------------------------
// Apps (connected SaaS applications)
// ---------------------------------------------------------------------------
export const apps = pgTable(
  "apps",
  {
    id: uuid("id").primaryKey().defaultRandom(),
    orgId: uuid("org_id")
      .references(() => organizations.id, { onDelete: "cascade" })
      .notNull(),
    adapterType: text("adapter_type").notNull(),
    // google_workspace | slack | github | aws | jira | notion | custom
    name: text("name").notNull(),                            // "Google Workspace", "GitHub — Acme Org"
    category: text("category"),                              // productivity | development | security | business
    iconUrl: text("icon_url"),
    connectionType: text("connection_type").notNull(),       // oauth | api_token | scim
    credentials: jsonb("credentials").default({}).$type<{
      accessToken?: string;        // AES-256-GCM encrypted
      refreshToken?: string;       // AES-256-GCM encrypted
      apiKey?: string;             // AES-256-GCM encrypted
      domain?: string;             // e.g., workspace domain, GitHub org
      scimEndpoint?: string;
      expiresAt?: string;          // ISO date for token expiry
      extra?: Record<string, string>; // adapter-specific encrypted fields
    }>(),
    syncFrequency: text("sync_frequency").default("daily").notNull(),
    // realtime | hourly | daily | weekly
    syncStatus: text("sync_status").default("pending").notNull(),
    // pending | syncing | synced | error
    lastSyncedAt: timestamp("last_synced_at", { withTimezone: true }),
    lastSyncError: text("last_sync_error"),
    totalUsers: integer("total_users").default(0).notNull(),
    licenseCount: integer("license_count"),
    costPerLicense: integer("cost_per_license"),             // cents
    renewalDate: timestamp("renewal_date", { withTimezone: true }),
    ownerMemberId: uuid("owner_member_id")
      .references(() => members.id, { onDelete: "set null" }),
    isSanctioned: boolean("is_sanctioned").default(true).notNull(),
    createdAt: timestamp("created_at", { withTimezone: true }).defaultNow().notNull(),
    updatedAt: timestamp("updated_at", { withTimezone: true }).defaultNow().notNull(),
  },
  (t) => [
    index("apps_org_idx").on(t.orgId),
    index("apps_adapter_type_idx").on(t.orgId, t.adapterType),
    index("apps_sync_status_idx").on(t.syncStatus),
  ],
);

// ---------------------------------------------------------------------------
// App Access (user-to-app access records)
// ---------------------------------------------------------------------------
export const appAccess = pgTable(
  "app_access",
  {
    id: uuid("id").primaryKey().defaultRandom(),
    orgId: uuid("org_id")
      .references(() => organizations.id, { onDelete: "cascade" })
      .notNull(),
    appId: uuid("app_id")
      .references(() => apps.id, { onDelete: "cascade" })
      .notNull(),
    userId: uuid("user_id")
      .references(() => managedUsers.id, { onDelete: "cascade" })
      .notNull(),
    role: text("role").notNull(),                            // admin | member | viewer | custom
    permissions: jsonb("permissions").default({}).$type<{
      scopes?: string[];           // e.g., ["repo:write", "admin:org"]
      groups?: string[];           // e.g., ["engineering", "platform"]
      customRole?: string;         // app-specific role name
    }>(),
    grantedAt: timestamp("granted_at", { withTimezone: true }).defaultNow().notNull(),
    grantedBy: text("granted_by"),                           // who provisioned: "sync", "manual", userId
    lastLoginAt: timestamp("last_login_at", { withTimezone: true }),
    status: text("status").default("active").notNull(),      // active | suspended | revoked
    expiresAt: timestamp("expires_at", { withTimezone: true }), // for time-bound access
    source: text("source").default("discovered").notNull(),
    // provisioned | manual | self_service | discovered
    externalId: text("external_id"),                         // provider-specific unique ID
    createdAt: timestamp("created_at", { withTimezone: true }).defaultNow().notNull(),
    updatedAt: timestamp("updated_at", { withTimezone: true }).defaultNow().notNull(),
  },
  (t) => [
    uniqueIndex("app_access_unique_idx").on(t.appId, t.userId, t.role),
    index("app_access_org_idx").on(t.orgId),
    index("app_access_app_idx").on(t.appId),
    index("app_access_user_idx").on(t.userId),
    index("app_access_status_idx").on(t.orgId, t.status),
    index("app_access_last_login_idx").on(t.lastLoginAt),
  ],
);

// ---------------------------------------------------------------------------
// Access Reviews (review campaigns)
// ---------------------------------------------------------------------------
export const accessReviews = pgTable(
  "access_reviews",
  {
    id: uuid("id").primaryKey().defaultRandom(),
    orgId: uuid("org_id")
      .references(() => organizations.id, { onDelete: "cascade" })
      .notNull(),
    name: text("name").notNull(),                            // "Q1 2026 Quarterly Review"
    description: text("description"),
    cadence: text("cadence"),                                // quarterly | semi_annual | annual | one_time
    scope: jsonb("scope").default({}).$type<{
      appIds?: string[];             // specific apps, or empty for all
      departments?: string[];        // specific departments
      includeAll?: boolean;          // review everything
    }>(),
    status: text("status").default("draft").notNull(),
    // draft | active | completed | cancelled
    startedAt: timestamp("started_at", { withTimezone: true }),
    dueDate: timestamp("due_date", { withTimezone: true }).notNull(),
    completedAt: timestamp("completed_at", { withTimezone: true }),
    createdByMemberId: uuid("created_by_member_id")
      .references(() => members.id, { onDelete: "set null" }),
    totalItems: integer("total_items").default(0).notNull(),
    completedItems: integer("completed_items").default(0).notNull(),
    createdAt: timestamp("created_at", { withTimezone: true }).defaultNow().notNull(),
    updatedAt: timestamp("updated_at", { withTimezone: true }).defaultNow().notNull(),
  },
  (t) => [
    index("access_reviews_org_idx").on(t.orgId),
    index("access_reviews_status_idx").on(t.orgId, t.status),
    index("access_reviews_due_idx").on(t.dueDate),
  ],
);

// ---------------------------------------------------------------------------
// Review Items (individual user-app review decisions)
// ---------------------------------------------------------------------------
export const reviewItems = pgTable(
  "review_items",
  {
    id: uuid("id").primaryKey().defaultRandom(),
    orgId: uuid("org_id")
      .references(() => organizations.id, { onDelete: "cascade" })
      .notNull(),
    reviewId: uuid("review_id")
      .references(() => accessReviews.id, { onDelete: "cascade" })
      .notNull(),
    appAccessId: uuid("app_access_id")
      .references(() => appAccess.id, { onDelete: "cascade" })
      .notNull(),
    reviewerMemberId: uuid("reviewer_member_id")
      .references(() => members.id, { onDelete: "set null" }),
    decision: text("decision").default("pending").notNull(),
    // pending | approved | revoked | flagged
    justification: text("justification"),
    decidedAt: timestamp("decided_at", { withTimezone: true }),
    revokeExecuted: boolean("revoke_executed").default(false).notNull(),
    revokeExecutedAt: timestamp("revoke_executed_at", { withTimezone: true }),
    revokeError: text("revoke_error"),
    createdAt: timestamp("created_at", { withTimezone: true }).defaultNow().notNull(),
    updatedAt: timestamp("updated_at", { withTimezone: true }).defaultNow().notNull(),
  },
  (t) => [
    index("review_items_review_idx").on(t.reviewId),
    index("review_items_reviewer_idx").on(t.reviewerMemberId),
    index("review_items_decision_idx").on(t.reviewId, t.decision),
    uniqueIndex("review_items_access_review_idx").on(t.reviewId, t.appAccessId),
  ],
);

// ---------------------------------------------------------------------------
// Role Templates (provisioning blueprints)
// ---------------------------------------------------------------------------
export const roleTemplates = pgTable(
  "role_templates",
  {
    id: uuid("id").primaryKey().defaultRandom(),
    orgId: uuid("org_id")
      .references(() => organizations.id, { onDelete: "cascade" })
      .notNull(),
    name: text("name").notNull(),                            // "Engineering", "Sales"
    description: text("description"),
    appAccesses: jsonb("app_accesses").default([]).$type<
      Array<{
        appId: string;
        role: string;                  // admin | member | viewer
        permissions?: Record<string, string[]>;
      }>
    >(),
    isDefault: boolean("is_default").default(false).notNull(),
    createdAt: timestamp("created_at", { withTimezone: true }).defaultNow().notNull(),
    updatedAt: timestamp("updated_at", { withTimezone: true }).defaultNow().notNull(),
  },
  (t) => [
    index("role_templates_org_idx").on(t.orgId),
  ],
);

// ---------------------------------------------------------------------------
// Access Requests (self-service)
// ---------------------------------------------------------------------------
export const accessRequests = pgTable(
  "access_requests",
  {
    id: uuid("id").primaryKey().defaultRandom(),
    orgId: uuid("org_id")
      .references(() => organizations.id, { onDelete: "cascade" })
      .notNull(),
    requesterUserId: uuid("requester_user_id")
      .references(() => managedUsers.id, { onDelete: "cascade" })
      .notNull(),
    appId: uuid("app_id")
      .references(() => apps.id, { onDelete: "cascade" })
      .notNull(),
    requestedRole: text("requested_role").notNull(),         // admin | member | viewer
    justification: text("justification").notNull(),
    duration: text("duration").default("permanent").notNull(), // permanent | temporary
    expiresAt: timestamp("expires_at", { withTimezone: true }), // for temporary access
    status: text("status").default("pending").notNull(),
    // pending | approved | denied | expired | cancelled
    approverMemberId: uuid("approver_member_id")
      .references(() => members.id, { onDelete: "set null" }),
    decidedAt: timestamp("decided_at", { withTimezone: true }),
    denialReason: text("denial_reason"),
    autoApproved: boolean("auto_approved").default(false).notNull(),
    createdAt: timestamp("created_at", { withTimezone: true }).defaultNow().notNull(),
    updatedAt: timestamp("updated_at", { withTimezone: true }).defaultNow().notNull(),
  },
  (t) => [
    index("access_requests_org_idx").on(t.orgId),
    index("access_requests_status_idx").on(t.orgId, t.status),
    index("access_requests_requester_idx").on(t.requesterUserId),
  ],
);

// ---------------------------------------------------------------------------
// Audit Log
// ---------------------------------------------------------------------------
export const auditLog = pgTable(
  "audit_log",
  {
    id: uuid("id").primaryKey().defaultRandom(),
    orgId: uuid("org_id")
      .references(() => organizations.id, { onDelete: "cascade" })
      .notNull(),
    actorType: text("actor_type").notNull(),                 // member | system | sync
    actorId: text("actor_id"),                               // member UUID or "system"
    action: text("action").notNull(),
    // access_granted | access_revoked | access_modified | review_started |
    // review_completed | review_decision | user_provisioned | user_deprovisioned |
    // app_connected | app_disconnected | sync_completed | sync_failed |
    // request_submitted | request_approved | request_denied | template_applied
    targetType: text("target_type"),                         // user | app | access | review | request
    targetId: text("target_id"),                             // UUID of the target record
    targetUserId: uuid("target_user_id")
      .references(() => managedUsers.id, { onDelete: "set null" }),
    appId: uuid("app_id")
      .references(() => apps.id, { onDelete: "set null" }),
    details: jsonb("details").default({}).$type<{
      before?: Record<string, unknown>;
      after?: Record<string, unknown>;
      reason?: string;
      syncStats?: { added: number; removed: number; modified: number };
      errorMessage?: string;
    }>(),
    ipAddress: text("ip_address"),
    createdAt: timestamp("created_at", { withTimezone: true }).defaultNow().notNull(),
  },
  (t) => [
    index("audit_log_org_idx").on(t.orgId),
    index("audit_log_action_idx").on(t.orgId, t.action),
    index("audit_log_target_user_idx").on(t.targetUserId),
    index("audit_log_app_idx").on(t.appId),
    index("audit_log_created_at_idx").on(t.orgId, t.createdAt),
  ],
);

// ---------------------------------------------------------------------------
// Sync Jobs (background sync execution records)
// ---------------------------------------------------------------------------
export const syncJobs = pgTable(
  "sync_jobs",
  {
    id: uuid("id").primaryKey().defaultRandom(),
    orgId: uuid("org_id")
      .references(() => organizations.id, { onDelete: "cascade" })
      .notNull(),
    appId: uuid("app_id")
      .references(() => apps.id, { onDelete: "cascade" })
      .notNull(),
    status: text("status").default("queued").notNull(),
    // queued | running | completed | failed
    triggeredBy: text("triggered_by").notNull(),             // schedule | manual | webhook
    startedAt: timestamp("started_at", { withTimezone: true }),
    completedAt: timestamp("completed_at", { withTimezone: true }),
    usersFound: integer("users_found").default(0).notNull(),
    accessRecordsFound: integer("access_records_found").default(0).notNull(),
    changesDetected: jsonb("changes_detected").default({}).$type<{
      added: number;
      removed: number;
      modified: number;
      details?: Array<{ userId: string; change: string; field?: string }>;
    }>(),
    errorMessage: text("error_message"),
    durationMs: integer("duration_ms"),
    createdAt: timestamp("created_at", { withTimezone: true }).defaultNow().notNull(),
  },
  (t) => [
    index("sync_jobs_org_idx").on(t.orgId),
    index("sync_jobs_app_idx").on(t.appId),
    index("sync_jobs_status_idx").on(t.status),
    index("sync_jobs_created_at_idx").on(t.createdAt),
  ],
);

// ---------------------------------------------------------------------------
// App Adapter Registry (available adapter metadata)
// ---------------------------------------------------------------------------
export const appAdapterRegistry = pgTable(
  "app_adapter_registry",
  {
    id: uuid("id").primaryKey().defaultRandom(),
    adapterType: text("adapter_type").unique().notNull(),
    // google_workspace | slack | github | aws | jira | notion ...
    displayName: text("display_name").notNull(),
    category: text("category").notNull(),
    // productivity | development | security | business | hr | support
    iconUrl: text("icon_url"),
    connectionTypes: jsonb("connection_types").default([]).$type<string[]>(),
    // ["oauth", "api_token"]
    capabilities: jsonb("capabilities").default({}).$type<{
      canListUsers: boolean;
      canReadRoles: boolean;
      canRevokeAccess: boolean;
      canGrantAccess: boolean;
      canReadLastLogin: boolean;
      supportsWebhook: boolean;
    }>(),
    oauthConfig: jsonb("oauth_config").$type<{
      authorizationUrl: string;
      tokenUrl: string;
      scopes: string[];
      requiresDomain?: boolean;
    }>(),
    isAvailable: boolean("is_available").default(true).notNull(),
    createdAt: timestamp("created_at", { withTimezone: true }).defaultNow().notNull(),
  },
  (t) => [
    index("adapter_registry_category_idx").on(t.category),
  ],
);

// ---------------------------------------------------------------------------
// Relations
// ---------------------------------------------------------------------------
export const organizationsRelations = relations(organizations, ({ many }) => ({
  members: many(members),
  managedUsers: many(managedUsers),
  apps: many(apps),
  appAccess: many(appAccess),
  accessReviews: many(accessReviews),
  reviewItems: many(reviewItems),
  accessRequests: many(accessRequests),
  auditLog: many(auditLog),
  syncJobs: many(syncJobs),
  roleTemplates: many(roleTemplates),
}));

export const membersRelations = relations(members, ({ one, many }) => ({
  organization: one(organizations, {
    fields: [members.orgId],
    references: [organizations.id],
  }),
  ownedApps: many(apps),
  reviewItems: many(reviewItems),
  createdReviews: many(accessReviews),
}));

export const managedUsersRelations = relations(managedUsers, ({ one, many }) => ({
  organization: one(organizations, {
    fields: [managedUsers.orgId],
    references: [organizations.id],
  }),
  manager: one(managedUsers, {
    fields: [managedUsers.managerId],
    references: [managedUsers.id],
    relationName: "managerReports",
  }),
  directReports: many(managedUsers, { relationName: "managerReports" }),
  appAccess: many(appAccess),
  accessRequests: many(accessRequests),
  auditLogEntries: many(auditLog),
}));

export const appsRelations = relations(apps, ({ one, many }) => ({
  organization: one(organizations, {
    fields: [apps.orgId],
    references: [organizations.id],
  }),
  owner: one(members, {
    fields: [apps.ownerMemberId],
    references: [members.id],
  }),
  appAccess: many(appAccess),
  syncJobs: many(syncJobs),
}));

export const appAccessRelations = relations(appAccess, ({ one, many }) => ({
  organization: one(organizations, {
    fields: [appAccess.orgId],
    references: [organizations.id],
  }),
  app: one(apps, {
    fields: [appAccess.appId],
    references: [apps.id],
  }),
  user: one(managedUsers, {
    fields: [appAccess.userId],
    references: [managedUsers.id],
  }),
  reviewItems: many(reviewItems),
}));

export const accessReviewsRelations = relations(accessReviews, ({ one, many }) => ({
  organization: one(organizations, {
    fields: [accessReviews.orgId],
    references: [organizations.id],
  }),
  createdBy: one(members, {
    fields: [accessReviews.createdByMemberId],
    references: [members.id],
  }),
  items: many(reviewItems),
}));

export const reviewItemsRelations = relations(reviewItems, ({ one }) => ({
  organization: one(organizations, {
    fields: [reviewItems.orgId],
    references: [organizations.id],
  }),
  review: one(accessReviews, {
    fields: [reviewItems.reviewId],
    references: [accessReviews.id],
  }),
  appAccess: one(appAccess, {
    fields: [reviewItems.appAccessId],
    references: [appAccess.id],
  }),
  reviewer: one(members, {
    fields: [reviewItems.reviewerMemberId],
    references: [members.id],
  }),
}));

export const roleTemplatesRelations = relations(roleTemplates, ({ one }) => ({
  organization: one(organizations, {
    fields: [roleTemplates.orgId],
    references: [organizations.id],
  }),
}));

export const accessRequestsRelations = relations(accessRequests, ({ one }) => ({
  organization: one(organizations, {
    fields: [accessRequests.orgId],
    references: [organizations.id],
  }),
  requester: one(managedUsers, {
    fields: [accessRequests.requesterUserId],
    references: [managedUsers.id],
  }),
  app: one(apps, {
    fields: [accessRequests.appId],
    references: [apps.id],
  }),
  approver: one(members, {
    fields: [accessRequests.approverMemberId],
    references: [members.id],
  }),
}));

export const auditLogRelations = relations(auditLog, ({ one }) => ({
  organization: one(organizations, {
    fields: [auditLog.orgId],
    references: [organizations.id],
  }),
  targetUser: one(managedUsers, {
    fields: [auditLog.targetUserId],
    references: [managedUsers.id],
  }),
  app: one(apps, {
    fields: [auditLog.appId],
    references: [apps.id],
  }),
}));

export const syncJobsRelations = relations(syncJobs, ({ one }) => ({
  organization: one(organizations, {
    fields: [syncJobs.orgId],
    references: [organizations.id],
  }),
  app: one(apps, {
    fields: [syncJobs.appId],
    references: [apps.id],
  }),
}));

// ---------------------------------------------------------------------------
// Type Exports
// ---------------------------------------------------------------------------
export type Organization = typeof organizations.$inferSelect;
export type NewOrganization = typeof organizations.$inferInsert;
export type Member = typeof members.$inferSelect;
export type NewMember = typeof members.$inferInsert;
export type ManagedUser = typeof managedUsers.$inferSelect;
export type NewManagedUser = typeof managedUsers.$inferInsert;
export type App = typeof apps.$inferSelect;
export type NewApp = typeof apps.$inferInsert;
export type AppAccess = typeof appAccess.$inferSelect;
export type NewAppAccess = typeof appAccess.$inferInsert;
export type AccessReview = typeof accessReviews.$inferSelect;
export type NewAccessReview = typeof accessReviews.$inferInsert;
export type ReviewItem = typeof reviewItems.$inferSelect;
export type NewReviewItem = typeof reviewItems.$inferInsert;
export type RoleTemplate = typeof roleTemplates.$inferSelect;
export type NewRoleTemplate = typeof roleTemplates.$inferInsert;
export type AccessRequest = typeof accessRequests.$inferSelect;
export type NewAccessRequest = typeof accessRequests.$inferInsert;
export type AuditLogEntry = typeof auditLog.$inferSelect;
export type NewAuditLogEntry = typeof auditLog.$inferInsert;
export type SyncJob = typeof syncJobs.$inferSelect;
export type NewSyncJob = typeof syncJobs.$inferInsert;
export type AppAdapterRegistryEntry = typeof appAdapterRegistry.$inferSelect;
```

### Initial Migration SQL

```sql
-- src/server/db/migrations/0000_init.sql

-- Enable required extensions
CREATE EXTENSION IF NOT EXISTS pg_trgm;

-- (All tables created by Drizzle push, then manually add performance indexes:)

-- Trigram index for fuzzy text search on managed user names
CREATE INDEX IF NOT EXISTS managed_users_name_trgm_idx
  ON managed_users USING gin (name gin_trgm_ops);

-- Trigram index for fuzzy text search on managed user emails
CREATE INDEX IF NOT EXISTS managed_users_email_trgm_idx
  ON managed_users USING gin (email gin_trgm_ops);

-- Trigram index for searching app names
CREATE INDEX IF NOT EXISTS apps_name_trgm_idx
  ON apps USING gin (name gin_trgm_ops);

-- Composite index for the unified access view (user -> apps with status)
CREATE INDEX IF NOT EXISTS app_access_unified_view_idx
  ON app_access (org_id, user_id, status)
  INCLUDE (app_id, role, last_login_at);

-- Partial index for active access records only (most common query)
CREATE INDEX IF NOT EXISTS app_access_active_idx
  ON app_access (org_id, app_id)
  WHERE status = 'active';

-- Index for stale access detection (no login within threshold)
CREATE INDEX IF NOT EXISTS app_access_stale_detection_idx
  ON app_access (org_id, last_login_at)
  WHERE status = 'active' AND last_login_at IS NOT NULL;

-- Seed the adapter registry with MVP adapters
INSERT INTO app_adapter_registry (adapter_type, display_name, category, icon_url, connection_types, capabilities, oauth_config, is_available)
VALUES
  ('google_workspace', 'Google Workspace', 'productivity', '/icons/google.svg',
   '["oauth"]'::jsonb,
   '{"canListUsers": true, "canReadRoles": true, "canRevokeAccess": true, "canGrantAccess": false, "canReadLastLogin": true, "supportsWebhook": false}'::jsonb,
   '{"authorizationUrl": "https://accounts.google.com/o/oauth2/v2/auth", "tokenUrl": "https://oauth2.googleapis.com/token", "scopes": ["https://www.googleapis.com/auth/admin.directory.user.readonly", "https://www.googleapis.com/auth/admin.directory.group.readonly"], "requiresDomain": true}'::jsonb,
   true),
  ('slack', 'Slack', 'productivity', '/icons/slack.svg',
   '["oauth"]'::jsonb,
   '{"canListUsers": true, "canReadRoles": true, "canRevokeAccess": true, "canGrantAccess": true, "canReadLastLogin": false, "supportsWebhook": true}'::jsonb,
   '{"authorizationUrl": "https://slack.com/oauth/v2/authorize", "tokenUrl": "https://slack.com/api/oauth.v2.access", "scopes": ["users:read", "users:read.email", "admin.users:read", "admin.users:write"], "requiresDomain": false}'::jsonb,
   true),
  ('github', 'GitHub', 'development', '/icons/github.svg',
   '["oauth"]'::jsonb,
   '{"canListUsers": true, "canReadRoles": true, "canRevokeAccess": true, "canGrantAccess": true, "canReadLastLogin": false, "supportsWebhook": true}'::jsonb,
   '{"authorizationUrl": "https://github.com/login/oauth/authorize", "tokenUrl": "https://github.com/login/oauth/access_token", "scopes": ["read:org", "admin:org"], "requiresDomain": false}'::jsonb,
   true),
  ('aws', 'AWS IAM', 'development', '/icons/aws.svg',
   '["api_token"]'::jsonb,
   '{"canListUsers": true, "canReadRoles": true, "canRevokeAccess": true, "canGrantAccess": false, "canReadLastLogin": true, "supportsWebhook": false}'::jsonb,
   NULL,
   true);
```

---

## Architecture Deep-Dives

### Deep-Dive 1: App Adapter Pattern + Google Workspace Adapter

```typescript
// src/server/adapters/types.ts

/**
 * Standardized user record returned by all adapters.
 * Each adapter maps its provider-specific user object to this shape.
 */
export interface AdapterUser {
  externalId: string;           // provider-specific unique user ID
  email: string;
  name: string;
  role: "admin" | "member" | "viewer" | "custom";
  customRole?: string;          // original role name from provider
  permissions?: {
    scopes?: string[];
    groups?: string[];
  };
  lastLoginAt?: Date;
  isSuspended?: boolean;
  avatarUrl?: string;
  metadata?: Record<string, string>;
}

/**
 * Result of a sync operation — the full current state of users in the app.
 */
export interface AdapterSyncResult {
  users: AdapterUser[];
  totalLicenses?: number;
  metadata?: Record<string, unknown>;
}

/**
 * Revocation result with status and optional error.
 */
export interface AdapterRevokeResult {
  success: boolean;
  error?: string;
  details?: string;
}

/**
 * The core adapter interface that every SaaS integration must implement.
 * Each method receives decrypted credentials — the adapter never handles encryption.
 */
export interface AppAdapter {
  readonly type: string;

  /** Test that the stored credentials are still valid */
  testConnection(credentials: Record<string, string>): Promise<{
    valid: boolean;
    error?: string;
    accountName?: string;
  }>;

  /** Fetch all users and their access levels from the connected app */
  syncUsers(credentials: Record<string, string>): Promise<AdapterSyncResult>;

  /** Revoke a specific user's access */
  revokeAccess(
    credentials: Record<string, string>,
    externalUserId: string,
  ): Promise<AdapterRevokeResult>;

  /** Grant access to a user (optional — not all apps support this) */
  grantAccess?(
    credentials: Record<string, string>,
    email: string,
    role: string,
  ): Promise<{ success: boolean; externalId?: string; error?: string }>;
}

// ---------------------------------------------------------------------------
// Adapter Registry — maps adapterType to implementation
// ---------------------------------------------------------------------------
// src/server/adapters/registry.ts

import { GoogleWorkspaceAdapter } from "./google-workspace";
import { SlackAdapter } from "./slack";
import { GitHubAdapter } from "./github";
import { AwsIamAdapter } from "./aws-iam";
import type { AppAdapter } from "./types";

const adapters: Record<string, AppAdapter> = {
  google_workspace: new GoogleWorkspaceAdapter(),
  slack: new SlackAdapter(),
  github: new GitHubAdapter(),
  aws: new AwsIamAdapter(),
};

export function getAdapter(adapterType: string): AppAdapter {
  const adapter = adapters[adapterType];
  if (!adapter) {
    throw new Error(`No adapter registered for type: ${adapterType}`);
  }
  return adapter;
}

export function listAvailableAdapters(): string[] {
  return Object.keys(adapters);
}

// ---------------------------------------------------------------------------
// Google Workspace Adapter — uses Google Admin SDK
// ---------------------------------------------------------------------------
// src/server/adapters/google-workspace.ts

import { google, admin_directory_v1 } from "googleapis";
import type {
  AppAdapter,
  AdapterUser,
  AdapterSyncResult,
  AdapterRevokeResult,
} from "./types";

export class GoogleWorkspaceAdapter implements AppAdapter {
  readonly type = "google_workspace";

  private getClient(credentials: Record<string, string>) {
    const oauth2Client = new google.auth.OAuth2();
    oauth2Client.setCredentials({
      access_token: credentials.accessToken,
      refresh_token: credentials.refreshToken,
    });
    return google.admin({ version: "directory_v1", auth: oauth2Client });
  }

  async testConnection(credentials: Record<string, string>) {
    try {
      const admin = this.getClient(credentials);
      const res = await admin.users.list({
        domain: credentials.domain,
        maxResults: 1,
      });
      return {
        valid: true,
        accountName: `Google Workspace — ${credentials.domain}`,
      };
    } catch (err: any) {
      return { valid: false, error: err.message ?? "Connection failed" };
    }
  }

  async syncUsers(credentials: Record<string, string>): Promise<AdapterSyncResult> {
    const admin = this.getClient(credentials);
    const users: AdapterUser[] = [];
    let pageToken: string | undefined;

    do {
      const res: any = await admin.users.list({
        domain: credentials.domain,
        maxResults: 200,
        pageToken,
        orderBy: "email",
        projection: "full",
      });

      for (const u of res.data.users ?? []) {
        users.push({
          externalId: u.id!,
          email: u.primaryEmail!,
          name: u.name?.fullName ?? u.primaryEmail!,
          role: u.isAdmin ? "admin" : "member",
          isSuspended: u.suspended ?? false,
          lastLoginAt: u.lastLoginTime ? new Date(u.lastLoginTime) : undefined,
          avatarUrl: u.thumbnailPhotoUrl ?? undefined,
          permissions: {
            groups: [], // populated separately if needed
          },
          metadata: {
            orgUnitPath: u.orgUnitPath ?? "/",
            isEnrolledIn2Sv: String(u.isEnrolledIn2Sv ?? false),
          },
        });
      }

      pageToken = res.data.nextPageToken ?? undefined;
    } while (pageToken);

    return {
      users,
      totalLicenses: users.length,
    };
  }

  async revokeAccess(
    credentials: Record<string, string>,
    externalUserId: string,
  ): Promise<AdapterRevokeResult> {
    try {
      const admin = this.getClient(credentials);
      // Suspend user rather than delete — safer, reversible
      await admin.users.update({
        userKey: externalUserId,
        requestBody: { suspended: true },
      });
      return { success: true, details: "User suspended in Google Workspace" };
    } catch (err: any) {
      return { success: false, error: err.message ?? "Revocation failed" };
    }
  }
}
```

### Deep-Dive 2: Access Review State Machine

```typescript
// src/server/services/review-machine.ts

/**
 * Access Review Campaign State Machine
 *
 * States: draft → active → completed | cancelled
 *
 * - draft: Review created, scope defined, items not yet generated
 * - active: Items generated, reviewers assigned, decisions in progress
 * - completed: All items decided (or deadline reached + auto-complete)
 * - cancelled: Review cancelled before completion
 */

// ---------------------------------------------------------------------------
// Campaign State Transitions
// ---------------------------------------------------------------------------
const CAMPAIGN_TRANSITIONS: Record<string, string[]> = {
  draft: ["active", "cancelled"],
  active: ["completed", "cancelled"],
  completed: [],  // terminal
  cancelled: [],  // terminal
};

export function canTransitionCampaign(from: string, to: string): boolean {
  return CAMPAIGN_TRANSITIONS[from]?.includes(to) ?? false;
}

/**
 * Review Item State Machine
 *
 * States: pending → approved | revoked | flagged
 *
 * - pending: Awaiting reviewer decision
 * - approved: Access confirmed as appropriate
 * - revoked: Access should be removed (triggers adapter revocation)
 * - flagged: Needs further investigation / escalation
 */
const ITEM_TRANSITIONS: Record<string, string[]> = {
  pending: ["approved", "revoked", "flagged"],
  approved: ["revoked", "flagged"],  // can change mind before campaign closes
  revoked: ["approved"],             // undo revoke before execution
  flagged: ["approved", "revoked"],  // resolve flag
};

export function canTransitionItem(from: string, to: string): boolean {
  return ITEM_TRANSITIONS[from]?.includes(to) ?? false;
}

// ---------------------------------------------------------------------------
// Campaign Lifecycle Operations
// ---------------------------------------------------------------------------
import { db } from "@/server/db";
import {
  accessReviews,
  reviewItems,
  appAccess,
  managedUsers,
  apps,
  members,
  auditLog,
} from "@/server/db/schema";
import { eq, and, inArray, sql } from "drizzle-orm";
import { getAdapter } from "@/server/adapters/registry";
import { decrypt } from "@/server/integrations/encryption";
import { TRPCError } from "@trpc/server";

interface StartCampaignResult {
  reviewId: string;
  totalItems: number;
}

/**
 * Activate a draft review campaign:
 * 1. Resolve scope to specific app-access records
 * 2. Generate one reviewItem per app-access in scope
 * 3. Assign reviewers (manager of each user, or app owner as fallback)
 * 4. Transition campaign to "active"
 */
export async function startCampaign(
  reviewId: string,
  orgId: string,
): Promise<StartCampaignResult> {
  const [review] = await db
    .select()
    .from(accessReviews)
    .where(and(eq(accessReviews.id, reviewId), eq(accessReviews.orgId, orgId)))
    .limit(1);

  if (!review) throw new TRPCError({ code: "NOT_FOUND" });
  if (!canTransitionCampaign(review.status, "active")) {
    throw new TRPCError({
      code: "BAD_REQUEST",
      message: `Cannot start review in status: ${review.status}`,
    });
  }

  // Resolve scope → list of appAccess IDs to review
  const scope = review.scope as { appIds?: string[]; departments?: string[]; includeAll?: boolean };
  let accessQuery = db
    .select({
      accessId: appAccess.id,
      userId: appAccess.userId,
      appId: appAccess.appId,
    })
    .from(appAccess)
    .innerJoin(managedUsers, eq(appAccess.userId, managedUsers.id))
    .where(
      and(
        eq(appAccess.orgId, orgId),
        eq(appAccess.status, "active"),
      ),
    );

  // Apply scope filters
  if (!scope.includeAll && scope.appIds?.length) {
    accessQuery = accessQuery.where(inArray(appAccess.appId, scope.appIds)) as any;
  }
  if (!scope.includeAll && scope.departments?.length) {
    accessQuery = accessQuery.where(
      inArray(managedUsers.department, scope.departments),
    ) as any;
  }

  const accessRecords = await accessQuery;

  // Look up app owners and user managers for reviewer assignment
  const appOwnerMap = new Map<string, string>();
  const appList = await db
    .select({ id: apps.id, ownerMemberId: apps.ownerMemberId })
    .from(apps)
    .where(eq(apps.orgId, orgId));
  for (const app of appList) {
    if (app.ownerMemberId) appOwnerMap.set(app.id, app.ownerMemberId);
  }

  const userManagerMap = new Map<string, string>();
  const userList = await db
    .select({ id: managedUsers.id, managerId: managedUsers.managerId, email: managedUsers.email })
    .from(managedUsers)
    .where(eq(managedUsers.orgId, orgId));
  for (const user of userList) {
    if (user.managerId) userManagerMap.set(user.id, user.managerId);
  }

  // Look up member IDs for managers (managedUser -> member mapping by email)
  const memberLookup = new Map<string, string>();
  const memberList = await db
    .select({ id: members.id, email: members.email })
    .from(members)
    .where(eq(members.orgId, orgId));
  for (const m of memberList) memberLookup.set(m.email, m.id);

  // Generate review items
  const itemValues = accessRecords.map((ar) => {
    // Reviewer priority: user's manager (if they're a member) → app owner → null
    let reviewerMemberId: string | null = null;
    const managerId = userManagerMap.get(ar.userId);
    if (managerId) {
      const managerUser = userList.find((u) => u.id === managerId);
      if (managerUser) reviewerMemberId = memberLookup.get(managerUser.email) ?? null;
    }
    if (!reviewerMemberId) {
      reviewerMemberId = appOwnerMap.get(ar.appId) ?? null;
    }

    return {
      orgId,
      reviewId,
      appAccessId: ar.accessId,
      reviewerMemberId,
      decision: "pending" as const,
    };
  });

  if (itemValues.length > 0) {
    await db.insert(reviewItems).values(itemValues);
  }

  // Update campaign status
  await db.update(accessReviews).set({
    status: "active",
    startedAt: new Date(),
    totalItems: itemValues.length,
    completedItems: 0,
    updatedAt: new Date(),
  }).where(eq(accessReviews.id, reviewId));

  return { reviewId, totalItems: itemValues.length };
}

/**
 * Record a reviewer's decision on a single review item.
 * If decision is "revoked" and autoRevoke is enabled, trigger adapter revocation.
 */
export async function recordDecision(
  itemId: string,
  orgId: string,
  reviewerMemberId: string,
  decision: "approved" | "revoked" | "flagged",
  justification: string | null,
  autoRevoke: boolean,
): Promise<void> {
  const [item] = await db
    .select()
    .from(reviewItems)
    .where(and(eq(reviewItems.id, itemId), eq(reviewItems.orgId, orgId)))
    .limit(1);

  if (!item) throw new TRPCError({ code: "NOT_FOUND" });
  if (!canTransitionItem(item.decision, decision)) {
    throw new TRPCError({
      code: "BAD_REQUEST",
      message: `Cannot transition from ${item.decision} to ${decision}`,
    });
  }

  // Record the decision
  await db.update(reviewItems).set({
    decision,
    justification,
    reviewerMemberId,
    decidedAt: new Date(),
    updatedAt: new Date(),
  }).where(eq(reviewItems.id, itemId));

  // Update campaign completed count
  const [countResult] = await db
    .select({ count: sql<number>`count(*)` })
    .from(reviewItems)
    .where(
      and(
        eq(reviewItems.reviewId, item.reviewId),
        sql`${reviewItems.decision} != 'pending'`,
      ),
    );

  await db.update(accessReviews).set({
    completedItems: Number(countResult.count),
    updatedAt: new Date(),
  }).where(eq(accessReviews.id, item.reviewId));

  // Log the decision in audit log
  await db.insert(auditLog).values({
    orgId,
    actorType: "member",
    actorId: reviewerMemberId,
    action: "review_decision",
    targetType: "access",
    targetId: item.appAccessId,
    details: { after: { decision, justification } },
  });

  // Execute revocation if decision is "revoked" and auto-revoke is enabled
  if (decision === "revoked" && autoRevoke) {
    await executeRevocation(item.id, item.appAccessId, orgId);
  }
}

/**
 * Execute the actual revocation via the app adapter.
 * Updates appAccess status and logs the result.
 */
async function executeRevocation(
  reviewItemId: string,
  appAccessId: string,
  orgId: string,
): Promise<void> {
  const [access] = await db
    .select({
      id: appAccess.id,
      appId: appAccess.appId,
      userId: appAccess.userId,
      externalId: appAccess.externalId,
      role: appAccess.role,
    })
    .from(appAccess)
    .where(eq(appAccess.id, appAccessId))
    .limit(1);

  if (!access) return;

  const [app] = await db
    .select()
    .from(apps)
    .where(eq(apps.id, access.appId))
    .limit(1);

  if (!app) return;

  try {
    const adapter = getAdapter(app.adapterType);
    const creds = decryptCredentials(app.credentials as Record<string, string>);
    const result = await adapter.revokeAccess(creds, access.externalId!);

    if (result.success) {
      // Update access status to revoked
      await db.update(appAccess).set({
        status: "revoked",
        updatedAt: new Date(),
      }).where(eq(appAccess.id, appAccessId));

      // Mark review item as executed
      await db.update(reviewItems).set({
        revokeExecuted: true,
        revokeExecutedAt: new Date(),
      }).where(eq(reviewItems.id, reviewItemId));

      // Audit log
      await db.insert(auditLog).values({
        orgId,
        actorType: "system",
        actorId: "review-revocation",
        action: "access_revoked",
        targetType: "access",
        targetId: appAccessId,
        targetUserId: access.userId,
        appId: access.appId,
        details: { reason: "review_revocation", after: { status: "revoked" } },
      });
    } else {
      await db.update(reviewItems).set({
        revokeError: result.error ?? "Unknown error",
      }).where(eq(reviewItems.id, reviewItemId));
    }
  } catch (err: any) {
    await db.update(reviewItems).set({
      revokeError: err.message ?? "Adapter error during revocation",
    }).where(eq(reviewItems.id, reviewItemId));
  }
}

function decryptCredentials(
  encrypted: Record<string, string>,
): Record<string, string> {
  const decrypted: Record<string, string> = {};
  for (const [key, value] of Object.entries(encrypted)) {
    if (["accessToken", "refreshToken", "apiKey"].includes(key) && value) {
      decrypted[key] = decrypt(value);
    } else {
      decrypted[key] = value;
    }
  }
  return decrypted;
}
```

### Deep-Dive 3: Unified Access View Aggregation

```typescript
// src/server/services/unified-access.ts

/**
 * The Unified Access View is the core value proposition of PermitFlow.
 * It aggregates access data across all connected apps and provides:
 *
 * 1. User → Apps view: "John has access to 23 apps: [list with roles]"
 * 2. App → Users view: "GitHub has 45 users: [list with roles]"
 * 3. Anomaly detection: stale access, orphaned accounts, excessive permissions
 * 4. Department-level aggregation for team access overview
 */

import { db } from "@/server/db";
import {
  appAccess,
  apps,
  managedUsers,
  organizations,
} from "@/server/db/schema";
import { eq, and, sql, lt, isNull, desc, asc, or, count } from "drizzle-orm";

// ---------------------------------------------------------------------------
// Types for the unified view
// ---------------------------------------------------------------------------
export interface UserAccessSummary {
  userId: string;
  userName: string;
  email: string;
  department: string | null;
  employmentStatus: string;
  totalApps: number;
  adminApps: number;
  lastActivityAt: Date | null;       // most recent login across all apps
  staleApps: number;                  // apps with no login in threshold days
  accessRecords: Array<{
    accessId: string;
    appId: string;
    appName: string;
    appIcon: string | null;
    role: string;
    lastLoginAt: Date | null;
    grantedAt: Date;
    source: string;
    isStale: boolean;
    daysSinceLogin: number | null;
  }>;
}

export interface AppAccessSummary {
  appId: string;
  appName: string;
  appIcon: string | null;
  category: string | null;
  totalUsers: number;
  adminUsers: number;
  staleUsers: number;
  licenseCount: number | null;
  utilizationPercent: number | null;
  accessRecords: Array<{
    accessId: string;
    userId: string;
    userName: string;
    email: string;
    department: string | null;
    role: string;
    lastLoginAt: Date | null;
    grantedAt: Date;
    isStale: boolean;
  }>;
}

export interface AccessAnomaly {
  type: "stale_access" | "orphaned_account" | "excessive_admin" | "no_login_ever";
  severity: "low" | "medium" | "high";
  userId: string;
  userName: string;
  email: string;
  appId: string;
  appName: string;
  role: string;
  description: string;
  daysSinceLogin: number | null;
  accessId: string;
}

// ---------------------------------------------------------------------------
// User → Apps aggregation
// ---------------------------------------------------------------------------
export async function getUserAccessSummaries(
  orgId: string,
  options: {
    search?: string;
    department?: string;
    employmentStatus?: string;
    sortBy?: "name" | "totalApps" | "lastActivity" | "staleApps";
    sortOrder?: "asc" | "desc";
    limit?: number;
    offset?: number;
  } = {},
): Promise<{ users: UserAccessSummary[]; total: number }> {
  const staleThresholdDays = 90; // TODO: pull from org settings

  // Build the base query for counting total
  const countQuery = db
    .select({ total: count() })
    .from(managedUsers)
    .where(
      and(
        eq(managedUsers.orgId, orgId),
        options.department ? eq(managedUsers.department, options.department) : undefined,
        options.employmentStatus
          ? eq(managedUsers.employmentStatus, options.employmentStatus)
          : undefined,
        options.search
          ? or(
              sql`${managedUsers.name} ILIKE ${"%" + options.search + "%"}`,
              sql`${managedUsers.email} ILIKE ${"%" + options.search + "%"}`,
            )
          : undefined,
      ),
    );

  const [{ total }] = await countQuery;

  // Main query: users with their aggregated access data
  const usersWithAccess = await db
    .select({
      userId: managedUsers.id,
      userName: managedUsers.name,
      email: managedUsers.email,
      department: managedUsers.department,
      employmentStatus: managedUsers.employmentStatus,
      totalApps: sql<number>`count(DISTINCT ${appAccess.appId})`,
      adminApps: sql<number>`count(DISTINCT CASE WHEN ${appAccess.role} = 'admin' THEN ${appAccess.appId} END)`,
      lastActivityAt: sql<Date>`max(${appAccess.lastLoginAt})`,
      staleApps: sql<number>`count(DISTINCT CASE
        WHEN ${appAccess.lastLoginAt} < now() - interval '${sql.raw(String(staleThresholdDays))} days'
        OR ${appAccess.lastLoginAt} IS NULL
        THEN ${appAccess.appId} END)`,
    })
    .from(managedUsers)
    .leftJoin(
      appAccess,
      and(
        eq(appAccess.userId, managedUsers.id),
        eq(appAccess.status, "active"),
      ),
    )
    .where(
      and(
        eq(managedUsers.orgId, orgId),
        options.department ? eq(managedUsers.department, options.department) : undefined,
        options.employmentStatus
          ? eq(managedUsers.employmentStatus, options.employmentStatus)
          : undefined,
        options.search
          ? or(
              sql`${managedUsers.name} ILIKE ${"%" + options.search + "%"}`,
              sql`${managedUsers.email} ILIKE ${"%" + options.search + "%"}`,
            )
          : undefined,
      ),
    )
    .groupBy(managedUsers.id)
    .orderBy(
      options.sortBy === "totalApps"
        ? (options.sortOrder === "asc" ? asc(sql`count(DISTINCT ${appAccess.appId})`) : desc(sql`count(DISTINCT ${appAccess.appId})`))
        : options.sortBy === "lastActivity"
          ? (options.sortOrder === "asc" ? asc(sql`max(${appAccess.lastLoginAt})`) : desc(sql`max(${appAccess.lastLoginAt})`))
          : (options.sortOrder === "asc" ? asc(managedUsers.name) : desc(managedUsers.name)),
    )
    .limit(options.limit ?? 50)
    .offset(options.offset ?? 0);

  return { users: usersWithAccess as unknown as UserAccessSummary[], total };
}

// ---------------------------------------------------------------------------
// Anomaly Detection
// ---------------------------------------------------------------------------
export async function detectAccessAnomalies(
  orgId: string,
  staleThresholdDays: number = 90,
): Promise<AccessAnomaly[]> {
  const anomalies: AccessAnomaly[] = [];

  // 1. Stale access: active access with no login in threshold days
  const staleAccess = await db
    .select({
      accessId: appAccess.id,
      userId: managedUsers.id,
      userName: managedUsers.name,
      email: managedUsers.email,
      appId: apps.id,
      appName: apps.name,
      role: appAccess.role,
      lastLoginAt: appAccess.lastLoginAt,
    })
    .from(appAccess)
    .innerJoin(managedUsers, eq(appAccess.userId, managedUsers.id))
    .innerJoin(apps, eq(appAccess.appId, apps.id))
    .where(
      and(
        eq(appAccess.orgId, orgId),
        eq(appAccess.status, "active"),
        eq(managedUsers.employmentStatus, "active"),
        lt(
          appAccess.lastLoginAt,
          sql`now() - interval '${sql.raw(String(staleThresholdDays))} days'`,
        ),
      ),
    );

  for (const row of staleAccess) {
    const daysSince = row.lastLoginAt
      ? Math.floor((Date.now() - new Date(row.lastLoginAt).getTime()) / 86400000)
      : null;

    anomalies.push({
      type: "stale_access",
      severity: daysSince && daysSince > 180 ? "high" : "medium",
      userId: row.userId,
      userName: row.userName,
      email: row.email,
      appId: row.appId,
      appName: row.appName,
      role: row.role,
      description: `${row.userName} hasn't used ${row.appName} in ${daysSince} days`,
      daysSinceLogin: daysSince,
      accessId: row.accessId,
    });
  }

  // 2. No login ever: access records with null lastLoginAt
  const neverLoggedIn = await db
    .select({
      accessId: appAccess.id,
      userId: managedUsers.id,
      userName: managedUsers.name,
      email: managedUsers.email,
      appId: apps.id,
      appName: apps.name,
      role: appAccess.role,
      grantedAt: appAccess.grantedAt,
    })
    .from(appAccess)
    .innerJoin(managedUsers, eq(appAccess.userId, managedUsers.id))
    .innerJoin(apps, eq(appAccess.appId, apps.id))
    .where(
      and(
        eq(appAccess.orgId, orgId),
        eq(appAccess.status, "active"),
        isNull(appAccess.lastLoginAt),
        lt(appAccess.grantedAt, sql`now() - interval '30 days'`),
      ),
    );

  for (const row of neverLoggedIn) {
    anomalies.push({
      type: "no_login_ever",
      severity: "medium",
      userId: row.userId,
      userName: row.userName,
      email: row.email,
      appId: row.appId,
      appName: row.appName,
      role: row.role,
      description: `${row.userName} was granted ${row.appName} access but has never logged in`,
      daysSinceLogin: null,
      accessId: row.accessId,
    });
  }

  // 3. Orphaned accounts: offboarded users with active access
  const orphaned = await db
    .select({
      accessId: appAccess.id,
      userId: managedUsers.id,
      userName: managedUsers.name,
      email: managedUsers.email,
      appId: apps.id,
      appName: apps.name,
      role: appAccess.role,
    })
    .from(appAccess)
    .innerJoin(managedUsers, eq(appAccess.userId, managedUsers.id))
    .innerJoin(apps, eq(appAccess.appId, apps.id))
    .where(
      and(
        eq(appAccess.orgId, orgId),
        eq(appAccess.status, "active"),
        eq(managedUsers.employmentStatus, "offboarded"),
      ),
    );

  for (const row of orphaned) {
    anomalies.push({
      type: "orphaned_account",
      severity: "high",
      userId: row.userId,
      userName: row.userName,
      email: row.email,
      appId: row.appId,
      appName: row.appName,
      role: row.role,
      description: `${row.userName} is offboarded but still has active ${row.appName} access`,
      daysSinceLogin: null,
      accessId: row.accessId,
    });
  }

  // 4. Excessive admin: users with admin access to more than 5 apps
  const adminCounts = await db
    .select({
      userId: managedUsers.id,
      userName: managedUsers.name,
      email: managedUsers.email,
      adminCount: sql<number>`count(*)`,
    })
    .from(appAccess)
    .innerJoin(managedUsers, eq(appAccess.userId, managedUsers.id))
    .where(
      and(
        eq(appAccess.orgId, orgId),
        eq(appAccess.status, "active"),
        eq(appAccess.role, "admin"),
      ),
    )
    .groupBy(managedUsers.id, managedUsers.name, managedUsers.email)
    .having(sql`count(*) > 5`);

  for (const row of adminCounts) {
    anomalies.push({
      type: "excessive_admin",
      severity: "medium",
      userId: row.userId,
      userName: row.userName,
      email: row.email,
      appId: "",
      appName: "",
      role: "admin",
      description: `${row.userName} has admin access to ${row.adminCount} apps`,
      daysSinceLogin: null,
      accessId: "",
    });
  }

  // Sort: high severity first, then by type
  const severityOrder = { high: 0, medium: 1, low: 2 };
  anomalies.sort((a, b) => severityOrder[a.severity] - severityOrder[b.severity]);

  return anomalies;
}
```

### Deep-Dive 4: Scheduled Sync Orchestrator

```typescript
// src/server/sync/orchestrator.ts

/**
 * Sync Orchestrator — BullMQ-based scheduled sync for all connected apps.
 *
 * Architecture:
 * 1. A repeatable "sync-scheduler" job runs every 15 minutes
 * 2. It checks which apps are due for a sync based on their syncFrequency
 * 3. For each due app, it enqueues an individual "sync-app" job
 * 4. The sync-app worker: fetches current state → diffs → updates DB → logs changes
 * 5. All changes are recorded in the audit log for compliance
 */

import { Queue, Worker, Job } from "bullmq";
import { Redis } from "ioredis";
import { db } from "@/server/db";
import {
  apps,
  appAccess,
  managedUsers,
  syncJobs,
  auditLog,
} from "@/server/db/schema";
import { eq, and, lt, sql, or } from "drizzle-orm";
import { getAdapter } from "@/server/adapters/registry";
import { decrypt } from "@/server/integrations/encryption";
import type { AdapterUser } from "@/server/adapters/types";

// ---------------------------------------------------------------------------
// Redis + Queue setup
// ---------------------------------------------------------------------------
const redis = new Redis(process.env.UPSTASH_REDIS_URL!, {
  maxRetriesPerRequest: null,
});

export const syncQueue = new Queue("app-sync", {
  connection: redis,
  defaultJobOptions: {
    attempts: 3,
    backoff: { type: "exponential", delay: 5000 },
    removeOnComplete: { age: 86400 },
    removeOnFail: { age: 604800 },
  },
});

// ---------------------------------------------------------------------------
// Sync Scheduler — enqueues sync jobs for apps that are due
// ---------------------------------------------------------------------------
export async function registerScheduler(): Promise<void> {
  // Remove any existing repeatable job to avoid duplicates
  const existing = await syncQueue.getRepeatableJobs();
  for (const job of existing) {
    if (job.name === "sync-scheduler") {
      await syncQueue.removeRepeatableByKey(job.key);
    }
  }

  // Run every 15 minutes
  await syncQueue.add(
    "sync-scheduler",
    {},
    {
      repeat: { every: 15 * 60 * 1000 },
      jobId: "sync-scheduler-repeatable",
    },
  );
}

async function handleSchedulerTick(): Promise<void> {
  const now = new Date();

  // Find all apps that are due for a sync
  const dueApps = await db
    .select({
      id: apps.id,
      orgId: apps.orgId,
      adapterType: apps.adapterType,
      name: apps.name,
      syncFrequency: apps.syncFrequency,
      lastSyncedAt: apps.lastSyncedAt,
    })
    .from(apps)
    .where(
      and(
        sql`${apps.syncStatus} != 'syncing'`, // not currently syncing
        or(
          // Never synced
          sql`${apps.lastSyncedAt} IS NULL`,
          // Hourly: last sync > 1 hour ago
          and(
            eq(apps.syncFrequency, "hourly"),
            lt(apps.lastSyncedAt, sql`now() - interval '1 hour'`),
          ),
          // Daily: last sync > 24 hours ago
          and(
            eq(apps.syncFrequency, "daily"),
            lt(apps.lastSyncedAt, sql`now() - interval '24 hours'`),
          ),
          // Weekly: last sync > 7 days ago
          and(
            eq(apps.syncFrequency, "weekly"),
            lt(apps.lastSyncedAt, sql`now() - interval '7 days'`),
          ),
        ),
      ),
    );

  for (const app of dueApps) {
    // Create sync job record
    const [syncJob] = await db.insert(syncJobs).values({
      orgId: app.orgId,
      appId: app.id,
      status: "queued",
      triggeredBy: "schedule",
    }).returning();

    // Enqueue BullMQ job
    await syncQueue.add("sync-app", {
      syncJobId: syncJob.id,
      appId: app.id,
      orgId: app.orgId,
    });

    // Mark app as syncing
    await db.update(apps).set({ syncStatus: "syncing" }).where(eq(apps.id, app.id));
  }
}

// ---------------------------------------------------------------------------
// Sync Worker — processes individual app sync jobs
// ---------------------------------------------------------------------------
interface SyncAppJobData {
  syncJobId: string;
  appId: string;
  orgId: string;
}

export const syncWorker = new Worker<SyncAppJobData | {}>(
  "app-sync",
  async (job: Job) => {
    if (job.name === "sync-scheduler") {
      await handleSchedulerTick();
      return;
    }

    if (job.name === "sync-app") {
      await handleAppSync(job.data as SyncAppJobData);
      return;
    }
  },
  { connection: redis, concurrency: 3 },
);

async function handleAppSync(data: SyncAppJobData): Promise<void> {
  const startTime = Date.now();

  // Mark sync job as running
  await db.update(syncJobs).set({
    status: "running",
    startedAt: new Date(),
  }).where(eq(syncJobs.id, data.syncJobId));

  try {
    // Fetch app details and decrypt credentials
    const [app] = await db.select().from(apps).where(eq(apps.id, data.appId)).limit(1);
    if (!app) throw new Error(`App ${data.appId} not found`);

    const adapter = getAdapter(app.adapterType);
    const creds = decryptAllCredentials(app.credentials as Record<string, string>);

    // Fetch current state from the external app
    const syncResult = await adapter.syncUsers(creds);

    // Fetch current state from our database
    const existingAccess = await db
      .select()
      .from(appAccess)
      .where(and(eq(appAccess.appId, data.appId), eq(appAccess.status, "active")));

    // Diff: compare adapter users against existing records
    const diff = computeDiff(existingAccess, syncResult.users, data.orgId, data.appId);

    // Apply changes
    let added = 0, removed = 0, modified = 0;

    // Process new users (exist in adapter but not in our DB)
    for (const newUser of diff.toAdd) {
      // Resolve or create managed user
      let userId = await resolveOrCreateManagedUser(data.orgId, newUser);

      await db.insert(appAccess).values({
        orgId: data.orgId,
        appId: data.appId,
        userId,
        role: newUser.role,
        permissions: newUser.permissions ?? {},
        lastLoginAt: newUser.lastLoginAt ?? null,
        status: newUser.isSuspended ? "suspended" : "active",
        source: "discovered",
        externalId: newUser.externalId,
        grantedBy: "sync",
      }).onConflictDoNothing();
      added++;
    }

    // Process removed users (exist in our DB but not in adapter)
    for (const removedAccess of diff.toRemove) {
      await db.update(appAccess).set({
        status: "revoked",
        updatedAt: new Date(),
      }).where(eq(appAccess.id, removedAccess.id));
      removed++;
    }

    // Process modified users (role or permissions changed)
    for (const mod of diff.toModify) {
      await db.update(appAccess).set({
        role: mod.newRole,
        permissions: mod.newPermissions,
        lastLoginAt: mod.lastLoginAt ?? undefined,
        status: mod.isSuspended ? "suspended" : "active",
        updatedAt: new Date(),
      }).where(eq(appAccess.id, mod.accessId));
      modified++;
    }

    // Update app metadata
    await db.update(apps).set({
      syncStatus: "synced",
      lastSyncedAt: new Date(),
      lastSyncError: null,
      totalUsers: syncResult.users.filter((u) => !u.isSuspended).length,
      licenseCount: syncResult.totalLicenses ?? null,
      updatedAt: new Date(),
    }).where(eq(apps.id, data.appId));

    // Complete sync job
    const durationMs = Date.now() - startTime;
    await db.update(syncJobs).set({
      status: "completed",
      completedAt: new Date(),
      usersFound: syncResult.users.length,
      accessRecordsFound: syncResult.users.length,
      changesDetected: { added, removed, modified },
      durationMs,
    }).where(eq(syncJobs.id, data.syncJobId));

    // Audit log
    if (added > 0 || removed > 0 || modified > 0) {
      await db.insert(auditLog).values({
        orgId: data.orgId,
        actorType: "sync",
        actorId: "sync-orchestrator",
        action: "sync_completed",
        targetType: "app",
        targetId: data.appId,
        appId: data.appId,
        details: {
          syncStats: { added, removed, modified },
          after: { totalUsers: syncResult.users.length },
        },
      });
    }
  } catch (err: any) {
    const durationMs = Date.now() - startTime;

    await db.update(syncJobs).set({
      status: "failed",
      completedAt: new Date(),
      errorMessage: err.message ?? "Unknown sync error",
      durationMs,
    }).where(eq(syncJobs.id, data.syncJobId));

    await db.update(apps).set({
      syncStatus: "error",
      lastSyncError: err.message ?? "Sync failed",
      updatedAt: new Date(),
    }).where(eq(apps.id, data.appId));

    await db.insert(auditLog).values({
      orgId: data.orgId,
      actorType: "sync",
      actorId: "sync-orchestrator",
      action: "sync_failed",
      targetType: "app",
      targetId: data.appId,
      appId: data.appId,
      details: { errorMessage: err.message },
    });

    throw err; // Let BullMQ handle retries
  }
}

// ---------------------------------------------------------------------------
// Diff computation
// ---------------------------------------------------------------------------
interface DiffResult {
  toAdd: AdapterUser[];
  toRemove: Array<{ id: string; externalId: string | null }>;
  toModify: Array<{
    accessId: string;
    newRole: string;
    newPermissions: Record<string, unknown>;
    lastLoginAt?: Date;
    isSuspended?: boolean;
  }>;
}

function computeDiff(
  existing: Array<{ id: string; externalId: string | null; role: string; permissions: unknown }>,
  adapterUsers: AdapterUser[],
  orgId: string,
  appId: string,
): DiffResult {
  const existingByExternalId = new Map(
    existing.filter((e) => e.externalId).map((e) => [e.externalId!, e]),
  );
  const adapterByExternalId = new Map(
    adapterUsers.map((u) => [u.externalId, u]),
  );

  const toAdd: AdapterUser[] = [];
  const toRemove: Array<{ id: string; externalId: string | null }> = [];
  const toModify: DiffResult["toModify"] = [];

  // New users from adapter
  for (const [extId, adapterUser] of adapterByExternalId) {
    const existingRecord = existingByExternalId.get(extId);
    if (!existingRecord) {
      toAdd.push(adapterUser);
    } else if (existingRecord.role !== adapterUser.role) {
      toModify.push({
        accessId: existingRecord.id,
        newRole: adapterUser.role,
        newPermissions: adapterUser.permissions ?? {},
        lastLoginAt: adapterUser.lastLoginAt,
        isSuspended: adapterUser.isSuspended,
      });
    }
  }

  // Removed users (in DB but not in adapter)
  for (const [extId, existingRecord] of existingByExternalId) {
    if (!adapterByExternalId.has(extId)) {
      toRemove.push({ id: existingRecord.id, externalId: extId });
    }
  }

  return { toAdd, toRemove, toModify };
}

// ---------------------------------------------------------------------------
// Helper: resolve or create a managed user by email
// ---------------------------------------------------------------------------
async function resolveOrCreateManagedUser(
  orgId: string,
  adapterUser: AdapterUser,
): Promise<string> {
  const [existing] = await db
    .select({ id: managedUsers.id })
    .from(managedUsers)
    .where(and(eq(managedUsers.orgId, orgId), eq(managedUsers.email, adapterUser.email)))
    .limit(1);

  if (existing) return existing.id;

  const [created] = await db.insert(managedUsers).values({
    orgId,
    email: adapterUser.email,
    name: adapterUser.name,
    employmentStatus: adapterUser.isSuspended ? "offboarded" : "active",
    metadata: adapterUser.metadata ?? {},
  }).returning();

  return created.id;
}

function decryptAllCredentials(
  encrypted: Record<string, string>,
): Record<string, string> {
  const result: Record<string, string> = {};
  const encryptedKeys = ["accessToken", "refreshToken", "apiKey"];
  for (const [key, value] of Object.entries(encrypted)) {
    result[key] = encryptedKeys.includes(key) && value ? decrypt(value) : value;
  }
  return result;
}
```

---

## Performance Benchmarks

| Operation | Target | Method |
|-----------|--------|--------|
| App adapter sync (single app) | < 30s | API fetch + diff + DB update |
| Unified access view load | < 1s | AG Grid with server-side row model |
| Access review campaign start | < 5s | Scope resolution + item generation |
| Revocation execution | < 10s | Adapter API call + DB update + audit log |
| Audit log query (30 days) | < 500ms | Indexed by org_id + created_at |
| Anomaly detection scan | < 10s | Indexed queries for stale/orphaned/excessive |
| 1000+ user sync (single adapter) | < 60s | Paginated API fetch + batch upsert |
| Permission diff for 500 roles | < 5s | In-memory diff after adapter fetch |
| All list endpoints | Paginated | Default 50/page, max 200, cursor-based for audit log |
| Dashboard page load | < 1s | Indexed queries + React Query cache |

> **Large org performance targets:** Unified access view loads in < 3s for orgs with 1,000+ users and 20+ connected apps. AG Grid virtual scrolling handles 50,000+ access records. Sync operations for large apps (1,000+ users) complete within 5 minutes per adapter. Review campaigns with 500+ items render the review interface in < 2s.

---

## Post-MVP Roadmap

| ID | Feature | Description | Priority |
|----|---------|-------------|----------|
| F5 | Access Request Workflows | Self-service access request portal with approval chains, time-bound access with auto-expiry, and manager/app-owner approval routing. Schema already includes `accessRequests` table. | High |
| F6 | Shadow IT Discovery | Detect unsanctioned SaaS usage via browser extension or OAuth grant monitoring. Flag apps not in the `isSanctioned` registry. Alert security team on new shadow apps. | High |
| F7 | Compliance Reporting | SOC 2, ISO 27001, and SOX compliance report generation. Pre-built templates mapping access reviews and audit logs to compliance controls. Scheduled PDF export. | Medium |
| F8 | SCIM 2.0 Provisioning | Implement SCIM server (RFC 7644) for automated user provisioning/deprovisioning from identity providers. Supports Okta, Azure AD, OneLogin inbound provisioning. | Medium |
| F9 | HRIS Sync | Direct integration with BambooHR, Workday, Rippling for employee lifecycle events (hire, transfer, terminate). Auto-trigger access reviews on department changes. | Medium |
| F10 | Additional Adapters (15+) | Expand adapter catalog: Okta, Azure AD, Jira, Notion, Salesforce, Datadog, PagerDuty, Zoom, Dropbox, Box, ServiceNow, Zendesk. Community adapter SDK. | High |
| F11 | Offboarding Automation | One-click offboarding: revoke all access across connected apps, generate compliance certificate, notify stakeholders. Integrate with HRIS terminate event. | High |
| F12 | Role Mining & Optimization | ML-based role suggestion: analyze access patterns to recommend optimal role templates. Detect over-privileged users and suggest least-privilege alternatives. | Low |

### Versioned Roadmap

- **v1.1**: Self-service access request portal with approval workflows (spec F5), HRIS integration for auto-onboarding/offboarding (BambooHR, Rippling), 5 additional adapters (Microsoft 365, Okta, Azure AD, Jira, Salesforce)
- **v2**: Shadow IT discovery via OAuth token analysis and email forwarding rules (spec F6), SOC 2 and ISO 27001 compliance report generation (spec F7), SCIM server for real-time provisioning, SaaS spend analytics and license optimization, 50+ total adapters
