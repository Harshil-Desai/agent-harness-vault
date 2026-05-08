---
date: 2026-05-07
tags: [framework, bmad, agent-harness, research]
source: ""
status: developing
---

# BMAD — Summary

Notes on BMAD's architecture and what to extract for my own harness. Companion to [[BMAD - Flow of Development]].

---

Let me map what we're actually looking at — because the folder structure reveals BMAD's architecture clearly.

**BMAD is essentially a 4-phase pipeline of skills:**

```
1-analysis        → understand the problem
2-plan-workflows  → define what to build (PRD, UX)
3-solutioning     → design how to build it (arch, epics, stories)
4-implementation  → build, test, correct, ship
```

Plus `core-skills` that cut across all phases (elicitation, brainstorming, review, distillation).
**The files that actually matter for your harness** (ignoring docs, website, installer, tooling):

```
src/bmm-skills/
  1-analysis/
    bmad-document-project/     ← codebase scanning + deep dives
    bmad-agent-analyst/        ← analysis agent persona
    research/                  ← domain/market/technical research steps

  2-plan-workflows/
    bmad-create-prd/           ← 12-step PRD generation
    bmad-validate-prd/         ← 13-step PRD validation
    bmad-edit-prd/

  3-solutioning/
    bmad-create-architecture/  ← 8-step arch generation
    bmad-create-epics-and-stories/
    bmad-check-implementation-readiness/
    bmad-generate-project-context/

  4-implementation/
    bmad-dev-story/            ← story execution
    bmad-quick-dev/            ← lightweight path (5 steps)
    bmad-correct-course/       ← self-correction
    bmad-checkpoint-preview/   ← pre-commit validation
    bmad-code-review/
    bmad-qa-generate-e2e-tests/
    bmad-sprint-planning/
    bmad-retrospective/

src/core-skills/
    bmad-advanced-elicitation/
    bmad-distillator/          ← context compression
    bmad-review-adversarial-general/
    bmad-shard-doc/            ← large doc splitting
```

---
**My read of what's genuinely interesting for your harness vs what's noise:**

**High value to extract:**
- `bmad-validate-prd` — 13 validation steps is a serious spec compliance mechanism
- `bmad-correct-course` — self-correction with a checklist
- `bmad-checkpoint-preview` — 5-step pre-commit validation (this is the testing gate)
- `bmad-quick-dev` — the lightweight path reveals BMAD's "happy path" assumptions
- `bmad-distillator` — context compression for large documents
- `bmad-generate-project-context` — how they encode codebase knowledge

**Medium value:**
- `bmad-create-architecture` — useful structure but you have GIS360 already
- `bmad-check-implementation-readiness` — gate before coding starts
- `bmad-advanced-elicitation` — useful for requirement clarification

**Low value for your harness:**
- Everything in `2-plan-workflows` (PRD creation UX design) — you're not building new products, you're working on an existing codebase
- `bmad-sprint-planning`, `bmad-sprint-status` — project management ceremony
- `bmad-retrospective` — human-facing, not automatable
- All the research skills — domain/market research not relevant to GIS360

---
## What bmad-quick-dev actually is

It's a **5-step state machine** for turning a requirement into reviewed, committed code. Every step is a separate file loaded just-in-time (deliberately — to avoid context bloat). Two utility sub-steps (`compile-epic-context`, `sync-sprint-status`) handle cross-cutting concerns.

The execution flow:

```
activate → load config → greet
           ↓
     step-01: clarify & route
           ↓                    ↘
     step-02: plan          step-oneshot (zero blast radius)
     (spec → approval)           ↓
           ↓              implement → adversarial review → present
     step-03: implement
           ↓
     step-04: review (adversarial, 3 subagents)
           ↓ (loop back if bad_spec / intent_gap)
     step-05: present (commit + review order)
```

---
## File-by-file breakdown

### SKILL.md — The orchestrator

**What it does:** Defines activation sequence, the "Ready for Development" standard, the scope standard (900–1600 tokens), and enforces just-in-time step loading.

**Key mechanisms:**

- Just-in-time loading: `NEVER load multiple step files simultaneously` — each step file is loaded only when reached. This is deliberate context management.
- Scope standard: single user-facing goal, 900–1600 tokens. Below 900 = ambiguous. Above 1600 = context rot in implementation agents.
- Two paths: `plan-code-review` (default) vs `one-shot` (zero blast radius only).

**For your harness:** The scope standard is directly extractable. The 900–1600 token target for specs is a real constraint you should encode. The just-in-time step loading pattern is what you want for your harness's orchestration — don't front-load context.

---
### customize.toml — Configuration layer

**What it does:** Three-level override system (defaults → team → personal). Defines `persistent_facts` (static context loaded on activation), `activation_steps_prepend/append` (pre/post hooks), and `on_complete` hook.

**Key insight:** `persistent_facts` loads `project-context.md` as foundational context for every run. This is their equivalent of your product knowledge layer — but it's flat file injection, not retrieval.

**For your harness:** The three-level config merge pattern (base → team → user) is worth stealing for harness configuration. The `persistent_facts` concept maps directly to your Product Constitution (Milestone 3, Layer 1). The `on_complete` hook is where you'd wire spec-compliance verification.

---
### spec-template.md — The spec format

**What it does:** Defines the artifact that drives the entire workflow. Key sections:

```
<frozen-after-approval>   ← human-owned, immutable after approval
  ## Intent               ← Problem + Approach (2 sentences each)
  ## Boundaries           ← Always / Ask First / Never
  ## I/O Matrix           ← edge cases as a table
</frozen-after-approval>

## Code Map               ← agent-populated: file paths + roles
## Tasks & Acceptance     ← checkboxes + Given/When/Then ACs
## Spec Change Log        ← append-only audit trail of review loops
## Design Notes           ← non-obvious rationale only
## Verification           ← CLI commands to confirm correctness
```

**The `<frozen-after-approval>` mechanism is the most important design decision in the entire framework.** It cleanly separates human intent (immutable) from agent-derived implementation plan (mutable). When a review loop finds a `bad_spec`, only the non-frozen sections are rewritten. The frozen intent survives.

**For your harness:** This is your core spec artifact. The frozen/mutable split directly solves the spec drift problem from your risk register. The "Ask First" boundary tier is a human-gating mechanism — equivalent to your human-in-the-loop checkpoints. The Spec Change Log is an audit trail that prevents the same bad derivation recurring across loops.

The token target (900–1300 for body, 1600 hard cap) is empirically derived and worth keeping.

---
### step-01-clarify-and-route.md — Routing logic

**What it does:** Determines where to start based on what already exists. Smart routing:

1. Checks for explicit argument (file path / spec name)
2. Checks recent conversation for clear intent
3. Falls back to scanning artifacts and asking

Then does:

- Multi-goal check (split or keep)
- Version control sanity check (clean tree? right branch?)
- Epic story detection → compiles/loads epic context
- Previous story continuity (loads last `done` spec for same epic)
- Routes to `one-shot` or `plan-code-review`

**The epic context caching** is clever: it checks if `epic-N-context.md` is newer than all planning artifacts. If valid cache exists, skip recompilation. If stale, recompile via subagent.

**For your harness:**

- The routing logic (check explicit → check conversation → scan artifacts → ask) is a good pattern for requirement intake.
- Multi-goal detection with explicit split/keep decision is directly applicable.
- Previous story continuity (load Code Map + Design Notes + Change Log from last done story) is how they maintain cross-task context — relevant to your Dynamic Memory (Milestone 5).
- The epic context caching pattern is your code memory retrieval strategy in miniature.

---
### step-02-plan.md — Spec generation

**What it does:**

1. Loads any existing draft (preserves frozen block)
2. Investigates codebase via subagents (distilled summaries only — explicitly prevents context snowballing)
3. Fills spec template
4. Self-reviews against Ready for Development standard
5. Token count check (>1600 → split or accept risk)
6. **CHECKPOINT 1**: human approves or edits
7. On approval: marks `ready-for-dev`, freezes intent block

**The subagent isolation for codebase investigation** is key: `instruct subagents to give you distilled summaries only`. This is their solution to the context window problem — exploration happens in isolation, only conclusions come back.

**For your harness:** The checkpoint pattern (present → halt → approve/edit loop) is your human-in-the-loop gate. The draft resume (preserve frozen block, regenerate rest) is how you handle spec iteration without losing human intent. The self-review against explicit standard before showing human is worth replicating.

---
### step-03-implement.md — Implementation

**What it does:**

1. Captures `baseline_commit` into spec frontmatter (critical for diff-based review)
2. Sets status `in-progress`
3. Loads `context:` files from spec frontmatter
4. Hands spec to subagent for implementation
5. Self-checks: every task `[x]` before proceeding

**Short file. Deliberately minimal.** The spec does all the work — implementation is just "hand this to an agent and verify the checklist."

**For your harness:** `baseline_commit` capture is essential — it's what makes the adversarial review in step-04 work. The context loading from spec frontmatter is how they do selective knowledge injection — only what this specific task needs.

---
### step-04-review.md — The adversarial review

**This is the most sophisticated component in the entire framework.**

**What it does:** Three parallel subagents, each with deliberately different context:

|Reviewer|Gets|Does|
|---|---|---|
|Blind hunter|Diff only|Finds issues with no spec anchoring — catches hallucinations|
|Edge case hunter|Diff + codebase access|Finds untested scenarios|
|Acceptance auditor|Diff + spec + context docs|Checks spec compliance|

Then classifies findings into:

- `intent_gap` — spec can't resolve it, need human (loop back to step-02)
- `bad_spec` — spec should have been clearer (rewrite non-frozen sections, loop back to step-03)
- `patch` — trivial fix, auto-apply
- `defer` — pre-existing issue, log to deferred-work.md
- `reject` — noise, drop

**The cascade rule:** if `intent_gap` or `bad_spec` exist, `patch` and `defer` are moot — code is being re-derived anyway. Loop counter caps at 5 iterations (circuit breaker).

**`bad_spec` handling with KEEP instructions** is particularly good: before reverting, extract what worked. The Spec Change Log records the bad state so it's never re-derived. This is how they avoid the infinite loop problem where the agent keeps making the same mistake.

**For your harness:** This is the self-correction mechanism from Milestone 5. The three-reviewer pattern with different context isolation is directly applicable. The classification taxonomy (`intent_gap` / `bad_spec` / `patch` / `defer` / `reject`) is clean and worth adopting. The loop counter as circuit breaker maps directly to your "error budget" risk mitigation.

---
### step-05-present.md — Presentation

**What it does:**

1. Generates Suggested Review Order — ordered by conceptual concern, not file. Entry point first, peripherals last.
2. Marks spec `done`
3. Commits with conventional message
4. Opens spec in editor
5. Runs `on_complete` hook

**The Suggested Review Order** is an underrated feature — it's a human-readable trail through the diff organized by what the reviewer needs to understand, not by what changed. Built into the spec file itself so it survives beyond the session.

**For your harness:** The `on_complete` hook is where spec-compliance verification fires. The review order generation is useful for the human-in-the-loop handoff. The status progression (`draft → ready-for-dev → in-progress → in-review → done`) is a clean state machine you should replicate.

---
### step-oneshot.md — Fast path

**What it does:** For zero-blast-radius changes, skips spec approval and planning. Implements directly, runs one adversarial review (blind hunter only, simplified classification), generates a minimal spec trace after the fact, commits.

**The key difference from full path:** Three categories only (patch / defer / reject) — no `intent_gap` or `bad_spec` because there's no spec to be wrong. If a finding is significant, halt and ask human.

**For your harness:** One-shot is your fast path for bug fixes and small changes. The distinction (zero blast radius = no spec approval needed) is a routing decision your harness should encode.

---
### compile-epic-context.md — Context compiler

**What it does:** Given an epic number, pulls only the relevant requirements, technical decisions, UX patterns, and cross-story dependencies from all planning artifacts. Hard rules:

- Scope aggressively — when in doubt, leave it out
- Describe by purpose, not by source (no "per PRD section 3.2")
- Target 800–1500 tokens
- Never quote source documents verbatim

**For your harness:** This is your product knowledge retrieval in action. The output format (Goal, Stories, Requirements, Technical Decisions, UX Patterns, Cross-Story Dependencies) is a clean context document structure. The "describe by purpose, not by source" rule is important — it makes the context stable even when the underlying docs change.

