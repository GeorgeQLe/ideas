# 19. WaitlistIQ — Smart Waitlist & Launch Manager

## Implementation Plan

**MVP Scope:** AI landing page generation from product description (GPT-4o two-step: page structure + section content), visual page editor with drag-and-drop section reordering (@dnd-kit), hosted landing pages at `{slug}.waitlistiq.io` with ISR and OG meta tags, waitlist signup with position tracking and anti-fraud pipeline (Cloudflare Turnstile + disposable email detection + rate limiting), referral engine with unique codes (nanoid) and configurable position bumps, subscriber confirmation page with social sharing, email drip sequences with BullMQ delayed jobs and merge field rendering via Resend, basic analytics dashboard (signups, sources, referral metrics), Stripe billing with three tiers (Free / Pro $19/mo / Growth $49/mo).

---

## Tech Decisions

| Layer | Technology | Notes |
|-------|-----------|-------|
| Framework | Next.js 14+ App Router | `src/` directory, TypeScript strict |
| API | tRPC v11 | superjson transformer, Clerk auth context |
| Database | Neon PostgreSQL | Projects, subscribers, pages, email sequences |
| ORM | Drizzle ORM | Push-based migrations, typed schema |
| Auth | Clerk | User-scoped (no orgs in MVP), `userId` from session |
| Billing | Stripe | Checkout Sessions + Customer Portal + Webhooks |
| AI — Page Generation | OpenAI GPT-4o | Two-step: JSON structure (temp 0.3) + section copy (temp 0.7) |
| AI — Email Copy | OpenAI GPT-4o | Temperature 0.7 for engaging email content |
| Email | Resend | Drip sequences, transactional (welcome, position updates) |
| Queue | BullMQ + Redis (Upstash) | Delayed email scheduling, AI generation jobs |
| Landing Page Hosting | Next.js ISR (Vercel) | `revalidateTag('page-{slug}')` on publish |
| Anti-Spam | Cloudflare Turnstile | Invisible challenge on signup forms |
| Drag & Drop | @dnd-kit/core + @dnd-kit/sortable | Section reordering in page editor |
| Animations | Framer Motion | Landing page animations, counter transitions |
| Image Storage | Cloudflare R2 | OG images, uploaded hero images, logos |
| Referral Codes | nanoid | 8-char alphanumeric unique codes |
| UI | Tailwind CSS, Radix UI, Recharts | |
| Hosting | Vercel | |
| Monitoring | Sentry, Vercel Analytics | |

### Key Architecture Decisions

1. **Two-step AI page generation over single-prompt**: Step 1 generates a JSON page structure (section order, section types, key messaging angles) using GPT-4o with temperature 0.3 for reliable structure. Step 2 iterates over each section and generates rich copy (headline, subheadline, body, CTAs) with temperature 0.7 for creative output. This separation keeps the structure predictable while allowing creative copywriting, and enables per-section regeneration without re-generating the entire page.

2. **ISR for hosted landing pages**: Public landing pages are statically generated and served from Vercel's edge CDN. On publish or edit, `revalidateTag('page-{slug}')` triggers on-demand revalidation. This gives sub-100ms page loads (critical for conversion) while keeping content fresh. The public route at `/p/[slug]` uses Next.js ISR with dynamic params.

3. **BullMQ delayed jobs for email drip scheduling**: When a subscriber signs up, the system enqueues one delayed job per sequence email (e.g., welcome at 0h, share reminder at 48h, sneak peek at 168h). Each job fires at its scheduled time, renders merge fields ({position}, {referral_link}, {first_name}), and sends via Resend. This avoids cron-based polling and gives exact timing control with automatic retry on failure.

4. **Anti-fraud pipeline with layered defense**: Signup requests pass through: (1) Cloudflare Turnstile invisible challenge verification, (2) rate limiting per IP (10 signups/hour via Upstash Redis), (3) disposable email domain blocklist check (500+ domains), (4) duplicate email check per project. Fraudulent referrals are detected by comparing IP addresses between referrer and referee, flagging same-IP signups.

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
// Users (product creators who use WaitlistIQ)
// ---------------------------------------------------------------------------
export const users = pgTable("users", {
  id: uuid("id").primaryKey().defaultRandom(),
  clerkUserId: text("clerk_user_id").unique().notNull(),
  email: text("email").notNull(),
  name: text("name"),
  plan: text("plan").default("free").notNull(), // free | pro | growth
  stripeCustomerId: text("stripe_customer_id"),
  stripeSubscriptionId: text("stripe_subscription_id"),
  onboardingCompleted: boolean("onboarding_completed").default(false).notNull(),
  createdAt: timestamp("created_at", { withTimezone: true }).defaultNow().notNull(),
  updatedAt: timestamp("updated_at", { withTimezone: true }).defaultNow().notNull(),
});

// ---------------------------------------------------------------------------
// Projects (each project = one waitlist / landing page)
// ---------------------------------------------------------------------------
export const projects = pgTable(
  "projects",
  {
    id: uuid("id").primaryKey().defaultRandom(),
    userId: uuid("user_id")
      .references(() => users.id, { onDelete: "cascade" })
      .notNull(),
    name: text("name").notNull(),
    slug: text("slug").unique().notNull(), // used in waitlistiq.io/{slug}
    productDescription: text("product_description").notNull(),
    tagline: text("tagline"),
    industry: text("industry"), // saas | consumer | newsletter | ecommerce | other
    targetAudience: text("target_audience"),
    customDomain: text("custom_domain"), // pro+ feature
    status: text("status").default("draft").notNull(), // draft | live | paused | launched
    subscriberCount: integer("subscriber_count").default(0).notNull(), // denormalized counter
    settings: jsonb("settings").default({}).$type<{
      brandColor?: string;
      accentColor?: string;
      logoUrl?: string;
      faviconUrl?: string;
      showWaitlistCount?: boolean;
      requireName?: boolean;
      customFields?: Array<{ key: string; label: string; type: "text" | "select"; options?: string[] }>;
      waitlistCap?: number | null;
      requireEmailVerification?: boolean;
    }>(),
    createdAt: timestamp("created_at", { withTimezone: true }).defaultNow().notNull(),
    updatedAt: timestamp("updated_at", { withTimezone: true }).defaultNow().notNull(),
  },
  (t) => [
    index("projects_user_id_idx").on(t.userId),
    uniqueIndex("projects_slug_idx").on(t.slug),
    index("projects_status_idx").on(t.status),
  ],
);

// ---------------------------------------------------------------------------
// Landing Pages (AI-generated, editable sections)
// ---------------------------------------------------------------------------
export const landingPages = pgTable(
  "landing_pages",
  {
    id: uuid("id").primaryKey().defaultRandom(),
    projectId: uuid("project_id")
      .references(() => projects.id, { onDelete: "cascade" })
      .notNull(),
    template: text("template").default("minimal").notNull(),
    // minimal | bold | gradient | dark | startup | product
    sections: jsonb("sections").default([]).$type<Array<{
      id: string;           // nanoid for stable key
      type: "hero" | "problem_solution" | "features" | "social_proof" | "faq" | "cta";
      order: number;
      visible: boolean;
      content: {
        headline?: string;
        subheadline?: string;
        body?: string;
        buttonText?: string;
        items?: Array<{
          title: string;
          description: string;
          icon?: string;
        }>;
        faqs?: Array<{
          question: string;
          answer: string;
        }>;
        imageUrl?: string;
      };
    }>>(),
    customCss: text("custom_css"),
    ogImageUrl: text("og_image_url"),
    seoTitle: text("seo_title"),
    seoDescription: text("seo_description"),
    publishedAt: timestamp("published_at", { withTimezone: true }),
    createdAt: timestamp("created_at", { withTimezone: true }).defaultNow().notNull(),
    updatedAt: timestamp("updated_at", { withTimezone: true }).defaultNow().notNull(),
  },
  (t) => [
    uniqueIndex("landing_pages_project_id_idx").on(t.projectId),
  ],
);

// ---------------------------------------------------------------------------
// Waitlist Configs (referral program settings)
// ---------------------------------------------------------------------------
export const waitlistConfigs = pgTable(
  "waitlist_configs",
  {
    id: uuid("id").primaryKey().defaultRandom(),
    projectId: uuid("project_id")
      .references(() => projects.id, { onDelete: "cascade" })
      .notNull(),
    positionsPerReferral: integer("positions_per_referral").default(5).notNull(),
    referralEnabled: boolean("referral_enabled").default(true).notNull(),
    leaderboardEnabled: boolean("leaderboard_enabled").default(false).notNull(),
    tiers: jsonb("tiers").default([]).$type<Array<{
      referralsRequired: number;
      rewardName: string;
      rewardDescription: string;
    }>>(),
    shareMessageTwitter: text("share_message_twitter")
      .default("I just signed up for {product_name} — join the waitlist! {referral_link}"),
    shareMessageLinkedin: text("share_message_linkedin")
      .default("Excited to be on the waitlist for {product_name}. Check it out: {referral_link}"),
    shareMessageGeneric: text("share_message_generic")
      .default("Join me on the waitlist for {product_name}: {referral_link}"),
    fraudDetectionEnabled: boolean("fraud_detection_enabled").default(true).notNull(),
    createdAt: timestamp("created_at", { withTimezone: true }).defaultNow().notNull(),
    updatedAt: timestamp("updated_at", { withTimezone: true }).defaultNow().notNull(),
  },
  (t) => [
    uniqueIndex("waitlist_configs_project_id_idx").on(t.projectId),
  ],
);

