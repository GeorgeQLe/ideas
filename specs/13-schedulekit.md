# ScheduleKit — Embeddable Scheduling Widget

## Executive Summary

ScheduleKit is a fully customizable, white-label scheduling widget you embed in your own website with one line of code. Unlike Calendly, it looks native to your site — matching your fonts, colors, and design system. It handles calendar sync, timezone detection, payments, and team round-robin scheduling.

---

## Problem Statement

**The pain:**
- Calendly works but it's obviously a third-party tool — the redirect breaks the user experience
- Embedding Calendly still looks like Calendly, not your brand
- SaaS companies want scheduling built into their app (onboarding calls, demos, support) but building it from scratch takes months
- Existing tools are "my booking page" oriented, not "embed in my product" oriented
- Agencies and consultants want scheduling that looks premium, not generic

---

## Target Users

### Primary Personas

**1. Mike — SaaS Product Manager**
- Wants to embed "Book a Demo" directly in the product, not redirect to Calendly
- Needs: JavaScript SDK, custom styling, webhook integration

**2. Angela — Freelance Consultant**
- Has a portfolio website and wants scheduling that matches her design
- Currently uses Calendly but hates the visual disconnect
- Needs: fully branded widget, payment collection

**3. Nate — Agency Owner**
- Builds websites for clients who need booking functionality
- Needs: white-label solution they can resell, different configs per client

---

## Core Features

### F1: Embeddable Widget
- **One-line embed**: `<script src="schedulekit.js" data-id="your-id"></script>`
- **React/Vue/Angular components**: first-class framework support
- **Display modes**:
  - Inline: renders within your page flow
  - Popup: triggered by button click
  - Slide-in: panel from side of screen
  - Full page: standalone booking page (schedulekit.io/you)
- **Responsive**: works on mobile, tablet, desktop
- **Lightweight**: <20KB gzipped
- **Accessible**: WCAG 2.1 AA compliant
- **No Powered By**: completely white-label on paid plans

### F2: Deep Customization
- **CSS variables**: override colors, fonts, spacing, border radius
- **Theme presets**: light, dark, minimal, rounded, sharp
- **Custom CSS injection**: full CSS override for complete control
- **Layout options**: calendar view, list view, compact view
- **Language/locale**: 15+ languages, custom translations
- **RTL support**: for Arabic, Hebrew, etc.
- **Custom fields**: add intake questions to the booking form
- **Custom confirmation page**: redirect or show custom content

### F3: Calendar Sync
- **Google Calendar**: two-way sync (availability + booked events)
- **Microsoft Outlook/365**: two-way sync
- **Apple iCal**: read-only availability
- **CalDAV**: generic calendar protocol support
- **Multiple calendars**: check availability across personal + work calendars
- **Buffer time**: configurable gaps between meetings (15, 30, 60 min)
- **Daily meeting limits**: max 5 meetings per day
- **Minimum notice**: require at least 24 hours advance booking
- **Date range**: allow bookings only within next N days/weeks

### F4: Timezone Intelligence
- Auto-detect visitor timezone via browser API
- Show all times in visitor's local timezone
- Timezone selector for manual override
- Display timezone abbreviation (PST, EST, CET, etc.)
- Handle DST transitions correctly

### F5: Meeting Types
- Multiple meeting types per account (30-min intro, 60-min deep dive, 15-min check-in)
- Per-type configuration: duration, buffer, description, price
- Location options per type: Zoom, Google Meet, Teams, Phone, In-person, Custom URL
- Auto-generate meeting links (Zoom/Google Meet integration)
- Secret/hidden meeting types (access via direct link only)
- Group meetings: multiple people book the same slot

### F6: Team Scheduling
- **Round-robin**: distribute bookings evenly across team members
- **Collective**: find time when all specified team members are free
- **Priority routing**: route to specific member based on booking form answers
- **Weighted distribution**: some members get more bookings
- Per-member availability and calendar sync
- Team member profiles (name, photo, bio shown to booker)

