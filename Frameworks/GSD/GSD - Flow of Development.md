---
date: 2026-05-07
tags: [framework, gsd, agent-harness, flow]
source: ""
status: developing
---

# GSD — Flow of Development

GSD's pipeline: every feature moves through typed artifacts, each consumed by the next stage. Companion to [[GSD - Summary]].

---

## The Core Idea

Every feature moves through five stages, each producing a typed artifact that the next stage consumes. No stage invents context — it reads what the previous stage produced.

```
Requirement
    ↓
CONTEXT.md      ← what the user wants (discuss-phase)
    ↓
RESEARCH.md     ← what we need to know (researcher)
    ↓
PATTERNS.md     ← what the codebase already does (pattern-mapper)
    ↓
PLAN.md         ← how to build it (planner + plan-checker)
    ↓
SUMMARY.md      ← what was built (executor)
    ↓
VERIFICATION.md ← did it actually work (verifier)
```

The artifact IS the contract between stages. If a stage produces a weak artifact, every stage after it inherits that weakness.

---

## Stage 1: Discuss — Capture Intent

**Command:** `/gsd-discuss-phase {N}` **Produces:** `CONTEXT.md` **Gate type:** Pre-flight (phase must exist in ROADMAP.md)

### What happens

You are the visionary. Claude is the builder. The discuss phase captures what you want implemented, not how to implement it. Claude identifies gray areas — implementation decisions that could go multiple ways and would change the result — and asks you to choose.

### What CONTEXT.md must contain

- **Locked Decisions (D-01, D-02, ...)** — non-negotiable, every downstream agent must honor exactly
- **Claude's Discretion** — areas where Claude chooses the approach
- **Deferred Ideas** — explicitly out of scope for this phase
- **Canonical Refs** — full file paths to every spec, ADR, or doc downstream agents must read

### Critical rules

- Scope is fixed by ROADMAP.md. Discussion clarifies HOW, never WHETHER to add new capabilities.
- When you mention a document or spec, Claude adds it to canonical refs immediately.
- If you've done this phase before, prior decisions are carried forward — you won't be re-asked.
- If a SPEC.md exists, requirements are locked. Discussion covers implementation decisions only.

### When to skip

Use `/gsd-plan-phase {N} --prd path/to/prd.md` to bypass discuss entirely if you have a written PRD. Claude generates CONTEXT.md from it directly.

---

## Stage 2: Research — Investigate the Domain

**Command:** `/gsd-plan-phase {N}` (triggers researcher automatically) **Produces:** `RESEARCH.md` **Gate type:** Pre-flight (CONTEXT.md should exist; researcher reads locked decisions before researching)

### What the researcher does

Answers: _"What do I need to know to plan this phase well?"_

- Verifies library versions against npm/registry (training data is 6-18 months stale)
- Maps each capability to its architectural tier (API, frontend, database, CDN)
- Audits the environment — are required tools actually installed?
- For rename/refactor phases: inventories runtime state that grep can't find (database keys, OS registrations, live service config)

### What RESEARCH.md must contain

Every claim is tagged with provenance:

- `[VERIFIED: npm registry]` — confirmed this session
- `[CITED: docs.example.com]` — from official docs
- `[ASSUMED]` — training knowledge, not verified

`[ASSUMED]` claims need your confirmation before becoming locked decisions. The Assumptions Log section lists all of them.

### The Architectural Responsibility Map

The most important section. Maps capabilities to tier owners:

|Capability|Primary Tier|Rationale|
|---|---|---|
|Auth validation|API / Backend|Security-sensitive, must not run in browser|
|Spatial query|Database|Owns geometry operations|
|Tile rendering|Tile server|Owns MVT generation|

The plan-checker enforces this map. Tasks that put work in the wrong tier fail.

### When to skip research

Add `--skip-research` if the task is a bug fix, simple refactor, or well-understood change. Research adds cost — only spend it when the domain is genuinely uncertain.

---

## Stage 3: Pattern Mapping — Find Codebase Analogs

**Automatic during:** `/gsd-plan-phase {N}` **Produces:** `PATTERNS.md` **Gate type:** None (non-blocking, non-producing agents continue without it)

### What the pattern mapper does

For each file the phase will create or modify:

1. Classifies it by role (controller, service, component, middleware, utility) and data flow (CRUD, streaming, event-driven, request-response)
2. Finds the closest existing file in the codebase with the same role + data flow
3. Extracts concrete code excerpts: imports, auth pattern, core pattern, error handling

### What PATTERNS.md must contain

Per-file pattern assignments with actual code excerpts and line numbers. "Copy auth pattern from `src/controllers/users.ts` lines 12-25" — not "follow the auth pattern."

