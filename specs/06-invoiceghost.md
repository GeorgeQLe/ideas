# InvoiceGhost — Automated Payment Follow-Up

## Executive Summary

InvoiceGhost connects to your invoicing tool and automatically sends polite, escalating payment reminders to clients with overdue invoices. It takes the awkwardness out of chasing payments by handling follow-ups on a customizable schedule with AI-written emails that match your tone.

---

## Problem Statement

**The pain:**
- Freelancers and small businesses are owed an average of $6,000+ in unpaid invoices at any time
- Following up on overdue payments is uncomfortable — people avoid it and lose money
- Manual follow-up is inconsistent: sometimes you remember, sometimes you don't
- Most invoicing tools send one reminder at best, then nothing
- Late payments cause cash flow crises that threaten business survival

**The numbers:**
- 29% of freelancer invoices are paid late (Freelancers Union)
- Small businesses spend 14+ hours/month chasing payments (QuickBooks)
- $825B in outstanding invoices in the US alone

---

## Target Users

### Primary Personas

**1. Nina — Freelance Designer**
- Invoices 5-10 clients per month
- Uses FreshBooks for invoicing
- Hates writing "friendly reminder" emails
- Needs: automated reminders that don't make her look desperate

**2. Raj — Agency Owner**
- Runs a 6-person development agency
- Has 20-30 active invoices at any time
- Lost $15K last year to unpaid invoices
- Needs: escalating sequences, payment links, team visibility

**3. Claire — Small Business Owner**
- Runs a consulting firm, invoices through QuickBooks
- Doesn't have a bookkeeper or collections person
- Needs: set-and-forget system, professional communication

---

## Core Features

### F1: Invoice Source Integration
- **Stripe Invoices**: real-time sync via API
- **QuickBooks Online**: OAuth integration
- **FreshBooks**: OAuth integration
- **Xero**: OAuth integration
- **Wave**: API integration
- **CSV import**: for manual invoices or unsupported tools
- **Manual entry**: add individual invoices directly
- Real-time status sync (paid, partial, overdue, void)
- Auto-detect overdue invoices on first connection
- Historical import for invoices within last 12 months

### F2: Smart Reminder Sequences
- Pre-built sequences:
  - **Gentle**: Day 1 (due date reminder) → Day 3 → Day 7 → Day 14 → Day 30
  - **Standard**: Day 1 → Day 5 → Day 14 → Day 30 → Day 45
  - **Firm**: Day 1 → Day 3 → Day 7 → Day 14 → Day 21 → Day 30
