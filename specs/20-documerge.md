# DocuMerge — API Documentation Generator

## Executive Summary

DocuMerge auto-generates beautiful, interactive API documentation from your OpenAPI spec, GraphQL schema, or code annotations. It keeps docs in sync with your codebase via GitHub integration, provides a live API playground for every endpoint, and auto-generates code examples in 6+ languages — so your API docs are always accurate, complete, and developer-friendly.

---

## Problem Statement

**The pain:**
- API documentation is always out of date because it's maintained separately from code
- Swagger UI is functional but ugly and provides a poor developer experience
- Writing code examples for every endpoint in every language is tedious and error-prone
- Developers avoid updating docs because the tooling is painful
- Bad API docs directly increase support burden and reduce API adoption

**The cost:**
- 90% of developers say documentation is important when evaluating APIs (SmartBear)
- Poor docs increase integration time by 3-5x
- Support tickets for undocumented behavior cost $15-25 each
- APIs with great docs see 2-3x higher adoption rates

---

## Target Users

### Primary Personas

**1. Sophie — Developer Relations Engineer**
- Maintains API docs for a developer platform
- Currently uses ReadMe but finds it expensive and clunky
- Needs: auto-sync from code, beautiful output, easy maintenance

**2. James — Backend Engineer**
- Builds APIs but hates writing documentation
- Wants docs that "just generate" from his OpenAPI spec
- Needs: zero-effort doc generation, playground for testing

**3. Daniel — CTO at API-First Startup**
- API is the product; docs are the storefront
- Needs docs that make developers say "wow, this is easy to integrate"
- Needs: custom branding, code examples, professional design

---

## Core Features

### F1: Source Import
- **OpenAPI/Swagger**: import from file (YAML/JSON) or URL
- **GraphQL**: import from schema file or introspection endpoint
- **AsyncAPI**: for event-driven APIs
- **Postman Collections**: import and convert
- **Code annotations** (v2): parse JSDoc, Python docstrings, Go doc comments
- **Manual entry**: add endpoints through a form-based editor
- Schema validation with error reporting
- Support for OpenAPI 3.0 and 3.1

### F2: GitHub Auto-Sync
- Connect GitHub repository
- Specify path to API spec file (e.g., `api/openapi.yaml`)
- Webhook: auto-rebuild docs on push to main branch
- Branch preview: generate preview docs for pull requests
- Multiple specs per repo (for microservices)
- GitLab support (v2)
- Bitbucket support (v2)

### F3: Documentation Site Generation
- Beautiful, modern documentation site
- **Navigation**: sidebar with endpoint grouping, search
- **Endpoint pages**: method badge (GET/POST/PUT/DELETE), URL, description, parameters, request/response schemas, examples
- **Authentication docs**: auto-generated from security schemes in spec
- **Error reference**: auto-generated from error response schemas
- **Changelog**: auto-generated from spec changes between versions
- **Getting started guide**: AI-generated quickstart from your API
- **Webhooks documentation**: for APIs with webhooks
- **Full-text search**: across all endpoints and descriptions
- **Responsive**: works on mobile and tablet

### F4: Interactive API Playground
- "Try it" button on every endpoint
- Request builder:
  - Fill in path parameters, query parameters, headers
  - JSON body editor with schema validation
  - File upload for multipart endpoints
- Authentication: enter API key, Bearer token, or OAuth credentials
- Send request to actual API (or sandbox environment)
- Response viewer: formatted JSON/XML with syntax highlighting
- Response headers and status code display
- Save and share requests as examples
- cURL command generation for every request
- Request history (per user session)

### F5: Auto-Generated Code Examples
- Generate idiomatic code examples for every endpoint in:
  - cURL
  - JavaScript (fetch / axios)
  - Python (requests / httpx)
  - Go (net/http)
  - Ruby (Net::HTTP / httparty)
  - PHP (Guzzle)
  - Java (HttpClient)
  - C# (HttpClient)
- Examples include: authentication, parameters, request body, error handling
- Copy-to-clipboard per example
- Language switcher (remembers user preference)
- Custom examples: override auto-generated with hand-written examples

### F6: Versioning
- Multiple API versions (v1, v2, v3)
- Version switcher in docs navigation
- Deprecation notices for old endpoints
- Diff view: what changed between versions
- Migration guide generation (AI-assisted)
- Pin specific versions

### F7: Custom Branding & Theming
- Custom domain (docs.yourapi.com)
- Logo, colors, fonts, favicon
- Theme presets: light, dark, high contrast
- Custom CSS injection
- Custom header/footer
- Custom landing page
- Custom 404 page
- OG/social sharing image

