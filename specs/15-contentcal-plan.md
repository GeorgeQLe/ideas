# 15. ContentCal — AI Social Media Content Planner

## Implementation Plan

**MVP Scope:** Business profile setup with brand voice, target audience, and content pillars, AI content generation via GPT-4o producing a full month of platform-optimized social media posts in one click, content calendar with drag-and-drop rescheduling, post editor with side-by-side platform previews (Twitter/X and LinkedIn), hashtag suggestion engine, direct publishing to Twitter/X (API v2) and LinkedIn (OAuth 2.0), auto-scheduling at optimal times with BullMQ-based publishing queue, basic scheduling dashboard, Stripe billing with four tiers (Free / Pro $19/mo / Business $49/mo / Agency $99/mo).

---

## Tech Decisions

| Layer | Technology | Notes |
|-------|-----------|-------|
| Framework | Next.js 14+ App Router | `src/` directory, TypeScript strict |
| API | tRPC v11 | superjson transformer, Clerk auth context |
| Database | Neon PostgreSQL | Brands, posts, variants, social accounts, metrics |
| ORM | Drizzle ORM | Push-based migrations, typed schema |
| Auth | Clerk | Organization-scoped, `orgId` from session |
| Billing | Stripe | Checkout Sessions + Customer Portal + Webhooks |
| AI — Content | OpenAI GPT-4o | Temperature 0.8 for creative social media copy |
| AI — Hashtags | OpenAI GPT-4o-mini | JSON mode for hashtag suggestions |
| Queue | BullMQ + Redis (Upstash) | Scheduled post publishing + content generation |
| Social — Twitter | Twitter API v2 (OAuth 2.0 PKCE) | Post tweets, threads (v2 only) |
| Social — LinkedIn | LinkedIn Marketing API (OAuth 2.0) | Share posts, articles |
| Image Storage | Cloudflare R2 | Media library, AI-generated images (v2) |
| Calendar UI | @hello-pangea/dnd | Drag-and-drop calendar cards |
| Email | Resend | Publishing confirmations, weekly digest |
| Hosting | Vercel (app), Upstash (Redis) | |
| Monitoring | Sentry, Vercel Analytics | |

### Key Architecture Decisions

1. **Post + PostVariant model for multi-platform content**: A `Post` is the parent concept (content pillar, scheduled time, status). Each `PostVariant` stores platform-specific text, hashtags, media, and thread/carousel data. This allows a single content idea to be adapted per platform with different copy lengths, hashtag strategies, and media formats — while maintaining a unified calendar view.

2. **BullMQ delayed jobs for scheduled publishing**: When a post is scheduled, a BullMQ job is created with a `delay` matching `scheduledAt - now`. The worker picks up the job at the exact time, calls the platform API, and updates the variant's `externalPostId`. This provides reliable at-time publishing without polling. Failed publishes are retried 3 times with exponential backoff, then marked as `failed` with an error message.

3. **AI content generation as batch pipeline**: "Generate a month of content" triggers a BullMQ job that produces 20-30 posts in a single GPT-4o call (using structured JSON output). Each post gets platform variants generated in a follow-up call. This batching reduces API costs vs. generating one post at a time and enables the AI to maintain content variety and strategic spacing of topics.

4. **OAuth token refresh with encrypted storage**: Twitter and LinkedIn OAuth tokens are stored AES-256-GCM encrypted in `socialAccounts.accessTokenEncrypted`. A `tokenRefreshMiddleware` checks `expiresAt` before every API call and refreshes if within 5 minutes of expiry. Refresh failures trigger an `expired` status on the social account and notify the user to reconnect.

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
  plan: text("plan").default("free").notNull(), // free | pro | business | agency
  stripeCustomerId: text("stripe_customer_id"),
  stripeSubscriptionId: text("stripe_subscription_id"),
  settings: jsonb("settings").default({}).$type<{
    onboardingCompleted?: boolean;
    defaultTimezone?: string;
    notificationEmail?: string;
  }>(),
  createdAt: timestamp("created_at", { withTimezone: true }).defaultNow().notNull(),
  updatedAt: timestamp("updated_at", { withTimezone: true }).defaultNow().notNull(),
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
    role: text("role").default("member").notNull(), // owner | admin | member
    email: text("email").notNull(),
    name: text("name"),
    createdAt: timestamp("created_at", { withTimezone: true }).defaultNow().notNull(),
  },
  (table) => [
    uniqueIndex("members_org_user_idx").on(table.orgId, table.clerkUserId),
  ],
);

// ---------------------------------------------------------------------------
// Brands (multi-brand support)
// ---------------------------------------------------------------------------
export const brands = pgTable(
  "brands",
  {
    id: uuid("id").primaryKey().defaultRandom(),
    orgId: uuid("org_id")
      .references(() => organizations.id, { onDelete: "cascade" })
      .notNull(),
    name: text("name").notNull(),
    industry: text("industry"),
    targetAudience: text("target_audience"),
    voiceDescription: text("voice_description"), // e.g., "professional yet approachable, witty"
    voicePreset: text("voice_preset").default("professional"),
    // professional | casual | witty | inspirational | educational
    contentPillars: jsonb("content_pillars").default([]).$type<string[]>(),
    // e.g., ["Product tips", "Industry news", "Behind the scenes", "Customer stories"]
    topics: jsonb("topics").default([]).$type<string[]>(),
    // Specific expertise areas and keywords
    competitors: jsonb("competitors").default([]).$type<string[]>(),
    // Competitor account handles to learn from
    logoUrl: text("logo_url"),
    brandColors: jsonb("brand_colors").default({}).$type<{
      primary?: string;
      secondary?: string;
      accent?: string;
    }>(),
    postingConfig: jsonb("posting_config").default({}).$type<{
      twitter?: { frequency: number; bestTimes: string[] };
      linkedin?: { frequency: number; bestTimes: string[] };
      instagram?: { frequency: number; bestTimes: string[] };
    }>(),
    isActive: boolean("is_active").default(true).notNull(),
    createdAt: timestamp("created_at", { withTimezone: true }).defaultNow().notNull(),
    updatedAt: timestamp("updated_at", { withTimezone: true }).defaultNow().notNull(),
  },
  (table) => [
    index("brands_org_idx").on(table.orgId),
  ],
);

// ---------------------------------------------------------------------------
// Social Accounts (connected platform accounts)
// ---------------------------------------------------------------------------
export const socialAccounts = pgTable(
  "social_accounts",
  {
    id: uuid("id").primaryKey().defaultRandom(),
    brandId: uuid("brand_id")
      .references(() => brands.id, { onDelete: "cascade" })
      .notNull(),
    orgId: uuid("org_id")
      .references(() => organizations.id, { onDelete: "cascade" })
      .notNull(),
    platform: text("platform").notNull(), // twitter | linkedin | instagram | facebook | tiktok
    accountName: text("account_name").notNull(), // @handle or profile name
    accountId: text("account_id").notNull(), // Platform-specific user ID
    accessTokenEncrypted: text("access_token_encrypted").notNull(), // AES-256-GCM
    refreshTokenEncrypted: text("refresh_token_encrypted"),
    tokenExpiresAt: timestamp("token_expires_at", { withTimezone: true }),
    profileImageUrl: text("profile_image_url"),
    followerCount: integer("follower_count"),
    status: text("status").default("connected").notNull(), // connected | expired | error
    errorMessage: text("error_message"),
    lastSyncedAt: timestamp("last_synced_at", { withTimezone: true }),
    createdAt: timestamp("created_at", { withTimezone: true }).defaultNow().notNull(),
  },
  (table) => [
    index("social_accounts_brand_idx").on(table.brandId),
    index("social_accounts_org_idx").on(table.orgId),
    uniqueIndex("social_accounts_platform_account_idx").on(
      table.platform,
      table.accountId,
    ),
  ],
);

// ---------------------------------------------------------------------------
// Posts (parent content concept)
// ---------------------------------------------------------------------------
export const posts = pgTable(
  "posts",
  {
    id: uuid("id").primaryKey().defaultRandom(),
    brandId: uuid("brand_id")
      .references(() => brands.id, { onDelete: "cascade" })
      .notNull(),
    orgId: uuid("org_id")
      .references(() => organizations.id, { onDelete: "cascade" })
      .notNull(),
    contentPillar: text("content_pillar"), // Which pillar this post belongs to
    contentType: text("content_type"), // tip | news | behind_the_scenes | testimonial | promo | engagement | holiday
    status: text("status").default("draft").notNull(),
    // draft | pending_approval | approved | scheduled | published | failed | archived
    scheduledAt: timestamp("scheduled_at", { withTimezone: true }),
    publishedAt: timestamp("published_at", { withTimezone: true }),
    isAiGenerated: boolean("is_ai_generated").default(false).notNull(),
    aiRegenerationCount: integer("ai_regeneration_count").default(0).notNull(),
    batchId: text("batch_id"), // Group posts generated in same AI batch
    calendarOrder: integer("calendar_order").default(0).notNull(), // For ordering within same day
    createdByUserId: text("created_by_user_id"), // Clerk user ID
    createdAt: timestamp("created_at", { withTimezone: true }).defaultNow().notNull(),
    updatedAt: timestamp("updated_at", { withTimezone: true }).defaultNow().notNull(),
  },
  (table) => [
    index("posts_brand_idx").on(table.brandId),
    index("posts_org_idx").on(table.orgId),
    index("posts_status_idx").on(table.orgId, table.status),
    index("posts_scheduled_idx").on(table.scheduledAt),
    index("posts_batch_idx").on(table.batchId),
  ],
);

// ---------------------------------------------------------------------------
// Post Variants (platform-specific content)
// ---------------------------------------------------------------------------
export const postVariants = pgTable(
  "post_variants",
  {
    id: uuid("id").primaryKey().defaultRandom(),
    postId: uuid("post_id")
      .references(() => posts.id, { onDelete: "cascade" })
      .notNull(),
    socialAccountId: uuid("social_account_id")
      .references(() => socialAccounts.id, { onDelete: "set null" }),
    platform: text("platform").notNull(), // twitter | linkedin
    text: text("text").notNull(),
    hashtags: jsonb("hashtags").default([]).$type<string[]>(),
    mediaUrls: jsonb("media_urls").default([]).$type<string[]>(), // R2 URLs
    linkUrl: text("link_url"),
    cta: text("cta"), // Call-to-action text
    threadTexts: jsonb("thread_texts").default([]).$type<string[]>(), // Twitter threads
    characterCount: integer("character_count").default(0).notNull(),
    externalPostId: text("external_post_id"), // Platform post ID after publishing
    externalPostUrl: text("external_post_url"), // URL to published post
    publishError: text("publish_error"),
    publishedAt: timestamp("published_at", { withTimezone: true }),
    createdAt: timestamp("created_at", { withTimezone: true }).defaultNow().notNull(),
    updatedAt: timestamp("updated_at", { withTimezone: true }).defaultNow().notNull(),
  },
  (table) => [
    index("post_variants_post_idx").on(table.postId),
    index("post_variants_platform_idx").on(table.platform),
    index("post_variants_social_account_idx").on(table.socialAccountId),
  ],
);