- Custom sequence builder (set your own timing and escalation)
- Escalation levels per step: friendly → firm → urgent → final notice
- Per-client overrides (different sequences for different clients)
- Pause sequence for specific invoices or clients
- Stop sequence automatically when payment is received
- Maximum reminders cap (don't send more than N emails total)
- Weekend/holiday skip option

### F3: AI Email Generation
- AI writes reminder emails matching your configured tone
- Tone options: friendly, professional, direct, empathetic
- Emails include: invoice number, amount, due date, days overdue
- Escalating urgency across the sequence
- Personalization with client name and project details
- Preview every email before the sequence starts
- Edit any individual email in the sequence
- "Write like me" training: paste 2-3 of your previous emails to match your voice
- Multi-language support (English, Spanish, French, German)

### F4: Payment Links
- Auto-generate payment links in every reminder
- Stripe Payment Links integration
- PayPal.me links
- Bank transfer details insertion
- Custom payment instructions
- "Pay Now" landing page with:
  - Invoice summary
  - Multiple payment options
  - Partial payment support
  - Receipt generation on payment

### F5: Invoice Dashboard
- Overview: total outstanding, total overdue, total collected this month
- Invoice list with status badges (on time, overdue, in sequence, paid, written off)
- Sort by: amount, days overdue, client, next reminder date
- Client view: all invoices grouped by client with payment history
- Aging report: 0-30, 31-60, 61-90, 90+ days
- Cash flow forecast based on outstanding invoices

### F6: Activity & Communication Log
- Full history per invoice: every email sent, opened, clicked, replied
- Email open tracking (optional)
- Payment link click tracking
- Client response detection (auto-pause if client replies)
- Timeline view per invoice
- Exportable activity report

### F7: Late Fees & Interest
- Configurable late fee policy (flat fee or percentage)
- Auto-calculate interest on overdue amount
- Include late fee information in reminder emails
- Late fee waiver option per invoice
- Compliance with local regulations (configurable)

### F8: Notifications & Reporting
- Daily digest: invoices becoming overdue today
- Weekly summary: collections progress, outstanding total
- Alert when a high-value invoice hits 30+ days overdue
- Monthly collections report (total recovered, success rate)
- Email to accountant/bookkeeper option

### F9: Team Features (Agency Plan)
- Multiple team members with role-based access
- Assign invoices to team members
- Client portfolio management
- White-label emails (send from your domain)
- Multi-entity support (multiple businesses)

---

## Technical Architecture

### Tech Stack

| Layer | Technology |
|-------|-----------|
| Frontend | Next.js 14+, Tailwind CSS, Recharts |
| Backend | Next.js API Routes, tRPC |
| Database | PostgreSQL (Neon) |
| Queue / Scheduler | BullMQ + Redis (Upstash) for scheduled email jobs |
| AI | OpenAI GPT-4o-mini (email generation) |
| Email Sending | Resend (with custom domain support) |
| Auth | Clerk |
| Payments | Stripe (billing) |
| Hosting | Vercel |

### Data Model

```
User
├── id, email, name, business_name, plan
├── tone_preference, custom_voice_samples (text[])
├── default_sequence_id
│
├── InvoiceSource[]
│   ├── id, user_id, type (stripe/quickbooks/freshbooks/xero/manual)
│   ├── credentials (encrypted JSON)
│   ├── last_synced_at, sync_status
│   └── auto_enroll_overdue (boolean)
│
├── Client[]
│   ├── id, user_id, name, email, company
│   ├── custom_sequence_id (nullable, overrides default)
│   ├── is_paused, notes
│   └── total_outstanding, total_paid, avg_days_to_pay
│
├── Invoice[]
│   ├── id, user_id, client_id, source_id
│   ├── external_invoice_id (from source)
│   ├── invoice_number, amount, currency
│   ├── issued_date, due_date
│   ├── status (draft/sent/overdue/in_sequence/paid/partial/void/written_off)
│   ├── amount_paid, paid_at
│   ├── payment_link_url
│   ├── late_fee_amount, late_fee_applied
│   ├── sequence_status (pending/active/paused/completed/stopped)
│   └── created_at, updated_at
│
├── ReminderSequence[]
│   ├── id, user_id, name, is_default
│   │
│   └── SequenceStep[]
│       ├── id, sequence_id, step_number
│       ├── delay_days (from due date)
│       ├── escalation_level (friendly/firm/urgent/final)
│       ├── email_template (nullable, custom override)
│       └── include_late_fee (boolean)
│
├── SentReminder[]
│   ├── id, invoice_id, step_id
│   ├── email_subject, email_body
│   ├── sent_at, opened_at, clicked_at
│   ├── status (scheduled/sent/opened/clicked/replied/bounced)
│   └── ai_generated (boolean)
│
└── ScheduledJob[]
    ├── id, invoice_id, step_id
    ├── scheduled_for, executed_at
    └── status (pending/executed/cancelled)
```

---

## UI/UX — Key Screens

### 1. Dashboard
- Large cards: Total Outstanding, Total Overdue, Collected This Month, Recovery Rate
- Aging chart (stacked bar: 0-30, 31-60, 61-90, 90+)
- "Needs Attention" list: invoices approaching escalation thresholds
- Recent activity feed: emails sent, payments received, sequences started

### 2. Invoice List
- Sortable/filterable table
- Status badges with color coding
- Next reminder date column
- Quick actions: pause sequence, mark as paid, write off
- Bulk actions: start sequence, pause, export

### 3. Invoice Detail
- Invoice info (number, amount, client, dates)
- Reminder timeline (visual): completed steps, upcoming steps
- Each step shows: scheduled date, email preview, status (sent/opened/clicked)
- Actions: send now, skip step, pause, edit upcoming email
- Payment history and activity log

### 4. Sequence Builder
- Step list with add/remove/reorder
- Per step: days after due date, escalation level, email preview
- "Preview full sequence" button: shows all emails in order
- Save as template

### 5. Client View
- Client profile with all invoices
- Payment behavior stats: average days to pay, on-time rate
- Override sequence for this client
- Pause/resume all reminders for client
- Notes section

### 6. Settings
- Integration connections (connect/disconnect invoice sources)
- Default sequence configuration
- Tone and voice settings
- Email sender configuration (custom domain)
- Late fee policy
- Notification preferences

---

## Monetization

### Free Tier
- 5 invoices per month
- 1 integration
- Basic sequence (3 reminders)
- InvoiceGhost branding in emails

### Pro — $12/month
- Unlimited invoices
- All integrations
- Custom sequences
- AI email generation
- Payment links
- Email open tracking
- No branding

### Agency — $29/month
- Everything in Pro
- Team members (up to 5)
- White-label emails (custom domain)
- Multi-entity support
- Client portal
- Priority support
- API access

---

## Go-to-Market Strategy

- Launch in freelancer communities (Freelancers Union, r/freelance, Indie Hackers)
- Content: "I recovered $X in unpaid invoices with one tool"
- SEO: "overdue invoice reminder", "payment follow up email template"
- Free email template library (lead magnet)
- Partnership with freelancer platforms (Toptal, Upwork blogs)
- Integration marketplace listings (QuickBooks App Store, Xero Marketplace)
- Referral program: get 1 month free for each referral

---

## Success Metrics

| Metric | Target (Month 6) | Target (Month 12) |
|--------|-------------------|---------------------|
| Active users | 500 | 2,000 |
| Invoices tracked | 5,000 | 25,000 |
| Revenue recovered (all users) | $500K | $3M |
| Reminders sent | 15,000/mo | 75,000/mo |
| Paying customers | 100 | 400 |
| MRR | $1,800 | $7,000 |
| Recovery rate (invoices that get paid after reminders) | 65% | 75% |

---

## Risks & Mitigations

| Risk | Mitigation |
|------|------------|
| Reminder emails go to spam | Use Resend with proper DKIM/SPF, custom domain sending, warm-up |
| AI tone is wrong, damages client relationships | Always preview before sequence starts, human-editable, tone controls |
| Integration APIs change or break | Abstract provider layer, monitoring on sync jobs, fallback to manual |
| Users set and forget, but circumstances change | Client reply detection auto-pauses, dashboard notifications, activity alerts |
| Legal issues with automated collections | Clear terms, late fee compliance by region, users responsible for content |

---

## MVP Scope (v1.0)

### In Scope
- Stripe Invoices integration
- Manual invoice entry
- Pre-built reminder sequences (3 templates)
- AI-generated reminder emails
- Basic dashboard (outstanding, overdue, collected)
- Email sending via Resend
- Auto-stop on payment detection

### Out of Scope (v1.1+)
- QuickBooks, FreshBooks, Xero integrations
- Custom sequence builder
- Late fees
- Payment links and "Pay Now" pages
- Email open/click tracking
- White-label sending
- Team features
- Client portal

### MVP Timeline: 5-6 weeks
- Week 1: Auth, data model, Stripe integration, invoice sync
- Week 2: Reminder sequence engine, scheduling system
- Week 3: AI email generation, email sending pipeline
- Week 4: Dashboard UI, invoice management
- Week 5: Auto-stop on payment, manual invoice entry
- Week 6: Billing, onboarding, polish, launch
