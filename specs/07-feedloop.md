# FeedLoop — Customer Feedback Aggregator

## Executive Summary

FeedLoop aggregates customer feedback from every channel — support tickets, reviews, surveys, social media, and sales calls — into one unified view. AI categorizes, deduplicates, and ranks feedback by impact so product teams can see exactly what customers want most, and close the loop when features ship.

---

## Problem Statement

**The pain:**
- Customer feedback is fragmented across 5-10+ tools (Intercom, Zendesk, G2, email, Slack, Twitter, sales calls)
- Product managers spend hours manually reading tickets, tagging requests, and maintaining spreadsheets
- The same feature is requested 50 times across different channels but looks like 50 different requests
- High-value customer feedback gets lost in the noise of small account requests
- When a feature ships, nobody tells the customers who asked for it — missing a retention and delight opportunity

**The cost:**
- Product teams build the wrong things because they can't see the true demand signal
- Customers churn because they feel unheard
- Sales deals stall because prospects hear "it's on the roadmap" with no follow-up

---

## Target Users

### Primary Personas

**1. Hannah — Product Manager**
- Manages roadmap for a B2B SaaS product with 500 customers
- Gets feedback from CS, sales, support, and directly from users
- Maintains a messy Notion database of feature requests
- Needs: single source of truth for what customers want, ranked by impact

**2. Derek — VP of Product**
- Oversees 3 product managers and a portfolio of products
- Needs to justify roadmap decisions to executives with data
- Needs: aggregated feedback analytics, revenue-weighted prioritization

**3. Sofia — Customer Success Manager**
- Talks to 50 customers regularly, captures their requests in Intercom notes
- Wants to tell customers when their requested feature ships
- Needs: easy way to log feedback, get notified when items are addressed

---

## Core Features

### F1: Multi-Channel Feedback Ingestion
- **Intercom**: import conversations tagged as feedback/feature request
- **Zendesk**: import tickets by tag, custom field, or macro
- **HubSpot**: import deal notes and feedback properties
- **Slack**: dedicated channel where team posts customer quotes, or bot captures reactions
- **Email**: forward feedback emails to a dedicated address (feedback@yourco.feedloop.io)
- **G2/Capterra reviews**: automated import
- **App Store / Play Store reviews**: automated import
- **Twitter/X mentions**: keyword monitoring
- **Survey tools** (Typeform, SurveyMonkey): webhook integration
- **Sales call transcripts** (Gong, Fireflies): automated extraction
- **Manual entry**: web form for team members to log feedback
- **API**: programmatic submission from any source
- **Browser extension**: highlight text on any page and submit to FeedLoop
- Each feedback item preserves: source, customer identity, verbatim quote, date, submitter

### F2: AI Categorization & Tagging
- Auto-categorize: Feature Request, Bug Report, Praise, Complaint, Question, Churn Risk
- Auto-tag with product area (e.g., "onboarding", "billing", "API", "mobile app")
- Sentiment analysis: positive, neutral, negative (with score)
- Auto-extract the core "ask" from verbose feedback
- Suggest existing feature request to link to (deduplication)
- Confidence score on all AI decisions
- Human override on any categorization

### F3: Deduplication & Clustering
- AI groups similar feedback into clusters (e.g., 47 people asked for "dark mode" in different words)
- Each cluster becomes a "Feature Request" with:
  - Canonical title and description
  - All linked feedback items (evidence)
  - Total request count
  - List of requesting customers with account details
- Merge/split clusters manually
- New incoming feedback auto-matched to existing clusters

### F4: Priority Scoring
- Composite score based on configurable weights:
  - **Frequency**: how many customers requested it
  - **Revenue weight**: total ARR of requesting customers
  - **Recency**: recent requests weighted higher
  - **Sentiment urgency**: frustrated customers weighted higher
  - **Strategic alignment**: manual boost for strategic initiatives
- Customizable scoring formula
- Sortable ranked list of all feature requests
- "Impact vs Effort" matrix view (effort is manually estimated)

### F5: Customer Voting Board (Optional Public)
- Public-facing portal where customers can:
  - Browse existing feature requests
  - Upvote requests they care about
  - Submit new requests
  - Comment and add context
