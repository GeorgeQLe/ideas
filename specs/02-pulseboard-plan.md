# 2. PulseBoard — Remote Team Energy Tracker

## Implementation Plan

**MVP Scope:** Web app daily check-in (1-5 energy scale with optional mood tags and optional note), Slack bot check-in via `/pulse` slash command and scheduled DM prompts, manager dashboard with team trend line and heatmap visualization, basic burnout risk alerts (3+ consecutive days with energy at or below 2), configurable alert rules per organization, weekly email digest with AI-generated summary and talking points (GPT-4o-mini), BullMQ scheduled jobs for daily reminders and weekly digest generation, Stripe billing with three tiers (Free up to 5 people, Team at $4/user/month, Enterprise at $8/user/month), Clerk-based auth with organization-scoped access, and Resend for transactional email delivery.

---

## Tech Decisions

| Layer | Technology | Notes |
|-------|-----------|-------|
| Framework | Next.js 14+ App Router | `src/` directory, TypeScript strict |
| API | tRPC v11 | superjson transformer, Clerk auth context |
| Database | Neon PostgreSQL | Serverless-compatible, no extensions required |
| ORM | Drizzle ORM | Push-based migrations, typed schema |
| Auth | Clerk | Organization-scoped, `orgId` from session |
| Billing | Stripe | Checkout Sessions + Customer Portal + Webhooks |
| AI | OpenAI GPT-4o-mini | Weekly summaries, talking points, temperature 0.3 |
| Queue | BullMQ + Redis (Upstash) | Cron-like scheduled jobs for reminders and digests |
| Slack | Slack Bolt SDK | Interactive modals, slash commands, scheduled DMs |
| Email | Resend | Weekly digests + alert notifications |
| UI | Tailwind CSS, Radix UI, Recharts, Framer Motion | Heatmap grid, trend charts, animated check-in |
| Hosting | Vercel (app), Upstash (Redis) | Serverless-compatible BullMQ via `@upstash/qstash` fallback |
| Monitoring | Sentry, Vercel Analytics | |

### Key Architecture Decisions

1. **Check-in data model — one row per user per day**: Each check-in is uniquely constrained on `(userId, date)` so a user can update their check-in throughout the day but never have duplicates. The `date` column is a plain `DATE` type (not timestamp) aligned to the user's configured timezone. Energy is stored as an integer 1-5, mood tags as a `JSONB` string array, and the optional note as encrypted text. The `source` column tracks whether the check-in came from `web`, `slack`, or `api`, enabling channel adoption metrics.

2. **Burnout detection as a computed risk score, not a simple flag**: Rather than a binary "burnout yes/no", the system computes a 0-100 risk score per member using a sliding window of the last 7 check-ins. The algorithm weighs consecutive low days (3+ days at energy level 1-2 triggers a critical alert), downward trend slope, and deviation from personal baseline. Alert rules are stored per-organization in the `alertRules` table, allowing managers to customize thresholds (e.g., "alert me if anyone scores below 2 for 2+ days" instead of the default 3). This avoids alert fatigue while catching genuine burnout signals.

3. **Slack Bolt SDK for interactive check-in modals**: Instead of basic Slack message buttons, PulseBoard uses Slack's Block Kit interactive modals opened via slash command (`/pulse`) or action buttons in scheduled DM messages. The modal presents five large emoji buttons for energy level, a multi-select for mood tags, and an optional text input for notes. This keeps the entire check-in flow inside Slack without redirecting to the web app. Emoji reactions on the DM prompt (mapping specific emoji to energy levels 1-5) provide an even faster path for users who prefer minimal interaction.

4. **BullMQ scheduled jobs for reminders and digests**: Two repeatable BullMQ jobs run on cron schedules. The daily reminder job runs every 15 minutes, queries for members whose configured check-in time has passed but who haven't submitted today, and sends Slack DMs or email nudges. The weekly digest job runs every Monday at 9 AM UTC, aggregates the prior week's check-ins per team, calls GPT-4o-mini to generate a narrative summary and 3-5 talking points for upcoming 1-on-1s, and emails the digest to team managers. Both jobs use Upstash Redis as the backing store and are designed to be idempotent (safe to retry on failure).

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
  date,
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
  plan: text("plan").default("free").notNull(), // free | team | enterprise
  stripeCustomerId: text("stripe_customer_id"),
  stripeSubscriptionId: text("stripe_subscription_id"),
  timezone: text("timezone").default("America/New_York").notNull(),
  defaultCheckInTime: text("default_check_in_time").default("09:00").notNull(), // HH:mm
  skipWeekends: boolean("skip_weekends").default(true).notNull(),
  settings: jsonb("settings").default({}).$type<{
    brandColor?: string;
    anonymousMode?: boolean;
    minTeamSizeForAnonymity?: number;
    onboardingCompleted?: boolean;
    digestDay?: string; // "monday" | "friday"
    digestEnabled?: boolean;
  }>(),
  createdAt: timestamp("created_at", { withTimezone: true }).defaultNow().notNull(),
  updatedAt: timestamp("updated_at", { withTimezone: true }).defaultNow().notNull(),
});

// ---------------------------------------------------------------------------
// Members (users within an organization)
// ---------------------------------------------------------------------------
export const members = pgTable(
  "members",
  {
    id: uuid("id").primaryKey().defaultRandom(),
    orgId: uuid("org_id")
      .references(() => organizations.id, { onDelete: "cascade" })
      .notNull(),
    clerkUserId: text("clerk_user_id").notNull(),
    teamId: uuid("team_id").references(() => teams.id, { onDelete: "set null" }),
    name: text("name").notNull(),
    email: text("email").notNull(),
    role: text("role").default("member").notNull(), // member | manager | admin
    timezone: text("timezone").default("America/New_York").notNull(),
    checkInTime: text("check_in_time"), // HH:mm override, null = use org default
    skipWeekends: boolean("skip_weekends"), // null = use org default
    slackUserId: text("slack_user_id"),
    status: text("status").default("active").notNull(), // active | paused | deactivated
    notifyVia: jsonb("notify_via").default(["email"]).$type<string[]>(), // ["email", "slack"]
    createdAt: timestamp("created_at", { withTimezone: true }).defaultNow().notNull(),
    updatedAt: timestamp("updated_at", { withTimezone: true }).defaultNow().notNull(),
  },
  (table) => [
    index("members_org_id_idx").on(table.orgId),
    index("members_clerk_user_id_idx").on(table.clerkUserId),
    index("members_team_id_idx").on(table.teamId),
    uniqueIndex("members_org_clerk_user_idx").on(table.orgId, table.clerkUserId),
    index("members_slack_user_id_idx").on(table.slackUserId),
  ]
);

// ---------------------------------------------------------------------------
// Teams
// ---------------------------------------------------------------------------
export const teams = pgTable(
  "teams",
  {
    id: uuid("id").primaryKey().defaultRandom(),
    orgId: uuid("org_id")
      .references(() => organizations.id, { onDelete: "cascade" })
      .notNull(),
    name: text("name").notNull(),
    managerMemberId: uuid("manager_member_id"), // FK to members.id, set after member creation
    anonymousMode: boolean("anonymous_mode").default(false).notNull(),
    slackChannelId: text("slack_channel_id"), // for posting team aggregates
    createdAt: timestamp("created_at", { withTimezone: true }).defaultNow().notNull(),
    updatedAt: timestamp("updated_at", { withTimezone: true }).defaultNow().notNull(),
  },
  (table) => [
    index("teams_org_id_idx").on(table.orgId),
  ]
);

// ---------------------------------------------------------------------------
// Check-Ins (one per user per day)
// ---------------------------------------------------------------------------
export const checkIns = pgTable(
  "check_ins",
  {
    id: uuid("id").primaryKey().defaultRandom(),
    orgId: uuid("org_id")
      .references(() => organizations.id, { onDelete: "cascade" })
      .notNull(),
    memberId: uuid("member_id")
      .references(() => members.id, { onDelete: "cascade" })
      .notNull(),
    teamId: uuid("team_id")
      .references(() => teams.id, { onDelete: "set null" }),
    date: date("date").notNull(), // DATE type, no timezone
    energy: integer("energy").notNull(), // 1-5
    moodTags: jsonb("mood_tags").default([]).$type<string[]>(),
    // predefined: stressed, motivated, bored, focused, overwhelmed, excited, anxious, calm
    note: text("note"), // optional free-text, visible to self + manager if opted in
    noteSharedWithManager: boolean("note_shared_with_manager").default(false).notNull(),
    source: text("source").default("web").notNull(), // web | slack | api
    createdAt: timestamp("created_at", { withTimezone: true }).defaultNow().notNull(),
    updatedAt: timestamp("updated_at", { withTimezone: true }).defaultNow().notNull(),
  },
  (table) => [
    uniqueIndex("check_ins_member_date_idx").on(table.memberId, table.date),
    index("check_ins_org_id_idx").on(table.orgId),
    index("check_ins_team_id_idx").on(table.teamId),
    index("check_ins_date_idx").on(table.date),
    index("check_ins_org_date_idx").on(table.orgId, table.date),
    index("check_ins_member_date_range_idx").on(table.memberId, table.date, table.energy),
  ]
);

// ---------------------------------------------------------------------------
// Alerts (burnout risk, team dip, low participation)
// ---------------------------------------------------------------------------
export const alerts = pgTable(
  "alerts",
  {
    id: uuid("id").primaryKey().defaultRandom(),
    orgId: uuid("org_id")
      .references(() => organizations.id, { onDelete: "cascade" })
      .notNull(),
    teamId: uuid("team_id")
      .references(() => teams.id, { onDelete: "cascade" }),
    memberId: uuid("member_id")
      .references(() => members.id, { onDelete: "cascade" }),
    ruleId: uuid("rule_id")
      .references(() => alertRules.id, { onDelete: "set null" }),
    type: text("type").notNull(),
    // individual_burnout | team_dip | low_participation
    severity: text("severity").default("warning").notNull(), // warning | critical
    title: text("title").notNull(),
    message: text("message").notNull(),
    riskScore: real("risk_score"), // 0-100 computed score
    metadata: jsonb("metadata").default({}).$type<{
      consecutiveLowDays?: number;
      avgEnergy?: number;
      previousAvg?: number;
      participationRate?: number;
      suggestedAction?: string;
    }>(),
    status: text("status").default("active").notNull(), // active | acknowledged | resolved
    acknowledgedAt: timestamp("acknowledged_at", { withTimezone: true }),
    acknowledgedBy: text("acknowledged_by"), // Clerk userId
    resolvedAt: timestamp("resolved_at", { withTimezone: true }),
    triggeredAt: timestamp("triggered_at", { withTimezone: true }).defaultNow().notNull(),
    createdAt: timestamp("created_at", { withTimezone: true }).defaultNow().notNull(),
  },
  (table) => [
    index("alerts_org_id_idx").on(table.orgId),
    index("alerts_team_id_idx").on(table.teamId),
    index("alerts_member_id_idx").on(table.memberId),
    index("alerts_status_idx").on(table.orgId, table.status),
    index("alerts_triggered_at_idx").on(table.triggeredAt),
  ]
);

// ---------------------------------------------------------------------------
// Alert Rules (configurable per organization)
// ---------------------------------------------------------------------------
export const alertRules = pgTable(
  "alert_rules",
  {
    id: uuid("id").primaryKey().defaultRandom(),
    orgId: uuid("org_id")
      .references(() => organizations.id, { onDelete: "cascade" })
      .notNull(),
    name: text("name").notNull(),
    type: text("type").notNull(),
    // individual_burnout | team_dip | low_participation
    enabled: boolean("enabled").default(true).notNull(),
    config: jsonb("config").notNull().$type<{
      // individual_burnout
      consecutiveDays?: number;     // default 3
      maxEnergyThreshold?: number;  // default 2
      // team_dip
      dropPercentage?: number;      // default 20 (percent)
      baselineDays?: number;        // default 14
      // low_participation
      minParticipationRate?: number; // default 50 (percent)
      lookbackDays?: number;        // default 7
    }>(),
    notifyRoles: jsonb("notify_roles").default(["manager"]).$type<string[]>(),
    // ["manager", "admin"]
    notifyChannels: jsonb("notify_channels").default(["email", "in_app"]).$type<string[]>(),
    // ["email", "slack", "in_app"]
    createdAt: timestamp("created_at", { withTimezone: true }).defaultNow().notNull(),
    updatedAt: timestamp("updated_at", { withTimezone: true }).defaultNow().notNull(),
  },
  (table) => [
    index("alert_rules_org_id_idx").on(table.orgId),
    index("alert_rules_type_idx").on(table.orgId, table.type),
  ]
);

