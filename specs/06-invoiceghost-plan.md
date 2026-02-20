# InvoiceGhost — Implementation Plan

---

## 6. InvoiceGhost — Automated Payment Follow-Up

**MVP Scope:** Stripe Invoices integration + manual invoice entry + 3 pre-built reminder sequences (Gentle/Standard/Firm) + AI-generated escalating reminder emails via GPT-4o-mini + dashboard with outstanding/overdue/collected metrics and aging chart + email sending via Resend + auto-stop on Stripe payment detection via webhooks + Stripe billing for Free/Pro/Agency tiers.

---

### Tech Decisions

- **Framework**: Next.js 14+ App Router. Consistent with DriftLog, SnipVault, FormForge. Server components for dashboard, client components for interactive sequence builder and invoice forms.
- **API Layer**: tRPC with Clerk auth context. Routers: `invoice`, `client`, `sequence`, `reminder`, `dashboard`, `settings`, `billing`. Server-side caller for webhook handlers and cron jobs.
- **Database**: PostgreSQL on Neon, Drizzle ORM. Seven core tables (see schema below). UUID primary keys, `timestamp` columns with `defaultNow()`, `jsonb` for flexible config. Drizzle Kit for migrations.
- **Auth**: Clerk with user-level accounts (not org-level for MVP). Clerk `userId` stored on all user-owned rows. Middleware protects `/dashboard/*` routes. Webhook sync for user creation.
- **Billing**: Stripe Checkout + Customer Portal. Three tiers: Free (5 invoices/mo, 1 integration, 3 reminders/sequence, branding), Pro $12/mo (unlimited, all integrations, AI emails, custom sequences, no branding), Agency $29/mo (team, white-label, multi-entity). Plan limits enforced in tRPC middleware.
- **Scheduler**: **QStash** (Upstash) over BullMQ+Redis on Railway. Rationale: QStash is serverless, requires no separate worker process, integrates natively with Vercel serverless functions via HTTP callbacks, handles retries/deduplication out of the box, and costs $0 for 500 messages/day (sufficient for MVP). BullMQ would require a persistent Railway worker ($5+/mo), Redis connection management, and separate deployment pipeline. QStash publishes a scheduled HTTP POST to `/api/jobs/send-reminder` at the calculated future timestamp. Each scheduled job gets a QStash message ID stored in `scheduled_jobs` for cancellation on payment.
- **Stripe Integration (Invoice Source)**: OAuth Connect is overkill for reading invoices. Instead, users paste a Stripe Restricted API Key (with `invoices:read` + `events:read` scope). Key is encrypted with AES-256-GCM using a `ENCRYPTION_KEY` env var before storage. A webhook endpoint (`/api/webhooks/stripe-invoices`) receives `invoice.paid`, `invoice.payment_succeeded`, `invoice.voided`, `invoice.marked_uncollectible` events to auto-update invoice status and cancel pending reminder jobs. Initial sync pulls last 90 days of open/overdue invoices via `stripe.invoices.list({ status: 'open' })`.
- **AI Email Generation**: GPT-4o-mini via OpenAI SDK. Cost: ~$0.00015 per reminder email (150 input tokens + 300 output tokens). Four tone presets (friendly/professional/direct/empathetic) map to different system prompts. Escalation level (friendly/firm/urgent/final) adjusts urgency within the chosen tone. Custom voice samples (user pastes 2-3 previous emails) are included as few-shot examples in the prompt. Emails are generated at send time (not pre-generated) to include real-time invoice data (current days overdue, amount remaining).
- **Email Sending**: Resend with custom domain support (Pro+ plans). Free tier emails sent from `reminders@invoiceghost.com` with InvoiceGhost branding footer. Resend domain verification via DNS TXT/CNAME records configured in Settings. Open/click tracking deferred to v1.1 (Resend supports it but UI for displaying it is out of MVP scope).
- **Encryption**: AES-256-GCM for Stripe API keys at rest. `encrypt()`/`decrypt()` utility using Node.js `crypto` module with a 32-byte `ENCRYPTION_KEY` from env. IV is prepended to ciphertext and stored as a single base64 string in the `credentials` column.
- **UI Components**: Tailwind CSS + shadcn/ui (consistent with other projects). Recharts for aging chart and dashboard metrics. `date-fns` for date manipulation and "X days overdue" calculations. `react-hook-form` + `zod` for invoice entry and settings forms.
- **Hosting**: Vercel. QStash callback URLs point to Vercel serverless functions. No separate worker infrastructure needed.

---

### Database Schema (Drizzle)

