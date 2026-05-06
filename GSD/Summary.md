This is a large framework. Before diving into files, let me map what I can already see from structure alone — this will help us read the files with the right questions in mind.

**What the structure reveals:**

GSD has two primary phases — **discuss** and **execute** — but the workflow list shows it's actually much more granular than that. There are roughly 5 conceptual layers buried in these flat files:

1. **Planning layer** — `plan-phase.md`, `ultraplan-phase.md`, `spec-phase.md`, `sketch.md`, `spike.md`
2. **Execution layer** — `execute-phase.md`, `execute-plan.md`, `do.md`, `fast.md`, `quick.md`, `autonomous.md`
3. **Validation layer** — `verify-phase.md`, `validate-phase.md`, `audit-fix.md`, `audit-milestone.md`, `code-review.md`
4. **State management** — `pause-work.md`, `resume-project.md`, `transition.md`, `progress.md`, `session-report.md`
5. **Meta/lifecycle** — `new-project.md`, `new-milestone.md`, `complete-milestone.md`, `graduation.md`

This is rich. Let me extract what's actually happening here before we read execute-phase.

---

**What discuss-phase actually is:**

It's a **structured context capture engine** — not a chat interface. Its job is to produce a `CONTEXT.md` that downstream agents (researcher, planner, executor) can act on without asking the user again. Every design decision points at that output.

---

**Key patterns worth extracting for your harness:**

**1. Downstream-awareness as a design principle**
The phase knows exactly who reads its output and what they need. `CONTEXT.md` feeds `gsd-phase-researcher` and `gsd-planner`. This isn't incidental — it's the *purpose* of the phase. Your harness should adopt this: every phase/agent should know its consumer and optimize its output for that consumer, not for human readability.

**2. Progressive disclosure / lazy loading**
The workflow file itself stays under 500 lines by lazy-loading mode files only when needed. This is a token budget enforcement mechanism baked into architecture. For your harness, this maps to: don't load all product knowledge + all code memory + all dynamic memory on every task. Load only what the current phase needs.

**3. Prior context accumulation prevents re-asking**
It reads prior CONTEXT.md files, PROJECT.md, spike/sketch findings — and explicitly marks already-decided questions as pre-answered. Your harness equivalent: product knowledge from Milestone 3 should pre-answer implementation questions so the agent doesn't re-derive what's already known.

**4. Canonical refs as a mandatory output**
Every file, spec, ADR that's mentioned during discussion gets tracked with full paths into a `<canonical_refs>` section. This is how downstream agents know exactly what to read. This is essentially a **scoped reading list** per task — a concrete answer to the context routing problem your harness needs to solve.

**5. Checkpoint pattern for interrupted sessions**
After each area completes, writes a checkpoint JSON. If session breaks, resumes from last completed area. For your harness: Milestone 5's dynamic memory needs exactly this — stateful progress tracking that survives context resets.

**6. Scope creep as an explicit enforcement mechanism**
Has a hard rule: discussion clarifies HOW to implement what's scoped, never WHETHER to add new capabilities. Deferred ideas go into a holding section, never acted on. Your harness needs this — agents tend to gold-plate. A scope boundary enforced at the specification stage prevents it propagating into planning and execution.

**7. The "User = visionary, Claude = builder" split**
The agent is explicitly prohibited from asking about codebase patterns, technical risks, or implementation approach — those are downstream responsibilities. This is a **responsibility boundary** that prevents the discussion agent from doing the researcher's or planner's job. Clean separation you should replicate in your multi-agent design.

---

**What's weak or not worth carrying forward:**

- The `AskUserQuestion` interaction model assumes a human in the loop. Your harness replaces this with spec/PRD input — the discuss phase becomes the **specification engine** from your Milestone 2.
- The advisor mode, text mode, batch mode variants add complexity that makes sense for a general-purpose tool but adds noise for a purpose-built harness targeting GIS360 specifically.
- The `git_commit` steps are framework bookkeeping — useful for GSD as a product, not relevant to your harness internals.

---

**What to carry into your harness from this file:**

| GSD Pattern | Your Harness Equivalent |
|---|---|
| Downstream-aware output (CONTEXT.md) | Every agent produces typed output for its consumer |
| Lazy loading by phase | Context routing — load product/code/dynamic memory per phase |
| Prior context → skip re-asking | Product knowledge base answers questions before agent asks |
| Canonical refs (mandatory) | Scoped reading list per task injected at task start |
| Checkpoint after each area | Dynamic memory writes progress state incrementally |
| Scope creep enforcement | Spec boundary enforced at specification engine stage |
| Responsibility split | Agent role separation in your multi-agent orchestration |

---

Execute-phase is the heart of GSD. It's significantly more complex than discuss-phase — let me break down what's actually happening and what's worth extracting.

---

**What execute-phase actually is:**

A **wave-based parallel orchestrator** that stays deliberately lean — it never executes code itself. Its only job is: discover plans → group into waves → spawn executor subagents → handle failures → verify → advance. Every actual work decision is delegated.

---

**Key patterns worth extracting:**

**1. Orchestrator/Executor split — the most important pattern**
The orchestrator does zero implementation work. It owns: wave coordination, state writes (STATE.md, ROADMAP.md), worktree merges, verification spawning. Executors own: code changes, commits, SUMMARY.md. This separation is explicit and enforced — executors are *prohibited* from writing shared artifacts. Your harness should enforce the same boundary: the agent that writes code should not also be the agent that updates task state.

**2. Wave-based dependency management**
Plans within a wave are independent and can parallelize. Plans in wave N+1 depend on wave N. The intra-wave file overlap check (before spawning) catches incorrectly grouped plans and forces sequential execution. For your harness: task decomposition needs to produce a dependency graph, not a flat list. Tasks with overlapping files must serialize.

