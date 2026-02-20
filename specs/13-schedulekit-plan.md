# 13. ScheduleKit — Embeddable Scheduling Widget

## Implementation Plan

**MVP Scope:** Embeddable JavaScript widget (Preact + Shadow DOM, <20KB gzipped) with inline and popup display modes, Google Calendar two-way sync (OAuth 2.0), timezone auto-detection with manual override, single meeting type per account, booking form (name, email, phone), email confirmations and reminders via Resend, basic color/logo customization via CSS variables, dashboard with booking list and calendar view, availability configuration with weekly schedule and date overrides, Stripe billing with three tiers (Free / Pro $15/mo / Business $39/mo).

---

## Tech Decisions

| Layer | Technology | Notes |
|-------|-----------|-------|
| Framework | Next.js 14+ App Router | `src/` directory, TypeScript strict |
| API | tRPC v11 | superjson transformer, Clerk auth context |
| Database | Neon PostgreSQL | Availability computation, booking conflict checks |
| ORM | Drizzle ORM | Push-based migrations, typed schema |
| Auth | Clerk | Organization-scoped, `orgId` from session |
| Billing | Stripe | Checkout Sessions + Customer Portal + Webhooks |
| Calendar | Google Calendar API v3 | OAuth 2.0, two-way sync with watch notifications |
| Email | Resend | Confirmation, reminder (24h/1h), cancellation emails |
| Widget | Preact 10 + Shadow DOM | <20KB gzipped, CSS variable theming, iframe-free |
| Widget CDN | Cloudflare R2 + CDN | Edge-cached widget bundle, versioned URLs |
| UI | Tailwind CSS, Radix UI, react-day-picker | |
| Hosting | Vercel (app), Cloudflare (widget) | |
| Monitoring | Sentry, Vercel Analytics | |

### Key Architecture Decisions

1. **Preact + Shadow DOM over iframe embed**: Iframes create scroll/resize issues and can't match host page fonts. Shadow DOM provides full style isolation while remaining part of the host page's DOM, enabling natural scroll behavior, responsive sizing, and CSS variable inheritance for theming. Preact keeps the bundle under 20KB gzipped.

2. **Availability computation on the server**: Slot availability is computed server-side by merging the user's configured weekly schedule with Google Calendar busy times and existing bookings. The widget fetches available slots for a given date range via a public API endpoint. This prevents client-side calendar data exposure and ensures consistency.

3. **Google Calendar watch notifications for real-time sync**: Instead of polling, we use Google Calendar's push notification API (watch channels) to receive webhooks when calendar events change. This keeps availability data fresh within seconds of a calendar change. Channels are renewed every 7 days via a cron job.

4. **Booking tokens for guest self-service**: Each booking generates a unique `reschedule_token` and `cancel_token` (nanoid). These are embedded in confirmation emails, allowing guests to reschedule or cancel without authentication. Tokens are single-use for cancel and reset on reschedule.

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
  time,
  date,
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
  timezone: text("timezone").default("America/New_York").notNull(),
  logoUrl: text("logo_url"),
  settings: jsonb("settings").default({}).$type<{
    brandColor?: string;
    notificationFromName?: string;
    notificationFromEmail?: string;
  }>(),
  createdAt: timestamp("created_at", { withTimezone: true }).defaultNow().notNull(),
  updatedAt: timestamp("updated_at", { withTimezone: true }).defaultNow().notNull(),
});

// ---------------------------------------------------------------------------
// Members (team members who accept bookings)
// ---------------------------------------------------------------------------
export const members = pgTable(
  "members",
  {
    id: uuid("id").primaryKey().defaultRandom(),
    orgId: uuid("org_id")
      .references(() => organizations.id, { onDelete: "cascade" })
      .notNull(),
    clerkUserId: text("clerk_user_id").notNull(),
    name: text("name").notNull(),
    email: text("email").notNull(),
    bio: text("bio"),
    avatarUrl: text("avatar_url"),
    timezone: text("timezone").default("America/New_York").notNull(),
    isActive: boolean("is_active").default(true).notNull(),
    createdAt: timestamp("created_at", { withTimezone: true }).defaultNow().notNull(),
  },
  (t) => [
    uniqueIndex("members_org_clerk_idx").on(t.orgId, t.clerkUserId),
  ],
);

// ---------------------------------------------------------------------------
// Calendar Connections (Google Calendar OAuth)
// ---------------------------------------------------------------------------
export const calendarConnections = pgTable(
  "calendar_connections",
  {
    id: uuid("id").primaryKey().defaultRandom(),
    memberId: uuid("member_id")
      .references(() => members.id, { onDelete: "cascade" })
      .notNull(),
    provider: text("provider").default("google").notNull(), // google | outlook (v2)
    accessToken: text("access_token").notNull(), // encrypted
    refreshToken: text("refresh_token").notNull(), // encrypted
    tokenExpiresAt: timestamp("token_expires_at", { withTimezone: true }),
    calendarId: text("calendar_id").default("primary").notNull(),
    watchChannelId: text("watch_channel_id"), // Google Calendar push notification channel
    watchResourceId: text("watch_resource_id"),
    watchExpiresAt: timestamp("watch_expires_at", { withTimezone: true }),
    syncEnabled: boolean("sync_enabled").default(true).notNull(),
    lastSyncedAt: timestamp("last_synced_at", { withTimezone: true }),
    createdAt: timestamp("created_at", { withTimezone: true }).defaultNow().notNull(),
  },
  (t) => [
    index("cal_conn_member_idx").on(t.memberId),
  ],
);

// ---------------------------------------------------------------------------
// Weekly Availability (recurring schedule per member)
// ---------------------------------------------------------------------------
export const weeklyAvailability = pgTable(
  "weekly_availability",
  {
    id: uuid("id").primaryKey().defaultRandom(),
    memberId: uuid("member_id")
      .references(() => members.id, { onDelete: "cascade" })
      .notNull(),
    dayOfWeek: integer("day_of_week").notNull(), // 0=Sunday, 6=Saturday
    startTime: time("start_time").notNull(), // e.g. "09:00"
    endTime: time("end_time").notNull(), // e.g. "17:00"
    isEnabled: boolean("is_enabled").default(true).notNull(),
  },
  (t) => [
    index("weekly_avail_member_idx").on(t.memberId),
    uniqueIndex("weekly_avail_member_day_idx").on(t.memberId, t.dayOfWeek),
  ],
);

// ---------------------------------------------------------------------------
// Availability Overrides (specific date exceptions)
// ---------------------------------------------------------------------------
export const availabilityOverrides = pgTable(
  "availability_overrides",
  {
    id: uuid("id").primaryKey().defaultRandom(),
    memberId: uuid("member_id")
      .references(() => members.id, { onDelete: "cascade" })
      .notNull(),
    overrideDate: date("override_date").notNull(),
    type: text("type").notNull(), // unavailable | custom_hours
    startTime: time("start_time"), // null if type=unavailable
    endTime: time("end_time"),
  },
  (t) => [
    uniqueIndex("avail_override_member_date_idx").on(t.memberId, t.overrideDate),
  ],
);

// ---------------------------------------------------------------------------
// Meeting Types
// ---------------------------------------------------------------------------
export const meetingTypes = pgTable(
  "meeting_types",
  {
    id: uuid("id").primaryKey().defaultRandom(),
    orgId: uuid("org_id")
      .references(() => organizations.id, { onDelete: "cascade" })
      .notNull(),
    name: text("name").notNull(),
    slug: text("slug").notNull(),
    description: text("description"),
    durationMinutes: integer("duration_minutes").default(30).notNull(),
    bufferBefore: integer("buffer_before").default(0).notNull(), // minutes
    bufferAfter: integer("buffer_after").default(0).notNull(),
    locationType: text("location_type").default("google_meet").notNull(),
    // google_meet | zoom | teams | phone | in_person | custom
    locationConfig: jsonb("location_config").default({}).$type<{
      customUrl?: string;
      phoneNumber?: string;
      address?: string;
    }>(),
    priceCents: integer("price_cents"), // null = free
    currency: text("currency").default("usd"),
    schedulingType: text("scheduling_type").default("individual").notNull(),
    // individual | round_robin | collective (v2)
    minNoticeHours: integer("min_notice_hours").default(1).notNull(),
    maxDaysAhead: integer("max_days_ahead").default(60).notNull(),
    dailyLimit: integer("daily_limit"), // null = unlimited
    customQuestions: jsonb("custom_questions").default([]).$type<
      Array<{
        id: string;
        type: "text" | "textarea" | "select" | "checkbox";
        label: string;
        required: boolean;
        options?: string[];
      }>
    >(),
    confirmationRedirectUrl: text("confirmation_redirect_url"),
    color: text("color").default("#3b82f6").notNull(), // calendar display color
    isSecret: boolean("is_secret").default(false).notNull(),
    isActive: boolean("is_active").default(true).notNull(),
    createdAt: timestamp("created_at", { withTimezone: true }).defaultNow().notNull(),
  },
  (t) => [
    uniqueIndex("meeting_type_org_slug_idx").on(t.orgId, t.slug),
    index("meeting_type_org_idx").on(t.orgId),
  ],
);

// ---------------------------------------------------------------------------
// Meeting Type ↔ Member assignment (for team scheduling)
// ---------------------------------------------------------------------------
export const meetingTypeMembers = pgTable(
  "meeting_type_members",
  {
    id: uuid("id").primaryKey().defaultRandom(),
    meetingTypeId: uuid("meeting_type_id")
      .references(() => meetingTypes.id, { onDelete: "cascade" })
      .notNull(),
    memberId: uuid("member_id")
      .references(() => members.id, { onDelete: "cascade" })
      .notNull(),
    weight: integer("weight").default(1).notNull(), // for weighted round-robin
  },
  (t) => [
    uniqueIndex("mtm_type_member_idx").on(t.meetingTypeId, t.memberId),
  ],
);

