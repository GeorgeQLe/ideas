# 18. MeetingTLDR — Meeting Summarizer

## Implementation Plan

**MVP Scope:** Google Calendar integration for auto-detecting meetings, Google Meet bot that joins via meeting URL and captures audio, real-time transcription with speaker diarization via Deepgram, AI summary generation with GPT-4o producing TL;DR + key decisions + action items with detected assignees and deadlines, meeting archive with full-text search, web dashboard for viewing meeting lists + summaries + transcripts + action items, email sharing of summaries via Resend, Slack integration for auto-posting summaries to channels, Stripe billing with three tiers (Free / Pro $18/mo / Team $12/user/mo).

---

## Tech Decisions

| Layer | Technology | Notes |
|-------|-----------|-------|
| Framework | Next.js 14+ App Router | `src/` directory, TypeScript strict |
| API | tRPC v11 | superjson transformer, Clerk auth context |
| Database | Neon PostgreSQL | Meetings, transcripts, summaries, action items |
| ORM | Drizzle ORM | Push-based migrations, typed schema |
| Auth | Clerk | Organization-scoped, `orgId` from session |
| Billing | Stripe | Checkout Sessions + Customer Portal + Webhooks |
| Meeting Bot | Recall.ai SDK | Join Google Meet, capture audio stream |
| Transcription | Deepgram Nova-2 | Real-time streaming, speaker diarization |
| AI — Summarization | OpenAI GPT-4o | Structured summary + action item extraction |
| AI — Embeddings | text-embedding-3-small | Semantic search over transcripts |
| Audio Storage | Cloudflare R2 | Meeting recordings |
| Search | pgvector + pg_trgm | Hybrid semantic + full-text transcript search |
| Queue | BullMQ + Redis (Upstash) | Async transcription + summarization pipeline |
| Calendar | Google Calendar API v3 | OAuth 2.0, watch notifications for events |
| Slack | Slack Web API (OAuth 2.0) | Post summaries to channels |
| Email | Resend | Summary sharing, weekly digest |
| UI | Tailwind CSS, Radix UI, Recharts | |
| Hosting | Vercel (app), Railway (bot worker) | Bot needs persistent connections |
| Monitoring | Sentry, Vercel Analytics | |

### Key Architecture Decisions

1. **Recall.ai for meeting bot instead of custom bot**: Building a custom Google Meet bot requires managing headless Chrome instances, WebRTC connections, and audio capture. Recall.ai abstracts all of this into an API: `POST /bot` with a meeting URL, and it returns an audio stream + webhook on meeting end. This saves weeks of bot infrastructure work and handles Google's anti-bot measures.

2. **Deepgram Nova-2 over Whisper for transcription**: Deepgram provides real-time streaming transcription with speaker diarization built-in, returning results as utterances happen. Whisper would require batch processing after the meeting ends (3-5 minute delay for a 60-minute meeting). Deepgram's streaming approach enables future real-time features and delivers summaries within 1-2 minutes of meeting end.

3. **pgvector + pg_trgm hybrid search**: Meeting archive search uses both semantic (pgvector cosine similarity on transcript embeddings) and keyword (pg_trgm trigram matching on full_text) approaches, merged via Reciprocal Rank Fusion. This handles queries like "what did we decide about pricing?" (semantic) and exact name/term searches (keyword) equally well.

4. **Railway for bot worker**: The meeting bot service runs on Railway as a long-lived worker process (not serverless) because it maintains persistent WebSocket connections to Recall.ai during meetings. The Vercel-hosted app communicates with it via BullMQ queues. The worker process monitors meeting start events from Google Calendar and dispatches bots to join at the right time.

---

## Database Schema (Drizzle)