// ---------------------------------------------------------------------------
// Subscribers (people who sign up for the waitlist)
// ---------------------------------------------------------------------------
export const subscribers = pgTable(
  "subscribers",
  {
    id: uuid("id").primaryKey().defaultRandom(),
    projectId: uuid("project_id")
      .references(() => projects.id, { onDelete: "cascade" })
      .notNull(),
    email: text("email").notNull(),
    name: text("name"),
    customFields: jsonb("custom_fields").default({}).$type<Record<string, string>>(),
    position: integer("position").notNull(),          // current position (recalculated on referrals)
    originalPosition: integer("original_position").notNull(), // position at signup time
    referralCode: text("referral_code").unique().notNull(),   // nanoid 8-char
    referredBySubscriberId: uuid("referred_by_subscriber_id")
      .references(() => subscribers.id, { onDelete: "set null" }),
    referralCount: integer("referral_count").default(0).notNull(),
    tierUnlocked: integer("tier_unlocked").default(0).notNull(), // index into tiers array, 0 = none
    emailVerified: boolean("email_verified").default(false).notNull(),
    verificationToken: text("verification_token"),
    ipAddress: text("ip_address"),
    utmSource: text("utm_source"),
    utmMedium: text("utm_medium"),
    utmCampaign: text("utm_campaign"),
    source: text("source").default("direct").notNull(), // direct | referral | social
    device: text("device"),    // mobile | desktop | tablet
    country: text("country"),  // ISO 3166-1 alpha-2
    status: text("status").default("waitlisted").notNull(), // waitlisted | invited | active
    inviteCode: text("invite_code"),   // generated when invited
    fraudFlag: boolean("fraud_flag").default(false).notNull(),
    subscribedAt: timestamp("subscribed_at", { withTimezone: true }).defaultNow().notNull(),
    createdAt: timestamp("created_at", { withTimezone: true }).defaultNow().notNull(),
  },
  (t) => [
    uniqueIndex("subscribers_project_email_idx").on(t.projectId, t.email),
    uniqueIndex("subscribers_referral_code_idx").on(t.referralCode),
    index("subscribers_project_id_idx").on(t.projectId),
    index("subscribers_position_idx").on(t.projectId, t.position),
    index("subscribers_referred_by_idx").on(t.referredBySubscriberId),
    index("subscribers_status_idx").on(t.projectId, t.status),
    index("subscribers_subscribed_at_idx").on(t.projectId, t.subscribedAt),
  ],
);

// ---------------------------------------------------------------------------
// Email Sequences (drip campaign definitions)
// ---------------------------------------------------------------------------
export const emailSequences = pgTable(
  "email_sequences",
  {
    id: uuid("id").primaryKey().defaultRandom(),
    projectId: uuid("project_id")
      .references(() => projects.id, { onDelete: "cascade" })
      .notNull(),
    name: text("name").notNull(),              // "Welcome Series", "Referral Nudge", etc.
    trigger: text("trigger").notNull(),         // signup | referral_milestone | manual | launch
    isActive: boolean("is_active").default(true).notNull(),
    createdAt: timestamp("created_at", { withTimezone: true }).defaultNow().notNull(),
    updatedAt: timestamp("updated_at", { withTimezone: true }).defaultNow().notNull(),
  },
  (t) => [
    index("email_sequences_project_id_idx").on(t.projectId),
  ],
);

// ---------------------------------------------------------------------------
// Sequence Emails (individual emails within a sequence)
// ---------------------------------------------------------------------------
export const sequenceEmails = pgTable(
  "sequence_emails",
  {
    id: uuid("id").primaryKey().defaultRandom(),
    sequenceId: uuid("sequence_id")
      .references(() => emailSequences.id, { onDelete: "cascade" })
      .notNull(),
    subject: text("subject").notNull(),
    bodyHtml: text("body_html").notNull(),
    delayHours: integer("delay_hours").default(0).notNull(), // hours after trigger
    sortOrder: integer("sort_order").default(0).notNull(),
    mergeFields: jsonb("merge_fields").default([]).$type<string[]>(),
    // ["first_name", "position", "referral_link", "referral_count", "product_name"]
    createdAt: timestamp("created_at", { withTimezone: true }).defaultNow().notNull(),
    updatedAt: timestamp("updated_at", { withTimezone: true }).defaultNow().notNull(),
  },
  (t) => [
    index("sequence_emails_sequence_id_idx").on(t.sequenceId),
    index("sequence_emails_sort_order_idx").on(t.sequenceId, t.sortOrder),
  ],
);

// ---------------------------------------------------------------------------
// Email Sends (tracking individual email deliveries)
// ---------------------------------------------------------------------------
export const emailSends = pgTable(
  "email_sends",
  {
    id: uuid("id").primaryKey().defaultRandom(),
    subscriberId: uuid("subscriber_id")
      .references(() => subscribers.id, { onDelete: "cascade" })
      .notNull(),
    sequenceEmailId: uuid("sequence_email_id")
      .references(() => sequenceEmails.id, { onDelete: "cascade" })
      .notNull(),
    projectId: uuid("project_id")
      .references(() => projects.id, { onDelete: "cascade" })
      .notNull(),
    status: text("status").default("pending").notNull(),
    // pending | queued | sent | opened | clicked | bounced | failed
    resendMessageId: text("resend_message_id"),
    scheduledFor: timestamp("scheduled_for", { withTimezone: true }).notNull(),
    sentAt: timestamp("sent_at", { withTimezone: true }),
    openedAt: timestamp("opened_at", { withTimezone: true }),
    clickedAt: timestamp("clicked_at", { withTimezone: true }),
    errorMessage: text("error_message"),
    createdAt: timestamp("created_at", { withTimezone: true }).defaultNow().notNull(),
  },
  (t) => [
    index("email_sends_subscriber_id_idx").on(t.subscriberId),
    index("email_sends_sequence_email_id_idx").on(t.sequenceEmailId),
    index("email_sends_project_id_idx").on(t.projectId),
    index("email_sends_status_idx").on(t.status),
    index("email_sends_scheduled_for_idx").on(t.scheduledFor),
  ],
);

// ---------------------------------------------------------------------------
// Page Views (analytics tracking)
// ---------------------------------------------------------------------------
export const pageViews = pgTable(
  "page_views",
  {
    id: uuid("id").primaryKey().defaultRandom(),
    projectId: uuid("project_id")
      .references(() => projects.id, { onDelete: "cascade" })
      .notNull(),
    sessionId: text("session_id"),             // anonymous session tracking
    utmSource: text("utm_source"),
    utmMedium: text("utm_medium"),
    utmCampaign: text("utm_campaign"),
    referrer: text("referrer"),
    device: text("device"),                    // mobile | desktop | tablet
    country: text("country"),
    converted: boolean("converted").default(false).notNull(), // did this view lead to a signup?
    viewedAt: timestamp("viewed_at", { withTimezone: true }).defaultNow().notNull(),
  },
  (t) => [
    index("page_views_project_id_idx").on(t.projectId),
    index("page_views_viewed_at_idx").on(t.projectId, t.viewedAt),
    index("page_views_converted_idx").on(t.projectId, t.converted),
  ],
);

// ---------------------------------------------------------------------------
// Referral Events (tracks each referral conversion)
// ---------------------------------------------------------------------------
export const referralEvents = pgTable(
  "referral_events",
  {
    id: uuid("id").primaryKey().defaultRandom(),
    projectId: uuid("project_id")
      .references(() => projects.id, { onDelete: "cascade" })
      .notNull(),
    referrerId: uuid("referrer_id")
      .references(() => subscribers.id, { onDelete: "cascade" })
      .notNull(),
    refereeId: uuid("referee_id")
      .references(() => subscribers.id, { onDelete: "cascade" })
      .notNull(),
    positionsBumped: integer("positions_bumped").notNull(), // how many positions the referrer moved up
    tierUnlockedByEvent: integer("tier_unlocked_by_event"), // tier index if this referral unlocked a new tier
    fraudSuspected: boolean("fraud_suspected").default(false).notNull(),
    fraudReason: text("fraud_reason"), // "same_ip" | "disposable_email" | "rapid_signup"
    createdAt: timestamp("created_at", { withTimezone: true }).defaultNow().notNull(),
  },
  (t) => [
    index("referral_events_project_id_idx").on(t.projectId),
    index("referral_events_referrer_id_idx").on(t.referrerId),
    index("referral_events_referee_id_idx").on(t.refereeId),
  ],
);

// ---------------------------------------------------------------------------
// A/B Test Variants (landing page experiments)
// ---------------------------------------------------------------------------
export const abTests = pgTable(
  "ab_tests",
  {
    id: uuid("id").primaryKey().defaultRandom(),
    projectId: uuid("project_id")
      .references(() => projects.id, { onDelete: "cascade" })
      .notNull(),
    name: text("name").notNull(),                    // "Hero headline test", "CTA color test"
    status: text("status").default("draft").notNull(), // draft | running | completed | archived
    trafficSplit: jsonb("traffic_split").default({}).$type<
      Record<string, number> // { "control": 50, "variant_a": 50 } — must sum to 100
    >(),
    targetMetric: text("target_metric").default("signup_rate").notNull(),
    // signup_rate | referral_rate | email_open_rate
    winnerVariantId: uuid("winner_variant_id"),
    significanceLevel: real("significance_level"), // p-value from chi-squared test
    minSampleSize: integer("min_sample_size").default(100).notNull(), // per variant
    startedAt: timestamp("started_at", { withTimezone: true }),
    endedAt: timestamp("ended_at", { withTimezone: true }),
    createdAt: timestamp("created_at", { withTimezone: true }).defaultNow().notNull(),
    updatedAt: timestamp("updated_at", { withTimezone: true }).defaultNow().notNull(),
  },
  (t) => [
    index("ab_tests_project_id_idx").on(t.projectId),
    index("ab_tests_status_idx").on(t.projectId, t.status),
  ],
);

export const abTestVariants = pgTable(
  "ab_test_variants",
  {
    id: uuid("id").primaryKey().defaultRandom(),
    testId: uuid("test_id")
      .references(() => abTests.id, { onDelete: "cascade" })
      .notNull(),
    name: text("name").notNull(),                    // "control", "variant_a", "variant_b"
    isControl: boolean("is_control").default(false).notNull(),
    sectionOverrides: jsonb("section_overrides").default([]).$type<Array<{
      sectionId: string;        // references landing page section ID
      field: string;            // "headline" | "subheadline" | "buttonText" | "body"
      value: string;            // override content
    }>>(),
    impressions: integer("impressions").default(0).notNull(),
    conversions: integer("conversions").default(0).notNull(), // signups
    conversionRate: real("conversion_rate").default(0).notNull(),
    createdAt: timestamp("created_at", { withTimezone: true }).defaultNow().notNull(),
  },
  (t) => [
    index("ab_test_variants_test_id_idx").on(t.testId),
  ],
);