### F7: Payment Collection
- Stripe integration: charge at time of booking
- Per-meeting-type pricing
- Deposits: charge partial amount, collect rest later
- Currency support (USD, EUR, GBP, CAD, AUD, etc.)
- Automatic refund on cancellation (configurable)
- Tax/VAT handling
- Invoice generation
- Coupon/discount codes

### F8: Notifications
- **Confirmation email**: to both parties on booking
- **Reminder emails**: configurable (24h, 1h before)
- **SMS reminders**: via Twilio
- **Calendar invites**: .ics attachment
- **Cancellation/reschedule notifications**
- **Custom email templates**: full HTML customization
- **Webhook**: POST booking data to your endpoint on every event

### F9: Booking Management
- Dashboard: upcoming bookings, past bookings, cancellations
- Reschedule and cancel (with configurable policies)
- Customer can self-reschedule via link in confirmation email
- No-show tracking
- Booking notes and tags
- Export bookings (CSV)
- Calendar view of all bookings

### F10: Analytics
- Bookings over time
- Conversion rate: widget views → bookings
- Most popular time slots
- Average booking lead time
- Cancellation and no-show rate
- Revenue tracking (if payments enabled)
- Source tracking (UTM parameters)

---

## Technical Architecture

### Tech Stack

| Layer | Technology |
|-------|-----------|
| Widget | Preact (lightweight React alternative, <20KB) + Shadow DOM for style isolation |
| Dashboard | Next.js 14+, Tailwind CSS |
| Backend | Next.js API Routes, tRPC |
| Database | PostgreSQL (Neon) |
| Calendar | Google Calendar API, Microsoft Graph API |
| Payments | Stripe |
| Email | Resend |
| SMS | Twilio |
| Video | Zoom API, Google Meet API |
| Auth | Clerk |
| CDN | Cloudflare (widget delivery) |
| Hosting | Vercel |

### Data Model

```
Organization
├── id, name, slug, plan
│
├── Member[]
│   ├── id, org_id, user_id, name, email, bio, avatar_url
│   ├── calendar_connections[] (JSON: google, outlook, ical)
│   ├── availability (JSON: { mon: [{start: "09:00", end: "17:00"}], ... })
│   ├── timezone
│   └── is_active
│
├── MeetingType[]
│   ├── id, org_id, name, slug, description
│   ├── duration_minutes, buffer_before, buffer_after
│   ├── location_type (zoom/google_meet/teams/phone/in_person/custom)
│   ├── location_config (JSON)
│   ├── price_cents, currency (nullable for free)
│   ├── scheduling_type (individual/round_robin/collective)
│   ├── assigned_members[] (→ Member)
│   ├── min_notice_hours, max_days_ahead
│   ├── daily_limit
│   ├── custom_questions (JSON[]: [{type, label, required}])
│   ├── confirmation_redirect_url
│   ├── is_secret, is_active
│   └── color (for calendar display)
│
├── Booking[]
│   ├── id, org_id, meeting_type_id
│   ├── assigned_member_id
│   ├── guest_name, guest_email, guest_phone, guest_timezone
│   ├── custom_answers (JSON)
│   ├── start_time, end_time (UTC)
│   ├── status (confirmed/cancelled/rescheduled/no_show/completed)
│   ├── meeting_link (auto-generated)
│   ├── payment_intent_id, amount_paid_cents
│   ├── cancellation_reason
│   ├── source_url (where the widget was embedded)
│   ├── utm_params (JSON)
│   ├── reschedule_token, cancel_token
│   └── created_at
│
├── WidgetConfig[]
│   ├── id, org_id, name
│   ├── meeting_type_ids[] (which types to show)
│   ├── theme (JSON: colors, font, border_radius, layout)
│   ├── custom_css
│   ├── display_mode (inline/popup/slide_in)
│   ├── locale, show_timezone_selector
│   └── embed_code (generated)
│
├── NotificationTemplate[]
│   ├── id, org_id, type (confirmation/reminder_24h/reminder_1h/cancellation)
│   ├── subject, body_html
│   ├── is_sms, sms_body
│   └── is_active
│
└── AvailabilityOverride[]
    ├── id, member_id, date
    ├── type (unavailable/custom_hours)
    └── hours (JSON, nullable)
```