**3. Checkpoint heartbeats as stream health**
The `[checkpoint]` pattern at every wave/plan boundary solves a real infrastructure problem — stream idle timeouts on long-running operations. For GIS360 tasks that might run 10-20 minutes, your harness will hit this. The fix is cheap: emit a literal text line at every meaningful boundary. This belongs in your dynamic memory / state tracking design.

**4. The post-merge gate solving the generator self-evaluation blind spot**
This is called out explicitly: *"agents reliably report Self-Check: PASSED even when merging their work creates failures."* Each worktree passes its own tests. But after merging, cross-plan conflicts (type definitions, shared imports, API contracts) break things. The post-merge gate runs the full test suite after merging all worktrees. This is directly relevant to your Milestone 1 testing evaluation — self-reported test results from an agent cannot be trusted. You need an independent post-execution validation layer.

**5. Spot-check verification instead of trusting completion signals**
When an agent doesn't return a completion signal (which happens), GSD falls back to filesystem + git verification: does SUMMARY.md exist? Are there commits matching this plan ID? This is more reliable than relying on protocol signals. Your harness dynamic memory should track task completion via observable artifacts (files created, commits present), not just agent-reported status.

**6. Blocking anti-patterns gate**
Before any execution, GSD checks `.continue-here.md` for patterns marked `blocking` — and forces the orchestrator to answer three questions about each one before proceeding. This is a **learned failure enforcement mechanism**. When something goes wrong badly enough to get recorded, it becomes a hard gate on the next run. For your harness: the dynamic memory system from Milestone 5 should feed back into pre-execution checks, not just inform agent behavior during execution.

**7. The three-question anti-pattern proof**
To pass the blocking gate, the orchestrator must answer: what is this pattern, how did it manifest, what *structural mechanism* (not acknowledgment) prevents it. The third question is the key one — it forces a concrete fix, not just awareness. This framing is worth using in your harness's self-correction design: when a failure is recorded, the system must record not just what failed but what mechanism prevents recurrence.

**8. Schema drift gate catching false-positive verification**
TypeScript types compile from config, not the live database. Tests pass. Verification passes. But the database is out of sync. GSD blocks verification if schema-relevant files changed but no push command was run. This is a domain-specific false positive pattern. GIS360 equivalent: MVT tile cache out of sync, pipe geometry indices stale, spatial reference system mismatches. Your harness needs similar domain-specific validation gates, not just generic test pass/fail.

**9. Worktree isolation with sequential dispatch**
Multiple agents parallelize — but `git worktree add` acquires an exclusive lock, so dispatch is sequential (one Task() call per message) while execution is parallel. The distinction matters: spawning is sequential, running is parallel. For your harness: if you ever parallelize GIS360 agents, the lock contention issue applies to any filesystem or database resource that has exclusive access semantics.

**10. Resumption via observable state**
Re-running execute-phase discovers completed SUMMARY.md files and skips those plans automatically. No explicit checkpoint management needed — the artifacts themselves are the state. Your harness should follow this: task completion is recorded in persistent artifacts, not in-memory state, so any crashed session can resume cleanly.

---

**What's NOT worth carrying forward:**

- The worktree merge machinery (5.5) — complex git plumbing that's only needed for true parallel execution. Your harness should start sequential.
- Cross-AI delegation (the `--cross-ai` step) — GSD-specific feature for routing to external AI runtimes.
- The `classifyHandoffIfNeeded` bug handling — Claude Code runtime-specific workaround.
- The decimal phase / gap-closure artifact closing — GSD's phase numbering system, not relevant to your architecture.

---

**Combined picture from both files — what GSD is actually doing:**

| Stage | File | Core mechanism |
|---|---|---|
| Capture intent | discuss-phase | CONTEXT.md with canonical refs + downstream-aware decisions |
| Execute work | execute-phase | Wave orchestration + subagent delegation + observable state |
| Validate output | execute-phase | Post-merge gate + independent verifier agent + domain-specific gates |
| Handle failures | execute-phase | Blocking anti-patterns → structural fix enforcement |
| Advance state | execute-phase | Single writer (orchestrator) for shared artifacts |

---

**What to carry into your harness from both files combined:**

| Pattern | Harness application |
|---|---|
| Downstream-aware outputs | Each agent produces typed artifacts for its consumer |
| Canonical refs per task | Scoped reading list injected at task start (your context router) |
| Wave dependency graph | Task decomposer produces dependency-ordered batches, not flat lists |
| Post-merge independent validation | Test results from the executing agent are untrusted; run independent validation after |
| Observable completion state | Task completion = artifact exists + commit present, not agent-reported |
| Blocking anti-patterns gate | Failure memory from Milestone 5 feeds pre-execution checks |
| Three-question failure proof | Self-correction records structural fix mechanism, not just acknowledgment |
| Single writer for shared state | One agent owns STATE.md / task tracking; executors never touch it |
| Domain-specific validation gates | GIS360-specific checks (spatial indices, MVT cache, pipe geometry) alongside generic tests |

---

**What plan-phase actually is:**

A **multi-agent orchestration pipeline** that transforms captured intent (CONTEXT.md) into executable work artifacts (PLAN.md files). It chains researcher → pattern-mapper → planner → plan-checker through a revision loop, with multiple domain-specific gates before execution can start.

**What autonomous actually is:**

The **top-level harness** — it's the closest thing in GSD to what you're building. It drives the entire discuss → plan → execute cycle for every phase in a milestone without human intervention, pausing only at explicit decision points.

---

**New patterns worth extracting:**

**1. The task quality contract (the most important thing in plan-phase)**

The `<deep_work_rules>` section in the planner prompt is the single most important pattern here. Every task must have three mandatory fields:

- `<read_first>` — files the executor *must* read before touching anything, including the file being modified
- `<acceptance_criteria>` — grep-verifiable conditions, no subjective language
- `<action>` — concrete values, never "align X with Y" without specifying the exact target state

