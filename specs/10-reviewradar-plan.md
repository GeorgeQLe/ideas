# 10. ReviewRadar — Multi-Platform Review Monitoring

## Implementation Plan

**MVP Scope:** Google Business Profile API integration + Yelp scraping via Playwright + G2 scraping via Playwright for unified review aggregation, real-time email and Slack alerts for new reviews with configurable alert rules (rating threshold, platform filter, keyword match), AI-powered response drafting with four tone options (grateful / professional / empathetic / apologetic) using GPT-4o-mini in JSON mode, sentiment analysis with topic extraction on every ingested review, a unified review feed with filtering by platform / rating / sentiment / responded-unresponded status, a rating trends chart over time using Recharts, organization and location management, and Stripe billing with three tiers: Free (1 location, 2 platforms, email alerts only, 30-day history, no AI), Pro at $29/month (1 location, 5 platforms, 50 AI responses/month, sentiment analysis, Slack integration, 1-year history), and Business at $69/month (5 locations, unlimited platforms, unlimited AI responses, all integrations, team collaboration with 5 users).

---

## Tech Decisions

| Layer | Technology | Notes |
|-------|-----------|-------|
| Framework | Next.js 14+ App Router | `src/` directory, TypeScript strict |
| API | tRPC v11 | superjson transformer, Clerk auth context |
| Database | Neon PostgreSQL | Managed serverless Postgres |
| ORM | Drizzle ORM | Push-based migrations, typed schema |
| Auth | Clerk | Organization-scoped, `orgId` from session |
| Billing | Stripe | Checkout Sessions + Customer Portal + Webhooks |
| AI | OpenAI GPT-4o-mini | JSON mode for sentiment analysis, topic extraction, and response drafting |
| Queue | BullMQ + Redis (Upstash) | Scheduled review polling per source, async AI processing |
| Scraping | Playwright | Headless browser for Yelp and G2 review scraping |
| Google API | Google Business Profile API | OAuth 2.0, official Reviews endpoint |
| Email | Resend | Transactional alert emails, review digests |
| Slack | Slack Web API (`@slack/web-api`) | Incoming webhook for review alerts |
| UI | Tailwind CSS, Radix UI, Recharts | Accessible primitives, data visualization |
| Hosting | Vercel (app), Railway (scraping workers) | Playwright requires persistent runtime, not serverless |
| Monitoring | Sentry | Error tracking, performance monitoring |

### Key Architecture Decisions

1. **Platform adapter pattern for review sources**: Each review platform (Google, Yelp, G2) implements a common `ReviewSourceAdapter` interface with methods `fetchReviews()`, `getListingInfo()`, and `isConfigValid()`. New platforms are added by writing a single adapter file. The adapter abstraction isolates scraping/API logic from business logic, allowing Playwright-based scrapers (Yelp, G2) and API-based adapters (Google) to coexist behind the same interface. Adapters run in Railway workers, not Vercel functions, because Playwright requires persistent browser processes.

2. **AI sentiment analysis pipeline**: Every ingested review passes through a GPT-4o-mini analysis step that returns structured JSON: sentiment label (positive / neutral / negative / mixed), sentiment score (-1.0 to 1.0), up to 5 extracted topics (e.g., "food quality", "wait time", "customer service"), and a category (praise / complaint / suggestion / question). This runs asynchronously via BullMQ after the review is stored, so scraping latency is not compounded by AI latency.

3. **BullMQ scheduled polling per source**: Each `reviewSource` row has a `checkIntervalMinutes` value (default 60, minimum 15). A repeatable BullMQ job runs every minute, queries for sources whose `nextCheckAt <= now()`, and enqueues individual `poll-source` jobs. Each poll job runs the appropriate platform adapter, diffs against known `externalReviewId` values, and inserts only new reviews. This approach avoids cron-based fan-out and lets per-source intervals vary by plan tier.

4. **AES-256-GCM encryption for API credentials**: Google OAuth tokens and Slack webhook URLs are stored in the `reviewSources.credentials` JSONB column, encrypted with AES-256-GCM using a separate `CREDENTIALS_ENCRYPTION_KEY` environment variable. The encryption produces an `iv:authTag:ciphertext` string. Decryption happens only in the worker process at poll time, never in the browser or API response.

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
  plan: text("plan").default("free").notNull(), // free | pro | business
  stripeCustomerId: text("stripe_customer_id"),
  stripeSubscriptionId: text("stripe_subscription_id"),
  settings: jsonb("settings").default({}).$type<{
    defaultResponseTone?: "grateful" | "professional" | "empathetic" | "apologetic";
    slackWebhookUrl?: string;         // AES-256-GCM encrypted
    slackChannelName?: string;
    digestEnabled?: boolean;
    digestFrequency?: "daily" | "weekly";
    digestDay?: number;               // 0-6 for weekly
    digestHour?: number;              // 0-23
    onboardingCompleted?: boolean;
    brandName?: string;
    timezone?: string;
  }>(),
  aiResponsesUsedThisMonth: integer("ai_responses_used_this_month").default(0).notNull(),
  aiResponsesResetAt: timestamp("ai_responses_reset_at", { withTimezone: true }),
  createdAt: timestamp("created_at", { withTimezone: true }).defaultNow().notNull(),
  updatedAt: timestamp("updated_at", { withTimezone: true }).defaultNow().notNull(),
});

// ---------------------------------------------------------------------------
// Members (Clerk users linked to organizations)
// ---------------------------------------------------------------------------
export const members = pgTable(
  "members",
  {
    id: uuid("id").primaryKey().defaultRandom(),
    orgId: uuid("org_id")
      .references(() => organizations.id, { onDelete: "cascade" })
      .notNull(),
    clerkUserId: text("clerk_user_id").notNull(),
    role: text("role").default("member").notNull(), // owner | admin | member
    displayName: text("display_name"),
    email: text("email"),
    avatarUrl: text("avatar_url"),
    createdAt: timestamp("created_at", { withTimezone: true }).defaultNow().notNull(),
  },
  (table) => [
    uniqueIndex("members_org_user_idx").on(table.orgId, table.clerkUserId),
    index("members_clerk_user_id_idx").on(table.clerkUserId),
  ]
);

// ---------------------------------------------------------------------------
// Locations (business locations being monitored)
// ---------------------------------------------------------------------------
export const locations = pgTable(
  "locations",
  {
    id: uuid("id").primaryKey().defaultRandom(),
    orgId: uuid("org_id")
      .references(() => organizations.id, { onDelete: "cascade" })
      .notNull(),
    name: text("name").notNull(),           // "Downtown Restaurant", "Main Office"
    address: text("address"),
    city: text("city"),
    state: text("state"),
    country: text("country").default("US"),
    currentRating: real("current_rating"),   // Aggregate across all sources
    totalReviewCount: integer("total_review_count").default(0).notNull(),
    createdAt: timestamp("created_at", { withTimezone: true }).defaultNow().notNull(),
    updatedAt: timestamp("updated_at", { withTimezone: true }).defaultNow().notNull(),
  },
  (table) => [
    index("locations_org_id_idx").on(table.orgId),
  ]
);

// ---------------------------------------------------------------------------
// Review Sources (platform connections per location)
// ---------------------------------------------------------------------------
export const reviewSources = pgTable(
  "review_sources",
  {
    id: uuid("id").primaryKey().defaultRandom(),
    locationId: uuid("location_id")
      .references(() => locations.id, { onDelete: "cascade" })
      .notNull(),
    orgId: uuid("org_id")
      .references(() => organizations.id, { onDelete: "cascade" })
      .notNull(),
    platform: text("platform").notNull(), // google | yelp | g2
    name: text("name").notNull(),         // "Google — Downtown Restaurant"
    externalUrl: text("external_url"),     // Public listing URL
    externalId: text("external_id"),       // Google place_id, Yelp biz alias, G2 product slug
    credentials: jsonb("credentials").default({}).$type<{
      // Google OAuth
      googleAccessToken?: string;          // AES-256-GCM encrypted
      googleRefreshToken?: string;         // AES-256-GCM encrypted
      googleAccountId?: string;
      googleLocationId?: string;
      // Yelp scraping
      yelpBusinessAlias?: string;
      // G2 scraping
      g2ProductSlug?: string;
    }>(),
    status: text("status").default("active").notNull(), // active | paused | error
    errorMessage: text("error_message"),
    checkIntervalMinutes: integer("check_interval_minutes").default(60).notNull(),
    lastCheckedAt: timestamp("last_checked_at", { withTimezone: true }),
    nextCheckAt: timestamp("next_check_at", { withTimezone: true }),
    currentRating: real("current_rating"),
    totalReviewCount: integer("total_review_count").default(0).notNull(),
    createdAt: timestamp("created_at", { withTimezone: true }).defaultNow().notNull(),
    updatedAt: timestamp("updated_at", { withTimezone: true }).defaultNow().notNull(),
  },
  (table) => [
    index("review_sources_org_id_idx").on(table.orgId),
    index("review_sources_location_id_idx").on(table.locationId),
    index("review_sources_next_check_idx").on(table.nextCheckAt),
    uniqueIndex("review_sources_platform_external_idx").on(table.platform, table.externalId),
  ]
);

// ---------------------------------------------------------------------------
// Reviews
// ---------------------------------------------------------------------------
export const reviews = pgTable(
  "reviews",
  {
    id: uuid("id").primaryKey().defaultRandom(),
    orgId: uuid("org_id")
      .references(() => organizations.id, { onDelete: "cascade" })
      .notNull(),
    locationId: uuid("location_id")
      .references(() => locations.id, { onDelete: "cascade" })
      .notNull(),
    sourceId: uuid("source_id")
      .references(() => reviewSources.id, { onDelete: "cascade" })
      .notNull(),
    platform: text("platform").notNull(),            // google | yelp | g2
    externalReviewId: text("external_review_id"),     // Platform-specific review ID
    reviewerName: text("reviewer_name"),
    reviewerAvatarUrl: text("reviewer_avatar_url"),
    rating: integer("rating").notNull(),              // 1-5 normalized
    rawRating: text("raw_rating"),                    // Original rating string (e.g., "4/5", "8/10")
    text: text("text"),                               // Review body text
    language: text("language").default("en"),
    // AI-populated fields
    sentimentScore: real("sentiment_score"),           // -1.0 to 1.0
    sentimentLabel: text("sentiment_label"),           // positive | neutral | negative | mixed
    topics: jsonb("topics").default([]).$type<string[]>(),       // ["food quality", "service", "ambiance"]
    category: text("category"),                        // praise | complaint | suggestion | question
    aiProcessed: boolean("ai_processed").default(false).notNull(),
    // Status
    responded: boolean("responded").default(false).notNull(),
    publishedAt: timestamp("published_at", { withTimezone: true }),
    fetchedAt: timestamp("fetched_at", { withTimezone: true }).defaultNow().notNull(),
    createdAt: timestamp("created_at", { withTimezone: true }).defaultNow().notNull(),
  },
  (table) => [
    index("reviews_org_id_idx").on(table.orgId),
    index("reviews_location_id_idx").on(table.locationId),
    index("reviews_source_id_idx").on(table.sourceId),
    index("reviews_platform_idx").on(table.orgId, table.platform),
    index("reviews_rating_idx").on(table.orgId, table.rating),
    index("reviews_sentiment_idx").on(table.orgId, table.sentimentLabel),
    index("reviews_responded_idx").on(table.orgId, table.responded),
    index("reviews_published_at_idx").on(table.publishedAt),
    uniqueIndex("reviews_external_id_idx").on(table.sourceId, table.externalReviewId),
  ]
);

