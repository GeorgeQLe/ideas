# 3. SnipVault — AI-Powered Code Snippet Manager

## Implementation Plan

**MVP Scope:** Web app with snippet CRUD and syntax highlighting (Monaco Editor), collections and tags with nested folder support, AI auto-tagging on save via GPT-4o-mini, semantic search via pgvector + text-embedding-3-small with hybrid full-text fallback (pg_trgm), VS Code extension (save selected code + search + insert), GitHub Gists one-click import, personal workspace with single-user access, Stripe billing with three tiers (Free 100 snippets / Pro $8/mo / Team $5/user/mo).

---

## Tech Decisions

| Layer | Technology | Notes |
|-------|-----------|-------|
| Framework | Next.js 14+ App Router | `src/` directory, TypeScript strict |
| API | tRPC v11 | superjson transformer, Clerk auth context |
| Database | Neon PostgreSQL + pgvector + pg_trgm | Hybrid semantic + fuzzy text search |
| ORM | Drizzle ORM | Push-based migrations, typed schema |
| Auth | Clerk | Organization-scoped, `orgId` from session |
| Billing | Stripe | Checkout Sessions + Customer Portal + Webhooks |
| AI — Tagging | OpenAI GPT-4o-mini | JSON mode, temperature 0.1 for classification |
| AI — Embeddings | text-embedding-3-small | 1536 dimensions, cosine similarity |
| Queue | BullMQ + Redis (Upstash) | Async embedding + AI tagging on snippet save |
| Code Editor | Monaco Editor | Syntax highlighting, multi-file tabs |
| VS Code Extension | VS Code Extension API | Save, search, insert from command palette |
| Search | pgvector (semantic) + pg_trgm (fuzzy) | Hybrid scoring with RRF merge |
| UI | Tailwind CSS, Radix UI | |
| Hosting | Vercel | |
| Monitoring | Sentry, Vercel Analytics | |

### Key Architecture Decisions

1. **Hybrid search (pgvector + pg_trgm) over pure vector search**: Semantic search alone misses exact keyword matches (e.g., searching for `useEffect` should find snippets containing that exact hook). We run both a pgvector cosine similarity query and a pg_trgm trigram similarity query, then merge results using Reciprocal Rank Fusion (RRF). This provides both "find by concept" and "find by keyword" in a single search.

2. **BullMQ for async AI processing**: When a snippet is saved, the API returns immediately. A BullMQ worker generates the embedding and runs AI tagging asynchronously. This keeps save operations under 200ms, critical for the VS Code extension experience.

3. **VS Code extension uses device code flow for auth**: Instead of embedding Clerk credentials in the extension, we use OAuth device code flow. The user authenticates in their browser, and the extension polls for the token. This is secure and doesn't require a redirect URI.

4. **Per-file embeddings for multi-file snippets**: Each `SnippetFile` gets its own embedding vector. Search queries match against individual files, but results are grouped by snippet. This allows finding a snippet by searching for code that appears in any of its files.

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
  vector,
} from "drizzle-orm/pg-core";

// ---------------------------------------------------------------------------
// Organizations (personal workspace or team)
// ---------------------------------------------------------------------------
export const organizations = pgTable("organizations", {
  id: uuid("id").primaryKey().defaultRandom(),
  clerkOrgId: text("clerk_org_id").unique().notNull(),
  name: text("name").notNull(),
  slug: text("slug").unique().notNull(),
  type: text("type").default("personal").notNull(), // personal | team
  plan: text("plan").default("free").notNull(), // free | pro | team
  stripeCustomerId: text("stripe_customer_id"),
  stripeSubscriptionId: text("stripe_subscription_id"),
  settings: jsonb("settings").default({}).$type<{
    defaultLanguage?: string;
    editorTheme?: string;
    snippetVisibility?: "private" | "workspace";
  }>(),
  createdAt: timestamp("created_at", { withTimezone: true }).defaultNow().notNull(),
  updatedAt: timestamp("updated_at", { withTimezone: true }).defaultNow().notNull(),
});

// ---------------------------------------------------------------------------
// Workspace Members (for team workspaces)
// ---------------------------------------------------------------------------
export const workspaceMembers = pgTable(
  "workspace_members",
  {
    id: uuid("id").primaryKey().defaultRandom(),
    orgId: uuid("org_id")
      .references(() => organizations.id, { onDelete: "cascade" })
      .notNull(),
    clerkUserId: text("clerk_user_id").notNull(),
    role: text("role").default("editor").notNull(), // admin | editor | viewer
    name: text("name").notNull(),
    email: text("email").notNull(),
    avatarUrl: text("avatar_url"),
    joinedAt: timestamp("joined_at", { withTimezone: true }).defaultNow().notNull(),
  },
  (t) => [
    uniqueIndex("wm_org_user_idx").on(t.orgId, t.clerkUserId),
    index("wm_org_idx").on(t.orgId),
  ],
);

// ---------------------------------------------------------------------------
// Collections (folders for organizing snippets)
// ---------------------------------------------------------------------------
export const collections = pgTable(
  "collections",
  {
    id: uuid("id").primaryKey().defaultRandom(),
    orgId: uuid("org_id")
      .references(() => organizations.id, { onDelete: "cascade" })
      .notNull(),
    name: text("name").notNull(),
    description: text("description"),
    icon: text("icon"), // emoji or icon name
    parentCollectionId: uuid("parent_collection_id"), // self-referencing for nesting
    isSmart: boolean("is_smart").default(false).notNull(),
    smartRules: jsonb("smart_rules").$type<{
      languages?: string[];
      tags?: string[];
      match?: "all" | "any";
    }>(),
    sortOrder: integer("sort_order").default(0).notNull(),
    createdAt: timestamp("created_at", { withTimezone: true }).defaultNow().notNull(),
  },
  (t) => [
    index("collection_org_idx").on(t.orgId),
    index("collection_parent_idx").on(t.parentCollectionId),
  ],
);

// ---------------------------------------------------------------------------
// Tags
// ---------------------------------------------------------------------------
export const tags = pgTable(
  "tags",
  {
    id: uuid("id").primaryKey().defaultRandom(),
    orgId: uuid("org_id")
      .references(() => organizations.id, { onDelete: "cascade" })
      .notNull(),
    name: text("name").notNull(),
    category: text("category").default("custom").notNull(),
    // language | framework | purpose | custom
    usageCount: integer("usage_count").default(0).notNull(),
    createdAt: timestamp("created_at", { withTimezone: true }).defaultNow().notNull(),
  },
  (t) => [
    uniqueIndex("tag_org_name_idx").on(t.orgId, t.name),
    index("tag_org_idx").on(t.orgId),
  ],
);

// ---------------------------------------------------------------------------
// Snippets
// ---------------------------------------------------------------------------
export const snippets = pgTable(
  "snippets",
  {
    id: uuid("id").primaryKey().defaultRandom(),
    orgId: uuid("org_id")
      .references(() => organizations.id, { onDelete: "cascade" })
      .notNull(),
    collectionId: uuid("collection_id")
      .references(() => collections.id, { onDelete: "set null" }),
    createdByUserId: text("created_by_user_id").notNull(), // Clerk user ID
    title: text("title").notNull(),
    description: text("description"), // markdown
    isFavorite: boolean("is_favorite").default(false).notNull(),
    isPublic: boolean("is_public").default(false).notNull(),
    source: text("source").default("web").notNull(),
    // web | vscode | cli | browser | slack | api
    sourceUrl: text("source_url"), // original page URL if from browser/web
    accessCount: integer("access_count").default(0).notNull(),
    lastAccessedAt: timestamp("last_accessed_at", { withTimezone: true }),
    createdAt: timestamp("created_at", { withTimezone: true }).defaultNow().notNull(),
    updatedAt: timestamp("updated_at", { withTimezone: true }).defaultNow().notNull(),
  },
  (t) => [
    index("snippet_org_idx").on(t.orgId),
    index("snippet_collection_idx").on(t.collectionId),
    index("snippet_created_by_idx").on(t.createdByUserId),
    index("snippet_favorite_idx").on(t.isFavorite),
    index("snippet_created_idx").on(t.createdAt),
  ],
);

