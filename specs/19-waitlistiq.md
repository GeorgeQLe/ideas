# WaitlistIQ — Smart Waitlist & Launch Manager

## Executive Summary

WaitlistIQ is an all-in-one pre-launch platform that combines landing page building, waitlist management with a referral program, email drip sequences, and launch analytics. Founders describe their product, WaitlistIQ generates a conversion-optimized landing page, and the built-in referral system turns early signups into viral ambassadors.

---

## Problem Statement

**The pain:**
- Pre-launch founders cobble together 3-5 tools: landing page (Carrd), email list (Mailchimp), referral system (custom code or Viral Loops), analytics (Google Analytics)
- Each tool costs money and requires integration work
- Building a referral-powered waitlist from scratch takes weeks of engineering
- Most pre-launch pages convert poorly because founders aren't landing page designers
- No visibility into which channels and messages drive the most signups

---

## Target Users

### Primary Personas

**1. Alex — Indie Hacker**
- Building a SaaS product, wants to validate demand before launching
- Needs: fast landing page, waitlist with referral incentive, zero engineering

**2. Maya — Startup Founder (pre-seed)**
- Preparing for Product Hunt launch, needs to build email list
- Wants social proof ("2,500 people are waiting") for investor conversations
- Needs: professional landing page, referral viral loop, launch blast email

**3. Chris — Product Manager at Growth-Stage Company**
- Launching a new product line, wants to test messaging and build anticipation
- Needs: A/B testing, analytics, integration with existing marketing stack

---

## Core Features

### F1: AI Landing Page Builder
- Describe your product in 2-3 sentences → AI generates a full landing page
- Page sections:
  - Hero (headline, subheadline, email capture, social proof counter)
  - Problem/Solution
  - Features/Benefits (3-4 cards)
  - Social proof (testimonials, logos, or waitlist count)
  - FAQ (AI-generated from product description)
  - Final CTA
- Template library (10+ designs: minimal, bold, gradient, dark, startup, product)
- Visual editor: click to edit any text, rearrange sections, change images
- Responsive by default (mobile-optimized)
- Custom domain support (CNAME)
- Hosted on waitlistiq.io/your-product by default
- AI-generated OG image for social sharing
- SEO meta tags auto-configured