// ---------------------------------------------------------------------------
// Post Metrics (per-variant engagement data)
// ---------------------------------------------------------------------------
export const postMetrics = pgTable(
  "post_metrics",
  {
    id: uuid("id").primaryKey().defaultRandom(),
    postVariantId: uuid("post_variant_id")
      .references(() => postVariants.id, { onDelete: "cascade" })
      .notNull(),
    platform: text("platform").notNull(),
    impressions: integer("impressions").default(0).notNull(),
    reach: integer("reach").default(0).notNull(),
    likes: integer("likes").default(0).notNull(),
    comments: integer("comments").default(0).notNull(),
    shares: integer("shares").default(0).notNull(),
    clicks: integer("clicks").default(0).notNull(),
    engagementRate: real("engagement_rate").default(0).notNull(), // (likes+comments+shares+clicks) / impressions
    fetchedAt: timestamp("fetched_at", { withTimezone: true }).defaultNow().notNull(),
  },
  (table) => [
    index("post_metrics_variant_idx").on(table.postVariantId),
    index("post_metrics_fetched_at_idx").on(table.fetchedAt),
  ],
);

// ---------------------------------------------------------------------------
// Media Library
// ---------------------------------------------------------------------------
export const mediaLibrary = pgTable(
  "media_library",
  {
    id: uuid("id").primaryKey().defaultRandom(),
    brandId: uuid("brand_id")
      .references(() => brands.id, { onDelete: "cascade" })
      .notNull(),
    orgId: uuid("org_id")
      .references(() => organizations.id, { onDelete: "cascade" })
      .notNull(),
    url: text("url").notNull(), // R2 URL
    r2Key: text("r2_key").notNull(),
    type: text("type").notNull(), // image | video
    source: text("source").default("upload").notNull(), // upload | ai_generated | unsplash
    fileName: text("file_name").notNull(),
    mimeType: text("mime_type").notNull(),
    sizeBytes: integer("size_bytes").notNull(),
    width: integer("width"),
    height: integer("height"),
    altText: text("alt_text"),
    tags: jsonb("tags").default([]).$type<string[]>(),
    createdAt: timestamp("created_at", { withTimezone: true }).defaultNow().notNull(),
  },
  (table) => [
    index("media_library_brand_idx").on(table.brandId),
    index("media_library_org_idx").on(table.orgId),
  ],
);

// ---------------------------------------------------------------------------
// Content Generation Jobs (track AI batch generation)
// ---------------------------------------------------------------------------
export const generationJobs = pgTable(
  "generation_jobs",
  {
    id: uuid("id").primaryKey().defaultRandom(),
    brandId: uuid("brand_id")
      .references(() => brands.id, { onDelete: "cascade" })
      .notNull(),
    orgId: uuid("org_id")
      .references(() => organizations.id, { onDelete: "cascade" })
      .notNull(),
    batchId: text("batch_id").notNull(), // Matches posts.batchId
    status: text("status").default("pending").notNull(), // pending | generating | completed | failed
    config: jsonb("config").default({}).$type<{
      platforms: string[];
      contentPillars?: string[];
      numberOfPosts: number;
      startDate: string; // ISO date
      endDate: string; // ISO date
      specificTopics?: string[];
    }>(),
    postsGenerated: integer("posts_generated").default(0).notNull(),
    errorMessage: text("error_message"),
    completedAt: timestamp("completed_at", { withTimezone: true }),
    createdAt: timestamp("created_at", { withTimezone: true }).defaultNow().notNull(),
  },
  (table) => [
    index("generation_jobs_brand_idx").on(table.brandId),
    index("generation_jobs_batch_idx").on(table.batchId),
  ],
);

// ---------------------------------------------------------------------------
// Publishing Queue Log
// ---------------------------------------------------------------------------
export const publishingLog = pgTable(
  "publishing_log",
  {
    id: uuid("id").primaryKey().defaultRandom(),
    postVariantId: uuid("post_variant_id")
      .references(() => postVariants.id, { onDelete: "cascade" })
      .notNull(),
    orgId: uuid("org_id")
      .references(() => organizations.id, { onDelete: "cascade" })
      .notNull(),
    action: text("action").notNull(), // scheduled | published | failed | retried | cancelled
    details: jsonb("details").default({}).$type<{
      externalPostId?: string;
      errorMessage?: string;
      retryCount?: number;
      jobId?: string;
    }>(),
    createdAt: timestamp("created_at", { withTimezone: true }).defaultNow().notNull(),
  },
  (table) => [
    index("publishing_log_variant_idx").on(table.postVariantId),
    index("publishing_log_org_idx").on(table.orgId),
  ],
);

// ---------------------------------------------------------------------------
// Relations
// ---------------------------------------------------------------------------
export const organizationsRelations = relations(organizations, ({ many }) => ({
  members: many(members),
  brands: many(brands),
  socialAccounts: many(socialAccounts),
  posts: many(posts),
  generationJobs: many(generationJobs),
  publishingLog: many(publishingLog),
}));

export const membersRelations = relations(members, ({ one }) => ({
  organization: one(organizations, {
    fields: [members.orgId],
    references: [organizations.id],
  }),
}));

export const brandsRelations = relations(brands, ({ one, many }) => ({
  organization: one(organizations, {
    fields: [brands.orgId],
    references: [organizations.id],
  }),
  socialAccounts: many(socialAccounts),
  posts: many(posts),
  mediaLibrary: many(mediaLibrary),
  generationJobs: many(generationJobs),
}));

export const socialAccountsRelations = relations(socialAccounts, ({ one, many }) => ({
  brand: one(brands, {
    fields: [socialAccounts.brandId],
    references: [brands.id],
  }),
  organization: one(organizations, {
    fields: [socialAccounts.orgId],
    references: [organizations.id],
  }),
  postVariants: many(postVariants),
}));

export const postsRelations = relations(posts, ({ one, many }) => ({
  brand: one(brands, {
    fields: [posts.brandId],
    references: [brands.id],
  }),
  organization: one(organizations, {
    fields: [posts.orgId],
    references: [organizations.id],
  }),
  variants: many(postVariants),
}));

export const postVariantsRelations = relations(postVariants, ({ one, many }) => ({
  post: one(posts, {
    fields: [postVariants.postId],
    references: [posts.id],
  }),
  socialAccount: one(socialAccounts, {
    fields: [postVariants.socialAccountId],
    references: [socialAccounts.id],
  }),
  metrics: many(postMetrics),
  publishingLog: many(publishingLog),
}));

export const postMetricsRelations = relations(postMetrics, ({ one }) => ({
  postVariant: one(postVariants, {
    fields: [postMetrics.postVariantId],
    references: [postVariants.id],
  }),
}));

export const mediaLibraryRelations = relations(mediaLibrary, ({ one }) => ({
  brand: one(brands, {
    fields: [mediaLibrary.brandId],
    references: [brands.id],
  }),
  organization: one(organizations, {
    fields: [mediaLibrary.orgId],
    references: [organizations.id],
  }),
}));

export const generationJobsRelations = relations(generationJobs, ({ one }) => ({
  brand: one(brands, {
    fields: [generationJobs.brandId],
    references: [brands.id],
  }),
  organization: one(organizations, {
    fields: [generationJobs.orgId],
    references: [organizations.id],
  }),
}));

export const publishingLogRelations = relations(publishingLog, ({ one }) => ({
  postVariant: one(postVariants, {
    fields: [publishingLog.postVariantId],
    references: [postVariants.id],
  }),
  organization: one(organizations, {
    fields: [publishingLog.orgId],
    references: [organizations.id],
  }),
}));
```

---

## Initial Migration

```sql
-- migrations/0000_init.sql

-- No extensions needed beyond standard PostgreSQL

-- No seed data needed — brands and social accounts are user-created
```

---

## Architecture Deep-Dives

### 1. AI Content Generation Pipeline

The content generation pipeline takes a brand profile and produces a full month of social media posts. It uses a two-stage approach: first generating the content calendar plan (topics, dates, pillars), then producing platform-optimized copy for each post.

```typescript
// src/server/ai/generate-content.ts

import OpenAI from "openai";
import { randomUUID } from "crypto";
import { addDays, format, eachDayOfInterval } from "date-fns";

const openai = new OpenAI();

interface GenerationConfig {
  brandName: string;
  industry: string;
  targetAudience: string;
  voiceDescription: string;
  contentPillars: string[];
  topics: string[];
  platforms: ("twitter" | "linkedin")[];
  postingConfig: {
    twitter?: { frequency: number; bestTimes: string[] };
    linkedin?: { frequency: number; bestTimes: string[] };
  };
  startDate: Date;
  endDate: Date;
  numberOfPosts: number;
}

interface GeneratedPost {
  contentPillar: string;
  contentType: string;
  scheduledDate: string; // ISO date
  scheduledTime: string; // HH:mm
  variants: {
    platform: string;
    text: string;
    hashtags: string[];
    cta?: string;
    threadTexts?: string[]; // Twitter threads
  }[];
}

const CALENDAR_SYSTEM_PROMPT = `You are a social media content strategist. Given a brand profile and date range, generate a content calendar.

For each post, provide:
- contentPillar: which content pillar it belongs to
- contentType: one of "tip", "news", "behind_the_scenes", "testimonial", "promo", "engagement", "holiday"
- scheduledDate: ISO date (YYYY-MM-DD)
- scheduledTime: optimal posting time (HH:mm, 24h format)
- topic: brief description of the post topic (1 sentence)

Rules:
1. Distribute posts evenly across the date range
2. Rotate through content pillars evenly
3. Mix content types: 40% educational (tips, news), 30% engagement, 20% promotional, 10% behind-the-scenes
4. Never schedule 2+ posts on the same day for the same platform
5. Use the provided best posting times
6. Include relevant holidays/awareness days if they fall in the date range
7. Vary topics to avoid repetition — each post should feel fresh

Return JSON: {"posts": [...]}`;

