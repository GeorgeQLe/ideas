# 4. FormForge — Natural Language Form Builder

## Implementation Plan

**MVP Scope:** Natural language form generation via GPT-4o (describe a form in plain English → get a fully functional form with appropriate field types, validation rules, and conditional logic), visual drag-and-drop form editor using dnd-kit with 10 field types (text, email, number, dropdown, radio, checkbox, textarea, date, rating, file upload), conditional show/hide logic with visual rule builder, 3 built-in themes (minimal, corporate, playful) with custom branding (colors, logo), hosted form pages at `{slug}.formforge.io/{form-slug}`, response collection with table view dashboard, email notifications to form owners on submission via Resend, file uploads to Cloudflare R2 with type/size validation, Cloudflare Turnstile spam protection on public forms, Stripe billing with three tiers (Free / Pro $15/mo / Business $39/mo).

---

## Tech Decisions

| Layer | Technology | Notes |
|-------|-----------|-------|
| Framework | Next.js 14+ App Router | `src/` directory, TypeScript strict |
| API | tRPC v11 | superjson transformer, Clerk auth context |
| Database | Neon PostgreSQL | Forms, fields, responses, themes |
| ORM | Drizzle ORM | Push-based migrations, typed schema |
| Auth | Clerk | Organization-scoped, `orgId` from session |
| Billing | Stripe | Checkout Sessions + Customer Portal + Webhooks |
| AI — Generation | OpenAI GPT-4o | JSON mode, structured form schema output |
| AI — Field Inference | OpenAI GPT-4o-mini | Temperature 0.1 for field type classification |
| Email | Resend | Submission notifications, respondent confirmations |
| File Storage | Cloudflare R2 | Uploaded files, logos, OG images |
| Drag & Drop | @dnd-kit/core + @dnd-kit/sortable | Field reordering + palette-to-canvas drops |
| Rich Text | Tiptap (ProseMirror) | Rich text field type + help text formatting |
| Spam Protection | Cloudflare Turnstile | Invisible challenge on public form pages |
| UI | Tailwind CSS, Radix UI, Recharts | |
| Hosting | Vercel | ISR for public form pages |
| Monitoring | Sentry, Vercel Analytics | |

### Key Architecture Decisions

1. **JSON Schema as canonical form representation**: Every form is stored as a JSON schema in `forms.schema` — an array of field definitions with types, validation rules, options, and conditional logic. The AI generates this schema, the editor manipulates it, and the renderer interprets it. This single source of truth eliminates translation layers between generation, editing, and rendering.

2. **Dual rendering: editor mode vs respondent mode**: The form renderer is a single React component (`<FormRenderer />`) that accepts a `mode` prop — `edit` enables field selection, property panels, and drag handles; `respond` renders a clean submission-ready form. The respondent mode is also used for the hosted form page, ensuring pixel-perfect WYSIWYG between editor preview and live form.

3. **File uploads via presigned R2 URLs**: File upload fields generate presigned PUT URLs from the server. The client uploads directly to R2, bypassing the Next.js server for large files. On form submission, only the R2 object keys are stored in `fieldResponses.fileKeys`. A cleanup cron (daily) removes orphaned uploads older than 24 hours that were never attached to a submission.

4. **Conditional logic as field-level metadata**: Each field stores its own visibility conditions in `conditionalLogic` — an array of `{ fieldId, operator, value }` clauses with AND/OR combinator. The renderer evaluates conditions client-side in real-time as the respondent fills out the form. This keeps the logic co-located with the field and avoids a separate conditional logic table.

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
  branding: jsonb("branding").default({}).$type<{
    logoUrl?: string;
    primaryColor?: string;
    accentColor?: string;
    fontFamily?: string;
    faviconUrl?: string;
  }>(),
  settings: jsonb("settings").default({}).$type<{
    notificationFromName?: string;
    notificationFromEmail?: string;
    onboardingCompleted?: boolean;
    defaultThemeId?: string;
  }>(),
  totalStorageBytes: integer("total_storage_bytes").default(0).notNull(),
  createdAt: timestamp("created_at", { withTimezone: true }).defaultNow().notNull(),
  updatedAt: timestamp("updated_at", { withTimezone: true }).defaultNow().notNull(),
});

// ---------------------------------------------------------------------------
// Organization Members
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
    index("members_org_idx").on(table.orgId),
  ],
);

// ---------------------------------------------------------------------------
// Themes
// ---------------------------------------------------------------------------
export const themes = pgTable(
  "themes",
  {
    id: uuid("id").primaryKey().defaultRandom(),
    orgId: uuid("org_id")
      .references(() => organizations.id, { onDelete: "cascade" }),
    name: text("name").notNull(),
    isSystem: boolean("is_system").default(false).notNull(),
    colors: jsonb("colors").default({}).$type<{
      background: string;
      surface: string;
      primary: string;
      primaryForeground: string;
      text: string;
      textSecondary: string;
      border: string;
      error: string;
      success: string;
    }>(),
    fontFamily: text("font_family").default("Inter"),
    borderRadius: text("border_radius").default("8px"),
    cssOverrides: text("css_overrides"),
    createdAt: timestamp("created_at", { withTimezone: true }).defaultNow().notNull(),
  },
  (table) => [
    index("themes_org_idx").on(table.orgId),
  ],
);

// ---------------------------------------------------------------------------
// Forms
// ---------------------------------------------------------------------------
export const forms = pgTable(
  "forms",
  {
    id: uuid("id").primaryKey().defaultRandom(),
    orgId: uuid("org_id")
      .references(() => organizations.id, { onDelete: "cascade" })
      .notNull(),
    title: text("title").notNull(),
    description: text("description"),
    slug: text("slug").notNull(),
    status: text("status").default("draft").notNull(), // draft | published | closed
    themeId: uuid("theme_id")
      .references(() => themes.id, { onDelete: "set null" }),
    customCss: text("custom_css"),
    schema: jsonb("schema").default([]).$type<FormFieldSchema[]>(),
    settings: jsonb("settings").default({}).$type<{
      requireLogin?: boolean;
      password?: string;
      responseLimit?: number;
      closeDate?: string; // ISO date
      redirectUrl?: string;
      successMessage?: string;
      notificationEmails?: string[];
      gdprConsentEnabled?: boolean;
      gdprConsentText?: string;
      turnstileEnabled?: boolean;
      brandingVisible?: boolean; // "Powered by FormForge"
      progressBar?: boolean;
    }>(),
    aiPrompt: text("ai_prompt"), // Original NL description used to generate this form
    responseCount: integer("response_count").default(0).notNull(),
    viewCount: integer("view_count").default(0).notNull(),
    publishedAt: timestamp("published_at", { withTimezone: true }),
    closedAt: timestamp("closed_at", { withTimezone: true }),
    createdAt: timestamp("created_at", { withTimezone: true }).defaultNow().notNull(),
    updatedAt: timestamp("updated_at", { withTimezone: true }).defaultNow().notNull(),
  },
  (table) => [
    index("forms_org_idx").on(table.orgId),
    uniqueIndex("forms_org_slug_idx").on(table.orgId, table.slug),
    index("forms_status_idx").on(table.orgId, table.status),
  ],
);

// ---------------------------------------------------------------------------
// Form Field Schema Type (used in forms.schema JSONB)
// ---------------------------------------------------------------------------
// Not a table — this is the TypeScript type for the JSONB schema array
export type FormFieldSchema = {
  id: string; // UUID, stable across edits
  type: FieldType;
  label: string;
  placeholder?: string;
  helpText?: string;
  required: boolean;
  sortOrder: number;
  options?: FieldOption[]; // For dropdown, radio, checkbox
  validation?: FieldValidation;
  conditionalLogic?: ConditionalLogic;
  pageNumber?: number; // For multi-page forms (v2 — see Multi-Page Form Rendering note below)
};

// NOTE: Multi-Page Form Rendering — deferred to v2. When `pageNumber` is set on
// fields, the renderer will group fields by page and display prev/next navigation
// with a progress bar. Page transitions will persist partial responses to
// sessionStorage and validate the current page before advancing. The form schema
// already supports `pageNumber` as an optional field; the v2 renderer will honor
// it while the v1 renderer ignores it and displays all fields on a single page.

export type FieldType =
  | "text"
  | "email"
  | "number"
  | "textarea"
  | "dropdown"
  | "radio"
  | "checkbox"
  | "date"
  | "rating"
  | "file_upload"
  | "section_header"
  | "hidden"
  | "signature"     // Post-MVP: signature capture via canvas/touch input, stored as PNG data URL in R2
  | "rich_text";    // Post-MVP: Tiptap-based rich text editor field for respondents (limited formatting)

export type FieldOption = {
  id: string;
  label: string;
  value: string;
};

export type FieldValidation = {
  minLength?: number;
  maxLength?: number;
  min?: number;
  max?: number;
  pattern?: string;
  patternMessage?: string;
  acceptedFileTypes?: string[]; // [".pdf", ".jpg", ".png"]
  maxFileSizeMb?: number;
  maxFileCount?: number;
};

export type ConditionalLogic = {
  action: "show" | "hide" | "require";
  combinator: "AND" | "OR";
  conditions: ConditionalCondition[];
};

export type ConditionalCondition = {
  fieldId: string;
  operator: "equals" | "not_equals" | "contains" | "greater_than" | "less_than" | "is_empty" | "is_not_empty";
  value: string;
};

// ---------------------------------------------------------------------------
// Form Responses
// ---------------------------------------------------------------------------
export const formResponses = pgTable(
  "form_responses",
  {
    id: uuid("id").primaryKey().defaultRandom(),
    formId: uuid("form_id")
      .references(() => forms.id, { onDelete: "cascade" })
      .notNull(),
    orgId: uuid("org_id")
      .references(() => organizations.id, { onDelete: "cascade" })
      .notNull(),
    status: text("status").default("new").notNull(), // new | read | starred | archived
    respondentEmail: text("respondent_email"),
    respondentIp: text("respondent_ip"), // nullable, anonymizable
    completionTimeSeconds: integer("completion_time_seconds"),
    utmParams: jsonb("utm_params").default({}).$type<{
      source?: string;
      medium?: string;
      campaign?: string;
      term?: string;
      content?: string;
    }>(),
    metadata: jsonb("metadata").default({}).$type<{
      userAgent?: string;
      referrer?: string;
      pageUrl?: string;
    }>(),
    submittedAt: timestamp("submitted_at", { withTimezone: true }).defaultNow().notNull(),
  },
  (table) => [
    index("responses_form_idx").on(table.formId),
    index("responses_org_idx").on(table.orgId),
    index("responses_status_idx").on(table.formId, table.status),
    index("responses_submitted_at_idx").on(table.formId, table.submittedAt),
  ],
);

// ---------------------------------------------------------------------------
// Field Responses (individual answer per field)
// ---------------------------------------------------------------------------
export const fieldResponses = pgTable(
  "field_responses",
  {
    id: uuid("id").primaryKey().defaultRandom(),
    responseId: uuid("response_id")
      .references(() => formResponses.id, { onDelete: "cascade" })
      .notNull(),
    fieldId: text("field_id").notNull(), // Matches FormFieldSchema.id
    fieldLabelSnapshot: text("field_label_snapshot").notNull(), // Historical accuracy
    value: text("value"), // Text value for most field types
    fileKeys: jsonb("file_keys").default([]).$type<string[]>(), // R2 object keys for file uploads
    createdAt: timestamp("created_at", { withTimezone: true }).defaultNow().notNull(),
  },
  (table) => [
    index("field_responses_response_idx").on(table.responseId),
    index("field_responses_field_idx").on(table.fieldId),
  ],
);