```typescript
// src/server/db/schema.ts

import { relations, sql } from "drizzle-orm";
import {
  pgTable,
  uuid,
  text,
  boolean,
  integer,
  timestamp,
  jsonb,
  real,
  index,
  uniqueIndex,
} from "drizzle-orm/pg-core";

// ---------------------------------------------------------------------------
// Organizations
// ---------------------------------------------------------------------------
export const organizations = pgTable("organizations", {
  id: uuid("id").primaryKey().defaultRandom(),
  clerkOrgId: text("clerk_org_id").unique().notNull(),
  name: text("name").notNull(),
  slug: text("slug").unique().notNull(),
  plan: text("plan").default("free").notNull(), // free | pro | team
  stripeCustomerId: text("stripe_customer_id"),
  stripeSubscriptionId: text("stripe_subscription_id"),
  settings: jsonb("settings").default({}).$type<{
    onboardingCompleted?: boolean;
    defaultSummaryTemplate?: string; // standup | planning | 1on1 | client | allhands | general
    autoShareConfig?: {
      slackEnabled?: boolean;
      slackChannelId?: string;
      emailEnabled?: boolean;
      emailRecipients?: string[];
    };
    timezone?: string;
  }>(),
  createdAt: timestamp("created_at", { withTimezone: true }).defaultNow().notNull(),
  updatedAt: timestamp("updated_at", { withTimezone: true }).defaultNow().notNull(),
});

// ---------------------------------------------------------------------------
// Members
// ---------------------------------------------------------------------------
export const members = pgTable(
  "members",
  {
    id: uuid("id").primaryKey().defaultRandom(),
    orgId: uuid("org_id")
      .references(() => organizations.id, { onDelete: "cascade" })
      .notNull(),
    clerkUserId: text("clerk_user_id").notNull(),
    role: text("role").default("member").notNull(), // owner | admin | member
    email: text("email").notNull(),
    name: text("name"),
    autoJoinPreference: text("auto_join_preference").default("all").notNull(),
    // all | selected | manual
    createdAt: timestamp("created_at", { withTimezone: true }).defaultNow().notNull(),
  },
  (table) => [
    uniqueIndex("members_org_user_idx").on(table.orgId, table.clerkUserId),
  ],
);

// ---------------------------------------------------------------------------
// Calendar Connections
// ---------------------------------------------------------------------------
export const calendarConnections = pgTable(
  "calendar_connections",
  {
    id: uuid("id").primaryKey().defaultRandom(),
    orgId: uuid("org_id")
      .references(() => organizations.id, { onDelete: "cascade" })
      .notNull(),
    memberId: uuid("member_id")
      .references(() => members.id, { onDelete: "cascade" })
      .notNull(),
    provider: text("provider").default("google").notNull(), // google | outlook (v2)
    providerEmail: text("provider_email").notNull(), // Google account email
    accessTokenEncrypted: text("access_token_encrypted").notNull(),
    refreshTokenEncrypted: text("refresh_token_encrypted"),
    tokenExpiresAt: timestamp("token_expires_at", { withTimezone: true }),
    watchChannelId: text("watch_channel_id"), // Google Calendar push notification channel
    watchResourceId: text("watch_resource_id"),
    watchExpiration: timestamp("watch_expiration", { withTimezone: true }),
    autoJoinSettings: jsonb("auto_join_settings").default({}).$type<{
      mode: "all" | "selected" | "manual";
      selectedCalendarIds?: string[];
      excludePatterns?: string[]; // e.g., ["1:1", "standup"] — meeting titles to skip
    }>(),
    status: text("status").default("connected").notNull(), // connected | expired | error
    errorMessage: text("error_message"),
    lastSyncedAt: timestamp("last_synced_at", { withTimezone: true }),
    createdAt: timestamp("created_at", { withTimezone: true }).defaultNow().notNull(),
  },
  (table) => [
    index("calendar_connections_org_idx").on(table.orgId),
    index("calendar_connections_member_idx").on(table.memberId),
  ],
);

// ---------------------------------------------------------------------------
// Meetings
// ---------------------------------------------------------------------------
export const meetings = pgTable(
  "meetings",
  {
    id: uuid("id").primaryKey().defaultRandom(),
    orgId: uuid("org_id")
      .references(() => organizations.id, { onDelete: "cascade" })
      .notNull(),
    title: text("title").notNull(),
    platform: text("platform").default("google_meet").notNull(),
    // google_meet | zoom | teams (v2)
    meetingUrl: text("meeting_url"),
    externalMeetingId: text("external_meeting_id"), // Google Calendar event ID
    calendarConnectionId: uuid("calendar_connection_id")
      .references(() => calendarConnections.id, { onDelete: "set null" }),
    startedAt: timestamp("started_at", { withTimezone: true }),
    endedAt: timestamp("ended_at", { withTimezone: true }),
    durationMinutes: integer("duration_minutes"),
    participants: jsonb("participants").default([]).$type<MeetingParticipant[]>(),
    recordingUrl: text("recording_url"), // R2 URL
    recordingR2Key: text("recording_r2_key"),
    recordingSizeBytes: integer("recording_size_bytes"),
    status: text("status").default("scheduled").notNull(),
    // scheduled | joining | recording | processing | ready | failed
    meetingType: text("meeting_type").default("general").notNull(),
    // standup | planning | 1on1 | client | allhands | brainstorm | general
    recallBotId: text("recall_bot_id"), // Recall.ai bot ID
    processingStartedAt: timestamp("processing_started_at", { withTimezone: true }),
    errorMessage: text("error_message"),
    tags: jsonb("tags").default([]).$type<string[]>(),
    createdAt: timestamp("created_at", { withTimezone: true }).defaultNow().notNull(),
    updatedAt: timestamp("updated_at", { withTimezone: true }).defaultNow().notNull(),
  },
  (table) => [
    index("meetings_org_idx").on(table.orgId),
    index("meetings_status_idx").on(table.orgId, table.status),
    index("meetings_started_at_idx").on(table.orgId, table.startedAt),
    index("meetings_external_id_idx").on(table.externalMeetingId),
  ],
);

export type MeetingParticipant = {
  name: string;
  email?: string;
  talkTimeSeconds: number;
  joinedAt?: string; // ISO
  leftAt?: string; // ISO
};

// ---------------------------------------------------------------------------
// Transcripts
// ---------------------------------------------------------------------------
export const transcripts = pgTable(
  "transcripts",
  {
    id: uuid("id").primaryKey().defaultRandom(),
    meetingId: uuid("meeting_id")
      .references(() => meetings.id, { onDelete: "cascade" })
      .notNull(),
    segments: jsonb("segments").default([]).$type<TranscriptSegment[]>(),
    fullText: text("full_text"), // Concatenated text for full-text search
    language: text("language").default("en"),
    wordCount: integer("word_count").default(0).notNull(),
    confidenceAvg: real("confidence_avg"), // Average transcription confidence 0-1
    // embedding vector(1536) — added via raw SQL migration
    createdAt: timestamp("created_at", { withTimezone: true }).defaultNow().notNull(),
    updatedAt: timestamp("updated_at", { withTimezone: true }).defaultNow().notNull(),
  },
  (table) => [
    index("transcripts_meeting_idx").on(table.meetingId),
  ],
);

export type TranscriptSegment = {
  speaker: string; // Speaker name or "Speaker 1"
  text: string;
  startMs: number;
  endMs: number;
  confidence: number; // 0-1
  words?: { word: string; startMs: number; endMs: number; confidence: number }[];
};

// ---------------------------------------------------------------------------
// Summaries
// ---------------------------------------------------------------------------
export const summaries = pgTable(
  "summaries",
  {
    id: uuid("id").primaryKey().defaultRandom(),
    meetingId: uuid("meeting_id")
      .references(() => meetings.id, { onDelete: "cascade" })
      .notNull(),
    tldr: text("tldr").notNull(), // 2-3 sentence overview
    decisions: jsonb("decisions").default([]).$type<string[]>(),
    discussionTopics: jsonb("discussion_topics").default([]).$type<DiscussionTopic[]>(),
    openQuestions: jsonb("open_questions").default([]).$type<string[]>(),
    sentiment: text("sentiment"), // productive | contentious | brainstorming | informational | mixed
    templateUsed: text("template_used").default("general"),
    isEdited: boolean("is_edited").default(false).notNull(),
    editedAt: timestamp("edited_at", { withTimezone: true }),
    editedByUserId: text("edited_by_user_id"),
    generatedAt: timestamp("generated_at", { withTimezone: true }).defaultNow().notNull(),
    regenerationCount: integer("regeneration_count").default(0).notNull(),
    createdAt: timestamp("created_at", { withTimezone: true }).defaultNow().notNull(),
  },
  (table) => [
    index("summaries_meeting_idx").on(table.meetingId),
  ],
);

export type DiscussionTopic = {
  title: string;
  summary: string;
  startMs: number;
  endMs: number;
};

// ---------------------------------------------------------------------------
// Action Items
// ---------------------------------------------------------------------------
export const actionItems = pgTable(
  "action_items",
  {
    id: uuid("id").primaryKey().defaultRandom(),
    meetingId: uuid("meeting_id")
      .references(() => meetings.id, { onDelete: "cascade" })
      .notNull(),
    orgId: uuid("org_id")
      .references(() => organizations.id, { onDelete: "cascade" })
      .notNull(),
    text: text("text").notNull(),
    assigneeName: text("assignee_name"), // Detected from "I'll handle that"
    assigneeEmail: text("assignee_email"),
    dueDate: timestamp("due_date", { withTimezone: true }),
    dueDateRaw: text("due_date_raw"), // "by Friday", "next week" — original text
    status: text("status").default("pending").notNull(), // pending | done | dismissed
    priority: text("priority").default("normal"), // low | normal | high
    sourceTranscriptOffsetMs: integer("source_transcript_offset_ms"),
    // Timestamp in transcript where this was detected
    externalTaskId: text("external_task_id"), // Jira/Linear task ID (v2)
    externalTaskUrl: text("external_task_url"),
    completedAt: timestamp("completed_at", { withTimezone: true }),
    createdAt: timestamp("created_at", { withTimezone: true }).defaultNow().notNull(),
  },
  (table) => [
    index("action_items_meeting_idx").on(table.meetingId),
    index("action_items_org_idx").on(table.orgId),
    index("action_items_status_idx").on(table.orgId, table.status),
    index("action_items_assignee_idx").on(table.orgId, table.assigneeEmail),
  ],
);

// ---------------------------------------------------------------------------
// Bookmarks
// ---------------------------------------------------------------------------
export const bookmarks = pgTable(
  "bookmarks",
  {
    id: uuid("id").primaryKey().defaultRandom(),
    meetingId: uuid("meeting_id")
      .references(() => meetings.id, { onDelete: "cascade" })
      .notNull(),
    userId: text("user_id").notNull(), // Clerk user ID
    timestampMs: integer("timestamp_ms").notNull(), // Point in transcript
    note: text("note"),
    createdAt: timestamp("created_at", { withTimezone: true }).defaultNow().notNull(),
  },
  (table) => [
    index("bookmarks_meeting_idx").on(table.meetingId),
    index("bookmarks_user_idx").on(table.userId),
  ],
);

// ---------------------------------------------------------------------------
// Shares (summary distribution log)
// ---------------------------------------------------------------------------
export const shares = pgTable(
  "shares",
  {
    id: uuid("id").primaryKey().defaultRandom(),
    meetingId: uuid("meeting_id")
      .references(() => meetings.id, { onDelete: "cascade" })
      .notNull(),
    orgId: uuid("org_id")
      .references(() => organizations.id, { onDelete: "cascade" })
      .notNull(),
    channel: text("channel").notNull(), // email | slack | notion (v2) | webhook (v2)
    recipient: text("recipient"), // Email address, Slack channel ID
    config: jsonb("config").default({}).$type<{
      slackChannelId?: string;
      slackChannelName?: string;
      emailAddresses?: string[];
      webhookUrl?: string;
      includeSections?: string[]; // ["tldr", "decisions", "action_items"]
    }>(),
    status: text("status").default("pending").notNull(), // pending | sent | failed
    sentAt: timestamp("sent_at", { withTimezone: true }),
    errorMessage: text("error_message"),
    metadata: jsonb("metadata").default({}).$type<{
      resendMessageId?: string;
      slackMessageTs?: string;
    }>(),
    createdAt: timestamp("created_at", { withTimezone: true }).defaultNow().notNull(),
  },
  (table) => [
    index("shares_meeting_idx").on(table.meetingId),
    index("shares_org_idx").on(table.orgId),
    index("shares_status_idx").on(table.status),
  ],
);

// ---------------------------------------------------------------------------
// Slack Connections
// ---------------------------------------------------------------------------
export const slackConnections = pgTable(
  "slack_connections",
  {
    id: uuid("id").primaryKey().defaultRandom(),
    orgId: uuid("org_id")
      .references(() => organizations.id, { onDelete: "cascade" })
      .notNull(),
    teamId: text("team_id").notNull(), // Slack workspace ID
    teamName: text("team_name").notNull(),
    accessTokenEncrypted: text("access_token_encrypted").notNull(),
    botUserId: text("bot_user_id"), // Slack bot user ID
    defaultChannelId: text("default_channel_id"),
    defaultChannelName: text("default_channel_name"),
    status: text("status").default("connected").notNull(), // connected | expired | error
    createdAt: timestamp("created_at", { withTimezone: true }).defaultNow().notNull(),
  },
  (table) => [
    index("slack_connections_org_idx").on(table.orgId),
    uniqueIndex("slack_connections_team_idx").on(table.orgId, table.teamId),
  ],
);

// ---------------------------------------------------------------------------
// Glossary (custom terms for better transcription)
// ---------------------------------------------------------------------------
export const glossary = pgTable(
  "glossary",
  {
    id: uuid("id").primaryKey().defaultRandom(),
    orgId: uuid("org_id")
      .references(() => organizations.id, { onDelete: "cascade" })
      .notNull(),
    term: text("term").notNull(),
    pronunciationHint: text("pronunciation_hint"),
    description: text("description"),
    createdAt: timestamp("created_at", { withTimezone: true }).defaultNow().notNull(),
  },
  (table) => [
    index("glossary_org_idx").on(table.orgId),
    uniqueIndex("glossary_org_term_idx").on(table.orgId, table.term),
  ],
);

// ---------------------------------------------------------------------------
// Relations
// ---------------------------------------------------------------------------
export const organizationsRelations = relations(organizations, ({ many }) => ({
  members: many(members),
  calendarConnections: many(calendarConnections),
  meetings: many(meetings),
  actionItems: many(actionItems),
  shares: many(shares),
  slackConnections: many(slackConnections),
  glossary: many(glossary),
}));

export const membersRelations = relations(members, ({ one, many }) => ({
  organization: one(organizations, {
    fields: [members.orgId],
    references: [organizations.id],
  }),
  calendarConnections: many(calendarConnections),
}));

export const calendarConnectionsRelations = relations(calendarConnections, ({ one, many }) => ({
  organization: one(organizations, {
    fields: [calendarConnections.orgId],
    references: [organizations.id],
  }),
  member: one(members, {
    fields: [calendarConnections.memberId],
    references: [members.id],
  }),
  meetings: many(meetings),
}));

export const meetingsRelations = relations(meetings, ({ one, many }) => ({
  organization: one(organizations, {
    fields: [meetings.orgId],
    references: [organizations.id],
  }),
  calendarConnection: one(calendarConnections, {
    fields: [meetings.calendarConnectionId],
    references: [calendarConnections.id],
  }),
  transcript: one(transcripts),
  summary: one(summaries),
  actionItems: many(actionItems),
  bookmarks: many(bookmarks),
  shares: many(shares),
}));

export const transcriptsRelations = relations(transcripts, ({ one }) => ({
  meeting: one(meetings, {
    fields: [transcripts.meetingId],
    references: [meetings.id],
  }),
}));

export const summariesRelations = relations(summaries, ({ one }) => ({
  meeting: one(meetings, {
    fields: [summaries.meetingId],
    references: [meetings.id],
  }),
}));

export const actionItemsRelations = relations(actionItems, ({ one }) => ({
  meeting: one(meetings, {
    fields: [actionItems.meetingId],
    references: [meetings.id],
  }),
  organization: one(organizations, {
    fields: [actionItems.orgId],
    references: [organizations.id],
  }),
}));

export const bookmarksRelations = relations(bookmarks, ({ one }) => ({
  meeting: one(meetings, {
    fields: [bookmarks.meetingId],
    references: [meetings.id],
  }),
}));

export const sharesRelations = relations(shares, ({ one }) => ({
  meeting: one(meetings, {
    fields: [shares.meetingId],
    references: [meetings.id],
  }),
  organization: one(organizations, {
    fields: [shares.orgId],
    references: [organizations.id],
  }),
}));

export const slackConnectionsRelations = relations(slackConnections, ({ one }) => ({
  organization: one(organizations, {
    fields: [slackConnections.orgId],
    references: [organizations.id],
  }),
}));

export const glossaryRelations = relations(glossary, ({ one }) => ({
  organization: one(organizations, {
    fields: [glossary.orgId],
    references: [organizations.id],
  }),
}));
```

---

## Initial Migration

```sql
-- migrations/0000_init.sql

-- Enable pgvector for semantic search
CREATE EXTENSION IF NOT EXISTS vector;

-- Enable pg_trgm for fuzzy text search
CREATE EXTENSION IF NOT EXISTS pg_trgm;

-- Add embedding column to transcripts (not supported in Drizzle schema)
ALTER TABLE transcripts ADD COLUMN embedding vector(1536);

-- Create HNSW index for fast similarity search
CREATE INDEX transcripts_embedding_idx ON transcripts
  USING hnsw (embedding vector_cosine_ops)
  WITH (m = 16, ef_construction = 64);

-- Create GIN trigram index for full-text search
CREATE INDEX transcripts_fulltext_idx ON transcripts
  USING gin (full_text gin_trgm_ops);
```

---

## Architecture Deep-Dives