### F8: Content Authoring
- **MDX support**: mix Markdown with interactive components
- **Custom pages**: add guides, tutorials, concepts beyond API reference
- **Code blocks**: with syntax highlighting and copy button
- **Callouts**: info, warning, danger, tip
- **Tabs**: for showing multiple code examples or platform-specific content
- **Diagrams**: Mermaid.js support
- **Inline annotations**: add notes to schema properties
- WYSIWYG and code editors

### F9: Analytics
- Page views per endpoint
- Most viewed endpoints ranking
- Playground usage: which endpoints are tested most
- Search queries: what are developers looking for
- 404 tracking: broken links and missing pages
- Time on page
- Geographic and referrer data

### F10: Developer Experience Features
- **SDK generation** (v2): auto-generate client libraries from spec
- **Webhook testing**: receive and inspect webhook payloads
- **Rate limit visualization**: show rate limit headers
- **Schema visualization**: interactive JSON schema explorer
- **OpenAPI linting**: validate and suggest improvements to your spec

---

## Technical Architecture

### Tech Stack

| Layer | Technology |
|-------|-----------|
| Frontend / Docs Site | Next.js 14+ (static generation), Tailwind CSS, MDX |
| Dashboard | Next.js, tRPC, Radix UI |
| Backend | Next.js API Routes |
| Database | PostgreSQL (Neon) |
| AI | OpenAI GPT-4o (quickstart generation, migration guides) |
| Code Examples | Custom template engine per language |
| API Playground | Sandboxed API proxy (to handle CORS) |
| GitHub App | Webhook-based auto-sync |
| Hosting | Vercel (dashboard), Cloudflare Pages (docs sites — edge deployed) |
| Auth | Clerk |
| Search | Algolia or Pagefind (static search index) |

### Data Model

```
Organization
├── id, name, slug, plan
│
├── ApiProject[]
│   ├── id, org_id, name, slug
│   ├── custom_domain
│   ├── branding (JSON: logo, colors, font, css)
│   ├── github_repo_url, github_spec_path
│   ├── github_app_installation_id
│   ├── auto_sync_enabled, last_synced_at
│   │
│   ├── ApiVersion[]
│   │   ├── id, project_id, version_name (e.g., "v2")
│   │   ├── spec_format (openapi/graphql/asyncapi)
│   │   ├── spec_content (JSON/YAML stored as text)
│   │   ├── parsed_spec (JSON, normalized)
│   │   ├── is_latest, is_deprecated
│   │   ├── created_at, published_at
│   │   │
│   │   ├── Endpoint[]
│   │   │   ├── id, version_id, method, path
│   │   │   ├── operation_id, summary, description
│   │   │   ├── tags[], deprecated
│   │   │   ├── parameters (JSON[])
│   │   │   ├── request_body (JSON)
│   │   │   ├── responses (JSON)
│   │   │   ├── security (JSON)
│   │   │   ├── code_examples (JSON: {language: code_string})
│   │   │   ├── custom_examples (JSON, overrides auto-gen)
│   │   │   └── sort_order
│   │   │
│   │   └── Schema[]
│   │       ├── id, version_id, name
│   │       ├── definition (JSON)
│   │       └── description
│   │
│   ├── CustomPage[]
│   │   ├── id, project_id, title, slug
│   │   ├── content_mdx
│   │   ├── sort_order, parent_page_id
│   │   └── published
│   │
│   ├── PlaygroundSession[]
│   │   ├── id, project_id, endpoint_id
│   │   ├── request (JSON: method, url, headers, body)
│   │   ├── response (JSON: status, headers, body)
│   │   ├── user_session_id
│   │   └── created_at
│   │
│   └── PageView[]
│       ├── id, project_id, path, endpoint_id
│       ├── referrer, country, device
│       └── viewed_at
│
└── GitHubWebhook[]
    ├── id, project_id, event_type
    ├── payload (JSON)
    └── processed_at
```

---

## UI/UX — Key Screens

### 1. Generated Docs Site (Developer-Facing)
- Left sidebar: navigation tree (groups, endpoints, custom pages)
- Center: endpoint documentation
  - Method badge + URL
  - Description
  - Parameters table (name, type, required, description)
  - Request body with schema viewer (expandable JSON tree)
  - Response examples with status code tabs (200, 400, 401, 500)
  - Code examples with language tabs
  - "Try it" button → opens playground panel
- Right sidebar (desktop): table of contents for current page
- Top: search bar, version switcher, dark mode toggle