// ---------------------------------------------------------------------------
// Form Integrations (webhook, email, Slack — v1.1)
// ---------------------------------------------------------------------------
export const formIntegrations = pgTable(
  "form_integrations",
  {
    id: uuid("id").primaryKey().defaultRandom(),
    formId: uuid("form_id")
      .references(() => forms.id, { onDelete: "cascade" })
      .notNull(),
    orgId: uuid("org_id")
      .references(() => organizations.id, { onDelete: "cascade" })
      .notNull(),
    type: text("type").notNull(), // email | slack | webhook | sheets | zapier
    config: jsonb("config").default({}).$type<{
      // Email
      recipientEmails?: string[];
      includeResponses?: boolean;
      // Webhook
      webhookUrl?: string;
      webhookSecret?: string;
      // Slack
      slackWebhookUrl?: string;
      slackChannelName?: string;
    }>(),
    enabled: boolean("enabled").default(true).notNull(),
    createdAt: timestamp("created_at", { withTimezone: true }).defaultNow().notNull(),
  },
  (table) => [
    index("integrations_form_idx").on(table.formId),
    index("integrations_org_idx").on(table.orgId),
  ],
);

// ---------------------------------------------------------------------------
// Form Views (analytics tracking)
// ---------------------------------------------------------------------------
export const formViews = pgTable(
  "form_views",
  {
    id: uuid("id").primaryKey().defaultRandom(),
    formId: uuid("form_id")
      .references(() => forms.id, { onDelete: "cascade" })
      .notNull(),
    visitorId: text("visitor_id"), // Anonymous fingerprint or session ID
    referrer: text("referrer"),
    utmSource: text("utm_source"),
    startedAt: boolean("started").default(false).notNull(), // Did they interact with any field?
    completedAt: boolean("completed").default(false).notNull(), // Did they submit?
    createdAt: timestamp("created_at", { withTimezone: true }).defaultNow().notNull(),
  },
  (table) => [
    index("form_views_form_idx").on(table.formId),
    index("form_views_created_at_idx").on(table.formId, table.createdAt),
  ],
);

// ---------------------------------------------------------------------------
// File Uploads (tracking for storage management)
// ---------------------------------------------------------------------------
export const fileUploads = pgTable(
  "file_uploads",
  {
    id: uuid("id").primaryKey().defaultRandom(),
    orgId: uuid("org_id")
      .references(() => organizations.id, { onDelete: "cascade" })
      .notNull(),
    formId: uuid("form_id")
      .references(() => forms.id, { onDelete: "cascade" })
      .notNull(),
    responseId: uuid("response_id")
      .references(() => formResponses.id, { onDelete: "cascade" }),
    r2Key: text("r2_key").notNull(),
    fileName: text("file_name").notNull(),
    mimeType: text("mime_type").notNull(),
    sizeBytes: integer("size_bytes").notNull(),
    isAttached: boolean("is_attached").default(false).notNull(), // Linked to a submitted response?
    createdAt: timestamp("created_at", { withTimezone: true }).defaultNow().notNull(),
  },
  (table) => [
    index("file_uploads_org_idx").on(table.orgId),
    index("file_uploads_form_idx").on(table.formId),
    index("file_uploads_response_idx").on(table.responseId),
    index("file_uploads_unattached_idx").on(table.isAttached, table.createdAt),
  ],
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
    formId: uuid("form_id")
      .references(() => forms.id, { onDelete: "cascade" })
      .notNull(),
    responseId: uuid("response_id")
      .references(() => formResponses.id, { onDelete: "cascade" }),
    type: text("type").notNull(), // submission_alert | respondent_confirmation
    channel: text("channel").default("email").notNull(), // email
    recipientEmail: text("recipient_email").notNull(),
    status: text("status").default("pending").notNull(), // pending | sent | failed
    sentAt: timestamp("sent_at", { withTimezone: true }),
    errorMessage: text("error_message"),
    metadata: jsonb("metadata").default({}).$type<{
      resendMessageId?: string;
      formTitle?: string;
      responsePreview?: string;
    }>(),
    createdAt: timestamp("created_at", { withTimezone: true }).defaultNow().notNull(),
  },
  (table) => [
    index("notification_log_org_idx").on(table.orgId),
    index("notification_log_form_idx").on(table.formId),
    index("notification_log_status_idx").on(table.status),
  ],
);

// ---------------------------------------------------------------------------
// Relations
// ---------------------------------------------------------------------------
export const organizationsRelations = relations(organizations, ({ many }) => ({
  members: many(members),
  forms: many(forms),
  themes: many(themes),
  formIntegrations: many(formIntegrations),
  fileUploads: many(fileUploads),
  notificationLog: many(notificationLog),
}));

export const membersRelations = relations(members, ({ one }) => ({
  organization: one(organizations, {
    fields: [members.orgId],
    references: [organizations.id],
  }),
}));

export const themesRelations = relations(themes, ({ one, many }) => ({
  organization: one(organizations, {
    fields: [themes.orgId],
    references: [organizations.id],
  }),
  forms: many(forms),
}));

export const formsRelations = relations(forms, ({ one, many }) => ({
  organization: one(organizations, {
    fields: [forms.orgId],
    references: [organizations.id],
  }),
  theme: one(themes, {
    fields: [forms.themeId],
    references: [themes.id],
  }),
  responses: many(formResponses),
  integrations: many(formIntegrations),
  views: many(formViews),
  fileUploads: many(fileUploads),
  notificationLog: many(notificationLog),
}));

export const formResponsesRelations = relations(formResponses, ({ one, many }) => ({
  form: one(forms, {
    fields: [formResponses.formId],
    references: [forms.id],
  }),
  organization: one(organizations, {
    fields: [formResponses.orgId],
    references: [organizations.id],
  }),
  fieldResponses: many(fieldResponses),
  fileUploads: many(fileUploads),
  notificationLog: many(notificationLog),
}));

export const fieldResponsesRelations = relations(fieldResponses, ({ one }) => ({
  response: one(formResponses, {
    fields: [fieldResponses.responseId],
    references: [formResponses.id],
  }),
}));

export const formIntegrationsRelations = relations(formIntegrations, ({ one }) => ({
  form: one(forms, {
    fields: [formIntegrations.formId],
    references: [forms.id],
  }),
  organization: one(organizations, {
    fields: [formIntegrations.orgId],
    references: [organizations.id],
  }),
}));

export const formViewsRelations = relations(formViews, ({ one }) => ({
  form: one(forms, {
    fields: [formViews.formId],
    references: [forms.id],
  }),
}));

export const fileUploadsRelations = relations(fileUploads, ({ one }) => ({
  organization: one(organizations, {
    fields: [fileUploads.orgId],
    references: [organizations.id],
  }),
  form: one(forms, {
    fields: [fileUploads.formId],
    references: [forms.id],
  }),
  response: one(formResponses, {
    fields: [fileUploads.responseId],
    references: [formResponses.id],
  }),
}));

export const notificationLogRelations = relations(notificationLog, ({ one }) => ({
  organization: one(organizations, {
    fields: [notificationLog.orgId],
    references: [organizations.id],
  }),
  form: one(forms, {
    fields: [notificationLog.formId],
    references: [forms.id],
  }),
  response: one(formResponses, {
    fields: [notificationLog.responseId],
    references: [formResponses.id],
  }),
}));
```

---

## Initial Migration

```sql
-- migrations/0000_init.sql

-- No pgvector needed for FormForge — standard PostgreSQL only

-- Seed system themes
INSERT INTO themes (id, name, is_system, colors, font_family, border_radius)
VALUES
  (gen_random_uuid(), 'Minimal', true,
   '{"background":"#ffffff","surface":"#f9fafb","primary":"#111827","primaryForeground":"#ffffff","text":"#111827","textSecondary":"#6b7280","border":"#e5e7eb","error":"#ef4444","success":"#22c55e"}',
   'Inter', '8px'),
  (gen_random_uuid(), 'Corporate', true,
   '{"background":"#f8fafc","surface":"#ffffff","primary":"#1e40af","primaryForeground":"#ffffff","text":"#0f172a","textSecondary":"#475569","border":"#cbd5e1","error":"#dc2626","success":"#16a34a"}',
   'Inter', '6px'),
  (gen_random_uuid(), 'Playful', true,
   '{"background":"#fefce8","surface":"#ffffff","primary":"#7c3aed","primaryForeground":"#ffffff","text":"#1e1b4b","textSecondary":"#6366f1","border":"#c4b5fd","error":"#f43f5e","success":"#10b981"}',
   'Nunito', '12px');
```

---

## Architecture Deep-Dives

### 1. NL → Form Schema AI Engine

The core AI pipeline takes a plain-English form description and generates a complete form schema with appropriate field types, validation rules, and conditional logic. It uses a two-step approach: GPT-4o generates the full schema, then a validation pass ensures type consistency and sanitizes the output.

```typescript
// src/server/ai/generate-form.ts

import OpenAI from "openai";
import { randomUUID } from "crypto";
import type { FormFieldSchema, FieldType, FieldOption, ConditionalLogic } from "../db/schema";

const openai = new OpenAI();

const SYSTEM_PROMPT = `You are a form builder AI. Given a natural language description of a form, generate a JSON array of form fields.

Each field must have this structure:
{
  "type": one of "text" | "email" | "number" | "textarea" | "dropdown" | "radio" | "checkbox" | "date" | "rating" | "file_upload" | "section_header",
  "label": string,
  "placeholder": string (optional),
  "helpText": string (optional),
  "required": boolean,
  "options": [{"label": string, "value": string}] (required for dropdown, radio, checkbox),
  "validation": {
    "minLength": number (optional),
    "maxLength": number (optional),
    "min": number (optional),
    "max": number (optional),
    "pattern": string (optional, regex),
    "patternMessage": string (optional),
    "acceptedFileTypes": string[] (optional, e.g. [".pdf", ".jpg"]),
    "maxFileSizeMb": number (optional),
    "maxFileCount": number (optional, default 1)
  } (optional),
  "conditionalLogic": {
    "action": "show" | "hide",
    "combinator": "AND" | "OR",
    "conditions": [{"fieldRef": string (label of the referenced field), "operator": "equals" | "not_equals" | "contains" | "greater_than" | "less_than" | "is_empty" | "is_not_empty", "value": string}]
  } (optional, only when the description implies conditional visibility)
}

Rules:
1. Infer field types from context: "email" → email, "phone" → text with pattern, "rating" → rating, "resume" or "upload" → file_upload, etc.
2. Set sensible defaults: email fields are usually required, ratings default to max 5, textareas get maxLength 2000.
3. Add validation automatically: email fields get email pattern, phone fields get phone pattern, etc.
4. Only add conditional logic when the description explicitly mentions conditions (e.g., "if X then show Y").
5. Use "section_header" type to group related fields when the form has distinct sections.
6. Order fields logically: personal info first, then topic-specific fields, then free-text fields last.
7. In conditional logic, use "fieldRef" as the label of the field being referenced (we'll resolve to IDs later).

Return ONLY the JSON array, no markdown, no explanation.`;

export async function generateFormSchema(
  prompt: string,
): Promise<{ fields: FormFieldSchema[]; title: string; description: string }> {
  // Step 1: Generate title + description
  const metaResponse = await openai.chat.completions.create({
    model: "gpt-4o-mini",
    temperature: 0.3,
    max_tokens: 200,
    response_format: { type: "json_object" },
    messages: [
      {
        role: "system",
        content:
          'Given a form description, generate a concise title (3-6 words) and a short description (1 sentence). Return JSON: {"title": "...", "description": "..."}',
      },
      { role: "user", content: prompt },
    ],
  });

  const meta = JSON.parse(metaResponse.choices[0].message.content!) as {
    title: string;
    description: string;
  };

  // Step 2: Generate field schema
  const schemaResponse = await openai.chat.completions.create({
    model: "gpt-4o",
    temperature: 0.4,
    max_tokens: 4000,
    response_format: { type: "json_object" },
    messages: [
      { role: "system", content: SYSTEM_PROMPT },
      {
        role: "user",
        content: `Generate form fields for: "${prompt}"\n\nReturn as {"fields": [...]}`,
      },
    ],
  });

  const rawFields = JSON.parse(schemaResponse.choices[0].message.content!) as {
    fields: RawAIField[];
  };

  // Step 3: Validate, assign IDs, resolve conditional logic references
  const fields = normalizeAIFields(rawFields.fields);

  return { fields, title: meta.title, description: meta.description };
}

