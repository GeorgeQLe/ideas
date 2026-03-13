# Code Review Remediation Plan

**Review Date:** 2026-03-13
**Reviewer:** Claude Opus 4.6 (automated expert panel review)
**Scope:** All 5 sub-projects — DriftLog, PulseBoard, SnipVault, FormForge, Asset Inventory DB
**Validation:** Cross-referenced against specs, plans, and interview docs. Withdrawn findings that were deliberate design decisions (e.g., Asset Inventory DB single-org model).

---

## Priority Legend

| Priority | SLA | Description |
|----------|-----|-------------|
| P0 — Critical | Fix before any deploy | Security vulnerabilities, data loss/corruption risks |
| P1 — High | Fix within current sprint | Performance, architecture, significant error handling gaps |
| P2 — Medium | Fix within next 2 sprints | Code quality, operational risks, silent failures |
| P3 — Low | Backlog | Minor improvements, style, hardening |

---

## P0 — Critical

### CR-001: SQL Injection in SnipVault Search

| Field | Value |
|-------|-------|
| **Project** | SnipVault |
| **File** | `snipvault/src/lib/trpc/routers/search.ts` |
| **Lines** | 53–75, 93, 107, 170 |
| **Category** | Security — SQL Injection |

**Problem:**
User-supplied `language` and `collectionId` parameters are interpolated directly into SQL filter strings via template literals, then executed with `sql.unsafe()`:

```typescript
const filters: string[] = [`s.workspace_id = '${ctx.workspaceId}'`];
if (language) {
  filters.push(`s.language = '${language}'`);  // user input
}
// ...
WHERE ${sql.unsafe(filterClause)}
```

An attacker can pass `language = "'; DROP TABLE snippets; --"` to execute arbitrary SQL.

**Remediation:**
Replace string concatenation with Drizzle's parameterized query builder. Build filter conditions as an array of `SQL` objects using `sql\`...\`` tagged templates with proper parameter binding:

```typescript
const conditions: SQL[] = [sql`s.workspace_id = ${ctx.workspaceId}`];
if (language) {
  conditions.push(sql`s.language = ${language}`);
}
if (collectionId) {
  conditions.push(sql`s.collection_id = ${collectionId}`);
}
const filterClause = sql.join(conditions, sql` AND `);
```

Remove all uses of `sql.unsafe()` in this file.

**Verification:**
- [ ] All filter parameters use parameterized queries
- [ ] No remaining `sql.unsafe()` calls in search router
- [ ] Manual test: pass `'; DROP TABLE --` as language filter and confirm it's treated as a literal string
- [ ] Semantic search still returns correct results

---

### CR-002: Unauthenticated S3 Presigned URL Endpoint

| Field | Value |
|-------|-------|
| **Project** | FormForge |
| **File** | `formforge/src/app/api/upload/presigned/route.ts` |
| **Lines** | 1–87 |
| **Category** | Security — Authentication Bypass |

**Problem:**
The endpoint imports `auth` from Clerk but never calls it. The middleware (`middleware.ts:10`) marks `/api/upload/presigned` as a public route. Any unauthenticated user can request unlimited presigned URLs to upload arbitrary files to the S3 bucket, enabling storage abuse and cost escalation.

**Remediation:**
Two options depending on intended use:

**Option A — If only authenticated users should upload (dashboard file fields):**
```typescript
export async function POST(request: NextRequest) {
  const { userId } = await auth();
  if (!userId) {
    return NextResponse.json({ error: "Unauthorized" }, { status: 401 });
  }
  // ... rest of handler
}
```
Also remove `/api/upload/presigned` from the public routes list in `middleware.ts`.

**Option B — If anonymous form respondents need to upload (public form file fields):**
Keep the route public but add:
1. Rate limiting (e.g., 10 requests per IP per minute via `@upstash/ratelimit`)
2. Require a valid form slug + field ID in the request body and validate the form exists and has a file upload field
3. Cap total upload size per form submission

