# 9. OnboardFlow — Employee Onboarding Automation

**MVP Scope:** Visual workflow builder (task nodes, delays, notifications via React Flow) + 3 built-in templates (general, engineering, sales) + task assignment and tracking + email notifications and reminders + new hire portal (token-based, no auth) + progress dashboard + Google Workspace provisioning + Slack integration + billing (Starter $99/mo, Growth $249/mo, Enterprise custom).

---

## Tech Decisions

### Workflow Engine: Cron-Based Task Scheduler on Vercel + QStash (NOT Temporal.io)

**Decision:** Use a simplified cron-based scheduler with Upstash QStash for durable task scheduling instead of Temporal.io.

**Rationale:**
- Temporal.io requires a separate AWS worker cluster, adds ~$50-100/mo infrastructure cost, and introduces significant operational complexity (gRPC, Temporal server, worker processes)
- Onboarding workflows are fundamentally **date-driven state machines**, not complex distributed transactions. Tasks become active on specific dates relative to `start_date`, and status transitions are triggered by user actions (completing tasks) or time passing (overdue checks)
- QStash provides durable HTTP-based scheduling with guaranteed delivery, retries, and dead-letter queues -- exactly what we need for "send reminder at 9am on Day 3" without running persistent workers
- Keeps the entire app deployable on Vercel with zero additional infrastructure
- If we outgrow this, Temporal can be introduced later as the execution layer without changing the data model

**How it works:**
1. When an onboarding is created, compute all absolute dates from relative offsets and `start_date`
2. For each task/notification, schedule a QStash HTTP call to `/api/cron/process-task` with the task ID and scheduled time
3. A Vercel cron job runs every 15 minutes (`/api/cron/check-overdue`) to mark overdue tasks and send reminders
4. Task completion triggers downstream dependency resolution: check if all parent nodes are complete, then activate child nodes
5. Integration actions (Google provisioning, Slack invites) execute immediately when their node activates

```
┌─────────────────────────────────────────────────┐
│  Onboarding Created                             │
│  1. Instantiate all nodes from template         │
│  2. Compute absolute due_dates                  │
│  3. Schedule QStash callbacks for each          │
│  4. Activate nodes with no dependencies         │
└────────────────────┬────────────────────────────┘
                     │
        ┌────────────┼────────────────┐
        ▼            ▼                ▼
   QStash HTTP    Vercel Cron      User Action
   (scheduled     (every 15min)    (task completed)
    activation)                          │
        │            │                   │
        ▼            ▼                   ▼
   Activate Node  Check Overdue    Resolve Dependencies
   Execute Action Send Reminders   Activate Next Nodes
   Send Notif.    Update %         Recompute %
```

### Visual Editor: React Flow (@xyflow/react)

**Decision:** Use `@xyflow/react` v12 for the drag-and-drop workflow canvas.

**Key implementation details:**
- Custom node types registered via `nodeTypes` prop: `TaskNode`, `DelayNode`, `NotificationNode`, `ActionNode`, `ApprovalNode`, `FormNode`
- Each custom node renders a card with icon, title, assignee badge, and due-date offset chip
- Edges represent dependencies (parent_node_id -> child_node_id), rendered as smoothstep edges with animated flow indicators
- Node palette on the left sidebar uses drag-from-panel pattern (HTML5 drag API + `onDrop` on ReactFlow pane)
- Node configuration opens in a right-side drawer on click (not a modal, to maintain canvas context)
- Position data (position_x, position_y) saved to DB on `onNodesChange` with debounced 500ms auto-save
- Parallel paths are visually represented as nodes at the same y-level with no edge between them, both connecting from the same parent

### New Hire Portal: Token-Based Access

**Decision:** Unique URL with cryptographic token, no Clerk auth required.

**Implementation:**
- Generate a `nanoid(32)` token stored in `new_hire_portals.access_token`
- Portal URL: `https://app.onboardflow.com/portal/[access_token]`
- Token validated via DB lookup on every request; portal data fetched in a single tRPC query
- Token expires via `expires_at` column (default: 90 days after start_date)
- No session/cookie -- stateless token validation per request
- New hire can check off their own tasks via a `POST /api/portal/[token]/tasks/[taskId]/complete` endpoint
- Rate limited to 60 requests/minute per token via Upstash Redis

### File Storage: AWS S3

**Decision:** S3 for document attachments (handbooks, signed forms, uploaded files).

- Presigned URL upload flow identical to FormForge pattern
- Files organized by: `onboardflow/{org_id}/{onboarding_id}/{document_id}/{filename}`
- Max file size: 10MB per document, 100MB per onboarding
- Presigned URLs expire after 1 hour for uploads, 24 hours for downloads

### Task Scheduling: Relative-to-Absolute Date Computation

**Algorithm for converting workflow template offsets to concrete dates:**

```typescript
// Offset format in WorkflowNode config:
// { relative_to: "start_date", offset_days: -3 }  // 3 days BEFORE start
// { relative_to: "start_date", offset_days: 0 }    // Day of start
// { relative_to: "start_date", offset_days: 5 }    // 5 business days after
// { relative_to: "start_date", offset_days: 30 }   // 30 days after

function computeDueDate(startDate: Date, offsetDays: number): Date {
  const due = new Date(startDate);
  due.setDate(due.getDate() + offsetDays);
  // If falls on weekend, push to next Monday
  const day = due.getDay();
  if (day === 0) due.setDate(due.getDate() + 1); // Sunday -> Monday
  if (day === 6) due.setDate(due.getDate() + 2); // Saturday -> Monday
  return due;
}
```

### Provisioning

- **Google Admin SDK**: Service account with domain-wide delegation. Scopes: `admin.directory.user`, `admin.directory.group`. Create user via `directory.users.insert`, add to groups via `directory.groups.members.insert`.
- **Slack API**: Bot token with `users:write`, `channels:manage`, `groups:write` scopes. Invite via `admin.users.invite`, channel add via `conversations.invite`.

---

## Database Schema (Drizzle)

