# 14. PriceWise — Dynamic Pricing Optimizer

## Implementation Plan

**MVP Scope:** Shopify integration via Admin API + OAuth (product catalog sync, order history import, inventory levels, webhook-based real-time updates for orders/products/inventory), competitor price monitoring (manual URL entry per product, daily Playwright-based scraping with BullMQ scheduled jobs, price change alerts), AI price recommendations powered by GPT-4o (analyzes historical sales velocity, competitor price positioning, cost/margin data, and seasonality signals to produce per-product recommended prices with human-readable reasoning), price history tracking as an append-only event log with revenue overlay charts, competitor comparison view showing your price vs. competitor prices over time, product pricing dashboard with accept/reject/modify recommendation workflow, and Stripe billing with two tiers (Starter $49/mo for 100 products + 5 competitors, Growth $149/mo for 1,000 products + 20 competitors + pricing rules).

---

## Tech Decisions

| Layer | Technology | Notes |
|-------|-----------|-------|
| Framework | Next.js 14+ App Router | `src/` directory, TypeScript strict |
| API | tRPC v11 | superjson transformer, Clerk auth context |
| Database | Neon PostgreSQL | Append-only price history, JSONB for flexible configs |
| ORM | Drizzle ORM | Push-based migrations, typed schema |
| Auth | Clerk | Organization-scoped, `orgId` from session |
| Billing | Stripe | Checkout Sessions + Customer Portal + Webhooks |
| AI | OpenAI GPT-4o | JSON mode, temperature 0.2 for price recommendations with reasoning |
| Queue | BullMQ + Redis (Upstash) | Scheduled competitor scraping + Shopify sync jobs |
| Shopify | Shopify Admin API + OAuth | Product, order, inventory sync via REST + webhooks |
| Scraping | Playwright | Headless Chromium for competitor price extraction |
| UI | Tailwind CSS, Radix UI, Recharts | Price history charts, competitor comparison views |
| Hosting | Vercel (app), Railway (scraping workers) | Workers need long-running processes + Playwright binary |
| Monitoring | Sentry | Error tracking + performance monitoring |

### Key Architecture Decisions

1. **Shopify OAuth + webhook-based sync**: The app authenticates via Shopify OAuth 2.0 and performs an initial bulk import of products, orders (last 12 months), and inventory levels. After the initial sync, Shopify webhooks (`products/update`, `orders/create`, `inventory_levels/update`) keep data current in real-time. The webhook receiver validates HMAC signatures, normalizes the payload, and upserts into local tables. This avoids polling and keeps data fresh within seconds of changes.

2. **AI recommendation engine with structured reasoning**: GPT-4o receives a structured context payload containing the product's sales history (units/revenue per week for the last 12 weeks), current and historical competitor prices, cost basis, current margin, inventory level, and category benchmarks. It returns a JSON response with `recommendedPrice`, `confidenceScore`, `expectedRevenueChangePct`, and a `reasoning` array of human-readable factor explanations. Temperature is set to 0.2 for consistent, conservative recommendations.

3. **BullMQ scheduled competitor scraping with Playwright**: A repeatable BullMQ job runs daily (configurable per org) that iterates through all active `competitorProducts` with URLs, launches Playwright in headless mode, navigates to each URL, and extracts the current price using CSS selectors stored in `competitorProducts.scrapeConfig`. Failed scrapes retry with exponential backoff. The worker runs on Railway (not Vercel) because Playwright requires a persistent Chromium binary and long execution times.

4. **Price history as append-only event log**: Every price change (whether from Shopify sync, manual edit, accepted recommendation, or pricing rule) is recorded as a new row in `priceHistory` with `effectiveAt` timestamp, source attribution, and the reason for the change. The current price is always derivable from the most recent entry. This enables revenue impact analysis by correlating price change events with order volume changes in the weeks following each change.

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
  numeric,
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
  plan: text("plan").default("starter").notNull(), // starter | growth
  stripeCustomerId: text("stripe_customer_id"),
  stripeSubscriptionId: text("stripe_subscription_id"),
  settings: jsonb("settings").default({}).$type<{
    defaultCurrency?: string;
    defaultMarginTarget?: number;       // e.g. 0.30 for 30%
    priceEndingRule?: string;            // "none" | "99" | "95" | "00"
    scrapeFrequencyHours?: number;       // default 24
    autoApplyRecommendations?: boolean;  // default false
    maxPriceChangePct?: number;          // guardrail: max % change per recommendation
    onboardingCompleted?: boolean;
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
    createdAt: timestamp("created_at", { withTimezone: true }).defaultNow().notNull(),
  },
  (table) => [
    uniqueIndex("members_org_user_idx").on(table.orgId, table.clerkUserId),
  ]
);

// ---------------------------------------------------------------------------
// Data Sources (Shopify stores)
// ---------------------------------------------------------------------------
export const dataSources = pgTable(
  "data_sources",
  {
    id: uuid("id").primaryKey().defaultRandom(),
    orgId: uuid("org_id")
      .references(() => organizations.id, { onDelete: "cascade" })
      .notNull(),
    type: text("type").notNull(), // shopify
    name: text("name").notNull(), // "My Shopify Store"
    config: jsonb("config").default({}).$type<{
      shopifyShop?: string;            // e.g. "my-store.myshopify.com"
      shopifyAccessToken?: string;     // AES-256-GCM encrypted
      shopifyWebhookSecret?: string;
      shopifyScopes?: string[];
    }>(),
    status: text("status").default("active").notNull(), // active | paused | error
    errorMessage: text("error_message"),
    lastSyncedAt: timestamp("last_synced_at", { withTimezone: true }),
    createdAt: timestamp("created_at", { withTimezone: true }).defaultNow().notNull(),
  },
  (table) => [
    index("data_sources_org_id_idx").on(table.orgId),
  ]
);

// ---------------------------------------------------------------------------
// Products
// ---------------------------------------------------------------------------
export const products = pgTable(
  "products",
  {
    id: uuid("id").primaryKey().defaultRandom(),
    orgId: uuid("org_id")
      .references(() => organizations.id, { onDelete: "cascade" })
      .notNull(),
    dataSourceId: uuid("data_source_id")
      .references(() => dataSources.id, { onDelete: "set null" }),
    externalId: text("external_id"),          // Shopify product ID
    variantId: text("variant_id"),            // Shopify variant ID
    name: text("name").notNull(),
    sku: text("sku"),
    currentPrice: numeric("current_price", { precision: 12, scale: 2 }).notNull(),
    cost: numeric("cost", { precision: 12, scale: 2 }),
    currency: text("currency").default("USD").notNull(),
    category: text("category"),
    tags: jsonb("tags").default([]).$type<string[]>(),
    inventoryLevel: integer("inventory_level"),
    imageUrl: text("image_url"),
    status: text("status").default("active").notNull(), // active | archived
    shopifyData: jsonb("shopify_data").default({}).$type<{
      productType?: string;
      vendor?: string;
      handle?: string;
      publishedAt?: string;
    }>(),
    lastRecommendationAt: timestamp("last_recommendation_at", { withTimezone: true }),
    priceLocked: boolean("price_locked").default(false).notNull(), // manual override lock
    createdAt: timestamp("created_at", { withTimezone: true }).defaultNow().notNull(),
    updatedAt: timestamp("updated_at", { withTimezone: true }).defaultNow().notNull(),
  },
  (table) => [
    index("products_org_id_idx").on(table.orgId),
    index("products_external_id_idx").on(table.orgId, table.externalId),
    index("products_sku_idx").on(table.orgId, table.sku),
    index("products_category_idx").on(table.orgId, table.category),
  ]
);

// ---------------------------------------------------------------------------
// Price History (append-only event log)
// ---------------------------------------------------------------------------
export const priceHistory = pgTable(
  "price_history",
  {
    id: uuid("id").primaryKey().defaultRandom(),
    orgId: uuid("org_id")
      .references(() => organizations.id, { onDelete: "cascade" })
      .notNull(),
    productId: uuid("product_id")
      .references(() => products.id, { onDelete: "cascade" })
      .notNull(),
    price: numeric("price", { precision: 12, scale: 2 }).notNull(),
    previousPrice: numeric("previous_price", { precision: 12, scale: 2 }),
    source: text("source").notNull(), // shopify_sync | manual | recommendation | rule
    reason: text("reason"),            // human-readable reason for the change
    recommendationId: uuid("recommendation_id"), // if accepted from a recommendation
    metadata: jsonb("metadata").default({}).$type<{
      ruleId?: string;
      userId?: string;          // Clerk userId who accepted/changed
      shopifyVariantId?: string;
    }>(),
    effectiveAt: timestamp("effective_at", { withTimezone: true }).defaultNow().notNull(),
    createdAt: timestamp("created_at", { withTimezone: true }).defaultNow().notNull(),
  },
  (table) => [
    index("price_history_product_id_idx").on(table.productId),
    index("price_history_effective_at_idx").on(table.productId, table.effectiveAt),
    index("price_history_org_id_idx").on(table.orgId),
  ]
);

// ---------------------------------------------------------------------------
// Competitors
// ---------------------------------------------------------------------------
export const competitors = pgTable(
  "competitors",
  {
    id: uuid("id").primaryKey().defaultRandom(),
    orgId: uuid("org_id")
      .references(() => organizations.id, { onDelete: "cascade" })
      .notNull(),
    name: text("name").notNull(),
    domain: text("domain"),
    notes: text("notes"),
    createdAt: timestamp("created_at", { withTimezone: true }).defaultNow().notNull(),
  },
  (table) => [
    index("competitors_org_id_idx").on(table.orgId),
  ]
);

// ---------------------------------------------------------------------------
// Competitor Products (tracked URLs)
// ---------------------------------------------------------------------------
export const competitorProducts = pgTable(
  "competitor_products",
  {
    id: uuid("id").primaryKey().defaultRandom(),
    orgId: uuid("org_id")
      .references(() => organizations.id, { onDelete: "cascade" })
      .notNull(),
    competitorId: uuid("competitor_id")
      .references(() => competitors.id, { onDelete: "cascade" })
      .notNull(),
    productId: uuid("product_id")
      .references(() => products.id, { onDelete: "cascade" })
      .notNull(),
    url: text("url").notNull(),
    name: text("name"),                // competitor's product name
    currentPrice: numeric("current_price", { precision: 12, scale: 2 }),
    currency: text("currency").default("USD").notNull(),
    inStock: boolean("in_stock").default(true),
    scrapeConfig: jsonb("scrape_config").default({}).$type<{
      priceSelector?: string;          // CSS selector for price element
      stockSelector?: string;          // CSS selector for stock status
      waitForSelector?: string;        // element to wait for before scraping
      priceRegex?: string;             // regex to extract numeric price from text
    }>(),
    lastCheckedAt: timestamp("last_checked_at", { withTimezone: true }),
    lastSuccessAt: timestamp("last_success_at", { withTimezone: true }),
    consecutiveFailures: integer("consecutive_failures").default(0).notNull(),
    status: text("status").default("active").notNull(), // active | paused | error
    createdAt: timestamp("created_at", { withTimezone: true }).defaultNow().notNull(),
  },
  (table) => [
    index("competitor_products_org_id_idx").on(table.orgId),
    index("competitor_products_competitor_id_idx").on(table.competitorId),
    index("competitor_products_product_id_idx").on(table.productId),
  ]
);