### 1. Meeting Bot Lifecycle with Recall.ai

The meeting bot lifecycle manages the full flow from calendar event detection to bot dispatch, audio capture, and meeting end handling. It coordinates between Google Calendar events, Recall.ai's bot API, and the internal processing pipeline.

```typescript
// src/server/services/meeting-bot.ts

import { db } from "../db";
import { meetings, calendarConnections } from "../db/schema";
import { eq } from "drizzle-orm";
import { decrypt } from "../integrations/encryption";

const RECALL_API_BASE = "https://api.recall.ai/api/v1";
const RECALL_API_KEY = process.env.RECALL_API_KEY!;

interface RecallBotResponse {
  id: string;
  status: string;
  meeting_url: string;
  join_at: string;
}

export async function dispatchBotToMeeting(meetingId: string): Promise<string> {
  const [meeting] = await db
    .select()
    .from(meetings)
    .where(eq(meetings.id, meetingId))
    .limit(1);

  if (!meeting) throw new Error(`Meeting ${meetingId} not found`);
  if (!meeting.meetingUrl) throw new Error("No meeting URL available");

  // Create Recall.ai bot
  const response = await fetch(`${RECALL_API_BASE}/bot/`, {
    method: "POST",
    headers: {
      Authorization: `Token ${RECALL_API_KEY}`,
      "Content-Type": "application/json",
    },
    body: JSON.stringify({
      meeting_url: meeting.meetingUrl,
      bot_name: "MeetingTLDR Notetaker",
      join_at: meeting.startedAt?.toISOString() || new Date().toISOString(),
      automatic_leave: {
        waiting_room_timeout: 600, // Leave after 10 min in waiting room
        noone_joined_timeout: 300, // Leave after 5 min if no one else
        everyone_left_timeout: 30, // Leave 30s after last person
      },
      real_time_transcription: {
        destination_url: `${process.env.NEXT_PUBLIC_APP_URL}/api/webhooks/recall/transcription`,
        partial_results: false,
      },
      transcription_options: {
        provider: "deepgram",
        language: "en",
      },
      recording_mode: "speaker_view",
      chat: {
        on_bot_join: {
          send_to: "everyone",
          message: "MeetingTLDR is recording this meeting. A summary will be shared after the meeting ends.",
        },
      },
    }),
  });

  if (!response.ok) {
    const error = await response.text();
    throw new Error(`Recall.ai API error: ${response.status} — ${error}`);
  }

  const bot: RecallBotResponse = await response.json();

  // Update meeting with bot ID
  await db
    .update(meetings)
    .set({
      recallBotId: bot.id,
      status: "joining",
      updatedAt: new Date(),
    })
    .where(eq(meetings.id, meetingId));

  return bot.id;
}

export async function handleBotStatusChange(
  recallBotId: string,
  status: string,
  data: Record<string, unknown>,
): Promise<void> {
  const [meeting] = await db
    .select()
    .from(meetings)
    .where(eq(meetings.recallBotId, recallBotId))
    .limit(1);

  if (!meeting) return;

  switch (status) {
    case "in_call_recording": {
      await db
        .update(meetings)
        .set({
          status: "recording",
          startedAt: meeting.startedAt || new Date(),
          updatedAt: new Date(),
        })
        .where(eq(meetings.id, meeting.id));
      break;
    }

    case "call_ended": {
      const endedAt = new Date();
      const durationMinutes = meeting.startedAt
        ? Math.round((endedAt.getTime() - meeting.startedAt.getTime()) / 60000)
        : 0;

      await db
        .update(meetings)
        .set({
          status: "processing",
          endedAt,
          durationMinutes,
          processingStartedAt: new Date(),
          updatedAt: new Date(),
        })
        .where(eq(meetings.id, meeting.id));

      // Fetch final recording from Recall.ai
      const recordingUrl = data.recording_url as string | undefined;
      if (recordingUrl) {
        await downloadAndStoreRecording(meeting.id, meeting.orgId, recordingUrl);
      }

      // Fetch final transcript from Recall.ai
      const transcriptData = await fetchFinalTranscript(recallBotId);

      // Enqueue processing pipeline
      const { processingQueue } = await import("../queue/processing-queue");
      await processingQueue.add("process-meeting", {
        meetingId: meeting.id,
        orgId: meeting.orgId,
        transcriptData,
      });

      break;
    }

    case "fatal": {
      await db
        .update(meetings)
        .set({
          status: "failed",
          errorMessage: (data.error as string) || "Bot encountered a fatal error",
          updatedAt: new Date(),
        })
        .where(eq(meetings.id, meeting.id));
      break;
    }
  }
}

async function fetchFinalTranscript(
  recallBotId: string,
): Promise<RecallTranscriptEntry[]> {
  const response = await fetch(
    `${RECALL_API_BASE}/bot/${recallBotId}/transcript/`,
    {
      headers: { Authorization: `Token ${RECALL_API_KEY}` },
    },
  );

  if (!response.ok) throw new Error("Failed to fetch transcript from Recall.ai");

  return response.json();
}

type RecallTranscriptEntry = {
  speaker: string;
  words: {
    text: string;
    start_time: number;
    end_time: number;
    confidence: number;
  }[];
};

async function downloadAndStoreRecording(
  meetingId: string,
  orgId: string,
  recordingUrl: string,
): Promise<void> {
  // Download recording from Recall.ai
  const response = await fetch(recordingUrl);
  if (!response.ok) return;

  const audioBuffer = await response.arrayBuffer();

  // Upload to R2
  const { S3Client, PutObjectCommand } = await import("@aws-sdk/client-s3");
  const s3 = new S3Client({
    region: "auto",
    endpoint: process.env.R2_ENDPOINT!,
    credentials: {
      accessKeyId: process.env.R2_ACCESS_KEY_ID!,
      secretAccessKey: process.env.R2_SECRET_ACCESS_KEY!,
    },
  });

  const r2Key = `recordings/${orgId}/${meetingId}/audio.webm`;
  await s3.send(
    new PutObjectCommand({
      Bucket: process.env.R2_BUCKET_NAME!,
      Key: r2Key,
      Body: new Uint8Array(audioBuffer),
      ContentType: "audio/webm",
    }),
  );

  await db
    .update(meetings)
    .set({
      recordingR2Key: r2Key,
      recordingUrl: `${process.env.R2_PUBLIC_URL}/${r2Key}`,
      recordingSizeBytes: audioBuffer.byteLength,
      updatedAt: new Date(),
    })
    .where(eq(meetings.id, meetingId));
}
```

### 2. AI Summarization + Action Item Extraction Pipeline

The processing pipeline takes a raw transcript and produces a structured summary with TL;DR, decisions, discussion topics, action items with assignees and deadlines, and sentiment analysis. It uses a two-pass GPT-4o approach: first extracting structured data, then generating the narrative summary.

```typescript
// src/server/ai/summarize-meeting.ts

import OpenAI from "openai";
import { db } from "../db";
import {
  meetings,
  transcripts,
  summaries,
  actionItems,
  type TranscriptSegment,
  type DiscussionTopic,
} from "../db/schema";
import { eq } from "drizzle-orm";
import { parseRelativeDate } from "../lib/date-parser";

const openai = new OpenAI();

interface ProcessingInput {
  meetingId: string;
  orgId: string;
  transcriptData: RecallTranscriptEntry[];
}

type RecallTranscriptEntry = {
  speaker: string;
  words: { text: string; start_time: number; end_time: number; confidence: number }[];
};

export async function processMeetingTranscript(input: ProcessingInput): Promise<void> {
  const { meetingId, orgId, transcriptData } = input;

  // Step 1: Normalize transcript into segments
  const segments = normalizeTranscript(transcriptData);
  const fullText = segments.map((s) => `${s.speaker}: ${s.text}`).join("\n");
  const wordCount = fullText.split(/\s+/).length;
  const confidenceAvg =
    segments.reduce((sum, s) => sum + s.confidence, 0) / segments.length;

  // Step 2: Store transcript
  const [transcript] = await db
    .insert(transcripts)
    .values({
      meetingId,
      segments,
      fullText,
      wordCount,
      confidenceAvg,
    })
    .returning();

  // Step 3: Generate embedding for semantic search
  const embeddingResponse = await openai.embeddings.create({
    model: "text-embedding-3-small",
    input: fullText.slice(0, 8000), // Limit input for embedding
  });

  await db.execute(
    `UPDATE transcripts SET embedding = $1 WHERE id = $2`,
    [JSON.stringify(embeddingResponse.data[0].embedding), transcript.id],
  );

  // Step 4: AI summary generation (two-pass)
  const meetingRecord = await db
    .select()
    .from(meetings)
    .where(eq(meetings.id, meetingId))
    .limit(1)
    .then((r) => r[0]);

  const summaryResult = await generateSummary(fullText, meetingRecord?.meetingType || "general");

  // Step 5: Store summary
  await db.insert(summaries).values({
    meetingId,
    tldr: summaryResult.tldr,
    decisions: summaryResult.decisions,
    discussionTopics: summaryResult.discussionTopics,
    openQuestions: summaryResult.openQuestions,
    sentiment: summaryResult.sentiment,
    templateUsed: meetingRecord?.meetingType || "general",
  });

  // Step 6: Store action items
  if (summaryResult.actionItems.length > 0) {
    await db.insert(actionItems).values(
      summaryResult.actionItems.map((item) => ({
        meetingId,
        orgId,
        text: item.text,
        assigneeName: item.assigneeName,
        assigneeEmail: item.assigneeEmail,
        dueDate: item.dueDate ? new Date(item.dueDate) : null,
        dueDateRaw: item.dueDateRaw,
        priority: item.priority || "normal",
        sourceTranscriptOffsetMs: item.sourceTranscriptOffsetMs,
      })),
    );
  }

  // Step 7: Update meeting status + participant talk times
  const participantTalkTimes = computeTalkTimes(segments);

  await db
    .update(meetings)
    .set({
      status: "ready",
      participants: participantTalkTimes,
      updatedAt: new Date(),
    })
    .where(eq(meetings.id, meetingId));
}

const SUMMARY_SYSTEM_PROMPT = `You are a meeting summarizer. Analyze the transcript and extract structured information.

Return JSON with these fields:
{
  "tldr": "2-3 sentence overview of the meeting",
  "decisions": ["Decision 1", "Decision 2", ...],
  "discussionTopics": [
    {"title": "Topic name", "summary": "Brief summary", "startMs": 0, "endMs": 0}
  ],
  "openQuestions": ["Unresolved question 1", ...],
  "actionItems": [
    {
      "text": "What needs to be done",
      "assigneeName": "Person who committed to it (from speech: 'I'll do X', 'John will handle Y')",
      "assigneeEmail": null,
      "dueDateRaw": "Original time reference ('by Friday', 'next week', 'end of month')",
      "priority": "low | normal | high",
      "sourceTranscriptOffsetMs": 0
    }
  ],
  "sentiment": "productive | contentious | brainstorming | informational | mixed"
}