```typescript
// src/server/db/schema.ts

// ─── Users ───────────────────────────────────────────────────────────────────
export const users = pgTable("users", {
  id:                  uuid("id").primaryKey().defaultRandom(),
  clerkUserId:         text("clerk_user_id").unique().notNull(),
  email:               text("email").notNull(),
  name:                text("name"),
  businessName:        text("business_name"),
  plan:                text("plan").default("free").notNull(),        // free | pro | agency
  stripeCustomerId:    text("stripe_customer_id"),
  stripeSubscriptionId:text("stripe_subscription_id"),
  tonePreference:      text("tone_preference").default("professional").notNull(), // friendly | professional | direct | empathetic
  customVoiceSamples:  jsonb("custom_voice_samples").default([]).$type<string[]>(), // up to 3 sample emails
  defaultSequenceId:   uuid("default_sequence_id"),                  // FK set after seed
  emailFromName:       text("email_from_name"),                      // Custom "From" name
  resendDomainId:      text("resend_domain_id"),                     // If custom domain verified
  createdAt:           timestamp("created_at").defaultNow().notNull(),
  updatedAt:           timestamp("updated_at").defaultNow().notNull(),
});

// ─── Invoice Sources ─────────────────────────────────────────────────────────
export const invoiceSources = pgTable("invoice_sources", {
  id:                  uuid("id").primaryKey().defaultRandom(),
  userId:              uuid("user_id").references(() => users.id, { onDelete: "cascade" }).notNull(),
  type:                text("type").notNull(),                       // stripe | manual
  label:               text("label").default("My Stripe Account"),   // User-facing name
  credentials:         text("credentials"),                          // AES-256-GCM encrypted JSON (Stripe restricted key + webhook secret)
  stripeWebhookId:     text("stripe_webhook_id"),                    // Stripe webhook endpoint ID for cleanup
  lastSyncedAt:        timestamp("last_synced_at"),
  syncStatus:          text("sync_status").default("idle"),          // idle | syncing | error
  syncError:           text("sync_error"),
  autoEnrollOverdue:   boolean("auto_enroll_overdue").default(true).notNull(),
  createdAt:           timestamp("created_at").defaultNow().notNull(),
});

// ─── Clients ─────────────────────────────────────────────────────────────────
export const clients = pgTable("clients", {
  id:                  uuid("id").primaryKey().defaultRandom(),
  userId:              uuid("user_id").references(() => users.id, { onDelete: "cascade" }).notNull(),
  name:                text("name").notNull(),
  email:               text("email").notNull(),
  company:             text("company"),
  stripeCustomerId:    text("stripe_customer_id"),                   // From Stripe invoice data
  customSequenceId:    uuid("custom_sequence_id"),                   // Override user default
  isPaused:            boolean("is_paused").default(false).notNull(),
  notes:               text("notes"),
  totalOutstanding:    integer("total_outstanding").default(0).notNull(), // cents
  totalPaid:           integer("total_paid").default(0).notNull(),       // cents
  avgDaysToPay:        integer("avg_days_to_pay"),
  invoiceCount:        integer("invoice_count").default(0).notNull(),
  createdAt:           timestamp("created_at").defaultNow().notNull(),
  updatedAt:           timestamp("updated_at").defaultNow().notNull(),
});

// ─── Invoices ────────────────────────────────────────────────────────────────
export const invoices = pgTable("invoices", {
  id:                  uuid("id").primaryKey().defaultRandom(),
  userId:              uuid("user_id").references(() => users.id, { onDelete: "cascade" }).notNull(),
  clientId:            uuid("client_id").references(() => clients.id, { onDelete: "cascade" }).notNull(),
  sourceId:            uuid("source_id").references(() => invoiceSources.id, { onDelete: "set null" }),
  externalInvoiceId:   text("external_invoice_id"),                  // Stripe invoice ID (in_xxxx)
  invoiceNumber:       text("invoice_number").notNull(),
  description:         text("description"),                          // Project/work description for AI context
  amount:              integer("amount").notNull(),                   // Total in cents
  currency:            text("currency").default("usd").notNull(),
  issuedDate:          timestamp("issued_date").notNull(),
  dueDate:             timestamp("due_date").notNull(),
  status:              text("status").default("sent").notNull(),      // draft | sent | overdue | in_sequence | paid | partial | void | written_off
  amountPaid:          integer("amount_paid").default(0).notNull(),   // cents
  paidAt:              timestamp("paid_at"),
  sequenceStatus:      text("sequence_status"),                      // null | pending | active | paused | completed | stopped
  sequenceId:          uuid("sequence_id"),                          // Which sequence is running
  currentStepNumber:   integer("current_step_number"),               // Which step we're on
  remindersSent:       integer("reminders_sent").default(0).notNull(),
  paymentLinkUrl:      text("payment_link_url"),                     // Stripe payment link or custom URL
  createdAt:           timestamp("created_at").defaultNow().notNull(),
  updatedAt:           timestamp("updated_at").defaultNow().notNull(),
});

// ─── Reminder Sequences ──────────────────────────────────────────────────────
export const reminderSequences = pgTable("reminder_sequences", {
  id:                  uuid("id").primaryKey().defaultRandom(),
  userId:              uuid("user_id").references(() => users.id, { onDelete: "cascade" }).notNull(),
  name:                text("name").notNull(),                       // "Gentle", "Standard", "Firm"
  description:         text("description"),
  isSystem:            boolean("is_system").default(false).notNull(), // true for pre-built templates
  isDefault:           boolean("is_default").default(false).notNull(),
  maxReminders:        integer("max_reminders").default(5).notNull(),
  skipWeekends:        boolean("skip_weekends").default(true).notNull(),
  createdAt:           timestamp("created_at").defaultNow().notNull(),
});

// ─── Sequence Steps ──────────────────────────────────────────────────────────
export const sequenceSteps = pgTable("sequence_steps", {
  id:                  uuid("id").primaryKey().defaultRandom(),
  sequenceId:          uuid("sequence_id").references(() => reminderSequences.id, { onDelete: "cascade" }).notNull(),
  stepNumber:          integer("step_number").notNull(),              // 1, 2, 3, ...
  delayDays:           integer("delay_days").notNull(),               // Days after due date
  escalationLevel:     text("escalation_level").notNull(),           // friendly | firm | urgent | final
  subjectTemplate:     text("subject_template"),                     // Optional custom subject override
  bodyTemplate:        text("body_template"),                        // Optional custom body override (bypasses AI)
});

// ─── Sent Reminders ──────────────────────────────────────────────────────────
export const sentReminders = pgTable("sent_reminders", {
  id:                  uuid("id").primaryKey().defaultRandom(),
  invoiceId:           uuid("invoice_id").references(() => invoices.id, { onDelete: "cascade" }).notNull(),
  stepId:              uuid("step_id").references(() => sequenceSteps.id, { onDelete: "set null" }),
  stepNumber:          integer("step_number").notNull(),
  emailSubject:        text("email_subject").notNull(),
  emailBody:           text("email_body").notNull(),
  recipientEmail:      text("recipient_email").notNull(),
  status:              text("status").default("sent").notNull(),     // sent | opened | clicked | replied | bounced
  resendMessageId:     text("resend_message_id"),                    // Resend API message ID
  aiGenerated:         boolean("ai_generated").default(true).notNull(),
  sentAt:              timestamp("sent_at").defaultNow().notNull(),
  openedAt:            timestamp("opened_at"),
  clickedAt:           timestamp("clicked_at"),
});

// ─── Scheduled Jobs ──────────────────────────────────────────────────────────
export const scheduledJobs = pgTable("scheduled_jobs", {
  id:                  uuid("id").primaryKey().defaultRandom(),
  invoiceId:           uuid("invoice_id").references(() => invoices.id, { onDelete: "cascade" }).notNull(),
  stepId:              uuid("step_id").references(() => sequenceSteps.id, { onDelete: "set null" }),
  stepNumber:          integer("step_number").notNull(),
  scheduledFor:        timestamp("scheduled_for").notNull(),
  executedAt:          timestamp("executed_at"),
  status:              text("status").default("pending").notNull(),   // pending | executed | cancelled | failed
  qstashMessageId:     text("qstash_message_id"),                   // For cancellation via QStash API
  failureReason:       text("failure_reason"),
  createdAt:           timestamp("created_at").defaultNow().notNull(),
});
```

**Pre-seeded Reminder Sequences** (inserted via migration seed):