// ---------------------------------------------------------------------------
// Digests (weekly summaries)
// ---------------------------------------------------------------------------
export const digests = pgTable(
  "digests",
  {
    id: uuid("id").primaryKey().defaultRandom(),
    orgId: uuid("org_id")
      .references(() => organizations.id, { onDelete: "cascade" })
      .notNull(),
    teamId: uuid("team_id")
      .references(() => teams.id, { onDelete: "cascade" }),
    periodStart: date("period_start").notNull(),
    periodEnd: date("period_end").notNull(),
    avgEnergy: real("avg_energy"),
    participationRate: real("participation_rate"), // 0-100
    totalCheckIns: integer("total_check_ins").default(0).notNull(),
    energyDistribution: jsonb("energy_distribution").default({}).$type<{
      1: number; 2: number; 3: number; 4: number; 5: number;
    }>(),
    topMoodTags: jsonb("top_mood_tags").default([]).$type<string[]>(),
    trendDirection: text("trend_direction"), // up | down | stable
    aiSummary: text("ai_summary"),
    aiTalkingPoints: jsonb("ai_talking_points").default([]).$type<string[]>(),
    sentToEmails: jsonb("sent_to_emails").default([]).$type<string[]>(),
    sentAt: timestamp("sent_at", { withTimezone: true }),
    createdAt: timestamp("created_at", { withTimezone: true }).defaultNow().notNull(),
  },
  (table) => [
    index("digests_org_id_idx").on(table.orgId),
    index("digests_team_id_idx").on(table.teamId),
    index("digests_period_idx").on(table.orgId, table.periodStart),
  ]
);

// ---------------------------------------------------------------------------
// Slack Connections (per organization)
// ---------------------------------------------------------------------------
export const slackConnections = pgTable(
  "slack_connections",
  {
    id: uuid("id").primaryKey().defaultRandom(),
    orgId: uuid("org_id")
      .references(() => organizations.id, { onDelete: "cascade" })
      .notNull(),
    slackTeamId: text("slack_team_id").notNull(),
    slackTeamName: text("slack_team_name"),
    botAccessToken: text("bot_access_token").notNull(), // encrypted
    botUserId: text("bot_user_id"),
    installedByClerkUserId: text("installed_by_clerk_user_id"),
    scopes: jsonb("scopes").default([]).$type<string[]>(),
    status: text("status").default("active").notNull(), // active | revoked | error
    errorMessage: text("error_message"),
    createdAt: timestamp("created_at", { withTimezone: true }).defaultNow().notNull(),
    updatedAt: timestamp("updated_at", { withTimezone: true }).defaultNow().notNull(),
  },
  (table) => [
    index("slack_connections_org_id_idx").on(table.orgId),
    uniqueIndex("slack_connections_slack_team_idx").on(table.slackTeamId),
  ]
);

// ---------------------------------------------------------------------------
// Notification Log
// ---------------------------------------------------------------------------
export const notificationLog = pgTable(
  "notification_log",
  {
    id: uuid("id").primaryKey().defaultRandom(),
    orgId: uuid("org_id")
      .references(() => organizations.id, { onDelete: "cascade" })
      .notNull(),
    recipientMemberId: uuid("recipient_member_id")
      .references(() => members.id, { onDelete: "set null" }),
    recipientEmail: text("recipient_email"),
    type: text("type").notNull(),
    // check_in_reminder | alert_notification | digest_delivery | welcome
    channel: text("channel").default("email").notNull(), // email | slack | in_app
    status: text("status").default("pending").notNull(), // pending | sent | failed
    subject: text("subject"),
    sentAt: timestamp("sent_at", { withTimezone: true }),
    errorMessage: text("error_message"),
    metadata: jsonb("metadata").default({}).$type<{
      alertId?: string;
      digestId?: string;
      resendMessageId?: string;
      slackMessageTs?: string;
    }>(),
    createdAt: timestamp("created_at", { withTimezone: true }).defaultNow().notNull(),
  },
  (table) => [
    index("notification_log_org_id_idx").on(table.orgId),
    index("notification_log_member_id_idx").on(table.recipientMemberId),
    index("notification_log_status_idx").on(table.status),
    index("notification_log_type_idx").on(table.orgId, table.type),
  ]
);

// ---------------------------------------------------------------------------
// Relations
// ---------------------------------------------------------------------------
export const organizationsRelations = relations(organizations, ({ many }) => ({
  members: many(members),
  teams: many(teams),
  checkIns: many(checkIns),
  alerts: many(alerts),
  alertRules: many(alertRules),
  digests: many(digests),
  slackConnections: many(slackConnections),
  notificationLog: many(notificationLog),
}));

export const membersRelations = relations(members, ({ one, many }) => ({
  organization: one(organizations, {
    fields: [members.orgId],
    references: [organizations.id],
  }),
  team: one(teams, {
    fields: [members.teamId],
    references: [teams.id],
  }),
  checkIns: many(checkIns),
  alerts: many(alerts),
  notificationLog: many(notificationLog),
}));

export const teamsRelations = relations(teams, ({ one, many }) => ({
  organization: one(organizations, {
    fields: [teams.orgId],
    references: [organizations.id],
  }),
  members: many(members),
  checkIns: many(checkIns),
  alerts: many(alerts),
  digests: many(digests),
}));

export const checkInsRelations = relations(checkIns, ({ one }) => ({
  organization: one(organizations, {
    fields: [checkIns.orgId],
    references: [organizations.id],
  }),
  member: one(members, {
    fields: [checkIns.memberId],
    references: [members.id],
  }),
  team: one(teams, {
    fields: [checkIns.teamId],
    references: [teams.id],
  }),
}));

export const alertsRelations = relations(alerts, ({ one }) => ({
  organization: one(organizations, {
    fields: [alerts.orgId],
    references: [organizations.id],
  }),
  team: one(teams, {
    fields: [alerts.teamId],
    references: [teams.id],
  }),
  member: one(members, {
    fields: [alerts.memberId],
    references: [members.id],
  }),
  rule: one(alertRules, {
    fields: [alerts.ruleId],
    references: [alertRules.id],
  }),
}));

export const alertRulesRelations = relations(alertRules, ({ one, many }) => ({
  organization: one(organizations, {
    fields: [alertRules.orgId],
    references: [organizations.id],
  }),
  alerts: many(alerts),
}));

export const digestsRelations = relations(digests, ({ one }) => ({
  organization: one(organizations, {
    fields: [digests.orgId],
    references: [organizations.id],
  }),
  team: one(teams, {
    fields: [digests.teamId],
    references: [teams.id],
  }),
}));

export const slackConnectionsRelations = relations(slackConnections, ({ one }) => ({
  organization: one(organizations, {
    fields: [slackConnections.orgId],
    references: [organizations.id],
  }),
}));

export const notificationLogRelations = relations(notificationLog, ({ one }) => ({
  organization: one(organizations, {
    fields: [notificationLog.orgId],
    references: [organizations.id],
  }),
  recipientMember: one(members, {
    fields: [notificationLog.recipientMemberId],
    references: [members.id],
  }),
}));

// ---------------------------------------------------------------------------
// Type Exports
// ---------------------------------------------------------------------------
export type Organization = typeof organizations.$inferSelect;
export type NewOrganization = typeof organizations.$inferInsert;
export type Member = typeof members.$inferSelect;
export type NewMember = typeof members.$inferInsert;
export type Team = typeof teams.$inferSelect;
export type NewTeam = typeof teams.$inferInsert;
export type CheckIn = typeof checkIns.$inferSelect;
export type NewCheckIn = typeof checkIns.$inferInsert;
export type Alert = typeof alerts.$inferSelect;
export type NewAlert = typeof alerts.$inferInsert;
export type AlertRule = typeof alertRules.$inferSelect;
export type NewAlertRule = typeof alertRules.$inferInsert;
export type Digest = typeof digests.$inferSelect;
export type NewDigest = typeof digests.$inferInsert;
export type SlackConnection = typeof slackConnections.$inferSelect;
export type NewSlackConnection = typeof slackConnections.$inferInsert;
export type NotificationLogRow = typeof notificationLog.$inferSelect;
export type NewNotificationLogRow = typeof notificationLog.$inferInsert;
```

### Initial Migration SQL

```sql
-- src/server/db/migrations/0000_init.sql

-- No extensions required for PulseBoard (no pgvector needed).
-- All tables are created by Drizzle push.

-- Additional indexes for common aggregate queries:

-- Fast date-range lookups for heatmap and trend queries
CREATE INDEX IF NOT EXISTS check_ins_org_team_date_idx
  ON check_ins (org_id, team_id, date);

-- Fast burnout detection: consecutive low-energy lookups per member
CREATE INDEX IF NOT EXISTS check_ins_member_energy_date_idx
  ON check_ins (member_id, date DESC, energy);

-- Active alert count per org (dashboard badge)
CREATE INDEX IF NOT EXISTS alerts_org_active_idx
  ON alerts (org_id) WHERE status = 'active';

-- Digest lookup for latest per team
CREATE INDEX IF NOT EXISTS digests_team_period_idx
  ON digests (team_id, period_end DESC);
```

---

## Architecture Deep-Dives

### 1. Burnout Detection Algorithm

```typescript
// src/server/services/burnout-detection.ts

import { db } from "@/server/db";
import { checkIns, members, alerts, alertRules, teams } from "@/server/db/schema";
import { eq, and, gte, lte, desc, sql, inArray } from "drizzle-orm";

interface RiskAssessment {
  memberId: string;
  memberName: string;
  teamId: string | null;
  riskScore: number;        // 0-100
  severity: "none" | "warning" | "critical";
  consecutiveLowDays: number;
  avgEnergy7d: number;
  personalBaseline: number; // 30-day avg
  trendSlope: number;       // negative = declining
  suggestedAction: string;
}

/**
 * Compute burnout risk scores for all active members in an organization.
 * Called by the daily alert-check BullMQ job.
 *
 * Algorithm:
 * 1. Consecutive low days (energy <= threshold): 0-40 points
 *    - 1 day: 5pts, 2 days: 15pts, 3 days: 30pts, 4+: 40pts
 * 2. Deviation from personal baseline: 0-30 points
 *    - (baseline - recent_avg) / baseline * 30, clamped to [0, 30]
 * 3. Downward trend slope: 0-20 points
 *    - Negative slope over 7 days, normalized
 * 4. Low absolute energy: 0-10 points
 *    - avg <= 2: 10pts, avg <= 2.5: 5pts
 */