The rationale given is direct: *"Vague instructions like 'update the config to match production' produce shallow one-line changes. Concrete instructions produce complete work."* For your harness, every task handed to an executor agent needs this contract. Vague tasks are the primary source of shallow, incomplete, or wrong implementation.

**2. The coverage gates chain — blocking vs. non-blocking**

Plan-phase runs four sequential coverage checks before allowing execution:

- Requirements coverage gate (§13) — are all REQ-IDs assigned to plans?
- Decision coverage gate (§13a) — are all CONTEXT.md decisions referenced in plans?
- Post-planning gap analysis (§13e) — non-blocking unified report

The distinction matters: requirements and decision gaps *block* execution. The post-planning gap analysis is *advisory*. Your harness needs this same tiering — some gaps should halt, others should surface and continue. The reasoning given is: *"failing here is cheap. Catching a dropped decision before execution beats discovering it after thousands of dollars of execution."*

**3. Pattern mapper as a pre-planning step**

The `gsd-pattern-mapper` agent runs *before* the planner and produces PATTERNS.md — a file that maps each file to be created/modified to the closest existing analog in the codebase, with actual code excerpts. The planner then reads this before writing plans. For your harness: before code generation, the code memory system (Milestone 4) should produce a similar scoped reading list — not just "these are relevant files" but "here is the existing pattern this new code should replicate."

**4. Source audit — the planner checks its own coverage**

The planner returns `## ⚠ Source Audit: Unplanned Items Found` when items from REQUIREMENTS.md, RESEARCH.md, ROADMAP goal, or CONTEXT.md have no corresponding plan. This is a self-audit built into the planner's output contract. Your harness equivalent: after the specification engine produces tasks, it should verify that every acceptance criterion from the spec maps to at least one task before handing off to execution.

**5. Chunked planning for crash resilience**

Chunked mode splits planning into an outline task followed by N single-plan tasks, each committed individually. If the session crashes, the next run resumes from the last committed plan by checking for existing PLAN.md files with valid frontmatter. This is a direct answer to the context reset problem your Milestone 5 dynamic memory needs to solve — intermediate artifacts committed to disk are more reliable than in-memory state.

**6. The autonomous pipeline's "pause only at explicit decisions" principle**

The autonomous workflow has one controlling principle: *pause only for explicit user decisions (grey area acceptance, blockers, validation requests)*. Everything else auto-advances. This is the right design for your harness — the agent should not pause at uncertainty, it should pause at *decision points*. Uncertainty is handled by the specification engine before execution starts.

**7. Interactive mode pipeline parallelism**

When `--interactive` is set, discuss runs inline (user answers questions) while plan+execute run as background agents. The main context only accumulates discuss conversations. This is a context budget management strategy — lightweight interactive work stays in the main context, heavy generation work is isolated in subagents. Your harness can apply the same pattern: product knowledge retrieval and context assembly are cheap operations that can share context with the orchestrator; code generation gets its own isolated context.

**8. The single gap closure retry limit**

When verification finds gaps, the autonomous workflow runs one gap closure attempt then stops rather than looping. This is the circuit breaker from your risk register made concrete — an explicit limit of one retry before escalating to the human. Your Milestone 5 self-correction needs the same: a bounded retry count before escalation, not an open-ended loop.

**9. Stall detection in revision loops**

The revision loop tracks `prev_issue_count` and detects when the issue count stops decreasing across iterations. When stalled, it asks the user rather than continuing to burn tokens on non-converging revisions. For your harness: the dynamic memory error pattern system should detect when self-correction is not making progress and escalate rather than retry.

---

**What the autonomous file reveals about the full system:**

This is the first file that shows the complete pipeline from the outside. The flow is:

```
discover phases
  → for each phase:
      smart discuss (batch table proposals, user accepts/overrides)
      → [if frontend] generate UI-SPEC
      → plan (research → pattern map → plan → verify)
      → execute (waves → post-merge gate → verify)
      → [if frontend] UI review audit
      → re-read ROADMAP (catch inserted phases)
  → lifecycle (audit → complete → cleanup)
```

The key observation: every phase re-reads ROADMAP after completion to catch dynamically inserted phases (decimal phases like 5.1 created during gap closure). Your harness needs the same — the task list is not static, it can grow during execution.

---

**Complete extraction table — across all four files:**

| GSD Pattern | Source | Harness Application |
|---|---|---|
| Downstream-aware outputs | discuss-phase | Every agent produces typed artifacts for its consumer |
| Canonical refs (mandatory) | discuss-phase | Scoped reading list per task |
| Lazy context loading | discuss-phase | Load product/code/dynamic memory per phase only |
| Prior context → skip re-asking | discuss-phase | Product knowledge base pre-answers known questions |
| Checkpoint after each area | discuss-phase | Dynamic memory writes progress state incrementally |
| Scope creep enforcement | discuss-phase | Spec boundary enforced at specification engine stage |
| Wave dependency graph | execute-phase | Task decomposer produces dependency-ordered batches |
| Post-merge independent validation | execute-phase | Untrusted self-reported test results; independent validation required |
| Observable completion state | execute-phase | Task completion = artifact present + commit visible |
| Blocking anti-patterns gate | execute-phase | Failure memory feeds pre-execution checks |
| Three-question failure proof | execute-phase | Self-correction records structural fix mechanism |
| Single writer for shared state | execute-phase | One agent owns task tracking; executors never touch it |
| Task quality contract | plan-phase | Every task needs read_first + verifiable criteria + concrete action |
| Coverage gates (blocking) | plan-phase | Spec-to-task translation is verified before execution |
| Pattern mapper pre-planning | plan-phase | Code memory produces analog file map before code generation |
| Source audit in planner output | plan-phase | Specification engine verifies all criteria map to tasks |
| Chunked planning for resilience | plan-phase | Intermediate artifacts committed; sessions resumable |
| Stall detection in revision loop | plan-phase | Self-correction escalates when not converging |
| Pause only at explicit decisions | autonomous | Harness halts at decision points, not uncertainty |
| Single gap closure retry | autonomous | Bounded retry count before human escalation |
| Dynamic phase insertion | autonomous | Task list can grow during execution; re-read after each phase |
| Pipeline parallelism by weight | autonomous | Heavy generation isolated in subagents; orchestrator stays lean |