const COPY_SYSTEM_PROMPT = `You are a social media copywriter. Write platform-optimized posts based on the provided topic and brand voice.

For Twitter/X:
- Max 280 characters (excluding hashtags)
- Punchy, concise, conversational
- Use line breaks for readability
- 2-4 relevant hashtags
- Optional: if the topic is complex, create a thread (2-5 tweets)

For LinkedIn:
- 150-300 words
- Professional but not boring
- Use line breaks and short paragraphs
- Start with a hook line
- End with a question or call-to-action
- 3-5 hashtags at the end

Match the brand voice exactly. If the voice is "witty", be genuinely witty. If "professional", maintain gravitas.

Return JSON: {"variants": [{"platform": "...", "text": "...", "hashtags": [...], "cta": "...", "threadTexts": [...]}]}`;

export async function generateMonthlyContent(
  config: GenerationConfig,
): Promise<GeneratedPost[]> {
  const batchId = randomUUID();

  // Stage 1: Generate content calendar (topics + schedule)
  const calendarResponse = await openai.chat.completions.create({
    model: "gpt-4o",
    temperature: 0.8,
    max_tokens: 4000,
    response_format: { type: "json_object" },
    messages: [
      { role: "system", content: CALENDAR_SYSTEM_PROMPT },
      {
        role: "user",
        content: `Brand: ${config.brandName}
Industry: ${config.industry}
Target Audience: ${config.targetAudience}
Brand Voice: ${config.voiceDescription}
Content Pillars: ${config.contentPillars.join(", ")}
Topics/Expertise: ${config.topics.join(", ")}
Platforms: ${config.platforms.join(", ")}
Posting Config: ${JSON.stringify(config.postingConfig)}
Date Range: ${format(config.startDate, "yyyy-MM-dd")} to ${format(config.endDate, "yyyy-MM-dd")}
Number of Posts: ${config.numberOfPosts}

Generate the content calendar as JSON: {"posts": [...]}`,
      },
    ],
  });

  const calendar = JSON.parse(calendarResponse.choices[0].message.content!) as {
    posts: {
      contentPillar: string;
      contentType: string;
      scheduledDate: string;
      scheduledTime: string;
      topic: string;
    }[];
  };

  // Stage 2: Generate copy for each post (batched, 5 posts per API call)
  const generatedPosts: GeneratedPost[] = [];
  const batches = chunkArray(calendar.posts, 5);

  for (const batch of batches) {
    const copyResponse = await openai.chat.completions.create({
      model: "gpt-4o",
      temperature: 0.8,
      max_tokens: 6000,
      response_format: { type: "json_object" },
      messages: [
        { role: "system", content: COPY_SYSTEM_PROMPT },
        {
          role: "user",
          content: `Brand: ${config.brandName}
Brand Voice: ${config.voiceDescription}
Target Audience: ${config.targetAudience}
Platforms: ${config.platforms.join(", ")}

Generate copy for these ${batch.length} posts:
${batch.map((p, i) => `${i + 1}. [${p.contentPillar}] ${p.contentType}: ${p.topic}`).join("\n")}

Return JSON: {"posts": [{"variants": [{"platform": "...", "text": "...", "hashtags": [...], "cta": "...", "threadTexts": [...]}]}]}`,
        },
      ],
    });

    const copyResult = JSON.parse(copyResponse.choices[0].message.content!) as {
      posts: { variants: GeneratedPost["variants"] }[];
    };

    // Merge calendar metadata with generated copy
    for (let i = 0; i < batch.length; i++) {
      const calendarEntry = batch[i];
      const copyEntry = copyResult.posts[i];

      if (!copyEntry) continue;

      generatedPosts.push({
        contentPillar: calendarEntry.contentPillar,
        contentType: calendarEntry.contentType,
        scheduledDate: calendarEntry.scheduledDate,
        scheduledTime: calendarEntry.scheduledTime,
        variants: copyEntry.variants.map((v) => ({
          ...v,
          // Enforce character limits
          text:
            v.platform === "twitter"
              ? v.text.slice(0, 280)
              : v.text.slice(0, 3000),
          hashtags: v.hashtags?.slice(0, 5) || [],
        })),
      });
    }
  }

  return generatedPosts;
}

export async function regeneratePost(
  brandVoice: string,
  targetAudience: string,
  topic: string,
  platforms: string[],
  feedback?: string,
): Promise<GeneratedPost["variants"]> {
  const prompt = feedback
    ? `Regenerate this post with feedback: "${feedback}"\n\nOriginal topic: ${topic}`
    : `Generate a fresh take on: ${topic}`;

  const response = await openai.chat.completions.create({
    model: "gpt-4o",
    temperature: 0.9,
    max_tokens: 2000,
    response_format: { type: "json_object" },
    messages: [
      { role: "system", content: COPY_SYSTEM_PROMPT },
      {
        role: "user",
        content: `Brand Voice: ${brandVoice}
Target Audience: ${targetAudience}
Platforms: ${platforms.join(", ")}

${prompt}

Return JSON: {"variants": [...]}`,
      },
    ],
  });

  const result = JSON.parse(response.choices[0].message.content!) as {
    variants: GeneratedPost["variants"];
  };

  return result.variants;
}

function chunkArray<T>(arr: T[], size: number): T[][] {
  const chunks: T[][] = [];
  for (let i = 0; i < arr.length; i += size) {
    chunks.push(arr.slice(i, i + size));
  }
  return chunks;
}
```

### 2. BullMQ Publishing Queue Worker

The publishing worker runs delayed jobs that fire at each post's scheduled time. It handles Twitter and LinkedIn API calls, manages OAuth token refresh, and logs all publishing activity for debugging.

```typescript
// src/server/queue/publish-worker.ts

import { Worker, type Job } from "bullmq";
import { db } from "../db";
import {
  posts,
  postVariants,
  socialAccounts,
  publishingLog,
} from "../db/schema";
import { eq, and } from "drizzle-orm";
import { decrypt } from "../integrations/encryption";
import { TwitterApi } from "twitter-api-v2";

const connection = {
  host: process.env.UPSTASH_REDIS_HOST!,
  port: Number(process.env.UPSTASH_REDIS_PORT || 6379),
  password: process.env.UPSTASH_REDIS_PASSWORD!,
  tls: {},
};

export type PublishJobData = {
  postId: string;
  postVariantId: string;
  socialAccountId: string;
  platform: "twitter" | "linkedin";
  orgId: string;
};

export const publishWorker = new Worker<PublishJobData>(
  "publish",
  async (job: Job<PublishJobData>) => {
    const { postId, postVariantId, socialAccountId, platform, orgId } = job.data;

    // Load variant + social account
    const [variant] = await db
      .select()
      .from(postVariants)
      .where(eq(postVariants.id, postVariantId))
      .limit(1);

    if (!variant) throw new Error(`Variant ${postVariantId} not found`);

    const [account] = await db
      .select()
      .from(socialAccounts)
      .where(eq(socialAccounts.id, socialAccountId))
      .limit(1);

    if (!account) throw new Error(`Social account ${socialAccountId} not found`);

    // Check token expiry + refresh if needed
    const accessToken = await ensureFreshToken(account);

    let externalPostId: string;
    let externalPostUrl: string;

    try {
      switch (platform) {
        case "twitter":
          ({ externalPostId, externalPostUrl } = await publishToTwitter(
            accessToken,
            variant,
          ));
          break;
        case "linkedin":
          ({ externalPostId, externalPostUrl } = await publishToLinkedIn(
            accessToken,
            account.accountId,
            variant,
          ));
          break;
        default:
          throw new Error(`Unsupported platform: ${platform}`);
      }
    } catch (error) {
      const errorMessage =
        error instanceof Error ? error.message : "Unknown publishing error";

      // Update variant with error
      await db
        .update(postVariants)
        .set({ publishError: errorMessage, updatedAt: new Date() })
        .where(eq(postVariants.id, postVariantId));

      // Log failure
      await db.insert(publishingLog).values({
        postVariantId,
        orgId,
        action: "failed",
        details: {
          errorMessage,
          retryCount: job.attemptsMade,
          jobId: job.id,
        },
      });

      // If max retries reached, mark post as failed
      if (job.attemptsMade >= 3) {
        await db
          .update(posts)
          .set({ status: "failed", updatedAt: new Date() })
          .where(eq(posts.id, postId));
      }

      throw error; // Re-throw for BullMQ retry
    }

    // Success: update variant + post
    await db
      .update(postVariants)
      .set({
        externalPostId,
        externalPostUrl,
        publishError: null,
        publishedAt: new Date(),
        updatedAt: new Date(),
      })
      .where(eq(postVariants.id, postVariantId));

    // Check if all variants for this post are published
    const remainingVariants = await db
      .select()
      .from(postVariants)
      .where(
        and(
          eq(postVariants.postId, postId),
          eq(postVariants.publishedAt, null),
        ),
      );

    if (remainingVariants.length === 0) {
      await db
        .update(posts)
        .set({ status: "published", publishedAt: new Date(), updatedAt: new Date() })
        .where(eq(posts.id, postId));
    }

    // Log success
    await db.insert(publishingLog).values({
      postVariantId,
      orgId,
      action: "published",
      details: { externalPostId, jobId: job.id },
    });
  },
  {
    connection,
    concurrency: 5,
    limiter: { max: 10, duration: 60_000 }, // 10 publishes per minute
    defaultJobOptions: {
      attempts: 3,
      backoff: { type: "exponential", delay: 30_000 },
      removeOnComplete: { count: 1000 },
      removeOnFail: { count: 5000 },
    },
  },
);