Rules:
1. TL;DR should capture the essence of the meeting in 2-3 sentences
2. Decisions are explicit agreements ("we decided...", "let's go with...", "agreed to...")
3. Action items must have a clear actionable task — not vague statements
4. Only assign a person to an action item if they explicitly committed ("I'll...", "I can...", "[Name] will...")
5. Due dates should capture the raw language ("by Friday", "next sprint") — we'll resolve to actual dates separately
6. Discussion topics should cover the major sections of the meeting with approximate timing
7. Open questions are items raised but not resolved in this meeting
8. Sentiment should reflect the overall tone — contentious doesn't mean bad, brainstorming doesn't mean unfocused`;

const TEMPLATE_ADDITIONS: Record<string, string> = {
  standup: "\nThis is a daily standup. Focus on: what each person did yesterday, what they're doing today, and any blockers.",
  planning: "\nThis is a sprint/project planning meeting. Focus on: scope decisions, priority rankings, assignments, and timeline commitments.",
  "1on1": "\nThis is a 1-on-1 meeting. Focus on: personal development topics, feedback given, career goals discussed, and follow-up items.",
  client: "\nThis is a client meeting. Focus on: requirements discussed, commitments made, pricing/timeline conversations, and next steps.",
  allhands: "\nThis is an all-hands meeting. Focus on: company updates, announcements, Q&A topics, and organizational decisions.",
  brainstorm: "\nThis is a brainstorming session. Focus on: ideas generated, ideas shortlisted, evaluation criteria discussed, and next steps to validate.",
};

async function generateSummary(
  fullText: string,
  meetingType: string,
): Promise<{
  tldr: string;
  decisions: string[];
  discussionTopics: DiscussionTopic[];
  openQuestions: string[];
  actionItems: {
    text: string;
    assigneeName: string | null;
    assigneeEmail: string | null;
    dueDateRaw: string | null;
    dueDate: string | null;
    priority: string;
    sourceTranscriptOffsetMs: number | null;
  }[];
  sentiment: string;
}> {
  const systemPrompt =
    SUMMARY_SYSTEM_PROMPT + (TEMPLATE_ADDITIONS[meetingType] || "");

  // Truncate transcript if too long (GPT-4o context: 128K tokens)
  const truncatedText =
    fullText.length > 100_000
      ? fullText.slice(0, 50_000) + "\n...[middle truncated]...\n" + fullText.slice(-50_000)
      : fullText;

  const response = await openai.chat.completions.create({
    model: "gpt-4o",
    temperature: 0.3,
    max_tokens: 4000,
    response_format: { type: "json_object" },
    messages: [
      { role: "system", content: systemPrompt },
      {
        role: "user",
        content: `Analyze this meeting transcript:\n\n${truncatedText}`,
      },
    ],
  });

  const result = JSON.parse(response.choices[0].message.content!);

  // Resolve relative due dates to actual dates
  for (const item of result.actionItems || []) {
    if (item.dueDateRaw) {
      item.dueDate = parseRelativeDate(item.dueDateRaw)?.toISOString() || null;
    }
  }

  return result;
}

function normalizeTranscript(
  rawEntries: RecallTranscriptEntry[],
): TranscriptSegment[] {
  return rawEntries.map((entry) => {
    const text = entry.words.map((w) => w.text).join(" ");
    const startMs = entry.words[0]?.start_time
      ? Math.round(entry.words[0].start_time * 1000)
      : 0;
    const endMs = entry.words[entry.words.length - 1]?.end_time
      ? Math.round(entry.words[entry.words.length - 1].end_time * 1000)
      : 0;
    const avgConfidence =
      entry.words.reduce((sum, w) => sum + w.confidence, 0) /
      entry.words.length;

    return {
      speaker: entry.speaker || "Unknown Speaker",
      text,
      startMs,
      endMs,
      confidence: avgConfidence,
      words: entry.words.map((w) => ({
        word: w.text,
        startMs: Math.round(w.start_time * 1000),
        endMs: Math.round(w.end_time * 1000),
        confidence: w.confidence,
      })),
    };
  });
}

function computeTalkTimes(
  segments: TranscriptSegment[],
): { name: string; email?: string; talkTimeSeconds: number }[] {
  const talkTimes = new Map<string, number>();

  for (const segment of segments) {
    const durationMs = segment.endMs - segment.startMs;
    const current = talkTimes.get(segment.speaker) || 0;
    talkTimes.set(segment.speaker, current + durationMs);
  }

  return Array.from(talkTimes.entries())
    .map(([name, ms]) => ({
      name,
      talkTimeSeconds: Math.round(ms / 1000),
    }))
    .sort((a, b) => b.talkTimeSeconds - a.talkTimeSeconds);
}
```

### 3. Google Calendar Integration with Auto-Join

The calendar integration syncs with Google Calendar to detect upcoming meetings with Google Meet links, and automatically dispatches bots to join them based on user preferences.

```typescript
// src/server/services/calendar-sync.ts

import { google, type calendar_v3 } from "googleapis";
import { db } from "../db";
import { calendarConnections, meetings, members } from "../db/schema";
import { eq, and, gte, lte } from "drizzle-orm";
import { decrypt, encrypt } from "../integrations/encryption";
import { addMinutes, addHours, startOfDay, endOfDay } from "date-fns";

const ENCRYPTION_KEY = process.env.INTEGRATION_ENCRYPTION_KEY!;

export async function syncCalendarEvents(
  connectionId: string,
): Promise<void> {
  const [connection] = await db
    .select()
    .from(calendarConnections)
    .where(eq(calendarConnections.id, connectionId))
    .limit(1);

  if (!connection) throw new Error("Calendar connection not found");

  // Decrypt and create OAuth client
  const accessToken = decrypt(connection.accessTokenEncrypted, ENCRYPTION_KEY);
  const refreshToken = connection.refreshTokenEncrypted
    ? decrypt(connection.refreshTokenEncrypted, ENCRYPTION_KEY)
    : undefined;

  const oauth2Client = new google.auth.OAuth2(
    process.env.GOOGLE_CLIENT_ID!,
    process.env.GOOGLE_CLIENT_SECRET!,
    `${process.env.NEXT_PUBLIC_APP_URL}/api/auth/google/callback`,
  );

  oauth2Client.setCredentials({
    access_token: accessToken,
    refresh_token: refreshToken,
    expiry_date: connection.tokenExpiresAt?.getTime(),
  });

  // Handle token refresh
  oauth2Client.on("tokens", async (tokens) => {
    if (tokens.access_token) {
      await db
        .update(calendarConnections)
        .set({
          accessTokenEncrypted: encrypt(tokens.access_token, ENCRYPTION_KEY),
          tokenExpiresAt: tokens.expiry_date
            ? new Date(tokens.expiry_date)
            : undefined,
          refreshTokenEncrypted: tokens.refresh_token
            ? encrypt(tokens.refresh_token, ENCRYPTION_KEY)
            : connection.refreshTokenEncrypted,
          status: "connected",
          lastSyncedAt: new Date(),
        })
        .where(eq(calendarConnections.id, connectionId));
    }
  });

  const calendar = google.calendar({ version: "v3", auth: oauth2Client });

  // Fetch events for the next 24 hours
  const now = new Date();
  const tomorrow = addHours(now, 24);

  const response = await calendar.events.list({
    calendarId: "primary",
    timeMin: now.toISOString(),
    timeMax: tomorrow.toISOString(),
    singleEvents: true,
    orderBy: "startTime",
  });

  const events = response.data.items || [];

  for (const event of events) {
    await processCalendarEvent(event, connection);
  }

  // Update last synced
  await db
    .update(calendarConnections)
    .set({ lastSyncedAt: new Date() })
    .where(eq(calendarConnections.id, connectionId));
}

async function processCalendarEvent(
  event: calendar_v3.Schema$Event,
  connection: typeof calendarConnections.$inferSelect,
): Promise<void> {
  if (!event.id || !event.start?.dateTime) return;

  // Extract Google Meet URL
  const meetUrl = extractMeetUrl(event);
  if (!meetUrl) return; // Skip events without video conferencing

  // Check auto-join settings
  const settings = connection.autoJoinSettings;
  if (settings?.mode === "manual") return;

  // Check exclude patterns
  if (settings?.excludePatterns?.length) {
    const title = event.summary?.toLowerCase() || "";
    const shouldExclude = settings.excludePatterns.some((pattern) =>
      title.includes(pattern.toLowerCase()),
    );
    if (shouldExclude) return;
  }

  // Check if meeting already exists
  const existing = await db
    .select()
    .from(meetings)
    .where(eq(meetings.externalMeetingId, event.id))
    .limit(1);

  if (existing.length > 0) return; // Already tracked

  // Detect meeting type from title
  const meetingType = detectMeetingType(event.summary || "");

  // Create meeting record
  const [meeting] = await db
    .insert(meetings)
    .values({
      orgId: connection.orgId,
      title: event.summary || "Untitled Meeting",
      platform: "google_meet",
      meetingUrl: meetUrl,
      externalMeetingId: event.id,
      calendarConnectionId: connection.id,
      startedAt: new Date(event.start.dateTime),
      endedAt: event.end?.dateTime ? new Date(event.end.dateTime) : null,
      meetingType,
      status: "scheduled",
    })
    .returning();

  // Schedule bot dispatch (join 1 minute before meeting start)
  const startTime = new Date(event.start.dateTime);
  const joinTime = addMinutes(startTime, -1);
  const delay = Math.max(0, joinTime.getTime() - Date.now());

  const { botQueue } = await import("../queue/bot-queue");
  await botQueue.add(
    "dispatch-bot",
    { meetingId: meeting.id },
    { delay, jobId: `bot-${meeting.id}` },
  );
}

function extractMeetUrl(event: calendar_v3.Schema$Event): string | null {
  // Check conferenceData for Google Meet
  if (event.conferenceData?.entryPoints) {
    const videoEntry = event.conferenceData.entryPoints.find(
      (ep) => ep.entryPointType === "video",
    );
    if (videoEntry?.uri) return videoEntry.uri;
  }

  // Check hangoutLink
  if (event.hangoutLink) return event.hangoutLink;

  // Check description for meeting URLs
  const description = event.description || "";
  const meetMatch = description.match(
    /https:\/\/meet\.google\.com\/[a-z]{3}-[a-z]{4}-[a-z]{3}/,
  );
  if (meetMatch) return meetMatch[0];

  return null;
}

function detectMeetingType(title: string): string {
  const lower = title.toLowerCase();
  if (lower.includes("standup") || lower.includes("stand-up") || lower.includes("daily sync"))
    return "standup";
  if (lower.includes("sprint") || lower.includes("planning") || lower.includes("backlog"))
    return "planning";
  if (lower.includes("1:1") || lower.includes("1-on-1") || lower.includes("one on one"))
    return "1on1";
  if (lower.includes("all hands") || lower.includes("all-hands") || lower.includes("town hall"))
    return "allhands";
  if (lower.includes("brainstorm") || lower.includes("ideation"))
    return "brainstorm";
  return "general";
}

// Set up Google Calendar push notifications
export async function setupCalendarWatch(
  connectionId: string,
): Promise<void> {
  const [connection] = await db
    .select()
    .from(calendarConnections)
    .where(eq(calendarConnections.id, connectionId))
    .limit(1);

  if (!connection) return;

  const accessToken = decrypt(connection.accessTokenEncrypted, ENCRYPTION_KEY);

  const oauth2Client = new google.auth.OAuth2();
  oauth2Client.setCredentials({ access_token: accessToken });

  const calendar = google.calendar({ version: "v3", auth: oauth2Client });

  const channelId = `cal-${connectionId}-${Date.now()}`;
  const expiration = new Date(Date.now() + 7 * 24 * 60 * 60 * 1000); // 7 days

  const response = await calendar.events.watch({
    calendarId: "primary",
    requestBody: {
      id: channelId,
      type: "web_hook",
      address: `${process.env.NEXT_PUBLIC_APP_URL}/api/webhooks/google-calendar`,
      expiration: expiration.getTime().toString(),
    },
  });

  await db
    .update(calendarConnections)
    .set({
      watchChannelId: channelId,
      watchResourceId: response.data.resourceId,
      watchExpiration: expiration,
    })
    .where(eq(calendarConnections.id, connectionId));
}
```