Shared patterns listed separately: auth middleware, error handling wrappers, logging — applied once, referenced by all relevant plans.

### Why this matters

Without PATTERNS.md, the planner invents patterns. With it, the executor replicates what already exists. Consistency is not a style preference — it's what makes code maintainable.

---

## Stage 4: Plan — Decompose Into Executable Tasks

**Command:** `/gsd-plan-phase {N}` **Produces:** `PLAN.md` files **Gate type:** Revision (plan-checker reviews, max 3 iterations)

### The core rule

**Plans are prompts, not documents.** A PLAN.md is directly consumed as the executor's input. Whatever the plan says is exactly what the executor receives. Vague plans produce shallow execution.

### Task anatomy — every task needs all four

```xml
<task type="auto">
  <name>Create auth endpoint</name>
  <files>src/app/api/auth/login/route.ts, src/types/auth.ts</files>
  <action>
    Create POST endpoint accepting {email, password}.
    Validate using bcrypt against User table.
    Return JWT in httpOnly cookie with 15-min expiry.
    Use jose library (not jsonwebtoken — CommonJS issues with Edge runtime).
    Reference auth pattern from src/controllers/users.ts lines 12-25.
  </action>
  <verify>
    grep -n "httpOnly" src/app/api/auth/login/route.ts
    grep -n "jose" src/app/api/auth/login/route.ts
  </verify>
  <done>
    POST /api/auth/login returns 200 with httpOnly cookie on valid credentials.
    Returns 401 on invalid credentials.
  </done>
</task>
```

**`<files>`** — exact paths, no vague descriptions **`<action>`** — concrete values: library names, function signatures, config keys, exact strings. Never "align X with Y" without specifying the target state. **`<verify>`** — grep-checkable commands, no subjective language ("looks correct" fails) **`<done>`** — measurable acceptance criteria, not task description

### Wave structure

Plans in the same wave execute in parallel. Plans in wave N+1 depend on wave N completing. The planner builds this dependency graph from what each plan creates and what each plan needs.

```
Wave 1: database schema + API routes (independent)
Wave 2: frontend components (needs API routes from Wave 1)
Wave 3: integration + wiring (needs both)
```

### Scope rule

2-3 tasks per plan, targeting ~50% of executor context. Quality degrades past that point — not as a guideline, as a measured fact. More plans, smaller scope, consistent quality.

### Forbidden language in task actions

These phrases in a task action mean the plan is delivering less than the requirement specifies. They are always BLOCKERS, never acceptable:

`"v1"`, `"static for now"`, `"hardcoded for now"`, `"simplified version"`, `"placeholder"`, `"future enhancement"`, `"will be wired later"`, `"skip for now"`

If the full scope can't fit in the plan budget, the planner returns `## PHASE SPLIT RECOMMENDED` and you decide how to split. Silent omission is never acceptable.

### The planner's authority limits

Only three legitimate reasons to split or flag a feature:

1. Context cost — implementation would exceed 50% of one agent's context
2. Missing information — required data isn't in any source artifact
3. Dependency conflict — feature can't be built until another phase ships

"Too complex" or "too difficult" are not valid reasons. If a feature has none of the three constraints above, it gets planned.

### Plan-checker quality gate

Before execution, the plan-checker verifies plans against 12 dimensions:

|Dimension|What it checks|
|---|---|
|Requirement coverage|Every REQ-ID has at least one task|
|Task completeness|Every task has files + action + verify + done|
|Dependency correctness|No cycles, no missing references, waves consistent|
|Key links planned|Artifacts are wired, not just created|
|Scope sanity|Within context budget|
|Context compliance|Locked decisions from CONTEXT.md are implemented|
|Scope reduction|No forbidden placeholder language|
|Architectural tier|Tasks assign work to the correct tier|
|Cross-plan data contracts|Shared data pipelines have compatible transforms|
|Research resolution|No open questions remain in RESEARCH.md|
|Pattern compliance|Plans reference analogs from PATTERNS.md|
|CLAUDE.md compliance|Plans follow project-specific conventions|

If issues are found, the planner revises. Max 3 iterations. If still failing after 3, escalates to you.

---

## Stage 5: Execute — Build the Code

**Command:** `/gsd-execute-phase {N}` **Produces:** `SUMMARY.md` per plan + commits **Gate type:** Revision (verifier reviews completeness after all waves)

### How execution works

The orchestrator stays lean — it coordinates, never executes. It:

1. Groups plans into waves by dependency
2. Spawns executor agents (one per plan, parallel within a wave)
3. Merges completed worktrees
4. Runs post-merge tests (independent from executor self-tests)
5. Spawns the verifier

### What the executor does per task

1. Reads the plan, reads `read_first` files
2. Executes the action
3. Verifies against `<verify>` criteria
4. Commits immediately (per task, not per plan)
5. Continues to next task