async function publishToTwitter(
  accessToken: string,
  variant: typeof postVariants.$inferSelect,
): Promise<{ externalPostId: string; externalPostUrl: string }> {
  const client = new TwitterApi(accessToken);

  // Build post text with hashtags
  const hashtagText = variant.hashtags?.length
    ? "\n\n" + variant.hashtags.map((h) => `#${h}`).join(" ")
    : "";

  const fullText = variant.text + hashtagText;

  // Handle threads
  if (variant.threadTexts?.length) {
    const tweets = [fullText, ...variant.threadTexts];
    let lastTweetId: string | undefined;

    for (const tweetText of tweets) {
      const params: Record<string, unknown> = { text: tweetText };
      if (lastTweetId) {
        params.reply = { in_reply_to_tweet_id: lastTweetId };
      }

      const result = await client.v2.tweet(params as never);
      lastTweetId = result.data.id;
    }

    return {
      externalPostId: lastTweetId!,
      externalPostUrl: `https://twitter.com/i/status/${lastTweetId}`,
    };
  }

  // Single tweet
  const result = await client.v2.tweet({ text: fullText });
  return {
    externalPostId: result.data.id,
    externalPostUrl: `https://twitter.com/i/status/${result.data.id}`,
  };
}

async function publishToLinkedIn(
  accessToken: string,
  linkedInPersonId: string,
  variant: typeof postVariants.$inferSelect,
): Promise<{ externalPostId: string; externalPostUrl: string }> {
  const hashtagText = variant.hashtags?.length
    ? "\n\n" + variant.hashtags.map((h) => `#${h}`).join(" ")
    : "";

  const fullText = variant.text + hashtagText;

  const response = await fetch("https://api.linkedin.com/v2/ugcPosts", {
    method: "POST",
    headers: {
      Authorization: `Bearer ${accessToken}`,
      "Content-Type": "application/json",
      "X-Restli-Protocol-Version": "2.0.0",
    },
    body: JSON.stringify({
      author: `urn:li:person:${linkedInPersonId}`,
      lifecycleState: "PUBLISHED",
      specificContent: {
        "com.linkedin.ugc.ShareContent": {
          shareCommentary: { text: fullText },
          shareMediaCategory: "NONE",
        },
      },
      visibility: {
        "com.linkedin.ugc.MemberNetworkVisibility": "PUBLIC",
      },
    }),
  });

  if (!response.ok) {
    const error = await response.text();
    throw new Error(`LinkedIn API error: ${response.status} — ${error}`);
  }

  const data = await response.json();
  const postUrn = data.id; // urn:li:share:12345
  const shareId = postUrn.split(":").pop();

  return {
    externalPostId: postUrn,
    externalPostUrl: `https://www.linkedin.com/feed/update/${postUrn}`,
  };
}

async function ensureFreshToken(
  account: typeof socialAccounts.$inferSelect,
): Promise<string> {
  const accessToken = decrypt(
    account.accessTokenEncrypted,
    process.env.INTEGRATION_ENCRYPTION_KEY!,
  );

  // Check if token expires within 5 minutes
  if (
    account.tokenExpiresAt &&
    account.tokenExpiresAt.getTime() - Date.now() < 5 * 60 * 1000
  ) {
    if (!account.refreshTokenEncrypted) {
      // No refresh token — mark as expired
      await db
        .update(socialAccounts)
        .set({ status: "expired", errorMessage: "Token expired, reconnection required" })
        .where(eq(socialAccounts.id, account.id));
      throw new Error("Token expired and no refresh token available");
    }

    const refreshToken = decrypt(
      account.refreshTokenEncrypted,
      process.env.INTEGRATION_ENCRYPTION_KEY!,
    );

    // Platform-specific refresh
    const newTokens = await refreshAccessToken(
      account.platform,
      refreshToken,
    );

    // Update stored tokens
    const { encrypt } = await import("../integrations/encryption");
    await db
      .update(socialAccounts)
      .set({
        accessTokenEncrypted: encrypt(
          newTokens.accessToken,
          process.env.INTEGRATION_ENCRYPTION_KEY!,
        ),
        refreshTokenEncrypted: newTokens.refreshToken
          ? encrypt(newTokens.refreshToken, process.env.INTEGRATION_ENCRYPTION_KEY!)
          : account.refreshTokenEncrypted,
        tokenExpiresAt: newTokens.expiresAt,
        status: "connected",
        errorMessage: null,
        lastSyncedAt: new Date(),
      })
      .where(eq(socialAccounts.id, account.id));

    return newTokens.accessToken;
  }

  return accessToken;
}

async function refreshAccessToken(
  platform: string,
  refreshToken: string,
): Promise<{ accessToken: string; refreshToken?: string; expiresAt: Date }> {
  switch (platform) {
    case "twitter": {
      const response = await fetch("https://api.twitter.com/2/oauth2/token", {
        method: "POST",
        headers: { "Content-Type": "application/x-www-form-urlencoded" },
        body: new URLSearchParams({
          grant_type: "refresh_token",
          refresh_token: refreshToken,
          client_id: process.env.TWITTER_CLIENT_ID!,
        }),
      });
      const data = await response.json();
      return {
        accessToken: data.access_token,
        refreshToken: data.refresh_token,
        expiresAt: new Date(Date.now() + data.expires_in * 1000),
      };
    }
    case "linkedin": {
      const response = await fetch(
        "https://www.linkedin.com/oauth/v2/accessToken",
        {
          method: "POST",
          headers: { "Content-Type": "application/x-www-form-urlencoded" },
          body: new URLSearchParams({
            grant_type: "refresh_token",
            refresh_token: refreshToken,
            client_id: process.env.LINKEDIN_CLIENT_ID!,
            client_secret: process.env.LINKEDIN_CLIENT_SECRET!,
          }),
        },
      );
      const data = await response.json();
      return {
        accessToken: data.access_token,
        refreshToken: data.refresh_token,
        expiresAt: new Date(Date.now() + data.expires_in * 1000),
      };
    }
    default:
      throw new Error(`Token refresh not implemented for ${platform}`);
  }
}
```

### 3. Content Calendar UI with Drag-and-Drop

The calendar view displays posts in a monthly grid with platform-colored cards. Users can drag posts between dates to reschedule, and click to edit. The calendar integrates with the content generation system to show gaps and suggest fills.

```typescript
// src/components/calendar/content-calendar.tsx

"use client";

import { useState, useMemo, useCallback } from "react";
import {
  DndContext,
  DragOverlay,
  type DragStartEvent,
  type DragEndEvent,
  pointerWithin,
} from "@dnd-kit/core";
import {
  startOfMonth,
  endOfMonth,
  eachDayOfInterval,
  startOfWeek,
  endOfWeek,
  format,
  isSameMonth,
  isSameDay,
  isToday,
  parseISO,
} from "date-fns";
import { api } from "@/lib/trpc/client";

interface CalendarPost {
  id: string;
  status: string;
  contentPillar: string | null;
  contentType: string | null;
  scheduledAt: string; // ISO datetime
  calendarOrder: number;
  variants: {
    id: string;
    platform: string;
    text: string;
    hashtags: string[];
    externalPostUrl: string | null;
  }[];
}

interface ContentCalendarProps {
  brandId: string;
  initialMonth: Date;
  posts: CalendarPost[];
}

const PLATFORM_COLORS: Record<string, string> = {
  twitter: "bg-sky-100 border-sky-300 text-sky-800",
  linkedin: "bg-blue-100 border-blue-300 text-blue-800",
  instagram: "bg-pink-100 border-pink-300 text-pink-800",
  facebook: "bg-indigo-100 border-indigo-300 text-indigo-800",
};

const STATUS_INDICATORS: Record<string, string> = {
  draft: "bg-gray-400",
  approved: "bg-green-400",
  scheduled: "bg-blue-400",
  published: "bg-emerald-500",
  failed: "bg-red-500",
};