---

## UI/UX — Key Screens

### 1. Widget (Booker-Facing)
- Step 1: Select meeting type (if multiple)
- Step 2: Calendar date picker with available dates highlighted
- Step 3: Time slot selection (available slots in booker's timezone)
- Step 4: Booking form (name, email, custom questions)
- Step 5: Payment (if applicable)
- Step 6: Confirmation with calendar add button and meeting details

### 2. Dashboard
- Today's bookings timeline
- Upcoming bookings list
- Quick stats: bookings this week, revenue, cancellation rate
- Action items: unconfirmed bookings, pending payments

### 3. Meeting Type Configuration
- Name, duration, description
- Availability settings
- Team assignment (round-robin config)
- Custom questions builder
- Payment configuration
- Notification settings
- Preview widget with this meeting type

### 4. Widget Customizer
- Live preview on left, controls on right
- Color pickers, font selector, border radius slider
- Theme presets (one-click apply)
- Custom CSS editor (advanced)
- Display mode toggle (inline/popup/slide-in)
- Copy embed code

### 5. Availability Settings
- Weekly schedule grid (click to toggle time blocks)
- Per-day override calendar
- Connected calendars list
- Buffer and limit settings

---

## Monetization

### Free Tier
- 1 meeting type
- 1 calendar connection
- Basic widget (with "Powered by ScheduleKit" badge)
- Email confirmations
- 20 bookings/month

### Pro — $15/month
- Unlimited meeting types
- Custom branding (remove badge)
- Multiple calendar connections
- Payment collection
- Custom email templates
- SMS reminders
- Analytics
- Unlimited bookings

### Business — $39/month
- Everything in Pro
- Team scheduling (up to 10 members)
- Round-robin and collective
- Multiple widget configurations
- Custom domain for hosted pages
- Webhook/API access
- Priority support

### Enterprise — Custom
- Unlimited members
- SSO
- SLA
- Custom integrations
- Dedicated support

---

## Go-to-Market Strategy

- Position as "Calendly but actually embeddable and brandable"
- Developer-focused launch: great docs, React/Vue components, API
- Content: "How to add scheduling to your website in 5 minutes"
- SEO: "embeddable scheduling widget", "white-label booking system"
- Compare with Calendly on customization and embedding
- Target SaaS companies with "Book a Demo" flows
- Agency partnership program: agencies resell to clients

---

## Success Metrics

| Metric | Target (Month 6) | Target (Month 12) |
|--------|-------------------|---------------------|
| Organizations | 400 | 2,000 |
| Bookings/month | 10,000 | 60,000 |
| Widget impressions | 500K | 3M |
| Paying customers | 100 | 500 |
| MRR | $3,000 | $15,000 |
| Conversion (view → book) | 8% | 12% |

---

## MVP Scope (v1.0)

### In Scope
- Embeddable JavaScript widget (inline + popup modes)
- Google Calendar sync
- 1 meeting type per account
- Timezone detection
- Booking form (name, email, phone)
- Email confirmations and reminders
- Basic customization (colors, logo)
- Dashboard with booking list

### Out of Scope
- React/Vue components
- Team scheduling
- Payments
- SMS
- Custom email templates
- Analytics
- Outlook sync
- Custom questions

### MVP Timeline: 6-7 weeks
- Week 1: Data model, availability engine, Google Calendar sync
- Week 2-3: Widget (Preact, Shadow DOM, slot selection)
- Week 4: Booking flow, email notifications
- Week 5: Dashboard UI, meeting type config, widget customizer
- Week 6: Hosted booking page, embed code generation
- Week 7: Billing, polish, launch