- Internal-only mode (team voting)
- Custom branding and domain
- SSO for customer authentication
- Status labels: Under Review, Planned, In Progress, Shipped, Won't Do
- Configurable: which requests are visible publicly

### F6: Close-the-Loop Notifications
- When a feature request status changes to "Shipped":
  - Auto-notify all customers who requested it (email)
  - Update the voting board
  - Optionally notify via Intercom/Zendesk as well
- Customizable notification email template
- Include link to changelog or release notes
- Track: notification sent, opened, feature adopted
- "You asked, we built" marketing content generation

### F7: Roadmap Integration
- Export feature requests to:
  - **Jira**: create epic/story with linked feedback
  - **Linear**: create issue with context
  - **Notion**: add to roadmap database
  - **GitHub Issues**: create issue
- Bi-directional sync: status changes in Jira/Linear update FeedLoop
- Attach feedback evidence to roadmap items
- "Customers waiting" count on roadmap items

### F8: Analytics & Reporting
- Feedback volume over time (by source, category, sentiment)
- Top requested features ranking
- Customer health signals (customers with most negative feedback)
- Response time to feedback
- Close-the-loop rate (% of requests that get a response)
- Revenue at risk (total ARR of customers with unaddressed complaints)
- Team performance (who logs the most feedback, response times)
- Exportable reports (PDF, CSV)

### F9: Customer Intelligence
- Per-customer feedback profile
- All feedback from a single customer across all channels
- Customer health score based on feedback sentiment trends
- Integration with CRM: enrich with ARR, plan, account owner
- Churn risk indicator based on feedback patterns

---

## Technical Architecture

### Tech Stack

| Layer | Technology |
|-------|-----------|
| Frontend | Next.js 14+, Tailwind CSS, Recharts, Radix UI |
| Backend | Next.js API Routes, tRPC |
| Database | PostgreSQL (Neon) |
| Vector DB | pgvector (for deduplication/clustering embeddings) |
| AI | OpenAI GPT-4o (categorization, extraction), text-embedding-3-small (clustering) |
| Queue | BullMQ + Redis |
| Auth | Clerk |
| Email | Resend |
| Hosting | Vercel |

### Data Model

```
Organization
├── id, name, slug, plan
├── scoring_config (JSON: weights for frequency, revenue, recency, sentiment)
│
├── FeedbackSource[]
│   ├── id, org_id, type (intercom/zendesk/slack/email/g2/manual/api)
│   ├── config (JSON, encrypted), status (active/paused/error)
│   └── last_synced_at
│
├── Customer[]
│   ├── id, org_id, name, email, company
│   ├── arr, plan_tier, account_owner
│   ├── health_score, total_feedback_count
│   └── external_ids (JSON: intercom_id, zendesk_id, hubspot_id)
│
├── FeedbackItem[]
│   ├── id, org_id, customer_id, source_id
│   ├── raw_text, extracted_ask (AI-generated)
│   ├── category (feature_request/bug/praise/complaint/question)
│   ├── sentiment (positive/neutral/negative), sentiment_score
│   ├── tags (text[])
│   ├── feature_request_id (→ FeatureRequest, nullable)
│   ├── source_url, source_metadata (JSON)
│   ├── submitted_by_user_id (team member who logged it)
│   ├── embedding (vector[1536])
│   └── created_at
│
├── FeatureRequest[] (clustered)
│   ├── id, org_id, title, description
│   ├── status (new/under_review/planned/in_progress/shipped/wont_do)
│   ├── priority_score, manual_effort_estimate
│   ├── request_count, total_arr_requesting
│   ├── is_public (visible on voting board)
│   ├── external_issue_id (jira/linear/github)
│   ├── external_issue_url
│   ├── shipped_at, closed_at
│   │
│   ├── FeedbackItem[] (linked evidence)
│   ├── Vote[] (from voting board)
│   │   ├── id, feature_request_id, customer_id, created_at
│   │   └── comment (optional)
│   └── FeatureRequestComment[]
│       ├── id, feature_request_id, user_id/customer_id
│       ├── body, created_at
│       └── is_internal
│
└── Notification[]
    ├── id, org_id, feature_request_id, customer_id
    ├── type (status_change/shipped/comment)
    ├── channel (email/intercom/zendesk)
    ├── sent_at, opened_at
    └── status (pending/sent/opened/failed)
```