// ---------------------------------------------------------------------------
// Review Responses (AI-drafted + human-edited responses)
// ---------------------------------------------------------------------------
export const reviewResponses = pgTable(
  "review_responses",
  {
    id: uuid("id").primaryKey().defaultRandom(),
    reviewId: uuid("review_id")
      .references(() => reviews.id, { onDelete: "cascade" })
      .notNull(),
    orgId: uuid("org_id")
      .references(() => organizations.id, { onDelete: "cascade" })
      .notNull(),
    tone: text("tone").notNull(),                    // grateful | professional | empathetic | apologetic
    aiDraft: text("ai_draft").notNull(),             // Original AI-generated text
    finalText: text("final_text").notNull(),         // Human-edited version (or same as aiDraft)
    posted: boolean("posted").default(false).notNull(),
    postedAt: timestamp("posted_at", { withTimezone: true }),
    postedByUserId: text("posted_by_user_id"),       // Clerk userId
    createdAt: timestamp("created_at", { withTimezone: true }).defaultNow().notNull(),
  },
  (table) => [
    index("review_responses_review_id_idx").on(table.reviewId),
    index("review_responses_org_id_idx").on(table.orgId),
  ]
);

// ---------------------------------------------------------------------------
// Alert Rules
// ---------------------------------------------------------------------------
export const alertRules = pgTable(
  "alert_rules",
  {
    id: uuid("id").primaryKey().defaultRandom(),
    orgId: uuid("org_id")
      .references(() => organizations.id, { onDelete: "cascade" })
      .notNull(),
    name: text("name").notNull(),                    // "Negative Review Alert"
    enabled: boolean("enabled").default(true).notNull(),
    conditions: jsonb("conditions").default({}).$type<{
      minRating?: number;                            // Alert if rating <= this value
      maxRating?: number;                            // Alert if rating >= this value
      platforms?: string[];                          // Only for these platforms
      keywords?: string[];                           // Must contain one of these keywords
      sentimentLabels?: string[];                    // Only these sentiments
      locationIds?: string[];                        // Only these locations
    }>(),
    channels: jsonb("channels").default([]).$type<Array<{
      type: "email" | "slack";
      target: string;                                // Email address or "default" for Slack
    }>>(),
    createdAt: timestamp("created_at", { withTimezone: true }).defaultNow().notNull(),
    updatedAt: timestamp("updated_at", { withTimezone: true }).defaultNow().notNull(),
  },
  (table) => [
    index("alert_rules_org_id_idx").on(table.orgId),
  ]
);

// ---------------------------------------------------------------------------
// Alert Log (record of triggered alerts)
// ---------------------------------------------------------------------------
export const alertLog = pgTable(
  "alert_log",
  {
    id: uuid("id").primaryKey().defaultRandom(),
    orgId: uuid("org_id")
      .references(() => organizations.id, { onDelete: "cascade" })
      .notNull(),
    alertRuleId: uuid("alert_rule_id")
      .references(() => alertRules.id, { onDelete: "set null" }),
    reviewId: uuid("review_id")
      .references(() => reviews.id, { onDelete: "cascade" })
      .notNull(),
    channel: text("channel").notNull(),              // email | slack
    recipient: text("recipient").notNull(),          // Email address or Slack channel
    status: text("status").default("pending").notNull(), // pending | sent | failed
    sentAt: timestamp("sent_at", { withTimezone: true }),
    errorMessage: text("error_message"),
    createdAt: timestamp("created_at", { withTimezone: true }).defaultNow().notNull(),
  },
  (table) => [
    index("alert_log_org_id_idx").on(table.orgId),
    index("alert_log_review_id_idx").on(table.reviewId),
    index("alert_log_status_idx").on(table.status),
  ]
);

// ---------------------------------------------------------------------------
// Sentiment Snapshots (daily aggregates for charting)
// ---------------------------------------------------------------------------
export const sentimentSnapshots = pgTable(
  "sentiment_snapshots",
  {
    id: uuid("id").primaryKey().defaultRandom(),
    orgId: uuid("org_id")
      .references(() => organizations.id, { onDelete: "cascade" })
      .notNull(),
    locationId: uuid("location_id")
      .references(() => locations.id, { onDelete: "cascade" }),
    platform: text("platform"),                      // null = aggregate across all
    date: timestamp("date", { withTimezone: true }).notNull(),
    averageRating: real("average_rating"),
    reviewCount: integer("review_count").default(0).notNull(),
    averageSentiment: real("average_sentiment"),      // -1.0 to 1.0
    positiveCount: integer("positive_count").default(0).notNull(),
    neutralCount: integer("neutral_count").default(0).notNull(),
    negativeCount: integer("negative_count").default(0).notNull(),
    topTopics: jsonb("top_topics").default([]).$type<Array<{
      topic: string;
      count: number;
      avgSentiment: number;
    }>>(),
    createdAt: timestamp("created_at", { withTimezone: true }).defaultNow().notNull(),
  },
  (table) => [
    index("sentiment_snapshots_org_date_idx").on(table.orgId, table.date),
    index("sentiment_snapshots_location_date_idx").on(table.locationId, table.date),
    uniqueIndex("sentiment_snapshots_unique_idx").on(
      table.orgId, table.locationId, table.platform, table.date
    ),
  ]
);

// ---------------------------------------------------------------------------
// Notification Log (general notification tracking)
// ---------------------------------------------------------------------------
export const notificationLog = pgTable(
  "notification_log",
  {
    id: uuid("id").primaryKey().defaultRandom(),
    orgId: uuid("org_id")
      .references(() => organizations.id, { onDelete: "cascade" })
      .notNull(),
    type: text("type").notNull(),                    // new_review_alert | digest | response_posted
    channel: text("channel").notNull(),              // email | slack
    recipient: text("recipient").notNull(),
    subject: text("subject"),
    status: text("status").default("pending").notNull(), // pending | sent | failed
    metadata: jsonb("metadata").default({}).$type<{
      reviewId?: string;
      alertRuleId?: string;
      resendMessageId?: string;
      slackTs?: string;
      digestPeriod?: string;
    }>(),
    sentAt: timestamp("sent_at", { withTimezone: true }),
    errorMessage: text("error_message"),
    createdAt: timestamp("created_at", { withTimezone: true }).defaultNow().notNull(),
  },
  (table) => [
    index("notification_log_org_id_idx").on(table.orgId),
    index("notification_log_status_idx").on(table.status),
    index("notification_log_type_idx").on(table.orgId, table.type),
  ]
);

// ---------------------------------------------------------------------------
// Relations
// ---------------------------------------------------------------------------
export const organizationsRelations = relations(organizations, ({ many }) => ({
  members: many(members),
  locations: many(locations),
  reviewSources: many(reviewSources),
  reviews: many(reviews),
  reviewResponses: many(reviewResponses),
  alertRules: many(alertRules),
  alertLog: many(alertLog),
  sentimentSnapshots: many(sentimentSnapshots),
  notificationLog: many(notificationLog),
}));

export const membersRelations = relations(members, ({ one }) => ({
  organization: one(organizations, {
    fields: [members.orgId],
    references: [organizations.id],
  }),
}));

export const locationsRelations = relations(locations, ({ one, many }) => ({
  organization: one(organizations, {
    fields: [locations.orgId],
    references: [organizations.id],
  }),
  reviewSources: many(reviewSources),
  reviews: many(reviews),
  sentimentSnapshots: many(sentimentSnapshots),
}));

export const reviewSourcesRelations = relations(reviewSources, ({ one, many }) => ({
  organization: one(organizations, {
    fields: [reviewSources.orgId],
    references: [organizations.id],
  }),
  location: one(locations, {
    fields: [reviewSources.locationId],
    references: [locations.id],
  }),
  reviews: many(reviews),
}));

export const reviewsRelations = relations(reviews, ({ one, many }) => ({
  organization: one(organizations, {
    fields: [reviews.orgId],
    references: [organizations.id],
  }),
  location: one(locations, {
    fields: [reviews.locationId],
    references: [locations.id],
  }),
  source: one(reviewSources, {
    fields: [reviews.sourceId],
    references: [reviewSources.id],
  }),
  responses: many(reviewResponses),
  alertLog: many(alertLog),
}));

export const reviewResponsesRelations = relations(reviewResponses, ({ one }) => ({
  review: one(reviews, {
    fields: [reviewResponses.reviewId],
    references: [reviews.id],
  }),
  organization: one(organizations, {
    fields: [reviewResponses.orgId],
    references: [organizations.id],
  }),
}));

export const alertRulesRelations = relations(alertRules, ({ one, many }) => ({
  organization: one(organizations, {
    fields: [alertRules.orgId],
    references: [organizations.id],
  }),
  alertLog: many(alertLog),
}));

export const alertLogRelations = relations(alertLog, ({ one }) => ({
  organization: one(organizations, {
    fields: [alertLog.orgId],
    references: [organizations.id],
  }),
  alertRule: one(alertRules, {
    fields: [alertLog.alertRuleId],
    references: [alertRules.id],
  }),
  review: one(reviews, {
    fields: [alertLog.reviewId],
    references: [reviews.id],
  }),
}));

export const sentimentSnapshotsRelations = relations(sentimentSnapshots, ({ one }) => ({
  organization: one(organizations, {
    fields: [sentimentSnapshots.orgId],
    references: [organizations.id],
  }),
  location: one(locations, {
    fields: [sentimentSnapshots.locationId],
    references: [locations.id],
  }),
}));