---

**What you now have enough to design:**

The four files give you a complete view of GSD's orchestration model. You can now design your harness with specific answers to the questions from your Milestone 6:

- **Task decomposition**: spec → coverage-verified task list with dependency graph
- **Orchestration model**: single orchestrator + role-specialized subagents (researcher, planner, executor, verifier), each with fresh context
- **Specification workflow**: discuss → research → pattern map → plan → coverage gates → execute
- **Human-in-the-loop**: pause at explicit decision points only (grey areas, blockers, validation)
- **Error budget**: single retry with stall detection → escalate to human
- **Context routing**: canonical refs per task, lazy-loaded by phase

---

**Agent taxonomy from the names alone:**

**Core pipeline agents (must read):**
- `gsd-executor.md` — writes code, the most critical contract to understand
- `gsd-verifier.md` — verifies phase goal achievement
- `gsd-planner.md` — creates PLAN.md files
- `gsd-plan-checker.md` — reviews plan quality

**Research agents (read one, understand the pattern):**
- `gsd-phase-researcher.md` — technical research before planning
- `gsd-pattern-mapper.md` — codebase analog mapping
- `gsd-codebase-mapper.md` — structural codebase analysis

**Specialist agents (read selectively):**
- `gsd-debugger.md` — failure diagnosis
- `gsd-code-reviewer.md` — code quality review
- `gsd-integration-checker.md` — cross-phase integration

**Advisory/meta agents (lower priority for harness design):**
- `gsd-advisor-researcher.md`, `gsd-user-profiler.md`, `gsd-domain-researcher.md`
- `gsd-security-auditor.md`, `gsd-nyquist-auditor.md`, `gsd-eval-auditor.md`
- Doc agents (`gsd-doc-*`), UI agents (`gsd-ui-*`)

---

1. `gsd-executor.md` — this is the contract between the orchestrator and code generation; everything else depends on what the executor can and cannot do
2. `gsd-verifier.md` — defines what "done" means, which shapes what the executor must produce
3. `gsd-plan-checker.md` — the quality gate between planning and execution
4. `gsd-debugger.md` — failure diagnosis patterns feed directly into your Milestone 5 self-correction design

---

## gsd-executor — What's actually here

**1. The deviation rule system — the self-correction model you need**

This is GSD's answer to your Milestone 5. Four rules, clearly tiered:

| Rule | Trigger | Action | Permission needed |
|---|---|---|---|
| Rule 1 | Code doesn't work as intended | Auto-fix bugs | None |
| Rule 2 | Missing critical functionality (security, correctness) | Auto-add | None |
| Rule 3 | Something blocks task completion | Auto-fix blocker | None |
| Rule 4 | Fix requires architectural change | STOP, checkpoint | User required |

The key design insight: **rules 1-3 execute without asking**. The agent doesn't pause to report every deviation — it fixes and documents. Only Rule 4 (architectural scope change) requires human input. For your harness, this is the self-correction boundary: fix anything that's a correctness or completeness problem, escalate only when the solution changes the system's structure.

The scope boundary is equally important: *"Only auto-fix issues directly caused by the current task's changes. Pre-existing issues go to deferred-items.md."* Without this, agents expand scope indefinitely.

**2. The fix attempt limit — your circuit breaker made concrete**

After 3 auto-fix attempts on a single task: stop fixing, document remaining issues in SUMMARY as "Deferred Issues", continue to the next task. This is the circuit breaker from your risk register implemented at the task level. Three attempts, then move on. Your harness dynamic memory needs this same counter per task.

**3. The analysis paralysis guard**

*"If you make 5+ consecutive Read/Grep/Glob calls without any Edit/Write/Bash action: STOP. State in one sentence why you haven't written anything yet."* This is a stuck detection mechanism built directly into the agent's behavior. Your harness can implement this as a tool-call pattern monitor — if an agent is only reading without writing past a threshold, it's stuck and needs intervention.

**4. Per-task atomic commits with exact staging**

Never `git add .` — stage task-related files individually. This is not just good practice; it's what makes the orchestrator's post-merge file conflict detection reliable. If executors stage everything, merge conflicts become undiagnosable. For your harness: each atomic unit of work must have a precise file set, and the commit must include only those files.

**5. Self-check as a mandatory final step**

After writing SUMMARY.md, the executor verifies its own claims: do the files it claims to have created actually exist? Do the commits it claims to have made appear in git log? This produces `## Self-Check: PASSED` or `## Self-Check: FAILED`. It's not trusted — the verifier still runs independently — but it catches obvious failures before they reach verification. Your harness should require this from every executor agent.

**6. Stub tracking before SUMMARY is written**

Before writing SUMMARY, the executor scans all created/modified files for stub patterns: hardcoded empty values, placeholder text, components with no data source wired. If stubs exist, they go into a `## Known Stubs` section. This is the executor self-reporting what it knows is incomplete — feeding directly into the verifier's work. Your harness: executor agents should declare known gaps, not hide them.

**7. Authentication gates as a distinct category**

Auth errors during execution are not bugs — they're gates. The executor recognizes them and returns a `checkpoint:human-action` rather than treating them as failures or trying to auto-fix. For your GIS360 harness: database connection errors, spatial service unavailability, API key requirements are all gates, not failures.

