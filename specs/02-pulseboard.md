# PulseBoard — Remote Team Energy Tracker

## Executive Summary

PulseBoard is a lightweight daily check-in tool that gives remote team leads real-time visibility into team morale and energy levels. Team members share a quick 5-second check-in, and PulseBoard aggregates the data into trends, heatmaps, and burnout risk alerts — enabling managers to intervene before people burn out.

---

## Problem Statement

**The pain:**
- Remote managers can't "read the room" — they have no way to sense team energy levels without physical proximity
- Burnout is invisible until someone quits or goes on medical leave — the warning signs are silent over Slack
- Traditional engagement surveys are quarterly, long, and by the time results arrive, the damage is done
- 1-on-1s are too infrequent (weekly at best) to catch rapid morale shifts
- Existing tools (Officevibe, Lattice) are heavyweight HR platforms, not lightweight daily pulse tools

**Current workarounds:**
- Managers ask "how's everyone doing?" in Slack standups (get polite non-answers)
- Annual/quarterly engagement surveys (too slow, too heavy)
- Gut feeling (unreliable, biased toward the loudest voices)
- No system at all (most common)

**The cost of inaction:**
- Average cost to replace a developer: $50K-$100K+
- Disengaged employees are 18% less productive (Gallup)
- Burnout leads to higher error rates, more bugs, more incidents

---

## Target Users

### Primary Personas

**1. David — Engineering Manager**
- Manages 8 engineers across 3 time zones
- Has weekly 1-on-1s but worries about blind spots between meetings
- Wants a lightweight pulse without being intrusive
- Needs: trends over time, early warning system, conversation starters

**2. Lisa — Head of People / HR**
- Responsible for culture and retention at a 60-person startup
- Needs company-wide visibility without micromanaging
- Wants data to present to leadership about team health
- Needs: aggregate trends, team comparisons, anonymous data

**3. Alex — Team Lead / IC Lead**
- Leads a squad of 4-5 people
- Wants to be a good manager but doesn't have formal training
- Needs: simple tool that tells them when to check in on someone

### Secondary Personas
- Remote-first founders wanting culture metrics
- Agile coaches tracking team health across sprints
- Project managers monitoring morale during high-pressure deadlines

---

## Solution Overview

PulseBoard is a 3-part system:
1. **Daily check-in** — a 5-second interaction via Slack bot, web app, or mobile
2. **Team dashboard** — real-time visibility into trends and alerts
3. **Insights engine** — AI-powered analysis of patterns and recommendations

---

## Core Features

### F1: Daily Check-In
- **Energy scale**: 1-5 (with emoji labels: 1=Exhausted, 2=Low, 3=Okay, 4=Good, 5=Great)
- **Optional mood tag**: select from predefined tags (stressed, motivated, bored, focused, overwhelmed, excited, anxious, calm)
- **Optional note**: free-text field (visible to self only, or shared with manager if opted in)
- **Configurable check-in time**: organization sets default, users can adjust
- **Reminder if not submitted by cutoff time**
- **Weekend/PTO skip**: auto-detect from calendar or manual setting
- **Historical personal view**: "Your last 30 days" for self-reflection

### F2: Slack / Teams Integration
- Slash command: `/pulse` opens modal with check-in
- Daily DM prompt at configured time
- Respond with emoji reaction (quick mode): 🔴🟠🟡🟢🔵 maps to 1-5
- Thread-based check-in for async teams
- Channel posting option: share team aggregate (not individual) to a channel
- Microsoft Teams bot with equivalent functionality

### F3: Team Dashboard
- **Heatmap view**: calendar grid showing team energy by day (color-coded)
- **Trend line**: rolling 7-day and 30-day average energy per team
- **Individual view** (manager-only): per-person trend line
- **Comparison view**: compare teams/departments
- **Distribution chart**: daily breakdown of 1-5 responses
- **Participation rate**: track who's checking in vs not
- **Time zone handling**: show check-ins aligned to each person's local time

### F4: Burnout Risk Alerts
- Trigger alert if individual scores ≤2 for 3+ consecutive days
- Trigger alert if team average drops 20%+ from baseline
- Trigger alert if participation rate drops below 50%
- Configurable thresholds per organization
- Alerts delivered via email, Slack DM, or in-app notification
- Suggested actions: "Schedule a 1-on-1 with [person]", "Consider a team social"
- Alert history log

### F5: Anonymous Mode
- Organization-level or team-level toggle
- When enabled: managers see aggregate data only, no individual scores
- Individual scores visible only to the person themselves
- Minimum team size for anonymity: 4 people (to prevent deduction)
- Anonymous comments/notes routed to managers without attribution
- Trust badge shown to team members: "Your responses are anonymous"

### F6: Weekly Summary Digest
- Auto-generated email/Slack message every Monday
- Contents: team average, trend direction, participation rate, notable patterns
- Highlight: "3 people had a tough week" (anonymous)
- AI-generated talking points for upcoming 1-on-1s
- Exportable PDF for leadership reviews

