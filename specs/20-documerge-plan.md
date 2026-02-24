# 20. DocuMerge — API Documentation Generator

## Implementation Plan

**MVP Scope:** OpenAPI 3.0/3.1 import (file upload and URL fetch), documentation site generation with beautiful default theme (Next.js static generation), interactive API playground with request builder and response viewer, auto-generated code examples in cURL + JavaScript + Python, GitHub App auto-sync (rebuild docs on push), basic branding (logo, colors), full-text search across endpoints, hosted on `{slug}.documerge.io`, Stripe billing with three tiers (Free / Pro $29/mo / Enterprise $99/mo).

---

## Tech Decisions

| Layer | Technology | Notes |
|-------|-----------|-------|
| Framework | Next.js 14+ App Router | `src/` directory, TypeScript strict |
| API | tRPC v11 | superjson transformer, Clerk auth context |
| Database | Neon PostgreSQL | Projects, versions, endpoints, pages |
| ORM | Drizzle ORM | Push-based migrations, typed schema |
| Auth | Clerk | Organization-scoped, `orgId` from session |
| Billing | Stripe | Checkout Sessions + Customer Portal + Webhooks |
| OpenAPI Parser | @readme/openapi-parser | Dereference + validate OpenAPI 3.0/3.1 |
| Docs Site | Next.js ISR | Static generation per project, revalidate on sync |
| Playground Proxy | Next.js API Route | CORS proxy for API requests from browser |
| Code Examples | Custom Handlebars templates | Per-language template engine |
| GitHub | GitHub App (webhooks) | Auto-rebuild on push to spec file |
| Search | Pagefind | Static search index, client-side, zero-config |
| AI | OpenAI GPT-4o | AI-generated quickstart guide |
| Email | Resend | Sync notifications |
| Storage | Cloudflare R2 | Logos, OG images |
| UI | Tailwind CSS, Radix UI | |
| Docs Theme | Tailwind CSS + custom components | Responsive, dark mode, sidebar nav |
| Hosting | Vercel | |
| Monitoring | Sentry, Vercel Analytics | |

### Key Architecture Decisions

1. **OpenAPI spec parsed and stored as normalized JSON**: On import, the OpenAPI spec is dereferenced (all `$ref` resolved), validated, and stored as both raw (for re-export) and normalized JSON (for rendering). Each endpoint, schema, and security scheme is extracted into individual database rows for efficient querying and rendering.

2. **Docs site generated via ISR, not SSG at build time**: Each project's documentation site is rendered on-demand using Incremental Static Regeneration. When the OpenAPI spec is updated (via GitHub sync or manual upload), we call `revalidateTag()` to regenerate affected pages. This avoids full rebuilds and keeps docs fresh within seconds.

3. **Playground CORS proxy**: Browser-based API testing requires a proxy to bypass CORS restrictions. The playground sends requests to our API route, which forwards them to the actual API server and returns the response. This adds latency but enables testing any API regardless of CORS configuration.

4. **Handlebars templates for code examples**: Code examples are generated using per-language Handlebars templates rather than AI. This is deterministic, fast (no API calls), and produces syntactically correct code every time. Templates handle authentication, parameters, request bodies, and error handling patterns for each language.

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
  plan: text("plan").default("free").notNull(), // free | pro | enterprise
  stripeCustomerId: text("stripe_customer_id"),
  stripeSubscriptionId: text("stripe_subscription_id"),
  settings: jsonb("settings").default({}).$type<{
    notificationFromName?: string;
    notificationFromEmail?: string;
  }>(),
  createdAt: timestamp("created_at", { withTimezone: true }).defaultNow().notNull(),
  updatedAt: timestamp("updated_at", { withTimezone: true }).defaultNow().notNull(),
});

// ---------------------------------------------------------------------------
// API Projects
// ---------------------------------------------------------------------------
export const apiProjects = pgTable(
  "api_projects",
  {
    id: uuid("id").primaryKey().defaultRandom(),
    orgId: uuid("org_id")
      .references(() => organizations.id, { onDelete: "cascade" })
      .notNull(),
    name: text("name").notNull(),
    slug: text("slug").notNull(),
    description: text("description"),
    customDomain: text("custom_domain"),
    branding: jsonb("branding").default({}).$type<{
      logoUrl?: string;
      primaryColor?: string;
      accentColor?: string;
      fontFamily?: string;
      faviconUrl?: string;
      headerHtml?: string;
      footerHtml?: string;
      ogImageUrl?: string;
    }>(),
    // GitHub sync config
    githubRepoUrl: text("github_repo_url"),
    githubSpecPath: text("github_spec_path"), // e.g. "api/openapi.yaml"
    githubInstallationId: text("github_installation_id"),
    githubBranch: text("github_branch").default("main"),
    autoSyncEnabled: boolean("auto_sync_enabled").default(false).notNull(),
    lastSyncedAt: timestamp("last_synced_at", { withTimezone: true }),
    lastSyncError: text("last_sync_error"),
    isActive: boolean("is_active").default(true).notNull(),
    createdAt: timestamp("created_at", { withTimezone: true }).defaultNow().notNull(),
    updatedAt: timestamp("updated_at", { withTimezone: true }).defaultNow().notNull(),
  },
  (t) => [
    uniqueIndex("project_org_slug_idx").on(t.orgId, t.slug),
    index("project_org_idx").on(t.orgId),
  ],
);

// ---------------------------------------------------------------------------
// API Versions
// ---------------------------------------------------------------------------
export const apiVersions = pgTable(
  "api_versions",
  {
    id: uuid("id").primaryKey().defaultRandom(),
    projectId: uuid("project_id")
      .references(() => apiProjects.id, { onDelete: "cascade" })
      .notNull(),
    versionName: text("version_name").notNull(), // e.g. "v1", "v2"
    specFormat: text("spec_format").default("openapi").notNull(), // openapi | graphql (v2)
    specRaw: text("spec_raw").notNull(), // original YAML/JSON as uploaded
    specParsed: jsonb("spec_parsed").notNull(), // dereferenced, normalized JSON
    specInfo: jsonb("spec_info").default({}).$type<{
      title?: string;
      description?: string;
      version?: string;
      baseUrl?: string;
      servers?: Array<{ url: string; description?: string }>;
      securitySchemes?: Record<string, {
        type: string;
        scheme?: string;
        bearerFormat?: string;
        in?: string;
        name?: string;
      }>;
    }>(),
    isLatest: boolean("is_latest").default(true).notNull(),
    isDeprecated: boolean("is_deprecated").default(false).notNull(),
    publishedAt: timestamp("published_at", { withTimezone: true }),
    createdAt: timestamp("created_at", { withTimezone: true }).defaultNow().notNull(),
  },
  (t) => [
    index("version_project_idx").on(t.projectId),
    uniqueIndex("version_project_name_idx").on(t.projectId, t.versionName),
  ],
);

// ---------------------------------------------------------------------------
// Endpoints (extracted from spec)
// ---------------------------------------------------------------------------
export const endpoints = pgTable(
  "endpoints",
  {
    id: uuid("id").primaryKey().defaultRandom(),
    versionId: uuid("version_id")
      .references(() => apiVersions.id, { onDelete: "cascade" })
      .notNull(),
    method: text("method").notNull(), // GET | POST | PUT | PATCH | DELETE
    path: text("path").notNull(), // /users/{id}
    operationId: text("operation_id"),
    summary: text("summary"),
    description: text("description"),
    tags: jsonb("tags").default([]).$type<string[]>(),
    deprecated: boolean("deprecated").default(false).notNull(),
    parameters: jsonb("parameters").default([]).$type<
      Array<{
        name: string;
        in: "path" | "query" | "header" | "cookie";
        required: boolean;
        description?: string;
        schema: Record<string, unknown>;
        example?: unknown;
      }>
    >(),
    requestBody: jsonb("request_body").$type<{
      required?: boolean;
      description?: string;
      content: Record<string, {
        schema: Record<string, unknown>;
        example?: unknown;
      }>;
    }>(),
    responses: jsonb("responses").default({}).$type<
      Record<string, {
        description: string;
        content?: Record<string, {
          schema: Record<string, unknown>;
          example?: unknown;
        }>;
      }>
    >(),
    security: jsonb("security").$type<Array<Record<string, string[]>>>(),
    codeExamples: jsonb("code_examples").default({}).$type<
      Record<string, string> // { curl: "...", javascript: "...", python: "..." }
    >(),
    customExamples: jsonb("custom_examples").$type<
      Record<string, string> // user overrides
    >(),
    sortOrder: integer("sort_order").default(0).notNull(),
    createdAt: timestamp("created_at", { withTimezone: true }).defaultNow().notNull(),
  },
  (t) => [
    index("endpoint_version_idx").on(t.versionId),
    index("endpoint_method_path_idx").on(t.method, t.path),
  ],
);