**Verification:**
- [ ] Unauthenticated request returns 401 (Option A) or is rate-limited (Option B)
- [ ] File upload still works in the intended flow
- [ ] Test with `curl` from an unauthenticated context

---

### CR-003: Bulk Status Update Missing Ownership Check

| Field | Value |
|-------|-------|
| **Project** | FormForge |
| **File** | `formforge/src/server/trpc/routers/response.ts` |
| **Lines** | 234–261 |
| **Category** | Security — Authorization Bypass |

**Problem:**
`bulkUpdateStatus` verifies the form belongs to the authenticated user (line 243–247) but the WHERE clause on the actual update (line 256) only filters by `formResponses.id` — it does not verify each response belongs to the specified form. An attacker who owns any form can update responses belonging to other users' forms by guessing UUID response IDs.

```typescript
// Current — no form scoping on the update:
.where(eq(formResponses.id, id))

// The same issue exists in the single `updateStatus` at line 225
// and `delete` at line 279
```

**Remediation:**
Add `formId` to the WHERE clause on all response mutations:

```typescript
await Promise.all(
  input.responseIds.map((id) =>
    ctx.db
      .update(formResponses)
      .set({ status: input.status })
      .where(
        and(
          eq(formResponses.id, id),
          eq(formResponses.formId, input.formId)
        )
      )
  )
);
```

Apply the same fix to:
- `updateStatus` (line 222–225)
- `delete` (line 277–279)

**Verification:**
- [ ] All three mutations (updateStatus, bulkUpdateStatus, delete) include `formId` in WHERE
- [ ] Test: create two users with separate forms; confirm user A cannot modify user B's responses
- [ ] Existing bulk operations still work for legitimate use

---

## P1 — High

### CR-004: N+1 Queries in Alert Detection Cron

| Field | Value |
|-------|-------|
| **Project** | PulseBoard |
| **File** | `pulseboard/src/server/cron/alert-detection.ts` |
| **Lines** | 51–147 |
| **Category** | Performance |

**Problem:**
The cron job iterates over every user in every org, executing an individual DB query per user to fetch recent check-ins, then individual email sends per manager. For 100 users across 5 orgs: 500+ DB queries per cron run.

**Remediation:**
Batch-fetch all check-ins for the org in a single query:

```typescript
const recentCheckIns = await db.select()
  .from(checkIns)
  .where(and(
    inArray(checkIns.userId, orgUserIds),
    gte(checkIns.date, fiveDaysAgo)
  ))
  .orderBy(checkIns.userId, desc(checkIns.date));

// Group in memory by userId
const checkInsByUser = new Map<string, typeof recentCheckIns>();
for (const ci of recentCheckIns) {
  if (!checkInsByUser.has(ci.userId)) checkInsByUser.set(ci.userId, []);
  checkInsByUser.get(ci.userId)!.push(ci);
}
```

Similarly, batch email sends or use a queue.

**Verification:**
- [ ] Single DB query per org instead of per-user
- [ ] Alert detection results are identical before/after refactor
- [ ] Cron execution time reduced (measure before/after)

---

### CR-005: Race Condition in Draft Release Creation

| Field | Value |
|-------|-------|
| **Project** | DriftLog |
| **File** | `driftlog/src/server/webhooks/process-merge-event.ts` |
| **Lines** | 204–228 |
| **Category** | Correctness — Race Condition |

**Problem:**
Check-then-act pattern without atomic transaction. Two simultaneous webhook events can both see "no draft release" and both create one, resulting in duplicate drafts.

**Remediation:**
Wrap in a serializable transaction or use `ON CONFLICT`:

```typescript
const [draftRelease] = await db.transaction(async (tx) => {
  const [existing] = await tx
    .select()
    .from(releases)
    .where(and(eq(releases.orgId, orgId), eq(releases.status, "draft")))
    .for("update")
    .limit(1);

  if (existing) return [existing];

  return tx.insert(releases).values({ orgId, status: "draft", ... }).returning();
});
```