// ---------------------------------------------------------------------------
// Snippet Files (code content, one or more per snippet)
// ---------------------------------------------------------------------------
export const snippetFiles = pgTable(
  "snippet_files",
  {
    id: uuid("id").primaryKey().defaultRandom(),
    snippetId: uuid("snippet_id")
      .references(() => snippets.id, { onDelete: "cascade" })
      .notNull(),
    filename: text("filename").notNull(), // e.g. "index.ts" or "main.py"
    language: text("language").notNull(), // detected or user-set
    content: text("content").notNull(),
    sortOrder: integer("sort_order").default(0).notNull(),
    embedding: vector("embedding", { dimensions: 1536 }),
    createdAt: timestamp("created_at", { withTimezone: true }).defaultNow().notNull(),
    updatedAt: timestamp("updated_at", { withTimezone: true }).defaultNow().notNull(),
  },
  (t) => [
    index("snippet_file_snippet_idx").on(t.snippetId),
    index("snippet_file_embedding_idx").using(
      "hnsw",
      t.embedding.op("vector_cosine_ops"),
    ),
  ],
);

// ---------------------------------------------------------------------------
// Snippet Tags (many-to-many)
// ---------------------------------------------------------------------------
export const snippetTags = pgTable(
  "snippet_tags",
  {
    id: uuid("id").primaryKey().defaultRandom(),
    snippetId: uuid("snippet_id")
      .references(() => snippets.id, { onDelete: "cascade" })
      .notNull(),
    tagId: uuid("tag_id")
      .references(() => tags.id, { onDelete: "cascade" })
      .notNull(),
    source: text("source").default("manual").notNull(), // ai | manual
  },
  (t) => [
    uniqueIndex("snippet_tag_pair_idx").on(t.snippetId, t.tagId),
    index("snippet_tag_snippet_idx").on(t.snippetId),
    index("snippet_tag_tag_idx").on(t.tagId),
  ],
);

// ---------------------------------------------------------------------------
// Snippet Versions (edit history)
// ---------------------------------------------------------------------------
export const snippetVersions = pgTable(
  "snippet_versions",
  {
    id: uuid("id").primaryKey().defaultRandom(),
    snippetId: uuid("snippet_id")
      .references(() => snippets.id, { onDelete: "cascade" })
      .notNull(),
    versionNumber: integer("version_number").notNull(),
    contentSnapshot: jsonb("content_snapshot").notNull().$type<
      Array<{
        filename: string;
        language: string;
        content: string;
      }>
    >(),
    changeNote: text("change_note"),
    createdByUserId: text("created_by_user_id").notNull(),
    createdAt: timestamp("created_at", { withTimezone: true }).defaultNow().notNull(),
  },
  (t) => [
    index("snippet_version_snippet_idx").on(t.snippetId),
    uniqueIndex("snippet_version_num_idx").on(t.snippetId, t.versionNumber),
  ],
);

// ---------------------------------------------------------------------------
// Import Jobs (Gist import, bulk import)
// ---------------------------------------------------------------------------
export const importJobs = pgTable(
  "import_jobs",
  {
    id: uuid("id").primaryKey().defaultRandom(),
    orgId: uuid("org_id")
      .references(() => organizations.id, { onDelete: "cascade" })
      .notNull(),
    source: text("source").notNull(), // gists | vscode | markdown | bulk
    status: text("status").default("pending").notNull(),
    // pending | processing | completed | failed
    totalItems: integer("total_items").default(0).notNull(),
    processedItems: integer("processed_items").default(0).notNull(),
    failedItems: integer("failed_items").default(0).notNull(),
    config: jsonb("config").default({}).$type<{
      githubAccessToken?: string; // encrypted
      collectionId?: string;
    }>(),
    errorMessage: text("error_message"),
    createdAt: timestamp("created_at", { withTimezone: true }).defaultNow().notNull(),
    completedAt: timestamp("completed_at", { withTimezone: true }),
  },
  (t) => [
    index("import_org_idx").on(t.orgId),
  ],
);

// ---------------------------------------------------------------------------
// Device Auth Codes (for VS Code extension)
// ---------------------------------------------------------------------------
export const deviceAuthCodes = pgTable(
  "device_auth_codes",
  {
    id: uuid("id").primaryKey().defaultRandom(),
    userCode: text("user_code").unique().notNull(), // 8-char display code
    deviceCode: text("device_code").unique().notNull(), // 32-char polling code
    clerkUserId: text("clerk_user_id"), // set when user authorizes
    orgId: uuid("org_id"), // set when user authorizes
    status: text("status").default("pending").notNull(), // pending | authorized | expired
    expiresAt: timestamp("expires_at", { withTimezone: true }).notNull(),
    createdAt: timestamp("created_at", { withTimezone: true }).defaultNow().notNull(),
  },
  (t) => [
    index("device_auth_status_idx").on(t.status),
  ],
);

// ---------------------------------------------------------------------------
// Activity Feed (for team workspaces)
// ---------------------------------------------------------------------------
export const activityFeed = pgTable(
  "activity_feed",
  {
    id: uuid("id").primaryKey().defaultRandom(),
    orgId: uuid("org_id")
      .references(() => organizations.id, { onDelete: "cascade" })
      .notNull(),
    userId: text("user_id").notNull(),
    action: text("action").notNull(), // created | updated | deleted | imported | shared
    targetType: text("target_type").notNull(), // snippet | collection | tag
    targetId: uuid("target_id").notNull(),
    targetTitle: text("target_title"),
    metadata: jsonb("metadata").default({}).$type<Record<string, unknown>>(),
    createdAt: timestamp("created_at", { withTimezone: true }).defaultNow().notNull(),
  },
  (t) => [
    index("activity_org_idx").on(t.orgId),
    index("activity_created_idx").on(t.createdAt),
  ],
);

// ---------------------------------------------------------------------------
// Relations
// ---------------------------------------------------------------------------
export const organizationsRelations = relations(organizations, ({ many }) => ({
  workspaceMembers: many(workspaceMembers),
  collections: many(collections),
  tags: many(tags),
  snippets: many(snippets),
  importJobs: many(importJobs),
  activityFeed: many(activityFeed),
}));

export const workspaceMembersRelations = relations(workspaceMembers, ({ one }) => ({
  organization: one(organizations, {
    fields: [workspaceMembers.orgId],
    references: [organizations.id],
  }),
}));

export const collectionsRelations = relations(collections, ({ one, many }) => ({
  organization: one(organizations, {
    fields: [collections.orgId],
    references: [organizations.id],
  }),
  parentCollection: one(collections, {
    fields: [collections.parentCollectionId],
    references: [collections.id],
    relationName: "parentChild",
  }),
  childCollections: many(collections, { relationName: "parentChild" }),
  snippets: many(snippets),
}));

export const tagsRelations = relations(tags, ({ one, many }) => ({
  organization: one(organizations, {
    fields: [tags.orgId],
    references: [organizations.id],
  }),
  snippetTags: many(snippetTags),
}));

export const snippetsRelations = relations(snippets, ({ one, many }) => ({
  organization: one(organizations, {
    fields: [snippets.orgId],
    references: [organizations.id],
  }),
  collection: one(collections, {
    fields: [snippets.collectionId],
    references: [collections.id],
  }),
  files: many(snippetFiles),
  snippetTags: many(snippetTags),
  versions: many(snippetVersions),
}));

export const snippetFilesRelations = relations(snippetFiles, ({ one }) => ({
  snippet: one(snippets, {
    fields: [snippetFiles.snippetId],
    references: [snippets.id],
  }),
}));

