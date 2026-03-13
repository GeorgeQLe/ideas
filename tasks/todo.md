# Implementation Plan: Code Review Remediation

> Generated from: specs/code-review-remediation.md
> Date: 2026-03-13
> Total Phases: 10

## Summary
Fix 19 validated code review findings across 5 sub-projects (DriftLog, PulseBoard, SnipVault, FormForge, Asset Inventory DB), ordered by severity — P0 critical security fixes first, then P1 architecture/performance, then P2 hardening/reliability, and P3 polish. Each phase is scoped to a single project or tightly related fixes to minimize context switching.

## Phase Overview
| Phase | Title | CRs | Key Deliverable | Est. Complexity |
|-------|-------|-----|-----------------|-----------------|
| 1     | Test Infrastructure Setup | — | Vitest configured for snipvault, formforge, pulseboard | M |
| 2     | SnipVault SQL Injection Fix | CR-001 | Parameterized queries in search router | S |
| 3     | FormForge Auth & AuthZ Fixes | CR-002, CR-003 | Presigned URL auth + response ownership checks | S |
| 4     | FormForge Validation & Performance | CR-006, CR-007 | Strict settings schema + bounded CSV export | S |
| 5     | DriftLog Race Condition + SnipVault Concurrency | CR-005, CR-008 | Atomic draft release creation + embedding rate limiter | M |
| 6     | PulseBoard N+1 Query Fix | CR-004 | Batched alert detection queries | M |
| 7     | PulseBoard Hardening | CR-009, CR-014, CR-015 | Cron auth + alert email tracking + digest delivery tracking | M |
| 8     | SnipVault Reliability | CR-012, CR-013, CR-018 | AI status tracking + null safety + proper TRPCError | S |
| 9     | FormForge & DriftLog Hardening | CR-010, CR-011 | Turnstile warning + AI fallback logging | S |
| 10    | Cross-Project Rate Limiting + Asset DB | CR-016, CR-017, CR-019 | Rate limiting on public endpoints + import warnings + E2E guard | M |

---

## Phase 1: Test Infrastructure Setup

### Tests First
- Step 1.1: Set up Vitest in SnipVault
  - Install vitest and dependencies: `npm i -D vitest @vitest/coverage-v8`
  - Create `snipvault/vitest.config.ts` with path aliases matching tsconfig (`@/*` -> `./src/*`)
  - Add `test` and `test:watch` scripts to `snipvault/package.json`
  - Write a smoke test `snipvault/src/lib/__tests__/smoke.test.ts` that imports a utility and asserts it loads
  - Run `npm test` — smoke test must pass (green)

- Step 1.2: Set up Vitest in FormForge
  - Install vitest and dependencies: `npm i -D vitest @vitest/coverage-v8`
  - Create `formforge/vitest.config.ts` with path aliases matching tsconfig (`@/*` -> `./src/*`)
  - Add `test` and `test:watch` scripts to `formforge/package.json`
  - Write a smoke test `formforge/src/lib/__tests__/smoke.test.ts`
  - Run `npm test` — smoke test must pass (green)

- Step 1.3: Set up Vitest in PulseBoard
  - Install vitest and dependencies: `npm i -D vitest @vitest/coverage-v8`
  - Create `pulseboard/vitest.config.ts` with path aliases matching tsconfig (`@/*` -> `./src/*`)
  - Add `test` and `test:watch` scripts to `pulseboard/package.json`
  - Write a smoke test `pulseboard/src/lib/__tests__/smoke.test.ts`
  - Run `npm test` — smoke test must pass (green)

### Implementation
(Test infra IS the implementation for this phase.)

### Green
- Step 1.4: Run `npm test` in all three projects and confirm smoke tests pass

### Milestone: Test Infrastructure Ready
**Acceptance Criteria:**
- [x] `npm test` runs and passes in snipvault (1 test, 1 file)
- [x] `npm test` runs and passes in formforge (1 test, 1 file)
- [x] `npm test` runs and passes in pulseboard (2 tests, 1 file)
- [x] All three have vitest.config.ts with correct path aliases
- [x] Existing driftlog `npm test` still passes (123 tests, 3 files)

**On Completion:**
- Deviations from plan: Skipped @vitest/coverage-v8 — not needed until coverage reporting is required. PulseBoard smoke test uses zod import instead of a project utility (no pure utility functions available without DB/env deps).
- Tech debt / follow-ups: None
- Ready for next phase: yes

