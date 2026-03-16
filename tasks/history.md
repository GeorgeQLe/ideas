# Session History

## 2026-03-13: Code Review & Remediation Planning

**What was done:**
- Conducted comprehensive code review across all 5 sub-projects (DriftLog, PulseBoard, SnipVault, FormForge, Asset Inventory DB)
- Validated findings against specs and plans — withdrew 3 findings that were deliberate design decisions
- Produced `specs/code-review-remediation.md` with 19 validated findings (3 P0, 5 P1, 9 P2, 2 P3)
- Created 10-phase implementation plan in `tasks/todo.md`
- Phase 1 (Test Infrastructure Setup) completed: Vitest installed and smoke tests passing in snipvault, formforge, pulseboard
- Note: Sub-project changes (vitest configs, smoke tests) are in their own repos — committed separately there

**Key files:**
- `specs/code-review-remediation.md` — full remediation doc with 19 CRs
- `tasks/todo.md` — phased plan (source of truth for /run-step)
- `docs/plan.md` — historical copy of the plan

## 2026-03-14: Phase 2 — SnipVault SQL Injection Fix (CR-001)

**What was done:**
- Fixed SQL injection vulnerability in `snipvault/src/lib/trpc/routers/search.ts`
- Replaced string-interpolated filter building + `sql.unsafe()` with parameterized queries using neon's `sql(queryString, params)` callable form
- All user inputs (workspaceId, language, collectionId, tagSnippetIds, query, limit) now go through numbered `$1`..`$N` placeholders
- Created `snipvault/src/lib/trpc/routers/__tests__/search.test.ts` with 4 static analysis + import tests
- All 5 tests passing across 2 test files (Phase 1 smoke + Phase 2 search)

**Key patterns:**
- Used `params.push(val)` returning 1-based index for `$N` placeholder generation
- Used `= ANY($N)` for array filtering instead of `IN (...)`
- Neon callable form `sql(queryString, params)` for dynamic WHERE clauses; tagged template `sql\`...\`` kept for static queries (similar)

## 2026-03-14: Phase 3 — FormForge Auth & AuthZ Fixes (CR-002, CR-003)

**What was done:**
- CR-002: Added `await auth()` check to presigned URL endpoint (`formforge/src/app/api/upload/presigned/route.ts`) — returns 401 for unauthenticated requests
- CR-002: Removed `/api/upload/presigned` from public routes in `formforge/src/middleware.ts`
- CR-003: Added `formId` scoping to 3 response mutations (`updateStatus`, `bulkUpdateStatus`, `delete`) in `formforge/src/server/trpc/routers/response.ts` — prevents cross-form response manipulation
- Created `formforge/src/server/trpc/routers/__tests__/response.test.ts` with 3 static analysis tests (all passing)
- Fixed pre-existing build errors: renamed `client.ts` → `client.tsx` (JSX parse error) and fixed Stripe SDK v20 API change in `billing.ts`
- All 4 tests passing across 2 test files in formforge; TypeScript compiles cleanly

**Key changes:**
- All `.where(eq(formResponses.id, ...))` in mutations now use `.where(and(eq(formResponses.id, ...), eq(formResponses.formId, ...)))`
- Presigned URL route now behind Clerk auth (both middleware route matcher and in-handler check)

## 2026-03-16: Phase 4 Step 4.1 — Write failing tests for CR-006, CR-007

**What was done:**
- Created `formforge/src/server/trpc/routers/__tests__/form-settings.test.ts` — 3 static analysis tests for CR-006 (settings validation):
  - `z.any()` should not appear in settings validation
  - Known settings fields should be present in a strict schema
  - `.strict()` should be used to reject unknown keys
- Created `formforge/src/server/trpc/routers/__tests__/response-export.test.ts` — 2 static analysis tests for CR-007 (CSV export limits):
  - formResponses query in exportCsv should include `.limit()`
  - Export return should include `truncated` field
- All 5 new tests FAIL as expected (red phase of TDD)
- All 4 existing tests still PASS (no regressions)

## 2026-03-16: Phase 4 Step 4.2 — Fix settings validation (CR-006) and CSV export limit (CR-007)

