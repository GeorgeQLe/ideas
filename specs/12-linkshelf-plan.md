# 12. LinkShelf — Team Bookmark Organizer

## Implementation Plan

**MVP Scope:** Chrome browser extension for one-click saving, web app with bookmark management and collection organization, AI categorization and tagging via GPT-4o-mini, semantic search via pgvector + text-embedding-3-small, collections with nested folder support, basic team workspace with invite-based member access and shared bookmarks, Chrome bookmark HTML import with deduplication, Stripe billing with three tiers (Free / Pro $6/mo / Team $4/user/mo).

---

## Tech Decisions

| Layer | Technology | Notes |
|-------|-----------|-------|
| Framework | Next.js 14+ App Router | `src/` directory, TypeScript strict |
| API | tRPC v11 | superjson transformer, Clerk auth context |
| Database | Neon PostgreSQL + pgvector | `vector(1536)` columns for semantic search |
| ORM | Drizzle ORM | Push-based migrations, typed schema |
| Auth | Clerk | Organization-scoped, `orgId` from session |
| Billing | Stripe | Checkout Sessions + Customer Portal + Webhooks |
| AI — Categorization | OpenAI GPT-4o-mini | JSON mode, temperature 0.1 for classification |
| AI — Embeddings | text-embedding-3-small | 1536 dimensions, cosine similarity |
| Queue | BullMQ + Redis (Upstash) | Async URL metadata extraction + AI processing |
| Storage | Cloudflare R2 | Favicons, screenshots, extracted content |
| Browser Extension | Chrome Manifest V3 | Popup + context menu, service worker background |
| UI | Tailwind CSS, Radix UI, Recharts | |
| Hosting | Vercel (app), Cloudflare (R2) | |
| Monitoring | Sentry, Vercel Analytics | |

### Key Architecture Decisions

1. **pgvector semantic search over keyword-only**: Bookmarks are embedded on save using page title + description + AI summary. Search queries are embedded at query time and matched via cosine similarity. This enables natural language queries like "that article about React server components" to find the right bookmark even without exact keyword matches.

2. **Async URL metadata extraction via BullMQ**: When a bookmark is saved, the API returns immediately. A BullMQ worker fetches the URL to extract title, description, favicon, Open Graph data, and generates AI categorization + embedding. This keeps save operations under 200ms.

3. **Browser extension communicates via REST API with Clerk session tokens**: The Chrome extension uses Clerk's `getToken()` to obtain a session JWT, which is sent as a Bearer token to the Next.js API. No separate auth system needed.

4. **Collection nesting via parent_collection_id (adjacency list)**: Simple recursive queries with a maximum depth of 5 levels. Breadcrumb paths are computed on read. This is simpler than materialized paths for the expected scale (<1000 collections per workspace).

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
  plan: text("plan").default("free").notNull(), // free | pro | team
  stripeCustomerId: text("stripe_customer_id"),
  stripeSubscriptionId: text("stripe_subscription_id"),
  settings: jsonb("settings").default({}).$type<{
    defaultCollectionId?: string;
    brandColor?: string;
    onboardingCompleted?: boolean;
  }>(),
  createdAt: timestamp("created_at", { withTimezone: true }).defaultNow().notNull(),
  updatedAt: timestamp("updated_at", { withTimezone: true }).defaultNow().notNull(),
});

// ---------------------------------------------------------------------------
// Workspaces (personal + team)
// ---------------------------------------------------------------------------
export const workspaces = pgTable("workspaces", {
  id: uuid("id").primaryKey().defaultRandom(),
  orgId: uuid("org_id")
    .references(() => organizations.id, { onDelete: "cascade" })
    .notNull(),
  name: text("name").notNull(),
  type: text("type").default("team").notNull(), // personal | team
  createdByUserId: text("created_by_user_id").notNull(),
  createdAt: timestamp("created_at", { withTimezone: true }).defaultNow().notNull(),
  updatedAt: timestamp("updated_at", { withTimezone: true }).defaultNow().notNull(),
});

// ---------------------------------------------------------------------------
// Workspace Members
// ---------------------------------------------------------------------------
export const workspaceMembers = pgTable(
  "workspace_members",
  {
    id: uuid("id").primaryKey().defaultRandom(),
    workspaceId: uuid("workspace_id")
      .references(() => workspaces.id, { onDelete: "cascade" })
      .notNull(),
    userId: text("user_id").notNull(), // Clerk userId
    role: text("role").default("contributor").notNull(), // viewer | contributor | admin
    joinedAt: timestamp("joined_at", { withTimezone: true }).defaultNow().notNull(),
  },
  (table) => [
    uniqueIndex("workspace_members_unique_idx").on(table.workspaceId, table.userId),
  ]
);

// ---------------------------------------------------------------------------
// Collections
// ---------------------------------------------------------------------------
export const collections = pgTable(
  "collections",
  {
    id: uuid("id").primaryKey().defaultRandom(),
    workspaceId: uuid("workspace_id")
      .references(() => workspaces.id, { onDelete: "cascade" })
      .notNull(),
    parentCollectionId: uuid("parent_collection_id"),
    name: text("name").notNull(),
    description: text("description"),
    icon: text("icon"), // emoji or icon name
    isPinned: boolean("is_pinned").default(false).notNull(),
    sortOrder: integer("sort_order").default(0).notNull(),
    createdByUserId: text("created_by_user_id").notNull(),
    createdAt: timestamp("created_at", { withTimezone: true }).defaultNow().notNull(),
    updatedAt: timestamp("updated_at", { withTimezone: true }).defaultNow().notNull(),
  },
  (table) => [
    index("collections_workspace_id_idx").on(table.workspaceId),
    index("collections_parent_id_idx").on(table.parentCollectionId),
  ]
);

// ---------------------------------------------------------------------------
// Tags
// ---------------------------------------------------------------------------
export const tags = pgTable(
  "tags",
  {
    id: uuid("id").primaryKey().defaultRandom(),
    workspaceId: uuid("workspace_id")
      .references(() => workspaces.id, { onDelete: "cascade" })
      .notNull(),
    name: text("name").notNull(),
    category: text("category"), // language | framework | topic | custom
    usageCount: integer("usage_count").default(0).notNull(),
    createdAt: timestamp("created_at", { withTimezone: true }).defaultNow().notNull(),
  },
  (table) => [
    uniqueIndex("tags_workspace_name_idx").on(table.workspaceId, table.name),
  ]
);

// ---------------------------------------------------------------------------
// Bookmarks
// ---------------------------------------------------------------------------
export const bookmarks = pgTable(
  "bookmarks",
  {
    id: uuid("id").primaryKey().defaultRandom(),
    workspaceId: uuid("workspace_id")
      .references(() => workspaces.id, { onDelete: "cascade" })
      .notNull(),
    collectionId: uuid("collection_id")
      .references(() => collections.id, { onDelete: "set null" }),
    url: text("url").notNull(),
    title: text("title"),
    description: text("description"),
    faviconUrl: text("favicon_url"),
    screenshotUrl: text("screenshot_url"),
    domain: text("domain"), // extracted from URL
    aiSummary: text("ai_summary"),
    aiCategory: text("ai_category"), // documentation | tutorial | tool | article | reference | video | other
    note: text("note"), // user-added note
    isFavorite: boolean("is_favorite").default(false).notNull(),
    isPinned: boolean("is_pinned").default(false).notNull(),
    savedByUserId: text("saved_by_user_id").notNull(),
    source: text("source").default("web").notNull(), // extension | web | api | import | slack
    linkStatus: text("link_status").default("alive").notNull(), // alive | broken | redirect
    lastCheckedAt: timestamp("last_checked_at", { withTimezone: true }),
    viewCount: integer("view_count").default(0).notNull(),
    aiProcessed: boolean("ai_processed").default(false).notNull(),
    // embedding vector(1536) — added via raw SQL migration
    createdAt: timestamp("created_at", { withTimezone: true }).defaultNow().notNull(),
    updatedAt: timestamp("updated_at", { withTimezone: true }).defaultNow().notNull(),
  },
  (table) => [
    index("bookmarks_workspace_id_idx").on(table.workspaceId),
    index("bookmarks_collection_id_idx").on(table.collectionId),
    index("bookmarks_domain_idx").on(table.workspaceId, table.domain),
    index("bookmarks_created_at_idx").on(table.createdAt),
    index("bookmarks_saved_by_idx").on(table.savedByUserId),
  ]
);

