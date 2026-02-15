# DriftLog — Automated Changelog Generator

## Executive Summary

DriftLog connects to your Git repository, analyzes commits, pull requests, and releases, and uses AI to generate polished, user-facing changelogs automatically. It eliminates the tedious manual process of writing changelogs while ensuring customers always know what's new.

---

## Problem Statement

**The pain:**
- Engineering teams ship features and fixes daily but rarely update changelogs
- When changelogs are written, they're inconsistent — some are too technical ("refactored AbstractFactoryProvider"), others too vague ("misc improvements")
- Product managers and developer advocates waste hours translating git history into user-friendly release notes
- Customers churn because they don't see the product improving
- Support teams get repeat questions about features that already shipped but were never communicated

**Current workarounds:**
- Manually writing release notes in Notion/Google Docs (time-consuming, often forgotten)
- Auto-generated changelogs from conventional commits (too technical for end users)
- Marketing team writes release notes (bottleneck, often inaccurate)
- No changelog at all (most common — and worst)

**Market size:** ~500K SaaS companies and open-source projects that ship software regularly. The changelog/release notes tooling market is fragmented with no dominant player for AI-powered generation.

---

## Target Users

### Primary Personas

**1. Sarah — Engineering Lead (IC turned manager)**
- Manages a team of 6-12 engineers
- Ships 2-3 releases per week
- Knows changelogs matter but never has time
- Needs: automation, minimal setup, looks professional

**2. Marcus — Developer Advocate / DevRel**
- Responsible for communicating product updates
- Spends 3-4 hours/week writing release notes
- Needs: AI draft to edit rather than writing from scratch

**3. Priya — Solo Founder / Indie Hacker**
- Ships daily, wears every hat
- Wants to show users the product is actively developed
- Needs: zero-effort changelog that "just works"

### Secondary Personas
- Product managers tracking feature delivery
- Open-source maintainers communicating with contributors
- Customer success managers sharing updates with clients

---

## Solution Overview

DriftLog is a GitHub/GitLab App that:
1. Watches for new releases, merged PRs, or tagged commits
2. Analyzes the changes using AI (code diffs + PR descriptions + commit messages)
3. Generates a user-facing changelog entry categorized by type
4. Publishes to a hosted changelog page and optionally notifies users
5. Allows manual editing before or after publishing

---

## Core Features

### F1: Git Provider Integration
- **GitHub App** with OAuth + webhook-based integration
- **GitLab** integration via API token + webhooks
- **Bitbucket** integration (v2)
- Auto-detect repositories and branches
- Configure which branches/tags trigger changelog generation
- Support for monorepo (generate per-package changelogs)

### F2: AI Changelog Generation
- Analyze PR titles, descriptions, commit messages, and code diffs
- Generate human-readable, non-technical summaries
- Categorize entries automatically:
  - New Feature
  - Improvement
  - Bug Fix
  - Breaking Change
  - Performance
  - Security
  - Deprecation
- Configurable tone: professional, casual, technical, marketing
- Configurable audience: end-users, developers, internal team
- Support for multiple languages (English, Spanish, French, German, Japanese)
- Ignore internal/infrastructure changes (CI config, dependency bumps) unless configured otherwise

### F3: Hosted Changelog Page
- Beautiful, responsive changelog page at `yourapp.driftlog.io`
- Custom domain support (CNAME)
- Customizable branding: logo, colors, fonts, favicon
- Search and filter by category, date range
- Pagination with infinite scroll
- RSS feed for changelog subscribers
- SEO-optimized with meta tags and structured data
- Dark mode support

### F4: Embeddable "What's New" Widget
- JavaScript snippet for embedding in your app
- Slide-out panel or modal triggered by bell icon
- Badge showing unread updates count
- Customizable appearance to match your app
- User-specific "last seen" tracking
- Lightweight (<15KB gzipped)