export async function computeBurnoutRisks(
  orgId: string
): Promise<RiskAssessment[]> {
  const today = new Date().toISOString().split("T")[0];
  const sevenDaysAgo = new Date(Date.now() - 7 * 86400000)
    .toISOString().split("T")[0];
  const thirtyDaysAgo = new Date(Date.now() - 30 * 86400000)
    .toISOString().split("T")[0];

  // Fetch all active members
  const activeMembers = await db
    .select()
    .from(members)
    .where(
      and(eq(members.orgId, orgId), eq(members.status, "active"))
    );

  // Fetch all check-ins for the last 30 days for this org
  const recentCheckIns = await db
    .select()
    .from(checkIns)
    .where(
      and(
        eq(checkIns.orgId, orgId),
        gte(checkIns.date, thirtyDaysAgo),
        lte(checkIns.date, today)
      )
    )
    .orderBy(desc(checkIns.date));

  // Fetch org's burnout alert rule for thresholds
  const [burnoutRule] = await db
    .select()
    .from(alertRules)
    .where(
      and(
        eq(alertRules.orgId, orgId),
        eq(alertRules.type, "individual_burnout"),
        eq(alertRules.enabled, true)
      )
    )
    .limit(1);

  const maxEnergyThreshold = burnoutRule?.config?.maxEnergyThreshold ?? 2;
  const consecutiveDaysThreshold = burnoutRule?.config?.consecutiveDays ?? 3;

  const assessments: RiskAssessment[] = [];

  for (const member of activeMembers) {
    const memberCheckIns = recentCheckIns
      .filter((c) => c.memberId === member.id)
      .sort((a, b) => b.date.localeCompare(a.date));

    if (memberCheckIns.length === 0) continue;

    // --- Factor 1: Consecutive low days ---
    let consecutiveLow = 0;
    for (const ci of memberCheckIns) {
      if (ci.energy <= maxEnergyThreshold) {
        consecutiveLow++;
      } else {
        break;
      }
    }
    let consecutiveScore = 0;
    if (consecutiveLow >= 4) consecutiveScore = 40;
    else if (consecutiveLow === 3) consecutiveScore = 30;
    else if (consecutiveLow === 2) consecutiveScore = 15;
    else if (consecutiveLow === 1) consecutiveScore = 5;

    // --- Factor 2: Deviation from personal baseline ---
    const last7 = memberCheckIns.filter((c) => c.date >= sevenDaysAgo);
    const last30 = memberCheckIns;
    const avg7d = last7.reduce((s, c) => s + c.energy, 0) / (last7.length || 1);
    const avg30d = last30.reduce((s, c) => s + c.energy, 0) / (last30.length || 1);
    const deviationScore = avg30d > 0
      ? Math.min(30, Math.max(0, ((avg30d - avg7d) / avg30d) * 30))
      : 0;

    // --- Factor 3: Trend slope (linear regression over 7 days) ---
    let trendSlope = 0;
    if (last7.length >= 3) {
      const n = last7.length;
      const indices = last7.map((_, i) => i);
      const energies = last7.map((c) => c.energy);
      const sumX = indices.reduce((a, b) => a + b, 0);
      const sumY = energies.reduce((a, b) => a + b, 0);
      const sumXY = indices.reduce((a, x, i) => a + x * energies[i], 0);
      const sumX2 = indices.reduce((a, x) => a + x * x, 0);
      trendSlope = (n * sumXY - sumX * sumY) / (n * sumX2 - sumX * sumX);
    }
    // Negative slope = declining; normalize to 0-20 points
    const trendScore = trendSlope < 0
      ? Math.min(20, Math.abs(trendSlope) * 20)
      : 0;

    // --- Factor 4: Low absolute energy ---
    let absoluteScore = 0;
    if (avg7d <= 2) absoluteScore = 10;
    else if (avg7d <= 2.5) absoluteScore = 5;

    // --- Combine ---
    const riskScore = Math.min(
      100,
      Math.round(consecutiveScore + deviationScore + trendScore + absoluteScore)
    );

    const severity: RiskAssessment["severity"] =
      riskScore >= 60 ? "critical" :
      riskScore >= 35 ? "warning" :
      "none";

    const suggestedAction =
      severity === "critical"
        ? `Schedule a private 1-on-1 with ${member.name} — they've had ${consecutiveLow} low-energy days in a row.`
        : severity === "warning"
        ? `Keep an eye on ${member.name} — their energy is trending down this week.`
        : "";

    assessments.push({
      memberId: member.id,
      memberName: member.name,
      teamId: member.teamId,
      riskScore,
      severity,
      consecutiveLowDays: consecutiveLow,
      avgEnergy7d: Math.round(avg7d * 10) / 10,
      personalBaseline: Math.round(avg30d * 10) / 10,
      trendSlope: Math.round(trendSlope * 100) / 100,
      suggestedAction,
    });
  }

  return assessments;
}

/**
 * Run burnout detection and create alerts for members exceeding thresholds.
 * Idempotent: does not create duplicate alerts for the same day.
 */
export async function runBurnoutDetection(orgId: string): Promise<number> {
  const assessments = await computeBurnoutRisks(orgId);
  const today = new Date().toISOString().split("T")[0];
  let alertsCreated = 0;

  for (const assessment of assessments) {
    if (assessment.severity === "none") continue;

    // Check for existing active alert for this member today
    const [existing] = await db
      .select()
      .from(alerts)
      .where(
        and(
          eq(alerts.orgId, orgId),
          eq(alerts.memberId, assessment.memberId),
          eq(alerts.type, "individual_burnout"),
          eq(alerts.status, "active")
        )
      )
      .limit(1);

    if (existing) continue; // Already has an active alert

    await db.insert(alerts).values({
      orgId,
      teamId: assessment.teamId,
      memberId: assessment.memberId,
      type: "individual_burnout",
      severity: assessment.severity,
      title: `Burnout risk: ${assessment.memberName}`,
      message: assessment.suggestedAction,
      riskScore: assessment.riskScore,
      metadata: {
        consecutiveLowDays: assessment.consecutiveLowDays,
        avgEnergy: assessment.avgEnergy7d,
        previousAvg: assessment.personalBaseline,
        suggestedAction: assessment.suggestedAction,
      },
    });

    alertsCreated++;
  }

  return alertsCreated;
}
```

### 2. Slack Bolt SDK Interactive Check-In Flow

```typescript
// src/server/slack/app.ts

import { App, BlockAction, ViewSubmitAction } from "@slack/bolt";
import { db } from "@/server/db";
import {
  slackConnections,
  members,
  checkIns,
  organizations,
} from "@/server/db/schema";
import { eq, and } from "drizzle-orm";

// Initialize Slack Bolt app
const slackApp = new App({
  signingSecret: process.env.SLACK_SIGNING_SECRET!,
  clientId: process.env.SLACK_CLIENT_ID!,
  clientSecret: process.env.SLACK_CLIENT_SECRET!,
  stateSecret: process.env.SLACK_STATE_SECRET!,
  installerOptions: {
    stateVerification: true,
  },
  authorize: async ({ teamId }) => {
    // Look up stored bot token for this Slack workspace
    const [conn] = await db
      .select()
      .from(slackConnections)
      .where(eq(slackConnections.slackTeamId, teamId!))
      .limit(1);

    if (!conn || conn.status !== "active") {
      throw new Error(`No active Slack connection for team ${teamId}`);
    }

    return {
      botToken: decrypt(conn.botAccessToken),
      botId: conn.botUserId ?? undefined,
      botUserId: conn.botUserId ?? undefined,
    };
  },
});

const ENERGY_EMOJI_MAP: Record<string, number> = {
  "energy_1": 1,
  "energy_2": 2,
  "energy_3": 3,
  "energy_4": 4,
  "energy_5": 5,
};

const ENERGY_LABELS: Record<number, string> = {
  1: "Exhausted",
  2: "Low",
  3: "Okay",
  4: "Good",
  5: "Great",
};

const MOOD_TAG_OPTIONS = [
  { text: { type: "plain_text" as const, text: "Stressed" }, value: "stressed" },
  { text: { type: "plain_text" as const, text: "Motivated" }, value: "motivated" },
  { text: { type: "plain_text" as const, text: "Bored" }, value: "bored" },
  { text: { type: "plain_text" as const, text: "Focused" }, value: "focused" },
  { text: { type: "plain_text" as const, text: "Overwhelmed" }, value: "overwhelmed" },
  { text: { type: "plain_text" as const, text: "Excited" }, value: "excited" },
  { text: { type: "plain_text" as const, text: "Anxious" }, value: "anxious" },
  { text: { type: "plain_text" as const, text: "Calm" }, value: "calm" },
];

// ----- Slash Command: /pulse -----
slackApp.command("/pulse", async ({ command, ack, client }) => {
  await ack();

  // Resolve the member from their Slack user ID
  const member = await resolveMemberBySlackId(command.user_id, command.team_id);
  if (!member) {
    await client.chat.postEphemeral({
      channel: command.channel_id,
      user: command.user_id,
      text: "You're not connected to PulseBoard yet. Visit your PulseBoard settings to link your Slack account.",
    });
    return;
  }

  // Open the check-in modal
  await client.views.open({
    trigger_id: command.trigger_id,
    view: buildCheckInModal(member.id),
  });
});

// ----- Build the Block Kit modal -----
function buildCheckInModal(memberId: string) {
  return {
    type: "modal" as const,
    callback_id: "pulse_checkin_modal",
    private_metadata: JSON.stringify({ memberId }),
    title: { type: "plain_text" as const, text: "Daily Pulse Check-In" },
    submit: { type: "plain_text" as const, text: "Submit" },
    close: { type: "plain_text" as const, text: "Cancel" },
    blocks: [
      {
        type: "section",
        text: {
          type: "mrkdwn",
          text: "How's your energy today? Pick a level:",
        },
      },
      {
        type: "actions",
        block_id: "energy_selection",
        elements: [1, 2, 3, 4, 5].map((level) => ({
          type: "button" as const,
          text: {
            type: "plain_text" as const,
            text: `${["", "\u{1F534}", "\u{1F7E0}", "\u{1F7E1}", "\u{1F7E2}", "\u{1F535}"][level]} ${level} — ${ENERGY_LABELS[level]}`,
          },
          action_id: `energy_${level}`,
          value: String(level),
        })),
      },
      {
        type: "input",
        block_id: "mood_tags",
        optional: true,
        label: { type: "plain_text" as const, text: "Mood tags (optional)" },
        element: {
          type: "multi_static_select" as const,
          action_id: "mood_select",
          placeholder: { type: "plain_text" as const, text: "Select mood tags..." },
          options: MOOD_TAG_OPTIONS,
        },
      },
      {
        type: "input",
        block_id: "note_block",
        optional: true,
        label: { type: "plain_text" as const, text: "Any notes? (optional)" },
        element: {
          type: "plain_text_input" as const,
          action_id: "note_input",
          multiline: true,
          max_length: 500,
          placeholder: {
            type: "plain_text" as const,
            text: "What's on your mind today?",
          },
        },
      },
    ],
  };
}

// ----- Handle energy button clicks in the modal -----
for (const level of [1, 2, 3, 4, 5]) {
  slackApp.action(`energy_${level}`, async ({ ack, body, client }) => {
    await ack();
    const actionBody = body as BlockAction;
    const viewId = actionBody.view?.id;
    if (!viewId) return;

    // Update the modal to show the selected energy level as highlighted
    const metadata = JSON.parse(actionBody.view?.private_metadata ?? "{}");
    metadata.selectedEnergy = level;

    await client.views.update({
      view_id: viewId,
      view: {
        ...buildCheckInModal(metadata.memberId),
        private_metadata: JSON.stringify(metadata),
        blocks: [
          {
            type: "section",
            text: {
              type: "mrkdwn",
              text: `*Selected: ${["", "\u{1F534}", "\u{1F7E0}", "\u{1F7E1}", "\u{1F7E2}", "\u{1F535}"][level]} ${level} — ${ENERGY_LABELS[level]}*`,
            },
          },
          // Keep remaining blocks
          ...buildCheckInModal(metadata.memberId).blocks.slice(1),
        ],
      },
    });
  });
}

// ----- Handle modal submission -----
slackApp.view("pulse_checkin_modal", async ({ ack, view, body }) => {
  const metadata = JSON.parse(view.private_metadata ?? "{}");
  const memberId = metadata.memberId as string;
  const energy = metadata.selectedEnergy as number;

  if (!energy || energy < 1 || energy > 5) {
    await ack({
      response_action: "errors",
      errors: { energy_selection: "Please select an energy level before submitting." },
    });
    return;
  }

  await ack();

  const moodValues = view.state?.values?.mood_tags?.mood_select?.selected_options;
  const moodTags = moodValues?.map((o: { value: string }) => o.value) ?? [];
  const note = view.state?.values?.note_block?.note_input?.value ?? null;
  const today = new Date().toISOString().split("T")[0];

  // Look up member to get orgId and teamId
  const [member] = await db
    .select()
    .from(members)
    .where(eq(members.id, memberId))
    .limit(1);

  if (!member) return;

  // Upsert check-in (one per day per member)
  await db
    .insert(checkIns)
    .values({
      orgId: member.orgId,
      memberId: member.id,
      teamId: member.teamId,
      date: today,
      energy,
      moodTags,
      note,
      source: "slack",
    })
    .onConflictDoUpdate({
      target: [checkIns.memberId, checkIns.date],
      set: {
        energy,
        moodTags,
        note,
        source: "slack",
        updatedAt: new Date(),
      },
    });
});

// ----- Handle emoji reactions on DM prompts -----
const REACTION_ENERGY_MAP: Record<string, number> = {
  red_circle: 1,          // 1 - Exhausted
  large_orange_circle: 2, // 2 - Low
  large_yellow_circle: 3, // 3 - Okay
  large_green_circle: 4,  // 4 - Good
  large_blue_circle: 5,   // 5 - Great
};