**What was done:**
- CR-006: Replaced `z.any().optional()` with strict Zod schema in `formforge/src/server/trpc/routers/form.ts` — validates 6 known fields (notificationEmails, responseLimit, closeDate, redirectUrl, successMessage, gdprConsentEnabled) with `.strict()` to reject unknown keys
- CR-007: Added `MAX_EXPORT_ROWS = 10_000` limit to CSV export query in `formforge/src/server/trpc/routers/response.ts` and added `truncated` boolean return field
- All 9 formforge tests passing (4 existing + 5 new from Step 4.1)
- Phase 4 complete

## 2026-03-16: Phase 5 — DriftLog Race Condition (CR-005) + SnipVault Concurrency (CR-008)

**What was done:**
- CR-005: Replaced check-then-create race condition in `driftlog/src/server/webhooks/process-merge-event.ts` with atomic `insert().onConflictDoNothing().returning()` + re-fetch pattern in both `processSingleRepoMerge()` and `processMonorepoMerge()`
- CR-005: Created partial unique index migration `driftlog/drizzle/0001_add_releases_draft_unique_index.sql` — `CREATE UNIQUE INDEX ON releases (org_id, repo_id) WHERE status = 'draft'`
- CR-005: Created `driftlog/src/server/webhooks/__tests__/process-merge-event.test.ts` with 4 static analysis tests (atomic pattern + migration)
- CR-008: Added `p-limit@5` to snipvault, wrapped `generateEmbeddingVector()` in `pLimit(5)` concurrency limiter in `snipvault/src/lib/ai/embeddings.ts`
- CR-008: Created `snipvault/src/lib/ai/__tests__/embeddings.test.ts` with 3 tests (limiter import, instance, wrapping)
- All tests passing: driftlog 127/127, snipvault 8/8
- Phase 5 complete

**Key decisions:**
- Used `onConflictDoNothing` instead of `db.transaction()` + `FOR UPDATE` because DriftLog's neon-http driver doesn't support transactions
- p-limit v5 (not v6+) because snipvault has no `"type": "module"` in package.json

## 2026-03-16: Phase 6 — PulseBoard N+1 Query Fix (CR-004)

**What was done:**
- Refactored all 3 alert detection functions in `pulseboard/src/server/cron/alert-detection.ts` to eliminate N+1 query patterns:
  - `detectIndividualBurnout`: Bulk-fetches all check-ins for org users + all managers with `inArray`, groups in memory via `Map`
  - `detectTeamDip`: Bulk-fetches all check-ins for org teams (30 days), computes baseline/recent averages in JS
  - `detectLowParticipation`: Batch-fetches member counts with `GROUP BY` + all check-ins for past 3 days, groups by (teamId, date) in JS
- Created `pulseboard/src/server/cron/__tests__/alert-detection.test.ts` with 4 static analysis tests verifying no per-entity DB queries inside loops
- All 6 pulseboard tests passing (2 smoke + 4 alert-detection)
- Phase 6 complete

**Key patterns:**
- `inArray` from drizzle-orm for batch WHERE IN clauses
- Manual `reduce()` for groupBy (avoids `Map.groupBy` Node version concerns)
- Alert duplicate checks still per-entity (low volume, only when thresholds exceeded)

## 2026-03-16: Phase 7 — PulseBoard Hardening (CR-009, CR-014, CR-015)

**What was done:**
- CR-009: Fixed cron auth bypass in all 3 route files — changed `cronSecret &&` to `!cronSecret ||` so requests are rejected when CRON_SECRET env var is unset
- CR-014: Added `notifiedManagerIds` and `failedManagerIds` json columns to alerts schema; modified `alert-detection.ts` to use `.returning()` for alert insert and track successful/failed email sends per manager
- CR-015: Added `sentTo` json column to digests schema; modified `weekly-digest.ts` to collect manager IDs on successful email delivery and update digest record
- Created 3 new test files: `cron-auth.test.ts` (3 tests), `alert-email-tracking.test.ts` (3 tests), `digest-delivery.test.ts` (2 tests)
- All 14 pulseboard tests passing (2 smoke + 4 N+1 + 3 cron-auth + 3 alert-tracking + 2 digest-tracking)
- Phase 7 complete

**Key changes:**
- Added `json` import to schema.ts (alongside existing `jsonb`)
- Alert insert now uses `.returning()` to capture `alertRecord.id` for subsequent update
- Digest already used `.returning()` — just needed `sentTo` tracking added to the email loop
