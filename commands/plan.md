---
description: "Feature planning: iterative requirements analysis -> audit -> standards -> owner review -> plan file ready for /workflow"
agent: build
---

You are generating an implementation plan for a new feature. The output is a plan file that `/workflow` can execute directly.

**Feature:** $ARGUMENTS

If $ARGUMENTS is empty, stop: "Usage: `/plan <feature description>`"

---

## CRITICAL: Your Role

**You are a PLANNER. You do NOT implement.**

You MUST: gather context from subagents, synthesize findings, write plan file, iterate with owner.
You MUST NOT: write production code, edit source files, implement anything.

**Cache optimization:** Send only relevant code snippets to each subagent, not full files.

---

## Overview: Iterative Planning

Planning is **iterative**, not one-shot. You run phases, present findings to owner, get feedback, refine. Minimum 1 owner review round. Complex features may need 2-3 rounds before the plan is ready for `/workflow`.

```
Phase 1 (BLA) -> Phase 2 (audit, parallel) -> Phase 3 (standards, parallel)
-> Phase 4 (synthesize) -> OWNER REVIEW ROUND 1
  -> [feedback] -> refine -> OWNER REVIEW ROUND 2 (if needed)
  -> [feedback] -> refine -> OWNER REVIEW ROUND 3 (max 3)
-> Final Plan -> ready for /workflow
```

**If scope is too large** (see Scope Splitting below), split into multiple plan files. Owner decides which to execute first.

---

## Plan File Convention (2-file — MANDATORY)

Every feature plan produces **2 files** from the very first draft. Follow `docs/PLAN-STANDARDS.md` (project repo) if it exists — the rules below mirror it.

| File | Role | Mutability | Read by |
|------|------|-----------|---------|
| `docs/plans/{feature}.md` | **Spec** — decisions + final code/intent. This is what `/workflow` reads. | **Replace-in-place** (refine affected section) | `/workflow`, reviewer round N, future reader |
| `docs/plans/{feature}.review.md` | **Audit log** — review rounds + resolutions. | Append-only, never trimmed | Optional: post-mortem. Reviewer round N does NOT read |

### Hard rules

1. **Spec = final state only.** NEVER paste reviewer commentary inline — no "### Vấn đề @check tìm ra", no "### Vấn đề review 6-agent (Round 2)", no "FIX R2-B3 — order rõ..." inside code blocks. These belong in the audit file.
2. **Refine in-place.** Every fix from a review round → UPDATE the affected section in spec (state replacement). NEVER append "### Round N also found..." into spec. Spec only grows when genuine new file/code/decision content is added.
3. **Reviewer reads spec only.** When dispatching a reviewer for round N, give it `docs/plans/{feature}.md` ONLY — never the audit file. Fresh eyes, no contamination from prior round's narrative.
4. **Audit entry format** (append to `{feature}.review.md`):
   ```markdown
   ## Round N — YYYY-MM-DD — @<reviewer>

   ### Findings
   - R6-B1: <one-line description> → §<spec section>
   - R6-H3: <one-line description> → §<spec section>

   ### Resolution
   - R6-B1 → §X updated: <what changed in spec>
   - R6-H3 → §Y updated: <what changed in spec>
   ```

### Why

