# OnboardFlow — Employee Onboarding Automation

## Executive Summary

OnboardFlow automates the entire employee onboarding process — from offer acceptance to full productivity. It orchestrates tasks across HR, IT, and managers through visual workflows, auto-provisions accounts, sends welcome sequences, and tracks completion so no step falls through the cracks.

---

## Problem Statement

**The pain:**
- Onboarding involves 30-50+ tasks across 5-10 people (HR, IT, manager, buddy, finance)
- Tasks are tracked in scattered spreadsheets, emails, and Slack messages
- Critical steps get missed: laptop not ordered, accounts not provisioned, access not granted
- New hires feel lost and unproductive for their first 2-4 weeks
- Every new hire is onboarded slightly differently, leading to inconsistent experiences
- Remote onboarding amplifies all these problems — no physical office to "figure things out"

**The cost:**
- 20% of employee turnover happens in the first 45 days (SHRM)
- Companies with strong onboarding improve new hire retention by 82% (Glassdoor)
- Poor onboarding costs up to $240K per bad departure (SHRM)
- HR teams spend 10+ hours per new hire on manual onboarding coordination

---

## Target Users

### Primary Personas

**1. Melissa — HR Manager**
- Handles onboarding for a 150-person company hiring 3-5 people/month
- Currently uses a Google Sheets checklist that's always out of date
- Needs: automated workflow, visibility into progress, zero dropped tasks

**2. Tomas — IT Administrator**
- Provisions accounts for every new hire (Google Workspace, Slack, GitHub, AWS)
- Gets last-minute requests: "New hire starts Monday, they need access to everything"
- Needs: automated provisioning, advance notice, standardized access by role

**3. Ava — Hiring Manager**
- Responsible for making sure her new hire has a great first week
- Often forgets to set up buddy, prepare first-week plan, or schedule intro meetings
- Needs: automated reminders, templates, structured first-week plan

---

## Core Features

### F1: Visual Workflow Builder
- Drag-and-drop workflow editor
- Node types:
  - **Task**: assigned to a person, with due date, instructions, and checklist
  - **Automated action**: trigger an integration (create account, send email, add to channel)
  - **Conditional branch**: if role = engineering, do X; if role = sales, do Y
  - **Delay/Wait**: wait N days or until a specific event
  - **Approval**: require sign-off before proceeding
  - **Notification**: send email/Slack message to someone
  - **Form**: collect information (e.g., equipment preferences, t-shirt size)
- Parallel paths: multiple task streams can run simultaneously
- Dependencies: task B can't start until task A is complete
- Templates: save and reuse workflows

### F2: Workflow Templates Library
- Pre-built templates:
  - Engineering onboarding (accounts, dev setup, architecture overview)
  - Sales onboarding (CRM access, product training, territory assignment)
  - Marketing onboarding (brand guidelines, tool access, content calendar)
  - Executive onboarding (leadership intros, strategy docs, board access)
  - General onboarding (universal steps for all roles)
- Customizable: use as starting point, modify to fit your process
- Community templates (shared by other organizations)

### F3: Task Management
- Task assignment to specific people or roles (e.g., "IT Admin", "Hiring Manager")
- Due dates relative to start date (e.g., "3 days before start", "Day 1", "End of Week 1")
- Task instructions with rich text, links, attachments
- Sub-tasks and checklists within tasks
- Task priority levels
- Comments and notes per task
- File attachments (upload documents, sign forms)
- Reminders: email/Slack notification when task is due, overdue

### F4: Automated Provisioning
- **Google Workspace**: create account, add to groups, set up email
- **Slack**: invite to workspace, add to channels based on role
- **GitHub/GitLab**: invite to organization, add to teams
- **Okta/Azure AD**: create user, assign applications
- **Notion**: invite to workspace, add to relevant pages
- **HRIS sync** (BambooHR, Rippling, Gusto): auto-import new hires and trigger workflows
- **Custom webhooks**: trigger any internal system
- De-provisioning support for offboarding (reverse all actions)

### F5: New Hire Portal
- Branded portal for the new hire to access:
  - Their personal onboarding checklist
  - Welcome message from manager
  - Pre-boarding forms (emergency contact, bank details, equipment preferences)
  - Company handbook and resources
  - Team directory
  - First-week schedule
  - FAQ section