type RawAIField = {
  type: string;
  label: string;
  placeholder?: string;
  helpText?: string;
  required?: boolean;
  options?: { label: string; value: string }[];
  validation?: Record<string, unknown>;
  conditionalLogic?: {
    action: string;
    combinator: string;
    conditions: { fieldRef: string; operator: string; value: string }[];
  };
};

const VALID_TYPES = new Set<FieldType>([
  "text", "email", "number", "textarea", "dropdown", "radio",
  "checkbox", "date", "rating", "file_upload", "section_header", "hidden",
]);

function normalizeAIFields(rawFields: RawAIField[]): FormFieldSchema[] {
  // First pass: assign IDs and validate types
  const fieldsWithIds = rawFields.map((raw, index) => {
    const id = randomUUID();
    const type: FieldType = VALID_TYPES.has(raw.type as FieldType)
      ? (raw.type as FieldType)
      : "text";

    // Ensure option-based types have options
    const needsOptions = ["dropdown", "radio", "checkbox"].includes(type);
    const options: FieldOption[] | undefined = needsOptions
      ? (raw.options || []).map((opt) => ({
          id: randomUUID(),
          label: opt.label,
          value: opt.value || opt.label.toLowerCase().replace(/\s+/g, "_"),
        }))
      : undefined;

    // Default validation for specific types
    const validation = normalizeValidation(type, raw.validation);

    return {
      id,
      type,
      label: raw.label || `Field ${index + 1}`,
      placeholder: raw.placeholder,
      helpText: raw.helpText,
      required: raw.required ?? false,
      sortOrder: index,
      options,
      validation,
      _rawConditionalLogic: raw.conditionalLogic,
    };
  });

  // Build label → ID map for resolving conditional logic references
  const labelToId = new Map<string, string>();
  for (const field of fieldsWithIds) {
    labelToId.set(field.label.toLowerCase(), field.id);
  }

  // Second pass: resolve conditional logic fieldRef → fieldId
  return fieldsWithIds.map(({ _rawConditionalLogic, ...field }) => {
    if (!_rawConditionalLogic?.conditions?.length) return field;

    const conditions = _rawConditionalLogic.conditions
      .map((cond) => {
        const fieldId = labelToId.get(cond.fieldRef?.toLowerCase() ?? "");
        if (!fieldId) return null;
        return {
          fieldId,
          operator: cond.operator as ConditionalLogic["conditions"][0]["operator"],
          value: cond.value || "",
        };
      })
      .filter(Boolean) as ConditionalLogic["conditions"];

    if (conditions.length === 0) return field;

    return {
      ...field,
      conditionalLogic: {
        action: (_rawConditionalLogic.action === "hide" ? "hide" : "show") as "show" | "hide",
        combinator: (_rawConditionalLogic.combinator === "OR" ? "OR" : "AND") as "AND" | "OR",
        conditions,
      },
    };
  });
}

function normalizeValidation(
  type: FieldType,
  raw?: Record<string, unknown>,
): FormFieldSchema["validation"] | undefined {
  const v: FormFieldSchema["validation"] = { ...raw } as FormFieldSchema["validation"];

  switch (type) {
    case "email":
      return {
        ...v,
        pattern: v?.pattern || "^[^\\s@]+@[^\\s@]+\\.[^\\s@]+$",
        patternMessage: v?.patternMessage || "Please enter a valid email address",
      };
    case "number":
      return v;
    case "textarea":
      return { maxLength: 2000, ...v };
    case "rating":
      return { min: 1, max: v?.max ?? 5, ...v };
    case "file_upload":
      return {
        maxFileSizeMb: v?.maxFileSizeMb ?? 10,
        maxFileCount: v?.maxFileCount ?? 1,
        acceptedFileTypes: v?.acceptedFileTypes ?? [".pdf", ".jpg", ".jpeg", ".png", ".doc", ".docx"],
        ...v,
      };
    default:
      return v && Object.keys(v).length > 0 ? v : undefined;
  }
}
```

### 2. Form Renderer with Conditional Logic Evaluation

The form renderer is the heart of the application — a single component that renders both the editor preview and the live respondent form. It evaluates conditional logic in real-time as the user fills out the form, handles all 10 field types, and integrates with react-hook-form for validation.

```typescript
// src/components/form-renderer/form-renderer.tsx

"use client";

import { useForm, Controller, type UseFormReturn } from "react-hook-form";
import { useCallback, useMemo } from "react";
import type {
  FormFieldSchema,
  ConditionalLogic,
  ConditionalCondition,
} from "@/server/db/schema";

type FormValues = Record<string, string | string[] | File[]>;

interface FormRendererProps {
  fields: FormFieldSchema[];
  mode: "respond" | "edit";
  theme: ThemeConfig;
  onSubmit?: (values: FormValues) => Promise<void>;
  onFieldSelect?: (fieldId: string) => void; // Editor mode only
  selectedFieldId?: string; // Editor mode only
  disabled?: boolean;
}

interface ThemeConfig {
  colors: Record<string, string>;
  fontFamily: string;
  borderRadius: string;
  cssOverrides?: string;
}

export function FormRenderer({
  fields,
  mode,
  theme,
  onSubmit,
  onFieldSelect,
  selectedFieldId,
  disabled,
}: FormRendererProps) {
  const form = useForm<FormValues>({
    defaultValues: buildDefaults(fields),
    mode: "onBlur",
  });

  const watchedValues = form.watch();

  // Evaluate conditional visibility for all fields
  const visibleFieldIds = useMemo(
    () => evaluateVisibility(fields, watchedValues),
    [fields, watchedValues],
  );

  const handleSubmit = useCallback(
    async (values: FormValues) => {
      // Filter out hidden field values before submission
      const visibleValues: FormValues = {};
      for (const fieldId of visibleFieldIds) {
        if (values[fieldId] !== undefined) {
          visibleValues[fieldId] = values[fieldId];
        }
      }
      await onSubmit?.(visibleValues);
    },
    [onSubmit, visibleFieldIds],
  );

  const cssVars = useMemo(
    () => ({
      "--ff-bg": theme.colors.background,
      "--ff-surface": theme.colors.surface,
      "--ff-primary": theme.colors.primary,
      "--ff-primary-fg": theme.colors.primaryForeground,
      "--ff-text": theme.colors.text,
      "--ff-text-secondary": theme.colors.textSecondary,
      "--ff-border": theme.colors.border,
      "--ff-error": theme.colors.error,
      "--ff-radius": theme.borderRadius,
      "--ff-font": theme.fontFamily,
    }),
    [theme],
  ) as React.CSSProperties;

  return (
    <form
      onSubmit={form.handleSubmit(handleSubmit)}
      className="ff-form"
      style={cssVars}
    >
      {fields
        .sort((a, b) => a.sortOrder - b.sortOrder)
        .map((field) => {
          const isVisible = visibleFieldIds.has(field.id);
          if (!isVisible && mode === "respond") return null;

          return (
            <FieldWrapper
              key={field.id}
              field={field}
              mode={mode}
              isVisible={isVisible}
              isSelected={selectedFieldId === field.id}
              onSelect={() => onFieldSelect?.(field.id)}
            >
              <FieldRenderer
                field={field}
                form={form}
                mode={mode}
                disabled={disabled || !isVisible}
              />
            </FieldWrapper>
          );
        })}

      {mode === "respond" && (
        <button
          type="submit"
          disabled={disabled || form.formState.isSubmitting}
          className="ff-submit-button"
        >
          {form.formState.isSubmitting ? "Submitting..." : "Submit"}
        </button>
      )}
    </form>
  );
}

// ---- Conditional Logic Evaluator ----

function evaluateVisibility(
  fields: FormFieldSchema[],
  values: FormValues,
): Set<string> {
  const visible = new Set<string>();

  for (const field of fields) {
    if (!field.conditionalLogic) {
      visible.add(field.id);
      continue;
    }

    const { action, combinator, conditions } = field.conditionalLogic;
    const conditionResults = conditions.map((cond) =>
      evaluateCondition(cond, values),
    );

    const allMatch =
      combinator === "AND"
        ? conditionResults.every(Boolean)
        : conditionResults.some(Boolean);

    if (action === "show") {
      if (allMatch) visible.add(field.id);
    } else {
      // action === "hide"
      if (!allMatch) visible.add(field.id);
    }
  }

  return visible;
}

function evaluateCondition(
  cond: ConditionalCondition,
  values: FormValues,
): boolean {
  const fieldValue = values[cond.fieldId];
  const rawValue = Array.isArray(fieldValue)
    ? fieldValue.join(", ")
    : String(fieldValue ?? "");

  switch (cond.operator) {
    case "equals":
      return rawValue === cond.value;
    case "not_equals":
      return rawValue !== cond.value;
    case "contains":
      return rawValue.toLowerCase().includes(cond.value.toLowerCase());
    case "greater_than":
      return Number(rawValue) > Number(cond.value);
    case "less_than":
      return Number(rawValue) < Number(cond.value);
    case "is_empty":
      return rawValue === "" || rawValue === "undefined";
    case "is_not_empty":
      return rawValue !== "" && rawValue !== "undefined";
    default:
      return true;
  }
}

function buildDefaults(fields: FormFieldSchema[]): FormValues {
  const defaults: FormValues = {};
  for (const field of fields) {
    switch (field.type) {
      case "checkbox":
        defaults[field.id] = [];
        break;
      case "rating":
        defaults[field.id] = "0";
        break;
      case "file_upload":
        defaults[field.id] = [];
        break;
      default:
        defaults[field.id] = "";
    }
  }
  return defaults;
}

// ---- Field Renderer (delegates to type-specific components) ----

function FieldRenderer({
  field,
  form,
  mode,
  disabled,
}: {
  field: FormFieldSchema;
  form: UseFormReturn<FormValues>;
  mode: "respond" | "edit";
  disabled?: boolean;
}) {
  const rules = mode === "respond" ? buildValidationRules(field) : undefined;

  switch (field.type) {
    case "section_header":
      return <h3 className="ff-section-header">{field.label}</h3>;
    case "text":
    case "email":
      return (
        <Controller
          name={field.id}
          control={form.control}
          rules={rules}
          render={({ field: f, fieldState }) => (
            <div>
              <input
                {...f}
                type={field.type}
                placeholder={field.placeholder}
                disabled={disabled}
                className={`ff-input ${fieldState.error ? "ff-input-error" : ""}`}
                value={f.value as string}
              />
              {fieldState.error && (
                <span className="ff-error">{fieldState.error.message}</span>
              )}
            </div>
          )}
        />
      );
    case "number":
      return (
        <Controller
          name={field.id}
          control={form.control}
          rules={rules}
          render={({ field: f, fieldState }) => (
            <div>
              <input
                {...f}
                type="number"
                placeholder={field.placeholder}
                min={field.validation?.min}
                max={field.validation?.max}
                disabled={disabled}
                className={`ff-input ${fieldState.error ? "ff-input-error" : ""}`}
                value={f.value as string}
              />
              {fieldState.error && (
                <span className="ff-error">{fieldState.error.message}</span>
              )}
            </div>
          )}
        />
      );
    case "textarea":
      return (
        <Controller
          name={field.id}
          control={form.control}
          rules={rules}
          render={({ field: f, fieldState }) => (
            <div>
              <textarea
                {...f}
                placeholder={field.placeholder}
                maxLength={field.validation?.maxLength}
                disabled={disabled}
                className={`ff-textarea ${fieldState.error ? "ff-input-error" : ""}`}
                value={f.value as string}
                rows={4}
              />
              {fieldState.error && (
                <span className="ff-error">{fieldState.error.message}</span>
              )}
            </div>
          )}
        />
      );
    case "dropdown":
      return (
        <Controller
          name={field.id}
          control={form.control}
          rules={rules}
          render={({ field: f, fieldState }) => (
            <div>
              <select
                {...f}
                disabled={disabled}
                className={`ff-select ${fieldState.error ? "ff-input-error" : ""}`}
                value={f.value as string}
              >
                <option value="">{field.placeholder || "Select..."}</option>
                {field.options?.map((opt) => (
                  <option key={opt.id} value={opt.value}>
                    {opt.label}
                  </option>
                ))}
              </select>
              {fieldState.error && (
                <span className="ff-error">{fieldState.error.message}</span>
              )}
            </div>
          )}
        />
      );
    case "radio":
      return (
        <Controller
          name={field.id}
          control={form.control}
          rules={rules}
          render={({ field: f, fieldState }) => (
            <div className="ff-radio-group">
              {field.options?.map((opt) => (
                <label key={opt.id} className="ff-radio-label">
                  <input
                    type="radio"
                    value={opt.value}
                    checked={f.value === opt.value}
                    onChange={f.onChange}
                    disabled={disabled}
                  />
                  <span>{opt.label}</span>
                </label>
              ))}
              {fieldState.error && (
                <span className="ff-error">{fieldState.error.message}</span>
              )}
            </div>
          )}
        />
      );
    case "rating":
      return (
        <Controller
          name={field.id}
          control={form.control}
          rules={rules}
          render={({ field: f }) => (
            <RatingInput
              value={Number(f.value) || 0}
              max={field.validation?.max ?? 5}
              onChange={(v) => f.onChange(String(v))}
              disabled={disabled}
            />
          )}
        />
      );
    default:
      return null;
  }
}