```typescript
import { relations } from "drizzle-orm";
import {
  pgTable,
  uuid,
  text,
  boolean,
  integer,
  timestamp,
  date,
  jsonb,
  real,
} from "drizzle-orm/pg-core";

// ---------------------------------------------------------------------------
// Organizations
// ---------------------------------------------------------------------------
export const organizations = pgTable("organizations", {
  id: uuid("id").primaryKey().defaultRandom(),
  clerkOrgId: text("clerk_org_id").unique().notNull(),
  name: text("name").notNull(),
  slug: text("slug").unique().notNull(),
  plan: text("plan").default("starter").notNull(), // starter | growth | enterprise
  stripeCustomerId: text("stripe_customer_id"),
  stripeSubscriptionId: text("stripe_subscription_id"),
  settings: jsonb("settings").default({}).$type<{
    logo?: string;
    primaryColor?: string;
    timezone?: string; // e.g., "America/New_York"
    defaultReminderHour?: number; // 0-23, when to send daily reminders
    portalWelcomeMessage?: string;
  }>(),
  createdAt: timestamp("created_at", { withTimezone: true }).defaultNow().notNull(),
  updatedAt: timestamp("updated_at", { withTimezone: true }).defaultNow().notNull(),
});

// ---------------------------------------------------------------------------
// Organization Members (synced from Clerk)
// ---------------------------------------------------------------------------
export const orgMembers = pgTable("org_members", {
  id: uuid("id").primaryKey().defaultRandom(),
  orgId: uuid("org_id")
    .references(() => organizations.id, { onDelete: "cascade" })
    .notNull(),
  clerkUserId: text("clerk_user_id").notNull(),
  email: text("email").notNull(),
  name: text("name").notNull(),
  role: text("role").default("member").notNull(), // admin | hr | manager | member
  avatarUrl: text("avatar_url"),
  createdAt: timestamp("created_at", { withTimezone: true }).defaultNow().notNull(),
});

// ---------------------------------------------------------------------------
// Workflow Templates
// ---------------------------------------------------------------------------
export type WorkflowTrigger = "hire" | "offboard" | "manual";

export const workflowTemplates = pgTable("workflow_templates", {
  id: uuid("id").primaryKey().defaultRandom(),
  orgId: uuid("org_id")
    .references(() => organizations.id, { onDelete: "cascade" })
    .notNull(),
  name: text("name").notNull(),
  description: text("description"),
  roleType: text("role_type"), // engineering | sales | marketing | general | custom
  trigger: text("trigger").default("hire").notNull().$type<WorkflowTrigger>(),
  isSystemTemplate: boolean("is_system_template").default(false).notNull(),
  isActive: boolean("is_active").default(true).notNull(),
  createdAt: timestamp("created_at", { withTimezone: true }).defaultNow().notNull(),
  updatedAt: timestamp("updated_at", { withTimezone: true }).defaultNow().notNull(),
});

// ---------------------------------------------------------------------------
// Workflow Nodes (template-level definition)
// ---------------------------------------------------------------------------
export type WorkflowNodeType =
  | "task"
  | "action"
  | "delay"
  | "notification"
  | "approval"
  | "form";

export type TaskAssigneeRole = "hr" | "manager" | "it_admin" | "buddy" | "new_hire" | "custom";

export type NodeConfig = {
  // Task nodes
  assigneeRole?: TaskAssigneeRole;
  customAssigneeId?: string;
  instructions?: string; // rich text / markdown
  subTasks?: { text: string }[];
  estimatedMinutes?: number;

  // Timing
  offsetDays?: number; // relative to start_date (-3 = 3 days before)
  relativeTo?: "start_date"; // future: could be "previous_node_complete"

  // Delay nodes
  delayDays?: number;

  // Notification nodes
  recipientRole?: TaskAssigneeRole;
  customRecipientEmail?: string;
  emailSubject?: string;
  emailBody?: string; // supports {{new_hire_name}}, {{start_date}}, {{manager_name}}

  // Action nodes (integrations)
  integrationAction?: "google_create_user" | "google_add_to_group" | "slack_invite"
    | "slack_add_to_channel" | "slack_send_message" | "webhook";
  integrationConfig?: {
    googleGroups?: string[]; // group emails to add user to
    slackChannels?: string[]; // channel IDs to add user to
    slackMessage?: string;
    webhookUrl?: string;
    webhookPayload?: Record<string, unknown>;
  };

  // Form nodes
  formFields?: {
    label: string;
    type: "text" | "select" | "checkbox" | "file";
    required: boolean;
    options?: string[];
  }[];

  // Approval nodes
  approverRole?: TaskAssigneeRole;

  // Phase label for grouping in UI
  phase?: "pre_boarding" | "day_1" | "week_1" | "week_2" | "month_1" | "ongoing";
};

export const workflowNodes = pgTable("workflow_nodes", {
  id: uuid("id").primaryKey().defaultRandom(),
  templateId: uuid("template_id")
    .references(() => workflowTemplates.id, { onDelete: "cascade" })
    .notNull(),
  type: text("type").notNull().$type<WorkflowNodeType>(),
  name: text("name").notNull(),
  config: jsonb("config").default({}).notNull().$type<NodeConfig>(),
  positionX: real("position_x").default(0).notNull(),
  positionY: real("position_y").default(0).notNull(),
  parentNodeIds: jsonb("parent_node_ids").default([]).$type<string[]>(),
  sortOrder: integer("sort_order").default(0).notNull(),
  createdAt: timestamp("created_at", { withTimezone: true }).defaultNow().notNull(),
});

// ---------------------------------------------------------------------------
// Onboardings (live instance per hire)
// ---------------------------------------------------------------------------
export type OnboardingStatus = "pre_boarding" | "active" | "completed" | "paused" | "cancelled";

export const onboardings = pgTable("onboardings", {
  id: uuid("id").primaryKey().defaultRandom(),
  orgId: uuid("org_id")
    .references(() => organizations.id, { onDelete: "cascade" })
    .notNull(),
  templateId: uuid("template_id")
    .references(() => workflowTemplates.id, { onDelete: "set null" }),
  // New hire info
  newHireName: text("new_hire_name").notNull(),
  newHireEmail: text("new_hire_email").notNull(),
  newHireRole: text("new_hire_role"), // "Senior Engineer", "Account Executive"
  department: text("department"),
  startDate: date("start_date").notNull(), // DATE type, no time component
  // Assignees
  managerMemberId: uuid("manager_member_id")
    .references(() => orgMembers.id, { onDelete: "set null" }),
  buddyMemberId: uuid("buddy_member_id")
    .references(() => orgMembers.id, { onDelete: "set null" }),
  // Status
  status: text("status").default("pre_boarding").notNull().$type<OnboardingStatus>(),
  completionPercentage: real("completion_percentage").default(0).notNull(),
  // Metadata
  createdAt: timestamp("created_at", { withTimezone: true }).defaultNow().notNull(),
  updatedAt: timestamp("updated_at", { withTimezone: true }).defaultNow().notNull(),
  completedAt: timestamp("completed_at", { withTimezone: true }),
});

// ---------------------------------------------------------------------------
// Onboarding Tasks (live instance of workflow nodes)
// ---------------------------------------------------------------------------
export type TaskStatus = "pending" | "active" | "in_progress" | "completed" | "skipped" | "overdue" | "blocked";

export type SubTaskItem = { text: string; completed: boolean };

export const onboardingTasks = pgTable("onboarding_tasks", {
  id: uuid("id").primaryKey().defaultRandom(),
  onboardingId: uuid("onboarding_id")
    .references(() => onboardings.id, { onDelete: "cascade" })
    .notNull(),
  nodeId: uuid("node_id")
    .references(() => workflowNodes.id, { onDelete: "set null" }),
  nodeType: text("node_type").notNull().$type<WorkflowNodeType>(),
  name: text("name").notNull(),
  config: jsonb("config").notNull().$type<NodeConfig>(), // snapshot of node config at creation time
  // Assignment
  assignedToMemberId: uuid("assigned_to_member_id")
    .references(() => orgMembers.id, { onDelete: "set null" }),
  assignedToEmail: text("assigned_to_email"), // for new_hire tasks or external
  // Status lifecycle
  status: text("status").default("pending").notNull().$type<TaskStatus>(),
  dueDate: date("due_date"), // absolute date, computed from start_date + offset
  activatedAt: timestamp("activated_at", { withTimezone: true }),
  completedAt: timestamp("completed_at", { withTimezone: true }),
  completedByMemberId: uuid("completed_by_member_id")
    .references(() => orgMembers.id, { onDelete: "set null" }),
  // Content
  notes: text("notes"),
  subTasks: jsonb("sub_tasks").default([]).$type<SubTaskItem[]>(),
  parentTaskIds: jsonb("parent_task_ids").default([]).$type<string[]>(),
  // Integration execution result
  executionResult: jsonb("execution_result").$type<{
    success: boolean;
    message?: string;
    executedAt?: string;
    retryCount?: number;
  }>(),
  phase: text("phase").default("day_1"), // pre_boarding | day_1 | week_1 | week_2 | month_1
  sortOrder: integer("sort_order").default(0).notNull(),
  createdAt: timestamp("created_at", { withTimezone: true }).defaultNow().notNull(),
});

// ---------------------------------------------------------------------------
// Integrations (org-level credentials)
// ---------------------------------------------------------------------------
export type IntegrationType = "google_workspace" | "slack";

export const integrations = pgTable("integrations", {
  id: uuid("id").primaryKey().defaultRandom(),
  orgId: uuid("org_id")
    .references(() => organizations.id, { onDelete: "cascade" })
    .notNull(),
  type: text("type").notNull().$type<IntegrationType>(),
  credentials: text("credentials").notNull(), // AES-256-GCM encrypted JSON string
  status: text("status").default("connected").notNull(), // connected | error | disconnected
  metadata: jsonb("metadata").$type<{
    googleDomain?: string;
    googleAdminEmail?: string;
    slackTeamId?: string;
    slackTeamName?: string;
    slackBotUserId?: string;
  }>(),
  lastSyncedAt: timestamp("last_synced_at", { withTimezone: true }),
  createdAt: timestamp("created_at", { withTimezone: true }).defaultNow().notNull(),
});

// ---------------------------------------------------------------------------
// New Hire Portals
// ---------------------------------------------------------------------------
export type PortalResource = {
  title: string;
  url?: string;
  description?: string;
  type: "link" | "document" | "video";
};

export const newHirePortals = pgTable("new_hire_portals", {
  id: uuid("id").primaryKey().defaultRandom(),
  onboardingId: uuid("onboarding_id")
    .references(() => onboardings.id, { onDelete: "cascade" })
    .notNull()
    .unique(),
  accessToken: text("access_token").unique().notNull(), // nanoid(32)
  welcomeMessage: text("welcome_message"), // markdown
  resources: jsonb("resources").default([]).$type<PortalResource[]>(),
  branding: jsonb("branding").$type<{
    logoUrl?: string;
    primaryColor?: string;
    companyName?: string;
    bannerImageUrl?: string;
  }>(),
  expiresAt: timestamp("expires_at", { withTimezone: true }),
  lastAccessedAt: timestamp("last_accessed_at", { withTimezone: true }),
  createdAt: timestamp("created_at", { withTimezone: true }).defaultNow().notNull(),
});

// ---------------------------------------------------------------------------
// Documents
// ---------------------------------------------------------------------------
export const documents = pgTable("documents", {
  id: uuid("id").primaryKey().defaultRandom(),
  onboardingId: uuid("onboarding_id")
    .references(() => onboardings.id, { onDelete: "cascade" })
    .notNull(),
  taskId: uuid("task_id")
    .references(() => onboardingTasks.id, { onDelete: "set null" }),
  name: text("name").notNull(),
  type: text("type").notNull(), // handbook | policy | form | signed_document | attachment
  fileUrl: text("file_url").notNull(), // S3 key
  fileSize: integer("file_size"), // bytes
  mimeType: text("mime_type"),
  requiresSignature: boolean("requires_signature").default(false).notNull(),
  signedAt: timestamp("signed_at", { withTimezone: true }),
  signedByEmail: text("signed_by_email"),
  uploadedAt: timestamp("uploaded_at", { withTimezone: true }).defaultNow().notNull(),
});

// ---------------------------------------------------------------------------
// Activity Log (audit trail)
// ---------------------------------------------------------------------------
export const activityLogs = pgTable("activity_logs", {
  id: uuid("id").primaryKey().defaultRandom(),
  orgId: uuid("org_id")
    .references(() => organizations.id, { onDelete: "cascade" })
    .notNull(),
  onboardingId: uuid("onboarding_id")
    .references(() => onboardings.id, { onDelete: "cascade" }),
  actorMemberId: uuid("actor_member_id")
    .references(() => orgMembers.id, { onDelete: "set null" }),
  actorEmail: text("actor_email"), // for new_hire actions via portal
  action: text("action").notNull(), // task_completed | task_assigned | onboarding_created | integration_executed | reminder_sent | ...
  targetType: text("target_type"), // onboarding | task | integration
  targetId: text("target_id"),
  metadata: jsonb("metadata").$type<Record<string, unknown>>(),
  createdAt: timestamp("created_at", { withTimezone: true }).defaultNow().notNull(),
});

// ---------------------------------------------------------------------------
// Relations
// ---------------------------------------------------------------------------
export const organizationsRelations = relations(organizations, ({ many }) => ({
  members: many(orgMembers),
  workflowTemplates: many(workflowTemplates),
  onboardings: many(onboardings),
  integrations: many(integrations),
  activityLogs: many(activityLogs),
}));

export const orgMembersRelations = relations(orgMembers, ({ one }) => ({
  organization: one(organizations, {
    fields: [orgMembers.orgId],
    references: [organizations.id],
  }),
}));

export const workflowTemplatesRelations = relations(workflowTemplates, ({ one, many }) => ({
  organization: one(organizations, {
    fields: [workflowTemplates.orgId],
    references: [organizations.id],
  }),
  nodes: many(workflowNodes),
  onboardings: many(onboardings),
}));

export const workflowNodesRelations = relations(workflowNodes, ({ one }) => ({
  template: one(workflowTemplates, {
    fields: [workflowNodes.templateId],
    references: [workflowTemplates.id],
  }),
}));

export const onboardingsRelations = relations(onboardings, ({ one, many }) => ({
  organization: one(organizations, {
    fields: [onboardings.orgId],
    references: [organizations.id],
  }),
  template: one(workflowTemplates, {
    fields: [onboardings.templateId],
    references: [workflowTemplates.id],
  }),
  manager: one(orgMembers, {
    fields: [onboardings.managerMemberId],
    references: [orgMembers.id],
    relationName: "manager",
  }),
  buddy: one(orgMembers, {
    fields: [onboardings.buddyMemberId],
    references: [orgMembers.id],
    relationName: "buddy",
  }),
  tasks: many(onboardingTasks),
  portal: many(newHirePortals),
  documents: many(documents),
  activityLogs: many(activityLogs),
}));

export const onboardingTasksRelations = relations(onboardingTasks, ({ one }) => ({
  onboarding: one(onboardings, {
    fields: [onboardingTasks.onboardingId],
    references: [onboardings.id],
  }),
  node: one(workflowNodes, {
    fields: [onboardingTasks.nodeId],
    references: [workflowNodes.id],
  }),
  assignedTo: one(orgMembers, {
    fields: [onboardingTasks.assignedToMemberId],
    references: [orgMembers.id],
    relationName: "assignedTo",
  }),
  completedBy: one(orgMembers, {
    fields: [onboardingTasks.completedByMemberId],
    references: [orgMembers.id],
    relationName: "completedBy",
  }),
}));

export const newHirePortalsRelations = relations(newHirePortals, ({ one }) => ({
  onboarding: one(onboardings, {
    fields: [newHirePortals.onboardingId],
    references: [onboardings.id],
  }),
}));

export const documentsRelations = relations(documents, ({ one }) => ({
  onboarding: one(onboardings, {
    fields: [documents.onboardingId],
    references: [onboardings.id],
  }),
  task: one(onboardingTasks, {
    fields: [documents.taskId],
    references: [onboardingTasks.id],
  }),
}));

export const integrationsRelations = relations(integrations, ({ one }) => ({
  organization: one(organizations, {
    fields: [integrations.orgId],
    references: [organizations.id],
  }),
}));

export const activityLogsRelations = relations(activityLogs, ({ one }) => ({
  organization: one(organizations, {
    fields: [activityLogs.orgId],
    references: [organizations.id],
  }),
  onboarding: one(onboardings, {
    fields: [activityLogs.onboardingId],
    references: [onboardings.id],
  }),
  actor: one(orgMembers, {
    fields: [activityLogs.actorMemberId],
    references: [orgMembers.id],
  }),
}));

// ---------------------------------------------------------------------------
// Type helpers
// ---------------------------------------------------------------------------
export type Organization = typeof organizations.$inferSelect;
export type NewOrganization = typeof organizations.$inferInsert;
export type OrgMember = typeof orgMembers.$inferSelect;
export type WorkflowTemplate = typeof workflowTemplates.$inferSelect;
export type WorkflowNode = typeof workflowNodes.$inferSelect;
export type Onboarding = typeof onboardings.$inferSelect;
export type OnboardingTask = typeof onboardingTasks.$inferSelect;
export type Integration = typeof integrations.$inferSelect;
export type NewHirePortal = typeof newHirePortals.$inferSelect;
export type Document = typeof documents.$inferSelect;
export type ActivityLog = typeof activityLogs.$inferSelect;
```