export const snippetTagsRelations = relations(snippetTags, ({ one }) => ({
  snippet: one(snippets, {
    fields: [snippetTags.snippetId],
    references: [snippets.id],
  }),
  tag: one(tags, {
    fields: [snippetTags.tagId],
    references: [tags.id],
  }),
}));

export const snippetVersionsRelations = relations(snippetVersions, ({ one }) => ({
  snippet: one(snippets, {
    fields: [snippetVersions.snippetId],
    references: [snippets.id],
  }),
}));

export const importJobsRelations = relations(importJobs, ({ one }) => ({
  organization: one(organizations, {
    fields: [importJobs.orgId],
    references: [organizations.id],
  }),
}));

export const activityFeedRelations = relations(activityFeed, ({ one }) => ({
  organization: one(organizations, {
    fields: [activityFeed.orgId],
    references: [organizations.id],
  }),
}));
```

---

## Initial Migration SQL

```sql
-- Enable required extensions
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
CREATE EXTENSION IF NOT EXISTS "vector";
CREATE EXTENSION IF NOT EXISTS "pg_trgm";

-- Organizations
CREATE TABLE organizations (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  clerk_org_id TEXT UNIQUE NOT NULL,
  name TEXT NOT NULL,
  slug TEXT UNIQUE NOT NULL,
  type TEXT NOT NULL DEFAULT 'personal',
  plan TEXT NOT NULL DEFAULT 'free',
  stripe_customer_id TEXT,
  stripe_subscription_id TEXT,
  settings JSONB DEFAULT '{}',
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Workspace Members
CREATE TABLE workspace_members (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  org_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
  clerk_user_id TEXT NOT NULL,
  role TEXT NOT NULL DEFAULT 'editor',
  name TEXT NOT NULL,
  email TEXT NOT NULL,
  avatar_url TEXT,
  joined_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE UNIQUE INDEX wm_org_user_idx ON workspace_members(org_id, clerk_user_id);
CREATE INDEX wm_org_idx ON workspace_members(org_id);

-- Collections
CREATE TABLE collections (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  org_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
  name TEXT NOT NULL,
  description TEXT,
  icon TEXT,
  parent_collection_id UUID,
  is_smart BOOLEAN NOT NULL DEFAULT false,
  smart_rules JSONB,
  sort_order INTEGER NOT NULL DEFAULT 0,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX collection_org_idx ON collections(org_id);
CREATE INDEX collection_parent_idx ON collections(parent_collection_id);

-- Tags
CREATE TABLE tags (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  org_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
  name TEXT NOT NULL,
  category TEXT NOT NULL DEFAULT 'custom',
  usage_count INTEGER NOT NULL DEFAULT 0,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE UNIQUE INDEX tag_org_name_idx ON tags(org_id, name);
CREATE INDEX tag_org_idx ON tags(org_id);

-- Snippets
CREATE TABLE snippets (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  org_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
  collection_id UUID REFERENCES collections(id) ON DELETE SET NULL,
  created_by_user_id TEXT NOT NULL,
  title TEXT NOT NULL,
  description TEXT,
  is_favorite BOOLEAN NOT NULL DEFAULT false,
  is_public BOOLEAN NOT NULL DEFAULT false,
  source TEXT NOT NULL DEFAULT 'web',
  source_url TEXT,
  access_count INTEGER NOT NULL DEFAULT 0,
  last_accessed_at TIMESTAMPTZ,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX snippet_org_idx ON snippets(org_id);
CREATE INDEX snippet_collection_idx ON snippets(collection_id);
CREATE INDEX snippet_created_by_idx ON snippets(created_by_user_id);
CREATE INDEX snippet_favorite_idx ON snippets(is_favorite);
CREATE INDEX snippet_created_idx ON snippets(created_at);

-- Snippet Files
CREATE TABLE snippet_files (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  snippet_id UUID NOT NULL REFERENCES snippets(id) ON DELETE CASCADE,
  filename TEXT NOT NULL,
  language TEXT NOT NULL,
  content TEXT NOT NULL,
  sort_order INTEGER NOT NULL DEFAULT 0,
  embedding vector(1536),
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX snippet_file_snippet_idx ON snippet_files(snippet_id);
CREATE INDEX snippet_file_embedding_idx ON snippet_files
  USING hnsw (embedding vector_cosine_ops);
-- Trigram index for fuzzy text search
CREATE INDEX snippet_file_content_trgm_idx ON snippet_files
  USING gin (content gin_trgm_ops);

-- Snippet Tags
CREATE TABLE snippet_tags (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  snippet_id UUID NOT NULL REFERENCES snippets(id) ON DELETE CASCADE,
  tag_id UUID NOT NULL REFERENCES tags(id) ON DELETE CASCADE,
  source TEXT NOT NULL DEFAULT 'manual'
);
CREATE UNIQUE INDEX snippet_tag_pair_idx ON snippet_tags(snippet_id, tag_id);
CREATE INDEX snippet_tag_snippet_idx ON snippet_tags(snippet_id);
CREATE INDEX snippet_tag_tag_idx ON snippet_tags(tag_id);

-- Snippet Versions
CREATE TABLE snippet_versions (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  snippet_id UUID NOT NULL REFERENCES snippets(id) ON DELETE CASCADE,
  version_number INTEGER NOT NULL,
  content_snapshot JSONB NOT NULL,
  change_note TEXT,
  created_by_user_id TEXT NOT NULL,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX snippet_version_snippet_idx ON snippet_versions(snippet_id);
CREATE UNIQUE INDEX snippet_version_num_idx ON snippet_versions(snippet_id, version_number);

-- Import Jobs
CREATE TABLE import_jobs (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  org_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
  source TEXT NOT NULL,
  status TEXT NOT NULL DEFAULT 'pending',
  total_items INTEGER NOT NULL DEFAULT 0,
  processed_items INTEGER NOT NULL DEFAULT 0,
  failed_items INTEGER NOT NULL DEFAULT 0,
  config JSONB DEFAULT '{}',
  error_message TEXT,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  completed_at TIMESTAMPTZ
);
CREATE INDEX import_org_idx ON import_jobs(org_id);

-- Device Auth Codes
CREATE TABLE device_auth_codes (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  user_code TEXT UNIQUE NOT NULL,
  device_code TEXT UNIQUE NOT NULL,
  clerk_user_id TEXT,
  org_id UUID,
  status TEXT NOT NULL DEFAULT 'pending',
  expires_at TIMESTAMPTZ NOT NULL,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX device_auth_status_idx ON device_auth_codes(status);

-- Activity Feed
CREATE TABLE activity_feed (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  org_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
  user_id TEXT NOT NULL,
  action TEXT NOT NULL,
  target_type TEXT NOT NULL,
  target_id UUID NOT NULL,
  target_title TEXT,
  metadata JSONB DEFAULT '{}',
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX activity_org_idx ON activity_feed(org_id);
CREATE INDEX activity_created_idx ON activity_feed(created_at);

-- Full-text search index on snippet titles and descriptions
CREATE INDEX snippet_title_trgm_idx ON snippets
  USING gin (title gin_trgm_ops);
```

---

## Architecture Deep-Dives

### 1. Hybrid Search Engine (pgvector + pg_trgm + RRF)

Search queries run two parallel searches: semantic (pgvector cosine similarity) and text (pg_trgm trigram matching). Results are merged using Reciprocal Rank Fusion to produce a single ranked list.

```typescript
// src/server/services/search.ts

import { sql, eq, and, inArray } from "drizzle-orm";
import { OpenAI } from "openai";
import { db } from "../db";
import { snippets, snippetFiles, snippetTags, tags } from "../db/schema";

const openai = new OpenAI();

interface SearchResult {
  snippetId: string;
  title: string;
  description: string | null;
  language: string;
  score: number;
  matchType: "semantic" | "text" | "hybrid";
  preview: string; // first 200 chars of matching file
}

interface SearchOptions {
  orgId: string;
  query: string;
  language?: string;
  tagIds?: string[];
  collectionId?: string;
  limit?: number;
}

export async function hybridSearch(
  options: SearchOptions,
): Promise<SearchResult[]> {
  const { orgId, query, language, tagIds, collectionId, limit = 20 } = options;
  const k = 60; // RRF constant

  // Run semantic and text search in parallel
  const [semanticResults, textResults] = await Promise.all([
    semanticSearch(orgId, query, language, limit * 2),
    textSearch(orgId, query, language, limit * 2),
  ]);

  // Apply collection and tag filters
  const filteredSemantic = await applyFilters(semanticResults, {
    collectionId,
    tagIds,
  });
  const filteredText = await applyFilters(textResults, {
    collectionId,
    tagIds,
  });

  // Reciprocal Rank Fusion
  const rrfScores = new Map<string, { score: number; result: SearchResult }>();

  filteredSemantic.forEach((result, rank) => {
    const rrfScore = 1 / (k + rank + 1);
    const existing = rrfScores.get(result.snippetId);
    if (existing) {
      existing.score += rrfScore;
      existing.result.matchType = "hybrid";
    } else {
      rrfScores.set(result.snippetId, {
        score: rrfScore,
        result: { ...result, matchType: "semantic" },
      });
    }
  });

  filteredText.forEach((result, rank) => {
    const rrfScore = 1 / (k + rank + 1);
    const existing = rrfScores.get(result.snippetId);
    if (existing) {
      existing.score += rrfScore;
      existing.result.matchType = "hybrid";
    } else {
      rrfScores.set(result.snippetId, {
        score: rrfScore,
        result: { ...result, matchType: "text" },
      });
    }
  });

  // Sort by RRF score and return top results
  return Array.from(rrfScores.values())
    .sort((a, b) => b.score - a.score)
    .slice(0, limit)
    .map((entry) => ({ ...entry.result, score: entry.score }));
}

async function semanticSearch(
  orgId: string,
  query: string,
  language: string | undefined,
  limit: number,
): Promise<SearchResult[]> {
  // Generate embedding for query
  const embeddingResult = await openai.embeddings.create({
    model: "text-embedding-3-small",
    input: query,
  });
  const queryEmbedding = embeddingResult.data[0].embedding;

  const languageFilter = language
    ? sql`AND sf.language = ${language}`
    : sql``;

  const results = await db.execute<{
    snippet_id: string;
    title: string;
    description: string | null;
    language: string;
    similarity: number;
    preview: string;
  }>(sql`
    SELECT DISTINCT ON (s.id)
      s.id as snippet_id,
      s.title,
      s.description,
      sf.language,
      1 - (sf.embedding <=> ${sql.raw(`'[${queryEmbedding.join(",")}]'::vector`)}) as similarity,
      LEFT(sf.content, 200) as preview
    FROM snippet_files sf
    JOIN snippets s ON s.id = sf.snippet_id
    WHERE s.org_id = ${orgId}
      AND sf.embedding IS NOT NULL
      ${languageFilter}
    ORDER BY s.id, sf.embedding <=> ${sql.raw(`'[${queryEmbedding.join(",")}]'::vector`)}
    LIMIT ${limit}
  `);

  return results.rows
    .filter((r) => r.similarity > 0.3)
    .map((r) => ({
      snippetId: r.snippet_id,
      title: r.title,
      description: r.description,
      language: r.language,
      score: r.similarity,
      matchType: "semantic" as const,
      preview: r.preview,
    }));
}

async function textSearch(
  orgId: string,
  query: string,
  language: string | undefined,
  limit: number,
): Promise<SearchResult[]> {
  const languageFilter = language
    ? sql`AND sf.language = ${language}`
    : sql``;

  const results = await db.execute<{
    snippet_id: string;
    title: string;
    description: string | null;
    language: string;
    similarity: number;
    preview: string;
  }>(sql`
    SELECT DISTINCT ON (s.id)
      s.id as snippet_id,
      s.title,
      s.description,
      sf.language,
      GREATEST(
        similarity(s.title, ${query}),
        similarity(sf.content, ${query})
      ) as similarity,
      LEFT(sf.content, 200) as preview
    FROM snippet_files sf
    JOIN snippets s ON s.id = sf.snippet_id
    WHERE s.org_id = ${orgId}
      AND (
        s.title % ${query}
        OR sf.content % ${query}
      )
      ${languageFilter}
    ORDER BY s.id, similarity DESC
    LIMIT ${limit}
  `);

  return results.rows.map((r) => ({
    snippetId: r.snippet_id,
    title: r.title,
    description: r.description,
    language: r.language,
    score: r.similarity,
    matchType: "text" as const,
    preview: r.preview,
  }));
}

async function applyFilters(
  results: SearchResult[],
  filters: { collectionId?: string; tagIds?: string[] },
): Promise<SearchResult[]> {
  if (!filters.collectionId && (!filters.tagIds || filters.tagIds.length === 0)) {
    return results;
  }

  const snippetIds = results.map((r) => r.snippetId);
  if (snippetIds.length === 0) return [];

  let filteredIds = new Set(snippetIds);

  if (filters.collectionId) {
    const inCollection = await db.query.snippets.findMany({
      where: and(
        inArray(snippets.id, snippetIds),
        eq(snippets.collectionId, filters.collectionId),
      ),
      columns: { id: true },
    });
    filteredIds = new Set(inCollection.map((s) => s.id));
  }

  if (filters.tagIds && filters.tagIds.length > 0) {
    const withTags = await db.query.snippetTags.findMany({
      where: and(
        inArray(snippetTags.snippetId, Array.from(filteredIds)),
        inArray(snippetTags.tagId, filters.tagIds),
      ),
      columns: { snippetId: true },
    });
    filteredIds = new Set(withTags.map((st) => st.snippetId));
  }

  return results.filter((r) => filteredIds.has(r.snippetId));
}
```

### 2. AI Auto-Tagging Pipeline

When a snippet is saved, a BullMQ worker analyzes the code and generates tags using GPT-4o-mini. Tags are created if they don't exist and linked to the snippet.

```typescript
// src/server/workers/snippet-processor.ts

import { Worker, Queue } from "bullmq";
import { OpenAI } from "openai";
import { eq, and } from "drizzle-orm";
import { db } from "../db";
import { snippets, snippetFiles, tags, snippetTags } from "../db/schema";

const openai = new OpenAI();

interface SnippetProcessingJob {
  snippetId: string;
  orgId: string;
}

export const snippetProcessingQueue = new Queue<SnippetProcessingJob>(
  "snippet-processing",
  { connection: { url: process.env.UPSTASH_REDIS_URL } },
);

const worker = new Worker<SnippetProcessingJob>(
  "snippet-processing",
  async (job) => {
    const { snippetId, orgId } = job.data;

    const snippet = await db.query.snippets.findFirst({
      where: eq(snippets.id, snippetId),
      with: { files: true },
    });
    if (!snippet) return;

    // Step 1: Generate embeddings for each file
    for (const file of snippet.files) {
      const textToEmbed = `${snippet.title}\n${snippet.description || ""}\n${file.content}`;
      const embeddingResult = await openai.embeddings.create({
        model: "text-embedding-3-small",
        input: textToEmbed.slice(0, 8000), // token limit safety
      });

      await db
        .update(snippetFiles)
        .set({ embedding: embeddingResult.data[0].embedding })
        .where(eq(snippetFiles.id, file.id));
    }

    // Step 2: AI auto-tagging
    const codePreview = snippet.files
      .map((f) => `// ${f.filename} (${f.language})\n${f.content.slice(0, 500)}`)
      .join("\n\n");

    const tagResult = await openai.chat.completions.create({
      model: "gpt-4o-mini",
      temperature: 0.1,
      response_format: { type: "json_object" },
      messages: [
        {
          role: "system",
          content: `Analyze this code snippet and return JSON with tags:
{
  "language": "the primary programming language",
  "framework": "framework if identifiable (e.g., react, express, django)" or null,
  "purpose": ["2-4 purpose tags from: authentication, database, api, testing, error-handling, validation, caching, logging, config, deployment, regex, parsing, encryption, file-io, date-time, networking, ui, animation, state-management, routing, middleware, cli, scraping, data-transform"],
  "description": "a one-sentence description if the snippet has no description" or null,
  "complexity": "beginner" | "intermediate" | "advanced"
}`,
        },
        {
          role: "user",
          content: `Title: ${snippet.title}
Description: ${snippet.description || "(none)"}

Code:
${codePreview}`,
        },
      ],
    });

    const classification = JSON.parse(
      tagResult.choices[0].message.content || "{}",
    );

    // Step 3: Create/link tags
    const tagNames: Array<{ name: string; category: string }> = [];

    if (classification.language) {
      tagNames.push({ name: classification.language.toLowerCase(), category: "language" });
    }
    if (classification.framework) {
      tagNames.push({ name: classification.framework.toLowerCase(), category: "framework" });
    }
    if (classification.purpose) {
      for (const purpose of classification.purpose) {
        tagNames.push({ name: purpose.toLowerCase(), category: "purpose" });
      }
    }

    for (const tagDef of tagNames) {
      // Upsert tag
      let tag = await db.query.tags.findFirst({
        where: and(
          eq(tags.orgId, orgId),
          eq(tags.name, tagDef.name),
        ),
      });

      if (!tag) {
        const [newTag] = await db
          .insert(tags)
          .values({
            orgId,
            name: tagDef.name,
            category: tagDef.category,
          })
          .returning();
        tag = newTag;
      }

      // Link tag to snippet (ignore if already exists)
      await db
        .insert(snippetTags)
        .values({
          snippetId,
          tagId: tag.id,
          source: "ai",
        })
        .onConflictDoNothing();

      // Increment usage count
      await db
        .update(tags)
        .set({ usageCount: sql`usage_count + 1` })
        .where(eq(tags.id, tag.id));
    }

    // Step 4: Update description if none provided
    if (!snippet.description && classification.description) {
      await db
        .update(snippets)
        .set({
          description: classification.description,
          updatedAt: new Date(),
        })
        .where(eq(snippets.id, snippetId));
    }
  },
  {
    connection: { url: process.env.UPSTASH_REDIS_URL },
    concurrency: 5,
    limiter: { max: 20, duration: 60_000 },
  },
);
```

### 3. VS Code Extension (Save + Search + Insert)

The extension communicates with the SnipVault API using device code flow authentication. Users can save selected code, search their library, and insert snippets directly into the editor.

```typescript
// vscode-extension/src/extension.ts

import * as vscode from "vscode";
import { SnipVaultAPI } from "./api";
import { DeviceCodeAuth } from "./auth";

let api: SnipVaultAPI;

export function activate(context: vscode.ExtensionContext) {
  const auth = new DeviceCodeAuth(context.globalState);
  api = new SnipVaultAPI(auth);

  // Command: Authenticate with SnipVault
  context.subscriptions.push(
    vscode.commands.registerCommand("snipvault.authenticate", async () => {
      const { userCode, verificationUrl } = await auth.startDeviceFlow();

      const action = await vscode.window.showInformationMessage(
        `Enter code ${userCode} at ${verificationUrl}`,
        "Open Browser",
        "Copy Code",
      );

      if (action === "Open Browser") {
        vscode.env.openExternal(vscode.Uri.parse(verificationUrl));
      } else if (action === "Copy Code") {
        await vscode.env.clipboard.writeText(userCode);
      }

      // Poll for authorization
      try {
        await auth.pollForAuthorization();
        vscode.window.showInformationMessage("SnipVault: Authentication successful!");
      } catch (err) {
        vscode.window.showErrorMessage("SnipVault: Authentication failed or expired.");
      }
    }),
  );

  // Command: Save selected code as snippet
  context.subscriptions.push(
    vscode.commands.registerCommand("snipvault.saveSnippet", async () => {
      const editor = vscode.window.activeTextEditor;
      if (!editor) {
        vscode.window.showWarningMessage("No active editor");
        return;
      }

      const selection = editor.selection;
      const selectedText = editor.document.getText(selection);
      if (!selectedText) {
        vscode.window.showWarningMessage("No text selected");
        return;
      }

      const language = editor.document.languageId;
      const filename = editor.document.fileName.split("/").pop() || "snippet";

      const title = await vscode.window.showInputBox({
        prompt: "Snippet title",
        value: `${filename} snippet`,
      });
      if (!title) return;

      const description = await vscode.window.showInputBox({
        prompt: "Description (optional)",
      });

      try {
        const snippet = await api.createSnippet({
          title,
          description: description || undefined,
          source: "vscode",
          sourceUrl: editor.document.uri.toString(),
          files: [
            {
              filename,
              language,
              content: selectedText,
              sortOrder: 0,
            },
          ],
        });

        vscode.window.showInformationMessage(
          `Saved "${title}" to SnipVault`,
        );
      } catch (err) {
        vscode.window.showErrorMessage(
          `Failed to save snippet: ${err instanceof Error ? err.message : String(err)}`,
        );
      }
    }),
  );

  // Command: Search and insert snippet
  context.subscriptions.push(
    vscode.commands.registerCommand("snipvault.searchAndInsert", async () => {
      const query = await vscode.window.showInputBox({
        prompt: "Search SnipVault",
        placeHolder: "e.g. postgres connection pool with retry",
      });
      if (!query) return;

      try {
        const results = await api.search(query);

        if (results.length === 0) {
          vscode.window.showInformationMessage("No snippets found");
          return;
        }

        const picked = await vscode.window.showQuickPick(
          results.map((r) => ({
            label: r.title,
            description: r.language,
            detail: r.preview,
            snippetId: r.snippetId,
          })),
          { placeHolder: "Select a snippet to insert" },
        );

        if (!picked) return;

        // Fetch full snippet content
        const snippet = await api.getSnippet(picked.snippetId);
        const content = snippet.files[0]?.content || "";

        // Insert at cursor position
        const editor = vscode.window.activeTextEditor;
        if (editor) {
          await editor.edit((editBuilder) => {
            editBuilder.insert(editor.selection.active, content);
          });

          // Track access
          await api.trackAccess(picked.snippetId);
        }
      } catch (err) {
        vscode.window.showErrorMessage(
          `Search failed: ${err instanceof Error ? err.message : String(err)}`,
        );
      }
    }),
  );

  // Sidebar: Recent/favorite snippets panel
  const snippetProvider = new SnippetTreeProvider(api);
  vscode.window.createTreeView("snipvault-snippets", {
    treeDataProvider: snippetProvider,
  });

  context.subscriptions.push(
    vscode.commands.registerCommand("snipvault.refreshSidebar", () => {
      snippetProvider.refresh();
    }),
  );
}

class SnippetTreeProvider implements vscode.TreeDataProvider<SnippetTreeItem> {
  private _onDidChangeTreeData = new vscode.EventEmitter<void>();
  readonly onDidChangeTreeData = this._onDidChangeTreeData.event;

  constructor(private api: SnipVaultAPI) {}

  refresh() {
    this._onDidChangeTreeData.fire();
  }

  async getChildren(element?: SnippetTreeItem): Promise<SnippetTreeItem[]> {
    if (element) return [];

    try {
      const [favorites, recent] = await Promise.all([
        this.api.listSnippets({ favorite: true, limit: 5 }),
        this.api.listSnippets({ sort: "recent", limit: 10 }),
      ]);

      return [
        new SnippetTreeItem("⭐ Favorites", vscode.TreeItemCollapsibleState.Expanded),
        ...favorites.map(
          (s) =>
            new SnippetTreeItem(
              s.title,
              vscode.TreeItemCollapsibleState.None,
              s.id,
              s.files[0]?.language,
            ),
        ),
        new SnippetTreeItem("🕐 Recent", vscode.TreeItemCollapsibleState.Expanded),
        ...recent.map(
          (s) =>
            new SnippetTreeItem(
              s.title,
              vscode.TreeItemCollapsibleState.None,
              s.id,
              s.files[0]?.language,
            ),
        ),
      ];
    } catch {
      return [new SnippetTreeItem("Sign in to see snippets")];
    }
  }

  getTreeItem(element: SnippetTreeItem) {
    return element;
  }
}

class SnippetTreeItem extends vscode.TreeItem {
  constructor(
    label: string,
    collapsibleState = vscode.TreeItemCollapsibleState.None,
    public snippetId?: string,
    public language?: string,
  ) {
    super(label, collapsibleState);
    if (language) this.description = language;
    if (snippetId) {
      this.command = {
        command: "snipvault.insertFromSidebar",
        title: "Insert Snippet",
        arguments: [snippetId],
      };
    }
  }
}

export function deactivate() {}
```

### 4. GitHub Gists Import Pipeline

The import pipeline fetches all gists from a user's GitHub account, converts them into SnipVault snippets, and processes them through the AI tagging pipeline.

```typescript
// src/server/services/gist-import.ts

import { Octokit } from "@octokit/rest";
import { eq } from "drizzle-orm";
import { db } from "../db";
import { importJobs, snippets, snippetFiles } from "../db/schema";
import { snippetProcessingQueue } from "../workers/snippet-processor";
import { decrypt } from "../lib/encryption";

interface GistFile {
  filename: string;
  language: string | null;
  content: string;
  size: number;
}

export async function processGistImport(importJobId: string): Promise<void> {
  const job = await db.query.importJobs.findFirst({
    where: eq(importJobs.id, importJobId),
  });
  if (!job) return;

  await db
    .update(importJobs)
    .set({ status: "processing" })
    .where(eq(importJobs.id, importJobId));

  try {
    const token = decrypt(job.config?.githubAccessToken || "");
    const octokit = new Octokit({ auth: token });

    // Fetch all gists (paginated)
    const allGists: any[] = [];
    let page = 1;
    while (true) {
      const { data } = await octokit.gists.list({ per_page: 100, page });
      allGists.push(...data);
      if (data.length < 100) break;
      page++;
    }

    await db
      .update(importJobs)
      .set({ totalItems: allGists.length })
      .where(eq(importJobs.id, importJobId));

    let processed = 0;
    let failed = 0;

    for (const gist of allGists) {
      try {
        // Fetch full gist content
        const { data: fullGist } = await octokit.gists.get({ gist_id: gist.id });
        const files = Object.values(fullGist.files || {}) as GistFile[];

        if (files.length === 0) {
          processed++;
          continue;
        }

        // Create snippet
        const title = gist.description || files[0]?.filename || "Untitled Gist";
        const primaryLanguage = files[0]?.language?.toLowerCase() || "text";

        const [snippet] = await db
          .insert(snippets)
          .values({
            orgId: job.orgId,
            createdByUserId: "import",
            title,
            description: gist.description || undefined,
            source: "web",
            sourceUrl: gist.html_url,
            collectionId: job.config?.collectionId || null,
          })
          .returning();

        // Create snippet files
        for (let i = 0; i < files.length; i++) {
          const file = files[i];
          if (!file.content) continue;

          await db.insert(snippetFiles).values({
            snippetId: snippet.id,
            filename: file.filename,
            language: file.language?.toLowerCase() || "text",
            content: file.content,
            sortOrder: i,
          });
        }

        // Enqueue for AI processing
        await snippetProcessingQueue.add("process", {
          snippetId: snippet.id,
          orgId: job.orgId,
        });

        processed++;
      } catch (err) {
        failed++;
        console.error(`Failed to import gist ${gist.id}:`, err);
      }

      // Update progress
      await db
        .update(importJobs)
        .set({ processedItems: processed, failedItems: failed })
        .where(eq(importJobs.id, importJobId));
    }

    await db
      .update(importJobs)
      .set({
        status: "completed",
        processedItems: processed,
        failedItems: failed,
        completedAt: new Date(),
      })
      .where(eq(importJobs.id, importJobId));
  } catch (err) {
    await db
      .update(importJobs)
      .set({
        status: "failed",
        errorMessage: err instanceof Error ? err.message : String(err),
      })
      .where(eq(importJobs.id, importJobId));
  }
}
```

---

## Phase Breakdown

### Phase 1 — Project Scaffold + Auth + DB (Days 1–4)

**Day 1: Repository and framework setup**
```bash
npx create-next-app@latest snipvault --ts --tailwind --app --src-dir
cd snipvault
npm i drizzle-orm @neondatabase/serverless
npm i -D drizzle-kit
npm i @trpc/server @trpc/client @trpc/next @trpc/react-query superjson zod
npm i @clerk/nextjs
npm i @monaco-editor/react
```
- `src/app/layout.tsx` — Clerk provider, tRPC provider
- `src/app/(dashboard)/layout.tsx` — authenticated shell with sidebar
- `src/server/db/index.ts` — Neon connection with pgvector + pg_trgm
- `drizzle.config.ts`
- `.env.local`

**Day 2: Full database schema**
- `src/server/db/schema.ts` — all 11 tables: organizations, workspaceMembers, collections, tags, snippets, snippetFiles, snippetTags, snippetVersions, importJobs, deviceAuthCodes, activityFeed
- All `relations()` blocks
- HNSW index on embeddings, pg_trgm indexes on content and title
- Run `npx drizzle-kit push`

**Day 3: tRPC setup and org initialization**
- `src/server/trpc.ts` — context, auth middleware, org-scoped procedure
- `src/server/routers/_app.ts` — root router
- `src/server/routers/org.ts` — getOrCreate, update settings

**Day 4: Auth flows and org seed**
- `src/app/(auth)/sign-in/[[...sign-in]]/page.tsx`
- `src/app/(auth)/sign-up/[[...sign-up]]/page.tsx`
- `src/server/lib/org-seed.ts` — create personal workspace + default "Inbox" collection

### Phase 2 — Snippet CRUD + Editor (Days 5–9)

**Day 5: Snippet creation**
- `src/server/routers/snippet.ts` — create snippet with files, detect language if not provided
- `src/app/(dashboard)/snippets/new/page.tsx` — create page: title, description, code editor (Monaco), language selector
- `src/components/snippets/SnippetForm.tsx` — form with Monaco Editor

**Day 6: Snippet list and detail**
- `src/server/routers/snippet.ts` — list (paginated, filtered by collection/tag/language/favorite), get by ID (with files, tags, versions)
- `src/app/(dashboard)/page.tsx` — dashboard: search bar, recent snippets, favorites
- `src/app/(dashboard)/snippets/page.tsx` — snippet library with grid/list toggle

**Day 7: Snippet detail and editor**
- `src/app/(dashboard)/snippets/[id]/page.tsx` — snippet detail: editable title, description (markdown), tabbed code files (Monaco), tags
- `src/components/snippets/CodeEditor.tsx` — Monaco Editor wrapper with language detection, copy button, theme
- `src/components/snippets/TagEditor.tsx` — tag chips with add/remove, autocomplete

**Day 8: Snippet update and version history**
- `src/server/routers/snippet.ts` — update (auto-creates version snapshot), delete, toggle favorite
- `src/server/routers/snippet-version.ts` — list versions, get version, diff between versions
- `src/components/snippets/VersionHistory.tsx` — version timeline with restore button

**Day 9: Collection management**
- `src/server/routers/collection.ts` — CRUD, nested collections (max depth 5), move snippet to collection
- `src/app/(dashboard)/collections/page.tsx` — collection tree in sidebar
- `src/components/collections/CollectionTree.tsx` — expandable tree with create/rename/delete
- Drag-and-drop: move snippets between collections

### Phase 3 — AI Pipeline (Days 10–14)

**Day 10: BullMQ setup + embedding generation**
```bash
npm i bullmq openai
```
- `src/server/workers/snippet-processor.ts` — BullMQ worker
- `src/server/lib/queue.ts` — queue config with Upstash Redis
- Step 1: generate embeddings for each snippet file
- Enqueue on snippet create/update

**Day 11: AI auto-tagging**
- `src/server/workers/snippet-processor.ts` — GPT-4o-mini classification step
- Detect: language, framework, purpose tags, complexity, auto-description
- Create/link tags with `source: "ai"`
- Handle edge cases: very short snippets, config files, markup

**Day 12: Tag management**
- `src/server/routers/tag.ts` — list, create, delete, popular tags, merge duplicate tags
- `src/components/tags/TagCloud.tsx` — visual tag cloud with usage counts
- Filter snippets by tag from sidebar
- Tag auto-complete in snippet editor

**Day 13: Hybrid search implementation**
- `src/server/services/search.ts` — semantic search (pgvector), text search (pg_trgm), RRF merge
- `src/server/routers/search.ts` — search endpoint with filters (language, tags, collection)
- `src/app/(dashboard)/search/page.tsx` — search results with relevance highlighting

**Day 14: Search UI and refinements**
- `src/components/search/SearchBar.tsx` — global search bar with keyboard shortcut (Cmd+K)
- `src/components/search/SearchResults.tsx` — result cards with code preview, match highlighting
- `src/components/search/SearchFilters.tsx` — filter chips: language, tags, collection
- Quick preview: hover to see full snippet without navigating

### Phase 4 — VS Code Extension (Days 15–20)

**Day 15: Extension scaffold**
```bash
npx yo code
```
- `vscode-extension/` directory
- `vscode-extension/package.json` — contributes: commands, views, keybindings
- `vscode-extension/src/extension.ts` — activate, register commands
- `vscode-extension/src/api.ts` — HTTP client for SnipVault API

**Day 16: Device code auth flow**
- `src/server/routers/device-auth.ts` — create device code, poll for authorization, authorize endpoint
- `src/app/(dashboard)/device/page.tsx` — device code entry page: enter code, click authorize
- `vscode-extension/src/auth.ts` — device code flow: start, poll, store token
- Token stored in VS Code `globalState`

**Day 17: Save from VS Code**
- `vscode-extension/src/commands/save.ts` — select code → right-click → "Save to SnipVault"
- Title input, optional description
- Auto-detect language from VS Code's language ID
- Post to `POST /api/snippets` with source: "vscode"
- Status bar confirmation

**Day 18: Search from VS Code**
- `vscode-extension/src/commands/search.ts` — command palette: "SnipVault: Search"
- Input box → search results in QuickPick menu
- Select → insert at cursor
- Track access count on insert

**Day 19: Sidebar panel**
- `vscode-extension/src/providers/snippetTree.ts` — TreeDataProvider showing favorites and recent snippets
- Refresh button
- Click to insert, right-click to copy or open in web app
- Badge showing snippet count

**Day 20: Extension polish and packaging**
- `vscode-extension/.vscodeignore` — exclude unnecessary files
- Extension icon and README
- Test: save, search, insert cycle
- Package: `vsce package`
- VS Code Marketplace listing preparation

### Phase 5 — Import + Billing (Days 21–26)

**Day 21: GitHub Gists import**
- `src/server/services/gist-import.ts` — fetch all gists, convert to snippets, enqueue AI processing
- `src/server/routers/import.ts` — start import, get progress, list past imports
- `src/app/(dashboard)/import/page.tsx` — "Import from GitHub Gists" with OAuth flow

**Day 22: Import UI and progress**
- GitHub OAuth popup for Gist read access
- Progress bar: X of Y gists imported
- Target collection selector
- Import summary: created, failed, skipped (duplicates)
- `src/components/import/ImportProgress.tsx` — real-time progress display

**Day 23: Stripe integration**
- `src/server/services/stripe.ts` — Stripe client
- `src/server/routers/billing.ts` — createCheckoutSession, createPortalSession
- `src/app/api/webhooks/stripe/route.ts` — subscription events
- Plan mapping: free (100 snippets, keyword search only, web only), pro ($8/mo: unlimited, semantic search, AI tags, VS Code, import), team ($5/user/mo: all pro + team workspace, permissions, activity feed)

**Day 24: Plan enforcement**
- `src/server/lib/plan-limits.ts` — check limits:
  - Free: 100 snippets, keyword search only (no semantic), no AI tags, no VS Code extension, no import
  - Pro: unlimited snippets, semantic search, AI auto-tagging, VS Code + CLI, import, version history
  - Team: all pro + team workspace, member management, activity feed, usage analytics
- Middleware enforcement on mutations

**Day 25: Billing UI**
- `src/app/(dashboard)/settings/billing/page.tsx` — plan comparison, usage meter, upgrade
- `src/components/billing/PlanCard.tsx` — plan comparison cards
- `src/components/billing/SnippetUsage.tsx` — snippet count / limit bar

**Day 26: Plan-gated features**
- Disable semantic search on free (show keyword results only)
- Disable AI tags on free (no auto-tag, manual tags still work)
- Disable VS Code extension auth on free
- Hide import option on free
- Upgrade prompts throughout UI

### Phase 6 — Team Features + Polish (Days 27–32)

**Day 27: Team workspace**
- `src/server/routers/workspace-member.ts` — invite, remove, update role
- `src/app/(dashboard)/settings/team/page.tsx` — team member management
- Invite flow: email invitation → join workspace

**Day 28: Activity feed**
- `src/server/lib/activity.ts` — log activity on: create/update/delete snippet, import, share
- `src/server/routers/activity.ts` — list feed for workspace
- `src/app/(dashboard)/activity/page.tsx` — activity feed timeline
- `src/components/activity/ActivityItem.tsx` — activity card

**Day 29: Smart collections**
- `src/server/routers/collection.ts` — smart collection CRUD with rules (languages, tags, match mode)
- Smart collection resolver: query snippets matching rules on access
- `src/components/collections/SmartRulesEditor.tsx` — rule builder

**Day 30: Snippet copy and access tracking**
- Copy button on every snippet (copies code to clipboard)
- Track access count + last accessed timestamp
- "Most accessed" sort option
- Copy feedback toast notification

**Day 31: UX polish**
- Keyboard shortcuts: Cmd+K search, Cmd+S save, Cmd+N new snippet
- Dark/light mode toggle
- Loading states with skeleton screens
- Error boundaries for code editor
- Mobile-responsive layout (simplified view)

**Day 32: Performance optimization**
- Search query optimization: ensure pg_trgm and HNSW indexes are used
- Snippet list pagination: cursor-based for large libraries
- Monaco Editor lazy loading
- Bundle analysis: keep page <500KB initial load
- API response caching for tag lists and collections

### Phase 7 — Launch Prep (Days 33–35)

**Day 33: Monitoring and observability**
- Sentry integration
- Vercel Analytics
- Custom logging for: search queries, AI processing, import jobs
- Alerting: queue depth, failed AI jobs, slow queries

**Day 34: End-to-end testing**
- Full flow: create snippet → AI tags applied → search finds it → copy to clipboard
- VS Code extension: authenticate → save → search → insert
- Gist import: authorize → import 50 gists → all tagged and searchable
- Team workspace: invite → member sees shared snippets

**Day 35: Deploy and launch**
- Production deploy
- VS Code Marketplace publish
- Verify Stripe webhooks in production
- Verify search quality with diverse test queries
- Performance benchmarks pass

---

## Critical Files

```
snipvault/
├── src/
│   ├── app/
│   │   ├── layout.tsx                            # Root layout
│   │   ├── (auth)/
│   │   │   ├── sign-in/[[...sign-in]]/page.tsx
│   │   │   └── sign-up/[[...sign-up]]/page.tsx
│   │   ├── (dashboard)/
│   │   │   ├── layout.tsx                        # Dashboard shell with sidebar
│   │   │   ├── page.tsx                          # Home: search bar + recent + favorites
│   │   │   ├── snippets/
│   │   │   │   ├── page.tsx                      # Snippet library (grid/list)
│   │   │   │   ├── new/page.tsx                  # Create snippet
│   │   │   │   └── [id]/page.tsx                 # Snippet detail + editor
│   │   │   ├── search/page.tsx                   # Search results
│   │   │   ├── collections/page.tsx              # Collection tree
│   │   │   ├── import/page.tsx                   # Gist import
│   │   │   ├── activity/page.tsx                 # Team activity feed
│   │   │   ├── device/page.tsx                   # Device code auth entry
│   │   │   └── settings/
│   │   │       ├── page.tsx                      # Workspace settings
│   │   │       ├── team/page.tsx                 # Team member management
│   │   │       └── billing/page.tsx              # Plan management
│   │   └── api/
│   │       ├── trpc/[trpc]/route.ts
│   │       └── webhooks/
│   │           └── stripe/route.ts
│   ├── server/
│   │   ├── db/
│   │   │   ├── index.ts                          # Neon + pgvector + pg_trgm
│   │   │   └── schema.ts                         # 11 tables + relations
│   │   ├── trpc.ts
│   │   ├── routers/
│   │   │   ├── _app.ts
│   │   │   ├── org.ts
│   │   │   ├── snippet.ts                        # Snippet CRUD + favorite
│   │   │   ├── snippet-version.ts                # Version history
│   │   │   ├── collection.ts                     # Collection CRUD + smart
│   │   │   ├── tag.ts                            # Tag CRUD + popular
│   │   │   ├── search.ts                         # Hybrid search endpoint
│   │   │   ├── import.ts                         # Import job management
│   │   │   ├── device-auth.ts                    # Device code flow
│   │   │   ├── workspace-member.ts               # Team member CRUD
│   │   │   ├── activity.ts                       # Activity feed
│   │   │   └── billing.ts                        # Stripe checkout/portal
│   │   ├── services/
│   │   │   ├── search.ts                         # Hybrid search engine (RRF)
│   │   │   ├── gist-import.ts                    # GitHub Gists import
│   │   │   ├── stripe.ts                         # Stripe client
│   │   │   └── language-detect.ts                # Code language detection
│   │   ├── workers/
│   │   │   └── snippet-processor.ts              # BullMQ: embed + AI tag
│   │   └── lib/
│   │       ├── org-seed.ts
│   │       ├── plan-limits.ts
│   │       ├── queue.ts
│   │       ├── encryption.ts
│   │       └── activity.ts                       # Activity logging helper
│   └── components/
│       ├── snippets/
│       │   ├── SnippetForm.tsx                    # Create/edit form
│       │   ├── SnippetCard.tsx                    # List view card
│       │   ├── CodeEditor.tsx                     # Monaco Editor wrapper
│       │   ├── TagEditor.tsx                      # Tag chips with autocomplete
│       │   └── VersionHistory.tsx                 # Version timeline
│       ├── search/
│       │   ├── SearchBar.tsx                      # Global search (Cmd+K)
│       │   ├── SearchResults.tsx                  # Result cards
│       │   └── SearchFilters.tsx                  # Filter chips
│       ├── collections/
│       │   ├── CollectionTree.tsx                 # Sidebar tree
│       │   └── SmartRulesEditor.tsx               # Smart collection rules
│       ├── import/
│       │   └── ImportProgress.tsx                 # Import progress bar
│       ├── activity/
│       │   └── ActivityItem.tsx                   # Activity card
│       └── billing/
│           ├── PlanCard.tsx
│           └── SnippetUsage.tsx
├── vscode-extension/
│   ├── package.json                              # Extension manifest
│   ├── tsconfig.json
│   └── src/
│       ├── extension.ts                          # Entry point
│       ├── api.ts                                # HTTP client
│       ├── auth.ts                               # Device code flow
│       ├── commands/
│       │   ├── save.ts                           # Save selected code
│       │   └── search.ts                         # Search and insert
│       └── providers/
│           └── snippetTree.ts                    # Sidebar tree provider
├── drizzle.config.ts
├── .env.local
└── package.json
```

---

## Verification

### Manual Testing Checklist

1. **Create snippet** — paste code, set title/language, save → appears in library
2. **AI auto-tagging** — create snippet, wait 5s, verify language + purpose tags auto-applied
3. **Auto-description** — create snippet without description, verify AI generates one
4. **Semantic search** — save a React hook, search "custom hook for form validation", verify found
5. **Text search** — search for exact function name (e.g. `useDebounce`), verify found via pg_trgm
6. **Hybrid ranking** — search term matching both semantic and keyword, verify hybrid results ranked highest
7. **Collection nesting** — create collection → subcollection → move snippet into subcollection
8. **Snippet versioning** — edit snippet code, verify version created, diff shows changes
9. **Favorite toggle** — star snippet, verify it appears in favorites section
10. **GitHub Gists import** — authorize GitHub, import 10 gists, verify all created with AI tags
11. **VS Code extension auth** — run "SnipVault: Authenticate", enter code in browser, verify connected
12. **VS Code save** — select code in editor, right-click "Save to SnipVault", verify appears in web app
13. **VS Code search + insert** — search from command palette, select result, verify inserted at cursor
14. **Plan limits** — free user: try creating 101st snippet, verify blocked with upgrade prompt
15. **Search on free plan** — verify only keyword search works, semantic search returns upgrade prompt

### SQL Verification Queries

```sql
-- 1. Snippets per org with AI tag coverage
SELECT o.name,
  COUNT(DISTINCT s.id) as snippet_count,
  COUNT(DISTINCT CASE WHEN st.source = 'ai' THEN st.id END) as ai_tagged
FROM organizations o
LEFT JOIN snippets s ON s.org_id = o.id
LEFT JOIN snippet_tags st ON st.snippet_id = s.id
GROUP BY o.name;

-- 2. Embedding coverage (should be 100% for pro+ users)
SELECT
  COUNT(sf.id) as total_files,
  SUM(CASE WHEN sf.embedding IS NOT NULL THEN 1 ELSE 0 END) as with_embedding,
  ROUND(100.0 * SUM(CASE WHEN sf.embedding IS NOT NULL THEN 1 ELSE 0 END) /
    NULLIF(COUNT(sf.id), 0), 1) as coverage_pct
FROM snippet_files sf;

-- 3. Most popular tags
SELECT t.name, t.category, t.usage_count
FROM tags t
ORDER BY t.usage_count DESC
LIMIT 20;

-- 4. Search quality check (semantic vs text)
-- Run after search benchmarks to verify both paths produce results

-- 5. Import job health
SELECT source, status, COUNT(*) as count,
  AVG(processed_items) as avg_items
FROM import_jobs
GROUP BY source, status;
```

### Performance Benchmarks

| Operation | Target | Method |
|-----------|--------|--------|
| Snippet save API response | <200ms | Immediate return, async AI processing |
| AI tagging (worker) | <3s | GPT-4o-mini classification + embedding |
| Semantic search | <300ms | pgvector HNSW query, 10K snippets |
| Text search (pg_trgm) | <100ms | Trigram index query |
| Hybrid search (RRF merge) | <500ms | Parallel semantic + text, RRF merge |
| Snippet detail page load | <800ms | Snippet + files + tags + versions |
| Library page (50 snippets) | <1s | Paginated with collection sidebar |
| VS Code save command | <1s | API call + confirmation |
| VS Code search command | <1.5s | Search + QuickPick display |
| Gist import (100 gists) | <60s | Fetch + create + enqueue AI |
