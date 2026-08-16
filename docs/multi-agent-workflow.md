# Multi-Agent Workflow — Reference Guide

This is the reference documentation for the workflow commands. See individual command files for the flow itself.

## Commands Overview

| Command | Purpose | Typical use |
|---------|---------|-------------|
| `/plan <feature>` | Iterative feature planning (owner review rounds) | New feature, before any code |
| `/workflow <task or plan>` | Execute implementation | After `/plan`, or for standalone tasks |
| `/build <bug or fix>` | Quick fix (test-first for bugs with logic) | Bug fixes, simple changes |
| `/review [target]` | Code review | Before push (direct to main) |
| `/design-review <component>` | Design spec vs live vs Pencil mock | UI redesign, design compliance |

## Task Spec Format

When dispatching subagents via Task tool, use this format:

```
## Task
[Clear description of what to implement]

## Acceptance Criteria
- [ ] [Criterion 1 — specific, testable]
- [ ] [Criterion 2]

## Code Context
[Relevant code snippets — READ actual files first, don't just reference paths]

## Files to Modify
- `path/to/file.ts` — [what to change]
- `path/to/new-file.ts` (create) — [what to create]

## Integration Contract (if applicable)
- Public interfaces: [signatures, endpoints, config keys affected]
- Invariants: [assumptions other code relies on]
- Task interactions: [which other tasks depend on this]
```

## Task Type → Agent Mapping

| Task Type | Agents Used | Flow |
|-----------|-------------|------|
| **Backend** | @test, @code, @server-actions | @test → @code → @server-actions |
| **Frontend** | @design, @ui-test | @design → @ui-test |
| **Full Stack** | All 5 | @test → @code → @server-actions → @design → @ui-test |

### When to use each agent

| Agent | Dispatch when task involves |
|-------|-----------------------------|
| `@test` | Business logic, domain rules, use cases, validation, API |
| `@code` | Backend implementation, entities, services, repositories, API routes, DB |
| `@server-actions` | Form submissions, server action bridge, Zod validation |
| `@design` | React components, page layouts, forms, styling, accessibility |
| `@ui-test` | Component tests, hook tests, rendering verification |
| `@check` | Risk review, security, edge cases, correctness |
| `@simplify` | Complexity review, overengineering check |
| `@docs` | Documentation updates |
| `@business-logic-architect` | Business rule analysis, gap finding, requirement translation |

## BLA Decision Table

| Condition | BLA Required? |
|-----------|---------------|
| Task involves business rules, domain logic, validation | YES |
| New feature | YES |
| Multi-file changes (2+ files) | YES |
| Ambiguous requirements | YES |
| State machine / workflow changes | YES |
| Config-only (eslint, prettier, tsconfig) | SKIP |
| Pure styling (CSS, Tailwind classes) | SKIP |
| Obvious bug fix (clear error, single-line) | SKIP |
| Documentation only | SKIP |
| ≤1 file AND no business logic | SKIP |
| Unsure | YES (err on the side of caution) |

## Quick Path Decision Table

| Condition | Self-fix or Dispatch? |
|-----------|----------------------|
| 1 file, no business logic, obvious fix | Self-fix OK |
| Fix typo in error message | Self-fix OK |
| Change CSS color/spacing | Self-fix OK |
| Add missing import | Self-fix OK |
| 2+ files need changes | DISPATCH |
| Business logic involved | DISPATCH |
| New feature or refactor | DISPATCH |
| Tests need to be written | DISPATCH |
| Server actions needed | DISPATCH |
| UI components with logic | DISPATCH |

## Verification Commands

| Agent | Commands |
|-------|----------|
| `@code` | `npm test`, `npm run lint`, `npm run typecheck` |
| `@design` | `npm run build`, `npm run lint` |
| `@server-actions` | `npm run typecheck` |
| `@test` | `npm test` (verify tests fail RED) |
| `@ui-test` | `npm test` |

## Review Severity Rules (Direct-to-Main)

This project pushes **directly to main** — no branch, no PR, no issue tracker. All BLOCK/HIGH/MEDIUM must be fixed before push.

| Severity | Action | Re-review? |
|----------|--------|-----------|
| **BLOCK** | Must fix before push | Yes |
| **HIGH** | Must fix before push | Yes |
| **MEDIUM** | Must fix before push | Yes |
| **LOW** | Document only | No |