---

## React Flow Node Type Definitions

```typescript
// src/lib/workflow/node-types.ts

import type { Node, NodeProps } from "@xyflow/react";
import type { NodeConfig, WorkflowNodeType } from "@/server/db/schema";

// ---------------------------------------------------------------------------
// Node data structure shared across all custom nodes
// ---------------------------------------------------------------------------
export type WorkflowNodeData = {
  label: string;
  nodeType: WorkflowNodeType;
  config: NodeConfig;
  isSelected?: boolean;
  isValid?: boolean; // false if required config is missing
};

// ---------------------------------------------------------------------------
// React Flow typed nodes
// ---------------------------------------------------------------------------
export type TaskFlowNode = Node<WorkflowNodeData, "task">;
export type DelayFlowNode = Node<WorkflowNodeData, "delay">;
export type NotificationFlowNode = Node<WorkflowNodeData, "notification">;
export type ActionFlowNode = Node<WorkflowNodeData, "action">;
export type ApprovalFlowNode = Node<WorkflowNodeData, "approval">;
export type FormFlowNode = Node<WorkflowNodeData, "form">;

export type AppNode =
  | TaskFlowNode
  | DelayFlowNode
  | NotificationFlowNode
  | ActionFlowNode
  | ApprovalFlowNode
  | FormFlowNode;

// ---------------------------------------------------------------------------
// Node palette definitions (for the left sidebar drag source)
// ---------------------------------------------------------------------------
export const NODE_PALETTE = [
  {
    type: "task" as const,
    label: "Task",
    description: "Assign a task to someone",
    icon: "CheckSquare", // lucide icon name
    color: "#3b82f6", // blue
    defaultConfig: {
      assigneeRole: "manager" as const,
      offsetDays: 0,
      relativeTo: "start_date" as const,
      phase: "day_1" as const,
    },
  },
  {
    type: "delay" as const,
    label: "Wait / Delay",
    description: "Pause workflow for N days",
    icon: "Clock",
    color: "#f59e0b", // amber
    defaultConfig: {
      delayDays: 1,
    },
  },
  {
    type: "notification" as const,
    label: "Send Notification",
    description: "Send email or Slack message",
    icon: "Bell",
    color: "#8b5cf6", // violet
    defaultConfig: {
      recipientRole: "manager" as const,
      emailSubject: "",
      emailBody: "",
    },
  },
  {
    type: "action" as const,
    label: "Integration Action",
    description: "Google, Slack, or webhook",
    icon: "Zap",
    color: "#10b981", // emerald
    defaultConfig: {
      integrationAction: "google_create_user" as const,
    },
  },
  {
    type: "approval" as const,
    label: "Approval Gate",
    description: "Require sign-off to proceed",
    icon: "ShieldCheck",
    color: "#ef4444", // red
    defaultConfig: {
      approverRole: "hr" as const,
    },
  },
  {
    type: "form" as const,
    label: "Collect Info",
    description: "Form for the new hire to fill",
    icon: "FileText",
    color: "#06b6d4", // cyan
    defaultConfig: {
      formFields: [],
    },
  },
] as const;

// ---------------------------------------------------------------------------
// Registration map for React Flow
// ---------------------------------------------------------------------------
// In the actual component file, each custom node is a React component:
//
//   import { TaskNodeComponent } from "./nodes/TaskNode";
//   import { DelayNodeComponent } from "./nodes/DelayNode";
//   ...
//
//   export const nodeTypes = {
//     task: TaskNodeComponent,
//     delay: DelayNodeComponent,
//     notification: NotificationNodeComponent,
//     action: ActionNodeComponent,
//     approval: ApprovalNodeComponent,
//     form: FormNodeComponent,
//   };
```

---

## Template Instantiation Process (Template -> Live Onboarding)