---
### sync-sprint-status.md — State tracker

**What it does:** Updates `sprint-status.yaml` when story status changes. Has idempotency check (never regress status), epic lift (when story goes `in-progress`, parent epic also goes `in-progress`). Skips silently if no sprint status file exists.

**For your harness:** Lightweight — this is just task state tracking. Your Dynamic Memory (Milestone 5) needs something like this for tracking execution state across sessions.

---
## Consolidated: Pros and Cons

**Genuine strengths:**

1. **The frozen/mutable spec split** — cleanest solution to spec drift I've seen. Human intent is structurally protected. Agent can only touch what it derives.
    
2. **The review taxonomy** — `intent_gap / bad_spec / patch / defer / reject` is precise. The cascade rule (intent_gap/bad_spec make patch/defer moot) prevents wasted work. The KEEP instruction mechanism prevents regression.
    
3. **Three-reviewer isolation pattern** — different context per reviewer prevents anchoring bias. Blind hunter (diff only) is the most important one — it finds what the spec-aware reviewers rationalize away.
    
4. **Just-in-time step loading** — context window discipline baked into the architecture. Only load what you need for the current step.
    
5. **Scope standard** (900–1600 tokens per spec) — empirically grounded. Prevents both under-specification and context rot.
    
6. **Epic context caching with staleness detection** — pragmatic solution to the "recompute vs reuse" problem for planning context.
    
7. **Spec Change Log** — append-only audit trail that prevents the agent from re-deriving the same bad state across review loops.
    

**Real weaknesses:**

1. **Heavy human-in-the-loop** — CHECKPOINT 1 in step-02 always halts for human approval. This is fine for interactive use, inappropriate for autonomous operation. Your harness needs to decide: automate approval with confidence scoring, or keep the gate but make it async.
    
2. **Sprint-status coupling** — `sync-sprint-status.md` assumes a BMAD project structure (`_bmad/`, `sprint-status.yaml`). This is framework lock-in, not a transferable pattern.
    
3. **No product knowledge retrieval** — `persistent_facts` just loads `project-context.md` flat. There's no retrieval, no ranking, no relevance filtering. For GIS360's scale this will flood the context window.
    
4. **No code memory** — the agent discovers codebase structure fresh every time via subagent exploration. The epic context cache helps for epic stories, but freeform tasks start blind.
    
5. **Self-correction is spec-level, not pattern-level** — the Spec Change Log prevents the same bad derivation recurring within a task, but it doesn't persist across tasks. The agent can make the same class of mistake on task 2 that it made on task 1.
    
6. **One-shot routing is manual** — blast radius assessment is left to the agent's judgment with no supporting data. On GIS360 this will be unreliable without codebase dependency information.
    
7. **The three-reviewer pattern requires subagents** — degrades to a human-paste workflow when subagents aren't available.
    

---
## What to extract for your harness

|BMAD mechanism|What to take|What to change|
|---|---|---|
|Frozen/mutable spec split|Take it wholesale|None|
|Review taxonomy|Take it wholesale|None|
|Spec Change Log|Take it wholesale|None|
|Three-reviewer pattern|Take the structure|Wire to automated test results, not just LLM reviewers|
|Scope standard (900–1600 tokens)|Take it|None|
|Status state machine|Take it|Add `validated` state after `in-review`|
|Epic context compiler|Take the output format|Replace subagent compilation with your retrieval system (Milestone 3/4)|
|`persistent_facts` / `project-context.md`|Take the concept|Replace flat file with your product knowledge retrieval|
|`baseline_commit` capture|Take it|None — essential for diff-based review|
|Just-in-time step loading|Take the principle|Implement as phase gating in your harness|
|Circuit breaker (loop counter ≤ 5)|Take it|Tune threshold per task type|
|One-shot / full-path routing|Take it|Replace blast radius judgment with dependency graph data|

---
## What bmad-checkpoint-preview actually is

