# MeetingTLDR — Meeting Summarizer

## Executive Summary

MeetingTLDR joins your Zoom, Google Meet, or Teams calls as a bot, records and transcribes the meeting, then generates a structured summary with key decisions, action items (with assignees), and discussion highlights. It auto-shares the summary to Slack, email, or Notion so your team never asks "what did we decide?" again.

---

## Problem Statement

**The pain:**
- People spend 31 hours per month in meetings (Atlassian)
- 73% of professionals admit to doing other work during meetings (HBR)
- Meeting notes are either not taken, incomplete, or never shared
- Action items from meetings are forgotten within 24 hours
- People who miss meetings have no reliable way to catch up
- Recording an entire meeting doesn't help — nobody rewatches an hour-long video

---

## Target Users

### Primary Personas

**1. Jordan — Engineering Manager**
- Has 15+ meetings per week, can't take notes in all of them
- Needs to remember what was decided and who committed to what
- Needs: auto-captured summaries, action items extracted

**2. Priya — Product Manager**
- Runs stakeholder meetings, design reviews, and sprint planning
- Currently takes notes in Google Docs but it's incomplete and inconsistent
- Needs: reliable summary with decisions highlighted

**3. Mark — Account Executive**
- Has 5-8 client calls per day
- Needs to capture client requirements and follow-up tasks
- Needs: call summary with CRM integration, action items

---

## Core Features

### F1: Meeting Bot
- Bot joins Zoom, Google Meet, or Microsoft Teams automatically
- Calendar integration: auto-join all scheduled meetings (or selected ones)
- Manual invite: add bot via meeting link
- Bot identifies itself: "[Company] Notetaker has joined"
- Minimal footprint: bot is audio-only, no video
- Participant consent notification
- Bot behavior configurable: auto-join, ask before joining, manual only

### F2: Real-Time Transcription
- Speaker identification (diarization): "Sarah said... then John responded..."
- Support for 30+ languages with auto-detection
- Punctuation and formatting
- Technical term handling (configurable glossary)
- Real-time streaming transcript (viewable during meeting)
- Filler word removal (um, uh, like)
- Confidence scores on transcription accuracy
- Manual transcript editing post-meeting

### F3: AI Summary Generation
- Generated immediately after meeting ends (within 2-3 minutes)
- Summary sections:
  - **TL;DR**: 2-3 sentence overview of the meeting
  - **Key Decisions**: bullet points of decisions made
  - **Action Items**: tasks with detected assignees and deadlines
  - **Discussion Topics**: major topics covered with brief summary of each
  - **Open Questions**: unresolved items for follow-up
  - **Sentiment/Vibe**: overall meeting tone (productive, contentious, brainstorming)
- Customizable summary templates per meeting type:
  - Standup, sprint planning, 1-on-1, client call, all-hands, brainstorm
- Regenerate summary with different focus or length
- Summary editing (human can refine AI output)

### F4: Action Item Extraction
- AI identifies action items from conversation context
- Assigns owner based on who committed ("I'll handle that" → assigned to speaker)
- Extracts deadlines when mentioned ("by Friday" → due date)
- Action items presented as checklist
- Push to task manager: Asana, Linear, Jira, Todoist, Notion
- Follow-up reminders for incomplete action items
- Track action item completion across meetings

### F5: Topic Segmentation
- Break transcript into topic segments
- Each segment: title, start time, end time, summary
- Click topic to jump to that part of transcript/recording
- Topic-level search: "Find where we discussed the pricing change"
- Table of contents for long meetings

### F6: Searchable Archive
- Full-text search across all meeting transcripts
- Search by: speaker, date range, meeting type, keyword
- "What did we decide about X?" → AI answer with transcript citation
- Meeting history timeline
- Bookmark important moments
- Tags for organizing meetings

### F7: Auto-Sharing
- Share summary via:
  - **Slack**: post to meeting channel or specified channel
  - **Email**: send to all participants or custom recipients
  - **Notion**: create page in specified database
  - **Google Docs**: create document in specified folder
  - **Confluence**: create page