// ---------------------------------------------------------------------------
// Bookmark Tags (many-to-many)
// ---------------------------------------------------------------------------
export const bookmarkTags = pgTable(
  "bookmark_tags",
  {
    id: uuid("id").primaryKey().defaultRandom(),
    bookmarkId: uuid("bookmark_id")
      .references(() => bookmarks.id, { onDelete: "cascade" })
      .notNull(),
    tagId: uuid("tag_id")
      .references(() => tags.id, { onDelete: "cascade" })
      .notNull(),
    source: text("source").default("ai").notNull(), // ai | manual
    createdAt: timestamp("created_at", { withTimezone: true }).defaultNow().notNull(),
  },
  (table) => [
    uniqueIndex("bookmark_tags_unique_idx").on(table.bookmarkId, table.tagId),
    index("bookmark_tags_tag_id_idx").on(table.tagId),
  ]
);

// ---------------------------------------------------------------------------
// Activity Feed
// ---------------------------------------------------------------------------
export const activityFeed = pgTable(
  "activity_feed",
  {
    id: uuid("id").primaryKey().defaultRandom(),
    workspaceId: uuid("workspace_id")
      .references(() => workspaces.id, { onDelete: "cascade" })
      .notNull(),
    userId: text("user_id").notNull(),
    type: text("type").notNull(), // bookmark_saved | bookmark_updated | collection_created | member_joined
    bookmarkId: uuid("bookmark_id")
      .references(() => bookmarks.id, { onDelete: "set null" }),
    collectionId: uuid("collection_id")
      .references(() => collections.id, { onDelete: "set null" }),
    metadata: jsonb("metadata").default({}).$type<{
      bookmarkTitle?: string;
      collectionName?: string;
      userName?: string;
    }>(),
    createdAt: timestamp("created_at", { withTimezone: true }).defaultNow().notNull(),
  },
  (table) => [
    index("activity_feed_workspace_idx").on(table.workspaceId),
    index("activity_feed_created_at_idx").on(table.createdAt),
  ]
);

// ---------------------------------------------------------------------------
// Import Jobs
// ---------------------------------------------------------------------------
export const importJobs = pgTable("import_jobs", {
  id: uuid("id").primaryKey().defaultRandom(),
  workspaceId: uuid("workspace_id")
    .references(() => workspaces.id, { onDelete: "cascade" })
    .notNull(),
  userId: text("user_id").notNull(),
  source: text("source").notNull(), // chrome | firefox | pocket | raindrop | csv
  status: text("status").default("pending").notNull(), // pending | processing | completed | failed
  totalItems: integer("total_items").default(0).notNull(),
  processedItems: integer("processed_items").default(0).notNull(),
  duplicateItems: integer("duplicate_items").default(0).notNull(),
  errorMessage: text("error_message"),
  createdAt: timestamp("created_at", { withTimezone: true }).defaultNow().notNull(),
  completedAt: timestamp("completed_at", { withTimezone: true }),
});

// ---------------------------------------------------------------------------
// Digest Configs
// ---------------------------------------------------------------------------
export const digestConfigs = pgTable("digest_configs", {
  id: uuid("id").primaryKey().defaultRandom(),
  userId: text("user_id").notNull(),
  workspaceId: uuid("workspace_id")
    .references(() => workspaces.id, { onDelete: "cascade" })
    .notNull(),
  frequency: text("frequency").default("weekly").notNull(), // daily | weekly | off
  preferences: jsonb("preferences").default({}).$type<{
    includeTeamActivity?: boolean;
    includeTrending?: boolean;
  }>(),
  createdAt: timestamp("created_at", { withTimezone: true }).defaultNow().notNull(),
});

// ---------------------------------------------------------------------------
// Relations
// ---------------------------------------------------------------------------
export const organizationsRelations = relations(organizations, ({ many }) => ({
  workspaces: many(workspaces),
}));

export const workspacesRelations = relations(workspaces, ({ one, many }) => ({
  organization: one(organizations, {
    fields: [workspaces.orgId],
    references: [organizations.id],
  }),
  members: many(workspaceMembers),
  collections: many(collections),
  bookmarks: many(bookmarks),
  tags: many(tags),
  activityFeed: many(activityFeed),
  importJobs: many(importJobs),
  digestConfigs: many(digestConfigs),
}));

export const workspaceMembersRelations = relations(workspaceMembers, ({ one }) => ({
  workspace: one(workspaces, {
    fields: [workspaceMembers.workspaceId],
    references: [workspaces.id],
  }),
}));

export const collectionsRelations = relations(collections, ({ one, many }) => ({
  workspace: one(workspaces, {
    fields: [collections.workspaceId],
    references: [workspaces.id],
  }),
  parentCollection: one(collections, {
    fields: [collections.parentCollectionId],
    references: [collections.id],
    relationName: "childCollections",
  }),
  childCollections: many(collections, { relationName: "childCollections" }),
  bookmarks: many(bookmarks),
}));

export const tagsRelations = relations(tags, ({ one, many }) => ({
  workspace: one(workspaces, {
    fields: [tags.workspaceId],
    references: [workspaces.id],
  }),
  bookmarkTags: many(bookmarkTags),
}));

export const bookmarksRelations = relations(bookmarks, ({ one, many }) => ({
  workspace: one(workspaces, {
    fields: [bookmarks.workspaceId],
    references: [workspaces.id],
  }),
  collection: one(collections, {
    fields: [bookmarks.collectionId],
    references: [collections.id],
  }),
  bookmarkTags: many(bookmarkTags),
  activityFeed: many(activityFeed),
}));

export const bookmarkTagsRelations = relations(bookmarkTags, ({ one }) => ({
  bookmark: one(bookmarks, {
    fields: [bookmarkTags.bookmarkId],
    references: [bookmarks.id],
  }),
  tag: one(tags, {
    fields: [bookmarkTags.tagId],
    references: [tags.id],
  }),
}));

export const activityFeedRelations = relations(activityFeed, ({ one }) => ({
  workspace: one(workspaces, {
    fields: [activityFeed.workspaceId],
    references: [workspaces.id],
  }),
  bookmark: one(bookmarks, {
    fields: [activityFeed.bookmarkId],
    references: [bookmarks.id],
  }),
  collection: one(collections, {
    fields: [activityFeed.collectionId],
    references: [collections.id],
  }),
}));

export const importJobsRelations = relations(importJobs, ({ one }) => ({
  workspace: one(workspaces, {
    fields: [importJobs.workspaceId],
    references: [workspaces.id],
  }),
}));

export const digestConfigsRelations = relations(digestConfigs, ({ one }) => ({
  workspace: one(workspaces, {
    fields: [digestConfigs.workspaceId],
    references: [workspaces.id],
  }),
}));

