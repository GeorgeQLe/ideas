# SnipVault — AI-Powered Code Snippet Manager

## Executive Summary

SnipVault is a searchable, AI-tagged code snippet library that lets developers save, organize, and retrieve code snippets from anywhere — web app, CLI, IDE extensions, or browser extension. AI auto-tags and enables semantic search so you can find that snippet you saved months ago by describing what it does.

---

## Problem Statement

**The pain:**
- Developers accumulate useful code patterns, configs, and one-liners over years
- These snippets live scattered across GitHub Gists, random .txt files, Notion pages, Slack bookmarks, and browser tabs
- Finding a previously saved snippet is nearly impossible — you remember what it does but not where you put it or what you named it
- GitHub Gists have no organization, no search by functionality, no tagging
- Stack Overflow answers get outdated; your own working, tested snippets are more valuable

**The cost:**
- Developers waste 15-30 minutes re-searching or re-writing snippets they've solved before
- Tribal knowledge is lost when developers leave a team
- Onboarding new devs is slower without a shared snippet library

---

## Target Users

### Primary Personas

**1. Jake — Senior Full-Stack Developer**
- 8 years experience, has accumulated hundreds of patterns and tricks
- Currently uses a mix of Gists, Notion, and Apple Notes
- Can never find what he saved; often re-Googles the same things
- Needs: fast save from anywhere, find by description, works in his IDE

**2. Emily — Tech Lead**
- Manages a team of 5 developers
- Wants to share common patterns, configs, and boilerplate across the team
- Tired of answering the same questions in Slack
- Needs: shared team library, permissioned collections

**3. Carlos — Freelance Developer**
- Works across many projects and tech stacks
- Has a personal wiki of snippets in Obsidian but it's getting unwieldy
- Needs: better organization, cross-device sync, quick access from IDE

---

## Core Features

### F1: Multi-Source Snippet Capture
- **Web app**: paste code, set language, add description
- **VS Code extension**: select code → right-click → "Save to SnipVault"
- **JetBrains plugin**: equivalent functionality
- **CLI tool**: `snip save --file utils.py --lines 10-25 --title "Rate limiter"`
- **Browser extension**: highlight code on any website → save with one click
- **Slack bot**: `/snip save` in a thread to capture code blocks
- **API**: programmatic snippet creation
- Auto-detect language from code content

### F2: AI Auto-Tagging & Categorization
- On save, AI analyzes the code and generates:
  - Language and framework tags
  - Purpose tags (e.g., "authentication", "error handling", "database", "regex")
  - Complexity level
  - Short description if none provided
- Users can edit/override all AI-generated metadata
- Tag suggestions improve over time based on user corrections

### F3: Semantic Search
- Search by natural language: "postgres connection pool with retry logic"
- Vector embedding-based search (not just keyword matching)
- Filter by: language, tags, collection, date range, source
- Full-text search across snippet content and descriptions
- Fuzzy matching for typos
- Recent and frequently accessed snippets surfaced first

### F4: Organization
- **Collections**: group related snippets (e.g., "Auth patterns", "SQL queries", "DevOps configs")
- **Tags**: multiple tags per snippet (language, framework, purpose)
- **Favorites**: star frequently used snippets
- **Nested folders**: hierarchical organization
- **Smart collections**: auto-populated based on rules (e.g., "All Python snippets tagged 'AWS'")

### F5: Snippet Editor
- Syntax highlighting for 100+ languages (via Shiki/Prism)
- Multiple files per snippet (e.g., a component + its test + its CSS)
- Markdown description support
- Version history (see how a snippet evolved)
- Fork: create a personal copy of a team snippet
- Diff view between versions
- Live preview for HTML/CSS/JS snippets

### F6: Team Sharing
- Shared team workspaces
- Permissions: viewer, editor, admin per collection
- "Share with team" toggle per snippet
- Team activity feed: see what teammates saved recently
- Request access to private snippets
- Usage analytics: which shared snippets are most popular