```typescript
// src/server/workflow/instantiate.ts

/**
 * Creates a live onboarding from a workflow template.
 *
 * Steps:
 * 1. Clone all WorkflowNodes into OnboardingTasks
 * 2. Compute absolute due_dates from start_date + offset
 * 3. Resolve assignee roles to actual org members
 * 4. Create the NewHirePortal with access token
 * 5. Schedule QStash callbacks for time-based activations
 * 6. Activate any root nodes (no parent dependencies)
 * 7. Log the activity
 */
export async function instantiateOnboarding(params: {
  orgId: string;
  templateId: string;
  newHireName: string;
  newHireEmail: string;
  newHireRole: string;
  department: string;
  startDate: string; // ISO date YYYY-MM-DD
  managerMemberId: string;
  buddyMemberId?: string;
}): Promise<{ onboardingId: string; portalUrl: string }> {
  // 1. Fetch template nodes
  const nodes = await db
    .select()
    .from(workflowNodes)
    .where(eq(workflowNodes.templateId, params.templateId))
    .orderBy(asc(workflowNodes.sortOrder));

  // 2. Create onboarding record
  const [onboarding] = await db.insert(onboardings).values({
    orgId: params.orgId,
    templateId: params.templateId,
    newHireName: params.newHireName,
    newHireEmail: params.newHireEmail,
    newHireRole: params.newHireRole,
    department: params.department,
    startDate: params.startDate,
    managerMemberId: params.managerMemberId,
    buddyMemberId: params.buddyMemberId,
    status: "pre_boarding",
    completionPercentage: 0,
  }).returning();

  // 3. Map template node IDs -> task IDs for dependency resolution
  const nodeIdToTaskId = new Map<string, string>();

  // 4. Create tasks from nodes
  const startDate = new Date(params.startDate);
  const tasksToInsert = nodes.map((node) => {
    const taskId = crypto.randomUUID();
    nodeIdToTaskId.set(node.id, taskId);

    const dueDate = node.config.offsetDays !== undefined
      ? computeDueDate(startDate, node.config.offsetDays)
      : null;

    const assignedToMemberId = resolveAssignee(
      node.config.assigneeRole ?? node.config.recipientRole ?? node.config.approverRole,
      params.managerMemberId,
      params.buddyMemberId,
    );

    return {
      id: taskId,
      onboardingId: onboarding.id,
      nodeId: node.id,
      nodeType: node.type,
      name: node.name,
      config: node.config, // snapshot
      assignedToMemberId:
        node.config.assigneeRole === "new_hire" ? null : assignedToMemberId,
      assignedToEmail:
        node.config.assigneeRole === "new_hire" ? params.newHireEmail : null,
      status: "pending" as const,
      dueDate: dueDate?.toISOString().split("T")[0] ?? null,
      subTasks: (node.config.subTasks ?? []).map((st) => ({
        text: st.text,
        completed: false,
      })),
      parentTaskIds: [], // filled in next step
      phase: node.config.phase ?? "day_1",
      sortOrder: node.sortOrder,
    };
  });

  // 5. Resolve parent task IDs (template node IDs -> task IDs)
  for (const task of tasksToInsert) {
    const originalNode = nodes.find((n) => nodeIdToTaskId.get(n.id) === task.id);
    if (originalNode?.parentNodeIds?.length) {
      task.parentTaskIds = originalNode.parentNodeIds
        .map((parentNodeId) => nodeIdToTaskId.get(parentNodeId))
        .filter(Boolean) as string[];
    }
  }

  await db.insert(onboardingTasks).values(tasksToInsert);

  // 6. Create portal
  const portalToken = nanoid(32);
  await db.insert(newHirePortals).values({
    onboardingId: onboarding.id,
    accessToken: portalToken,
    welcomeMessage: `Welcome to the team, ${params.newHireName}!`,
    resources: [],
    expiresAt: new Date(startDate.getTime() + 90 * 24 * 60 * 60 * 1000), // 90 days
  });

  // 7. Activate root tasks (no parent dependencies)
  const rootTasks = tasksToInsert.filter((t) => t.parentTaskIds.length === 0);
  for (const task of rootTasks) {
    await activateTask(task.id, onboarding.id);
  }

  // 8. Schedule QStash callbacks for time-based tasks
  for (const task of tasksToInsert) {
    if (task.dueDate) {
      await scheduleTaskReminder(task.id, new Date(task.dueDate));
    }
  }

  // 9. Send welcome email to new hire with portal link
  await sendNewHireWelcomeEmail({
    to: params.newHireEmail,
    name: params.newHireName,
    portalUrl: `${env.NEXT_PUBLIC_APP_URL}/portal/${portalToken}`,
    startDate: params.startDate,
  });

  return {
    onboardingId: onboarding.id,
    portalUrl: `${env.NEXT_PUBLIC_APP_URL}/portal/${portalToken}`,
  };
}

function resolveAssignee(
  role: TaskAssigneeRole | undefined,
  managerMemberId: string,
  buddyMemberId?: string,
): string | null {
  switch (role) {
    case "manager": return managerMemberId;
    case "buddy": return buddyMemberId ?? null;
    case "new_hire": return null; // handled separately
    case "hr": return null; // resolved at org level from orgMembers with role="hr"
    case "it_admin": return null; // resolved at org level from orgMembers with role="admin"
    default: return null;
  }
}
```

---

## Workflow Execution Engine Logic

### How Nodes Trigger in Sequence and Parallel

```typescript
// src/server/workflow/engine.ts

/**
 * Core execution engine. Called when:
 * - A task is marked complete
 * - A delay timer fires
 * - An approval is granted
 * - A cron job detects it's time to activate a pending task
 */

// --- Task Completion Handler ---
export async function completeTask(
  taskId: string,
  completedByMemberId: string | null, // null if completed by new hire via portal
  completedByEmail?: string,
) {
  // 1. Update task status
  await db.update(onboardingTasks)
    .set({
      status: "completed",
      completedAt: new Date(),
      completedByMemberId,
    })
    .where(eq(onboardingTasks.id, taskId));

  const [task] = await db.select().from(onboardingTasks).where(eq(onboardingTasks.id, taskId));

  // 2. Log activity
  await logActivity({
    orgId: task.onboardingId, // resolved via join
    onboardingId: task.onboardingId,
    actorMemberId: completedByMemberId,
    actorEmail: completedByEmail,
    action: "task_completed",
    targetType: "task",
    targetId: taskId,
  });

  // 3. Resolve downstream dependencies
  await resolveDownstreamTasks(task.onboardingId, taskId);

  // 4. Recompute completion percentage
  await recomputeCompletionPercentage(task.onboardingId);
}

// --- Dependency Resolution ---
async function resolveDownstreamTasks(onboardingId: string, completedTaskId: string) {
  // Find all tasks in this onboarding that have completedTaskId as a parent
  const allTasks = await db.select().from(onboardingTasks)
    .where(eq(onboardingTasks.onboardingId, onboardingId));

  const childTasks = allTasks.filter((t) =>
    (t.parentTaskIds as string[]).includes(completedTaskId) && t.status === "pending"
  );

  for (const child of childTasks) {
    // Check if ALL parents are completed
    const allParentsComplete = (child.parentTaskIds as string[]).every((parentId) => {
      const parent = allTasks.find((t) => t.id === parentId);
      return parent?.status === "completed" || parent?.status === "skipped";
    });

    if (allParentsComplete) {
      await activateTask(child.id, onboardingId);
    }
  }
}

// --- Task Activation ---
async function activateTask(taskId: string, onboardingId: string) {
  const [task] = await db.select().from(onboardingTasks)
    .where(eq(onboardingTasks.id, taskId));

  if (!task || task.status !== "pending") return;

  const config = task.config as NodeConfig;

  switch (task.nodeType) {
    case "task":
    case "form":
    case "approval":
      // Human tasks: set to active, send notification to assignee
      await db.update(onboardingTasks)
        .set({ status: "active", activatedAt: new Date() })
        .where(eq(onboardingTasks.id, taskId));
      await sendTaskAssignedEmail(task);
      break;

    case "action":
      // Integration actions: execute immediately, then auto-complete
      await db.update(onboardingTasks)
        .set({ status: "in_progress", activatedAt: new Date() })
        .where(eq(onboardingTasks.id, taskId));
      const result = await executeIntegrationAction(task);
      await db.update(onboardingTasks)
        .set({
          status: result.success ? "completed" : "overdue",
          completedAt: result.success ? new Date() : null,
          executionResult: result,
        })
        .where(eq(onboardingTasks.id, taskId));
      if (result.success) {
        await resolveDownstreamTasks(onboardingId, taskId);
        await recomputeCompletionPercentage(onboardingId);
      }
      break;

    case "notification":
      // Send notification immediately, then auto-complete
      await db.update(onboardingTasks)
        .set({ status: "in_progress", activatedAt: new Date() })
        .where(eq(onboardingTasks.id, taskId));
      await sendWorkflowNotification(task);
      await db.update(onboardingTasks)
        .set({ status: "completed", completedAt: new Date() })
        .where(eq(onboardingTasks.id, taskId));
      await resolveDownstreamTasks(onboardingId, taskId);
      await recomputeCompletionPercentage(onboardingId);
      break;

    case "delay":
      // Schedule activation of downstream tasks after delay period
      const delayDays = config.delayDays ?? 1;
      const activateAt = new Date();
      activateAt.setDate(activateAt.getDate() + delayDays);
      await db.update(onboardingTasks)
        .set({ status: "in_progress", activatedAt: new Date() })
        .where(eq(onboardingTasks.id, taskId));
      await scheduleQStashCallback({
        url: `${env.NEXT_PUBLIC_APP_URL}/api/cron/delay-complete`,
        body: { taskId, onboardingId },
        notBefore: activateAt,
      });
      break;
  }
}

// --- Completion Percentage ---
async function recomputeCompletionPercentage(onboardingId: string) {
  const tasks = await db.select().from(onboardingTasks)
    .where(eq(onboardingTasks.onboardingId, onboardingId));

  // Only count completable tasks (not delays or notifications, which auto-complete)
  const completable = tasks.filter((t) =>
    ["task", "form", "approval", "action"].includes(t.nodeType)
  );
  const completed = completable.filter((t) =>
    t.status === "completed" || t.status === "skipped"
  );

  const percentage = completable.length > 0
    ? Math.round((completed.length / completable.length) * 100)
    : 0;

  await db.update(onboardings)
    .set({
      completionPercentage: percentage,
      updatedAt: new Date(),
      ...(percentage === 100
        ? { status: "completed", completedAt: new Date() }
        : {}),
    })
    .where(eq(onboardings.id, onboardingId));
}
```