export function ContentCalendar({
  brandId,
  initialMonth,
  posts,
}: ContentCalendarProps) {
  const [currentMonth, setCurrentMonth] = useState(initialMonth);
  const [draggedPost, setDraggedPost] = useState<CalendarPost | null>(null);
  const [view, setView] = useState<"month" | "week" | "list">("month");
  const [platformFilter, setPlatformFilter] = useState<string | null>(null);
  const [statusFilter, setStatusFilter] = useState<string | null>(null);

  const reschedulePost = api.post.reschedule.useMutation();

  // Generate calendar grid (6 weeks × 7 days)
  const calendarDays = useMemo(() => {
    const monthStart = startOfMonth(currentMonth);
    const monthEnd = endOfMonth(currentMonth);
    const calendarStart = startOfWeek(monthStart, { weekStartsOn: 0 });
    const calendarEnd = endOfWeek(monthEnd, { weekStartsOn: 0 });

    return eachDayOfInterval({ start: calendarStart, end: calendarEnd });
  }, [currentMonth]);

  // Group posts by date
  const postsByDate = useMemo(() => {
    const grouped = new Map<string, CalendarPost[]>();

    for (const post of posts) {
      if (!post.scheduledAt) continue;

      // Apply filters
      if (platformFilter) {
        const hasPlatform = post.variants.some(
          (v) => v.platform === platformFilter,
        );
        if (!hasPlatform) continue;
      }
      if (statusFilter && post.status !== statusFilter) continue;

      const dateKey = format(parseISO(post.scheduledAt), "yyyy-MM-dd");
      const existing = grouped.get(dateKey) || [];
      grouped.set(dateKey, [...existing, post].sort(
        (a, b) => a.calendarOrder - b.calendarOrder,
      ));
    }

    return grouped;
  }, [posts, platformFilter, statusFilter]);

  // Drag handlers
  const handleDragStart = (event: DragStartEvent) => {
    const post = posts.find((p) => p.id === event.active.id);
    setDraggedPost(post || null);
  };

  const handleDragEnd = useCallback(
    (event: DragEndEvent) => {
      setDraggedPost(null);
      const { active, over } = event;
      if (!over) return;

      const postId = active.id as string;
      const targetDate = over.id as string; // "yyyy-MM-dd"

      const post = posts.find((p) => p.id === postId);
      if (!post) return;

      const currentDate = format(parseISO(post.scheduledAt), "yyyy-MM-dd");
      if (currentDate === targetDate) return;

      // Preserve time, change date
      const currentTime = format(parseISO(post.scheduledAt), "HH:mm:ss");
      const newScheduledAt = `${targetDate}T${currentTime}`;

      reschedulePost.mutate({
        postId,
        scheduledAt: newScheduledAt,
      });
    },
    [posts, reschedulePost],
  );

  // Gap detection
  const gapDays = useMemo(() => {
    const gaps: string[] = [];
    const monthStart = startOfMonth(currentMonth);
    const monthEnd = endOfMonth(currentMonth);
    const daysInMonth = eachDayOfInterval({ start: monthStart, end: monthEnd });

    for (const day of daysInMonth) {
      const dateKey = format(day, "yyyy-MM-dd");
      const dayPosts = postsByDate.get(dateKey);
      if (!dayPosts || dayPosts.length === 0) {
        // Only flag weekdays as gaps
        const dayOfWeek = day.getDay();
        if (dayOfWeek !== 0 && dayOfWeek !== 6) {
          gaps.push(dateKey);
        }
      }
    }

    return new Set(gaps);
  }, [currentMonth, postsByDate]);

  return (
    <div className="flex flex-col h-full">
      {/* Toolbar */}
      <div className="flex items-center justify-between px-4 py-3 border-b">
        <div className="flex items-center gap-2">
          <button
            onClick={() =>
              setCurrentMonth(
                new Date(currentMonth.getFullYear(), currentMonth.getMonth() - 1),
              )
            }
            className="p-1 hover:bg-gray-100 rounded"
          >
            ←
          </button>
          <h2 className="text-lg font-semibold">
            {format(currentMonth, "MMMM yyyy")}
          </h2>
          <button
            onClick={() =>
              setCurrentMonth(
                new Date(currentMonth.getFullYear(), currentMonth.getMonth() + 1),
              )
            }
            className="p-1 hover:bg-gray-100 rounded"
          >
            →
          </button>
        </div>

        <div className="flex items-center gap-4">
          {/* Platform filter */}
          <select
            value={platformFilter || ""}
            onChange={(e) =>
              setPlatformFilter(e.target.value || null)
            }
            className="text-sm border rounded px-2 py-1"
          >
            <option value="">All platforms</option>
            <option value="twitter">Twitter/X</option>
            <option value="linkedin">LinkedIn</option>
          </select>

          {/* Status filter */}
          <select
            value={statusFilter || ""}
            onChange={(e) =>
              setStatusFilter(e.target.value || null)
            }
            className="text-sm border rounded px-2 py-1"
          >
            <option value="">All statuses</option>
            <option value="draft">Draft</option>
            <option value="approved">Approved</option>
            <option value="scheduled">Scheduled</option>
            <option value="published">Published</option>
          </select>

          {/* View toggle */}
          <div className="flex border rounded overflow-hidden">
            {(["month", "week", "list"] as const).map((v) => (
              <button
                key={v}
                onClick={() => setView(v)}
                className={`px-3 py-1 text-sm ${
                  view === v ? "bg-gray-900 text-white" : "hover:bg-gray-100"
                }`}
              >
                {v.charAt(0).toUpperCase() + v.slice(1)}
              </button>
            ))}
          </div>
        </div>
      </div>

      {/* Calendar grid */}
      <DndContext
        collisionDetection={pointerWithin}
        onDragStart={handleDragStart}
        onDragEnd={handleDragEnd}
      >
        <div className="grid grid-cols-7 flex-1">
          {/* Day headers */}
          {["Sun", "Mon", "Tue", "Wed", "Thu", "Fri", "Sat"].map((day) => (
            <div
              key={day}
              className="px-2 py-1 text-xs font-medium text-gray-500 text-center border-b"
            >
              {day}
            </div>
          ))}

          {/* Day cells */}
          {calendarDays.map((day) => {
            const dateKey = format(day, "yyyy-MM-dd");
            const dayPosts = postsByDate.get(dateKey) || [];
            const isCurrentMonth = isSameMonth(day, currentMonth);
            const isGap = gapDays.has(dateKey);

            return (
              <CalendarDayCell
                key={dateKey}
                date={day}
                dateKey={dateKey}
                posts={dayPosts}
                isCurrentMonth={isCurrentMonth}
                isToday={isToday(day)}
                isGap={isGap}
              />
            );
          })}
        </div>

        <DragOverlay>
          {draggedPost && (
            <PostCard post={draggedPost} isDragging />
          )}
        </DragOverlay>
      </DndContext>
    </div>
  );
}

function CalendarDayCell({
  date,
  dateKey,
  posts,
  isCurrentMonth,
  isToday: isTodayFlag,
  isGap,
}: {
  date: Date;
  dateKey: string;
  posts: CalendarPost[];
  isCurrentMonth: boolean;
  isToday: boolean;
  isGap: boolean;
}) {
  return (
    <div
      id={dateKey}
      className={`border-b border-r p-1 min-h-[120px] ${
        isCurrentMonth ? "bg-white" : "bg-gray-50"
      } ${isGap ? "bg-amber-50" : ""}`}
    >
      <div className="flex items-center justify-between mb-1">
        <span
          className={`text-xs ${
            isTodayFlag
              ? "bg-blue-600 text-white w-5 h-5 rounded-full flex items-center justify-center"
              : isCurrentMonth
                ? "text-gray-700"
                : "text-gray-400"
          }`}
        >
          {format(date, "d")}
        </span>
        {isGap && (
          <span className="text-xs text-amber-600">Gap</span>
        )}
      </div>

      <div className="space-y-1">
        {posts.slice(0, 3).map((post) => (
          <PostCard key={post.id} post={post} />
        ))}
        {posts.length > 3 && (
          <span className="text-xs text-gray-500">
            +{posts.length - 3} more
          </span>
        )}
      </div>
    </div>
  );
}

function PostCard({
  post,
  isDragging,
}: {
  post: CalendarPost;
  isDragging?: boolean;
}) {
  const primaryVariant = post.variants[0];
  const platforms = post.variants.map((v) => v.platform);

  return (
    <div
      className={`px-2 py-1 rounded border text-xs cursor-pointer
        ${PLATFORM_COLORS[primaryVariant?.platform || "twitter"]}
        ${isDragging ? "shadow-lg opacity-80" : "hover:shadow-sm"}
        transition-shadow`}
    >
      <div className="flex items-center gap-1">
        <span
          className={`w-1.5 h-1.5 rounded-full ${
            STATUS_INDICATORS[post.status] || "bg-gray-400"
          }`}
        />
        <span className="truncate">
          {primaryVariant?.text.slice(0, 40) || "Untitled"}
        </span>
      </div>
      {platforms.length > 1 && (
        <div className="mt-0.5 text-[10px] opacity-60">
          {platforms.join(" + ")}
        </div>
      )}
    </div>
  );
}
```

### 4. Twitter and LinkedIn OAuth 2.0 Integration

Social account connection uses OAuth 2.0 flows for both Twitter (PKCE) and LinkedIn. Tokens are encrypted at rest and automatically refreshed before expiry.

```typescript
// src/app/api/auth/twitter/route.ts

import { NextRequest, NextResponse } from "next/server";
import { auth } from "@clerk/nextjs/server";
import { randomBytes, createHash } from "crypto";
import { cookies } from "next/headers";

export async function GET(req: NextRequest) {
  const { orgId } = await auth();
  if (!orgId) return NextResponse.json({ error: "Unauthorized" }, { status: 401 });

  const brandId = req.nextUrl.searchParams.get("brandId");
  if (!brandId) return NextResponse.json({ error: "Missing brandId" }, { status: 400 });

  // PKCE: generate code verifier + challenge
  const codeVerifier = randomBytes(32).toString("base64url");
  const codeChallenge = createHash("sha256")
    .update(codeVerifier)
    .digest("base64url");

  const state = randomBytes(16).toString("hex");

  // Store in cookies for callback verification
  const cookieStore = await cookies();
  cookieStore.set("twitter_code_verifier", codeVerifier, {
    httpOnly: true,
    secure: true,
    sameSite: "lax",
    maxAge: 600, // 10 minutes
  });
  cookieStore.set("twitter_state", state, {
    httpOnly: true,
    secure: true,
    sameSite: "lax",
    maxAge: 600,
  });
  cookieStore.set("twitter_brand_id", brandId, {
    httpOnly: true,
    secure: true,
    sameSite: "lax",
    maxAge: 600,
  });

  const params = new URLSearchParams({
    response_type: "code",
    client_id: process.env.TWITTER_CLIENT_ID!,
    redirect_uri: `${process.env.NEXT_PUBLIC_APP_URL}/api/auth/twitter/callback`,
    scope: "tweet.read tweet.write users.read offline.access",
    state,
    code_challenge: codeChallenge,
    code_challenge_method: "S256",
  });

  return NextResponse.redirect(
    `https://twitter.com/i/oauth2/authorize?${params.toString()}`,
  );
}

// src/app/api/auth/twitter/callback/route.ts

import { NextRequest, NextResponse } from "next/server";
import { auth } from "@clerk/nextjs/server";
import { cookies } from "next/headers";
import { db } from "@/server/db";
import { socialAccounts, brands } from "@/server/db/schema";
import { encrypt } from "@/server/integrations/encryption";
import { eq } from "drizzle-orm";