| Sequence | Steps (delay_days → escalation_level) |
|----------|---------------------------------------|
| **Gentle** | Day 1 → friendly, Day 3 → friendly, Day 7 → firm, Day 14 → firm, Day 30 → urgent |
| **Standard** | Day 1 → friendly, Day 5 → firm, Day 14 → urgent, Day 30 → urgent, Day 45 → final |
| **Firm** | Day 1 → firm, Day 3 → firm, Day 7 → urgent, Day 14 → urgent, Day 21 → final, Day 30 → final |

**Indexes**: `invoices(userId, status)`, `invoices(userId, dueDate)`, `invoices(externalInvoiceId)`, `scheduledJobs(status, scheduledFor)`, `scheduledJobs(invoiceId, status)`, `clients(userId, email)`, `sentReminders(invoiceId, sentAt)`.

---

### AI Prompt Strategy for Escalating Tone

The AI email generation uses a layered prompt system with three components:

**1. System Prompt (sets tone baseline)**

```
TONE: friendly
---
You are a helpful payment reminder assistant. Write a brief, warm email
reminding a client about an outstanding invoice. Be polite, assume good
intent (they probably just forgot), and keep the email under 150 words.
Never threaten legal action. Always include the invoice details provided.
Sign off with the sender's name.

TONE: professional
---
You are a business communication assistant. Write a concise, professional
payment reminder email. Maintain a neutral, businesslike tone. Reference
the invoice details precisely. Keep under 150 words. Sign off formally.

TONE: direct
---
You are a payment collections assistant. Write a clear, no-nonsense
reminder email. Be straightforward about the outstanding balance and
expectations. Avoid excessive pleasantries but remain respectful.
Keep under 120 words.

TONE: empathetic
---
You are a thoughtful payment reminder assistant. Write a gentle reminder
that acknowledges the client may be dealing with challenges. Offer to
discuss payment arrangements if needed. Be understanding but clear about
the outstanding amount. Keep under 150 words.
```