Alternatively, add a partial unique index: `CREATE UNIQUE INDEX ON releases (org_id) WHERE status = 'draft'` and use `ON CONFLICT DO NOTHING` + re-fetch.

**Verification:**
- [ ] Concurrent webhook test: fire 10 merge events simultaneously, confirm exactly 1 draft release
- [ ] Normal flow still creates draft releases correctly

---

### CR-006: Unvalidated Form Settings Schema

| Field | Value |
|-------|-------|
| **Project** | FormForge |
| **File** | `formforge/src/server/trpc/routers/form.ts` |
| **Line** | 136 |
| **Category** | Security — Input Validation |

**Problem:**
`settings: z.any().optional()` accepts arbitrary JSON, enabling injection of unexpected data into the settings column.

**Remediation:**
Replace with a strict schema matching the documented settings fields:

```typescript
settings: z.object({
  notificationEmails: z.string().optional(),
  responseLimit: z.number().int().positive().optional(),
  closeDate: z.string().datetime().optional(),
  redirectUrl: z.string().url().optional(),
  successMessage: z.string().max(500).optional(),
  gdprConsentEnabled: z.boolean().optional(),
}).optional(),
```

**Verification:**
- [ ] Existing form settings still save/load correctly
- [ ] Submitting unexpected keys (e.g., `{"malicious": true}`) is rejected
- [ ] All dashboard settings UI fields still function

---

### CR-007: Unbounded CSV Export

| Field | Value |
|-------|-------|
| **Project** | FormForge |
| **File** | `formforge/src/server/trpc/routers/response.ts` |
| **Lines** | 287+ |
| **Category** | Performance — OOM Risk |

**Problem:**
`exportCsv` fetches all form responses without any LIMIT. A form with 1M responses will load everything into memory and likely crash the server.

**Remediation:**
Add a hard cap and use streaming for large exports:

```typescript
const MAX_EXPORT_ROWS = 10_000;

const responses = await ctx.db
  .select()
  .from(formResponses)
  .where(eq(formResponses.formId, input.formId))
  .orderBy(desc(formResponses.submittedAt))
  .limit(MAX_EXPORT_ROWS);
```

For exports exceeding the cap, consider a background job that writes to S3 and emails a download link.

**Verification:**
- [ ] Export works for forms with < 10k responses
- [ ] Forms with > 10k responses return first 10k with a warning (or queue a background job)
- [ ] No server OOM under load

---

### CR-008: No Concurrency Control on OpenAI Embedding Calls

| Field | Value |
|-------|-------|
| **Project** | SnipVault |
| **File** | `snipvault/src/lib/ai/embeddings.ts` |
| **Lines** | 36–78 |
| **Category** | Performance — API Rate Limiting |

**Problem:**
Each snippet creation fires an individual OpenAI API call with no batching or concurrency cap. Bulk imports or rapid creation could exhaust API rate limits, causing all embedding generation to fail.

**Remediation:**
Add a concurrency limiter:

```typescript
import pLimit from "p-limit";
const embeddingLimit = pLimit(5); // max 5 concurrent calls

export async function generateEmbedding(...) {
  return embeddingLimit(async () => {
    // existing implementation
  });
}
```

For bulk imports, consider batching with OpenAI's batch embedding endpoint.

**Verification:**
- [ ] Creating 50 snippets rapidly doesn't trigger rate limit errors
- [ ] Embeddings still generate correctly for individual snippet creation

---

## P2 — Medium

### CR-009: Cron Endpoints Open When Secret Unset

| Field | Value |
|-------|-------|
| **Project** | PulseBoard |
| **Files** | `pulseboard/src/app/api/cron/alerts/route.ts:12-18`, `cron/digest/route.ts:12-16` |
| **Category** | Security — Configuration Risk |

**Problem:**
The guard `if (cronSecret && authHeader !== ...)` allows all requests when `CRON_SECRET` is undefined. This is a dev convenience pattern but dangerous if it reaches production.