slackApp.event("reaction_added", async ({ event }) => {
  const energy = REACTION_ENERGY_MAP[event.reaction];
  if (!energy) return;

  // Only process reactions on messages from our bot
  const member = await resolveMemberBySlackId(event.user, event.item.channel);
  if (!member) return;

  const today = new Date().toISOString().split("T")[0];
  const [memberRecord] = await db
    .select()
    .from(members)
    .where(eq(members.id, member.id))
    .limit(1);

  if (!memberRecord) return;

  await db
    .insert(checkIns)
    .values({
      orgId: memberRecord.orgId,
      memberId: memberRecord.id,
      teamId: memberRecord.teamId,
      date: today,
      energy,
      moodTags: [],
      source: "slack",
    })
    .onConflictDoUpdate({
      target: [checkIns.memberId, checkIns.date],
      set: { energy, source: "slack", updatedAt: new Date() },
    });
});

// ----- Helper: resolve member by Slack user ID -----
async function resolveMemberBySlackId(
  slackUserId: string,
  slackTeamId: string
): Promise<{ id: string } | null> {
  const [conn] = await db
    .select()
    .from(slackConnections)
    .where(eq(slackConnections.slackTeamId, slackTeamId))
    .limit(1);

  if (!conn) return null;

  const [member] = await db
    .select({ id: members.id })
    .from(members)
    .where(
      and(
        eq(members.orgId, conn.orgId),
        eq(members.slackUserId, slackUserId),
        eq(members.status, "active")
      )
    )
    .limit(1);

  return member ?? null;
}

function decrypt(encrypted: string): string {
  // Delegate to shared encryption utility
  const { decrypt: dec } = require("@/server/lib/encryption");
  return dec(encrypted);
}

export { slackApp };
```

### 3. Weekly Digest Generation with AI Summaries

```typescript
// src/server/jobs/weekly-digest.ts

import { Queue, Worker, Job } from "bullmq";
import { redis } from "@/server/queue/connection";
import { db } from "@/server/db";
import {
  organizations,
  teams,
  members,
  checkIns,
  digests,
  notificationLog,
} from "@/server/db/schema";
import { eq, and, gte, lte, sql, desc } from "drizzle-orm";
import OpenAI from "openai";
import { Resend } from "resend";

const openai = new OpenAI({ apiKey: process.env.OPENAI_API_KEY });
const resend = new Resend(process.env.RESEND_API_KEY);

interface DigestJobData {
  orgId: string;
  teamId: string;
  periodStart: string; // YYYY-MM-DD
  periodEnd: string;   // YYYY-MM-DD
}

// ----- Queue definition -----
export const digestQueue = new Queue<DigestJobData>("weekly-digest", {
  connection: redis,
  defaultJobOptions: {
    attempts: 3,
    backoff: { type: "exponential", delay: 5000 },
    removeOnComplete: { age: 604800 },  // 7 days
    removeOnFail: { age: 2592000 },     // 30 days
  },
});

// ----- Cron job: enqueue digest generation for all active teams -----
export const digestScheduler = new Queue("digest-scheduler", {
  connection: redis,
});

// Add repeatable job: every Monday at 9:00 AM UTC
await digestScheduler.add(
  "schedule-digests",
  {},
  {
    repeat: { pattern: "0 9 * * 1" }, // cron: 9 AM UTC every Monday
    jobId: "weekly-digest-scheduler",
  }
);

// Scheduler worker: fans out to individual team digest jobs
const schedulerWorker = new Worker(
  "digest-scheduler",
  async () => {
    const now = new Date();
    const periodEnd = new Date(now);
    periodEnd.setDate(periodEnd.getDate() - 1); // Yesterday (Sunday)
    const periodStart = new Date(periodEnd);
    periodStart.setDate(periodStart.getDate() - 6); // Monday of previous week

    const allTeams = await db
      .select({
        teamId: teams.id,
        orgId: teams.orgId,
      })
      .from(teams)
      .innerJoin(organizations, eq(teams.orgId, organizations.id))
      .where(
        sql`${organizations.settings}->>'digestEnabled' != 'false'`
      );

    for (const team of allTeams) {
      await digestQueue.add("generate-digest", {
        orgId: team.orgId,
        teamId: team.teamId,
        periodStart: periodStart.toISOString().split("T")[0],
        periodEnd: periodEnd.toISOString().split("T")[0],
      });
    }

    console.log(`Enqueued ${allTeams.length} digest jobs`);
  },
  { connection: redis }
);

// ----- Digest generation worker -----
const digestWorker = new Worker<DigestJobData>(
  "weekly-digest",
  async (job: Job<DigestJobData>) => {
    const { orgId, teamId, periodStart, periodEnd } = job.data;

    // Step 1: Aggregate check-in data for the period
    const weekCheckIns = await db
      .select()
      .from(checkIns)
      .where(
        and(
          eq(checkIns.orgId, orgId),
          eq(checkIns.teamId, teamId),
          gte(checkIns.date, periodStart),
          lte(checkIns.date, periodEnd)
        )
      );

    if (weekCheckIns.length === 0) return; // No data, skip

    // Step 2: Compute statistics
    const totalCheckIns = weekCheckIns.length;
    const avgEnergy =
      weekCheckIns.reduce((s, c) => s + c.energy, 0) / totalCheckIns;

    const teamMembers = await db
      .select()
      .from(members)
      .where(
        and(eq(members.orgId, orgId), eq(members.teamId, teamId), eq(members.status, "active"))
      );
    const expectedCheckIns = teamMembers.length * 5; // 5 weekdays
    const participationRate =
      expectedCheckIns > 0
        ? Math.round((totalCheckIns / expectedCheckIns) * 100)
        : 0;

    // Energy distribution
    const distribution = { 1: 0, 2: 0, 3: 0, 4: 0, 5: 0 };
    for (const ci of weekCheckIns) {
      distribution[ci.energy as keyof typeof distribution]++;
    }

    // Top mood tags
    const tagCounts: Record<string, number> = {};
    for (const ci of weekCheckIns) {
      for (const tag of (ci.moodTags ?? [])) {
        tagCounts[tag] = (tagCounts[tag] ?? 0) + 1;
      }
    }
    const topMoodTags = Object.entries(tagCounts)
      .sort((a, b) => b[1] - a[1])
      .slice(0, 5)
      .map(([tag]) => tag);

    // Compare to previous week for trend
    const prevStart = new Date(periodStart);
    prevStart.setDate(prevStart.getDate() - 7);
    const prevEnd = new Date(periodStart);
    prevEnd.setDate(prevEnd.getDate() - 1);

    const [prevWeekStats] = await db
      .select({
        avg: sql<number>`AVG(${checkIns.energy})`,
      })
      .from(checkIns)
      .where(
        and(
          eq(checkIns.orgId, orgId),
          eq(checkIns.teamId, teamId),
          gte(checkIns.date, prevStart.toISOString().split("T")[0]),
          lte(checkIns.date, prevEnd.toISOString().split("T")[0])
        )
      );

    const prevAvg = prevWeekStats?.avg ?? avgEnergy;
    const trendDirection =
      avgEnergy > prevAvg + 0.2 ? "up" :
      avgEnergy < prevAvg - 0.2 ? "down" :
      "stable";

    // Step 3: Generate AI summary and talking points
    const [team] = await db
      .select()
      .from(teams)
      .where(eq(teams.id, teamId))
      .limit(1);

    const aiPrompt = `You are an AI assistant helping a manager understand their team's weekly energy check-in data. Generate a brief, empathetic summary and actionable talking points.

TEAM: ${team?.name ?? "Team"}
PERIOD: ${periodStart} to ${periodEnd}
TOTAL CHECK-INS: ${totalCheckIns} (${participationRate}% participation)
AVERAGE ENERGY: ${avgEnergy.toFixed(1)} / 5.0
PREVIOUS WEEK AVERAGE: ${(prevAvg ?? 0).toFixed(1)} / 5.0
TREND: ${trendDirection}
ENERGY DISTRIBUTION: ${JSON.stringify(distribution)}
TOP MOOD TAGS: ${topMoodTags.join(", ") || "none reported"}
NUMBER OF TEAM MEMBERS: ${teamMembers.length}

LOW ENERGY DAYS: ${weekCheckIns.filter((c) => c.energy <= 2).length} check-ins at energy 1-2
HIGH ENERGY DAYS: ${weekCheckIns.filter((c) => c.energy >= 4).length} check-ins at energy 4-5

Return JSON with:
- "summary": A 2-3 sentence narrative summary of the week (empathetic tone, no jargon).
- "talking_points": Array of 3-5 specific, actionable talking points the manager can use in 1-on-1s this week. Reference the data where relevant. Be concrete, not generic.`;

    const aiResponse = await openai.chat.completions.create({
      model: "gpt-4o-mini",
      messages: [{ role: "user", content: aiPrompt }],
      temperature: 0.3,
      max_tokens: 500,
      response_format: { type: "json_object" },
    });

    const aiContent = aiResponse.choices[0]?.message?.content ?? "{}";
    let aiParsed: { summary?: string; talking_points?: string[] };
    try {
      aiParsed = JSON.parse(aiContent);
    } catch {
      aiParsed = {
        summary: `This week, ${team?.name ?? "the team"} averaged ${avgEnergy.toFixed(1)}/5 energy across ${totalCheckIns} check-ins.`,
        talking_points: ["Review individual energy trends", "Check in with anyone who scored low"],
      };
    }

    // Step 4: Save digest to database
    const [digest] = await db
      .insert(digests)
      .values({
        orgId,
        teamId,
        periodStart,
        periodEnd,
        avgEnergy: Math.round(avgEnergy * 10) / 10,
        participationRate,
        totalCheckIns,
        energyDistribution: distribution,
        topMoodTags,
        trendDirection,
        aiSummary: aiParsed.summary ?? null,
        aiTalkingPoints: aiParsed.talking_points ?? [],
      })
      .returning();

    // Step 5: Send digest email to team manager(s)
    const managers = teamMembers.filter(
      (m) => m.role === "manager" || m.role === "admin"
    );
    const sentToEmails: string[] = [];

    for (const manager of managers) {
      try {
        await resend.emails.send({
          from: "PulseBoard <digest@pulseboard.io>",
          to: manager.email,
          subject: `Weekly Pulse: ${team?.name ?? "Your Team"} — ${periodStart} to ${periodEnd}`,
          html: buildDigestEmail({
            managerName: manager.name,
            teamName: team?.name ?? "Your Team",
            periodStart,
            periodEnd,
            avgEnergy,
            participationRate,
            trendDirection,
            distribution,
            topMoodTags,
            summary: aiParsed.summary ?? "",
            talkingPoints: aiParsed.talking_points ?? [],
          }),
        });

        sentToEmails.push(manager.email);

        await db.insert(notificationLog).values({
          orgId,
          recipientMemberId: manager.id,
          recipientEmail: manager.email,
          type: "digest_delivery",
          channel: "email",
          status: "sent",
          subject: `Weekly Pulse: ${team?.name ?? "Your Team"}`,
          sentAt: new Date(),
          metadata: { digestId: digest.id },
        });
      } catch (error) {
        await db.insert(notificationLog).values({
          orgId,
          recipientMemberId: manager.id,
          recipientEmail: manager.email,
          type: "digest_delivery",
          channel: "email",
          status: "failed",
          errorMessage: error instanceof Error ? error.message : "Unknown",
          metadata: { digestId: digest.id },
        });
      }
    }

    // Update digest with sent info
    await db
      .update(digests)
      .set({ sentToEmails, sentAt: new Date() })
      .where(eq(digests.id, digest.id));
  },
  { connection: redis, concurrency: 3 }
);