---

## Google Admin SDK Provisioning Steps

```typescript
// src/server/integrations/google.ts

import { google } from "googleapis";
import { decryptCredentials } from "@/server/lib/encryption";

/**
 * Google Workspace provisioning flow:
 *
 * Prerequisites:
 * 1. Google Cloud project with Admin SDK enabled
 * 2. Service account with domain-wide delegation
 * 3. Service account granted scopes in Google Workspace Admin:
 *    - https://www.googleapis.com/auth/admin.directory.user
 *    - https://www.googleapis.com/auth/admin.directory.group
 * 4. Admin email stored in integration metadata (needed for impersonation)
 *
 * OAuth setup (done once per org during integration connect):
 * - Admin authorizes domain-wide delegation for our service account
 * - We store the service account credentials (encrypted) in integrations table
 */

export async function provisionGoogleWorkspaceUser(params: {
  integration: Integration;
  newHireEmail: string; // desired email, e.g., "jane.doe@company.com"
  firstName: string;
  lastName: string;
  department?: string;
  title?: string;
  managerEmail?: string;
  orgUnit?: string; // e.g., "/Engineering"
  groups?: string[]; // group email addresses to add user to
}): Promise<{ success: boolean; message: string }> {
  try {
    const creds = decryptCredentials(params.integration.credentials);
    const metadata = params.integration.metadata;

    // Step 1: Authenticate via service account with domain-wide delegation
    const auth = new google.auth.JWT({
      email: creds.client_email,
      key: creds.private_key,
      scopes: [
        "https://www.googleapis.com/auth/admin.directory.user",
        "https://www.googleapis.com/auth/admin.directory.group",
      ],
      subject: metadata?.googleAdminEmail, // impersonate the admin
    });

    const admin = google.admin({ version: "directory_v1", auth });

    // Step 2: Create user account
    const tempPassword = generateSecurePassword(); // 16-char random
    await admin.users.insert({
      requestBody: {
        primaryEmail: params.newHireEmail,
        name: {
          givenName: params.firstName,
          familyName: params.lastName,
        },
        password: tempPassword,
        changePasswordAtNextLogin: true,
        orgUnitPath: params.orgUnit ?? "/",
        organizations: params.department
          ? [{ department: params.department, title: params.title }]
          : undefined,
        relations: params.managerEmail
          ? [{ value: params.managerEmail, type: "manager" }]
          : undefined,
      },
    });

    // Step 3: Add user to groups
    if (params.groups?.length) {
      for (const groupEmail of params.groups) {
        try {
          await admin.members.insert({
            groupKey: groupEmail,
            requestBody: {
              email: params.newHireEmail,
              role: "MEMBER",
            },
          });
        } catch (groupErr: any) {
          // Log but don't fail the whole operation if one group add fails
          console.error(`Failed to add ${params.newHireEmail} to group ${groupEmail}:`, groupErr.message);
        }
      }
    }

    // Step 4: Return result (temp password sent separately or via portal)
    return {
      success: true,
      message: `Created ${params.newHireEmail} in Google Workspace. Added to ${params.groups?.length ?? 0} groups. Temporary password set (user must change on first login).`,
    };
  } catch (err: any) {
    return {
      success: false,
      message: `Google provisioning failed: ${err.message}`,
    };
  }
}
```

---

## Slack Integration

```typescript
// src/server/integrations/slack.ts

import { WebClient } from "@slack/web-api";
import { decryptCredentials } from "@/server/lib/encryption";

/**
 * Slack integration flow:
 *
 * Setup (done once per org):
 * 1. User installs OnboardFlow Slack app via OAuth
 * 2. Bot token stored encrypted in integrations table
 * 3. Bot needs scopes: channels:manage, groups:write, chat:write,
 *    users:read, users:read.email, admin.users:write (for Enterprise Grid invite)
 *
 * Per-onboarding actions:
 * - Invite new hire to workspace (if not already a member)
 * - Add to specified channels
 * - Send welcome message to a channel or DM
 */

export async function executeSlackAction(params: {
  integration: Integration;
  action: "slack_invite" | "slack_add_to_channel" | "slack_send_message";
  newHireEmail: string;
  newHireName: string;
  channels?: string[]; // channel IDs
  message?: string; // for send_message action
}): Promise<{ success: boolean; message: string }> {
  const creds = decryptCredentials(params.integration.credentials);
  const slack = new WebClient(creds.bot_token);

  try {
    switch (params.action) {
      case "slack_invite": {
        // Invite user to workspace via admin API (requires admin.users:write)
        // NOTE: This only works for Slack Enterprise Grid or Business+ plans.
        // For standard plans, the user must be manually invited or use email invite.
        try {
          await slack.admin.users.invite({
            team_id: params.integration.metadata?.slackTeamId!,
            email: params.newHireEmail,
            channel_ids: params.channels?.join(",") ?? "",
          });
          return { success: true, message: `Invited ${params.newHireEmail} to Slack workspace` };
        } catch (inviteErr: any) {
          if (inviteErr.data?.error === "already_in_team") {
            return { success: true, message: `${params.newHireEmail} is already in the Slack workspace` };
          }
          throw inviteErr;
        }
      }

      case "slack_add_to_channel": {
        // Look up user by email
        const userResp = await slack.users.lookupByEmail({ email: params.newHireEmail });
        const userId = userResp.user?.id;
        if (!userId) {
          return { success: false, message: `Could not find Slack user for ${params.newHireEmail}` };
        }

        // Add to each channel
        const results: string[] = [];
        for (const channelId of params.channels ?? []) {
          try {
            await slack.conversations.invite({ channel: channelId, users: userId });
            results.push(`Added to ${channelId}`);
          } catch (chErr: any) {
            if (chErr.data?.error === "already_in_channel") {
              results.push(`Already in ${channelId}`);
            } else {
              results.push(`Failed ${channelId}: ${chErr.data?.error}`);
            }
          }
        }
        return { success: true, message: results.join("; ") };
      }

      case "slack_send_message": {
        // Send a message to a channel (first channel in list, or DM to user)
        const channelId = params.channels?.[0];
        if (!channelId) {
          return { success: false, message: "No channel specified for message" };
        }

        const message = (params.message ?? "")
          .replace(/\{\{new_hire_name\}\}/g, params.newHireName)
          .replace(/\{\{new_hire_email\}\}/g, params.newHireEmail);

        await slack.chat.postMessage({ channel: channelId, text: message });
        return { success: true, message: `Sent message to channel ${channelId}` };
      }

      default:
        return { success: false, message: `Unknown Slack action: ${params.action}` };
    }
  } catch (err: any) {
    return { success: false, message: `Slack action failed: ${err.message}` };
  }
}
```

---

## Phase Breakdown (10 weeks) -- Day-by-Day

### Phase 1: Foundation (Week 1-2)

**Week 1 -- Project Setup + Data Model + Auth**

| Day | Tasks |
|-----|-------|
| Day 1 | `npx create-next-app@latest onboardflow --ts --tailwind --app --src-dir`. Install core deps: `@clerk/nextjs`, `drizzle-orm`, `@neondatabase/serverless`, `@trpc/server`, `@trpc/client`, `@trpc/react-query`, `@tanstack/react-query`, `stripe`, `resend`, `zod`, `superjson`, `nanoid`, `lucide-react`. Create Neon database. Configure `drizzle.config.ts`, `src/server/db/index.ts`, `.env.example`, `src/env.ts` (Zod validation). Set up Clerk app with organization support enabled. |
| Day 2 | Write full Drizzle schema in `src/server/db/schema.ts`: all 9 tables (organizations, orgMembers, workflowTemplates, workflowNodes, onboardings, onboardingTasks, integrations, newHirePortals, documents) + activityLogs. Run `npx drizzle-kit push`. Verify all tables exist in Neon. |
| Day 3 | Set up tRPC: `src/server/trpc/trpc.ts` (publicProcedure, protectedProcedure, orgProcedure with Clerk org resolution), `src/server/trpc/context.ts` (DB + auth context), `src/server/trpc/routers/_app.ts` (root router). Wire up `src/app/api/trpc/[trpc]/route.ts`. Set up client: `src/lib/trpc/client.ts` (React Query provider), `src/lib/trpc/server.ts` (server-side caller). |
| Day 4 | tRPC router: `src/server/trpc/routers/organization.ts` -- getOrg, updateSettings, syncMembers (from Clerk). Router: `src/server/trpc/routers/member.ts` -- list, updateRole. Build `src/middleware.ts` for Clerk auth + org redirect. Create `src/app/(dashboard)/layout.tsx` with sidebar shell. |
| Day 5 | Build base UI components: sidebar navigation (`src/components/dashboard/sidebar.tsx`), top bar with org switcher, empty state cards. Install and configure shadcn/ui-style primitives: `src/components/ui/button.tsx`, `card.tsx`, `dialog.tsx`, `input.tsx`, `select.tsx`, `badge.tsx`, `toast.tsx`, `dropdown-menu.tsx`, `tabs.tsx`, `progress.tsx`. |

**Week 2 -- Workflow Template CRUD + Built-in Templates**