function buildValidationRules(field: FormFieldSchema) {
  const rules: Record<string, unknown> = {};

  if (field.required) {
    rules.required = `${field.label} is required`;
  }

  if (field.validation?.minLength) {
    rules.minLength = {
      value: field.validation.minLength,
      message: `Minimum ${field.validation.minLength} characters`,
    };
  }

  if (field.validation?.maxLength) {
    rules.maxLength = {
      value: field.validation.maxLength,
      message: `Maximum ${field.validation.maxLength} characters`,
    };
  }

  if (field.validation?.pattern) {
    rules.pattern = {
      value: new RegExp(field.validation.pattern),
      message: field.validation.patternMessage || "Invalid format",
    };
  }

  return rules;
}

function RatingInput({
  value,
  max,
  onChange,
  disabled,
}: {
  value: number;
  max: number;
  onChange: (v: number) => void;
  disabled?: boolean;
}) {
  return (
    <div className="ff-rating" role="radiogroup" aria-label="Rating">
      {Array.from({ length: max }, (_, i) => i + 1).map((star) => (
        <button
          key={star}
          type="button"
          className={`ff-star ${star <= value ? "ff-star-filled" : ""}`}
          onClick={() => !disabled && onChange(star)}
          disabled={disabled}
          aria-label={`${star} of ${max} stars`}
        >
          ★
        </button>
      ))}
    </div>
  );
}
```

### 3. Drag-and-Drop Form Editor with dnd-kit

The visual form editor uses @dnd-kit for drag-and-drop field reordering and palette-to-canvas field additions. It manages the form schema as local state, syncing to the server via tRPC mutations with debounced auto-save.

```typescript
// src/components/editor/form-editor.tsx

"use client";

import { useState, useCallback, useRef, useEffect } from "react";
import {
  DndContext,
  DragOverlay,
  closestCenter,
  KeyboardSensor,
  PointerSensor,
  useSensor,
  useSensors,
  type DragStartEvent,
  type DragEndEvent,
} from "@dnd-kit/core";
import {
  SortableContext,
  sortableKeyboardCoordinates,
  verticalListSortingStrategy,
  useSortable,
  arrayMove,
} from "@dnd-kit/sortable";
import { CSS } from "@dnd-kit/utilities";
import { randomUUID } from "crypto";
import { api } from "@/lib/trpc/client";
import type { FormFieldSchema, FieldType } from "@/server/db/schema";
import { FormRenderer } from "../form-renderer/form-renderer";
import { FieldPropertyPanel } from "./field-property-panel";
import { ConditionalLogicBuilder } from "./conditional-logic-builder";

// Available field types in the palette
const FIELD_PALETTE: { type: FieldType; label: string; icon: string }[] = [
  { type: "text", label: "Short Text", icon: "Aa" },
  { type: "email", label: "Email", icon: "@" },
  { type: "number", label: "Number", icon: "#" },
  { type: "textarea", label: "Long Text", icon: "¶" },
  { type: "dropdown", label: "Dropdown", icon: "▾" },
  { type: "radio", label: "Single Choice", icon: "◉" },
  { type: "checkbox", label: "Multiple Choice", icon: "☑" },
  { type: "date", label: "Date", icon: "📅" },
  { type: "rating", label: "Rating", icon: "★" },
  { type: "file_upload", label: "File Upload", icon: "📎" },
  { type: "section_header", label: "Section", icon: "─" },
];

interface FormEditorProps {
  formId: string;
  initialFields: FormFieldSchema[];
  theme: ThemeConfig;
}

export function FormEditor({ formId, initialFields, theme }: FormEditorProps) {
  const [fields, setFields] = useState<FormFieldSchema[]>(initialFields);
  const [selectedFieldId, setSelectedFieldId] = useState<string | null>(null);
  const [activeId, setActiveId] = useState<string | null>(null);
  const [undoStack, setUndoStack] = useState<FormFieldSchema[][]>([]);
  const [redoStack, setRedoStack] = useState<FormFieldSchema[][]>([]);

  const saveTimeoutRef = useRef<ReturnType<typeof setTimeout>>();
  const updateForm = api.form.updateSchema.useMutation();

  const sensors = useSensors(
    useSensor(PointerSensor, { activationConstraint: { distance: 5 } }),
    useSensor(KeyboardSensor, { coordinateGetter: sortableKeyboardCoordinates }),
  );

  // Debounced auto-save (save 1s after last change)
  const autoSave = useCallback(
    (newFields: FormFieldSchema[]) => {
      if (saveTimeoutRef.current) clearTimeout(saveTimeoutRef.current);
      saveTimeoutRef.current = setTimeout(() => {
        updateForm.mutate({ formId, schema: newFields });
      }, 1000);
    },
    [formId, updateForm],
  );

  // Undo/redo support
  const pushUndo = useCallback(
    (currentFields: FormFieldSchema[]) => {
      setUndoStack((prev) => [...prev.slice(-50), currentFields]);
      setRedoStack([]);
    },
    [],
  );

  const undo = useCallback(() => {
    setUndoStack((prev) => {
      if (prev.length === 0) return prev;
      const last = prev[prev.length - 1];
      setRedoStack((r) => [...r, fields]);
      setFields(last);
      autoSave(last);
      return prev.slice(0, -1);
    });
  }, [fields, autoSave]);

  const redo = useCallback(() => {
    setRedoStack((prev) => {
      if (prev.length === 0) return prev;
      const last = prev[prev.length - 1];
      setUndoStack((u) => [...u, fields]);
      setFields(last);
      autoSave(last);
      return prev.slice(0, -1);
    });
  }, [fields, autoSave]);

  // Keyboard shortcuts for undo/redo
  useEffect(() => {
    const handler = (e: KeyboardEvent) => {
      if ((e.metaKey || e.ctrlKey) && e.key === "z") {
        e.preventDefault();
        if (e.shiftKey) redo();
        else undo();
      }
    };
    window.addEventListener("keydown", handler);
    return () => window.removeEventListener("keydown", handler);
  }, [undo, redo]);

  // Add a new field from the palette
  const addField = useCallback(
    (type: FieldType) => {
      pushUndo(fields);
      const newField: FormFieldSchema = {
        id: randomUUID(),
        type,
        label: `New ${type.replace("_", " ")} field`,
        required: false,
        sortOrder: fields.length,
        options:
          type === "dropdown" || type === "radio" || type === "checkbox"
            ? [
                { id: randomUUID(), label: "Option 1", value: "option_1" },
                { id: randomUUID(), label: "Option 2", value: "option_2" },
              ]
            : undefined,
      };
      const updated = [...fields, newField];
      setFields(updated);
      setSelectedFieldId(newField.id);
      autoSave(updated);
    },
    [fields, pushUndo, autoSave],
  );

  // Delete a field
  const deleteField = useCallback(
    (fieldId: string) => {
      pushUndo(fields);
      const updated = fields
        .filter((f) => f.id !== fieldId)
        .map((f, i) => ({ ...f, sortOrder: i }));
      setFields(updated);
      if (selectedFieldId === fieldId) setSelectedFieldId(null);
      autoSave(updated);
    },
    [fields, selectedFieldId, pushUndo, autoSave],
  );

  // Duplicate a field
  const duplicateField = useCallback(
    (fieldId: string) => {
      pushUndo(fields);
      const original = fields.find((f) => f.id === fieldId);
      if (!original) return;

      const insertIndex = fields.findIndex((f) => f.id === fieldId) + 1;
      const duplicate: FormFieldSchema = {
        ...structuredClone(original),
        id: randomUUID(),
        label: `${original.label} (copy)`,
        sortOrder: insertIndex,
      };

      const updated = [
        ...fields.slice(0, insertIndex),
        duplicate,
        ...fields.slice(insertIndex),
      ].map((f, i) => ({ ...f, sortOrder: i }));

      setFields(updated);
      setSelectedFieldId(duplicate.id);
      autoSave(updated);
    },
    [fields, pushUndo, autoSave],
  );

  // Update field properties
  const updateField = useCallback(
    (fieldId: string, updates: Partial<FormFieldSchema>) => {
      pushUndo(fields);
      const updated = fields.map((f) =>
        f.id === fieldId ? { ...f, ...updates } : f,
      );
      setFields(updated);
      autoSave(updated);
    },
    [fields, pushUndo, autoSave],
  );

  // Drag handlers
  const handleDragStart = (event: DragStartEvent) => {
    setActiveId(event.active.id as string);
  };

  const handleDragEnd = (event: DragEndEvent) => {
    setActiveId(null);
    const { active, over } = event;
    if (!over || active.id === over.id) return;

    pushUndo(fields);
    const oldIndex = fields.findIndex((f) => f.id === active.id);
    const newIndex = fields.findIndex((f) => f.id === over.id);
    const reordered = arrayMove(fields, oldIndex, newIndex).map((f, i) => ({
      ...f,
      sortOrder: i,
    }));
    setFields(reordered);
    autoSave(reordered);
  };

  const selectedField = fields.find((f) => f.id === selectedFieldId);

  return (
    <div className="grid grid-cols-[240px_1fr_320px] h-full">
      {/* Left: Field Palette */}
      <div className="border-r p-4 space-y-2 overflow-y-auto">
        <h3 className="text-sm font-semibold text-gray-500 uppercase mb-3">
          Add Field
        </h3>
        {FIELD_PALETTE.map(({ type, label, icon }) => (
          <button
            key={type}
            onClick={() => addField(type)}
            className="w-full flex items-center gap-2 px-3 py-2 text-sm rounded-md
                       hover:bg-gray-100 text-left transition-colors"
          >
            <span className="w-6 text-center text-gray-400">{icon}</span>
            {label}
          </button>
        ))}
      </div>

      {/* Center: Form Canvas */}
      <div className="p-6 overflow-y-auto bg-gray-50">
        <div className="max-w-2xl mx-auto bg-white rounded-lg shadow-sm p-8">
          <DndContext
            sensors={sensors}
            collisionDetection={closestCenter}
            onDragStart={handleDragStart}
            onDragEnd={handleDragEnd}
          >
            <SortableContext
              items={fields.map((f) => f.id)}
              strategy={verticalListSortingStrategy}
            >
              <FormRenderer
                fields={fields}
                mode="edit"
                theme={theme}
                onFieldSelect={setSelectedFieldId}
                selectedFieldId={selectedFieldId ?? undefined}
              />
            </SortableContext>
            <DragOverlay>
              {activeId ? (
                <div className="bg-white p-3 rounded shadow-lg border opacity-80">
                  {fields.find((f) => f.id === activeId)?.label}
                </div>
              ) : null}
            </DragOverlay>
          </DndContext>
        </div>
      </div>

      {/* Right: Property Panel */}
      <div className="border-l overflow-y-auto">
        {selectedField ? (
          <FieldPropertyPanel
            field={selectedField}
            allFields={fields}
            onUpdate={(updates) => updateField(selectedField.id, updates)}
            onDelete={() => deleteField(selectedField.id)}
            onDuplicate={() => duplicateField(selectedField.id)}
          />
        ) : (
          <div className="p-6 text-center text-gray-400">
            <p className="text-sm">Select a field to edit its properties</p>
          </div>
        )}
      </div>
    </div>
  );
}