// ----- Email template builder -----
function buildDigestEmail(opts: {
  managerName: string;
  teamName: string;
  periodStart: string;
  periodEnd: string;
  avgEnergy: number;
  participationRate: number;
  trendDirection: string;
  distribution: Record<number, number>;
  topMoodTags: string[];
  summary: string;
  talkingPoints: string[];
}): string {
  const trendEmoji =
    opts.trendDirection === "up" ? "&#x2197;" :
    opts.trendDirection === "down" ? "&#x2198;" :
    "&#x2192;";

  const talkingPointsHtml = opts.talkingPoints
    .map((tp) => `<li style="margin-bottom:8px;">${tp}</li>`)
    .join("");

  return `
    <div style="font-family:-apple-system,system-ui,sans-serif;max-width:600px;margin:0 auto;">
      <div style="background:#6366f1;padding:24px;border-radius:8px 8px 0 0;">
        <h1 style="color:white;margin:0;font-size:20px;">PulseBoard Weekly Digest</h1>
        <p style="color:rgba(255,255,255,0.8);margin:4px 0 0;font-size:14px;">${opts.teamName} &mdash; ${opts.periodStart} to ${opts.periodEnd}</p>
      </div>
      <div style="padding:24px;border:1px solid #e5e7eb;border-top:none;">
        <p>Hi ${opts.managerName},</p>
        <p>${opts.summary}</p>
        <div style="display:flex;gap:16px;margin:20px 0;">
          <div style="background:#f3f4f6;padding:16px;border-radius:8px;flex:1;text-align:center;">
            <div style="font-size:28px;font-weight:bold;color:#6366f1;">${opts.avgEnergy.toFixed(1)}</div>
            <div style="font-size:12px;color:#6b7280;">Avg Energy</div>
          </div>
          <div style="background:#f3f4f6;padding:16px;border-radius:8px;flex:1;text-align:center;">
            <div style="font-size:28px;font-weight:bold;color:#6366f1;">${opts.participationRate}%</div>
            <div style="font-size:12px;color:#6b7280;">Participation</div>
          </div>
          <div style="background:#f3f4f6;padding:16px;border-radius:8px;flex:1;text-align:center;">
            <div style="font-size:28px;">${trendEmoji}</div>
            <div style="font-size:12px;color:#6b7280;">Trend</div>
          </div>
        </div>
        ${opts.topMoodTags.length > 0 ? `<p style="color:#6b7280;font-size:14px;">Top mood tags: <strong>${opts.topMoodTags.join(", ")}</strong></p>` : ""}
        <h3 style="margin-top:24px;">Talking Points for 1-on-1s</h3>
        <ul style="padding-left:20px;">${talkingPointsHtml}</ul>
        <div style="margin-top:24px;text-align:center;">
          <a href="${process.env.NEXT_PUBLIC_APP_URL}/dashboard" style="background:#6366f1;color:white;padding:12px 24px;border-radius:6px;text-decoration:none;font-weight:500;">View Full Dashboard</a>
        </div>
        <p style="color:#9ca3af;font-size:12px;margin-top:24px;text-align:center;">PulseBoard &mdash; Helping remote teams thrive</p>
      </div>
    </div>
  `;
}
```

### 4. Heatmap Data Aggregation Query

```typescript
// src/server/services/heatmap.ts

import { db } from "@/server/db";
import { checkIns, members, teams } from "@/server/db/schema";
import { eq, and, gte, lte, sql, asc } from "drizzle-orm";

interface HeatmapCell {
  memberId: string;
  memberName: string;
  date: string;          // YYYY-MM-DD
  energy: number | null; // null = no check-in
}

interface HeatmapRow {
  memberId: string;
  memberName: string;
  cells: Record<string, number | null>; // date -> energy
  avgEnergy: number;
  checkInCount: number;
}

interface HeatmapData {
  rows: HeatmapRow[];
  dates: string[];        // ordered list of dates in range
  teamAvgByDate: Record<string, number>;
  overallAvg: number;
  participationByDate: Record<string, number>; // date -> percentage
}

/**
 * Build the heatmap grid: team members x dates.
 * Each cell contains the energy level (1-5) or null (no check-in).
 *
 * This is the core query behind the manager dashboard heatmap view.
 * It generates a complete grid including days with no check-ins.
 */
export async function getTeamHeatmap(opts: {
  orgId: string;
  teamId: string;
  startDate: string; // YYYY-MM-DD
  endDate: string;   // YYYY-MM-DD
}): Promise<HeatmapData> {
  const { orgId, teamId, startDate, endDate } = opts;

  // Step 1: Get all active team members
  const teamMembers = await db
    .select({
      id: members.id,
      name: members.name,
    })
    .from(members)
    .where(
      and(
        eq(members.orgId, orgId),
        eq(members.teamId, teamId),
        eq(members.status, "active")
      )
    )
    .orderBy(asc(members.name));

  if (teamMembers.length === 0) {
    return { rows: [], dates: [], teamAvgByDate: {}, overallAvg: 0, participationByDate: {} };
  }

  // Step 2: Generate the date range
  const dates: string[] = [];
  const current = new Date(startDate);
  const end = new Date(endDate);
  while (current <= end) {
    dates.push(current.toISOString().split("T")[0]);
    current.setDate(current.getDate() + 1);
  }

  // Step 3: Fetch all check-ins for the team in the date range (single query)
  const teamCheckIns = await db
    .select({
      memberId: checkIns.memberId,
      date: checkIns.date,
      energy: checkIns.energy,
    })
    .from(checkIns)
    .where(
      and(
        eq(checkIns.orgId, orgId),
        eq(checkIns.teamId, teamId),
        gte(checkIns.date, startDate),
        lte(checkIns.date, endDate)
      )
    );

  // Step 4: Build a lookup map: memberId:date -> energy
  const checkInMap = new Map<string, number>();
  for (const ci of teamCheckIns) {
    checkInMap.set(`${ci.memberId}:${ci.date}`, ci.energy);
  }

  // Step 5: Build heatmap rows (one per member)
  const rows: HeatmapRow[] = teamMembers.map((member) => {
    const cells: Record<string, number | null> = {};
    let totalEnergy = 0;
    let checkInCount = 0;

    for (const date of dates) {
      const energy = checkInMap.get(`${member.id}:${date}`) ?? null;
      cells[date] = energy;
      if (energy !== null) {
        totalEnergy += energy;
        checkInCount++;
      }
    }

    return {
      memberId: member.id,
      memberName: member.name,
      cells,
      avgEnergy: checkInCount > 0
        ? Math.round((totalEnergy / checkInCount) * 10) / 10
        : 0,
      checkInCount,
    };
  });

  // Step 6: Compute team averages and participation per date
  const teamAvgByDate: Record<string, number> = {};
  const participationByDate: Record<string, number> = {};
  let overallTotal = 0;
  let overallCount = 0;

  for (const date of dates) {
    let dayTotal = 0;
    let dayCount = 0;

    for (const member of teamMembers) {
      const energy = checkInMap.get(`${member.id}:${date}`);
      if (energy !== undefined) {
        dayTotal += energy;
        dayCount++;
      }
    }

    teamAvgByDate[date] = dayCount > 0
      ? Math.round((dayTotal / dayCount) * 10) / 10
      : 0;

    participationByDate[date] = teamMembers.length > 0
      ? Math.round((dayCount / teamMembers.length) * 100)
      : 0;

    overallTotal += dayTotal;
    overallCount += dayCount;
  }

  const overallAvg = overallCount > 0
    ? Math.round((overallTotal / overallCount) * 10) / 10
    : 0;

  return {
    rows,
    dates,
    teamAvgByDate,
    overallAvg,
    participationByDate,
  };
}

/**
 * Get trend data: rolling 7-day average for a team.
 * Used for the trend line chart on the manager dashboard.
 */
export async function getTeamTrend(opts: {
  orgId: string;
  teamId: string;
  days: number; // 30 or 90
}): Promise<Array<{ date: string; avg: number; count: number }>> {
  const endDate = new Date().toISOString().split("T")[0];
  const startDate = new Date(Date.now() - opts.days * 86400000)
    .toISOString().split("T")[0];

  const result = await db
    .select({
      date: checkIns.date,
      avg: sql<number>`ROUND(AVG(${checkIns.energy})::numeric, 2)`,
      count: sql<number>`COUNT(*)`,
    })
    .from(checkIns)
    .where(
      and(
        eq(checkIns.orgId, opts.orgId),
        eq(checkIns.teamId, opts.teamId),
        gte(checkIns.date, startDate),
        lte(checkIns.date, endDate)
      )
    )
    .groupBy(checkIns.date)
    .orderBy(asc(checkIns.date));

  return result.map((r) => ({
    date: r.date,
    avg: Number(r.avg),
    count: Number(r.count),
  }));
}

/**
 * SQL-only heatmap aggregation for large teams (50+ members).
 * Returns pre-aggregated data using a cross-join with generate_series.
 */