// ---------------------------------------------------------------------------
// Competitor Price Checks (scrape results log)
// ---------------------------------------------------------------------------
export const competitorPriceChecks = pgTable(
  "competitor_price_checks",
  {
    id: uuid("id").primaryKey().defaultRandom(),
    competitorProductId: uuid("competitor_product_id")
      .references(() => competitorProducts.id, { onDelete: "cascade" })
      .notNull(),
    price: numeric("price", { precision: 12, scale: 2 }),
    previousPrice: numeric("previous_price", { precision: 12, scale: 2 }),
    inStock: boolean("in_stock"),
    success: boolean("success").default(true).notNull(),
    errorMessage: text("error_message"),
    screenshotUrl: text("screenshot_url"),   // optional debug screenshot
    checkedAt: timestamp("checked_at", { withTimezone: true }).defaultNow().notNull(),
  },
  (table) => [
    index("competitor_price_checks_cp_id_idx").on(table.competitorProductId),
    index("competitor_price_checks_checked_at_idx").on(table.competitorProductId, table.checkedAt),
  ]
);

// ---------------------------------------------------------------------------
// Recommendations
// ---------------------------------------------------------------------------
export const recommendations = pgTable(
  "recommendations",
  {
    id: uuid("id").primaryKey().defaultRandom(),
    orgId: uuid("org_id")
      .references(() => organizations.id, { onDelete: "cascade" })
      .notNull(),
    productId: uuid("product_id")
      .references(() => products.id, { onDelete: "cascade" })
      .notNull(),
    currentPrice: numeric("current_price", { precision: 12, scale: 2 }).notNull(),
    recommendedPrice: numeric("recommended_price", { precision: 12, scale: 2 }).notNull(),
    expectedRevenueChangePct: real("expected_revenue_change_pct"), // e.g. 0.12 for +12%
    expectedMargin: real("expected_margin"),
    confidenceScore: real("confidence_score"),  // 0-1
    reasoning: jsonb("reasoning").default([]).$type<{
      factor: string;
      direction: "up" | "down" | "neutral";
      detail: string;
    }[]>(),
    factors: jsonb("factors").default({}).$type<{
      salesVelocity?: { last4Weeks: number; trend: string };
      competitorAvgPrice?: number;
      competitorMinPrice?: number;
      competitorMaxPrice?: number;
      currentMargin?: number;
      inventoryLevel?: number;
      categoryAvgPrice?: number;
    }>(),
    status: text("status").default("pending").notNull(), // pending | accepted | rejected | expired
    respondedAt: timestamp("responded_at", { withTimezone: true }),
    respondedByUserId: text("responded_by_user_id"),
    appliedPrice: numeric("applied_price", { precision: 12, scale: 2 }), // actual price set (may differ)
    createdAt: timestamp("created_at", { withTimezone: true }).defaultNow().notNull(),
    expiresAt: timestamp("expires_at", { withTimezone: true }),
  },
  (table) => [
    index("recommendations_org_id_idx").on(table.orgId),
    index("recommendations_product_id_idx").on(table.productId),
    index("recommendations_status_idx").on(table.orgId, table.status),
    index("recommendations_created_at_idx").on(table.createdAt),
  ]
);

// ---------------------------------------------------------------------------
// Pricing Rules (Growth plan)
// ---------------------------------------------------------------------------
export const pricingRules = pgTable(
  "pricing_rules",
  {
    id: uuid("id").primaryKey().defaultRandom(),
    orgId: uuid("org_id")
      .references(() => organizations.id, { onDelete: "cascade" })
      .notNull(),
    name: text("name").notNull(),
    type: text("type").notNull(), // competitor_based | inventory_based | margin_floor
    conditions: jsonb("conditions").default({}).$type<{
      // competitor_based
      competitorId?: string;
      priceRelation?: "below" | "match" | "above";
      priceOffset?: number;          // absolute dollar offset
      priceOffsetPct?: number;       // percentage offset
      // inventory_based
      inventoryBelow?: number;
      inventoryAbove?: number;
      priceAdjustPct?: number;
      // margin_floor
      minMarginPct?: number;
    }>(),
    appliesTo: jsonb("applies_to").default({}).$type<{
      productIds?: string[];
      categories?: string[];
      allProducts?: boolean;
    }>(),
    priority: integer("priority").default(0).notNull(),
    isActive: boolean("is_active").default(true).notNull(),
    lastTriggeredAt: timestamp("last_triggered_at", { withTimezone: true }),
    createdAt: timestamp("created_at", { withTimezone: true }).defaultNow().notNull(),
    updatedAt: timestamp("updated_at", { withTimezone: true }).defaultNow().notNull(),
  },
  (table) => [
    index("pricing_rules_org_id_idx").on(table.orgId),
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
    type: text("type").notNull(), // price_change_alert | competitor_alert | recommendation_ready | rule_triggered
    channel: text("channel").default("in_app").notNull(), // in_app | email
    title: text("title").notNull(),
    body: text("body"),
    metadata: jsonb("metadata").default({}).$type<{
      productId?: string;
      competitorProductId?: string;
      recommendationId?: string;
      ruleId?: string;
      priceFrom?: string;
      priceTo?: string;
    }>(),
    readAt: timestamp("read_at", { withTimezone: true }),
    createdAt: timestamp("created_at", { withTimezone: true }).defaultNow().notNull(),
  },
  (table) => [
    index("notification_log_org_id_idx").on(table.orgId),
    index("notification_log_read_at_idx").on(table.orgId, table.readAt),
  ]
);

// ---------------------------------------------------------------------------
// Relations
// ---------------------------------------------------------------------------
export const organizationsRelations = relations(organizations, ({ many }) => ({
  members: many(members),
  dataSources: many(dataSources),
  products: many(products),
  competitors: many(competitors),
  recommendations: many(recommendations),
  pricingRules: many(pricingRules),
  notificationLog: many(notificationLog),
}));

export const membersRelations = relations(members, ({ one }) => ({
  organization: one(organizations, {
    fields: [members.orgId],
    references: [organizations.id],
  }),
}));

export const dataSourcesRelations = relations(dataSources, ({ one, many }) => ({
  organization: one(organizations, {
    fields: [dataSources.orgId],
    references: [organizations.id],
  }),
  products: many(products),
}));

export const productsRelations = relations(products, ({ one, many }) => ({
  organization: one(organizations, {
    fields: [products.orgId],
    references: [organizations.id],
  }),
  dataSource: one(dataSources, {
    fields: [products.dataSourceId],
    references: [dataSources.id],
  }),
  priceHistory: many(priceHistory),
  competitorProducts: many(competitorProducts),
  recommendations: many(recommendations),
}));

export const priceHistoryRelations = relations(priceHistory, ({ one }) => ({
  product: one(products, {
    fields: [priceHistory.productId],
    references: [products.id],
  }),
  organization: one(organizations, {
    fields: [priceHistory.orgId],
    references: [organizations.id],
  }),
}));

export const competitorsRelations = relations(competitors, ({ one, many }) => ({
  organization: one(organizations, {
    fields: [competitors.orgId],
    references: [organizations.id],
  }),
  competitorProducts: many(competitorProducts),
}));

export const competitorProductsRelations = relations(competitorProducts, ({ one, many }) => ({
  competitor: one(competitors, {
    fields: [competitorProducts.competitorId],
    references: [competitors.id],
  }),
  product: one(products, {
    fields: [competitorProducts.productId],
    references: [products.id],
  }),
  priceChecks: many(competitorPriceChecks),
}));

export const competitorPriceChecksRelations = relations(competitorPriceChecks, ({ one }) => ({
  competitorProduct: one(competitorProducts, {
    fields: [competitorPriceChecks.competitorProductId],
    references: [competitorProducts.id],
  }),
}));

export const recommendationsRelations = relations(recommendations, ({ one }) => ({
  organization: one(organizations, {
    fields: [recommendations.orgId],
    references: [organizations.id],
  }),
  product: one(products, {
    fields: [recommendations.productId],
    references: [products.id],
  }),
}));