- Progress tracker showing completion percentage
- Mobile-friendly responsive design
- Accessible before start date (pre-boarding)

### F6: Pre-Boarding Workflows
- Triggered on offer acceptance (before Day 1)
- Tasks: order laptop, set up desk/home office stipend, send welcome kit
- New hire tasks: fill out forms, review handbook, choose equipment
- Welcome email drip sequence:
  - Day of offer: "Welcome to the team!"
  - 1 week before start: "Here's what to expect on Day 1"
  - 1 day before: "See you tomorrow! Here's your schedule"
- Background check and document collection integration

### F7: Progress Tracking Dashboard
- Organization overview: all active onboardings with completion percentage
- Per-hire view: detailed task list with status (pending, in progress, completed, overdue)
- Bottleneck identification: which tasks are blocking progress
- Team view: all hires under a specific manager
- Overdue task alerts
- Time-to-productivity tracking (custom milestone)
- Historical analytics: average onboarding completion time, common bottlenecks

### F8: Offboarding Workflows
- Triggered on departure date or manual activation
- Account deprovisioning (reverse of provisioning)
- Equipment return tracking
- Exit interview scheduling
- Knowledge transfer task assignment
- Access revocation audit trail

### F9: E-Signature Collection
- Embed documents requiring signature in the workflow
- Collect digital signatures from new hires
- Countersignature from HR/manager
- Signed document storage
- Integration with DocuSign, HelloSign (v2)
- Built-in simple signature pad for basic needs

### F10: Reporting & Analytics
- Onboarding completion rate
- Average time to complete onboarding
- Task completion rate by assignee
- Most commonly overdue tasks
- New hire satisfaction survey results
- Onboarding NPS
- Cost per onboarding (if equipment/license costs tracked)

---

## Technical Architecture

### Tech Stack

| Layer | Technology |
|-------|-----------|
| Frontend | Next.js 14+, Tailwind CSS, React Flow (workflow editor), Radix UI |
| Backend | Next.js API Routes, tRPC |
| Database | PostgreSQL (Neon) |
| Workflow Engine | Temporal.io (durable execution for long-running workflows) |
| Auth | Clerk (with SSO for enterprise) |
| Email | Resend |
| File Storage | AWS S3 |
| Integrations | Google Admin SDK, Slack API, GitHub API, Okta API, etc. |
| Hosting | Vercel (app), AWS (Temporal workers) |

### Data Model

```
Organization
├── id, name, slug, plan
│
├── WorkflowTemplate[]
│   ├── id, org_id, name, description, role_type
│   ├── is_system_template, is_active
│   ├── trigger (hire/offboard/manual)
│   │
│   └── WorkflowNode[]
│       ├── id, template_id, type (task/action/condition/delay/approval/notification/form)
│       ├── config (JSON: assignee_role, instructions, integration, delay_days, condition_rules)
│       ├── position_x, position_y (for visual editor)
│       ├── parent_node_ids[] (dependencies)
│       └── sort_order
│
├── Onboarding[] (instance of a workflow for a specific hire)
│   ├── id, org_id, template_id
│   ├── new_hire_name, new_hire_email, role, department
│   ├── start_date, manager_user_id
│   ├── status (pre_boarding/active/completed/cancelled)
│   ├── completion_percentage
│   ├── temporal_workflow_id
│   ├── created_at, completed_at
│   │
│   └── OnboardingTask[] (instances of workflow nodes)
│       ├── id, onboarding_id, node_id
│       ├── assigned_to_user_id
│       ├── status (pending/in_progress/completed/skipped/overdue)
│       ├── due_date (absolute, calculated from start_date)
│       ├── completed_at, completed_by
│       ├── notes, attachments[]
│       └── sub_tasks (JSON[]: [{text, completed}])
│
├── Integration[]
│   ├── id, org_id, type (google/slack/github/okta/bamboohr)
│   ├── credentials (encrypted JSON)
│   ├── status (connected/error/disconnected)
│   └── last_synced_at
│
├── NewHirePortal[]
│   ├── id, onboarding_id, access_token (unique URL)
│   ├── welcome_message, resources (JSON)
│   ├── branding (JSON)
│   └── expires_at
│
└── Document[]
    ├── id, onboarding_id, name, type (handbook/form/signed_document)
    ├── file_url, requires_signature
    ├── signed_at, signed_by
    └── uploaded_at
```