- ≤2 issues → 1 re-review round after fixes
- 3-5 issues → up to 2 re-review rounds
- 6+ issues with BLOCK/HIGH → up to 2 re-review rounds, then escalate to user
- **Max 2 re-review rounds.** After 2 rounds: BLOCK/HIGH remaining → STOP, report to user. MEDIUM/LOW remaining → push, document as `// TODO:`.

## Test-First Enforcement

For business logic tasks: `@test` MUST run before `@code`. No exceptions. The workflow command dispatches `@test` first, then `@code` with RED test output attached. If `@code` reports GREEN without RED evidence from `@test` → reject, re-dispatch.

## Agent Bash Permissions

> `npm run dev` is intentionally NOT allowed for any subagent — it starts a long-running dev server that hangs the agent. Use `npm run build` for verification.

> All agents use `"*": deny` catch-all. Only explicitly listed commands are allowed. Deny rules are NOT repeated after allows (last-match-wins semantics — trailing denies would override earlier allows).

| Agent | Allowed Commands |
|-------|-----------------|
| `@code` | `npm test*`, `npm run test*`, `npx vitest*`, `npm run lint*`, `npx eslint*`, `npm run typecheck*`, `npx tsc*`, `npm run build*`, `npx prisma migrate*`, `npx prisma generate*`, `npx prisma db push*`, `mkdir*`, `ls*`, `rg*`, `diff*`, `git diff --name-only*`, `graphify*` |
| `@design` | `npm run build*`, `npm run typecheck*`, `npm run lint*`, `npm test*`, `npx vitest*`, `npx eslint*`, `npx tsc*`, `mkdir*`, `ls*`, `rg*`, `diff*`, `git diff --name-only*`, `graphify*` |
| `@server-actions` | `npm test*`, `npm run test*`, `npx vitest*`, `npm run lint*`, `npx eslint*`, `npm run build*`, `npm run typecheck*`, `npx tsc*`, `mkdir*`, `ls*`, `rg*`, `diff*`, `git diff --name-only*`, `graphify*` |
| `@test` | `npm test*`, `npm run test*`, `npx vitest*`, `npm run lint*`, `npx eslint*`, `npx tsc*`, `npm run typecheck*`, `mkdir*`, `ls*`, `wc*`, `which*`, `diff*`, `rg*`, `git diff --name-only*`, `graphify*` |
| `@ui-test` | `npm test*`, `npm run test*`, `npx vitest*`, `npm run lint*`, `npx eslint*`, `npm run typecheck*`, `npx tsc*`, `mkdir*`, `ls*`, `rg*`, `diff*`, `git diff --name-only*`, `graphify*` |
| `@check` | `graphify*`, `git diff --name-only*`, `ls*`, `rg*`, `diff*` |
| `@simplify` | `graphify*`, `git diff --name-only*`, `ls*`, `rg*`, `diff*` |
| `@business-logic-architect` | `graphify*`, `git diff --name-only*`, `ls*`, `rg*`, `diff*` |

| Code | Meaning | Valid RED? |
|------|---------|-----------|
| `MISSING_BEHAVIOR` | Function/class doesn't exist yet | Yes |
| `ASSERTION_MISMATCH` | Code exists but wrong behavior | Yes |
| `TEST_BROKEN` | Test itself has errors | No — fix first |
| `ENV_BROKEN` | Environment issue | No — BLOCKED |

## File Boundaries

| Agent | Can Touch | Cannot Touch |
|-------|-----------|-------------|
| `@code` | `src/domain/`, `src/application/`, `src/infrastructure/`, `src/app/api/`, `prisma/` | UI, server actions |
| `@design` | `src/presentation/`, `src/app/` (pages), `src/components/ui/` | Domain, API, DB |
| `@server-actions` | `src/app/_actions/` | Domain, UI, DB |
| `@test` | `tests/unit/domain/`, `tests/unit/application/`, `tests/unit/_actions/` | Production code |
| `@ui-test` | `tests/unit/components/`, `tests/unit/presentation/` | Production code |
| `@docs` | `docs/`, `README.md` | Source code |

## Agent Capabilities