### 4. Hybrid Transcript Search (pgvector + pg_trgm)

The search system combines semantic search (for conceptual queries like "what did we decide about pricing?") with keyword search (for exact names and terms), merged via Reciprocal Rank Fusion.

```typescript
// src/server/services/search.ts

import OpenAI from "openai";
import { db } from "../db";
import { meetings, transcripts, summaries } from "../db/schema";
import { eq, sql, and, desc } from "drizzle-orm";

const openai = new OpenAI();

interface SearchResult {
  meetingId: string;
  meetingTitle: string;
  meetingDate: Date;
  score: number;
  matchType: "semantic" | "keyword" | "hybrid";
  snippet: string; // Relevant excerpt from transcript
  highlightedText?: string;
}

export async function searchTranscripts(
  orgId: string,
  query: string,
  options?: {
    limit?: number;
    dateFrom?: Date;
    dateTo?: Date;
    meetingType?: string;
    speakerName?: string;
  },
): Promise<SearchResult[]> {
  const limit = options?.limit ?? 20;
  const k = 60; // RRF constant

  // Generate embedding for semantic search
  const embeddingResponse = await openai.embeddings.create({
    model: "text-embedding-3-small",
    input: query,
  });
  const queryEmbedding = embeddingResponse.data[0].embedding;

  // Build date filter clause
  const dateFilter = [];
  if (options?.dateFrom) {
    dateFilter.push(sql`m.started_at >= ${options.dateFrom.toISOString()}`);
  }
  if (options?.dateTo) {
    dateFilter.push(sql`m.started_at <= ${options.dateTo.toISOString()}`);
  }
  if (options?.meetingType) {
    dateFilter.push(sql`m.meeting_type = ${options.meetingType}`);
  }

  const filterClause =
    dateFilter.length > 0
      ? sql`AND ${sql.join(dateFilter, sql` AND `)}`
      : sql``;

  // Semantic search: cosine similarity on transcript embeddings
  const semanticResults = await db.execute<{
    meeting_id: string;
    title: string;
    started_at: Date;
    similarity: number;
    full_text: string;
  }>(sql`
    SELECT m.id AS meeting_id, m.title, m.started_at,
           1 - (t.embedding <=> ${JSON.stringify(queryEmbedding)}::vector) AS similarity,
           t.full_text
    FROM transcripts t
    JOIN meetings m ON m.id = t.meeting_id
    WHERE m.org_id = ${orgId}
      AND m.status = 'ready'
      AND t.embedding IS NOT NULL
      ${filterClause}
    ORDER BY t.embedding <=> ${JSON.stringify(queryEmbedding)}::vector
    LIMIT ${limit}
  `);

  // Keyword search: pg_trgm similarity on full_text
  const keywordResults = await db.execute<{
    meeting_id: string;
    title: string;
    started_at: Date;
    similarity: number;
    full_text: string;
  }>(sql`
    SELECT m.id AS meeting_id, m.title, m.started_at,
           similarity(t.full_text, ${query}) AS similarity,
           t.full_text
    FROM transcripts t
    JOIN meetings m ON m.id = t.meeting_id
    WHERE m.org_id = ${orgId}
      AND m.status = 'ready'
      AND t.full_text % ${query}
      ${filterClause}
    ORDER BY similarity(t.full_text, ${query}) DESC
    LIMIT ${limit}
  `);

  // Reciprocal Rank Fusion
  const semanticRanks = new Map<string, number>();
  semanticResults.rows.forEach((r, i) => {
    semanticRanks.set(r.meeting_id, i + 1);
  });

  const keywordRanks = new Map<string, number>();
  keywordResults.rows.forEach((r, i) => {
    keywordRanks.set(r.meeting_id, i + 1);
  });

  // Collect all unique meeting IDs
  const allMeetingIds = new Set([
    ...semanticRanks.keys(),
    ...keywordRanks.keys(),
  ]);

  // Compute RRF scores
  const rrfScores: {
    meetingId: string;
    score: number;
    matchType: "semantic" | "keyword" | "hybrid";
    fullText: string;
    title: string;
    startedAt: Date;
  }[] = [];

  for (const meetingId of allMeetingIds) {
    const semanticRank = semanticRanks.get(meetingId);
    const keywordRank = keywordRanks.get(meetingId);

    const semanticScore = semanticRank ? 1 / (k + semanticRank) : 0;
    const keywordScore = keywordRank ? 1 / (k + keywordRank) : 0;
    const rrfScore = semanticScore + keywordScore;

    const matchType: "semantic" | "keyword" | "hybrid" =
      semanticRank && keywordRank
        ? "hybrid"
        : semanticRank
          ? "semantic"
          : "keyword";

    // Get full text from whichever result set has it
    const semanticRow = semanticResults.rows.find(
      (r) => r.meeting_id === meetingId,
    );
    const keywordRow = keywordResults.rows.find(
      (r) => r.meeting_id === meetingId,
    );
    const row = semanticRow || keywordRow!;

    rrfScores.push({
      meetingId,
      score: rrfScore,
      matchType,
      fullText: row.full_text,
      title: row.title,
      startedAt: row.started_at,
    });
  }

  // Sort by RRF score descending
  rrfScores.sort((a, b) => b.score - a.score);

  // Extract relevant snippets
  return rrfScores.slice(0, limit).map((result) => ({
    meetingId: result.meetingId,
    meetingTitle: result.title,
    meetingDate: result.startedAt,
    score: result.score,
    matchType: result.matchType,
    snippet: extractSnippet(result.fullText, query),
  }));
}

function extractSnippet(fullText: string, query: string): string {
  const lowerText = fullText.toLowerCase();
  const lowerQuery = query.toLowerCase();
  const queryWords = lowerQuery.split(/\s+/);

  // Find the best matching position
  let bestPos = 0;
  let bestScore = 0;

  for (let i = 0; i < lowerText.length - 200; i += 50) {
    const window = lowerText.slice(i, i + 200);
    let score = 0;
    for (const word of queryWords) {
      if (window.includes(word)) score++;
    }
    if (score > bestScore) {
      bestScore = score;
      bestPos = i;
    }
  }

  // Extract 200 chars around the best position
  const start = Math.max(0, bestPos - 50);
  const end = Math.min(fullText.length, start + 300);
  let snippet = fullText.slice(start, end).trim();

  if (start > 0) snippet = "..." + snippet;
  if (end < fullText.length) snippet = snippet + "...";

  return snippet;
}

// AI-powered question answering over transcripts
export async function askAboutMeetings(
  orgId: string,
  question: string,
): Promise<{ answer: string; citations: { meetingId: string; meetingTitle: string; excerpt: string }[] }> {
  // Search for relevant transcripts
  const results = await searchTranscripts(orgId, question, { limit: 5 });

  if (results.length === 0) {
    return {
      answer: "I couldn't find any relevant meetings for your question.",
      citations: [],
    };
  }

  // Build context from top results
  const context = results
    .map(
      (r) =>
        `[Meeting: "${r.meetingTitle}" on ${r.meetingDate.toLocaleDateString()}]\n${r.snippet}`,
    )
    .join("\n\n---\n\n");

  const response = await openai.chat.completions.create({
    model: "gpt-4o",
    temperature: 0.2,
    max_tokens: 500,
    messages: [
      {
        role: "system",
        content:
          "You are a meeting knowledge assistant. Answer questions based on the provided meeting transcript excerpts. Cite which meeting(s) your answer comes from. Be concise and specific.",
      },
      {
        role: "user",
        content: `Question: ${question}\n\nMeeting context:\n${context}`,
      },
    ],
  });

  return {
    answer: response.choices[0].message.content || "Unable to generate answer.",
    citations: results.slice(0, 3).map((r) => ({
      meetingId: r.meetingId,
      meetingTitle: r.meetingTitle,
      excerpt: r.snippet,
    })),
  };
}
```

---

## Phase Breakdown

### Phase 1: Project Scaffold + Database (Days 1–5)

**Day 1 — Initialize project**
```bash
npx create-next-app@latest meetingtldr --typescript --tailwind --eslint --app --src-dir
cd meetingtldr
npm install drizzle-orm @neondatabase/serverless
npm install -D drizzle-kit
npm install @trpc/server @trpc/client @trpc/react-query @trpc/next superjson
npm install @clerk/nextjs
npm install zod date-fns
```
- Configure `tsconfig.json`: strict mode, path aliases
- Create `src/env.ts` with Zod-validated environment variables:
  - `DATABASE_URL`, `CLERK_SECRET_KEY`, `NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY`
  - `OPENAI_API_KEY`, `STRIPE_SECRET_KEY`, `STRIPE_WEBHOOK_SECRET`
  - `RECALL_API_KEY`, `DEEPGRAM_API_KEY`
  - `GOOGLE_CLIENT_ID`, `GOOGLE_CLIENT_SECRET`
  - `SLACK_CLIENT_ID`, `SLACK_CLIENT_SECRET`, `SLACK_SIGNING_SECRET`
  - `R2_ACCESS_KEY_ID`, `R2_SECRET_ACCESS_KEY`, `R2_BUCKET_NAME`, `R2_ENDPOINT`
  - `UPSTASH_REDIS_HOST`, `UPSTASH_REDIS_PASSWORD`
  - `INTEGRATION_ENCRYPTION_KEY`, `RESEND_API_KEY`