**Remediation:**
```typescript
const cronSecret = process.env.CRON_SECRET;
if (!cronSecret || authHeader !== `Bearer ${cronSecret}`) {
  return NextResponse.json({ error: "Unauthorized" }, { status: 401 });
}
```

Apply to all three cron routes: `reminders`, `alerts`, `digest`.

**Verification:**
- [ ] Cron endpoints return 401 when `CRON_SECRET` is unset
- [ ] Cron endpoints work correctly when `CRON_SECRET` is set and header matches
- [ ] Local dev instructions updated to include `CRON_SECRET` in `.env.local`

---

### CR-010: Turnstile Verification Silently Disabled

| Field | Value |
|-------|-------|
| **Project** | FormForge |
| **File** | `formforge/src/app/api/submit/[slug]/route.ts` |
| **Lines** | 13–14 |
| **Category** | Security — Silent Degradation |

**Problem:**
`if (!secret) return true;` means missing Turnstile config = no bot protection. No warning is emitted.

**Remediation:**
```typescript
async function verifyTurnstile(token: string): Promise<boolean> {
  const secret = process.env.TURNSTILE_SECRET_KEY;
  if (!secret) {
    console.warn("[Turnstile] TURNSTILE_SECRET_KEY not set — bot verification disabled");
    return true;
  }
  // ...
}
```

Additionally, add `TURNSTILE_SECRET_KEY` to the Zod env validation in `env.ts` (at least as optional with a startup warning).

**Verification:**
- [ ] Warning logged at request time when secret is missing
- [ ] Turnstile verification works correctly when configured

---

### CR-011: Silent AI Categorization Fallback

| Field | Value |
|-------|-------|
| **Project** | DriftLog |
| **File** | `driftlog/src/server/ai/generate-changelog.ts` |
| **Lines** | 290–304 |
| **Category** | Correctness — Silent Failure |

**Problem:**
When OpenAI returns malformed JSON, the category silently defaults to `"improvement"` with no logging of the raw response.

**Remediation:**
```typescript
} catch (parseError) {
  console.warn(
    "[AI] Failed to parse category response, defaulting to 'improvement':",
    { raw: content, error: parseError }
  );
  parsed = { category: "improvement" };
}
```

Consider also storing an `aiCategoryFailed: boolean` flag on the changelog entry so the UI can surface "AI categorization needs review."

**Verification:**
- [ ] Malformed AI response is logged with raw content
- [ ] Default category still applied gracefully

---

### CR-012: Fire-and-Forget AI Tasks in SnipVault

| Field | Value |
|-------|-------|
| **Project** | SnipVault |
| **File** | `snipvault/src/lib/trpc/routers/snippet.ts` |
| **Lines** | 200–219, 280–286 |
| **Category** | Correctness — Silent Degradation |

**Problem:**
AI tagging and embedding generation are fire-and-forget with only `console.error` on failure. Users have no visibility when these features silently fail.

**Remediation:**
Track AI processing status on the snippet record:

1. Add `aiStatus` column to snippets: `'pending' | 'complete' | 'failed'`
2. Set to `'pending'` on create, update to `'complete'` or `'failed'` in the `.catch()` handler
3. Surface status in the UI (e.g., "AI tagging in progress..." or "AI tagging failed — retry?")

Short-term fix (if schema change is too heavy): at minimum log with snippet ID for debugging:
```typescript
.catch((err) => {
  console.error(`[Snippet ${snippet.id}] Background AI tasks failed:`, err);
});
```

**Verification:**
- [ ] Failed AI tasks are traceable to specific snippet IDs
- [ ] (If schema change) UI shows pending/failed state

---

### CR-013: Unsafe Non-Null Assertion After Race Condition

| Field | Value |
|-------|-------|
| **Project** | SnipVault |
| **File** | `snipvault/src/lib/ai/tagging.ts` |
| **Lines** | ~127 |
| **Category** | Correctness — Crash Risk |