export const notificationLogRelations = relations(notificationLog, ({ one }) => ({
  organization: one(organizations, {
    fields: [notificationLog.orgId],
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
export type Location = typeof locations.$inferSelect;
export type NewLocation = typeof locations.$inferInsert;
export type ReviewSource = typeof reviewSources.$inferSelect;
export type NewReviewSource = typeof reviewSources.$inferInsert;
export type Review = typeof reviews.$inferSelect;
export type NewReview = typeof reviews.$inferInsert;
export type ReviewResponse = typeof reviewResponses.$inferSelect;
export type NewReviewResponse = typeof reviewResponses.$inferInsert;
export type AlertRule = typeof alertRules.$inferSelect;
export type NewAlertRule = typeof alertRules.$inferInsert;
export type AlertLogEntry = typeof alertLog.$inferSelect;
export type SentimentSnapshot = typeof sentimentSnapshots.$inferSelect;
export type NotificationLogEntry = typeof notificationLog.$inferSelect;
```

### Initial Migration SQL

```sql
-- src/server/db/migrations/0000_init.sql

-- Enable required extensions
CREATE EXTENSION IF NOT EXISTS pg_trgm;

-- (All tables created by Drizzle push, then manually add text search indexes:)

-- Trigram index for fuzzy text search on review body
CREATE INDEX IF NOT EXISTS reviews_text_trgm_idx
  ON reviews USING gin (text gin_trgm_ops);

-- Trigram index for searching reviewer names
CREATE INDEX IF NOT EXISTS reviews_reviewer_name_trgm_idx
  ON reviews USING gin (reviewer_name gin_trgm_ops);

-- Partial index for unresponded reviews (common filter)
CREATE INDEX IF NOT EXISTS reviews_unresponded_idx
  ON reviews (org_id, published_at DESC)
  WHERE responded = false;

-- Composite index for feed queries with platform + rating filter
CREATE INDEX IF NOT EXISTS reviews_feed_composite_idx
  ON reviews (org_id, platform, rating, published_at DESC);
```

---

## Architecture Deep-Dives

### 1. Platform Adapter Pattern

```typescript
// src/server/adapters/types.ts

export interface FetchedReview {
  externalReviewId: string;
  reviewerName: string | null;
  reviewerAvatarUrl: string | null;
  rating: number;              // Normalized to 1-5
  rawRating: string;           // Original format
  text: string | null;
  language: string;
  publishedAt: Date;
}

export interface ListingInfo {
  name: string;
  address: string | null;
  currentRating: number | null;
  totalReviewCount: number | null;
  externalUrl: string;
}

export interface ReviewSourceAdapter {
  platform: "google" | "yelp" | "g2";

  /** Validate that the credentials/config are sufficient to fetch reviews */
  isConfigValid(credentials: Record<string, unknown>): boolean;

  /** Fetch reviews published after the given date. Returns newest first. */
  fetchReviews(
    credentials: Record<string, unknown>,
    since: Date | null,
    maxResults?: number
  ): Promise<FetchedReview[]>;

  /** Fetch current listing info (name, rating, review count) */
  getListingInfo(
    credentials: Record<string, unknown>
  ): Promise<ListingInfo>;
}

// src/server/adapters/google-adapter.ts

import type { ReviewSourceAdapter, FetchedReview, ListingInfo } from "./types";
import { decrypt } from "../integrations/encryption";

export class GoogleAdapter implements ReviewSourceAdapter {
  platform = "google" as const;

  isConfigValid(credentials: Record<string, unknown>): boolean {
    return !!(
      credentials.googleAccessToken &&
      credentials.googleAccountId &&
      credentials.googleLocationId
    );
  }

  async fetchReviews(
    credentials: Record<string, unknown>,
    since: Date | null,
    maxResults = 50
  ): Promise<FetchedReview[]> {
    const accessToken = decrypt(credentials.googleAccessToken as string);
    const accountId = credentials.googleAccountId as string;
    const locationId = credentials.googleLocationId as string;

    const url = `https://mybusiness.googleapis.com/v4/accounts/${accountId}/locations/${locationId}/reviews?pageSize=${maxResults}&orderBy=updateTime desc`;

    const res = await fetch(url, {
      headers: { Authorization: `Bearer ${accessToken}` },
    });

    if (!res.ok) {
      const error = await res.text();
      throw new Error(`Google API error ${res.status}: ${error}`);
    }

    const data = await res.json();
    const reviews: FetchedReview[] = [];

    for (const r of data.reviews ?? []) {
      const publishedAt = new Date(r.updateTime ?? r.createTime);
      if (since && publishedAt <= since) break;

      reviews.push({
        externalReviewId: r.reviewId ?? r.name,
        reviewerName: r.reviewer?.displayName ?? null,
        reviewerAvatarUrl: r.reviewer?.profilePhotoUrl ?? null,
        rating: this.normalizeGoogleRating(r.starRating),
        rawRating: r.starRating,
        text: r.comment ?? null,
        language: r.comment ? "en" : "en",
        publishedAt,
      });
    }

    return reviews;
  }

  async getListingInfo(credentials: Record<string, unknown>): Promise<ListingInfo> {
    const accessToken = decrypt(credentials.googleAccessToken as string);
    const accountId = credentials.googleAccountId as string;
    const locationId = credentials.googleLocationId as string;

    const url = `https://mybusiness.googleapis.com/v4/accounts/${accountId}/locations/${locationId}`;
    const res = await fetch(url, {
      headers: { Authorization: `Bearer ${accessToken}` },
    });
    const data = await res.json();

    return {
      name: data.locationName ?? data.name ?? "Unknown",
      address: data.address?.addressLines?.join(", ") ?? null,
      currentRating: data.metadata?.averageRating ?? null,
      totalReviewCount: data.metadata?.reviewCount ?? null,
      externalUrl: data.metadata?.mapsUrl ?? `https://maps.google.com/?cid=${locationId}`,
    };
  }

  private normalizeGoogleRating(starRating: string): number {
    const map: Record<string, number> = {
      ONE: 1, TWO: 2, THREE: 3, FOUR: 4, FIVE: 5,
    };
    return map[starRating] ?? 3;
  }
}

// src/server/adapters/yelp-adapter.ts

import { chromium, type Browser, type Page } from "playwright";
import type { ReviewSourceAdapter, FetchedReview, ListingInfo } from "./types";

export class YelpAdapter implements ReviewSourceAdapter {
  platform = "yelp" as const;
  private browser: Browser | null = null;

  isConfigValid(credentials: Record<string, unknown>): boolean {
    return !!credentials.yelpBusinessAlias;
  }

  async fetchReviews(
    credentials: Record<string, unknown>,
    since: Date | null,
    maxResults = 30
  ): Promise<FetchedReview[]> {
    const alias = credentials.yelpBusinessAlias as string;
    const page = await this.getPage();

    try {
      await page.goto(`https://www.yelp.com/biz/${alias}?sort_by=date_desc`, {
        waitUntil: "domcontentloaded",
        timeout: 30000,
      });

      await page.waitForSelector('[data-testid="review-card"]', { timeout: 15000 });

      const reviews = await page.$$eval(
        '[data-testid="review-card"]',
        (cards, maxRes) => {
          return cards.slice(0, maxRes).map((card) => {
            const nameEl = card.querySelector('[data-testid="user-passport"] a');
            const avatarEl = card.querySelector('[data-testid="user-passport"] img');
            const ratingEl = card.querySelector('[aria-label*="star rating"]');
            const textEl = card.querySelector('[data-testid="review-text"] span');
            const dateEl = card.querySelector('[data-testid="review-date"] span');

            const ratingLabel = ratingEl?.getAttribute("aria-label") ?? "3 star rating";
            const ratingNum = parseInt(ratingLabel.match(/(\d)/)?.[1] ?? "3", 10);

            return {
              reviewerName: nameEl?.textContent?.trim() ?? null,
              reviewerAvatarUrl: avatarEl?.getAttribute("src") ?? null,
              rating: ratingNum,
              rawRating: `${ratingNum}/5`,
              text: textEl?.textContent?.trim() ?? null,
              dateText: dateEl?.textContent?.trim() ?? null,
            };
          });
        },
        maxResults
      );

      const fetchedReviews: FetchedReview[] = [];
      for (const r of reviews) {
        const publishedAt = this.parseYelpDate(r.dateText);
        if (since && publishedAt <= since) break;

        fetchedReviews.push({
          externalReviewId: `yelp_${alias}_${publishedAt.getTime()}_${r.reviewerName ?? "anon"}`,
          reviewerName: r.reviewerName,
          reviewerAvatarUrl: r.reviewerAvatarUrl,
          rating: r.rating,
          rawRating: r.rawRating,
          text: r.text,
          language: "en",
          publishedAt,
        });
      }

      return fetchedReviews;
    } finally {
      await page.close();
    }
  }

  async getListingInfo(credentials: Record<string, unknown>): Promise<ListingInfo> {
    const alias = credentials.yelpBusinessAlias as string;
    const page = await this.getPage();

    try {
      await page.goto(`https://www.yelp.com/biz/${alias}`, {
        waitUntil: "domcontentloaded",
        timeout: 30000,
      });

      const info = await page.evaluate(() => {
        const nameEl = document.querySelector("h1");
        const ratingEl = document.querySelector('[aria-label*="star rating"]');
        const countEl = document.querySelector('a[href*="reviews"]');
        const addressEl = document.querySelector('[data-testid="biz-address"] p');

        const ratingLabel = ratingEl?.getAttribute("aria-label") ?? "";
        const ratingMatch = ratingLabel.match(/([\d.]+)/);
        const countMatch = countEl?.textContent?.match(/([\d,]+)/);

        return {
          name: nameEl?.textContent?.trim() ?? "Unknown",
          address: addressEl?.textContent?.trim() ?? null,
          currentRating: ratingMatch ? parseFloat(ratingMatch[1]) : null,
          totalReviewCount: countMatch ? parseInt(countMatch[1].replace(",", ""), 10) : null,
        };
      });

      return {
        ...info,
        externalUrl: `https://www.yelp.com/biz/${alias}`,
      };
    } finally {
      await page.close();
    }
  }

  private parseYelpDate(dateText: string | null): Date {
    if (!dateText) return new Date();
    // Yelp formats: "1/15/2025", "Jan 15, 2025"
    const parsed = new Date(dateText);
    return isNaN(parsed.getTime()) ? new Date() : parsed;
  }

  private async getPage(): Promise<Page> {
    if (!this.browser) {
      this.browser = await chromium.launch({
        headless: true,
        args: ["--no-sandbox", "--disable-setuid-sandbox"],
      });
    }
    const context = await this.browser.newContext({
      userAgent:
        "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/120.0.0.0 Safari/537.36",
    });
    return context.newPage();
  }
}
```

### 2. AI Sentiment Analysis + Topic Extraction Pipeline

```typescript
// src/server/ai/analyze-review.ts

import OpenAI from "openai";

const openai = new OpenAI({ apiKey: process.env.OPENAI_API_KEY });

export interface ReviewAnalysis {
  sentimentLabel: "positive" | "neutral" | "negative" | "mixed";
  sentimentScore: number;          // -1.0 to 1.0
  topics: string[];                // Up to 5 extracted topics
  category: "praise" | "complaint" | "suggestion" | "question";
  confidence: number;              // 0-1
  summary: string;                 // One-sentence summary of the review
}

const ANALYSIS_PROMPT = `You are a review analysis system for a business reputation monitoring platform. Analyze the following customer review and return structured JSON.

CONTEXT:
- Platform: {platform}
- Rating: {rating}/5 stars
- Business name: {business_name}

RULES:
- "sentiment_label" must be exactly one of: "positive", "neutral", "negative", "mixed"
- "sentiment_score" is a float from -1.0 (extremely negative) to 1.0 (extremely positive). A 5-star review with glowing text should be 0.8-1.0. A 1-star review with complaints should be -0.8 to -1.0. Mixed reviews (e.g., "food was great but service was terrible") should be near 0.0.
- "topics" is an array of 1-5 specific topics mentioned in the review. Use short, lowercase phrases (e.g., "food quality", "wait time", "customer service", "price value", "cleanliness", "product features", "user interface", "onboarding", "billing support"). Be specific to what the reviewer actually discussed.
- "category" must be exactly one of: "praise" (positive feedback), "complaint" (negative feedback), "suggestion" (constructive feedback with an improvement idea), "question" (asking something)
- "confidence" is how confident you are in the analysis, from 0.0 to 1.0
- "summary" is a single sentence summarizing the key point of the review

REVIEW TEXT:
---
{review_text}
---

Return ONLY valid JSON:
{"sentiment_label": "...", "sentiment_score": 0.0, "topics": [...], "category": "...", "confidence": 0.0, "summary": "..."}`;

export async function analyzeReview(opts: {
  reviewText: string;
  rating: number;
  platform: string;
  businessName: string;
}): Promise<ReviewAnalysis> {
  if (!opts.reviewText || opts.reviewText.trim().length === 0) {
    return inferFromRatingOnly(opts.rating);
  }

  const truncated = opts.reviewText.slice(0, 3000);
  const prompt = ANALYSIS_PROMPT
    .replace("{platform}", opts.platform)
    .replace("{rating}", String(opts.rating))
    .replace("{business_name}", opts.businessName)
    .replace("{review_text}", truncated);

  const response = await openai.chat.completions.create({
    model: "gpt-4o-mini",
    messages: [{ role: "user", content: prompt }],
    temperature: 0.1,
    max_tokens: 400,
    response_format: { type: "json_object" },
  });

  const content = response.choices[0]?.message?.content ?? "{}";
  let parsed: Record<string, unknown>;
  try {
    parsed = JSON.parse(content);
  } catch {
    return inferFromRatingOnly(opts.rating);
  }

  const validSentiments = ["positive", "neutral", "negative", "mixed"];
  const validCategories = ["praise", "complaint", "suggestion", "question"];

  return {
    sentimentLabel: validSentiments.includes(parsed.sentiment_label as string)
      ? (parsed.sentiment_label as ReviewAnalysis["sentimentLabel"])
      : inferSentimentFromRating(opts.rating),
    sentimentScore: typeof parsed.sentiment_score === "number"
      ? Math.max(-1, Math.min(1, parsed.sentiment_score))
      : (opts.rating - 3) / 2,
    topics: Array.isArray(parsed.topics)
      ? parsed.topics
          .filter((t: unknown) => typeof t === "string")
          .map((t: string) => t.toLowerCase().trim())
          .slice(0, 5)
      : [],
    category: validCategories.includes(parsed.category as string)
      ? (parsed.category as ReviewAnalysis["category"])
      : opts.rating >= 4 ? "praise" : "complaint",
    confidence: typeof parsed.confidence === "number"
      ? Math.max(0, Math.min(1, parsed.confidence))
      : 0.7,
    summary: typeof parsed.summary === "string"
      ? parsed.summary.slice(0, 200)
      : `${opts.rating}-star review on ${opts.platform}`,
  };
}

function inferFromRatingOnly(rating: number): ReviewAnalysis {
  return {
    sentimentLabel: inferSentimentFromRating(rating),
    sentimentScore: (rating - 3) / 2,
    topics: [],
    category: rating >= 4 ? "praise" : "complaint",
    confidence: 0.3,
    summary: `Rating-only review (${rating}/5 stars)`,
  };
}

function inferSentimentFromRating(rating: number): ReviewAnalysis["sentimentLabel"] {
  if (rating >= 4) return "positive";
  if (rating === 3) return "neutral";
  return "negative";
}
```

### 3. AI Response Drafting Engine

```typescript
// src/server/ai/draft-response.ts

import OpenAI from "openai";

const openai = new OpenAI({ apiKey: process.env.OPENAI_API_KEY });

export type ResponseTone = "grateful" | "professional" | "empathetic" | "apologetic";

export interface DraftResponseOpts {
  reviewText: string;
  rating: number;
  reviewerName: string | null;
  platform: string;
  businessName: string;
  tone: ResponseTone;
  topics: string[];
  sentimentLabel: string;
  existingResponses?: string[];    // Previous drafts for this review (to avoid repetition)
}

export interface DraftResponseResult {
  responseText: string;
  tokensUsed: number;
}

const TONE_INSTRUCTIONS: Record<ResponseTone, string> = {
  grateful: `Tone: GRATEFUL. Express sincere appreciation for their feedback and time. Highlight specific things they praised. Be warm, personal, and enthusiastic without being over-the-top. End with an invitation to return.`,
  professional: `Tone: PROFESSIONAL. Maintain a courteous, business-appropriate tone. Acknowledge their feedback objectively. Address any concerns with clear, actionable information. Be polite but not overly casual. Sign off formally.`,
  empathetic: `Tone: EMPATHETIC. Show genuine understanding of their experience. Validate their feelings before addressing any issues. Use phrases like "I understand how frustrating that must have been" or "We hear you." Offer concrete next steps to make things right.`,
  apologetic: `Tone: APOLOGETIC. Lead with a sincere, specific apology — not generic "sorry for the inconvenience." Acknowledge exactly what went wrong based on their review. Take responsibility without making excuses. Offer a clear resolution or invite them to reach out directly so you can fix the situation.`,
};

const RESPONSE_PROMPT = `You are an AI assistant helping a business owner respond to a customer review. Write a response that the business owner can post publicly on {platform}.

BUSINESS: {business_name}
PLATFORM: {platform}
REVIEWER NAME: {reviewer_name}
RATING: {rating}/5 stars
DETECTED TOPICS: {topics}
SENTIMENT: {sentiment}

{tone_instruction}

RULES:
1. Address the reviewer by their first name if available, otherwise use a warm greeting.
2. Reference SPECIFIC details from their review — never write a generic response.
3. Keep the response between 50-150 words. Concise but personal.
4. Do NOT include subject lines, signatures, or the business name at the end.
5. Do NOT use exclamation marks excessively (maximum 2).
6. For negative reviews: acknowledge the issue, apologize where appropriate, and invite them to reach out directly (provide a general contact method, not a specific email/phone).
7. For positive reviews: express genuine gratitude and highlight what they enjoyed.
8. Never use phrases like "as an AI" or "I'm a language model."
9. Write as if you ARE the business owner or manager.
10. Match the formality level to the platform (Yelp/Google = casual-professional, G2 = professional).
{avoid_repetition}

REVIEW TEXT:
---
{review_text}
---

Write the response text only. No JSON wrapper. No quotation marks around the entire response.`;

export async function draftResponse(opts: DraftResponseOpts): Promise<DraftResponseResult> {
  const reviewerFirstName = opts.reviewerName?.split(" ")[0] ?? null;
  const toneInstruction = TONE_INSTRUCTIONS[opts.tone];

  let avoidRepetition = "";
  if (opts.existingResponses && opts.existingResponses.length > 0) {
    avoidRepetition = `\n11. IMPORTANT: The business owner already rejected these drafts, so write something meaningfully different:\n${opts.existingResponses.map((r, i) => `  Draft ${i + 1}: "${r.slice(0, 100)}..."`).join("\n")}`;
  }

  const prompt = RESPONSE_PROMPT
    .replace("{platform}", opts.platform)
    .replace("{platform}", opts.platform)
    .replace("{business_name}", opts.businessName)
    .replace("{reviewer_name}", reviewerFirstName ?? "Valued Customer")
    .replace("{rating}", String(opts.rating))
    .replace("{topics}", opts.topics.length > 0 ? opts.topics.join(", ") : "general feedback")
    .replace("{sentiment}", opts.sentimentLabel)
    .replace("{tone_instruction}", toneInstruction)
    .replace("{avoid_repetition}", avoidRepetition)
    .replace("{review_text}", opts.reviewText.slice(0, 2000));

  const response = await openai.chat.completions.create({
    model: "gpt-4o-mini",
    messages: [{ role: "user", content: prompt }],
    temperature: 0.7,
    max_tokens: 300,
  });

  const responseText = response.choices[0]?.message?.content?.trim() ?? "";
  const tokensUsed = response.usage?.total_tokens ?? 0;

  // Clean up: remove wrapping quotes if present
  const cleaned = responseText
    .replace(/^["']|["']$/g, "")
    .replace(/^Response:\s*/i, "")
    .trim();

  return {
    responseText: cleaned,
    tokensUsed,
  };
}

export async function draftResponseWithUsageCheck(
  orgId: string,
  orgPlan: string,
  aiResponsesUsed: number,
  opts: DraftResponseOpts
): Promise<DraftResponseResult> {
  const limits: Record<string, number> = {
    free: 0,
    pro: 50,
    business: Infinity,
  };

  const limit = limits[orgPlan] ?? 0;
  if (aiResponsesUsed >= limit) {
    throw new Error(
      limit === 0
        ? "AI response drafting is not available on the Free plan. Please upgrade to Pro."
        : `You've used all ${limit} AI responses this month. Please upgrade to Business for unlimited responses.`
    );
  }

  const result = await draftResponse(opts);
  return result;
}
```

### 4. BullMQ Scheduled Review Polling Worker

```typescript
// src/server/queue/review-poll-worker.ts

import { Queue, Worker, Job } from "bullmq";
import { redis } from "./connection";
import { db } from "../db";
import { reviewSources, reviews, locations, organizations } from "../db/schema";
import { eq, and, lte, sql } from "drizzle-orm";
import { getAdapter } from "../adapters";
import { analyzeReview } from "../ai/analyze-review";
import { evaluateAlertRules } from "../services/alert-evaluator";
import type { FetchedReview } from "../adapters/types";

// ---------------------------------------------------------------------------
// Queue definitions
// ---------------------------------------------------------------------------
export interface PollSourceJobData {
  sourceId: string;
  orgId: string;
}

export interface AnalyzeReviewJobData {
  reviewId: string;
  orgId: string;
}

export const pollSchedulerQueue = new Queue("poll-scheduler", {
  connection: redis,
  defaultJobOptions: {
    removeOnComplete: { age: 3600 },
    removeOnFail: { age: 86400 },
  },
});

export const pollSourceQueue = new Queue<PollSourceJobData>("poll-source", {
  connection: redis,
  defaultJobOptions: {
    attempts: 3,
    backoff: { type: "exponential", delay: 5000 },
    removeOnComplete: { age: 3600 },
    removeOnFail: { age: 86400 },
  },
});

export const analyzeReviewQueue = new Queue<AnalyzeReviewJobData>("analyze-review", {
  connection: redis,
  defaultJobOptions: {
    attempts: 3,
    backoff: { type: "exponential", delay: 2000 },
    removeOnComplete: { age: 86400 },
    removeOnFail: { age: 604800 },
  },
});

// ---------------------------------------------------------------------------
// Scheduler: runs every minute, enqueues sources due for polling
// ---------------------------------------------------------------------------
export async function startScheduler(): Promise<void> {
  // Add a repeatable job that fires every 60 seconds
  await pollSchedulerQueue.add(
    "check-due-sources",
    {},
    { repeat: { every: 60_000 }, jobId: "scheduler-tick" }
  );
}

const schedulerWorker = new Worker(
  "poll-scheduler",
  async () => {
    const now = new Date();

    // Find all active sources where nextCheckAt has passed
    const dueSources = await db
      .select({
        id: reviewSources.id,
        orgId: reviewSources.orgId,
      })
      .from(reviewSources)
      .where(
        and(
          eq(reviewSources.status, "active"),
          lte(reviewSources.nextCheckAt, now)
        )
      )
      .limit(100);

    // Enqueue each source for polling, deduplicated by sourceId
    for (const source of dueSources) {
      await pollSourceQueue.add(
        `poll-${source.id}`,
        { sourceId: source.id, orgId: source.orgId },
        { jobId: `poll-${source.id}-${Date.now()}` }
      );
    }

    console.log(`[scheduler] Enqueued ${dueSources.length} sources for polling`);
  },
  { connection: redis, concurrency: 1 }
);

// ---------------------------------------------------------------------------
// Poll Worker: fetches reviews from a single source via its adapter
// ---------------------------------------------------------------------------
const pollSourceWorker = new Worker<PollSourceJobData>(
  "poll-source",
  async (job: Job<PollSourceJobData>) => {
    const { sourceId, orgId } = job.data;

    // Load the source with its location and org
    const [source] = await db
      .select()
      .from(reviewSources)
      .where(eq(reviewSources.id, sourceId))
      .limit(1);

    if (!source || source.status !== "active") {
      console.log(`[poll] Source ${sourceId} not active, skipping`);
      return;
    }

    const [location] = await db
      .select()
      .from(locations)
      .where(eq(locations.id, source.locationId))
      .limit(1);

    const adapter = getAdapter(source.platform);
    if (!adapter.isConfigValid(source.credentials ?? {})) {
      await db.update(reviewSources).set({
        status: "error",
        errorMessage: "Invalid or missing credentials",
        updatedAt: new Date(),
      }).where(eq(reviewSources.id, sourceId));
      return;
    }

    let fetchedReviews: FetchedReview[];
    try {
      fetchedReviews = await adapter.fetchReviews(
        source.credentials ?? {},
        source.lastCheckedAt
      );
    } catch (error) {
      const message = error instanceof Error ? error.message : "Unknown fetch error";
      await db.update(reviewSources).set({
        status: "error",
        errorMessage: message,
        updatedAt: new Date(),
      }).where(eq(reviewSources.id, sourceId));
      throw error; // Let BullMQ retry
    }

    // Diff against existing reviews to find truly new ones
    let newCount = 0;
    for (const fetched of fetchedReviews) {
      const existing = await db
        .select({ id: reviews.id })
        .from(reviews)
        .where(
          and(
            eq(reviews.sourceId, sourceId),
            eq(reviews.externalReviewId, fetched.externalReviewId)
          )
        )
        .limit(1);

      if (existing.length > 0) continue;

      // Insert new review
      const [newReview] = await db.insert(reviews).values({
        orgId,
        locationId: source.locationId,
        sourceId,
        platform: source.platform,
        externalReviewId: fetched.externalReviewId,
        reviewerName: fetched.reviewerName,
        reviewerAvatarUrl: fetched.reviewerAvatarUrl,
        rating: fetched.rating,
        rawRating: fetched.rawRating,
        text: fetched.text,
        language: fetched.language,
        publishedAt: fetched.publishedAt,
        fetchedAt: new Date(),
        aiProcessed: false,
        responded: false,
      }).returning();

      newCount++;

      // Enqueue AI analysis
      await analyzeReviewQueue.add(
        `analyze-${newReview.id}`,
        { reviewId: newReview.id, orgId }
      );
    }

    // Update source metadata
    const nextCheck = new Date(Date.now() + source.checkIntervalMinutes * 60_000);
    await db.update(reviewSources).set({
      lastCheckedAt: new Date(),
      nextCheckAt: nextCheck,
      status: "active",
      errorMessage: null,
      updatedAt: new Date(),
    }).where(eq(reviewSources.id, sourceId));

    // Update listing info periodically
    try {
      const listingInfo = await adapter.getListingInfo(source.credentials ?? {});
      await db.update(reviewSources).set({
        currentRating: listingInfo.currentRating,
        totalReviewCount: listingInfo.totalReviewCount,
      }).where(eq(reviewSources.id, sourceId));
    } catch {
      // Non-critical, don't fail the job
    }

    console.log(`[poll] Source ${sourceId} (${source.platform}): ${newCount} new reviews`);
  },
  { connection: redis, concurrency: 5 }
);

// ---------------------------------------------------------------------------
// Analyze Worker: runs AI sentiment + triggers alerts
// ---------------------------------------------------------------------------
const analyzeReviewWorker = new Worker<AnalyzeReviewJobData>(
  "analyze-review",
  async (job: Job<AnalyzeReviewJobData>) => {
    const { reviewId, orgId } = job.data;

    const [review] = await db
      .select()
      .from(reviews)
      .where(eq(reviews.id, reviewId))
      .limit(1);

    if (!review || review.aiProcessed) return;

    const [location] = await db
      .select()
      .from(locations)
      .where(eq(locations.id, review.locationId))
      .limit(1);

    // Run AI analysis
    const analysis = await analyzeReview({
      reviewText: review.text ?? "",
      rating: review.rating,
      platform: review.platform,
      businessName: location?.name ?? "Business",
    });

    // Store results
    await db.update(reviews).set({
      sentimentScore: analysis.sentimentScore,
      sentimentLabel: analysis.sentimentLabel,
      topics: analysis.topics,
      category: analysis.category,
      aiProcessed: true,
    }).where(eq(reviews.id, reviewId));

    // Evaluate alert rules
    await evaluateAlertRules(orgId, {
      reviewId: review.id,
      rating: review.rating,
      platform: review.platform,
      sentimentLabel: analysis.sentimentLabel,
      topics: analysis.topics,
      text: review.text ?? "",
      locationId: review.locationId,
    });
  },
  { connection: redis, concurrency: 10 }
);

// Export for graceful shutdown
export function stopWorkers(): Promise<void[]> {
  return Promise.all([
    schedulerWorker.close(),
    pollSourceWorker.close(),
    analyzeReviewWorker.close(),
  ]);
}
```

---

## Phase Breakdown (7 phases, 36 days)

### Phase 1: Scaffold + Database (Days 1-5)

**Day 1 — Project scaffold + environment**
- `npx create-next-app@latest reviewradar --typescript --tailwind --app --src-dir`
- Install dependencies: `drizzle-orm @neondatabase/serverless @clerk/nextjs @trpc/server @trpc/client @trpc/next superjson zod openai stripe resend @slack/web-api bullmq ioredis`
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
    GOOGLE_CLIENT_ID: z.string().min(1),
    GOOGLE_CLIENT_SECRET: z.string().min(1),
    CREDENTIALS_ENCRYPTION_KEY: z.string().length(64), // 32-byte hex
    NEXT_PUBLIC_APP_URL: z.string().url(),
  });
  ```
- Create Neon database on Neon console
- Configure `drizzle.config.ts` with Neon connection string

**Day 2 — Database schema + Clerk auth**
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
- Run migration SQL for trigram indexes
- Set up Clerk: `src/middleware.ts` with `clerkMiddleware()`, protect `/dashboard/*` routes
- Create `src/app/layout.tsx` with `<ClerkProvider>`
- Test: verify Clerk sign-in works, org creation works

**Day 3 — tRPC foundation + org resolution**
- Create `src/server/trpc/trpc.ts`: init tRPC with superjson, auth middleware, org middleware
- Create `src/server/trpc/context.ts`: pull `auth()` from Clerk, inject `db`
- Create `src/app/api/trpc/[trpc]/route.ts`: Next.js route handler
- Create `src/lib/trpc/client.ts`: React Query client with superjson
- Create `src/lib/trpc/server.ts`: server-side caller
- Build plan-limit middleware:
  ```typescript
  const PLAN_LIMITS = {
    free: { maxLocations: 1, maxSourcesPerLocation: 2, maxAiResponses: 0, maxMembers: 1, historyDays: 30 },
    pro: { maxLocations: 1, maxSourcesPerLocation: 5, maxAiResponses: 50, maxMembers: 1, historyDays: 365 },
    business: { maxLocations: 5, maxSourcesPerLocation: Infinity, maxAiResponses: Infinity, maxMembers: 5, historyDays: Infinity },
  } as const;
  ```
- Create `src/server/trpc/routers/_app.ts`: merge routers

**Day 4 — Organization + Location + ReviewSource tRPC routers**
- `src/server/trpc/routers/organization.ts`:
  - `getOrCreate`: find by clerkOrgId or create new
  - `update`: name, settings (defaultResponseTone, timezone, brandName)
  - `getSettings`: return current org settings
- `src/server/trpc/routers/location.ts`:
  - `list`: all locations for org
  - `create`: name + address, enforce plan location limit
  - `update`: name, address
  - `delete`: cascade removes sources and reviews
  - `getById`: single location with source summary stats
- `src/server/trpc/routers/reviewSource.ts`:
  - `list`: all sources for a location
  - `create`: platform + credentials + externalUrl
  - `update`: credentials, checkIntervalMinutes, status
  - `delete`: cascade cleanup
  - `testConnection`: run adapter's `getListingInfo()` to validate config
- Create `src/server/integrations/encryption.ts`:
  ```typescript
  import crypto from "crypto";

  const ALGORITHM = "aes-256-gcm";
  const IV_LENGTH = 16;
  const AUTH_TAG_LENGTH = 16;

  export function encrypt(plaintext: string): string {
    const key = Buffer.from(process.env.CREDENTIALS_ENCRYPTION_KEY!, "hex");
    const iv = crypto.randomBytes(IV_LENGTH);
    const cipher = crypto.createCipheriv(ALGORITHM, key, iv);
    let encrypted = cipher.update(plaintext, "utf8", "hex");
    encrypted += cipher.final("hex");
    const authTag = cipher.getAuthTag();
    return iv.toString("hex") + ":" + authTag.toString("hex") + ":" + encrypted;
  }

  export function decrypt(ciphertext: string): string {
    const key = Buffer.from(process.env.CREDENTIALS_ENCRYPTION_KEY!, "hex");
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
- Install UI deps: `@radix-ui/react-dialog @radix-ui/react-dropdown-menu @radix-ui/react-tabs @radix-ui/react-select @radix-ui/react-tooltip @radix-ui/react-popover @radix-ui/react-switch class-variance-authority clsx tailwind-merge lucide-react recharts`
- Create shared UI components in `src/components/ui/`: button, card, badge, input, textarea, select, dialog, dropdown-menu, tabs, toast, skeleton, switch, tooltip
- Build app shell layout: `src/app/dashboard/layout.tsx`
  - Left sidebar: ReviewRadar logo, nav links (Reviews, Analytics, Alerts, Settings), location switcher, user menu
  - Main content area with header breadcrumb
- Create stub pages:
  - `src/app/dashboard/page.tsx` (overview)
  - `src/app/dashboard/reviews/page.tsx`
  - `src/app/dashboard/analytics/page.tsx`
  - `src/app/dashboard/alerts/page.tsx`
  - `src/app/dashboard/settings/page.tsx`
  - `src/app/dashboard/settings/sources/page.tsx`
  - `src/app/dashboard/settings/billing/page.tsx`
  - `src/app/dashboard/settings/locations/page.tsx`

---

### Phase 2: Review Source Adapters (Days 6-12)

**Day 6 — Platform adapter interface + Google OAuth flow**
- Create `src/server/adapters/types.ts`: `ReviewSourceAdapter` interface (as shown in Deep-Dive)
- Create `src/server/adapters/index.ts`: adapter registry
  ```typescript
  import { GoogleAdapter } from "./google-adapter";
  import { YelpAdapter } from "./yelp-adapter";
  import { G2Adapter } from "./g2-adapter";
  import type { ReviewSourceAdapter } from "./types";

  const adapters: Record<string, ReviewSourceAdapter> = {
    google: new GoogleAdapter(),
    yelp: new YelpAdapter(),
    g2: new G2Adapter(),
  };

  export function getAdapter(platform: string): ReviewSourceAdapter {
    const adapter = adapters[platform];
    if (!adapter) throw new Error(`No adapter for platform: ${platform}`);
    return adapter;
  }
  ```
- Create `src/app/api/auth/google/route.ts`: redirect to Google OAuth with Business Profile scope
  ```typescript
  export async function GET(req: NextRequest) {
    const orgId = req.nextUrl.searchParams.get("orgId");
    const locationId = req.nextUrl.searchParams.get("locationId");
    const state = crypto.randomBytes(16).toString("hex");
    await redis.set(`google_oauth:${state}`, JSON.stringify({ orgId, locationId }), { ex: 600 });

    const url = new URL("https://accounts.google.com/o/oauth2/v2/auth");
    url.searchParams.set("client_id", process.env.GOOGLE_CLIENT_ID!);
    url.searchParams.set("redirect_uri", `${process.env.NEXT_PUBLIC_APP_URL}/api/auth/google/callback`);
    url.searchParams.set("response_type", "code");
    url.searchParams.set("scope", "https://www.googleapis.com/auth/business.manage");
    url.searchParams.set("access_type", "offline");
    url.searchParams.set("state", state);

    return NextResponse.redirect(url.toString());
  }
  ```
- Create `src/app/api/auth/google/callback/route.ts`: exchange code for tokens, store encrypted in `review_sources`

**Day 7 — Google adapter implementation**
- Create `src/server/adapters/google-adapter.ts` (as shown in Deep-Dive)
- Implement token refresh logic: if API returns 401, use `googleRefreshToken` to get a new access token, re-encrypt and update the source row
- Test with Google Business Profile API sandbox
- Handle pagination: Google reviews API returns `nextPageToken`, iterate until done or `since` date reached

**Day 8 — Yelp adapter (Playwright scraper)**
- Create `src/server/adapters/yelp-adapter.ts` (as shown in Deep-Dive)
- Add robust error handling for Yelp's anti-bot measures:
  - Random delay between page loads (2-5 seconds)
  - User-Agent rotation
  - Handle CAPTCHA pages gracefully (set source to "error" status with message)
- Test with 3-4 real Yelp business pages
- Handle Yelp date formats: "1/15/2025", "Jan 15, 2025", relative dates ("3 days ago")

**Day 9 — G2 adapter (Playwright scraper)**
- Create `src/server/adapters/g2-adapter.ts`:
  - Navigate to `https://www.g2.com/products/{slug}/reviews`
  - Parse review cards: rating (stars out of 5), title, review text, reviewer name, date
  - G2 uses 10-star scale internally but displays as 5 stars — normalize correctly
  - Handle G2's "What do you like/dislike" structured format: concatenate into single text
- Test with 3-4 G2 product pages
- Add rate limiting: minimum 5-second gap between G2 requests

**Day 10 — BullMQ queue setup + poll scheduler**
- Create `src/server/queue/connection.ts`:
  ```typescript
  import { Redis } from "ioredis";
  export const redis = new Redis(process.env.UPSTASH_REDIS_URL!, {
    maxRetriesPerRequest: null,
  });
  ```
- Create `src/server/queue/review-poll-worker.ts` (as shown in Deep-Dive):
  - `poll-scheduler` queue: repeatable job every 60 seconds
  - `poll-source` queue: individual source polling with adapter
  - `analyze-review` queue: AI analysis after review is stored
- Create `src/server/queue/start-workers.ts`: entry point for Railway worker process
  ```typescript
  import { startScheduler, stopWorkers } from "./review-poll-worker";

  async function main() {
    console.log("[worker] Starting ReviewRadar workers...");
    await startScheduler();
    console.log("[worker] Scheduler started. Workers are running.");

    process.on("SIGTERM", async () => {
      console.log("[worker] SIGTERM received, shutting down...");
      await stopWorkers();
      process.exit(0);
    });
  }

  main().catch(console.error);
  ```

**Day 11 — Source connection UI**
- Create `src/app/dashboard/settings/sources/page.tsx`:
  - List of connected review sources with platform icon, name, status badge (green/red/gray), last checked, review count
  - "Add Source" button opens dialog with platform selection
- Create `src/components/sources/add-source-dialog.tsx`:
  - Step 1: Select platform (Google, Yelp, G2) with icons
  - Step 2 (Google): "Connect with Google" button -> OAuth flow
  - Step 2 (Yelp): Input Yelp business URL or alias, test connection
  - Step 2 (G2): Input G2 product URL or slug, test connection
  - Step 3: Confirmation with listing info preview (name, current rating, review count)
- Create `src/components/sources/source-card.tsx`: status indicator, actions (pause/resume, edit, delete, test connection)
- Wire up to `reviewSource` tRPC router

**Day 12 — Location management UI + source wiring**
- Create `src/app/dashboard/settings/locations/page.tsx`:
  - List of locations with name, address, aggregate rating, total sources, total reviews
  - "Add Location" dialog: name, address, city, state
  - Click into location to see its sources
- Create `src/components/locations/location-card.tsx`: summary card with sparkline
- Wire "Add Source" flow to require selecting a location first
- End-to-end test: add location -> add Yelp source -> trigger manual poll -> verify reviews stored in DB

---

### Phase 3: AI Sentiment + Response (Days 13-17)

**Day 13 — AI sentiment analysis implementation**
- Create `src/server/ai/analyze-review.ts` (as shown in Deep-Dive)
- Wire into `analyze-review` BullMQ worker: after each new review is polled, enqueue analysis
- Test with sample reviews: verify sentiment scores, topics, categories populated
- Handle edge cases: empty review text (infer from rating), non-English reviews, very short reviews

**Day 14 — AI response drafting**
- Create `src/server/ai/draft-response.ts` (as shown in Deep-Dive)
- `src/server/trpc/routers/reviewResponse.ts`:
  - `draft`: accepts `{ reviewId, tone }`, calls `draftResponse()`, increments `aiResponsesUsedThisMonth`, returns draft text
  - `save`: accepts `{ reviewId, tone, aiDraft, finalText }`, stores in `review_responses`
  - `markPosted`: set `posted = true`, `postedAt = now()`, mark review as `responded = true`
  - `listForReview`: all responses for a review
  - `regenerate`: call `draftResponse()` again with existing drafts passed for diversity

**Day 15 — AI response panel UI**
- Create `src/components/reviews/response-panel.tsx`:
  - Left side: review card (platform icon, stars, reviewer name, text, date, sentiment badge)
  - Right side: AI response area
  - Tone selector: four buttons (Grateful / Professional / Empathetic / Apologetic) with icons
  - "Generate Response" button -> calls `reviewResponse.draft`
  - Editable textarea for the response text
  - "Regenerate" button for a new draft
  - "Copy to Clipboard" button (for all platforms)
  - "Mark as Responded" button
  - Response history: list of previous drafts with timestamps
- Create `src/components/reviews/tone-selector.tsx`: styled radio group with descriptions

**Day 16 — Usage tracking + plan enforcement for AI**
- Add AI usage counter reset logic: check `aiResponsesResetAt`, if past, reset count to 0 and set new reset date
- Enforce limits in `reviewResponse.draft` mutation:
  ```typescript
  // Check monthly AI response limit
  const org = ctx.org;
  if (org.plan === "free") {
    throw new TRPCError({ code: "FORBIDDEN", message: "AI responses require Pro plan or higher." });
  }
  if (org.plan === "pro" && org.aiResponsesUsedThisMonth >= 50) {
    throw new TRPCError({ code: "FORBIDDEN", message: "Monthly AI response limit reached (50/50). Upgrade to Business for unlimited." });
  }
  ```
- Show usage indicator in response panel: "12/50 AI responses used this month"
- Add upgrade prompt when limit reached

**Day 17 — Sentiment snapshot aggregation**
- Create `src/server/services/snapshot-aggregator.ts`:
  ```typescript
  /**
   * Compute daily sentiment snapshots for analytics charts.
   * Run once per day via BullMQ repeatable job, or on-demand for backfill.
   */
  export async function aggregateDailySnapshot(orgId: string, date: Date): Promise<void> {
    const dayStart = new Date(date);
    dayStart.setHours(0, 0, 0, 0);
    const dayEnd = new Date(dayStart);
    dayEnd.setDate(dayEnd.getDate() + 1);

    // Aggregate per-platform and overall
    const platforms = ["google", "yelp", "g2", null]; // null = all platforms

    for (const platform of platforms) {
      const conditions = [
        eq(reviews.orgId, orgId),
        sql`${reviews.publishedAt} >= ${dayStart.toISOString()}`,
        sql`${reviews.publishedAt} < ${dayEnd.toISOString()}`,
      ];
      if (platform) conditions.push(eq(reviews.platform, platform));

      const [stats] = await db
        .select({
          avgRating: sql<number>`COALESCE(AVG(${reviews.rating}), 0)`,
          count: sql<number>`COUNT(*)`,
          avgSentiment: sql<number>`COALESCE(AVG(${reviews.sentimentScore}), 0)`,
          positiveCount: sql<number>`COUNT(*) FILTER (WHERE ${reviews.sentimentLabel} = 'positive')`,
          neutralCount: sql<number>`COUNT(*) FILTER (WHERE ${reviews.sentimentLabel} = 'neutral')`,
          negativeCount: sql<number>`COUNT(*) FILTER (WHERE ${reviews.sentimentLabel} = 'negative')`,
        })
        .from(reviews)
        .where(and(...conditions));

      if (Number(stats.count) === 0) continue;

      // Upsert snapshot
      await db.insert(sentimentSnapshots).values({
        orgId,
        locationId: null,
        platform,
        date: dayStart,
        averageRating: Number(stats.avgRating),
        reviewCount: Number(stats.count),
        averageSentiment: Number(stats.avgSentiment),
        positiveCount: Number(stats.positiveCount),
        neutralCount: Number(stats.neutralCount),
        negativeCount: Number(stats.negativeCount),
      }).onConflictDoUpdate({
        target: [sentimentSnapshots.orgId, sentimentSnapshots.locationId, sentimentSnapshots.platform, sentimentSnapshots.date],
        set: {
          averageRating: Number(stats.avgRating),
          reviewCount: Number(stats.count),
          averageSentiment: Number(stats.avgSentiment),
          positiveCount: Number(stats.positiveCount),
          neutralCount: Number(stats.neutralCount),
          negativeCount: Number(stats.negativeCount),
        },
      });
    }
  }
  ```
- Add daily snapshot job to BullMQ scheduler: runs at midnight UTC

---

### Phase 4: Alert System (Days 18-21)

**Day 18 — Alert rule engine**
- Create `src/server/services/alert-evaluator.ts`:
  ```typescript
  export interface ReviewAlertContext {
    reviewId: string;
    rating: number;
    platform: string;
    sentimentLabel: string;
    topics: string[];
    text: string;
    locationId: string;
  }

  export async function evaluateAlertRules(
    orgId: string,
    context: ReviewAlertContext
  ): Promise<void> {
    const rules = await db
      .select()
      .from(alertRules)
      .where(and(eq(alertRules.orgId, orgId), eq(alertRules.enabled, true)));

    for (const rule of rules) {
      if (matchesRule(rule.conditions, context)) {
        await triggerAlert(orgId, rule, context);
      }
    }
  }

  function matchesRule(
    conditions: AlertRule["conditions"],
    ctx: ReviewAlertContext
  ): boolean {
    if (!conditions) return true;

    if (conditions.minRating != null && ctx.rating > conditions.minRating) return false;
    if (conditions.maxRating != null && ctx.rating < conditions.maxRating) return false;
    if (conditions.platforms?.length && !conditions.platforms.includes(ctx.platform)) return false;
    if (conditions.sentimentLabels?.length && !conditions.sentimentLabels.includes(ctx.sentimentLabel)) return false;
    if (conditions.locationIds?.length && !conditions.locationIds.includes(ctx.locationId)) return false;
    if (conditions.keywords?.length) {
      const lowerText = ctx.text.toLowerCase();
      const hasKeyword = conditions.keywords.some((kw) => lowerText.includes(kw.toLowerCase()));
      if (!hasKeyword) return false;
    }

    return true;
  }
  ```

**Day 19 — Alert channels: Email + Slack**
- Create `src/server/services/alert-dispatcher.ts`:
  ```typescript
  import { Resend } from "resend";
  import { WebClient } from "@slack/web-api";

  const resend = new Resend(process.env.RESEND_API_KEY);

  async function triggerAlert(
    orgId: string,
    rule: AlertRule,
    ctx: ReviewAlertContext
  ): Promise<void> {
    const [review] = await db.select().from(reviews).where(eq(reviews.id, ctx.reviewId)).limit(1);
    const [org] = await db.select().from(organizations).where(eq(organizations.id, orgId)).limit(1);

    for (const channel of rule.channels ?? []) {
      if (channel.type === "email") {
        await sendEmailAlert(org, review, channel.target);
      } else if (channel.type === "slack") {
        await sendSlackAlert(org, review);
      }

      // Log the alert
      await db.insert(alertLog).values({
        orgId,
        alertRuleId: rule.id,
        reviewId: ctx.reviewId,
        channel: channel.type,
        recipient: channel.target,
        status: "sent",
        sentAt: new Date(),
      });
    }
  }
  ```
- Implement `sendEmailAlert()`: HTML email template with review details, star rating, sentiment badge, "Respond Now" CTA link
- Implement `sendSlackAlert()`: Slack Block Kit message with review card, rating, reviewer name, review text snippet, "View in ReviewRadar" button
- Handle Slack webhook URL decryption from org settings
- Error handling: if send fails, log with status "failed" and error message

**Day 20 — Alert rule management UI**
- `src/server/trpc/routers/alertRule.ts`:
  - `list`: all rules for org
  - `create`: name, conditions, channels
  - `update`: conditions, channels, enabled toggle
  - `delete`: remove rule + keep alert log history
  - `getAlertLog`: paginated log of triggered alerts for org
- Create `src/app/dashboard/alerts/page.tsx`:
  - List of alert rules with name, conditions summary, channels, enabled toggle, last triggered date
  - "Create Alert Rule" button
  - Alert history tab: log of all triggered alerts with review link, channel, status, timestamp
- Create `src/components/alerts/alert-rule-form.tsx`:
  - Name input
  - Conditions builder: rating threshold slider, platform checkboxes, keyword tags input, sentiment checkboxes, location multi-select
  - Channel configuration: email address input, Slack toggle (requires org Slack integration)
  - Enable/disable switch

**Day 21 — Slack integration setup + default alert rules**
- Create `src/app/dashboard/settings/integrations/page.tsx`:
  - Slack section: "Connect Slack" button that accepts a webhook URL
  - Webhook URL stored encrypted in `organizations.settings.slackWebhookUrl`
  - Test button: sends a test message to the webhook
- Auto-create default alert rules for new organizations:
  - "All New Reviews" (email, all ratings)
  - "Negative Review Alert" (email + Slack if connected, rating <= 2)
- Enforce plan limits on Slack: Free plan shows "Upgrade to Pro for Slack alerts"

---

### Phase 5: Dashboard + Analytics (Days 22-27)

**Day 22 — Review feed page**
- Create `src/app/dashboard/reviews/page.tsx`:
  - Unified stream of all reviews, newest first (infinite scroll)
  - Quick stats bar at top: "12 new today", "8 unresponded", "4.3 avg rating"
  - Each review card shows:
    - Platform icon (Google, Yelp, G2 logos)
    - Star rating (colored: green 4-5, yellow 3, red 1-2)
    - Reviewer name + avatar
    - Review text (truncated to 3 lines, expandable)
    - Sentiment badge (Positive green, Neutral gray, Negative red, Mixed yellow)
    - Topic pills
    - Published date (relative: "2 hours ago")
    - Location name
    - "Respond" button -> opens response panel
    - "Responded" checkmark if already responded
- Filter bar: platform dropdown, rating range, sentiment multi-select, responded/unresponded toggle, date range picker, location filter
- Search box: full-text search via pg_trgm on review text

**Day 23 — Review feed tRPC router + filters**
- `src/server/trpc/routers/review.ts`:
  - `list`: paginated feed with all filter options
    ```typescript
    list: orgProcedure
      .input(z.object({
        cursor: z.string().uuid().optional(),
        limit: z.number().min(1).max(100).default(20),
        platform: z.enum(["google", "yelp", "g2"]).optional(),
        minRating: z.number().min(1).max(5).optional(),
        maxRating: z.number().min(1).max(5).optional(),
        sentimentLabel: z.array(z.string()).optional(),
        responded: z.boolean().optional(),
        locationId: z.string().uuid().optional(),
        search: z.string().optional(),
        dateFrom: z.date().optional(),
        dateTo: z.date().optional(),
      }))
    ```
  - `getById`: single review with responses and source info
  - `getStats`: overview stats (total reviews, unresponded count, avg rating, reviews today)
  - `search`: text search using pg_trgm
- Build dynamic filter WHERE clause:
  ```typescript
  const conditions = [eq(reviews.orgId, ctx.org.id)];
  if (input.platform) conditions.push(eq(reviews.platform, input.platform));
  if (input.minRating) conditions.push(sql`${reviews.rating} >= ${input.minRating}`);
  if (input.maxRating) conditions.push(sql`${reviews.rating} <= ${input.maxRating}`);
  if (input.responded !== undefined) conditions.push(eq(reviews.responded, input.responded));
  if (input.locationId) conditions.push(eq(reviews.locationId, input.locationId));
  if (input.search) conditions.push(sql`${reviews.text} % ${input.search}`);
  ```

**Day 24 — Review detail page + response flow**
- Create `src/app/dashboard/reviews/[id]/page.tsx`:
  - Full review display: all fields, expandable text
  - AI analysis section: sentiment score, topics, category, confidence indicator
  - Response panel (as built in Phase 3)
  - Response history: all previous drafts and posted responses
  - Source link: "View on Google" / "View on Yelp" external link
- Wire up response panel to tRPC mutations
- "Mark as Responded" updates `reviews.responded = true`

**Day 25 — Analytics tRPC router**
- Create `src/server/trpc/routers/analytics.ts`:
  - `ratingTrend`: average rating per week for the last 90 days, grouped by platform
  - `reviewVolume`: review count per week, grouped by platform
  - `sentimentBreakdown`: pie chart data — positive/neutral/negative/mixed counts
  - `topTopics`: top 20 topics across all reviews with count and avg sentiment
  - `platformComparison`: per-platform stats (avg rating, review count, response rate)
  - `overviewStats`: total reviews, avg rating, response rate, sentiment trend direction
  - `recentReviews`: last 5 reviews for dashboard widget
  - `ratingDistribution`: count of reviews per star rating (1-5)

**Day 26 — Analytics dashboard UI**
- Create `src/app/dashboard/analytics/page.tsx`:
  - Overview cards row: Total Reviews, Average Rating, Response Rate (%), Sentiment Score
  - Rating trend line chart (Recharts `LineChart` with platform color lines)
  - Review volume stacked bar chart (Recharts `BarChart` per platform)
  - Sentiment distribution pie chart (Recharts `PieChart` — green/gray/red/yellow)
  - Top topics horizontal bar chart: topic name, count, colored by avg sentiment
  - Platform comparison table: platform icon, avg rating, review count, response rate
  - Rating distribution bar chart: 1-star through 5-star counts
- Create individual chart components:
  - `src/components/analytics/rating-trend-chart.tsx`
  - `src/components/analytics/volume-chart.tsx`
  - `src/components/analytics/sentiment-pie.tsx`
  - `src/components/analytics/topics-chart.tsx`
  - `src/components/analytics/stats-cards.tsx`
  - `src/components/analytics/platform-table.tsx`

**Day 27 — Dashboard overview page**
- Create `src/app/dashboard/page.tsx`:
  - Welcome header with org name
  - Quick stats row: New Reviews Today, Unresponded Reviews, Average Rating (with trend arrow), Active Sources
  - "Recent Reviews" — last 5 reviews with platform icon, stars, text snippet, "Respond" link
  - "Needs Attention" — unresponded negative reviews (rating <= 2), sorted by age
  - "Rating Trend" — small sparkline for last 30 days
  - Quick action buttons: "Add Source", "Create Alert Rule", "View All Reviews"

---

### Phase 6: Billing + Settings (Days 28-31)

**Day 28 — Stripe billing setup**
- Create `src/server/billing/stripe.ts`:
  ```typescript
  import Stripe from "stripe";

  const stripe = new Stripe(process.env.STRIPE_SECRET_KEY!, {
    apiVersion: "2025-04-30.basil",
  });

  const PRICE_IDS: Record<string, string> = {
    pro: process.env.STRIPE_PRO_PRICE_ID ?? "price_pro_placeholder",
    business: process.env.STRIPE_BUSINESS_PRICE_ID ?? "price_business_placeholder",
  };

  export const PLANS = {
    free: {
      name: "Free",
      price: 0,
      maxLocations: 1,
      maxSourcesPerLocation: 2,
      maxAiResponses: 0,
      maxMembers: 1,
      historyDays: 30,
      features: [
        "1 location",
        "2 review platforms",
        "Email alerts",
        "30-day review history",
        "Review feed with filters",
      ],
    },
    pro: {
      name: "Pro",
      price: 29,
      maxLocations: 1,
      maxSourcesPerLocation: 5,
      maxAiResponses: 50,
      maxMembers: 1,
      historyDays: 365,
      features: [
        "1 location",
        "5 review platforms",
        "50 AI responses/month",
        "Sentiment analysis",
        "Slack alerts",
        "1-year review history",
        "Analytics dashboard",
      ],
    },
    business: {
      name: "Business",
      price: 69,
      maxLocations: 5,
      maxSourcesPerLocation: Infinity,
      maxAiResponses: Infinity,
      maxMembers: 5,
      historyDays: Infinity,
      features: [
        "5 locations",
        "Unlimited platforms",
        "Unlimited AI responses",
        "All integrations",
        "Team collaboration (5 users)",
        "Unlimited history",
        "Priority support",
      ],
    },
  } as const;

  export async function createCheckoutSession(opts: {
    orgId: string;
    clerkOrgId: string;
    stripeCustomerId?: string;
    plan: "pro" | "business";
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

**Day 29 — Stripe webhook handler**
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

**Day 30 — Billing tRPC router + UI**
- `src/server/trpc/routers/billing.ts`:
  - `getCurrentPlan`: return org plan + limits + usage (AI responses used, locations count, sources count)
  - `createCheckout`: generate Stripe checkout URL for plan upgrade
  - `createPortal`: generate Stripe portal URL for managing subscription
- Create `src/app/dashboard/settings/billing/page.tsx`:
  - Current plan card with usage meters (AI responses used / limit, locations used / limit, sources per location)
  - Three plan cards: Free, Pro ($29/mo), Business ($69/mo) with feature lists and checkmarks
  - "Upgrade" or "Manage Subscription" buttons
  - Success/canceled URL parameter handling with toast messages

**Day 31 — Settings pages**
- Create `src/app/dashboard/settings/page.tsx`:
  - Organization name (editable)
  - Brand name (for AI responses: "the team at {brandName}")
  - Default response tone selector
  - Timezone selector (for digest scheduling)
- Create `src/app/dashboard/settings/integrations/page.tsx`:
  - Slack webhook URL configuration (encrypted storage)
  - Test Slack connection button
  - Email digest settings: enable/disable, frequency (daily/weekly), day and hour
- Create `src/app/dashboard/settings/members/page.tsx` (Business plan):
  - Team member list from Clerk organization
  - Invite form
  - Role badges (owner, admin, member)
  - Plan enforcement: show "Upgrade to Business for team collaboration"

---

### Phase 7: Polish + Launch (Days 32-36)

**Day 32 — Plan enforcement + rate limiting**
- Implement plan enforcement across all tRPC routers:
  - Location creation: check `maxLocations`
  - Source creation: check `maxSourcesPerLocation`
  - AI response drafting: check `maxAiResponses`
  - Slack alerts: require Pro+ plan
  - Analytics dashboard: require Pro+ plan (Free shows limited stats)
  - Member management: require Business plan
- Review history enforcement: for Free plan (30-day), filter queries to only show reviews from last 30 days
- Add usage banner in dashboard when approaching limits (>80%):
  ```typescript
  // Show upgrade prompt when usage exceeds threshold
  const usagePercent = (org.aiResponsesUsedThisMonth / planLimits.maxAiResponses) * 100;
  if (usagePercent >= 80) {
    showBanner("You've used 80% of your AI responses this month. Upgrade for more.");
  }
  ```

**Day 33 — Onboarding flow**
- Create `src/app/onboarding/page.tsx`:
  - Step 1: Name your organization + create first location (name, address)
  - Step 2: Connect first review source (Google / Yelp / G2 with setup instructions)
  - Step 3: Wait for first reviews to import (show progress spinner, then preview of imported reviews)
  - Step 4: Set up first alert rule (pre-populated "Negative Review Alert")
  - Progress indicator (4 steps)
  - Skip option to go straight to dashboard
- Store onboarding completion in `organizations.settings.onboardingCompleted`
- Redirect new orgs to onboarding if not completed

**Day 34 — Loading states, error handling, empty states**
- Add Suspense boundaries with skeleton loaders for all data-fetching pages
- Create empty state illustrations:
  - Review feed empty: "No reviews yet — connect a review source to start monitoring"
  - Analytics empty: "Need more data — analytics appear after 10+ reviews are imported"
  - Alerts empty: "No alert rules — create your first alert to get notified about new reviews"
  - Sources empty: "No sources connected — add Google, Yelp, or G2 to start monitoring"
- Toast notifications for all mutations (success/error)
- Global error boundary with retry
- 404 page for invalid routes
- tRPC error handling: map error codes to user-friendly messages

**Day 35 — Responsive design + performance**
- Mobile-responsive sidebar: collapsible on small screens, bottom nav on mobile
- Review feed: single-column layout on mobile, swipe actions for "Respond"
- Analytics: stack charts vertically on mobile
- All interactive elements: focus rings, keyboard navigation, ARIA labels
- Add `loading.tsx` files for route-level streaming
- Optimize database queries: ensure all list queries use indexed columns
- Add React Query `staleTime` and `gcTime` settings:
  - Review feed: staleTime 15s (near real-time)
  - Analytics: staleTime 5min
  - Sources list: staleTime 30s
  - Billing: staleTime 1min
- Server-side rendering for dashboard pages using tRPC server caller

**Day 36 — Landing page + deployment**
- Create `src/app/page.tsx`:
  - Hero: "Monitor Every Review. Respond in Seconds." with CTA
  - How it works: 3-step visual (Connect Sources -> AI Analyzes -> You Respond)
  - Features grid: Multi-Platform Monitoring, AI Responses, Real-Time Alerts, Sentiment Analytics, Team Collaboration
  - Pricing table with three tiers
  - Testimonial placeholder
  - Footer with links
- Create `src/app/pricing/page.tsx`: detailed pricing comparison table
- Set up Vercel project + environment variables
- Set up Railway project for scraping workers with Playwright
- Configure custom domain
- Set up Sentry for error tracking
- Deploy to production
- Stripe webhook endpoint registered in Stripe dashboard
- Google OAuth redirect URIs updated to production URLs
- End-to-end test flow:
  1. Sign up -> Create org -> Onboarding
  2. Add location -> Connect Yelp source -> Verify reviews polled
  3. Verify AI sentiment analysis runs on imported reviews
  4. Draft AI response with different tones -> Copy to clipboard
  5. Create alert rule -> Verify alert triggers on next poll
  6. Check analytics dashboard populates with charts
  7. Upgrade plan via Stripe (test mode)
  8. Verify plan limits enforced

---

## Critical Files

```
reviewradar/
├── src/
│   ├── env.ts                                       # Zod-validated environment variables
│   ├── middleware.ts                                 # Clerk auth middleware, route protection
│   │
│   ├── app/
│   │   ├── layout.tsx                               # Root layout with ClerkProvider
│   │   ├── page.tsx                                  # Landing page
│   │   ├── pricing/page.tsx                          # Pricing comparison page
│   │   ├── onboarding/page.tsx                       # 4-step onboarding wizard
│   │   │
│   │   ├── dashboard/
│   │   │   ├── layout.tsx                           # App shell: sidebar + header
│   │   │   ├── page.tsx                             # Dashboard overview
│   │   │   ├── reviews/
│   │   │   │   ├── page.tsx                         # Review feed (stream + filters)
│   │   │   │   └── [id]/page.tsx                    # Review detail + response panel
│   │   │   ├── analytics/page.tsx                   # Analytics dashboard (charts)
│   │   │   ├── alerts/page.tsx                      # Alert rules + alert history
│   │   │   └── settings/
│   │   │       ├── page.tsx                         # Organization settings
│   │   │       ├── locations/page.tsx               # Location management
│   │   │       ├── sources/page.tsx                 # Review source management
│   │   │       ├── integrations/page.tsx            # Slack + email digest settings
│   │   │       ├── billing/page.tsx                 # Plan + usage + upgrade
│   │   │       └── members/page.tsx                 # Team member management
│   │   │
│   │   └── api/
│   │       ├── trpc/[trpc]/route.ts                 # tRPC HTTP handler
│   │       ├── auth/
│   │       │   ├── google/route.ts                  # Google OAuth start
│   │       │   └── google/callback/route.ts         # Google OAuth callback
│   │       └── webhooks/
│   │           └── stripe/route.ts                  # Stripe webhook handler
│   │
│   ├── server/
│   │   ├── db/
│   │   │   ├── schema.ts                           # Drizzle schema (all tables + relations)
│   │   │   ├── index.ts                            # Neon database client
│   │   │   └── migrations/
│   │   │       └── 0000_init.sql                   # Trigram + partial indexes
│   │   │
│   │   ├── trpc/
│   │   │   ├── trpc.ts                             # tRPC init, auth/org/plan middleware
│   │   │   ├── context.ts                          # Request context (auth + db)
│   │   │   └── routers/
│   │   │       ├── _app.ts                         # Merged app router
│   │   │       ├── organization.ts                 # Org CRUD + settings
│   │   │       ├── location.ts                     # Location CRUD
│   │   │       ├── reviewSource.ts                 # Source CRUD + connection test
│   │   │       ├── review.ts                       # Review feed, filters, search, stats
│   │   │       ├── reviewResponse.ts               # AI draft, save, mark posted
│   │   │       ├── alertRule.ts                    # Alert rule CRUD + alert log
│   │   │       ├── analytics.ts                    # Dashboard data queries
│   │   │       └── billing.ts                      # Plan info + Stripe checkout/portal
│   │   │
│   │   ├── adapters/
│   │   │   ├── types.ts                            # ReviewSourceAdapter interface
│   │   │   ├── index.ts                            # Adapter registry
│   │   │   ├── google-adapter.ts                   # Google Business Profile API adapter
│   │   │   ├── yelp-adapter.ts                     # Yelp Playwright scraper adapter
│   │   │   └── g2-adapter.ts                       # G2 Playwright scraper adapter
│   │   │
│   │   ├── ai/
│   │   │   ├── analyze-review.ts                   # GPT-4o-mini sentiment + topic extraction
│   │   │   └── draft-response.ts                   # GPT-4o-mini response drafting (4 tones)
│   │   │
│   │   ├── queue/
│   │   │   ├── connection.ts                       # Redis/ioredis connection
│   │   │   ├── review-poll-worker.ts               # Scheduler + poll + analyze workers
│   │   │   └── start-workers.ts                    # Worker process entry point (Railway)
│   │   │
│   │   ├── services/
│   │   │   ├── alert-evaluator.ts                  # Alert rule matching engine
│   │   │   ├── alert-dispatcher.ts                 # Email + Slack alert sending
│   │   │   └── snapshot-aggregator.ts              # Daily sentiment snapshot computation
│   │   │
│   │   ├── integrations/
│   │   │   └── encryption.ts                       # AES-256-GCM encrypt/decrypt for credentials
│   │   │
│   │   ├── billing/
│   │   │   └── stripe.ts                           # Plan definitions, checkout, portal
│   │   │
│   │   └── email/
│   │       └── templates.ts                        # HTML email templates for alerts + digests
│   │
│   ├── components/
│   │   ├── ui/                                     # Shared Radix UI primitives
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
│   │   │   ├── switch.tsx
│   │   │   └── popover.tsx
│   │   │
│   │   ├── layout/
│   │   │   ├── sidebar.tsx                         # Navigation sidebar
│   │   │   ├── header.tsx                          # Page header with breadcrumbs
│   │   │   └── location-switcher.tsx               # Location context switcher
│   │   │
│   │   ├── reviews/
│   │   │   ├── review-card.tsx                     # Reusable review card
│   │   │   ├── review-list.tsx                     # Infinite scroll review feed
│   │   │   ├── review-filters.tsx                  # Filter bar (platform, rating, sentiment)
│   │   │   ├── response-panel.tsx                  # AI response drafting panel
│   │   │   └── tone-selector.tsx                   # Tone radio group
│   │   │
│   │   ├── sources/
│   │   │   ├── add-source-dialog.tsx               # Multi-step source connection wizard
│   │   │   └── source-card.tsx                     # Source status card
│   │   │
│   │   ├── locations/
│   │   │   └── location-card.tsx                   # Location summary card
│   │   │
│   │   ├── alerts/
│   │   │   ├── alert-rule-form.tsx                 # Alert rule condition builder
│   │   │   └── alert-log-table.tsx                 # Alert history table
│   │   │
│   │   └── analytics/
│   │       ├── rating-trend-chart.tsx              # Recharts line chart
│   │       ├── volume-chart.tsx                    # Recharts stacked bar chart
│   │       ├── sentiment-pie.tsx                   # Recharts pie chart
│   │       ├── topics-chart.tsx                    # Horizontal bar chart
│   │       ├── platform-table.tsx                  # Platform comparison table
│   │       └── stats-cards.tsx                     # Overview metric cards
│   │
│   └── lib/
│       ├── trpc/
│       │   ├── client.ts                           # React Query tRPC client
│       │   └── server.ts                           # Server-side tRPC caller
│       ├── utils.ts                                # cn() helper, date formatting, rating colors
│       └── constants.ts                            # Platform icons, sentiment labels, plan limits
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

1. **Auth flow**: Sign up with Clerk -> create organization -> verify org record created in DB with `plan = "free"`
2. **Google OAuth**: Click "Connect Google" -> complete OAuth -> verify encrypted tokens stored in `review_sources.credentials`
3. **Yelp source**: Enter Yelp business alias -> test connection -> verify listing info returned (name, rating, review count)
4. **G2 source**: Enter G2 product slug -> test connection -> verify listing info returned
5. **Review polling**: Add a source -> trigger manual poll -> verify reviews stored in `reviews` table with correct platform, rating, text
6. **AI sentiment**: After reviews are polled, verify `sentiment_score`, `sentiment_label`, `topics`, and `category` are populated within 30 seconds
7. **Deduplication**: Poll the same source again -> verify no duplicate reviews are created (check `external_review_id` uniqueness)
8. **AI response drafting**: Open a review -> select "empathetic" tone -> generate response -> verify response references specific details from the review text
9. **Tone variation**: Generate responses with all four tones for the same review -> verify each tone produces a meaningfully different response
10. **AI usage limit**: On Pro plan, generate 50 responses -> verify 51st is blocked with upgrade prompt
11. **Alert rule creation**: Create "Negative Review" alert (rating <= 2, email channel) -> verify rule stored in DB
12. **Alert triggering**: Poll a source with a 1-star review -> verify alert email sent and logged in `alert_log`
13. **Slack alert**: Configure Slack webhook -> trigger alert -> verify Slack message delivered with review card
14. **Review feed filters**: Apply platform filter -> verify only matching platform reviews shown. Apply rating filter -> verify correct range. Apply responded/unresponded toggle -> verify correct results.
15. **Analytics charts**: After 10+ reviews imported, verify rating trend chart, volume chart, sentiment pie chart, and topics chart all render with data
16. **Plan limits (locations)**: On Free plan, attempt to add 2nd location -> verify error "Free plan allows 1 location"
17. **Plan limits (sources)**: On Free plan, attempt to add 3rd source to a location -> verify error
18. **Billing upgrade**: Complete Stripe checkout for Pro -> verify plan updated in DB, AI responses unlocked
19. **Stripe webhook (cancellation)**: Simulate subscription deletion -> verify plan reverts to "free"
20. **Onboarding**: New org -> verify redirected to onboarding -> complete all steps -> verify `onboardingCompleted = true`

### Key SQL Queries for Verification

```sql
-- Check reviews were imported with AI analysis
SELECT id, platform, rating, reviewer_name,
       LEFT(text, 80) AS text_preview,
       sentiment_label, sentiment_score,
       topics, category, ai_processed
FROM reviews
WHERE org_id = 'ORG_ID'
ORDER BY published_at DESC
LIMIT 10;

-- Check source polling status
SELECT rs.id, rs.platform, rs.name, rs.status,
       rs.last_checked_at, rs.next_check_at,
       rs.current_rating, rs.total_review_count,
       rs.error_message
FROM review_sources rs
WHERE rs.org_id = 'ORG_ID';

-- Check alert delivery log
SELECT al.id, al.channel, al.recipient, al.status, al.sent_at,
       r.platform, r.rating, r.reviewer_name
FROM alert_log al
JOIN reviews r ON r.id = al.review_id
WHERE al.org_id = 'ORG_ID'
ORDER BY al.created_at DESC;

-- Monthly AI response usage
SELECT ai_responses_used_this_month, ai_responses_reset_at, plan
FROM organizations
WHERE id = 'ORG_ID';

-- Sentiment trends (for chart verification)
SELECT date, platform, average_rating, review_count,
       average_sentiment, positive_count, neutral_count, negative_count
FROM sentiment_snapshots
WHERE org_id = 'ORG_ID'
ORDER BY date DESC
LIMIT 30;

-- Top topics across all reviews
SELECT topic, COUNT(*) AS mention_count, AVG(r.sentiment_score) AS avg_sentiment
FROM reviews r, jsonb_array_elements_text(r.topics::jsonb) AS topic
WHERE r.org_id = 'ORG_ID'
  AND r.ai_processed = true
GROUP BY topic
ORDER BY mention_count DESC
LIMIT 20;

-- Review response rate by platform
SELECT platform,
       COUNT(*) AS total,
       COUNT(*) FILTER (WHERE responded = true) AS responded,
       ROUND(100.0 * COUNT(*) FILTER (WHERE responded = true) / COUNT(*), 1) AS response_rate
FROM reviews
WHERE org_id = 'ORG_ID'
GROUP BY platform;
```

### Performance Benchmarks

| Operation | Target | Method |
|-----------|--------|--------|
| Source poll cycle (Yelp, 30 reviews) | < 15s | Playwright headless, single page load |
| Source poll cycle (Google API, 50 reviews) | < 3s | REST API, no browser |
| Source poll cycle (G2, 20 reviews) | < 12s | Playwright headless, single page load |
| AI sentiment analysis (single review) | < 2s | GPT-4o-mini with 400 max_tokens, JSON mode |
| AI response drafting (single review) | < 3s | GPT-4o-mini with 300 max_tokens |
| Review feed page load (50 reviews) | < 500ms | Indexed queries + React Query cache |
| Alert rule evaluation (10 rules) | < 50ms | In-memory rule matching, no DB per rule |
| Email alert dispatch | < 2s | Resend API |
| Slack alert dispatch | < 1s | Slack Web API |
| Analytics dashboard load | < 800ms | Pre-aggregated sentiment snapshots |
| Scheduler tick (find due sources) | < 100ms | Indexed `next_check_at` query |
| Daily snapshot aggregation (1000 reviews) | < 5s | SQL aggregate with FILTER clauses |
