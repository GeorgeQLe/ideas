# StatusPing — Uptime Monitoring with AI Incident Reports

## Executive Summary

StatusPing monitors your endpoints from multiple global regions and auto-generates incident reports when downtime is detected. It publishes to a hosted status page, notifies subscribers, and uses AI to draft post-mortems — so you never have to write "We're investigating the issue" by hand again.

---

## Problem Statement

**The pain:**
- When an outage hits, the last thing engineers want to do is write public-facing incident updates
- Status page updates are delayed because the person who can fix the issue is also the person expected to communicate about it
- Existing monitoring tools alert you but don't help you communicate — you still need a separate status page tool
- Small teams can't afford both Datadog/PagerDuty AND Statuspage.io
- Post-mortems are either never written or written weeks after the incident

**Current landscape:**
- **Better Uptime / UptimeRobot**: Good monitoring, basic status pages, no AI
- **Atlassian Statuspage**: Industry standard but expensive ($79+/mo), no monitoring built in
- **Instatus**: Modern alternative but still manual incident writing
- No tool combines monitoring + status page + AI-drafted communications

---

## Target Users

### Primary Personas

**1. Aiden — Solo DevOps / SRE**
- Only infrastructure person at a 20-person startup
- Manages 15+ services and APIs
- Gets paged at 3am and needs to communicate status while debugging
- Needs: auto-generated incident updates so he can focus on fixing

**2. Rachel — CTO at Early-Stage Startup**
- Wears the DevOps hat alongside everything else
- Customers ask "is the API down?" when issues happen
- Wants a professional status page without spending hours maintaining it
- Needs: set-it-and-forget-it monitoring with auto-communication

**3. Omar — Platform Team Lead**
- Manages reliability for 50+ microservices
- Team of 4 SREs, needs coordinated incident response
- Needs: multi-service monitoring, incident timelines, SLA tracking

---

## Core Features

### F1: Endpoint Monitoring
- **HTTP/HTTPS monitoring**: GET/POST/PUT with expected status codes, response body assertions, header checks
- **TCP port monitoring**: check if port is open and responding
- **DNS monitoring**: verify DNS resolution, record types, propagation
- **SSL certificate monitoring**: expiry alerts (30, 14, 7, 1 day)
- **Keyword monitoring**: check for presence/absence of specific strings in response
- **Multi-step API checks** (v2): chain requests (login → fetch → verify)
- Check intervals: 30 seconds, 1 minute, 5 minutes, 10 minutes
- Check from 6+ global regions (US-East, US-West, EU-West, EU-Central, Asia-Pacific, Australia)
- Confirmation from 2+ regions before declaring incident (prevent false positives)
- Response time tracking with percentile charts (p50, p95, p99)

### F2: Alerting
- Alert channels: email, Slack, Discord, Microsoft Teams, PagerDuty, Opsgenie, SMS, webhook
- Escalation policies: if not acknowledged in X minutes, alert next person
- On-call rotation schedules
- Alert grouping: don't spam when multiple monitors fail simultaneously
- Maintenance windows: suppress alerts during planned downtime
- Alert history and acknowledgment tracking

### F3: Hosted Status Page
- Beautiful, customizable status page at `yourapp.statusping.io`
- Custom domain with automatic SSL
- Branding: logo, colors, favicon, custom CSS
- Components: group monitors into customer-facing services
- Component groups (e.g., "API", "Dashboard", "Integrations")
- Real-time status indicators: Operational, Degraded, Partial Outage, Major Outage
- 90-day uptime history bars per component
- Response time graphs
- Scheduled maintenance display
- Subscriber management (email, SMS, webhook, RSS)
- Embed badge/widget for your own site
- Multiple status pages for different audiences (public, internal, partners)

### F4: AI Incident Management
- **Auto-detection**: monitoring failure triggers incident creation automatically
- **AI-drafted updates**: generates initial incident report from monitoring data:
  - "We are investigating increased error rates on the API. HTTP checks from US-East and EU-West are returning 503 errors since 14:32 UTC."
- **Progress updates**: AI drafts follow-up updates based on ongoing monitoring data
- **Resolution summary**: when monitors recover, AI drafts resolution message with timeline
- **Post-mortem generation**: AI creates draft post-mortem from incident timeline, including:
  - Timeline of events
  - Impact assessment (duration, affected components)
  - Root cause (if provided by team)
  - Action items template