---

## gsd-verifier — What's actually here

**1. The adversarial stance — the most important design decision**

*"Assume the phase goal was not achieved until codebase evidence proves it. Your starting hypothesis: tasks completed, goal missed. Falsify the SUMMARY.md narrative."*

This is the design that makes verification meaningful. The verifier is explicitly not trying to confirm what the executor said — it's trying to disprove it. For your harness: the validation agent must be structurally adversarial. If it reads from the executor's output to decide what to verify, it's not independent.

**2. Four-level artifact verification — the hierarchy of evidence**

| Level | Check | What it catches |
|---|---|---|
| 1 | Exists | Missing files |
| 2 | Substantive | Stub files (3 lines, placeholder content) |
| 3 | Wired | Orphaned files (exist but nothing imports/uses them) |
| 4 | Data flows | Hollow wiring (imported and used, but data source returns empty) |

Level 4 is the critical one that most verification systems miss. A component can be wired to an API route that returns `[]`. The component renders, the tests pass, the build succeeds — but the feature doesn't work. Level 4 traces the data source and checks whether it produces real data or static/empty returns. For your GIS360 harness: spatial queries that return empty geometry sets, MVT tiles that serve empty layers, pipe tracking queries that return no features — all Level 4 failures.

**3. Task completion ≠ goal achievement — stated explicitly**

*"A task 'create chat component' can be marked complete when the component is a placeholder. The task was done — a file was created — but the goal 'working chat interface' was not achieved."*

This is why verification must be goal-backward, not task-forward. The verifier starts from what the phase *should deliver* and works backward to what must exist, what must be wired, what must flow — not from what tasks were marked done.

**4. The deferred items filter — prevents false-positive gaps**

Before reporting a gap, the verifier checks if it's explicitly addressed in a later milestone phase. If Phase 3 intentionally doesn't implement feature X because Phase 5 does, Phase 3's verification should not report it as a gap. This requires the verifier to read the full roadmap, not just the current phase. For your harness: the product knowledge system (Milestone 3) needs to include the full feature roadmap so verification can distinguish "not done yet" from "not done and should be."

**5. Override mechanism for intentional deviations**

When a must-have fails but there's evidence of an intentional alternative implementation, the verifier can suggest an override rather than blocking. The override is recorded in VERIFICATION.md frontmatter and carried forward. This prevents the same gap being re-surfaced on every verification run when the deviation is known and accepted. Your harness needs this: a way to record "this is intentionally different from the spec, and it's been accepted."

**6. Behavioral spot-checks separate from anti-pattern scanning**

Anti-pattern scanning finds code smells. Behavioral spot-checks actually invoke the code and check output. These are two distinct verification layers. For GIS360: grep-based checks can verify that a spatial query exists; a behavioral spot-check can verify that `curl /api/pipes` returns non-empty GeoJSON. The constraint — each check under 10 seconds, no server startup, no state mutation — makes this practical for automated execution.

**7. Requirements traceability as a verification dimension**

The verifier checks that every requirement ID declared in PLAN frontmatter has evidence of fulfillment. Orphaned requirements — ones mapped to the phase in REQUIREMENTS.md but not claimed by any plan — are flagged as verification failures. This closes the loop from specification through planning through execution through validation.

---

## Combined picture — what executor + verifier together reveal

The relationship between these two agents is the core of GSD's quality model:

```
Executor                    Verifier
─────────                   ────────
Executes tasks              Ignores task completion
Writes SUMMARY.md           Distrusts SUMMARY.md
Self-checks own claims      Independently verifies claims
Documents stubs (known)     Detects stubs (adversarial)
Fixes deviations (R1-3)     Reports unresolved gaps
Escalates architecture (R4) Classifies gaps as blocker/warning
```

The separation works because neither agent reads the other's reports to determine what to do. The executor produces artifacts; the verifier independently checks those artifacts against the phase goal.

---

## What to carry into your harness

| Pattern | Source | Harness application |
|---|---|---|
| Four-tier deviation rules | executor | Self-correction: fix/add/unblock auto; architectural changes escalate |
| Fix attempt limit (3) | executor | Circuit breaker at task level |
| Analysis paralysis guard | executor | Tool-call pattern monitor: reading without writing = stuck |
| Atomic commits with precise file staging | executor | Each task declares its file set; only those files committed |
| Executor self-check | executor | Agents verify their own claims before reporting complete |
| Stub declaration in SUMMARY | executor | Executors self-report known gaps |
| Auth/service gates as distinct category | executor | GIS360-specific: DB, spatial services, API keys are gates not failures |
| Adversarial verifier stance | verifier | Validation agent structurally independent from executor |
| Four-level artifact verification | verifier | Exists → substantive → wired → data flows |
| Goal-backward verification | verifier | Start from what phase should deliver, work backward |
| Deferred items filter | verifier | Product knowledge roadmap needed for verification to distinguish "not yet" from "missing" |
| Override mechanism | verifier | Accepted deviations recorded and carried forward |
| Behavioral spot-checks | verifier | GIS360: spatial query outputs, MVT tile content, pipe data returns |
| Requirements traceability | verifier | Every REQ-ID must have evidence; orphaned requirements are failures |

---

## gsd-plan-checker — What's actually here

**1. Pre-execution adversarial stance mirrors the verifier**

Same principle applied earlier in the pipeline: *"Assume every plan set is flawed until evidence proves otherwise."* The plan-checker is the verifier's pre-execution equivalent — same goal-backward methodology, different subject matter (plans vs. code). For your harness: adversarial checking should happen twice — once before execution (plans will achieve the goal?) and once after (code did achieve the goal?).

**2. Dimension 7b: Scope reduction detection — the most insidious failure mode**

