---
description: "Execute implementation: read plan file OR lightweight plan → dispatch subagents → review → ship"
agent: build
---

You are executing the multi-agent workflow based on a plan.

**Input:** $ARGUMENTS

If $ARGUMENTS is empty, stop: "Usage: `/workflow @docs/plans/<plan>.md` or `/workflow <task description>`"

---

## CRITICAL: Your Role

**You are an ORCHESTRATOR. You do NOT implement code — except for trivial quick fixes.**

Rules:
- 2+ files or business logic → DISPATCH subagent via Task tool
- 1 file, no business logic, obvious fix → MAY self-fix
- If editing 2nd file or touching business logic → STOP, dispatch subagent

---

## Phase 1: Parse Input

- **Plan spec (`@docs/plans/xxx.md`):** Read → extract task specs → skip to Phase 3. This is the SPEC file — final state only, no review commentary (per `docs/PLAN-STANDARDS.md` 2-file convention).
- **Audit file guard:** If input is `@docs/plans/xxx.review.md` (`.review.md` suffix = audit log), do NOT execute it — warn user and redirect to the spec sibling (strip `.review` → read `xxx.md` instead). Audit log is historical, not execution-relevant.
- **Task description (no `@`):** Determine scope → lightweight plan (skip BLA if obvious) → Phase 3.
- **Ambiguous:** Ask 2-3 clarifying questions max.

---

## Phase 2: Lightweight Plan (task descriptions only)

1. Business logic? → dispatch `@business-logic-architect`
2. Pick flow: **Backend** @test→@code→@server-actions | **Frontend** @design→@ui-test | **Full Stack** all
3. Write minimal task specs

---

## Phase 3: Implement (TEST-FIRST)

**GRAPHIFY:** When dispatching subagents, include in Task prompt: *"Before grep/Read, run graphify query or browse graphify-out/wiki/. Fall back to raw files only if graph doesn't answer."*

### ⚠ TEST-FIRST — DO NOT SKIP

For any task touching business logic, domain rules, or validation:
1. **Dispatch `@test` FIRST** — write failing tests (RED)
2. **THEN dispatch `@code`** with test output attached — implement until tests pass (GREEN)
3. **Verify RED→GREEN evidence** in `@code` output before proceeding

**Common skip reasons (INVALID):** "it's just a small change", "tests already exist", "I'll add tests after". If the plan specifies `@test` in dependency order → `@test` runs first. No exceptions for business logic.

**Skip `@test` ONLY when:** frontend-only (no logic), pure styling, config-only, or plan explicitly says NOT_TESTABLE.

### DISPATCH — do NOT implement yourself:

- `@test` → `@code` (TDD). Skip: frontend-only, NOT_TESTABLE.
- `@code` — with task spec + test output. Skip: frontend-only.
- `@server-actions` — depends on `@code`. Skip: no server actions.
- `@design` — with data shapes from `@code`. Skip: backend-only.
- `@ui-test` — depends on `@design`. Skip: backend-only, pure styling.

Follow plan dependency order. Dispatch order: backend flow → frontend flow.

### ⚠ Schema changes → migration REQUIRED

If `@code` (or any agent) touches the project's schema source, the implementing agent MUST create + apply a proper migration — never bypass migrations (which causes silent drift between environments). Exact tool depends on the project stack:

- **Prisma projects** (`prisma/schema.prisma` exists): run `npx prisma migrate dev --name <snake_case_name>` (underscore, not hyphen — Prisma rejects hyphens in some versions). NEVER `npx prisma db push` (bypasses migrations). Confirm migration file created under `prisma/migrations/<timestamp>_<name>/migration.sql` and applied to local DB.
- **Alembic projects** (`alembic.ini`/`alembic/` exists): run `alembic revision --autogenerate -m "<name>"` then `alembic upgrade head`. Verify migration script in `alembic/versions/<rev>_<name>.py` and applied to DB.
- **Other stacks:** consult the project's `docs/DEPLOYMENT.md` + project `AGENTS.md` "Schema Migration Workflow" section for the correct tool + commands.

The agent reports the migration name in its output so the orchestrator can verify in Phase 4d.

**Reject agent output if it used a bypass command** (`prisma db push`, raw DDL, manual SQL without a migration file) — re-dispatch with explicit migration instruction.

---

## Phase 4: Review → Fix → Ship

### 4a. Review

Dispatch `@check` (+ optionally `@simplify`).