// ---------------------------------------------------------------------------
// Type Exports
// ---------------------------------------------------------------------------
export type Organization = typeof organizations.$inferSelect;
export type NewOrganization = typeof organizations.$inferInsert;
export type Workspace = typeof workspaces.$inferSelect;
export type NewWorkspace = typeof workspaces.$inferInsert;
export type WorkspaceMember = typeof workspaceMembers.$inferSelect;
export type Collection = typeof collections.$inferSelect;
export type NewCollection = typeof collections.$inferInsert;
export type Tag = typeof tags.$inferSelect;
export type Bookmark = typeof bookmarks.$inferSelect;
export type NewBookmark = typeof bookmarks.$inferInsert;
export type BookmarkTag = typeof bookmarkTags.$inferSelect;
export type ActivityFeedItem = typeof activityFeed.$inferSelect;
export type ImportJob = typeof importJobs.$inferSelect;
export type DigestConfig = typeof digestConfigs.$inferSelect;
```

### Initial Migration SQL

```sql
-- src/server/db/migrations/0000_init.sql

-- Enable required extensions
CREATE EXTENSION IF NOT EXISTS vector;
CREATE EXTENSION IF NOT EXISTS pg_trgm;

-- (All tables created by Drizzle push, then manually add vector column:)
ALTER TABLE bookmarks ADD COLUMN IF NOT EXISTS embedding vector(1536);

-- HNSW index for fast cosine similarity search on bookmark embeddings
CREATE INDEX IF NOT EXISTS bookmarks_embedding_idx
  ON bookmarks USING hnsw (embedding vector_cosine_ops)
  WITH (m = 16, ef_construction = 64);

-- Trigram index for fuzzy text search on bookmark titles
CREATE INDEX IF NOT EXISTS bookmarks_title_trgm_idx
  ON bookmarks USING gin (title gin_trgm_ops);

-- Trigram index for fuzzy text search on bookmark URLs
CREATE INDEX IF NOT EXISTS bookmarks_url_trgm_idx
  ON bookmarks USING gin (url gin_trgm_ops);

-- Trigram index for searching collection names
CREATE INDEX IF NOT EXISTS collections_name_trgm_idx
  ON collections USING gin (name gin_trgm_ops);
```

---

## Architecture Deep-Dives

### 1. URL Metadata Extraction Service

```typescript
// src/server/services/url-metadata.ts

import { JSDOM } from "jsdom";

export interface UrlMetadata {
  title: string | null;
  description: string | null;
  faviconUrl: string | null;
  ogImage: string | null;
  domain: string;
  contentText: string | null; // extracted readable text for embedding
}

const MAX_CONTENT_LENGTH = 10000;
const FETCH_TIMEOUT_MS = 10000;

export async function extractUrlMetadata(url: string): Promise<UrlMetadata> {
  const parsedUrl = new URL(url);
  const domain = parsedUrl.hostname.replace(/^www\./, "");

  try {
    const controller = new AbortController();
    const timeout = setTimeout(() => controller.abort(), FETCH_TIMEOUT_MS);

    const response = await fetch(url, {
      headers: {
        "User-Agent": "LinkShelfBot/1.0 (+https://linkshelf.io/bot)",
        Accept: "text/html,application/xhtml+xml",
      },
      signal: controller.signal,
      redirect: "follow",
    });
    clearTimeout(timeout);

    if (!response.ok) {
      return { title: null, description: null, faviconUrl: null, ogImage: null, domain, contentText: null };
    }

    const html = await response.text();
    const dom = new JSDOM(html, { url });
    const doc = dom.window.document;

    // Extract title: og:title > twitter:title > <title>
    const ogTitle = doc.querySelector('meta[property="og:title"]')?.getAttribute("content");
    const twitterTitle = doc.querySelector('meta[name="twitter:title"]')?.getAttribute("content");
    const htmlTitle = doc.querySelector("title")?.textContent;
    const title = ogTitle || twitterTitle || htmlTitle || null;

    // Extract description: og:description > meta description > twitter:description
    const ogDesc = doc.querySelector('meta[property="og:description"]')?.getAttribute("content");
    const metaDesc = doc.querySelector('meta[name="description"]')?.getAttribute("content");
    const twitterDesc = doc.querySelector('meta[name="twitter:description"]')?.getAttribute("content");
    const description = ogDesc || metaDesc || twitterDesc || null;

    // Extract favicon
    const iconLink = doc.querySelector('link[rel="icon"], link[rel="shortcut icon"]');
    const appleTouchIcon = doc.querySelector('link[rel="apple-touch-icon"]');
    let faviconUrl = iconLink?.getAttribute("href") || appleTouchIcon?.getAttribute("href");
    if (faviconUrl && !faviconUrl.startsWith("http")) {
      faviconUrl = new URL(faviconUrl, url).toString();
    }
    if (!faviconUrl) {
      faviconUrl = `https://www.google.com/s2/favicons?domain=${domain}&sz=64`;
    }

    // Extract OG image
    const ogImage = doc.querySelector('meta[property="og:image"]')?.getAttribute("content") || null;

    // Extract readable content text (strip nav, footer, sidebar, scripts)
    const body = doc.querySelector("article") || doc.querySelector("main") || doc.querySelector("body");
    const contentText = body
      ? body.textContent
          ?.replace(/\s+/g, " ")
          .trim()
          .slice(0, MAX_CONTENT_LENGTH) || null
      : null;

    return { title, description, faviconUrl, ogImage, domain, contentText };
  } catch (error) {
    return {
      title: null,
      description: null,
      faviconUrl: `https://www.google.com/s2/favicons?domain=${domain}&sz=64`,
      ogImage: null,
      domain,
      contentText: null,
    };
  }
}
```

### 2. AI Categorization Service

```typescript
// src/server/ai/categorize-bookmark.ts

import OpenAI from "openai";

const openai = new OpenAI({ apiKey: process.env.OPENAI_API_KEY });

export interface BookmarkCategorizationResult {
  category: "documentation" | "tutorial" | "tool" | "article" | "reference" | "video" | "repository" | "other";
  tags: string[];
  summary: string;
  confidence: number;
}

const CATEGORIZATION_PROMPT = `You are a bookmark categorization system. Analyze the following bookmark data and return structured JSON.

RULES:
- "category" must be exactly one of: "documentation", "tutorial", "tool", "article", "reference", "video", "repository", "other"
- "tags" is an array of 2-5 topic tags (lowercase, hyphenated). Examples: "react", "devops", "machine-learning", "css", "api-design", "typescript"
- "summary" is a concise 1-2 sentence description of what this bookmark is about
- "confidence" is how confident you are in the categorization (0.0 to 1.0)

BOOKMARK DATA:
- URL: {url}
- Title: {title}
- Description: {description}
- Content excerpt: {content}

Return ONLY valid JSON: {"category": "...", "tags": [...], "summary": "...", "confidence": 0.0}`;