This dimension exists because of a real recurring failure: planners reference a user decision (D-26) but silently deliver a reduced version of it. The keywords that flag this are precise and worth keeping verbatim:

*"v1", "v2", "simplified", "static for now", "hardcoded", "future enhancement", "placeholder", "basic version", "will be wired later", "dynamic in future", "skip for now", "not wired to", "stub"*

Severity is always BLOCKER, never WARNING. The reasoning: *"Scope reduction is never a warning — it means the user's decision will not be delivered."* For your harness: when the specification engine produces tasks, scan action descriptions for these patterns before handing to execution. A task that contains "static for now" is a planning failure, not an execution input.

**3. Dimension 7c: Architectural tier compliance**

The checker verifies that each task assigns capabilities to the correct tier as defined in the Architectural Responsibility Map from RESEARCH.md. Auth validation belongs in API tier, not client tier. Security-sensitive capabilities in the wrong tier are always BLOCKER; non-security tier mismatches are WARNING. For your GIS360 harness: spatial operations, MVT tile generation, pipe geometry calculations — all have defined tier ownership. A task that puts geometry validation in the frontend is a planning failure.

**4. Dimension 9: Cross-plan data contracts**

When two plans share a data pipeline, one plan's transformation can invalidate another's assumptions. Plan A strips/sanitizes data; Plan B needs the original. This creates a silent correctness failure that neither executor will catch. For your harness: task decomposition must produce a data contract check — when two tasks touch the same data entity, verify their transformations are compatible before execution.

**5. Dimension 11: Research questions must be resolved before planning**

If RESEARCH.md has an `## Open Questions` section without a `(RESOLVED)` suffix, planning is blocked. Unresolved research questions flowing into planning produce underspecified tasks. For your harness: the specification engine must verify that every open question from the research phase has been answered before generating tasks.

**6. Dimension 12: Pattern compliance**

Plans must reference the correct analog files from PATTERNS.md for each file to be created or modified. If PATTERNS.md maps `src/controllers/auth.ts` to analog `src/controllers/users.ts`, the plan's action must reference that analog. This ensures executors replicate existing patterns rather than inventing new ones. For your harness: the code memory system (Milestone 4) doesn't just surface relevant files — it must be referenced in task actions, not just loaded into context.

---

## gsd-debugger — What's actually here

**1. The debug file as persistent investigation brain**

The debug file is structured state that survives context resets. It has hard rules about mutation:

| Section | Rule |
|---|---|
| Current Focus | OVERWRITE before every action |
| Symptoms | IMMUTABLE after gathering |
| Eliminated | APPEND only |
| Evidence | APPEND only |
| Resolution | OVERWRITE as understanding evolves |

The key design: *"Update the file BEFORE taking action, not after. If context resets mid-action, the file shows what was about to happen."* For your harness dynamic memory (Milestone 5): this is the correct write ordering. State is written before action, not after. An interrupted session always knows what was in progress.

**2. The knowledge base — pattern matching across sessions**

Every resolved debug session appends to `.planning/debug/knowledge-base.md`. At the start of every new investigation, the debugger reads the knowledge base, extracts keywords from the current symptoms, and checks for 2+ word overlap with past entries. If a match is found, that past root cause becomes the first hypothesis to test.

The design constraint is important: *"A match is a hypothesis candidate, not a confirmed diagnosis."* The knowledge base surfaces possibilities, it doesn't short-circuit investigation. For your harness Milestone 5: cross-session learning works the same way — past failures inform hypothesis ordering, they don't replace investigation.

**3. The structured reasoning checkpoint — mandatory before any fix**

Before touching code, the debugger must write this block:

```yaml
reasoning_checkpoint:
  hypothesis: "[exact statement — X causes Y because Z]"
  confirming_evidence: [specific items]
  falsification_test: "[what would prove this wrong]"
  fix_rationale: "[why fix addresses root cause, not symptom]"
  blind_spots: "[what hasn't been tested that could invalidate this]"
```

If any field is vague or empty — the root cause is not confirmed and investigation must continue. *"The cost of insufficient verification: bug returns, user frustration, emergency debugging, rollbacks."* For your harness self-correction: before an agent applies a fix, it must be able to state the root cause, the evidence, and the mechanism. A fix that can't be explained shouldn't be applied.

**4. The analysis paralysis guard applied to debugging**

The 5-consecutive-reads-without-writing guard from the executor applies here too. In the investigation loop: if the agent is only reading without forming hypotheses or running tests, it's stuck. For your harness: stuck detection is a universal pattern — apply it to every agent type, not just executors.

**5. The `follow the indirection` technique**

This technique is worth extracting verbatim for GIS360: *"Never assume a constructed path is correct. Resolve it to its actual value and verify the other side agrees. When two systems share a resource, trace the full path in both."*

For GIS360 specifically: spatial layer configurations, MVT tile paths, pipe geometry cache keys, work correction reference IDs — all constructed from variables. When a spatial feature disappears from the map, the first investigation step is: does the path the writer uses match the path the reader uses?

**6. The `go/no-go` decision criteria for fixes**

Act only when you can answer YES to all four:
1. Understand the mechanism (not just what fails, but why)
2. Can reproduce reliably
3. Have evidence, not just theory
4. Ruled out alternatives

*"Don't act if: 'I think it might be X' or 'Let me try changing Y and see'"* — this is the exact failure mode your risk register calls "agent makes incorrect assumptions." The debugger operationalizes the check.

**7. One hypothesis at a time — strict discipline**

*"If you change three things and it works, you don't know which one fixed it."* The debugger enforces single-hypothesis testing. Each test runs against one hypothesis; elimination is recorded with evidence. For your dynamic memory: when recording a fix, record which specific hypothesis was confirmed, not just "tried X, worked."

---

## Combined picture — what plan-checker + debugger add

These two files complete the quality loop. The full pipeline now is:

```
discuss-phase     → captures intent, canonical refs
plan-phase        → research → pattern map → plan → plan-checker (pre-execution quality gate)
execute-phase     → wave execution → post-merge gate
verifier          → goal-backward adversarial check (post-execution quality gate)
debugger          → investigation when verifier finds gaps, persistent across sessions
```

The plan-checker and verifier are symmetric: same methodology (goal-backward, adversarial), same dimensions (coverage, artifacts, wiring), different timing (before vs. after execution).

The debugger is what happens when the verifier says `gaps_found` and the gap closure planner can't determine root cause. It's the fallback investigation layer.

---

## Complete harness extraction — all six files combined

| Pattern | Source | Harness component |
|---|---|---|
| Downstream-aware outputs | discuss | Specification engine produces typed artifacts |
| Canonical refs per task | discuss | Context router: scoped reading list per task |
| Lazy context loading | discuss | Load product/code/dynamic memory by phase |
| Prior context prevents re-asking | discuss | Product KB pre-answers known questions |
| Checkpoint after each area | discuss | Dynamic memory writes state incrementally |
| Wave dependency graph | execute | Task decomposer produces dependency-ordered batches |
| Post-merge independent validation | execute | Trusted test runner independent from executor |
| Observable completion state | execute | Task completion = artifact + commit, not agent report |
| Blocking anti-patterns gate | execute | Failure memory feeds pre-execution checks |
| Single writer for shared state | execute | Orchestrator owns state; executors never touch it |
| Task quality contract | plan | Every task: read_first + verifiable criteria + concrete action |
| Coverage gates (blocking) | plan | Spec-to-task translation verified before execution |
| Pattern mapper pre-planning | plan | Code memory produces analog map before generation |
| Scope reduction detection | plan-checker | Scan task actions for placeholder/stub language before execution |
| Architectural tier compliance | plan-checker | GIS360 tier map enforced at planning stage |
| Cross-plan data contracts | plan-checker | Shared data entities checked for transform compatibility |
| Open questions gate | plan-checker | Research questions resolved before planning proceeds |
| Four-level artifact verification | verifier | Exists → substantive → wired → data flows |
| Goal-backward adversarial check | verifier | Start from goal, work backward to evidence |
| Deferred items filter | verifier | Roadmap needed to distinguish "not yet" from "missing" |
| Deviation rules 1-3 (auto-fix) | executor | Self-correction: fix bugs/missing/blockers without asking |
| Rule 4 (escalate architecture) | executor | Structural changes escalate to human |
| Fix attempt limit (3) | executor | Circuit breaker at task level |
| Analysis paralysis guard | executor | Reading without writing = stuck signal |
| Stub declaration in SUMMARY | executor | Executors self-report known gaps |
| Debug file as persistent brain | debugger | Dynamic memory: write-before-action, append evidence |
| Knowledge base cross-session | debugger | Past failures inform hypothesis ordering |
| Structured reasoning checkpoint | debugger | Fix requires stated: hypothesis + evidence + mechanism |
| One hypothesis at a time | debugger | Self-correction records which specific hypothesis was confirmed |
| Follow the indirection | debugger | GIS360: verify path writer and reader agree |

---

## gsd-phase-researcher — What's actually here

**1. Claim provenance — the most important design principle in this file**

Every factual claim must be tagged with its source:
- `[VERIFIED: npm registry]` — confirmed via tool
- `[CITED: docs.example.com]` — from official documentation
- `[ASSUMED]` — training knowledge, not verified this session

The `[ASSUMED]` tag is the key mechanism. It signals to downstream agents that a claim needs user confirmation before becoming a locked decision. *"Never present assumed knowledge as verified fact — especially for compliance requirements, retention policies, security standards, or performance targets."* For your harness: research outputs need provenance tagging. When the product knowledge base or code memory stores a claim, it should store its source and verification status. A stale `[ASSUMED]` claim from Milestone 3 shouldn't be treated the same as a `[VERIFIED]` claim from an active codebase scan.

**2. Training data is a hypothesis, not a fact**

*"Treat pre-existing knowledge as hypothesis, not fact. Verify before asserting."* The researcher is explicitly designed to distrust its own training data. Every library version claim must be verified against the npm registry. Every capability claim must be verified against current docs. For your harness: the code memory system (Milestone 4) should distinguish between facts derived from the actual codebase (verified) and facts from training (assumed). When the agent says "GIS360 uses library X for spatial operations," that should trace to a grep result, not training knowledge.

**3. Architectural Responsibility Map as a mandatory research output**

Before any framework research, the researcher maps each capability to its tier owner. This map is then consumed by the plan-checker (Dimension 7c) to verify tier compliance in tasks. The flow is explicit: researcher produces the map → plan-checker enforces it. For your harness: tier ownership needs to be determined at research time and enforced at planning time. For GIS360 specifically: MVT tile generation belongs in the tile server tier, spatial queries belong in the database tier, geometry validation belongs in the API tier. Tasks that put these in the wrong tier should fail the plan-checker.

**4. Runtime State Inventory for rename/refactor phases**

For any phase involving renaming or migration, the researcher must explicitly answer five questions about runtime state that grep can't find: stored data (database keys, collection names), live service config (n8n workflows in SQLite, not in git), OS-registered state (Windows Task Scheduler, pm2), secrets, build artifacts. *"After every file in the repo is updated, what runtime systems still have the old string cached, stored, or registered?"* For GIS360: spatial layer names, MVT tile cache keys, work correction reference IDs, pipe tracking database indices — these exist at runtime, not just in source files. Rename phases that don't account for runtime state produce partially-renamed systems.

**5. Environment availability audit**

