# PermitFlow — Access Control Dashboard

## Executive Summary

PermitFlow provides a centralized dashboard to view, manage, and audit user access across all your SaaS tools. It connects to 100+ apps, shows who has access to what, automates access reviews, provisions/deprovisions accounts on hire/departure, and generates compliance reports for SOC 2 and ISO 27001.

---

## Problem Statement

**The pain:**
- As companies grow past 50 employees, nobody knows who has access to what across dozens of SaaS tools
- Former employees still have active accounts months after leaving (security risk)
- Provisioning a new hire requires manually creating accounts in 10-15 tools
- Quarterly access reviews are done in spreadsheets — hours of manual work per review
- SOC 2 auditors ask for access control evidence that takes days to compile
- Shadow IT: employees sign up for tools that IT doesn't know about

**The cost:**
- Average company has 254 SaaS apps (Productiv data)
- 43% of ex-employees still have access to corporate apps after leaving (Oomnitza)
- Manual provisioning takes 4+ hours per new hire
- Failed SOC 2 audits delay enterprise deals by months

---

## Target Users

### Primary Personas

**1. Chris — IT Administrator**
- Manages 80+ SaaS tools for a 200-person company
- Gets Slack messages daily: "Can I get access to X?"
- Needs: self-service access requests, automated provisioning, central view

**2. Laura — Security/Compliance Manager**
- Responsible for SOC 2 and ISO 27001 compliance
- Spends 2 weeks per quarter doing manual access reviews
- Needs: automated access reviews, compliance reports, audit trails

**3. Diana — Head of IT**
- Reports to CTO on security posture
- Needs to justify SaaS spend and identify redundant tools
- Needs: SaaS discovery, license utilization, cost tracking

---

## Core Features

### F1: SaaS App Connections (100+)
- **Productivity**: Google Workspace, Microsoft 365, Slack, Zoom, Notion, Confluence
- **Development**: GitHub, GitLab, Bitbucket, AWS, GCP, Azure, Vercel, Heroku
- **Security/Identity**: Okta, Azure AD, Google Identity, OneLogin, JumpCloud
- **Business**: Salesforce, HubSpot, Jira, Linear, Asana, Monday
- **Finance**: QuickBooks, Stripe, Brex, Expensify
- **HR**: BambooHR, Rippling, Gusto, Workday
- **Support**: Zendesk, Intercom, Freshdesk
- **Analytics**: Mixpanel, Amplitude, Datadog, Sentry
- Connection via: OAuth, API token, SCIM, SAML, or read-only admin API
- Sync frequency: real-time (webhook) or scheduled (hourly/daily)

### F2: Unified Access View
- Single dashboard showing all users and their app access
- Per-user view: "John has access to 23 apps: [list with roles/permissions]"
- Per-app view: "GitHub has 45 users: [list with roles]"
- Permission level detail: admin, editor, viewer, custom roles
- Last login date per app per user
- Access anomalies highlighted: "Jane hasn't used Figma in 90 days"
- Group/team access view: what does the Engineering team have access to?
- Searchable and filterable across all dimensions

### F3: Automated Access Reviews
- Schedule quarterly, semi-annual, or custom cadence reviews
- Review campaigns: select scope (all apps, specific apps, specific teams)
- Assign reviewers: manager reviews their reports, app owners review their apps
- Review interface: for each user-app combination, reviewer selects: Approve, Revoke, Flag
- Bulk approve/revoke for efficiency
- Reviewer reminders and escalation if incomplete
- Review completion tracking dashboard
- Audit trail: every decision logged with reviewer, timestamp, justification
- Auto-revoke on "Revoke" decision (with confirmation)

### F4: Automated Provisioning & Deprovisioning
- **New hire provisioning**: connect to HRIS, automatically create accounts based on role
  - Role templates: "Engineering" → GitHub, AWS, Slack (channels), Jira, Datadog
  - Department templates: "Sales" → Salesforce, HubSpot, Gong, Slack (channels)