### F7: IDE Integration
- **VS Code extension**:
  - Command palette: "SnipVault: Search" opens inline search
  - Insert snippet directly into editor
  - Save selected code as snippet
  - Sidebar panel showing recent/favorite snippets
- **JetBrains plugin**: equivalent features
- **Neovim plugin** (v2): Telescope integration

### F8: Import/Export
- Import from: GitHub Gists, VS Code snippets, Notion, Markdown files
- Export to: Gists, Markdown, JSON
- Bulk import via API
- Migration wizard for first-time users

### F9: Snippet Execution (v2)
- Run snippets in sandboxed environment (JS, Python, Ruby, Go)
- See output inline
- Share runnable snippet links
- Environment variable support for API snippets

---

## Technical Architecture

### Tech Stack

| Layer | Technology |
|-------|-----------|
| Frontend | Next.js 14+, Tailwind CSS, Monaco Editor (code editing) |
| Backend | Next.js API Routes, tRPC |
| Database | PostgreSQL (Neon) — relational data |
| Vector DB | pgvector extension (embeddings for semantic search) |
| AI | OpenAI text-embedding-3-small (embeddings), GPT-4o-mini (tagging) |
| Auth | Clerk |
| IDE | VS Code Extension API, JetBrains Plugin SDK |
| Browser Ext | Chrome Extension Manifest V3 |
| CLI | Node.js CLI (published to npm) |
| Hosting | Vercel |
| Search | pgvector + pg_trgm (trigram for fuzzy match) |

### Data Model

```
User
├── id, email, name, avatar_url, plan
├── settings (default_language, theme, editor_prefs)
│
├── Workspace[] (personal + team)
│   ├── id, name, type (personal/team), owner_id
│   ├── members[] → WorkspaceMember (user_id, role)
│   │
│   ├── Collection[]
│   │   ├── id, workspace_id, name, description, icon
│   │   ├── parent_collection_id (for nesting)
│   │   ├── is_smart, smart_rules (JSON)
│   │   └── sort_order
│   │
│   └── Snippet[]
│       ├── id, workspace_id, collection_id
│       ├── title, description (markdown)
│       ├── is_favorite, is_public
│       ├── source (web/vscode/cli/browser/slack/api)
│       ├── source_url (original page URL if from browser)
│       ├── created_at, updated_at, last_accessed_at
│       ├── access_count
│       │
│       ├── SnippetFile[]
│       │   ├── id, snippet_id, filename, language
│       │   ├── content (text), sort_order
│       │   └── embedding (vector[1536])
│       │
│       ├── SnippetTag[]
│       │   ├── tag_id, snippet_id
│       │   └── source (ai/manual)
│       │
│       └── SnippetVersion[]
│           ├── id, snippet_id, version_number
│           ├── content_snapshot (JSON), created_at
│           └── change_note
│
└── Tag[]
    ├── id, name, category (language/framework/purpose/custom)
    └── usage_count
```

### API Design

```
Snippets:
GET    /api/snippets                  # List (paginated, filtered)
POST   /api/snippets                  # Create
GET    /api/snippets/:id              # Get detail
PATCH  /api/snippets/:id              # Update
DELETE /api/snippets/:id              # Delete
POST   /api/snippets/:id/fork         # Fork to personal workspace
GET    /api/snippets/:id/versions     # Version history

Search:
GET    /api/search?q=...&lang=...     # Semantic + text search
GET    /api/search/suggest?q=...      # Autocomplete suggestions

Collections:
GET    /api/collections               # List
POST   /api/collections               # Create
PATCH  /api/collections/:id           # Update
DELETE /api/collections/:id           # Delete

Tags:
GET    /api/tags                      # List all tags
GET    /api/tags/popular              # Most used tags

Workspaces:
GET    /api/workspaces                # List my workspaces
POST   /api/workspaces                # Create team workspace
POST   /api/workspaces/:id/invite     # Invite member

Import:
POST   /api/import/gists              # Import from GitHub Gists
POST   /api/import/vscode             # Import VS Code snippets
POST   /api/import/markdown           # Import from markdown files
POST   /api/import/bulk               # Bulk JSON import

CLI/Extension Auth:
POST   /api/auth/device-code          # Device flow for CLI/IDE auth
GET    /api/auth/device-code/:code    # Poll for completion
```