**Problem:**
After `onConflictDoNothing()` returns undefined (race condition), the code does `return found!.id`. If `found` is null, the server crashes.

**Remediation:**
```typescript
const found = await db.query.tags.findFirst({ where: ... });
if (!found) {
  throw new Error(`Tag race condition: could not find or create tag "${name}" in workspace ${workspaceId}`);
}
return found.id;
```

**Verification:**
- [ ] No `!` non-null assertions on potentially null DB query results
- [ ] Error message includes enough context for debugging

---

### CR-014: Failed Alert Emails Only Logged

| Field | Value |
|-------|-------|
| **Project** | PulseBoard |
| **File** | `pulseboard/src/server/cron/alert-detection.ts` |
| **Lines** | 131–144 |
| **Category** | Reliability — Lost Notifications |

**Problem:**
If sending a burnout alert email to a manager fails, it's `console.error`'d and skipped. The manager never receives the alert. No retry mechanism exists.

**Remediation:**
Track email delivery status on the alert record:

1. Add `notifiedManagerIds` and `failedManagerIds` arrays to the alerts table
2. On failure, record the manager ID in `failedManagerIds`
3. Add a follow-up cron pass that retries failed notifications (max 3 attempts)

Short-term: surface failed sends in an admin/manager dashboard.

**Verification:**
- [ ] Failed email sends are persisted (not just logged)
- [ ] Retry mechanism attempts re-delivery
- [ ] Managers eventually receive critical alerts

---

### CR-015: Digest Insert and Email Not Atomic

| Field | Value |
|-------|-------|
| **Project** | PulseBoard |
| **File** | `pulseboard/src/server/cron/weekly-digest.ts` |
| **Lines** | 199–214 |
| **Category** | Correctness — Partial Failure |

**Problem:**
The digest record is inserted into DB, then emails are sent in a loop. If email sending fails mid-loop, the digest exists but some managers never receive it. There's no way to know which managers were notified.

**Remediation:**
Add a `sentTo` JSON array on the digest record. Update it after each successful email send:

```typescript
const [digest] = await db.insert(digests).values({ ..., sentTo: [] }).returning();

for (const manager of managers) {
  try {
    await sendDigestEmail({ ... });
    await db.update(digests)
      .set({ sentTo: sql`${digests.sentTo} || ${JSON.stringify([manager.id])}::jsonb` })
      .where(eq(digests.id, digest.id));
  } catch (err) {
    console.error(`Failed to send digest to manager ${manager.id}:`, err);
  }
}
```

**Verification:**
- [ ] Digest records track which managers were successfully notified
- [ ] Partial failures don't block remaining sends

---

### CR-016: No Rate Limiting on Public Endpoints

| Field | Value |
|-------|-------|
| **Project** | All projects |
| **Category** | Security — DoS / Abuse |

**Affected endpoints:**
- FormForge: `/api/submit/[slug]` (form submissions)
- DriftLog: `/[orgSlug]` public changelog, `/api/public/track` analytics
- PulseBoard: None (all routes authenticated)
- SnipVault: `/auth/device` device code flow

**Remediation:**
Add middleware-level rate limiting using `@upstash/ratelimit` or similar:

```typescript
import { Ratelimit } from "@upstash/ratelimit";
import { Redis } from "@upstash/redis";

const ratelimit = new Ratelimit({
  redis: Redis.fromEnv(),
  limiter: Ratelimit.slidingWindow(10, "1 m"), // 10 req/min
});

// In route handler:
const ip = request.headers.get("x-forwarded-for") ?? "unknown";
const { success } = await ratelimit.limit(ip);
if (!success) {
  return NextResponse.json({ error: "Rate limited" }, { status: 429 });
}
```

**Verification:**
- [ ] Public endpoints return 429 after threshold exceeded
- [ ] Legitimate users are not impacted under normal use

---

### CR-017: Silent Null Owner on CSV Import