// ---------------------------------------------------------------------------
// Relations
// ---------------------------------------------------------------------------
export const usersRelations = relations(users, ({ many }) => ({
  projects: many(projects),
}));

export const projectsRelations = relations(projects, ({ one, many }) => ({
  user: one(users, {
    fields: [projects.userId],
    references: [users.id],
  }),
  landingPage: one(landingPages),
  waitlistConfig: one(waitlistConfigs),
  subscribers: many(subscribers),
  emailSequences: many(emailSequences),
  emailSends: many(emailSends),
  pageViews: many(pageViews),
  referralEvents: many(referralEvents),
  abTests: many(abTests),
}));

export const landingPagesRelations = relations(landingPages, ({ one }) => ({
  project: one(projects, {
    fields: [landingPages.projectId],
    references: [projects.id],
  }),
}));

export const waitlistConfigsRelations = relations(waitlistConfigs, ({ one }) => ({
  project: one(projects, {
    fields: [waitlistConfigs.projectId],
    references: [projects.id],
  }),
}));

export const subscribersRelations = relations(subscribers, ({ one, many }) => ({
  project: one(projects, {
    fields: [subscribers.projectId],
    references: [projects.id],
  }),
  referredBy: one(subscribers, {
    fields: [subscribers.referredBySubscriberId],
    references: [subscribers.id],
    relationName: "referralChain",
  }),
  referrals: many(subscribers, { relationName: "referralChain" }),
  emailSends: many(emailSends),
  referralEventsAsReferrer: many(referralEvents, { relationName: "referrerEvents" }),
  referralEventsAsReferee: many(referralEvents, { relationName: "refereeEvents" }),
}));

export const emailSequencesRelations = relations(emailSequences, ({ one, many }) => ({
  project: one(projects, {
    fields: [emailSequences.projectId],
    references: [projects.id],
  }),
  emails: many(sequenceEmails),
}));

export const sequenceEmailsRelations = relations(sequenceEmails, ({ one, many }) => ({
  sequence: one(emailSequences, {
    fields: [sequenceEmails.sequenceId],
    references: [emailSequences.id],
  }),
  sends: many(emailSends),
}));

export const emailSendsRelations = relations(emailSends, ({ one }) => ({
  subscriber: one(subscribers, {
    fields: [emailSends.subscriberId],
    references: [subscribers.id],
  }),
  sequenceEmail: one(sequenceEmails, {
    fields: [emailSends.sequenceEmailId],
    references: [sequenceEmails.id],
  }),
  project: one(projects, {
    fields: [emailSends.projectId],
    references: [projects.id],
  }),
}));

export const pageViewsRelations = relations(pageViews, ({ one }) => ({
  project: one(projects, {
    fields: [pageViews.projectId],
    references: [projects.id],
  }),
}));

export const referralEventsRelations = relations(referralEvents, ({ one }) => ({
  project: one(projects, {
    fields: [referralEvents.projectId],
    references: [projects.id],
  }),
  referrer: one(subscribers, {
    fields: [referralEvents.referrerId],
    references: [subscribers.id],
    relationName: "referrerEvents",
  }),
  referee: one(subscribers, {
    fields: [referralEvents.refereeId],
    references: [subscribers.id],
    relationName: "refereeEvents",
  }),
}));

export const abTestsRelations = relations(abTests, ({ one, many }) => ({
  project: one(projects, {
    fields: [abTests.projectId],
    references: [projects.id],
  }),
  variants: many(abTestVariants),
}));

export const abTestVariantsRelations = relations(abTestVariants, ({ one }) => ({
  test: one(abTests, {
    fields: [abTestVariants.testId],
    references: [abTests.id],
  }),
}));

// ---------------------------------------------------------------------------
// Type Exports
// ---------------------------------------------------------------------------
export type User = typeof users.$inferSelect;
export type NewUser = typeof users.$inferInsert;
export type Project = typeof projects.$inferSelect;
export type NewProject = typeof projects.$inferInsert;
export type LandingPage = typeof landingPages.$inferSelect;
export type NewLandingPage = typeof landingPages.$inferInsert;
export type WaitlistConfig = typeof waitlistConfigs.$inferSelect;
export type NewWaitlistConfig = typeof waitlistConfigs.$inferInsert;
export type Subscriber = typeof subscribers.$inferSelect;
export type NewSubscriber = typeof subscribers.$inferInsert;
export type EmailSequence = typeof emailSequences.$inferSelect;
export type NewEmailSequence = typeof emailSequences.$inferInsert;
export type SequenceEmail = typeof sequenceEmails.$inferSelect;
export type NewSequenceEmail = typeof sequenceEmails.$inferInsert;
export type EmailSend = typeof emailSends.$inferSelect;
export type NewEmailSend = typeof emailSends.$inferInsert;
export type PageView = typeof pageViews.$inferSelect;
export type NewPageView = typeof pageViews.$inferInsert;
export type ReferralEvent = typeof referralEvents.$inferSelect;
export type NewReferralEvent = typeof referralEvents.$inferInsert;
export type AbTest = typeof abTests.$inferSelect;
export type NewAbTest = typeof abTests.$inferInsert;
export type AbTestVariant = typeof abTestVariants.$inferSelect;
export type NewAbTestVariant = typeof abTestVariants.$inferInsert;
```

### Initial Migration SQL

```sql
-- src/server/db/migrations/0000_init.sql

-- Enable required extensions
CREATE EXTENSION IF NOT EXISTS pg_trgm;

-- (All tables created by Drizzle push, then add supplementary indexes:)

-- Trigram index for searching subscriber emails within a project
CREATE INDEX IF NOT EXISTS subscribers_email_trgm_idx
  ON subscribers USING gin (email gin_trgm_ops);

-- Partial index for active subscribers only (most queries filter by status)
CREATE INDEX IF NOT EXISTS subscribers_active_idx
  ON subscribers (project_id, position)
  WHERE status = 'waitlisted';

-- Composite index for email scheduling queries
CREATE INDEX IF NOT EXISTS email_sends_pending_schedule_idx
  ON email_sends (scheduled_for, status)
  WHERE status IN ('pending', 'queued');

-- Index for referral leaderboard queries
CREATE INDEX IF NOT EXISTS subscribers_referral_leaderboard_idx
  ON subscribers (project_id, referral_count DESC)
  WHERE referral_count > 0;

-- Index for daily signup aggregation (analytics)
CREATE INDEX IF NOT EXISTS subscribers_daily_signups_idx
  ON subscribers (project_id, subscribed_at);

-- Index for page view conversion funnel
CREATE INDEX IF NOT EXISTS page_views_funnel_idx
  ON page_views (project_id, viewed_at, converted);