export async function getTeamHeatmapSQL(opts: {
  orgId: string;
  teamId: string;
  startDate: string;
  endDate: string;
}): Promise<HeatmapCell[]> {
  const result = await db.execute(sql`
    WITH date_range AS (
      SELECT generate_series(
        ${opts.startDate}::date,
        ${opts.endDate}::date,
        '1 day'::interval
      )::date AS date
    ),
    team_members AS (
      SELECT id, name
      FROM members
      WHERE org_id = ${opts.orgId}
        AND team_id = ${opts.teamId}
        AND status = 'active'
    ),
    grid AS (
      SELECT
        tm.id AS member_id,
        tm.name AS member_name,
        dr.date
      FROM team_members tm
      CROSS JOIN date_range dr
    )
    SELECT
      g.member_id,
      g.member_name,
      g.date::text,
      ci.energy
    FROM grid g
    LEFT JOIN check_ins ci
      ON ci.member_id = g.member_id
      AND ci.date = g.date
    ORDER BY g.member_name, g.date
  `);

  return (result.rows as any[]).map((r) => ({
    memberId: r.member_id,
    memberName: r.member_name,
    date: r.date,
    energy: r.energy ? Number(r.energy) : null,
  }));
}
```

---

## Phase Breakdown (5 weeks) -- Day-by-Day

### Phase 1: Scaffold + Database (Days 1-5)

**Day 1 -- Project scaffold + environment**
- `npx create-next-app@latest pulseboard --typescript --tailwind --app --src-dir`
- Install dependencies:
  ```bash
  npm install drizzle-orm @neondatabase/serverless @clerk/nextjs @trpc/server @trpc/client @trpc/next superjson zod openai stripe resend @slack/bolt bullmq ioredis framer-motion recharts
  ```
- Install dev dependencies:
  ```bash
  npm install -D drizzle-kit @types/node
  ```
- Configure `src/env.ts` with Zod validation:
  ```typescript
  import { z } from "zod";

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
    SLACK_CLIENT_ID: z.string().min(1),
    SLACK_CLIENT_SECRET: z.string().min(1),
    SLACK_SIGNING_SECRET: z.string().min(1),
    SLACK_STATE_SECRET: z.string().min(1),
    NEXT_PUBLIC_APP_URL: z.string().url(),
  });

  export const env = envSchema.parse(process.env);
  ```
- Create Neon database
- Configure `drizzle.config.ts` with Neon connection string

**Day 2 -- Database schema + Clerk auth**
- Write full Drizzle schema (as above) in `src/server/db/schema.ts`
- Create `src/server/db/index.ts` with Neon serverless driver:
  ```typescript
  import { neon } from "@neondatabase/serverless";
  import { drizzle } from "drizzle-orm/neon-http";
  import * as schema from "./schema";

  const sql = neon(process.env.DATABASE_URL!);
  export const db = drizzle(sql, { schema });
  ```
- Run `npx drizzle-kit push` to apply schema
- Run migration SQL for additional indexes
- Set up Clerk: `src/middleware.ts` with `clerkMiddleware()`, protect `/dashboard/*` routes
- Create `src/app/layout.tsx` with `<ClerkProvider>`
- Test: verify Clerk sign-in works, org creation works

**Day 3 -- tRPC foundation + org resolution**
- Create `src/server/trpc/trpc.ts`: init tRPC with superjson, auth middleware, org middleware
- Create `src/server/trpc/context.ts`: pull `auth()` from Clerk, inject `db`
- Create `src/app/api/trpc/[trpc]/route.ts`: Next.js route handler
- Create `src/lib/trpc/client.ts`: React Query client with superjson
- Create `src/lib/trpc/server.ts`: server-side caller
- Build plan-limit middleware:
  ```typescript
  const PLAN_LIMITS = {
    free: { maxMembers: 5, features: ["basic_checkin"] },
    team: { maxMembers: Infinity, features: ["basic_checkin", "mood_tags", "slack", "heatmap", "alerts", "digest"] },
    enterprise: { maxMembers: Infinity, features: ["basic_checkin", "mood_tags", "slack", "heatmap", "alerts", "digest", "anonymous", "sso", "api"] },
  } as const;
  ```
- Create `src/server/trpc/routers/_app.ts`: merge routers

**Day 4 -- Organization + Member tRPC routers**
- `src/server/trpc/routers/organization.ts`:
  - `getOrCreate`: find by clerkOrgId or create new with default alert rules
  - `update`: name, timezone, settings, defaultCheckInTime
  - `getSettings`: return org settings
- `src/server/trpc/routers/member.ts`:
  - `list`: all members for org with team info
  - `getMe`: current user's member record
  - `upsert`: create or update member from Clerk user data
  - `updateProfile`: timezone, checkInTime, skipWeekends, notifyVia
  - `invite`: send invite email via Resend
  - `deactivate`: set status to deactivated
- `src/server/trpc/routers/team.ts`:
  - `list`: all teams for org with member counts
  - `create`: name, managerMemberId
  - `update`: name, manager, anonymousMode
  - `delete`: reassign members to null team
  - `getById`: team with members list
- Seed default alert rules on org creation (3 rules: individual_burnout, team_dip, low_participation)

**Day 5 -- Base UI: layout, navigation, dashboard shell**
- Install UI deps:
  ```bash
  npm install @radix-ui/react-dialog @radix-ui/react-dropdown-menu @radix-ui/react-tabs @radix-ui/react-select @radix-ui/react-tooltip @radix-ui/react-popover @radix-ui/react-slider @radix-ui/react-switch class-variance-authority clsx tailwind-merge lucide-react
  ```
- Create shared UI components in `src/components/ui/`: button, card, badge, input, textarea, select, dialog, dropdown-menu, tabs, toast, skeleton, slider, switch
- Build app shell layout: `src/app/dashboard/layout.tsx`
  - Left sidebar: logo, nav links (Check-In, Dashboard, Teams, Alerts, Digests, Settings), org switcher, user menu
  - Main content area with header breadcrumb
- Create stub pages:
  - `src/app/dashboard/page.tsx` (manager overview)
  - `src/app/dashboard/checkin/page.tsx` (personal check-in)
  - `src/app/dashboard/teams/page.tsx` (team list)
  - `src/app/dashboard/teams/[id]/page.tsx` (team detail + heatmap)
  - `src/app/dashboard/alerts/page.tsx` (alert list)
  - `src/app/dashboard/digests/page.tsx` (digest history)
  - `src/app/dashboard/settings/page.tsx` (org settings)
  - `src/app/dashboard/settings/billing/page.tsx` (billing)
  - `src/app/dashboard/settings/integrations/page.tsx` (Slack connection)

---

### Phase 2: Check-In System (Days 6-10)

**Day 6 -- Check-in tRPC router + submission**
- `src/server/trpc/routers/checkin.ts`:
  - `submit`: accepts `{ energy: 1-5, moodTags?: string[], note?: string, noteSharedWithManager?: boolean }`, upserts check-in for today
  - `getToday`: return current user's check-in for today (or null)
  - `getMyHistory`: paginated personal check-in history (last 30/90 days)
  - `getTeamToday`: all check-ins for a team today (manager-only)
  - `getTeamRange`: check-ins for a team over date range (for heatmap + trends)
- Implement upsert logic using `onConflictDoUpdate` on the unique `(memberId, date)` constraint
- Date computation: resolve "today" using the member's configured timezone

**Day 7 -- Check-in UI: personal check-in page**
- Create `src/app/dashboard/checkin/page.tsx`:
  - Five large energy buttons with emoji labels (Framer Motion scale animation on hover/tap)
  - Optional mood tag chips below (multi-select, styled pills)
  - Optional note textarea that expands on click
  - "Skip today" link
  - Streak counter (consecutive check-in days)
  - Confirmation animation (Framer Motion check mark + confetti-like pulse)
  - Takes under 5 seconds for minimal check-in
- Create `src/components/checkin/energy-picker.tsx`: reusable energy button row
- Create `src/components/checkin/mood-tags.tsx`: clickable mood tag pills
- Create `src/components/checkin/streak-badge.tsx`: streak counter with flame icon

**Day 8 -- Personal history view**
- Create `src/app/dashboard/checkin/history/page.tsx`:
  - Calendar grid view with color-coded days (red-to-green gradient for 1-5)
  - 30-day trend line (Recharts `LineChart`)
  - Most common mood tags summary
  - Journal entries list (days where notes were added)
- Create `src/components/checkin/calendar-grid.tsx`: month-view calendar with colored cells
- Create `src/components/checkin/trend-chart.tsx`: Recharts line chart wrapper

**Day 9 -- Check-in API endpoint for external integrations**
- Create `src/app/api/checkin/route.ts`: REST endpoint for bot-submitted check-ins
  - POST accepts `{ slackUserId, energy, moodTags?, note? }` with API key auth
  - Resolves member by slackUserId, upserts check-in
  - Returns `{ success: true, checkIn: { ... } }`
- Create `src/server/lib/encryption.ts`:
  ```typescript
  import crypto from "crypto";

  const ALGORITHM = "aes-256-gcm";
  const IV_LENGTH = 16;

  export function encrypt(plaintext: string): string {
    const key = Buffer.from(process.env.SLACK_STATE_SECRET!, "hex");
    const iv = crypto.randomBytes(IV_LENGTH);
    const cipher = crypto.createCipheriv(ALGORITHM, key, iv);
    let encrypted = cipher.update(plaintext, "utf8", "hex");
    encrypted += cipher.final("hex");
    const authTag = cipher.getAuthTag();
    return iv.toString("hex") + ":" + authTag.toString("hex") + ":" + encrypted;
  }

  export function decrypt(ciphertext: string): string {
    const key = Buffer.from(process.env.SLACK_STATE_SECRET!, "hex");
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

**Day 10 -- BullMQ queue setup + daily reminder job**
- Create `src/server/queue/connection.ts`:
  ```typescript
  import { Redis } from "ioredis";
  export const redis = new Redis(process.env.UPSTASH_REDIS_URL!, {
    maxRetriesPerRequest: null, // required by BullMQ
  });
  ```
- Create `src/server/queue/reminder-queue.ts`: BullMQ queue definition for daily reminders
- Create `src/server/jobs/daily-reminders.ts`:
  - Repeatable job runs every 15 minutes
  - For each active member: check if current time (in their timezone) is past their check-in time, and if they haven't checked in today
  - If missing: send Slack DM (if connected) or email nudge (via Resend)
  - Idempotent: tracks sent reminders in `notificationLog` to avoid duplicates
- Test with a manual enqueue and verify DM/email delivery

---

### Phase 3: Slack Integration (Days 11-15)

**Day 11 -- Slack OAuth + connection management**
- Create `src/app/api/auth/slack/route.ts`: Slack OAuth start (redirect to Slack authorization URL)
- Create `src/app/api/auth/slack/callback/route.ts`: exchange code for bot token, store encrypted in `slackConnections`
- Create `src/app/dashboard/settings/integrations/page.tsx`:
  - "Connect to Slack" button (starts OAuth)
  - Connection status display (workspace name, connected date, status)
  - "Disconnect" button
  - "Test connection" button (sends test DM to installer)
- Wire up `slackConnections` CRUD to tRPC router

**Day 12 -- Slack Bolt app: slash command + modal**
- Create `src/server/slack/app.ts` (as shown in Architecture Deep-Dive #2)
- Register `/pulse` slash command handler
- Build Block Kit modal with energy buttons, mood tag multi-select, note input
- Handle modal submission: upsert check-in with `source: "slack"`
- Create `src/app/api/webhooks/slack/route.ts`: route Slack events to Bolt app

**Day 13 -- Slack DM prompt + emoji reactions**
- Implement scheduled DM sending in the daily reminder job:
  ```typescript
  // Send Slack DM with energy selection buttons
  await slackClient.chat.postMessage({
    channel: member.slackUserId,
    text: "Time for your daily pulse check-in!",
    blocks: [
      {
        type: "section",
        text: { type: "mrkdwn", text: "How's your energy today?" },
      },
      {
        type: "actions",
        elements: [1, 2, 3, 4, 5].map((level) => ({
          type: "button",
          text: { type: "plain_text", text: `${level} ${ENERGY_LABELS[level]}` },
          action_id: `dm_energy_${level}`,
          value: String(level),
        })),
      },
    ],
  });
  ```
- Handle emoji reaction check-ins (`reaction_added` event)
- Handle DM button clicks (`dm_energy_*` action handlers): upsert check-in, update message to show confirmation

**Day 14 -- Slack user mapping + team channel aggregates**
- Create `src/server/slack/user-mapping.ts`: service to map Slack user IDs to PulseBoard members
  - On first `/pulse` usage, prompt user to link their account
  - Admin bulk-import: fetch Slack workspace members, match by email
- Implement team channel aggregates:
  - After daily check-in cutoff, post team summary to configured Slack channel
  - Summary shows: "Team energy today: 3.8 avg (8/10 checked in)" with distribution bar
  - No individual scores shared to channel

**Day 15 -- Slack integration testing + edge cases**
- End-to-end test: OAuth flow, `/pulse` command, modal submission, DM prompt, emoji reaction
- Handle edge cases: member not found, workspace disconnected, rate limits
- Add Slack connection health check job (runs daily, marks connections as error if token is revoked)
- Verify idempotent check-in upserts (same user, same day, multiple submissions updates rather than duplicates)

---

### Phase 4: Manager Dashboard + Heatmap (Days 16-20)

**Day 16 -- Dashboard overview page**
- Create `src/server/trpc/routers/dashboard.ts`:
  - `getOverview`: org-wide stats (teams count, active members, today's check-in rate, active alerts)
  - `getTeamCards`: per-team summary cards (today's avg, 7-day trend direction, participation rate, active alerts count)
- Create `src/app/dashboard/page.tsx`:
  - Welcome header with org name and today's date
  - Alert banner at top if any active critical alerts
  - Team overview cards grid (one per team)
  - Each card: team name, today's average energy (large number), 7-day sparkline, participation rate, active alerts badge
  - Quick action buttons: "Submit Check-In", "View Alerts"
- Create `src/components/dashboard/team-card.tsx`: summary card component
- Create `src/components/dashboard/sparkline.tsx`: small Recharts sparkline for 7-day trend

**Day 17 -- Team detail page with heatmap**
- Create `src/app/dashboard/teams/[id]/page.tsx`:
  - Team header: name, manager, member count, date range selector
  - Heatmap grid (rows = members, columns = dates, cells = color-coded 1-5)
  - Trend line chart (7-day and 30-day rolling average)
  - Participation rate bar chart by day
  - Individual member rows (click to expand personal trend)
- Create `src/components/dashboard/heatmap.tsx`:
  - CSS Grid layout with dynamic column count
  - Color scale: `#ef4444` (red, 1) -> `#f97316` (orange, 2) -> `#eab308` (yellow, 3) -> `#22c55e` (green, 4) -> `#3b82f6` (blue, 5), `#f3f4f6` (gray, no data)
  - Hover tooltip: member name, date, energy level, mood tags
  - Framer Motion fade-in animation on cells
- Wire up `getTeamHeatmap` from Architecture Deep-Dive #4

**Day 18 -- Trend charts and participation metrics**
- Create `src/components/dashboard/trend-chart.tsx`: Recharts `AreaChart` with gradient fill
  - 7-day rolling average line
  - 30-day baseline reference line
  - Hover tooltip with date, average, count
- Create `src/components/dashboard/participation-chart.tsx`: Recharts `BarChart`
  - Daily participation rate (percentage of team who checked in)
  - Target line at org's configured threshold
- Create `src/components/dashboard/energy-distribution.tsx`: Recharts `PieChart`
  - Donut chart showing distribution of 1-5 responses for the selected period
- Wire up `getTeamTrend` from Architecture Deep-Dive #4

**Day 19 -- Individual member view (manager-only)**
- Create `src/app/dashboard/teams/[id]/members/[memberId]/page.tsx`:
  - Personal trend line (30-day)
  - Calendar grid with color-coded energy
  - Mood tag frequency chart
  - Note history (if shared with manager)
  - Risk score indicator (from burnout detection)
  - "Schedule 1-on-1" quick action button
- Role-based access: only managers of this team or org admins can view
- Create `src/components/dashboard/member-detail.tsx`: member detail view component

**Day 20 -- Dashboard refinements + responsive design**
- Add date range picker (Radix Popover + calendar component) to heatmap and trend views
- Add team comparison view: side-by-side trend lines for multiple teams
- Mobile-responsive heatmap: horizontal scroll, condensed cell size
- Loading skeletons for all dashboard components
- Empty states: "No check-ins yet", "Invite team members to get started"
- Add Framer Motion page transitions between dashboard views

---

### Phase 5: Alerts + Digests (Days 21-25)

**Day 21 -- Alert system implementation**
- Create `src/server/services/burnout-detection.ts` (as shown in Architecture Deep-Dive #1)
- Create `src/server/services/team-alerts.ts`:
  - `checkTeamDip`: compare team's 7-day avg to 14-day baseline, alert if drop >= configured percentage
  - `checkLowParticipation`: alert if team participation drops below configured threshold
- Create `src/server/jobs/alert-check.ts`: BullMQ job runs daily at end of check-in window
  - Iterates all orgs, runs all three alert type checks
  - Creates alert records, sends notifications to managers

**Day 22 -- Alert tRPC router + UI**
- `src/server/trpc/routers/alert.ts`:
  - `list`: paginated alerts for org, filterable by type, severity, status
  - `getActive`: active alerts count (for dashboard badge)
  - `acknowledge`: set acknowledgedAt and acknowledgedBy
  - `resolve`: mark as resolved
  - `getHistory`: past alerts for trend analysis
- Create `src/app/dashboard/alerts/page.tsx`:
  - Active alerts at top (sorted by severity, then date)
  - Each alert card: severity badge (warning = yellow, critical = red), title, message, suggested action, "Acknowledge" / "Resolve" buttons
  - Acknowledged/resolved alerts below in collapsible section
  - Filter tabs: All, Burnout Risk, Team Dip, Low Participation
- Create `src/components/alerts/alert-card.tsx`: alert display component

**Day 23 -- Configurable alert rules UI**
- `src/server/trpc/routers/alertRule.ts`:
  - `list`: all rules for org
  - `update`: modify rule config (thresholds, notification channels)
  - `toggle`: enable/disable a rule
- Create `src/app/dashboard/settings/alerts/page.tsx`:
  - Three rule cards (one per alert type)
  - Each card: toggle switch (enabled/disabled), configurable thresholds via sliders
  - Individual burnout: "Alert when energy is at or below [slider: 1-3] for [slider: 2-5] consecutive days"
  - Team dip: "Alert when team average drops [slider: 10-50]% from [slider: 7-30] day baseline"
  - Low participation: "Alert when less than [slider: 30-80]% of team checks in over [slider: 3-14] days"
  - Notification channel checkboxes: Email, Slack, In-App
  - "Save" button per card

**Day 24 -- Weekly digest generation**
- Create `src/server/jobs/weekly-digest.ts` (as shown in Architecture Deep-Dive #3)
- Create `src/server/trpc/routers/digest.ts`:
  - `list`: paginated digest history for org
  - `getLatest`: most recent digest per team
  - `getById`: full digest detail with AI summary and talking points
  - `regenerate`: manually trigger digest regeneration for a team
- Test digest generation: manually enqueue a job, verify AI summary quality, verify email delivery

**Day 25 -- Digest UI + history**
- Create `src/app/dashboard/digests/page.tsx`:
  - Latest digest cards per team
  - Each card: team name, period, average energy, trend arrow, participation rate
  - Click to expand: full AI summary, talking points list, energy distribution chart
  - "View Past Digests" link to history
- Create `src/app/dashboard/digests/[id]/page.tsx`:
  - Full digest detail page
  - AI summary in highlighted box
  - Talking points as actionable checklist
  - Energy distribution pie chart
  - Mood tag summary
  - "Email this digest" button (resend to managers)
- Create `src/components/digests/digest-card.tsx`: compact digest summary card

---

### Phase 6: Billing + Settings (Days 26-29)

**Day 26 -- Stripe billing setup**
- Create `src/server/billing/stripe.ts`:
  ```typescript
  import Stripe from "stripe";

  const stripe = new Stripe(process.env.STRIPE_SECRET_KEY!, {
    apiVersion: "2025-04-30.basil",
  });

  const PRICE_IDS: Record<string, string> = {
    team: process.env.STRIPE_TEAM_PRICE_ID ?? "price_team_placeholder",
    enterprise: process.env.STRIPE_ENTERPRISE_PRICE_ID ?? "price_enterprise_placeholder",
  };

  export const PLANS = {
    free: {
      name: "Free",
      price: 0,
      maxMembers: 5,
      features: [
        "Up to 5 team members",
        "Daily energy check-in (web only)",
        "7-day history",
        "Basic trend chart",
      ],
    },
    team: {
      name: "Team",
      pricePerUser: 4,
      maxMembers: Infinity,
      features: [
        "Unlimited team members",
        "Unlimited history",
        "Mood tags and notes",
        "Slack integration",
        "Manager dashboard with heatmaps",
        "Burnout risk alerts",
        "Weekly AI digest",
        "Email notifications",
      ],
    },
    enterprise: {
      name: "Enterprise",
      pricePerUser: 8,
      maxMembers: Infinity,
      features: [
        "Everything in Team",
        "Anonymous mode",
        "SSO / SAML",
        "API access",
        "Custom integrations",
        "Advanced analytics",
        "Data retention policies",
        "Dedicated support",
      ],
    },
  } as const;

  export async function createCheckoutSession(opts: {
    orgId: string;
    clerkOrgId: string;
    stripeCustomerId?: string;
    plan: "team" | "enterprise";
    memberCount: number;
  }): Promise<string> {
    const appUrl = process.env.NEXT_PUBLIC_APP_URL!;

    const sessionParams: Stripe.Checkout.SessionCreateParams = {
      mode: "subscription",
      payment_method_types: ["card"],
      line_items: [{
        price: PRICE_IDS[opts.plan],
        quantity: opts.memberCount,
      }],
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

**Day 27 -- Stripe webhook handler + billing tRPC**
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
            plan: "free",
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
- `src/server/trpc/routers/billing.ts`:
  - `getCurrentPlan`: return org plan + limits + usage (member count, features available)
  - `createCheckout`: generate Stripe checkout URL for plan upgrade
  - `createPortal`: generate Stripe portal URL for managing subscription

**Day 28 -- Billing UI + settings pages**
- Create `src/app/dashboard/settings/billing/page.tsx`:
  - Current plan card with feature list and member count
  - Three plan cards: Free, Team, Enterprise with feature comparison
  - "Upgrade" button on higher tiers, "Manage Subscription" on current tier
  - Usage meter: "5/5 members" on free tier (with upgrade prompt when full)
  - Success/canceled URL parameter handling with toast messages
- Create `src/app/dashboard/settings/page.tsx`:
  - Organization name and slug (editable)
  - Default timezone selector
  - Default check-in time picker
  - Skip weekends toggle
  - Digest day selector (Monday / Friday)
  - Digest enabled toggle
- Create `src/app/dashboard/settings/members/page.tsx`:
  - Team member list from Clerk organization membership
  - Team assignment dropdown per member
  - Role selector (member / manager / admin)
  - Invite form (email input + role selector)
  - "Remove" button with confirmation

**Day 29 -- Plan enforcement middleware**
- Implement `planGuardMiddleware` in tRPC:
  ```typescript
  export const planGuardMiddleware = (requiredFeature: string) =>
    middleware(async ({ ctx, next }) => {
      const plan = ctx.org.plan as keyof typeof PLAN_LIMITS;
      const limits = PLAN_LIMITS[plan];

      if (!limits.features.includes(requiredFeature)) {
        throw new TRPCError({
          code: "FORBIDDEN",
          message: `The ${requiredFeature} feature requires a paid plan. Please upgrade.`,
        });
      }

      return next();
    });
  ```
- Apply to: Slack check-in (requires `slack`), mood tags (requires `mood_tags`), heatmap (requires `heatmap`), alerts (requires `alerts`), digest (requires `digest`)
- Member count enforcement on free tier: reject member creation beyond 5
- Add upgrade prompt banners in UI when features are gated

---

### Phase 7: Polish + Launch (Days 30-35)

**Day 30 -- Onboarding flow**
- Create `src/app/onboarding/page.tsx`:
  - Step 1: Create/name organization, set timezone
  - Step 2: Create first team, assign yourself as manager
  - Step 3: Invite team members (email list input)
  - Step 4: Submit your first check-in (interactive energy picker)
  - Step 5: "Connect Slack" (optional, skip available)
  - Progress bar indicator (5 steps)
  - Skip option to go straight to dashboard
- Store onboarding completion in `organizations.settings.onboardingCompleted`
- Redirect new orgs to onboarding if not completed

**Day 31 -- Loading states, error handling, empty states**
- Add Suspense boundaries with skeleton loaders for all data-fetching pages
- Create `src/app/dashboard/loading.tsx` and route-level `loading.tsx` files
- Create empty state components:
  - Dashboard: "No teams yet -- create your first team to get started"
  - Heatmap: "No check-ins yet -- invite your team and start checking in"
  - Alerts: "No active alerts -- your team is doing great!"
  - Digests: "Your first weekly digest will arrive next Monday"
- Toast notifications for all mutations (success/error) using Radix Toast
- Global error boundary with retry
- 404 page for invalid routes
- tRPC error handling: map error codes to user-friendly messages

**Day 32 -- Landing page + pricing**
- Create `src/app/page.tsx`:
  - Hero: "See how your remote team is really doing" with CTA
  - How it works: 3-step visual (Check-in -> Dashboard -> Act)
  - Features grid: Daily Pulse, Slack Bot, Heatmaps, Burnout Alerts, AI Digests
  - Social proof placeholder
  - Footer with links
- Create `src/app/pricing/page.tsx`:
  - Three tier cards with feature comparison table
  - FAQ section
  - "Start Free" CTA on free tier, "Start Trial" on paid tiers

**Day 33 -- Responsive design + accessibility**
- Mobile-responsive sidebar: collapsible on small screens, bottom nav on mobile
- Heatmap: horizontal scroll on mobile, touch-friendly cell size
- Check-in page: full-width energy buttons on mobile, stacked layout
- All interactive elements: focus rings, keyboard navigation, ARIA labels
- Color contrast verification for heatmap colors (ensure energy levels are distinguishable)
- Screen reader text for icon-only buttons
- Test with keyboard-only navigation

**Day 34 -- Performance optimization**
- Add `loading.tsx` files for route-level streaming
- Optimize database queries: verify all list queries use indexed columns
- Add React Query `staleTime` and `gcTime` settings:
  - Heatmap data: staleTime 60s
  - Dashboard overview: staleTime 30s
  - Check-in today: staleTime 10s
  - Alert count: staleTime 30s
  - Digest list: staleTime 5min
  - Settings: staleTime Infinity (manual invalidation)
- Server-side rendering for dashboard pages using tRPC server caller
- Vercel Analytics integration
- Sentry error tracking setup

**Day 35 -- Final testing + deployment**
- End-to-end test flow:
  1. Sign up -> Create org -> Onboarding
  2. Create team, invite members
  3. Submit check-in via web and via Slack `/pulse`
  4. Verify heatmap populates, trend chart updates
  5. Simulate burnout: submit 3 consecutive low-energy check-ins -> verify alert created
  6. Verify weekly digest generation (manual trigger) -> email received with AI summary
  7. Acknowledge and resolve alerts
  8. Upgrade plan via Stripe (test mode) -> verify features unlocked
  9. Downgrade via Stripe portal -> verify plan reverts to free
  10. Verify free tier limits enforced (6th member rejected)
- Set up Vercel project + environment variables
- Configure custom domain
- Stripe webhook endpoint registered in Stripe dashboard
- Slack app submitted to Slack App Directory (or set to distribute within workspace)
- Deploy to production

---

## Critical Files

```
pulseboard/
├── src/
│   ├── env.ts                                    # Zod-validated environment variables
│   ├── middleware.ts                              # Clerk auth middleware, route protection
│   │
│   ├── app/
│   │   ├── layout.tsx                            # Root layout with ClerkProvider
│   │   ├── page.tsx                              # Landing page
│   │   ├── pricing/page.tsx                      # Pricing comparison page
│   │   ├── onboarding/page.tsx                   # 5-step onboarding wizard
│   │   │
│   │   ├── dashboard/
│   │   │   ├── layout.tsx                        # App shell: sidebar + header
│   │   │   ├── page.tsx                          # Manager overview: team cards, alerts banner
│   │   │   ├── loading.tsx                       # Route-level skeleton loader
│   │   │   ├── checkin/
│   │   │   │   ├── page.tsx                      # Personal check-in: energy picker + mood tags
│   │   │   │   └── history/page.tsx              # 30-day calendar + trend line
│   │   │   ├── teams/
│   │   │   │   ├── page.tsx                      # Team list with summary cards
│   │   │   │   └── [id]/
│   │   │   │       ├── page.tsx                  # Team detail: heatmap + trends + participation
│   │   │   │       └── members/
│   │   │   │           └── [memberId]/page.tsx   # Individual member detail (manager-only)
│   │   │   ├── alerts/page.tsx                   # Active + historical alerts
│   │   │   ├── digests/
│   │   │   │   ├── page.tsx                      # Digest list with latest per team
│   │   │   │   └── [id]/page.tsx                 # Full digest detail + AI talking points
│   │   │   └── settings/
│   │   │       ├── page.tsx                      # Org settings: timezone, check-in time, digest
│   │   │       ├── alerts/page.tsx               # Alert rule configuration (thresholds)
│   │   │       ├── billing/page.tsx              # Plan + usage + upgrade
│   │   │       ├── members/page.tsx              # Member management + invites
│   │   │       └── integrations/page.tsx         # Slack connection management
│   │   │
│   │   └── api/
│   │       ├── trpc/[trpc]/route.ts              # tRPC HTTP handler
│   │       ├── checkin/route.ts                  # REST check-in endpoint (for Slack bot)
│   │       ├── auth/
│   │       │   ├── slack/route.ts                # Slack OAuth start
│   │       │   └── slack/callback/route.ts       # Slack OAuth callback
│   │       └── webhooks/
│   │           ├── slack/route.ts                # Slack events + interactions receiver
│   │           └── stripe/route.ts               # Stripe webhook handler
│   │
│   ├── server/
│   │   ├── db/
│   │   │   ├── schema.ts                        # Drizzle schema (10 tables + relations)
│   │   │   ├── index.ts                         # Neon database client
│   │   │   └── migrations/
│   │   │       └── 0000_init.sql                # Additional indexes
│   │   │
│   │   ├── trpc/
│   │   │   ├── trpc.ts                          # tRPC init, auth/org/plan middleware
│   │   │   ├── context.ts                       # Request context (auth + db)
│   │   │   └── routers/
│   │   │       ├── _app.ts                      # Merged app router
│   │   │       ├── organization.ts              # Org CRUD + settings
│   │   │       ├── member.ts                    # Member CRUD + invite
│   │   │       ├── team.ts                      # Team CRUD
│   │   │       ├── checkin.ts                   # Check-in submit + history + team data
│   │   │       ├── dashboard.ts                 # Overview stats + team cards
│   │   │       ├── alert.ts                     # Alert list + acknowledge + resolve
│   │   │       ├── alertRule.ts                 # Alert rule config
│   │   │       ├── digest.ts                    # Digest list + detail + regenerate
│   │   │       └── billing.ts                   # Plan info + Stripe checkout/portal
│   │   │
│   │   ├── services/
│   │   │   ├── burnout-detection.ts             # Risk score algorithm (4-factor model)
│   │   │   ├── team-alerts.ts                   # Team dip + low participation checks
│   │   │   ├── heatmap.ts                       # Heatmap data aggregation + trend queries
│   │   │   └── notifications.ts                 # Resend email dispatch for alerts + digests
│   │   │
│   │   ├── slack/
│   │   │   ├── app.ts                           # Slack Bolt app: commands, modals, reactions
│   │   │   └── user-mapping.ts                  # Slack user ID <-> PulseBoard member mapping
│   │   │
│   │   ├── queue/
│   │   │   └── connection.ts                    # Redis/ioredis connection
│   │   │
│   │   ├── jobs/
│   │   │   ├── daily-reminders.ts               # BullMQ: check-in reminder DMs/emails
│   │   │   ├── weekly-digest.ts                 # BullMQ: digest generation + AI + email
│   │   │   └── alert-check.ts                   # BullMQ: daily burnout + team alert checks
│   │   │
│   │   ├── billing/
│   │   │   └── stripe.ts                        # Plan definitions, checkout, portal
│   │   │
│   │   ├── lib/
│   │   │   └── encryption.ts                    # AES-256-GCM encrypt/decrypt for tokens
│   │   │
│   │   └── email/
│   │       └── templates.ts                     # HTML email templates (digest, alert, reminder)
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
│   │   │   ├── popover.tsx
│   │   │   ├── slider.tsx
│   │   │   └── switch.tsx
│   │   │
│   │   ├── layout/
│   │   │   ├── sidebar.tsx                      # Navigation sidebar
│   │   │   ├── header.tsx                       # Page header with breadcrumbs
│   │   │   └── org-switcher.tsx                 # Clerk org switcher wrapper
│   │   │
│   │   ├── checkin/
│   │   │   ├── energy-picker.tsx                # Five large energy buttons with animation
│   │   │   ├── mood-tags.tsx                    # Clickable mood tag pills
│   │   │   ├── streak-badge.tsx                 # Check-in streak counter
│   │   │   ├── calendar-grid.tsx                # Month-view calendar with colored cells
│   │   │   └── trend-chart.tsx                  # Personal Recharts trend line
│   │   │
│   │   ├── dashboard/
│   │   │   ├── team-card.tsx                    # Team summary card (avg, trend, participation)
│   │   │   ├── sparkline.tsx                    # Small 7-day sparkline chart
│   │   │   ├── heatmap.tsx                      # Team x date heatmap grid
│   │   │   ├── trend-chart.tsx                  # Team Recharts area chart
│   │   │   ├── participation-chart.tsx          # Daily participation bar chart
│   │   │   ├── energy-distribution.tsx          # Donut chart for 1-5 distribution
│   │   │   └── member-detail.tsx                # Individual member expanded view
│   │   │
│   │   ├── alerts/
│   │   │   ├── alert-card.tsx                   # Alert display with severity + actions
│   │   │   └── alert-rule-editor.tsx            # Threshold sliders for alert config
│   │   │
│   │   └── digests/
│   │       ├── digest-card.tsx                  # Compact digest summary card
│   │       └── digest-detail.tsx                # Full AI summary + talking points
│   │
│   └── lib/
│       ├── trpc/
│       │   ├── client.ts                        # React Query tRPC client
│       │   └── server.ts                        # Server-side tRPC caller
│       ├── utils.ts                             # cn() helper, date formatting, timezone utils
│       └── constants.ts                         # Energy labels, mood tag options, color scales
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

1. **Auth flow**: Sign up with Clerk -> create organization -> verify org record created in DB with default alert rules
2. **Onboarding**: Complete 5-step onboarding -> verify team, members, and first check-in created
3. **Web check-in**: Submit energy 4 with mood tags ["focused", "calm"] + note -> verify check-in row in DB with correct date
4. **Check-in update**: Submit again same day with energy 3 -> verify upsert (single row, updated energy)
5. **Slack OAuth**: Click "Connect to Slack" -> complete OAuth -> verify encrypted bot token stored in `slack_connections`
6. **Slack /pulse command**: Type `/pulse` in Slack -> verify modal opens with energy buttons, mood tags, note input
7. **Slack modal submit**: Select energy + tags + note -> verify check-in created with `source: "slack"`
8. **Slack emoji reaction**: React with green circle emoji on DM -> verify check-in created with energy 4
9. **Daily reminder**: Trigger reminder job -> verify Slack DM sent to members who haven't checked in
10. **Heatmap**: View team heatmap for last 14 days -> verify grid shows correct colors for each member/date
11. **Trend chart**: Verify 7-day rolling average line chart matches manual calculation
12. **Burnout detection**: Create 3 consecutive check-ins with energy 2 -> trigger alert check -> verify alert created with severity "critical" and risk score >= 60
13. **Team dip alert**: Create check-ins averaging 2.0 this week vs 4.0 last week -> verify team_dip alert triggered
14. **Alert acknowledge**: Click "Acknowledge" on alert -> verify acknowledgedAt set, alert moves to acknowledged section
15. **Alert rule config**: Change burnout threshold to 4 consecutive days -> save -> verify rule updated in DB
16. **Weekly digest**: Manually trigger digest generation -> verify digest row created with AI summary and talking points
17. **Digest email**: Verify email sent to team manager with correct stats, summary, and talking points
18. **Plan limits (free)**: On free plan, attempt to add 6th member -> verify error returned
19. **Feature gating (free)**: On free plan, attempt to access Slack integration -> verify upgrade prompt shown
20. **Billing upgrade**: Complete Stripe checkout for Team plan -> verify plan updated in DB, features unlocked
21. **Stripe webhook**: Simulate subscription deletion -> verify plan reverts to "free"
22. **Responsive**: Verify heatmap scrolls horizontally on mobile, check-in buttons stack vertically
23. **Timezone handling**: Member in UTC+9 submits check-in at 1 AM UTC -> verify date resolves to their local date

### Key SQL Queries for Verification

```sql
-- Verify check-in uniqueness (one per member per day)
SELECT member_id, date, COUNT(*) AS dupes
FROM check_ins
WHERE org_id = 'ORG_ID'
GROUP BY member_id, date
HAVING COUNT(*) > 1;
-- Expected: 0 rows

-- Check burnout candidates (3+ consecutive low days)
WITH ranked AS (
  SELECT
    member_id,
    date,
    energy,
    date - ROW_NUMBER() OVER (PARTITION BY member_id ORDER BY date)::int * interval '1 day' AS grp
  FROM check_ins
  WHERE org_id = 'ORG_ID'
    AND energy <= 2
    AND date >= CURRENT_DATE - INTERVAL '14 days'
)
SELECT member_id, COUNT(*) AS consecutive_low_days, MIN(date) AS start_date, MAX(date) AS end_date
FROM ranked
GROUP BY member_id, grp
HAVING COUNT(*) >= 3
ORDER BY consecutive_low_days DESC;

-- Weekly team averages (for digest verification)
SELECT
  t.name AS team_name,
  ROUND(AVG(ci.energy)::numeric, 2) AS avg_energy,
  COUNT(DISTINCT ci.member_id) AS members_checked_in,
  COUNT(*) AS total_check_ins
FROM check_ins ci
JOIN teams t ON t.id = ci.team_id
WHERE ci.org_id = 'ORG_ID'
  AND ci.date >= CURRENT_DATE - INTERVAL '7 days'
GROUP BY t.name
ORDER BY avg_energy;

-- Alert history and resolution times
SELECT
  type,
  severity,
  title,
  status,
  triggered_at,
  acknowledged_at,
  EXTRACT(EPOCH FROM (acknowledged_at - triggered_at)) / 3600 AS hours_to_acknowledge
FROM alerts
WHERE org_id = 'ORG_ID'
ORDER BY triggered_at DESC
LIMIT 20;

-- Participation rate by team (last 7 days)
SELECT
  t.name AS team_name,
  COUNT(DISTINCT m.id) AS total_members,
  COUNT(DISTINCT ci.member_id) AS active_members,
  ROUND(COUNT(DISTINCT ci.member_id)::numeric / NULLIF(COUNT(DISTINCT m.id), 0) * 100, 1) AS participation_pct
FROM teams t
JOIN members m ON m.team_id = t.id AND m.status = 'active'
LEFT JOIN check_ins ci ON ci.member_id = m.id AND ci.date >= CURRENT_DATE - INTERVAL '7 days'
WHERE t.org_id = 'ORG_ID'
GROUP BY t.name;

-- Digest delivery status
SELECT
  d.period_start, d.period_end, d.avg_energy, d.participation_rate,
  d.trend_direction, d.sent_at,
  ARRAY_LENGTH(d.sent_to_emails, 1) AS emails_sent,
  LEFT(d.ai_summary, 100) AS summary_preview
FROM digests d
WHERE d.org_id = 'ORG_ID'
ORDER BY d.period_end DESC
LIMIT 10;
```

### Performance Benchmarks

| Operation | Target | Method |
|-----------|--------|--------|
| Check-in submission (web) | < 200ms | Single upsert with conflict resolution |
| Check-in submission (Slack) | < 500ms | Bolt handler + upsert |
| Slash command -> modal open | < 1s | Slack API `views.open` |
| Heatmap load (10 members x 30 days) | < 300ms | Single indexed query + in-memory grid build |
| Heatmap load (50 members x 30 days) | < 600ms | SQL cross-join with generate_series |
| Team trend chart (90 days) | < 200ms | Indexed GROUP BY on date |
| Burnout detection (50 members) | < 2s | Batch query + in-memory scoring |
| Weekly digest AI generation | < 5s | GPT-4o-mini with 500 max_tokens |
| Weekly digest full pipeline (per team) | < 10s | Aggregate + AI + email delivery |
| Dashboard overview load | < 400ms | Parallel tRPC queries with React Query cache |
| Alert check (all org, 100 members) | < 3s | Batch queries + 3 alert type checks |
| Stripe checkout redirect | < 1s | Stripe API session creation |
| Daily reminder job (100 members) | < 30s | Batch query + parallel Slack DM sends |