// ---------------------------------------------------------------------------
// Bookings
// ---------------------------------------------------------------------------
export const bookings = pgTable(
  "bookings",
  {
    id: uuid("id").primaryKey().defaultRandom(),
    orgId: uuid("org_id")
      .references(() => organizations.id, { onDelete: "cascade" })
      .notNull(),
    meetingTypeId: uuid("meeting_type_id")
      .references(() => meetingTypes.id, { onDelete: "set null" }),
    assignedMemberId: uuid("assigned_member_id")
      .references(() => members.id, { onDelete: "set null" }),
    guestName: text("guest_name").notNull(),
    guestEmail: text("guest_email").notNull(),
    guestPhone: text("guest_phone"),
    guestTimezone: text("guest_timezone").notNull(),
    customAnswers: jsonb("custom_answers").default({}).$type<
      Record<string, string | boolean>
    >(),
    startTime: timestamp("start_time", { withTimezone: true }).notNull(),
    endTime: timestamp("end_time", { withTimezone: true }).notNull(),
    status: text("status").default("confirmed").notNull(),
    // confirmed | cancelled | rescheduled | no_show | completed
    meetingLink: text("meeting_link"), // auto-generated Google Meet / Zoom link
    calendarEventId: text("calendar_event_id"), // Google Calendar event ID
    stripePaymentIntentId: text("stripe_payment_intent_id"),
    amountPaidCents: integer("amount_paid_cents"),
    cancellationReason: text("cancellation_reason"),
    sourceUrl: text("source_url"), // where the widget was embedded
    utmParams: jsonb("utm_params").default({}).$type<{
      source?: string;
      medium?: string;
      campaign?: string;
    }>(),
    rescheduleToken: text("reschedule_token").notNull(),
    cancelToken: text("cancel_token").notNull(),
    reminderSentAt: jsonb("reminder_sent_at").default({}).$type<{
      h24?: string; // ISO timestamp
      h1?: string;
    }>(),
    createdAt: timestamp("created_at", { withTimezone: true }).defaultNow().notNull(),
  },
  (t) => [
    index("booking_org_idx").on(t.orgId),
    index("booking_member_idx").on(t.assignedMemberId),
    index("booking_start_idx").on(t.startTime),
    index("booking_status_idx").on(t.status),
    uniqueIndex("booking_reschedule_token_idx").on(t.rescheduleToken),
    uniqueIndex("booking_cancel_token_idx").on(t.cancelToken),
  ],
);

// ---------------------------------------------------------------------------
// Widget Configurations
// ---------------------------------------------------------------------------
export const widgetConfigs = pgTable(
  "widget_configs",
  {
    id: uuid("id").primaryKey().defaultRandom(),
    orgId: uuid("org_id")
      .references(() => organizations.id, { onDelete: "cascade" })
      .notNull(),
    name: text("name").default("Default").notNull(),
    meetingTypeIds: jsonb("meeting_type_ids").default([]).$type<string[]>(),
    theme: jsonb("theme").default({}).$type<{
      primaryColor?: string;
      backgroundColor?: string;
      textColor?: string;
      fontFamily?: string;
      borderRadius?: number; // px
      accentColor?: string;
    }>(),
    customCss: text("custom_css"),
    displayMode: text("display_mode").default("inline").notNull(), // inline | popup | slide_in
    locale: text("locale").default("en").notNull(),
    showTimezoneSelector: boolean("show_timezone_selector").default(true).notNull(),
    showPoweredBy: boolean("show_powered_by").default(true).notNull(),
    createdAt: timestamp("created_at", { withTimezone: true }).defaultNow().notNull(),
  },
  (t) => [
    index("widget_config_org_idx").on(t.orgId),
  ],
);

// ---------------------------------------------------------------------------
// Notification Templates
// ---------------------------------------------------------------------------
export const notificationTemplates = pgTable(
  "notification_templates",
  {
    id: uuid("id").primaryKey().defaultRandom(),
    orgId: uuid("org_id")
      .references(() => organizations.id, { onDelete: "cascade" })
      .notNull(),
    type: text("type").notNull(),
    // confirmation | reminder_24h | reminder_1h | cancellation | reschedule
    subject: text("subject").notNull(),
    bodyHtml: text("body_html").notNull(),
    isActive: boolean("is_active").default(true).notNull(),
  },
  (t) => [
    uniqueIndex("notif_template_org_type_idx").on(t.orgId, t.type),
  ],
);

// ---------------------------------------------------------------------------
// Cached Calendar Events (from Google Calendar sync)
// ---------------------------------------------------------------------------
export const cachedCalendarEvents = pgTable(
  "cached_calendar_events",
  {
    id: uuid("id").primaryKey().defaultRandom(),
    calendarConnectionId: uuid("calendar_connection_id")
      .references(() => calendarConnections.id, { onDelete: "cascade" })
      .notNull(),
    externalEventId: text("external_event_id").notNull(),
    summary: text("summary"),
    startTime: timestamp("start_time", { withTimezone: true }).notNull(),
    endTime: timestamp("end_time", { withTimezone: true }).notNull(),
    status: text("status").default("confirmed").notNull(), // confirmed | tentative | cancelled
    isAllDay: boolean("is_all_day").default(false).notNull(),
    updatedAt: timestamp("updated_at", { withTimezone: true }).defaultNow().notNull(),
  },
  (t) => [
    index("cached_event_conn_idx").on(t.calendarConnectionId),
    index("cached_event_time_idx").on(t.startTime, t.endTime),
    uniqueIndex("cached_event_conn_ext_idx").on(t.calendarConnectionId, t.externalEventId),
  ],
);

// ---------------------------------------------------------------------------
// Page Views (widget analytics)
// ---------------------------------------------------------------------------
export const widgetPageViews = pgTable(
  "widget_page_views",
  {
    id: uuid("id").primaryKey().defaultRandom(),
    orgId: uuid("org_id")
      .references(() => organizations.id, { onDelete: "cascade" })
      .notNull(),
    widgetConfigId: uuid("widget_config_id"),
    meetingTypeId: uuid("meeting_type_id"),
    sourceUrl: text("source_url"),
    visitorTimezone: text("visitor_timezone"),
    device: text("device"), // desktop | mobile | tablet
    converted: boolean("converted").default(false).notNull(),
    viewedAt: timestamp("viewed_at", { withTimezone: true }).defaultNow().notNull(),
  },
  (t) => [
    index("widget_pv_org_idx").on(t.orgId),
    index("widget_pv_viewed_idx").on(t.viewedAt),
  ],
);

// ---------------------------------------------------------------------------
// Relations
// ---------------------------------------------------------------------------
export const organizationsRelations = relations(organizations, ({ many }) => ({
  members: many(members),
  meetingTypes: many(meetingTypes),
  bookings: many(bookings),
  widgetConfigs: many(widgetConfigs),
  notificationTemplates: many(notificationTemplates),
  widgetPageViews: many(widgetPageViews),
}));

export const membersRelations = relations(members, ({ one, many }) => ({
  organization: one(organizations, {
    fields: [members.orgId],
    references: [organizations.id],
  }),
  calendarConnections: many(calendarConnections),
  weeklyAvailability: many(weeklyAvailability),
  availabilityOverrides: many(availabilityOverrides),
  meetingTypeMembers: many(meetingTypeMembers),
  bookings: many(bookings),
}));

export const calendarConnectionsRelations = relations(calendarConnections, ({ one, many }) => ({
  member: one(members, {
    fields: [calendarConnections.memberId],
    references: [members.id],
  }),
  cachedEvents: many(cachedCalendarEvents),
}));

export const weeklyAvailabilityRelations = relations(weeklyAvailability, ({ one }) => ({
  member: one(members, {
    fields: [weeklyAvailability.memberId],
    references: [members.id],
  }),
}));

export const availabilityOverridesRelations = relations(availabilityOverrides, ({ one }) => ({
  member: one(members, {
    fields: [availabilityOverrides.memberId],
    references: [members.id],
  }),
}));

export const meetingTypesRelations = relations(meetingTypes, ({ one, many }) => ({
  organization: one(organizations, {
    fields: [meetingTypes.orgId],
    references: [organizations.id],
  }),
  meetingTypeMembers: many(meetingTypeMembers),
  bookings: many(bookings),
}));

export const meetingTypeMembersRelations = relations(meetingTypeMembers, ({ one }) => ({
  meetingType: one(meetingTypes, {
    fields: [meetingTypeMembers.meetingTypeId],
    references: [meetingTypes.id],
  }),
  member: one(members, {
    fields: [meetingTypeMembers.memberId],
    references: [members.id],
  }),
}));

export const bookingsRelations = relations(bookings, ({ one }) => ({
  organization: one(organizations, {
    fields: [bookings.orgId],
    references: [organizations.id],
  }),
  meetingType: one(meetingTypes, {
    fields: [bookings.meetingTypeId],
    references: [meetingTypes.id],
  }),
  assignedMember: one(members, {
    fields: [bookings.assignedMemberId],
    references: [members.id],
  }),
}));

export const widgetConfigsRelations = relations(widgetConfigs, ({ one }) => ({
  organization: one(organizations, {
    fields: [widgetConfigs.orgId],
    references: [organizations.id],
  }),
}));

export const notificationTemplatesRelations = relations(notificationTemplates, ({ one }) => ({
  organization: one(organizations, {
    fields: [notificationTemplates.orgId],
    references: [organizations.id],
  }),
}));

export const cachedCalendarEventsRelations = relations(cachedCalendarEvents, ({ one }) => ({
  calendarConnection: one(calendarConnections, {
    fields: [cachedCalendarEvents.calendarConnectionId],
    references: [calendarConnections.id],
  }),
}));