export async function GET(req: NextRequest) {
  const { orgId } = await auth();
  if (!orgId) return NextResponse.redirect(`${process.env.NEXT_PUBLIC_APP_URL}/dashboard?error=unauthorized`);

  const code = req.nextUrl.searchParams.get("code");
  const state = req.nextUrl.searchParams.get("state");

  const cookieStore = await cookies();
  const storedState = cookieStore.get("twitter_state")?.value;
  const codeVerifier = cookieStore.get("twitter_code_verifier")?.value;
  const brandId = cookieStore.get("twitter_brand_id")?.value;

  if (!code || !state || state !== storedState || !codeVerifier || !brandId) {
    return NextResponse.redirect(
      `${process.env.NEXT_PUBLIC_APP_URL}/dashboard?error=invalid_callback`,
    );
  }

  // Exchange code for tokens
  const tokenResponse = await fetch("https://api.twitter.com/2/oauth2/token", {
    method: "POST",
    headers: { "Content-Type": "application/x-www-form-urlencoded" },
    body: new URLSearchParams({
      code,
      grant_type: "authorization_code",
      client_id: process.env.TWITTER_CLIENT_ID!,
      redirect_uri: `${process.env.NEXT_PUBLIC_APP_URL}/api/auth/twitter/callback`,
      code_verifier: codeVerifier,
    }),
  });

  if (!tokenResponse.ok) {
    return NextResponse.redirect(
      `${process.env.NEXT_PUBLIC_APP_URL}/dashboard?error=token_exchange_failed`,
    );
  }

  const tokens = await tokenResponse.json();

  // Get user info
  const userResponse = await fetch("https://api.twitter.com/2/users/me", {
    headers: { Authorization: `Bearer ${tokens.access_token}` },
  });

  const userData = await userResponse.json();
  const user = userData.data;

  // Look up the brand to get orgId
  const [brand] = await db
    .select()
    .from(brands)
    .where(eq(brands.id, brandId))
    .limit(1);

  if (!brand) {
    return NextResponse.redirect(
      `${process.env.NEXT_PUBLIC_APP_URL}/dashboard?error=brand_not_found`,
    );
  }

  const encryptionKey = process.env.INTEGRATION_ENCRYPTION_KEY!;

  // Upsert social account
  await db
    .insert(socialAccounts)
    .values({
      brandId,
      orgId: brand.orgId,
      platform: "twitter",
      accountName: `@${user.username}`,
      accountId: user.id,
      accessTokenEncrypted: encrypt(tokens.access_token, encryptionKey),
      refreshTokenEncrypted: tokens.refresh_token
        ? encrypt(tokens.refresh_token, encryptionKey)
        : null,
      tokenExpiresAt: new Date(Date.now() + tokens.expires_in * 1000),
      profileImageUrl: user.profile_image_url,
      status: "connected",
    })
    .onConflictDoUpdate({
      target: [socialAccounts.platform, socialAccounts.accountId],
      set: {
        accessTokenEncrypted: encrypt(tokens.access_token, encryptionKey),
        refreshTokenEncrypted: tokens.refresh_token
          ? encrypt(tokens.refresh_token, encryptionKey)
          : undefined,
        tokenExpiresAt: new Date(Date.now() + tokens.expires_in * 1000),
        status: "connected",
        errorMessage: null,
        lastSyncedAt: new Date(),
      },
    });

  // Clean up cookies
  cookieStore.delete("twitter_code_verifier");
  cookieStore.delete("twitter_state");
  cookieStore.delete("twitter_brand_id");

  return NextResponse.redirect(
    `${process.env.NEXT_PUBLIC_APP_URL}/dashboard/brands/${brandId}/accounts?success=twitter`,
  );
}

// src/server/integrations/encryption.ts

import { createCipheriv, createDecipheriv, randomBytes } from "crypto";

const ALGORITHM = "aes-256-gcm";
const IV_LENGTH = 12;
const TAG_LENGTH = 16;

export function encrypt(plaintext: string, key: string): string {
  const keyBuffer = Buffer.from(key, "hex"); // 32 bytes = 256 bits
  const iv = randomBytes(IV_LENGTH);
  const cipher = createCipheriv(ALGORITHM, keyBuffer, iv);

  let encrypted = cipher.update(plaintext, "utf8", "hex");
  encrypted += cipher.final("hex");
  const authTag = cipher.getAuthTag();

  // Format: iv:encrypted:authTag (all hex)
  return `${iv.toString("hex")}:${encrypted}:${authTag.toString("hex")}`;
}