- **Tone control**: professional, empathetic, technical
- **Human review**: all AI drafts require one-click approval before publishing (configurable to auto-publish)

### F5: Incident Timeline
- Chronological event log per incident
- Automatic entries: detected, acknowledged, investigating, update posted, resolved
- Manual entries: team members can add notes
- Severity levels: Minor, Major, Critical
- Duration tracking
- Affected components tagging
- Internal notes (visible only to team, not on status page)

### F6: SLA Reporting
- Define SLA targets per component (99.9%, 99.95%, 99.99%)
- Real-time SLA tracking with remaining error budget
- Monthly SLA report generation (PDF export)
- SLA breach alerts
- Historical SLA compliance charts
- Error budget burn rate visualization

### F7: Analytics & Reporting
- Uptime percentage over custom date ranges
- Response time trends with anomaly highlighting
- Incident frequency and MTTR (mean time to resolution)
- Most unreliable components ranking
- Downtime calendar view
- Exportable reports (PDF, CSV)

---

## Technical Architecture

### Tech Stack

| Layer | Technology |
|-------|-----------|
| Monitoring Workers | Go (lightweight, concurrent, deployed to multiple regions) |
| API / Dashboard | Next.js 14+, tRPC, Tailwind CSS |
| Database | PostgreSQL (Neon) |
| Time-series data | TimescaleDB extension (response times) |
| Cache / Pub-Sub | Redis (Upstash) |
| AI | OpenAI GPT-4o (incident drafting) |
| Status Pages | Cloudflare Workers + KV (edge-rendered, fast globally) |
| Email | Resend |
| SMS | Twilio |
| Hosting | Vercel (dashboard), Fly.io (monitoring workers in 6+ regions) |

### Data Model

```
Organization
├── id, name, slug, plan, created_at
│
├── Monitor[]
│   ├── id, org_id, name, type (http/tcp/dns/ssl)
│   ├── config (JSON: url, method, headers, body, assertions, port, hostname)
│   ├── interval_seconds, regions[]
│   ├── status (up/down/degraded/paused)
│   ├── last_checked_at, last_status_change_at
│   ├── alert_channels[] (→ AlertChannel)
│   └── component_id
│
├── Check[] (time-series, partitioned by month)
│   ├── id, monitor_id, region
│   ├── status (up/down), response_time_ms
│   ├── status_code, error_message
│   └── checked_at
│
├── Component[] (status page grouping)
│   ├── id, org_id, name, description, group_name
│   ├── sort_order, status (operational/degraded/partial/major)
│   └── sla_target_percent
│
├── StatusPage[]
│   ├── id, org_id, slug, custom_domain
│   ├── title, description, logo_url
│   ├── branding (JSON: colors, font, css)
│   ├── components[] (→ Component)
│   └── is_public
│
├── Incident[]
│   ├── id, org_id, title, status (investigating/identified/monitoring/resolved)
│   ├── severity (minor/major/critical)
│   ├── started_at, resolved_at
│   ├── affected_components[]
│   ├── auto_generated (boolean)
│   │
│   ├── IncidentUpdate[]
│   │   ├── id, incident_id, status, message
│   │   ├── is_ai_generated, is_internal
│   │   ├── posted_by, posted_at
│   │   └── ai_draft (text, before human edit)
│   │
│   └── PostMortem
│       ├── id, incident_id
│       ├── timeline_md, impact_md, root_cause_md, action_items_md
│       ├── ai_generated_draft, published_at
│       └── status (draft/published)
│
├── AlertChannel[]
│   ├── id, org_id, type (email/slack/discord/pagerduty/sms/webhook)
│   └── config (JSON, encrypted)
│
├── Subscriber[]
│   ├── id, status_page_id, type (email/sms/webhook)
│   ├── endpoint (email or phone or url)
│   └── confirmed, subscribed_at
│
└── MaintenanceWindow[]
    ├── id, org_id, title, description
    ├── starts_at, ends_at
    ├── affected_components[]
    └── auto_created_from_incident
```

---

## UI/UX — Key Screens

### 1. Monitor Dashboard
- Grid/list of all monitors with status badges (green/yellow/red)
- Sparkline showing last 24h response time per monitor
- Uptime percentage for last 7/30/90 days
- Quick actions: pause, edit, delete
- "Add monitor" prominent button
- Filter by status, component, region