export async function categorizeBookmark(opts: {
  url: string;
  title: string | null;
  description: string | null;
  contentText: string | null;
}): Promise<BookmarkCategorizationResult> {
  const prompt = CATEGORIZATION_PROMPT
    .replace("{url}", opts.url)
    .replace("{title}", opts.title ?? "N/A")
    .replace("{description}", opts.description ?? "N/A")
    .replace("{content}", opts.contentText?.slice(0, 2000) ?? "N/A");

  const response = await openai.chat.completions.create({
    model: "gpt-4o-mini",
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
    return { category: "other", tags: [], summary: opts.description ?? "", confidence: 0 };
  }

  const validCategories = ["documentation", "tutorial", "tool", "article", "reference", "video", "repository", "other"];

  return {
    category: validCategories.includes(parsed.category as string)
      ? (parsed.category as BookmarkCategorizationResult["category"])
      : "other",
    tags: Array.isArray(parsed.tags)
      ? parsed.tags.filter((t: unknown) => typeof t === "string").slice(0, 5)
      : [],
    summary: typeof parsed.summary === "string" ? parsed.summary : (opts.description ?? ""),
    confidence: typeof parsed.confidence === "number"
      ? Math.max(0, Math.min(1, parsed.confidence))
      : 0.5,
  };
}
```

### 3. Semantic Search Implementation

```typescript
// src/server/services/search.ts

import { neon } from "@neondatabase/serverless";
import OpenAI from "openai";
import { db } from "../db";
import { bookmarks, bookmarkTags, tags } from "../db/schema";
import { and, eq, ilike, or, sql, desc } from "drizzle-orm";

const openai = new OpenAI({ apiKey: process.env.OPENAI_API_KEY });

export interface SearchOptions {
  workspaceId: string;
  query: string;
  collectionId?: string;
  tagIds?: string[];
  domain?: string;
  limit?: number;
  offset?: number;
}

export interface SearchResult {
  bookmarkId: string;
  title: string | null;
  url: string;
  description: string | null;
  aiSummary: string | null;
  domain: string | null;
  similarity: number;
  matchType: "semantic" | "text" | "both";
}

/**
 * Hybrid search: combine pgvector semantic similarity with pg_trgm fuzzy text matching.
 * Results are ranked by a weighted combination of both scores.
 */
export async function searchBookmarks(opts: SearchOptions): Promise<SearchResult[]> {
  const limit = opts.limit ?? 20;
  const offset = opts.offset ?? 0;
  const sqlClient = neon(process.env.DATABASE_URL!);

  // Step 1: Generate embedding for the search query
  const embeddingResponse = await openai.embeddings.create({
    model: "text-embedding-3-small",
    input: opts.query,
    dimensions: 1536,
  });
  const queryEmbedding = embeddingResponse.data[0].embedding;
  const vectorStr = `[${queryEmbedding.join(",")}]`;

  // Step 2: Hybrid search combining semantic + text similarity
  const results = await sqlClient`
    WITH semantic AS (
      SELECT
        id,
        title,
        url,
        description,
        ai_summary,
        domain,
        1 - (embedding <=> ${vectorStr}::vector) AS semantic_score
      FROM bookmarks
      WHERE workspace_id = ${opts.workspaceId}
        AND embedding IS NOT NULL
        ${opts.collectionId ? sqlClient`AND collection_id = ${opts.collectionId}` : sqlClient``}
        ${opts.domain ? sqlClient`AND domain = ${opts.domain}` : sqlClient``}
      ORDER BY embedding <=> ${vectorStr}::vector
      LIMIT 50
    ),
    text_match AS (
      SELECT
        id,
        title,
        url,
        description,
        ai_summary,
        domain,
        GREATEST(
          COALESCE(similarity(title, ${opts.query}), 0),
          COALESCE(similarity(url, ${opts.query}), 0),
          COALESCE(similarity(ai_summary, ${opts.query}), 0)
        ) AS text_score
      FROM bookmarks
      WHERE workspace_id = ${opts.workspaceId}
        AND (
          title % ${opts.query}
          OR url % ${opts.query}
          OR ai_summary % ${opts.query}
        )
        ${opts.collectionId ? sqlClient`AND collection_id = ${opts.collectionId}` : sqlClient``}
        ${opts.domain ? sqlClient`AND domain = ${opts.domain}` : sqlClient``}
      LIMIT 50
    )
    SELECT
      COALESCE(s.id, t.id) AS bookmark_id,
      COALESCE(s.title, t.title) AS title,
      COALESCE(s.url, t.url) AS url,
      COALESCE(s.description, t.description) AS description,
      COALESCE(s.ai_summary, t.ai_summary) AS ai_summary,
      COALESCE(s.domain, t.domain) AS domain,
      COALESCE(s.semantic_score, 0) AS semantic_score,
      COALESCE(t.text_score, 0) AS text_score,
      -- Weighted combination: 70% semantic, 30% text
      (COALESCE(s.semantic_score, 0) * 0.7 + COALESCE(t.text_score, 0) * 0.3) AS combined_score,
      CASE
        WHEN s.id IS NOT NULL AND t.id IS NOT NULL THEN 'both'
        WHEN s.id IS NOT NULL THEN 'semantic'
        ELSE 'text'
      END AS match_type
    FROM semantic s
    FULL OUTER JOIN text_match t ON s.id = t.id
    WHERE COALESCE(s.semantic_score, 0) >= 0.3
       OR COALESCE(t.text_score, 0) >= 0.2
    ORDER BY combined_score DESC
    LIMIT ${limit}
    OFFSET ${offset}
  `;

  return results.map((r: any) => ({
    bookmarkId: r.bookmark_id,
    title: r.title,
    url: r.url,
    description: r.description,
    aiSummary: r.ai_summary,
    domain: r.domain,
    similarity: parseFloat(r.combined_score),
    matchType: r.match_type as "semantic" | "text" | "both",
  }));
}
```

### 4. Chrome Extension Auth Flow

```typescript
// extension/src/background.ts — Chrome Extension Service Worker

const API_BASE = "https://linkshelf.io";

interface AuthState {
  token: string | null;
  expiresAt: number;
}

let authState: AuthState = { token: null, expiresAt: 0 };

/**
 * Chrome extension auth flow using Clerk's publishable key.
 * The extension opens a popup to the web app's auth page,
 * which returns a session token via postMessage.
 */
export async function getAuthToken(): Promise<string> {
  if (authState.token && Date.now() < authState.expiresAt) {
    return authState.token;
  }

  // Try to get token from storage first
  const stored = await chrome.storage.local.get(["authToken", "tokenExpiresAt"]);
  if (stored.authToken && Date.now() < stored.tokenExpiresAt) {
    authState = { token: stored.authToken, expiresAt: stored.tokenExpiresAt };
    return stored.authToken;
  }

  // Need to authenticate — open auth popup
  return new Promise((resolve, reject) => {
    chrome.identity.launchWebAuthFlow(
      {
        url: `${API_BASE}/api/auth/extension?redirect_uri=${chrome.identity.getRedirectURL()}`,
        interactive: true,
      },
      async (responseUrl) => {
        if (chrome.runtime.lastError || !responseUrl) {
          reject(new Error("Auth failed: " + chrome.runtime.lastError?.message));
          return;
        }

        const url = new URL(responseUrl);
        const token = url.searchParams.get("token");
        const expiresIn = parseInt(url.searchParams.get("expires_in") ?? "3600");

        if (!token) {
          reject(new Error("No token received"));
          return;
        }

        const expiresAt = Date.now() + expiresIn * 1000;
        authState = { token, expiresAt };
        await chrome.storage.local.set({ authToken: token, tokenExpiresAt: expiresAt });

        resolve(token);
      }
    );
  });
}

/**
 * Save a bookmark from the extension to the API.
 */
export async function saveBookmark(data: {
  url: string;
  title: string;
  collectionId?: string;
  tags?: string[];
  note?: string;
}): Promise<{ id: string }> {
  const token = await getAuthToken();

  const response = await fetch(`${API_BASE}/api/extension/bookmarks`, {
    method: "POST",
    headers: {
      "Content-Type": "application/json",
      Authorization: `Bearer ${token}`,
    },
    body: JSON.stringify(data),
  });

  if (!response.ok) {
    if (response.status === 401) {
      // Token expired, clear and retry
      authState = { token: null, expiresAt: 0 };
      await chrome.storage.local.remove(["authToken", "tokenExpiresAt"]);
      return saveBookmark(data);
    }
    throw new Error(`Save failed: ${response.status}`);
  }

  return response.json();
}

// Context menu: "Save to LinkShelf"
chrome.contextMenus.create({
  id: "save-to-linkshelf",
  title: "Save to LinkShelf",
  contexts: ["page", "link"],
});

chrome.contextMenus.onClicked.addListener(async (info, tab) => {
  if (info.menuItemId === "save-to-linkshelf") {
    const url = info.linkUrl || info.pageUrl;
    const title = tab?.title || url;

    try {
      await saveBookmark({ url: url!, title: title! });
      // Show success badge
      chrome.action.setBadgeText({ text: "✓", tabId: tab?.id });
      chrome.action.setBadgeBackgroundColor({ color: "#22c55e" });
      setTimeout(() => chrome.action.setBadgeText({ text: "", tabId: tab?.id }), 2000);
    } catch (error) {
      chrome.action.setBadgeText({ text: "!", tabId: tab?.id });
      chrome.action.setBadgeBackgroundColor({ color: "#ef4444" });
    }
  }
});
```

---

## Phase Breakdown (8 weeks) — Day-by-Day

### Phase 1: Foundation (Week 1) — Project Setup, Auth, Database, Core Models

**Day 1 — Project scaffold + environment**
- `npx create-next-app@latest linkshelf --typescript --tailwind --app --src-dir`
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
    OPENAI_API_KEY: z.string().min(1),
    UPSTASH_REDIS_URL: z.string().url(),
    UPSTASH_REDIS_TOKEN: z.string().min(1),
    R2_ACCOUNT_ID: z.string().min(1),
    R2_ACCESS_KEY_ID: z.string().min(1),
    R2_SECRET_ACCESS_KEY: z.string().min(1),
    R2_BUCKET_NAME: z.string().min(1),
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
    free: { maxBookmarks: 50, maxCollections: 1, maxWorkspaceMembers: 1, semanticSearch: false },
    pro: { maxBookmarks: Infinity, maxCollections: Infinity, maxWorkspaceMembers: 1, semanticSearch: true },
    team: { maxBookmarks: Infinity, maxCollections: Infinity, maxWorkspaceMembers: Infinity, semanticSearch: true },
  } as const;
  ```
- Create `src/server/trpc/routers/_app.ts`: merge routers

**Day 4 — Workspace + Collection tRPC routers**
- `src/server/trpc/routers/workspace.ts`:
  - `getOrCreate`: find by clerkOrgId or create personal workspace
  - `list`: all workspaces for user
  - `update`: name
  - `getMembers`: list members with roles
  - `inviteMember`: add by email (Clerk invitation)
  - `removeMember`: remove member
  - `updateMemberRole`: change role
- `src/server/trpc/routers/collection.ts`:
  - `list`: all collections for workspace, with bookmark counts
  - `getById`: single collection with child collections
  - `create`: name, parentCollectionId, icon
  - `update`: name, description, icon, isPinned
  - `delete`: cascade delete bookmarks option or move to uncategorized
  - `reorder`: update sortOrder for drag-and-drop
  - `getBreadcrumbs`: parent chain for navigation

**Day 5 — Base UI: layout, navigation, dashboard shell**
- Install UI deps: `@radix-ui/react-dialog @radix-ui/react-dropdown-menu @radix-ui/react-tabs @radix-ui/react-select @radix-ui/react-tooltip @radix-ui/react-popover class-variance-authority clsx tailwind-merge lucide-react`
- Create shared UI components in `src/components/ui/`: button, card, badge, input, textarea, select, dialog, dropdown-menu, tabs, toast, skeleton
- Build app shell layout: `src/app/dashboard/layout.tsx`
  - Left sidebar: logo, nav links (Library, Favorites, Collections, Activity, Settings), workspace switcher
  - Main content area with header + search bar
- Create stub pages:
  - `src/app/dashboard/page.tsx` (library/home)
  - `src/app/dashboard/favorites/page.tsx`
  - `src/app/dashboard/collections/page.tsx`
  - `src/app/dashboard/activity/page.tsx`
  - `src/app/dashboard/settings/page.tsx`
  - `src/app/dashboard/settings/billing/page.tsx`

---

### Phase 2: Bookmark Core (Week 2) — CRUD, Metadata Extraction, Storage

**Day 6 — Bookmark tRPC router**
- `src/server/trpc/routers/bookmark.ts`:
  - `create`: accepts `{ url, collectionId?, tags?, note? }`, extracts domain, enqueues for async processing
  - `list`: paginated, filterable by collection, tag, domain, favorite, date range; sortable by created_at, title, domain
  - `getById`: full detail with tags + collection
  - `update`: title, description, note, collectionId, isPinned
  - `delete`: hard delete
  - `toggleFavorite`: flip isFavorite
  - `bulkMove`: move multiple bookmarks to collection
  - `bulkTag`: add tag to multiple bookmarks
  - `bulkDelete`: delete multiple bookmarks
  - `incrementViewCount`: called when user clicks bookmark URL
  - `getByUrl`: check if URL already saved (for extension)

**Day 7 — URL metadata extraction service**
- Create `src/server/services/url-metadata.ts` (as shown in architecture deep-dive above)
- Install: `jsdom`
- Create `src/server/services/r2-storage.ts`:
  ```typescript
  import { S3Client, PutObjectCommand } from "@aws-sdk/client-s3";

  const s3 = new S3Client({
    region: "auto",
    endpoint: `https://${process.env.R2_ACCOUNT_ID}.r2.cloudflarestorage.com`,
    credentials: {
      accessKeyId: process.env.R2_ACCESS_KEY_ID!,
      secretAccessKey: process.env.R2_SECRET_ACCESS_KEY!,
    },
  });

  export async function uploadToR2(key: string, body: Buffer, contentType: string): Promise<string> {
    await s3.send(new PutObjectCommand({
      Bucket: process.env.R2_BUCKET_NAME!,
      Key: key,
      Body: body,
      ContentType: contentType,
    }));
    return `${process.env.R2_PUBLIC_URL}/${key}`;
  }
  ```

**Day 8 — BullMQ queue for async processing**
- Create `src/server/queue/connection.ts`: Redis/ioredis connection
- Create `src/server/queue/bookmark-queue.ts`:
  ```typescript
  import { Queue, Worker, Job } from "bullmq";
  import { redis } from "./connection";

  export interface BookmarkJobData {
    bookmarkId: string;
    workspaceId: string;
    url: string;
  }

  export const bookmarkQueue = new Queue<BookmarkJobData>("bookmark-processing", {
    connection: redis,
    defaultJobOptions: {
      attempts: 3,
      backoff: { type: "exponential", delay: 2000 },
      removeOnComplete: { age: 86400 },
      removeOnFail: { age: 604800 },
    },
  });
  ```
- Create `src/server/queue/bookmark-worker.ts`:
  - Step 1: Extract URL metadata (title, description, favicon, content)
  - Step 2: Upload favicon to R2
  - Step 3: AI categorize (category, tags, summary)
  - Step 4: Generate embedding from title + description + summary + content
  - Step 5: Store embedding via raw SQL
  - Step 6: Update bookmark record with all extracted data
  - Step 7: Resolve or create tags
  - Step 8: Log activity feed entry

**Day 9 — Embedding generation + storage**
- Create `src/server/ai/embeddings.ts`:
  ```typescript
  import OpenAI from "openai";
  import { neon } from "@neondatabase/serverless";

  const openai = new OpenAI({ apiKey: process.env.OPENAI_API_KEY });

  function buildEmbeddingInput(opts: {
    title: string | null;
    description: string | null;
    aiSummary: string | null;
    contentText: string | null;
  }): string {
    const parts: string[] = [];
    if (opts.title) parts.push(opts.title);
    if (opts.aiSummary) parts.push(opts.aiSummary);
    if (opts.description) parts.push(opts.description);
    if (opts.contentText) parts.push(opts.contentText.slice(0, 4000));
    return parts.join("\n\n").slice(0, 8000);
  }

  export async function generateBookmarkEmbedding(opts: {
    title: string | null;
    description: string | null;
    aiSummary: string | null;
    contentText: string | null;
  }): Promise<number[]> {
    const input = buildEmbeddingInput(opts);
    if (!input.trim()) return [];
    const response = await openai.embeddings.create({
      model: "text-embedding-3-small",
      input,
      dimensions: 1536,
    });
    return response.data[0].embedding;
  }

  export async function storeBookmarkEmbedding(bookmarkId: string, embedding: number[]): Promise<void> {
    const sql = neon(process.env.DATABASE_URL!);
    const vectorStr = `[${embedding.join(",")}]`;
    await sql`UPDATE bookmarks SET embedding = ${vectorStr}::vector WHERE id = ${bookmarkId}`;
  }
  ```

**Day 10 — Bookmark list UI with grid/list toggle**
- Create `src/components/bookmarks/bookmark-card.tsx`: favicon, title, domain, tags, summary snippet, save date, favorite star
- Create `src/components/bookmarks/bookmark-list.tsx`: grid + list view toggle, infinite scroll
- Create `src/components/bookmarks/bookmark-filters.tsx`: collection dropdown, tag multi-select, domain filter, date range
- Create `src/app/dashboard/page.tsx`: search bar, tabs (All | Favorites | Recent), bookmark grid
- Create `src/components/layout/collection-sidebar.tsx`: collapsible collection tree in left sidebar

---

### Phase 3: AI Pipeline + Search (Week 3) — Categorization, Embeddings, Semantic Search

**Day 11 — AI categorization service**
- Create `src/server/ai/categorize-bookmark.ts` (as shown in architecture deep-dive)
- Wire into bookmark-worker.ts pipeline

**Day 12 — Tag resolution service**
- Create `src/server/services/tag-resolver.ts`:
  ```typescript
  export async function resolveOrCreateTags(
    workspaceId: string,
    tagNames: string[],
    source: "ai" | "manual"
  ): Promise<{ tagId: string; name: string }[]> {
    const resolved: { tagId: string; name: string }[] = [];
    for (const name of tagNames) {
      const normalized = name.toLowerCase().trim();
      const [existing] = await db
        .select()
        .from(tags)
        .where(and(eq(tags.workspaceId, workspaceId), eq(tags.name, normalized)))
        .limit(1);

      if (existing) {
        await db.update(tags)
          .set({ usageCount: sql`usage_count + 1` })
          .where(eq(tags.id, existing.id));
        resolved.push({ tagId: existing.id, name: existing.name });
      } else {
        const [created] = await db.insert(tags).values({
          workspaceId,
          name: normalized,
          usageCount: 1,
        }).returning();
        resolved.push({ tagId: created.id, name: created.name });
      }
    }
    return resolved;
  }
  ```

**Day 13 — Semantic search implementation**
- Create `src/server/services/search.ts` (as shown in architecture deep-dive)
- Add search tRPC router: `src/server/trpc/routers/search.ts`
  - `search`: accepts query + filters, returns ranked results
  - `suggest`: autocomplete from recent searches and popular tags

**Day 14 — Full-text search with pg_trgm fuzzy matching**
- Add text search fallback in search service for users on Free plan (no semantic)
- Create `src/server/services/text-search.ts`: pg_trgm-based search on title, URL, description

**Day 15 — Search UI**
- Create `src/components/search/search-bar.tsx`: prominent search input with keyboard shortcut (Cmd+K)
- Create `src/components/search/search-results.tsx`: ranked results with match highlighting
- Create `src/components/search/search-filters.tsx`: filter chips for collection, tag, domain, date
- Wire search into dashboard header — global search accessible from anywhere

---

### Phase 4: Chrome Extension (Week 4) — Manifest V3, Popup, Context Menu

**Day 16 — Extension scaffold**
- Create `extension/` directory with Manifest V3 structure
- Install: `vite` for extension build
- Create `extension/manifest.json`:
  ```json
  {
    "manifest_version": 3,
    "name": "LinkShelf",
    "version": "1.0.0",
    "description": "Save and organize bookmarks with AI",
    "permissions": ["activeTab", "contextMenus", "storage", "identity"],
    "action": { "default_popup": "popup.html", "default_icon": "icon-48.png" },
    "background": { "service_worker": "background.js", "type": "module" },
    "host_permissions": ["https://linkshelf.io/*"],
    "icons": { "16": "icon-16.png", "48": "icon-48.png", "128": "icon-128.png" }
  }
  ```
- Create `extension/src/popup.tsx`: Preact popup component
- Create `extension/src/background.ts`: service worker (as shown in architecture deep-dive)

**Day 17 — Extension auth flow**
- Create `src/app/api/auth/extension/route.ts`: generates a short-lived token from Clerk session
- Implement `getAuthToken()` in extension service worker
- Test: sign in via extension → verify token stored in chrome.storage

**Day 18 — Save bookmark popup**
- Create extension popup UI:
  - Current page info (title, URL, favicon)
  - Collection dropdown (fetched from API)
  - Tag input (autocomplete from existing tags)
  - Note field
  - "Save" button with loading state
  - Recently saved bookmarks list
- Create `src/app/api/extension/bookmarks/route.ts`: REST endpoint for extension saves (not tRPC — simpler for extension)

**Day 19 — Context menu + keyboard shortcut**
- Implement context menu "Save to LinkShelf" (as shown in background.ts)
- Add keyboard shortcut: `Alt+Shift+S` to trigger save
- Badge counter: show number of unsaved tabs
- Duplicate detection: check if current page URL already saved, show indicator

**Day 20 — Extension ↔ web app sync**
- Extension popup shows "Already saved" if URL exists in workspace
- "Open in LinkShelf" button to jump to bookmark detail in web app
- Extension settings page: default collection, auto-save options
- Build extension: `vite build` → output to `extension/dist/`

---

### Phase 5: Team Features (Week 5) — Workspaces, Sharing, Activity Feed

**Day 21 — Team workspace creation + invite flow**
- Create `src/app/dashboard/settings/workspace/page.tsx`: create team workspace form
- Workspace invite flow: enter email → Clerk organization invitation → member record created
- Invite acceptance: webhook handler creates workspace member on org join

**Day 22 — Workspace member management + permissions**
- Create `src/components/workspace/members-list.tsx`: member list with role badges
- Permission enforcement in tRPC middleware:
  - Viewer: read bookmarks and collections
  - Contributor: save bookmarks, create collections, edit own bookmarks
  - Admin: manage members, delete any bookmark, manage settings
- Workspace switcher in sidebar

**Day 23 — Activity feed**
- Create `src/server/trpc/routers/activity.ts`:
  - `list`: paginated activity for workspace, filterable by user and type
- Create `src/app/dashboard/activity/page.tsx`:
  - Feed of recent saves: "Ryan saved 'React 19 Migration Guide' to Engineering Resources"
  - Filter by team member
  - Activity card with bookmark preview
- Wire activity logging into bookmark create/update/delete mutations

**Day 24 — Collection sharing + workspace switcher**
- Personal vs team bookmark visibility toggle
- Workspace switcher in sidebar header
- Collection sharing: generate sharable link for a collection (read-only, no auth required)
- Create `src/app/shared/[collectionSlug]/page.tsx`: public collection view

---

### Phase 6: Import + Billing (Week 6)

**Day 25 — Chrome bookmarks HTML import parser**
- Create `src/server/services/import/chrome-parser.ts`:
  ```typescript
  import { JSDOM } from "jsdom";

  export interface ImportedBookmark {
    url: string;
    title: string;
    addDate: Date | null;
    folderPath: string[]; // ["Bookmarks Bar", "Dev", "React"]
  }

  /**
   * Parse Chrome bookmarks HTML export format.
   * Chrome exports bookmarks as nested <DL><DT> lists.
   */
  export function parseChromeBookmarks(html: string): ImportedBookmark[] {
    const dom = new JSDOM(html);
    const bookmarks: ImportedBookmark[] = [];

    function walk(node: Element, path: string[]) {
      const items = node.children;
      for (let i = 0; i < items.length; i++) {
        const item = items[i];
        if (item.tagName === "DT") {
          const link = item.querySelector(":scope > A");
          if (link) {
            const url = link.getAttribute("HREF");
            const title = link.textContent?.trim() || url || "";
            const addDateStr = link.getAttribute("ADD_DATE");
            const addDate = addDateStr ? new Date(parseInt(addDateStr) * 1000) : null;
            if (url && url.startsWith("http")) {
              bookmarks.push({ url, title, addDate, folderPath: [...path] });
            }
          }
          const sublist = item.querySelector(":scope > DL");
          if (sublist) {
            const folderHeader = item.querySelector(":scope > H3");
            const folderName = folderHeader?.textContent?.trim() || "Untitled";
            walk(sublist, [...path, folderName]);
          }
        }
      }
    }

    const rootDl = dom.window.document.querySelector("DL");
    if (rootDl) walk(rootDl, []);
    return bookmarks;
  }
  ```

**Day 26 — Import wizard UI with deduplication**
- Create `src/app/dashboard/import/page.tsx`:
  - Upload Chrome bookmarks HTML file
  - Preview parsed bookmarks in table
  - Deduplication check: highlight duplicates already in workspace
  - Collection mapping: auto-create collections from folder structure
  - "Import" button → creates import job, processes in background
- Import worker: batch process bookmarks (create records, enqueue for AI processing)

**Day 27 — Stripe billing setup**
- Create `src/server/billing/stripe.ts`: plan definitions, checkout session, portal session
  ```typescript
  export const PLANS = {
    free: { name: "Free", price: 0, maxBookmarks: 50, maxCollections: 1, semanticSearch: false },
    pro: { name: "Pro", price: 6, maxBookmarks: Infinity, maxCollections: Infinity, semanticSearch: true },
    team: { name: "Team", pricePerUser: 4, minUsers: 3, maxBookmarks: Infinity, maxCollections: Infinity, semanticSearch: true },
  } as const;
  ```
- Create `src/app/api/webhooks/stripe/route.ts`: handle checkout.session.completed, subscription.updated, subscription.deleted
- Create `src/server/trpc/routers/billing.ts`: getCurrentPlan, createCheckout, createPortal

**Day 28 — Billing UI + plan enforcement**
- Create `src/app/dashboard/settings/billing/page.tsx`: current plan, usage meters, plan cards, upgrade buttons
- Plan enforcement middleware: check bookmark count, collection count, semantic search access
- Usage banner when approaching limits

**Day 29 — Settings pages**
- Create `src/app/dashboard/settings/page.tsx`: workspace name, default collection
- Create `src/app/dashboard/settings/members/page.tsx`: member management (Team plan only)
- Profile settings: display name, avatar

---

### Phase 7: Polish + Launch (Week 7-8)

**Day 30 — Dead link detection**
- Create `src/app/api/cron/check-links/route.ts`: Vercel Cron job (weekly)
  - Fetch bookmarks where `lastCheckedAt < 7 days ago` in batches of 100
  - HEAD request to each URL, check for 404/timeout/redirect
  - Update `linkStatus` and `lastCheckedAt`
  - Flag broken links in UI with warning badge

**Day 31 — Onboarding flow**
- Create `src/app/onboarding/page.tsx`:
  - Step 1: Name your workspace
  - Step 2: Save your first bookmark (paste URL or install extension)
  - Step 3: See AI categorization result → "Your library is ready!"
- Skip option to go straight to dashboard

**Day 32 — Loading states, error handling, empty states**
- Suspense boundaries with skeleton loaders
- Empty state components: "No bookmarks yet — save your first link"
- Toast notifications for all mutations
- Global error boundary

**Day 33 — Landing page + pricing**
- Create `src/app/page.tsx`: hero, features, pricing table, CTA
- Create `src/app/pricing/page.tsx`: detailed comparison

**Day 34 — Final testing + deployment**
- End-to-end test: sign up → save bookmark → search → extension → import → billing
- Vercel deployment + environment variables
- Chrome Web Store submission for extension
- Sentry error tracking setup

---

## Critical Files

```
linkshelf/
├── src/
│   ├── env.ts                                    # Zod-validated environment variables
│   ├── middleware.ts                              # Clerk auth middleware, route protection
│   │
│   ├── app/
│   │   ├── layout.tsx                            # Root layout with ClerkProvider
│   │   ├── page.tsx                              # Landing page
│   │   ├── pricing/page.tsx                      # Pricing page
│   │   ├── onboarding/page.tsx                   # 3-step onboarding wizard
│   │   │
│   │   ├── dashboard/
│   │   │   ├── layout.tsx                        # App shell: sidebar + header + search
│   │   │   ├── page.tsx                          # Library home (bookmark grid + filters)
│   │   │   ├── favorites/page.tsx                # Favorite bookmarks
│   │   │   ├── collections/page.tsx              # Collection browser
│   │   │   ├── collections/[id]/page.tsx         # Single collection view
│   │   │   ├── activity/page.tsx                 # Team activity feed
│   │   │   ├── import/page.tsx                   # Import wizard
│   │   │   └── settings/
│   │   │       ├── page.tsx                      # Workspace settings
│   │   │       ├── billing/page.tsx              # Plan + usage + upgrade
│   │   │       ├── members/page.tsx              # Team member management
│   │   │       └── workspace/page.tsx            # Workspace creation/management
│   │   │
│   │   ├── shared/
│   │   │   └── [collectionSlug]/page.tsx         # Public shared collection view
│   │   │
│   │   └── api/
│   │       ├── trpc/[trpc]/route.ts              # tRPC HTTP handler
│   │       ├── auth/extension/route.ts           # Extension auth token generation
│   │       ├── extension/bookmarks/route.ts      # REST API for extension saves
│   │       ├── cron/check-links/route.ts         # Dead link detection cron
│   │       └── webhooks/
│   │           └── stripe/route.ts               # Stripe webhook handler
│   │
│   ├── server/
│   │   ├── db/
│   │   │   ├── schema.ts                        # Drizzle schema (all tables + relations)
│   │   │   ├── index.ts                         # Neon database client
│   │   │   └── migrations/
│   │   │       └── 0000_init.sql                # pgvector + pg_trgm + indexes
│   │   │
│   │   ├── trpc/
│   │   │   ├── trpc.ts                          # tRPC init, auth/org/plan middleware
│   │   │   ├── context.ts                       # Request context (auth + db)
│   │   │   └── routers/
│   │   │       ├── _app.ts                      # Merged app router
│   │   │       ├── workspace.ts                 # Workspace CRUD + members
│   │   │       ├── collection.ts                # Collection CRUD + reorder
│   │   │       ├── bookmark.ts                  # Bookmark CRUD + favorites + bulk ops
│   │   │       ├── tag.ts                       # Tag list + popular
│   │   │       ├── search.ts                    # Semantic + text search
│   │   │       ├── activity.ts                  # Activity feed
│   │   │       └── billing.ts                   # Plan info + Stripe checkout/portal
│   │   │
│   │   ├── ai/
│   │   │   ├── categorize-bookmark.ts           # GPT-4o-mini categorization
│   │   │   └── embeddings.ts                    # text-embedding-3-small + vector storage
│   │   │
│   │   ├── queue/
│   │   │   ├── connection.ts                    # Redis/ioredis connection
│   │   │   ├── bookmark-queue.ts                # BullMQ queue definition
│   │   │   └── bookmark-worker.ts               # Worker: metadata → categorize → embed
│   │   │
│   │   ├── services/
│   │   │   ├── url-metadata.ts                  # URL fetching + OG tag extraction
│   │   │   ├── search.ts                        # Hybrid semantic + text search
│   │   │   ├── text-search.ts                   # pg_trgm fallback for free tier
│   │   │   ├── tag-resolver.ts                  # Find-or-create tags
│   │   │   ├── r2-storage.ts                    # Cloudflare R2 upload
│   │   │   └── import/
│   │   │       └── chrome-parser.ts             # Chrome bookmarks HTML parser
│   │   │
│   │   └── billing/
│   │       └── stripe.ts                        # Plan definitions, checkout, portal
│   │
│   ├── components/
│   │   ├── ui/                                  # Shared Radix UI primitives
│   │   │   ├── button.tsx, card.tsx, badge.tsx, input.tsx, textarea.tsx
│   │   │   ├── select.tsx, dialog.tsx, dropdown-menu.tsx
│   │   │   ├── tabs.tsx, toast.tsx, skeleton.tsx, tooltip.tsx, popover.tsx
│   │   │   └── command.tsx                      # Cmd+K search dialog
│   │   │
│   │   ├── layout/
│   │   │   ├── sidebar.tsx                      # Navigation sidebar with collection tree
│   │   │   ├── header.tsx                       # Page header with search bar
│   │   │   └── workspace-switcher.tsx           # Workspace selector
│   │   │
│   │   ├── bookmarks/
│   │   │   ├── bookmark-card.tsx                # Grid/list bookmark card
│   │   │   ├── bookmark-list.tsx                # Infinite scroll + view toggle
│   │   │   ├── bookmark-detail.tsx              # Full bookmark detail panel
│   │   │   ├── bookmark-filters.tsx             # Filter controls
│   │   │   └── save-bookmark-dialog.tsx         # Add bookmark from web app
│   │   │
│   │   ├── collections/
│   │   │   ├── collection-tree.tsx              # Nested collection tree
│   │   │   ├── collection-card.tsx              # Collection preview card
│   │   │   └── create-collection-dialog.tsx     # Create collection form
│   │   │
│   │   ├── search/
│   │   │   ├── search-bar.tsx                   # Global search input
│   │   │   ├── search-results.tsx               # Ranked results
│   │   │   └── search-filters.tsx               # Filter chips
│   │   │
│   │   ├── workspace/
│   │   │   ├── members-list.tsx                 # Member management
│   │   │   └── invite-dialog.tsx                # Invite member form
│   │   │
│   │   └── import/
│   │       ├── import-wizard.tsx                # Multi-step import flow
│   │       └── import-preview.tsx               # Preview parsed bookmarks
│   │
│   └── lib/
│       ├── trpc/
│       │   ├── client.ts                        # React Query tRPC client
│       │   └── server.ts                        # Server-side tRPC caller
│       ├── utils.ts                             # cn() helper, date formatting, URL helpers
│       └── constants.ts                         # Category labels, source icons
│
├── extension/
│   ├── manifest.json                            # Chrome Manifest V3
│   ├── src/
│   │   ├── background.ts                        # Service worker (auth, context menu, API)
│   │   ├── popup.tsx                            # Popup UI (Preact)
│   │   ├── popup.html                           # Popup HTML shell
│   │   └── styles.css                           # Popup styles
│   ├── icons/                                   # Extension icons
│   └── vite.config.ts                           # Build config
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