### F5: Notification System
- Email digest (per release or weekly summary)
- Slack channel integration (post to #product-updates)
- Webhook for custom integrations
- In-app widget notifications
- Subscriber management (users can self-subscribe)
- Segmentation (notify only about relevant categories)

### F6: Editorial Workflow
- AI generates draft → human reviews → publish
- Rich text editor for editing generated entries
- Preview before publishing
- Schedule future publication
- Team approval workflow (reviewer must approve before publish)
- Revision history with diff view
- Bulk editing for batch releases

### F7: Release Management
- Auto-detect releases from GitHub Releases, tags, or custom triggers
- Manual release creation
- Semantic versioning suggestions based on change types
- Group multiple PRs into a single release
- Release date override
- Draft releases (work in progress)

### F8: Analytics
- Changelog page views and unique visitors
- Most-viewed entries
- Widget interaction rates
- Subscriber growth over time
- Notification open rates
- Referral sources

---

## Technical Architecture

### System Diagram

```
GitHub/GitLab ──webhook──▶ Webhook Ingestion (API)
                                │
                                ▼
                         Job Queue (Redis/BullMQ)
                                │
                                ▼
                    ┌──── AI Processing Worker ────┐
                    │  1. Fetch PR/commit details   │
                    │  2. Analyze diffs              │
                    │  3. Generate changelog entry   │
                    │  4. Categorize                  │
                    └──────────┬─────────────────────┘
                               │
                               ▼
                          PostgreSQL
                               │
                    ┌──────────┼──────────┐
                    ▼          ▼          ▼
              Dashboard    Changelog    Widget
              (Next.js)    Page (SSG)   (JS SDK)
```

### Tech Stack

| Layer | Technology |
|-------|-----------|
| Frontend | Next.js 14+ (App Router), Tailwind CSS, Radix UI |
| Backend API | Next.js API Routes / tRPC |
| Database | PostgreSQL (Neon or Supabase) |
| Queue | Redis + BullMQ |
| AI | OpenAI GPT-4o (primary), Claude as fallback |
| Auth | Clerk or NextAuth.js |
| Hosting | Vercel (app) + Cloudflare (changelog pages) |
| Email | Resend |
| File Storage | Cloudflare R2 (logos, assets) |
| Monitoring | Sentry, Axiom |

### Data Model

```
Organization
├── id, name, slug, plan, created_at
├── branding (logo_url, primary_color, font, custom_domain)
│
├── Repository[]
│   ├── id, org_id, provider (github/gitlab), provider_repo_id
│   ├── name, url, default_branch, is_monorepo
│   └── config (tone, audience, ignored_paths, auto_publish)
│
├── Release[]
│   ├── id, org_id, repo_id, version, title
│   ├── status (draft/review/published/scheduled)
│   ├── published_at, scheduled_at, created_at
│   └── source (auto/manual)
│
├── ChangelogEntry[]
│   ├── id, release_id, org_id
│   ├── category (feature/improvement/bugfix/breaking/etc)
│   ├── title, body (rich text)
│   ├── ai_generated_body, human_edited (boolean)
│   ├── source_pr_id, source_commits[]
│   └── sort_order
│
├── Subscriber[]
│   ├── id, org_id, email, preferences
│   └── last_seen_release_id
│
└── WidgetConfig
    ├── id, org_id, position, theme
    └── trigger_selector
```

### API Design

```
Auth:
POST   /api/auth/github          # OAuth flow
POST   /api/auth/gitlab           # OAuth flow

Repositories:
GET    /api/repos                  # List connected repos
POST   /api/repos                  # Connect a repo
DELETE /api/repos/:id              # Disconnect
PATCH  /api/repos/:id/config       # Update repo settings

Releases:
GET    /api/releases               # List releases
POST   /api/releases               # Create manual release
PATCH  /api/releases/:id           # Update release
POST   /api/releases/:id/publish   # Publish
POST   /api/releases/:id/schedule  # Schedule publication

Entries:
GET    /api/releases/:id/entries   # List entries in release
POST   /api/releases/:id/entries   # Add entry
PATCH  /api/entries/:id            # Edit entry
POST   /api/entries/:id/regenerate # Re-run AI generation
DELETE /api/entries/:id            # Remove entry

Public (no auth):
GET    /api/public/:org/changelog  # Public changelog feed (JSON)
GET    /api/public/:org/rss        # RSS feed
GET    /api/public/:org/widget     # Widget config + recent entries

Webhooks:
POST   /api/webhooks/github        # GitHub webhook receiver
POST   /api/webhooks/gitlab        # GitLab webhook receiver
```

---

## UI/UX — Key Screens

### 1. Dashboard / Home
- Overview cards: total releases, entries this month, page views, subscribers
- Recent activity feed
- Quick actions: create release, connect repo

### 2. Repository Settings
- List of connected repos with status indicators
- Per-repo configuration: tone, audience, auto-publish toggle, ignored paths
- Test generation: pick a recent PR and preview the AI output

### 3. Release Editor
- Left panel: list of changelog entries (drag to reorder)
- Center: rich text editor with AI draft + diff toggle
- Right panel: metadata (version, date, status)
- Top bar: preview, save draft, publish, schedule buttons
- "Regenerate" button per entry to re-run AI with different parameters

### 4. Changelog Page (Public)
- Clean, minimal design with organization branding
- Entries grouped by release, newest first
- Category badges (color-coded)
- Search bar and category filter
- Subscribe button (email)

### 5. Widget Preview & Config
- Live preview of the embeddable widget
- Position selector (bottom-right, bottom-left, custom)
- Theme customization
- Copy embed code button

### 6. Analytics
- Time-series chart of page views
- Top entries by views
- Subscriber growth chart
- Notification engagement metrics

---

## Monetization

### Free Tier
- 1 repository
- 10 changelog entries per month
- Hosted changelog page (driftlog.io subdomain)
- DriftLog branding on page and widget

### Pro — $19/month
- Unlimited repositories
- Unlimited entries
- Custom domain
- Remove DriftLog branding
- Email notifications to subscribers
- Widget embed
- Analytics

### Team — $49/month
- Everything in Pro
- Approval workflow (reviewer must approve)
- Multiple team members (up to 10)
- Slack integration
- Priority AI processing
- Custom tone/voice training
- API access

### Enterprise — Custom
- SSO/SAML
- Unlimited team members
- SLA guarantee
- Dedicated support
- Custom integrations
- On-premise option

---

## Go-to-Market Strategy

### Phase 1: Developer Community (Month 1-3)
- Launch on Product Hunt
- Post on Hacker News (Show HN)
- Write blog posts: "We automated our changelog with AI"
- Free tier for open-source projects (with backlink)
- GitHub Marketplace listing

### Phase 2: Content & SEO (Month 3-6)
- Blog: "How to write great changelogs" (SEO play)
- Comparisons: "DriftLog vs manually writing changelogs"
- Template gallery: "Changelog templates for SaaS"
- YouTube: demo videos and tutorials

### Phase 3: Integrations & Partnerships (Month 6-12)
- Integrate with Notion, Confluence
- Partner with DevRel tools and communities
- Sponsor developer podcasts
- Attend developer conferences

### Acquisition Channels
- Organic search (target "changelog generator", "release notes tool")
- GitHub Marketplace discovery
- Word of mouth from free tier users
- Dev Twitter/X and LinkedIn content

---

## Success Metrics & KPIs

| Metric | Target (Month 6) | Target (Month 12) |
|--------|-------------------|---------------------|
| Connected repos | 500 | 2,000 |
| Monthly entries generated | 5,000 | 25,000 |
| Paying customers | 50 | 200 |
| MRR | $2,000 | $8,000 |
| Changelog page views (all customers) | 100K/mo | 500K/mo |
| Free → Paid conversion | 5% | 8% |
| Churn rate | <5%/mo | <4%/mo |

---

## Competitive Landscape

| Competitor | Strengths | Weaknesses | DriftLog Advantage |
|-----------|-----------|------------|-------------------|
| Headway | Beautiful UI, established | No AI generation, manual-only | AI automation |
| Beamer | In-app widget, notifications | Expensive, no git integration | Git-native, cheaper |
| Changelogfy | Simple and affordable | No AI, limited customization | AI + full customization |
| GitHub Releases | Free, built-in | Too technical, ugly, no widget | User-facing, beautiful |
| release-please | Free, automated | Generates dev changelogs only | User-facing AI summaries |

---

## Risks & Mitigations

| Risk | Impact | Likelihood | Mitigation |
|------|--------|------------|------------|
| AI generates inaccurate summaries | High | Medium | Always generate drafts, require human review by default, confidence scoring |
| GitHub/GitLab changes API or webhook format | Medium | Low | Abstract provider layer, comprehensive integration tests |
| Low adoption of changelog pages | Medium | Medium | Focus on widget (in-app) as primary distribution |
| AI costs eat margins | Medium | Medium | Cache similar patterns, use smaller models for simple changes, batch processing |
| Enterprise deals require on-premise | Low | Medium | Offer self-hosted Docker option at Enterprise tier |

---

## MVP Scope (v1.0)

### In Scope
- GitHub integration (App + webhooks)
- AI changelog generation from PRs
- Hosted changelog page with basic branding
- Manual editing of generated entries
- Email notifications to subscribers

### Out of Scope (v1.1+)
- GitLab and Bitbucket
- Embeddable widget
- Approval workflows
- Analytics
- Custom domain
- Monorepo support
- Multi-language generation

### MVP Timeline: 6-8 weeks
- Week 1-2: GitHub App integration, webhook handling, data model
- Week 3-4: AI processing pipeline, generation quality tuning
- Week 5-6: Dashboard UI, changelog page, editor
- Week 7-8: Email notifications, billing, polish, launch prep