// Sortable field wrapper
function SortableField({
  id,
  children,
}: {
  id: string;
  children: React.ReactNode;
}) {
  const {
    attributes,
    listeners,
    setNodeRef,
    transform,
    transition,
    isDragging,
  } = useSortable({ id });

  const style = {
    transform: CSS.Transform.toString(transform),
    transition,
    opacity: isDragging ? 0.5 : 1,
  };

  return (
    <div ref={setNodeRef} style={style} {...attributes}>
      <div className="group relative">
        <button
          {...listeners}
          className="absolute -left-6 top-1/2 -translate-y-1/2 opacity-0 group-hover:opacity-100
                     cursor-grab active:cursor-grabbing p-1 text-gray-400"
          aria-label="Drag to reorder"
        >
          ⠿
        </button>
        {children}
      </div>
    </div>
  );
}
```

### 4. Hosted Form Page with Turnstile + Response Submission

The public form page is served via ISR at `/{org-slug}/{form-slug}`. It loads the form schema, renders it using the FormRenderer in respondent mode, validates the Turnstile challenge, and submits responses with file upload coordination.

```typescript
// src/app/f/[orgSlug]/[formSlug]/page.tsx

import { notFound } from "next/navigation";
import { db } from "@/server/db";
import { forms, organizations, themes } from "@/server/db/schema";
import { eq, and } from "drizzle-orm";
import { PublicFormClient } from "./client";

interface PageProps {
  params: { orgSlug: string; formSlug: string };
}

export async function generateMetadata({ params }: PageProps) {
  const form = await getForm(params.orgSlug, params.formSlug);
  if (!form) return { title: "Form not found" };

  return {
    title: form.title,
    description: form.description || `Fill out ${form.title}`,
    openGraph: {
      title: form.title,
      description: form.description || `Fill out ${form.title}`,
    },
  };
}

async function getForm(orgSlug: string, formSlug: string) {
  const result = await db
    .select({
      form: forms,
      orgName: organizations.name,
      orgSlug: organizations.slug,
      orgPlan: organizations.plan,
      orgBranding: organizations.branding,
      theme: themes,
    })
    .from(forms)
    .innerJoin(organizations, eq(forms.orgId, organizations.id))
    .leftJoin(themes, eq(forms.themeId, themes.id))
    .where(
      and(
        eq(organizations.slug, orgSlug),
        eq(forms.slug, formSlug),
        eq(forms.status, "published"),
      ),
    )
    .limit(1);

  return result[0] ?? null;
}

export default async function PublicFormPage({ params }: PageProps) {
  const data = await getForm(params.orgSlug, params.formSlug);
  if (!data) notFound();

  const { form, orgName, orgPlan, orgBranding, theme } = data;

  // Check form availability
  if (form.settings?.closeDate && new Date(form.settings.closeDate) < new Date()) {
    return <FormClosedMessage title={form.title} />;
  }

  if (form.settings?.responseLimit && form.responseCount >= form.settings.responseLimit) {
    return <FormClosedMessage title={form.title} message="This form has reached its response limit." />;
  }

  const themeConfig = theme?.colors
    ? {
        colors: theme.colors as Record<string, string>,
        fontFamily: theme.fontFamily || "Inter",
        borderRadius: theme.borderRadius || "8px",
        cssOverrides: theme.cssOverrides || undefined,
      }
    : DEFAULT_THEME;

  const showBranding = orgPlan === "free" || form.settings?.brandingVisible !== false;

  return (
    <PublicFormClient
      formId={form.id}
      title={form.title}
      description={form.description}
      fields={form.schema || []}
      theme={themeConfig}
      showBranding={showBranding}
      turnstileEnabled={form.settings?.turnstileEnabled ?? true}
      successMessage={form.settings?.successMessage}
      redirectUrl={form.settings?.redirectUrl}
      orgBranding={orgBranding}
    />
  );
}

// src/app/f/[orgSlug]/[formSlug]/client.tsx

"use client";

import { useState, useRef, useCallback } from "react";
import { Turnstile, type TurnstileInstance } from "@marsidev/react-turnstile";
import { FormRenderer } from "@/components/form-renderer/form-renderer";
import type { FormFieldSchema } from "@/server/db/schema";

interface PublicFormClientProps {
  formId: string;
  title: string;
  description: string | null;
  fields: FormFieldSchema[];
  theme: ThemeConfig;
  showBranding: boolean;
  turnstileEnabled: boolean;
  successMessage?: string;
  redirectUrl?: string;
  orgBranding: Record<string, string> | null;
}

export function PublicFormClient({
  formId,
  title,
  description,
  fields,
  theme,
  showBranding,
  turnstileEnabled,
  successMessage,
  redirectUrl,
  orgBranding,
}: PublicFormClientProps) {
  const [submitted, setSubmitted] = useState(false);
  const [error, setError] = useState<string | null>(null);
  const turnstileRef = useRef<TurnstileInstance>(null);
  const [turnstileToken, setTurnstileToken] = useState<string | null>(null);
  const startTimeRef = useRef<number>(Date.now());

  const handleSubmit = useCallback(
    async (values: Record<string, string | string[] | File[]>) => {
      setError(null);

      // Validate Turnstile
      if (turnstileEnabled && !turnstileToken) {
        setError("Please complete the verification challenge.");
        return;
      }

      const completionTimeSeconds = Math.round(
        (Date.now() - startTimeRef.current) / 1000,
      );

      // Step 1: Upload any files first
      const fileFieldIds = fields
        .filter((f) => f.type === "file_upload")
        .map((f) => f.id);

      const fileKeys: Record<string, string[]> = {};

      for (const fieldId of fileFieldIds) {
        const files = values[fieldId] as File[];
        if (!files?.length) continue;

        const keys: string[] = [];
        for (const file of files) {
          // Get presigned upload URL
          const res = await fetch("/api/upload/presign", {
            method: "POST",
            headers: { "Content-Type": "application/json" },
            body: JSON.stringify({
              formId,
              fileName: file.name,
              mimeType: file.type,
              sizeBytes: file.size,
            }),
          });

          if (!res.ok) throw new Error("Failed to get upload URL");
          const { uploadUrl, key } = await res.json();

          // Upload directly to R2
          await fetch(uploadUrl, {
            method: "PUT",
            body: file,
            headers: { "Content-Type": file.type },
          });

          keys.push(key);
        }
        fileKeys[fieldId] = keys;
      }

      // Step 2: Submit form response
      const response = await fetch("/api/forms/submit", {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify({
          formId,
          values: serializeValues(values, fileKeys, fields),
          turnstileToken,
          completionTimeSeconds,
          utmParams: getUtmParams(),
          metadata: {
            userAgent: navigator.userAgent,
            referrer: document.referrer,
            pageUrl: window.location.href,
          },
        }),
      });

      if (!response.ok) {
        const data = await response.json();
        setError(data.error || "Failed to submit form. Please try again.");
        turnstileRef.current?.reset();
        setTurnstileToken(null);
        return;
      }

      if (redirectUrl) {
        window.location.href = redirectUrl;
        return;
      }

      setSubmitted(true);
    },
    [formId, fields, turnstileEnabled, turnstileToken, redirectUrl],
  );

  if (submitted) {
    return (
      <div className="min-h-screen flex items-center justify-center p-4">
        <div className="text-center max-w-md">
          <div className="text-4xl mb-4">✓</div>
          <h2 className="text-xl font-semibold mb-2">Thank you!</h2>
          <p className="text-gray-600">
            {successMessage || "Your response has been recorded."}
          </p>
        </div>
      </div>
    );
  }

  return (
    <div className="min-h-screen flex items-center justify-center p-4 bg-gray-50">
      <div className="w-full max-w-2xl">
        {orgBranding?.logoUrl && (
          <img
            src={orgBranding.logoUrl}
            alt=""
            className="h-8 mb-4 mx-auto"
          />
        )}

        <div className="bg-white rounded-lg shadow-sm p-8">
          <h1 className="text-2xl font-bold mb-2">{title}</h1>
          {description && (
            <p className="text-gray-600 mb-6">{description}</p>
          )}

          <FormRenderer
            fields={fields}
            mode="respond"
            theme={theme}
            onSubmit={handleSubmit}
          />

          {turnstileEnabled && (
            <div className="mt-4">
              <Turnstile
                ref={turnstileRef}
                siteKey={process.env.NEXT_PUBLIC_TURNSTILE_SITE_KEY!}
                onSuccess={setTurnstileToken}
                onExpire={() => setTurnstileToken(null)}
              />
            </div>
          )}

          {error && (
            <div className="mt-4 p-3 bg-red-50 text-red-700 rounded text-sm">
              {error}
            </div>
          )}
        </div>

        {showBranding && (
          <p className="text-center text-xs text-gray-400 mt-4">
            Powered by FormForge
          </p>
        )}
      </div>
    </div>
  );
}

function serializeValues(
  values: Record<string, string | string[] | File[]>,
  fileKeys: Record<string, string[]>,
  fields: FormFieldSchema[],
): Record<string, { value?: string; fileKeys?: string[] }> {
  const serialized: Record<string, { value?: string; fileKeys?: string[] }> = {};

  for (const field of fields) {
    const val = values[field.id];
    if (field.type === "file_upload") {
      serialized[field.id] = { fileKeys: fileKeys[field.id] || [] };
    } else if (Array.isArray(val)) {
      serialized[field.id] = { value: (val as string[]).join(", ") };
    } else {
      serialized[field.id] = { value: val as string };
    }
  }

  return serialized;
}