### Deviation rules — self-correction during execution

The executor handles deviations automatically without asking:

|Rule|Trigger|Action|
|---|---|---|
|Rule 1|Code doesn't work as intended|Auto-fix bugs|
|Rule 2|Missing critical functionality (security, correctness)|Auto-add|
|Rule 3|Something blocks task completion|Auto-fix blocker|
|Rule 4|Fix requires architectural change|STOP, return checkpoint|

Rules 1-3 need no permission. Rule 4 always escalates to you.

**Fix attempt limit:** 3 auto-fix attempts per task. After 3, document as deferred issue and move to next task. This is the circuit breaker.

**Scope boundary:** Only fix issues directly caused by the current task's changes. Pre-existing issues go to `deferred-items.md`, never touched.

**Analysis paralysis guard:** If the executor makes 5+ consecutive reads without writing anything, it must stop and either write code or declare what's blocking it.

### What SUMMARY.md must contain

- Per-task commits with hashes
- All deviations documented (or "None — executed exactly as written")
- Known stubs declared (hardcoded values, placeholder data, unwired components)
- Self-check: PASSED or FAILED (executor verifies its own claims before writing summary)

### Post-merge gate — the critical independent test

After all worktrees in a wave are merged back, the orchestrator runs the full test suite independently. This catches what individual executors can't see: type conflicts from parallel plans modifying the same module, broken imports, API contract mismatches. Individual self-checks pass in isolation; the post-merge gate catches what merging breaks.

---

## Stage 6: Verify — Confirm Goal Achievement

**Automatic after execution** **Produces:** `VERIFICATION.md` **Gate type:** Escalation (surfaces gaps to you for decision)

### The adversarial stance

The verifier starts with this hypothesis: _"Tasks completed. Goal not achieved."_ Its job is to falsify the SUMMARY.md, not confirm it. SUMMARY.md documents what the executor claimed. VERIFICATION.md documents what actually exists in the code.

### Four-level artifact verification

|Level|Check|What it catches|
|---|---|---|
|1|Exists|Missing files|
|2|Substantive|Stub files (placeholder content, < minimum lines)|
|3|Wired|Orphaned files (exist but nothing imports or uses them)|
|4|Data flows|Hollow wiring (imported and used but data source returns empty)|

Level 4 is the one most systems miss. A component wired to an API route that returns `[]` passes levels 1-3. Level 4 traces the data source and verifies it produces real data.

### Verification outcomes

**`passed`** — All must-haves verified. Ready for next phase.

**`human_needed`** — Automated checks passed. Some behaviors need you to verify (visual appearance, user flows, real-time behavior).

**`gaps_found`** — One or more must-haves failed. Gaps are structured in YAML frontmatter for automatic gap closure planning.

### Gap closure

When `gaps_found`:

```
/gsd-plan-phase {N} --gaps
```

This reads VERIFICATION.md, creates gap-closure plans targeting exactly the failed items, and re-executes. One automatic retry. If gaps persist after closure, escalates to you.

---

## Stage 7: Debug — When Things Go Wrong