### 2. API Playground
- Split view: request builder on left, response on right
- Request: URL with parameter inputs, headers editor, body editor (JSON)
- Authentication section (auto-populated from spec)
- "Send Request" button
- Response: status code, timing, formatted body, headers
- Generated cURL below

### 3. Dashboard — Project List
- Card per API project: name, domain, last synced, version count
- Quick actions: view docs, edit, sync now
- Create new project button

### 4. Project Settings
- GitHub connection and sync configuration
- Custom domain setup
- Branding customization (live preview)
- Version management
- Team members
- Analytics overview

### 5. Content Editor
- MDX editor with live preview
- Custom page management (sidebar tree editor)
- Endpoint-level annotation editor
- Custom code example override editor

### 6. Analytics
- Most viewed endpoints chart
- Search query log
- Playground usage stats
- Traffic over time
- 404 errors list

---

## Monetization

### Free Tier
- 1 API project
- Basic theme (no custom branding)
- API playground
- Auto-generated code examples (3 languages)
- DocuMerge branding
- Community support

### Pro — $29/month
- Custom domain
- Full branding customization
- GitHub auto-sync
- All languages for code examples
- Custom pages (MDX)
- Analytics
- Versioning (3 versions)
- Remove branding

### Enterprise — $99/month
- Multiple API projects
- Unlimited versions
- SSO for docs access (private docs)
- Custom CSS injection
- Team collaboration
- Priority support
- SLA
- API access
- Webhook testing

---

## Go-to-Market Strategy

- Target developer-facing SaaS companies and API-first startups
- Content: "Your API docs are losing you customers" (thought leadership)
- SEO: "API documentation tool", "OpenAPI docs generator", "interactive API docs"
- Free tier with "Powered by DocuMerge" link (backlink + discovery)
- Compare with ReadMe, Stoplight, Swagger UI on features and price
- Open-source the docs site theme (MIT) — drives awareness and trust
- Developer conference sponsorships
- Integration with popular API frameworks (Express, FastAPI, Rails, Go)

---

## Success Metrics

| Metric | Target (Month 6) | Target (Month 12) |
|--------|-------------------|---------------------|
| API projects | 200 | 1,000 |
| Docs page views (all projects) | 500K/mo | 3M/mo |
| Playground requests | 50K/mo | 300K/mo |
| Paying customers | 40 | 150 |
| MRR | $2,500 | $12,000 |
| GitHub syncs/week | 500 | 3,000 |

---

## Competitive Landscape

| Competitor | Strengths | Weaknesses | DocuMerge Advantage |
|-----------|-----------|------------|---------------------|
| ReadMe | Beautiful, established, feature-rich | Expensive ($99+/mo), complex | Affordable, simpler, auto-sync |
| Stoplight | Design-first, powerful editor | Steep learning curve, expensive | AI-powered, developer-friendly |
| Swagger UI | Free, ubiquitous | Ugly, no customization, no playground | Beautiful, branded, interactive |
| Redocly | Good open-source option | Limited cloud features | Full platform with playground |
| GitBook | Great for general docs | Not API-specific, no playground | Purpose-built for APIs |

---

## Risks & Mitigations

| Risk | Mitigation |
|------|------------|
| OpenAPI specs are messy/incomplete | Linting and validation on import, helpful error messages |
| Playground CORS issues | Sandboxed proxy server for API requests |
| Generated code examples have bugs | Template testing suite, user feedback loop, custom override option |
| Free tier abused for hosting | Rate limiting, reasonable resource caps |
| GitHub sync breaks on complex repos | Comprehensive integration tests, fallback to manual upload |

---

## MVP Scope (v1.0)

### In Scope
- OpenAPI 3.0/3.1 import (file upload or URL)
- Documentation site generation (beautiful default theme)
- Interactive API playground
- Auto-generated code examples (cURL, JavaScript, Python)
- GitHub auto-sync (rebuild on push)
- Basic branding (logo, colors)
- Full-text search
- Hosted on documerge.io subdomain

### Out of Scope
- GraphQL/AsyncAPI support
- Custom pages (MDX)
- Versioning
- Custom domain
- Advanced analytics
- SDK generation
- Private docs (SSO-protected)
- Custom code example overrides

### MVP Timeline: 7-8 weeks
- Week 1-2: OpenAPI parser, data model, endpoint extraction
- Week 3-4: Documentation site generation (Next.js static), theme, layout
- Week 5: API playground (request builder, proxy, response viewer)
- Week 6: Code example generator (cURL, JS, Python), GitHub App integration
- Week 7: Search, branding, hosting infrastructure
- Week 8: Billing, onboarding, polish, launch