export const pricingRulesRelations = relations(pricingRules, ({ one }) => ({
  organization: one(organizations, {
    fields: [pricingRules.orgId],
    references: [organizations.id],
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
export type DataSource = typeof dataSources.$inferSelect;
export type Product = typeof products.$inferSelect;
export type NewProduct = typeof products.$inferInsert;
export type PriceHistoryRow = typeof priceHistory.$inferSelect;
export type Competitor = typeof competitors.$inferSelect;
export type CompetitorProduct = typeof competitorProducts.$inferSelect;
export type CompetitorPriceCheck = typeof competitorPriceChecks.$inferSelect;
export type Recommendation = typeof recommendations.$inferSelect;
export type PricingRule = typeof pricingRules.$inferSelect;
export type NotificationLogRow = typeof notificationLog.$inferSelect;
```

### Initial Migration SQL

```sql
-- src/server/db/migrations/0000_init.sql

-- Enable required extensions
CREATE EXTENSION IF NOT EXISTS pg_trgm;

-- (All tables created by Drizzle push)

-- Trigram index for fuzzy product name search
CREATE INDEX IF NOT EXISTS products_name_trgm_idx
  ON products USING gin (name gin_trgm_ops);

-- Partial index for pending recommendations (most-queried status)
CREATE INDEX IF NOT EXISTS recommendations_pending_idx
  ON recommendations (org_id, created_at DESC)
  WHERE status = 'pending';

-- Composite index for price history time-range queries
CREATE INDEX IF NOT EXISTS price_history_org_time_idx
  ON price_history (org_id, effective_at DESC);
```

---

## Architecture Deep-Dives

### 1. Shopify OAuth + Product/Order Sync Pipeline

```typescript
// src/server/integrations/shopify/oauth.ts

import crypto from "crypto";
import { db } from "@/server/db";
import { dataSources, organizations } from "@/server/db/schema";
import { encrypt } from "@/server/integrations/encryption";
import { eq, and } from "drizzle-orm";

const SHOPIFY_API_VERSION = "2024-10";

interface ShopifyTokenResponse {
  access_token: string;
  scope: string;
}

export function buildInstallUrl(shop: string, orgId: string, state: string): string {
  const scopes = [
    "read_products", "write_products",
    "read_orders",
    "read_inventory",
  ].join(",");

  const redirectUri = `${process.env.NEXT_PUBLIC_APP_URL}/api/auth/shopify/callback`;

  return `https://${shop}/admin/oauth/authorize?` +
    `client_id=${process.env.SHOPIFY_CLIENT_ID}` +
    `&scope=${scopes}` +
    `&redirect_uri=${encodeURIComponent(redirectUri)}` +
    `&state=${state}`;
}

export async function exchangeCodeForToken(
  shop: string,
  code: string
): Promise<ShopifyTokenResponse> {
  const res = await fetch(
    `https://${shop}/admin/oauth/access_token`,
    {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({
        client_id: process.env.SHOPIFY_CLIENT_ID,
        client_secret: process.env.SHOPIFY_CLIENT_SECRET,
        code,
      }),
    }
  );

  if (!res.ok) {
    throw new Error(`Shopify token exchange failed: ${res.status}`);
  }

  return res.json();
}

export function verifyHmac(query: Record<string, string>): boolean {
  const { hmac, ...params } = query;
  const sorted = Object.keys(params).sort()
    .map((key) => `${key}=${params[key]}`)
    .join("&");

  const computed = crypto
    .createHmac("sha256", process.env.SHOPIFY_CLIENT_SECRET!)
    .update(sorted)
    .digest("hex");

  return crypto.timingSafeEqual(
    Buffer.from(computed, "hex"),
    Buffer.from(hmac, "hex")
  );
}

export function verifyWebhookHmac(body: string, hmacHeader: string): boolean {
  const computed = crypto
    .createHmac("sha256", process.env.SHOPIFY_CLIENT_SECRET!)
    .update(body, "utf8")
    .digest("base64");

  return crypto.timingSafeEqual(
    Buffer.from(computed, "base64"),
    Buffer.from(hmacHeader, "base64")
  );
}

// --- Sync pipeline ---

export async function syncShopifyProducts(dataSourceId: string): Promise<number> {
  const [source] = await db.select().from(dataSources)
    .where(eq(dataSources.id, dataSourceId)).limit(1);

  if (!source || !source.config?.shopifyAccessToken) {
    throw new Error("Invalid data source or missing credentials");
  }

  const shop = source.config.shopifyShop!;
  const token = decrypt(source.config.shopifyAccessToken);
  let synced = 0;
  let pageInfo: string | null = null;

  do {
    const url = pageInfo
      ? `https://${shop}/admin/api/${SHOPIFY_API_VERSION}/products.json?limit=250&page_info=${pageInfo}`
      : `https://${shop}/admin/api/${SHOPIFY_API_VERSION}/products.json?limit=250&status=active`;

    const res = await fetch(url, {
      headers: { "X-Shopify-Access-Token": token },
    });
    const data = await res.json();

    for (const product of data.products) {
      for (const variant of product.variants) {
        await upsertProductFromShopify(source.orgId, dataSourceId, product, variant);
        synced++;
      }
    }

    // Parse Link header for pagination
    const linkHeader = res.headers.get("link");
    pageInfo = parseLinkHeader(linkHeader, "next");
  } while (pageInfo);

  await db.update(dataSources)
    .set({ lastSyncedAt: new Date() })
    .where(eq(dataSources.id, dataSourceId));

  return synced;
}
```

### 2. AI Price Recommendation Engine

```typescript
// src/server/ai/recommend-price.ts

import OpenAI from "openai";
import { db } from "@/server/db";
import {
  products, priceHistory, competitorProducts,
  competitorPriceChecks, recommendations,
} from "@/server/db/schema";
import { eq, and, desc, gte, sql } from "drizzle-orm";

const openai = new OpenAI({ apiKey: process.env.OPENAI_API_KEY });

interface SalesDataPoint {
  week: string;
  unitsSold: number;
  revenue: number;
  avgPrice: number;
}

interface CompetitorSnapshot {
  competitorName: string;
  currentPrice: number;
  priceHistory: { date: string; price: number }[];
}

interface RecommendationResult {
  recommendedPrice: number;
  expectedRevenueChangePct: number;
  expectedMargin: number;
  confidenceScore: number;
  reasoning: { factor: string; direction: "up" | "down" | "neutral"; detail: string }[];
  factors: Record<string, unknown>;
}

async function gatherProductContext(productId: string, orgId: string) {
  const [product] = await db.select().from(products)
    .where(and(eq(products.id, productId), eq(products.orgId, orgId)))
    .limit(1);

  if (!product) throw new Error("Product not found");

  // Last 12 weeks of sales data from orders
  const salesData = await db.execute<SalesDataPoint>(sql`
    SELECT
      date_trunc('week', ph.effective_at) AS week,
      COUNT(*) AS units_sold,
      SUM(ph.price::numeric) AS revenue,
      AVG(ph.price::numeric) AS avg_price
    FROM price_history ph
    WHERE ph.product_id = ${productId}
      AND ph.effective_at >= NOW() - INTERVAL '12 weeks'
    GROUP BY date_trunc('week', ph.effective_at)
    ORDER BY week DESC
  `);

  // Competitor prices
  const compProducts = await db.select().from(competitorProducts)
    .where(eq(competitorProducts.productId, productId));

  const competitorSnapshots: CompetitorSnapshot[] = [];
  for (const cp of compProducts) {
    const checks = await db.select().from(competitorPriceChecks)
      .where(eq(competitorPriceChecks.competitorProductId, cp.id))
      .orderBy(desc(competitorPriceChecks.checkedAt))
      .limit(30);

    competitorSnapshots.push({
      competitorName: cp.name ?? "Unknown",
      currentPrice: Number(cp.currentPrice ?? 0),
      priceHistory: checks
        .filter((c) => c.price !== null)
        .map((c) => ({
          date: c.checkedAt.toISOString().split("T")[0],
          price: Number(c.price),
        })),
    });
  }

  // Recent price history for this product
  const recentPrices = await db.select().from(priceHistory)
    .where(eq(priceHistory.productId, productId))
    .orderBy(desc(priceHistory.effectiveAt))
    .limit(20);

  return { product, salesData, competitorSnapshots, recentPrices };
}

const RECOMMENDATION_PROMPT = `You are a pricing optimization AI for e-commerce products. Analyze the provided product data and recommend an optimal price.

PRODUCT CONTEXT:
- Name: {productName}
- Current Price: ${"{currentPrice}"}
- Cost: ${"{cost}"} (margin: {currentMargin}%)
- Inventory Level: {inventoryLevel} units
- Category: {category}

WEEKLY SALES DATA (last 12 weeks):
{salesData}

COMPETITOR PRICES:
{competitorData}

RECENT PRICE CHANGES:
{priceChanges}

RULES:
1. Never recommend a price below cost (maintain positive margin)
2. Consider sales velocity trends — declining sales may indicate overpricing
3. If competitors are consistently cheaper, consider a modest reduction
4. If you are already the cheapest and sales are strong, consider a modest increase
5. Factor in inventory levels — low stock suggests price can increase
6. Be conservative: recommend changes of 3-15% unless data strongly supports more
7. Express confidence from 0-1 based on data quality and signal strength

Return ONLY valid JSON:
{
  "recommendedPrice": 0.00,
  "expectedRevenueChangePct": 0.00,
  "expectedMargin": 0.00,
  "confidenceScore": 0.00,
  "reasoning": [
    { "factor": "sales_velocity", "direction": "up|down|neutral", "detail": "..." },
    { "factor": "competitor_pricing", "direction": "up|down|neutral", "detail": "..." },
    { "factor": "margin_analysis", "direction": "up|down|neutral", "detail": "..." },
    { "factor": "inventory", "direction": "up|down|neutral", "detail": "..." }
  ]
}`;

export async function generateRecommendation(
  productId: string,
  orgId: string
): Promise<RecommendationResult> {
  const ctx = await gatherProductContext(productId, orgId);
  const { product, salesData, competitorSnapshots, recentPrices } = ctx;

  const cost = Number(product.cost ?? 0);
  const currentPrice = Number(product.currentPrice);
  const currentMargin = cost > 0 ? ((currentPrice - cost) / currentPrice * 100).toFixed(1) : "unknown";

  const prompt = RECOMMENDATION_PROMPT
    .replace("{productName}", product.name)
    .replace("{currentPrice}", currentPrice.toFixed(2))
    .replace("{cost}", cost > 0 ? cost.toFixed(2) : "unknown")
    .replace("{currentMargin}", String(currentMargin))
    .replace("{inventoryLevel}", String(product.inventoryLevel ?? "unknown"))
    .replace("{category}", product.category ?? "uncategorized")
    .replace("{salesData}", JSON.stringify(salesData, null, 2))
    .replace("{competitorData}", JSON.stringify(competitorSnapshots, null, 2))
    .replace("{priceChanges}", JSON.stringify(
      recentPrices.map((p) => ({
        date: p.effectiveAt.toISOString().split("T")[0],
        price: Number(p.price),
        source: p.source,
        reason: p.reason,
      })),
      null, 2
    ));

  const response = await openai.chat.completions.create({
    model: "gpt-4o",
    messages: [{ role: "user", content: prompt }],
    temperature: 0.2,
    max_tokens: 600,
    response_format: { type: "json_object" },
  });

  const content = response.choices[0]?.message?.content ?? "{}";
  const parsed = JSON.parse(content);

  // Enforce cost floor
  let recPrice = Number(parsed.recommendedPrice);
  if (cost > 0 && recPrice < cost * 1.05) {
    recPrice = Math.ceil(cost * 1.05 * 100) / 100;
  }

  return {
    recommendedPrice: recPrice,
    expectedRevenueChangePct: Number(parsed.expectedRevenueChangePct ?? 0),
    expectedMargin: Number(parsed.expectedMargin ?? 0),
    confidenceScore: Math.max(0, Math.min(1, Number(parsed.confidenceScore ?? 0.5))),
    reasoning: Array.isArray(parsed.reasoning) ? parsed.reasoning : [],
    factors: {
      competitorAvgPrice: competitorSnapshots.length > 0
        ? competitorSnapshots.reduce((s, c) => s + c.currentPrice, 0) / competitorSnapshots.length
        : undefined,
      competitorMinPrice: competitorSnapshots.length > 0
        ? Math.min(...competitorSnapshots.map((c) => c.currentPrice))
        : undefined,
      currentMargin: cost > 0 ? (currentPrice - cost) / currentPrice : undefined,
      inventoryLevel: product.inventoryLevel ?? undefined,
    },
  };
}
```

### 3. Playwright Competitor Price Scraping Worker

```typescript
// src/server/workers/scrape-worker.ts

import { Worker, Job, Queue } from "bullmq";
import { chromium, Browser, Page } from "playwright";
import { db } from "@/server/db";
import {
  competitorProducts, competitorPriceChecks, notificationLog,
} from "@/server/db/schema";
import { eq, and, sql } from "drizzle-orm";
import { redis } from "@/server/queue/connection";

export interface ScrapeJobData {
  competitorProductId: string;
  orgId: string;
  url: string;
  scrapeConfig: {
    priceSelector?: string;
    stockSelector?: string;
    waitForSelector?: string;
    priceRegex?: string;
  };
}

export const scrapeQueue = new Queue<ScrapeJobData>("competitor-scraping", {
  connection: redis,
  defaultJobOptions: {
    attempts: 3,
    backoff: { type: "exponential", delay: 5000 },
    removeOnComplete: { age: 86400 },
    removeOnFail: { age: 604800 },
  },
});

let browser: Browser | null = null;

async function getBrowser(): Promise<Browser> {
  if (!browser || !browser.isConnected()) {
    browser = await chromium.launch({
      headless: true,
      args: [
        "--no-sandbox",
        "--disable-setuid-sandbox",
        "--disable-dev-shm-usage",
        "--disable-gpu",
      ],
    });
  }
  return browser;
}

function extractPrice(text: string, regex?: string): number | null {
  const pattern = regex
    ? new RegExp(regex)
    : /\$?\s*(\d{1,3}(?:,\d{3})*(?:\.\d{2})?)/;

  const match = text.match(pattern);
  if (!match) return null;

  const priceStr = (match[1] ?? match[0]).replace(/[$,\s]/g, "");
  const price = parseFloat(priceStr);
  return isNaN(price) ? null : price;
}

async function scrapeSingleProduct(
  page: Page,
  job: ScrapeJobData
): Promise<{ price: number | null; inStock: boolean }> {
  const { url, scrapeConfig } = job;

  await page.goto(url, { waitUntil: "domcontentloaded", timeout: 30000 });

  if (scrapeConfig.waitForSelector) {
    await page.waitForSelector(scrapeConfig.waitForSelector, { timeout: 10000 })
      .catch(() => { /* element not found, proceed anyway */ });
  }

  // Wait for dynamic content
  await page.waitForTimeout(2000);

  // Extract price
  let price: number | null = null;
  const priceSelector = scrapeConfig.priceSelector ?? '[class*="price"], [data-price], .price';

  const priceElements = await page.$$(priceSelector);
  for (const el of priceElements) {
    const text = await el.textContent();
    if (text) {
      price = extractPrice(text.trim(), scrapeConfig.priceRegex);
      if (price !== null && price > 0 && price < 100000) break;
    }
  }

  // Extract stock status
  let inStock = true;
  if (scrapeConfig.stockSelector) {
    const stockEl = await page.$(scrapeConfig.stockSelector);
    if (stockEl) {
      const stockText = await stockEl.textContent();
      inStock = !/(out of stock|sold out|unavailable)/i.test(stockText ?? "");
    }
  }

  return { price, inStock };
}

const scrapeWorker = new Worker<ScrapeJobData>(
  "competitor-scraping",
  async (job: Job<ScrapeJobData>) => {
    const data = job.data;
    const browserInstance = await getBrowser();
    const context = await browserInstance.newContext({
      userAgent: "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) " +
        "AppleWebKit/537.36 (KHTML, like Gecko) Chrome/120.0.0.0 Safari/537.36",
    });
    const page = await context.newPage();

    try {
      const result = await scrapeSingleProduct(page, data);

      // Get previous price for comparison
      const [cp] = await db.select().from(competitorProducts)
        .where(eq(competitorProducts.id, data.competitorProductId))
        .limit(1);

      const previousPrice = cp?.currentPrice ? Number(cp.currentPrice) : null;

      // Record the check
      await db.insert(competitorPriceChecks).values({
        competitorProductId: data.competitorProductId,
        price: result.price?.toString() ?? null,
        previousPrice: previousPrice?.toString() ?? null,
        inStock: result.inStock,
        success: result.price !== null,
        errorMessage: result.price === null ? "Could not extract price" : null,
        checkedAt: new Date(),
      });

      // Update competitor product current price
      if (result.price !== null) {
        await db.update(competitorProducts).set({
          currentPrice: result.price.toString(),
          inStock: result.inStock,
          lastCheckedAt: new Date(),
          lastSuccessAt: new Date(),
          consecutiveFailures: 0,
        }).where(eq(competitorProducts.id, data.competitorProductId));

        // Alert on significant price change (>5%)
        if (previousPrice && Math.abs(result.price - previousPrice) / previousPrice > 0.05) {
          await db.insert(notificationLog).values({
            orgId: data.orgId,
            type: "competitor_alert",
            channel: "in_app",
            title: `Competitor price changed: ${cp?.name ?? "Product"}`,
            body: `Price moved from $${previousPrice.toFixed(2)} to $${result.price.toFixed(2)}`,
            metadata: {
              competitorProductId: data.competitorProductId,
              priceFrom: previousPrice.toFixed(2),
              priceTo: result.price.toFixed(2),
            },
          });
        }
      } else {
        // Increment failure counter
        await db.update(competitorProducts).set({
          lastCheckedAt: new Date(),
          consecutiveFailures: sql`consecutive_failures + 1`,
          status: sql`CASE WHEN consecutive_failures + 1 >= 5 THEN 'error' ELSE status END`,
        }).where(eq(competitorProducts.id, data.competitorProductId));
      }
    } finally {
      await page.close();
      await context.close();
    }
  },
  {
    connection: redis,
    concurrency: 3,
    limiter: { max: 10, duration: 60000 },
  }
);

// Schedule daily scrape jobs for all active competitor products
export async function scheduleDailyScrapes(): Promise<number> {
  const activeProducts = await db.select().from(competitorProducts)
    .where(eq(competitorProducts.status, "active"));

  let queued = 0;
  for (const cp of activeProducts) {
    await scrapeQueue.add("scrape", {
      competitorProductId: cp.id,
      orgId: cp.orgId,
      url: cp.url,
      scrapeConfig: cp.scrapeConfig ?? {},
    }, {
      jobId: `scrape-${cp.id}-${new Date().toISOString().split("T")[0]}`,
      delay: queued * 5000, // stagger by 5s to avoid rate limiting
    });
    queued++;
  }
  return queued;
}

export { scrapeWorker };
```

### 4. Price History Tracking + Revenue Impact Calculator

```typescript
// src/server/services/price-tracker.ts

import { db } from "@/server/db";
import {
  products, priceHistory, notificationLog,
} from "@/server/db/schema";
import { eq, and, desc, gte, lte, sql, between } from "drizzle-orm";

interface PriceChangeInput {
  productId: string;
  orgId: string;
  newPrice: number;
  source: "shopify_sync" | "manual" | "recommendation" | "rule";
  reason?: string;
  recommendationId?: string;
  userId?: string;
  ruleId?: string;
}

/**
 * Record a price change in the append-only log and update the product's current price.
 * Returns the new price history row ID.
 */
export async function recordPriceChange(input: PriceChangeInput): Promise<string> {
  const [product] = await db.select().from(products)
    .where(and(eq(products.id, input.productId), eq(products.orgId, input.orgId)))
    .limit(1);

  if (!product) throw new Error("Product not found");

  const previousPrice = Number(product.currentPrice);
  const newPrice = input.newPrice;

  // Skip if price hasn't changed
  if (Math.abs(previousPrice - newPrice) < 0.01) {
    return "";
  }

  // Insert into append-only price history log
  const [entry] = await db.insert(priceHistory).values({
    orgId: input.orgId,
    productId: input.productId,
    price: newPrice.toString(),
    previousPrice: previousPrice.toString(),
    source: input.source,
    reason: input.reason,
    recommendationId: input.recommendationId ?? null,
    metadata: {
      userId: input.userId,
      ruleId: input.ruleId,
    },
    effectiveAt: new Date(),
  }).returning();

  // Update current price on the product
  await db.update(products).set({
    currentPrice: newPrice.toString(),
    updatedAt: new Date(),
  }).where(eq(products.id, input.productId));

  return entry.id;
}

// --- Revenue Impact Calculator ---

interface RevenueImpactResult {
  productId: string;
  productName: string;
  priceChangeDate: string;
  oldPrice: number;
  newPrice: number;
  changePct: number;
  // 4-week comparison windows
  revenueBeforeChange: number;
  revenueAfterChange: number;
  revenueChangePct: number;
  unitsBeforeChange: number;
  unitsAfterChange: number;
  unitsChangePct: number;
}

/**
 * Calculate the revenue impact of a specific price change by comparing
 * 4-week windows before and after the change.
 */
export async function calculateRevenueImpact(
  priceHistoryId: string
): Promise<RevenueImpactResult | null> {
  const [entry] = await db.select().from(priceHistory)
    .where(eq(priceHistory.id, priceHistoryId))
    .limit(1);

  if (!entry) return null;

  const [product] = await db.select().from(products)
    .where(eq(products.id, entry.productId))
    .limit(1);

  if (!product) return null;

  const changeDate = entry.effectiveAt;
  const fourWeeksBefore = new Date(changeDate.getTime() - 28 * 86400000);
  const fourWeeksAfter = new Date(changeDate.getTime() + 28 * 86400000);

  // Revenue from order data (simplified — in production this queries an orders table)
  // For MVP we approximate from price history frequency
  const [beforeStats] = await db.execute<{
    total_revenue: string;
    order_count: string;
  }>(sql`
    SELECT
      COALESCE(SUM(ph.price::numeric), 0) AS total_revenue,
      COUNT(*) AS order_count
    FROM price_history ph
    WHERE ph.product_id = ${entry.productId}
      AND ph.source = 'shopify_sync'
      AND ph.effective_at >= ${fourWeeksBefore.toISOString()}
      AND ph.effective_at < ${changeDate.toISOString()}
  `);

  const [afterStats] = await db.execute<{
    total_revenue: string;
    order_count: string;
  }>(sql`
    SELECT
      COALESCE(SUM(ph.price::numeric), 0) AS total_revenue,
      COUNT(*) AS order_count
    FROM price_history ph
    WHERE ph.product_id = ${entry.productId}
      AND ph.source = 'shopify_sync'
      AND ph.effective_at >= ${changeDate.toISOString()}
      AND ph.effective_at < ${fourWeeksAfter.toISOString()}
  `);

  const revBefore = Number(beforeStats?.total_revenue ?? 0);
  const revAfter = Number(afterStats?.total_revenue ?? 0);
  const unitsBefore = Number(beforeStats?.order_count ?? 0);
  const unitsAfter = Number(afterStats?.order_count ?? 0);

  return {
    productId: entry.productId,
    productName: product.name,
    priceChangeDate: changeDate.toISOString().split("T")[0],
    oldPrice: Number(entry.previousPrice ?? 0),
    newPrice: Number(entry.price),
    changePct: Number(entry.previousPrice) > 0
      ? (Number(entry.price) - Number(entry.previousPrice)) / Number(entry.previousPrice)
      : 0,
    revenueBeforeChange: revBefore,
    revenueAfterChange: revAfter,
    revenueChangePct: revBefore > 0 ? (revAfter - revBefore) / revBefore : 0,
    unitsBeforeChange: unitsBefore,
    unitsAfterChange: unitsAfter,
    unitsChangePct: unitsBefore > 0 ? (unitsAfter - unitsBefore) / unitsBefore : 0,
  };
}

/**
 * Build price history timeline for charting — returns daily price points
 * with revenue overlay data for a given product.
 */
export async function getPriceTimeline(
  productId: string,
  orgId: string,
  days: number = 90
): Promise<{
  date: string;
  price: number;
  source: string;
  dailyRevenue: number;
  dailyUnits: number;
}[]> {
  const since = new Date(Date.now() - days * 86400000);

  const timeline = await db.execute(sql`
    WITH daily_prices AS (
      SELECT DISTINCT ON (date_trunc('day', effective_at))
        date_trunc('day', effective_at) AS day,
        price::numeric AS price,
        source
      FROM price_history
      WHERE product_id = ${productId}
        AND org_id = ${orgId}
        AND effective_at >= ${since.toISOString()}
      ORDER BY date_trunc('day', effective_at), effective_at DESC
    ),
    daily_orders AS (
      SELECT
        date_trunc('day', effective_at) AS day,
        SUM(price::numeric) AS revenue,
        COUNT(*) AS units
      FROM price_history
      WHERE product_id = ${productId}
        AND source = 'shopify_sync'
        AND effective_at >= ${since.toISOString()}
      GROUP BY date_trunc('day', effective_at)
    )
    SELECT
      dp.day::text AS date,
      dp.price,
      dp.source,
      COALESCE(do2.revenue, 0) AS daily_revenue,
      COALESCE(do2.units, 0) AS daily_units
    FROM daily_prices dp
    LEFT JOIN daily_orders do2 ON dp.day = do2.day
    ORDER BY dp.day ASC
  `);

  return (timeline as any[]).map((r) => ({
    date: r.date,
    price: Number(r.price),
    source: r.source,
    dailyRevenue: Number(r.daily_revenue),
    dailyUnits: Number(r.daily_units),
  }));
}
```

---

## Phase Breakdown (8.5 weeks) — Day-by-Day

### Phase 1: Scaffold + Database (Days 1-5)

**Day 1 — Project scaffold + environment**
- `npx create-next-app@latest pricewise --typescript --tailwind --app --src-dir`
- Install dependencies: `drizzle-orm @neondatabase/serverless @clerk/nextjs @trpc/server @trpc/client @trpc/next superjson zod openai stripe bullmq ioredis`
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
    OPENAI_API_KEY: z.string().min(1),
    UPSTASH_REDIS_URL: z.string().url(),
    UPSTASH_REDIS_TOKEN: z.string().min(1),
    SHOPIFY_CLIENT_ID: z.string().min(1),
    SHOPIFY_CLIENT_SECRET: z.string().min(1),
    INTEGRATION_ENCRYPTION_KEY: z.string().length(64),
    NEXT_PUBLIC_APP_URL: z.string().url(),
  });
  ```
- Create Neon database, enable pg_trgm extension
- Configure `drizzle.config.ts` with Neon connection string

**Day 2 — Database schema + Clerk auth**
- Write full Drizzle schema (as above) in `src/server/db/schema.ts`
- Create `src/server/db/index.ts` with Neon serverless driver
- Run `drizzle-kit push` to apply schema
- Run migration SQL for trigram + partial indexes
- Set up Clerk organization: `src/middleware.ts` with `clerkMiddleware()`, protect `/dashboard/*` routes
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
    starter: { maxProducts: 100, maxCompetitors: 5, maxRules: 0, maxUsers: 1 },
    growth: { maxProducts: 1000, maxCompetitors: 20, maxRules: Infinity, maxUsers: 3 },
  } as const;
  ```
- Create `src/server/trpc/routers/_app.ts`: merge routers

**Day 4 — Organization + DataSource tRPC routers**
- `src/server/trpc/routers/organization.ts`:
  - `getOrCreate`: find by clerkOrgId or create new
  - `update`: name, settings
  - `getSettings`: return current org settings
  - `updateSettings`: validate and persist settings JSONB
- `src/server/trpc/routers/dataSource.ts`:
  - `list`: all sources for org
  - `create`: type + name + config
  - `update`: config, status
  - `delete`: cascade cleanup
  - `triggerSync`: manually trigger product/order sync
- Create `src/server/integrations/encryption.ts`:
  ```typescript
  import crypto from "crypto";

  const ALGORITHM = "aes-256-gcm";
  const IV_LENGTH = 16;

  export function encrypt(plaintext: string): string {
    const key = Buffer.from(process.env.INTEGRATION_ENCRYPTION_KEY!, "hex");
    const iv = crypto.randomBytes(IV_LENGTH);
    const cipher = crypto.createCipheriv(ALGORITHM, key, iv);
    let encrypted = cipher.update(plaintext, "utf8", "hex");
    encrypted += cipher.final("hex");
    const authTag = cipher.getAuthTag();
    return iv.toString("hex") + ":" + authTag.toString("hex") + ":" + encrypted;
  }

  export function decrypt(ciphertext: string): string {
    const key = Buffer.from(process.env.INTEGRATION_ENCRYPTION_KEY!, "hex");
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
- Install UI deps: `@radix-ui/react-dialog @radix-ui/react-dropdown-menu @radix-ui/react-tabs @radix-ui/react-select @radix-ui/react-tooltip @radix-ui/react-popover class-variance-authority clsx tailwind-merge lucide-react recharts`
- Create shared UI components in `src/components/ui/`: button, card, badge, input, select, dialog, dropdown-menu, tabs, toast, skeleton
- Build app shell layout: `src/app/dashboard/layout.tsx`
  - Left sidebar: logo, nav links (Dashboard, Products, Competitors, Recommendations, Rules, Settings), org switcher, user menu
  - Main content area with header breadcrumb
- Create stub pages:
  - `src/app/dashboard/page.tsx` (overview)
  - `src/app/dashboard/products/page.tsx`
  - `src/app/dashboard/competitors/page.tsx`
  - `src/app/dashboard/recommendations/page.tsx`
  - `src/app/dashboard/rules/page.tsx`
  - `src/app/dashboard/settings/page.tsx`
  - `src/app/dashboard/settings/billing/page.tsx`

---

### Phase 2: Shopify Integration (Days 6-12)

**Day 6 — Shopify OAuth flow**
- Create `src/app/api/auth/shopify/route.ts`:
  - GET: validate shop param, generate state, store in Redis, redirect to Shopify OAuth
- Create `src/app/api/auth/shopify/callback/route.ts`:
  - GET: verify HMAC, verify state, exchange code for access token, store encrypted in `data_sources`
- Use the `buildInstallUrl` and `exchangeCodeForToken` functions from the architecture deep-dive

**Day 7 — Shopify product sync**
- Create `src/server/integrations/shopify/sync-products.ts`:
  - Paginate through `/admin/api/2024-10/products.json`
  - For each product variant: upsert into `products` table with `externalId` + `variantId`
  - Record initial price in `priceHistory` with source `shopify_sync`
  - Handle product images, categories (product_type), tags
- Create `src/server/integrations/shopify/helpers.ts`:
  - `upsertProductFromShopify()`: find-or-create by externalId + variantId
  - `parseLinkHeader()`: extract pagination cursor from Link header

**Day 8 — Shopify order history import**
- Create `src/server/integrations/shopify/sync-orders.ts`:
  - Paginate through `/admin/api/2024-10/orders.json?status=any&created_at_min={12_months_ago}`
  - For each line item: match to local product by variant_id
  - Store order data for sales velocity calculations (use a lightweight `orders` tracking approach via the priceHistory log or a separate summary table)
- Create `src/server/integrations/shopify/sync-inventory.ts`:
  - Fetch inventory levels via `/admin/api/2024-10/inventory_levels.json`
  - Update `products.inventoryLevel` for each item

**Day 9 — Shopify webhook receivers**
- Create `src/app/api/webhooks/shopify/route.ts`:
  ```typescript
  export async function POST(req: NextRequest) {
    const payload = await req.text();
    const hmac = req.headers.get("x-shopify-hmac-sha256");
    const topic = req.headers.get("x-shopify-topic");
    const shop = req.headers.get("x-shopify-shop-domain");

    if (!hmac || !verifyWebhookHmac(payload, hmac)) {
      return NextResponse.json({ error: "Invalid HMAC" }, { status: 401 });
    }

    const body = JSON.parse(payload);

    switch (topic) {
      case "products/update":
      case "products/create":
        await handleProductWebhook(shop!, body);
        break;
      case "orders/create":
        await handleOrderWebhook(shop!, body);
        break;
      case "inventory_levels/update":
        await handleInventoryWebhook(shop!, body);
        break;
    }

    return NextResponse.json({ ok: true });
  }
  ```
- Implement `handleProductWebhook`: upsert product, record price change if price differs
- Implement `handleOrderWebhook`: update sales tracking data
- Implement `handleInventoryWebhook`: update `products.inventoryLevel`

**Day 10 — Register Shopify webhooks programmatically**
- Create `src/server/integrations/shopify/register-webhooks.ts`:
  - After OAuth, register webhooks for: `products/create`, `products/update`, `orders/create`, `inventory_levels/update`, `app/uninstalled`
  - Use `POST /admin/api/2024-10/webhooks.json`
  - Handle `app/uninstalled`: mark data source as paused, clear access token

**Day 11 — BullMQ queue setup for sync jobs**
- Create `src/server/queue/connection.ts`:
  ```typescript
  import { Redis } from "ioredis";
  export const redis = new Redis(process.env.UPSTASH_REDIS_URL!, {
    maxRetriesPerRequest: null,
  });
  ```
- Create `src/server/queue/sync-queue.ts`:
  - Define `shopify-sync` queue for full product/order sync jobs
  - Default: retry 3x with exponential backoff
- Create `src/server/queue/sync-worker.ts`:
  - Worker that processes full sync jobs (called on initial connection + manual trigger)
  - Steps: sync products, sync orders (last 12 months), sync inventory

**Day 12 — Product tRPC router + data source settings page**
- `src/server/trpc/routers/product.ts`:
  - `list`: paginated product list with filters (category, price range, status), sortable by name/price/margin/inventory
  - `getById`: full product detail with recent price history, competitor prices, latest recommendation
  - `updatePrice`: manual price change (records in priceHistory, pushes to Shopify via Admin API)
  - `togglePriceLock`: enable/disable manual override lock
- Create `src/app/dashboard/settings/connections/page.tsx`:
  - "Connect Shopify" button that triggers OAuth flow
  - Connected store card: store name, last synced, product count, sync status
  - "Re-sync" button to trigger manual full sync
  - "Disconnect" button

---

### Phase 3: Competitor Monitoring (Days 13-18)

**Day 13 — Competitor + CompetitorProduct tRPC routers**
- `src/server/trpc/routers/competitor.ts`:
  - `list`: all competitors for org with product count
  - `create`: name + domain
  - `update`: name, domain, notes
  - `delete`: cascade to competitorProducts
- `src/server/trpc/routers/competitorProduct.ts`:
  - `list`: competitor products for a given product or competitor
  - `create`: url + competitorId + productId + optional scrapeConfig
  - `update`: url, scrapeConfig, status
  - `delete`: remove tracking
  - `triggerScrape`: manually trigger a single scrape job
  - `getPriceHistory`: return competitorPriceChecks for charting

**Day 14 — Playwright scraping worker**
- Create `src/server/workers/scrape-worker.ts` (as in architecture deep-dive)
- Install Playwright: `npm install playwright @playwright/test` on the Railway worker
- Test with sample competitor URLs: Amazon product page, Shopify store product page
- Build price extraction heuristics for common e-commerce platforms

**Day 15 — Scheduled scraping + scrape configuration**
- Create `src/server/queue/scrape-scheduler.ts`:
  - Repeatable BullMQ job that runs every 24 hours (configurable via org settings)
  - Calls `scheduleDailyScrapes()` which enqueues individual scrape jobs for all active competitorProducts
- Create `src/components/competitors/scrape-config-form.tsx`:
  - URL input with "Test Scrape" button (runs scrape immediately, shows extracted price)
  - CSS selector override fields (price, stock, wait-for)
  - Price regex override field
  - Preview of last successful scrape result

**Day 16 — Competitor dashboard UI**
- Create `src/app/dashboard/competitors/page.tsx`:
  - Competitor list with expand/collapse for tracked products
  - For each competitor: name, domain, product count, last checked
  - "Add Competitor" dialog
  - Per competitor product: URL, your price vs. their price, price delta, last checked, status
- Create `src/components/competitors/add-tracking-dialog.tsx`:
  - Select product from your catalog (search)
  - Select or create competitor
  - Enter competitor product URL
  - Optional: configure scrape selectors
  - "Test & Save" button

**Day 17 — Competitor comparison view**
- Create `src/app/dashboard/products/[id]/competitors/page.tsx`:
  - For a single product, show all tracked competitor prices
  - Price comparison chart (Recharts BarChart): your price vs. each competitor
  - Historical price chart (Recharts LineChart): your price + competitor prices over time
  - Price position indicator: "You are the 2nd cheapest of 4 sellers"
- Create `src/components/competitors/price-comparison-chart.tsx`
- Create `src/components/competitors/competitor-history-chart.tsx`

**Day 18 — Competitor alerts + notification system**
- Implement price change alert logic in scrape worker (>5% change triggers notification)
- Create `src/server/trpc/routers/notification.ts`:
  - `list`: paginated in-app notifications for org, filterable by type, read/unread
  - `markRead`: mark single notification as read
  - `markAllRead`: mark all as read
  - `getUnreadCount`: for notification bell badge
- Create `src/components/layout/notification-bell.tsx`:
  - Bell icon in header with unread count badge
  - Dropdown panel showing recent notifications
  - Click notification to navigate to relevant product/competitor

---

### Phase 4: AI Recommendations (Days 19-24)

**Day 19 — AI recommendation engine**
- Create `src/server/ai/recommend-price.ts` (as in architecture deep-dive)
- Implement `gatherProductContext`: fetches sales data, competitor prices, price history
- Implement GPT-4o prompt construction with structured context payload
- Implement response parsing with validation and cost floor enforcement

**Day 20 — Batch recommendation generation**
- Create `src/server/queue/recommendation-queue.ts`:
  - `recommendation-batch` queue for processing all products in an org
  - Each job processes a single product recommendation
- Create `src/server/queue/recommendation-worker.ts`:
  - Worker processes one product at a time: gather context, call GPT-4o, store recommendation
  - Rate limited: max 5 concurrent, respects OpenAI rate limits
  - On completion: create `recommendation_ready` notification
- Create `src/server/services/batch-recommend.ts`:
  - `triggerBatchRecommendations(orgId)`: find all active, non-locked products with sufficient data, enqueue recommendation job for each
  - Expire old pending recommendations (>7 days old) before generating new ones
  - Scheduled via BullMQ repeatable job: runs daily at 6am UTC

**Day 21 — Recommendation tRPC router**
- `src/server/trpc/routers/recommendation.ts`:
  - `list`: paginated recommendations for org, filterable by status (pending/accepted/rejected), sortable by expectedRevenueChangePct or confidenceScore
  - `getByProduct`: latest recommendation for a product
  - `accept`: mark as accepted, record price change in priceHistory, update product.currentPrice, push to Shopify via Admin API
  - `reject`: mark as rejected with optional reason
  - `acceptWithModification`: accept but set a different price (stores in appliedPrice)
  - `getPendingCount`: for dashboard badge
  - `getStats`: total accepted/rejected/expired, average revenue impact of accepted recommendations

**Day 22 — Accept/reject workflow + Shopify push-back**
- Create `src/server/integrations/shopify/update-price.ts`:
  ```typescript
  export async function pushPriceToShopify(
    dataSourceId: string,
    variantId: string,
    newPrice: number
  ): Promise<boolean> {
    const [source] = await db.select().from(dataSources)
      .where(eq(dataSources.id, dataSourceId)).limit(1);

    if (!source?.config?.shopifyAccessToken) return false;

    const shop = source.config.shopifyShop!;
    const token = decrypt(source.config.shopifyAccessToken);

    const res = await fetch(
      `https://${shop}/admin/api/2024-10/variants/${variantId}.json`,
      {
        method: "PUT",
        headers: {
          "X-Shopify-Access-Token": token,
          "Content-Type": "application/json",
        },
        body: JSON.stringify({ variant: { price: newPrice.toFixed(2) } }),
      }
    );

    return res.ok;
  }
  ```
- Wire `accept` mutation: recordPriceChange() + pushPriceToShopify() + create notification

**Day 23 — Recommendation detail + reasoning display**
- Create `src/components/recommendations/recommendation-card.tsx`:
  - Current price vs. recommended price with arrow indicator
  - Expected revenue change percentage badge (green for positive, red for negative)
  - Confidence score indicator (progress bar)
  - Reasoning list: each factor with direction icon (up/down/neutral) and detail text
  - Accept / Reject / Modify buttons
- Create `src/components/recommendations/modify-price-dialog.tsx`:
  - Shows recommended price as default
  - User can adjust the price
  - Shows real-time margin calculation
  - Confirm button

**Day 24 — Recommendations page + product integration**
- Create `src/app/dashboard/recommendations/page.tsx`:
  - Tab bar: Pending | Accepted | Rejected
  - List of recommendation cards with product info
  - Bulk accept button (apply top N recommendations)
  - Summary stats: total pending, average expected revenue lift, highest confidence
- Wire recommendations into product detail page:
  - `src/app/dashboard/products/[id]/page.tsx`: show latest recommendation in sidebar
  - Quick accept/reject from product view

---

### Phase 5: Dashboard + Product Views (Days 25-30)

**Day 25 — Product list page**
- Create `src/app/dashboard/products/page.tsx`:
  - Table with columns: Product Name, SKU, Current Price, Cost, Margin %, Inventory, Competitor Range, Recommendation, Status
  - Margin column: color-coded (red < 15%, yellow 15-30%, green > 30%)
  - Competitor Range: shows min-max of tracked competitor prices
  - Recommendation column: shows recommended price delta if pending recommendation exists
  - Sortable columns, search by name/SKU
  - Filter by: category, status, has recommendation, price-locked
  - Pagination

**Day 26 — Product detail page**
- Create `src/app/dashboard/products/[id]/page.tsx`:
  - Header: product name, image, SKU, status badge, lock toggle
  - Stats row: Current Price, Cost, Margin %, Inventory Level
  - Tabs: Price History | Competitors | Recommendations
- Create `src/components/products/price-history-chart.tsx`:
  - Recharts ComposedChart: price line + revenue area overlay
  - Price change annotations (vertical dashed lines with source labels)
  - Time range selector: 30d / 90d / 1y
- Create `src/components/products/product-stats-cards.tsx`

**Day 27 — Product detail tabs**
- Price History tab:
  - Chart (from Day 26)
  - Table of all price changes: date, old price, new price, source, reason
  - Revenue impact badge for each change (if enough data)
- Competitors tab:
  - Competitor price comparison bar chart
  - Historical competitor price overlay on your price chart
  - Table: competitor name, their price, delta from your price, last checked
  - "Add competitor tracking" button
- Recommendations tab:
  - Latest recommendation card with full reasoning
  - History of past recommendations with outcomes (accepted/rejected/expired)

**Day 28 — Dashboard overview page**
- Create `src/app/dashboard/page.tsx`:
  - Welcome header with org name
  - Quick stats row: Total Products, Pending Recommendations, Competitor Alerts (today), Avg Margin %
  - Revenue trend chart (Recharts AreaChart, last 30 days)
  - "Top Opportunities" section: products with highest expected revenue lift from pending recommendations
  - "Recent Competitor Changes" section: latest competitor price movements
  - "Recent Price Changes" section: your recent price updates with source attribution
  - Quick action buttons: "Generate Recommendations", "Run Competitor Scan"

**Day 29 — Analytics charts**
- Create `src/server/trpc/routers/analytics.ts`:
  - `revenueTrend`: daily revenue from orders over last 30/90 days
  - `marginDistribution`: histogram of margin percentages across products
  - `recommendationStats`: acceptance rate, average revenue impact of accepted recommendations
  - `competitorPositioning`: how many products you are cheapest/mid/most expensive
  - `priceChangeLog`: recent price changes with source breakdown
  - `topOpportunities`: products with highest pending revenue lift
- Create `src/components/analytics/revenue-trend-chart.tsx`
- Create `src/components/analytics/margin-distribution-chart.tsx`
- Create `src/components/analytics/competitor-position-chart.tsx`

**Day 30 — Analytics page assembly**
- Create `src/app/dashboard/analytics/page.tsx`:
  - Revenue trend area chart with price change annotations
  - Recommendation performance: acceptance rate donut + average revenue impact bar
  - Margin distribution histogram
  - Competitor positioning: stacked bar (cheapest / mid / expensive)
  - Price change activity: timeline of recent changes by source
  - Export data button (CSV)

---

### Phase 6: Pricing Rules (Days 31-34)

**Day 31 — Pricing rules tRPC router**
- `src/server/trpc/routers/pricingRule.ts`:
  - `list`: all rules for org, ordered by priority
  - `create`: name, type, conditions, appliesTo, priority
  - `update`: conditions, appliesTo, isActive, priority
  - `delete`: remove rule
  - `preview`: given a rule, return which products would be affected and what their new prices would be (dry run)
  - `reorder`: update priority values for drag-and-drop ordering
- Plan enforcement: rules are Growth-plan only

**Day 32 — Rule engine evaluation**
- Create `src/server/services/rule-engine.ts`:
  ```typescript
  export async function evaluateRulesForProduct(
    productId: string,
    orgId: string
  ): Promise<{ ruleId: string; newPrice: number; reason: string } | null> {
    // Fetch active rules ordered by priority
    // For each rule, check if product matches appliesTo
    // Evaluate conditions against current product state
    // Return first matching rule's action (highest priority wins)
    // Return null if no rules match
  }
  ```
- Implement condition evaluators for each rule type:
  - `competitor_based`: compare current price to competitor's price, apply offset
  - `inventory_based`: check inventory level thresholds, apply percentage adjustment
  - `margin_floor`: ensure price maintains minimum margin above cost

**Day 33 — Rule builder UI**
- Create `src/app/dashboard/rules/page.tsx`:
  - Rule list with priority ordering (drag to reorder)
  - Each rule card: name, type badge, conditions summary, applies to summary, active toggle
  - "Create Rule" button
- Create `src/components/rules/rule-form.tsx`:
  - Rule type selector (competitor-based, inventory-based, margin floor)
  - Dynamic condition fields based on type
  - "Applies to" selector: specific products, categories, or all products
  - Priority field
  - "Preview" button: shows affected products with simulated price changes
- Create `src/components/rules/rule-preview.tsx`:
  - Table of products that would be affected: product name, current price, new price, change %

**Day 34 — Rule execution + audit log**
- Create `src/server/queue/rule-evaluation-queue.ts`:
  - Scheduled job: evaluate all active rules daily (after competitor scraping completes)
  - For each matched product: record price change, push to Shopify, create notification
- Wire rule-triggered price changes into `priceHistory` with `source = 'rule'` and `metadata.ruleId`
- Create `src/components/rules/rule-audit-log.tsx`:
  - Shows history of rule-triggered price changes: date, product, old price, new price, rule name
  - Accessible from each rule's detail view

---

### Phase 7: Billing + Settings (Days 35-38)

**Day 35 — Stripe billing setup**
- Create `src/server/billing/stripe.ts`:
  ```typescript
  import Stripe from "stripe";

  const stripe = new Stripe(process.env.STRIPE_SECRET_KEY!, {
    apiVersion: "2025-04-30.basil",
  });

  export const PLANS = {
    starter: {
      name: "Starter",
      price: 49,
      maxProducts: 100,
      maxCompetitors: 5,
      maxRules: 0,
      maxUsers: 1,
      features: [
        "100 products",
        "5 competitor trackers",
        "AI price recommendations",
        "Price history & analytics",
        "Shopify integration",
        "Daily competitor monitoring",
      ],
    },
    growth: {
      name: "Growth",
      price: 149,
      maxProducts: 1000,
      maxCompetitors: 20,
      maxRules: Infinity,
      maxUsers: 3,
      features: [
        "1,000 products",
        "20 competitor trackers",
        "Dynamic pricing rules",
        "Revenue impact analysis",
        "Bulk recommendation actions",
        "Priority support",
        "3 team members",
      ],
    },
  } as const;

  export async function createCheckoutSession(opts: {
    orgId: string;
    clerkOrgId: string;
    stripeCustomerId?: string;
    plan: "starter" | "growth";
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
    if (opts.stripeCustomerId) sessionParams.customer = opts.stripeCustomerId;
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

**Day 36 — Stripe webhook handler + billing tRPC router**
- Create `src/app/api/webhooks/stripe/route.ts`:
  - `checkout.session.completed`: update org plan + stripeCustomerId + stripeSubscriptionId
  - `customer.subscription.updated`: update org plan from subscription metadata
  - `customer.subscription.deleted`: revert org plan to starter
- `src/server/trpc/routers/billing.ts`:
  - `getCurrentPlan`: return org plan + limits + usage (products count, competitors count)
  - `createCheckout`: generate Stripe checkout URL for plan upgrade
  - `createPortal`: generate Stripe portal URL for managing subscription

**Day 37 — Billing UI + settings pages**
- Create `src/app/dashboard/settings/billing/page.tsx`:
  - Current plan card with usage meters (products used / limit, competitors used / limit)
  - Two plan cards: Starter, Growth with feature lists
  - "Upgrade" or "Manage Subscription" buttons
  - Success/canceled URL parameter handling with toast messages
- Create `src/app/dashboard/settings/page.tsx`:
  - Organization name and slug (editable)
  - Default currency selector
  - Default margin target input
  - Price ending rule selector (none, .99, .95, .00)
  - Max price change percentage guardrail
  - Notification email
  - Scrape frequency (hours) for Growth plan

**Day 38 — Plan enforcement + onboarding**
- Implement plan limit enforcement middleware in tRPC:
  - Product count check on `product.create` (from sync or manual)
  - Competitor count check on `competitor.create`
  - Rules blocked entirely on Starter plan
- Add usage banner in dashboard when approaching limits (>80%)
- Create `src/app/onboarding/page.tsx`:
  - Step 1: Name your organization
  - Step 2: Connect Shopify store (OAuth flow)
  - Step 3: Wait for initial sync to complete (progress indicator)
  - Step 4: "Your store is ready! Explore your products."
  - Skip option to go straight to dashboard
- Store onboarding completion in `organizations.settings.onboardingCompleted`

---

### Phase 8: Polish + Launch (Days 39-43)

**Day 39 — Loading states, error handling, empty states**
- Add Suspense boundaries with skeleton loaders for all data-fetching pages
- Create empty state illustrations:
  - Products empty: "Connect your Shopify store to import products"
  - Competitors empty: "Add competitor URLs to start monitoring prices"
  - Recommendations empty: "Recommendations will appear after your first competitor scan"
  - Rules empty: "Create pricing rules to automate price adjustments" (Growth plan CTA for Starter)
- Toast notifications for all mutations (success/error)
- Global error boundary with retry
- 404 page for invalid routes

**Day 40 — Responsive design + accessibility**
- Mobile-responsive sidebar: collapsible on small screens
- Product table: horizontal scroll on mobile
- All interactive elements: focus rings, keyboard navigation, ARIA labels
- Color contrast verification for all badges and status indicators
- Screen reader text for icon-only buttons

**Day 41 — Performance optimization**
- Add `loading.tsx` files for route-level streaming
- Optimize database queries: ensure all list queries use indexed columns
- React Query cache configuration:
  - Product list: staleTime 30s
  - Product detail: staleTime 15s
  - Competitor prices: staleTime 5min
  - Recommendations: staleTime 30s
  - Analytics: staleTime 5min
- Server-side rendering for dashboard pages using tRPC server caller
- Vercel Analytics + Sentry integration

**Day 42 — Landing page**
- Create `src/app/page.tsx`:
  - Hero: "Optimize Your Prices with AI" with CTA
  - How it works: 3-step visual (Connect Store -> Monitor Competitors -> Get AI Recommendations)
  - Features grid: Shopify Sync, Competitor Monitoring, AI Recommendations, Price History, Pricing Rules
  - Pricing table with two tiers
  - FAQ section
  - Footer with links
- Create `src/app/pricing/page.tsx`: detailed pricing comparison table

**Day 43 — Final testing + deployment**
- End-to-end test flow:
  1. Sign up with Clerk -> create org -> onboarding
  2. Connect Shopify store (test store) -> verify products imported
  3. Add competitor with product URL -> trigger manual scrape -> verify price extracted
  4. Trigger AI recommendations -> verify recommendation generated with reasoning
  5. Accept recommendation -> verify price updated in DB + pushed to Shopify
  6. Verify price history chart shows the change
  7. Create pricing rule (Growth plan) -> verify preview shows correct products
  8. Upgrade plan via Stripe (test mode) -> verify plan limits updated
  9. Verify analytics dashboard populates with real data
- Set up Vercel project + environment variables
- Set up Railway project for scraping worker + environment variables
- Configure custom domain
- Set up Sentry for error tracking
- Deploy to production
- Stripe webhook endpoint registered in Stripe dashboard
- Shopify OAuth redirect URIs updated to production URLs

---

## Critical Files

```
pricewise/
├── src/
│   ├── env.ts                                        # Zod-validated environment variables
│   ├── middleware.ts                                  # Clerk auth middleware, route protection
│   │
│   ├── app/
│   │   ├── layout.tsx                                # Root layout with ClerkProvider
│   │   ├── page.tsx                                  # Landing page
│   │   ├── pricing/page.tsx                          # Pricing comparison page
│   │   ├── onboarding/page.tsx                       # 4-step onboarding wizard
│   │   │
│   │   ├── dashboard/
│   │   │   ├── layout.tsx                            # App shell: sidebar + header + notification bell
│   │   │   ├── page.tsx                              # Dashboard overview (stats, opportunities, alerts)
│   │   │   ├── products/
│   │   │   │   ├── page.tsx                          # Product list table with filters
│   │   │   │   └── [id]/
│   │   │   │       ├── page.tsx                      # Product detail (price history, competitors, recs)
│   │   │   │       └── competitors/page.tsx          # Per-product competitor comparison view
│   │   │   ├── competitors/
│   │   │   │   └── page.tsx                          # Competitor list + tracked product URLs
│   │   │   ├── recommendations/
│   │   │   │   └── page.tsx                          # Pending/accepted/rejected recommendations
│   │   │   ├── rules/
│   │   │   │   └── page.tsx                          # Pricing rules (Growth plan only)
│   │   │   ├── analytics/
│   │   │   │   └── page.tsx                          # Revenue trends, margin distribution, positioning
│   │   │   └── settings/
│   │   │       ├── page.tsx                          # Organization settings + defaults
│   │   │       ├── connections/page.tsx              # Shopify store connection management
│   │   │       ├── billing/page.tsx                  # Plan + usage + upgrade
│   │   │       └── members/page.tsx                  # Team member management (Growth plan)
│   │   │
│   │   └── api/
│   │       ├── trpc/[trpc]/route.ts                  # tRPC HTTP handler
│   │       ├── auth/
│   │       │   ├── shopify/route.ts                  # Shopify OAuth start
│   │       │   └── shopify/callback/route.ts         # Shopify OAuth callback + token exchange
│   │       └── webhooks/
│   │           ├── shopify/route.ts                  # Shopify webhook receiver (products, orders, inventory)
│   │           └── stripe/route.ts                   # Stripe webhook handler (subscriptions)
│   │
│   ├── server/
│   │   ├── db/
│   │   │   ├── schema.ts                            # Drizzle schema (12 tables + relations)
│   │   │   ├── index.ts                             # Neon database client
│   │   │   └── migrations/
│   │   │       └── 0000_init.sql                    # pg_trgm extension + custom indexes
│   │   │
│   │   ├── trpc/
│   │   │   ├── trpc.ts                              # tRPC init, auth/org/plan middleware
│   │   │   ├── context.ts                           # Request context (auth + db)
│   │   │   └── routers/
│   │   │       ├── _app.ts                          # Merged app router
│   │   │       ├── organization.ts                  # Org CRUD + settings
│   │   │       ├── dataSource.ts                    # Shopify connection CRUD + sync trigger
│   │   │       ├── product.ts                       # Product list + detail + manual price update
│   │   │       ├── competitor.ts                    # Competitor CRUD
│   │   │       ├── competitorProduct.ts             # Competitor product tracking + scrape trigger
│   │   │       ├── recommendation.ts               # Accept/reject/modify workflow + stats
│   │   │       ├── pricingRule.ts                   # Rule CRUD + preview + reorder
│   │   │       ├── notification.ts                  # In-app notification list + read status
│   │   │       ├── analytics.ts                     # Dashboard data queries
│   │   │       └── billing.ts                       # Plan info + Stripe checkout/portal
│   │   │
│   │   ├── ai/
│   │   │   └── recommend-price.ts                   # GPT-4o recommendation engine + context gathering
│   │   │
│   │   ├── queue/
│   │   │   ├── connection.ts                        # Redis/ioredis connection
│   │   │   ├── sync-queue.ts                        # BullMQ queue for Shopify sync jobs
│   │   │   ├── sync-worker.ts                       # Worker: full product/order/inventory sync
│   │   │   ├── scrape-scheduler.ts                  # Daily competitor scrape scheduling
│   │   │   ├── recommendation-queue.ts              # BullMQ queue for batch recommendations
│   │   │   ├── recommendation-worker.ts             # Worker: generate per-product recommendations
│   │   │   └── rule-evaluation-queue.ts             # Scheduled pricing rule evaluation
│   │   │
│   │   ├── workers/
│   │   │   └── scrape-worker.ts                     # Playwright competitor price scraping
│   │   │
│   │   ├── services/
│   │   │   ├── price-tracker.ts                     # Append-only price change recording + revenue impact
│   │   │   ├── rule-engine.ts                       # Pricing rule condition evaluation
│   │   │   └── batch-recommend.ts                   # Batch recommendation orchestration
│   │   │
│   │   ├── integrations/
│   │   │   ├── encryption.ts                        # AES-256-GCM encrypt/decrypt for OAuth tokens
│   │   │   └── shopify/
│   │   │       ├── oauth.ts                         # OAuth flow helpers + HMAC verification
│   │   │       ├── sync-products.ts                 # Product catalog sync with pagination
│   │   │       ├── sync-orders.ts                   # Order history import (last 12 months)
│   │   │       ├── sync-inventory.ts                # Inventory level sync
│   │   │       ├── register-webhooks.ts             # Programmatic webhook registration
│   │   │       ├── update-price.ts                  # Push price changes back to Shopify
│   │   │       └── helpers.ts                       # Upsert helpers + Link header parser
│   │   │
│   │   └── billing/
│   │       └── stripe.ts                            # Plan definitions, checkout, portal
│   │
│   ├── components/
│   │   ├── ui/                                      # Shared Radix UI primitives
│   │   │   ├── button.tsx
│   │   │   ├── card.tsx
│   │   │   ├── badge.tsx
│   │   │   ├── input.tsx
│   │   │   ├── select.tsx
│   │   │   ├── dialog.tsx
│   │   │   ├── dropdown-menu.tsx
│   │   │   ├── tabs.tsx
│   │   │   ├── toast.tsx
│   │   │   ├── skeleton.tsx
│   │   │   ├── tooltip.tsx
│   │   │   └── popover.tsx
│   │   │
│   │   ├── layout/
│   │   │   ├── sidebar.tsx                          # Navigation sidebar
│   │   │   ├── header.tsx                           # Page header with breadcrumbs
│   │   │   ├── org-switcher.tsx                     # Clerk org switcher wrapper
│   │   │   └── notification-bell.tsx                # Notification dropdown in header
│   │   │
│   │   ├── products/
│   │   │   ├── product-table.tsx                    # Sortable product list table
│   │   │   ├── product-stats-cards.tsx              # Price, cost, margin, inventory cards
│   │   │   ├── price-history-chart.tsx              # Recharts ComposedChart: price + revenue overlay
│   │   │   └── manual-price-dialog.tsx              # Manual price change form
│   │   │
│   │   ├── competitors/
│   │   │   ├── competitor-list.tsx                  # Competitor cards with tracked products
│   │   │   ├── add-tracking-dialog.tsx              # Add competitor product URL
│   │   │   ├── scrape-config-form.tsx               # CSS selector + regex config
│   │   │   ├── price-comparison-chart.tsx           # Bar chart: your price vs competitors
│   │   │   └── competitor-history-chart.tsx          # Line chart: prices over time
│   │   │
│   │   ├── recommendations/
│   │   │   ├── recommendation-card.tsx              # Accept/reject card with reasoning
│   │   │   ├── recommendation-list.tsx              # Paginated recommendation list
│   │   │   └── modify-price-dialog.tsx              # Adjust recommended price before accepting
│   │   │
│   │   ├── rules/
│   │   │   ├── rule-list.tsx                        # Draggable rule list
│   │   │   ├── rule-form.tsx                        # Create/edit rule with dynamic conditions
│   │   │   ├── rule-preview.tsx                     # Dry-run preview table
│   │   │   └── rule-audit-log.tsx                   # History of rule-triggered changes
│   │   │
│   │   └── analytics/
│   │       ├── revenue-trend-chart.tsx              # Recharts area chart
│   │       ├── margin-distribution-chart.tsx        # Histogram
│   │       ├── competitor-position-chart.tsx         # Stacked bar chart
│   │       └── stats-cards.tsx                      # Overview metric cards
│   │
│   └── lib/
│       ├── trpc/
│       │   ├── client.ts                            # React Query tRPC client
│       │   └── server.ts                            # Server-side tRPC caller
│       ├── utils.ts                                 # cn() helper, date/currency formatting
│       └── constants.ts                             # Plan limits, price sources, rule types
│
├── worker/
│   └── index.ts                                     # Railway entry point: starts scrape + recommendation workers
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

1. **Auth flow**: Sign up with Clerk -> create organization -> verify org record created in DB
2. **Shopify OAuth**: Click "Connect Shopify" -> complete OAuth with test store -> verify encrypted token stored in `data_sources`
3. **Product sync**: After OAuth, verify products imported into `products` table with correct prices, SKUs, images
4. **Order history**: Verify last 12 months of orders synced for sales velocity data
5. **Inventory sync**: Verify `inventory_level` populated on products
6. **Shopify webhook — product update**: Change a price in Shopify admin -> verify `priceHistory` entry created with source `shopify_sync`
7. **Shopify webhook — order create**: Place test order -> verify order data captured for analytics
8. **Add competitor**: Create competitor -> add product tracking URL -> verify stored in `competitor_products`
9. **Competitor scrape**: Trigger manual scrape -> verify price extracted and stored in `competitor_price_checks`
10. **Competitor alert**: Scrape detects >5% price change -> verify notification created in `notification_log`
11. **AI recommendation**: Trigger recommendation for product with competitor data -> verify recommendation created with reasoning factors
12. **Accept recommendation**: Accept recommendation -> verify price updated in `products`, recorded in `priceHistory`, pushed to Shopify variant
13. **Reject recommendation**: Reject -> verify status updated, no price change
14. **Accept with modification**: Accept with custom price -> verify `applied_price` differs from `recommended_price`
15. **Price history chart**: Navigate to product detail -> verify price chart renders with revenue overlay
16. **Competitor comparison**: Navigate to product competitor view -> verify bar chart + historical line chart render
17. **Pricing rule (Growth)**: Create competitor-based rule -> preview shows correct products -> activate -> verify prices update on next evaluation
18. **Plan limits**: On Starter plan, attempt to add 6th competitor -> verify error returned
19. **Billing upgrade**: Complete Stripe checkout -> verify plan updated to Growth in DB
20. **Stripe webhook**: Simulate subscription deletion -> verify plan reverts to Starter
21. **Dashboard**: Verify overview stats, opportunities, and recent changes populate correctly

### Key SQL Queries for Verification

```sql
-- Verify products imported from Shopify
SELECT name, sku, current_price, cost, inventory_level, external_id, variant_id
FROM products
WHERE org_id = 'ORG_ID'
ORDER BY created_at DESC
LIMIT 20;

-- Check price history for a product
SELECT ph.price, ph.previous_price, ph.source, ph.reason, ph.effective_at
FROM price_history ph
WHERE ph.product_id = 'PRODUCT_ID'
ORDER BY ph.effective_at DESC
LIMIT 20;

-- Verify competitor price checks
SELECT cp.name, cpc.price, cpc.previous_price, cpc.success, cpc.checked_at
FROM competitor_price_checks cpc
JOIN competitor_products cp ON cp.id = cpc.competitor_product_id
WHERE cp.org_id = 'ORG_ID'
ORDER BY cpc.checked_at DESC
LIMIT 20;

-- Check pending recommendations with expected impact
SELECT p.name, r.current_price, r.recommended_price,
       r.expected_revenue_change_pct, r.confidence_score, r.status
FROM recommendations r
JOIN products p ON p.id = r.product_id
WHERE r.org_id = 'ORG_ID' AND r.status = 'pending'
ORDER BY r.expected_revenue_change_pct DESC;

-- Monthly product count (for plan enforcement)
SELECT COUNT(*) FROM products
WHERE org_id = 'ORG_ID' AND status = 'active';

-- Competitor count (for plan enforcement)
SELECT COUNT(*) FROM competitors
WHERE org_id = 'ORG_ID';

-- Revenue impact of accepted recommendations
SELECT p.name, r.current_price, r.recommended_price, r.applied_price,
       r.expected_revenue_change_pct, r.responded_at
FROM recommendations r
JOIN products p ON p.id = r.product_id
WHERE r.org_id = 'ORG_ID' AND r.status = 'accepted'
ORDER BY r.responded_at DESC;
```

### Performance Benchmarks

| Operation | Target | Method |
|-----------|--------|--------|
| Shopify webhook receive -> 200 response | < 500ms | Validate HMAC + upsert, async heavy processing |
| Product list page load (100 products) | < 400ms | Indexed queries + React Query cache |
| Single competitor scrape (Playwright) | < 15s | Page load + wait + extract |
| Daily scrape batch (100 URLs) | < 30min | 3 concurrent workers, 5s stagger between jobs |
| AI recommendation (single product) | < 5s | GPT-4o with 600 max_tokens + context gathering |
| Batch recommendations (100 products) | < 15min | 5 concurrent, rate-limited to 10 req/min |
| Price history chart data query | < 200ms | Indexed on (product_id, effective_at) |
| Competitor comparison query (5 competitors) | < 300ms | Indexed on competitor_product_id + checked_at |
| Dashboard overview load | < 600ms | Parallel tRPC queries + SSR |
| Pricing rule evaluation (all products) | < 10s | Single pass through products with rule matching |
| Shopify price push-back (variant update) | < 2s | Single Shopify Admin API call |

---

## Explicit Connector Scope

**MVP: Shopify-only** via Shopify Admin REST API + GraphQL Admin API.

- Product catalog sync via `GET /admin/api/2024-10/products.json` (REST) and bulk queries via GraphQL for efficiency on large catalogs
- Order history import via `GET /admin/api/2024-10/orders.json` (last 12 months)
- Inventory levels via `GET /admin/api/2024-10/inventory_levels.json`
- Real-time updates via Shopify webhooks: `products/update`, `orders/create`, `inventory_levels/update`
- Price push-back via `PUT /admin/api/2024-10/variants/{id}.json` to update variant price after recommendation acceptance

**Deferred to post-MVP:**
- **WooCommerce**: REST API v3, similar product/order/inventory structure but different auth (consumer key/secret)
- **BigCommerce**: REST API v3 + webhooks, catalog and order management APIs
- **Magento**: REST API with OAuth 1.0a, product and sales order endpoints
- **Custom API connectors**: User-configurable REST API connector with mapping templates for product schema normalization

---

## Price Elasticity Analysis (spec F3)

Price elasticity is a core capability that powers accurate pricing recommendations:

- **Demand curve modeling**: For each product, the system builds a demand curve by correlating historical price points (from `priceHistory`) with sales velocity (units sold per week from order data). The curve is modeled as a log-linear regression: `ln(quantity) = a + b * ln(price)`, where `b` is the elasticity coefficient.
- **Price sensitivity scoring per product/segment**: Each product receives an elasticity score from -3.0 (highly elastic, demand drops sharply with price increase) to 0 (perfectly inelastic). Products are also scored at the category/segment level to provide baseline estimates for products with limited individual history.
- **Elasticity coefficient calculation**: Requires a minimum of 3 distinct price points with at least 2 weeks of sales data at each point. For products below this threshold, category-level elasticity is used as a proxy. The coefficient is recalculated weekly as new order data arrives.
- **Visualization of price-demand relationship**: The product detail page includes a demand curve chart (Recharts ScatterChart with regression line) showing historical price-quantity data points overlaid with the modeled demand curve. Users can hover over the curve to see estimated units at any price point.

---

## A/B Price Testing (spec F5)

A/B price testing allows merchants to validate pricing changes with statistical rigor:

- **Test creation flow**: From the product detail page, click "Start Price Test". Configure: control price (current), variant price (recommended or custom), traffic split (default 50/50), minimum duration (default 14 days), and success metric (revenue per visitor or conversion rate).
- **Variant assignment (hash-based)**: Visitors are deterministically assigned to control or variant using a murmur3 hash of `productId + visitorId (Shopify customer ID or session cookie)`. This ensures consistent pricing per visitor across sessions. Assignment is recorded in a `priceTestAssignments` table.
- **Statistical significance calculator (chi-squared)**: The system continuously computes the chi-squared test statistic comparing conversion rates between control and variant. Results are displayed as a live significance meter in the test dashboard, showing current p-value, confidence level, and estimated days to reach 95% significance.
- **Minimum sample size recommendations**: Before starting a test, the system estimates the required sample size based on the expected effect size (derived from the price difference) and baseline conversion rate. Displayed as "Estimated time to significance: X days at your current traffic level."
- **Auto-winner selection**: When the test reaches 95% statistical significance and the minimum duration has elapsed, the system automatically declares a winner and optionally applies the winning price (if `autoApplyRecommendations` is enabled in org settings). A notification is sent to the team with full results.

---

## Revenue Impact Simulator (spec F6)

The revenue impact simulator helps merchants understand the financial consequences of pricing decisions:

- **Phase 1 -- Basic Calculator (MVP)**: A simple calculator on the product detail page that takes a proposed price and displays: estimated revenue change (%), estimated margin change, break-even point (units needed to maintain current revenue). Calculations use the product's elasticity coefficient and current sales velocity. Available on all plans.
- **Phase 2 -- Monte Carlo Simulation (Post-MVP, Growth plan)**: A more sophisticated simulator that runs 1,000 Monte Carlo trials per scenario, varying demand elasticity, competitor response probability, and seasonal factors. Outputs a probability distribution of revenue outcomes (10th percentile / median / 90th percentile). Results are displayed as a violin plot chart.
- **Scenario comparison**: Users can create multiple "what-if" scenarios (e.g., "price increase 10%", "match competitor", "seasonal discount") and compare them side by side. Each scenario card shows projected revenue, margin, and units with confidence intervals.

---

## Verification Targets

| Area | Target | Method |
|------|--------|--------|
| Price recommendation accuracy | >70% of accepted recommendations lead to revenue increase within 4 weeks | Compare pre/post revenue for accepted recommendations via `priceHistory` + order data |
| A/B test statistical validity | p < 0.05 false positive rate confirmed via simulation | Run 1000 A/A tests (same price both arms) and verify <5% declare a winner |
| Connector sync reliability | >99.5% webhook processing success rate | Monitor `data_sources.lastSyncedAt` freshness and Shopify webhook delivery logs |
| Dashboard load time | < 600ms for product list (100 products) | Lighthouse CI measurement on staging environment with production-like data volume |

---

## Post-MVP Roadmap

| Feature | Description | Priority |
|---------|-------------|----------|
| WooCommerce & Stripe Billing Connectors (F1) | Extend data integration beyond Shopify to WooCommerce (REST API) and Stripe Billing (subscriptions + invoices) for broader e-commerce coverage | High |
| Dynamic Pricing Rules (F7) | Automated rule engine: "If competitor price drops below X, match within Y hours" or "If inventory > Z, reduce price by N%". Rules execute via scheduled evaluation with approval workflow. | High |
| Advanced Price Elasticity (F3) | Seasonal decomposition, promotional event exclusion, and competitor price correlation in elasticity modeling. Bayesian estimation for products with limited data. | Medium |
| Multi-Variant A/B Testing (F5) | Expand beyond 2 variants to multivariate testing. Bayesian statistical model for faster winner detection with smaller sample sizes. Auto-apply winning prices. | Medium |
| SaaS-Specific Features (F9) | Subscription pricing optimization: plan tier analysis, annual vs monthly discount optimization, usage-based pricing recommendations, churn-price sensitivity analysis | Medium |
| BigCommerce & Amazon Connectors | Additional e-commerce platform support for BigCommerce (REST API) and Amazon Seller Central (SP-API) | Low |
| Custom API Connector | Generic REST API connector for importing pricing and sales data from custom or niche e-commerce platforms | Low |