-- A/B Test tables
CREATE TABLE IF NOT EXISTS ab_tests (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
  name TEXT NOT NULL,
  status TEXT NOT NULL DEFAULT 'draft',
  traffic_split JSONB DEFAULT '{}',
  target_metric TEXT NOT NULL DEFAULT 'signup_rate',
  winner_variant_id UUID,
  significance_level REAL,
  min_sample_size INTEGER NOT NULL DEFAULT 100,
  started_at TIMESTAMPTZ,
  ended_at TIMESTAMPTZ,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX IF NOT EXISTS ab_tests_project_id_idx ON ab_tests(project_id);
CREATE INDEX IF NOT EXISTS ab_tests_status_idx ON ab_tests(project_id, status);

CREATE TABLE IF NOT EXISTS ab_test_variants (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  test_id UUID NOT NULL REFERENCES ab_tests(id) ON DELETE CASCADE,
  name TEXT NOT NULL,
  is_control BOOLEAN NOT NULL DEFAULT false,
  section_overrides JSONB DEFAULT '[]',
  impressions INTEGER NOT NULL DEFAULT 0,
  conversions INTEGER NOT NULL DEFAULT 0,
  conversion_rate REAL NOT NULL DEFAULT 0,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX IF NOT EXISTS ab_test_variants_test_id_idx ON ab_test_variants(test_id);
```

---

## Architecture Deep-Dives

### Deep-Dive 1: AI Landing Page Generation Engine

```typescript
// src/server/ai/generate-landing-page.ts

import OpenAI from "openai";
import { nanoid } from "nanoid";

const openai = new OpenAI({ apiKey: process.env.OPENAI_API_KEY });

// ---------------------------------------------------------------------------
// Types
// ---------------------------------------------------------------------------
export interface PageSection {
  id: string;
  type: "hero" | "problem_solution" | "features" | "social_proof" | "faq" | "cta";
  order: number;
  visible: boolean;
  content: {
    headline?: string;
    subheadline?: string;
    body?: string;
    buttonText?: string;
    items?: Array<{ title: string; description: string; icon?: string }>;
    faqs?: Array<{ question: string; answer: string }>;
    imageUrl?: string;
  };
}

interface PageStructure {
  seoTitle: string;
  seoDescription: string;
  sections: Array<{
    type: PageSection["type"];
    purpose: string;       // brief description for step 2
    messagingAngle: string; // key point to emphasize
  }>;
}

// ---------------------------------------------------------------------------
// Step 1: Generate page structure (low temperature for reliability)
// ---------------------------------------------------------------------------
const STRUCTURE_PROMPT = `You are an expert landing page strategist. Given a product description, generate the optimal landing page structure.

Return ONLY valid JSON with this shape:
{
  "seoTitle": "string (60 chars max, include product name)",
  "seoDescription": "string (155 chars max, compelling meta description)",
  "sections": [
    {
      "type": "hero" | "problem_solution" | "features" | "social_proof" | "faq" | "cta",
      "purpose": "1-sentence description of what this section achieves",
      "messagingAngle": "the key emotional or logical hook for this section"
    }
  ]
}

RULES:
- Always start with "hero" and end with "cta"
- Include 4-6 sections total
- Every page must have: hero, at least one of (features OR problem_solution), and cta
- Choose sections based on the product type and audience
- For B2B: emphasize problem_solution, features, social_proof
- For consumer: emphasize hero (emotional), features, faq
- For newsletters: keep it short — hero, social_proof, cta

PRODUCT DESCRIPTION:
---
{product_description}
---

INDUSTRY: {industry}
TARGET AUDIENCE: {target_audience}`;

async function generatePageStructure(
  productDescription: string,
  industry: string,
  targetAudience: string,
): Promise<PageStructure> {
  const prompt = STRUCTURE_PROMPT
    .replace("{product_description}", productDescription)
    .replace("{industry}", industry || "general")
    .replace("{target_audience}", targetAudience || "general audience");

  const response = await openai.chat.completions.create({
    model: "gpt-4o",
    messages: [{ role: "user", content: prompt }],
    temperature: 0.3,
    max_tokens: 500,
    response_format: { type: "json_object" },
  });

  const content = response.choices[0]?.message?.content ?? "{}";
  const parsed = JSON.parse(content) as PageStructure;

  // Validate required sections
  const types = parsed.sections.map((s) => s.type);
  if (!types.includes("hero")) {
    parsed.sections.unshift({ type: "hero", purpose: "Main headline and CTA", messagingAngle: "Primary value prop" });
  }
  if (!types.includes("cta")) {
    parsed.sections.push({ type: "cta", purpose: "Final conversion", messagingAngle: "Urgency" });
  }

  return parsed;
}

// ---------------------------------------------------------------------------
// Step 2: Generate content for each section (higher temperature for creativity)
// ---------------------------------------------------------------------------
const SECTION_PROMPTS: Record<PageSection["type"], string> = {
  hero: `Generate landing page HERO section copy for this product. Return JSON:
{
  "headline": "string (6-12 words, compelling, benefit-focused)",
  "subheadline": "string (15-25 words, elaborates on the headline)",
  "buttonText": "string (2-4 words, action-oriented CTA)"
}

Product: {product_description}
Messaging angle: {messaging_angle}`,

  problem_solution: `Generate a PROBLEM/SOLUTION section. Return JSON:
{
  "headline": "string (problem-focused headline)",
  "items": [
    { "title": "Problem point 1", "description": "2-sentence elaboration", "icon": "alert-triangle" },
    { "title": "Solution point 1", "description": "2-sentence elaboration", "icon": "check-circle" }
  ]
}
Include 2-3 problem/solution pairs. Use lucide icon names.

Product: {product_description}
Messaging angle: {messaging_angle}`,

  features: `Generate a FEATURES section. Return JSON:
{
  "headline": "string (benefit-oriented section headline)",
  "subheadline": "string (optional supporting text)",
  "items": [
    { "title": "Feature name", "description": "1-2 sentence benefit description", "icon": "lucide-icon-name" }
  ]
}
Include 3-4 features. Focus on BENEFITS not specifications. Use lucide icon names.

Product: {product_description}
Messaging angle: {messaging_angle}`,

  social_proof: `Generate a SOCIAL PROOF section. Return JSON:
{
  "headline": "string (e.g., 'Join X people already waiting')",
  "subheadline": "string (builds credibility)",
  "body": "string (a short testimonial-style quote or social proof statement)"
}

Product: {product_description}
Messaging angle: {messaging_angle}`,

  faq: `Generate an FAQ section. Return JSON:
{
  "headline": "Frequently Asked Questions",
  "faqs": [
    { "question": "string", "answer": "string (2-3 sentences)" }
  ]
}
Include 4-5 FAQs that address common objections and questions about joining the waitlist.

Product: {product_description}
Messaging angle: {messaging_angle}`,

  cta: `Generate a final CTA section. Return JSON:
{
  "headline": "string (urgency-driven, 4-8 words)",
  "subheadline": "string (reinforces value, creates FOMO)",
  "buttonText": "string (2-4 words, action CTA)"
}

Product: {product_description}
Messaging angle: {messaging_angle}`,
};

async function generateSectionContent(
  sectionType: PageSection["type"],
  productDescription: string,
  messagingAngle: string,
): Promise<PageSection["content"]> {
  const promptTemplate = SECTION_PROMPTS[sectionType];
  const prompt = promptTemplate
    .replace("{product_description}", productDescription)
    .replace("{messaging_angle}", messagingAngle);

  const response = await openai.chat.completions.create({
    model: "gpt-4o",
    messages: [{ role: "user", content: prompt }],
    temperature: 0.7,
    max_tokens: 600,
    response_format: { type: "json_object" },
  });

  const content = response.choices[0]?.message?.content ?? "{}";
  return JSON.parse(content) as PageSection["content"];
}

// ---------------------------------------------------------------------------
// Main entry point: full page generation
// ---------------------------------------------------------------------------
export async function generateLandingPage(input: {
  productDescription: string;
  industry: string;
  targetAudience: string;
}): Promise<{
  sections: PageSection[];
  seoTitle: string;
  seoDescription: string;
}> {
  // Step 1: Get optimal page structure
  const structure = await generatePageStructure(
    input.productDescription,
    input.industry,
    input.targetAudience,
  );

  // Step 2: Generate content for each section in parallel
  const sectionPromises = structure.sections.map(async (section, index) => {
    const content = await generateSectionContent(
      section.type,
      input.productDescription,
      section.messagingAngle,
    );

    return {
      id: nanoid(10),
      type: section.type,
      order: index,
      visible: true,
      content,
    } satisfies PageSection;
  });

  const sections = await Promise.all(sectionPromises);

  return {
    sections,
    seoTitle: structure.seoTitle,
    seoDescription: structure.seoDescription,
  };
}

// ---------------------------------------------------------------------------
// Regenerate a single section (used from the editor)
// ---------------------------------------------------------------------------
export async function regenerateSection(input: {
  sectionType: PageSection["type"];
  productDescription: string;
  messagingAngle?: string;
}): Promise<PageSection["content"]> {
  return generateSectionContent(
    input.sectionType,
    input.productDescription,
    input.messagingAngle ?? "primary value proposition",
  );
}
```

### Deep-Dive 2: Waitlist Signup + Referral Engine

```typescript
// src/server/services/waitlist-signup.ts

import { nanoid } from "nanoid";
import { eq, and, sql, desc } from "drizzle-orm";
import { db } from "@/server/db";
import {
  subscribers,
  projects,
  waitlistConfigs,
  referralEvents,
} from "@/server/db/schema";
import { TRPCError } from "@trpc/server";
import { verifyTurnstileToken } from "./turnstile";
import { isDisposableEmail } from "./disposable-emails";
import { checkRateLimit } from "./rate-limit";
import { enqueueSignupEmails } from "@/server/queue/email-scheduler";

// ---------------------------------------------------------------------------
// Types
// ---------------------------------------------------------------------------
interface SignupInput {
  projectId: string;
  email: string;
  name?: string;
  customFields?: Record<string, string>;
  referralCode?: string;       // referral code of the person who referred them
  turnstileToken: string;
  ipAddress: string;
  userAgent: string;
  utmSource?: string;
  utmMedium?: string;
  utmCampaign?: string;
}

interface SignupResult {
  subscriberId: string;
  position: number;
  totalSubscribers: number;
  referralCode: string;
  referredBy?: {
    name: string | null;
    newPosition: number;
  };
}

// ---------------------------------------------------------------------------
// Anti-fraud pipeline
// ---------------------------------------------------------------------------
async function runFraudChecks(input: SignupInput): Promise<{
  passed: boolean;
  reason?: string;
}> {
  // Check 1: Cloudflare Turnstile verification
  const turnstileValid = await verifyTurnstileToken(input.turnstileToken, input.ipAddress);
  if (!turnstileValid) {
    return { passed: false, reason: "turnstile_failed" };
  }

  // Check 2: Rate limit — 10 signups per IP per hour
  const rateLimitOk = await checkRateLimit(`signup:${input.ipAddress}`, 10, 3600);
  if (!rateLimitOk) {
    return { passed: false, reason: "rate_limited" };
  }

  // Check 3: Disposable email detection
  const domain = input.email.split("@")[1]?.toLowerCase();
  if (domain && isDisposableEmail(domain)) {
    return { passed: false, reason: "disposable_email" };
  }

  // Check 4: Duplicate email check for this project
  const [existing] = await db
    .select({ id: subscribers.id })
    .from(subscribers)
    .where(
      and(
        eq(subscribers.projectId, input.projectId),
        eq(subscribers.email, input.email.toLowerCase().trim()),
      ),
    )
    .limit(1);

  if (existing) {
    return { passed: false, reason: "duplicate_email" };
  }

  return { passed: true };
}

// ---------------------------------------------------------------------------
// Referral fraud detection (same IP, rapid signups)
// ---------------------------------------------------------------------------
async function checkReferralFraud(
  referrerId: string,
  refereeIp: string,
  projectId: string,
): Promise<{ suspected: boolean; reason?: string }> {
  // Check if referrer signed up from the same IP
  const [referrer] = await db
    .select({ ipAddress: subscribers.ipAddress })
    .from(subscribers)
    .where(eq(subscribers.id, referrerId))
    .limit(1);

  if (referrer?.ipAddress === refereeIp) {
    return { suspected: true, reason: "same_ip" };
  }

  // Check for rapid referral signups (> 5 referrals in 10 minutes from same referrer)
  const recentReferrals = await db
    .select({ count: sql<number>`count(*)` })
    .from(referralEvents)
    .where(
      and(
        eq(referralEvents.referrerId, referrerId),
        eq(referralEvents.projectId, projectId),
        sql`${referralEvents.createdAt} > now() - interval '10 minutes'`,
      ),
    );

  if (Number(recentReferrals[0]?.count ?? 0) > 5) {
    return { suspected: true, reason: "rapid_signup" };
  }

  return { suspected: false };
}

// ---------------------------------------------------------------------------
// Position calculation: count of non-fraud subscribers with lower position
// ---------------------------------------------------------------------------
async function getNextPosition(projectId: string): Promise<number> {
  const [result] = await db
    .select({ maxPos: sql<number>`COALESCE(MAX(${subscribers.originalPosition}), 0)` })
    .from(subscribers)
    .where(eq(subscribers.projectId, projectId));

  return (result?.maxPos ?? 0) + 1;
}

// ---------------------------------------------------------------------------
// Process referral: bump referrer position, update counts, check tiers
// ---------------------------------------------------------------------------
async function processReferral(input: {
  projectId: string;
  referrerId: string;
  refereeId: string;
  refereeIp: string;
}): Promise<{ newPosition: number; tierUnlocked: number | null }> {
  // Get waitlist config for position bump amount and tiers
  const [config] = await db
    .select()
    .from(waitlistConfigs)
    .where(eq(waitlistConfigs.projectId, input.projectId))
    .limit(1);

  const positionsPerReferral = config?.positionsPerReferral ?? 5;
  const tiers = config?.tiers ?? [];

  // Check for referral fraud
  const fraudCheck = await checkReferralFraud(
    input.referrerId,
    input.refereeIp,
    input.projectId,
  );

  // Increment referral count on the referrer
  const [referrer] = await db
    .update(subscribers)
    .set({
      referralCount: sql`${subscribers.referralCount} + 1`,
      // Only bump position if not fraud-suspected
      position: fraudCheck.suspected
        ? subscribers.position
        : sql`GREATEST(${subscribers.position} - ${positionsPerReferral}, 1)`,
    })
    .where(eq(subscribers.id, input.referrerId))
    .returning();

  // Check if a new tier was unlocked
  let tierUnlocked: number | null = null;
  const newReferralCount = referrer.referralCount;
  for (let i = tiers.length - 1; i >= 0; i--) {
    if (newReferralCount >= tiers[i].referralsRequired && referrer.tierUnlocked < i + 1) {
      tierUnlocked = i + 1;
      await db
        .update(subscribers)
        .set({ tierUnlocked: i + 1 })
        .where(eq(subscribers.id, input.referrerId));
      break;
    }
  }

  // Record the referral event
  await db.insert(referralEvents).values({
    projectId: input.projectId,
    referrerId: input.referrerId,
    refereeId: input.refereeId,
    positionsBumped: fraudCheck.suspected ? 0 : positionsPerReferral,
    tierUnlockedByEvent: tierUnlocked,
    fraudSuspected: fraudCheck.suspected,
    fraudReason: fraudCheck.reason ?? null,
  });

  return { newPosition: referrer.position, tierUnlocked };
}

// ---------------------------------------------------------------------------
// Main signup function (called from public API route)
// ---------------------------------------------------------------------------
export async function handleWaitlistSignup(input: SignupInput): Promise<SignupResult> {
  // Verify project exists and is live
  const [project] = await db
    .select()
    .from(projects)
    .where(and(eq(projects.id, input.projectId), eq(projects.status, "live")))
    .limit(1);

  if (!project) {
    throw new TRPCError({ code: "NOT_FOUND", message: "Waitlist not found or not active" });
  }

  // Check waitlist cap
  if (project.settings?.waitlistCap && project.subscriberCount >= project.settings.waitlistCap) {
    throw new TRPCError({ code: "PRECONDITION_FAILED", message: "Waitlist is full" });
  }

  // Run anti-fraud pipeline
  const fraudResult = await runFraudChecks(input);
  if (!fraudResult.passed) {
    if (fraudResult.reason === "duplicate_email") {
      throw new TRPCError({ code: "CONFLICT", message: "Email already registered" });
    }
    throw new TRPCError({ code: "BAD_REQUEST", message: "Signup verification failed" });
  }

  // Determine device type from user agent
  const device = /mobile|android|iphone/i.test(input.userAgent) ? "mobile"
    : /tablet|ipad/i.test(input.userAgent) ? "tablet" : "desktop";

  // Get next position
  const position = await getNextPosition(input.projectId);

  // Resolve referrer if referral code provided
  let referredBySubscriberId: string | undefined;
  if (input.referralCode) {
    const [referrer] = await db
      .select({ id: subscribers.id })
      .from(subscribers)
      .where(
        and(
          eq(subscribers.referralCode, input.referralCode),
          eq(subscribers.projectId, input.projectId),
        ),
      )
      .limit(1);
    referredBySubscriberId = referrer?.id;
  }

  // Create the subscriber
  const [subscriber] = await db
    .insert(subscribers)
    .values({
      projectId: input.projectId,
      email: input.email.toLowerCase().trim(),
      name: input.name?.trim() || null,
      customFields: input.customFields ?? {},
      position,
      originalPosition: position,
      referralCode: nanoid(8),
      referredBySubscriberId: referredBySubscriberId ?? null,
      referralCount: 0,
      tierUnlocked: 0,
      emailVerified: false,
      verificationToken: nanoid(32),
      ipAddress: input.ipAddress,
      utmSource: input.utmSource ?? null,
      utmMedium: input.utmMedium ?? null,
      utmCampaign: input.utmCampaign ?? null,
      source: referredBySubscriberId ? "referral" : "direct",
      device,
    })
    .returning();

  // Increment project subscriber count
  await db
    .update(projects)
    .set({ subscriberCount: sql`${projects.subscriberCount} + 1` })
    .where(eq(projects.id, input.projectId));

  // Process referral if applicable
  let referredByInfo: SignupResult["referredBy"] | undefined;
  if (referredBySubscriberId) {
    const referralResult = await processReferral({
      projectId: input.projectId,
      referrerId: referredBySubscriberId,
      refereeId: subscriber.id,
      refereeIp: input.ipAddress,
    });

    const [referrer] = await db
      .select({ name: subscribers.name })
      .from(subscribers)
      .where(eq(subscribers.id, referredBySubscriberId))
      .limit(1);

    referredByInfo = {
      name: referrer?.name ?? null,
      newPosition: referralResult.newPosition,
    };
  }

  // Enqueue drip emails for the new subscriber
  await enqueueSignupEmails(subscriber.id, input.projectId);

  return {
    subscriberId: subscriber.id,
    position: subscriber.position,
    totalSubscribers: project.subscriberCount + 1,
    referralCode: subscriber.referralCode,
    referredBy: referredByInfo,
  };
}
```

```typescript
// src/server/services/turnstile.ts

export async function verifyTurnstileToken(
  token: string,
  ipAddress: string,
): Promise<boolean> {
  const response = await fetch(
    "https://challenges.cloudflare.com/turnstile/v0/siteverify",
    {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({
        secret: process.env.TURNSTILE_SECRET_KEY!,
        response: token,
        remoteip: ipAddress,
      }),
    },
  );

  const data = (await response.json()) as { success: boolean };
  return data.success;
}
```

```typescript
// src/server/services/rate-limit.ts

import { Redis } from "@upstash/redis";

const redis = new Redis({
  url: process.env.UPSTASH_REDIS_URL!,
  token: process.env.UPSTASH_REDIS_TOKEN!,
});

export async function checkRateLimit(
  key: string,
  maxRequests: number,
  windowSeconds: number,
): Promise<boolean> {
  const current = await redis.incr(key);
  if (current === 1) {
    await redis.expire(key, windowSeconds);
  }
  return current <= maxRequests;
}
```

### Deep-Dive 3: Hosted Landing Page Renderer with ISR

```typescript
// src/app/p/[slug]/page.tsx

import { Metadata } from "next";
import { notFound } from "next/navigation";
import { db } from "@/server/db";
import { projects, landingPages, waitlistConfigs } from "@/server/db/schema";
import { eq, and } from "drizzle-orm";
import { LandingPageRenderer } from "@/components/landing/landing-page-renderer";
import { PageViewTracker } from "@/components/landing/page-view-tracker";

interface PageProps {
  params: Promise<{ slug: string }>;
  searchParams: Promise<{ ref?: string; utm_source?: string; utm_medium?: string; utm_campaign?: string }>;
}

// ISR: revalidate on demand via revalidateTag("page-{slug}")
export const revalidate = 3600; // fallback: revalidate every hour

async function getPageData(slug: string) {
  const [project] = await db
    .select()
    .from(projects)
    .where(and(eq(projects.slug, slug), eq(projects.status, "live")))
    .limit(1);

  if (!project) return null;

  const [page] = await db
    .select()
    .from(landingPages)
    .where(eq(landingPages.projectId, project.id))
    .limit(1);

  const [config] = await db
    .select()
    .from(waitlistConfigs)
    .where(eq(waitlistConfigs.projectId, project.id))
    .limit(1);

  return { project, page, config };
}

// Dynamic OG metadata generation
export async function generateMetadata({ params }: PageProps): Promise<Metadata> {
  const { slug } = await params;
  const data = await getPageData(slug);
  if (!data) return {};

  const { project, page } = data;
  const title = page?.seoTitle ?? project.name;
  const description = page?.seoDescription ?? project.tagline ?? project.productDescription.slice(0, 155);

  return {
    title,
    description,
    openGraph: {
      title,
      description,
      type: "website",
      url: `https://waitlistiq.io/p/${slug}`,
      images: page?.ogImageUrl
        ? [{ url: page.ogImageUrl, width: 1200, height: 630, alt: title }]
        : [],
      siteName: "WaitlistIQ",
    },
    twitter: {
      card: "summary_large_image",
      title,
      description,
      images: page?.ogImageUrl ? [page.ogImageUrl] : [],
    },
    robots: { index: true, follow: true },
  };
}

export default async function PublicLandingPage({ params, searchParams }: PageProps) {
  const { slug } = await params;
  const { ref, utm_source, utm_medium, utm_campaign } = await searchParams;
  const data = await getPageData(slug);

  if (!data || !data.page) notFound();

  const { project, page, config } = data;
  const sections = (page.sections ?? []).filter((s) => s.visible).sort((a, b) => a.order - b.order);

  return (
    <>
      <PageViewTracker
        projectId={project.id}
        utmSource={utm_source}
        utmMedium={utm_medium}
        utmCampaign={utm_campaign}
      />
      <LandingPageRenderer
        sections={sections}
        project={{
          id: project.id,
          name: project.name,
          slug: project.slug,
          subscriberCount: project.subscriberCount,
          showWaitlistCount: project.settings?.showWaitlistCount ?? true,
          requireName: project.settings?.requireName ?? false,
          brandColor: project.settings?.brandColor ?? "#6366f1",
          accentColor: project.settings?.accentColor ?? "#8b5cf6",
          logoUrl: project.settings?.logoUrl,
        }}
        referralCode={ref}
        template={page.template}
      />
    </>
  );
}
```

```typescript
// src/components/landing/landing-page-renderer.tsx
"use client";

import { motion } from "framer-motion";
import { HeroSection } from "./sections/hero-section";
import { ProblemSolutionSection } from "./sections/problem-solution-section";
import { FeaturesSection } from "./sections/features-section";
import { SocialProofSection } from "./sections/social-proof-section";
import { FaqSection } from "./sections/faq-section";
import { CtaSection } from "./sections/cta-section";
import type { PageSection } from "@/server/ai/generate-landing-page";

interface ProjectContext {
  id: string;
  name: string;
  slug: string;
  subscriberCount: number;
  showWaitlistCount: boolean;
  requireName: boolean;
  brandColor: string;
  accentColor: string;
  logoUrl?: string;
}

interface Props {
  sections: PageSection[];
  project: ProjectContext;
  referralCode?: string;
  template: string;
}

const SECTION_COMPONENTS: Record<PageSection["type"], React.ComponentType<any>> = {
  hero: HeroSection,
  problem_solution: ProblemSolutionSection,
  features: FeaturesSection,
  social_proof: SocialProofSection,
  faq: FaqSection,
  cta: CtaSection,
};

const TEMPLATE_CLASSES: Record<string, string> = {
  minimal: "bg-white text-gray-900",
  bold: "bg-gray-950 text-white",
  gradient: "bg-gradient-to-br from-indigo-50 via-white to-purple-50 text-gray-900",
  dark: "bg-gray-900 text-gray-100",
  startup: "bg-white text-gray-800",
  product: "bg-slate-50 text-gray-900",
};

export function LandingPageRenderer({ sections, project, referralCode, template }: Props) {
  const baseClass = TEMPLATE_CLASSES[template] ?? TEMPLATE_CLASSES.minimal;

  return (
    <div className={`min-h-screen ${baseClass}`}>
      {project.logoUrl && (
        <header className="flex items-center justify-between px-6 py-4 max-w-6xl mx-auto">
          <img src={project.logoUrl} alt={project.name} className="h-8" />
        </header>
      )}

      {sections.map((section, index) => {
        const Component = SECTION_COMPONENTS[section.type];
        if (!Component) return null;

        return (
          <motion.div
            key={section.id}
            initial={{ opacity: 0, y: 20 }}
            whileInView={{ opacity: 1, y: 0 }}
            viewport={{ once: true, margin: "-50px" }}
            transition={{ duration: 0.5, delay: index * 0.1 }}
          >
            <Component
              content={section.content}
              project={project}
              referralCode={referralCode}
              brandColor={project.brandColor}
            />
          </motion.div>
        );
      })}

      <footer className="text-center py-6 text-sm text-gray-400">
        Powered by <a href="https://waitlistiq.io" className="underline hover:text-gray-600">WaitlistIQ</a>
      </footer>
    </div>
  );
}
```

```typescript
// src/components/landing/sections/hero-section.tsx
"use client";

import { useState } from "react";
import { motion, AnimatePresence } from "framer-motion";
import { SignupForm } from "../signup-form";
import type { PageSection } from "@/server/ai/generate-landing-page";

interface Props {
  content: PageSection["content"];
  project: {
    id: string;
    name: string;
    subscriberCount: number;
    showWaitlistCount: boolean;
    requireName: boolean;
  };
  referralCode?: string;
  brandColor: string;
}

export function HeroSection({ content, project, referralCode, brandColor }: Props) {
  const [signedUp, setSignedUp] = useState(false);

  return (
    <section className="px-6 py-20 md:py-32 max-w-4xl mx-auto text-center">
      {project.showWaitlistCount && project.subscriberCount > 0 && (
        <motion.div
          initial={{ scale: 0.9, opacity: 0 }}
          animate={{ scale: 1, opacity: 1 }}
          className="inline-flex items-center gap-2 px-4 py-1.5 rounded-full bg-gray-100 text-sm text-gray-600 mb-6"
        >
          <span className="w-2 h-2 rounded-full bg-green-500 animate-pulse" />
          <span>{project.subscriberCount.toLocaleString()} people on the waitlist</span>
        </motion.div>
      )}

      <h1 className="text-4xl md:text-6xl font-bold tracking-tight mb-6 leading-tight">
        {content.headline}
      </h1>

      <p className="text-lg md:text-xl text-gray-500 max-w-2xl mx-auto mb-10">
        {content.subheadline}
      </p>

      <AnimatePresence mode="wait">
        {!signedUp ? (
          <motion.div key="form" exit={{ opacity: 0, y: -10 }}>
            <SignupForm
              projectId={project.id}
              referralCode={referralCode}
              buttonText={content.buttonText ?? "Join the Waitlist"}
              requireName={project.requireName}
              brandColor={brandColor}
              onSuccess={() => setSignedUp(true)}
            />
          </motion.div>
        ) : (
          <motion.div
            key="success"
            initial={{ opacity: 0, y: 10 }}
            animate={{ opacity: 1, y: 0 }}
            className="text-green-600 font-medium"
          >
            You are on the list! Check your email for your position.
          </motion.div>
        )}
      </AnimatePresence>
    </section>
  );
}
```

### Deep-Dive 4: Email Drip Scheduler

```typescript
// src/server/queue/email-scheduler.ts

import { Queue, Worker, Job } from "bullmq";
import { Redis } from "ioredis";
import { db } from "@/server/db";
import {
  emailSequences,
  sequenceEmails,
  emailSends,
  subscribers,
  projects,
  waitlistConfigs,
} from "@/server/db/schema";
import { eq, and, asc } from "drizzle-orm";
import { Resend } from "resend";

const redis = new Redis(process.env.UPSTASH_REDIS_URL!, {
  maxRetriesPerRequest: null,
});

const resend = new Resend(process.env.RESEND_API_KEY);

// ---------------------------------------------------------------------------
// Queue definition
// ---------------------------------------------------------------------------
export interface EmailJobData {
  emailSendId: string;      // ID of the emailSends record
  subscriberId: string;
  sequenceEmailId: string;
  projectId: string;
}

export const emailQueue = new Queue<EmailJobData>("email-sends", {
  connection: redis,
  defaultJobOptions: {
    attempts: 3,
    backoff: { type: "exponential", delay: 5000 },
    removeOnComplete: { age: 86400 * 7 },  // keep 7 days
    removeOnFail: { age: 86400 * 30 },     // keep 30 days
  },
});

// ---------------------------------------------------------------------------
// Enqueue all drip emails for a new subscriber
// ---------------------------------------------------------------------------
export async function enqueueSignupEmails(
  subscriberId: string,
  projectId: string,
): Promise<void> {
  // Get all active signup-triggered sequences for this project
  const sequences = await db
    .select()
    .from(emailSequences)
    .where(
      and(
        eq(emailSequences.projectId, projectId),
        eq(emailSequences.trigger, "signup"),
        eq(emailSequences.isActive, true),
      ),
    );

  for (const sequence of sequences) {
    // Get all emails in this sequence, ordered
    const emails = await db
      .select()
      .from(sequenceEmails)
      .where(eq(sequenceEmails.sequenceId, sequence.id))
      .orderBy(asc(sequenceEmails.sortOrder));

    for (const email of emails) {
      const delayMs = email.delayHours * 60 * 60 * 1000;
      const scheduledFor = new Date(Date.now() + delayMs);

      // Create the emailSend record (pending)
      const [send] = await db
        .insert(emailSends)
        .values({
          subscriberId,
          sequenceEmailId: email.id,
          projectId,
          status: "queued",
          scheduledFor,
        })
        .returning();

      // Enqueue the BullMQ job with a delay
      await emailQueue.add(
        `send-${send.id}`,
        {
          emailSendId: send.id,
          subscriberId,
          sequenceEmailId: email.id,
          projectId,
        },
        {
          delay: delayMs,
          jobId: `email-${send.id}`, // deduplicate
        },
      );
    }
  }
}

// ---------------------------------------------------------------------------
// Merge field rendering
// ---------------------------------------------------------------------------
interface MergeContext {
  first_name: string;
  position: string;
  original_position: string;
  referral_link: string;
  referral_count: string;
  product_name: string;
  waitlist_count: string;
  tier_name: string;
}

function renderMergeFields(template: string, context: MergeContext): string {
  return template.replace(
    /\{(\w+)\}/g,
    (match, key) => (context as Record<string, string>)[key] ?? match,
  );
}

async function buildMergeContext(
  subscriberId: string,
  projectId: string,
): Promise<MergeContext> {
  const [subscriber] = await db
    .select()
    .from(subscribers)
    .where(eq(subscribers.id, subscriberId))
    .limit(1);

  const [project] = await db
    .select()
    .from(projects)
    .where(eq(projects.id, projectId))
    .limit(1);

  const [config] = await db
    .select()
    .from(waitlistConfigs)
    .where(eq(waitlistConfigs.projectId, projectId))
    .limit(1);

  const tiers = config?.tiers ?? [];
  const currentTier = subscriber?.tierUnlocked ?? 0;
  const tierName = currentTier > 0 && tiers[currentTier - 1]
    ? tiers[currentTier - 1].rewardName
    : "None yet";

  const referralLink = `https://waitlistiq.io/p/${project?.slug}?ref=${subscriber?.referralCode}`;

  return {
    first_name: subscriber?.name?.split(" ")[0] ?? "there",
    position: String(subscriber?.position ?? 0),
    original_position: String(subscriber?.originalPosition ?? 0),
    referral_link: referralLink,
    referral_count: String(subscriber?.referralCount ?? 0),
    product_name: project?.name ?? "our product",
    waitlist_count: String(project?.subscriberCount ?? 0),
    tier_name: tierName,
  };
}

// ---------------------------------------------------------------------------
// Worker: processes each delayed email job
// ---------------------------------------------------------------------------
export const emailWorker = new Worker<EmailJobData>(
  "email-sends",
  async (job: Job<EmailJobData>) => {
    const { emailSendId, subscriberId, sequenceEmailId, projectId } = job.data;

    // Verify the subscriber still exists and is still waitlisted
    const [subscriber] = await db
      .select()
      .from(subscribers)
      .where(eq(subscribers.id, subscriberId))
      .limit(1);

    if (!subscriber || subscriber.status !== "waitlisted") {
      await db
        .update(emailSends)
        .set({ status: "failed", errorMessage: "Subscriber no longer waitlisted" })
        .where(eq(emailSends.id, emailSendId));
      return;
    }

    // Get the email template
    const [sequenceEmail] = await db
      .select()
      .from(sequenceEmails)
      .where(eq(sequenceEmails.id, sequenceEmailId))
      .limit(1);

    if (!sequenceEmail) {
      await db
        .update(emailSends)
        .set({ status: "failed", errorMessage: "Sequence email template not found" })
        .where(eq(emailSends.id, emailSendId));
      return;
    }

    // Get project for the "from" address
    const [project] = await db
      .select()
      .from(projects)
      .where(eq(projects.id, projectId))
      .limit(1);

    // Build merge context and render the email
    const context = await buildMergeContext(subscriberId, projectId);
    const renderedSubject = renderMergeFields(sequenceEmail.subject, context);
    const renderedBody = renderMergeFields(sequenceEmail.bodyHtml, context);

    try {
      const result = await resend.emails.send({
        from: `${project?.name ?? "WaitlistIQ"} <waitlist@waitlistiq.io>`,
        to: subscriber.email,
        subject: renderedSubject,
        html: renderedBody,
      });

      await db
        .update(emailSends)
        .set({
          status: "sent",
          sentAt: new Date(),
          resendMessageId: result.data?.id,
        })
        .where(eq(emailSends.id, emailSendId));
    } catch (error) {
      const errorMessage = error instanceof Error ? error.message : "Unknown error";
      await db
        .update(emailSends)
        .set({ status: "failed", errorMessage })
        .where(eq(emailSends.id, emailSendId));
      throw error; // Re-throw so BullMQ retries
    }
  },
  {
    connection: redis,
    concurrency: 10,
    limiter: {
      max: 50,       // max 50 emails per
      duration: 1000, // 1 second (Resend rate limit)
    },
  },
);

// ---------------------------------------------------------------------------
// Resend webhook handler for tracking opens/clicks
// ---------------------------------------------------------------------------
export async function handleResendWebhook(payload: {
  type: "email.opened" | "email.clicked" | "email.bounced" | "email.delivered";
  data: { email_id: string };
}): Promise<void> {
  const resendMessageId = payload.data.email_id;

  const [send] = await db
    .select()
    .from(emailSends)
    .where(eq(emailSends.resendMessageId, resendMessageId))
    .limit(1);

  if (!send) return;

  switch (payload.type) {
    case "email.opened":
      if (!send.openedAt) {
        await db
          .update(emailSends)
          .set({ status: "opened", openedAt: new Date() })
          .where(eq(emailSends.id, send.id));
      }
      break;
    case "email.clicked":
      await db
        .update(emailSends)
        .set({ status: "clicked", clickedAt: new Date() })
        .where(eq(emailSends.id, send.id));
      break;
    case "email.bounced":
      await db
        .update(emailSends)
        .set({ status: "bounced" })
        .where(eq(emailSends.id, send.id));
      break;
  }
}
```

### Deep-Dive 5: A/B Testing Engine

The A/B testing engine enables landing page experiments with deterministic variant assignment, conversion tracking, and statistical significance calculation.

```typescript
// src/server/services/ab-testing.ts

import { eq, and, sql } from "drizzle-orm";
import { db } from "@/server/db";
import {
  abTests,
  abTestVariants,
  pageViews,
  subscribers,
} from "@/server/db/schema";
import murmurhash from "murmurhash";

// ---------------------------------------------------------------------------
// Traffic Splitter — deterministic variant assignment via murmur3 hash
// ---------------------------------------------------------------------------
export function assignVariant(
  testId: string,
  visitorSessionId: string,
  trafficSplit: Record<string, number>,
): string {
  // Murmur3 hash ensures same visitor always sees same variant
  const hash = murmurhash.v3(`${testId}:${visitorSessionId}`);
  const bucket = hash % 100; // 0-99

  let cumulative = 0;
  for (const [variantName, percentage] of Object.entries(trafficSplit)) {
    cumulative += percentage;
    if (bucket < cumulative) {
      return variantName;
    }
  }

  // Fallback to first variant
  return Object.keys(trafficSplit)[0] ?? "control";
}

// ---------------------------------------------------------------------------
// Get active test for a project with variant assignment
// ---------------------------------------------------------------------------
export async function getActiveTestVariant(
  projectId: string,
  sessionId: string,
): Promise<{
  testId: string;
  variantId: string;
  variantName: string;
  sectionOverrides: Array<{ sectionId: string; field: string; value: string }>;
} | null> {
  const [activeTest] = await db
    .select()
    .from(abTests)
    .where(
      and(
        eq(abTests.projectId, projectId),
        eq(abTests.status, "running"),
      ),
    )
    .limit(1);

  if (!activeTest) return null;

  const variantName = assignVariant(
    activeTest.id,
    sessionId,
    activeTest.trafficSplit as Record<string, number>,
  );

  const [variant] = await db
    .select()
    .from(abTestVariants)
    .where(
      and(
        eq(abTestVariants.testId, activeTest.id),
        eq(abTestVariants.name, variantName),
      ),
    )
    .limit(1);

  if (!variant) return null;

  // Increment impressions
  await db
    .update(abTestVariants)
    .set({ impressions: sql`${abTestVariants.impressions} + 1` })
    .where(eq(abTestVariants.id, variant.id));

  return {
    testId: activeTest.id,
    variantId: variant.id,
    variantName: variant.name,
    sectionOverrides: variant.sectionOverrides ?? [],
  };
}

// ---------------------------------------------------------------------------
// Record conversion (called on successful signup)
// ---------------------------------------------------------------------------
export async function recordConversion(
  testId: string,
  variantId: string,
): Promise<void> {
  await db
    .update(abTestVariants)
    .set({
      conversions: sql`${abTestVariants.conversions} + 1`,
      conversionRate: sql`CASE
        WHEN ${abTestVariants.impressions} > 0
        THEN (${abTestVariants.conversions} + 1)::real / ${abTestVariants.impressions}::real
        ELSE 0 END`,
    })
    .where(eq(abTestVariants.id, variantId));
}

// ---------------------------------------------------------------------------
// Winner selection via chi-squared test (p < 0.05)
// ---------------------------------------------------------------------------
export async function checkForWinner(testId: string): Promise<{
  hasWinner: boolean;
  winnerId?: string;
  pValue?: number;
}> {
  const variants = await db
    .select()
    .from(abTestVariants)
    .where(eq(abTestVariants.testId, testId));

  if (variants.length < 2) return { hasWinner: false };

  // Check minimum sample size
  const [test] = await db
    .select()
    .from(abTests)
    .where(eq(abTests.id, testId))
    .limit(1);

  const minSample = test?.minSampleSize ?? 100;
  if (variants.some((v) => v.impressions < minSample)) {
    return { hasWinner: false };
  }

  // Chi-squared test for independence
  const totalImpressions = variants.reduce((s, v) => s + v.impressions, 0);
  const totalConversions = variants.reduce((s, v) => s + v.conversions, 0);
  const expectedConversionRate = totalConversions / totalImpressions;

  let chiSquared = 0;
  for (const variant of variants) {
    const expectedConversions = variant.impressions * expectedConversionRate;
    const expectedNonConversions = variant.impressions * (1 - expectedConversionRate);

    if (expectedConversions > 0) {
      chiSquared += Math.pow(variant.conversions - expectedConversions, 2) / expectedConversions;
    }
    if (expectedNonConversions > 0) {
      const nonConversions = variant.impressions - variant.conversions;
      chiSquared += Math.pow(nonConversions - expectedNonConversions, 2) / expectedNonConversions;
    }
  }

  // df = (rows - 1) * (cols - 1) = (variants - 1) * 1
  // For df=1, chi-squared critical value at p<0.05 is 3.841
  const df = variants.length - 1;
  const criticalValues: Record<number, number> = { 1: 3.841, 2: 5.991, 3: 7.815 };
  const criticalValue = criticalValues[df] ?? 3.841;

  if (chiSquared >= criticalValue) {
    // Find variant with highest conversion rate
    const winner = variants.reduce((best, v) =>
      v.conversionRate > best.conversionRate ? v : best,
    );

    // Approximate p-value (simplified)
    const pValue = chiSquared >= 10.83 ? 0.001
      : chiSquared >= 6.635 ? 0.01
      : chiSquared >= 3.841 ? 0.05
      : 1;

    return { hasWinner: true, winnerId: winner.id, pValue };
  }

  return { hasWinner: false };
}
```

### Deep-Dive 6: Referral Milestone Trigger Logic

The referral milestone system detects when a referrer crosses tier thresholds, triggers automated reward emails, and includes fraud detection for suspicious referral patterns.

```typescript
// src/server/services/referral-milestones.ts

import { eq, and, sql } from "drizzle-orm";
import { db } from "@/server/db";
import {
  subscribers,
  waitlistConfigs,
  referralEvents,
  emailSequences,
  sequenceEmails,
} from "@/server/db/schema";
import { enqueueEmailForSubscriber } from "@/server/queue/email-scheduler";

// ---------------------------------------------------------------------------
// SLA Tier Definitions (configurable per project via waitlistConfigs.tiers)
// ---------------------------------------------------------------------------
// Default tiers if none configured:
// Tier 1: 3 referrals → "Early Access" (priority position bump)
// Tier 2: 5 referrals → "VIP Access" (skip to top 10%)
// Tier 3: 10 referrals → "Founding Member" (lifetime discount + beta access)
// Tier 4: 25 referrals → "Ambassador" (featured on landing page + swag)

// ---------------------------------------------------------------------------
// Milestone Detection — called after each successful referral
// ---------------------------------------------------------------------------
export async function detectAndProcessMilestone(
  referrerId: string,
  projectId: string,
): Promise<{
  milestoneReached: boolean;
  tierIndex: number | null;
  tierName: string | null;
}> {
  const [referrer] = await db
    .select()
    .from(subscribers)
    .where(eq(subscribers.id, referrerId))
    .limit(1);

  if (!referrer) return { milestoneReached: false, tierIndex: null, tierName: null };

  const [config] = await db
    .select()
    .from(waitlistConfigs)
    .where(eq(waitlistConfigs.projectId, projectId))
    .limit(1);

  const tiers = config?.tiers ?? [];
  if (tiers.length === 0) return { milestoneReached: false, tierIndex: null, tierName: null };

  // Check each tier from highest to lowest
  for (let i = tiers.length - 1; i >= 0; i--) {
    const tier = tiers[i];
    if (
      referrer.referralCount >= tier.referralsRequired &&
      referrer.tierUnlocked < i + 1
    ) {
      // New milestone reached!
      await db
        .update(subscribers)
        .set({ tierUnlocked: i + 1 })
        .where(eq(subscribers.id, referrerId));

      // Trigger milestone email sequence
      await triggerMilestoneEmail(referrerId, projectId, tier.rewardName);

      return {
        milestoneReached: true,
        tierIndex: i + 1,
        tierName: tier.rewardName,
      };
    }
  }

  return { milestoneReached: false, tierIndex: null, tierName: null };
}

// ---------------------------------------------------------------------------
// Trigger milestone reward email
// ---------------------------------------------------------------------------
async function triggerMilestoneEmail(
  subscriberId: string,
  projectId: string,
  tierName: string,
): Promise<void> {
  // Find the referral_milestone sequence for this project
  const [sequence] = await db
    .select()
    .from(emailSequences)
    .where(
      and(
        eq(emailSequences.projectId, projectId),
        eq(emailSequences.trigger, "referral_milestone"),
        eq(emailSequences.isActive, true),
      ),
    )
    .limit(1);

  if (!sequence) return;

  // Enqueue all emails in the milestone sequence
  await enqueueEmailForSubscriber(subscriberId, sequence.id, projectId);
}

// ---------------------------------------------------------------------------
// Referral fraud detection (enhanced)
// ---------------------------------------------------------------------------
export async function detectReferralFraud(
  referrerId: string,
  refereeEmail: string,
  refereeIp: string,
  projectId: string,
): Promise<{
  isFraudulent: boolean;
  reasons: string[];
}> {
  const reasons: string[] = [];

  // Check 1: Same IP as referrer
  const [referrer] = await db
    .select({ ipAddress: subscribers.ipAddress, email: subscribers.email })
    .from(subscribers)
    .where(eq(subscribers.id, referrerId))
    .limit(1);

  if (referrer?.ipAddress === refereeIp) {
    reasons.push("same_ip");
  }

  // Check 2: Same email domain pattern (e.g., user+1@gmail.com, user+2@gmail.com)
  const referrerBase = referrer?.email?.replace(/\+.*@/, "@");
  const refereeBase = refereeEmail.replace(/\+.*@/, "@");
  if (referrerBase && referrerBase === refereeBase) {
    reasons.push("same_email_base");
  }

  // Check 3: Rapid referral velocity (> 5 in 10 minutes)
  const [rapidCount] = await db
    .select({ count: sql<number>`count(*)` })
    .from(referralEvents)
    .where(
      and(
        eq(referralEvents.referrerId, referrerId),
        eq(referralEvents.projectId, projectId),
        sql`${referralEvents.createdAt} > now() - interval '10 minutes'`,
      ),
    );

  if (Number(rapidCount?.count ?? 0) > 5) {
    reasons.push("rapid_velocity");
  }

  // Check 4: Disposable email domain on referee
  const { isDisposableEmail } = await import("./disposable-emails");
  const refereeDomain = refereeEmail.split("@")[1]?.toLowerCase();
  if (refereeDomain && isDisposableEmail(refereeDomain)) {
    reasons.push("disposable_referee_email");
  }

  return {
    isFraudulent: reasons.length > 0,
    reasons,
  };
}
```

---

## A/B Testing Implementation Phase

> **Critical Coverage Gap:** A/B testing is listed in MVP scope but had zero implementation coverage. The following phase addresses this gap with a complete implementation plan.

### Phase — A/B Testing (2-3 Days, inserted after Landing Page Editor)

**Day A: A/B test CRUD + variant assignment**
- `src/server/routers/ab-test.ts`:
  - `create`: create test with name, target metric, traffic split percentages
  - `addVariant`: add variant with section overrides (headline/subheadline/CTA overrides per section)
  - `start`: validate >=2 variants, traffic split sums to 100, transition to "running"
  - `stop`: end test, calculate final stats
  - `getResults`: return variant stats with conversion rates
- `src/server/services/ab-testing.ts`:
  - `assignVariant(testId, sessionId)`: murmur3 hash for deterministic, cookie-stable assignment
  - `getActiveTestVariant(projectId, sessionId)`: return variant + section overrides for rendering
  - `recordConversion(testId, variantId)`: increment conversions + recalculate rate

**Day B: Landing page integration + results UI**
- Modify `src/app/p/[slug]/page.tsx`:
  - On page load, check for active A/B test via `getActiveTestVariant()`
  - Apply `sectionOverrides` from assigned variant to landing page sections before rendering
  - Store variant assignment in session cookie for consistent experience
  - On successful signup, call `recordConversion()` with the assigned variant
- Modify `src/components/landing/landing-page-renderer.tsx`:
  - Accept optional `sectionOverrides` prop
  - Override section content fields (headline, subheadline, buttonText) with variant values
- `src/app/(dashboard)/projects/[slug]/ab-tests/page.tsx`:
  - Test list with status badges (draft, running, completed)
  - Create test wizard: select sections to test, enter variant copy, set traffic split
  - Results view: conversion rate per variant, impressions, statistical significance indicator

**Day C: Winner selection + auto-apply**
- `src/server/services/ab-testing.ts` → `checkForWinner()`:
  - Chi-squared test for independence across variants
  - Significance threshold: p < 0.05 (chi-squared critical value 3.841 for df=1)
  - Minimum sample size enforcement (default 100 impressions per variant)
- Auto-check for winner on every 50th conversion (via modulo check in `recordConversion`)
- When winner found:
  - Mark test as "completed" with winner variant ID and p-value
  - Send notification email to project owner
  - Dashboard shows "Apply Winner" button to merge winning copy into landing page
- `src/server/routers/ab-test.ts` → `applyWinner`:
  - Copy winning variant's section overrides into the landing page sections permanently
  - Archive the test
  - Trigger ISR revalidation

---

## Social Proof Widget (Expanded)

The social proof widget provides real-time signup notifications and customizable display options for the landing page:

- **Real-time signup popups**: Toast-style notifications showing recent signups ("Sarah from New York just joined!"). Configurable display frequency (every 30s, 60s, or on-scroll). Privacy controls: show first name + city only, anonymize ("Someone from California"), or disable entirely.
- **Customizable styles**: Match brand colors, position (bottom-left, bottom-right, top), animation style (slide-in, fade, bounce). Dark mode support.
- **Counter widget**: Animated counter showing total waitlist signups. Configurable format: exact count, rounded ("2,400+"), or milestone-based ("Over 2,000").
- **Privacy controls**: GDPR-compliant opt-in for name display. Subscribers can opt out of being shown in social proof. City derived from IP geolocation (country-level only, no precise location).
- **Implementation**: Client component fetching from `/api/public/social-proof/[projectId]` endpoint returning latest 5 signups (name, city, timestamp). Polling every 30 seconds when tab is visible.

---

## Launch Day Tools (F8)

Launch day tools enable project owners to transition from waitlist to active product:

- **Access Codes**: Generate batch access codes (configurable format: alphanumeric 8-char, nanoid). Distribute via CSV export or email blast. One-time-use with expiry dates. Track redemption status.
- **Batch Invite**: Select subscribers by position range (e.g., positions 1-100), referral count, or tier. Send invite email with unique access code. Rate-limited to 500 invites per batch to avoid email provider throttling.
- **Queue Priority Override**: Admin ability to manually move subscribers to specific positions. Bulk position adjustment for VIP/press/partner lists. Audit trail for all manual position changes.
- **Countdown Timer Widget**: Embeddable countdown timer component for landing page. Configurable target date/time with timezone. Auto-transition to "Doors Open" state. Integrates with project status change (draft -> live -> launched).

---

## Performance Benchmarks

| Operation | Target | Method |
|-----------|--------|--------|
| Landing page load (ISR) | < 100ms | Static HTML, edge-cached via Vercel CDN |
| AI page generation (full) | < 15s | Step 1 structure (2s) + Step 2 parallel sections (8-12s) |
| Waitlist signup API | < 500ms | Fraud checks + DB insert + queue enqueue |
| Referral position recalculation | < 200ms | Single UPDATE with GREATEST() |
| Email drip scheduling | < 100ms | BullMQ delayed job enqueue per email |
| A/B variant assignment | < 5ms | Murmur3 hash, no DB lookup needed |
| A/B conversion recording | < 50ms | Single atomic UPDATE |
| A/B winner check (chi-squared) | < 100ms | In-memory calculation on variant stats |
| Social proof widget load | < 200ms | Cached endpoint, 5 recent signups |
| Analytics dashboard query | < 1s | Indexed aggregation queries |
| Subscriber list (paginated) | < 500ms | Indexed by project + position |
| Page editor save | < 300ms | JSON update + ISR revalidation trigger |

---

## Post-MVP Roadmap

| ID | Feature | Description | Priority |
|----|---------|-------------|----------|
| F5 | Multi-Variant A/B Testing | Expand beyond 2 variants to support multivariate testing (test multiple sections simultaneously). Bayesian statistical model for faster winner detection with smaller sample sizes. | High |
| F6 | Custom Domain SSL | Full custom domain support with automated SSL provisioning via Vercel. CNAME verification flow, subdomain and apex domain support. | High |
| F7 | Webhook Integrations | Outbound webhooks on signup, referral milestone, and launch events. Zapier/Make integration for connecting to CRMs, Slack, and other tools. | Medium |
| F8 | Launch Day Tools | Access code generation and batch distribution, batch invite by position/tier, queue priority override for VIP lists, embeddable countdown timer widget. | High |
| F9 | Advanced Referral Rewards | Physical reward fulfillment integration (Printful, Shopify). Digital reward codes (coupon generation). Gamification: referral leaderboard with weekly prizes. | Medium |
| F10 | Email Template Builder | Visual drag-and-drop email template editor (MJML-based). Preview across email clients. Template library for common waitlist email patterns. | Medium |
| F11 | Multi-Language Landing Pages | I18n support for landing pages. AI translation of page sections. Geo-targeted language selection. RTL layout support. | Low |
| F12 | Team Collaboration | Multi-user access per project with role-based permissions (owner, editor, viewer). Activity log for all changes. Comment threads on landing page sections. | Low |