**2. Escalation Layer (injected after system prompt based on step's escalation_level)**

```
ESCALATION: friendly
---
This is an early, gentle reminder. The client may not even realize the
invoice is overdue. Approach with "just a friendly reminder" energy.

ESCALATION: firm
---
This is a follow-up reminder. The invoice is clearly overdue and previous
reminders have been sent. Be polite but clear that payment is expected
promptly. Mention the specific number of days overdue.

ESCALATION: urgent
---
This invoice is significantly overdue. Express that this is an urgent
matter. Mention that previous reminders have gone unanswered. Request
immediate attention. Mention potential consequences like service
suspension (if applicable) but remain professional.

ESCALATION: final
---
This is a final notice before escalation. State clearly that this is the
last reminder. Mention that the account may be referred to collections or
that services may be suspended. Maintain professionalism but convey
absolute seriousness.
```

**3. User Message (dynamic data)**

```
Write a payment reminder email with these details:
- Sender: {{businessName}} ({{senderName}})
- Client: {{clientName}} at {{clientCompany}}
- Invoice #: {{invoiceNumber}}
- Description: {{description}}
- Amount due: {{amountDue}} {{currency}}
- Original due date: {{dueDate}}
- Days overdue: {{daysOverdue}}
- Previous reminders sent: {{reminderCount}}
- Payment link: {{paymentLinkUrl}}

{{#if customVoiceSamples}}
Match the writing style of these example emails from the sender:
---
{{customVoiceSamples}}
---
{{/if}}
```

**Token budget**: ~150 input + ~300 output = ~450 tokens per email. At GPT-4o-mini pricing ($0.15/1M input, $0.60/1M output), cost is ~$0.0002 per reminder email. For 10,000 reminders/month = ~$2/month in AI costs.

---

### Phase Breakdown (5.5 weeks)

#### Phase 1: Foundation + Stripe Integration (Days 1-7)

**Day 1 — Project Bootstrap**
- `npx create-next-app@latest invoiceghost --typescript --tailwind --app --src-dir`
- Install core deps: `drizzle-orm`, `drizzle-kit`, `@trpc/server`, `@trpc/client`, `@trpc/next`, `@clerk/nextjs`, `zod`, `superjson`
- Install UI deps: `@radix-ui/react-*` (dialog, dropdown, tabs, tooltip), `recharts`, `date-fns`, `lucide-react`, `class-variance-authority`, `clsx`, `tailwind-merge`
- Configure `drizzle.config.ts` with Neon connection string
- Set up `src/env.ts` with Zod env validation (DATABASE_URL, CLERK_*, STRIPE_*, OPENAI_API_KEY, QSTASH_*, RESEND_API_KEY, ENCRYPTION_KEY)
- Create `src/server/db/index.ts` (Drizzle client with Neon serverless driver)
- Create `src/server/db/schema.ts` with all 7 tables + relations
- Run `drizzle-kit generate` and `drizzle-kit push` for initial migration
- Set up Clerk: `src/middleware.ts` protecting `/dashboard/*` routes
- Create `src/app/layout.tsx` with `<ClerkProvider>` and Tailwind globals
- Create `src/app/(auth)/sign-in/[[...sign-in]]/page.tsx` and sign-up equivalent

**Day 2 — tRPC Setup + User Sync**
- Create `src/server/trpc/trpc.ts` — base router, auth middleware, plan-limit middleware
- Create `src/server/trpc/context.ts` — extract Clerk userId, load user from DB
- Create `src/server/trpc/routers/_app.ts` — merge all routers
- Create `src/app/api/trpc/[trpc]/route.ts` — tRPC HTTP handler
- Create `src/lib/trpc/client.ts` — client-side tRPC hooks
- Create `src/lib/trpc/server.ts` — server-side caller
- Create `src/app/api/webhooks/clerk/route.ts` — handle `user.created` event: insert into `users` table, clone 3 system sequences as user's personal sequences, set default sequence
- Create `src/server/db/seed-sequences.ts` — function to create Gentle/Standard/Firm sequences + steps for a user
- Test: sign up via Clerk, verify user row created with 3 sequences

**Day 3 — Encryption Utilities + Invoice Source Management**
- Create `src/lib/encryption.ts`:
  - `encrypt(plaintext: string): string` — AES-256-GCM, random 12-byte IV, return `base64(iv + ciphertext + authTag)`
  - `decrypt(encrypted: string): string` — extract IV, decrypt, verify auth tag
  - Unit tests for round-trip encryption
- Create `src/server/trpc/routers/settings.ts`:
  - `settings.getProfile` — return user profile, tone, voice samples
  - `settings.updateProfile` — update businessName, tonePreference, emailFromName
  - `settings.updateVoiceSamples` — validate max 3 samples, max 2000 chars each. **Voice/tone sample validation**: each sample must be a minimum of 500 characters to provide enough stylistic signal for the AI. On save, a quick GPT-4o-mini call computes a tone consistency score (0-1) comparing the sample against the user's selected `tonePreference`. If the score is < 0.7, the sample is accepted but a warning is shown ("Your sample doesn't closely match your selected tone — the AI will fall back to neutral tone for this sample"). At email generation time, samples with consistency score < 0.7 are excluded from the few-shot examples and the AI falls back to the selected tone preset without custom voice styling.
- Create `src/server/trpc/routers/source.ts`:
  - `source.list` — list user's invoice sources
  - `source.connectStripe` — accept Stripe restricted API key, validate by calling `stripe.invoices.list({ limit: 1 })`, encrypt and store. Register a Stripe webhook endpoint pointing to our callback URL. Store webhook secret encrypted.
  - `source.disconnect` — delete webhook endpoint from Stripe, delete source + cascading invoices
  - `source.syncNow` — trigger manual sync (calls sync function directly)
  - **Monthly credential re-validation**: a Vercel cron job (`/api/cron/validate-credentials`) runs on the 1st of each month. For each active `invoiceSource` with type `stripe`, it decrypts the stored API key, calls `stripe.invoices.list({ limit: 1 })`, and verifies the key is still valid. If the call fails with a 401 (invalid key) or 403 (insufficient permissions), the source status is set to `error` with `syncError = "Stripe API key is no longer valid — please reconnect"`, and an email notification is sent to the user prompting them to update their credentials. This prevents silent sync failures when users rotate their Stripe API keys.

**Day 4 — Stripe Invoice Sync Engine**
- Create `src/server/integrations/stripe-sync.ts`:
  - `syncStripeInvoices(source: InvoiceSource)` — decrypt credentials, call `stripe.invoices.list({ status: 'open', limit: 100 })` with pagination, also fetch `past_due` invoices
  - For each Stripe invoice: upsert `clients` row (match by `customer.email`), upsert `invoices` row (match by `externalInvoiceId`)
  - Map Stripe statuses: `open` → `sent`, `past_due` with due_date < now → `overdue`, `paid` → `paid`, `void` → `void`, `uncollectible` → `written_off`
  - Update `source.lastSyncedAt` and `syncStatus`
  - Handle API errors gracefully: store error in `syncError`, set status to `error`
- Create `src/server/integrations/stripe-client.ts` — factory: `getStripeClient(encryptedKey: string)` returns configured Stripe instance with decrypted key
- Add index on `invoices(externalInvoiceId)` for fast upsert lookups
- Test: connect a Stripe test account, verify invoices sync into DB

**Day 5 — Stripe Webhook Handler (Payment Detection)**
- Create `src/app/api/webhooks/stripe-invoices/route.ts`:
  - Verify webhook signature using the per-source webhook secret (look up source by matching the webhook endpoint URL or a custom header)
  - Handle events:
    - `invoice.paid` / `invoice.payment_succeeded`: update invoice `status` → `paid`, set `amountPaid`, `paidAt`. Cancel all pending `scheduledJobs` for this invoice via QStash API. Update invoice `sequenceStatus` → `stopped`. Update client `totalPaid` and `totalOutstanding` aggregates.
    - `invoice.voided`: update invoice `status` → `void`, cancel pending jobs
    - `invoice.marked_uncollectible`: update invoice `status` → `written_off`, cancel pending jobs
    - `invoice.updated`: re-sync amount, due date changes
    - `invoice.payment_failed`: log for activity feed (no action on sequence)
  - Idempotency: check if event already processed by storing Stripe event ID in a lightweight `processedEvents` set (or check if invoice already in target status)
- Create `src/server/scheduler/cancel-jobs.ts`:
  - `cancelPendingJobs(invoiceId: string)` — query `scheduledJobs` where `invoiceId` and `status = 'pending'`, call QStash delete for each `qstashMessageId`, update status to `cancelled`
- Test: send a test webhook via Stripe CLI (`stripe trigger invoice.paid`), verify invoice status updates and jobs cancelled

**Day 6 — Manual Invoice Entry**
- Create `src/server/trpc/routers/invoice.ts`:
  - `invoice.list` — paginated, filterable by status/client/date range, sortable by amount/dueDate/status. Include `nextReminderDate` computed from scheduledJobs.
  - `invoice.getById` — full invoice with client, sent reminders, scheduled jobs
  - `invoice.create` — manual entry: validate fields (invoiceNumber, amount, dueDate, clientEmail, clientName). Auto-create or match `clients` row by email. Source is the user's "manual" source (auto-created if not exists). Plan limit check: free tier max 5 invoices/month.
  - **Free tier counting clarification**: the 5-invoice limit uses a **rolling calendar month window** aligned to **UTC day boundaries**. The counter counts invoices where `createdAt >= start of current UTC month`. The counter resets at `00:00:00 UTC` on the 1st of each month. There is no proration for mid-month signups (a user who signs up on the 28th still gets 5 invoices for that partial month). The count includes both Stripe-synced and manually created invoices. Deleted invoices are subtracted from the count. The enforcement query is: `SELECT COUNT(*) FROM invoices WHERE user_id = ? AND created_at >= date_trunc('month', now() AT TIME ZONE 'UTC')`.
  - `invoice.update` — edit amount, dueDate, description, status
  - `invoice.markAsPaid` — set status to `paid`, cancel pending jobs, update client aggregates
  - `invoice.writeOff` — set status to `written_off`, cancel pending jobs
  - `invoice.delete` — soft considerations, but hard delete for MVP with cascade
- Create `src/server/trpc/routers/client.ts`:
  - `client.list` — list clients with aggregated stats
  - `client.getById` — client with all invoices and payment history
  - `client.update` — update notes, customSequenceId, isPaused
  - `client.togglePause` — pause/unpause all reminders for client (cancel/reschedule jobs)

**Day 7 — Sequence Management + Enrollment**
- Create `src/server/trpc/routers/sequence.ts`:
  - `sequence.list` — list user's sequences (system + custom)
  - `sequence.getById` — sequence with all steps
  - `sequence.create` — custom sequence (Pro+ only). Validate step ordering, delay_days must be ascending.
  - `sequence.update` — edit name, steps. Cannot edit system sequences (clone instead).
  - `sequence.delete` — cannot delete system sequences
  - `sequence.preview` — given a sequence ID and sample invoice data, return AI-generated preview of all emails in the sequence (so user can see what each step looks like before enrolling)
- Create `src/server/scheduler/enroll-invoice.ts`:
  - `enrollInvoice(invoiceId: string)` — determine which sequence to use (client override > user default). For each step: calculate `scheduledFor = dueDate + delay_days` (skip weekends if enabled). If `scheduledFor` is in the past (invoice already overdue beyond this step), skip it. Publish QStash message to `/api/jobs/send-reminder` with `{ invoiceId, stepId, stepNumber }` scheduled for `scheduledFor`. Store `scheduledJobs` row with `qstashMessageId`. Update invoice `sequenceStatus` → `active`, `sequenceId`.
  - Handle auto-enrollment: when invoice status becomes `overdue` (via sync or manual), check `source.autoEnrollOverdue` or user setting, then enroll.
- Create `src/app/api/jobs/send-reminder/route.ts` — stub (implemented Day 9)

---

#### Phase 2: AI Email Generation + Sending Pipeline (Days 8-14)

**Day 8 — AI Email Generation Service**
- Create `src/server/ai/generate-reminder-email.ts`:
  - `generateReminderEmail(params: { invoice, client, user, step, previousReminders })` → `{ subject: string, body: string }`
  - Build system prompt from user's `tonePreference` + step's `escalationLevel` (see Prompt Strategy above)
  - Build user message with invoice data, client info, days overdue, payment link
  - If user has `customVoiceSamples`, append as few-shot examples
  - If step has `bodyTemplate` (custom override), skip AI and use template with variable substitution
  - Call `openai.chat.completions.create({ model: 'gpt-4o-mini', messages, temperature: 0.7, max_tokens: 500 })`
  - Parse response: extract subject line and body (instruct model to return `SUBJECT: ...\n\nBODY: ...` format)
  - Fallback: if AI fails, use a hardcoded template per escalation level
- Create `src/server/ai/prompts.ts` — all system prompts, escalation layers, and the user message template as typed constants
- Create `src/server/ai/fallback-templates.ts` — static email templates per escalation level for when AI is unavailable
- Test: unit test with mock invoice data, verify each tone+escalation combination produces appropriate output

**Day 9 — Email Sending Pipeline + QStash Job Handler**
- Create `src/server/email/send-reminder.ts`:
  - `sendReminderEmail(params: { to, from, fromName, subject, body, replyTo })` → `{ messageId: string }`
  - Use Resend SDK: `resend.emails.send({ from, to, subject, html: renderEmailHtml(body), text: body })`
  - Free tier: `from = 'InvoiceGhost Reminders <reminders@invoiceghost.com>'`, append branding footer
  - Pro+: `from = '{{businessName}} <reminders@{{customDomain}}>'` if custom domain verified, else use InvoiceGhost domain with user's fromName
- Create `src/server/email/templates/reminder-email.tsx` — React Email template with clean layout: logo/business name header, email body (markdown-rendered), invoice summary table (number, amount, due date, days overdue), payment link CTA button, footer with unsubscribe/branding
- Implement `src/app/api/jobs/send-reminder/route.ts` (full implementation):
  - Verify QStash signature (`Receiver.verify()` from `@upstash/qstash`)
  - Extract `{ invoiceId, stepId, stepNumber }` from body
  - Load invoice, client, user, step from DB
  - Guard checks: invoice not already paid/void/written_off? Client not paused? Sequence not paused/stopped?
  - If all checks pass: call `generateReminderEmail()`, then `sendReminderEmail()`
  - Insert `sentReminders` row with messageId, subject, body
  - Update `scheduledJobs` row: `status` → `executed`, `executedAt` → now
  - Update invoice: `remindersSent` += 1, `currentStepNumber` = stepNumber
  - If this was the last step in the sequence: update `sequenceStatus` → `completed`
  - On failure: update `scheduledJobs.status` → `failed`, store error in `failureReason`. QStash automatic retry will re-attempt (configure 3 retries with exponential backoff).

**Day 10 — Sequence Enrollment Flow (End-to-End)**
- Wire up auto-enrollment in Stripe sync: after syncing, if any invoice is newly overdue and `autoEnrollOverdue` is true, call `enrollInvoice()`
- Add manual enrollment: `invoice.startSequence` tRPC mutation — user picks an invoice and optionally a sequence, calls `enrollInvoice()`
- Add `invoice.pauseSequence` — set `sequenceStatus` → `paused`, cancel all pending scheduled jobs (store step info so we can resume)
- Add `invoice.resumeSequence` — re-enroll starting from the next pending step
- Add `invoice.stopSequence` — set `sequenceStatus` → `stopped`, cancel all pending jobs permanently
- Test end-to-end: create invoice → enroll → verify QStash messages scheduled → trigger one manually → verify email sent + sentReminders row created

**Day 11 — Cron: Overdue Detection + Sync**
- Create `src/app/api/cron/sync-invoices/route.ts`:
  - Triggered by Vercel Cron (every 6 hours) or QStash scheduled message
  - For each active Stripe invoice source: call `syncStripeInvoices()`
  - After sync: scan for invoices where `dueDate < now` and `status = 'sent'`, update to `overdue`
  - Auto-enroll newly overdue invoices if `autoEnrollOverdue` is enabled
  - Log sync results for debugging
- Create `src/app/api/cron/check-overdue/route.ts`:
  - Triggered daily at 8am UTC
  - Find all invoices where `status = 'sent'` and `dueDate < now`, update to `overdue`
  - This catches manual invoices that don't have Stripe sync
- Add `vercel.json` cron configuration:
  ```json
  { "crons": [
    { "path": "/api/cron/sync-invoices", "schedule": "0 */6 * * *" },
    { "path": "/api/cron/check-overdue", "schedule": "0 8 * * *" }
  ]}
  ```

**Day 12 — Dashboard tRPC Router + Data Aggregation**
- Create `src/server/trpc/routers/dashboard.ts`:
  - `dashboard.overview` — returns: totalOutstanding (sum of unpaid invoice amounts), totalOverdue (sum where status in overdue/in_sequence), collectedThisMonth (sum of paid invoices where paidAt in current month), collectedLastMonth, recoveryRate (paid-after-reminder / total-in-sequence), activeSequences count
  - `dashboard.agingBreakdown` — group overdue invoices into buckets: 1-30 days, 31-60, 61-90, 90+. Return count and total amount per bucket.
  - `dashboard.activityFeed` — union query of recent events: sentReminders (last 20), invoice status changes, new invoices synced, payments received. Sorted by timestamp.
  - `dashboard.upcomingReminders` — next 10 scheduled jobs with invoice and client info

**Day 13 — Email Preview + Inline Editing**
- Add `sequence.previewEmail` tRPC query — given invoiceId + stepId, generate the AI email and return subject+body without sending. Allows user to see exactly what will be sent.
- Add `invoice.editUpcomingReminder` mutation — for a pending scheduledJob, user can provide custom subject+body override. Store as `bodyTemplate`/`subjectTemplate` on a per-job basis (add `customSubject`/`customBody` nullable fields to `scheduledJobs`). When the job executes, use custom content instead of AI generation.
- Add `invoice.sendNow` mutation — immediately send the next pending reminder (skip the schedule). Execute the job handler logic inline, cancel the QStash message, update scheduledJob.
- Add `invoice.skipStep` mutation — cancel a specific pending job, advance `currentStepNumber`

**Day 14 — Buffer/Testing Day**
- Write integration tests for the full pipeline: invoice creation → enrollment → job scheduling → job execution → email sending → payment webhook → job cancellation
- Test edge cases: invoice paid between scheduling and execution, client paused mid-sequence, all steps in past (already very overdue), weekend skipping logic
- Test Stripe webhook idempotency (same event delivered twice)
- Test encryption round-trip with different key lengths and special characters
- Verify QStash signature verification in job handler

---

#### Phase 3: Dashboard + Invoice Management UI (Days 15-23)

**Day 15 — App Shell + Navigation**
- Create `src/app/(dashboard)/layout.tsx` — sidebar nav (Dashboard, Invoices, Clients, Sequences, Settings), top bar with user menu (Clerk `<UserButton />`), mobile responsive hamburger menu
- Create `src/components/ui/` — set up shadcn/ui components: Button, Card, Badge, Table, Dialog, DropdownMenu, Input, Select, Textarea, Tabs, Tooltip, Skeleton
- Create `src/components/layout/sidebar.tsx` — navigation with active state, icons (Lucide: LayoutDashboard, FileText, Users, Repeat, Settings, CreditCard)
- Create `src/components/layout/page-header.tsx` — reusable page header with title, description, action buttons

**Day 16 — Dashboard Page**
- Create `src/app/(dashboard)/dashboard/page.tsx`:
  - Server component that fetches `dashboard.overview` via server-side tRPC caller
  - Four metric cards at top: Total Outstanding (formatted currency), Total Overdue (with red accent), Collected This Month (with green accent + vs last month comparison), Active Sequences
  - Each card uses `src/components/dashboard/metric-card.tsx` with icon, value, label, trend indicator
- Create `src/components/dashboard/aging-chart.tsx` — Recharts stacked bar chart showing invoice aging buckets. X-axis: time buckets (1-30, 31-60, 61-90, 90+). Y-axis: dollar amount. Color gradient from yellow to red.
- Create `src/components/dashboard/activity-feed.tsx` — scrollable list of recent events. Each item: icon (email sent, payment received, invoice added), description, relative timestamp. Click navigates to invoice detail.
- Create `src/components/dashboard/upcoming-reminders.tsx` — table of next scheduled reminders with invoice number, client, step name, scheduled date, "Send Now" and "Skip" quick actions.

**Day 17 — Invoice List Page**
- Create `src/app/(dashboard)/invoices/page.tsx`:
  - Client component with `useSearchParams` for filter state
  - Fetch `invoice.list` with filters
  - Toolbar: status filter dropdown (All, Sent, Overdue, In Sequence, Paid, Written Off), search by invoice number or client name, sort dropdown (Amount, Due Date, Days Overdue, Client)
  - "Add Invoice" button → opens manual entry dialog
- Create `src/components/invoices/invoice-table.tsx` — data table with columns: Invoice #, Client, Amount, Due Date, Days Overdue, Status (badge), Sequence Status (badge), Next Reminder, Actions (dropdown: Start Sequence, Pause, Mark Paid, Write Off)
- Create `src/components/invoices/status-badge.tsx` — color-coded badges: draft (gray), sent (blue), overdue (orange), in_sequence (purple), paid (green), partial (yellow), void (slate), written_off (red)
- Create `src/components/invoices/add-invoice-dialog.tsx` — form with: client name, client email, client company, invoice number, description, amount (currency input), issued date, due date, payment link URL (optional). Uses react-hook-form + zod validation. On submit: `invoice.create` mutation, close dialog, refresh list.

**Day 18 — Invoice Detail Page**
- Create `src/app/(dashboard)/invoices/[id]/page.tsx`:
  - Server component fetching `invoice.getById`
  - Header: invoice number, status badge, amount (large), client name link
  - Info grid: issued date, due date, days overdue, amount paid, payment link
  - Action buttons: Start/Pause/Resume Sequence, Send Reminder Now, Mark as Paid, Write Off, Edit
- Create `src/components/invoices/reminder-timeline.tsx` — vertical timeline showing:
  - Completed steps: green checkmark, sent date, subject line preview, open/click status
  - Current step: pulsing dot, "Sending on [date]", preview button
  - Future steps: gray dots, scheduled dates, escalation level label
  - Each step is expandable to show full email subject + body
- Create `src/components/invoices/email-preview-dialog.tsx` — modal showing the AI-generated email for a specific step. "Regenerate" button to get a new version. "Edit" to switch to textarea editing mode. "Send Now" to send immediately. "Approve" to keep as-is.
- Create `src/components/invoices/invoice-activity.tsx` — chronological log: invoice created, sequence started, reminder 1 sent, reminder 1 opened, payment received, etc.

**Day 19 — Client List + Client Detail Pages**
- Create `src/app/(dashboard)/clients/page.tsx`:
  - Table: client name, company, email, total outstanding, total paid, invoice count, avg days to pay, paused status
  - Search by name/email
  - Click row → client detail
- Create `src/app/(dashboard)/clients/[id]/page.tsx`:
  - Client profile header: name, company, email, paused toggle
  - Stats cards: total outstanding, total paid, avg days to pay, on-time rate
  - Invoice list for this client (reuse invoice table component with client filter)
  - Sequence override dropdown: select a different sequence for this client
  - Notes section: editable textarea
- Create `src/components/clients/payment-history-chart.tsx` — Recharts line/bar showing payment timeline: each invoice as a bar showing time from due date to paid date (negative = early, positive = late)

**Day 20 — Sequence Builder Page**
- Create `src/app/(dashboard)/sequences/page.tsx`:
  - List of sequences: name, step count, "Default" badge, "System" badge
  - "Create Sequence" button (Pro+ only, show upgrade prompt for free tier)
- Create `src/app/(dashboard)/sequences/[id]/page.tsx`:
  - Sequence name (editable if not system)
  - Step list as cards: Step number, "Day X after due date", escalation level badge, email preview snippet
  - "Add Step" button at bottom
  - Each step card: editable delay_days input, escalation level dropdown, "Preview Email" button, delete button (if not system)
  - "Preview Full Sequence" button — opens dialog showing all emails in order with sample invoice data
  - Save button persists all changes in one `sequence.update` mutation
- Create `src/components/sequences/step-card.tsx` — individual step editor
- Create `src/components/sequences/sequence-preview-dialog.tsx` — shows all generated emails sequentially with sample data, each with escalation level and timing label

**Day 21 — Settings Pages**
- Create `src/app/(dashboard)/settings/page.tsx` with tab navigation:
  - **Integrations tab** (`src/components/settings/integrations-tab.tsx`):
    - Connected sources list with status indicator (connected/syncing/error/disconnected)
    - "Connect Stripe" button → dialog: paste restricted API key, validate, save
    - Sync status and last synced timestamp
    - "Sync Now" and "Disconnect" buttons per source
    - Auto-enroll overdue toggle per source
  - **Tone & Voice tab** (`src/components/settings/tone-tab.tsx`):
    - Tone preference radio group: friendly/professional/direct/empathetic with description for each
    - Custom voice samples: 3 textarea fields for pasting example emails
    - "Preview with my tone" button — generates a sample reminder with current settings
  - **Email tab** (`src/components/settings/email-tab.tsx`):
    - From name input
    - Custom domain setup (Pro+): instructions for DNS records, verification status, "Verify Domain" button
    - Email signature textarea (appended to all emails)
  - **Profile tab** (`src/components/settings/profile-tab.tsx`):
    - Business name, name, email (from Clerk, read-only)
    - Default sequence selector

**Day 22 — Billing Page**
- Create `src/app/(dashboard)/billing/page.tsx`:
  - Current plan card with feature list
  - Usage this month: invoices tracked (X / limit), reminders sent, integrations connected
  - Upgrade CTAs for higher tiers
  - "Manage Subscription" button → Stripe Customer Portal
- Create `src/server/trpc/routers/billing.ts`:
  - `billing.getStatus` — current plan, usage counts, subscription details
  - `billing.createCheckout` — create Stripe Checkout session for Pro or Agency
  - `billing.createPortal` — create Stripe Customer Portal session
- Create `src/server/billing/stripe.ts` — plan definitions (PLANS constant), checkout session creation, portal session creation (same pattern as DriftLog)
- Create `src/server/billing/plan-limits.ts` — `checkPlanLimit(userId, action)` → throws if limit exceeded. Actions: `create_invoice` (free: 5/mo), `create_sequence` (free: not allowed), `connect_source` (free: 1)
- Create `src/app/api/webhooks/stripe-billing/route.ts` — handle `checkout.session.completed`, `customer.subscription.updated`, `customer.subscription.deleted`. Update user plan in DB.

**Day 23 — Onboarding Flow**
- Create `src/app/(dashboard)/onboarding/page.tsx` — multi-step onboarding:
  1. **Welcome**: business name, name
  2. **Connect Invoices**: "Connect Stripe" or "I'll add invoices manually" → skip
  3. **Choose Your Tone**: select tone preference, optionally paste voice samples
  4. **Choose Default Sequence**: show 3 pre-built options with step previews
  5. **Done**: redirect to dashboard with confetti animation
- Redirect new users (no `businessName` set) to `/onboarding` from dashboard layout
- Store onboarding completion flag on user record

---

#### Phase 4: Polish, Testing, Launch (Days 24-30)

**Day 24 — Empty States + Loading States**
- Design empty states for all pages: dashboard (no invoices yet → CTA to connect Stripe or add invoice), invoice list (no invoices), client list (no clients), sequences (show system sequences even when empty)
- Add skeleton loading states for all server-fetched pages using shadcn Skeleton component
- Add error boundaries with retry buttons
- Add toast notifications (sonner) for: invoice created, reminder sent, sequence started, payment detected, sync completed, errors

**Day 25 — Responsive Design + Mobile**
- Ensure sidebar collapses to bottom nav on mobile
- Invoice table converts to card list on mobile
- Dashboard metric cards stack vertically
- Dialog/modals are full-screen on mobile
- Test on iPhone SE, iPhone 14, iPad, desktop breakpoints
- Timeline component adapts to horizontal scroll on mobile

**Day 26 — Edge Cases + Error Handling**
- **Stripe API Rate Limit Handling** — all Stripe API calls go through a centralized `stripeRequest()` wrapper that implements:
  - **Exponential backoff**: base delay 1s, multiplied by 2 on each 429 response, capped at 30s maximum delay, with random jitter (+/-500ms) to avoid synchronized retries across users
  - **Queue-based webhook processing**: incoming Stripe webhooks are written to a lightweight in-memory queue (max 500 items) and processed sequentially with a 100ms delay between Stripe API calls to stay well under the 100 req/s limit; if the queue fills, new events are deferred to a QStash delayed message
  - **Per-account rate tracking**: each `invoiceSource` tracks its last Stripe API call timestamp and remaining rate limit headers (`Stripe-RateLimit-Remaining`); sync operations voluntarily throttle to 25 req/s per account to leave headroom for webhook-triggered calls
- Handle Resend sending failures with retry logic in job handler
- Handle QStash callback failures (configure dead letter queue)
- Handle Clerk webhook failures (idempotent user creation)
- Handle stale data: if invoice paid between page load and action, show conflict message
- Add request timeout handling for AI generation (15 second timeout, fallback to templates)
- Rate limit tRPC endpoints: sync (1/minute), email preview (10/minute), create invoice (30/minute)

**Day 27 — Landing Page + Marketing Site**
- Create `src/app/(marketing)/page.tsx` — landing page:
  - Hero: "Stop Chasing Payments. Start Getting Paid." with CTA
  - How it works: 3-step visual (Connect → Set Sequence → Get Paid)
  - Feature grid: AI emails, auto-detection, escalating sequences, Stripe sync
  - Pricing table (3 tiers)
  - Social proof section (placeholder for testimonials)
  - FAQ section
  - Footer with links
- Create `src/app/(marketing)/pricing/page.tsx` — detailed pricing comparison table
- Create `src/app/(marketing)/layout.tsx` — marketing layout with nav (Features, Pricing, Sign In)

**Day 28 — Security Audit + Performance**
- Audit all tRPC routes: ensure userId is checked on every query/mutation (no cross-user data access)
- Audit webhook handlers: verify all signatures (Stripe, QStash, Clerk)
- Audit encryption: verify no plaintext credentials in logs or error messages
- Add CSP headers in `next.config.js`
- Run Lighthouse audit on landing page and dashboard
- Optimize DB queries: ensure all list queries are paginated, add missing indexes
- Test with 100+ invoices per user for dashboard performance

**Day 29 — End-to-End Testing + Bug Fixes**
- Manual QA: full user journey from sign-up → onboarding → connect Stripe → sync invoices → enroll in sequence → receive reminder → payment detected → sequence stops
- Manual QA: manual invoice flow: add invoice → start sequence → preview emails → send now → mark as paid
- Manual QA: free tier limits enforced, upgrade flow works, downgrade doesn't break existing data
- Test Stripe webhook reliability with `stripe listen --forward-to localhost:3000/api/webhooks/stripe-invoices`
- Fix all discovered bugs, UI inconsistencies, and error states

**Day 30 — Deploy + Launch Prep**
- Set up Vercel project with all environment variables
- Configure custom domain for app (app.invoiceghost.com)
- Configure Resend sending domain (invoiceghost.com)
- Set up Vercel Cron jobs
- Configure QStash callback URL to production
- Set up Stripe products/prices for Pro ($12/mo) and Agency ($29/mo)
- Create Stripe webhook endpoint for billing events
- Set up Sentry for error monitoring
- Final smoke test on production
- Deploy and announce

---

### Critical Files

| File | Purpose |
|------|---------|
| `src/server/db/schema.ts` | All 7 Drizzle table definitions, relations, type exports. Central source of truth for the data model. Includes users, invoiceSources, clients, invoices, reminderSequences, sequenceSteps, sentReminders, scheduledJobs. |
| `src/server/scheduler/enroll-invoice.ts` | Core scheduling logic. Takes an invoice, determines the correct sequence (client override > user default), calculates future send dates for each step (with weekend skipping), publishes QStash messages, and creates scheduledJobs rows. This is the "brain" of the reminder system. |
| `src/app/api/jobs/send-reminder/route.ts` | QStash callback handler. Verifies QStash signature, loads invoice/client/user/step data, runs guard checks (not paid, not paused), calls AI generation, sends email via Resend, records sentReminder, updates job status. The most critical execution path in the app. |
| `src/server/ai/generate-reminder-email.ts` | AI email generation with layered prompt system. Combines user tone preference, step escalation level, invoice data, and optional custom voice samples into a GPT-4o-mini call. Includes fallback to static templates on AI failure. Controls the "voice" of all outgoing reminders. |
| `src/app/api/webhooks/stripe-invoices/route.ts` | Stripe webhook handler for invoice events. Processes `invoice.paid`, `invoice.voided`, `invoice.marked_uncollectible`, `invoice.updated`. Auto-stops sequences on payment detection by cancelling QStash jobs. Ensures the system reacts to real-world payment events in real time. |
| `src/server/billing/plan-limits.ts` | Plan enforcement middleware. Checks usage against tier limits before allowing invoice creation, sequence creation, and source connection. Called by tRPC middleware on relevant mutations. Prevents free tier abuse and drives upgrade conversion. |

---

### Verification

#### Unit Tests
- **Encryption**: round-trip encrypt/decrypt with various inputs, verify different IVs produce different ciphertexts, verify wrong key fails decryption, verify tampered ciphertext fails auth tag verification
- **AI Prompts**: verify correct system prompt selected for each tone+escalation combination (16 combinations), verify user message template populates all variables, verify fallback templates cover all escalation levels
- **Plan Limits**: verify free tier blocks at 5 invoices/month, allows 5th, blocks 6th. Verify Pro allows unlimited. Verify source limit (free: 1, Pro: unlimited). Verify custom sequence creation blocked on free tier
- **Date Calculations**: verify weekend skipping logic (Friday + 2 days = Monday, not Sunday), verify step scheduling with delay_days, verify overdue detection threshold
- **Stripe Status Mapping**: verify all Stripe invoice statuses map correctly to internal statuses

#### Integration Tests
- **Stripe Sync Pipeline**: mock Stripe API → call syncStripeInvoices → verify correct invoices and clients created in DB, verify status mapping, verify idempotent on re-run
- **Enrollment Pipeline**: create invoice → enroll → verify N scheduledJobs created with correct timestamps and QStash message IDs → verify invoice sequenceStatus updated
- **Job Execution Pipeline**: mock QStash callback → verify guard checks (paid invoice rejected, paused client rejected) → verify AI called with correct params → verify Resend called → verify sentReminder created → verify scheduledJob updated
- **Payment Webhook Pipeline**: mock Stripe webhook → verify invoice status updated → verify all pending jobs cancelled → verify client aggregates updated
- **Billing Webhook Pipeline**: mock checkout.session.completed → verify user plan updated → verify new limits apply immediately

#### E2E Tests (Playwright)
- **Onboarding Flow**: sign up → complete 4-step onboarding → arrive at dashboard with correct defaults
- **Manual Invoice Flow**: navigate to invoices → add invoice via dialog → verify appears in list → start sequence → verify timeline shows on detail page → mark as paid → verify sequence stopped
- **Stripe Connection Flow**: navigate to settings → integrations → paste test API key → verify connection succeeds → verify invoices appear after sync
- **Sequence Preview**: navigate to sequences → click system sequence → preview full sequence → verify 5 emails displayed with escalating tone
- **Billing Upgrade**: navigate to billing → click upgrade to Pro → verify redirected to Stripe Checkout (mock)

#### Manual QA Checklist
- [ ] Sign up fresh account, verify onboarding redirects
- [ ] Complete onboarding with each tone option, verify tone persists
- [ ] Connect Stripe test account with restricted key, verify sync pulls invoices
- [ ] Verify overdue invoices auto-detected and badged correctly
- [ ] Start sequence on overdue invoice, verify timeline appears with correct dates
- [ ] Trigger "Send Now" on first pending step, verify email received in test inbox
- [ ] Verify email content matches selected tone and escalation level
- [ ] Send Stripe `invoice.paid` webhook, verify sequence auto-stops and invoice status updates
- [ ] Verify free tier limit: 6th invoice in same month is rejected with upgrade prompt
- [ ] Verify pause/resume client pauses all their invoice sequences
- [ ] Verify custom voice samples appear in AI-generated email style
- [ ] Add manual invoice with all fields, verify client auto-created
- [ ] Test dashboard with 0 invoices (empty state), 1 invoice, 50+ invoices
- [ ] Verify aging chart shows correct bucket distribution
- [ ] Verify activity feed shows chronological events across all invoices
- [ ] Test on mobile Safari and Chrome (responsive layout)
- [ ] Verify Stripe billing checkout creates subscription and updates plan
- [ ] Verify Stripe Customer Portal accessible and returns to app correctly