### 2. Monitor Detail
- Large response time chart (1h, 24h, 7d, 30d views)
- Uptime bar (green/red segments over time)
- Check log table (recent checks with status, response time, region)
- Configuration panel
- Incident history for this monitor
- SLA compliance meter

### 3. Status Page Editor
- Live preview of public status page
- Component list with drag-to-reorder
- Branding customization panel
- Custom domain configuration
- Subscriber settings

### 4. Incident View
- Timeline of updates (visual, chronological)
- AI draft panel: shows generated draft with "Approve", "Edit", "Regenerate"
- Affected components selector
- Severity selector
- Internal notes section
- Resolution button with auto-generated summary

### 5. Post-Mortem Editor
- AI-generated draft with sections: Timeline, Impact, Root Cause, Action Items
- Rich text editor for each section
- Publish to status page button
- Export as PDF

### 6. Alerting Configuration
- Connected alert channels list
- Per-monitor alert routing
- Escalation policy builder
- On-call schedule calendar
- Test alert button

---

## Monetization

### Free Tier
- 5 monitors
- 5-minute check interval
- 1 status page (statusping.io subdomain)
- Email alerts only
- 7-day data retention
- No AI features

### Pro — $29/month
- 50 monitors
- 30-second check interval
- Custom domain status page
- All alert channels
- AI incident drafts
- 90-day data retention
- SLA tracking
- SSL monitoring

### Business — $79/month
- Unlimited monitors
- 30-second interval
- Multiple status pages
- AI post-mortem generation
- 1-year data retention
- On-call schedules and escalation
- SMS alerts
- API access
- Priority support

### Enterprise — Custom
- Dedicated monitoring infrastructure
- Custom check regions
- SSO/SAML
- SLA guarantee on StatusPing itself
- Unlimited data retention
- Dedicated support

---

## Success Metrics

| Metric | Target (Month 6) | Target (Month 12) |
|--------|-------------------|---------------------|
| Active monitors | 5,000 | 25,000 |
| Organizations | 300 | 1,200 |
| Status page monthly visitors (all customers) | 200K | 1M |
| AI drafts generated | 500/mo | 3,000/mo |
| MRR | $4,000 | $18,000 |
| Paying customers | 80 | 300 |

---

## Competitive Landscape

| Competitor | Strengths | Weaknesses | StatusPing Advantage |
|-----------|-----------|------------|---------------------|
| Better Uptime | Good UX, free tier | No AI, basic status pages | AI incident management |
| UptimeRobot | Cheap, reliable, established | Dated UI, no AI, basic status | Modern UX, AI drafts |
| Atlassian Statuspage | Industry standard | Expensive, no monitoring, no AI | All-in-one, cheaper, AI |
| Instatus | Modern, fast | Manual incidents, no monitoring | Monitoring + AI = fully automated |
| Pingdom | Enterprise-grade | Expensive, complex, legacy | Simple, affordable, AI-native |

---

## Risks & Mitigations

| Risk | Mitigation |
|------|------------|
| False positives trigger wrong AI incidents | Multi-region confirmation, configurable sensitivity, manual approval default |
| AI writes inaccurate incident details | Always show as draft with human approval, conservative language |
| Monitoring infrastructure itself goes down | Multi-cloud deployment, self-monitoring, automated failover |
| Free tier users cost more than they're worth | Rate limit free checks, push upgrade for more monitors/speed |

---

## MVP Scope (v1.0)

### In Scope
- HTTP/HTTPS monitoring (5 regions, 1-min minimum interval)
- Email and Slack alerts
- Hosted status page with basic branding
- Auto-incident creation on downtime
- AI-drafted incident updates (with approval)
- Basic response time charts

### Out of Scope (v1.1+)
- TCP/DNS/SSL monitoring
- AI post-mortems
- Custom domains
- On-call schedules and escalation
- SLA tracking
- SMS alerts
- Multi-step API checks

### MVP Timeline: 7-8 weeks
- Week 1-2: Monitoring worker (Go), multi-region deployment, check storage
- Week 3: Alert system, Slack/email integration
- Week 4-5: Dashboard UI, monitor management, charts
- Week 6: Status page generation, subscriber system
- Week 7: AI incident draft pipeline, incident management UI
- Week 8: Billing, polish, launch