**Day 2 — Database schema + connection**
- Create `src/server/db/schema.ts` with all tables: `organizations`, `members`, `calendarConnections`, `meetings`, `transcripts`, `summaries`, `actionItems`, `bookmarks`, `shares`, `slackConnections`, `glossary`
- Create `src/server/db/index.ts`: Neon serverless client
- Create `drizzle.config.ts`
- Run `npx drizzle-kit push`
- Apply migration: pgvector + pg_trgm extensions + embedding column + HNSW index + trigram index

**Day 3 — tRPC setup + Clerk auth**
- Create `src/server/trpc/trpc.ts` with `publicProcedure`, `protectedProcedure`, `orgProcedure`
- Create context, merged router, HTTP handler
- Create React Query tRPC client + server-side caller

**Day 4 — Clerk middleware + app shell**
- Create `src/middleware.ts`: protect dashboard routes
- Create `src/app/layout.tsx` with ClerkProvider
- Create `src/app/dashboard/layout.tsx`: sidebar (Meetings, Action Items, Search, Settings) + header
- Install Radix UI primitives
- Create shared UI components

**Day 5 — Organization + member sync**
- Create `src/server/trpc/routers/organization.ts`
- Create Clerk webhook handler
- Create `src/server/integrations/encryption.ts`: AES-256-GCM encrypt/decrypt

---

### Phase 2: Google Calendar Integration (Days 6–10)

**Day 6 — Google Calendar OAuth**
```bash
npm install googleapis
```
- Create `src/app/api/auth/google/route.ts`: initiate OAuth
  - Scopes: `calendar.readonly`, `calendar.events.readonly`
- Create `src/app/api/auth/google/callback/route.ts`: exchange code for tokens
- Store encrypted tokens in `calendarConnections`

**Day 7 — Calendar sync service**
- Create `src/server/services/calendar-sync.ts`:
  - `syncCalendarEvents(connectionId)`: fetch events for next 24 hours
  - `processCalendarEvent(event, connection)`: extract Meet URL, create meeting record
  - `extractMeetUrl(event)`: parse conferenceData, hangoutLink, or description URLs
  - `detectMeetingType(title)`: classify from title keywords

**Day 8 — Calendar push notifications**
- Create `src/server/services/calendar-sync.ts` → `setupCalendarWatch(connectionId)`:
  - Register webhook for calendar changes
  - Renew watch every 7 days
- Create `src/app/api/webhooks/google-calendar/route.ts`:
  - Verify channel ID + resource ID
  - Trigger incremental sync on calendar change
  - Debounce: max 1 sync per 60 seconds per connection

**Day 9 — Calendar connection UI**
- Create `src/app/dashboard/settings/calendar/page.tsx`:
  - "Connect Google Calendar" button → OAuth flow
  - Connected accounts list with status
  - Auto-join settings per connection:
    - All meetings / Selected calendars / Manual only
    - Exclude patterns (e.g., "Lunch", "Focus time")
  - Disconnect button with confirmation
  - Last sync timestamp

**Day 10 — Upcoming meetings list**
- Create `src/server/trpc/routers/meeting.ts`:
  - `listUpcoming`: meetings with status=scheduled, ordered by startedAt
  - `listPast`: meetings with status=ready, paginated
  - `getById`: single meeting with transcript + summary + action items
- Create `src/app/dashboard/page.tsx`:
  - Upcoming meetings with auto-join toggle per meeting
  - Past meetings with "Summary ready" / "Processing" badges
  - Search bar (routes to search page)

---

### Phase 3: Meeting Bot + Transcription (Days 11–17)

**Day 11 — Recall.ai bot integration**
- Create `src/server/services/meeting-bot.ts`:
  - `dispatchBotToMeeting(meetingId)`: create Recall.ai bot for meeting URL
  - Configure bot: name, auto-leave settings, chat message
- Create bot dispatch queue:
  ```bash
  npm install bullmq ioredis
  ```
  - `src/server/queue/connection.ts`
  - `src/server/queue/bot-queue.ts`
  - `src/server/queue/bot-worker.ts`: dispatch bot at scheduled time

**Day 12 — Recall.ai webhook handlers**
- Create `src/app/api/webhooks/recall/route.ts`:
  - `bot.status_change`: handle joining/recording/ended/fatal statuses
  - `bot.transcription`: receive real-time transcription segments
  - Verify webhook signatures
- Create `src/server/services/meeting-bot.ts` → `handleBotStatusChange()`:
  - `in_call_recording`: update meeting status to "recording"
  - `call_ended`: trigger processing pipeline
  - `fatal`: mark meeting as failed

**Day 13 — Audio recording storage**
- Create recording download + R2 upload flow:
  - Download recording from Recall.ai temporary URL
  - Upload to R2 at `recordings/{orgId}/{meetingId}/audio.webm`
  - Store R2 key + size in meeting record
- Create `src/server/services/storage.ts`:
  - `generatePresignedPlaybackUrl(r2Key)`: 1-hour expiry GET URL for audio player

**Day 14 — Transcript storage + normalization**
- Create `src/server/services/transcript.ts`:
  - Normalize Recall.ai transcript format → internal `TranscriptSegment` format
  - Merge consecutive segments from same speaker (within 2s gap)
  - Calculate word count, average confidence
  - Store in `transcripts` table
- Handle speaker name resolution:
  - Map Recall.ai speaker IDs to participant names from calendar event
  - Fallback to "Speaker 1", "Speaker 2" numbering

**Day 15 — Real-time transcription buffering**
- Create `src/server/queue/transcription-buffer.ts`:
  - Buffer real-time segments from Recall.ai webhooks
  - Store in Redis sorted set (by timestamp)
  - On meeting end: merge buffered segments → finalize transcript
- Handle late-arriving segments (out-of-order delivery)

**Day 16 — Bot management UI**
- Add to meeting list page:
  - "Recording" status indicator for active meetings
  - "Join now" button for manual bot dispatch
  - "Remove bot" button for active meetings
  - Bot status badge: Joining, Recording, Processing
- Create meeting detail stub page:
  - Real-time transcript viewer (if meeting in progress)
  - "Processing..." state with spinner

**Day 17 — Error handling + edge cases**
- Handle bot failures:
  - Waiting room timeout → mark as failed, notify user
  - Bot kicked → mark as failed, notify user
  - Network interruption → auto-reconnect (Recall.ai handles)
- Handle empty meetings (no one spoke → no transcript)
- Handle very short meetings (<2 minutes → skip summarization)
- Handle very long meetings (>4 hours → truncate transcript for summarization)

---

### Phase 4: AI Summarization (Days 18–23)

**Day 18 — Processing pipeline**
- Create `src/server/queue/processing-queue.ts` + `processing-worker.ts`:
  - Job data: `{ meetingId, orgId, transcriptData }`
  - Pipeline steps:
    1. Normalize transcript
    2. Store transcript + generate embedding
    3. Generate summary via GPT-4o
    4. Extract action items
    5. Update meeting status to "ready"
  - Retry logic: 3 attempts with exponential backoff

**Day 19 — Summarization engine + topic segmentation**
- Create `src/server/ai/summarize-meeting.ts`:
  - `generateSummary(fullText, meetingType)`: GPT-4o structured JSON output
  - Template additions per meeting type (standup, planning, 1on1, client, allhands)
  - Extract: TL;DR, decisions, discussion topics, open questions, action items, sentiment
- Create `src/server/ai/topic-segmentation.ts`:
  - Pre-processing step before summarization: split transcript into coherent topic segments
  - Algorithm: sliding window (5-minute chunks) with embedding similarity between adjacent windows
  - Topic boundary detection: when cosine similarity between adjacent windows drops below 0.7 threshold, mark as topic boundary
  - Each segment gets a topic label via GPT-4o-mini (single-sentence classification, temperature 0.2)
  - Segments stored in `summaries.discussionTopics` with accurate `startMs`/`endMs` timestamps
  - Enables "jump to topic" navigation in the transcript viewer

**Day 20 — Action item extraction + date parsing**
- Create `src/server/lib/date-parser.ts`:
  - `parseRelativeDate(raw)`: resolve "by Friday", "next week", "end of month" to actual dates
  - Handle relative references based on meeting date
- Action item extraction rules:
  - "I'll handle X" → assignee = current speaker
  - "[Name] will do Y" → assignee = named person
  - "by [date]" → due date
  - Priority inference: "urgent", "critical", "ASAP" → high

**Day 21 — Transcript embedding generation**
- Generate text-embedding-3-small embedding for each transcript
- Store in pgvector column
- Create HNSW index for fast similarity search
- Handle long transcripts: embed first 8000 tokens (covers most meetings)

**Day 22 — Summary view page**
- Create `src/app/dashboard/meetings/[meetingId]/page.tsx`:
  - Header: title, date, duration, participants (with avatars)
  - TL;DR section (prominently displayed)
  - Decisions list (highlighted with check icons)
  - Action items checklist (with assignee badges)
  - Discussion topics (expandable sections)
  - Open questions list
  - Sentiment badge (productive, contentious, etc.)
  - "Share" button, "Regenerate" button
  - Tabs: Summary | Transcript | Action Items

**Day 23 — Transcript view**
- Create transcript tab on meeting detail page:
  - Scrollable transcript with speaker labels and timestamps
  - Color-coded speakers
  - Click timestamp → play audio from that point (audio player integration)
  - Search within transcript (highlight matching text)
  - Bookmark button per segment (save with optional note)
  - Speaker filter: show only one person's contributions

---

### Phase 5: Search + Sharing (Days 24–29)

**Day 24 — Hybrid search engine**
- Create `src/server/services/search.ts`:
  - `searchTranscripts(orgId, query, options)`: pgvector + pg_trgm + RRF
  - Date range, meeting type, and speaker filters
  - Snippet extraction with query-relevant excerpts

**Day 25 — Search UI**
- Create `src/app/dashboard/search/page.tsx`:
  - Search bar with instant results (debounced 300ms)
  - Results list: meeting title, date, relevance score, snippet with highlights
  - Filters sidebar: date range, meeting type, speaker
  - Click result → navigate to meeting summary

**Day 26 — AI question answering**
- Create `src/server/services/search.ts` → `askAboutMeetings()`:
  - Search top 5 relevant transcripts
  - Feed context to GPT-4o with question
  - Return answer with meeting citations
- Add "Ask a question" mode to search page:
  - Toggle between "Search transcripts" and "Ask a question"
  - AI answer displayed as a card with cited meetings below

**Day 27 — Email sharing**
- Create `src/server/services/sharing.ts`:
  - `shareViaEmail(meetingId, recipientEmails, sections)`:
    - Generate HTML email with TL;DR, decisions, action items
    - Send via Resend
    - Log in `shares` table
- Create `src/server/email/templates.ts`:
  - Meeting summary email template
  - Includes: meeting title, date, duration, participants, selected sections
  - "View full summary" link to web dashboard