function getUtmParams() {
  const params = new URLSearchParams(window.location.search);
  return {
    source: params.get("utm_source") || undefined,
    medium: params.get("utm_medium") || undefined,
    campaign: params.get("utm_campaign") || undefined,
    term: params.get("utm_term") || undefined,
    content: params.get("utm_content") || undefined,
  };
}
```

---

## Phase Breakdown

### Phase 1: Project Scaffold + Database (Days 1–5)

**Day 1 — Initialize project**
```bash
npx create-next-app@latest formforge --typescript --tailwind --eslint --app --src-dir
cd formforge
npm install drizzle-orm @neondatabase/serverless
npm install -D drizzle-kit
npm install @trpc/server @trpc/client @trpc/react-query @trpc/next superjson
npm install @clerk/nextjs
npm install zod
```
- Configure `tsconfig.json`: strict mode, path aliases (`@/*` → `src/*`)
- Create `src/env.ts` with Zod validation for environment variables:
  - `DATABASE_URL`, `CLERK_SECRET_KEY`, `NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY`
  - `OPENAI_API_KEY`, `STRIPE_SECRET_KEY`, `STRIPE_WEBHOOK_SECRET`
  - `R2_ACCESS_KEY_ID`, `R2_SECRET_ACCESS_KEY`, `R2_BUCKET_NAME`, `R2_ENDPOINT`
  - `NEXT_PUBLIC_TURNSTILE_SITE_KEY`, `TURNSTILE_SECRET_KEY`
  - `RESEND_API_KEY`

**Day 2 — Database schema + connection**
- Create `src/server/db/schema.ts` with all tables: `organizations`, `members`, `themes`, `forms`, `formResponses`, `fieldResponses`, `formIntegrations`, `formViews`, `fileUploads`, `notificationLog`
- Create `src/server/db/index.ts`: Neon serverless client + Drizzle instance
- Create `drizzle.config.ts` pointing to `DATABASE_URL`
- Run `npx drizzle-kit push` to sync schema
- Create `migrations/0000_init.sql` to seed 3 system themes

**Day 3 — tRPC setup + Clerk auth**
- Create `src/server/trpc/trpc.ts`: base procedures with Clerk auth context
  - `publicProcedure`: no auth required
  - `protectedProcedure`: requires Clerk session, resolves `orgId`
  - `orgProcedure`: extends protected, resolves organization record + plan limits
- Create `src/server/trpc/context.ts`: extract auth from request headers
- Create `src/server/trpc/routers/_app.ts`: merged router
- Create `src/app/api/trpc/[trpc]/route.ts`: HTTP handler
- Create `src/lib/trpc/client.ts`: React Query tRPC client
- Create `src/lib/trpc/server.ts`: Server-side caller

**Day 4 — Clerk middleware + app shell**
- Create `src/middleware.ts`: protect `/dashboard/*` routes, allow public form routes
- Create `src/app/layout.tsx`: root layout with `ClerkProvider`, TRPCProvider
- Create `src/app/dashboard/layout.tsx`: app shell with sidebar + header
- Install Radix UI primitives:
  ```bash
  npm install @radix-ui/react-dialog @radix-ui/react-dropdown-menu @radix-ui/react-tabs
  npm install @radix-ui/react-select @radix-ui/react-tooltip @radix-ui/react-popover
  npm install @radix-ui/react-toast @radix-ui/react-switch @radix-ui/react-label
  ```
- Create shared UI components in `src/components/ui/`: button, card, badge, input, textarea, select, dialog, dropdown-menu, tabs, toast, skeleton, tooltip

**Day 5 — Organization + member sync**
- Create `src/server/trpc/routers/organization.ts`:
  - `getOrCreate`: sync Clerk org → local DB on first load
  - `update`: update org name, slug, branding, settings
  - `getBySlug`: public org lookup for form pages
- Create Clerk webhook handler `src/app/api/webhooks/clerk/route.ts`:
  - `organization.created` → create org record
  - `organizationMembership.created` → create member record
  - `organizationMembership.deleted` → delete member record
- Test: sign up → create org → verify DB records

---

### Phase 2: AI Form Generation (Days 6–10)

**Day 6 — AI generation engine**
- Create `src/server/ai/generate-form.ts`:
  - `generateFormSchema(prompt)`: two-step GPT-4o pipeline
    - Step 1: GPT-4o-mini generates title + description
    - Step 2: GPT-4o generates field schema array (JSON mode)
  - `normalizeAIFields(rawFields)`: validate types, assign UUIDs, resolve conditional refs
  - `normalizeValidation(type, raw)`: apply type-specific validation defaults
- Create TypeScript types for `FormFieldSchema`, `FieldType`, `FieldOption`, `FieldValidation`, `ConditionalLogic`, `ConditionalCondition` (in schema.ts as exports)

**Day 7 — AI generation tRPC router**
- Create `src/server/trpc/routers/ai.ts`:
  - `generateForm`: accepts prompt string, returns `{ title, description, fields }`
  - Input validation: prompt must be 10-2000 characters
  - Plan enforcement: free tier gets 3 AI generations/day, pro unlimited
- Create `src/app/dashboard/forms/new/page.tsx`:
  - Large textarea: "Describe the form you want to create..."
  - Example prompts carousel below the textarea
  - "Generate" button with loading spinner
  - Split view: description on left, generated form preview on right
  - "Use this form" → creates form in DB + redirects to editor
  - "Regenerate" → re-runs AI with same prompt
  - "Start from scratch" → redirects to blank editor

**Day 8 — Form CRUD router**
- Create `src/server/trpc/routers/form.ts`:
  - `list`: paginated forms for org, filter by status
  - `getById`: single form with response count
  - `create`: create from AI output or blank template
  - `updateSchema`: update form schema (field array) — used by auto-save
  - `updateSettings`: update form settings (notifications, limits, etc.)
  - `updateMeta`: update title, description, slug
  - `publish`: set status to published, set publishedAt
  - `unpublish`: revert to draft
  - `close`: set status to closed
  - `delete`: soft delete (set status to closed) or hard delete
  - `duplicate`: deep copy form with "(copy)" suffix

**Day 9 — Form list page + dashboard overview**
- Create `src/app/dashboard/page.tsx`:
  - Total forms count, total responses (this month), recent activity
  - Quick action: "Create new form" button
- Create `src/app/dashboard/forms/page.tsx`:
  - Card grid of forms with: title, status badge, response count, last updated
  - Search bar
  - Filter tabs: All, Published, Draft, Closed
  - Click card → navigate to editor
  - Three-dot menu: Edit, Duplicate, Delete, View responses
  - "New form" button → `/dashboard/forms/new`

**Day 10 — AI prompt refinement + edge cases**
- Test AI generation with 20+ diverse prompts:
  - Simple: "Contact form with name, email, message"
  - Complex: "Job application with resume upload, if role is engineering show coding language dropdown"
  - Edge cases: non-English prompts, very long descriptions, adversarial inputs
- Add error handling for OpenAI API failures (retry with exponential backoff)
- Add prompt sanitization (strip HTML, limit length)
- Cache generation results for 5 minutes (avoid duplicate charges on page refresh)

---

### Phase 3: Visual Form Editor (Days 11–17)

**Day 11 — dnd-kit setup + sortable fields**
```bash
npm install @dnd-kit/core @dnd-kit/sortable @dnd-kit/utilities
```
- Create `src/components/editor/form-editor.tsx`:
  - Three-column layout: palette | canvas | property panel
  - DndContext with SortableContext for field reordering
  - `arrayMove` on drag end to reorder fields
  - Auto-save via debounced tRPC mutation (1s after last change)

**Day 12 — Field palette + add/delete/duplicate**
- Create field palette component with all 10 field types + section header
- Click palette item → append new field with sensible defaults
- Delete button on each field (with confirmation for fields that have responses)
- Duplicate button → deep clone with new UUID
- Undo/redo stack (50 levels) with Cmd+Z / Cmd+Shift+Z

**Day 13 — Field property panel**
- Create `src/components/editor/field-property-panel.tsx`:
  - Label (text input, required)
  - Placeholder (text input)
  - Help text (text input)
  - Required toggle (switch)
  - Field type selector (can change type, preserving label)
  - Type-specific settings:
    - Dropdown/Radio/Checkbox: option list with add/remove/reorder
    - Number: min/max inputs
    - Text/Textarea: minLength/maxLength
    - File upload: accepted types, max size, max count
    - Rating: max stars (3-10)
    - Date: min/max date
  - Delete field button
  - Duplicate field button

**Day 14 — Option editor for dropdown/radio/checkbox**
- Create `src/components/editor/option-editor.tsx`:
  - Inline editing of option labels
  - Drag-to-reorder options
  - Add option button
  - Delete option button (min 1 option)
  - Auto-generate value from label (slugified)
  - Bulk add: paste multi-line text → each line becomes an option

**Day 15 — Conditional logic builder**
- Create `src/components/editor/conditional-logic-builder.tsx`:
  - Toggle: "Add conditional logic" switch
  - Action selector: "Show this field when..." or "Hide this field when..."
  - Condition rows:
    - Field selector (dropdown of all other fields)
    - Operator selector (equals, not_equals, contains, etc.)
    - Value input (text input, or option picker for dropdown/radio source fields)
  - Combinator toggle: AND / OR (shown when 2+ conditions)
  - Add condition button (max 5 conditions)
  - Remove condition button
  - Preview button: highlights which fields this depends on

**Day 16 — Form preview modes**
- Create `src/components/editor/preview-toolbar.tsx`:
  - Device toggle: Desktop (default) / Tablet / Mobile
  - Preview button: opens form in new tab as respondent would see it
  - Theme selector dropdown (3 system themes + any custom)
  - "Share preview link" → generates temporary preview URL
- Implement responsive preview:
  - Desktop: full width
  - Tablet: max-w-lg with border frame
  - Mobile: max-w-sm with phone frame border

**Day 17 — Auto-save + save status indicator**
- Implement save state machine: `saved` → `unsaved` → `saving` → `saved`
- Show status in editor toolbar: "All changes saved" / "Saving..." / "Unsaved changes"
- Warn on navigation if unsaved changes (beforeunload event)
- Handle optimistic updates: show changes immediately, revert on save failure
- Handle concurrent editing: last-write-wins with `updatedAt` check
- Test: rapid field additions/deletions → verify no race conditions in auto-save

---

### Phase 4: Form Renderer + Public Pages (Days 18–24)

**Day 18 — Form renderer component**
- Create `src/components/form-renderer/form-renderer.tsx`:
  - Accept `fields`, `mode`, `theme`, `onSubmit` props
  - Render all 10 field types with proper HTML inputs
  - Integrate `react-hook-form` for validation
  ```bash
  npm install react-hook-form
  ```
  - Build validation rules from field schema
  - Handle required fields, pattern validation, min/max

**Day 19 — Conditional logic evaluation**
- Implement `evaluateVisibility(fields, values)`:
  - Watch all form values via `form.watch()`
  - For each field with `conditionalLogic`, evaluate conditions against current values
  - Return `Set<string>` of visible field IDs
  - Skip hidden field values on submission
- Implement each operator: equals, not_equals, contains, greater_than, less_than, is_empty, is_not_empty
- Handle circular dependencies gracefully (field A shows based on B, B shows based on A → both hidden)

**Day 20 — Theme system + CSS variables**
- Create `src/lib/theme.ts`:
  - `themeToCSS(theme)`: convert theme config to CSS custom properties
  - Default theme fallback
- Create form renderer CSS using CSS custom properties (`--ff-primary`, `--ff-bg`, etc.)
- Test all 3 system themes with a complex form (10+ fields, mixed types)
- Support `cssOverrides` from theme config (injected as `<style>` tag)

**Day 21 — Public form page (ISR)**
- Create `src/app/f/[orgSlug]/[formSlug]/page.tsx`:
  - Server component: fetch form + org + theme from DB
  - Check form status (must be "published")
  - Check availability constraints (closeDate, responseLimit)
  - Render `PublicFormClient` with form data
  - SEO metadata: title, description, OG tags
- Configure ISR: revalidate on form publish/unpublish via `revalidateTag`

**Day 22 — Turnstile integration**
```bash
npm install @marsidev/react-turnstile
```
- Add Turnstile widget to public form page (invisible mode)
- Create `src/app/api/forms/verify-turnstile.ts`:
  - Server-side verification against Cloudflare API
  - Return 403 if verification fails
- Enable/disable Turnstile per form in settings
- Skip Turnstile in development mode

**Day 23 — Form submission API**
- Create `src/app/api/forms/submit/route.ts` (public, no auth):
  - Validate Turnstile token (server-side)
  - Validate form exists and is published
  - Check response limit not exceeded
  - Validate required fields present
  - Create `formResponse` record
  - Create `fieldResponse` records for each field
  - Increment `forms.responseCount`
  - Track `formViews.completed` for analytics
  - Trigger email notifications (async)
  - Return 200 with success message
- Rate limit: 10 submissions per IP per minute per form

**Day 24 — Form view tracking**
- Create `src/app/api/forms/track/route.ts` (public, no auth):
  - Record form view in `formViews` table
  - Track: visitorId (fingerprint), referrer, UTM source
  - Debounce: one view per visitorId per form per hour
  - Track "started" when first field is interacted with
- Add client-side tracking script to public form page:
  - Send view event on page load
  - Send "started" event on first field focus
  - Use `navigator.sendBeacon` for reliability

---

### Phase 5: Response Management (Days 25–30)

**Day 25 — Response list page**
- Create `src/server/trpc/routers/response.ts`:
  - `list`: paginated responses for a form, filter by status, sort by date
  - `getById`: single response with all field responses
  - `updateStatus`: mark as read/starred/archived
  - `bulkUpdateStatus`: bulk status changes
  - `delete`: delete response + associated field responses
  - `bulkDelete`: bulk delete
  - `export`: generate CSV/JSON of all responses
- Create `src/app/dashboard/forms/[formId]/responses/page.tsx`:
  - Table view: columns = form fields, rows = responses
  - Truncate long text values in cells
  - Status column with colored badges (new, read, starred, archived)
  - Click row to expand response detail
  - Checkbox column for bulk actions

**Day 26 — Response detail view**
- Create `src/app/dashboard/forms/[formId]/responses/[responseId]/page.tsx`:
  - Full response display: field label + value pairs
  - File upload fields: show file name + download link (presigned URL)
  - Rating fields: show star display
  - Metadata section: submitted at, completion time, IP, user agent, UTM params
  - Status actions: Mark as read, Star, Archive
  - Previous/Next navigation between responses
  - Delete button with confirmation

**Day 27 — Response export**
- Implement CSV export:
  - Headers = field labels
  - Rows = response values
  - Handle multi-value fields (checkbox → comma-separated)
  - File upload fields → R2 URLs
  - Include metadata columns: submitted_at, respondent_email, completion_time
- Implement JSON export:
  - Array of response objects with field labels as keys
- Download via streaming response for large datasets
- Plan limit: free tier exports max 100 responses, pro unlimited

**Day 28 — Response summary / mini-analytics**
- Create response summary panel on form responses page:
  - Total responses, responses today, average completion time
  - Per-field summaries:
    - Dropdown/Radio: pie chart of option distribution
    - Rating: average rating + distribution bar
    - Number: average, min, max
    - Text/Email: word cloud (v2) or just count
  - Completion rate: views → starts → completions funnel
- Use Recharts for visualization:
  ```bash
  npm install recharts
  ```

**Day 29 — Email notifications on submission**
- Create `src/server/email/templates.ts`:
  - `submissionAlert`: email to form owner with response summary
  - Template includes: form title, respondent email (if collected), first 5 field values, link to view full response
- Create `src/server/services/notifications.ts`:
  - `sendSubmissionAlert(formId, responseId)`: send to all emails in `form.settings.notificationEmails`
  - Create `notificationLog` record for tracking
  - Handle Resend API errors with retry
- Integrate with form submission API: fire async after response is stored

**Day 30 — Respondent confirmation email**
- If form has an email field and respondent provided email:
  - Send confirmation email: "Your response to {form title} has been received"
  - Include response summary in email
  - Configurable per form in settings (off by default)
- Create email template: clean, matches form branding (logo, colors)

---

### Phase 6: File Uploads + Storage (Days 31–33)

**Day 31 — R2 presigned URL generation**
```bash
npm install @aws-sdk/client-s3 @aws-sdk/s3-request-presigner
```
- Create `src/server/services/storage.ts`:
  - `generatePresignedUploadUrl(formId, fileName, mimeType)`: generate PUT presigned URL
  - Key format: `uploads/{orgId}/{formId}/{uuid}/{fileName}`
  - URL expiry: 15 minutes
  - `generatePresignedDownloadUrl(key)`: generate GET presigned URL (1 hour expiry)
  - `deleteObject(key)`: delete from R2
- Create `src/app/api/upload/presign/route.ts`:
  - Accept: formId, fileName, mimeType, sizeBytes
  - Validate: form exists, file type allowed, file size within limits
  - Check org storage quota (free: 100MB, pro: 1GB, business: 10GB)
  - Create `fileUpload` record with `isAttached: false`
  - Return presigned URL + key

**Day 32 — File upload field component**
- Create `src/components/form-renderer/file-upload-field.tsx`:
  - Drag-and-drop zone with click-to-browse fallback
  - File type filter based on field validation
  - Progress bar per file
  - Image preview for image uploads
  - File size validation (client-side, before upload)
  - Multi-file support (up to maxFileCount)
  - Remove uploaded file button
- Integrate with form renderer: file upload fields store R2 keys in form state

**Day 33 — Storage management + cleanup**
- Create daily cleanup cron (`src/server/cron/cleanup-uploads.ts`):
  - Find `fileUploads` where `isAttached = false` and `createdAt < 24 hours ago`
  - Delete from R2 + delete DB records
  - Log cleanup results
- Mark files as attached on form submission (update `isAttached = true`, set `responseId`)
- Create storage usage dashboard in org settings:
  - Total storage used / quota
  - Usage by form
  - Warning when approaching quota (>80%)

---

### Phase 7: Billing + Settings (Days 34–37)

**Day 34 — Stripe integration**
```bash
npm install stripe
```
- Create `src/server/billing/stripe.ts`:
  - Stripe client initialization
  - Plan definitions: Free (no charge), Pro ($15/mo), Business ($39/mo)
  - `createCheckoutSession(orgId, plan)`: generate Stripe Checkout URL
  - `createPortalSession(stripeCustomerId)`: generate Customer Portal URL
  - Plan limits:
    ```typescript
    const PLAN_LIMITS = {
      free: { maxForms: 3, maxResponsesPerMonth: 100, maxStorageMb: 100, aiGenerationsPerDay: 3, brandingRequired: true },
      pro: { maxForms: Infinity, maxResponsesPerMonth: Infinity, maxStorageMb: 1024, aiGenerationsPerDay: Infinity, brandingRequired: false },
      business: { maxForms: Infinity, maxResponsesPerMonth: Infinity, maxStorageMb: 10240, aiGenerationsPerDay: Infinity, brandingRequired: false },
    };
    ```

**Day 35 — Stripe webhook handler**
- Create `src/app/api/webhooks/stripe/route.ts`:
  - Verify webhook signature
  - Handle events:
    - `checkout.session.completed`: update org plan + store stripeCustomerId/subscriptionId
    - `customer.subscription.updated`: sync plan changes
    - `customer.subscription.deleted`: revert to free plan
  - Idempotent: check subscription metadata before applying

**Day 36 — Billing UI + plan enforcement**
- Create `src/server/trpc/routers/billing.ts`:
  - `getCurrentPlan`: return plan name, limits, current usage
  - `createCheckout`: generate Stripe Checkout URL
  - `createPortal`: generate Customer Portal URL
- Create `src/app/dashboard/settings/billing/page.tsx`:
  - Current plan card with usage meters
  - Three plan cards with feature comparison
  - Upgrade/Manage buttons
- Implement plan enforcement middleware in tRPC:
  - Form count limit on `form.create`
  - Response count limit on submission API
  - Storage limit on presigned URL generation
  - AI generation daily limit

**Day 37 — Settings pages**
- Create `src/app/dashboard/settings/page.tsx`:
  - Organization name and slug
  - Logo upload (to R2)
  - Brand colors picker
  - Default theme selector
- Create `src/app/dashboard/settings/members/page.tsx`:
  - Team member list from Clerk org
  - Invite member form
  - Role badges
  - Remove member button
- Create form-level settings page `src/app/dashboard/forms/[formId]/settings/page.tsx`:
  - General: title, description, slug
  - Behavior: success message, redirect URL, response limit, close date
  - Notifications: email recipients list
  - Compliance: GDPR consent toggle + text
  - Embed: copy iframe embed code + direct link
  - Danger zone: close form, delete form

---

### Phase 8: Polish + Launch (Days 38–42)

**Day 38 — Onboarding flow**
- Create `src/app/onboarding/page.tsx`:
  - Step 1: Create/name organization, upload logo
  - Step 2: Describe your first form in plain English (AI generation)
  - Step 3: Review generated form → "Edit" or "Use this"
  - Step 4: Publish form → "Your form is live!" with shareable link
  - Progress indicator (4 steps)
  - Skip option to go straight to dashboard
- Store onboarding completion in `organizations.settings`
- Redirect new orgs to onboarding

**Day 39 — Loading states, error handling, empty states**
- Add Suspense boundaries with skeleton loaders for all data-fetching pages
- Create empty state components:
  - Forms empty: "Create your first form" with AI prompt suggestion
  - Responses empty: "No responses yet — share your form to start collecting"
  - No published forms: "Publish a form to start receiving responses"
- Toast notifications for all mutations (save, publish, delete)
- Global error boundary with retry
- tRPC error mapping to user-friendly messages

**Day 40 — Responsive design + accessibility**
- Mobile-responsive sidebar: collapsible, bottom nav on mobile
- Form editor: single-column layout on mobile (palette → canvas → panel stacked)
- Response table: horizontal scroll on mobile
- All interactive elements: focus rings, keyboard navigation, ARIA labels
- Color contrast verification for all themes
- Screen reader text for icon-only buttons
- Tab order on form renderer matches visual order

**Day 41 — Performance + SEO**
- Add `loading.tsx` files for route-level streaming
- Optimize database queries: ensure all list queries use indexed columns
- React Query settings:
  - Forms list: staleTime 30s
  - Form editor schema: staleTime Infinity (manual invalidation)
  - Responses: staleTime 10s
- Server-side rendering for dashboard pages
- Public form pages: full SSR + ISR (revalidate on publish)
- Landing page SEO: meta tags, Open Graph, JSON-LD structured data

**Day 42 — Landing page + final testing**
- Create `src/app/page.tsx`:
  - Hero: "Describe a form in plain English. Get it in seconds." with CTA
  - Demo: animated GIF/video of AI generating a form
  - How it works: 3-step visual (Describe → Customize → Publish)
  - Features grid: AI Generation, Visual Editor, Conditional Logic, File Uploads, Themes
  - Pricing table with three tiers
  - Template gallery preview
  - Footer
- Create `src/app/pricing/page.tsx`: detailed pricing comparison
- End-to-end test:
  1. Sign up → Create org → Onboarding
  2. Generate form with AI → Verify fields generated correctly
  3. Edit form: reorder fields, add conditional logic, change theme
  4. Publish form → Visit public URL → Submit response
  5. Verify response appears in dashboard
  6. Verify email notification sent
  7. Upload file in form → Verify stored in R2
  8. Export responses to CSV → Verify format
  9. Upgrade plan via Stripe (test mode)
  10. Verify plan limits enforced
- Deploy to Vercel + configure custom domain
- Set up Sentry for error tracking
- Register Stripe webhook endpoint

---

## Critical Files

```
formforge/
├── src/
│   ├── env.ts                                    # Zod-validated environment variables
│   ├── middleware.ts                              # Clerk auth middleware, route protection
│   │
│   ├── app/
│   │   ├── layout.tsx                            # Root layout with ClerkProvider
│   │   ├── page.tsx                              # Landing page
│   │   ├── pricing/page.tsx                      # Pricing page
│   │   ├── onboarding/page.tsx                   # 4-step onboarding wizard
│   │   │
│   │   ├── dashboard/
│   │   │   ├── layout.tsx                        # App shell: sidebar + header
│   │   │   ├── page.tsx                          # Dashboard overview
│   │   │   ├── forms/
│   │   │   │   ├── page.tsx                      # Form list (card grid)
│   │   │   │   ├── new/page.tsx                  # AI form generation page
│   │   │   │   └── [formId]/
│   │   │   │       ├── page.tsx                  # Form editor (dnd-kit canvas)
│   │   │   │       ├── settings/page.tsx         # Form settings
│   │   │   │       └── responses/
│   │   │   │           ├── page.tsx              # Response table + summary
│   │   │   │           └── [responseId]/page.tsx # Response detail
│   │   │   └── settings/
│   │   │       ├── page.tsx                      # Organization settings
│   │   │       ├── billing/page.tsx              # Plan + usage + upgrade
│   │   │       └── members/page.tsx              # Team member management
│   │   │
│   │   ├── f/
│   │   │   └── [orgSlug]/
│   │   │       └── [formSlug]/
│   │   │           ├── page.tsx                  # Public form page (ISR)
│   │   │           └── client.tsx                # Client-side form + Turnstile
│   │   │
│   │   └── api/
│   │       ├── trpc/[trpc]/route.ts              # tRPC HTTP handler
│   │       ├── forms/
│   │       │   ├── submit/route.ts               # Public form submission endpoint
│   │       │   └── track/route.ts                # Form view tracking
│   │       ├── upload/
│   │       │   └── presign/route.ts              # R2 presigned URL generation
│   │       └── webhooks/
│   │           ├── clerk/route.ts                # Clerk org/member sync
│   │           └── stripe/route.ts               # Stripe webhook handler
│   │
│   ├── server/
│   │   ├── db/
│   │   │   ├── schema.ts                        # Drizzle schema (all tables + relations + types)
│   │   │   ├── index.ts                         # Neon database client
│   │   │   └── migrations/
│   │   │       └── 0000_init.sql                # Seed system themes
│   │   │
│   │   ├── trpc/
│   │   │   ├── trpc.ts                          # tRPC init, auth/org/plan middleware
│   │   │   ├── context.ts                       # Request context (auth + db)
│   │   │   └── routers/
│   │   │       ├── _app.ts                      # Merged app router
│   │   │       ├── organization.ts              # Org CRUD + branding
│   │   │       ├── form.ts                      # Form CRUD + schema updates
│   │   │       ├── ai.ts                        # AI form generation
│   │   │       ├── response.ts                  # Response list + detail + export
│   │   │       └── billing.ts                   # Plan info + Stripe checkout/portal
│   │   │
│   │   ├── ai/
│   │   │   └── generate-form.ts                 # GPT-4o form generation + schema normalization
│   │   │
│   │   ├── services/
│   │   │   ├── storage.ts                       # R2 presigned URLs + file management
│   │   │   └── notifications.ts                 # Resend email dispatch for submissions
│   │   │
│   │   ├── billing/
│   │   │   └── stripe.ts                        # Plan definitions, checkout, portal
│   │   │
│   │   ├── email/
│   │   │   └── templates.ts                     # HTML email templates
│   │   │
│   │   └── cron/
│   │       └── cleanup-uploads.ts               # Daily orphaned file cleanup
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
│   │   │   ├── switch.tsx
│   │   │   └── popover.tsx
│   │   │
│   │   ├── layout/
│   │   │   ├── sidebar.tsx                      # Navigation sidebar
│   │   │   ├── header.tsx                       # Page header with breadcrumbs
│   │   │   └── org-switcher.tsx                 # Clerk org switcher wrapper
│   │   │
│   │   ├── editor/
│   │   │   ├── form-editor.tsx                  # Main editor: palette + canvas + panel
│   │   │   ├── field-property-panel.tsx          # Right panel: field config
│   │   │   ├── option-editor.tsx                # Option list editor for dropdown/radio/checkbox
│   │   │   ├── conditional-logic-builder.tsx     # Visual condition builder
│   │   │   ├── preview-toolbar.tsx              # Device toggle + theme selector
│   │   │   └── save-status.tsx                  # Auto-save indicator
│   │   │
│   │   ├── form-renderer/
│   │   │   ├── form-renderer.tsx                # Universal renderer (edit + respond modes)
│   │   │   ├── field-wrapper.tsx                # Label + help text + error wrapper
│   │   │   ├── file-upload-field.tsx            # Drag-and-drop file upload
│   │   │   ├── rating-input.tsx                 # Star rating component
│   │   │   └── theme.css                        # CSS with custom properties
│   │   │
│   │   └── responses/
│   │       ├── response-table.tsx               # DataTable with sortable columns
│   │       ├── response-detail.tsx              # Full response display
│   │       ├── response-summary.tsx             # Charts + stats
│   │       └── export-dialog.tsx                # CSV/JSON export options
│   │
│   └── lib/
│       ├── trpc/
│       │   ├── client.ts                        # React Query tRPC client
│       │   └── server.ts                        # Server-side tRPC caller
│       ├── utils.ts                             # cn() helper, date formatting, slug generation
│       └── constants.ts                         # Field type labels, icons, plan limits
│
├── drizzle.config.ts
├── tailwind.config.ts
├── tsconfig.json
├── next.config.ts
├── package.json
└── .env.local
```

---

## Verification / Testing

### QA Checklist

This comprehensive checklist covers all critical testing areas. Each item should be verified before launch.

1. **Unit tests for form logic** — validate field schema parsing, default value injection, required-field detection, sort order normalization, and field ID uniqueness enforcement. Cover all `FieldType` variants including edge cases (empty options arrays, zero-length validation ranges).
2. **Integration tests for form submissions** — end-to-end submission pipeline: POST form data to public endpoint, verify `formResponses` and `fieldResponses` rows created with correct field-to-value mapping, response count incremented, and notification job enqueued.
3. **E2E tests for form builder** — Playwright tests covering: open editor, add 5 field types via drag-and-drop from palette, reorder fields, edit labels/placeholders, toggle required, save, publish, visit public URL, fill out form, submit, verify response appears in dashboard.
4. **Accessibility testing** — run axe-core on the public form page and form builder. Verify: all inputs have associated labels, focus order is logical, keyboard navigation works for all field types (including drag-and-drop via keyboard), color contrast meets WCAG AA, screen reader announces field errors.
5. **Mobile responsiveness** — test public form and form builder on iPhone SE (375px), iPhone 14 (390px), iPad (768px), and desktop (1280px+). Verify: fields stack vertically on mobile, file upload tap targets are 44px minimum, rating stars are tappable, drag-and-drop has touch fallback in editor.
6. **Conditional logic validation** — create forms with conditional show/hide rules: single condition, multiple AND conditions, multiple OR conditions, nested dependencies (field A shows B which shows C), circular dependency detection (A depends on B depends on A — both should hide gracefully).
7. **File upload limits** — verify: oversized files rejected client-side before upload, wrong MIME types rejected, max file count enforced per field, presigned URL expires after 15 minutes, orphaned uploads cleaned up by daily cron after 24 hours, total org storage limit enforced.
8. **Payment webhook testing** — (for future Stripe payment fields) verify webhook signature validation, idempotent processing of duplicate events, correct status updates on payment success/failure.
9. **Email notification delivery** — verify: owner notification sent on submission with correct form title and response preview, respondent confirmation email sent when enabled, Resend message ID stored in notification_log, failed sends recorded with error message.
10. **Analytics accuracy** — verify: view count increments on page load (not on bot visits), response count matches actual formResponses rows, completion rate = (submitted views / total views) * 100, UTM parameters captured from query string.
11. **API endpoint security** — verify: all tRPC mutations require Clerk auth, public form submission endpoint validates Turnstile token, form retrieval only returns published forms for public access, org-scoping prevents cross-org data access on every query.
12. **Rate limiting** — verify: public form submission endpoint rate limited to 10/minute per IP, AI generation endpoint rate limited to 5/minute per user, file upload presigned URL generation limited to 20/minute per user.
13. **CSRF protection** — verify: Turnstile token required on all public form submissions, Clerk CSRF protection active on authenticated endpoints, webhook endpoints validate signatures.
14. **Form embedding cross-origin** — verify: embedded iframe form renders correctly on external domains, `X-Frame-Options` allows embedding for published forms, postMessage communication works for height adjustment, custom CSS does not leak to parent page.
15. **Template rendering** — verify: all 3 system themes (Minimal, Corporate, Playful) render correctly with all field types, custom CSS overrides apply without breaking layout, theme switching in editor reflects immediately in preview.
16. **Multi-page form navigation** — (v2 deferred) verify: page transitions preserve entered data, back button restores previous page state, per-page validation prevents advancing with errors, progress bar reflects current page accurately.
17. **Export functionality** — verify: CSV export includes all field labels as headers and all response values, file upload fields export as R2 URLs, date fields export in ISO 8601 format, JSON export matches the documented response schema, exports with 1000+ responses complete within 3 seconds.
18. **Load testing** — verify: form builder handles forms with 50+ fields without UI jank, public form page loads in <300ms with ISR cache hit, submission endpoint handles 100 concurrent submissions without errors or data loss, response table paginates smoothly with 10K+ responses.

### Manual Testing Checklist

1. **Auth flow**: Sign up with Clerk → create organization → verify org record created in DB
2. **AI generation**: Enter "Contact form with name, email, phone, message" → verify 4 fields generated with correct types (text, email, text with phone pattern, textarea)
3. **AI conditional logic**: Enter "Bug report with severity dropdown, if critical show urgency field" → verify conditional logic generated and evaluated correctly
4. **Form editor drag-and-drop**: Reorder fields → verify sortOrder updated and auto-saved
5. **Field property editing**: Change label, toggle required, add validation → verify changes auto-saved
6. **Option editing**: Add/remove/reorder dropdown options → verify changes persist
7. **Conditional logic builder**: Add "Show when severity equals critical" → verify field shows/hides in preview
8. **Undo/redo**: Make 5 changes → Cmd+Z 3 times → Cmd+Shift+Z 2 times → verify state correct
9. **Theme switching**: Switch between 3 themes → verify visual changes in preview
10. **Form publish**: Publish form → visit public URL → verify form renders correctly
11. **Turnstile**: Submit form → verify Turnstile challenge completes → response saved
12. **Form submission**: Fill out all field types → submit → verify response in dashboard
13. **File upload**: Upload a 5MB PDF → verify stored in R2 → download from response detail
14. **Email notification**: Submit form → verify owner receives email with response summary
15. **Response table**: View 20+ responses → verify pagination, sorting, filtering work
16. **CSV export**: Export responses → verify headers match field labels, values correct
17. **Response limit**: Set limit to 5 → submit 5 responses → verify 6th is rejected
18. **Plan limits**: On free plan, create 4th form → verify error message
19. **Billing upgrade**: Complete Stripe checkout (test mode) → verify plan updated
20. **Public form closed**: Close form → visit URL → verify "Form closed" message

### Key SQL Queries for Verification

```sql
-- Check form schema stored correctly
SELECT id, title, status, jsonb_array_length(schema) AS field_count,
       response_count, view_count
FROM forms
WHERE org_id = 'ORG_ID'
ORDER BY created_at DESC;

-- Check response with field values
SELECT fr.id, fr.submitted_at, fr.completion_time_seconds,
       fld.field_label_snapshot, fld.value, fld.file_keys
FROM form_responses fr
JOIN field_responses fld ON fld.response_id = fr.id
WHERE fr.form_id = 'FORM_ID'
ORDER BY fr.submitted_at DESC, fld.created_at;

-- Check notification delivery
SELECT nl.recipient_email, nl.status, nl.sent_at,
       nl.metadata->>'formTitle' AS form_title
FROM notification_log nl
WHERE nl.form_id = 'FORM_ID'
ORDER BY nl.created_at DESC;

-- Check storage usage per org
SELECT o.name, o.total_storage_bytes,
       COUNT(fu.id) AS file_count,
       SUM(fu.size_bytes) AS actual_storage
FROM organizations o
LEFT JOIN file_uploads fu ON fu.org_id = o.id AND fu.is_attached = true
WHERE o.id = 'ORG_ID'
GROUP BY o.id;

-- Orphaned uploads (should be cleaned up daily)
SELECT COUNT(*), SUM(size_bytes) AS wasted_bytes
FROM file_uploads
WHERE is_attached = false
  AND created_at < NOW() - INTERVAL '24 hours';

-- Monthly response count (for plan enforcement)
SELECT COUNT(*) FROM form_responses
WHERE org_id = 'ORG_ID'
  AND submitted_at >= date_trunc('month', now());
```

### Performance Benchmarks

| Operation | Target | Method |
|-----------|--------|--------|
| AI form generation (full pipeline) | < 8s | GPT-4o-mini meta + GPT-4o schema |
| Form schema auto-save | < 200ms | Single JSONB update |
| Public form page load (SSR) | < 300ms | ISR with on-demand revalidation |
| Form submission (no files) | < 500ms | Insert response + field responses |
| File upload presigned URL | < 100ms | R2 SDK call |
| File upload (10MB) | < 5s | Direct to R2 via presigned URL |
| Response list (100 responses) | < 400ms | Indexed query + React Query cache |
| CSV export (1000 responses) | < 3s | Streaming response |
| Turnstile verification | < 500ms | Cloudflare API call |
| Email notification dispatch | < 2s | Resend API |

---

## Post-MVP Roadmap

| Feature | Description | Priority |
|---------|-------------|----------|
| Payment Fields (F4) | Stripe-powered payment collection fields with configurable amounts, one-time and recurring payment support, and receipt generation | High |
| Multi-Page Wizard Mode (F5) | Step-by-step wizard navigation with progress bar, per-page validation, and back/forward controls — schema already supports `pages` array | High |
| Advanced Analytics (F9) | Funnel visualization, field-level drop-off analysis, time-to-complete metrics, UTM source breakdown, and embeddable analytics widget | Medium |
| Signature Field | Canvas/touch-based signature capture stored as PNG data URL in R2, with replay animation in response viewer | Medium |
| Rich Text Field | Tiptap-based rich text editor for respondents with limited formatting (bold, italic, lists, links) | Medium |
| Hidden Fields | Pre-populated hidden fields from URL params or embed context for tracking attribution and user identity | Low |
| Zapier/Webhook Templates | Pre-built Zapier triggers and webhook payload templates for common integrations (Slack, Notion, Airtable, Google Sheets) | Low |
| Custom CSS Injection | Pro-tier custom CSS editor for advanced form styling beyond theme presets | Low |