- Share immediately after meeting or on schedule (e.g., within 1 hour)
- Configurable per meeting type
- Attendees get a link to full transcript + summary

### F8: Meeting Analytics
- Personal metrics: meeting hours per week, trending up/down
- Team metrics: most meeting-heavy people, meeting distribution
- Talk time per participant: who dominates, who's quiet
- Meeting efficiency score: decisions per meeting, action items per meeting
- Meeting cost calculator (estimated hourly rates × attendees × duration)
- "Meeting-free time" tracking
- Recurring meeting value assessment

### F9: Integration Hub
- **Calendar**: Google Calendar, Outlook (for auto-join scheduling)
- **Video**: Zoom, Google Meet, Microsoft Teams
- **Task Managers**: Asana, Linear, Jira, Todoist
- **Knowledge Base**: Notion, Confluence, Google Docs
- **Communication**: Slack, Microsoft Teams, Email
- **CRM**: Salesforce, HubSpot (for sales call notes)
- **Webhook**: POST summary to any endpoint

---

## Technical Architecture

### Tech Stack

| Layer | Technology |
|-------|-----------|
| Frontend | Next.js 14+, Tailwind CSS, Radix UI |
| Backend | Next.js API Routes, tRPC |
| Database | PostgreSQL (Neon) |
| Bot / Recording | Custom bot service (Go), Recall.ai or Daily.co for meeting joining |
| Transcription | Deepgram (real-time streaming), Whisper (batch fallback) |
| AI | OpenAI GPT-4o (summarization, action item extraction) |
| Audio Storage | AWS S3 / Cloudflare R2 |
| Search | pgvector (semantic) + full-text (pg_trgm) |
| Queue | BullMQ + Redis |
| Auth | Clerk |
| Hosting | Vercel (app), AWS ECS (bot workers) |

### Data Model

```
Organization
├── id, name, slug, plan
│
├── CalendarConnection[]
│   ├── id, org_id, user_id
│   ├── type (google/outlook)
│   ├── access_token (encrypted), refresh_token
│   └── auto_join_settings (JSON: all/selected/manual)
│
├── Meeting[]
│   ├── id, org_id, title
│   ├── platform (zoom/google_meet/teams)
│   ├── meeting_url, external_meeting_id
│   ├── started_at, ended_at, duration_minutes
│   ├── participants (JSON[]: [{name, email, talk_time_seconds}])
│   ├── recording_url, recording_size_bytes
│   ├── status (scheduled/recording/processing/ready/failed)
│   ├── meeting_type (standup/planning/1on1/client/allhands/other)
│   ├── calendar_event_id
│   │
│   ├── Transcript
│   │   ├── id, meeting_id
│   │   ├── segments (JSON[]: [{speaker, text, start_ms, end_ms, confidence}])
│   │   ├── full_text (searchable)
│   │   ├── language
│   │   ├── word_count
│   │   └── embedding (vector[1536])
│   │
│   ├── Summary
│   │   ├── id, meeting_id
│   │   ├── tldr, decisions (text[])
│   │   ├── discussion_topics (JSON[]: [{title, summary, start_ms, end_ms}])
│   │   ├── open_questions (text[])
│   │   ├── sentiment
│   │   ├── template_used
│   │   ├── is_edited, edited_at
│   │   └── generated_at
│   │
│   ├── ActionItem[]
│   │   ├── id, meeting_id, text
│   │   ├── assignee_name, assignee_email
│   │   ├── due_date, status (pending/done)
│   │   ├── external_task_id, external_task_url
│   │   └── source_transcript_offset_ms
│   │
│   ├── Bookmark[]
│   │   ├── id, meeting_id, user_id
│   │   ├── timestamp_ms, note
│   │   └── created_at
│   │
│   └── Share[]
│       ├── id, meeting_id, channel (slack/email/notion/docs)
│       ├── config (JSON), sent_at, status
│       └── recipient
│
└── Glossary[]
    ├── id, org_id, term, pronunciation_hint
    └── description
```