---

## UI/UX — Key Screens

### 1. Dashboard / Home
- Search bar (prominent, top center)
- Recent snippets grid
- Favorite snippets
- Quick-save button
- Activity feed (team workspace)

### 2. Snippet List / Library
- Left sidebar: collections tree, tags, filters
- Main area: snippet cards in grid or list view
- Each card: title, language badge, first few lines of code preview, tags
- Bulk actions: move to collection, add tag, delete

### 3. Snippet Detail / Editor
- Header: title (editable inline), collection breadcrumb
- Tab bar for multi-file snippets
- Code editor (Monaco) with syntax highlighting
- Right panel: tags (editable), description (markdown), metadata
- Bottom bar: version history, share settings, delete
- Copy button (copies code to clipboard)

### 4. Search Results
- Search bar with active filters shown as chips
- Results ranked by relevance with match highlighting
- Quick preview on hover
- "Insert" button (when opened from IDE)

### 5. Team Workspace
- Member list with roles
- Shared collections
- Activity feed
- "Most popular snippets" section

---

## Monetization

### Free Tier
- 100 snippets
- Personal workspace only
- Basic search (keyword, no semantic)
- Web app only

### Pro — $8/month
- Unlimited snippets
- Semantic search (AI-powered)
- AI auto-tagging
- IDE extensions and CLI
- Browser extension
- Import/export
- Version history

### Team — $5/user/month
- Everything in Pro
- Shared team workspace
- Permissions and roles
- Activity feed
- Usage analytics
- Priority support

---

## Go-to-Market Strategy

- Launch on Product Hunt and Hacker News
- VS Code Marketplace listing (primary discovery channel)
- Dev Twitter content: "I built an AI-powered snippet manager"
- Blog: "Why your GitHub Gists are a graveyard"
- Free import tool from Gists (drives signups)
- Partner with developer YouTubers for reviews
- Sponsor relevant newsletters (TLDR, Bytes)

---

## Success Metrics

| Metric | Target (Month 6) | Target (Month 12) |
|--------|-------------------|---------------------|
| Registered users | 5,000 | 20,000 |
| Snippets saved | 50,000 | 250,000 |
| Searches per week | 10,000 | 50,000 |
| Paying users | 200 | 800 |
| MRR | $2,000 | $8,000 |
| DAU/MAU ratio | 25% | 35% |

---

## Risks & Mitigations

| Risk | Mitigation |
|------|------------|
| Users don't change habits (keep using Gists) | One-click Gist import, browser extension makes saving frictionless |
| Semantic search quality is poor | Hybrid search (vector + keyword), user feedback loop to improve |
| IDE extension adoption is low | Make web app sufficient standalone, IDE is bonus |
| AI tagging is inaccurate | Allow easy correction, learn from corrections |
| Security concerns (code in cloud) | E2E encryption option, SOC 2, self-hosted option for Enterprise |

---

## MVP Scope (v1.0)

### In Scope
- Web app with snippet CRUD
- Syntax highlighting (50+ languages)
- Collections and tags
- AI auto-tagging on save
- Semantic search
- VS Code extension (save + search)
- GitHub Gists import

### Out of Scope (v1.1+)
- JetBrains plugin, Neovim
- Browser extension
- CLI tool
- Team workspaces
- Version history
- Snippet execution
- Multi-file snippets

### MVP Timeline: 5-6 weeks
- Week 1: Auth, data model, snippet CRUD, basic UI
- Week 2: Syntax highlighting, collections, tags
- Week 3: AI tagging pipeline, embedding generation
- Week 4: Semantic search implementation
- Week 5: VS Code extension
- Week 6: Gist import, billing, polish, launch
