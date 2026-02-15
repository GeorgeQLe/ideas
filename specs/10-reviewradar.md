# ReviewRadar — Multi-Platform Review Monitoring

## Executive Summary

ReviewRadar aggregates reviews from 20+ platforms (Google, Yelp, G2, App Store, Trustpilot, etc.) into a single dashboard. It alerts you in real-time, uses AI to draft responses, tracks sentiment trends, monitors competitors, and helps you proactively request reviews from happy customers.

---

## Problem Statement

**The pain:**
- Businesses get reviews on 5-15 different platforms and can't monitor them all
- Negative reviews sit unanswered for days or weeks, damaging reputation
- Responding to reviews is time-consuming; each platform has its own interface
- No way to see overall reputation trends without manually checking each platform
- Competitors' review strategies are invisible

**The impact:**
- 93% of consumers say online reviews impact their purchasing decisions
- Businesses responding to reviews earn 12% more revenue (Harvard Business Review)
- A 1-star increase on Yelp leads to 5-9% revenue increase

---

## Target Users

### Primary Personas

**1. Maria — Local Business Owner (restaurant)**
- Reviews on Google, Yelp, TripAdvisor, DoorDash
- No time to check 4 platforms daily
- Needs: alerts, fast response, reputation tracking

**2. Jason — SaaS Marketing Manager**
- Reviews on G2, Capterra, Product Hunt, App Store, Trustpilot
- Needs to maintain 4.5+ star rating for sales enablement
- Needs: trends, AI responses, review generation campaigns

**3. Karen — Multi-Location Franchise Manager**
- 12 restaurant locations, each with Google/Yelp/TripAdvisor listings
- Needs consistent response quality across all locations
- Needs: team response management, location comparison, templates

---

## Core Features

### F1: Review Aggregation
- **Platforms supported:**
  - General: Google Business Profile, Yelp, Trustpilot, BBB, Facebook
  - Software: G2, Capterra, Product Hunt, GetApp, TrustRadius
  - Mobile: Apple App Store, Google Play Store
  - Travel/Food: TripAdvisor, DoorDash, Uber Eats, OpenTable
  - E-commerce: Amazon, Shopify product reviews
  - Specialty: Glassdoor (employer), Zillow (real estate), Healthgrades (healthcare)
- Real-time monitoring (check frequency: 15 min to 1 hour depending on platform)
- Historical import of existing reviews
- Unified star rating normalization (different scales mapped to 5-star)

### F2: Real-Time Alerts
- New review notification via: email, Slack, SMS, Microsoft Teams, push notification
- Configurable alert rules:
  - All new reviews
  - Only negative reviews (below X stars)
  - Only reviews mentioning specific keywords
  - Only from specific platforms
- Alert routing: negative reviews → manager, positive reviews → marketing
- Daily/weekly digest option instead of real-time

### F3: AI-Powered Response Drafting
- One-click AI response generation for any review
- Tone options: grateful, professional, empathetic, apologetic
- Response includes:
  - Acknowledgment of specific points mentioned in review
  - Appropriate apology or thanks
  - Invitation to continue conversation offline (for negative reviews)
  - Personalization with reviewer name and details mentioned
- Response templates library for common scenarios
- Team review: draft responses can be approved before posting
- Direct posting to some platforms (Google, Facebook) via API
- Copy-to-clipboard for platforms without API access

### F4: Sentiment Analysis & Trends
- Per-platform and aggregate sentiment score over time
- Topic extraction: what are people praising/complaining about most
- Word cloud of frequently mentioned topics
- Sentiment comparison across platforms
- Trend alerts: "Your Google rating dropped 0.2 stars this month"
- Breakdown by star rating: 5-star topics vs 1-star topics

### F5: Competitor Monitoring
- Add competitor businesses to track their reviews
- Side-by-side rating comparison
- Competitor review volume trends
- Sentiment comparison on shared topics
- Alert when competitor rating changes significantly
- "Competitor weakness" report: topics where competitors score poorly

### F6: Review Request Campaigns
- Request reviews from happy customers via email or SMS
- Timing triggers: X days after purchase, after positive support interaction
- Landing page with links to preferred review platforms
- Negative sentiment gate: if customer indicates low satisfaction, redirect to private feedback form instead of public review
- Drip sequences: initial request → reminder after 3 days
- Campaign analytics: sent, opened, reviews generated
- Integration with CRM for customer list targeting

### F7: Embeddable Review Widget
- Display your best reviews on your website
- Configurable: platform filter, minimum rating, layout (carousel, grid, list)
- Auto-updating with new positive reviews
- Custom styling to match your brand
- Rich snippets for SEO (structured data markup)
- Lightweight embed (<10KB)

### F8: Reporting
- Monthly reputation report (PDF export)
- Key metrics: average rating, review volume, response rate, response time
- Platform breakdown
- Sentiment trends
- Top positive/negative themes
- Competitor comparison section
- Scheduled email delivery to stakeholders

### F9: Multi-Location Management
- Location hierarchy (brand → region → location)
- Per-location dashboards and alerts
- Location comparison: rating, volume, sentiment
- Centralized response templates with location-specific customization
- Roll-up reporting across all locations
- Per-location team assignment

---

## Technical Architecture

### Tech Stack

| Layer | Technology |
|-------|-----------|
| Frontend | Next.js 14+, Tailwind CSS, Recharts |
| Backend | Next.js API Routes, tRPC |
| Database | PostgreSQL (Neon) |
| Scraping / Import | Playwright workers (for platforms without API), official APIs where available |
| AI | OpenAI GPT-4o-mini (response generation, sentiment analysis) |
| Queue | BullMQ + Redis (scheduled review checks) |
| Email/SMS | Resend (email), Twilio (SMS) |
| Auth | Clerk |
| Hosting | Vercel (app), Fly.io (scraping workers) |