| Day | Tasks |
|-----|-------|
| Day 6 | tRPC router: `src/server/trpc/routers/workflow-template.ts` -- create, list, getById, update, delete, duplicate. Include full node tree in getById response. Write Zod schemas for template input validation. |
| Day 7 | tRPC router: `src/server/trpc/routers/workflow-node.ts` -- create, update, delete, updatePositions (batch position update for React Flow drag), updateParents (edge creation/deletion). |
| Day 8 | Build template list page `src/app/(dashboard)/workflows/page.tsx`: card grid showing template name, role type, node count, active/inactive toggle. "Create Workflow" button opens dialog with name, description, role_type. |
| Day 9 | Create the 3 built-in system templates as seed data in `src/server/db/seed-templates.ts`. **General template** (15 nodes): Welcome email -> Order laptop -> Set up desk -> Create accounts -> Day 1 orientation -> HR paperwork -> Team intro meeting -> Buddy assignment -> Week 1 check-in -> Set 30-day goals -> Week 2 review -> IT setup verification -> Benefits enrollment -> Month 1 review -> Probation complete. **Engineering template** (20 nodes): General + Dev environment setup -> GitHub access -> Architecture overview -> First PR -> Code review walkthrough -> CI/CD training -> On-call rotation intro. **Sales template** (18 nodes): General + CRM access -> Product training -> Territory assignment -> Shadow calls -> First solo call -> Pipeline setup -> Quota discussion. |
| Day 10 | Write `src/server/db/seed.ts` that inserts the 3 templates with all their nodes, edges (parentNodeIds), positions, and configs. Run seed. Verify templates load correctly via tRPC. Wire seed to `package.json` script: `"db:seed": "npx tsx src/server/db/seed.ts"`. |

### Phase 2: Visual Workflow Builder (Week 3-4)

**Week 3 -- React Flow Canvas + Node Components**

| Day | Tasks |
|-----|-------|
| Day 11 | Install `@xyflow/react` v12. Build the workflow editor page `src/app/(dashboard)/workflows/[id]/editor/page.tsx`. Set up `ReactFlow` component with `nodeTypes`, `onNodesChange`, `onEdgesChange`, `onConnect`. Load template data via tRPC and convert WorkflowNodes to React Flow nodes/edges. |
| Day 12 | Build all 6 custom node components in `src/components/workflow/nodes/`: `TaskNode.tsx` (blue card with assignee avatar, due offset badge, task name), `DelayNode.tsx` (amber card with clock icon, "Wait N days" label), `NotificationNode.tsx` (violet card with bell, recipient), `ActionNode.tsx` (emerald card with zap icon, integration type badge), `ApprovalNode.tsx` (red card with shield, approver), `FormNode.tsx` (cyan card with form icon, field count). Each renders a 180x80px card with input/output handles using `<Handle>`. |
| Day 13 | Build node palette sidebar `src/components/workflow/node-palette.tsx`. Implement drag-from-palette: each palette item has `draggable` with `onDragStart` setting `dataTransfer`. ReactFlow pane has `onDrop` that reads the node type, calculates drop position via `screenToFlowPosition`, creates a new node via tRPC mutation, and adds to the flow. |
| Day 14 | Build node configuration drawer `src/components/workflow/node-config-drawer.tsx`. Opens on node click (right side panel, 400px wide). Contains a form that varies by node type: **Task** -- name, assignee role dropdown, offset days number input, phase dropdown, instructions textarea, sub-tasks list. **Delay** -- delay days. **Notification** -- recipient role, subject, body with template variable chips. **Action** -- integration type dropdown, dynamic config fields per integration. **Approval** -- approver role. **Form** -- dynamic field builder. Auto-saves on field blur with debounced tRPC mutation. |
| Day 15 | Implement edge management: `onConnect` handler creates a new edge and updates child node's `parentNodeIds` via tRPC. `onEdgesDelete` removes parent references. Visual styling: smoothstep edges with animated marching ants pattern using `animated: true`. Add minimap (`<MiniMap>`) and controls (`<Controls>`). |

**Week 4 -- Editor Polish + Template Management**

| Day | Tasks |
|-----|-------|
| Day 16 | Implement auto-save with debounce: `onNodesChange` (position changes) batched and saved every 500ms via `useDebouncedCallback`. Show save indicator ("Saving..." / "Saved") in top bar. Add undo/redo using React Flow's `useReactFlow` store snapshots (keep last 20 states in memory). |
| Day 17 | Build template variable system for notification/email nodes. Variables: `{{new_hire_name}}`, `{{new_hire_email}}`, `{{start_date}}`, `{{manager_name}}`, `{{buddy_name}}`, `{{role}}`, `{{department}}`, `{{portal_url}}`. In the config drawer, show clickable chips that insert variables into text fields. Preview panel shows resolved example values. |
| Day 18 | Add validation layer: before saving a template as "active", validate that all task nodes have assignee roles set, all notification nodes have subject/body, all action nodes have integration type selected, all edges form a valid DAG (no cycles -- check via topological sort). Show validation errors as red badges on invalid nodes. Implement in `src/lib/workflow/validate.ts`. |
| Day 19 | Build "Test Run" / preview mode: button in editor that simulates instantiation without creating real records. Shows a timeline preview of what would happen: "Day -3: IT Admin gets 'Order laptop' task", "Day 0: New hire gets welcome email", "Day 1: Manager gets 'Orientation' task", etc. Render as a vertical timeline component. |
| Day 20 | Template management features: duplicate template (deep clone all nodes), import/export template as JSON, archive template (soft delete). Build template settings panel (accessed via gear icon): name, description, role_type, trigger type, active toggle. |

### Phase 3: Onboarding Management + Task Engine (Week 5-6)

**Week 5 -- Onboarding Creation + Task Management**

| Day | Tasks |
|-----|-------|
| Day 21 | tRPC router: `src/server/trpc/routers/onboarding.ts` -- create, list (with filters), getById (with tasks + portal), update, cancel, pause, resume. Build `src/server/workflow/instantiate.ts` with the full template-to-onboarding instantiation logic (clone nodes to tasks, compute dates, resolve assignees, create portal, send welcome email). |
| Day 22 | Build "Start New Onboarding" flow: `src/app/(dashboard)/onboardings/new/page.tsx`. Multi-step form: Step 1 -- Select template (card grid of active templates). Step 2 -- New hire details (name, email, role, department, start_date date picker). Step 3 -- Assign team (manager dropdown from org members, optional buddy). Step 4 -- Review and confirm (summary card showing template, hire info, computed timeline preview). Submit calls `onboarding.create` tRPC mutation which calls `instantiateOnboarding`. |
| Day 23 | tRPC router: `src/server/trpc/routers/task.ts` -- list (by onboarding), updateStatus, assignTo, addNote, updateSubTasks, skip. Build the workflow execution engine `src/server/workflow/engine.ts`: `completeTask`, `resolveDownstreamTasks`, `activateTask`, `recomputeCompletionPercentage`. |
| Day 24 | Build Active Onboardings list page `src/app/(dashboard)/onboardings/page.tsx`. Card grid layout: each card shows new_hire avatar placeholder (initials), name, role, start date, progress bar (percentage), status badge, manager name. Filter bar: by status (all, pre_boarding, active, completed), department, manager. Sort by start date, completion %, name. |
| Day 25 | Build Onboarding Detail page `src/app/(dashboard)/onboardings/[id]/page.tsx`. Header: new hire info card (name, email, role, department, start_date, manager, buddy, portal link). Body: tabbed view -- "Tasks" tab (default) and "Timeline" tab. Tasks tab shows tasks grouped by phase (Pre-boarding, Day 1, Week 1, Week 2, Month 1) with collapsible sections. Each task row: status icon, name, assignee avatar, due date, overdue indicator. Click to expand: instructions, sub-tasks (checkable), notes field, file attachments. |

**Week 6 -- Email Notifications + Reminders + Cron Jobs**

| Day | Tasks |
|-----|-------|
| Day 26 | Set up Resend email system. Build email templates in `src/server/email/templates/`: `welcome-new-hire.tsx` (React Email component with portal link), `task-assigned.tsx` (task name, due date, instructions, action link), `task-reminder.tsx` (overdue warning), `task-completed-notify.tsx` (notify manager when a task completes). Build `src/server/email/send.ts` with generic send function wrapping Resend. |
| Day 27 | Implement QStash integration in `src/server/lib/qstash.ts`. Functions: `scheduleTaskReminder(taskId, sendAt)`, `scheduleDelayComplete(taskId, activateAt)`, `cancelScheduled(scheduleId)`. Configure QStash signing verification. Build reminder scheduling: when a task is activated, schedule a reminder email for 24h before due date and another on due date morning. |
| Day 28 | Build Vercel cron endpoint `src/app/api/cron/check-overdue/route.ts`. Runs every 15 minutes via `vercel.json` cron config. Logic: query all active onboarding tasks where `due_date < today AND status IN ('active', 'in_progress')`. For each, update status to "overdue" and send overdue notification to assignee + manager. Also: check for tasks that should activate today based on `due_date` and `status = 'pending'`. |
| Day 29 | Build cron endpoint `src/app/api/cron/delay-complete/route.ts` (called by QStash when delay timer fires). Marks the delay task as complete and triggers `resolveDownstreamTasks`. Build `src/app/api/cron/daily-digest/route.ts`: runs once daily at 9am, sends each HR admin a summary email of today's tasks across all active onboardings (tasks due today, overdue tasks, upcoming start dates this week). |
| Day 30 | End-to-end test of the full workflow: create template with 5 nodes (task -> delay -> notification -> action -> task), create onboarding from it, verify tasks created with correct dates, complete first task, verify delay activates, complete delay, verify notification sends, verify action executes, verify final task activates, verify completion percentage reaches 100%. Fix all edge cases discovered. |

