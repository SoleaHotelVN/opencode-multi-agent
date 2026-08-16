---
description: "Quick fix: bug fixes and simple changes — auto-detect backend/frontend → dispatch → verify"
agent: build
---

You are fixing a bug or making a simple change. Speed matters. No planning, no review unless critical.

**Task:** $ARGUMENTS

If `$ARGUMENTS` is empty, stop: "Usage: `/build <bug description or simple change>`"

---

## Step 1: Auto-Detect + Quick Lookup

**GRAPHIFY FIRST:** `graphify query "<concept>"` or `graphify explain "<symbol>"` before reading files. Faster than grep.

| Task Location | Agent |
|---------------|-------|
| `src/domain/`, `src/application/`, `src/infrastructure/`, `src/app/api/`, `prisma/` | `@code` |
| `src/presentation/`, `src/app/` (pages), `src/components/ui/` | `@design` |
| `src/app/_actions/` | `@server-actions` |

**Quick Path (self-fix):** 1 file, no business logic, obvious fix (typo, import, CSS). Else → dispatch.

---

## Step 2: Dispatch + Verify

### Test-First (bugs with business logic)

For bugs touching business logic, domain rules, or validation:
1. **Dispatch `@test` FIRST** to write a failing reproduction test (RED)
2. **Dispatch `@code` with test output** to fix (GREEN)
3. Verify: `npm test && npm run lint && npm run typecheck`

If `@code` reports GREEN without RED evidence from `@test` -> reject, re-dispatch with explicit instruction to check test output.

### Direct Fix (no business logic)

- **Backend fix:** `@code` -> verify: `npm test && npm run lint && npm run typecheck`
- **Frontend fix:** `@design` -> verify: `npm run build && npm run lint`
- **Server action:** `@server-actions` -> verify: `npm run typecheck`

### Verification Loop

After dispatch, check results. If tests/lint/typecheck FAIL -> re-dispatch fix agent with error output. Max 2 retry rounds. If still failing after 2 rounds -> report what's broken and let user decide.

---

## Step 3: Ship (Direct to Main)

This project pushes **directly to main**. No branch, no PR.

`git add -A` -> commit -> `git push origin main` -> report.
Commit message: match repo style (check `git log --oneline -5`).
Skip review/docs unless complex or user-facing. If user-facing or 3+ files changed -> run `/review` first.

---

## When NOT to use /build

`/workflow` for: new features, 3+ files, business logic changes, needs review.
`/plan` for: comprehensive planning needed.