**Day 28 — Slack integration**
- Create `src/app/api/auth/slack/route.ts`: Slack OAuth (Bot Token Scopes: `chat:write`, `channels:read`)
- Create `src/app/api/auth/slack/callback/route.ts`: store encrypted token
- Create `src/server/services/sharing.ts` → `shareToSlack()`:
  - Format summary as Slack Block Kit message
  - Post to specified channel
  - Log in `shares` table

**Day 29 — Auto-sharing configuration**
- Create auto-share settings on organization settings page:
  - "After every meeting, automatically share to:"
  - Slack channel selector (fetch channels via API)
  - Email recipients list
  - Include sections: TL;DR, Decisions, Action Items (checkboxes)
- Trigger auto-share after processing pipeline completes
- Create `src/server/trpc/routers/share.ts`:
  - `share`: manually share a meeting summary
  - `listShares`: share history for a meeting
  - `updateAutoShareConfig`: update org auto-share settings

---

### Phase 6: Action Items + Analytics + Glossary Wiring (Days 30–35)

> **Note:** This phase was extended by 2 days (originally Days 30-33, now Days 30-35) to accommodate the complexity of cross-meeting action item tracking, analytics query optimization, and glossary integration wiring into both Deepgram and GPT-4o pipelines.

**Day 30 — Action item tracker**
- Create `src/server/trpc/routers/actionItem.ts`:
  - `list`: all action items for org, filter by status/assignee/meeting
  - `getById`: single action item with meeting context
  - `updateStatus`: toggle pending/done
  - `bulkUpdateStatus`: bulk mark done/dismiss
  - `reassign`: change assignee
- Create `src/app/dashboard/action-items/page.tsx`:
  - Cross-meeting action item list
  - Filter by: assignee, status, meeting, due date
  - Overdue items highlighted in red
  - Quick "Mark done" checkbox
  - Click item → navigate to source meeting transcript at offset

**Day 31 — Meeting list enhancements**
- Enhance `src/app/dashboard/page.tsx`:
  - Upcoming meetings: next 7 days, auto-join toggles
  - Recent meetings: last 10, with summary previews
  - Action items due this week
  - Quick stats: meetings this week, action items pending
- Add meeting tags: click to add/remove tags for organization

**Day 32 — Basic analytics**
- Create `src/server/trpc/routers/analytics.ts`:
  - `meetingStats`: total meetings, avg duration, meetings per week trend
  - `actionItemStats`: created vs completed, completion rate
  - `talkTimeStats`: per-person talk time across meetings
- Create `src/app/dashboard/analytics/page.tsx`:
  - Meeting hours per week chart (area chart)
  - Average meeting duration trend (line chart)
  - Action items: created vs completed (bar chart)
  - Talk time distribution per meeting (horizontal bar chart)
  ```bash
  npm install recharts
  ```

**Day 33 — Meeting detail enhancements**
- Add audio player to meeting detail page:
  - Play/pause, seek bar, playback speed (1x, 1.5x, 2x)
  - Click transcript segment → seek to that timestamp
  - Keyboard shortcuts: space (play/pause), arrow keys (seek ±10s)
- Summary editing:
  - Click TL;DR/decision/action item to edit inline
  - "Regenerate summary" button with template selector
  - Mark summary as edited, track editor

---

### Phase 7: Billing + Settings (Days 36–39)

**Day 34 — Stripe integration**
```bash
npm install stripe
```
- Create `src/server/billing/stripe.ts`:
  - Plan definitions:
    ```typescript
    const PLAN_LIMITS = {
      free: { maxMeetingsPerMonth: 5, maxDurationMin: 30, retentionDays: 7, search: false, analytics: false },
      pro: { maxMeetingsPerMonth: Infinity, maxDurationMin: 120, retentionDays: Infinity, search: true, analytics: true },
      team: { maxMeetingsPerMonth: Infinity, maxDurationMin: 240, retentionDays: Infinity, search: true, analytics: true },
    };
    ```
  - Per-user pricing for team plan

**Day 35 — Stripe webhook + billing UI**
- Create `src/app/api/webhooks/stripe/route.ts`
- Create `src/server/trpc/routers/billing.ts`
- Create `src/app/dashboard/settings/billing/page.tsx`:
  - Current plan with usage (meetings this month, retention status)
  - Three plan cards: Free, Pro ($18/mo), Team ($12/user/mo)
  - Upgrade / Manage Subscription buttons

**Day 36 — Plan enforcement + settings**
- Apply plan limits:
  - Meeting count: block bot dispatch when limit reached
  - Duration: auto-remove bot after max duration
  - Retention: archive/delete transcripts older than retention period
  - Search: gate search page behind pro+ plan
  - Analytics: gate analytics page behind pro+ plan
- Create `src/app/dashboard/settings/page.tsx`:
  - Organization name
  - Default summary template (dropdown)
  - Timezone
  - Glossary management (add/edit/delete terms)

**Day 37 — Glossary management + integration wiring**
- Create `src/server/trpc/routers/glossary.ts`:
  - `list`, `create`, `update`, `delete`
- Create glossary section on settings page:
  - Add term: term + pronunciation hint + description
  - Edit/delete existing terms
- **Deepgram vocabulary wiring** (`src/server/services/meeting-bot.ts`):
  - On bot dispatch, fetch org glossary terms and pass as Deepgram `keywords` parameter
  - Map glossary entries to Deepgram format: `{ "keywords": [{ "word": term, "boost": 5 }] }` for each glossary term
  - Include `pronunciationHint` as Deepgram `search` terms for phonetic matching
  - Rebuild keywords list on glossary update (cached in Redis, TTL 1 hour)
- **GPT-4o summarization wiring** (`src/server/ai/summarize-meeting.ts`):
  - Prepend glossary context block to the system prompt:
    ```
    COMPANY GLOSSARY (use these exact terms in your summary):
    - {term}: {description}
    ```
  - Glossary terms injected into action item extraction to improve assignee name matching
  - Glossary terms used as seed vocabulary for topic segmentation labels

---

### Phase 8: Polish + Launch (Days 40–44)

**Day 38 — Onboarding flow**
- Create `src/app/onboarding/page.tsx`:
  - Step 1: Create organization
  - Step 2: Connect Google Calendar → show upcoming meetings
  - Step 3: Configure auto-join preference (all / selected / manual)
  - Step 4: "Ready! Your next meeting will be automatically recorded."
  - Skip option to go straight to dashboard

**Day 39 — Loading states, error handling, empty states**
- Skeleton loaders for meeting list, summary view, search results
- Empty states:
  - No meetings: "Connect your calendar to start recording meetings"
  - No summary yet: "Processing your meeting... summary will be ready in 2-3 minutes"
  - No action items: "No action items from your meetings yet"
  - No search results: "No meetings found matching your query"
- Toast notifications for all actions
- Error boundary with retry

**Day 40 — Responsive design + accessibility**
- Mobile: single-column layout, collapsible sidebar
- Meeting summary: readable on mobile, expandable sections
- Transcript: scrollable with fixed audio player
- ARIA labels on all interactive elements
- Keyboard navigation through transcript

**Day 41 — Performance + SEO**
- React Query cache settings:
  - Meeting list: staleTime 30s
  - Meeting detail: staleTime 5min
  - Action items: staleTime 30s
  - Search results: staleTime 0 (always fresh)
- Server-side rendering for dashboard pages
- Landing page SEO: meta tags, Open Graph

**Day 42 — Landing page + final testing**
- Create `src/app/page.tsx`:
  - Hero: "Never miss a meeting decision again" with CTA
  - How it works: 3-step (Connect Calendar → Bot Joins → Get Summary)
  - Features: AI Summary, Action Items, Search, Sharing
  - Pricing table (3 tiers)
  - Footer
- End-to-end test:
  1. Sign up → Create org → Connect Google Calendar
  2. Create test Google Meet → verify meeting detected
  3. Start meeting → verify bot joins
  4. Talk for 5 minutes → end meeting
  5. Verify transcript appears within 1 minute
  6. Verify summary appears within 2-3 minutes
  7. Check action items extracted correctly
  8. Share via email → verify email received
  9. Share to Slack → verify message posted
  10. Search for meeting topic → verify results
  11. Upgrade plan → verify limits change
- Deploy: Vercel (app) + Railway (bot worker)
- Configure Sentry, Vercel Analytics
- Register Stripe webhook + Google Calendar push notification URLs

---

## Critical Files