// ---------------------------------------------------------------------------
// Schemas (extracted from spec components)
// ---------------------------------------------------------------------------
export const schemas = pgTable(
  "schemas",
  {
    id: uuid("id").primaryKey().defaultRandom(),
    versionId: uuid("version_id")
      .references(() => apiVersions.id, { onDelete: "cascade" })
      .notNull(),
    name: text("name").notNull(), // e.g. "User", "Error"
    definition: jsonb("definition").notNull(), // JSON Schema object
    description: text("description"),
    createdAt: timestamp("created_at", { withTimezone: true }).defaultNow().notNull(),
  },
  (t) => [
    index("schema_version_idx").on(t.versionId),
    uniqueIndex("schema_version_name_idx").on(t.versionId, t.name),
  ],
);

// ---------------------------------------------------------------------------
// Custom Pages (MDX guides, tutorials)
// ---------------------------------------------------------------------------
export const customPages = pgTable(
  "custom_pages",
  {
    id: uuid("id").primaryKey().defaultRandom(),
    projectId: uuid("project_id")
      .references(() => apiProjects.id, { onDelete: "cascade" })
      .notNull(),
    title: text("title").notNull(),
    slug: text("slug").notNull(),
    contentMdx: text("content_mdx").notNull(),
    parentPageId: uuid("parent_page_id"),
    sortOrder: integer("sort_order").default(0).notNull(),
    published: boolean("published").default(true).notNull(),
    createdAt: timestamp("created_at", { withTimezone: true }).defaultNow().notNull(),
    updatedAt: timestamp("updated_at", { withTimezone: true }).defaultNow().notNull(),
  },
  (t) => [
    uniqueIndex("custom_page_project_slug_idx").on(t.projectId, t.slug),
    index("custom_page_project_idx").on(t.projectId),
  ],
);

// ---------------------------------------------------------------------------
// Playground Sessions (saved API requests)
// ---------------------------------------------------------------------------
export const playgroundSessions = pgTable(
  "playground_sessions",
  {
    id: uuid("id").primaryKey().defaultRandom(),
    projectId: uuid("project_id")
      .references(() => apiProjects.id, { onDelete: "cascade" })
      .notNull(),
    endpointId: uuid("endpoint_id")
      .references(() => endpoints.id, { onDelete: "cascade" }),
    request: jsonb("request").notNull().$type<{
      method: string;
      url: string;
      headers: Record<string, string>;
      body?: string;
      params?: Record<string, string>;
    }>(),
    response: jsonb("response").$type<{
      status: number;
      statusText: string;
      headers: Record<string, string>;
      body: string;
      timing: number; // ms
    }>(),
    sessionId: text("session_id").notNull(), // anonymous user session
    createdAt: timestamp("created_at", { withTimezone: true }).defaultNow().notNull(),
  },
  (t) => [
    index("playground_project_idx").on(t.projectId),
    index("playground_session_idx").on(t.sessionId),
  ],
);

// ---------------------------------------------------------------------------
// Page Views (docs analytics)
// ---------------------------------------------------------------------------
export const pageViews = pgTable(
  "page_views",
  {
    id: uuid("id").primaryKey().defaultRandom(),
    projectId: uuid("project_id")
      .references(() => apiProjects.id, { onDelete: "cascade" })
      .notNull(),
    path: text("path").notNull(),
    endpointId: uuid("endpoint_id"),
    referrer: text("referrer"),
    country: text("country"),
    device: text("device"),
    viewedAt: timestamp("viewed_at", { withTimezone: true }).defaultNow().notNull(),
  },
  (t) => [
    index("pv_project_idx").on(t.projectId),
    index("pv_viewed_idx").on(t.viewedAt),
  ],
);

// ---------------------------------------------------------------------------
// GitHub Webhook Events (sync log)
// ---------------------------------------------------------------------------
export const githubWebhookEvents = pgTable(
  "github_webhook_events",
  {
    id: uuid("id").primaryKey().defaultRandom(),
    projectId: uuid("project_id")
      .references(() => apiProjects.id, { onDelete: "cascade" })
      .notNull(),
    eventType: text("event_type").notNull(),
    payload: jsonb("payload"),
    status: text("status").default("pending").notNull(),
    processedAt: timestamp("processed_at", { withTimezone: true }),
    errorMessage: text("error_message"),
    receivedAt: timestamp("received_at", { withTimezone: true }).defaultNow().notNull(),
  },
  (t) => [
    index("gh_webhook_project_idx").on(t.projectId),
  ],
);

// ---------------------------------------------------------------------------
// Relations
// ---------------------------------------------------------------------------
export const organizationsRelations = relations(organizations, ({ many }) => ({
  apiProjects: many(apiProjects),
}));

export const apiProjectsRelations = relations(apiProjects, ({ one, many }) => ({
  organization: one(organizations, {
    fields: [apiProjects.orgId],
    references: [organizations.id],
  }),
  versions: many(apiVersions),
  customPages: many(customPages),
  playgroundSessions: many(playgroundSessions),
  pageViews: many(pageViews),
  githubWebhookEvents: many(githubWebhookEvents),
}));

export const apiVersionsRelations = relations(apiVersions, ({ one, many }) => ({
  project: one(apiProjects, {
    fields: [apiVersions.projectId],
    references: [apiProjects.id],
  }),
  endpoints: many(endpoints),
  schemas: many(schemas),
}));

export const endpointsRelations = relations(endpoints, ({ one }) => ({
  version: one(apiVersions, {
    fields: [endpoints.versionId],
    references: [apiVersions.id],
  }),
}));

export const schemasRelations = relations(schemas, ({ one }) => ({
  version: one(apiVersions, {
    fields: [schemas.versionId],
    references: [apiVersions.id],
  }),
}));

export const customPagesRelations = relations(customPages, ({ one }) => ({
  project: one(apiProjects, {
    fields: [customPages.projectId],
    references: [apiProjects.id],
  }),
}));

export const playgroundSessionsRelations = relations(playgroundSessions, ({ one }) => ({
  project: one(apiProjects, {
    fields: [playgroundSessions.projectId],
    references: [apiProjects.id],
  }),
  endpoint: one(endpoints, {
    fields: [playgroundSessions.endpointId],
    references: [endpoints.id],
  }),
}));

export const pageViewsRelations = relations(pageViews, ({ one }) => ({
  project: one(apiProjects, {
    fields: [pageViews.projectId],
    references: [apiProjects.id],
  }),
}));

export const githubWebhookEventsRelations = relations(githubWebhookEvents, ({ one }) => ({
  project: one(apiProjects, {
    fields: [githubWebhookEvents.projectId],
    references: [apiProjects.id],
  }),
}));
```

---

## Initial Migration SQL

```sql
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";