### F7: 1-on-1 and Retro Tools
- Auto-generated conversation prompts based on recent check-in patterns
- "Your team's energy dipped on Wednesday — was that related to the production incident?"
- Sprint retro integration: overlay check-in data with sprint timelines
- Shareable team health summary for retro meetings
- Track improvement actions and their impact on subsequent scores

### F8: Integrations
- **Calendar integration** (Google Calendar, Outlook): auto-skip check-ins on PTO/holidays
- **Jira/Linear**: correlate energy with sprint velocity
- **BambooHR/Rippling**: auto-sync team structure and PTO
- **Webhook/API**: send data to your own systems
- **Zapier**: connect to 5000+ apps

### F9: Admin & Configuration
- Organization structure: teams, departments, managers
- Bulk user import (CSV, Google Workspace, Okta)
- Check-in schedule configuration (days, time, timezone handling)
- Custom energy labels (replace 1-5 with custom text)
- Custom mood tags
- Data retention policies
- GDPR data export and deletion

---

## Technical Architecture

### System Diagram

```
Slack/Teams ──bot event──▶ Bot Service (webhooks)
                                  │
Web App / Mobile ────────────────▶│
                                  │
                                  ▼
                           API Server (Next.js)
                                  │
                    ┌─────────────┼─────────────┐
                    ▼             ▼             ▼
               PostgreSQL     Redis         AI Service
               (check-ins,   (sessions,    (weekly summaries,
                users,        rate limit,    insights,
                teams)        pub/sub)       prompts)
                    │
                    ▼
              ┌───────────┐
              │ Cron Jobs  │
              │ - Reminders│
              │ - Digests  │
              │ - Alerts   │
              └───────────┘
```

### Tech Stack

| Layer | Technology |
|-------|-----------|
| Frontend | Next.js 14+, Tailwind CSS, Recharts, Framer Motion |
| Backend | Next.js API Routes / tRPC |
| Database | PostgreSQL (Supabase) |
| Cache/Queue | Redis (Upstash) |
| Auth | Clerk (SSO support) |
| Bot | Slack Bolt SDK, Microsoft Bot Framework |
| AI | OpenAI GPT-4o-mini (summaries and insights) |
| Email | Resend |
| Hosting | Vercel |
| Monitoring | Sentry, Vercel Analytics |

### Data Model

```
Organization
├── id, name, slug, plan, timezone, settings (JSON)
│
├── Team[]
│   ├── id, org_id, name, manager_user_id
│   └── anonymous_mode (boolean)
│
├── User[]
│   ├── id, org_id, team_id, name, email, role (member/manager/admin)
│   ├── timezone, slack_user_id, teams_user_id
│   ├── check_in_time, skip_weekends
│   └── status (active/paused/deactivated)
│
├── CheckIn[]
│   ├── id, user_id, team_id, org_id
│   ├── date (DATE), energy (1-5)
│   ├── mood_tags (text[]), note (text, encrypted)
│   ├── source (slack/web/mobile/teams)
│   └── created_at
│
├── Alert[]
│   ├── id, org_id, team_id, user_id (nullable if team-level)
│   ├── type (individual_burnout/team_dip/low_participation)
│   ├── severity (warning/critical)
│   ├── message, triggered_at, acknowledged_at
│   └── acknowledged_by
│
└── Digest[]
    ├── id, org_id, team_id, period_start, period_end
    ├── avg_energy, participation_rate
    ├── ai_summary, ai_talking_points
    └── created_at
```

### API Design

```
Auth:
POST   /api/auth/login              # Email magic link or SSO
POST   /api/auth/slack               # Slack OAuth

Check-ins:
POST   /api/checkins                 # Submit check-in
GET    /api/checkins/me              # My check-in history
GET    /api/checkins/today           # Today's status

Teams:
GET    /api/teams                    # List teams
GET    /api/teams/:id/dashboard      # Team dashboard data
GET    /api/teams/:id/heatmap        # Heatmap data (date range)
GET    /api/teams/:id/trends         # Trend data

Alerts:
GET    /api/alerts                   # Active alerts
PATCH  /api/alerts/:id/acknowledge   # Acknowledge alert

Digests:
GET    /api/digests                  # Past digests
GET    /api/digests/latest           # Most recent

Admin:
GET    /api/admin/users              # List users
POST   /api/admin/users/invite       # Invite users
PATCH  /api/admin/settings           # Org settings
POST   /api/admin/import             # Bulk CSV import

Integrations:
POST   /api/webhooks/slack           # Slack events
POST   /api/webhooks/teams           # Teams events
GET    /api/integrations             # List connected integrations

Public (widget):
POST   /api/bot/checkin              # Bot-submitted check-in
```

---

## UI/UX — Key Screens

### 1. Check-In Screen (Web/Mobile)
- Large, friendly interface with 5 energy buttons (emoji-based)
- Optional mood tag chips below
- Optional text field that expands on click
- "Skip today" link
- Streak counter ("12-day check-in streak!")
- Takes <5 seconds for minimal check-in