```
meetingtldr/
├── src/
│   ├── env.ts                                    # Zod-validated environment variables
│   ├── middleware.ts                              # Clerk auth middleware
│   │
│   ├── app/
│   │   ├── layout.tsx                            # Root layout with ClerkProvider
│   │   ├── page.tsx                              # Landing page
│   │   ├── pricing/page.tsx                      # Pricing page
│   │   ├── onboarding/page.tsx                   # 4-step onboarding wizard
│   │   │
│   │   ├── dashboard/
│   │   │   ├── layout.tsx                        # App shell: sidebar + header
│   │   │   ├── page.tsx                          # Dashboard: upcoming + recent meetings
│   │   │   ├── meetings/
│   │   │   │   └── [meetingId]/
│   │   │   │       └── page.tsx                  # Meeting detail: summary + transcript + action items
│   │   │   ├── action-items/page.tsx             # Cross-meeting action item tracker
│   │   │   ├── search/page.tsx                   # Transcript search + AI Q&A
│   │   │   ├── analytics/page.tsx                # Meeting analytics dashboard
│   │   │   └── settings/
│   │   │       ├── page.tsx                      # Organization settings + glossary
│   │   │       ├── calendar/page.tsx             # Calendar connections
│   │   │       ├── sharing/page.tsx              # Auto-share config (Slack, email)
│   │   │       ├── billing/page.tsx              # Plan + usage
│   │   │       └── members/page.tsx              # Team members
│   │   │
│   │   └── api/
│   │       ├── trpc/[trpc]/route.ts              # tRPC HTTP handler
│   │       ├── auth/
│   │       │   ├── google/route.ts               # Google Calendar OAuth start
│   │       │   ├── google/callback/route.ts      # Google Calendar OAuth callback
│   │       │   ├── slack/route.ts                # Slack OAuth start
│   │       │   └── slack/callback/route.ts       # Slack OAuth callback
│   │       └── webhooks/
│   │           ├── clerk/route.ts                # Clerk org/member sync
│   │           ├── stripe/route.ts               # Stripe webhook handler
│   │           ├── recall/route.ts               # Recall.ai bot status + transcription
│   │           └── google-calendar/route.ts      # Google Calendar push notifications
│   │
│   ├── server/
│   │   ├── db/
│   │   │   ├── schema.ts                        # Drizzle schema (all tables + relations)
│   │   │   ├── index.ts                         # Neon database client
│   │   │   └── migrations/
│   │   │       └── 0000_init.sql                # pgvector, pg_trgm, embedding + indexes
│   │   │
│   │   ├── trpc/
│   │   │   ├── trpc.ts                          # tRPC init, middleware
│   │   │   ├── context.ts                       # Request context
│   │   │   └── routers/
│   │   │       ├── _app.ts                      # Merged router
│   │   │       ├── organization.ts              # Org CRUD
│   │   │       ├── meeting.ts                   # Meeting list + detail
│   │   │       ├── actionItem.ts                # Action item CRUD
│   │   │       ├── share.ts                     # Sharing management
│   │   │       ├── analytics.ts                 # Meeting analytics queries
│   │   │       ├── glossary.ts                  # Custom glossary management
│   │   │       └── billing.ts                   # Plan + Stripe
│   │   │
│   │   ├── ai/
│   │   │   └── summarize-meeting.ts             # GPT-4o summarization + action item extraction
│   │   │
│   │   ├── services/
│   │   │   ├── meeting-bot.ts                   # Recall.ai bot dispatch + lifecycle
│   │   │   ├── calendar-sync.ts                 # Google Calendar sync + push notifications
│   │   │   ├── search.ts                        # Hybrid search (pgvector + pg_trgm + RRF)
│   │   │   ├── sharing.ts                       # Email + Slack summary sharing
│   │   │   ├── storage.ts                       # R2 recording upload + presigned URLs
│   │   │   └── transcript.ts                    # Transcript normalization
│   │   │
│   │   ├── queue/
│   │   │   ├── connection.ts                    # Redis connection config
│   │   │   ├── bot-queue.ts                     # Bot dispatch queue
│   │   │   ├── bot-worker.ts                    # Bot dispatch worker
│   │   │   ├── processing-queue.ts              # Meeting processing queue
│   │   │   ├── processing-worker.ts             # Transcript → Summary pipeline worker
│   │   │   └── transcription-buffer.ts          # Real-time transcription buffering
│   │   │
│   │   ├── integrations/
│   │   │   └── encryption.ts                    # AES-256-GCM encrypt/decrypt
│   │   │
│   │   ├── billing/
│   │   │   └── stripe.ts                        # Plan definitions, checkout, portal
│   │   │
│   │   ├── email/
│   │   │   └── templates.ts                     # Meeting summary email template
│   │   │
│   │   └── lib/
│   │       └── date-parser.ts                   # Relative date parsing ("by Friday" → Date)
│   │
│   ├── components/
│   │   ├── ui/                                  # Shared Radix UI primitives
│   │   │
│   │   ├── layout/
│   │   │   ├── sidebar.tsx                      # Navigation sidebar
│   │   │   ├── header.tsx                       # Page header
│   │   │   └── org-switcher.tsx                 # Clerk org switcher
│   │   │
│   │   ├── meeting/
│   │   │   ├── meeting-list.tsx                 # Upcoming + past meetings
│   │   │   ├── meeting-card.tsx                 # Meeting card with status
│   │   │   ├── summary-view.tsx                 # Full summary display
│   │   │   ├── transcript-viewer.tsx            # Scrollable transcript with search
│   │   │   ├── audio-player.tsx                 # Meeting audio playback
│   │   │   ├── action-item-list.tsx             # Action items checklist
│   │   │   ├── participant-list.tsx             # Participants with talk time bars
│   │   │   └── share-dialog.tsx                 # Share summary via email/Slack
│   │   │
│   │   ├── search/
│   │   │   ├── search-bar.tsx                   # Search input with instant results
│   │   │   ├── search-results.tsx               # Results list with snippets
│   │   │   └── ai-answer.tsx                    # AI question answering card
│   │   │
│   │   └── analytics/
│   │       ├── meeting-hours-chart.tsx           # Area chart
│   │       ├── action-items-chart.tsx            # Bar chart
│   │       ├── talk-time-chart.tsx               # Horizontal bar chart
│   │       └── stats-cards.tsx                   # Overview metric cards
│   │
│   └── lib/
│       ├── trpc/
│       │   ├── client.ts                        # React Query tRPC client
│       │   └── server.ts                        # Server-side caller
│       ├── utils.ts                             # cn(), formatDuration, formatTime
│       └── constants.ts                         # Meeting type labels, sentiment labels, status colors
│
├── drizzle.config.ts
├── tailwind.config.ts
├── tsconfig.json
├── next.config.ts
├── package.json
└── .env.local
```

---

## Verification

### Manual Testing Checklist

1. **Auth flow**: Sign up → create org → verify DB records
2. **Google Calendar OAuth**: Connect Google account → verify encrypted tokens stored
3. **Calendar sync**: Create a Google Meet event → verify meeting detected and created in DB
4. **Auto-join settings**: Set to "manual" → verify bot does NOT auto-dispatch
5. **Bot dispatch**: Click "Join" on meeting → verify Recall.ai bot created
6. **Bot status tracking**: Start meeting → verify status updates: joining → recording → processing → ready
7. **Transcription**: Have 2+ people speak → verify speaker diarization (speakers labeled)
8. **Transcript storage**: End meeting → verify transcript stored with segments + full_text + embedding
9. **AI summary**: Verify TL;DR, decisions, discussion topics, and sentiment generated
10. **Action items**: Say "I'll send the report by Friday" → verify action item extracted with assignee + due date
11. **Summary view**: Navigate to meeting → verify all sections render correctly
12. **Transcript view**: Click transcript segment → verify audio seeks to correct position
13. **Search — keyword**: Search for a specific term mentioned in meeting → verify result found
14. **Search — semantic**: Search "pricing discussion" → verify relevant meeting returned
15. **AI Q&A**: Ask "What did we decide about the deadline?" → verify answer with citation
16. **Email sharing**: Share summary via email → verify email received with formatted content
17. **Slack sharing**: Share summary to Slack → verify message posted with Block Kit format
18. **Auto-share**: Configure auto-share → end meeting → verify auto-shared to Slack + email
19. **Action item tracker**: Mark action item as done → verify status updated
20. **Plan limits**: On free tier, process 6th meeting → verify blocked with upgrade prompt
21. **Billing**: Upgrade to Pro → verify meetings limit lifted

### Key SQL Queries for Verification

```sql
-- Check meeting processing pipeline
SELECT id, title, status, duration_minutes, recall_bot_id,
       processing_started_at,
       EXTRACT(EPOCH FROM (updated_at - processing_started_at)) AS processing_seconds
FROM meetings
WHERE org_id = 'ORG_ID'
ORDER BY created_at DESC;

-- Check transcript was stored with embedding
SELECT t.id, t.word_count, t.confidence_avg,
       t.embedding IS NOT NULL AS has_embedding,
       jsonb_array_length(t.segments) AS segment_count
FROM transcripts t
JOIN meetings m ON m.id = t.meeting_id
WHERE m.org_id = 'ORG_ID'
ORDER BY t.created_at DESC;

-- Check action items extracted
SELECT ai.text, ai.assignee_name, ai.due_date, ai.due_date_raw,
       ai.priority, ai.status, m.title AS meeting
FROM action_items ai
JOIN meetings m ON m.id = ai.meeting_id
WHERE ai.org_id = 'ORG_ID'
ORDER BY ai.created_at DESC;

-- Check sharing log
SELECT s.channel, s.recipient, s.status, s.sent_at,
       m.title AS meeting,
       s.metadata->>'slackMessageTs' AS slack_ts
FROM shares s
JOIN meetings m ON m.id = s.meeting_id
WHERE s.org_id = 'ORG_ID'
ORDER BY s.created_at DESC;

-- Semantic search test
SELECT m.title, m.started_at,
       1 - (t.embedding <=> (
         SELECT embedding FROM transcripts
         WHERE meeting_id = (SELECT id FROM meetings WHERE org_id = 'ORG_ID' LIMIT 1)
       )) AS similarity
FROM transcripts t
JOIN meetings m ON m.id = t.meeting_id
WHERE m.org_id = 'ORG_ID' AND t.embedding IS NOT NULL
ORDER BY similarity DESC
LIMIT 5;

-- Monthly meeting count (plan enforcement)
SELECT COUNT(*) FROM meetings
WHERE org_id = 'ORG_ID'
  AND created_at >= date_trunc('month', now())
  AND status != 'scheduled';
```

### Performance Benchmarks

| Operation | Target | Method |
|-----------|--------|--------|
| Bot dispatch (Recall.ai API) | < 5s | Single API call |
| Transcription (real-time) | < 1s latency | Deepgram streaming via Recall.ai |
| Transcript finalization | < 30s after meeting end | Recall.ai webhook + DB insert |
| AI summarization (60-min meeting) | < 30s | GPT-4o with 4K max_tokens |
| Embedding generation | < 2s | text-embedding-3-small on truncated text |
| Total processing time (end-to-end) | < 2 min after meeting ends | Bot webhook → transcript → summary → share |
| Semantic search (1000 transcripts) | < 200ms | HNSW index on pgvector |
| Keyword search (pg_trgm) | < 150ms | GIN trigram index |
| Hybrid search (RRF merge) | < 500ms | Parallel semantic + keyword + merge |
| Meeting list page load | < 400ms | Indexed query + React Query cache |
| Summary page load | < 300ms | Single join query |
| Email sharing | < 3s | Resend API |
| Slack sharing | < 2s | Slack Web API |

---

## Post-MVP Roadmap

| ID | Feature | Description | Priority |
|----|---------|-------------|----------|
| F5 | Topic Segmentation | Pre-summarization step that segments transcripts into coherent topics using embedding similarity on sliding windows. Enables "jump to topic" navigation and per-topic summaries. Algorithm already designed in Phase 4. | High |
| F6 | Multi-Language Transcription | Support for non-English meetings via Deepgram's multilingual models. Language auto-detection, mixed-language meeting handling, and translated summaries via GPT-4o. | High |
| F7 | Speaker Coaching Analytics | Per-person analytics: talk-to-listen ratio, interruption frequency, question-asking rate, filler word detection. Private coaching dashboard with trend tracking across meetings. | Medium |
| F8 | Meeting Templates | Customizable summary templates beyond the built-in types. Template editor with configurable extraction rules (e.g., "extract all budget mentions", "list all technical decisions"). Share templates across org. | Medium |
| F9 | CRM / Salesforce Integration | Auto-log meeting summaries and action items to Salesforce opportunity records. Match meeting participants to CRM contacts. Auto-create follow-up tasks in Salesforce. | Medium |
| F10 | Zoom + Microsoft Teams Support | Expand beyond Google Meet to Zoom (via Zoom SDK bot) and Microsoft Teams (via Graph API bot). Unified meeting archive across all platforms. | High |
| F11 | Weekly Digest Email | Automated weekly summary email: meetings attended, total action items created/completed, key decisions across all meetings, upcoming meetings with auto-join status. | Low |
| F12 | Notion / Confluence Export | One-click export of meeting summaries to Notion pages or Confluence spaces. Auto-create structured pages with linked action items. Template mapping per workspace. | Low |