---

## Phase 2: SnipVault SQL Injection Fix (CR-001)

### Detailed Implementation Plan (for fresh context)

**Problem:** `snipvault/src/lib/trpc/routers/search.ts` uses `sql.unsafe()` with string-concatenated user input on lines 53-75, 93, 107, 170. The `language` and `collectionId` params are interpolated directly into filter strings.

**Architecture:** The file uses `neon()` from `@neondatabase/serverless` (NOT drizzle's `sql`). The `neon()` tagged template function (`const sql = neon(...)` at line 50) already supports parameterized queries via its tagged template syntax. The key issue is lines 53-75 build filters as plain strings, then `sql.unsafe(filterClause)` injects them raw.

**Approach — Extract `buildFilterClause` helper:**
1. Create a helper function `buildFilterClause(params: { workspaceId, language?, collectionId?, tagSnippetIds? })` that returns an object `{ text: string, values: unknown[] }` using numbered placeholders (`$1`, `$2`...).
2. However, since neon tagged template handles parameterization automatically, the better approach is to build the WHERE clause as multiple `AND` conditions directly in the tagged template, using conditional SQL fragments.
3. **Simplest fix:** Replace the string concat + `sql.unsafe()` pattern with individual parameterized conditions chained with `AND` in each SQL query. Use neon's tagged template `${value}` for all user inputs.

**Specific changes to `snipvault/src/lib/trpc/routers/search.ts`:**
- Remove lines 52-75 (string filter building)
- In each of the 3 SQL queries (lines 83-132, 150-173, and the `similar` query which is already safe):
  - Replace `${sql.unsafe(filterClause)}` with inline parameterized conditions:
    ```sql
    AND s.workspace_id = ${workspaceId}
    AND (${language}::text IS NULL OR s.language = ${language})
    AND (${collectionId}::text IS NULL OR s.collection_id = ${collectionId}::uuid)
    AND (${tagSnippetIds}::text[] IS NULL OR s.id = ANY(${tagSnippetIds}))
    ```
  - This uses PostgreSQL's `IS NULL` trick: when the param is null, the condition is always true (no filtering).
- Remove the `filterClause` variable entirely
- Remove `sql.unsafe` import/usage

**Test strategy:** Since the neon driver needs a real DB, unit tests should verify the _absence_ of `sql.unsafe()` and the _presence_ of parameterized patterns. The functional test is a grep-based static analysis + build verification.

### Tests First
- Step 2.1: Write tests for parameterized search filters
  - File: create `snipvault/src/lib/trpc/routers/__tests__/search.test.ts`
  - Test cases:
    - Static analysis: read search.ts source, assert zero occurrences of `sql.unsafe`
    - Static analysis: read search.ts source, assert zero occurrences of string template `'${` (single-quote interpolation in filter strings)
    - Verify the file exports a `searchRouter` (import test)
  - All tests should FAIL initially (file still has sql.unsafe)

### Implementation
- Step 2.2: Refactor filter construction in search router
  - File: modify `snipvault/src/lib/trpc/routers/search.ts`
  - Remove string concatenation filter building (lines 53-75)
  - Replace all 3 `sql.unsafe(filterClause)` calls (lines 93, 107, 170) with inline parameterized conditions using neon's tagged template
  - Use the `IS NULL OR` pattern for optional filters
  - For tagSnippetIds: use `= ANY(${tagSnippetIds})` with the array passed as a parameter
  - Remove ALL `sql.unsafe()` usage from this file

### Green
- Step 2.3: Run tests and verify all pass
- Step 2.4: Grep the entire file for `sql.unsafe` — must return zero results
- Step 2.5: Manually verify semantic search still returns expected results (if dev DB available)

### Milestone: SQL Injection Eliminated
**Acceptance Criteria:**
- [ ] Zero `sql.unsafe()` calls in `snipvault/src/lib/trpc/routers/search.ts`
- [ ] All filter parameters use Drizzle's parameterized `sql` tagged templates
- [ ] Test with `language = "'; DROP TABLE snippets; --"` treats value as literal string
- [ ] Semantic search (vector + text hybrid) returns correct results
- [ ] All phase tests pass
- [ ] No regressions in Phase 1 tests

**On Completion:**
- Deviations from plan:
- Tech debt / follow-ups:
- Ready for next phase: yes/no

---

## Phase 3: FormForge Auth & AuthZ Fixes (CR-002, CR-003)

### Tests First
- Step 3.1: Write tests for presigned URL auth and response ownership
  - File: create `formforge/src/server/__tests__/presigned-url-auth.test.ts`
  - Test cases (CR-002):
    - Request without auth returns 401
    - Request with valid auth returns presigned URL
    - (If Option B chosen) Rate limiting kicks in after threshold
  - File: create `formforge/src/server/trpc/routers/__tests__/response.test.ts`
  - Test cases (CR-003):
    - `updateStatus` with response belonging to user's form succeeds
    - `updateStatus` with response NOT belonging to user's form returns 0 rows updated (no error, just no-op)
    - `bulkUpdateStatus` only updates responses that belong to the specified formId
    - `delete` only deletes responses that belong to the specified formId
  - All tests should FAIL initially

### Implementation
- Step 3.2: Add auth check to presigned URL endpoint (CR-002)
  - File: modify `formforge/src/app/api/upload/presigned/route.ts`
  - Add `const { userId } = await auth(); if (!userId) return NextResponse.json({ error: "Unauthorized" }, { status: 401 });` at top of POST handler
  - File: modify `formforge/src/middleware.ts`
  - Remove `/api/upload/presigned` from public routes list (if it needs to stay public for anonymous form respondents, instead add rate limiting + form slug validation)

- Step 3.3: Add formId scoping to response mutations (CR-003)
  - File: modify `formforge/src/server/trpc/routers/response.ts`
  - `updateStatus` (~line 225): change `.where(eq(formResponses.id, input.id))` to `.where(and(eq(formResponses.id, input.id), eq(formResponses.formId, input.formId)))`
  - `bulkUpdateStatus` (~line 256): change `.where(eq(formResponses.id, id))` to `.where(and(eq(formResponses.id, id), eq(formResponses.formId, input.formId)))`
  - `delete` (~line 279): change `.where(eq(formResponses.id, input.id))` to `.where(and(eq(formResponses.id, input.id), eq(formResponses.formId, input.formId)))`

### Green
- Step 3.4: Run tests and verify all pass
- Step 3.5: Verify existing form submission and response management flows still work

### Milestone: FormForge Auth & AuthZ Secured
**Acceptance Criteria:**
- [ ] Unauthenticated POST to `/api/upload/presigned` returns 401
- [ ] `updateStatus`, `bulkUpdateStatus`, and `delete` all include `formId` in WHERE clause
- [ ] Attempting to update/delete a response from a different user's form has no effect
- [ ] Legitimate bulk operations still work for the form owner
- [ ] All phase tests pass
- [ ] No regressions in previous phase tests

**On Completion:**
- Deviations from plan:
- Tech debt / follow-ups:
- Ready for next phase: yes/no

---

## Phase 4: FormForge Validation & Performance (CR-006, CR-007)

### Tests First
- Step 4.1: Write tests for settings validation and CSV export limits
  - File: create `formforge/src/server/trpc/routers/__tests__/form-settings.test.ts`
  - Test cases (CR-006):
    - Valid settings object with all known fields passes validation
    - Settings with unknown extra keys is rejected
    - Settings with invalid types (e.g., `responseLimit: "abc"`) is rejected
    - Empty/omitted settings is valid
  - File: create `formforge/src/server/trpc/routers/__tests__/response-export.test.ts`
  - Test cases (CR-007):
    - Export with fewer than 10,000 responses returns all
    - Export query includes `LIMIT 10000` (or configured max)
    - Response includes `truncated: true` flag when limit is hit
  - Tests should FAIL initially

### Implementation
- Step 4.2: Replace `z.any()` with strict settings schema (CR-006)
  - File: modify `formforge/src/server/trpc/routers/form.ts`
  - At line 136, replace `settings: z.any().optional()` with:
    ```
    settings: z.object({
      notificationEmails: z.string().optional(),
      responseLimit: z.number().int().positive().optional(),
      closeDate: z.string().datetime().optional(),
      redirectUrl: z.string().url().optional(),
      successMessage: z.string().max(500).optional(),
      gdprConsentEnabled: z.boolean().optional(),
    }).optional()
    ```
  - Check if this schema is used in both `create` and `update` mutations — apply to both
  - Verify the settings page UI still saves correctly

- Step 4.3: Add LIMIT to CSV export query (CR-007)
  - File: modify `formforge/src/server/trpc/routers/response.ts`
  - In `exportCsv` procedure (~line 287+), add `.limit(10_000)` to the responses query
  - Add a `truncated` boolean to the return value: `truncated: responses.length >= 10_000`
  - Update the frontend export handler to show a warning if `truncated` is true

### Green
- Step 4.4: Run tests and verify all pass
- Step 4.5: Verify form settings save/load correctly in the dashboard

### Milestone: FormForge Input Validation & Export Safety
**Acceptance Criteria:**
- [ ] `z.any()` removed from form settings — strict Zod schema enforced
- [ ] Submitting `{"malicious": true}` as settings is rejected by validation
- [ ] CSV export query includes `.limit(10_000)`
- [ ] Export response includes `truncated` flag when limit is reached
- [ ] All phase tests pass
- [ ] No regressions in previous phase tests

**On Completion:**
- Deviations from plan:
- Tech debt / follow-ups:
- Ready for next phase: yes/no

---

## Phase 5: DriftLog Race Condition + SnipVault Concurrency (CR-005, CR-008)

### Tests First
- Step 5.1: Write tests for atomic draft release creation and embedding concurrency
  - File: create `driftlog/src/server/webhooks/__tests__/process-merge-event.test.ts`
  - Test cases (CR-005):
    - `getOrCreateDraftRelease` returns existing draft if one exists
    - `getOrCreateDraftRelease` creates new draft if none exists
    - Two concurrent calls to `getOrCreateDraftRelease` for the same org produce exactly 1 draft (race condition test)
  - File: create `snipvault/src/lib/ai/__tests__/embeddings.test.ts`
  - Test cases (CR-008):
    - `generateEmbedding` wraps call in concurrency limiter
    - Firing 20 concurrent calls results in max 5 simultaneous OpenAI API calls
    - Individual embedding generation still works correctly
  - Tests should FAIL initially

### Implementation
- Step 5.2: Make draft release creation atomic (CR-005)
  - File: modify `driftlog/src/server/webhooks/process-merge-event.ts`
  - Wrap the check-then-act block (~lines 204-228) in a `db.transaction()` with `SELECT ... FOR UPDATE`
  - Pattern: `tx.select().from(releases).where(...).for("update").limit(1)` then insert if not found
  - Alternative: add a partial unique index migration and use `ON CONFLICT DO NOTHING` + re-fetch

- Step 5.3: Add concurrency limiter to embedding generation (CR-008)
  - File: modify `snipvault/src/lib/ai/embeddings.ts`
  - Install `p-limit` in snipvault: `npm i p-limit`
  - Create module-level limiter: `const embeddingLimit = pLimit(5);`
  - Wrap the `generateEmbeddingVector` call inside `embeddingLimit(async () => { ... })`
  - Ensure the limiter is shared across all callers (module singleton)

### Green
- Step 5.4: Run driftlog tests and verify all pass
- Step 5.5: Run snipvault tests and verify all pass

### Milestone: Concurrency Safety
**Acceptance Criteria:**
- [ ] Draft release creation is wrapped in a transaction or uses ON CONFLICT
- [ ] No duplicate draft releases possible under concurrent webhook events
- [ ] OpenAI embedding calls are capped at 5 concurrent via p-limit
- [ ] Individual snippet creation still triggers embedding generation
- [ ] All phase tests pass
- [ ] No regressions in previous phase tests

**On Completion:**
- Deviations from plan:
- Tech debt / follow-ups:
- Ready for next phase: yes/no

---

## Phase 6: PulseBoard N+1 Query Fix (CR-004)

### Tests First
- Step 6.1: Write tests for batched alert detection
  - File: create `pulseboard/src/server/cron/__tests__/alert-detection.test.ts`
  - Test cases:
    - `detectIndividualBurnout` with batched check-ins correctly identifies users with 3+ consecutive low-energy days
    - `detectIndividualBurnout` with batched check-ins correctly identifies warning (3 days) vs critical (5 days)
    - Batched fetch returns same results as individual per-user fetch (equivalence test)
    - Org with 0 users produces no alerts
    - User with no recent check-ins is not flagged
  - Tests should FAIL initially

### Implementation
- Step 6.2: Refactor alert detection to use batch queries
  - File: modify `pulseboard/src/server/cron/alert-detection.ts`
  - Replace the per-user loop (~lines 64-68) with a single batched query:
    - Fetch all check-ins for all org users in the past N days with `WHERE userId IN (...)`
    - Group results in memory by userId using a Map
  - Iterate over the grouped results instead of making individual queries
  - Keep the per-manager email loop (batching emails is a separate concern)
  - Preserve all existing alert detection logic (burnout, team dip, low participation)

### Green
- Step 6.3: Run tests and verify all pass
- Step 6.4: Count SQL queries before/after (log or mock) — should be O(orgs) not O(users)

### Milestone: Alert Detection Performance
**Acceptance Criteria:**
- [ ] One DB query per org for check-in data (not per-user)
- [ ] Alert detection produces identical results to the original N+1 implementation
- [ ] Individual burnout, team dip, and low participation detection all still function
- [ ] All phase tests pass
- [ ] No regressions in previous phase tests

**On Completion:**
- Deviations from plan:
- Tech debt / follow-ups:
- Ready for next phase: yes/no

---

## Phase 7: PulseBoard Hardening (CR-009, CR-014, CR-015)

### Tests First
- Step 7.1: Write tests for cron auth, alert email tracking, and digest delivery
  - File: create `pulseboard/src/app/api/cron/__tests__/cron-auth.test.ts`
  - Test cases (CR-009):
    - Request without CRON_SECRET env returns 401
    - Request with wrong Bearer token returns 401
    - Request with correct Bearer token returns 200
    - Missing Authorization header returns 401
  - File: create `pulseboard/src/server/cron/__tests__/alert-email-tracking.test.ts`
  - Test cases (CR-014):
    - Failed email send records manager ID in failedManagerIds on alert
    - Successful email send records manager ID in notifiedManagerIds
    - Partial failure (1 of 3 managers fails) records both notified and failed
  - File: create `pulseboard/src/server/cron/__tests__/digest-delivery.test.ts`
  - Test cases (CR-015):
    - Successful digest email updates sentTo array with manager ID
    - Failed digest email does not add manager to sentTo
    - Digest record created before emails sent (so it exists even if all emails fail)
  - Tests should FAIL initially

### Implementation
- Step 7.2: Fix cron secret validation (CR-009)
  - Files: modify `pulseboard/src/app/api/cron/alerts/route.ts`, `cron/digest/route.ts`, `cron/reminders/route.ts`
  - Change `if (cronSecret && authHeader !== ...)` to `if (!cronSecret || authHeader !== ...)`
  - Apply the same pattern to all three cron route files

- Step 7.3: Add email delivery tracking to alert detection (CR-014)
  - File: modify `pulseboard/src/server/db/schema.ts`
  - Add `notifiedManagerIds` (json array, default `[]`) and `failedManagerIds` (json array, default `[]`) columns to alerts table
  - File: modify `pulseboard/src/server/cron/alert-detection.ts`
  - After successful email send, update alert with manager ID in `notifiedManagerIds`
  - In catch block, update alert with manager ID in `failedManagerIds`

- Step 7.4: Add delivery tracking to digest emails (CR-015)
  - File: modify `pulseboard/src/server/db/schema.ts`
  - Add `sentTo` (json array, default `[]`) column to digests table
  - File: modify `pulseboard/src/server/cron/weekly-digest.ts`
  - After each successful `sendDigestEmail`, update digest record's `sentTo` array

### Green
- Step 7.5: Run all pulseboard tests and verify they pass
- Step 7.6: Generate and apply DB migration for new columns

### Milestone: PulseBoard Notification Reliability
**Acceptance Criteria:**
- [ ] All three cron routes reject requests when CRON_SECRET is unset
- [ ] Alert records track which managers were notified and which failed
- [ ] Digest records track which managers received the email
- [ ] Partial email failures don't block remaining sends
- [ ] All phase tests pass
- [ ] No regressions in previous phase tests

**On Completion:**
- Deviations from plan:
- Tech debt / follow-ups:
- Ready for next phase: yes/no

---

## Phase 8: SnipVault Reliability (CR-012, CR-013, CR-018)

### Tests First
- Step 8.1: Write tests for AI status tracking, null safety, and TRPCError usage
  - File: create `snipvault/src/lib/ai/__tests__/tagging.test.ts`
  - Test cases (CR-013):
    - `findOrCreateTag` returns existing tag ID when tag exists
    - `findOrCreateTag` creates tag and returns ID when tag doesn't exist
    - `findOrCreateTag` handles race condition gracefully (onConflictDoNothing + re-fetch) without crash
    - `findOrCreateTag` throws descriptive error if re-fetch also returns null
  - File: create `snipvault/src/lib/trpc/routers/__tests__/snippet-errors.test.ts`
  - Test cases (CR-018):
    - `getById` for nonexistent snippet throws TRPCError with code NOT_FOUND
    - Error response does not contain stack trace
  - Test cases (CR-012):
    - After snippet creation, fire-and-forget catch handler logs with snippet ID
  - Tests should FAIL initially

### Implementation
- Step 8.2: Fix non-null assertion in tag creation (CR-013)
  - File: modify `snipvault/src/lib/ai/tagging.ts`
  - At ~line 127, replace `return found!.id` with null check:
    ```
    if (!found) throw new Error(`Tag race condition: could not find or create tag "${name}" in workspace ${workspaceId}`);
    return found.id;
    ```

- Step 8.3: Replace generic Error with TRPCError (CR-018)
  - File: modify `snipvault/src/lib/trpc/routers/snippet.ts`
  - At lines 130, 246, 314: replace `throw new Error('Snippet not found')` with `throw new TRPCError({ code: "NOT_FOUND", message: "Snippet not found" })`
  - Import TRPCError if not already imported

- Step 8.4: Improve fire-and-forget error logging (CR-012)
  - File: modify `snipvault/src/lib/trpc/routers/snippet.ts`
  - At lines 217-219 and 283-285: change `.catch((err) => { console.error(...) })` to include snippet ID:
    ```
    .catch((err) => { console.error(`[Snippet ${snippet.id}] Background AI tasks failed:`, err); });
    ```

### Green
- Step 8.5: Run all snipvault tests and verify they pass

### Milestone: SnipVault Error Handling
**Acceptance Criteria:**
- [ ] No `!` non-null assertions on potentially null DB results in tagging.ts
- [ ] Race condition in tag creation throws descriptive error instead of crashing
- [ ] All `throw new Error('Snippet not found')` replaced with `TRPCError({ code: "NOT_FOUND" })`
- [ ] Fire-and-forget catch blocks include snippet ID in log message
- [ ] All phase tests pass
- [ ] No regressions in previous phase tests

**On Completion:**
- Deviations from plan:
- Tech debt / follow-ups:
- Ready for next phase: yes/no

---

## Phase 9: FormForge & DriftLog Hardening (CR-010, CR-011)

### Tests First
- Step 9.1: Write tests for Turnstile warning and AI fallback logging
  - File: create `formforge/src/app/api/submit/__tests__/turnstile.test.ts`
  - Test cases (CR-010):
    - `verifyTurnstile` with no secret set logs warning and returns true
    - `verifyTurnstile` with secret set and valid token returns true
    - `verifyTurnstile` with secret set and invalid token returns false
    - Console.warn is called exactly once when secret is missing
  - File: create `driftlog/src/server/ai/__tests__/generate-changelog.test.ts`
  - Test cases (CR-011):
    - Valid JSON from AI is parsed correctly
    - Malformed JSON triggers console.warn with raw content
    - Malformed JSON still returns default "improvement" category
  - Tests should FAIL initially

### Implementation
- Step 9.2: Add warning log when Turnstile secret is missing (CR-010)
  - File: modify `formforge/src/app/api/submit/[slug]/route.ts`
  - At lines 13-14, add `console.warn("[Turnstile] TURNSTILE_SECRET_KEY not set — bot verification disabled");` before `return true;`

- Step 9.3: Add logging for AI categorization fallback (CR-011)
  - File: modify `driftlog/src/server/ai/generate-changelog.ts`
  - At ~line 300 (catch block), add:
    ```
    console.warn("[AI] Failed to parse category response, defaulting to 'improvement':", { raw: content, error: parseError });
    ```

### Green
- Step 9.4: Run formforge and driftlog tests and verify all pass

### Milestone: Silent Failures Made Visible
**Acceptance Criteria:**
- [ ] Missing Turnstile secret produces a console.warn with descriptive message
- [ ] Malformed AI JSON response is logged with raw content before falling back
- [ ] Normal operation (secrets present, valid AI JSON) produces no warnings
- [ ] All phase tests pass
- [ ] No regressions in previous phase tests

**On Completion:**
- Deviations from plan:
- Tech debt / follow-ups:
- Ready for next phase: yes/no

---

## Phase 10: Cross-Project Rate Limiting + Asset DB Fixes (CR-016, CR-017, CR-019)

### Tests First
- Step 10.1: Write tests for rate limiting, import warnings, and E2E guard
  - File: create `formforge/src/app/api/submit/__tests__/rate-limit.test.ts`
  - Test cases (CR-016):
    - First 10 requests from same IP within 1 minute succeed
    - 11th request from same IP returns 429
    - Requests from different IPs are independently limited
  - File: create `asset-inventory-db/src/server/routers/__tests__/import-warnings.test.ts`
  - Test cases (CR-017):
    - Import validation with all known owner emails produces no warnings
    - Import validation with unknown owner email produces warning listing the email
    - Import with unknown owner still creates asset with null ownerId
  - File: create `asset-inventory-db/src/lib/__tests__/auth-guard.test.ts`
  - Test cases (CR-019):
    - With `NODE_ENV=production`, credentials provider is not added even if `E2E_TESTING=true`
    - With `NODE_ENV=test` and `E2E_TESTING=true`, credentials provider is added
  - Tests should FAIL initially

### Implementation
- Step 10.2: Add rate limiting to FormForge public submission endpoint (CR-016)
  - Install rate limiting package in formforge (e.g., `npm i @upstash/ratelimit @upstash/redis` or implement in-memory rate limiter for simpler setup)
  - File: modify `formforge/src/app/api/submit/[slug]/route.ts`
  - Add rate limiting check at top of POST handler (10 requests per IP per minute)
  - Return 429 with `{ error: "Too many requests" }` when exceeded
  - Apply similar pattern to DriftLog `driftlog/src/app/api/public/track/route.ts` and SnipVault `snipvault/src/app/auth/device/` endpoints

- Step 10.3: Add import validation warnings for unresolved owner emails (CR-017)
  - File: modify `asset-inventory-db/src/server/routers/import-export.ts`
  - In the validation step (~lines 253-273), collect unresolved emails into a `warnings` array
  - Return `warnings` alongside `validatedRows` in the validation response
  - File: modify `asset-inventory-db/src/components/import/validation-results.tsx`
  - Display warnings (amber/yellow) in the validation step UI

- Step 10.4: Add production guard to E2E credentials provider (CR-019)
  - File: modify `asset-inventory-db/src/lib/auth.ts`
  - At line 14, change `if (process.env.E2E_TESTING === "true")` to `if (process.env.E2E_TESTING === "true" && process.env.NODE_ENV !== "production")`

### Green
- Step 10.5: Run all project tests and verify they pass
- Step 10.6: Run asset-inventory-db E2E tests to confirm no regressions

### Milestone: Cross-Project Hardening Complete
**Acceptance Criteria:**
- [ ] FormForge form submission endpoint returns 429 after 10 requests/minute from same IP
- [ ] DriftLog analytics tracking endpoint is rate-limited
- [ ] SnipVault device code endpoint is rate-limited
- [ ] CSV import validation surfaces unresolved owner emails as warnings
- [ ] E2E credentials provider blocked in production even if env var is set
- [ ] Asset Inventory DB E2E tests still pass
- [ ] All phase tests pass
- [ ] No regressions in previous phase tests

**On Completion:**
- Deviations from plan:
- Tech debt / follow-ups:
- Ready for next phase: yes/no

---

## Cross-Phase Concerns

### Integration Tests
After all phases are complete, write integration tests that verify:
- [ ] SnipVault: end-to-end search flow with parameterized filters returns correct results for edge-case languages (e.g., `C++`, `C#`, names with special characters)
- [ ] FormForge: full form lifecycle — create form, submit response via public endpoint (authed upload, rate-limited), bulk manage responses (ownership enforced), export CSV (bounded)
- [ ] PulseBoard: full cron cycle — create check-ins, trigger alert detection (batched), verify alert with email tracking, trigger digest with delivery tracking
- [ ] DriftLog: simulate concurrent PR merges via webhook, verify exactly one draft release created

### Non-Functional Requirements
- **Security scan**: After Phase 3 completion, run `grep -r "sql.unsafe\|z.any()" --include="*.ts"` across all projects to confirm no remaining instances
- **Performance baseline**: After Phase 6, measure PulseBoard alert detection cron execution time with 100+ users to confirm improvement
- **Dependency audit**: Run `npm audit` in each project after any new package installs (Phases 1, 5, 10) and address any high/critical findings
- **Build verification**: After each phase, run `npm run build` in the affected project(s) to confirm no TypeScript errors introduced