CREATE TABLE organizations (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  clerk_org_id TEXT UNIQUE NOT NULL,
  name TEXT NOT NULL,
  slug TEXT UNIQUE NOT NULL,
  plan TEXT NOT NULL DEFAULT 'free',
  stripe_customer_id TEXT,
  stripe_subscription_id TEXT,
  settings JSONB DEFAULT '{}',
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE api_projects (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  org_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
  name TEXT NOT NULL,
  slug TEXT NOT NULL,
  description TEXT,
  custom_domain TEXT,
  branding JSONB DEFAULT '{}',
  github_repo_url TEXT,
  github_spec_path TEXT,
  github_installation_id TEXT,
  github_branch TEXT DEFAULT 'main',
  auto_sync_enabled BOOLEAN NOT NULL DEFAULT false,
  last_synced_at TIMESTAMPTZ,
  last_sync_error TEXT,
  is_active BOOLEAN NOT NULL DEFAULT true,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE UNIQUE INDEX project_org_slug_idx ON api_projects(org_id, slug);
CREATE INDEX project_org_idx ON api_projects(org_id);

CREATE TABLE api_versions (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  project_id UUID NOT NULL REFERENCES api_projects(id) ON DELETE CASCADE,
  version_name TEXT NOT NULL,
  spec_format TEXT NOT NULL DEFAULT 'openapi',
  spec_raw TEXT NOT NULL,
  spec_parsed JSONB NOT NULL,
  spec_info JSONB DEFAULT '{}',
  is_latest BOOLEAN NOT NULL DEFAULT true,
  is_deprecated BOOLEAN NOT NULL DEFAULT false,
  published_at TIMESTAMPTZ,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX version_project_idx ON api_versions(project_id);
CREATE UNIQUE INDEX version_project_name_idx ON api_versions(project_id, version_name);

CREATE TABLE endpoints (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  version_id UUID NOT NULL REFERENCES api_versions(id) ON DELETE CASCADE,
  method TEXT NOT NULL,
  path TEXT NOT NULL,
  operation_id TEXT,
  summary TEXT,
  description TEXT,
  tags JSONB DEFAULT '[]',
  deprecated BOOLEAN NOT NULL DEFAULT false,
  parameters JSONB DEFAULT '[]',
  request_body JSONB,
  responses JSONB DEFAULT '{}',
  security JSONB,
  code_examples JSONB DEFAULT '{}',
  custom_examples JSONB,
  sort_order INTEGER NOT NULL DEFAULT 0,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX endpoint_version_idx ON endpoints(version_id);
CREATE INDEX endpoint_method_path_idx ON endpoints(method, path);

CREATE TABLE schemas (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  version_id UUID NOT NULL REFERENCES api_versions(id) ON DELETE CASCADE,
  name TEXT NOT NULL,
  definition JSONB NOT NULL,
  description TEXT,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX schema_version_idx ON schemas(version_id);
CREATE UNIQUE INDEX schema_version_name_idx ON schemas(version_id, name);

CREATE TABLE custom_pages (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  project_id UUID NOT NULL REFERENCES api_projects(id) ON DELETE CASCADE,
  title TEXT NOT NULL,
  slug TEXT NOT NULL,
  content_mdx TEXT NOT NULL,
  parent_page_id UUID,
  sort_order INTEGER NOT NULL DEFAULT 0,
  published BOOLEAN NOT NULL DEFAULT true,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE UNIQUE INDEX custom_page_project_slug_idx ON custom_pages(project_id, slug);
CREATE INDEX custom_page_project_idx ON custom_pages(project_id);

CREATE TABLE playground_sessions (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  project_id UUID NOT NULL REFERENCES api_projects(id) ON DELETE CASCADE,
  endpoint_id UUID REFERENCES endpoints(id) ON DELETE CASCADE,
  request JSONB NOT NULL,
  response JSONB,
  session_id TEXT NOT NULL,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX playground_project_idx ON playground_sessions(project_id);
CREATE INDEX playground_session_idx ON playground_sessions(session_id);

CREATE TABLE page_views (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  project_id UUID NOT NULL REFERENCES api_projects(id) ON DELETE CASCADE,
  path TEXT NOT NULL,
  endpoint_id UUID,
  referrer TEXT,
  country TEXT,
  device TEXT,
  viewed_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX pv_project_idx ON page_views(project_id);
CREATE INDEX pv_viewed_idx ON page_views(viewed_at);

CREATE TABLE github_webhook_events (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  project_id UUID NOT NULL REFERENCES api_projects(id) ON DELETE CASCADE,
  event_type TEXT NOT NULL,
  payload JSONB,
  status TEXT NOT NULL DEFAULT 'pending',
  processed_at TIMESTAMPTZ,
  error_message TEXT,
  received_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX gh_webhook_project_idx ON github_webhook_events(project_id);
```

---

## Architecture Deep-Dives

### 1. OpenAPI Parser + Endpoint Extraction

The parser takes a raw OpenAPI 3.0/3.1 spec (YAML or JSON), validates it, dereferences all `$ref` pointers, extracts endpoints and schemas into individual database rows, and generates code examples for each endpoint.

```typescript
// src/server/services/openapi-parser.ts

import SwaggerParser from "@readme/openapi-parser";
import yaml from "js-yaml";
import { eq, sql } from "drizzle-orm";
import { db } from "../db";
import {
  apiVersions,
  endpoints,
  schemas as schemasTable,
} from "../db/schema";
import { generateCodeExamples } from "./code-examples";

interface ParseResult {
  versionId: string;
  endpointCount: number;
  schemaCount: number;
  errors: string[];
}

export async function parseAndStoreSpec(
  projectId: string,
  versionName: string,
  specContent: string,
): Promise<ParseResult> {
  const errors: string[] = [];

  // Detect format (YAML or JSON)
  let rawSpec: Record<string, unknown>;
  try {
    rawSpec = specContent.trim().startsWith("{")
      ? JSON.parse(specContent)
      : (yaml.load(specContent) as Record<string, unknown>);
  } catch (err) {
    throw new Error(`Invalid spec format: ${err instanceof Error ? err.message : String(err)}`);
  }

  // Validate and dereference
  const api = await SwaggerParser.validate(rawSpec as any);
  const dereferenced = await SwaggerParser.dereference(rawSpec as any);

  // Extract spec info
  const specInfo = {
    title: (dereferenced as any).info?.title,
    description: (dereferenced as any).info?.description,
    version: (dereferenced as any).info?.version,
    baseUrl: (dereferenced as any).servers?.[0]?.url,
    servers: (dereferenced as any).servers || [],
    securitySchemes: (dereferenced as any).components?.securitySchemes || {},
  };

  // Mark previous version as not latest
  await db
    .update(apiVersions)
    .set({ isLatest: false })
    .where(eq(apiVersions.projectId, projectId));

  // Create version record
  const [version] = await db
    .insert(apiVersions)
    .values({
      projectId,
      versionName,
      specFormat: "openapi",
      specRaw: specContent,
      specParsed: dereferenced as any,
      specInfo,
      isLatest: true,
      publishedAt: new Date(),
    })
    .onConflictDoUpdate({
      target: [apiVersions.projectId, apiVersions.versionName],
      set: {
        specRaw: specContent,
        specParsed: dereferenced as any,
        specInfo,
        isLatest: true,
        publishedAt: new Date(),
      },
    })
    .returning();

  // Delete old endpoints/schemas for this version (if updating)
  await db.delete(endpoints).where(eq(endpoints.versionId, version.id));
  await db.delete(schemasTable).where(eq(schemasTable.versionId, version.id));

  // Extract endpoints
  const paths = (dereferenced as any).paths || {};
  let sortOrder = 0;
  let endpointCount = 0;

  for (const [path, methods] of Object.entries(paths)) {
    for (const [method, operation] of Object.entries(methods as Record<string, any>)) {
      if (!["get", "post", "put", "patch", "delete", "options", "head"].includes(method)) {
        continue;
      }

      // Generate code examples
      const codeExamples = generateCodeExamples({
        method: method.toUpperCase(),
        path,
        baseUrl: specInfo.baseUrl || "https://api.example.com",
        parameters: operation.parameters || [],
        requestBody: operation.requestBody,
        security: operation.security || (dereferenced as any).security,
        securitySchemes: specInfo.securitySchemes || {},
      });

      await db.insert(endpoints).values({
        versionId: version.id,
        method: method.toUpperCase(),
        path,
        operationId: operation.operationId,
        summary: operation.summary,
        description: operation.description,
        tags: operation.tags || [],
        deprecated: operation.deprecated || false,
        parameters: operation.parameters || [],
        requestBody: operation.requestBody || null,
        responses: operation.responses || {},
        security: operation.security,
        codeExamples,
        sortOrder: sortOrder++,
      });
      endpointCount++;
    }
  }

  // Extract component schemas
  const componentSchemas = (dereferenced as any).components?.schemas || {};
  let schemaCount = 0;

  for (const [name, definition] of Object.entries(componentSchemas)) {
    await db.insert(schemasTable).values({
      versionId: version.id,
      name,
      definition: definition as Record<string, unknown>,
      description: (definition as any).description,
    });
    schemaCount++;
  }

  return { versionId: version.id, endpointCount, schemaCount, errors };
}
```

### 2. Code Example Generation Engine

Generates idiomatic code examples for each endpoint in multiple languages using Handlebars templates. Each template handles authentication, parameters, request bodies, and response handling.

```typescript
// src/server/services/code-examples.ts

import Handlebars from "handlebars";

interface CodeExampleInput {
  method: string;
  path: string;
  baseUrl: string;
  parameters: Array<{
    name: string;
    in: string;
    required: boolean;
    schema: Record<string, unknown>;
    example?: unknown;
  }>;
  requestBody?: {
    content: Record<string, { schema: Record<string, unknown>; example?: unknown }>;
  };
  security?: Array<Record<string, string[]>>;
  securitySchemes: Record<string, {
    type: string;
    scheme?: string;
    in?: string;
    name?: string;
  }>;
}

const TEMPLATES: Record<string, string> = {
  curl: `curl -X {{method}} '{{fullUrl}}{{#if queryString}}?{{queryString}}{{/if}}'{{#if hasAuth}} \\
  -H 'Authorization: Bearer YOUR_API_KEY'{{/if}}{{#if hasJsonBody}} \\
  -H 'Content-Type: application/json' \\
  -d '{{jsonBody}}'{{/if}}`,

  javascript: `const response = await fetch('{{fullUrl}}{{#if queryString}}?{{queryString}}{{/if}}', {
  method: '{{method}}',
  headers: {
    {{#if hasAuth}}'Authorization': 'Bearer YOUR_API_KEY',
    {{/if}}{{#if hasJsonBody}}'Content-Type': 'application/json',{{/if}}
  },{{#if hasJsonBody}}
  body: JSON.stringify({{jsonBodyPretty}}),{{/if}}
});

const data = await response.json();
console.log(data);`,

  python: `import requests

{{#if hasJsonBody}}payload = {{pythonDict}}

{{/if}}response = requests.{{lowerMethod}}(
    '{{fullUrl}}',{{#if queryParams}}
    params={{pythonParams}},{{/if}}{{#if hasAuth}}
    headers={'Authorization': 'Bearer YOUR_API_KEY'},{{/if}}{{#if hasJsonBody}}
    json=payload,{{/if}}
)

data = response.json()
print(data)`,
};

export function generateCodeExamples(input: CodeExampleInput): Record<string, string> {
  const pathParams = input.parameters.filter((p) => p.in === "path");
  const queryParams = input.parameters.filter((p) => p.in === "query");

  // Build URL with path parameters filled in
  let fullUrl = `${input.baseUrl}${input.path}`;
  for (const param of pathParams) {
    const example = param.example || getExampleValue(param.schema);
    fullUrl = fullUrl.replace(`{${param.name}}`, String(example));
  }

  // Build query string
  const queryParts = queryParams
    .filter((p) => p.required)
    .map((p) => {
      const val = p.example || getExampleValue(p.schema);
      return `${p.name}=${encodeURIComponent(String(val))}`;
    });
  const queryString = queryParts.join("&");

  // Determine auth requirement
  const hasAuth = input.security && input.security.length > 0;

  // Build request body
  const jsonContent = input.requestBody?.content?.["application/json"];
  const hasJsonBody = !!jsonContent;
  let jsonBody = "";
  let jsonBodyPretty = "";
  let pythonDict = "";

  if (jsonContent) {
    const example = jsonContent.example || generateExampleFromSchema(jsonContent.schema);
    jsonBody = JSON.stringify(example);
    jsonBodyPretty = JSON.stringify(example, null, 2);
    pythonDict = jsonToPython(example);
  }

  // Python params
  const pythonParams = queryParams.length > 0
    ? `{${queryParams.filter((p) => p.required).map((p) => {
        const val = p.example || getExampleValue(p.schema);
        return `'${p.name}': ${typeof val === "string" ? `'${val}'` : val}`;
      }).join(", ")}}`
    : "";

  const context = {
    method: input.method,
    lowerMethod: input.method.toLowerCase(),
    fullUrl,
    queryString,
    queryParams: queryParams.filter((p) => p.required),
    hasAuth,
    hasJsonBody,
    jsonBody,
    jsonBodyPretty,
    pythonDict,
    pythonParams,
  };

  const results: Record<string, string> = {};
  for (const [lang, template] of Object.entries(TEMPLATES)) {
    const compiled = Handlebars.compile(template, { noEscape: true });
    results[lang] = compiled(context).trim();
  }

  return results;
}

function getExampleValue(schema: Record<string, unknown>): unknown {
  const type = schema.type as string;
  if (schema.example !== undefined) return schema.example;
  if (schema.default !== undefined) return schema.default;
  switch (type) {
    case "string": return schema.format === "email" ? "user@example.com" : "string";
    case "integer": case "number": return 0;
    case "boolean": return true;
    case "array": return [];
    case "object": return {};
    default: return "value";
  }
}

function generateExampleFromSchema(schema: Record<string, unknown>): unknown {
  if (schema.example) return schema.example;
  if (schema.type === "object" && schema.properties) {
    const obj: Record<string, unknown> = {};
    for (const [key, prop] of Object.entries(schema.properties as Record<string, any>)) {
      obj[key] = getExampleValue(prop);
    }
    return obj;
  }
  return getExampleValue(schema);
}

function jsonToPython(obj: unknown): string {
  if (obj === null) return "None";
  if (obj === true) return "True";
  if (obj === false) return "False";
  if (typeof obj === "string") return `'${obj}'`;
  if (typeof obj === "number") return String(obj);
  if (Array.isArray(obj)) return `[${obj.map(jsonToPython).join(", ")}]`;
  if (typeof obj === "object") {
    const entries = Object.entries(obj as Record<string, unknown>)
      .map(([k, v]) => `'${k}': ${jsonToPython(v)}`)
      .join(", ");
    return `{${entries}}`;
  }
  return String(obj);
}
```

### 3. Interactive API Playground with CORS Proxy

The playground lets users build and send API requests from the browser. Requests go through our proxy to bypass CORS restrictions. The proxy validates the request, forwards it, and returns the response with timing information.

```typescript
// src/app/api/public/playground/proxy/route.ts

import { NextRequest, NextResponse } from "next/server";

const MAX_BODY_SIZE = 1024 * 1024; // 1MB
const REQUEST_TIMEOUT = 30_000; // 30s
const BLOCKED_HOSTS = ["localhost", "127.0.0.1", "0.0.0.0", "169.254.169.254"];

export async function POST(req: NextRequest) {
  const body = await req.json();
  const { method, url, headers, requestBody, projectSlug } = body;

  // Validate URL
  let parsedUrl: URL;
  try {
    parsedUrl = new URL(url);
  } catch {
    return NextResponse.json({ error: "Invalid URL" }, { status: 400 });
  }

  // Block internal/private URLs
  if (BLOCKED_HOSTS.includes(parsedUrl.hostname)) {
    return NextResponse.json(
      { error: "Cannot proxy to internal addresses" },
      { status: 403 },
    );
  }

  // Strip sensitive proxy headers
  const proxyHeaders: Record<string, string> = {};
  for (const [key, value] of Object.entries(headers || {})) {
    const lowerKey = key.toLowerCase();
    if (!["host", "origin", "referer", "cookie"].includes(lowerKey)) {
      proxyHeaders[key] = value as string;
    }
  }

  const startTime = Date.now();

  try {
    const controller = new AbortController();
    const timeout = setTimeout(() => controller.abort(), REQUEST_TIMEOUT);

    const proxyResponse = await fetch(url, {
      method: method || "GET",
      headers: proxyHeaders,
      body: requestBody ? JSON.stringify(requestBody) : undefined,
      signal: controller.signal,
    });

    clearTimeout(timeout);

    const timing = Date.now() - startTime;
    const responseBody = await proxyResponse.text();

    // Extract response headers
    const responseHeaders: Record<string, string> = {};
    proxyResponse.headers.forEach((value, key) => {
      responseHeaders[key] = value;
    });

    return NextResponse.json({
      status: proxyResponse.status,
      statusText: proxyResponse.statusText,
      headers: responseHeaders,
      body: responseBody,
      timing,
    });
  } catch (err) {
    const timing = Date.now() - startTime;
    if (err instanceof Error && err.name === "AbortError") {
      return NextResponse.json(
        { error: "Request timed out", timing },
        { status: 504 },
      );
    }
    return NextResponse.json(
      { error: `Request failed: ${err instanceof Error ? err.message : String(err)}`, timing },
      { status: 502 },
    );
  }
}
```

### 4. GitHub Auto-Sync Pipeline

When a push is made to the configured branch, the GitHub webhook triggers a re-fetch of the spec file, re-parse, and ISR revalidation of the docs site.

```typescript
// src/server/services/github-sync.ts

import { Octokit } from "@octokit/rest";
import { createAppAuth } from "@octokit/auth-app";
import { eq } from "drizzle-orm";
import { db } from "../db";
import { apiProjects, githubWebhookEvents } from "../db/schema";
import { parseAndStoreSpec } from "./openapi-parser";
import { revalidateTag } from "next/cache";

export async function handleGitHubPush(
  projectId: string,
  webhookEventId: string,
): Promise<void> {
  const project = await db.query.apiProjects.findFirst({
    where: eq(apiProjects.id, projectId),
  });
  if (!project || !project.autoSyncEnabled) return;
  if (!project.githubInstallationId || !project.githubSpecPath) return;

  try {
    // Get installation Octokit
    const auth = createAppAuth({
      appId: process.env.GITHUB_APP_ID!,
      privateKey: process.env.GITHUB_PRIVATE_KEY!,
      installationId: Number(project.githubInstallationId),
    });
    const { token } = await auth({ type: "installation" });
    const octokit = new Octokit({ auth: token });

    // Extract owner/repo from URL
    const repoMatch = project.githubRepoUrl?.match(
      /github\.com\/([^\/]+)\/([^\/]+)/,
    );
    if (!repoMatch) throw new Error("Invalid GitHub repo URL");
    const [, owner, repo] = repoMatch;

    // Fetch the spec file content
    const { data: fileData } = await octokit.repos.getContent({
      owner,
      repo: repo.replace(/\.git$/, ""),
      path: project.githubSpecPath,
      ref: project.githubBranch || "main",
    });

    if (!("content" in fileData)) {
      throw new Error("Spec file not found or is a directory");
    }

    const specContent = Buffer.from(fileData.content, "base64").toString("utf-8");

    // Determine version name from spec content or use "latest"
    let versionName = "latest";
    try {
      const parsed = specContent.trim().startsWith("{")
        ? JSON.parse(specContent)
        : require("js-yaml").load(specContent);
      versionName = parsed.info?.version || "latest";
    } catch {}

    // Parse and store
    const result = await parseAndStoreSpec(project.id, versionName, specContent);

    // Update project sync status
    await db
      .update(apiProjects)
      .set({
        lastSyncedAt: new Date(),
        lastSyncError: null,
        updatedAt: new Date(),
      })
      .where(eq(apiProjects.id, project.id));

    // Revalidate docs pages
    revalidateTag(`docs-${project.slug}`);

    // Update webhook event
    await db
      .update(githubWebhookEvents)
      .set({ status: "completed", processedAt: new Date() })
      .where(eq(githubWebhookEvents.id, webhookEventId));
  } catch (err) {
    const errorMsg = err instanceof Error ? err.message : String(err);

    await db
      .update(apiProjects)
      .set({ lastSyncError: errorMsg, updatedAt: new Date() })
      .where(eq(apiProjects.id, projectId));

    await db
      .update(githubWebhookEvents)
      .set({
        status: "failed",
        errorMessage: errorMsg,
        processedAt: new Date(),
      })
      .where(eq(githubWebhookEvents.id, webhookEventId));

    throw err;
  }
}
```

---

## Phase Breakdown

### Phase 1 — Project Scaffold + Auth + DB (Days 1–4)

**Day 1: Repository and framework setup**
```bash
npx create-next-app@latest documerge --ts --tailwind --app --src-dir
cd documerge
npm i drizzle-orm @neondatabase/serverless
npm i -D drizzle-kit
npm i @trpc/server @trpc/client @trpc/next @trpc/react-query superjson zod
npm i @clerk/nextjs
npm i @readme/openapi-parser js-yaml handlebars
```
- `src/app/layout.tsx`, `src/app/(dashboard)/layout.tsx`
- `src/server/db/index.ts`, `drizzle.config.ts`, `.env.local`

**Day 2: Full database schema**
- `src/server/db/schema.ts` — all 10 tables + relations
- Run `npx drizzle-kit push`

**Day 3: tRPC setup and org initialization**
- `src/server/trpc.ts`, `src/server/routers/_app.ts`, `src/server/routers/org.ts`
- `src/app/api/trpc/[trpc]/route.ts`

**Day 4: Auth flows and org seed**
- Sign-in/sign-up pages
- `src/server/lib/org-seed.ts` — create org on first login

### Phase 2 — OpenAPI Parser + Import (Days 5–10)

**Day 5: OpenAPI parser service**
- `src/server/services/openapi-parser.ts` — validate, dereference, extract endpoints + schemas
- Support OpenAPI 3.0 and 3.1
- Comprehensive error handling with user-friendly messages

**Day 6: Code example generator**
- `src/server/services/code-examples.ts` — Handlebars templates for cURL, JavaScript, Python
- Handle auth headers, path params, query params, request body
- Generate example values from schemas

> **Code Example Language Roadmap**
>
> *MVP (3 languages)*: cURL, JavaScript (fetch), Python (requests) — shipped Day 6 with Handlebars templates.
>
> *v2 expansion (6+ languages)*: Go (net/http), Ruby (net/http), PHP (cURL), Java (HttpClient), C# (HttpClient), Rust (reqwest). Each language template follows a standard interface: `{ auth, pathParams, queryParams, requestBody, responseHandling, errorHandling }`. Templates stored in `src/server/templates/code/{lang}.hbs`, loaded dynamically at parse time. New languages require only: (1) a `.hbs` template file and (2) a language config entry in `src/server/services/code-examples.ts` specifying syntax highlighting grammar and display name.
>
> *Community contribution framework*: Public GitHub repo `documerge/code-templates` with CI validation — each PR runs the template against 10 reference OpenAPI snippets (covering auth types, file uploads, pagination, nested bodies) and diffs output against golden files. Merged templates auto-deploy to production within 30 minutes via GitHub Actions webhook. Contributors recognized on docs site language selector with "Community" badge.
>
> **Code example language roadmap:** MVP ships with 3 languages (cURL, JavaScript/fetch, Python/requests). Each uses a Handlebars template in `src/lib/codegen/templates/{lang}.hbs`. v1.1 adds Go (net/http) and Ruby (net/http). v2 adds PHP (cURL), Java (HttpClient), and C# (HttpClient). Community-contributed templates can be added by creating a new `.hbs` file and registering it in the language registry.

**Day 7: Spec import — file upload**
- `src/server/routers/project.ts` — create project, import spec (file upload)
- `src/app/(dashboard)/projects/new/page.tsx` — create project wizard
- Step 1: name + slug, Step 2: upload spec file (YAML/JSON), Step 3: preview result

**Day 8: Spec import — URL fetch**
- `src/server/routers/project.ts` — import spec from URL
- Fetch URL, parse, same flow as file upload
- Support for public spec URLs

**Day 9: Project list and management**
- `src/app/(dashboard)/projects/page.tsx` — project list with cards
- `src/app/(dashboard)/projects/[slug]/page.tsx` — project overview: endpoints count, last sync, status
- `src/server/routers/project.ts` — list, get, update, delete, re-import spec

**Day 10: Spec validation feedback**
- Validation error display with line numbers
- Warning for unsupported features
- "Fix and retry" guidance
- Spec file re-upload without losing project config

### Phase 3 — Documentation Site Generation (Days 11–17)

**Day 11: Docs site routing**
- `src/app/(docs)/[projectSlug]/layout.tsx` — docs layout with sidebar + header
- `src/app/(docs)/[projectSlug]/page.tsx` — docs home (API overview, getting started)
- ISR with `revalidateTag('docs-{projectSlug}')`

**Day 12: Sidebar navigation**
- `src/components/docs/Sidebar.tsx` — navigation tree grouped by tags
- Endpoint entries: method badge (GET green, POST blue, etc.) + path
- Collapsible groups
- Active page highlighting
- Mobile-responsive (hamburger menu)

**Day 13: Endpoint documentation page**
- `src/app/(docs)/[projectSlug]/[...path]/page.tsx` — individual endpoint page
- `src/components/docs/EndpointPage.tsx` — method badge + URL, description, parameters table, request body schema viewer, response schema with status code tabs
- `src/components/docs/SchemaViewer.tsx` — interactive JSON schema tree (expandable nested objects)

**Day 14: Code examples panel**
- `src/components/docs/CodeExamples.tsx` — language tabs (cURL, JavaScript, Python)
- Syntax highlighting via Shiki
- Copy-to-clipboard per example
- Language preference stored in localStorage

**Day 15: Authentication documentation**
- `src/components/docs/AuthDocs.tsx` — auto-generated from security schemes
- Shows auth type (Bearer, API Key, OAuth), where to send credentials, example headers
- Authentication section in sidebar

**Day 16: Docs site theming**
- `src/components/docs/DocsTheme.tsx` — apply project branding (logo, colors, font)
- Dark mode toggle (stored in localStorage)
- Responsive layout: sidebar on desktop, drawer on mobile
- `src/components/docs/Header.tsx` — logo, project name, version switcher, dark mode, search

**Day 17: Full-text search (Pagefind)**
- Integrate Pagefind for static search indexing
- `src/components/docs/SearchDialog.tsx` — Cmd+K search dialog
- Search across endpoint summaries, descriptions, parameters, schemas
- Result highlighting with navigation

### Phase 4 — API Playground (Days 18–22)

**Day 18: Playground UI — request builder**
- `src/components/playground/PlaygroundPanel.tsx` — split view: request left, response right
- `src/components/playground/RequestBuilder.tsx` — method + URL, path parameter inputs, query parameter inputs, header editor
- Pre-fill from endpoint spec (parameter names, types, examples)

**Day 19: Playground — body editor + auth**
- `src/components/playground/BodyEditor.tsx` — JSON editor with schema validation hints
- `src/components/playground/AuthInput.tsx` — auth credential input based on spec (API key input, Bearer token input)
- Auto-populate from saved credentials (localStorage)

**Day 20: Playground — CORS proxy and response viewer**
- `src/app/api/public/playground/proxy/route.ts` — CORS proxy endpoint
- `src/components/playground/ResponseViewer.tsx` — status code badge, timing, formatted JSON body (syntax highlighted), response headers table
- Error handling: timeout, connection refused, invalid JSON
- **Multipart/file-upload support**: When endpoint `requestBody` contains `multipart/form-data` content type:
  - `src/components/playground/FileUploadField.tsx` — drag-and-drop file input rendered for `type: string, format: binary` schema fields
  - Body editor switches from JSON textarea to form-field mode with individual inputs per form part
  - CORS proxy accepts `FormData` via `req.formData()` and forwards as `multipart/form-data` to target API (preserves original field names, filenames, content types)
  - Max file size: 10MB per file, 25MB total per request (validated client-side and server-side)
  - Supported detection: auto-detect from `requestBody.content['multipart/form-data']` in endpoint spec, show file input for `format: binary` fields and text inputs for other fields

> **Multipart/file-upload support detail:** For endpoints with `multipart/form-data` request bodies, the playground renders a file picker input alongside text fields for non-file parts. Files are sent directly to the target API (not proxied through DocuMerge servers) to avoid payload size limits. The `Content-Type: multipart/form-data` boundary is auto-generated.

**Day 21: Playground — cURL generation**
- `src/components/playground/CurlGenerator.tsx` — generate cURL command from current request state
- Copy cURL to clipboard
- "Try it" button integration on endpoint documentation pages

**Day 22: Playground — session history**
- `src/components/playground/RequestHistory.tsx` — list of recent requests with status codes
- Click to restore a previous request
- Store in playground_sessions table (anonymous via session cookie)

### Phase 5 — GitHub Auto-Sync (Days 23–27)

**Day 23: GitHub App setup and OAuth**
- Register GitHub App with read access to repository contents
- `src/app/api/auth/github-app/route.ts` — installation redirect
- `src/app/api/auth/github-app/callback/route.ts` — save installation ID
- `src/server/services/github-sync.ts` — Octokit setup

**Day 24: GitHub connection UI**
- `src/app/(dashboard)/projects/[slug]/settings/github/page.tsx` — connect GitHub repo
- Repo selector: list repos from installation
- Spec file path input with file browser
- Branch selector
- Auto-sync toggle

**Day 25: GitHub webhook receiver**
- `src/app/api/webhooks/github/route.ts` — verify signature, store event, process
- Handle `push` events: check if spec file was modified, trigger re-sync
- `src/server/services/github-sync.ts` — fetch spec file, re-parse, revalidate ISR

**Day 26: Sync status and error handling**
- Sync status indicator on project card (synced, syncing, error)
- Sync error display with last error message
- Manual "Sync Now" button
- Sync history log: `src/app/(dashboard)/projects/[slug]/settings/sync-log/page.tsx`

**Day 27: Branch and path configuration**
- Multiple spec paths per project (for microservice repos)
- Branch selection for spec file
- Webhook filtering: only trigger on changes to configured path

### Phase 6 — Branding + Billing (Days 28–33)

**Day 28: Branding customizer**
- `src/app/(dashboard)/projects/[slug]/settings/branding/page.tsx` — logo upload, color pickers, font selector
- `src/components/settings/BrandingPreview.tsx` — live preview of docs site with branding
- R2 upload for logos and favicons
- **Free tier**: Primary color + accent color only, DocuMerge branding badge visible in footer
- **Pro tier ($29/mo)**: Logo upload, full color palette (primary, accent, background, text), font selector from 12 curated Google Fonts, remove DocuMerge branding badge
- **Enterprise tier ($99/mo)**: All Pro features + custom CSS editor (`<style>` injection scoped to `.documerge-docs` container, max 10KB, sanitized with DOMPurify to strip `position:fixed`, `z-index>1000`, external `url()` references), custom header/footer HTML (sanitized, max 5KB each)

> **Branding tier separation:** **Free tier** includes logo upload, primary/accent color selection, and font family choice (from 5 curated options). **Pro tier** adds custom CSS injection (validated and sandboxed via CSP) and custom domain support. Day 28 implementation covers Free tier branding only; custom CSS injection is a Pro-tier feature implemented in the billing enforcement layer.

**Day 29: Stripe integration**
- `src/server/services/stripe.ts`, `src/server/routers/billing.ts`
- `src/app/api/webhooks/stripe/route.ts`
- Plan mapping: free (1 project, basic theme, playground, 3 languages, documerge branding), pro ($29/mo: custom domain, full branding, GitHub sync, all languages, no branding, analytics), enterprise ($99/mo: multiple projects, custom CSS, team collaboration, API access)

**Day 30: Plan enforcement**
- `src/server/lib/plan-limits.ts`:
  - Free: 1 project, 3 code languages (cURL, JS, Python), documerge branding, no GitHub sync
  - Pro: 1 project, all languages, custom domain, full branding, GitHub sync, analytics
  - Enterprise: multiple projects, custom CSS, team access
- Middleware enforcement

**Day 31: Billing UI**
- `src/app/(dashboard)/settings/billing/page.tsx`
- Plan comparison, upgrade buttons

**Day 32: Custom domain setup**
- `src/app/(dashboard)/projects/[slug]/settings/domain/page.tsx` — custom domain CNAME instructions
- Domain verification via DNS TXT record
- SSL provisioning notes (Vercel handles via proxy)

**Day 33: OG image and SEO**
- Auto-generate OG image for each project (project name, endpoint count, logo)
- SEO meta tags on all docs pages
- Structured data (JSON-LD) for API documentation
- Sitemap generation per project

### Phase 7 — Analytics + Polish (Days 34–38)

**Day 34: Page view tracking**
- `src/app/api/public/track/route.ts` — beacon endpoint
- Track: endpoint views, search queries, playground usage
- `src/server/routers/analytics.ts` — analytics queries

**Day 35: Analytics dashboard**
- `src/app/(dashboard)/projects/[slug]/analytics/page.tsx`
- Most viewed endpoints
- Playground usage (which endpoints tested most)
- Search queries (what developers look for)
- Traffic over time

**Day 36: Error reference and schema browser**
- `src/app/(docs)/[projectSlug]/errors/page.tsx` — auto-generated error reference from response schemas
- `src/app/(docs)/[projectSlug]/schemas/page.tsx` — schema browser with interactive JSON viewer
- `src/app/(docs)/[projectSlug]/schemas/[name]/page.tsx` — individual schema detail
- `src/components/docs/InteractiveSchemaExplorer.tsx` — interactive schema browser component:
  - **Tree view**: Expandable/collapsible property tree with type badges (`string`, `integer`, `array`, `object`, `enum`), required field indicators (red asterisk), and deprecation strikethrough
  - **Example generation**: "Generate Example" button produces a valid JSON instance from the schema (respects `example`, `default`, `enum`, `format` fields; recursion depth capped at 5 levels)
  - **Copy as TypeScript**: One-click conversion of JSON Schema to TypeScript interface (handles `allOf`/`oneOf`/`anyOf` as intersection/union types, `$ref` resolved to interface name)
  - **Relationship graph**: Minimap showing schema references — clickable nodes linking to referenced schemas (rendered with `reactflow` in a 300px collapsible panel)
  - **Search within schema**: Filter properties by name with instant highlight (debounced 150ms, matches across nested depths)
  - **"Used by" section**: List of endpoints that reference this schema in request body or response (linked to endpoint page)

> **Interactive schema browser:** A dedicated `SchemaViewer` component renders JSON Schema definitions as an expandable tree with: (1) type badges (string, number, object, array, enum), (2) required field indicators, (3) description tooltips, (4) example values inline, (5) nested object expansion (click to drill down). Referenced schemas (`$ref`) are resolved and displayed inline with a 'Jump to definition' link. This component is used both on endpoint detail pages and as a standalone schema explorer accessible via the sidebar.

**Day 37: Polish and edge cases**
- Large spec handling (1000+ endpoints): pagination, lazy-load sidebar groups
- Invalid spec recovery: show partial docs with warnings
- Playground rate limiting
- Mobile docs layout optimization
- Print-friendly CSS

> **Large Spec Handling Test** (1000+ endpoints)
>
> Generate a synthetic OpenAPI 3.1 spec with 1,200 endpoints across 40 tags, 300 component schemas with cross-references, and 150 security-scoped paths. Test targets:
> - **Full render (docs site homepage + sidebar)**: <3s initial load — sidebar uses virtual scrolling (`@tanstack/react-virtual`, 50px row height, 500px visible window) rendering only visible tag groups; endpoint list within each group lazy-loads on expand
> - **Endpoint page render**: <500ms — individual endpoint page with 15+ parameters and nested request/response schemas
> - **Search across all endpoints**: <500ms for query-to-results — Pagefind index pre-built at parse time, client-side filtering with debounced 200ms input
> - **Sidebar scroll performance**: 60fps during fast scroll — virtual scrolling ensures <50 DOM nodes rendered at any time regardless of total endpoint count
> - **Spec re-parse on GitHub sync**: <30s for 1,200 endpoints — batch INSERT (100 endpoints/batch) with transaction, code examples generated in parallel via `Promise.allSettled` (concurrency: 10)
> - **Schema browser load**: <1s for 300 schemas — paginated list (50/page), individual schema detail pages use the same `InteractiveSchemaExplorer` component with recursion depth cap

**Day 38: Launch preparation**
- End-to-end: upload spec → docs generated → playground works → GitHub sync updates
- Test with real-world specs: Stripe, GitHub, Shopify APIs
- Cross-browser docs site testing
- Performance: docs page load <1s, playground response <500ms
- Deploy to production

---

## Critical Files

```
documerge/
├── src/
│   ├── app/
│   │   ├── layout.tsx
│   │   ├── (auth)/
│   │   │   ├── sign-in/[[...sign-in]]/page.tsx
│   │   │   └── sign-up/[[...sign-up]]/page.tsx
│   │   ├── (dashboard)/
│   │   │   ├── layout.tsx                        # Dashboard shell
│   │   │   ├── projects/
│   │   │   │   ├── page.tsx                      # Project list
│   │   │   │   ├── new/page.tsx                  # Create project wizard
│   │   │   │   └── [slug]/
│   │   │   │       ├── page.tsx                  # Project overview
│   │   │   │       ├── analytics/page.tsx        # Analytics
│   │   │   │       └── settings/
│   │   │   │           ├── page.tsx              # General settings
│   │   │   │           ├── branding/page.tsx     # Logo, colors, fonts
│   │   │   │           ├── github/page.tsx       # GitHub sync config
│   │   │   │           ├── domain/page.tsx       # Custom domain
│   │   │   │           └── sync-log/page.tsx     # Sync history
│   │   │   └── settings/
│   │   │       └── billing/page.tsx              # Plan management
│   │   ├── (docs)/
│   │   │   └── [projectSlug]/
│   │   │       ├── layout.tsx                    # Docs layout (sidebar + header)
│   │   │       ├── page.tsx                      # API overview
│   │   │       ├── [...path]/page.tsx            # Endpoint pages (catch-all)
│   │   │       ├── errors/page.tsx               # Error reference
│   │   │       └── schemas/
│   │   │           ├── page.tsx                  # Schema browser
│   │   │           └── [name]/page.tsx           # Schema detail
│   │   └── api/
│   │       ├── trpc/[trpc]/route.ts
│   │       ├── auth/github-app/
│   │       │   ├── route.ts                      # GitHub App install
│   │       │   └── callback/route.ts             # Install callback
│   │       ├── public/
│   │       │   ├── playground/proxy/route.ts     # CORS proxy
│   │       │   └── track/route.ts                # Analytics beacon
│   │       └── webhooks/
│   │           ├── github/route.ts               # Push webhook
│   │           └── stripe/route.ts               # Billing webhook
│   ├── server/
│   │   ├── db/
│   │   │   ├── index.ts
│   │   │   └── schema.ts                        # 10 tables + relations
│   │   ├── trpc.ts
│   │   ├── routers/
│   │   │   ├── _app.ts
│   │   │   ├── org.ts
│   │   │   ├── project.ts                       # Project CRUD + import
│   │   │   ├── version.ts                       # API version management
│   │   │   ├── endpoint.ts                      # Endpoint queries
│   │   │   ├── analytics.ts                     # Analytics queries
│   │   │   └── billing.ts                       # Stripe checkout/portal
│   │   ├── services/
│   │   │   ├── openapi-parser.ts                # Parse + extract + store
│   │   │   ├── code-examples.ts                 # Handlebars code gen
│   │   │   ├── github-sync.ts                   # Fetch spec + re-parse
│   │   │   ├── stripe.ts
│   │   │   ├── r2.ts                            # Logo/asset storage
│   │   │   └── email.ts                         # Resend client
│   │   └── lib/
│   │       ├── org-seed.ts
│   │       ├── plan-limits.ts
│   │       └── github-webhook-verify.ts
│   └── components/
│       ├── docs/
│       │   ├── Sidebar.tsx                      # Navigation tree
│       │   ├── Header.tsx                       # Logo, search, version, dark mode
│       │   ├── EndpointPage.tsx                 # Full endpoint documentation
│       │   ├── SchemaViewer.tsx                 # Interactive JSON schema tree
│       │   ├── CodeExamples.tsx                 # Language tabs + copy
│       │   ├── AuthDocs.tsx                     # Authentication section
│       │   ├── SearchDialog.tsx                 # Cmd+K search (Pagefind)
│       │   └── DocsTheme.tsx                    # Branding application
│       ├── playground/
│       │   ├── PlaygroundPanel.tsx              # Split request/response
│       │   ├── RequestBuilder.tsx               # URL + params + headers
│       │   ├── BodyEditor.tsx                   # JSON body editor
│       │   ├── AuthInput.tsx                    # Auth credential input
│       │   ├── ResponseViewer.tsx               # Status + body + headers
│       │   ├── CurlGenerator.tsx                # cURL command display
│       │   └── RequestHistory.tsx               # Recent requests
│       ├── settings/
│       │   └── BrandingPreview.tsx              # Live docs preview
│       └── billing/
│           └── PlanCard.tsx
├── drizzle.config.ts
├── vercel.json
├── .env.local
└── package.json
```

---

## Verification

### Manual Testing Checklist

1. **Upload OpenAPI spec** — upload Petstore spec (YAML), verify parsed, endpoints extracted
2. **URL import** — import from public spec URL, same result as file upload
3. **Validation errors** — upload invalid spec, verify clear error messages
4. **Docs site renders** — visit `/{projectSlug}`, verify sidebar + endpoint pages
5. **Endpoint documentation** — each endpoint shows: method, path, parameters, request body, responses, code examples
6. **Code examples** — cURL, JS, Python generated with correct auth, params, body
7. **Copy code** — click copy button, verify code copied to clipboard
8. **Schema viewer** — expand nested objects, verify all properties displayed with types
9. **API playground** — open "Try it", fill parameters, send request, verify response displayed
10. **Playground CORS proxy** — test against API without CORS headers, verify proxy works
11. **Playground auth** — enter API key, verify included in request
12. **Dark mode** — toggle dark mode, verify all docs pages render correctly
13. **Search** — search for endpoint by keyword, verify results link to correct page
14. **GitHub sync** — connect repo, push spec change, verify docs update within 60s
15. **Branding** — set logo + colors, verify reflected on docs site
16. **Plan limits** — free user: try creating second project, verify blocked
17. **Large spec handling test** — import an OpenAPI spec with 200+ endpoints and 50+ schema components, then verify: (1) import completes in < 30 seconds, (2) sidebar renders all endpoint groups without lag, (3) search across all endpoints returns results in < 500ms, (4) schema browser handles deeply nested objects (5+ levels) without stack overflow

### SQL Verification Queries

```sql
-- 1. Projects per org with endpoint counts
SELECT o.name, ap.name as project, ap.slug,
  COUNT(e.id) as endpoint_count,
  ap.last_synced_at, ap.auto_sync_enabled
FROM organizations o
JOIN api_projects ap ON ap.org_id = o.id
LEFT JOIN api_versions av ON av.project_id = ap.id AND av.is_latest = true
LEFT JOIN endpoints e ON e.version_id = av.id
GROUP BY o.name, ap.name, ap.slug, ap.last_synced_at, ap.auto_sync_enabled;

-- 2. Most viewed endpoints
SELECT e.method, e.path, e.summary, COUNT(pv.id) as views
FROM page_views pv
JOIN endpoints e ON e.id = pv.endpoint_id
WHERE pv.project_id = '<project_id>'
  AND pv.viewed_at >= NOW() - INTERVAL '30 days'
GROUP BY e.method, e.path, e.summary
ORDER BY views DESC LIMIT 10;

-- 3. Playground usage
SELECT e.method, e.path, COUNT(ps.id) as requests
FROM playground_sessions ps
JOIN endpoints e ON e.id = ps.endpoint_id
WHERE ps.project_id = '<project_id>'
GROUP BY e.method, e.path
ORDER BY requests DESC LIMIT 10;

-- 4. GitHub sync health
SELECT ap.name, ap.last_synced_at, ap.last_sync_error,
  ap.auto_sync_enabled,
  COUNT(gwe.id) as webhook_count,
  SUM(CASE WHEN gwe.status = 'failed' THEN 1 ELSE 0 END) as failed_count
FROM api_projects ap
LEFT JOIN github_webhook_events gwe ON gwe.project_id = ap.id
WHERE ap.auto_sync_enabled = true
GROUP BY ap.name, ap.last_synced_at, ap.last_sync_error, ap.auto_sync_enabled;

-- 5. Endpoint coverage (how many endpoints have code examples)
SELECT
  COUNT(*) as total_endpoints,
  SUM(CASE WHEN code_examples != '{}' THEN 1 ELSE 0 END) as with_examples
FROM endpoints e
JOIN api_versions av ON av.id = e.version_id AND av.is_latest = true;
```

### Performance Benchmarks

| Operation | Target | Method |
|-----------|--------|--------|
| Spec parsing (50 endpoints) | <3s | Parse + extract + code gen |
| Spec parsing (500 endpoints) | <15s | Large spec handling |
| Docs page load (ISR) | <500ms | Static HTML, edge-cached |
| Docs page ISR revalidation | <5s | On-demand revalidation |
| Endpoint page render | <200ms | Server component with DB query |
| Playground proxy request | <30s | Forward + response (timeout limit) |
| Playground cURL generation | <50ms | Client-side template |
| Search (Pagefind) | <100ms | Client-side static index |
| GitHub sync (spec update) | <10s | Fetch + parse + revalidate |
| Code example generation | <1s | Handlebars template per endpoint |
| Large spec render (1200 endpoints) | <3s | Virtual scrolling sidebar, lazy-load groups |
| Large spec search (1200 endpoints) | <500ms | Pre-built Pagefind index, client-side |
| Large spec re-parse (1200 endpoints) | <30s | Batch INSERT (100/batch), parallel code gen |
| Schema browser (300 schemas) | <1s | Paginated list, 50/page |

---

## Post-MVP Roadmap

| # | Feature | Description | Est. Effort |
|---|---------|-------------|-------------|
| F5 | GraphQL Spec Support | Import GraphQL SDL / introspection, generate query explorer + schema docs | 2 weeks |
| F6 | Versioned Docs Diff | Side-by-side changelog between API versions highlighting added/removed/changed endpoints | 1 week |
| F7 | Team Collaboration | Comments on endpoints, suggested edits, review workflows (Enterprise) | 2 weeks |
| F8 | Webhook Documentation | Dedicated webhook section with payload schemas, delivery logs, test sender | 1 week |
| F9 | AI Quickstart Guide | GPT-4o generates a "Getting Started" tutorial from the spec (auth setup, first request, pagination) | 3 days |
| F10 | Interactive Schema Explorer | Full schema browser with relationship graph, TypeScript export, example generation (see Day 36 spec) | 1 week |
| F11 | Additional Code Languages (v2) | Go, Ruby, PHP, Java, C#, Rust templates + community contribution framework | 2 weeks |
| F12 | SDK Generation | Auto-generate typed SDK packages (TypeScript, Python) from OpenAPI spec, publish to npm/PyPI | 3 weeks |
| F13 | Custom Domain SSL | Automated SSL provisioning via Let's Encrypt for custom domains (currently relies on Vercel proxy) | 1 week |
| F14 | API Changelog Feed | RSS/Atom feed + email digest of spec changes, subscriber management | 1 week |
| F15 | Embed Widgets | Embeddable endpoint documentation widgets (`<iframe>` or web components) for external sites | 1 week |
