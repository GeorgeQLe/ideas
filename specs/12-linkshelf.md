# LinkShelf — Team Bookmark Organizer

## Executive Summary

LinkShelf is a shared team bookmark library with AI categorization, full-text search across saved page content, and a browser extension for one-click saving. It replaces the chaos of Slack bookmarks, browser bookmark bars, and scattered Notion pages with a searchable, organized, team-wide knowledge base of useful links.

---

## Problem Statement

**The pain:**
- Teams share useful links in Slack channels, but they're buried within days and impossible to find
- Browser bookmarks are personal, unsearchable, and don't sync across team
- Notion/Confluence pages of "useful links" become stale within weeks
- When someone asks "does anyone have that link to...", it takes 10+ minutes of Slack searching to find it
- Knowledge walks out the door when team members leave

---

## Target Users

### Primary Personas

**1. Chloe — Engineering Team Lead**
- Team shares API docs, architecture references, and debugging guides in Slack
- Can never find them when needed
- Needs: team link library that actually gets used

**2. Ryan — Product Manager**
- Saves competitor analyses, research articles, and design references everywhere
- Needs to share curated link collections with different stakeholders
- Needs: organized collections with sharing and search

**3. Diana — Operations Manager**
- Manages a wiki of SOPs, vendor portals, and internal tools
- Links change and break; nobody maintains the wiki
- Needs: auto-organized bookmarks with dead link detection

---

## Core Features

### F1: Save From Anywhere
- **Browser extension** (Chrome, Firefox, Safari): one-click save from any page
  - Right-click context menu: "Save to LinkShelf"
  - Popup: add tags, select collection, add note
  - Auto-captures: title, URL, description, favicon, screenshot
- **Slack bot**: react with bookmark emoji or use `/shelf save` command
- **Web app**: paste URL to save
- **Mobile share sheet** (v2): share from any mobile app
- **API**: programmatic saving
- **Email**: forward links to save@linkshelf.io

### F2: AI Auto-Categorization
- On save, AI analyzes the page content and generates:
  - Category (documentation, tutorial, tool, article, reference, video, etc.)
  - Topic tags (react, devops, marketing, design, etc.)
  - Short summary (1-2 sentences)
- Auto-suggest existing collections to add to
- Improve suggestions based on user corrections over time

### F3: Full-Text Search
- Search across: titles, URLs, page content, tags, notes, summaries
- Semantic search: "that article about React server components performance" finds the right link
- Filter by: collection, tag, saver, date range, domain
- Fuzzy matching for typos
- Search within a specific collection
- Recent and trending bookmarks surfaced in suggestions

### F4: Collections & Organization
- Collections: group links by topic (e.g., "Onboarding resources", "Design inspiration", "Competitor analysis")
- Nested collections (folders within folders)
- Tags: multiple per bookmark, combinable for filtering
- Favorites: quick access to most-used bookmarks
- Smart collections: auto-populated by rules (e.g., "All links from github.com tagged 'devops'")
- Pin important bookmarks to top of collection
- Bulk organize: select multiple, move to collection, add tags

### F5: Team Spaces
- Create team workspaces with invite-based access
- Permissions: viewer (see only), contributor (save and edit), admin (manage collections and members)
- Personal bookmarks visible only to you
- Team activity feed: who saved what recently
- "@mention" team members on bookmarks
- Bookmark discussions: comment threads on shared links

### F6: Content Extraction & Archival
- Extract and store page content on save (not just URL)
- Full-page screenshot on save
- Reader view: stripped-down content view without ads
- Annotation and highlights (v2): highlight text on saved pages
- Wayback Machine integration: link to archived version if page goes down
- Content available even when original page is down

### F7: Dead Link Detection
- Weekly scan of all bookmarks for broken links (404, timeout, domain expired)
- Alert digest: "5 bookmarks have broken links"
- Suggest alternatives (similar content found elsewhere)
- Auto-archive broken links
- Redirect detection: alert if page now redirects

### F8: Discovery & Digests
- Weekly digest email: "Popular with your team this week" (top saved/viewed links)
- "Trending" tab: most-saved links across your team
- "Recommended for you": AI suggests links saved by teammates that match your interests
- New member onboarding: curated collection of "Essential reading for new team members"

### F9: Import & Export
- Import from: Chrome bookmarks, Firefox bookmarks, Pocket, Raindrop.io, Notion, CSV
- Export to: HTML (browser bookmarks format), CSV, JSON, Notion
- Bulk import wizard with deduplication

---

## Technical Architecture

### Tech Stack

| Layer | Technology |
|-------|-----------|
| Frontend | Next.js 14+, Tailwind CSS |
| Backend | Next.js API Routes, tRPC |
| Database | PostgreSQL (Neon) |
| Search | pgvector (semantic) + pg_trgm (fuzzy text) |
| AI | OpenAI text-embedding-3-small, GPT-4o-mini (categorization) |
| Content Extraction | Puppeteer / Playwright (headless browser for content + screenshots) |
| Browser Extension | Chrome Extension Manifest V3 |
| Slack Bot | Slack Bolt SDK |
| Auth | Clerk |
| Storage | Cloudflare R2 (screenshots, extracted content) |
| Hosting | Vercel (app), Fly.io (extraction workers) |