export function decrypt(ciphertext: string, key: string): string {
  const [ivHex, encryptedHex, authTagHex] = ciphertext.split(":");
  const keyBuffer = Buffer.from(key, "hex");
  const iv = Buffer.from(ivHex, "hex");
  const authTag = Buffer.from(authTagHex, "hex");

  const decipher = createDecipheriv(ALGORITHM, keyBuffer, iv);
  decipher.setAuthTag(authTag);

  let decrypted = decipher.update(encryptedHex, "hex", "utf8");
  decrypted += decipher.final("utf8");

  return decrypted;
}
```

---

## Phase Breakdown

### Phase 1: Project Scaffold + Database (Days 1–5)

**Day 1 — Initialize project**
```bash
npx create-next-app@latest contentcal --typescript --tailwind --eslint --app --src-dir
cd contentcal
npm install drizzle-orm @neondatabase/serverless
npm install -D drizzle-kit
npm install @trpc/server @trpc/client @trpc/react-query @trpc/next superjson
npm install @clerk/nextjs
npm install zod date-fns
```
- Configure `tsconfig.json`: strict mode, path aliases
- Create `src/env.ts` with Zod-validated environment variables:
  - `DATABASE_URL`, `CLERK_SECRET_KEY`, `NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY`
  - `OPENAI_API_KEY`, `STRIPE_SECRET_KEY`, `STRIPE_WEBHOOK_SECRET`
  - `TWITTER_CLIENT_ID`, `TWITTER_CLIENT_SECRET`
  - `LINKEDIN_CLIENT_ID`, `LINKEDIN_CLIENT_SECRET`
  - `R2_ACCESS_KEY_ID`, `R2_SECRET_ACCESS_KEY`, `R2_BUCKET_NAME`, `R2_ENDPOINT`
  - `UPSTASH_REDIS_HOST`, `UPSTASH_REDIS_PASSWORD`
  - `INTEGRATION_ENCRYPTION_KEY`, `RESEND_API_KEY`

**Day 2 — Database schema + connection**
- Create `src/server/db/schema.ts` with all tables: `organizations`, `members`, `brands`, `socialAccounts`, `posts`, `postVariants`, `postMetrics`, `mediaLibrary`, `generationJobs`, `publishingLog`
- Create `src/server/db/index.ts`: Neon serverless client
- Create `drizzle.config.ts`
- Run `npx drizzle-kit push`

**Day 3 — tRPC setup + Clerk auth**
- Create `src/server/trpc/trpc.ts`:
  - `publicProcedure`, `protectedProcedure`, `orgProcedure`
  - Plan limit middleware
- Create `src/server/trpc/context.ts`
- Create `src/server/trpc/routers/_app.ts`
- Create `src/app/api/trpc/[trpc]/route.ts`
- Create `src/lib/trpc/client.ts` + `server.ts`

**Day 4 — Clerk middleware + app shell**
- Create `src/middleware.ts`: protect dashboard routes
- Create `src/app/layout.tsx` with ClerkProvider
- Create `src/app/dashboard/layout.tsx`: sidebar (brands, calendar, analytics, settings) + header
- Install Radix UI:
  ```bash
  npm install @radix-ui/react-dialog @radix-ui/react-dropdown-menu @radix-ui/react-tabs
  npm install @radix-ui/react-select @radix-ui/react-tooltip @radix-ui/react-popover
  npm install @radix-ui/react-toast @radix-ui/react-switch
  ```
- Create shared UI components in `src/components/ui/`

**Day 5 — Organization + member sync**
- Create `src/server/trpc/routers/organization.ts`
- Create Clerk webhook handler `src/app/api/webhooks/clerk/route.ts`
- Test auth flow: sign up → create org → verify DB

---

### Phase 2: Brand Profile + Content Strategy (Days 6–10)

**Day 6 — Brand CRUD router**
- Create `src/server/trpc/routers/brand.ts`:
  - `list`: all brands for org
  - `getById`: single brand with social account count
  - `create`: create brand from onboarding wizard data
  - `update`: update brand profile, voice, pillars, topics
  - `delete`: cascade delete brand + posts + social accounts
- Plan limits: free 1 brand, pro 1 brand, business 5 brands, agency unlimited

**Day 7 — Brand onboarding wizard**
- Create `src/app/dashboard/brands/new/page.tsx`:
  - Step 1: Business name + industry (dropdown with 20+ industries)
  - Step 2: Target audience description (textarea)
  - Step 3: Brand voice selector (5 presets with descriptions + custom textarea)
  - Step 4: Content pillars (suggest 5 based on industry, user selects 3-5)
  - Step 5: Topics / expertise keywords (tag input)
  - Step 6: Platform selection (Twitter, LinkedIn checkboxes)
  - Step 7: Posting frequency per platform (slider: 1-7 posts/week)
  - Progress bar across all steps

**Day 8 — AI content strategy generation**
- Create `src/server/ai/generate-strategy.ts`:
  - Given brand profile → generate content strategy document
  - Output: recommended pillars, posting schedule, content mix ratios
  - Store strategy preview in onboarding for user review
- Add "Generate content strategy" step to onboarding wizard:
  - AI generates strategy → user reviews → "Looks good" or "Adjust"

**Day 9 — Brand settings page**
- Create `src/app/dashboard/brands/[brandId]/page.tsx`:
  - Edit all brand profile fields
  - Social account connections list (connect/disconnect)
  - Content pillar management (add/remove/reorder)
  - Brand voice preview (show example post in current voice)
- Create `src/app/dashboard/brands/[brandId]/accounts/page.tsx`:
  - Connected accounts list with status badges
  - "Connect Twitter" / "Connect LinkedIn" buttons
  - Disconnect button with confirmation

**Day 10 — Social account management**
- Create `src/server/trpc/routers/socialAccount.ts`:
  - `list`: all accounts for a brand
  - `getById`: single account with status
  - `disconnect`: revoke token + delete record
  - `testConnection`: verify token is still valid
- Implement encryption module `src/server/integrations/encryption.ts`:
  - AES-256-GCM encrypt/decrypt for OAuth tokens

---

### Phase 3: AI Content Generation (Days 11–17)

**Day 11 — Content generation engine**
- Create `src/server/ai/generate-content.ts`:
  - `generateMonthlyContent(config)`: two-stage pipeline
    - Stage 1: calendar plan (topics, dates, pillars)
    - Stage 2: platform copy for each post (batched 5 at a time)
  - `regeneratePost(...)`: single post regeneration with optional feedback

**Day 12 — Generation queue + worker**
```bash
npm install bullmq ioredis
```
- Create `src/server/queue/connection.ts`: Redis connection config
- Create `src/server/queue/generation-queue.ts`: BullMQ queue for content generation
- Create `src/server/queue/generation-worker.ts`:
  - Process generation job: call AI → create posts + variants in DB
  - Update `generationJobs` status as progress changes
  - Handle errors with retry (3 attempts)

**Day 13 — Generation tRPC router + UI**
- Create `src/server/trpc/routers/generation.ts`:
  - `startGeneration`: enqueue generation job, return batchId
  - `getJobStatus`: poll generation job status + progress
  - `cancelJob`: cancel in-progress generation
- Create `src/app/dashboard/brands/[brandId]/generate/page.tsx`:
  - Configuration form:
    - Date range picker (default: next 30 days)
    - Number of posts (slider: 10-60)
    - Content pillars to include (checkboxes)
    - Specific topics (optional textarea)
    - Platforms (checkboxes, pre-selected from brand config)
  - "Generate" button → loading state with progress bar
  - Preview generated posts in scrollable list
  - "Add all to calendar" or cherry-pick individual posts

**Day 14 — Post CRUD router**
- Create `src/server/trpc/routers/post.ts`:
  - `list`: paginated posts for brand, filter by status/platform/date range
  - `getById`: single post with all variants
  - `create`: manual post creation
  - `update`: update post metadata
  - `updateVariant`: update platform-specific content
  - `reschedule`: change scheduledAt (drag-and-drop calendar)
  - `approve`: change status to approved
  - `schedule`: change status to scheduled, create BullMQ delayed job
  - `unschedule`: cancel scheduled job
  - `delete`: delete post + variants
  - `duplicate`: copy post with new scheduled date
  - `regenerate`: AI regeneration of a single post

**Day 15 — Hashtag suggestion engine**
- Create `src/server/ai/hashtags.ts`:
  - `suggestHashtags(text, platform, industry)`: GPT-4o-mini JSON mode
  - Returns: trending hashtags + evergreen hashtags + niche hashtags
  - Cache suggestions for similar topics (5 minute TTL)
- Integrate with post editor: show hashtag suggestions below text editor
- Allow click-to-add hashtags from suggestions

**Day 16 — Post editor page**
- Create `src/app/dashboard/brands/[brandId]/posts/[postId]/page.tsx`:
  - Split view: editor on left, platform preview on right
  - Platform variant tabs (Twitter | LinkedIn)
  - Text editor per variant with character count
  - Hashtag suggestions panel
  - Schedule picker: date + time with "Best time" suggestions
  - Status bar: Draft → Approved → Scheduled → Published
  - "Approve" / "Schedule" / "Publish Now" buttons
  - "Regenerate" button with optional feedback text input

**Day 17 — Platform preview components**
- Create `src/components/post-editor/twitter-preview.tsx`:
  - Mock Twitter card: profile image, name, handle, text, hashtags, timestamp
  - Character count with color indicator (green < 250, yellow 250-270, red > 270)
  - Thread preview: show thread continuation indicator
- Create `src/components/post-editor/linkedin-preview.tsx`:
  - Mock LinkedIn post card: profile, name, headline, text, hashtags
  - "See more" truncation for long posts
  - Engagement action bar (like, comment, repost, send)

---

### Phase 4: Social Platform Publishing (Days 18–24)

**Day 18 — Twitter OAuth 2.0 PKCE flow**
- Create `src/app/api/auth/twitter/route.ts`: initiate OAuth with PKCE
- Create `src/app/api/auth/twitter/callback/route.ts`: exchange code for tokens
- Store encrypted tokens in `socialAccounts`
- Test: connect Twitter account → verify token stored

**Day 19 — LinkedIn OAuth 2.0 flow**
- Create `src/app/api/auth/linkedin/route.ts`: initiate LinkedIn OAuth
- Create `src/app/api/auth/linkedin/callback/route.ts`: exchange code for tokens
- Request scopes: `w_member_social`, `r_liteprofile`
- Store encrypted tokens in `socialAccounts`
- Test: connect LinkedIn account → verify token stored

**Day 20 — Publishing worker**
- Create `src/server/queue/publish-queue.ts`: BullMQ queue for publishing
- Create `src/server/queue/publish-worker.ts`:
  - `publishToTwitter(token, variant)`: post tweet or thread via Twitter API v2
  - `publishToLinkedIn(token, accountId, variant)`: post via LinkedIn API
  - Token refresh middleware: check expiry, refresh if needed
  - Retry logic: 3 attempts with exponential backoff (30s, 60s, 120s)
  - Update variant with `externalPostId` and `externalPostUrl` on success
  - Log all publish attempts in `publishingLog`

**Day 21 — Scheduling flow**
- When user clicks "Schedule":
  - Validate at least one platform variant exists
  - Validate social account is connected and not expired
  - Create BullMQ delayed job with `delay = scheduledAt - now`
  - Update post status to "scheduled"
- When user clicks "Unschedule":
  - Remove BullMQ job by ID
  - Revert post status to "approved" or "draft"
- When user clicks "Publish Now":
  - Create BullMQ job with delay = 0
  - Immediately publish to all platforms

**Day 22 — Publishing queue dashboard**
- Create `src/app/dashboard/brands/[brandId]/queue/page.tsx`:
  - Upcoming scheduled posts in chronological order
  - Status indicators: waiting, publishing, published, failed
  - Cancel button for waiting posts
  - Retry button for failed posts
  - Publishing log: recent publish events with timestamps

**Day 23 — Token refresh + error handling**
- Implement token refresh for Twitter (PKCE refresh_token flow)
- Implement token refresh for LinkedIn (refresh_token flow)
- Handle expired tokens: mark account as "expired", notify user
- Handle API rate limits: respect Twitter 300 tweets/3h, LinkedIn 100 posts/day
- Create reconnection flow: when account is expired, show banner + reconnect button

**Day 24 — Publish status sync**
- After publishing, update post status to "published" when all variants are published
- If any variant fails after retries, mark post as "failed"
- Show publish errors in post editor with retry option
- Create email notification for publish failures:
  - "Your scheduled post failed to publish to Twitter: [error message]"
  - Include retry link

---

### Phase 5: Content Calendar UI (Days 25–29)

**Day 25 — Calendar component**
```bash
npm install @hello-pangea/dnd
```
- Create `src/components/calendar/content-calendar.tsx`:
  - Monthly grid view (6 weeks × 7 days)
  - Day cells showing post cards (platform-colored)
  - Month navigation (prev/next buttons)
  - Today highlight

**Day 26 — Calendar drag-and-drop**
- Implement DnD: drag posts between calendar days to reschedule
- On drop: call `post.reschedule` mutation, preserve original time
- Optimistic update: move card immediately, revert on error
- Prevent dropping on past dates

**Day 27 — Calendar filters + views**
- Platform filter: show only Twitter / LinkedIn / All
- Status filter: Draft / Approved / Scheduled / Published / All
- Content pillar filter
- Gap detection: highlight weekdays with no scheduled posts
- Week view: expanded day columns with more detail
- List view: chronological list with full post preview

**Day 28 — Calendar tRPC router**
- Create `src/server/trpc/routers/calendar.ts`:
  - `getMonth`: all posts for brand + month, with variants
  - `getWeek`: posts for a specific week
  - `getGaps`: dates with no scheduled posts
  - `bulkApprove`: approve all draft posts for a date range
  - `bulkDelete`: delete all posts for a date range

**Day 29 — Calendar actions**
- Click post card → open post editor in slide-over panel
- Right-click context menu: Edit, Duplicate, Delete, Approve, Schedule
- "Generate more" button for gap dates: generate content for empty days
- Bulk actions toolbar: select multiple posts → approve all / delete all
- Keyboard shortcuts: arrow keys to navigate days, Enter to open post

---

### Phase 6: Billing + Settings (Days 30–33)

**Day 30 — Stripe integration**
```bash
npm install stripe
```
- Create `src/server/billing/stripe.ts`:
  - Plan definitions:
    ```typescript
    const PLAN_LIMITS = {
      free: { maxPostsPerMonth: 5, maxPlatforms: 1, maxBrands: 1, aiGenerations: 5 },
      pro: { maxPostsPerMonth: 30, maxPlatforms: 3, maxBrands: 1, aiGenerations: Infinity },
      business: { maxPostsPerMonth: Infinity, maxPlatforms: Infinity, maxBrands: 5, aiGenerations: Infinity },
      agency: { maxPostsPerMonth: Infinity, maxPlatforms: Infinity, maxBrands: Infinity, aiGenerations: Infinity },
    };
    ```
  - Checkout session + portal session helpers

**Day 31 — Stripe webhook + billing UI**
- Create `src/app/api/webhooks/stripe/route.ts`
- Create `src/server/trpc/routers/billing.ts`
- Create `src/app/dashboard/settings/billing/page.tsx`:
  - Current plan with usage meters (posts this month, brands, platforms)
  - Four plan cards with feature comparison
  - Upgrade / Manage buttons

**Day 32 — Plan enforcement**
- Apply limits in tRPC middleware:
  - Post creation: check monthly post count
  - Brand creation: check brand count
  - Social account connection: check platform count
  - AI generation: check daily generation count (free tier)
- Show upgrade prompts when limits approached (>80%)

**Day 33 — Organization settings**
- Create `src/app/dashboard/settings/page.tsx`:
  - Organization name
  - Default timezone (for scheduling)
  - Notification email
- Create `src/app/dashboard/settings/members/page.tsx`:
  - Team member list from Clerk
  - Invite + remove members

---

### Phase 7: Polish + Launch (Days 34–38)

**Day 34 — Onboarding flow**
- Create `src/app/onboarding/page.tsx`:
  - Step 1: Create brand (name + industry)
  - Step 2: Set brand voice + audience
  - Step 3: Connect first social account (Twitter or LinkedIn)
  - Step 4: Generate first batch of content → preview
  - Step 5: "Your content calendar is ready!" → go to calendar
- Redirect new orgs to onboarding
- Skip option at every step

**Day 35 — Loading states, error handling, empty states**
- Skeleton loaders for calendar, post list, brand list
- Empty states:
  - No brands: "Create your first brand to get started"
  - No posts: "Generate your first batch of content"
  - No social accounts: "Connect a social account to start publishing"
  - Calendar empty: "Your calendar is empty — generate some content"
- Toast notifications for all actions
- Error boundary with retry

**Day 36 — Responsive design + accessibility**
- Mobile calendar: switch to list view on small screens
- Mobile post editor: stack editor + preview vertically
- Sidebar: collapsible on mobile, bottom nav
- ARIA labels on all interactive elements
- Keyboard navigation through calendar

**Day 37 — Performance + SEO**
- Optimize calendar rendering: virtualize days outside viewport
- React Query cache settings:
  - Calendar posts: staleTime 30s
  - Brand profile: staleTime 5min
  - Social accounts: staleTime 1min
- Server-side rendering for dashboard pages
- Landing page SEO

**Day 38 — Landing page + final testing**
- Create `src/app/page.tsx`:
  - Hero: "A month of social media content in 2 minutes" with CTA
  - Demo video: AI generating content → calendar → publish
  - Features: AI Generation, Calendar, Multi-Platform Publishing, Brand Voice
  - Pricing table (4 tiers)
  - Footer
- End-to-end test:
  1. Sign up → Create brand → Set voice + pillars
  2. Connect Twitter + LinkedIn (sandbox/test accounts)
  3. Generate 20 posts → verify in calendar
  4. Edit a post → change text → verify auto-saved
  5. Schedule a post → verify BullMQ job created
  6. Publish now → verify tweet/LinkedIn post created
  7. Drag post to different day → verify rescheduled
  8. Upgrade plan → verify limits change
- Deploy to Vercel
- Configure Sentry + Vercel Analytics

---

## Critical Files

```
contentcal/
├── src/
│   ├── env.ts                                    # Zod-validated environment variables
│   ├── middleware.ts                              # Clerk auth middleware
│   │
│   ├── app/
│   │   ├── layout.tsx                            # Root layout with ClerkProvider
│   │   ├── page.tsx                              # Landing page
│   │   ├── pricing/page.tsx                      # Pricing page
│   │   ├── onboarding/page.tsx                   # 5-step onboarding wizard
│   │   │
│   │   ├── dashboard/
│   │   │   ├── layout.tsx                        # App shell: sidebar + header
│   │   │   ├── page.tsx                          # Dashboard overview
│   │   │   ├── brands/
│   │   │   │   ├── page.tsx                      # Brand list
│   │   │   │   ├── new/page.tsx                  # Brand creation wizard
│   │   │   │   └── [brandId]/
│   │   │   │       ├── page.tsx                  # Brand settings
│   │   │   │       ├── accounts/page.tsx         # Social account connections
│   │   │   │       ├── generate/page.tsx         # AI content generation
│   │   │   │       ├── calendar/page.tsx         # Content calendar
│   │   │   │       ├── queue/page.tsx            # Publishing queue
│   │   │   │       └── posts/
│   │   │   │           └── [postId]/page.tsx     # Post editor
│   │   │   └── settings/
│   │   │       ├── page.tsx                      # Organization settings
│   │   │       ├── billing/page.tsx              # Plan + usage
│   │   │       └── members/page.tsx              # Team members
│   │   │
│   │   └── api/
│   │       ├── trpc/[trpc]/route.ts              # tRPC HTTP handler
│   │       ├── auth/
│   │       │   ├── twitter/route.ts              # Twitter OAuth start
│   │       │   ├── twitter/callback/route.ts     # Twitter OAuth callback
│   │       │   ├── linkedin/route.ts             # LinkedIn OAuth start
│   │       │   └── linkedin/callback/route.ts    # LinkedIn OAuth callback
│   │       └── webhooks/
│   │           ├── clerk/route.ts                # Clerk org/member sync
│   │           └── stripe/route.ts               # Stripe webhook handler
│   │
│   ├── server/
│   │   ├── db/
│   │   │   ├── schema.ts                        # Drizzle schema (all tables + relations)
│   │   │   └── index.ts                         # Neon database client
│   │   │
│   │   ├── trpc/
│   │   │   ├── trpc.ts                          # tRPC init, middleware
│   │   │   ├── context.ts                       # Request context
│   │   │   └── routers/
│   │   │       ├── _app.ts                      # Merged router
│   │   │       ├── organization.ts              # Org CRUD
│   │   │       ├── brand.ts                     # Brand CRUD + profile
│   │   │       ├── socialAccount.ts             # Social account management
│   │   │       ├── post.ts                      # Post CRUD + scheduling
│   │   │       ├── calendar.ts                  # Calendar queries
│   │   │       ├── generation.ts                # AI generation job management
│   │   │       └── billing.ts                   # Plan + Stripe
│   │   │
│   │   ├── ai/
│   │   │   ├── generate-content.ts              # Monthly content generation pipeline
│   │   │   ├── generate-strategy.ts             # Content strategy generation
│   │   │   └── hashtags.ts                      # Hashtag suggestion engine
│   │   │
│   │   ├── queue/
│   │   │   ├── connection.ts                    # Redis connection config
│   │   │   ├── generation-queue.ts              # Content generation queue
│   │   │   ├── generation-worker.ts             # Generation worker
│   │   │   ├── publish-queue.ts                 # Publishing queue
│   │   │   └── publish-worker.ts                # Publishing worker (Twitter + LinkedIn)
│   │   │
│   │   ├── integrations/
│   │   │   └── encryption.ts                    # AES-256-GCM encrypt/decrypt
│   │   │
│   │   ├── billing/
│   │   │   └── stripe.ts                        # Plan definitions, checkout, portal
│   │   │
│   │   └── email/
│   │       └── templates.ts                     # Publish failure notifications
│   │
│   ├── components/
│   │   ├── ui/                                  # Shared Radix UI primitives
│   │   │
│   │   ├── layout/
│   │   │   ├── sidebar.tsx                      # Navigation sidebar
│   │   │   ├── header.tsx                       # Page header
│   │   │   └── org-switcher.tsx                 # Clerk org switcher
│   │   │
│   │   ├── calendar/
│   │   │   ├── content-calendar.tsx             # Monthly calendar grid + DnD
│   │   │   ├── calendar-day-cell.tsx            # Day cell with post cards
│   │   │   ├── post-card.tsx                    # Compact post card
│   │   │   ├── calendar-toolbar.tsx             # Filters + view toggle
│   │   │   └── gap-indicator.tsx                # Missing content day marker
│   │   │
│   │   ├── post-editor/
│   │   │   ├── post-editor.tsx                  # Main editor: text + preview split
│   │   │   ├── variant-editor.tsx               # Platform-specific text editor
│   │   │   ├── twitter-preview.tsx              # Mock Twitter card
│   │   │   ├── linkedin-preview.tsx             # Mock LinkedIn post
│   │   │   ├── hashtag-suggestions.tsx          # AI hashtag panel
│   │   │   ├── schedule-picker.tsx              # Date + time + best time hints
│   │   │   └── status-bar.tsx                   # Draft → Published flow
│   │   │
│   │   ├── brand/
│   │   │   ├── brand-wizard.tsx                 # Multi-step brand setup
│   │   │   ├── voice-selector.tsx               # Voice preset cards
│   │   │   ├── pillar-editor.tsx                # Content pillar tag editor
│   │   │   └── account-card.tsx                 # Connected social account card
│   │   │
│   │   └── generation/
│   │       ├── generation-config.tsx            # Generation settings form
│   │       ├── generation-progress.tsx          # Progress bar + status
│   │       └── post-preview-list.tsx            # Review generated posts
│   │
│   └── lib/
│       ├── trpc/
│       │   ├── client.ts                        # React Query tRPC client
│       │   └── server.ts                        # Server-side caller
│       ├── utils.ts                             # cn(), date helpers
│       └── constants.ts                         # Platform colors, content types, voice presets
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