---

## UI/UX — Key Screens

### 1. Meeting List
- Upcoming meetings (from calendar) with auto-join toggle
- Past meetings with status: summary ready, processing, failed
- Search bar across all meetings
- Filter by date, type, participant
- Click meeting to view summary

### 2. Meeting Summary View
- Header: title, date, duration, participants (avatars)
- TL;DR section (prominent)
- Decisions list (highlighted)
- Action items checklist (with assignees)
- Topic segments (expandable, click to jump to transcript)
- Open questions
- Share button, export button

### 3. Transcript View
- Scrollable transcript with speaker labels
- Click-to-play audio at any point
- Highlight/bookmark capability
- Search within transcript
- Topic markers in scroll bar
- Speaker filter (show only Sarah's contributions)

### 4. Action Item Tracker
- Cross-meeting action item list
- Filter by assignee, status, meeting
- Overdue items highlighted
- Quick mark as done
- Push to Jira/Linear button

### 5. Analytics Dashboard
- Meeting hours this week/month chart
- Average meeting duration trend
- Talk time distribution per meeting
- Action items created vs completed
- Meeting cost estimation
- Most common meeting topics

### 6. Settings
- Calendar connections
- Auto-join preferences (all meetings, specific calendars, manual)
- Default summary template
- Sharing defaults (auto-share to Slack after every meeting)
- Glossary management
- Integration connections

---

## Monetization

### Free Tier
- 5 meetings per month
- 30-minute meeting limit
- AI summary + action items
- 7-day transcript retention
- Email sharing

### Pro — $18/month
- Unlimited meetings
- 2-hour meeting limit
- Full transcription with speaker identification
- Searchable archive (unlimited retention)
- Topic segmentation
- Slack + Notion sharing
- Meeting analytics

### Team — $12/user/month (min 3)
- Everything in Pro
- 4-hour meeting limit
- Action item tracking across meetings
- All integrations (Jira, Linear, Salesforce)
- Team analytics
- Custom summary templates
- Glossary
- Priority support

### Enterprise — Custom
- Unlimited meeting duration
- SSO/SAML
- Data residency options
- Custom integrations
- SLA
- Dedicated support
- Admin controls

---

## Go-to-Market Strategy

- Launch targeting remote teams and meeting-heavy roles (PMs, managers, sales)
- Viral loop: bot joins meetings with "Powered by MeetingTLDR" in participant name
- Content: "I got my life back by automating meeting notes"
- SEO: "AI meeting notes", "meeting summarizer", "meeting transcription"
- Free tier generous enough for individual adoption → team spread
- Calendar integration as primary onboarding hook
- Partnership with remote work tools and communities

---

## Success Metrics

| Metric | Target (Month 6) | Target (Month 12) |
|--------|-------------------|---------------------|
| Users | 3,000 | 15,000 |
| Meetings processed/month | 15,000 | 80,000 |
| Action items extracted | 30,000/mo | 160,000/mo |
| Paying users | 300 | 1,500 |
| MRR | $6,000 | $30,000 |
| Summary satisfaction rate | 80% | 90% |

---

## MVP Scope (v1.0)

### In Scope
- Google Meet bot (join via link)
- Google Calendar integration (auto-detect meetings)
- Real-time transcription with speaker identification
- AI summary generation (TL;DR, decisions, action items)
- Meeting archive with search
- Email + Slack sharing
- Web dashboard

### Out of Scope
- Zoom and Teams bots
- Topic segmentation
- Action item tracking across meetings
- Meeting analytics
- CRM integration
- Custom summary templates
- Glossary

### MVP Timeline: 7-8 weeks
- Week 1-2: Meeting bot service (Google Meet joining, audio capture)
- Week 3: Transcription pipeline (Deepgram), speaker diarization
- Week 4-5: AI summarization, action item extraction
- Week 6: Dashboard UI (meeting list, summary view, transcript)
- Week 7: Calendar integration, auto-sharing (email + Slack)
- Week 8: Billing, onboarding, polish, launch