| Field | Value |
|-------|-------|
| **Project** | Asset Inventory DB |
| **File** | `asset-inventory-db/src/server/routers/import-export.ts` |
| **Lines** | 253–273 |
| **Category** | UX — Silent Data Loss |

**Problem:**
When a CSV import references an email not found in the system, the asset is created with `ownerId: null`. No warning is returned to the user.

**Remediation:**
During the validation step (before commit), collect unresolved emails and return them as warnings:

```typescript
const unresolvedEmails: string[] = [];
// ... in mapping loop:
if (ownerEmail && !emailToUserId.has(ownerEmail.toLowerCase())) {
  unresolvedEmails.push(ownerEmail);
}

return {
  validatedRows,
  warnings: unresolvedEmails.length > 0
    ? [`${unresolvedEmails.length} owner email(s) not found: ${unresolvedEmails.join(", ")}`]
    : [],
};
```

Surface warnings in the import wizard's validation step UI.

**Verification:**
- [ ] Import with unknown emails shows warning in preview step
- [ ] User can proceed with null owners if they choose
- [ ] Known emails still resolve correctly

---

## P3 — Low

### CR-018: Generic Error Instead of TRPCError

| Field | Value |
|-------|-------|
| **Project** | SnipVault |
| **File** | `snipvault/src/lib/trpc/routers/snippet.ts` |
| **Lines** | 130, 246, 314 |
| **Category** | Code Quality |

**Problem:**
`throw new Error('Snippet not found')` returns HTTP 500 with potential stack trace instead of proper 404.

**Remediation:**
```typescript
throw new TRPCError({ code: "NOT_FOUND", message: "Snippet not found" });
```

**Verification:**
- [ ] Client receives 404 status for missing snippets
- [ ] No stack traces in production responses

---

### CR-019: E2E Testing Credentials Provider

| Field | Value |
|-------|-------|
| **Project** | Asset Inventory DB |
| **File** | `asset-inventory-db/src/lib/auth.ts` |
| **Lines** | 14–30 |
| **Category** | Security — Configuration Safety |

**Problem:**
Standard testing pattern, but if `E2E_TESTING=true` leaks to production, OAuth is bypassed.

**Remediation:**
Add a double-guard:

```typescript
if (process.env.E2E_TESTING === "true" && process.env.NODE_ENV !== "production") {
  providers.push(CredentialsProvider({ ... }));
}
```

Ensure deployment configs (Vercel, Docker, etc.) never include `E2E_TESTING`.

**Verification:**
- [ ] E2E tests still pass locally
- [ ] Production build doesn't include credentials provider even if env var leaks

---

## Tracking Checklist

| ID | Priority | Project | Status |
|----|----------|---------|--------|
| CR-001 | P0 | SnipVault | [ ] Not started |
| CR-002 | P0 | FormForge | [ ] Not started |
| CR-003 | P0 | FormForge | [ ] Not started |
| CR-004 | P1 | PulseBoard | [ ] Not started |
| CR-005 | P1 | DriftLog | [ ] Not started |
| CR-006 | P1 | FormForge | [ ] Not started |
| CR-007 | P1 | FormForge | [ ] Not started |
| CR-008 | P1 | SnipVault | [ ] Not started |
| CR-009 | P2 | PulseBoard | [ ] Not started |
| CR-010 | P2 | FormForge | [ ] Not started |
| CR-011 | P2 | DriftLog | [ ] Not started |
| CR-012 | P2 | SnipVault | [ ] Not started |
| CR-013 | P2 | SnipVault | [ ] Not started |
| CR-014 | P2 | PulseBoard | [ ] Not started |
| CR-015 | P2 | PulseBoard | [ ] Not started |
| CR-016 | P2 | All | [ ] Not started |
| CR-017 | P2 | Asset Inventory DB | [ ] Not started |
| CR-018 | P3 | SnipVault | [ ] Not started |
| CR-019 | P3 | Asset Inventory DB | [ ] Not started |