**Command:** `/gsd-debug` **Produces:** `.planning/debug/{slug}.md` (persistent session state) **Gate type:** Escalation (when investigation can't continue without human input)

### The debug file is the investigation brain

The file survives context resets. Rules:

- **Current Focus** — overwritten before every action (what's happening NOW)
- **Symptoms** — immutable after gathering (prevents drift)
- **Eliminated** — append only (prevents re-investigating dead ends)
- **Evidence** — append only (builds the case)
- **Update before acting**, not after — if context resets mid-action, file shows what was in progress

### Investigation discipline

One hypothesis at a time. Each tested with a falsifiability requirement: _"If this hypothesis is true, I will observe X."_ If you can't design an experiment to disprove it, the hypothesis isn't useful yet.

Before any fix is applied, the structured reasoning checkpoint is mandatory:

```yaml
reasoning_checkpoint:
  hypothesis: "X causes Y because Z"
  confirming_evidence: [specific observations]
  falsification_test: "what would prove this wrong"
  fix_rationale: "why fix addresses root cause, not symptom"
  blind_spots: "what hasn't been tested"
```

If any field is vague, root cause is not confirmed. Keep investigating.

### Knowledge base — cross-session learning

Every resolved debug session appends to `.planning/debug/knowledge-base.md`. At the start of every new investigation, the debugger checks for keyword overlap with past entries. Past root causes become first hypotheses to test. A match is a candidate, not a diagnosis — it gets tested like any other hypothesis.

---

## The Gate System — How Quality Is Enforced

Every validation point in GSD maps to one of four gate types. Knowing which gate applies tells you exactly what will happen when something fails.

|Gate|When used|On failure|
|---|---|---|
|**Pre-flight**|Entry points (does PLAN.md exist? is phase in ROADMAP?)|Blocks before any work. Fix precondition, retry.|
|**Revision**|After producer output (plan quality, summary completeness)|Loops back to producer with specific feedback. Max 3 iterations.|
|**Escalation**|When automation can't resolve (revision loop exhausted, ambiguous requirement)|Pauses. Presents options. Waits for your decision.|
|**Abort**|When continuing causes damage (context critically low, state error)|Stops immediately. Preserves state. Reports reason.|

Selection rule: Start with pre-flight. If the check happens after work is produced → revision. If revision can't resolve → escalation. If continuing is dangerous → abort.

---

## Artifact Chain — What Each Stage Reads and Produces

|Stage|Reads|Produces|Consumed by|
|---|---|---|---|
|Discuss|ROADMAP.md, prior CONTEXT.md files|CONTEXT.md|Researcher, planner, plan-checker, executor|
|Research|CONTEXT.md, codebase|RESEARCH.md|Pattern mapper, planner, plan-checker|
|Pattern map|CONTEXT.md, RESEARCH.md, codebase|PATTERNS.md|Planner, plan-checker|
|Plan|CONTEXT.md, RESEARCH.md, PATTERNS.md|PLAN.md files|Plan-checker, executor|
|Plan-check|PLAN.md, CONTEXT.md, RESEARCH.md|Issues or approval|Planner (revision) or executor|
|Execute|PLAN.md, codebase|SUMMARY.md + commits|Verifier|
|Verify|PLAN.md, SUMMARY.md, codebase|VERIFICATION.md|You, or gap-closure planner|
|Debug|Codebase + debug file|Fixed code + KB entry|Future investigations|

---

## Common Failure Modes and How GSD Handles Them

|Failure|Where it surfaces|How it's handled|
|---|---|---|
|Requirement not planned|Plan-checker Dimension 1|BLOCKER — revision required|
|Task action too vague|Plan-checker Dimension 2|BLOCKER — must specify concrete values|
|User decision ignored|Plan-checker Dimension 7|BLOCKER — revision required|
|Scope reduction in task|Plan-checker Dimension 7b|Always BLOCKER, never warning|
|Wrong architectural tier|Plan-checker Dimension 7c|BLOCKER if security-sensitive|
|File exists but is a stub|Verifier Level 2|FAILED — gap closure triggered|
|File exists, not wired|Verifier Level 3|FAILED — gap closure triggered|
|Wired but data empty|Verifier Level 4|FAILED — gap closure triggered|
|Executor infinite retry|Fix attempt limit (3)|Stops, documents, moves on|
|Agent stuck reading|Analysis paralysis guard (5 reads)|Forces action or blocked declaration|
|Architectural change needed|Deviation Rule 4|STOPS, returns checkpoint to you|
|Post-merge test failure|Post-merge gate|Wave halts, you decide to fix or continue|
|Research open questions|Plan-checker Dimension 11|BLOCKER — resolve before planning|

---

## Quick Reference — Key Rules

**Planning**

- 2-3 tasks per plan, ~50% context target
- Every task: files + action + verify + done (all four required)
- Action must contain concrete values, not references to find elsewhere
- Forbidden in actions: "static for now", "v1", "placeholder", "will wire later"
- If scope can't fit: PHASE SPLIT, not silent omission

**Execution**

- Per-task commits, never bulk staging (`git add .` is prohibited)
- Fix attempts capped at 3 per task
- Scope boundary: only fix what the current task changed
- Self-check before SUMMARY is written
- Stubs must be declared, never hidden

**Verification**

- Verifier starts adversarial: assumes goal not achieved until proven
- Task completion ≠ goal achievement
- SUMMARY.md claims are not evidence — only codebase state is
- Level 4 (data flows) is required for any artifact that renders dynamic data

**Context management**

- Orchestrator reads frontmatter only (not full bodies) when context < 500k
- MCP schemas inject per-turn regardless of use — disable unused servers before long phases
- Warning signs of context pressure: increasing vagueness, skipped steps, partial completion

---

## The Autonomous Path

For a full feature without manual checkpoints:

```
/gsd-discuss-phase {N}      ← answer the gray area questions
/gsd-plan-phase {N} --auto  ← research + plan + execute + verify in sequence
```

The `--auto` flag chains all stages. It pauses only at explicit decision points: architectural escalations, gap-closure decisions, human verification items.

For a full milestone without manual transitions between phases:

```
/gsd-autonomous
```

Runs every incomplete phase in order: discuss → plan → execute → verify, with lifecycle (audit → complete → cleanup) at the end. Pauses only at blockers, gaps requiring your decision, and human verification items.