Before planning, the researcher checks whether required external tools actually exist: databases running, CLI tools installed, services accessible. Missing dependencies with no fallback are flagged as blockers. Missing dependencies with fallbacks become plan instructions. For your harness: a task that assumes PostgreSQL spatial extensions are installed will fail silently if they're not. The environment audit makes this a pre-execution check, not a runtime discovery.

**6. "Be prescriptive, not exploratory"**

The researcher is explicitly told: *"'Use X' not 'Consider X or Y.'"* Research output feeds directly into planning — if the researcher says "consider X or Y," the planner must choose, which creates an unnecessary decision point. For your harness: when the specification engine produces research, it should make recommendations, not present options. Options belong in the discuss phase (user decision); recommendations belong in the research phase (technical judgment).

---

## gsd-pattern-mapper — What's actually here

**1. The read-once constraint**

*"Never re-read a range already in context. Small files: one Read call, extract everything."* This is a token budget enforcement rule baked into the agent's behavior. The constraint exists because re-reading wastes context on content that's already present. For your harness: the code memory system should track what has already been loaded into context per task. If a file is already in context, don't fetch it again.

**2. Early stopping at 3-5 strong analogs**

*"Stop analog search once you have 3-5 strong matches. There is no benefit to finding a 10th analog."* This is a diminishing-returns rule. Broader search past a threshold produces redundant information at increasing token cost. For your harness: retrieval systems should have a confidence threshold — once you have enough high-quality matches, stop searching. More results don't improve execution quality.

**3. Role + data flow classification as the matching dimensions**

The mapper classifies each file on two axes: role (controller, component, service, model, middleware, utility) and data flow (CRUD, streaming, file-I/O, event-driven, request-response). Analog matching is best-first on both dimensions simultaneously. For your harness code memory: indexing files by role and data flow produces better retrieval than pure semantic similarity. A new service file needs an analog service file, not the most semantically similar file regardless of type.

**4. Concrete excerpts with line numbers, not abstract descriptions**

The mapper extracts four specific pattern categories from each analog: imports, auth/guard, core pattern, error handling. Each extract includes file path and line numbers. The planner copies these excerpts directly into task action descriptions. For your harness: code memory retrieval should return usable code excerpts, not just file names. "This file is relevant" is less useful than "use this import pattern from lines 12-25."

**5. Shared patterns as a distinct output**

Cross-cutting concerns (auth middleware, error handling wrappers, logging, response formatting) are extracted once and listed separately. Every new file in a given category inherits these patterns. For GIS360: spatial reference system handling, MVT tile error responses, pipe geometry null checks — these are cross-cutting and should be extracted once, not re-discovered for each new file.

**6. "No analog found" as an explicit output**

Files with no close match in the codebase are listed explicitly. The planner falls back to RESEARCH.md patterns for these. This is an honest output — admitting that the codebase has no precedent for a given pattern is more useful than forcing a weak analog match. For your harness: when code memory can't find a relevant analog, it should say so explicitly rather than returning a weak match that misleads the executor.

---

## What these two files complete in the pipeline

The researcher and pattern-mapper together form the **context assembly layer** — the stage that transforms the phase goal into actionable context for the planner. The complete context assembly chain is:

```
discuss-phase (CONTEXT.md)
    ↓
phase-researcher (RESEARCH.md)
    - claim provenance tagging
    - architectural responsibility map
    - environment availability audit
    - runtime state inventory
    ↓
pattern-mapper (PATTERNS.md)
    - file classification (role + data flow)
    - analog matching (3-5 best matches)
    - concrete code excerpts
    - shared patterns
    ↓
planner (PLAN.md)
    - reads all three
    - produces tasks with read_first + concrete actions
```

---

## Final synthesis — what the complete agent set reveals about your harness design

Reading all seven files together, the architecture of GSD resolves into five distinct concerns, each handled by a dedicated agent or mechanism:

**1. Intent capture** — discuss-phase + CONTEXT.md
Captures what the user wants with canonical refs, prior context, and scope boundaries.

**2. Knowledge assembly** — researcher + pattern-mapper + RESEARCH.md + PATTERNS.md
Transforms intent into technical context: verified claims, architectural tier map, environment state, concrete code patterns.

**3. Work specification** — planner + plan-checker + PLAN.md
Transforms context into executable tasks with quality gates at every boundary: coverage, completeness, scope, tier compliance, data contracts.

**4. Execution** — executor + deviation rules + SUMMARY.md
Executes tasks atomically, self-corrects within bounds, declares gaps, writes verifiable state.

**5. Validation** — verifier + VERIFICATION.md + debugger
Independently verifies goal achievement with adversarial stance, four-level artifact checking, behavioral spot-checks, persistent failure investigation.

---

## For your harness specifically

Your six milestones map directly onto these five concerns:

| Your milestone | GSD concern | Key patterns to extract |
|---|---|---|
| M1: Foundation & Testing | Validation (layer 5) | Post-merge gate, independent validator, behavioral spot-checks |
| M2: Spec-Driven Development | Work specification (layer 3) | Plan quality contract, coverage gates, scope reduction detection |
| M3: Product Knowledge | Intent capture (layer 1) | Canonical refs, prior context accumulation, downstream-aware outputs |
| M4: Code Memory | Knowledge assembly (layer 2) | Role+data flow classification, claim provenance, analog matching with excerpts |
| M5: Dynamic Memory | Execution (layer 4) | Deviation rules, fix attempt limits, persistent debug brain, cross-session KB |
| M6: Agent Harness | All five, orchestrated | Wave execution, single-state writer, adversarial verifier, escalation gates |

The GSD architecture is not a monolith — it's five loosely coupled concerns connected by typed artifacts (CONTEXT.md, RESEARCH.md, PATTERNS.md, PLAN.md, SUMMARY.md, VERIFICATION.md). Your harness should follow the same principle: each milestone produces a typed artifact that the next milestone consumes. The artifact is the contract.