### Phase 4: New Hire Portal (Week 7)

**Week 7 -- Token-Based Portal**

| Day | Tasks |
|-----|-------|
| Day 31 | Build portal route group `src/app/(portal)/portal/[token]/page.tsx`. No Clerk auth wrapper -- this route is public. Server component that validates token via DB lookup (check `access_token` match + `expires_at` not passed), fetches onboarding data + tasks + documents. Returns 404 page if invalid/expired token. |
| Day 32 | Build portal layout `src/app/(portal)/layout.tsx`: clean, branded layout (no sidebar, no app navigation). Shows org logo from branding settings, company name, new hire name in header. Build portal homepage: welcome message (markdown rendered), start date countdown ("Your first day is in 5 days!"), personal progress bar, quick links to sections. |
| Day 33 | Build portal task checklist section `src/components/portal/task-checklist.tsx`. Shows only tasks assigned to the new hire (`assignedToEmail = new_hire_email` or `assigneeRole = "new_hire"`). Grouped by phase with checkboxes. Checking a task calls `POST /api/portal/[token]/tasks/[taskId]/complete` -- a dedicated API route (not tRPC) that validates the token, verifies the task belongs to this onboarding and is assigned to the new hire, then calls `completeTask`. |
| Day 34 | Build portal resources section `src/components/portal/resources.tsx`: renders the `resources` JSON from newHirePortals as a list of cards (links, documents, videos). Build portal forms section: if any tasks of type "form" are assigned to the new hire, render the form fields inline. On submit, store responses in the task's `notes` field as JSON and complete the task. |
| Day 35 | Build portal document section: list documents associated with this onboarding, with download links (S3 presigned URLs generated on request). Build portal "My Team" section: show manager name/avatar/email, buddy name/avatar/email, fetched from orgMembers. Add rate limiting middleware for portal routes (60 req/min per token via Upstash Redis). |

### Phase 5: Integrations (Week 8)

**Week 8 -- Google Workspace + Slack + Billing**

| Day | Tasks |
|-----|-------|
| Day 36 | tRPC router: `src/server/trpc/routers/integration.ts` -- list, connect, disconnect, testConnection. Build integration management page `src/app/(dashboard)/settings/integrations/page.tsx`: cards for Google Workspace and Slack, each with connect/disconnect button and status indicator. Build `src/server/lib/encryption.ts`: AES-256-GCM encrypt/decrypt functions for storing integration credentials. Encryption key from `INTEGRATION_ENCRYPTION_KEY` env var. |
| Day 37 | Build Google Workspace OAuth flow: `src/app/api/integrations/google/authorize/route.ts` (redirect to Google consent screen with Admin SDK scopes), `src/app/api/integrations/google/callback/route.ts` (exchange code for service account credentials, store encrypted). Build `src/server/integrations/google.ts` with `provisionGoogleWorkspaceUser` (create account, add to groups) and `testGoogleConnection` (list one user to verify access). |
| Day 38 | Build Slack OAuth flow: `src/app/api/integrations/slack/authorize/route.ts` (redirect to Slack OAuth with required bot scopes), `src/app/api/integrations/slack/callback/route.ts` (exchange code for bot token, store encrypted with team metadata). Build `src/server/integrations/slack.ts` with `executeSlackAction` (invite, add to channels, send message) and `testSlackConnection` (call `auth.test` to verify token). Install `@slack/web-api`. |
| Day 39 | Build `src/server/integrations/execute.ts` -- the dispatcher that the workflow engine calls. Reads integration credentials from DB, decrypts, routes to Google or Slack handler based on `integrationAction` in the node config. Implements retry logic (3 attempts with exponential backoff: 1s, 5s, 15s). Stores execution result in `onboardingTasks.executionResult`. |
| Day 40 | Set up Stripe billing. Create products/prices in Stripe dashboard: Starter ($99/mo, `price_starter`), Growth ($249/mo, `price_growth`). Build `src/server/billing/stripe.ts`: `createCheckoutSession`, `createPortalSession`, `getSubscriptionDetails`. Build webhook handler `src/app/api/webhooks/stripe/route.ts`: handle `checkout.session.completed` (set plan + stripeCustomerId), `customer.subscription.updated` (plan changes), `customer.subscription.deleted` (downgrade to starter). Build plan enforcement in tRPC middleware: Starter = 5 hires/mo + 3 templates + 2 integrations; Growth = 25 hires/mo + unlimited templates + unlimited integrations. |

### Phase 6: Dashboard + Analytics + Polish (Week 9-10)

**Week 9 -- Dashboard + Analytics**

| Day | Tasks |
|-----|-------|
| Day 41 | Build main dashboard page `src/app/(dashboard)/page.tsx`. Overview cards: Active Onboardings count, Hires This Month, Average Completion Rate, Overdue Tasks count. Each card with trend indicator (up/down arrow comparing to previous period). Query via `src/server/trpc/routers/dashboard.ts` with aggregation queries. |
| Day 42 | Build dashboard charts: "Onboarding Progress" bar chart (each active onboarding as a horizontal bar showing completion %), "Upcoming Start Dates" calendar strip (next 14 days with avatars on start dates), "Recent Activity" feed (last 20 activity log entries rendered as a timeline). Install `recharts` for chart components. |
| Day 43 | Build analytics tab `src/app/(dashboard)/analytics/page.tsx` (Growth plan only -- gate with plan check middleware). Charts: Completion Rate Over Time (line chart, last 6 months), Average Days to Complete (number card with trend), Top Bottleneck Tasks (bar chart showing tasks most frequently overdue), Task Completion by Assignee Role (pie chart). |
| Day 44 | Build onboarding timeline/Gantt view as alternative to task list on the onboarding detail page. `src/components/onboarding/timeline-view.tsx`: horizontal bars for each task, positioned by due_date, colored by status (gray=pending, blue=active, green=completed, red=overdue). Scrollable horizontally across the onboarding timespan. Uses HTML/CSS grid, not a heavy Gantt library. |
| Day 45 | Build billing page `src/app/(dashboard)/settings/billing/page.tsx`: current plan card, usage metrics (hires this month / limit, templates / limit, integrations / limit), upgrade button (opens Stripe Checkout), manage subscription button (opens Stripe Portal). Pricing comparison table for Starter vs Growth. |

**Week 10 -- Polish, Edge Cases, Launch Prep**

| Day | Tasks |
|-----|-------|
| Day 46 | Build settings pages: `src/app/(dashboard)/settings/page.tsx` (general org settings -- name, logo upload to S3, timezone, default reminder time), `src/app/(dashboard)/settings/team/page.tsx` (member management -- invite via Clerk, role assignment, list/remove). Build portal customization in settings: welcome message editor, resource manager (add/remove links/docs). |
| Day 47 | Error handling pass: add error boundaries to all page components, add toast notifications for all mutations (success/error), add loading skeletons for all data-fetching pages. Build empty states for: no workflows, no onboardings, no integrations. Add confirmation dialogs for destructive actions (cancel onboarding, delete template, disconnect integration). |
| Day 48 | Mobile responsiveness pass: ensure sidebar collapses to hamburger menu on mobile, onboarding cards stack vertically, task list is touch-friendly, portal is fully mobile-optimized (this is critical -- new hires will likely check on phone). Test all pages at 375px, 768px, 1024px, 1440px widths. |
| Day 49 | Security hardening: verify all tRPC procedures check org membership, verify portal token validation is timing-safe (`crypto.timingSafeEqual`), verify S3 presigned URLs have proper expiry, verify Stripe webhook signature validation, add CSRF protection to portal form submissions, add rate limiting to auth-less endpoints. Run through OWASP top 10 checklist for the portal routes. |
| Day 50 | Deploy to Vercel. Configure environment variables. Set up Vercel cron jobs in `vercel.json`: `check-overdue` every 15 min, `daily-digest` at 9am UTC. Connect custom domain. Smoke test full flow: sign up -> create org -> connect Google + Slack -> create template -> start onboarding -> complete tasks via dashboard + portal -> verify emails sent -> verify integrations fired -> verify completion. |

---

## Critical Files