### Data Model

```
User
├── id, email, name, avatar_url, plan
│
├── Workspace[] (personal + team)
│   ├── id, name, type (personal/team)
│   ├── members[] → WorkspaceMember (user_id, role)
│   │
│   ├── Collection[]
│   │   ├── id, workspace_id, name, description, icon
│   │   ├── parent_collection_id
│   │   ├── is_smart, smart_rules (JSON)
│   │   ├── is_pinned, sort_order
│   │   └── created_by
│   │
│   └── Bookmark[]
│       ├── id, workspace_id, collection_id
│       ├── url, title, description, favicon_url
│       ├── screenshot_url, content_text (extracted)
│       ├── ai_summary, ai_category
│       ├── tags (text[])
│       ├── note (user-added)
│       ├── is_favorite, is_pinned
│       ├── saved_by_user_id
│       ├── source (extension/slack/web/api/email)
│       ├── domain (extracted)
│       ├── embedding (vector[1536])
│       ├── link_status (alive/broken/redirect)
│       ├── last_checked_at
│       ├── view_count, save_count
│       ├── created_at, updated_at
│       │
│       └── BookmarkComment[]
│           ├── id, bookmark_id, user_id
│           ├── text, created_at
│           └── parent_comment_id (threading)
│
└── DigestConfig
    ├── id, user_id, frequency (weekly/daily/off)
    └── preferences (JSON)
```

---

## UI/UX — Key Screens

### 1. Library / Home
- Search bar (prominent, top center) with recent searches
- Tabs: All | Favorites | Recent | Trending
- Bookmark grid/list view (toggle)
- Each card: favicon, title, domain, tags, summary snippet, save date
- Left sidebar: collections tree, tags, team switcher

### 2. Bookmark Detail
- Page screenshot/preview
- Title, URL (clickable), description
- AI summary and category
- Tags (editable)
- User note (editable)
- Saved by, date, source
- Comments thread
- Related bookmarks
- "Open" button, "Copy URL" button, "Archive" button

### 3. Collection View
- Collection header with description
- Bookmark grid within collection
- Sort: newest, oldest, most viewed, alphabetical
- Sub-collections listed at top
- Share collection (generate link or invite)

### 4. Team Activity
- Feed of recent saves across team
- "Ryan saved 'React 19 Migration Guide' to Engineering Resources"
- Filter by team member
- Like/react to saves

### 5. Browser Extension Popup
- Shows current page info (title, URL)
- Collection dropdown
- Tag input
- Note field
- "Save" button
- Recently saved bookmarks list

---

## Monetization

### Free Tier
- 50 bookmarks
- Personal workspace only
- Basic search (keyword)
- Browser extension
- 1 collection

### Pro — $6/month
- Unlimited bookmarks
- AI categorization and semantic search
- Unlimited collections
- Full-text content search
- Dead link detection
- Import/export
- Slack bot

### Team — $4/user/month (min 3 users)
- Everything in Pro
- Team workspaces
- Permissions and roles
- Activity feed
- Weekly digest
- Smart collections
- Priority support

---

## Go-to-Market Strategy

- Chrome Web Store listing (primary discovery)
- Product Hunt launch
- Content: "Your team's Slack bookmarks are a mess — here's the fix"
- Free import tool from Pocket/Raindrop (migration hook)
- SEO: "team bookmark manager", "shared bookmarks", "bookmark organizer"
- Dev Twitter/X showcase
- Remote work community targeting

---

## Success Metrics

| Metric | Target (Month 6) | Target (Month 12) |
|--------|-------------------|---------------------|
| Users | 3,000 | 15,000 |
| Bookmarks saved | 100,000 | 600,000 |
| Searches/week | 5,000 | 30,000 |
| Paying users | 150 | 600 |
| MRR | $1,500 | $6,000 |
| Extension installs | 5,000 | 25,000 |

---

## MVP Scope (v1.0)

### In Scope
- Chrome browser extension (save with one click)
- Web app with bookmark management
- AI categorization and tagging
- Semantic search
- Collections
- Basic team workspace (invite members, shared bookmarks)
- Chrome bookmark import

### Out of Scope
- Slack bot, Firefox/Safari extensions
- Content extraction and archival
- Dead link detection
- Smart collections
- Weekly digests
- Annotations/highlights
- Comments

### MVP Timeline: 5-6 weeks
- Week 1: Data model, bookmark CRUD, AI categorization pipeline
- Week 2: Search (semantic + keyword), collections
- Week 3: Chrome extension
- Week 4: Team workspace, sharing
- Week 5: Import, basic dashboard
- Week 6: Billing, polish, launch