- **Offboarding deprovisioning**: triggered by HRIS departure event
  - Revoke access across all connected apps
  - Transfer ownership of files/data
  - Generate offboarding report
  - Grace period option (keep accounts suspended for N days before deletion)
- **SCIM protocol**: for apps supporting automated user lifecycle management
- **Custom webhooks**: for internal tools

### F5: Self-Service Access Requests
- Employee portal: "Request access to [app]"
- Request form: app, permission level, justification, duration (permanent or temporary)
- Approval workflow: request → manager approval → IT approval (configurable chain)
- Slack integration: approve/deny requests from Slack
- Auto-approval rules: "Auto-approve Notion access for all employees"
- Time-bound access: automatic revocation after specified period
- Request history and status tracking

### F6: Shadow IT Discovery
- Monitor SSO logs and email for SaaS app usage not in your inventory
- Browser extension (optional): detect SaaS apps visited by employees
- Expense report scanning: find SaaS subscriptions in credit card statements
- Alert on new, unrecognized apps
- App catalog: known apps with security/compliance ratings
- "Sanctioned" vs "Unsanctioned" classification

### F7: Compliance Reporting
- **SOC 2** report package:
  - Access control policies evidence
  - User access review evidence
  - Provisioning/deprovisioning evidence
  - Least privilege compliance evidence
- **ISO 27001** evidence collection
- **Custom compliance frameworks**: define your own controls
- Automated evidence collection on schedule
- Report export: PDF, CSV
- Auditor view: read-only access for external auditors

### F8: SaaS Spend & License Management
- Track license counts and costs per app
- Identify unused licenses: "12 people have Figma licenses but haven't logged in for 60 days"
- Cost per user/team reporting
- License utilization percentage
- Renewal date tracking and alerts
- "You could save $X/month by removing unused licenses"
- Vendor contract metadata storage

### F9: Audit Log
- Every action logged: who accessed what, when, from where
- Filter by user, app, action type, date range
- Exportable for compliance and investigation
- Real-time alert on suspicious access patterns
- API access to audit data

---

## Technical Architecture

### Tech Stack

| Layer | Technology |
|-------|-----------|
| Frontend | Next.js 14+, Tailwind CSS, Radix UI, AG Grid |
| Backend | Next.js API Routes, tRPC |
| Database | PostgreSQL (Neon) |
| Background Sync | Temporal.io (durable workflow engine for sync jobs) |
| Auth | Clerk (with SAML/SSO for enterprise) |
| Email | Resend |
| Hosting | Vercel (app), AWS ECS (sync workers) |

### Data Model

```
Organization
├── id, name, slug, plan
│
├── User[]
│   ├── id, org_id, name, email, department, role, manager_id
│   ├── employment_status (active/offboarding/offboarded)
│   ├── hris_id, start_date, end_date
│   └── last_synced_at
│
├── App[]
│   ├── id, org_id, name, category, icon_url
│   ├── connection_type (oauth/scim/api_token)
│   ├── credentials (encrypted JSON)
│   ├── sync_status, last_synced_at
│   ├── license_count, cost_per_license, renewal_date
│   ├── owner_user_id
│   └── is_sanctioned
│
├── AppAccess[]
│   ├── id, app_id, user_id
│   ├── role (admin/member/viewer/custom)
│   ├── permissions (JSON, app-specific)
│   ├── granted_at, granted_by
│   ├── last_login_at
│   ├── status (active/suspended/revoked)
│   ├── expires_at (for time-bound access)
│   └── source (provisioned/manual/self_service/discovered)
│
├── RoleTemplate[]
│   ├── id, org_id, name (e.g., "Engineering")
│   ├── description
│   └── app_accesses (JSON[]: [{app_id, role}])
│
├── AccessReview[]
│   ├── id, org_id, name, cadence
│   ├── scope (JSON: {apps, teams, all})
│   ├── status (scheduled/in_progress/completed)
│   ├── started_at, due_date, completed_at
│   │
│   └── ReviewItem[]
│       ├── id, review_id, app_access_id
│       ├── reviewer_user_id
│       ├── decision (pending/approved/revoked/flagged)
│       ├── justification
│       └── decided_at
│
├── AccessRequest[]
│   ├── id, org_id, requester_user_id
│   ├── app_id, requested_role, justification
│   ├── duration (permanent/temporary + end_date)
│   ├── status (pending/approved/denied/expired)
│   ├── approved_by, decided_at
│   └── auto_approved (boolean)
│
└── AuditLog[]
    ├── id, org_id, actor_user_id
    ├── action (granted/revoked/reviewed/requested/provisioned/deprovisioned)
    ├── target_user_id, app_id
    ├── details (JSON)
    └── created_at
```