---

## UI/UX — Key Screens

### 1. Workflow Builder
- Canvas with drag-and-drop nodes connected by edges
- Left panel: node type palette
- Click node to configure: assignee, instructions, due date offset, integration
- Conditional branches shown as diamond nodes with labeled paths
- Parallel paths visually separated
- Save as template, test run

### 2. Active Onboardings
- Card grid or table: one card per active onboarding
- Each card: hire name, role, start date, progress bar, manager name
- Status badges: Pre-boarding, Week 1, Week 2, Completed
- Sort by start date, completion %, status
- Filter by department, manager, status

### 3. Onboarding Detail
- Header: hire info, start date, progress percentage
- Task list: grouped by phase (Pre-boarding, Day 1, Week 1, Week 2, Month 1)
- Each task: assignee avatar, status, due date, expand for details
- Timeline view alternative: Gantt-like chart
- Comments/activity feed
- Actions: add task, reassign, skip, mark complete

### 4. New Hire Portal
- Clean, branded welcome page
- Personal checklist with checkboxes
- Welcome video/message from CEO or manager
- Quick links to important resources
- Forms to fill out (expandable inline)
- "Day 1 Schedule" section
- Team member photos and bios

### 5. Dashboard / Analytics
- Active onboardings count, average completion time
- Overdue tasks requiring attention
- Completion rate chart over time
- Top bottleneck tasks
- Team leaderboard: fastest average onboarding
- Upcoming start dates calendar

---

## Monetization

### Starter — $99/month
- Up to 5 hires per month
- 3 workflow templates
- Basic task management
- Email notifications
- New hire portal
- 2 integrations

### Growth — $249/month
- Up to 25 hires per month
- Unlimited templates
- All integrations
- Auto-provisioning
- Pre-boarding workflows
- E-signature collection
- Analytics dashboard
- 10 admin users

### Enterprise — Custom
- Unlimited hires
- SSO/SAML
- Custom integrations
- Offboarding workflows
- HRIS sync
- API access
- SLA
- Dedicated support
- Unlimited admin users

---

## Go-to-Market Strategy

- Target HR communities (SHRM, HR Twitter, People Operations groups)
- Content: "The true cost of bad onboarding" + "Your onboarding checklist template (free)"
- SEO: "employee onboarding software", "onboarding checklist template", "automated onboarding"
- Free onboarding checklist template (PDF) as lead magnet
- Partner with HRIS platforms for cross-promotion
- Sponsor HR conferences and People Ops meetups
- Case studies showing reduced time-to-productivity

---

## Success Metrics

| Metric | Target (Month 6) | Target (Month 12) |
|--------|-------------------|---------------------|
| Organizations | 50 | 200 |
| Hires onboarded | 500 | 3,000 |
| Tasks completed | 15,000 | 100,000 |
| Average completion rate | 85% | 92% |
| MRR | $10,000 | $40,000 |
| Paying customers | 40 | 120 |
| NPS from new hires | 50 | 65 |

---

## MVP Scope (v1.0)

### In Scope
- Visual workflow builder (task nodes, delays, notifications)
- 3 built-in templates (general, engineering, sales)
- Task assignment and tracking
- Email notifications and reminders
- New hire portal (basic)
- Progress dashboard
- Google Workspace provisioning
- Slack integration

### Out of Scope (v1.1+)
- Conditional branching
- Offboarding workflows
- E-signature collection
- HRIS sync
- Advanced analytics
- Pre-boarding email sequences
- GitHub/Okta provisioning
- Document management

### MVP Timeline: 8-10 weeks
- Week 1-2: Data model, workflow engine (Temporal setup), template system
- Week 3-4: Workflow builder UI (React Flow)
- Week 5: Task management, assignment, notifications
- Week 6: New hire portal
- Week 7: Google Workspace + Slack integrations
- Week 8: Dashboard, progress tracking
- Week 9-10: Billing, onboarding flow for admins, polish, launch