### Data Model

```
Organization
├── id, name, slug, plan
│
├── Location[]
│   ├── id, org_id, name, address
│   ├── parent_location_id (for hierarchy)
│   │
│   └── ReviewSource[]
│       ├── id, location_id, platform (google/yelp/g2/...)
│       ├── external_url, external_id
│       ├── credentials (JSON, encrypted, if API-based)
│       ├── last_checked_at, check_interval_minutes
│       └── current_rating, total_review_count
│
├── Competitor[]
│   ├── id, org_id, name
│   └── CompetitorSource[] (same as ReviewSource)
│
├── Review[]
│   ├── id, source_id, location_id, org_id
│   ├── platform, external_review_id
│   ├── reviewer_name, reviewer_avatar_url
│   ├── rating (1-5 normalized), raw_rating
│   ├── text, language
│   ├── sentiment_score (-1 to 1), sentiment_label
│   ├── topics (text[]), keywords (text[])
│   ├── published_at, fetched_at
│   ├── is_competitor (boolean)
│   │
│   └── ReviewResponse[]
│       ├── id, review_id, text
│       ├── ai_generated, ai_draft (original AI text)
│       ├── posted (boolean), posted_at
│       ├── posted_by_user_id
│       └── created_at
│
├── ReviewCampaign[]
│   ├── id, org_id, name
│   ├── target_platforms[], landing_page_url
│   ├── email_template, sms_template
│   ├── status (draft/active/paused/completed)
│   │
│   └── CampaignRecipient[]
│       ├── id, campaign_id, name, email, phone
│       ├── sent_at, opened_at, clicked_at
│       └── review_generated (boolean)
│
└── AlertRule[]
    ├── id, org_id, conditions (JSON)
    ├── channels (email/slack/sms/teams)
    └── recipients[]
```

---

## UI/UX — Key Screens

### 1. Review Feed
- Unified stream of all reviews, newest first
- Each review: platform icon, stars, reviewer name, text, date, sentiment badge
- "Respond" button opens AI response panel
- Filter by: platform, rating, sentiment, location, date, responded/unresponded
- Quick stats at top: new reviews today, unresponded count, average rating

### 2. AI Response Panel
- Review text on the left
- AI-generated response on the right
- Tone selector (grateful/professional/empathetic/apologetic)
- Edit response text
- "Regenerate" button for alternative
- "Post" button (if API supported) or "Copy" button
- History of past responses for this reviewer

### 3. Analytics Dashboard
- Overall rating trend line (all platforms combined)
- Review volume bar chart over time
- Sentiment distribution pie chart
- Top positive themes word cloud
- Top negative themes word cloud
- Platform comparison table (rating, volume, response rate)
- Competitor comparison (if configured)

### 4. Campaign Builder
- Campaign name, target platforms
- Email/SMS template editor with merge fields
- Recipient list (upload CSV or select from CRM)
- Negative sentiment gate configuration
- Preview landing page
- Campaign analytics after launch

### 5. Location Manager
- List/grid of all locations with ratings and review counts
- Location comparison chart
- Click into location for dedicated dashboard
- Assign team members per location

---

## Monetization

### Free Tier
- 1 location, 2 platforms
- Real-time alerts (email only)
- 30-day review history
- No AI responses

### Pro — $29/month
- 1 location, 5 platforms
- AI response drafting (50/month)
- Sentiment analysis
- Slack integration
- 1-year history
- Review widget

### Business — $69/month
- 5 locations, unlimited platforms
- Unlimited AI responses
- Competitor monitoring (3 competitors)
- Review request campaigns
- All integrations
- Reporting (PDF export)
- Team collaboration (5 users)

### Enterprise — $199/month
- Unlimited locations
- Unlimited competitors
- Multi-location management
- SSO
- API access
- Custom integrations
- Priority support
- Dedicated account manager

---

## Go-to-Market Strategy

- Target local businesses first (restaurants, healthcare, service businesses)
- SEO: "online review management", "respond to Google reviews", "review monitoring tool"
- Content: "How to respond to negative reviews (with templates)"
- Free review audit tool (enter your business, get a reputation report)
- Partner with local marketing agencies
- Google Business Profile integration as primary hook
- Expand to SaaS/software review management as second market

---

## Success Metrics

| Metric | Target (Month 6) | Target (Month 12) |
|--------|-------------------|---------------------|
| Organizations | 200 | 800 |
| Reviews monitored | 100K | 500K |
| AI responses generated | 5,000/mo | 25,000/mo |
| Paying customers | 80 | 300 |
| MRR | $4,000 | $18,000 |
| Average response time (customer reviews) | <4 hours | <2 hours |

---

## MVP Scope (v1.0)

### In Scope
- Google Business Profile + Yelp + G2 monitoring
- Real-time email/Slack alerts
- AI response drafting
- Basic sentiment analysis
- Review feed with filtering
- Rating trends chart

### Out of Scope (v1.1+)
- Competitor monitoring
- Review request campaigns
- Review widget
- Multi-location management
- SMS alerts
- Reporting/PDF export
- Most platforms beyond initial 3

### MVP Timeline: 6-7 weeks
- Week 1-2: Review scraping/API workers for Google, Yelp, G2
- Week 3: Review storage, alert system
- Week 4: AI response generation, sentiment analysis
- Week 5-6: Dashboard UI, review feed, analytics
- Week 7: Billing, onboarding, polish, launch