| Agent | Read | Write | Bash | Internet |
|-------|------|-------|------|----------|
| `@code` | Yes | Listed files | Sandboxed | Yes |
| `@design` | Yes | Listed files | Sandboxed | Yes |
| `@server-actions` | Yes | Listed files | Sandboxed | No |
| `@test` | Yes | Test files only | Sandboxed | No |
| `@ui-test` | Yes | Test files only | Sandboxed | No |
| `@check` | Yes | No | graphify/rg only | Yes |
| `@simplify` | Yes | No | graphify/rg only | Yes |
| `@docs` | Yes | Docs only | No | No |
| `@business-logic-architect` | Yes | Docs only | graphify/rg only | Yes |

## Example Task Spec

```
## Task
Add a `validateBookingDates()` function that checks startDate < endDate
and dates are not in the past.

## Acceptance Criteria
- [ ] Function throws `ValidationError` if startDate >= endDate
- [ ] Function throws `ValidationError` if dates are in the past
- [ ] Function returns true if valid
- [ ] Unit tests cover all three cases
- [ ] Vietnamese error messages

## Code Context
// src/domain/entities/booking.entity.ts (current)
export class Booking {
  startDate: Date;
  endDate: Date;
  // ...
}

## Files to Modify
- `src/application/dto/booking.dto.ts` — add validateBookingDates()

## Test File
- `tests/unit/application/dto/booking-dto.test.ts` (create)

## Constraints
- Follow existing validation pattern (Zod schema)
- Use Vietnamese for error messages
- Keep function pure
```

## `/plan` Command Reference

### Iterative Planning Process

Planning is **iterative** -- not one-shot. Owner reviews + refines before `/workflow` execution.

Phase 1 (BLA) -> Phase 2 (audit, parallel) -> Phase 3 (standards, parallel) -> Phase 4 (synthesize) -> OWNER REVIEW ROUND 1 -> [feedback] -> refine -> ROUND 2 (if needed) -> ROUND 3 (max 3) -> Final Plan (approved) -> ready for /workflow

Minimum 1 owner review round. Complex features: 2-3 rounds. Max 3 rounds.

### Phases

1. **Requirements Analysis** -- `@business-logic-architect` analyzes domain, assesses scope
2. **Infrastructure Audit** -- `@code` + `@design` scan codebase in parallel
3. **Standards Check** -- `@check` + `@simplify` review in parallel, validate scope
4. **Synthesize + Draft** -- Reconcile findings, write draft plan
5. **Owner Review (iterative)** -- Present to user, get feedback, refine. Max 3 rounds.
6. **Final Plan** -- Approved plan ready for `/workflow`

### Scope Splitting

If scope too large (20+ tasks, 4+ independent sub-features, 8+ HIGH risks, 12+ hours):
- Split into `docs/plans/<feature>-part1.md`, `<feature>-part2.md`
- Order by dependency. Each part gets own owner review round.

### Plan File Structure

- Summary
- Business Context (from @BLA)
- Scope Assessment (single/split + effort estimate)
- Files to Create/Modify (with agent assignment)
- Task Specs (per agent, pre-written, with @test FIRST for business logic)
- Dependency Order (test-first enforced)
- Risks & Mitigations (from @check)
- Standards Notes (from @check / @simplify)
- API Contract (if backend <-> frontend)
- Testing Strategy (unit/integration/manual)
- Owner Review Log (rounds + approval)
### Cost Optimization

- **Cache hits:** agents share plan file (static) via references, not full copy
- **Context isolation:** each subagent gets only relevant sections
- **Output structure:** all agents output in standard format for easy parsing

## Cache Optimization Guide

OpenCode Go supports input cache with **99% cost reduction** on cached reads.

### Max cache hit rules:
1. Keep agent prompt files IMMUTABLE after setup
2. Structure prompts: static content at TOP, dynamic at BOTTOM
3. Send only relevant snippets to subagents (20-50 lines, not full files)
4. Use plan files as external memory instead of chat history
5. Never modify prompt files unnecessarily — changes invalidate cache

### Cache-friendly context passing:
```diff
- "@code: here's the full analysis from @BLA [2000 lines]"
+ "@code: @BLA found 3 gaps — see docs/plans/xxx.md Section 2"
```