**UI-heavy work** — determined by the plan's `UI-heavy: yes/no` header (set by `/plan`). If the plan lacks the flag, fall back to these criteria:
- Plan has a `docs/design/{name}.md` spec (Pencil mock reference exists)
- Plan creates/modifies a **page** (`src/app/**/page.tsx`) or a **layout** component
- Plan creates/modifies a **new component** in `src/presentation/components/**` or `src/components/ui/**` (not a trivial 1-line tweak)
- Plan touches **3+ UI files** (`.tsx`/`.css`) in one feature
- Plan changes **visual tokens** (globals.css, theme, brand colors, typography, spacing)

**NOT UI-heavy** (skip `@design` audit): backend-only, server-action-only, pure logic, config-only, docs-only, or a trivial single-file UI tweak (e.g. one className change, one icon swap).

When UI-heavy: ALSO dispatch `@design` for a **design-compliance audit** in parallel with `@check`. `@check` reviews code correctness/architecture only — it does NOT check design-standard compliance.

**`@design` design-compliance audit prompt:**
> "Design-compliance audit for {component}. Read `docs/design/{name}.md` (if exists), `docs/DESIGN-STANDARDS.md`, `docs/BRAND-GUIDELINE.md`, and `globals.css`. Review the changed UI files against the spec. Check:
> 1. Hardcoded hex in JSX (`#[0-9a-fA-F]` in className/style) — must use CSS variables/Tailwind tokens
> 2. Light-only classes (`bg-white`, `bg-gray-*`, `text-black`) — must work in dark mode
> 3. Missing CSS variables (globals.css)
> 4. Inline fontFamily (should use Tailwind classes)
> 5. Wrong lucide icons/sizes
> 6. Typography/spacing/radius/shadows match spec
> 7. Responsive breakpoints work (1→2→3 cols)
> Return: file:line references + severity (BLOCK/HIGH/MEDIUM/LOW)."

Design-compliance issues feed into the same Fix-All policy (4b) and re-review rounds (4c) as `@check` issues.

### 4b. Fix ALL Issues (Fix-All Policy)

**Fix EVERY issue found by `@check` — BLOCK, HIGH, MEDIUM, and LOW.** No deferring, no documenting-only, no follow-up tickets. This project pushes directly to main, so all issues must be resolved before ship.

- **BLOCK / HIGH** → must fix, dispatch fix agent (`@code`/`@design`/`@server-actions` per file boundary)
- **MEDIUM** → must fix
- **LOW** → must fix (no exceptions)

### 4c. Re-Review Rounds (max 4)

After fixing, **re-dispatch `@check`** to verify fixes. For UI-heavy work, also re-dispatch `@design` to verify design-compliance fixes (same dual-verify + new-issue-scan rules below).

**NO re-review needed when** the review verdict was **ACCEPTABLE** AND the round found **no MEDIUM (or higher)** issues — only LOW or none. In that case fix the LOW issues and ship directly (skip 4c).

**This applies to EVERY round, including re-review rounds.** If a re-review round comes back ACCEPTABLE with no MEDIUM+ issues, stop re-reviewing and ship — do not run another round just because a prior round had issues.

**Re-review required when** previous round had:
- Any BLOCK/HIGH issue, OR
- Any MEDIUM issue, OR
- 4+ LOW issues