1. **Auth flow**: Sign up with Clerk → create organization → verify org + workspace created in DB
2. **Save bookmark (web)**: Paste URL → verify metadata extraction runs → title, favicon, AI summary populated
3. **AI categorization**: Save bookmark → verify category, tags, summary populated within 30s
4. **Embedding storage**: Verify `embedding` column is non-null on processed bookmarks
5. **Semantic search**: Search "react performance optimization" → find relevant bookmarks even without exact keywords
6. **Text search**: Search with typo "recat hooks" → verify fuzzy match returns React hooks bookmarks
7. **Collections**: Create nested collections → verify breadcrumb navigation works
8. **Chrome extension install**: Load unpacked extension → sign in → save current page
9. **Extension context menu**: Right-click → "Save to LinkShelf" → verify bookmark appears in web app
10. **Extension duplicate detection**: Visit already-saved URL → verify "Already saved" indicator
11. **Team workspace**: Create team → invite member → verify shared bookmarks visible
12. **Activity feed**: Save bookmark as team member → verify activity appears for other members
13. **Chrome import**: Upload Chrome bookmarks HTML → verify bookmarks and folder structure imported
14. **Dead link detection**: Bookmark a known 404 URL → trigger cron → verify `linkStatus` set to "broken"
15. **Plan limits**: On Free plan, save 51st bookmark → verify error returned
16. **Billing upgrade**: Complete Stripe checkout → verify plan updated, semantic search enabled
17. **Stripe webhook**: Simulate subscription deletion → verify plan reverts to free