export const widgetPageViewsRelations = relations(widgetPageViews, ({ one }) => ({
  organization: one(organizations, {
    fields: [widgetPageViews.orgId],
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
  timezone TEXT NOT NULL DEFAULT 'America/New_York',
  logo_url TEXT,
  settings JSONB DEFAULT '{}',
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Members
CREATE TABLE members (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  org_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
  clerk_user_id TEXT NOT NULL,
  name TEXT NOT NULL,
  email TEXT NOT NULL,
  bio TEXT,
  avatar_url TEXT,
  timezone TEXT NOT NULL DEFAULT 'America/New_York',
  is_active BOOLEAN NOT NULL DEFAULT true,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE UNIQUE INDEX members_org_clerk_idx ON members(org_id, clerk_user_id);

-- Calendar Connections
CREATE TABLE calendar_connections (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  member_id UUID NOT NULL REFERENCES members(id) ON DELETE CASCADE,
  provider TEXT NOT NULL DEFAULT 'google',
  access_token TEXT NOT NULL,
  refresh_token TEXT NOT NULL,
  token_expires_at TIMESTAMPTZ,
  calendar_id TEXT NOT NULL DEFAULT 'primary',
  watch_channel_id TEXT,
  watch_resource_id TEXT,
  watch_expires_at TIMESTAMPTZ,
  sync_enabled BOOLEAN NOT NULL DEFAULT true,
  last_synced_at TIMESTAMPTZ,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX cal_conn_member_idx ON calendar_connections(member_id);

-- Weekly Availability
CREATE TABLE weekly_availability (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  member_id UUID NOT NULL REFERENCES members(id) ON DELETE CASCADE,
  day_of_week INTEGER NOT NULL,
  start_time TIME NOT NULL,
  end_time TIME NOT NULL,
  is_enabled BOOLEAN NOT NULL DEFAULT true
);
CREATE INDEX weekly_avail_member_idx ON weekly_availability(member_id);
CREATE UNIQUE INDEX weekly_avail_member_day_idx ON weekly_availability(member_id, day_of_week);

-- Availability Overrides
CREATE TABLE availability_overrides (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  member_id UUID NOT NULL REFERENCES members(id) ON DELETE CASCADE,
  override_date DATE NOT NULL,
  type TEXT NOT NULL,
  start_time TIME,
  end_time TIME
);
CREATE UNIQUE INDEX avail_override_member_date_idx ON availability_overrides(member_id, override_date);

-- Meeting Types
CREATE TABLE meeting_types (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  org_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
  name TEXT NOT NULL,
  slug TEXT NOT NULL,
  description TEXT,
  duration_minutes INTEGER NOT NULL DEFAULT 30,
  buffer_before INTEGER NOT NULL DEFAULT 0,
  buffer_after INTEGER NOT NULL DEFAULT 0,
  location_type TEXT NOT NULL DEFAULT 'google_meet',
  location_config JSONB DEFAULT '{}',
  price_cents INTEGER,
  currency TEXT DEFAULT 'usd',
  scheduling_type TEXT NOT NULL DEFAULT 'individual',
  min_notice_hours INTEGER NOT NULL DEFAULT 1,
  max_days_ahead INTEGER NOT NULL DEFAULT 60,
  daily_limit INTEGER,
  custom_questions JSONB DEFAULT '[]',
  confirmation_redirect_url TEXT,
  color TEXT NOT NULL DEFAULT '#3b82f6',
  is_secret BOOLEAN NOT NULL DEFAULT false,
  is_active BOOLEAN NOT NULL DEFAULT true,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE UNIQUE INDEX meeting_type_org_slug_idx ON meeting_types(org_id, slug);
CREATE INDEX meeting_type_org_idx ON meeting_types(org_id);

-- Meeting Type Members
CREATE TABLE meeting_type_members (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  meeting_type_id UUID NOT NULL REFERENCES meeting_types(id) ON DELETE CASCADE,
  member_id UUID NOT NULL REFERENCES members(id) ON DELETE CASCADE,
  weight INTEGER NOT NULL DEFAULT 1
);
CREATE UNIQUE INDEX mtm_type_member_idx ON meeting_type_members(meeting_type_id, member_id);

-- Bookings
CREATE TABLE bookings (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  org_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
  meeting_type_id UUID REFERENCES meeting_types(id) ON DELETE SET NULL,
  assigned_member_id UUID REFERENCES members(id) ON DELETE SET NULL,
  guest_name TEXT NOT NULL,
  guest_email TEXT NOT NULL,
  guest_phone TEXT,
  guest_timezone TEXT NOT NULL,
  custom_answers JSONB DEFAULT '{}',
  start_time TIMESTAMPTZ NOT NULL,
  end_time TIMESTAMPTZ NOT NULL,
  status TEXT NOT NULL DEFAULT 'confirmed',
  meeting_link TEXT,
  calendar_event_id TEXT,
  stripe_payment_intent_id TEXT,
  amount_paid_cents INTEGER,
  cancellation_reason TEXT,
  source_url TEXT,
  utm_params JSONB DEFAULT '{}',
  reschedule_token TEXT NOT NULL,
  cancel_token TEXT NOT NULL,
  reminder_sent_at JSONB DEFAULT '{}',
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX booking_org_idx ON bookings(org_id);
CREATE INDEX booking_member_idx ON bookings(assigned_member_id);
CREATE INDEX booking_start_idx ON bookings(start_time);
CREATE INDEX booking_status_idx ON bookings(status);
CREATE UNIQUE INDEX booking_reschedule_token_idx ON bookings(reschedule_token);
CREATE UNIQUE INDEX booking_cancel_token_idx ON bookings(cancel_token);

-- Widget Configurations
CREATE TABLE widget_configs (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  org_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
  name TEXT NOT NULL DEFAULT 'Default',
  meeting_type_ids JSONB DEFAULT '[]',
  theme JSONB DEFAULT '{}',
  custom_css TEXT,
  display_mode TEXT NOT NULL DEFAULT 'inline',
  locale TEXT NOT NULL DEFAULT 'en',
  show_timezone_selector BOOLEAN NOT NULL DEFAULT true,
  show_powered_by BOOLEAN NOT NULL DEFAULT true,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX widget_config_org_idx ON widget_configs(org_id);

-- Notification Templates
CREATE TABLE notification_templates (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  org_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
  type TEXT NOT NULL,
  subject TEXT NOT NULL,
  body_html TEXT NOT NULL,
  is_active BOOLEAN NOT NULL DEFAULT true
);
CREATE UNIQUE INDEX notif_template_org_type_idx ON notification_templates(org_id, type);

-- Cached Calendar Events
CREATE TABLE cached_calendar_events (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  calendar_connection_id UUID NOT NULL REFERENCES calendar_connections(id) ON DELETE CASCADE,
  external_event_id TEXT NOT NULL,
  summary TEXT,
  start_time TIMESTAMPTZ NOT NULL,
  end_time TIMESTAMPTZ NOT NULL,
  status TEXT NOT NULL DEFAULT 'confirmed',
  is_all_day BOOLEAN NOT NULL DEFAULT false,
  updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX cached_event_conn_idx ON cached_calendar_events(calendar_connection_id);
CREATE INDEX cached_event_time_idx ON cached_calendar_events(start_time, end_time);
CREATE UNIQUE INDEX cached_event_conn_ext_idx ON cached_calendar_events(calendar_connection_id, external_event_id);

-- Widget Page Views
CREATE TABLE widget_page_views (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  org_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
  widget_config_id UUID,
  meeting_type_id UUID,
  source_url TEXT,
  visitor_timezone TEXT,
  device TEXT,
  converted BOOLEAN NOT NULL DEFAULT false,
  viewed_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX widget_pv_org_idx ON widget_page_views(org_id);
CREATE INDEX widget_pv_viewed_idx ON widget_page_views(viewed_at);
```

---

## Architecture Deep-Dives

### 1. Availability Computation Engine

The core scheduling algorithm computes available time slots for a given date range by merging three data sources: configured weekly schedule, Google Calendar busy times, and existing bookings. Slots are generated in the member's timezone, then converted to UTC for storage and the guest's timezone for display.

```typescript
// src/server/services/availability.ts

import { and, eq, gte, lte, not, inArray } from "drizzle-orm";
import {
  addMinutes,
  startOfDay,
  endOfDay,
  eachDayOfInterval,
  format,
  setHours,
  setMinutes,
  isAfter,
  isBefore,
  areIntervalsOverlapping,
} from "date-fns";
import { toZonedTime, fromZonedTime } from "date-fns-tz";
import type { MeetingType, Member } from "../db/schema";

interface TimeSlot {
  start: Date;   // UTC
  end: Date;     // UTC
}

interface AvailableSlot {
  start: string; // ISO 8601 UTC
  end: string;
  memberIds: string[]; // which members are available for this slot
}

interface BusyBlock {
  start: Date; // UTC
  end: Date;
}

export async function getAvailableSlots(
  db: DrizzleDB,
  meetingType: typeof MeetingType.$inferSelect,
  members: (typeof Member.$inferSelect)[],
  dateRange: { start: Date; end: Date },
  guestTimezone: string,
): Promise<AvailableSlot[]> {
  const slotDuration = meetingType.durationMinutes;
  const bufferBefore = meetingType.bufferBefore;
  const bufferAfter = meetingType.bufferAfter;
  const stepMinutes = 15; // slot granularity

  const allSlots: AvailableSlot[] = [];

  for (const member of members) {
    const memberTz = member.timezone;

    // 1. Fetch weekly availability for this member
    const weeklySchedule = await db.query.weeklyAvailability.findMany({
      where: and(
        eq(weeklyAvailability.memberId, member.id),
        eq(weeklyAvailability.isEnabled, true),
      ),
    });

    // 2. Fetch date overrides in the range
    const overrides = await db.query.availabilityOverrides.findMany({
      where: and(
        eq(availabilityOverrides.memberId, member.id),
        gte(availabilityOverrides.overrideDate, format(dateRange.start, "yyyy-MM-dd")),
        lte(availabilityOverrides.overrideDate, format(dateRange.end, "yyyy-MM-dd")),
      ),
    });

    // 3. Fetch existing bookings in the range (include buffer)
    const existingBookings = await db.query.bookings.findMany({
      where: and(
        eq(bookings.assignedMemberId, member.id),
        not(inArray(bookings.status, ["cancelled"])),
        gte(bookings.startTime, dateRange.start),
        lte(bookings.endTime, dateRange.end),
      ),
    });

    // 4. Fetch cached Google Calendar events in the range
    const calConnections = await db.query.calendarConnections.findMany({
      where: eq(calendarConnections.memberId, member.id),
    });
    const connIds = calConnections.map((c) => c.id);
    const calendarEvents = connIds.length > 0
      ? await db.query.cachedCalendarEvents.findMany({
          where: and(
            inArray(cachedCalendarEvents.calendarConnectionId, connIds),
            not(eq(cachedCalendarEvents.status, "cancelled")),
            gte(cachedCalendarEvents.startTime, dateRange.start),
            lte(cachedCalendarEvents.endTime, dateRange.end),
          ),
        })
      : [];

    // 5. Build busy blocks array (bookings + calendar events with buffers)
    const busyBlocks: BusyBlock[] = [
      ...existingBookings.map((b) => ({
        start: addMinutes(b.startTime, -bufferBefore),
        end: addMinutes(b.endTime, bufferAfter),
      })),
      ...calendarEvents.map((e) => ({
        start: e.startTime,
        end: e.endTime,
      })),
    ];

    // 6. Generate candidate slots per day
    const days = eachDayOfInterval({ start: dateRange.start, end: dateRange.end });

    for (const day of days) {
      const dayInMemberTz = toZonedTime(day, memberTz);
      const dayOfWeek = dayInMemberTz.getDay();
      const dateStr = format(dayInMemberTz, "yyyy-MM-dd");

      // Check for override
      const override = overrides.find((o) => o.overrideDate === dateStr);
      let windowStart: Date;
      let windowEnd: Date;

      if (override) {
        if (override.type === "unavailable") continue; // skip this day
        // custom hours
        const [sh, sm] = override.startTime!.split(":").map(Number);
        const [eh, em] = override.endTime!.split(":").map(Number);
        windowStart = fromZonedTime(setMinutes(setHours(dayInMemberTz, sh), sm), memberTz);
        windowEnd = fromZonedTime(setMinutes(setHours(dayInMemberTz, eh), em), memberTz);
      } else {
        // Use weekly schedule
        const schedule = weeklySchedule.find((s) => s.dayOfWeek === dayOfWeek);
        if (!schedule) continue; // no availability this day
        const [sh, sm] = schedule.startTime.split(":").map(Number);
        const [eh, em] = schedule.endTime.split(":").map(Number);
        windowStart = fromZonedTime(setMinutes(setHours(dayInMemberTz, sh), sm), memberTz);
        windowEnd = fromZonedTime(setMinutes(setHours(dayInMemberTz, eh), em), memberTz);
      }

      // Generate slots within window
      let cursor = windowStart;
      while (isAfter(windowEnd, addMinutes(cursor, slotDuration))) {
        const slotStart = cursor;
        const slotEnd = addMinutes(cursor, slotDuration);
        const slotWithBuffer = {
          start: addMinutes(slotStart, -bufferBefore),
          end: addMinutes(slotEnd, bufferAfter),
        };

        // Check min notice
        if (isAfter(addMinutes(new Date(), meetingType.minNoticeHours * 60), slotStart)) {
          cursor = addMinutes(cursor, stepMinutes);
          continue;
        }

        // Check against busy blocks
        const hasConflict = busyBlocks.some((busy) =>
          areIntervalsOverlapping(
            { start: slotWithBuffer.start, end: slotWithBuffer.end },
            { start: busy.start, end: busy.end },
          ),
        );

        if (!hasConflict) {
          // Check daily limit
          const dayBookingCount = existingBookings.filter((b) => {
            const bDay = format(toZonedTime(b.startTime, memberTz), "yyyy-MM-dd");
            return bDay === dateStr;
          }).length;

          if (!meetingType.dailyLimit || dayBookingCount < meetingType.dailyLimit) {
            allSlots.push({
              start: slotStart.toISOString(),
              end: slotEnd.toISOString(),
              memberIds: [member.id],
            });
          }
        }

        cursor = addMinutes(cursor, stepMinutes);
      }
    }
  }

  // Merge slots across members (same start/end → combine memberIds)
  const merged = new Map<string, AvailableSlot>();
  for (const slot of allSlots) {
    const key = `${slot.start}_${slot.end}`;
    const existing = merged.get(key);
    if (existing) {
      existing.memberIds.push(...slot.memberIds);
    } else {
      merged.set(key, { ...slot });
    }
  }

  return Array.from(merged.values()).sort(
    (a, b) => new Date(a.start).getTime() - new Date(b.start).getTime(),
  );
}
```

### 2. Embeddable Widget Architecture (Preact + Shadow DOM)

The widget is a standalone Preact application that attaches to the host page via Shadow DOM. It reads configuration from a `data-*` attribute or a global `window.ScheduleKit` object, fetches available slots from the public API, and renders the booking flow. CSS variables allow theming from the host page.

```typescript
// widget/src/index.ts

import { h, render } from "preact";
import { App } from "./App";
import { defaultTheme, type WidgetTheme } from "./theme";

interface ScheduleKitConfig {
  orgSlug: string;
  meetingTypeSlug?: string;
  displayMode: "inline" | "popup";
  theme?: Partial<WidgetTheme>;
  locale?: string;
  onBooked?: (booking: BookingResult) => void;
  apiBaseUrl?: string;
}

interface BookingResult {
  id: string;
  startTime: string;
  endTime: string;
  meetingLink?: string;
}

const API_BASE = "https://app.schedulekit.io/api/public";

function mount(config: ScheduleKitConfig) {
  // Find or create container
  const scriptTag = document.currentScript as HTMLScriptElement | null;
  let container: HTMLElement;

  if (config.displayMode === "popup") {
    container = document.createElement("div");
    container.id = "schedulekit-popup-root";
    document.body.appendChild(container);
  } else {
    // Inline: look for data-schedulekit-container or insert after script tag
    container =
      document.querySelector("[data-schedulekit-container]") ||
      scriptTag?.parentElement ||
      document.body;
  }

  // Create Shadow DOM for style isolation
  const shadowHost = document.createElement("div");
  shadowHost.id = "schedulekit-widget";
  container.appendChild(shadowHost);

  const shadow = shadowHost.attachShadow({ mode: "open" });

  // Inject styles into shadow root
  const styleEl = document.createElement("style");
  styleEl.textContent = getWidgetCSS(config.theme);
  shadow.appendChild(styleEl);

  // Create mount point inside shadow
  const mountPoint = document.createElement("div");
  mountPoint.className = "sk-root";
  shadow.appendChild(mountPoint);

  // Render Preact app into shadow DOM
  render(
    h(App, {
      orgSlug: config.orgSlug,
      meetingTypeSlug: config.meetingTypeSlug,
      displayMode: config.displayMode,
      theme: { ...defaultTheme, ...config.theme },
      locale: config.locale || "en",
      onBooked: config.onBooked,
      apiBaseUrl: config.apiBaseUrl || API_BASE,
    }),
    mountPoint,
  );
}

function getWidgetCSS(themeOverrides?: Partial<WidgetTheme>): string {
  const theme = { ...defaultTheme, ...themeOverrides };
  return `
    :host {
      --sk-primary: ${theme.primaryColor};
      --sk-bg: ${theme.backgroundColor};
      --sk-text: ${theme.textColor};
      --sk-accent: ${theme.accentColor};
      --sk-radius: ${theme.borderRadius}px;
      --sk-font: ${theme.fontFamily};
    }
    .sk-root {
      font-family: var(--sk-font), -apple-system, BlinkMacSystemFont, sans-serif;
      color: var(--sk-text);
      background: var(--sk-bg);
      border-radius: var(--sk-radius);
      max-width: 420px;
      margin: 0 auto;
    }
    .sk-root * { box-sizing: border-box; }
    /* ... 200+ lines of widget CSS omitted for brevity ... */
  `;
}

// Auto-initialize from script tag attributes
function autoInit() {
  const scripts = document.querySelectorAll("script[data-schedulekit]");
  scripts.forEach((script) => {
    const el = script as HTMLScriptElement;
    mount({
      orgSlug: el.dataset.org || "",
      meetingTypeSlug: el.dataset.meetingType,
      displayMode: (el.dataset.mode as "inline" | "popup") || "inline",
      theme: el.dataset.theme ? JSON.parse(el.dataset.theme) : undefined,
    });
  });
}

// Expose global API
(window as any).ScheduleKit = { mount };

// Auto-init on DOM ready
if (document.readyState === "loading") {
  document.addEventListener("DOMContentLoaded", autoInit);
} else {
  autoInit();
}
```

### 3. Google Calendar OAuth + Watch Notifications

Google Calendar integration uses OAuth 2.0 for authorization and push notifications (watch channels) for real-time sync. When a calendar event changes, Google sends a webhook to our endpoint, which triggers a sync of the affected calendar.

```typescript
// src/server/services/google-calendar.ts

import { google } from "googleapis";
import { and, eq } from "drizzle-orm";
import { encrypt, decrypt } from "../lib/encryption";

const oauth2Client = new google.auth.OAuth2(
  process.env.GOOGLE_CLIENT_ID,
  process.env.GOOGLE_CLIENT_SECRET,
  process.env.GOOGLE_REDIRECT_URI,
);

// Step 1: Generate OAuth URL for user to authorize
export function getAuthUrl(state: string): string {
  return oauth2Client.generateAuthUrl({
    access_type: "offline",
    prompt: "consent",
    scope: [
      "https://www.googleapis.com/auth/calendar.readonly",
      "https://www.googleapis.com/auth/calendar.events",
    ],
    state,
  });
}

// Step 2: Exchange code for tokens and save connection
export async function handleOAuthCallback(
  db: DrizzleDB,
  memberId: string,
  code: string,
): Promise<void> {
  const { tokens } = await oauth2Client.getToken(code);
  if (!tokens.access_token || !tokens.refresh_token) {
    throw new Error("Missing tokens from Google OAuth");
  }

  // Encrypt tokens before storage
  const encryptedAccess = encrypt(tokens.access_token);
  const encryptedRefresh = encrypt(tokens.refresh_token);

  const [connection] = await db
    .insert(calendarConnections)
    .values({
      memberId,
      provider: "google",
      accessToken: encryptedAccess,
      refreshToken: encryptedRefresh,
      tokenExpiresAt: tokens.expiry_date ? new Date(tokens.expiry_date) : null,
      calendarId: "primary",
      syncEnabled: true,
    })
    .returning();

  // Initial sync
  await syncCalendarEvents(db, connection.id);

  // Set up watch channel for push notifications
  await setupWatchChannel(db, connection.id);
}

// Step 3: Sync events from Google Calendar to local cache
export async function syncCalendarEvents(
  db: DrizzleDB,
  connectionId: string,
): Promise<void> {
  const connection = await db.query.calendarConnections.findFirst({
    where: eq(calendarConnections.id, connectionId),
  });
  if (!connection) throw new Error("Connection not found");

  const auth = getAuthClient(connection);
  const calendar = google.calendar({ version: "v3", auth });

  // Fetch events for the next 90 days
  const now = new Date();
  const maxDate = new Date();
  maxDate.setDate(maxDate.getDate() + 90);

  const response = await calendar.events.list({
    calendarId: connection.calendarId,
    timeMin: now.toISOString(),
    timeMax: maxDate.toISOString(),
    singleEvents: true,
    orderBy: "startTime",
    maxResults: 2500,
  });

  const events = response.data.items || [];

  // Upsert events into cached_calendar_events
  for (const event of events) {
    if (!event.id || !event.start || !event.end) continue;
    const startTime = event.start.dateTime
      ? new Date(event.start.dateTime)
      : new Date(event.start.date!);
    const endTime = event.end.dateTime
      ? new Date(event.end.dateTime)
      : new Date(event.end.date!);

    await db
      .insert(cachedCalendarEvents)
      .values({
        calendarConnectionId: connectionId,
        externalEventId: event.id,
        summary: event.summary || null,
        startTime,
        endTime,
        status: event.status || "confirmed",
        isAllDay: !!event.start.date,
      })
      .onConflictDoUpdate({
        target: [
          cachedCalendarEvents.calendarConnectionId,
          cachedCalendarEvents.externalEventId,
        ],
        set: {
          summary: event.summary || null,
          startTime,
          endTime,
          status: event.status || "confirmed",
          isAllDay: !!event.start.date,
          updatedAt: new Date(),
        },
      });
  }

  // Update last synced timestamp
  await db
    .update(calendarConnections)
    .set({ lastSyncedAt: new Date() })
    .where(eq(calendarConnections.id, connectionId));
}

// Step 4: Set up Google Calendar push notifications
export async function setupWatchChannel(
  db: DrizzleDB,
  connectionId: string,
): Promise<void> {
  const connection = await db.query.calendarConnections.findFirst({
    where: eq(calendarConnections.id, connectionId),
  });
  if (!connection) return;

  const auth = getAuthClient(connection);
  const calendar = google.calendar({ version: "v3", auth });

  const channelId = `schedulekit-${connectionId}-${Date.now()}`;
  const response = await calendar.events.watch({
    calendarId: connection.calendarId,
    requestBody: {
      id: channelId,
      type: "web_hook",
      address: `${process.env.NEXT_PUBLIC_APP_URL}/api/webhooks/google-calendar`,
      expiration: String(Date.now() + 7 * 24 * 60 * 60 * 1000), // 7 days
    },
  });

  await db
    .update(calendarConnections)
    .set({
      watchChannelId: channelId,
      watchResourceId: response.data.resourceId || null,
      watchExpiresAt: response.data.expiration
        ? new Date(Number(response.data.expiration))
        : null,
    })
    .where(eq(calendarConnections.id, connectionId));
}

// Webhook handler: Google notifies us of calendar changes
export async function handleCalendarWebhook(
  db: DrizzleDB,
  channelId: string,
  resourceId: string,
): Promise<void> {
  // Find the connection by channel ID
  const connection = await db.query.calendarConnections.findFirst({
    where: and(
      eq(calendarConnections.watchChannelId, channelId),
      eq(calendarConnections.watchResourceId, resourceId),
    ),
  });
  if (!connection) return;

  // Re-sync events
  await syncCalendarEvents(db, connection.id);
}

function getAuthClient(connection: typeof calendarConnections.$inferSelect) {
  const client = new google.auth.OAuth2(
    process.env.GOOGLE_CLIENT_ID,
    process.env.GOOGLE_CLIENT_SECRET,
    process.env.GOOGLE_REDIRECT_URI,
  );
  client.setCredentials({
    access_token: decrypt(connection.accessToken),
    refresh_token: decrypt(connection.refreshToken),
  });
  return client;
}
```

### 4. Booking Flow with Conflict Prevention

The booking creation endpoint uses a database transaction with row-level locking to prevent double-bookings. It also creates a Google Calendar event for the host and sends confirmation emails to both parties.

```typescript
// src/server/routers/public-booking.ts

import { z } from "zod";
import { publicProcedure, router } from "../trpc";
import { nanoid } from "nanoid";
import { and, eq, gte, lte, not } from "drizzle-orm";
import { addMinutes } from "date-fns";
import { sendBookingConfirmation } from "../services/email";
import { createCalendarEvent } from "../services/google-calendar";

export const publicBookingRouter = router({
  // Public: get available slots (no auth required)
  getAvailableSlots: publicProcedure
    .input(z.object({
      orgSlug: z.string(),
      meetingTypeSlug: z.string(),
      startDate: z.string(), // ISO date
      endDate: z.string(),
      timezone: z.string(),
    }))
    .query(async ({ ctx, input }) => {
      const org = await ctx.db.query.organizations.findFirst({
        where: eq(organizations.slug, input.orgSlug),
      });
      if (!org) throw new Error("Organization not found");

      const meetingType = await ctx.db.query.meetingTypes.findFirst({
        where: and(
          eq(meetingTypes.orgId, org.id),
          eq(meetingTypes.slug, input.meetingTypeSlug),
          eq(meetingTypes.isActive, true),
        ),
      });
      if (!meetingType) throw new Error("Meeting type not found");

      // Get assigned members
      const typeMembers = await ctx.db.query.meetingTypeMembers.findMany({
        where: eq(meetingTypeMembers.meetingTypeId, meetingType.id),
        with: { member: true },
      });
      const memberList = typeMembers.map((tm) => tm.member);

      return getAvailableSlots(
        ctx.db,
        meetingType,
        memberList,
        { start: new Date(input.startDate), end: new Date(input.endDate) },
        input.timezone,
      );
    }),

  // Public: create a booking (no auth required)
  createBooking: publicProcedure
    .input(z.object({
      orgSlug: z.string(),
      meetingTypeSlug: z.string(),
      startTime: z.string(), // ISO datetime
      guestName: z.string().min(1).max(200),
      guestEmail: z.string().email(),
      guestPhone: z.string().optional(),
      guestTimezone: z.string(),
      customAnswers: z.record(z.union([z.string(), z.boolean()])).optional(),
      memberId: z.string().uuid().optional(), // for round-robin override
      sourceUrl: z.string().optional(),
      utmParams: z.object({
        source: z.string().optional(),
        medium: z.string().optional(),
        campaign: z.string().optional(),
      }).optional(),
    }))
    .mutation(async ({ ctx, input }) => {
      const org = await ctx.db.query.organizations.findFirst({
        where: eq(organizations.slug, input.orgSlug),
      });
      if (!org) throw new Error("Organization not found");

      const meetingType = await ctx.db.query.meetingTypes.findFirst({
        where: and(
          eq(meetingTypes.orgId, org.id),
          eq(meetingTypes.slug, input.meetingTypeSlug),
        ),
      });
      if (!meetingType) throw new Error("Meeting type not found");

      const startTime = new Date(input.startTime);
      const endTime = addMinutes(startTime, meetingType.durationMinutes);

      // Select member (first assigned, or round-robin in v2)
      const typeMembers = await ctx.db.query.meetingTypeMembers.findMany({
        where: eq(meetingTypeMembers.meetingTypeId, meetingType.id),
        with: { member: true },
      });
      const assignedMember = input.memberId
        ? typeMembers.find((tm) => tm.member.id === input.memberId)?.member
        : typeMembers[0]?.member;
      if (!assignedMember) throw new Error("No available member");

      // Transaction with conflict check
      return ctx.db.transaction(async (tx) => {
        // Check for conflicting bookings (with buffer)
        const bufferStart = addMinutes(startTime, -meetingType.bufferBefore);
        const bufferEnd = addMinutes(endTime, meetingType.bufferAfter);

        const conflicts = await tx.query.bookings.findMany({
          where: and(
            eq(bookings.assignedMemberId, assignedMember.id),
            not(eq(bookings.status, "cancelled")),
            lte(bookings.startTime, bufferEnd),
            gte(bookings.endTime, bufferStart),
          ),
        });

        if (conflicts.length > 0) {
          throw new Error("Time slot is no longer available");
        }

        const rescheduleToken = nanoid(32);
        const cancelToken = nanoid(32);

        // Create booking
        const [booking] = await tx
          .insert(bookings)
          .values({
            orgId: org.id,
            meetingTypeId: meetingType.id,
            assignedMemberId: assignedMember.id,
            guestName: input.guestName,
            guestEmail: input.guestEmail,
            guestPhone: input.guestPhone,
            guestTimezone: input.guestTimezone,
            customAnswers: input.customAnswers || {},
            startTime,
            endTime,
            status: "confirmed",
            rescheduleToken,
            cancelToken,
            sourceUrl: input.sourceUrl,
            utmParams: input.utmParams || {},
          })
          .returning();

        // Create Google Calendar event for host (outside tx, non-critical)
        createCalendarEvent(tx, booking.id, assignedMember.id, {
          summary: `${meetingType.name} with ${input.guestName}`,
          startTime,
          endTime,
          guestEmail: input.guestEmail,
          description: `Booked via ScheduleKit\n\nGuest: ${input.guestName} (${input.guestEmail})`,
        }).catch((err) => console.error("Calendar event creation failed:", err));

        // Send confirmation emails (outside tx)
        sendBookingConfirmation({
          booking,
          meetingType,
          member: assignedMember,
          org,
        }).catch((err) => console.error("Confirmation email failed:", err));

        return {
          id: booking.id,
          startTime: booking.startTime.toISOString(),
          endTime: booking.endTime.toISOString(),
          meetingLink: booking.meetingLink,
          rescheduleUrl: `${process.env.NEXT_PUBLIC_APP_URL}/reschedule/${rescheduleToken}`,
          cancelUrl: `${process.env.NEXT_PUBLIC_APP_URL}/cancel/${cancelToken}`,
        };
      });
    }),
});
```

---

## Phase Breakdown

### Phase 1 — Project Scaffold + Auth + DB (Days 1–4)

**Day 1: Repository and framework setup**
```bash
npx create-next-app@latest schedulekit --ts --tailwind --app --src-dir
cd schedulekit
npm i drizzle-orm @neondatabase/serverless
npm i -D drizzle-kit
npm i @trpc/server @trpc/client @trpc/next @trpc/react-query superjson zod
npm i @clerk/nextjs
```
- `src/app/layout.tsx` — Clerk provider, tRPC provider, global layout
- `src/app/(dashboard)/layout.tsx` — authenticated shell with sidebar
- `src/server/db/index.ts` — Neon connection pool
- `src/server/db/schema.ts` — organizations, members tables
- `drizzle.config.ts` — Neon connection string
- `.env.local` — `DATABASE_URL`, `CLERK_*`, `NEXT_PUBLIC_CLERK_*`

**Day 2: Full database schema**
- `src/server/db/schema.ts` — all 12 tables: organizations, members, calendarConnections, weeklyAvailability, availabilityOverrides, meetingTypes, meetingTypeMembers, bookings, widgetConfigs, notificationTemplates, cachedCalendarEvents, widgetPageViews
- All `relations()` blocks
- Run `npx drizzle-kit push` to apply schema

**Day 3: tRPC setup and org initialization**
- `src/server/trpc.ts` — base context, auth middleware, org-scoped procedure
- `src/server/routers/_app.ts` — root router with sub-routers
- `src/server/routers/org.ts` — getOrCreate org, update settings
- `src/server/routers/member.ts` — CRUD for team members
- `src/app/api/trpc/[trpc]/route.ts` — tRPC handler

**Day 4: Auth flows and org seed**
- `src/app/(auth)/sign-in/[[...sign-in]]/page.tsx` — Clerk sign-in
- `src/app/(auth)/sign-up/[[...sign-up]]/page.tsx` — Clerk sign-up
- `src/server/lib/org-seed.ts` — on first login: create org, create member, seed default weekly availability (Mon–Fri 9–5), create default meeting type ("30-Minute Meeting")
- Verify: sign up → org + member + defaults created

### Phase 2 — Availability Engine + Calendar Sync (Days 5–10)

**Day 5: Weekly availability CRUD**
- `src/server/routers/availability.ts` — get/set weekly schedule, toggle days
- `src/app/(dashboard)/availability/page.tsx` — weekly grid editor (click to toggle time blocks)
- `src/components/availability/WeeklyScheduleEditor.tsx` — grid component with start/end time pickers per day

**Day 6: Availability overrides**
- `src/server/routers/availability.ts` — add override (unavailable date or custom hours), delete override
- `src/app/(dashboard)/availability/page.tsx` — calendar showing overrides, click date to add exception
- `src/components/availability/OverrideCalendar.tsx` — react-day-picker with override markers

**Day 7: Google Calendar OAuth flow**
- `src/server/services/google-calendar.ts` — OAuth URL generation, callback handler, token encryption
- `src/server/lib/encryption.ts` — AES-256-GCM encrypt/decrypt for tokens
- `src/app/api/auth/google-calendar/route.ts` — redirect to Google OAuth
- `src/app/api/auth/google-calendar/callback/route.ts` — exchange code, save connection
- `src/app/(dashboard)/integrations/page.tsx` — "Connect Google Calendar" button, connected accounts list

**Day 8: Calendar event sync**
- `src/server/services/google-calendar.ts` — syncCalendarEvents function (fetch events, upsert to cache)
- Initial sync on OAuth callback
- `src/server/routers/calendar.ts` — manual sync trigger, list connections, disconnect

**Day 9: Google Calendar watch notifications**
- `src/server/services/google-calendar.ts` — setupWatchChannel, channel renewal
- `src/app/api/webhooks/google-calendar/route.ts` — handle push notifications, trigger re-sync
- `src/server/cron/renew-watch-channels.ts` — renew channels expiring in 24h (Vercel cron)
- `vercel.json` — cron job: `renew-watch-channels` every 6 hours

**Day 10: Availability computation engine**
- `src/server/services/availability.ts` — getAvailableSlots function (merge weekly schedule + overrides + calendar events + existing bookings)
- `src/server/routers/public-booking.ts` — public `getAvailableSlots` endpoint (no auth)
- Unit tests for slot generation logic:
  - Basic slots from weekly schedule
  - Slots with calendar conflicts removed
  - Buffer time subtracted
  - Min notice enforcement
  - Daily limit enforcement
  - Date override handling

### Phase 3 — Meeting Types + Booking Flow (Days 11–16)

**Day 11: Meeting type CRUD**
- `src/server/routers/meeting-type.ts` — create, update, delete, list, get
- `src/app/(dashboard)/meeting-types/page.tsx` — list of meeting types with create button
- `src/app/(dashboard)/meeting-types/[slug]/page.tsx` — edit form: name, slug, duration, buffer, location, min notice, max days ahead, daily limit

**Day 12: Meeting type configuration UI**
- `src/components/meeting-types/MeetingTypeForm.tsx` — full form with all fields
- `src/components/meeting-types/MemberAssignment.tsx` — assign members to meeting type (multi-select with weight config)
- `src/components/meeting-types/CustomQuestionsBuilder.tsx` — drag-and-drop question builder (text, textarea, select)

**Day 13: Booking creation endpoint**
- `src/server/routers/public-booking.ts` — `createBooking` mutation with transaction + conflict check
- Token generation (nanoid) for reschedule/cancel links
- Google Calendar event creation on booking
- Input validation (email, timezone, min notice, max days)

**Day 14: Email notification system**
- `src/server/services/email.ts` — Resend client setup
- `src/server/services/email-templates/confirmation.tsx` — React email template for booking confirmation
- `src/server/services/email-templates/reminder.tsx` — reminder template (24h / 1h)
- `src/server/services/email-templates/cancellation.tsx` — cancellation template
- Send confirmation to both guest and host on booking
- Include reschedule/cancel links, calendar .ics attachment

**Day 15: Reschedule and cancel flows**
- `src/app/reschedule/[token]/page.tsx` — public reschedule page: shows original booking, allows picking new slot
- `src/app/cancel/[token]/page.tsx` — public cancel page: confirm cancellation with optional reason
- `src/server/routers/public-booking.ts` — `reschedule` mutation (cancel old, create new), `cancel` mutation
- Update Google Calendar event on reschedule/cancel
- Send notification emails on reschedule/cancel

**Day 16: Booking reminder cron**
- `src/server/cron/send-reminders.ts` — find bookings starting in 24h or 1h that haven't had reminders sent
- Update `reminder_sent_at` JSON after sending
- `vercel.json` — add cron: `send-reminders` every 15 minutes
- Manual test: create booking for near-future, verify reminder sends

### Phase 4 — Embeddable Widget (Days 17–23)

**Day 17: Widget project setup**
- `widget/` directory at repo root
- `widget/package.json` — preact, preact-hooks, esbuild
- `widget/tsconfig.json` — preact JSX config
- `widget/esbuild.config.ts` — bundle to single file, minify, target ES2020
- `widget/src/index.ts` — auto-init from script tag attributes, Shadow DOM mount

**Day 18: Widget — date picker step**
- `widget/src/App.tsx` — multi-step booking flow state machine
- `widget/src/components/DatePicker.tsx` — calendar grid showing available dates (fetched from API), disabled dates with no slots
- `widget/src/api.ts` — fetch available slots from `/api/public/slots`
- `widget/src/theme.ts` — CSS variable definitions, default theme

**Day 19: Widget — time slot selection**
- `widget/src/components/TimeSlots.tsx` — scrollable list of available times for selected date, timezone display
- `widget/src/components/TimezoneSelector.tsx` — dropdown for manual timezone override
- Auto-detect timezone via `Intl.DateTimeFormat().resolvedOptions().timeZone`
- Display times in guest's local timezone

**Day 20: Widget — booking form and confirmation**
- `widget/src/components/BookingForm.tsx` — name, email, phone, custom questions
- `widget/src/components/Confirmation.tsx` — success screen: date, time, meeting link, add to calendar button (.ics download)
- `widget/src/api.ts` — POST booking to `/api/public/book`
- Loading states, error handling, retry logic

**Day 21: Widget — popup mode**
- `widget/src/components/PopupTrigger.tsx` — floating button or custom trigger
- `widget/src/components/PopupOverlay.tsx` — modal overlay with booking widget
- Animation: slide-up on mobile, fade-in modal on desktop
- Close on click outside, escape key

**Day 22: Widget — theming and customization**
- `widget/src/theme.ts` — full CSS variable system: colors, fonts, border radius, spacing
- Theme application via `data-theme` attribute or `ScheduleKit.mount({ theme: {...} })`
- Host page CSS variable inheritance
- Powered-by badge (removable on paid plans)
- Test with various host page styles to ensure isolation

**Day 23: Widget build and CDN deployment**
- `widget/esbuild.config.ts` — production build: tree-shake, minify, gzip
- Output: `widget/dist/schedulekit.js` — single file <20KB gzipped
- Version tag in filename: `schedulekit.1.0.0.js`
- `scripts/deploy-widget.ts` — upload to Cloudflare R2 with cache headers
- Embed code generator in dashboard: `<script src="https://cdn.schedulekit.io/v1/schedulekit.js" data-schedulekit data-org="your-slug" data-mode="inline"></script>`

### Phase 5 — Dashboard UI (Days 24–28)

**Day 24: Booking management dashboard**
- `src/app/(dashboard)/bookings/page.tsx` — upcoming bookings list with status badges
- `src/app/(dashboard)/bookings/[id]/page.tsx` — booking detail: guest info, meeting details, cancel/reschedule actions
- `src/server/routers/booking.ts` — list (with filters: status, date range, member), get, update status, mark no-show
- `src/components/bookings/BookingList.tsx` — sortable table with pagination

**Day 25: Calendar view**
- `src/app/(dashboard)/calendar/page.tsx` — week/month calendar view of bookings
- `src/components/calendar/CalendarView.tsx` — using @fullcalendar/react or custom grid
- Color-coded by meeting type
- Click event to open booking detail

**Day 26: Widget customizer**
- `src/app/(dashboard)/widgets/page.tsx` — list widget configs, create new
- `src/app/(dashboard)/widgets/[id]/page.tsx` — customizer: live preview on left, controls on right
- `src/components/widgets/WidgetCustomizer.tsx` — color pickers, font selector, border radius slider, display mode toggle
- `src/components/widgets/WidgetPreview.tsx` — iframe preview of widget with live theme updates
- Copy embed code button

**Day 27: Settings and integrations**
- `src/app/(dashboard)/settings/page.tsx` — org settings: name, timezone, logo, brand color
- `src/app/(dashboard)/settings/notifications/page.tsx` — notification template editor: subject, body with merge field insertion ({guest_name}, {meeting_type}, {start_time}, etc.)
- `src/app/(dashboard)/integrations/page.tsx` — connected calendars, sync status, disconnect

**Day 28: Hosted booking page**
- `src/app/(public)/[orgSlug]/page.tsx` — standalone booking page showing all active meeting types
- `src/app/(public)/[orgSlug]/[meetingTypeSlug]/page.tsx` — booking page for specific meeting type with embedded widget
- Org branding applied (logo, colors)
- SEO meta tags, OG image
- Mobile-responsive layout

### Phase 6 — Billing + Plan Enforcement (Days 29–32)

**Day 29: Stripe integration**
- `src/server/services/stripe.ts` — Stripe client, product/price IDs from env
- `src/server/routers/billing.ts` — createCheckoutSession, createPortalSession, getCurrentPlan
- `src/app/api/webhooks/stripe/route.ts` — handle `checkout.session.completed`, `customer.subscription.updated`, `customer.subscription.deleted`
- Plan mapping: free (1 meeting type, 20 bookings/mo, powered-by badge), pro ($15/mo: unlimited types, no badge, custom emails), business ($39/mo: team scheduling, widgets, analytics)

**Day 30: Plan enforcement middleware**
- `src/server/lib/plan-limits.ts` — check limits per action:
  - Free: 1 meeting type, 20 bookings/month, 1 calendar, no custom CSS, powered-by required
  - Pro: unlimited types, unlimited bookings, 3 calendars, custom CSS, no badge
  - Business: all pro + team members (10), multiple widgets, analytics
- `src/server/middleware/plan-check.ts` — tRPC middleware that validates plan limits before mutations
- Upgrade prompts in UI when hitting limits

**Day 31: Billing UI**
- `src/app/(dashboard)/settings/billing/page.tsx` — current plan, usage stats, upgrade/manage buttons
- `src/components/billing/PlanCard.tsx` — plan comparison cards
- `src/components/billing/UsageBar.tsx` — booking count / limit progress bar
- Checkout redirect and portal redirect

**Day 32: Plan-gated features**
- Hide/disable features based on plan: custom CSS (pro+), team members (business+), multiple widgets (business+), analytics (business+)
- "Upgrade to unlock" callout on gated features
- Powered-by badge enforcement on free plan
- Test all plan transitions: free→pro, pro→business, downgrade flows

### Phase 7 — Analytics + Polish (Days 33–36)

**Day 33: Widget analytics**
- `src/app/api/public/track/route.ts` — receive widget view events (no auth)
- `src/server/routers/analytics.ts` — bookings over time, conversion rate (views→bookings), popular time slots, cancellation rate
- `src/app/(dashboard)/analytics/page.tsx` — analytics dashboard with charts

**Day 34: Analytics charts**
- `src/components/analytics/BookingsChart.tsx` — bookings per day/week/month (Recharts area chart)
- `src/components/analytics/ConversionFunnel.tsx` — widget views → form starts → bookings
- `src/components/analytics/PopularSlots.tsx` — heatmap of most booked time slots
- `src/components/analytics/SourceBreakdown.tsx` — bookings by source URL / UTM

**Day 35: Edge cases and error handling**
- Double-booking prevention stress test
- Timezone edge cases: DST transitions, UTC+13/+14 timezones
- Calendar sync failure recovery: retry with exponential backoff
- Widget error states: API unreachable, no available slots, booking conflict
- Rate limiting on public endpoints (booking creation, slot fetching)

**Day 36: Performance and UX polish**
- Widget bundle optimization: verify <20KB gzipped
- Dashboard loading states with skeleton screens
- Form validation with Zod + react-hook-form
- Error boundary for dashboard pages
- Mobile-responsive dashboard layout
- Accessibility audit: keyboard navigation, ARIA labels, focus management

### Phase 8 — Launch Prep (Days 37–39)

**Day 37: Documentation and embed guide**
- Embed code documentation: script tag, React component, Vue component examples
- Theme customization guide with CSS variable reference
- API documentation for public endpoints (slots, booking)
- Getting started guide: sign up → connect calendar → configure meeting type → embed widget

**Day 38: Monitoring and observability**
- Sentry integration for error tracking (dashboard + widget)
- Vercel Analytics for dashboard performance
- Custom logging for critical flows: booking creation, calendar sync, webhook processing
- Alerting: failed calendar syncs, webhook delivery failures

**Day 39: Final testing and launch**
- End-to-end test: embed widget on test page → select slot → book → confirm email → reschedule → cancel
- Cross-browser widget testing: Chrome, Firefox, Safari, Edge
- Mobile widget testing: iOS Safari, Chrome Android
- Load test public endpoints (100 concurrent slot fetches)
- Deploy to production
- Verify Stripe webhooks in production mode

---

## Critical Files

```
schedulekit/
├── src/
│   ├── app/
│   │   ├── layout.tsx                          # Root layout: Clerk + tRPC providers
│   │   ├── (auth)/
│   │   │   ├── sign-in/[[...sign-in]]/page.tsx
│   │   │   └── sign-up/[[...sign-up]]/page.tsx
│   │   ├── (dashboard)/
│   │   │   ├── layout.tsx                      # Dashboard shell with sidebar nav
│   │   │   ├── bookings/
│   │   │   │   ├── page.tsx                    # Booking list with filters
│   │   │   │   └── [id]/page.tsx               # Booking detail
│   │   │   ├── calendar/page.tsx               # Calendar view of bookings
│   │   │   ├── meeting-types/
│   │   │   │   ├── page.tsx                    # Meeting type list
│   │   │   │   └── [slug]/page.tsx             # Meeting type editor
│   │   │   ├── availability/page.tsx           # Weekly schedule + overrides
│   │   │   ├── widgets/
│   │   │   │   ├── page.tsx                    # Widget config list
│   │   │   │   └── [id]/page.tsx               # Widget customizer with preview
│   │   │   ├── analytics/page.tsx              # Booking analytics dashboard
│   │   │   ├── integrations/page.tsx           # Calendar connections
│   │   │   └── settings/
│   │   │       ├── page.tsx                    # Org settings
│   │   │       ├── notifications/page.tsx      # Email template editor
│   │   │       └── billing/page.tsx            # Plan management
│   │   ├── (public)/
│   │   │   └── [orgSlug]/
│   │   │       ├── page.tsx                    # Hosted booking page (all types)
│   │   │       └── [meetingTypeSlug]/page.tsx   # Specific meeting type booking
│   │   ├── reschedule/[token]/page.tsx         # Guest reschedule page
│   │   ├── cancel/[token]/page.tsx             # Guest cancel page
│   │   └── api/
│   │       ├── trpc/[trpc]/route.ts            # tRPC handler
│   │       ├── auth/
│   │       │   └── google-calendar/
│   │       │       ├── route.ts                # OAuth redirect
│   │       │       └── callback/route.ts       # OAuth callback
│   │       ├── public/
│   │       │   ├── slots/route.ts              # Public slot availability (GET)
│   │       │   ├── book/route.ts               # Public booking creation (POST)
│   │       │   └── track/route.ts              # Widget view tracking (POST)
│   │       └── webhooks/
│   │           ├── google-calendar/route.ts    # Calendar push notifications
│   │           └── stripe/route.ts             # Stripe payment webhooks
│   ├── server/
│   │   ├── db/
│   │   │   ├── index.ts                        # Neon connection
│   │   │   └── schema.ts                       # 12 tables + relations
│   │   ├── trpc.ts                             # tRPC base config + middleware
│   │   ├── routers/
│   │   │   ├── _app.ts                         # Root router
│   │   │   ├── org.ts                          # Organization CRUD
│   │   │   ├── member.ts                       # Team member CRUD
│   │   │   ├── meeting-type.ts                 # Meeting type CRUD
│   │   │   ├── availability.ts                 # Weekly schedule + overrides
│   │   │   ├── calendar.ts                     # Calendar connection management
│   │   │   ├── booking.ts                      # Authenticated booking management
│   │   │   ├── public-booking.ts               # Public slot + booking endpoints
│   │   │   ├── widget-config.ts                # Widget config CRUD
│   │   │   ├── analytics.ts                    # Analytics queries
│   │   │   └── billing.ts                      # Stripe checkout + portal
│   │   ├── services/
│   │   │   ├── availability.ts                 # Slot computation engine
│   │   │   ├── google-calendar.ts              # OAuth, sync, watch channels
│   │   │   ├── email.ts                        # Resend client + send helpers
│   │   │   ├── stripe.ts                       # Stripe client + helpers
│   │   │   └── email-templates/
│   │   │       ├── confirmation.tsx            # Booking confirmation email
│   │   │       ├── reminder.tsx                # Reminder email (24h/1h)
│   │   │       └── cancellation.tsx            # Cancellation email
│   │   ├── lib/
│   │   │   ├── encryption.ts                   # AES-256-GCM for OAuth tokens
│   │   │   ├── org-seed.ts                     # First-login org initialization
│   │   │   └── plan-limits.ts                  # Plan limit definitions + checker
│   │   └── cron/
│   │       ├── send-reminders.ts               # Booking reminder sender
│   │       └── renew-watch-channels.ts         # Google Calendar channel renewal
│   └── components/
│       ├── availability/
│       │   ├── WeeklyScheduleEditor.tsx        # Weekly time block grid
│       │   └── OverrideCalendar.tsx            # Date override picker
│       ├── meeting-types/
│       │   ├── MeetingTypeForm.tsx              # Full meeting type config form
│       │   ├── MemberAssignment.tsx             # Member selector for type
│       │   └── CustomQuestionsBuilder.tsx       # Question builder
│       ├── bookings/
│       │   └── BookingList.tsx                  # Sortable booking table
│       ├── calendar/
│       │   └── CalendarView.tsx                 # Week/month calendar
│       ├── widgets/
│       │   ├── WidgetCustomizer.tsx             # Theme controls
│       │   └── WidgetPreview.tsx                # Live widget preview
│       ├── analytics/
│       │   ├── BookingsChart.tsx                # Area chart
│       │   ├── ConversionFunnel.tsx             # Funnel visualization
│       │   ├── PopularSlots.tsx                 # Time heatmap
│       │   └── SourceBreakdown.tsx              # Source pie chart
│       └── billing/
│           ├── PlanCard.tsx                     # Plan comparison
│           └── UsageBar.tsx                     # Usage progress bar
├── widget/
│   ├── package.json                            # Preact, esbuild deps
│   ├── tsconfig.json                           # Preact JSX config
│   ├── esbuild.config.ts                       # Build config
│   └── src/
│       ├── index.ts                            # Entry: auto-init, Shadow DOM
│       ├── App.tsx                             # Multi-step booking flow
│       ├── api.ts                              # Public API client
│       ├── theme.ts                            # CSS variable definitions
│       └── components/
│           ├── DatePicker.tsx                   # Calendar date grid
│           ├── TimeSlots.tsx                    # Available time list
│           ├── TimezoneSelector.tsx             # TZ dropdown
│           ├── BookingForm.tsx                  # Guest form
│           ├── Confirmation.tsx                 # Success screen
│           ├── PopupTrigger.tsx                 # Floating button
│           └── PopupOverlay.tsx                 # Modal container
├── scripts/
│   └── deploy-widget.ts                        # R2 upload script
├── drizzle.config.ts
├── vercel.json                                 # Cron jobs config
├── .env.local
└── package.json
```

---

## Verification

### Manual Testing Checklist

1. **Sign up → org + member + default schedule created** — weekly schedule shows Mon–Fri 9–5, default meeting type exists
2. **Set weekly availability** — toggle days on/off, adjust start/end times, verify saved
3. **Add date override** — mark a date as unavailable, verify it's excluded from available slots
4. **Connect Google Calendar** — OAuth flow completes, events sync, connection shows in integrations
5. **Calendar sync updates availability** — create event in Google Calendar, verify slots update within 60s (via watch notification)
6. **Widget renders in iframe isolation** — embed widget on test page, verify no style bleed in either direction
7. **Widget shows available slots** — available dates highlighted, times match configured schedule minus busy times
8. **Complete booking flow** — select date → time → fill form → submit → confirmation screen shown
9. **Double-booking prevention** — try booking same slot simultaneously in two browser tabs, second should fail gracefully
10. **Confirmation email received** — both guest and host receive email with correct details, .ics attachment works
11. **Reschedule via token link** — click reschedule link in email, pick new slot, verify old booking cancelled + new created
12. **Cancel via token link** — click cancel link, confirm, verify booking status updated + notification sent
13. **Reminder emails** — create booking 25h in future, verify 24h reminder sends at correct time
14. **Widget popup mode** — click trigger button, overlay opens, booking flow works, close on escape/outside click
15. **Widget theming** — change colors/font/radius in customizer, verify widget preview updates, verify embedded widget matches
16. **Hosted booking page** — visit `/{orgSlug}/{meetingTypeSlug}`, full booking flow works
17. **Plan limits** — on free plan: can't create second meeting type, powered-by badge shows, upgrade prompt works

### SQL Verification Queries

```sql
-- 1. Organizations with plan distribution
SELECT plan, COUNT(*) as count FROM organizations GROUP BY plan;

-- 2. Bookings per org this month
SELECT o.name, COUNT(b.id) as booking_count
FROM organizations o
LEFT JOIN bookings b ON b.org_id = o.id
  AND b.created_at >= date_trunc('month', NOW())
  AND b.status != 'cancelled'
GROUP BY o.name ORDER BY booking_count DESC;

-- 3. Available slot computation verification (for debugging)
SELECT m.name as member, wa.day_of_week, wa.start_time, wa.end_time, wa.is_enabled
FROM weekly_availability wa
JOIN members m ON m.id = wa.member_id
WHERE m.org_id = '<org_id>'
ORDER BY m.name, wa.day_of_week;

-- 4. Calendar sync health check
SELECT m.name, cc.provider, cc.last_synced_at,
  cc.watch_expires_at, cc.sync_enabled,
  COUNT(ce.id) as cached_events
FROM calendar_connections cc
JOIN members m ON m.id = cc.member_id
LEFT JOIN cached_calendar_events ce ON ce.calendar_connection_id = cc.id
GROUP BY m.name, cc.provider, cc.last_synced_at, cc.watch_expires_at, cc.sync_enabled;

-- 5. Booking conflict check
SELECT b1.id, b1.start_time, b1.end_time, b1.status,
  b2.id as conflicting_id, b2.start_time as conf_start, b2.end_time as conf_end
FROM bookings b1
JOIN bookings b2 ON b1.assigned_member_id = b2.assigned_member_id
  AND b1.id != b2.id
  AND b1.start_time < b2.end_time
  AND b1.end_time > b2.start_time
  AND b1.status != 'cancelled'
  AND b2.status != 'cancelled';
```

### Performance Benchmarks

| Operation | Target | Method |
|-----------|--------|--------|
| Widget initial load | <200ms | Lighthouse, <20KB gzipped bundle |
| Slot availability fetch | <300ms | API response time for 30-day range, 1 member |
| Slot availability fetch (team) | <800ms | API response time for 30-day range, 10 members |
| Booking creation | <500ms | API response time including conflict check |
| Calendar sync (incremental) | <2s | Re-sync after webhook notification |
| Calendar sync (full, 90 days) | <5s | Initial sync with 500 events |
| Widget re-render on date change | <50ms | Client-side slot filtering |
| Dashboard page load | <1.5s | Full page with booking list |
| Embed code copy-paste test | 1 step | Script tag → widget visible in <3s |
| Concurrent booking test | 0 conflicts | 50 simultaneous bookings to same slot, exactly 1 succeeds |

---

## Daily Meeting Limits (spec F4)

The `meetingTypes` table includes a `dailyLimit` column (nullable integer) for configuring maximum meetings per day:

- **Configurable max meetings per day**: Default is 8 meetings per day per member. Set to `null` for unlimited. Enforced during slot computation -- once the daily limit is reached for a member on a given date, remaining slots for that date are excluded from the availability response.
- **Buffer time between meetings**: Configurable via `bufferBefore` and `bufferAfter` on each meeting type (default 15 minutes each). The availability engine subtracts buffer time from available windows, ensuring the host has breathing room between back-to-back meetings. For example, a 30-minute meeting with 15-minute buffers blocks a 60-minute window in the calendar.
- **Lunch block auto-detection**: The availability engine detects lunch blocks by finding gaps in the weekly schedule configuration. If a member's schedule has a gap between 11:00-14:00 (e.g., available 09:00-12:00 and 13:00-17:00), the gap is automatically treated as a lunch block. No meetings are scheduled during this gap. Members can also explicitly mark lunch blocks via date overrides.

---

## Video Conferencing Link Generation

When a booking is confirmed, the system automatically generates a video conferencing link based on the meeting type's `locationType`:

- **Google Meet**: Uses the Google Calendar API v3. When creating the calendar event via `events.insert`, set `conferenceDataVersion: 1` and `conferenceData.createRequest.requestId` to a unique ID. Google automatically generates a Meet link and returns it in the event response. The link is stored in `bookings.meetingLink`. Requires the connected Google Calendar OAuth scope `https://www.googleapis.com/auth/calendar.events`.
- **Zoom**: Requires a Zoom OAuth app with the `meeting:write` scope. On booking confirmation, call `POST /users/me/meetings` with the meeting details (topic, start time, duration). The response contains the `join_url` which is stored in `bookings.meetingLink`. Token refresh is handled via the same encrypted credential flow as Google Calendar.
- **Custom link support**: For other platforms (Microsoft Teams, Webex, or any custom URL), the meeting type's `locationConfig.customUrl` is used directly as the meeting link. Template variables are supported: `{{booking_id}}`, `{{guest_name}}`, `{{start_time}}` are replaced before storing.

---

## Post-MVP Roadmap

The following features extend ScheduleKit beyond the core MVP:

- **RTL layout support**: Full right-to-left layout for the widget and dashboard to support Arabic, Hebrew, and other RTL languages. Requires CSS logical properties and mirrored component layouts.
- **Custom CSS theming**: Beyond CSS variable theming, allow Pro/Business users to inject custom CSS into the widget Shadow DOM for pixel-perfect brand matching.
- **15+ language translations (spec F2)**: Internationalize all widget strings, email templates, and hosted booking pages. Priority languages: Spanish, French, German, Portuguese, Japanese, Korean, Chinese (Simplified), Italian, Dutch, Polish, Turkish, Russian, Arabic, Hindi, Thai.
- **Team scheduling modes -- round-robin and collective (spec F6)**: Round-robin assigns bookings to team members in rotation (weighted by the `meetingTypeMembers.weight` field). Collective mode requires all assigned members to be available simultaneously. Both modes extend the availability computation engine to query multiple members.
- **Payment integration -- Stripe pre-payment (spec F7)**: Require guests to pay before confirming a booking. The `meetingTypes.priceCents` field drives a Stripe Checkout session inserted into the booking flow between form submission and confirmation. Supports refunds on cancellation within a configurable window.
- **SMS notifications via Twilio (spec F8)**: Send booking confirmations and reminders via SMS in addition to email. Requires Twilio integration with the `guest_phone` field on bookings. SMS is opt-in per guest and configurable per meeting type.
