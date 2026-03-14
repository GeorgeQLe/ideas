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