### 2. Personal History
- Calendar view with color-coded days
- 30-day trend line
- Most common mood tags word cloud
- Journal entries (if notes were added)
- Export data option

### 3. Manager Dashboard
- Team overview cards (one per team)
- Each card shows: today's average, 7-day trend arrow, participation rate
- Click into team for detailed heatmap and individual trends
- Alert banner at top if any active alerts
- Quick action buttons: "Send encouragement", "Schedule 1-on-1"

### 4. Heatmap View
- Rows = team members (or anonymous IDs), Columns = dates
- Color scale from red (1) to green (5)
- Hover for details
- Filter by date range
- Toggle between individual and team aggregate view

### 5. Settings / Admin
- Organization settings (name, timezone, check-in schedule)
- Team management (create, assign members, set managers)
- Integration connections (Slack, Calendar, HR tools)
- Privacy settings (anonymous mode toggle per team)
- Billing and plan management

---

## Monetization

### Free Tier
- Up to 5 people
- Basic check-in (energy only, no mood tags)
- 7-day history
- No Slack integration

### Team — $4/user/month
- Unlimited history
- Mood tags and notes
- Slack/Teams integration
- Manager dashboard with heatmaps
- Burnout risk alerts
- Weekly digest
- Calendar integration

### Enterprise — $8/user/month
- Everything in Team
- Anonymous mode
- SSO/SAML
- API access
- Custom integrations
- Advanced analytics
- Dedicated support
- Data retention policies
- HIPAA compliance option

---

## Go-to-Market Strategy

### Phase 1: Early Adopters (Month 1-3)
- Launch in remote work communities (RemoteOK, We Work Remotely, Indie Hackers)
- Free tool for open-source teams
- Write thought leadership: "The invisible burnout crisis in remote teams"
- Partner with remote work influencers / podcasters

### Phase 2: Product-Led Growth (Month 3-6)
- Free tier with viral loop: team members who leave see "Powered by PulseBoard"
- Slack App Directory listing
- Content marketing: "5 signs your remote team is burning out"
- Retro template partnerships (Miro, FigJam, Retrium)

### Phase 3: B2B Sales (Month 6-12)
- Target companies with 50-500 remote employees
- HR/People team outreach
- Case studies from early adopters
- SOC 2 compliance for enterprise deals
- Partnership with HR platforms (BambooHR, Rippling, Gusto)

---

## Success Metrics & KPIs

| Metric | Target (Month 6) | Target (Month 12) |
|--------|-------------------|---------------------|
| Active organizations | 200 | 800 |
| Active users | 2,000 | 10,000 |
| Daily check-in rate | 65% | 75% |
| MRR | $3,000 | $15,000 |
| Avg check-ins per user/month | 18 | 20 |
| Alert → 1-on-1 conversion | 30% | 45% |
| NPS | 40 | 55 |

---

## Competitive Landscape

| Competitor | Strengths | Weaknesses | PulseBoard Advantage |
|-----------|-----------|------------|---------------------|
| Officevibe | Full engagement platform | Heavy, expensive, survey fatigue | Lightweight, 5-second daily pulse |
| Friday.app | Check-ins + goals | Broader focus, not mood-specific | Purpose-built for energy/burnout |
| Geekbot | Async standups | No mood tracking or analytics | Dedicated mood/energy focus |
| Lattice | Full HR suite | Enterprise-only, complex setup | Simple, affordable, fast setup |
| Custom Slack bots | Free, customizable | No analytics, no alerts, fragile | Polished, analytics, alerts |

---

## Risks & Mitigations

| Risk | Impact | Likelihood | Mitigation |
|------|--------|------------|------------|
| Employees feel surveilled | High | High | Anonymous mode, transparent data policy, employee-first design, optional participation |
| Low daily participation | High | Medium | Slack integration (low friction), gamification (streaks), manager modeling behavior |
| Data privacy concerns (GDPR) | High | Medium | Data encryption, retention policies, right to delete, European hosting option |
| Managers ignore alerts | Medium | Medium | Actionable suggestions, 1-on-1 prompts, track alert acknowledgment |
| "Survey fatigue" — people stop caring | Medium | Medium | Keep it under 5 seconds, vary prompts, show value back to individuals |

---

## MVP Scope (v1.0)

### In Scope
- Web app check-in (1-5 energy + optional note)
- Slack bot check-in
- Manager dashboard with trend line
- Team heatmap
- Basic burnout alerts (3 consecutive days ≤2)
- Email digest (weekly)

### Out of Scope (v1.1+)
- Microsoft Teams
- Mood tags
- Anonymous mode
- Calendar integration
- AI-generated 1-on-1 prompts
- Sprint/velocity correlation
- Mobile app

### MVP Timeline: 5-6 weeks
- Week 1: Auth, data model, check-in API, basic UI
- Week 2: Slack bot integration
- Week 3: Manager dashboard, heatmap, trend charts
- Week 4: Alert system, email digest
- Week 5: Billing, onboarding flow, polish
- Week 6: Beta launch, iterate on feedback