Observed drift: `transfer-collateral-spec-review.md` grew to 276KB / 1761 lines (3.5× the clean final's 77KB / 926 lines) because commentary was pasted inline instead of refine-in-place. Reviewer round N had to lọc 835 dòng commentary cũ để spec hiện hành. The 2-file split keeps spec always-current and cheap to read.

---

## Phase 1: Requirements Analysis

**Dispatch `@business-logic-architect`** via Task tool.

Input:
- User request: $ARGUMENTS
- Existing docs: ARCHITECTURE.md, BUSINESS-LOGIC.md, DATABASE.md, API.md, MASTER-PLAN.md

BLA analyzes:
- Business rules impacted by this feature
- Domain gaps that need filling
- Business decisions that apply
- Suggested approach with rationale
- Acceptance criteria
- **Scope assessment:** Is this one plan or needs splitting?

**Output format:** structured analysis with sections: business context, gaps, approach, risks, scope assessment.

---

## Phase 2: Infrastructure Audit (parallel)

**GRAPHIFY FIRST:** Use graphify query/path/explain and graphify-out/wiki/ for audit. Raw files only as fallback.

Dispatch `@code` + `@design` simultaneously:

**@code** — Backend: existing entities/VOs/enums/use cases/API routes. Missing pieces. DI readiness. Prisma schema changes needed.
**@design** — Frontend: existing components/pages/forms. Missing pieces. Data shapes needed. UI patterns. Design specs in docs/design/.

Output: "exists" vs "needs creation" lists with file paths.

---

## Phase 3: Standards Check (parallel)

Dispatch `@check` (+ optionally `@simplify`). Review the combined findings from Phase 1+2:
- Architecture compliance (Clean Architecture layer rules)
- Security risks
- Testability
- Business logic compliance
- Over-engineering risks (MVP phase context)
- **Scope validation:** Is this plan executable in one `/workflow` session, or too large?

### @simplify findings — DO NOT auto-incorporate

`@simplify` findings are **advisory only**. Do NOT automatically incorporate them into the plan — not in Phase 4, and not in any later owner-review round.

Before incorporating any `@simplify` finding:
1. **Self-check against long-term project vision** — is the simplification aligned with the project's long-term direction, or is it a short-term shortcut that conflicts with planned architecture/scale? (e.g. a simplification that removes a layer boundary, skips a planned abstraction, or trades correctness for brevity is likely short-term.)
2. **Present each finding to the owner for approval** — show the finding, your long-term assessment, and the proposed plan change. Incorporate ONLY after the owner explicitly approves.

`@check` findings (BLOCK/HIGH/MEDIUM) are still incorporated directly — this gate applies to `@simplify` only.

---

## Phase 4: Synthesize + Draft Plan

Synthesize all findings. Reconcile:
- API <-> UI alignment (data shapes match)
- Business <-> implementation mapping (every rule has a task)
- Standards compliance (every @check risk has mitigation)

Write DRAFT plan to `docs/plans/<slugified-feature-name>.md` using the Plan Structure below.

**2-file convention (mandatory):**
1. **Spec** (`docs/plans/<name>.md`) — final state only: decisions, code/intent, file list, invariants, ACs. Synthesize findings — do NOT paste commentary inline. No "### Vấn đề @check", no "### Round 2 findings", no "FIX R2-B3" narrative chèn giữa code block. Those are audit file material.
2. **Audit file** (`docs/plans/<name>.review.md`) — initialize with all findings gathered in Phase 1-3 (one section per agent dispatched): `## Phase 1 — @BLA`, `## Phase 2 — @code`, `## Phase 2 — @design`, `## Phase 3 — @check`, `## Phase 3 — @simplify`. Each entry: finding + spec-section-pointer + what spec decision it drove. This is the audit trail for the plan.

**Rule:** the spec is the deliverable; the audit file is the trace. If a finding doesn't change a spec decision, log it in audit (so it's not lost) but the spec doesn't grow around it.

**Mark `UI-heavy: yes/no`** in the plan header using the same criteria as `/workflow` Phase 4a:
- Has a `docs/design/{name}.md` spec
- Creates/modifies a **page** (`src/app/**/page.tsx`) or **layout**
- Creates/modifies a **new component** in `src/presentation/components/**` or `src/components/ui/**`
- Touches **3+ UI files** (`.tsx`/`.css`) in one feature
- Changes **visual tokens** (globals.css, theme, brand colors, typography, spacing)

`/workflow` reads this flag to decide whether to run the `@design` design-compliance audit — do NOT make it re-derive from file paths.

---

## Phase 5: Owner Review (Iterative)

**STOP and present to the user.** This is mandatory. Do NOT proceed to `/workflow` without explicit approval.

### Round 1 Presentation

Present from the owner's perspective:

1. **What will change** — high-level, non-technical summary (Vietnamese if owner communicates in Vietnamese)
2. **Key decisions to confirm** — assumptions, design choices, trade-offs. Do NOT guess or fill gaps yourself.
3. **Questions for the user** — missing context, ambiguous requirements, edge cases not covered
4. **Risk highlights** — what could go wrong, what mitigations exist
5. **Scope check** — is this one session or needs splitting? If large, present split proposal.
6. **@simplify findings for approval** — list each `@simplify` finding with your long-term assessment (aligned vs short-term shortcut) and proposed plan change. Owner approves/rejects each. Do NOT incorporate any until approved.

**WAIT for user response.** Do NOT run `/workflow` until the user explicitly approves.

### Iteration Rounds (max 3)