---

## UI/UX — Key Screens

### 1. Feedback Inbox
- Stream of incoming feedback items, newest first
- Each item: customer name, source icon, verbatim quote, AI category badge, sentiment indicator
- Quick actions: link to existing request, create new request, dismiss, tag
- Filters: source, category, sentiment, date, customer
- "AI suggested match" banner on items similar to existing requests

### 2. Feature Request Board
- Kanban view: New → Under Review → Planned → In Progress → Shipped
- Or ranked list view sorted by priority score
- Each card: title, request count, total ARR, top customer logos
- Click into card for full detail with linked feedback evidence

### 3. Feature Request Detail
- Title, description, status, priority score
- "Evidence" tab: all linked feedback items with source and verbatim quotes
- "Customers" tab: list of requesting customers with ARR
- "Activity" tab: status changes, comments, notifications sent
- Actions: change status, link to Jira/Linear, notify customers, edit

### 4. Voting Board (Public)
- Clean, branded portal
- Feature list with vote counts
- Status badges
- Submit new request form
- Search and filter

### 5. Analytics Dashboard
- Feedback volume chart (by source, over time)
- Top 10 feature requests by score
- Sentiment trend line
- Revenue at risk metric
- Close-the-loop rate
- Source breakdown pie chart

### 6. Customer Profile
- Customer details + CRM data
- All feedback from this customer (timeline)
- Feature requests they've voted on
- Health score with trend
- Notes from account team

---

## Monetization

### Starter — $49/month
- 3 feedback sources
- 500 feedback items/month
- AI categorization
- Basic deduplication
- 1 user

### Growth — $99/month
- Unlimited sources
- 5,000 feedback items/month
- Advanced clustering and deduplication
- Priority scoring
- Voting board
- Close-the-loop notifications
- Jira/Linear integration
- 5 users

### Enterprise — $249/month
- Unlimited everything
- Customer intelligence
- Revenue-weighted scoring (CRM integration)
- Advanced analytics
- API access
- SSO
- Unlimited users
- Dedicated support

---

## Go-to-Market Strategy

- Launch targeting product managers (ProductHunt, Mind the Product, Lenny's Newsletter audience)
- Content: "We analyzed 10,000 pieces of customer feedback and here's what we learned"
- SEO: "customer feedback tool", "feature request management", "product feedback"
- Free feedback analysis report (submit your Intercom export, get insights)
- Partner with PM coaching/community platforms
- Sponsor product management conferences and meetups

---

## Success Metrics

| Metric | Target (Month 6) | Target (Month 12) |
|--------|-------------------|---------------------|
| Organizations | 100 | 400 |
| Feedback items processed | 50,000/mo | 300,000/mo |
| Feature requests created | 5,000 | 25,000 |
| Close-the-loop notifications sent | 2,000 | 15,000 |
| MRR | $8,000 | $35,000 |
| Paying customers | 60 | 200 |

---

## MVP Scope (v1.0)

### In Scope
- Intercom + Zendesk integration
- Manual feedback entry
- AI categorization and tagging
- Basic deduplication (AI-suggested matches)
- Feature request board (kanban + list)
- Priority scoring (frequency + recency)
- Email notifications when status changes

### Out of Scope (v1.1+)
- Public voting board
- CRM integration (revenue weighting)
- Advanced clustering
- Jira/Linear sync
- Analytics dashboard
- Customer intelligence profiles
- Close-the-loop automation
- G2/App Store review import

### MVP Timeline: 7-8 weeks
- Week 1-2: Data model, Intercom/Zendesk integrations, feedback ingestion
- Week 3: AI categorization pipeline, embedding generation
- Week 4: Deduplication/matching system
- Week 5: Feature request board UI, priority scoring
- Week 6: Feedback inbox, linking workflow
- Week 7: Notifications, basic dashboard
- Week 8: Billing, onboarding, polish, launch