**Each re-review round does BOTH:**
1. **Verify prior fixes** — confirm each issue from the previous round is actually resolved (not just claimed). Re-check the exact `file:line` locations.
2. **Scan for NEW issues** — the previous round may have missed things (fixes can introduce regressions, or the reviewer's earlier scope was narrower). Instruct `@check` to re-run its full review framework on the changed files, not just diff against the old issue list. Report any newly-found issues at their severity.

**Dispatch prompt for re-review must state explicitly** (adapt `{framework}` per agent — `@check` = "full review framework", `@design` = "full compliance checklist"):
> "Round N re-review. (1) Verify each issue from round N-1 is fixed at its exact location. (2) Re-run your {framework} on the changed files — do NOT limit yourself to the previous issue list. Report any NEW issues missed in earlier rounds."

| Issue Count | Severity Mix | Re-Review Rounds |
|-------------|-------------|------------------|
| ≤2 issues | Any | 1 re-review |
| 3-5 issues | Any | Up to 2 re-reviews |
| 6+ issues | Has BLOCK/HIGH | Up to 8 re-reviews, then escalate to user |

**Max 8 re-review rounds.** If after 8 rounds issues remain:
- Any remaining → **STOP, report to user** (do not push broken code)

### 4d. Migration & Schema Verification (if schema touched)

If any agent reported a schema change (schema source file modified, migration file created, or migration name in its output), verify BEFORE shipping. Skip this step only if no schema touch occurred this workflow.

**Project stack detection** — pick the right tool for verification:

**Prisma projects** (`prisma/schema.prisma`):
1. `npx prisma migrate status` → must report `Database schema is up to date`. Confirm `git status prisma/migrations/` → every new migration dir (`<timestamp>_<name>/migration.sql`) is staged or stageable. Orphan migration folders from cancelled attempts → delete, don't commit.
3. `npx prisma validate` passes; `migrate status` shows zero drift between schema, migrations, and DB. If drift detected → STOP, re-run `npx prisma migrate dev` to reconcile before push.
4. No bypass migrations via `prisma db push` (per Phase 3 rule). If used → STOP, convert to proper migration via `npx prisma migrate dev --name <name>`.

**Alembic projects** (`alembic.ini`/`alembic/`):
1. `alembic current` → must show the head revision applied to the DB.
2. `git status alembic/versions/` → every new migration file (`<rev>_<name>.py`) is staged; orphan files from cancelled attempts → delete.
3. `alembic history` is clean and unbroken; latest revision matches the one applied locally.
4. `alembic check` (Alembic 1.10+) — no autogenerate drift between models and DB. If drift detected → STOP, regenerate via `alembic revision --autogenerate` then retry.

**Other stacks:** consult project `AGENTS.md` "Schema Migration Workflow" section. If absent → STOP + ask user.

**If any check fails → STOP, fix before `git add`.** Don't push a commit containing schema changes without a matching migration file — silent drift between environments (per each project's `AGENTS.md` migration rules; the same drift risk applies to every DB-backed project).

### 4e. Ship (Direct to Main)

This project pushes **directly to main**. No branch, no PR.

```
git add -A
git commit -m "<conventional message>"
git push origin main
```

Commit message: match repo style (check `git log --oneline -5`). Vietnamese acceptable if repo uses it.

### 4f. Verify Deploy (REQUIRED — do not skip)

After push, **verify the deploy succeeds** before considering the workflow done:

1. **CI must pass** — if the repo has CI (GitHub Actions, etc.), confirm the run for the pushed commit is green. If CI fails → fix and re-push until green.
2. **Deploy must succeed** — trigger/confirm the deploy (see the repo's `docs/RUNBOOK.md` / `docs/DEPLOYMENT.md` for the exact command).
3. **Health check** — confirm the app is healthy after deploy (e.g. `curl <app-url>/health` returns OK, container is up, migrations ran cleanly).
4. **VPS migration status (if schema touched in this workflow)** — after deploy, verify migrations applied on the VPS:
   - SSH to VPS (see the project `docs/DEPLOYMENT.md` for the exact ssh command — projects may use a non-default key).
   - **Prisma:** `cd <app-dir> && npx prisma migrate status` → must show `Database schema is up to date`; if pending → `npx prisma migrate deploy` (or the documented migration-apply command).
   - **Alembic:** `cd <app-dir> && alembic current` → head revision applied. If pending → `alembic upgrade head` (or whatever the project's deploy script does).
   - **Other stacks:** per project `AGENTS.md` Schema Migration Workflow section.
   - If VPS reports drift vs local → reconcile per the project's migration rules in `AGENTS.md` (schema fields without corresponding migrations cause silent drift).

**If deploy fails or health check fails → FIX the issue and re-push until the deploy succeeds.** Do NOT mark the workflow complete while the deployed app is broken. Only after deploy + health check + (if applicable) VPS migration check pass, proceed to `@docs`.

Then: dispatch `@docs` → update affected docs → report summary.

**Archive:** If workflow used a plan spec (`@docs/plans/xxx.md`), move it to `docs/plans/archive/` after successful completion. If the 2-file convention was followed (per `docs/PLAN-STANDARDS.md`), also move the audit sibling `docs/plans/xxx.review.md` to `docs/plans/archive/` — keep both as post-mortem. (If audit file does not exist — e.g. lightweight/older plan — skip silently.)

---

## Failure Handling

Unrecoverable → commit `wip: incomplete workflow` → push → summarize. Never hang on prompts.