It's a **human-in-the-loop review guide** — a structured walkthrough that helps a human reviewer understand a change before deciding to ship or rework it. It's designed to be triggered after quick-dev completes (or when reviewing someone else's PR).

The flow:

```
activate
    ↓
step-01: orientation    — find the change, compute stats, generate trail if needed
    ↓
step-02: walkthrough    — explain the change by concern, not by file
    ↓
step-03: detail pass    — surface risk spots and machine hardening findings
    ↓
step-04: testing        — suggest manual observations to build confidence
    ↓
step-05: wrapup         — Approve / Rework / Discuss → act on decision
```

Any step from 02 onward can early-exit to step-05 if the human signals a decision.

---
## File-by-file breakdown

### SKILL.md — Orchestrator

**What it does:** Activation (same pattern as quick-dev — config load, persistent facts, greet), then launches step-01. Defines two global rules that apply to every step:

- **Path:line format** — every code reference is `path:line`, CWD-relative, no leading `/` — for IDE terminal clickability
- **Front-load then shut up** — entire step output in one message. No drip-feeding, no mid-step questions.

**The "front-load then shut up" rule is significant.** It's a deliberate UX decision: the agent presents everything it knows, then waits. This prevents the back-and-forth pattern where the agent dribbles out observations one at a time. It also makes the output more reviewable — the human gets the full picture before responding.

**For your harness:** This is a validation/handoff pattern, not a build pattern. The two global rules (path:line format, front-load) are worth adopting for any step that presents output to humans. The `on_complete` hook is again where you'd wire post-review actions.

---
### step-01-orientation.md — Change discovery + stats

**What it does:** Finds the change being reviewed through a 5-level cascade:

1. Explicit argument (PR, commit SHA, branch, spec file)
2. Recent conversation
3. Sprint status file (stories with `review` status)
4. Current git state (confirm HEAD + branch)
5. Ask (max 3 exchanges before HALT)

Then **enriches**: if you have a spec, find the commit. If you have a commit, find the spec. Tries to always have both.

Sets two key variables:

- `change_type` — PR / commit / branch / user's words / "change"
- `review_mode` — `full-trail` / `spec-only` / `bare-commit`

Produces:

- **Intent summary** — from spec verbatim, or distilled from commit message
- **Surface area stats** — files changed, modules touched, lines of logic, boundary crossings, new public interfaces

Then: if not `full-trail`, calls `generate-trail.md` to build one from the diff before proceeding.

**The surface area stats are genuinely useful for your harness.** `boundary crossings` (changes spanning more than one top-level module) is a proxy for blast radius — exactly what quick-dev's one-shot routing needs. You could compute this automatically rather than relying on agent judgment.

**For your harness:** The 5-level change discovery cascade is a good pattern for any intake step. The `review_mode` concept (full-trail / spec-only / bare-commit) degrades gracefully — it works even when the spec is missing. The surface area stats are automatable metrics that feed routing decisions.

---
### generate-trail.md — Fallback trail builder

**What it does:** When no author-produced Suggested Review Order exists, generates one from the diff. Steps:

1. Get full diff against appropriate baseline
2. Read changed files **in full** (not just diff hunks) — surrounding code reveals intent
3. If spec exists, use Intent section to anchor concern identification
4. Identify 2–5 concerns (cohesive design intents)
5. For each concern, select 1–4 `path:line` stops — entry points, decision points, boundary crossings
6. Lead with entry point, end with peripherals

The key insight: **"Read changed files in full — not just diff hunks. Surrounding code reveals intent that hunks alone miss."** This is an explicit counter to the naive approach of just reading the diff.

**For your harness:** The fallback trail generation is your autonomous review order when the implementation agent didn't produce a Suggested Review Order. The "read full files, not just hunks" rule is important for your adversarial review subagents in quick-dev's step-04.

---
### step-02-walkthrough.md — Concern-based explanation

**What it does:** Takes the Suggested Review Order (or generated trail) and builds a structured walkthrough. Key design decisions:

- **Organize by concern, not by file.** A concern is a cohesive design intent. One file can appear under multiple concerns. One concern can span multiple files.
- **Design judgment, not correctness checking.** Frame as "here's what this change does and why" — the human evaluates fit, not correctness (that's step-03's job).
- Each concern gets: heading (design intent phrase, not file name), why (1–2 sentences on approach vs alternatives), stops (`path:line` + ≤15 word framing).
- Target 2–5 concerns. >7 concerns signals scope may be too large.

Ends with an open invitation for the human to explore before saying "next."

**Early exit to step-05** if the human signals a decision at any point.

**The "concern not file" organization is the core intellectual contribution of this entire skill.** Reviewers naturally think about changes in terms of what they're trying to accomplish, not which files were touched. Organizing the walkthrough by design intent makes the review significantly faster and catches more issues.

**For your harness:** This is your human handoff format. When your autonomous system produces code and presents it for human review, the Suggested Review Order (already generated by quick-dev's step-05) should be consumed by this structure. The concern-based organization is also useful for your spec compliance auditor — group acceptance criteria by concern, not by file.

---
### step-03-detail-pass.md — Risk surface identification

**What it does:** Identifies 2–5 high blast-radius spots in the diff using a defined taxonomy:

```
[auth]       — authentication, authorization, session, token, permission
[public API] — new/changed endpoints, exports, public methods
[schema]     — DB migrations, schema changes, data model, serialization
[billing]    — payment, pricing, subscription, metering
[infra]      — deployment, CI/CD, env vars, config, infrastructure
[security]   — input validation, sanitization, crypto, secrets, CORS
[config]     — feature flags, env-dependent behavior, defaults
[other]      — concurrency, data privacy, backwards compatibility (with descriptive tag)
```

Sequenced by **blast radius** (how much breaks if wrong), not by diff order.

Also surfaces **Spec Change Log entries** — findings from quick-dev's adversarial review loops that are instructive for the human. Not bugs that were already fixed, but decisions the review flagged that the human should know about.

Supports **targeted re-review**: "dig into [area]" triggers a correctness-focused deep dive on specific code locations — reads full file context, not just hunks, traces edge cases, boundary conditions, error handling.

**For your harness:** The risk taxonomy is directly extractable. These are the categories your automated review should flag first. The blast-radius sequencing (not diff order, not file order) is the right mental model for prioritizing review attention. The Spec Change Log surfacing is how machine hardening findings survive into the human review — this is the bridge between your autonomous review loop and the human gate.

---
### step-04-testing.md — Manual observation suggestions

**What it does:** Identifies 2–5 observable behaviors a human could verify manually. Categories:

- UI changes (screens, layouts, interactions, error states)
- CLI/terminal output (commands, flags, output format)
- API responses (endpoints, payloads, status codes)
- State changes (DB records, filesystem artifacts, config effects)
- Error paths (bad input, missing deps, edge conditions)

For each: what to do, what to expect, why bother.

**Explicitly does not duplicate CI/automated tests.** This is manual confidence-building only. If the change has no observable behavior, says so explicitly rather than padding.

**For your harness:** This is your manual test suggestion generator — the output your system produces for human QA before shipping. For GIS360 specifically, the "UI changes" category maps to Computer Use validation, and the "API responses" category maps to MCP-based testing. The step effectively categorizes what kind of testing is appropriate, which feeds your hybrid testing strategy from Milestone 1.

---
### step-05-wrapup.md — Decision handling

**What it does:** Three options:

- **Approve** — acknowledge, offer interactive patching, optionally `gh pr review --approve` (confirms with human first — it's a visible action on a shared resource)
- **Rework** — ask what went wrong (approach? spec? implementation?), help decide next steps, draft actionable feedback tied to `path:line` locations
- **Discuss** — open conversation, then return to decision prompt

The `on_complete` hook fires after decision.

**The distinction between approach/spec/implementation failure is important.** If the approach was wrong, rework means going back to planning. If the spec was wrong, rework means amending the frozen intent. If the implementation was wrong, rework means re-running step-03 of quick-dev. These are different loopback targets.

**For your harness:** This is the human decision gate. The three failure categories (approach / spec / implementation) map directly to quick-dev's review taxonomy (`intent_gap` → approach/spec, `bad_spec` → spec/implementation). Your harness needs to surface which category failed so the human knows where to re-enter the loop.

---
## Consolidated: Pros and Cons

**Genuine strengths:**

1. **Concern-based organization** — the most transferable idea in this skill. Reviewers think in design intents, not files. This belongs in your spec output format and your human handoff.
    
2. **Risk taxonomy with blast-radius sequencing** — the 8-category risk classification (`[auth]`, `[public API]`, `[schema]`, etc.) is precise and automatable. Sequencing by blast radius (not diff order) is the right prioritization.
    
3. **Graceful degradation** — `review_mode` (full-trail / spec-only / bare-commit) means the workflow functions even without a spec. The fallback trail generation ensures the concern-based walkthrough is always available.
    
4. **Surface area stats** — boundary crossings is a computable proxy for blast radius. This feeds routing decisions upstream (one-shot vs full path).
    
5. **Spec Change Log surfacing** — machine hardening findings survive into human review. This is the critical bridge between autonomous review loops and the human gate.
    
6. **Targeted re-review** — "dig into [area]" on demand, with full-file context (not just diff hunks). This is the right model for correctness checking vs the design review in the walkthrough.
    
7. **Front-load then shut up** — entire step output in one message before waiting. Eliminates drip-feeding which wastes human attention.
    

**Real weaknesses:**

1. **Entirely human-facing** — this skill produces nothing a machine can act on. Every output is for human consumption. For your autonomous harness, the value is in the patterns it uses, not the workflow itself.
    
2. **No automated test execution** — step-04 suggests manual observations but doesn't run anything. For your harness, you want automated test execution feeding this step, not just suggestions.
    
3. **Change discovery cascade assumes interactive context** — the 5-level cascade (explicit → conversation → sprint status → git state → ask) works for human-triggered reviews. In autonomous operation, you'd just pass the spec and commit directly.
    
4. **No quantitative risk scoring** — blast radius is ordered qualitatively. For your harness, you want numerical risk signals (dependency graph depth, test coverage of changed lines, number of callers of changed functions).
    
5. **Decision handling is terminal** — step-05's rework path helps the human decide next steps but doesn't re-enter the build loop automatically. In your harness, rework should trigger an automated re-entry into the appropriate quick-dev step.
    

---
## What to extract for your harness

|Checkpoint mechanism|What to take|What to change|
|---|---|---|
|Concern-based walkthrough|Take it wholesale — use for spec structure and human handoff format|None|
|Risk taxonomy (8 categories)|Take it wholesale|Add GIS360-specific categories: `[spatial]`, `[MVT]`, `[pipe-data]`|
|Blast-radius sequencing|Take the principle|Replace qualitative ordering with computed dependency depth|
|Surface area stats|Take `boundary_crossings` and `new_public_interfaces` as routing signals|Automate computation rather than LLM estimation|
|`review_mode` degradation|Take it — your harness needs graceful degradation|None|
|Spec Change Log → human review bridge|Take it wholesale|None|
|Full-file reading (not just hunks)|Take the rule — apply to your adversarial review subagents|None|
|Front-load then shut up|Take as a rule for all human-facing output steps|None|
|Approach/spec/implementation failure categories|Take it — maps to your intent_gap/bad_spec taxonomy|None|
|Targeted re-review on demand|Take the concept — "dig into [area]" maps to your spec compliance auditor|Wire to automated test results, not just LLM analysis|

---
## How checkpoint-preview connects to quick-dev

They're designed as a **pair**:

```
quick-dev produces:          checkpoint-preview consumes:
─────────────────────────    ──────────────────────────────
baseline_commit          →   diff baseline for all stats
Suggested Review Order   →   full-trail mode (primary path)
Spec Change Log          →   machine hardening section
spec status: done        →   sprint tracking (review status)
```

For your harness: quick-dev is the build loop, checkpoint-preview is the validation gate. The Suggested Review Order generated in quick-dev's step-05 is the primary input to checkpoint-preview's step-02. This handoff is clean and worth preserving.

---
## What bmad-correct-course actually is

It's a **sprint change management workflow** — a structured process for when something significant goes wrong mid-sprint and the plan needs to change. It's about updating PRDs, epics, architecture docs, and routing the change to the right person.

It is **not** an agent self-correction loop. It's a human-collaborative replanning tool.

```
activate
    ↓
step-1: understand the trigger (what went wrong, what type of problem)
    ↓
step-2: run the checklist (6 sections, systematic impact analysis)
    ↓
step-3: draft specific change proposals (old → new format, per artifact)
    ↓
step-4: generate Sprint Change Proposal document
    ↓
step-5: get approval, classify scope (minor/moderate/major), route to right agent/human
    ↓
step-6: confirm handoff, complete
```

---
## File-by-file breakdown

### SKILL.md — The workflow

**What it does:** 6-step change management process. The key structural decision is loading strategy — it FULL_LOADs PRD, epics, architecture, UX, and specs because course correction needs broad project context to assess change impact accurately. This is deliberately the opposite of quick-dev's selective loading.

**The scope classification at step-5** is what determines routing:

|Scope|Handler|Trigger|
|---|---|---|
|Minor|Developer agent directly|New story or story modification|
|Moderate|PO + Developer|Backlog reorganization needed|
|Major|PM + Architect|Fundamental replan required|

This is BMAD's escalation model — the three-tier routing based on how much of the plan breaks.

**For your harness:** The scope classification taxonomy (minor/moderate/major) and the escalation routing are directly applicable. When your autonomous agent hits a problem it can't resolve, it needs to classify severity and route appropriately — this is exactly your circuit breaker pattern from the risk register.

**What's wrong with it for autonomous use:** It's designed to be run interactively with a human at every step. Step-1 asks the user to describe the issue. Step-3 presents proposals one by one for approval. Step-5 asks for explicit yes/no. None of this is automatable as-is.

---
### checklist.md — The impact analysis checklist

**This is the most valuable file in the folder.** It's a systematic 6-section checklist for understanding the full impact of a change:

**Section 1: Understand the Trigger**

- What story triggered this
- Issue type classification:
    - Technical limitation discovered during implementation
    - New requirement from stakeholders
    - Misunderstanding of original requirements
    - Strategic pivot
    - Failed approach requiring different solution
- Halt conditions: can't proceed without clear trigger + concrete evidence

**Section 2: Epic Impact Assessment**

- Can the current epic still complete as planned?
- Do future epics need changes?
- Does the issue invalidate any planned epics?
- Should epic order/priority change?

**Section 3: Artifact Conflict Analysis**

- PRD conflicts (goals, requirements, MVP scope)
- Architecture conflicts (components, patterns, tech stack, data models, APIs, integrations)
- UX conflicts (components, flows, wireframes, interactions, accessibility)
- Secondary artifacts (deployment, IaC, monitoring, testing, CI/CD)

**Section 4: Path Forward Evaluation** Three options evaluated with effort estimate + risk level:

1. **Direct Adjustment** — modify/add stories within current epic structure
2. **Potential Rollback** — revert completed stories to simplify resolution
3. **PRD MVP Review** — reduce scope or redefine goals

**Section 5: Sprint Change Proposal Components** — just a compilation step

**Section 6: Final Review and Handoff** — approval gate + sprint-status.yaml update

**The issue type classification in Section 1 is the most extractable piece.** Five distinct failure types, each pointing to a different remediation path. This is more precise than quick-dev's `intent_gap / bad_spec / patch` taxonomy — it operates at a higher level (why did the plan fail vs why did the implementation fail).

**The three path-forward options in Section 4 are also directly applicable.** Every failure your autonomous harness encounters maps to one of: fix within current scope, revert and try differently, or reduce scope. This is the decision tree your circuit breaker needs.

**For your harness:** The checklist structure is worth automating. Sections 1-4 can be executed by your harness without human input on most problems — the harness can classify the trigger type, assess epic impact, check artifact conflicts, and evaluate path options autonomously. Human escalation only needed for Moderate/Major scope.

---
### customize.toml — Configuration

Same pattern as the others. `persistent_facts` loads `project-context.md`. Nothing new here.

---
## Consolidated: Pros and Cons

**Genuine strengths:**

1. **Issue type classification** — five categories (technical limitation / new requirement / misunderstanding / strategic pivot / failed approach) is precise. Each maps to a different remediation path. This is more useful than generic "something went wrong."
    
2. **Three path-forward options** — Direct Adjustment / Rollback / MVP Review is a clean decision tree. Every failure scenario maps to one of these. The effort + risk framing per option is the right evaluation structure.
    
3. **Three-tier scope escalation** — Minor (dev handles) / Moderate (PO+dev) / Major (PM+architect) is directly applicable to your harness's circuit breaker design.
    
4. **Full artifact impact assessment** — the checklist covers PRD, epics, architecture, UX, and secondary artifacts (CI/CD, deployment, monitoring). For GIS360, this translates to: pipe data model, MVT rendering pipeline, spatial operations, work correction logic.
    
5. **Old → new format for change proposals** — explicit before/after with rationale per artifact. This is how you present course corrections to humans in a reviewable way.
    

**Real weaknesses:**

1. **Not agent self-correction — it's replanning** — this runs when the _plan_ is wrong, not when the _implementation_ is wrong. Quick-dev's step-04 handles implementation correction. Correct-course handles when the spec, epic, or PRD itself needs to change. For your harness, these are different loops.
    
2. **Entirely human-gated** — every step halts for user input. The checklist has explicit HALT conditions that require human resolution before proceeding. Unusable for autonomous operation without significant redesign.
    
3. **FULL_LOAD strategy at scale** — loading PRD + epics + architecture + UX + specs entirely is fine for small projects. For GIS360's scale, this will overflow the context window. The selective loading logic from quick-dev's epic context compiler is more appropriate.
    
4. **No connection to quick-dev's review loop** — correct-course is triggered manually ("when the user says 'correct course'"). There's no automatic promotion from quick-dev's `intent_gap` finding to a correct-course run. This gap means issues that require replanning get stuck at quick-dev's step-04 with no path forward.
    
5. **Sprint-status coupling** — checklist item 6.4 updates sprint-status.yaml. This is BMAD framework lock-in, not a transferable pattern.
    
6. **No learning across corrections** — once the Sprint Change Proposal is written and approved, there's no mechanism to update the agent's memory so it doesn't make the same class of mistake again. Each correct-course run is isolated.
    

---
## What to extract for your harness

|Correct-course mechanism|What to take|What to change|
|---|---|---|
|Issue type classification (5 categories)|Take it wholesale|Add GIS360-specific types: spatial data model mismatch, MVT rendering constraint violation|
|Three path-forward options (Adjust/Rollback/MVP Review)|Take the structure|Automate options 1 and 2; only escalate option 3 to human|
|Three-tier scope escalation (Minor/Moderate/Major)|Take it wholesale|Wire to your circuit breaker threshold — Minor = auto-resolve, Moderate = notify human, Major = HALT|
|Artifact impact checklist (PRD/arch/UX/secondary)|Take the categories|Replace FULL_LOAD with selective retrieval from your product knowledge system|
|Old → new proposal format|Take for human-facing output|None|
|HALT conditions with specific triggers|Take the pattern|Make them automatic thresholds, not human prompts|
|Escalation routing by scope|Take it|Route Minor/Moderate to automated sub-agents, Major to human|

---
## The critical insight: where this fits in your harness

BMAD has **two separate correction mechanisms** that solve different problems:

```
quick-dev step-04 review        correct-course
────────────────────────        ──────────────────────────
Implementation is wrong    vs.  Plan is wrong
Bad_spec / intent_gap      vs.  Epic / PRD / architecture wrong
Loop back within task      vs.  Replanning across tasks
Spec Change Log tracks it  vs.  Sprint Change Proposal documents it
Circuit breaker at 5 loops vs.  Scope classification → escalation
```

Your harness needs **both loops**:

- Inner loop: quick-dev step-04 (implementation correction, automatic)
- Outer loop: correct-course equivalent (plan correction, human-gated for Major scope)

The connection between them — promoting an `intent_gap` that exceeds 5 loops into a correct-course run — is **missing in BMAD**. That gap is where regressions accumulate. Your harness should explicitly wire this: when the inner loop circuit breaker trips, automatically classify the failure type, assess scope, and either resolve (Minor/Moderate) or escalate (Major).

---
## What bmad-validate-prd actually is

A **spec compliance verification system** — it takes a PRD and runs it through 13 sequential validation checks, building an append-only report as it goes. Most steps run autonomously (no human input). Human interaction only at start (step-01) and end (step-13).

The flow:

```
step-01: discovery          — find PRD, load inputs, init report
step-02: format detection   — BMAD Standard / Variant / Non-Standard
  └─ step-02b: parity check — if Non-Standard, gap analysis (branch)
step-03: density            — anti-pattern scan (auto)
step-04: brief coverage     — PRD vs Product Brief mapping (auto, conditional)
step-05: measurability      — FR/NFR testability (auto)
step-06: traceability       — chain: vision → criteria → journeys → FRs (auto)
step-07: implementation leakage — WHAT not HOW (auto)
step-08: domain compliance  — healthcare/fintech/govtech requirements (auto, conditional)
step-09: project-type       — api_backend vs web_app vs mobile_app etc. (auto)
step-10: SMART scoring      — score every FR 1-5 on 5 axes (auto)
step-11: holistic quality   — whole-document assessment + rating/5 (auto)
step-12: completeness       — final gate: template vars, missing sections (auto)
step-13: report complete    — synthesize, present, offer next steps (human)
```

---
## File-by-file breakdown

### SKILL.md + customize.toml — Orchestrator

Same activation pattern as the others. The `on_complete` hook only fires on `[X] Exit` from step-13 — not on `[E] Edit`, `[R] Review`, or `[F] Fix` loops. This is precise: the hook is for true completion, not intermediate loops.

**For your harness:** The `on_complete` specificity (exit only, not loop) is a good pattern. Your spec compliance gate should distinguish between "validation passed, proceed" and "validation found issues, loop" at the hook level.

---
### prd-purpose.md — The validation standard

This is the **ground truth document** that defines what a valid spec looks like. Every subsequent step validates against this. Key principles it encodes:

**Information density:** Every sentence carries weight. Anti-patterns are explicit: "The system will allow users to..." → "Users can..."

**Traceability chain:**

```
Vision → Success Criteria → User Journeys → Functional Requirements
```

Every FR must trace back through this chain. Orphan FRs (no traceable source) are a critical violation.

**SMART FRs:** Specific, Measurable, Attainable, Relevant, Traceable. Template: `"[Actor] can [capability]"`. No subjective adjectives (easy, fast, intuitive). No vague quantifiers (multiple, several, some).

**SMART NFRs:** Template: `"The system shall [metric] [condition] [measurement method]"`. Never unmeasurable ("scalable" → "handles 10x load through horizontal scaling").

**Implementation leakage:** FRs specify WHAT not HOW. No technology names, library names, or architecture patterns in FRs unless capability-relevant.

**For your harness:** This document is your GIS360 spec standard. You need an equivalent — a `gis360-spec-standard.md` that encodes what a valid GIS360 requirement looks like, including domain-specific rules (spatial operations, MVT rendering constraints, pipe data model requirements). The prd-purpose.md structure (anti-patterns with examples, SMART criteria, traceability chain) is directly reusable.

---
### step-v-01-discovery.md — Setup

Finds the PRD, loads all input documents from frontmatter, initializes the validation report. Key design decisions:

- Loads `prd-purpose.md` first — internalizes the standard before any validation begins
- Asks about additional reference documents — handles the case where relevant context isn't in the frontmatter
- Initializes report with YAML frontmatter tracking `validationStatus: IN_PROGRESS` and `validationStepsCompleted: []`
- Append-only report: each subsequent step appends findings, never overwrites

**For your harness:** The append-only validation report pattern is directly applicable. Your spec compliance checker should build a cumulative report across all validation phases — not a pass/fail flag but a structured findings document. The `validationStepsCompleted` array lets you resume interrupted validation.

---
### step-v-02-format-detection.md + step-v-02b-parity-check.md — Format routing

Classifies PRD format by counting which of 6 core sections are present:

- **BMAD Standard** (5-6 sections): proceed directly to validation
- **BMAD Variant** (3-4 sections): proceed directly to validation
- **Non-Standard** (<3 sections): pause, offer parity check or validate-as-is

The parity check (step-02b) analyzes each missing section for effort: Minimal / Moderate / Significant, then overall: Quick / Moderate / Substantial. User decides whether to proceed or fix gaps first.

**For your harness:** The format detection → routing pattern is directly applicable to your requirement intake. When a requirement comes in, classify it: does it have enough structure to generate a spec from? The three-tier classification (sufficient / partial / insufficient) with different routing per tier is clean. The parity check's effort estimation (Minimal/Moderate/Significant per section) is how you should communicate spec gaps to the human in your human-in-the-loop gate.

---
### step-v-03-density-validation.md — Anti-pattern scan

**Runs autonomously.** Scans for three anti-pattern categories:

1. Conversational filler: "The system will allow users to...", "It is important to note that..."
2. Wordy phrases: "Due to the fact that", "In the event of", "At this point in time"
3. Redundant phrases: "Future plans", "Past history", "Absolutely essential"

Severity: Critical (>10), Warning (5-10), Pass (<5).

Attempts subprocess (Task tool) first, degrades gracefully to inline analysis if unavailable.

**For your harness:** This is automatable. You can run this as a deterministic text scan, not an LLM check — regex patterns are more reliable than LLM judgment for this. The subprocess → graceful degradation pattern is good for any validation step where you want parallelism but can't depend on it.

---
### step-v-04-brief-coverage-validation.md — Source traceability

**Conditional: only runs if a Product Brief was loaded.** Maps brief content (vision, users, problem, features, goals, differentiators) to PRD sections. Four-way classification per item:

- Fully Covered
- Partially Covered
- Not Found
- Intentionally Excluded

Severity per gap: Critical / Moderate / Informational.

**For your harness:** This is your requirement-to-spec traceability check. For GIS360, the equivalent is: does the spec trace back to the feature request / bug report / user story that triggered it? The four-way coverage classification (Fully/Partially/Not Found/Intentionally Excluded) is more precise than binary pass/fail and is worth adopting. "Intentionally Excluded" is important — it acknowledges valid scoping decisions rather than flagging them as gaps.

---
### step-v-05-measurability-validation.md — FR/NFR testability

Extracts all FRs and NFRs and checks each for:

**FRs:**

- `[Actor] can [capability]` format compliance
- No subjective adjectives (easy, fast, simple, intuitive, responsive)
- No vague quantifiers (multiple, several, some, many, various)
- No implementation details (React, PostgreSQL, AWS, etc.)

**NFRs:**

- Specific metric with measurement method
- Template compliance (criterion + metric + measurement method + context)

Severity: Critical (>10 violations), Warning (5-10), Pass (<5).

**For your harness:** This is your acceptance criteria validator. Your spec template already uses Given/When/Then — this is the equivalent check for that format. For GIS360's acceptance criteria specifically, you'd add: spatial precision requirements (coordinate accuracy), performance bounds (MVT tile render time), and data integrity constraints (pipe geometry validity). The FR violation taxonomy (format / subjective adjective / vague quantifier / implementation leakage) is the right set of checks to run before allowing a spec to proceed to implementation.

---
### step-v-06-traceability-validation.md — Chain integrity

Validates the complete traceability chain:

```
Executive Summary → Success Criteria (vision alignment)
Success Criteria → User Journeys (are criteria achievable via the defined flows?)
User Journeys → FRs (does each FR trace to a journey?)
Scope → FRs (does MVP scope align with essential FRs?)
```

Identifies **orphan FRs** — requirements with no traceable source. Orphan FRs are a Critical violation (the harshest severity in the entire validation pipeline).

**For your harness:** The orphan FR concept maps directly to your spec's `Tasks & Acceptance` section. Every acceptance criterion should trace to the spec's Intent section. Any AC that can't be traced to the Intent is an orphan — it signals scope creep or misaligned implementation. The traceability matrix structure (build it, then find broken chains) is your spec compliance auditor's primary output.

---
### step-v-07-implementation-leakage-validation.md — WHAT not HOW

Scans FRs and NFRs for technology names and implementation terms, with an important nuance: **distinguishes capability-relevant from implementation leakage**.

- "API consumers can access data via REST endpoints" → REST is capability-relevant (WHAT)
- "React component fetches data using Redux" → React/Redux is implementation leakage (HOW)

Categories: Frontend frameworks, Backend frameworks, Databases, Cloud platforms, Infrastructure, Libraries, Data formats, Architecture patterns.

Severity: Critical (>5), Warning (2-5), Pass (<2).

**For your harness:** The capability-relevant vs. implementation-leakage distinction is the most nuanced check in the pipeline and the hardest to automate reliably. For GIS360, you'd add GIS-specific terms to the scan list: PostGIS, Mapbox, Leaflet, OpenLayers, proj4, turf.js — these are implementation details. The capability-relevant framing would be: "Users can view pipe routes on a map" not "Leaflet renders pipe geometry using GeoJSON."

---
### step-v-08-domain-compliance-validation.md — Regulatory requirements

**Conditional: only runs detailed checks for high-complexity domains.** Uses `domain-complexity.csv` to determine complexity level. High-complexity domains (Healthcare, Fintech, GovTech, EdTech, Legal) have required sections:

- Healthcare: Clinical Requirements, Regulatory Pathway, HIPAA Compliance, Patient Safety
- Fintech: Compliance Matrix, Security Architecture, Audit Requirements, Fraud Prevention
- GovTech: WCAG 2.1 AA, Section 508, FedRAMP, Data Residency

Low-complexity domains skip this check with N/A.

**For your harness:** GIS360 is a GIS application for field operations — this maps to GovTech/infrastructure depending on the customer. Your equivalent domain compliance check would cover: spatial data accuracy standards, coordinate system requirements (projection compliance), pipe data regulatory standards (if gas/water utilities). The domain-complexity.csv pattern (external data file drives conditional validation logic) is the right architecture for domain-specific checks — it makes the rules maintainable without changing the validation code.

---
### step-v-09-project-type-validation.md — Platform-specific requirements

Uses `project-types.csv` to determine required and excluded sections per project type:

- `api_backend`: Required = endpoint specs, auth model, data schemas. Excluded = UX/UI sections
- `mobile_app`: Required = platform requirements, device permissions, offline mode
- `cli_tool`: Required = command structure, output formats, config schema. Excluded = visual design, touch interactions
- `ml_system`: Required = model requirements, training data, inference requirements, model performance

Both presence and absence are validated — an api_backend PRD with UX sections is a violation.

**For your harness:** GIS360 spans multiple project types — it has a backend (spatial data APIs), a web frontend (map UI), and possibly mobile. The project-type classification needs to be explicit in the spec frontmatter so validation applies the right ruleset. The excluded-sections check is underrated — it prevents scope inflation where agents add irrelevant sections because they "seem thorough."

---
### step-v-10-smart-validation.md — FR quality scoring

Scores every FR on all 5 SMART criteria (1-5 scale each):

|Criterion|5|3|1|
|---|---|---|---|
|Specific|Clear, unambiguous|Somewhat clear|Vague|
|Measurable|Quantifiable, testable|Partially measurable|Subjective|
|Attainable|Realistic with constraints|Probably achievable|Infeasible|
|Relevant|Clear alignment to user needs|Somewhat relevant|Doesn't align|
|Traceable|Clear source journey/objective|Partially traceable|Orphan|

FRs with any score <3 get improvement suggestions. Overall quality: % of FRs with all scores ≥3.

Severity: Critical (>30% flagged), Warning (10-30%), Pass (<10%).

**This is the most actionable validation output in the entire pipeline.** It produces a per-FR quality table with specific improvement suggestions, not just a pass/fail verdict.

**For your harness:** The SMART scoring table is the output you want from your spec compliance checker — not "this spec has 3 violations" but "FR-007 scores 2 on Measurable because it uses 'fast' without a metric, suggested fix: add response time bound." The 1-5 scale per criterion gives you gradients rather than binary flags. The overall quality threshold (>30% flagged = Critical) gives you an automatable gate for whether a spec is ready for implementation.

---
### step-v-11-holistic-quality-validation.md — Whole-document assessment

The only step that evaluates the PRD as a complete document rather than checking individual components. Uses Advanced Elicitation (multi-perspective evaluation):

**Four perspectives:**

1. Document flow and coherence — narrative, transitions, consistency, readability
2. Dual audience effectiveness — for humans (executives, developers, designers, stakeholders) AND for LLMs (UX readiness, architecture readiness, epic/story readiness)
3. BMAD PRD principles compliance — 7 principles scored as Met/Partial/Not Met
4. Overall quality rating — 1-5 scale (Problematic → Excellent)

Outputs: rating/5, top 3 improvements.

**The "LLM readiness" assessment is the most novel idea here.** It explicitly asks: can an LLM generate UX designs, architecture, epics, and stories from this PRD? This is BMAD's acknowledgment that the PRD is consumed by downstream agents, not just humans. The dual audience framing (human-readable AND LLM-consumable) is the central design philosophy of the whole PRD format.

**For your harness:** Your spec equivalent of holistic quality is: can your implementation agent implement this spec correctly without hallucinating missing details? The "LLM readiness" check maps to: is the Code Map complete? Are the ACs unambiguous? Is the Intent section tight enough to anchor the frozen block? The 5-point scale and top-3-improvements format is how you should present spec quality to the human at CHECKPOINT 1 in quick-dev's step-02.

---
### step-v-12-completeness-validation.md — Final gate

The last automated check before the human sees results. Four checks:

1. **Template completeness** — scan for unfilled variables (`{variable}`, `{{variable}}`, `[placeholder]`)
2. **Content completeness** — each required section has actual content
3. **Section-specific completeness** — success criteria are measurable, journeys cover all user types, FRs cover MVP scope, NFRs have specific criteria
4. **Frontmatter completeness** — `stepsCompleted`, `classification`, `inputDocuments`, `date` all present

Severity: Critical if template variables exist or critical sections missing.

**For your harness:** The template variable scan is directly applicable and automatable — regex check for `{variable}` patterns in the spec before handing to implementation. The frontmatter completeness check maps to your spec's required frontmatter fields (`title`, `type`, `status`, `baseline_commit`). This is the gate that prevents a half-filled spec from proceeding to implementation.

---
### step-v-13-report-complete.md — Final presentation

Synthesizes all findings from steps 1-12. Computes overall status (Pass/Warning/Critical) from all findings. Presents a quick results table, critical issues, warnings, strengths, holistic rating, and top 3 improvements.

Four exit options:

- **[R] Review** — walk through section by section (loop)
- **[E] Edit** — invoke `bmad-edit-prd` with the validation report as context (hand off)
- **[F] Fix simpler items** — immediate fixes for anti-patterns, leakage, missing headers
- **[X] Exit** — save and exit (triggers `on_complete` hook)

The `[E] Edit` path is the most important: it hands the validation report to the edit workflow as context, enabling systematic remediation rather than manual reading-and-fixing.

**For your harness:** The four-option menu is your post-validation decision gate. The equivalent for your harness: after spec compliance validation, the agent can (E) fix the spec autonomously for bad_spec issues, (F) apply trivial patches for minor issues, (R) present findings to human for intent_gap issues, or (X) approve and proceed to implementation. The validation report as context for the fix workflow is exactly how quick-dev's Spec Change Log feeds into the re-derivation loop.

---
## Consolidated: Pros and Cons

**Genuine strengths:**

1. **13-step automated pipeline** — each check is isolated, focused, and autonomous. This is not ad-hoc review — it's a systematic quality gate with defined criteria for every dimension.
    
2. **Append-only validation report** — builds a complete audit trail across all checks. The report persists and can be consumed by the edit/fix workflow. This is your Spec Change Log equivalent at the PRD level.
    
3. **Conditional validation** — steps 04, 08, 09 skip gracefully when prerequisites aren't present. The domain and project-type checks use external CSV data files to drive conditional logic — making rules maintainable without code changes.
    
4. **Subprocess architecture with graceful degradation** — every automated step tries the Task tool first, falls back to inline analysis. Your harness should use the same pattern for any validation step where parallelism is desirable but not required.
    
5. **SMART scoring table (step-10)** — per-FR quality scores with specific improvement suggestions. This is the most actionable output in the pipeline — tells you exactly which requirements need work and why.
    
6. **Dual audience framing (step-11)** — explicitly evaluates LLM readiness alongside human readability. Most validation systems only check the latter. For your harness, LLM readiness is the primary audience.
    
7. **Four-way coverage classification** (Fully/Partially/Not Found/Intentionally Excluded) — more precise than binary pass/fail, and explicitly handles valid scoping decisions.
    
8. **Template variable scanning (step-12)** — catches unfilled placeholders before implementation. Simple but critical.
    

**Real weaknesses:**

1. **Designed for PRDs, not implementation specs** — the validation pipeline checks vision → criteria → journeys → FRs. Your harness needs an equivalent for spec → Code Map → Tasks → ACs. The structure is right but the content is wrong for your use case.
    
2. **Human-gated at step-01 and step-13 only** — for interactive use this is appropriate. For your autonomous harness, you want the validation to run end-to-end without step-01 setup and with automated routing at step-13 based on severity.
    
3. **Severity thresholds are arbitrary** — Critical (>10 violations), Warning (5-10), Pass (<5) are round numbers without empirical basis. For GIS360, you need domain-calibrated thresholds based on what violation counts actually predict implementation failure.
    
4. **No connection to implementation outcome** — the validation report tells you if the PRD is well-formed. It doesn't tell you if a well-formed PRD will produce correct implementations. Your harness's equivalent needs to close this loop — validate the spec, implement, check if validation score predicts implementation quality.
    
5. **CSV data files are static** — `domain-complexity.csv` and `project-types.csv` encode domain rules that you'd want to update as GIS360 evolves. The architecture is right (external data drives validation logic) but the format (CSV) is fragile. YAML or JSON with schema validation would be more maintainable.
    
6. **Step-02b parity check adds human interaction for non-standard PRDs** — for your harness, non-conforming specs should be auto-routed to the spec editor, not paused for human input.
    

---
## What to extract for your harness

| Validate-PRD mechanism                               | What to take                                              | What to change                                                                                                       |
| ---------------------------------------------------- | --------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------- |
| 13-step pipeline structure                           | Take the architecture — isolated, sequential, append-only | Adapt checks from PRD validation to spec validation                                                                  |
| prd-purpose.md as ground truth                       | Take the concept — create `gis360-spec-standard.md`       | Replace PRD rules with spec rules + GIS360-specific constraints                                                      |
| Append-only validation report                        | Take wholesale                                            | Add `validationPredictedOutcome` field to report frontmatter                                                         |
| Density scan (step-03)                               | Take it — automatable as regex                            | Replace BMAD anti-patterns with your spec anti-patterns                                                              |
| Four-way coverage classification                     | Take it                                                   | Apply to requirement→spec traceability, not brief→PRD                                                                |
| Measurability check (step-05)                        | Take wholesale                                            | Add GIS360-specific requirement formats (spatial precision, tile render time)                                        |
| Traceability chain (step-06)                         | Take wholesale                                            | Chain: Intent → Boundaries → ACs → Tasks (your spec structure)                                                       |
| Implementation leakage (step-07)                     | Take wholesale                                            | Add GIS360 tech stack to scan list (PostGIS, Mapbox, tile server names)                                              |
| Domain/project-type conditional checks (steps 08-09) | Take the pattern — CSV-driven conditional logic           | Create GIS360-specific validation rules per feature area                                                             |
| SMART scoring table (step-10)                        | Take wholesale                                            | This directly replaces BMAD's FR quality check for your ACs                                                          |
| Holistic quality + LLM readiness (step-11)           | Take the LLM readiness framing                            | Replace dual-audience with: "Can the implementation agent implement this correctly?"                                 |
| Template variable scan (step-12)                     | Take wholesale — automatable                              | Add your spec frontmatter required fields                                                                            |
| Four exit options at step-13                         | Take the structure                                        | Automate E (fix bad_spec), F (patch), keep R (human review for intent_gap), automate X (approve → step-03 implement) |
| Subprocess + graceful degradation                    | Take as a universal pattern for all validation steps      | None                                                                                                                 |
| Severity thresholds (Critical/Warning/Pass)          | Take the three-tier structure                             | Calibrate thresholds empirically against GIS360 task outcomes                                                        |

---
## How validate-prd connects to the other skills

```
validate-prd output feeds:          consumed by:
───────────────────────             ────────────────────────────────
validation report (full)       →    bmad-edit-prd (systematic fixes)
SMART scoring table            →    quick-dev step-02 (spec quality at CHECKPOINT 1)
Holistic quality rating        →    quick-dev step-01 (routing: is spec ready?)
Critical findings              →    correct-course (plan-level issues)
Traceability violations        →    quick-dev step-04 review (intent_gap detection)
Template variable scan         →    quick-dev step-03 (pre-implementation gate)
```

**For your harness:** validate-prd is the spec quality gate that runs between quick-dev's step-02 (plan) and step-03 (implement). In the full pipeline:

```
quick-dev step-01: intake + route
quick-dev step-02: generate spec → CHECKPOINT 1 (human approves)
[YOUR EQUIVALENT OF validate-prd]: automated spec quality check
    → SMART score < threshold → loop back to step-02
    → traceability gaps → loop back to step-02
    → implementation leakage → auto-fix + continue
    → template variables → auto-fix + continue
    → passes all gates → proceed to step-03
quick-dev step-03: implement
quick-dev step-04: adversarial review
```

This is the missing layer between human approval at CHECKPOINT 1 and implementation. A human can approve a spec that's still poorly formed. The automated validation catches what human review misses.

---
## What bmad-distillator actually is

A **lossless context compression system**. It takes large source documents, extracts every discrete piece of information, compresses the language, deduplicates across sources, and produces a token-efficient distillate that can reconstruct the original. The round-trip reconstructor then proves the distillate is complete by rebuilding the source from the distillate alone.

The architecture is a three-agent pipeline:

```
analyze_sources.py          → routing decision (single vs fan-out, split prediction)
distillate-compressor.md    → extraction + compression + output
round-trip-reconstructor.md → completeness verification (reconstruct from distillate only)
```

The SKILL.md (not uploaded but referenced) orchestrates these three. The agents and scripts are the actual implementation.

---
## File-by-file breakdown

### analyze_sources.py — The routing brain

**What it does:** A pure Python script (no dependencies) that analyzes input files and makes two decisions before any LLM call happens:

**Decision 1: Routing** — single compressor or fan-out?

- Single: ≤3 files AND ≤15,000 estimated tokens
- Fan-out: >3 files OR >15,000 tokens

**Decision 2: Split prediction** — will the distillate need semantic splitting?

- Distillate estimated at ~1/3 of source size (empirical ratio)
- If estimated distillate >5,000 tokens → splitting likely

Also does:

- File discovery (files, folders, globs, recursive)
- Skips `node_modules`, `.git`, `__pycache__`, `.venv`, `_bmad-output`
- Includes only `.md`, `.txt`, `.yaml`, `.yml`, `.json`
- Document type detection from filename patterns (14 types: product-brief, discovery-notes, architecture-doc, prd, specification, etc.)
- Group detection — pairs related documents by naming convention (brief + discovery-notes, base + appendix, base + review)

**Output JSON structure:**

```json
{
  "status": "ok",
  "files": [{"path", "filename", "size_bytes", "estimated_tokens", "doc_type"}],
  "summary": {"total_files", "total_size_bytes", "total_estimated_tokens"},
  "groups": [{"group_key", "files": [{"path", "filename", "role": "primary|companion|standalone"}]}],
  "routing": {"recommendation": "single|fan-out", "reason"},
  "split_prediction": {"prediction": "likely|unlikely", "reason", "estimated_distillate_tokens"}
}
```

**The 1/3 compression ratio is the critical empirical assumption.** The script estimates distillate size as source_tokens ÷ 3. If this ratio is wrong for your content type (GIS360 has dense technical documentation), the routing decisions will be wrong too.

**For your harness:** This script is directly usable. You need this exact analysis step before loading any context into your agents — you need to know token counts, identify related document groups, and decide whether to compress before injecting. The document type detection and grouping patterns are things you'd extend for GIS360 (add: `pipe-model`, `mvt-spec`, `spatial-constraints`, `work-correction-logic`).

---
### distillate-compressor.md — The compression engine

**What it does:** A 7-step extraction and compression process:

**Step 1: Read sources** — identify document type from content + naming

**Step 2: Extract** — entity extraction pass. Every discrete piece of information regardless of position:

- Facts and data points (numbers, dates, versions, percentages)
- Decisions made + rationale
- **Rejected alternatives + why** (explicitly called out)
- Requirements and constraints (explicit and implicit)
- Relationships and dependencies
- Named entities
- Open questions and unresolved items
- Scope boundaries (in/out/deferred)
- Success criteria and validation methods
- Risks and opportunities
- User segments and success definitions

**Step 3: Deduplicate** — per compression-rules.md

**Step 4: Filter** — only if `downstream_consumer` is specified. "When uncertain, keep the item." Never drop: decisions, rejected alternatives, open questions, constraints, scope boundaries.

**Step 5: Group thematically** — derived from content, not a fixed template. Common groupings: core concept, solution/approach, users/segments, technical decisions, scope, competitive context, success criteria, rejected alternatives, open questions, risks.

**Step 6: Compress language** — per compression-rules.md:

- Strip prose transitions and connective tissue
- Remove hedging and rhetoric
- Remove common knowledge explanations
- Preserve specific details (numbers, names, versions)
- Make relationships explicit ("X because Y", "X blocks Y", "X replaces Y")
- Each bullet self-contained

**Step 7: Format output** — `##` headings, `-` bullets only. No prose paragraphs. Semicolons for related short items. No decorative formatting.

**Returns structured JSON** back to the calling skill (not conversational text):

```json
{
  "distillate_content": "...",
  "source_headings": ["..."],
  "source_named_entities": ["..."],
  "token_estimate": N,
  "sections": null or [{"topic": "...", "content": "..."}]
}
```

**The downstream_consumer filter is the most important mechanism.** When you specify "PRD creation" as the consumer, the compressor drops information irrelevant to PRD creation. When you specify "code memory for MVT rendering", it drops product strategy information and keeps only technical decisions relevant to that module. This is selective context injection without a retrieval system — the LLM does the filtering.

**For your harness:** This is your context compression for Product Knowledge (Milestone 3) and Code Memory (Milestone 4). The `downstream_consumer` parameter maps directly to your knowledge routing decision — which knowledge type (product vs code vs dynamic) to inject for which task phase. The entity extraction taxonomy (facts, decisions, rejected alternatives, constraints, relationships, open questions, scope, success criteria, risks) is your distillate schema.

---
### round-trip-reconstructor.md — Completeness verifier

**What it does:** Takes ONLY the distillate (no access to originals) and reconstructs the source documents. If the reconstruction is faithful, the distillate is proven complete. If gaps exist, they surface as `[POSSIBLE GAP]` markers.

Process:

1. Read distillate frontmatter (sources, downstream_consumer, parts count)
2. Infer document types from source names + distillate content
3. Reconstruct each source as full prose — expand bullets back to natural language, restore transitions and framing
4. Flag gaps: `[POSSIBLE GAP: missing context for X]`, `[POSSIBLE GAP: expected more on X for a document of this type]`, `[POSSIBLE GAP: relationship between X and Y unclear]`
5. Save reconstruction files adjacent to distillate
6. Return JSON: reconstruction file paths, possible gaps list, source count

**The blind constraint is what makes this work.** The reconstructor must NOT see the originals. If it could, it would fill gaps from the originals rather than the distillate, making the test meaningless. This is the same logic as quick-dev's Blind Hunter in step-04 — information isolation prevents anchoring bias.

**For your harness:** This is your context quality validation. Before injecting a distillate into your implementation agent, you could run the round-trip test to verify the distillate is complete. More practically: the `[POSSIBLE GAP]` markers are your signal that the compression lost information you need. In your harness, a distillate with >N possible gaps gets flagged for human review before being added to the product knowledge base.

---
### compression-rules.md — The compression standard

**Three categories:**

**Strip — Remove entirely:**

- Prose transitions ("As mentioned earlier", "In addition to this")
- Rhetoric and persuasion ("This is a game-changer")
- Hedging ("We believe", "It's likely that", "Perhaps")
- Self-reference ("This document describes", "As outlined above")
- **Common knowledge explanations** ("Vercel is a cloud platform", "MIT is an open-source license") — important for GIS360: don't explain what PostGIS is
- Repeated introductions of the same concept
- Formatting-only elements

**Preserve — Keep always:**

- Specific numbers, dates, versions, percentages
- Named entities
- Decisions + rationale
- **Rejected alternatives + why** — explicitly always preserved
- Explicit constraints and non-negotiables
- Dependencies and ordering relationships
- Open questions and unresolved items
- Scope boundaries
- Success criteria and validation methods
- User segments and success definitions
- Risks with severity signals
- **Conflicts between source documents** — surfaced explicitly

**Transform — Change form for efficiency:**

- Long prose → single dense bullet
- "We decided to use X because Y and Z" → "X (rationale: Y, Z)"
- "Risk: ... Severity: high" → "HIGH RISK: ..."
- Conditional statements → "If X → Y" form
- Multi-sentence explanations → semicolon-separated form
- "X is used for Y" → "X: Y" when context is clear

**Deduplication rules:**

- Same fact in multiple documents → keep version with most context
- Same concept at different detail levels → keep detailed version
- Overlapping lists → merge, no duplicates
- **Source documents disagree** → "Brief says X; discovery notes say Y — unresolved"
- Executive summary points expanded elsewhere → keep only expanded version

**The conflict detection rule is underrated.** When two source documents disagree about the same fact, the distillate surfaces the conflict explicitly rather than picking one silently. For GIS360, where requirements might conflict between the original spec, the field operations team's input, and the engineering constraints, this is critical.

**For your harness:** These compression rules are your distillate quality standard. Apply them when building your product knowledge base (Milestone 3) and your epic context compiler (which already has similar rules but less precisely specified). The "never drop rejected alternatives" rule maps directly to your Agent Decision Records — rejected approaches should always be preserved.

---
### distillate-format-reference.md — Output examples

Shows the before/after transformation with concrete examples. Key demonstrations:

**Prose paragraph → 2 dense bullets:** A 120-word paragraph about competitive positioning becomes 2 bullets of ~30 words total. ~75% token reduction while preserving all factual content.

**Technical details → compressed facts:** 8-item competitive landscape list becomes 5 bullets — deduplication removes one redundant entry, no-code platforms merged into single bullet with parenthetical list.

**Deduplication across documents:** Brief says "bmad-help must always be included." Discovery notes say the same but adds "(solves discoverability problem)." Distillate keeps the more contextual version, drops the shorter one.

**Decision/rationale compression:** Multi-sentence rationale for a rejected approach → single bullet: "Rejected: own platform support matrix. Reason: unsustainable at 40+ platforms; delegate to Vercel CLI ecosystem."

**The full example distillate** is the most instructive piece. It shows a complete distillate from a product brief + discovery notes, targeted at PRD creation. Key observations:

- ~1,450 tokens total from what was likely 4,000-6,000 token sources (~1/3 ratio confirmed)
- 14 thematic sections
- Every rejected alternative preserved with rationale
- Every open question preserved
- Conflict between sources would be noted
- The frontmatter records sources, consumer, date, token estimate, and parts count

**For your harness:** The full example distillate format is your target output format for the epic context compiler (compile-epic-context.md). Your current compile-epic-context.md produces a similar structure but without the compression discipline, frontmatter tracking, or explicit rejected alternatives section. Adding these would significantly improve your product knowledge quality.

---
### splitting-strategy.md — Large document handling

**When to split:** Source content >~15,000 tokens, or estimated distillate >5,000 tokens.

**Why semantic over size-based:** Arbitrary splits break coherence. A downstream agent loading "part 2 of 4" gets context fragments with no orientation. Semantic splits produce self-contained topic clusters that can be loaded selectively.

**Splitting process:**

1. Identify natural boundaries: distinct problem domains, different stakeholder perspectives, temporal boundaries (current vs future), scope boundaries (in/out/deferred), phase boundaries
2. Assign items to sections — cross-cutting items go in root
3. Root distillate: 3-5 bullet orientation + cross-references + cross-cutting items
4. Section distillates: self-sufficient, with 1-line context header + content + cross-references

**Output structure:**

```
{base-name}-distillate/
├── _index.md           ← root: orientation, cross-cutting, section manifest
├── 01-{topic-slug}.md  ← self-contained section
├── 02-{topic-slug}.md
└── 03-{topic-slug}.md
```

**Size targets:**

- Root: ~20% of token budget
- Sections: remaining 80% split proportionally by content density
- No budget specified: aim for 3,000-5,000 tokens per section

**The root distillate as orientation layer** is the key design decision. When your implementation agent needs context, it loads the root first, then selectively loads only the section distillates relevant to the current task. This is selective retrieval without a vector database — the root acts as an index.

**For your harness:** This is your product knowledge retrieval architecture for large documents. Your epic context compiler currently loads entire planning artifacts. For GIS360's scale, you need this split structure: `_index.md` as the always-loaded layer, section distillates loaded selectively per task. The 3,000-5,000 token target per section aligns with your 900-1600 token spec target — sections are roughly 2-3 specs worth of context.

---
## Consolidated: Pros and Cons

**Genuine strengths:**

1. **The 1/3 compression ratio with empirical basis** — source docs compress to ~1/3 their original size losslessly. This is the most important number in the whole system. If true for GIS360's documentation, it means your 45,000-token GIS360 codebase context becomes 15,000 tokens — still large but manageable.
    
2. **Round-trip verification** — proving distillate completeness by reconstruction is the right quality gate. The blind constraint (reconstructor can't see originals) is what makes it meaningful. This is the only quality signal in BMAD that's actually testable.
    
3. **`downstream_consumer` filtering** — specifying the consumer focuses the distillate on what the downstream agent actually needs. This is selective retrieval without a vector database — the LLM does the filtering during compression, not retrieval.
    
4. **Explicit conflict detection** — when source documents disagree, the distillate surfaces the conflict rather than picking one silently. For GIS360's multi-stakeholder requirements, this prevents silent requirement conflicts from propagating.
    
5. **Rejected alternatives always preserved** — the compression rules explicitly protect this. For your harness, knowing why an approach was rejected is as valuable as knowing what was chosen.
    
6. **Semantic splitting with selective loading** — the root + sections architecture enables loading only relevant context, not everything. This is the right architecture for your Product Knowledge System (Milestone 3).
    
7. **Pure Python script for routing** — `analyze_sources.py` has zero dependencies, runs fast, and makes routing decisions before any LLM call. This is the right separation of concerns.
    
8. **Structured JSON return from agents** — both the compressor and reconstructor return JSON, not conversational text. This makes them composable — the calling skill can process the output programmatically.
    

**Real weaknesses:**

1. **The 1/3 ratio is an assumption** — the script estimates distillate as source ÷ 3, but this varies by document type. Dense technical specs (GIS360's pipe data model, MVT rendering pipeline) may compress less than prose product briefs. The routing decisions based on this estimate may be wrong.
    
2. **No staleness tracking** — the distillate frontmatter records `created` date but there's no mechanism to detect when source documents have changed and the distillate is stale. This is your memory staleness risk from the risk register.
    
3. **Filtering relies on LLM judgment** — the `downstream_consumer` filter asks the LLM "would the downstream workflow need this?" This is unreliable for edge cases. You need more deterministic filtering for your harness.
    
4. **No retrieval from the distillate** — once compressed, the distillate is loaded as a flat file. There's no query interface — you load all of it or none of it. For large distillates that survive splitting, this is still a context problem.
    
5. **Semantic splitting boundaries are LLM-determined** — natural boundaries are identified by the compressor agent, not by a deterministic algorithm. This makes splitting non-reproducible — the same source could split differently on different runs.
    
6. **Round-trip verification is expensive** — running the reconstructor on every distillate doubles the token cost. In practice, you'd run it on initial distillation and after source updates, not on every retrieval.
    
7. **No version control for distillates** — if the source changes and you re-distillate, there's no diff between old and new distillate. Your agent could have stale distillate sections in memory while other sections are fresh.
    
8. **Group detection is pattern-matching only** — `analyze_sources.py` detects related documents by filename patterns. It can't detect semantic relationships between files with non-standard names.
    

---
## What to extract for your harness

|Distillator mechanism|What to take|What to change|
|---|---|---|
|`analyze_sources.py`|Take the entire script|Extend: add GIS360 doc types (pipe-model, mvt-spec, spatial-schema), tune the 1/3 ratio from empirical measurement|
|7-step extraction process|Take wholesale|Add GIS360-specific extraction: spatial constraints, coordinate system decisions, field operation workflows|
|`downstream_consumer` filter|Take the concept|Replace LLM filtering with deterministic tag-based filtering — tag each extracted item with `[product]`, `[code]`, `[UX]` during extraction|
|Compression rules (strip/preserve/transform)|Take wholesale|Add GIS360-specific: never strip spatial precision numbers, always preserve coordinate system decisions|
|Conflict detection|Take wholesale|None|
|Rejected alternatives preservation|Take wholesale|Wire to your Agent Decision Records (Milestone 3, Layer 3)|
|Round-trip verification|Take the concept|Run on first distillation only, then on source changes — not on every retrieval|
|Semantic splitting with root + sections|Take the architecture|Replace LLM-determined boundaries with deterministic section markers in your source docs|
|Structured JSON return from agents|Take as a universal pattern for all your subagents|None|
|Distillate frontmatter (`sources`, `created`, `token_estimate`, `parts`)|Take it|Add: `last_source_modified`, `staleness_threshold`, `consumer_tags`|
|`[POSSIBLE GAP]` markers from reconstructor|Take for your quality pipeline|Wire to your validation report (Milestone 3's retrieval accuracy measurement)|

---
## The critical insight: where this fits in your harness

Distillator solves the **context preparation problem** — how to make large documents consumable by agents with limited context windows. In your harness, it sits between your knowledge stores and your agents:

```
Raw knowledge sources                Your agents
────────────────────                 ───────────────────────────────
GIS360 PRD (50K tokens)             quick-dev step-02 (spec generation)
Architecture docs (30K tokens)  →   distillate (15K tokens) →  agent gets
Pipe data model (20K tokens)        selective section (3K tokens)   right context
Field ops workflows (15K tokens)    at right granularity
MVT rendering spec (10K tokens)
```

The connection to your six-layer product knowledge model from Milestone 3:

|Layer|Distillator role|
|---|---|
|Layer 1: Product Constitution|Root distillate (`_index.md`) — always loaded, cross-cutting facts|
|Layer 2: Feature Specifications|Section distillates — loaded selectively per epic|
|Layer 3: Decision Memory|Preserved in every distillate: decisions + rationale + rejected alternatives|
|Layer 4: Product Knowledge Graph|Not addressed by distillator — this is where you need Zep/Graphiti or similar|
|Layer 5: Design and UX Memory|Section distillate — loaded for UI-touching tasks|
|Layer 6: Session and Progress State|Not addressed — this is your Dynamic Memory (Milestone 5)|

**Distillator covers Layers 1-3 well. It doesn't address Layers 4-6.** That's not a weakness — it's scope definition. Your harness needs distillator for static knowledge compression and separate systems for graph relationships and runtime state.

---
## Overall BMAD Analysis: What to build for your harness

Now that we've covered all five key skills, here's the consolidated view of what BMAD gives you and what it doesn't:

**The complete picture of what BMAD does well:**

- Spec format (frozen/mutable split, SMART ACs, Code Map, Change Log)
- Review taxonomy (intent_gap/bad_spec/patch/defer/reject)
- Three-reviewer adversarial pattern with context isolation
- Spec compliance validation pipeline (13 steps)
- Context compression (distillator)
- Concern-based human review presentation
- Risk taxonomy with blast-radius sequencing
- Sprint change management with scope escalation

**What BMAD explicitly does not solve (your harness must build):**

- Product knowledge retrieval (graph + vector + keyword hybrid)
- Code memory (tree-sitter + PageRank, Aider-style)
- Dynamic memory (cross-session learning, error pattern recognition)
- Automated test execution (MCP + Computer Use)
- Blast radius computation (dependency graph, not LLM judgment)
- The inner→outer loop connection (intent_gap → correct-course promotion)
- Distillate staleness detection and invalidation

**The file that should inform your harness design most directly is `distillate-format-reference.md`** — specifically the full example distillate. That output format, combined with quick-dev's spec template, validate-prd's 13-step pipeline adapted for specs, checkpoint-preview's concern-based walkthrough, and correct-course's scope escalation taxonomy, gives you the complete architecture for your harness.

---
## What bmad-generate-project-context actually is

A **3-step collaborative interview** that produces `project-context.md` — the file loaded as `persistent_facts` in every single BMAD skill we've read. It's the foundation everything else depends on.

```
step-01: discover    — scan codebase, identify tech stack, find existing patterns
step-02: generate    — collaboratively build rules per category (7 categories)
step-03: complete    — optimize for LLM efficiency, finalize
```

---

## What it produces

A `project-context.md` with this structure:

```markdown
## Technology Stack & Versions
## Critical Implementation Rules
  ### Language-Specific Rules
  ### Framework-Specific Rules
  ### Testing Rules
  ### Code Quality & Style Rules
  ### Development Workflow Rules
  ### Critical Don't-Miss Rules
## Usage Guidelines
```

With YAML frontmatter tracking: `project_name`, `date`, `sections_completed`, `status`, `rule_count`, `optimized_for_llm`.

---
## The core design philosophy

**"Focus on unobvious details that LLMs need to be reminded of."**

This is the entire point of the file. Not "we use React" — agents know React. Instead: "hooks must use the custom `useProjectHook` wrapper, not raw React hooks, because of our SSR hydration setup." The rules are things that would cause a silent failure if missed.

Step-02 makes this explicit per category: "Don't include obvious rules that agents already know." This is the compression-rules.md equivalent for this skill — strip common knowledge, preserve only what agents would get wrong without being told.

---
## What it's missing — and why that matters for your harness

This skill is **far weaker than the distillator** for knowledge management. The critical gaps:

**1. No compression discipline.** The distillator has 7 extraction steps, deduplication rules, downstream_consumer filtering, and the 1/3 token ratio. Generate-project-context just asks the LLM to "keep content lean." There are no rules governing what gets preserved vs stripped. For a small project this is fine. For GIS360, this will produce an inconsistent, bloated file.

**2. No staleness mechanism.** The file has a `date` field and a "review quarterly" note in the usage guidelines. That's it. No source tracking, no `last_source_modified`, no invalidation trigger. Every other skill loads this file as ground truth — if it's stale, every agent downstream inherits stale context.

**3. Human-interview-only generation.** Every rule is elicited through conversation. There's no automated codebase analysis — step-01 looks at package.json and config files, but the actual rule generation in step-02 is entirely human-input-driven. This means the file quality depends entirely on what the human thinks to mention. For GIS360's domain-specific complexity (pipe geometry, MVT rendering, spatial operations), a human might forget half the critical rules.

**4. Flat structure, no retrieval.** The output is a single markdown file loaded entirely on every activation. For a small project, 1-2K tokens, this is fine. For GIS360, the project-context.md could easily hit 5-10K tokens if done properly, and it's loaded on every single task regardless of relevance. No selective loading, no distillate splitting.

**5. No validation.** Unlike validate-prd's 13-step pipeline, there's no quality gate on what goes into project-context.md. A poorly written rule goes in, stays in, and gets loaded into every agent session until someone notices.

---
## Pros and Cons

**Genuine strengths:**

1. **The "unobvious details" framing** — the explicit goal to exclude common knowledge and focus only on what agents would get wrong is the right filter. This is the same principle as compression-rules.md's "strip common knowledge explanations" but applied to knowledge capture rather than compression.
    
2. **7-category structure** — Technology Stack, Language Rules, Framework Rules, Testing Rules, Code Quality, Workflow Rules, Critical Don't-Miss Rules. This is a clean taxonomy for implementation knowledge. The "Critical Don't-Miss Rules" category (anti-patterns, edge cases, security, performance gotchas) is the most valuable — it captures exactly what quick-dev's adversarial reviewers look for.
    
3. **LLM efficiency as an explicit goal** — step-03 has a dedicated optimization pass: remove redundant rules, combine related rules, remove obvious information. The intent is right even if the mechanism (LLM judgment) is weak.
    
4. **Frontmatter tracking** — `sections_completed`, `status`, `rule_count`, `optimized_for_llm`. Provides a machine-readable completeness signal.
    
5. **Usage guidelines baked in** — the file includes instructions for both agents ("read before implementing, follow all rules, when in doubt prefer the more restrictive option") and humans ("keep lean, update when stack changes, review quarterly"). This makes the file self-documenting.
    

**Real weaknesses:**

1. **Fully human-gated** — every category requires human input. Cannot run autonomously. For your harness, you need automated codebase analysis feeding this, not just a conversation interface.
    
2. **No compression rules** — unlike distillator, there's no objective standard for what a rule should look like. Rules will vary in quality, specificity, and density depending on the human's communication style.
    
3. **Single flat file** — no semantic splitting, no selective loading. Grows unbounded as the project grows.
    
4. **No staleness detection** — the most dangerous weakness given that this file is loaded on every single task.
    
5. **No connection to codebase** — step-01 reads config files, but there's no tree-sitter analysis, no dependency graph, no pattern detection beyond what the LLM can infer from reading package.json. This is the gap your Code Memory system (Milestone 4) fills.
    
6. **No validation** — any rule that gets in stays in with no quality check.
    

---
## What to extract for your harness

| Generate-project-context mechanism                                | What to take                                                            | What to change                                                                                                               |
| ----------------------------------------------------------------- | ----------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------- |
| "Unobvious details only" framing                                  | Take it wholesale as the filter for your Product Constitution (Layer 1) | None                                                                                                                         |
| 7-category rule taxonomy                                          | Take the categories                                                     | Add GIS360-specific categories: Spatial Operations Rules, MVT Rendering Rules, Pipe Data Model Rules, Field Operations Rules |
| Critical Don't-Miss Rules category                                | Take it — this maps to your Spec Change Log patterns                    | Wire to your error pattern recognition system (Milestone 5)                                                                  |
| Frontmatter tracking (`sections_completed`, `status`)             | Take the pattern                                                        | Add `last_source_modified`, `staleness_hash`, `consumer_tags`                                                                |
| "When in doubt, prefer more restrictive option" agent instruction | Take it — this is a good default for your harness                       | None                                                                                                                         |
| LLM efficiency optimization pass                                  | Take the intent                                                         | Replace LLM judgment with compression-rules.md rules applied programmatically                                                |
| Usage guidelines section                                          | Take it                                                                 | Add: how your retrieval system decides whether to load this file                                                             |

---
## The critical realization

Now that we've seen all the files that load `project-context.md`, the picture is complete:

```
project-context.md is loaded by:
  quick-dev         → persistent_facts (every task)
  checkpoint-preview → persistent_facts (every review)
  correct-course    → persistent_facts (every course correction)
  validate-prd      → persistent_facts (every validation)
  generate-project-context → persistent_facts (ironically, even when creating it)
```

**This single file is BMAD's entire product knowledge layer.** Everything from Layer 1 through Layer 5 of your six-layer model gets crammed into one flat file that's loaded regardless of task relevance. It's simple, it works for small projects, and it completely breaks down at GIS360's scale.

Your harness's Milestone 3 (Product Knowledge System) is essentially: replace this single flat file with the six-layer model. The generate-project-context skill shows you what BMAD put into that file — your product knowledge system needs to store and retrieve the same categories of information, just with proper retrieval, compression, staleness detection, and selective loading.

---
## What bmad-review-adversarial-general actually is

A **131-word prompt** that produces the Blind Hunter from quick-dev's step-04. The entire skill is three steps and two halt conditions.

The design is intentional: **minimal instruction = minimal anchoring bias.** The reviewer is given attitude ("cynical, jaded", "clueless weasel submitted this"), a floor ("find at least ten issues"), and almost no other guidance. No spec to check against. No codebase access. Just the content and skepticism.

---
## The mechanics

**Input:** Content to review (diff, spec, story, doc, any artifact) + optional `also_consider` areas.

**Step 1:** Load content. Identify type. Abort if empty.

**Step 2:** Adversarial analysis. Assume problems exist. Find at least 10.

**Step 3:** Output as markdown list. Descriptions only.

**Halt conditions:**

- Zero findings → suspicious, re-analyze. This is a circuit breaker against the LLM's politeness bias — finding nothing is treated as a failure mode, not a pass.
- Empty or unreadable content → abort.

---
## The "at least ten" rule

This is the most important design decision in the entire file. It forces the reviewer past the first few obvious issues into territory it might otherwise rationalize away. An LLM naturally stops when it finds 2-3 significant issues and starts feeling like it's being fair. The floor of ten prevents that — it forces looking for what's _missing_, not just what's _wrong_.

The HALT on zero findings is the enforcement mechanism. "Zero findings is suspicious" — this explicitly overrides the LLM's tendency to say "looks good to me."

---
## The context isolation

The reviewer persona ("zero patience for sloppy work", "submitted by a clueless weasel") is doing specific work: it counteracts the LLM's politeness and in-group bias. When a reviewer knows it's looking at output from "the same system" that prompted it, it unconsciously goes easy. The adversarial framing breaks this by creating psychological distance from the content.

The `also_consider` optional parameter is the only hook into external context. Everything else — no spec, no codebase access, no project context — is deliberately withheld. This is what makes it the "Blind Hunter": it finds issues the spec-aware reviewers rationalize away because they know what the code is _supposed_ to do.

---
## Pros and Cons

**Genuine strengths:**

1. **Minimal and reliable** — 131 words. Nothing to misinterpret, no complex conditional logic, no state tracking. This runs the same way every time.
    
2. **Floor enforcement** — the "at least ten" rule + "zero findings is suspicious" HALT directly counteracts LLM politeness bias. This is the right mechanism.
    
3. **Context isolation by design** — no spec, no codebase, no project context. The Blind Hunter sees only what's actually there, not what was intended. This is the most valuable of the three reviewers precisely because of what it _doesn't_ have.
    
4. **`also_consider` escape hatch** — you can inject domain-specific focus areas without breaking the blind constraint. For GIS360: "also_consider spatial precision and coordinate system consistency" without giving the spec.
    
5. **"Look for what's missing, not just what's wrong"** — this explicit instruction shifts the reviewer from correction mode (finding bugs) to completeness mode (finding gaps). Gaps are harder to find and more expensive to miss.
    

**Real weaknesses:**

1. **No severity classification** — findings are a flat markdown list. The caller (quick-dev step-04) does the classification into intent_gap / bad_spec / patch / defer / reject. But the reviewer itself gives no signal about which findings are critical vs minor. This means the classifier has to do more work and may misclassify.
    
2. **"At least ten" is arbitrary** — for a small change (one-shot path), ten findings may include noise. For a large multi-file change, ten may be too few. The floor doesn't scale with scope.
    
3. **No structured output** — findings are markdown prose, not JSON. The calling skill parses them by reading, not programmatically. This makes deduplication across the three reviewers LLM-dependent.
    
4. **The persona may backfire on legitimate work** — "submitted by a clueless weasel" primes the reviewer to find problems whether they exist or not. For genuinely clean code, this could generate noise that the classifier has to reject. The HALT on zero findings compounds this — the reviewer may invent issues rather than halt.
    
5. **No domain awareness** — the `also_consider` field is optional and requires the calling skill to populate it. In practice, if the caller doesn't specify GIS360-specific concerns, the reviewer applies generic software review heuristics, which may miss domain-specific issues entirely.
    

---
## What to extract for your harness

|Adversarial review mechanism|What to take|What to change|
|---|---|---|
|Blind Hunter concept (no spec, no codebase)|Take wholesale|None — context isolation is the value|
|"At least ten" floor + HALT on zero|Take it|Make the floor scale with diff size: min(10, lines_changed ÷ 20)|
|"Look for missing, not just wrong" instruction|Take it — wire this into your acceptance auditor too|None|
|`also_consider` injection point|Take it|Pre-populate with GIS360-specific concerns: spatial precision, MVT tile validity, pipe geometry integrity, work correction logic|
|Persona-based bias breaking|Take the concept|Tone down "clueless weasel" for your harness — the bias-breaking effect works without the aggression|
|Output as markdown list|Replace with structured JSON|Add severity signal: `[CRITICAL]`, `[MODERATE]`, `[MINOR]` per finding — reduces classifier workload|

---
## How it connects to what we've already covered

Now the full three-reviewer picture from quick-dev step-04 is clear:

|Reviewer|Skill|Gets|Finds|
|---|---|---|---|
|Blind Hunter|`bmad-review-adversarial-general`|Diff only|Issues from pure content inspection, no anchoring|
|Edge Case Hunter|`bmad-review-edge-case-hunter`|Diff + codebase|Untested scenarios, boundary conditions|
|Acceptance Auditor|(inline in step-04)|Diff + spec + context docs|Spec compliance violations|

The Blind Hunter is the most independent and therefore the most likely to catch what the other two rationalize away. That's why it runs first and why it has the strictest bias-breaking controls.

---
## What bmad-review-edge-case-hunter actually is

A **method-driven path tracer** — not attitude-driven like the Blind Hunter. It mechanically walks every branching path and boundary condition, reports only unhandled ones, and returns structured JSON. No editorializing, no opinions, no tone.

The description says it explicitly: "Orthogonal to adversarial review — method-driven not attitude-driven."

---
## The key design decisions

**Scope is diff-first, not file-first.** When a diff is provided, it scans only the diff hunks and reports only boundaries reachable from changed lines. It doesn't review the whole file — just what changed. This prevents noise from pre-existing issues that aren't caused by the current change, which maps directly to quick-dev step-04's classification rule: findings must be "caused by the change."

**Exhaustive path enumeration, not intuition.** The Blind Hunter hunts by attitude — assume problems, find ten. The Edge Case Hunter hunts by method — derive edge classes from the content itself, walk every branch, report gaps. No fixed checklist. The edge classes emerge from the code: conditionals, loops, error handlers, early returns, domain transitions.

**Two-pass validation.** Step 2 finds unhandled paths. Step 3 revisits every edge class from Step 2 to catch anything missed on the first pass. This is a completeness check on the completeness check — the reviewer audits its own work before reporting.

**Structured JSON output.** This is the biggest contrast with the Blind Hunter. Every finding has exactly four fields:

```json
{
  "location": "file:start-end",
  "trigger_condition": "one-line, max 15 words",
  "guard_snippet": "minimal code sketch that closes the gap",
  "potential_consequence": "what goes wrong, max 15 words"
}
```

The `guard_snippet` is particularly valuable — it's not just "this is a problem" but "here's the minimal fix." This gives the classifier in step-04 concrete material to work with when deciding `patch` vs `bad_spec`.

**Empty array is valid.** Unlike the Blind Hunter which HALTs on zero findings (treating it as suspicious), the Edge Case Hunter explicitly allows `[]`. The reasoning is correct: exhaustive path enumeration is a deterministic method. If you walk every path and find no unhandled ones, that's a valid result, not a suspicious one.

**Ignores external functions by default.** "Ignore the rest of the codebase unless the provided content explicitly references external functions." This keeps scope tight and findings actionable. It doesn't speculate about what might be wrong in code it hasn't seen.

---
## Pros and Cons

**Genuine strengths:**

1. **Structured JSON output** — the only reviewer with machine-parseable output. `location`, `trigger_condition`, `guard_snippet`, `potential_consequence` are all directly usable by your classifier without LLM parsing. The `guard_snippet` field is particularly valuable — it halves the work of the `patch` classification.
    
2. **Two-pass validation** — the reviewer audits its own completeness before reporting. Step 3 explicitly revisits every edge class from Step 2. This is the right quality gate for a method-driven review.
    
3. **Diff-scoped by default** — only reports paths reachable from changed lines. This directly prevents the "pre-existing issue not caused by this story" noise that pollutes the `defer` pile in step-04.
    
4. **Empty array is valid** — unlike the Blind Hunter, zero findings is a legitimate result. This is methodologically correct for path enumeration.
    
5. **15-word field limits** — `trigger_condition` and `potential_consequence` are capped at 15 words each. Forces precision, prevents rambling, makes deduplication across reviewers easier.
    
6. **`also_consider` injection** — same pattern as the Blind Hunter. For GIS360: "also_consider coordinate system boundary conditions, MVT tile size limits, pipe geometry validity at junction points."
    
7. **Edge classes derived from content** — no fixed checklist means it adapts to whatever code it sees. Spatial operations will trigger spatial edge classes. Async pipe operations will trigger race condition analysis. The examples given (null/empty inputs, off-by-one loops, arithmetic overflow, implicit type coercion, race conditions, timeout gaps) are illustrative, not exhaustive.
    

**Real weaknesses:**

1. **No codebase access for context** — it ignores external functions unless explicitly referenced. For GIS360, where a spatial validation call might be correct in isolation but wrong given what the calling function expects, this scope restriction could miss cross-function edge cases. The Edge Case Hunter gets "diff + codebase access" per quick-dev step-04, but its own SKILL.md says to ignore external functions by default. There's a tension here.
    
2. **`guard_snippet` quality is LLM-dependent** — "minimal code sketch that closes the gap" is good intent, but the sketch may be wrong, incomplete, or use incorrect APIs for the project. For GIS360's spatial operations, a guard snippet that doesn't account for coordinate system context could be worse than no snippet.
    
3. **No severity signal** — the JSON has `potential_consequence` but no explicit severity field. Your classifier has to infer severity from the consequence description. Adding a `severity` field (`critical/moderate/minor`) would significantly improve classifier reliability.
    
4. **Two-pass is still LLM-dependent** — Step 3 says "revisit every edge class from Step 2." But the LLM's memory of what it covered in Step 2 is imperfect within a single context. A truly exhaustive review would need the edge class list explicitly tracked in a structured format, not held in working memory.
    
5. **Diff-scoped restriction may miss integration edge cases** — a change to a pipe geometry parser that's correct in isolation may create edge cases at the point where it integrates with the MVT renderer. Since the renderer isn't in the diff, those boundary conditions are out of scope.
    

---
## The three-reviewer system: complete picture

Now we have all three:

|Dimension|Blind Hunter|Edge Case Hunter|Acceptance Auditor|
|---|---|---|---|
|Context|Diff only|Diff + codebase|Diff + spec + context docs|
|Method|Attitude-driven skepticism|Exhaustive path enumeration|Spec compliance checking|
|Output|Markdown list|Structured JSON|(inline in step-04)|
|Zero findings|HALT — suspicious|Valid — return `[]`|N/A|
|Floor|At least 10|None|None|
|Scope|Everything in diff|Changed paths only|Acceptance criteria only|
|Purpose|Find what others rationalize|Find what logic misses|Find what spec requires|

**The three are designed to be non-overlapping:**

- Blind Hunter finds what spec-aware reviewers rationalize away
- Edge Case Hunter finds paths that aren't wrong in spirit but are incomplete in execution
- Acceptance Auditor finds where implementation diverges from stated intent

**The deduplication step in quick-dev step-04 merges all three.** Findings that appear in multiple reviewers are collapsed. The classification taxonomy (`intent_gap / bad_spec / patch / defer / reject`) then processes the merged set.

---
## What to extract for your harness

|Edge Case Hunter mechanism|What to take|What to change|
|---|---|---|
|Structured JSON output (4 fields)|Take wholesale|Add `severity: "critical\|moderate\|minor"` fifth field|
|Two-pass validation (Step 2 + Step 3)|Take it|Make edge class list explicit and structured between passes|
|Diff-scoped by default|Take it|Add: when `blast_radius > threshold`, expand scope to include callers|
|`guard_snippet` field|Take it|Add validation: guard snippet must use actual project APIs|
|`also_consider` injection|Take it|Pre-populate with GIS360 spatial edge classes: coordinate bounds, projection validity, geometry null-check, tile size overflow, junction point integrity|
|Empty array as valid|Take it — apply to all your reviewers|None|
|"Derive edge classes from content" (not fixed checklist)|Take the principle|Build a GIS360 edge class seed list that supplements content-derived ones|
|15-word field limits|Take them — apply to all structured review outputs|None|

---
## The gap this reveals in your harness

Looking at both reviewer skills together, the adversarial system has a structural weakness for your use case: **neither reviewer knows about GIS360's domain**.

The Blind Hunter has `also_consider` but it's optional and caller-populated. The Edge Case Hunter derives edge classes from content. Neither has GIS360 spatial knowledge baked in.

For your harness, you need a **fourth reviewer** that doesn't exist in BMAD: a **Domain Reviewer** that knows GIS360-specific failure modes:

- Spatial precision: are coordinates being truncated incorrectly?
- Projection consistency: are coordinate systems being mixed?
- MVT tile validity: does the geometry survive tile encoding?
- Pipe geometry: does the change handle junction points and disconnected segments?
- Work correction logic: are correction chains being maintained correctly?

This fourth reviewer would receive: diff + GIS360 domain knowledge distillate (from your Milestone 3 product knowledge system). It's the missing piece that makes your adversarial review system GIS360-aware rather than generic.

---