### F2: Waitlist with Position Tracking
- Email capture form (name + email, or just email)
- Unique waitlist position per signup (#47 of 2,312)
- Position displayed on confirmation page and in welcome email
- Position updates via email when they move up (from referrals)
- Anti-fraud: email verification, rate limiting, disposable email detection
- Custom fields option (role, company, interest)
- Waitlist cap option (first 1,000 only, then closed)

### F3: Referral Program
- Each signup gets unique referral link
- Referral dashboard: "You've referred 5 people, moved up 25 spots"
- Configurable rewards:
  - Move up X positions per referral
  - Unlock tiers: 3 referrals = early access, 5 = lifetime discount, 10 = founding member
  - Custom rewards per tier
- Social sharing buttons: Twitter, LinkedIn, Facebook, WhatsApp, copy link
- Pre-written share messages (editable)
- Referral leaderboard (top referrers)
- Fraud detection: same IP, disposable emails, suspicious patterns
- Viral coefficient tracking

### F4: Email Drip Sequences
- Pre-built sequences:
  - **Welcome**: immediate — "You're #X on the waitlist!"
  - **Day 2**: "Share with friends to move up"
  - **Day 7**: "Here's a sneak peek of what we're building"
  - **Day 14**: "You've moved to position #X"
  - **Pre-launch**: "We're launching tomorrow!"
  - **Launch day**: "You're in! Here's your access"
- Custom sequence builder: add/remove/reorder emails
- Rich text email editor with merge fields ({first_name}, {position}, {referral_link})
- AI email copy generation based on product description
- A/B test subject lines
- Deliverability: DKIM, SPF, dedicated IP option

### F5: Social Proof Widgets
- Embeddable counter: "2,847 people are waiting"
- Styles: minimal text, badge, animated counter, notification popup
- Real-time updating
- Customizable design (colors, font)
- Embed on any website, blog, or social profile
- "Recent signup" popup notification (last person to join)

### F6: A/B Testing
- Test different headlines, hero images, page layouts
- Traffic split (50/50 or custom)
- Conversion rate tracking per variant
- Statistical significance calculator
- Auto-select winner
- Test subject lines in email sequences

### F7: Analytics Dashboard
- Total signups, daily signups trend
- Conversion rate (page views → signups)
- Traffic sources breakdown (direct, social, referral, organic)
- UTM parameter tracking
- Referral metrics: viral coefficient, referral conversion rate, top referrers
- Email metrics: open rate, click rate per sequence email
- Geographic breakdown
- Device breakdown (mobile vs desktop)
- Funnel: page view → form start → signup → referral share → referral conversion

### F8: Launch Day Tools
- **Launch blast email**: send to entire waitlist
- **Batch invites**: let people in waves (first 100, then 200, etc.)
- **Access code generation**: unique invite codes per user
- **Priority queue**: referral leaders get access first
- **Countdown timer**: embed on landing page
- **Product Hunt integration**: align launch timing, add PH badge

### F9: Integrations
- **Email**: export list to Mailchimp, ConvertKit, Resend
- **CRM**: push signups to HubSpot, Salesforce
- **Analytics**: Google Analytics, Mixpanel events
- **Webhook**: POST signup data to any URL
- **Zapier**: connect to 5000+ apps
- **Slack**: notification on signups (milestones and individual)

---

## Technical Architecture

### Tech Stack

| Layer | Technology |
|-------|-----------|
| Frontend | Next.js 14+, Tailwind CSS, Framer Motion |
| Landing Pages | Next.js static generation + ISR (Incremental Static Regeneration) |
| Backend | Next.js API Routes, tRPC |
| Database | PostgreSQL (Neon) |
| AI | OpenAI GPT-4o (page generation, email copy) |
| Email | Resend (with custom domain) |
| CDN | Cloudflare (landing pages, edge caching) |
| Auth | Clerk |
| Analytics | Custom + Google Analytics integration |
| Hosting | Vercel |

### Data Model

```
User (product creator)
├── id, email, name, plan
│
├── Project[]
│   ├── id, user_id, name, slug
│   ├── product_description, tagline
│   ├── custom_domain, status (draft/live/launched)
│   │
│   ├── LandingPage
│   │   ├── id, project_id
│   │   ├── template, sections (JSON[])
│   │   ├── custom_css, og_image_url
│   │   ├── seo_title, seo_description
│   │   ├── favicon_url
│   │   └── ab_variants (JSON[])
│   │
│   ├── WaitlistConfig
│   │   ├── id, project_id
│   │   ├── fields (JSON: name, email, custom)
│   │   ├── cap (max signups, nullable)
│   │   ├── require_email_verification
│   │   └── fraud_detection_enabled
│   │
│   ├── ReferralConfig
│   │   ├── id, project_id
│   │   ├── positions_per_referral (e.g., 5)
│   │   ├── tiers (JSON[]: [{referrals, reward_name, reward_description}])
│   │   ├── share_message_twitter, share_message_linkedin
│   │   └── leaderboard_enabled
│   │
│   ├── Subscriber[]
│   │   ├── id, project_id, email, name
│   │   ├── custom_fields (JSON)
│   │   ├── position (integer, recalculated on referrals)
│   │   ├── original_position (at signup time)
│   │   ├── referral_code (unique string)
│   │   ├── referred_by_subscriber_id
│   │   ├── referral_count
│   │   ├── tier_unlocked (which reward tier)
│   │   ├── email_verified, ip_address
│   │   ├── utm_source, utm_medium, utm_campaign
│   │   ├── source (direct/referral/social)
│   │   ├── device, country
│   │   ├── status (waitlisted/invited/active)
│   │   ├── invite_code (generated on invite)
│   │   └── subscribed_at
│   │
│   ├── EmailSequence[]
│   │   ├── id, project_id, name, trigger (signup/referral/manual/launch)
│   │   │
│   │   └── SequenceEmail[]
│   │       ├── id, sequence_id, subject, body_html
│   │       ├── delay_hours (after trigger)
│   │       ├── ab_subject_variant
│   │       └── sort_order
│   │
│   ├── EmailSend[]
│   │   ├── id, subscriber_id, sequence_email_id
│   │   ├── sent_at, opened_at, clicked_at
│   │   └── status (pending/sent/opened/clicked/bounced)
│   │
│   └── PageView[]
│       ├── id, project_id, timestamp
│       ├── utm_source, utm_medium, referrer
│       ├── device, country
│       ├── ab_variant (if testing)
│       └── converted (boolean)
```

---

## UI/UX — Key Screens

### 1. Project Setup Wizard
- Step 1: Describe your product (text input + industry, audience)
- Step 2: AI generates landing page — preview with customization options
- Step 3: Configure waitlist (fields, referral program, tiers)
- Step 4: Set up email sequence (pre-built, editable)
- Step 5: Connect domain, go live

### 2. Landing Page Editor
- Full visual editor with section-by-section editing
- Click any text to edit inline
- Section reordering (drag)
- Image upload/replacement
- Color and font customization
- Mobile/desktop preview toggle
- Publish button

### 3. Waitlist Dashboard
- Total subscribers counter (prominent)
- Daily signups chart
- Conversion rate
- Source breakdown (direct, referral, social)
- Referral metrics: viral coefficient, average referrals per user
- Top referrers leaderboard
- Recent signups list

### 4. Subscriber Manager
- Sortable table: email, name, position, referral count, tier, source, date
- Filter by tier, source, verified/unverified
- Export CSV
- Bulk actions: invite, email, delete
- Individual view: referral chain visualization

### 5. Email Builder
- Sequence timeline (visual)
- Click email to edit: subject, body, delay
- Merge field inserter ({first_name}, {position}, {referral_link})
- Preview as rendered email
- Test send to yourself

### 6. Analytics
- Funnel: views → signups → shares → referred signups
- Traffic sources with conversion rates
- A/B test results
- Email performance metrics
- Geographic and device breakdown

### 7. Subscriber Confirmation Page
- "You're #47 on the waitlist!"
- Referral section: "Share to move up" with social buttons
- Referral link (copy to clipboard)
- Tier progress: "Refer 3 more friends to unlock early access"

---

## Monetization

### Free Tier
- 100 signups
- 1 landing page
- Basic email sequence (3 emails)
- Waitlist counter
- WaitlistIQ branding

### Pro — $19/month
- 5,000 signups
- Custom domain
- Full referral program
- Custom email sequences (unlimited)
- A/B testing
- Analytics
- Remove WaitlistIQ branding

### Growth — $49/month
- Unlimited signups
- Multiple projects
- Launch day tools (batch invites, access codes)
- All integrations (Mailchimp, HubSpot, Zapier)
- Priority email deliverability
- Team access (3 users)
- API

---

## Go-to-Market Strategy

- Target indie hackers and startup founders (Indie Hackers, Twitter/X, Product Hunt)
- Build in public: "How we built our pre-launch tool with our own pre-launch tool"
- Free for projects with <100 signups (very generous entry)
- Content: "How to build a 10,000-person waitlist before launch"
- Template gallery: "Pre-launch pages for SaaS, consumer apps, newsletters"
- Partner with accelerators and startup programs
- Product Hunt launch (meta: use WaitlistIQ for the WaitlistIQ launch)

---

## Success Metrics

| Metric | Target (Month 6) | Target (Month 12) |
|--------|-------------------|---------------------|
| Projects created | 1,000 | 5,000 |
| Total subscribers (all projects) | 100,000 | 500,000 |
| Average viral coefficient | 1.2 | 1.5 |
| Paying customers | 80 | 350 |
| MRR | $2,500 | $12,000 |
| Landing page conversion rate (avg) | 20% | 25% |

---

## MVP Scope (v1.0)

### In Scope
- AI landing page generation from product description
- Visual page editor (basic)
- Waitlist with position tracking
- Referral program (move up positions)
- Welcome email + share reminder email
- Social sharing buttons
- Basic analytics (signups, sources)
- Hosted on waitlistiq.io

### Out of Scope
- Custom domain
- A/B testing
- Full email sequence builder
- Launch day tools (batch invites, access codes)
- Integrations (Mailchimp, Zapier)
- Social proof widget
- Multiple projects

### MVP Timeline: 5-6 weeks
- Week 1: AI landing page generation, template system, page editor
- Week 2: Waitlist engine (signup, position tracking, referral codes)
- Week 3: Referral system, share buttons, confirmation page
- Week 4: Email system (welcome + reminder), subscriber dashboard
- Week 5: Analytics, landing page hosting
- Week 6: Billing, polish, launch