```
onboardflow/
├── .env.example
├── drizzle.config.ts
├── package.json
├── vercel.json                                     # Cron job config
├── src/
│   ├── env.ts                                      # Zod env validation
│   ├── middleware.ts                                # Clerk auth middleware
│   │
│   ├── app/
│   │   ├── layout.tsx                              # Root layout (Clerk + TRPCProvider)
│   │   ├── page.tsx                                # Landing / redirect
│   │   │
│   │   ├── (dashboard)/
│   │   │   ├── layout.tsx                          # Dashboard shell (sidebar + topbar)
│   │   │   ├── page.tsx                            # Main dashboard (overview cards, charts)
│   │   │   │
│   │   │   ├── workflows/
│   │   │   │   ├── page.tsx                        # Template list (card grid)
│   │   │   │   └── [id]/
│   │   │   │       └── editor/
│   │   │   │           └── page.tsx                # React Flow workflow builder
│   │   │   │
│   │   │   ├── onboardings/
│   │   │   │   ├── page.tsx                        # Active onboardings list
│   │   │   │   ├── new/
│   │   │   │   │   └── page.tsx                    # Start new onboarding wizard
│   │   │   │   └── [id]/
│   │   │   │       └── page.tsx                    # Onboarding detail (tasks + timeline)
│   │   │   │
│   │   │   ├── analytics/
│   │   │   │   └── page.tsx                        # Analytics dashboard (Growth plan)
│   │   │   │
│   │   │   └── settings/
│   │   │       ├── page.tsx                        # General org settings
│   │   │       ├── team/
│   │   │       │   └── page.tsx                    # Member management
│   │   │       ├── integrations/
│   │   │       │   └── page.tsx                    # Google + Slack integration mgmt
│   │   │       └── billing/
│   │   │           └── page.tsx                    # Plan + usage + upgrade
│   │   │
│   │   ├── (portal)/
│   │   │   ├── layout.tsx                          # Portal layout (no sidebar, branded)
│   │   │   └── portal/
│   │   │       └── [token]/
│   │   │           └── page.tsx                    # New hire portal
│   │   │
│   │   └── api/
│   │       ├── trpc/[trpc]/route.ts                # tRPC HTTP handler
│   │       ├── webhooks/stripe/route.ts            # Stripe webhook
│   │       ├── portal/[token]/tasks/[taskId]/
│   │       │   └── complete/route.ts               # Portal task completion (no auth)
│   │       ├── integrations/
│   │       │   ├── google/authorize/route.ts       # Google OAuth start
│   │       │   ├── google/callback/route.ts        # Google OAuth callback
│   │       │   ├── slack/authorize/route.ts        # Slack OAuth start
│   │       │   └── slack/callback/route.ts         # Slack OAuth callback
│   │       ├── upload/presigned/route.ts           # S3 presigned URL generation
│   │       └── cron/
│   │           ├── check-overdue/route.ts          # 15-min cron: overdue detection
│   │           ├── delay-complete/route.ts         # QStash callback: delay node done
│   │           └── daily-digest/route.ts           # Daily summary email
│   │
│   ├── server/
│   │   ├── db/
│   │   │   ├── schema.ts                          # Full Drizzle schema (9 tables + activityLogs)
│   │   │   ├── index.ts                           # DB connection (Neon serverless)
│   │   │   ├── seed.ts                            # Seed 3 built-in templates
│   │   │   └── seed-templates.ts                  # Template definitions (general, eng, sales)
│   │   │
│   │   ├── trpc/
│   │   │   ├── trpc.ts                            # tRPC init, auth/org/plan middleware
│   │   │   ├── context.ts                         # Request context (db + auth)
│   │   │   └── routers/
│   │   │       ├── _app.ts                        # Root router (merges all sub-routers)
│   │   │       ├── organization.ts                # Org CRUD + settings
│   │   │       ├── member.ts                      # Org member list + role update
│   │   │       ├── workflow-template.ts           # Template CRUD + duplicate
│   │   │       ├── workflow-node.ts               # Node CRUD + batch position update
│   │   │       ├── onboarding.ts                  # Onboarding CRUD + lifecycle
│   │   │       ├── task.ts                        # Task status updates + assignment
│   │   │       ├── integration.ts                 # Integration CRUD + test
│   │   │       ├── dashboard.ts                   # Aggregated dashboard queries
│   │   │       └── billing.ts                     # Checkout + portal + usage
│   │   │
│   │   ├── workflow/
│   │   │   ├── instantiate.ts                     # Template -> live onboarding
│   │   │   ├── engine.ts                          # Task completion + dependency resolution
│   │   │   └── validate.ts                        # Template validation (DAG check, required fields)
│   │   │
│   │   ├── integrations/
│   │   │   ├── google.ts                          # Google Admin SDK provisioning
│   │   │   ├── slack.ts                           # Slack API actions
│   │   │   └── execute.ts                         # Integration action dispatcher + retry
│   │   │
│   │   ├── email/
│   │   │   ├── send.ts                            # Generic Resend send wrapper
│   │   │   └── templates/
│   │   │       ├── welcome-new-hire.tsx            # React Email: welcome + portal link
│   │   │       ├── task-assigned.tsx               # React Email: new task notification
│   │   │       ├── task-reminder.tsx               # React Email: overdue warning
│   │   │       ├── task-completed-notify.tsx       # React Email: task done (to manager)
│   │   │       └── daily-digest.tsx               # React Email: daily summary
│   │   │
│   │   ├── billing/
│   │   │   └── stripe.ts                          # Stripe checkout/portal/plans
│   │   │
│   │   └── lib/
│   │       ├── encryption.ts                      # AES-256-GCM encrypt/decrypt
│   │       ├── qstash.ts                          # QStash scheduling helpers
│   │       └── date-utils.ts                      # computeDueDate, business day logic
│   │
│   ├── lib/
│   │   ├── trpc/
│   │   │   ├── client.ts                          # tRPC React Query client
│   │   │   └── server.ts                          # tRPC server-side caller
│   │   └── workflow/
│   │       ├── node-types.ts                      # React Flow node type definitions
│   │       ├── validate.ts                        # Client-side validation helpers
│   │       └── template-variables.ts              # {{variable}} definitions + resolver
│   │
│   └── components/
│       ├── ui/                                    # Shared UI primitives
│       │   ├── button.tsx
│       │   ├── card.tsx
│       │   ├── dialog.tsx
│       │   ├── input.tsx
│       │   ├── select.tsx
│       │   ├── badge.tsx
│       │   ├── toast.tsx
│       │   ├── tabs.tsx
│       │   ├── progress.tsx
│       │   └── dropdown-menu.tsx
│       │
│       ├── dashboard/
│       │   ├── sidebar.tsx                        # App sidebar navigation
│       │   ├── overview-cards.tsx                  # Dashboard stat cards
│       │   └── activity-feed.tsx                   # Recent activity timeline
│       │
│       ├── workflow/
│       │   ├── workflow-canvas.tsx                 # Main React Flow wrapper
│       │   ├── node-palette.tsx                    # Drag-source sidebar
│       │   ├── node-config-drawer.tsx              # Right-side config panel
│       │   ├── template-preview.tsx                # Timeline preview of template
│       │   └── nodes/
│       │       ├── TaskNode.tsx                    # Custom React Flow node
│       │       ├── DelayNode.tsx
│       │       ├── NotificationNode.tsx
│       │       ├── ActionNode.tsx
│       │       ├── ApprovalNode.tsx
│       │       └── FormNode.tsx
│       │
│       ├── onboarding/
│       │   ├── onboarding-card.tsx                # Card for the list grid
│       │   ├── task-list.tsx                       # Grouped task list with expand
│       │   ├── task-row.tsx                        # Individual task row
│       │   ├── timeline-view.tsx                   # Gantt-like timeline
│       │   ├── new-onboarding-wizard.tsx           # Multi-step creation form
│       │   └── progress-ring.tsx                   # Circular progress indicator
│       │
│       └── portal/
│           ├── portal-header.tsx                   # Branded header with logo
│           ├── task-checklist.tsx                  # New hire checkable task list
│           ├── resources.tsx                       # Resource cards (links, docs, videos)
│           ├── team-section.tsx                    # Manager + buddy info
│           └── countdown.tsx                       # Days until start date
```

---

## Verification

### Smoke Test Checklist

1. **Auth + Org**: Sign up via Clerk -> create organization -> verify `organizations` and `orgMembers` rows created
2. **Template CRUD**: Create a workflow template -> add 5 nodes via React Flow drag-and-drop -> connect edges -> verify `workflowNodes` with correct `parentNodeIds` and positions saved
3. **Built-in Templates**: Verify 3 seed templates (general, engineering, sales) load correctly in template list and render in React Flow editor
4. **Instantiation**: Select engineering template -> fill new hire form -> submit -> verify `onboardings` row with correct dates, `onboardingTasks` rows with computed absolute `due_date` values, `newHirePortals` row with valid token, welcome email sent via Resend
5. **Task Completion Flow**: Complete a task via dashboard -> verify downstream dependency resolved (blocked task becomes active) -> verify completion percentage updates -> verify activity log entry created
6. **Portal Access**: Open `portal/[token]` URL in incognito (no Clerk session) -> verify portal loads with welcome message, task checklist, resources -> check off a task -> verify it completes in the main dashboard
7. **Expired Portal**: Set `expires_at` to past date -> verify portal returns 404
8. **Email Notifications**: Verify task-assigned email sends when a task activates -> verify overdue reminder sends when cron runs with an overdue task
9. **Google Provisioning**: Connect Google integration (test with sandbox domain) -> create onboarding with "google_create_user" action node -> verify user created in Google Admin console + added to specified groups
10. **Slack Integration**: Connect Slack integration via OAuth -> create onboarding with "slack_add_to_channel" action node -> verify bot adds user to channel
11. **Cron Jobs**: Manually trigger `/api/cron/check-overdue` -> verify tasks past due_date marked as "overdue" and notification sent -> trigger `/api/cron/daily-digest` -> verify summary email sent to HR admin
12. **Billing**: Click upgrade to Growth -> complete Stripe Checkout -> verify plan updated in DB -> verify increased limits (25 hires/mo, unlimited templates, all integrations) -> verify analytics page accessible
13. **Plan Enforcement**: On Starter plan, attempt to create 6th hire in a month -> verify error "Plan limit reached" -> attempt to create 4th template -> verify error -> attempt to connect 3rd integration -> verify error
14. **Workflow Validation**: Create template with a node missing required config (task without assignee) -> attempt to activate -> verify validation error displayed on the node in the editor
15. **Delay Nodes**: Create onboarding with a delay node (2 days) -> verify delay task enters "in_progress" -> manually trigger QStash callback -> verify delay completes and downstream task activates