1. **Auth flow**: Sign up → create org → verify DB records
2. **Brand creation**: Complete brand wizard → verify brand stored with all fields
3. **Twitter OAuth**: Connect Twitter → verify encrypted token stored, account name displayed
4. **LinkedIn OAuth**: Connect LinkedIn → verify encrypted token stored
5. **AI strategy generation**: Generate content strategy → verify pillars and schedule recommended
6. **Monthly content generation**: Generate 20 posts → verify all posts + variants created in DB
7. **Post variants**: Verify Twitter variant ≤280 chars, LinkedIn variant 150-300 words
8. **Hashtag suggestions**: Open post editor → verify hashtags generated for post text
9. **Calendar view**: Verify posts displayed on correct dates with platform colors
10. **Calendar drag-and-drop**: Drag post to different day → verify `scheduledAt` updated
11. **Gap detection**: Remove posts from a weekday → verify "Gap" indicator appears
12. **Post editing**: Edit post text → verify auto-saved, character count updated
13. **Schedule post**: Click "Schedule" → verify BullMQ job created with correct delay
14. **Publish now**: Click "Publish Now" → verify tweet/LinkedIn post appears
15. **Thread publishing**: Schedule Twitter thread → verify all tweets posted in sequence
16. **Token refresh**: Simulate expired token → verify automatic refresh on publish
17. **Publish failure**: Simulate API error → verify retry (3 attempts) → mark as failed
18. **Plan limits**: On free tier, create 6th post → verify limit error
19. **Billing upgrade**: Complete Stripe checkout → verify plan updated
20. **Onboarding**: Complete all 5 steps → verify brand, account, and first content batch created

### Key SQL Queries for Verification

```sql
-- Check brand content pillars and voice
SELECT name, industry, voice_description, content_pillars, posting_config
FROM brands
WHERE org_id = 'ORG_ID';

-- Check social account status and token expiry
SELECT platform, account_name, status, token_expires_at,
       CASE WHEN token_expires_at < NOW() THEN 'EXPIRED' ELSE 'VALID' END AS token_status
FROM social_accounts
WHERE org_id = 'ORG_ID';

-- Posts per content pillar distribution
SELECT content_pillar, COUNT(*) AS count,
       COUNT(*) FILTER (WHERE status = 'published') AS published
FROM posts
WHERE brand_id = 'BRAND_ID'
GROUP BY content_pillar;

-- Publishing log (recent activity)
SELECT pl.action, pl.created_at,
       pv.platform, pv.external_post_url,
       pl.details->>'errorMessage' AS error
FROM publishing_log pl
JOIN post_variants pv ON pv.id = pl.post_variant_id
WHERE pl.org_id = 'ORG_ID'
ORDER BY pl.created_at DESC
LIMIT 20;

-- Monthly post count (plan enforcement)
SELECT COUNT(*) FROM posts
WHERE org_id = 'ORG_ID'
  AND created_at >= date_trunc('month', now());

-- Generation job history
SELECT status, posts_generated, config->>'numberOfPosts' AS requested,
       completed_at, error_message
FROM generation_jobs
WHERE brand_id = 'BRAND_ID'
ORDER BY created_at DESC;
```

### Performance Benchmarks

| Operation | Target | Method |
|-----------|--------|--------|
| AI content generation (20 posts) | < 30s | Batched GPT-4o calls (5 per batch) |
| Single post regeneration | < 5s | GPT-4o single call |
| Hashtag suggestion | < 2s | GPT-4o-mini with caching |
| Calendar page load (30 days, 60 posts) | < 500ms | Indexed query + React Query cache |
| Post reschedule (drag-and-drop) | < 200ms | Single UPDATE + optimistic UI |
| Twitter publish | < 3s | Twitter API v2 |
| LinkedIn publish | < 3s | LinkedIn API |
| Token refresh | < 2s | Platform OAuth refresh endpoint |
| Publishing queue check (100 scheduled) | < 300ms | BullMQ getDelayed() |
| Onboarding wizard load | < 400ms | Minimal queries per step |