### Key SQL Queries for Verification

```sql
-- Check embedding was stored and find similar bookmarks
SELECT id, title, url,
       embedding IS NOT NULL AS has_embedding,
       1 - (embedding <=> (SELECT embedding FROM bookmarks WHERE id = 'SOME_ID')) AS similarity
FROM bookmarks
WHERE workspace_id = 'WS_ID'
ORDER BY embedding <=> (SELECT embedding FROM bookmarks WHERE id = 'SOME_ID')
LIMIT 5;

-- Check AI processing completed
SELECT COUNT(*) AS total,
       COUNT(*) FILTER (WHERE ai_processed = true) AS processed,
       COUNT(*) FILTER (WHERE embedding IS NOT NULL) AS embedded
FROM bookmarks WHERE workspace_id = 'WS_ID';

-- Check tag distribution
SELECT t.name, t.usage_count, COUNT(bt.id) AS actual_count
FROM tags t
LEFT JOIN bookmark_tags bt ON bt.tag_id = t.id
WHERE t.workspace_id = 'WS_ID'
GROUP BY t.id
ORDER BY t.usage_count DESC;

-- Check dead links
SELECT url, link_status, last_checked_at
FROM bookmarks
WHERE workspace_id = 'WS_ID' AND link_status != 'alive';

-- Monthly bookmark count (for plan enforcement)
SELECT COUNT(*) FROM bookmarks
WHERE workspace_id = 'WS_ID'
  AND created_at >= date_trunc('month', now());
```

### Performance Benchmarks

| Operation | Target | Method |
|-----------|--------|--------|
| Bookmark save (API response) | < 200ms | Enqueue only, process async |
| URL metadata extraction | < 5s | Timeout at 10s, fallback on failure |
| AI categorization (single) | < 2s | GPT-4o-mini with 300 max_tokens |
| Embedding generation | < 1s | text-embedding-3-small |
| Semantic search (10K bookmarks) | < 200ms | HNSW index on pgvector |
| Text search (fuzzy) | < 100ms | pg_trgm GIN index |
| Bookmark list page load (50 items) | < 400ms | Indexed queries + React Query cache |
| Extension popup open | < 300ms | Cached collections + auth token |
| Chrome import (1000 bookmarks) | < 60s | Batch insert + async AI processing |
| Dead link check (100 URLs) | < 30s | Parallel HEAD requests with timeout |