After Round 1 feedback:
- **Update the spec file** (`{feature}.md`) with changes — refine affected sections in-place. NEVER append "### Round 2 also changed..." into spec.
- **Append Owner Review Round 1 entry to audit file** (`{feature}.review.md`): owner feedback summary + which spec sections updated + why.
- Re-dispatch affected agents if feedback changes requirements/scope (e.g. BLA for business rule changes, @code for new backend pieces). Their findings → audit file (`## Owner-review Round 1 follow-up — @<agent>`); spec absorbs the result via section updates.
- Present changes in Round 2 (delta only — what changed since Round 1)
- Reviewer round N reads **spec only** — never hand it the audit file.
- Repeat until: (a) user approves, OR (b) 3 rounds reached

**After 3 rounds without approval:** STOP. Present current state + remaining open questions. Let user decide next step.

---

## Scope Splitting

If the plan scope is too large for one `/workflow` session, split into multiple plan files:

**Too large signals:**
- 20+ task specs across agents
- 4+ distinct sub-features that could ship independently
- @check flags 8+ HIGH risks
- Estimated 12+ hours of implementation

**Split approach:**
1. Identify independent sub-features (can ship + test independently)
2. Create separate plan files: `docs/plans/<feature>-part1.md`, `<feature>-part2.md`
3. Order by dependency (part1 must complete before part2)
4. Present split proposal to owner — let them choose execution order
5. Each part gets its own owner review round

---

## Plan Structure (Output File)

```markdown
# Plan: [Feature Name]
> Date: [today] | Status: Draft -> Approved | Rounds: [N]
> UI-heavy: [yes/no]   # yes -> /workflow runs @design design-compliance audit
> Ready for: `/workflow @docs/plans/<name>.md`
> Audit: `docs/plans/<name>.review.md` (review rounds + resolutions — append-only)

## Summary
[1-2 sentences]

## Business Context (from @BLA)
- Rules impacted
- Gaps identified
- Acceptance criteria

## Scope Assessment
- Single session OR split (list parts if split)
- Estimated effort: [hours]

## Files to Create/Modify
| File | Action | Agent |
|------|--------|-------|
| path/to/file.ts | create/modify | @code |

## Task Specs

### [@test] [Title] (if business logic)
**AC:**
- [ ] [criteria]
**Test File:** 	ests/unit/...
**Code Context:** [snippets]

### [@code] [Title]
**AC:**
- [ ] [criteria]
**Files:** path/to/file.ts
**Code Context:** [snippets]
**Depends on:** @test (RED output)

### [@server-actions] [Title] (if needed)
**AC:**
- [ ] [criteria]
**Files:** src/app/_actions/...
**Depends on:** @code

### [@design] [Title] (if frontend)
**AC:**
- [ ] [criteria]
- [ ] Design compliance: CSS vars (no hardcoded hex), dark mode, typography/spacing/radius/shadows, icons, responsive — per docs/DESIGN-STANDARDS.md + docs/BRAND-GUIDELINE.md
**Files:** src/presentation/...
**Data Shape:** [from @code output]
**Design Spec:** docs/design/{name}.md (if exists)

### [@ui-test] [Title] (if frontend with logic)
**AC:**
- [ ] [criteria]
**Test File:** 	ests/unit/components/...
**Depends on:** @design

## Dependency Order
1. @test -> @code -> @server-actions
2. @design -> @ui-test
3. @docs (after all)

## Risks & Mitigations (from @check)
| Risk | Severity | Mitigation |
|------|----------|------------|

## Standards Notes (from @check/@simplify)
- [notes]

## API Contract (if backend <-> frontend)
- Endpoints, data shapes, request/response schemas

## Testing Strategy
- Unit: [what @test/@ui-test covers]
- Integration: [if needed]
- Manual: [what to verify by hand]

## Owner Review Log

> Audit trail lives in `docs/plans/<name>.review.md` (append-only). This section = final approval state only.

- Final: APPROVED on [date]
- Rounds: [N]
```

---

## Report

After final plan (approved by owner):
- Plan spec location: `docs/plans/<name>.md`
- Audit log location: `docs/plans/<name>.review.md` (Phase 1-3 findings + owner review rounds + resolutions)
- Audit summary: top 3 findings that drove spec changes
- Key risks: top 3 (from @check)
- Estimated effort
- Scope: single session or split (list parts)
- Next: `/workflow @docs/plans/<name>.md` (workflow reads spec only — not audit)

If plan was split, list all parts in execution order.