---

## UI/UX — Key Screens

### 1. Organization Dashboard
- Summary cards: total users, total apps, active access reviews, pending requests
- Security score: based on review completion, unused accounts, offboarded cleanup
- Alert feed: anomalies, expired reviews, shadow IT discoveries
- Quick actions: start access review, provision new hire

### 2. User Directory
- Searchable/filterable user list
- Each row: name, department, role, number of apps, last activity
- Click into user: full app access list, request history, review history
- "Offboard" button with confirmation and impact preview

### 3. App Inventory
- Grid/table of all connected apps with status
- Each app: name, icon, user count, license utilization, cost, sync status
- Click into app: user list with roles, last login dates, usage stats
- "Connect new app" flow

### 4. Access Review
- Review campaign dashboard: progress bar, pending reviews, completion rate
- Reviewer view: table of user-app pairs to review
- Bulk approve, individual revoke with justification
- Completion tracking per reviewer

### 5. Access Request Portal (Employee-Facing)
- App catalog with search
- "Request access" form per app
- My requests: status tracking
- Manager view: pending approvals

### 6. Compliance Center
- Framework selector (SOC 2, ISO 27001, custom)
- Control checklist with evidence status
- Generate report button
- Evidence artifact viewer
- Audit log search

---

## Monetization

### Starter — $199/month
- Up to 50 users
- 10 app connections
- Unified access view
- Basic audit log
- Manual provisioning only

### Business — $499/month
- Up to 200 users
- Unlimited app connections
- Automated access reviews
- Self-service access requests
- Automated provisioning (SCIM)
- Compliance reports
- Slack integration

### Enterprise — Custom
- Unlimited users
- SSO/SAML
- Shadow IT discovery
- SaaS spend management
- Custom compliance frameworks
- API access
- SLA
- Dedicated support
- On-premise option

---

## Go-to-Market Strategy

- Target IT admins and security teams at 50-500 person companies
- Content: "Your ex-employees still have access to everything" (security angle)
- SOC 2 readiness guide (lead magnet)
- SEO: "SaaS access management", "access review tool", "SOC 2 access control"
- Partner with Okta/Azure AD ecosystem
- Security conference sponsorships
- Compliance consultant referral program

---

## Success Metrics

| Metric | Target (Month 6) | Target (Month 12) |
|--------|-------------------|---------------------|
| Organizations | 30 | 100 |
| Users under management | 5,000 | 20,000 |
| App connections | 500 | 2,500 |
| Access reviews completed | 50 | 300 |
| MRR | $12,000 | $50,000 |
| Paying customers | 20 | 60 |

---

## MVP Scope (v1.0)

### In Scope
- 15 app connections (Google Workspace, Slack, GitHub, AWS, Jira, Notion, + 9 more)
- Unified access view (user → apps, app → users)
- Basic access review workflow
- Manual provisioning assistance (checklist, not automated)
- Audit log
- Email notifications

### Out of Scope
- SCIM automated provisioning
- Self-service access requests
- Shadow IT discovery
- Compliance report generation
- SaaS spend tracking
- Slack integration

### MVP Timeline: 10-12 weeks
- Week 1-3: Integration framework, first 8 app connectors
- Week 4-5: Unified access view, user directory, app inventory
- Week 6-7: Access review workflow
- Week 8-9: Remaining 7 connectors, audit log
- Week 10: Dashboard, notifications
- Week 11-12: Billing, onboarding, security review, launch
