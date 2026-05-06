## gstack Architecture Map

From the folder structure, gstack has **5 distinct layers**:

**1. Skill layer** — `office-hours/`, `review/`, `qa/`, `ship/`, etc. Each skill is a folder with `SKILL.md` (the actual prompt/instruction) and `SKILL.md.tmpl` (the template that gets compiled). Skills aren't static — they're generated at runtime.

**2. Resolver/preamble system** — `scripts/resolvers/preamble/` This is the most interesting layer. Look at those filenames:

```
generate-confusion-protocol.ts
generate-context-health.ts
generate-search-before-building.ts
generate-continuous-checkpoint.ts
generate-context-recovery.ts
generate-routing-injection.ts
generate-test-failure-triage.ts
generate-spawned-session-check.ts
```

These dynamically inject context into skills at runtime. This is gstack's **context engineering engine** — it assembles the right preamble for each skill invocation.

**3. Browser layer** — `browse/src/` (~50 source files) Full Playwright/CDP automation. This is a substantial engineering effort — tab isolation, security classifier, domain skills, cookie management, tunnel support. Deep and production-grade.

**4. Memory/state layer** — `bin/gstack-*`, `learn/`, gbrain integration `gstack-learnings-log`, `gstack-memory-ingest.ts`, `gstack-brain-*` — persistent cross-session memory with a full lifecycle.

**5. Host adapter layer** — `hosts/` `claude.ts`, `codex.ts`, `cursor.ts`, `kiro.ts` etc. — abstracts across 10 AI coding agents.

---

## Sprint Workflow (the orchestration model)

```
/office-hours      → forces product clarity before code
/plan-ceo-review   → scope challenge
/plan-eng-review   → architecture + test plan
/plan-design-review→ UI/UX review
/autoplan          → runs all three automatically
        ↓
    [implement]
        ↓
/review            → staff engineer review (with specialist army)
/qa                → browser-based QA with bug fixing
/cso               → security audit
        ↓
/ship              → sync, test, PR
/land-and-deploy   → merge + verify production
/canary            → post-deploy monitoring
        ↓
/retro             → weekly reflection + metrics
```

Each stage **writes artifacts** that downstream stages read. This is the key design principle.

---

## Preliminary Pros/Cons

**Pros (things worth extracting):**

- **Preamble composition system** — `generate-*.ts` files are a clean way to inject dynamic context. The confusion protocol, context health, search-before-building — these are all reusable patterns.
- **Sprint structure as orchestration** — the Think→Plan→Build→Review→Test→Ship→Reflect sequence is well-thought-out and matches the four-phase pattern your plan already references.
- **Review army** — `review/specialists/` (api-contract, data-migration, maintainability, performance, red-team, security, testing) — each specialist is a focused sub-prompt. Directly applicable to your harness.
- **Continuous checkpoint mode** — WIP commits with structured `[gstack-context]` body. Crash-safe, resumable. Your harness needs this.
- **/learn + /retro** — cross-session memory with decay. Directly maps to your Milestone 5 (Dynamic Memory).
- **/investigate** — root cause methodology with a hard stop after 3 failed fixes. Maps to your circuit breaker requirement.
- **Checkpoint squashing in /ship** — WIP commits get filter-squashed before PR. Clean git history despite continuous checkpointing.
- **Confusion protocol** — stops the agent from guessing on ambiguous decisions. Relevant to your spec compliance problem.

**Cons / things to leave behind:**

- **Browser layer** — 50+ source files, Chrome extension, sidebar, security classifier. Massive. For GIS360 you need browser testing but you don't need gstack's full browser stack. Evaluate separately.
- **Design tooling** — `/design-shotgun`, `/design-html`, design taste memory. Not relevant to your harness.
- **Multi-host abstraction** — `hosts/` layer. You're targeting Claude Code specifically.
- **Garry-specific tooling** — `gstack-developer-profile`, `gstack-builder-profile`, throughput metrics. Personal productivity layer, not a harness pattern.
- **The marketing layer** — significant README real estate is Garry's personal productivity story. The actual engineering is more modest than it reads.

---

## What I'd suggest we read next

In priority order for your harness:

1. **`scripts/resolvers/preamble/generate-confusion-protocol.ts`** — the spec compliance mechanism
2. **`review/SKILL.md` + `review/specialists/*.md`** — the review army pattern
3. **`scripts/resolvers/preamble/generate-continuous-checkpoint.ts`** — checkpoint/recovery
4. **`learn/SKILL.md`** — cross-session memory
5. **`autoplan/SKILL.md`** — how it chains skills automatically
6. **`office-hours/SKILL.md`** — the product interrogation pattern
7. **`investigate/SKILL.md`** — root cause + circuit breaker
8. **`scripts/resolvers/index.ts`** — how the resolver system assembles everything

---
## File-by-File Analysis

### 1. `generate-completion-status.ts` — Two distinct things combined

**`generatePlanModeInfo`** — handles plan mode semantics. Not very relevant to your harness (plan mode is a Claude Code UI concept).

**`generateCompletionStatus`** — this one is valuable. Three patterns worth extracting:

**Pattern A: Structured completion states**

```
DONE / DONE_WITH_CONCERNS / BLOCKED / NEEDS_CONTEXT
```

With escalation after 3 failed attempts. This is a clean circuit breaker protocol. Your harness needs exactly this — not an open-ended retry loop, but a state machine with defined exit conditions. The `BLOCKED` and `NEEDS_CONTEXT` states are especially useful for human escalation handoff.

**Pattern B: Operational self-improvement hook**

```bash
gstack-learnings-log '{"skill":"...","type":"operational","key":"...","insight":"...","confidence":N}'
```

Every skill completion triggers a check: _"did I discover something durable?"_ Only logs if it saves 5+ minutes next time. This is a lightweight Milestone 5 pattern — not full RL, just structured observation at skill end. The `confidence` field and the "don't log obvious facts" guard are important — they prevent memory poisoning.

**Pattern C: Session telemetry** The `_TEL_START` / `_TEL_END` / duration tracking per skill is a baseline measurement system. Maps directly to your Milestone 1 token economics requirement.

---

### 2. `generate-ask-user-format.ts` — Decision brief protocol

This is one of gstack's strongest patterns. Key insight: **every human-in-the-loop pause is a structured decision brief, not a freeform question.**

The format enforces:

- `D<N>` numbering (tracks decisions across a session)
- ELI10 (plain English stakes)
- Explicit recommendation (agent never says "it's up to you")
- Completeness scoring (coverage vs kind — honest distinction)
- Dual-scale effort labels: `human: ~2 days / CC: ~15 min` — makes AI speed compression visible

**What's most extractable for your harness:** the `AUTO_DECIDE` path. When `/plan-tune` enables `AUTO_DECIDE`, the `(recommended)` label is machine-readable — the agent picks automatically without human input. This is how you get from _"pause for review"_ to _"autonomous with escalation only for genuine ambiguity"_. Your harness can adopt this: format every decision as a brief, define an autonomy threshold, auto-pick below threshold, escalate above.

**The fallback chain** is also clean:

1. MCP AskUserQuestion variant (host-registered)
2. Native AskUserQuestion
3. Plan file `## Decisions to confirm` section + ExitPlanMode
4. Never silently auto-decide

---

### 3. `generate-brain-health-instruction.ts` — Host-specific conditional

Short file. Only relevant if `ctx.host === 'gbrain' || ctx.host === 'hermes'`.

**Pattern worth noting:** conditional preamble injection based on host context. Your harness equivalent: inject different context blocks based on which knowledge system is available (product knowledge loaded vs not, code memory indexed vs not). The `if (ctx.host !== ...) return ''` pattern is clean — zero cost when feature is absent.

---

### 4. `generate-brain-sync-block.ts` — The most sophisticated file in this batch

This is a full session lifecycle management pattern. What it does at **skill start**:

1. Checks if gbrain is configured and indexed
2. If indexed → emits guidance to prefer `gbrain search` over grep
3. If not indexed → warns and falls back to grep
4. Checks sync mode, runs git fetch + ff-only merge (24h throttle)
5. Drains the sync queue
6. Emits `BRAIN_SYNC: mode=X | last_push=Y | queue=Z` — a health status line

At **skill end**:

```bash
gstack-brain-sync --discover-new
gstack-brain-sync --once
```

**Key architectural insight:** the sync is **bidirectional and throttled**. Pull at start (24h cache), push at end. The 24h pull throttle prevents hammering the remote on every skill invocation. The `--discover-new` at end captures new knowledge formed during the skill.

**Privacy stop-gate** is interesting — first-time sync setup is triggered inline as an `AskUserQuestion`, not a separate setup command. Setup happens opportunistically during normal workflow.

**What's directly applicable to your harness:**

- The `BRAIN_SYNC: off` / `mode=X | queue=Z` health status line pattern. Your harness should emit equivalent status lines for each knowledge system: `PRODUCT_MEM: loaded=Y | staleness=Z`, `CODE_MEM: indexed=Y | last_sync=Z`
- The 24h pull throttle with `--ff-only` merge — safe cross-session knowledge sync without conflicts
- The `--discover-new` pattern at skill end — passive knowledge accumulation

**What to leave behind:** the gbrain-specific infrastructure (Supabase, git-backed JSONL) is their specific implementation. Your harness needs the pattern, not the plumbing.

---

### 5. `generate-completeness-section.ts` — "Boil the Lake"

Short but sharp. The core principle: _AI makes completeness cheap, so always recommend the complete solution, but distinguish lakes (doable) from oceans (multi-quarter rewrites)._

The `Completeness: X/10` scoring in decision briefs is the mechanical enforcement of this. 10 = all edge cases, 7 = happy path, 3 = shortcut.

**Harness relevance:** this maps to your spec compliance problem. When the agent generates a plan, scoring completeness against acceptance criteria (not just "did it work?" but "did it handle all the cases?") is a lightweight version of SpecMem's SpecImpact Graph.

---
### 6. `generate-confusion-protocol.ts` — Small but critical

The entire file is 3 sentences:

> For **high-stakes ambiguity** (architecture, data model, destructive scope, missing context), STOP. Name it in one sentence, present 2-3 options with tradeoffs, and ask. Do not use for routine coding or obvious changes.

This looks deceptively simple, but it's doing something important: it defines a **trigger condition** for human escalation that is qualitative, not just retry-count-based. The categories — architecture, data model, destructive scope, missing context — are the specific cases where agent guessing is most dangerous.

**What makes it work:** the "Do not use for routine coding" guard is as important as the protocol itself. Without it, agents would stop on every minor decision. The protocol is opt-in for high stakes only, auto-decide for everything else.

**Harness application:** your Milestone 6 harness needs exactly this distinction. Map it to GIS360's specific high-stakes categories: spatial data model changes, pipe tracking schema, MVT rendering pipeline decisions, work correction logic — these are GIS360's "architecture/data model/destructive scope" equivalents. Everything else runs autonomously.

Combined with the `AskUserQuestion` decision brief from Batch 1, you now have the full escalation stack:

- Confusion Protocol → triggers the decision brief format
- Decision brief → structures the question
- AUTO_DECIDE → determines if it escalates or self-resolves

---

### 7. `generate-context-health.ts` — Two things in one file

**The function itself:**

```
[PROGRESS] summaries: done, next, surprises
STOP if looping on same diagnostic / same file / failed fix variants
```

The loop detection heuristic is simple but effective: _same diagnostic, same file, or failed fix variants_ = you're spinning. This is the behavioral signal that triggers circuit breaker escalation in your Milestone 5.

The `surprises` field in progress summaries is underrated — it's where the agent records unexpected discoveries that might be worth persisting to memory.

**The comment block at the bottom** is the most valuable thing in this file:

```
// Preamble Composition (tier → sections)
// T1: core + upgrade + lake + telemetry + voice(trimmed) + completion
// T2: T1 + voice(full) + ask + completeness + context-recovery
// T3: T2 + repo-mode + search
// T4: (same as T3 — TEST_FAILURE_TRIAGE is a separate placeholder)
//
// Skills by tier:
//   T1: browse, setup-cookies, benchmark
//   T2: investigate, cso, retro, doc-release, setup-deploy, canary
//   T3: autoplan, codex, design-consult, office-hours, ceo/design/eng-review
//   T4: ship, review, qa, qa-only, design-review, land-deploy
```

This reveals gstack's **context budget management strategy**. Skills get different preamble tiers based on how much context they need. Lightweight tools (browse, benchmark) get minimal preamble. Heavy workflow skills (ship, review, qa) get the full stack including test failure triage.

This is a direct answer to your Milestone 1 context efficiency problem. You don't inject all knowledge into every task — you tier it by task type. For your harness: bug fixes need code memory + context recovery. Feature additions need product knowledge + code memory + spec compliance. Refactors need all of it. Map your task types to preamble tiers.

---

### 8. `generate-context-recovery.ts` — The session reconstruction pattern

This is the most immediately applicable file in this batch for your harness. What it does at session start:

```bash
# 1. Resolve project slug from git remote
eval "$(gstack-slug)"

# 2. Find recent artifacts for this project+branch
find "$_PROJ/ceo-plans" "$_PROJ/checkpoints" -type f -name "*.md" | head -3

# 3. Count reviews on this branch
wc -l < "$_PROJ/${_BRANCH}-reviews.jsonl"

# 4. Read last 5 timeline entries
tail -5 "$_PROJ/timeline.jsonl"

# 5. Find last completed skill on this branch
grep "branch":"${_BRANCH}" timeline.jsonl | grep "completed" | tail -1

# 6. Surface recent skill pattern
# → "RECENT_PATTERN: office-hours,plan-eng-review,review,"
```

Then the LLM instruction:

- If artifacts exist → read the newest useful one
- If `LAST_SESSION` exists → give a 2-sentence welcome back summary
- If `RECENT_PATTERN` implies a next skill → suggest it once

**Three patterns to extract:**

**Pattern A: Branch-scoped artifact storage.** Everything is stored under `projects/{slug}/{branch}-*`. This means context recovery is naturally scoped to the current branch — you pick up exactly where you left off, not from a generic project state.

**Pattern B: Timeline as session history.** The `timeline.jsonl` is a lightweight event log (skill, event, branch, outcome, duration, session). This is your Milestone 5 dynamic memory at its simplest — before you build anything sophisticated, a JSONL event log already gives you last session state, recent pattern detection, and skill sequencing.

**Pattern C: `RECENT_PATTERN` for next-step suggestion.** This is a lightweight version of your harness's orchestration — if the last three skills on this branch were `office-hours → plan-eng-review → [nothing]`, suggest `/review` or implementation. Pattern recognition from history, no ML needed.

**For your GIS360 harness:** store per-branch artifacts under `projects/{repo}/{branch}/`. Timeline JSONL captures every task attempt. Session recovery reads the last 5 entries and reconstructs state. This is cheaper and more reliable than trying to maintain a full memory graph from the start.

---

### 9. `generate-continuous-checkpoint.ts` — The most complete pattern in both batches

The commit format is worth quoting in full:

```
WIP: <concise description>

[gstack-context]
Decisions: <key choices made this step>
Remaining: <what's left in the logical unit>
Tried: <failed approaches worth recording>
Skill: </skill-name>
[/gstack-context]
```

The rules:

- Stage only intentional files, never `git add -A`
- Never commit broken tests or mid-edit state
- Push only if `CHECKPOINT_PUSH=true` (default: local only, no CI triggered)
- `/ship` squashes WIP commits before PR

**Why this matters for your harness:** this is crash-safe, resumable, and CI-safe all at once. The `[gstack-context]` block in the commit message is machine-readable state — `context-restore` parses it to reconstruct what the agent was doing. `Tried:` is especially valuable — it prevents the agent from attempting the same failed approach after a session reset, which is exactly your Milestone 5 "avoid past mistakes" requirement.

The `ship` squash behavior keeps git history clean for code review while preserving full WIP granularity for recovery. This is the right default — developers see a clean diff, the agent sees full state.

**Two modes** (`continuous` vs `explicit`) let you tune by task type. For your harness: autonomous tasks run in `continuous` mode (every logical unit checkpointed), interactive tasks run in `explicit` mode.

---
### 10. `generate-search-before-building.ts` — Three-layer knowledge hierarchy

The three layers:

- **Layer 1** (tried and true) — don't reinvent
- **Layer 2** (new and popular) — scrutinize
- **Layer 3** (first principles) — prize above all

Useful framing but not the main pattern here. The **Eureka log** is:

```bash
jq -n ... --arg insight "ONE_LINE_SUMMARY" '{ts,skill,branch,insight}' >> ~/.gstack/analytics/eureka.jsonl
```

This is a lightweight insight capture — when first-principles reasoning contradicts conventional wisdom, the agent logs it. This is distinct from `gstack-learnings-log` (operational fixes) and `gstack-timeline-log` (events). Three separate streams:

|Log|Captures|
|---|---|
|`eureka.jsonl`|First-principles insights that contradict convention|
|`learnings.jsonl`|Operational project quirks (saves 5+ min next time)|
|`skill-usage.jsonl`|Telemetry (skill, duration, outcome)|
|`timeline.jsonl`|Session events (started, completed, branch)|

**Harness application:** your Milestone 5 memory formation layer needs exactly this separation. Don't dump everything into one memory store — different insight types need different retrieval patterns and different expiry policies.

---

### 11. `generate-preamble-bash.ts` — The session bootstrap engine

This is the most important file in all three batches. Every skill invocation starts with this bash block. Let me map what it captures:

```bash
# 1. Auto-update check (throttled)
_UPD=$(gstack-update-check)

# 2. Parallel session count (120-min window)
_SESSIONS=$(find ~/.gstack/sessions -mmin -120 | wc -l)

# 3. Branch name
_BRANCH=$(git branch --show-current)

# 4. Config flags
PROACTIVE, SKILL_PREFIX, REPO_MODE, LAKE_INTRO, TELEMETRY, EXPLAIN_LEVEL, QUESTION_TUNING

# 5. Session ID = PID + timestamp
_SESSION_ID="$$-$(date +%s)"

# 6. Learnings injection
_LEARN_FILE="~/.gstack/projects/${SLUG}/learnings.jsonl"
→ count entries
→ if >5, run gstack-learnings-search --limit 3  # top-3 relevant learnings auto-injected

# 7. Timeline: log skill started
gstack-timeline-log '{"skill":"...","event":"started","branch":"...","session":"..."}'

# 8. Routing state
HAS_ROUTING, ROUTING_DECLINED, VENDORED_GSTACK

# 9. Checkpoint mode
CHECKPOINT_MODE, CHECKPOINT_PUSH

# 10. Spawned session flag
[ -n "$OPENCLAW_SESSION" ] && echo "SPAWNED_SESSION: true"

# 11. Brain health (host-conditional)
gbrain doctor --fast --json → BRAIN_HEALTH score
```

**Five patterns worth extracting:**

**Pattern A: Session identity.** `_SESSION_ID="$$-$(date +%s)"` — PID + timestamp. Cheap, collision-resistant, no external state. Every event logged against this ID. Lets you reconstruct a complete session trace from JSONL after the fact.

**Pattern B: Learnings auto-injection.** This is the payoff for all the `gstack-learnings-log` calls:

```bash
if [ "$_LEARN_COUNT" -gt 5 ]; then
  gstack-learnings-search --limit 3
fi
```

At session start, the top 3 relevant learnings are surfaced automatically. The agent doesn't have to remember to check — memory retrieval is built into the preamble. Threshold of 5 prevents noise when the knowledge base is sparse.

**Pattern C: Parallel session awareness.** Counting active sessions (120-min window) gives the agent context about whether it's running solo or alongside other agents. For your harness with Conductor-style parallelism, this is how you detect conflicts and coordinate.

**Pattern D: Pending telemetry cleanup.** The `.pending-*` file pattern handles crashed sessions — if a previous session didn't complete its telemetry, it gets cleaned up on next start. This is a lightweight crash recovery signal.

**Pattern E: The full environment as structured output.** Every variable is echoed as `KEY: value`. The LLM reads this as a structured status block before any skill instructions. This is context engineering in its purest form — the agent's operating environment is explicit state, not implicit assumption.

**For your GIS360 harness:** adapt this as your session initialization block. Replace gstack-specific variables with yours:

```
BRANCH: feature/pipe-tracking-fix
TASK_TYPE: bug_fix
PRODUCT_MEM: loaded=yes | features=3 | staleness=2d
CODE_MEM: indexed=yes | last_sync=4h | utilization=12%
LEARNINGS: 8 entries | top-3 injected
CHECKPOINT_MODE: continuous
SESSION_ID: 12847-1746123456
```

---
### 12. `generate-repo-mode-section.ts` — Useful scope boundary pattern

```
solo → you own everything, investigate and fix proactively
collaborative/unknown → flag via AskUserQuestion, don't fix
```

This is a simple but important **scope boundary** for autonomous agents. Without it, the agent might "helpfully" fix unrelated code it noticed while working on something else — classic over-compliance.

**Harness application:** your GIS360 harness needs equivalent scope boundaries. Map to your task definition: the harness should only touch files within the scope of the current requirement. Anything outside scope → flag, don't fix. This directly addresses your Milestone 1 failure mode: "agent modified unrelated files."

The `REPO_MODE` flag being runtime-configurable is the right design — some GIS360 tasks should be tight-scoped, others (refactors, dependency updates) legitimately touch many files.

---

### 13. `generate-routing-injection.ts` — The CLAUDE.md self-setup pattern

This auto-generates skill routing rules in `CLAUDE.md` on first use. The routing table is worth reading:

```markdown
## Skill routing
- Product ideas/brainstorming → /office-hours
- Strategy/scope → /plan-ceo-review
- Architecture → /plan-eng-review
- Bugs/errors → /investigate
- QA/testing → /qa or /qa-only
- Code review → /review
- Ship/deploy → /ship or /land-and-deploy
- Save progress → /context-save
- Resume context → /context-restore
```

**The pattern that matters:** routing rules are **written to the project file**, not held in the agent's context. This makes routing persistent and inspectable. Any developer can open CLAUDE.md and see how requests map to workflows.

**Harness application:** your `CLAUDE.md` for GIS360 should have an equivalent routing table for your harness workflows. When someone files a requirement, the harness reads this table to select the right pipeline. This is your Milestone 6 knowledge routing decision externalized as configuration.

---
1. generatePreambleBash        → env vars, session ID, learnings injection
2. generatePlanModeInfo        → plan mode semantics
3. generateUpgradeCheck        → (not yet seen)
4. generateBrainSyncBlock      → knowledge sync
5. generateContextRecovery     → artifact recovery
6. generateRepoModeSection     → scope boundary
7. generateSearchBeforeBuilding→ knowledge hierarchy
8. generateConfusionProtocol   → escalation trigger
9. generateContextHealth       → loop detection
10. generateCompletenessSection → completeness scoring
11. generateAskUserFormat      → decision brief format
12. generateCompletionStatus   → exit states + telemetry

---

### 14. `generate-spawned-session-check.ts` — Critical for multi-agent harness

Short file, maximum importance:

```
If SPAWNED_SESSION is "true":
- Do NOT use AskUserQuestion for interactive prompts → auto-choose recommended option
- Do NOT run upgrade checks, telemetry prompts, routing injection, or lake intro
- Focus on completing the task
- End with completion report: what shipped, decisions made, anything uncertain
```

This is the **orchestrated vs interactive mode switch**. When your harness spawns a subagent, that subagent needs to know it's not talking to a human — it should suppress all interactive prompts, auto-pick recommended options, and report results as structured output instead.

This solves a real problem you'll hit in Milestone 6: if your Agent Harness spawns subagents (MetaGPT-style roles — Architect, Engineer, QA), each subagent will try to ask clarifying questions. `SPAWNED_SESSION` suppresses that entire interaction layer and routes decisions through `AUTO_DECIDE` on the `(recommended)` label.

The **completion report** at the end (`what shipped, decisions made, anything uncertain`) is the subagent's return value to the orchestrator. This is the handoff protocol.

For your harness: every spawned subagent gets `SPAWNED_SESSION=true` in its preamble. The orchestrator reads the completion report, extracts decisions and uncertainties, and routes accordingly.

---

### 15. `generate-test-failure-triage.ts` — The most complete pattern in all four batches

This is a full protocol, not just a prompt. Let me map the structure:

**Step T1: Classify each failure**

```bash
git diff origin/<base>...HEAD --name-only
```

Then for each failing test:

- **In-branch** = test file or code it tests was modified on this branch
- **Likely pre-existing** = no connection to branch diff
- **When ambiguous, default to in-branch** — safer to stop than to let a broken test ship

**Step T2: In-branch failures → STOP** Hard stop. The agent does not proceed. These are the agent's own failures to fix.

**Step T3: Pre-existing failures → decision by REPO_MODE**

`solo` repo:

```
A) Investigate and fix now (human: ~2-4h / CC: ~15min) — Completeness: 10/10
B) Add as P0 TODO — Completeness: 7/10
C) Skip and ship anyway — Completeness: 3/10
```

`collaborative` repo:

```
A) Fix now — Completeness: 10/10
B) Blame + assign GitHub issue — Completeness: 9/10
C) Add as P0 TODO — Completeness: 7/10
D) Skip — Completeness: 3/10
```

**Step T4: Execute** — each choice has a concrete implementation path including `git blame` logic (prefer production code author over test file author — they likely introduced the regression), `gh issue create` with templated body, and `TODOS.md` entry format.

**Five patterns worth extracting:**

**Pattern A: Branch diff as failure classifier.** Before deciding what to do about a test failure, determine ownership via `git diff origin/<base>...HEAD`. This prevents the agent from either blindly fixing other people's broken tests or ignoring its own regressions. For your GIS360 harness, this is the regression detection mechanism — if a test was passing before the agent's changes and is now failing, it's the agent's fault.

**Pattern B: Conservative default on ambiguity.** "When ambiguous, default to in-branch." This is the right failure mode — it's better to stop the agent unnecessarily than to let a real regression ship. Your harness should encode the same conservatism.

**Pattern C: Completeness scores on test decisions.** The `Completeness: 10/10 / 7/10 / 3/10` scoring makes the tradeoff explicit. Shipping with a known failing test is a 3/10 completeness choice, not a hidden technical debt. This makes the agent's choices auditable.

**Pattern D: `git blame` for pre-existing failures.** The protocol prefers the production code author over the test file author when assigning blame. This is a non-obvious heuristic — whoever last touched the code the test covers is more likely the actual regression introducer. Extractable directly.

**Pattern E: Separate commit for pre-existing fixes.** If the agent fixes a pre-existing failure, it commits separately from branch changes: `git commit -m "fix: pre-existing test failure in <test-file>"`. Clean separation of concerns in git history.

**For your GIS360 harness:** this entire protocol maps directly to your Milestone 1 acceptance criteria — "implemented spec-compliance verification: agent checks its output against acceptance criteria." Your version: after every implementation, run tests, classify failures by branch ownership, stop on in-branch failures, triage pre-existing ones with the completeness scale.

---

### 16. `generate-voice-directive.ts` — One extractable principle

The tier 2 voice guidance is worth reading once:

> "Good: `auth.ts:47 returns undefined when the session cookie expires. Users hit a white screen. Fix: add a null check and redirect to /login. Two lines.` Bad: `I've identified a potential issue in the authentication flow that may cause problems under certain conditions.`"

The concrete vs vague distinction is a prompt quality principle. For your harness: when the agent reports findings (from `/review` equivalent, QA, spec compliance), require the format `file:line → symptom → fix → impact`. No vague summaries.

The word blacklist (`delve, crucial, robust, comprehensive, nuanced, multifaceted`) is real — these words consistently appear when models are uncertain and padding. Encoding this in your preamble reduces noise in agent output.

### 17. `generate-writing-style.ts` — One mechanism worth noting

```typescript
function loadJargonList(): string[] {
  // loads from scripts/jargon-list.json
  // graceful degradation if missing
}
```

The **externalized jargon list** pattern — domain vocabulary is stored in a JSON file, not hardcoded in the prompt. For GIS360 your jargon list would include: MVT, pipe tracking, work correction, spatial join, feature layer, tile cache. Glossing these on first use in any agent output keeps the team aligned without assuming everyone knows the domain.

---
### Extract: The Harness Preamble Stack

Adapt this initialization sequence for GIS360:

```
SESSION INIT (preamble-bash)
├── Session ID = PID + timestamp
├── Branch name
├── Task type (from requirement metadata)
├── Knowledge system health
│   ├── PRODUCT_MEM: loaded=Y | features=N | staleness=Xd
│   ├── CODE_MEM: indexed=Y | last_sync=Xh | utilization=Y%
│   └── LEARNINGS: N entries | top-3 injected
├── Checkpoint mode (continuous / explicit)
├── Harness mode (autonomous / supervised)
└── Spawned session flag

KNOWLEDGE CONTEXT (conditional by tier)
├── T1 (bug fix): code memory + context recovery
├── T2 (feature): product knowledge + code memory + spec compliance
└── T3 (refactor): all knowledge systems

BEHAVIORAL PROTOCOLS
├── Scope boundary (REPO_MODE → file scope)
├── Confusion protocol (GIS360-specific high-stakes categories)
├── Loop detection (same file / same diagnostic = STOP)
├── Spawned session mode (auto-decide, no interactive prompts)
└── Completeness scoring

COMPLETION PROTOCOL
├── DONE / DONE_WITH_CONCERNS / BLOCKED / NEEDS_CONTEXT
├── Operational learning log (if durable insight found)
├── Timeline event log (completed, outcome, duration)
└── Completion report for orchestrator (spawned sessions)
```

---

### Extract: The Test Failure Triage Protocol

Directly usable in Milestone 1 and Milestone 6:

```
1. git diff origin/<base>...HEAD --name-only
2. Classify each failure: in-branch vs pre-existing
3. In-branch → STOP, agent fixes before proceeding
4. Pre-existing → triage with completeness scores
5. Pre-existing fixes committed separately
```

---

### Extract: The Memory Stream Architecture

Four separate logs, each with distinct purpose:

|Stream|Content|Retention|
|---|---|---|
|`timeline.jsonl`|Session events (started/completed/branch/outcome)|Full history|
|`learnings.jsonl`|Operational quirks (5+ min savings)|28-day expiry|
|`eureka.jsonl`|First-principles insights|Permanent|
|`skill-usage.jsonl`|Telemetry (duration, outcome)|Aggregate|

---

### Extract: The Decision Brief Protocol

Every human-in-the-loop pause uses this format:

- D\<N> header
- ELI10 stakes
- Explicit recommendation with `(recommended)` label
- Completeness score
- Dual-scale effort (human time / agent time)
- `AUTO_DECIDE` reads `(recommended)` when in autonomous mode

---

### Leave Behind

|Component|Reason|
|---|---|
|Browser layer (`browse/src/`)|Full Playwright stack — evaluate separately for GIS360 testing|
|Design tooling (shotgun, HTML, consultation)|Not relevant to GIS360 harness|
|Multi-host adapters (`hosts/`)|You're targeting Claude Code specifically|
|GBrain infrastructure|Use the pattern (bidirectional throttled sync), build your own storage|
|Garry-specific profiling (developer-profile, builder-profile)|Personal productivity, not harness|
|Telemetry/upgrade/onboarding prompts|gstack product concerns|
|Vendoring deprecation|Distribution concern|

---

### What gstack doesn't have that your harness needs

After reading all 15 preamble generators, these gaps are clear:

1. **No product knowledge layer.** gstack has `learnings.jsonl` for operational quirks but nothing for requirements, design intent, or business rules. Your Milestone 3 is a genuine addition, not a reinvention of something gstack already solved.
    
2. **No spec compliance verification.** The test failure triage is excellent for code correctness, but there's no protocol for "does this implementation match the acceptance criteria in the PRD?" Your SpecMem pattern fills this gap.
    
3. **No cross-session failure pattern detection.** `Tried:` in WIP commits records failed approaches per-session, but there's no aggregation across sessions to detect _recurring_ failure patterns. Your Milestone 5 Reflexion/ExpeL layer adds this.
    
4. **No task decomposition.** gstack assumes a human defines the task. Your harness needs to take a natural language requirement and decompose it into a plan and task list — the four-phase pipeline (specify, plan, task, implement) from Milestone 2.
    
5. **No retrieval system.** `gstack-learnings-search` is a simple JSONL search. No vector embeddings, no graph traversal, no hybrid retrieval. Your Milestones 3-4 are the knowledge infrastructure gstack never built.
    

---
## The Review Army Architecture

Before the individual files, the structural insight: gstack's `/review` runs **two passes in parallel**:

```
Pass 1 (main agent, sequential):
  CRITICAL checks → SQL safety, race conditions, LLM trust, shell injection, enum completeness

Pass 2 (parallel specialist subagents):
  api-contract | data-migration | maintainability | performance | red-team | security | testing
```

Each specialist outputs **structured JSON**, one finding per line:

```json
{"severity":"CRITICAL","confidence":8,"path":"file.ts","line":47,
 "category":"security","summary":"...","fix":"...","fingerprint":"path:line:security","specialist":"security"}
```

The `fingerprint` field (`path:line:category`) enables deduplication across specialists. The `confidence` field (1-10) gates what gets surfaced. This is a production-grade finding pipeline, not a freeform review prompt.

---

### `checklist.md` — The Main Review Protocol

**Two-pass structure:**

- Pass 1 = CRITICAL (SQL, race conditions, LLM trust, shell injection, enum completeness)
- Pass 2 = INFORMATIONAL (async/sync mixing, column safety, dead code, LLM prompts, completeness gaps, time windows, type coercion, view issues, CI/CD)
- Specialist categories = handled by parallel subagents, NOT the main checklist

**The Fix-First Heuristic** is the most extractable pattern:

```
AUTO-FIX (no human needed):          ASK (needs human judgment):
├─ Dead code / unused variables      ├─ Security (auth, XSS, injection)
├─ N+1 queries                       ├─ Race conditions
├─ Stale comments                    ├─ Design decisions
├─ Magic numbers → named constants   ├─ Large fixes (>20 lines)
├─ Missing LLM output validation     ├─ Enum completeness
├─ Version/path mismatches           ├─ Removing functionality
└─ Inline styles, O(n*m) lookups     └─ Anything changing user-visible behavior

Rule: if a senior engineer would apply it without discussion → AUTO-FIX
      if reasonable engineers could disagree → ASK
```

This is your Milestone 6 autonomy threshold encoded as a decision rule. Critical findings default to ASK, informational default to AUTO-FIX.

**The Suppressions list** is equally important — it tells the agent what NOT to flag:

- Harmless redundancy that aids readability
- Threshold/constant comments (they rot)
- Tighter assertions that already cover the behavior
- Consistency-only changes
- Anything already addressed in the diff being reviewed

Without explicit suppressions, agents generate noise. For GIS360, you'd add: spatial operation patterns that look like N+1 but are actually intentional geometry batching, MVT tile cache invalidations that look like redundant queries, etc.

**Enum completeness trace** is a standout pattern:

> _Trace it through every consumer. Read (not grep — READ) each file that switches on the value._

This is the difference between a grep-based review and a semantic review. For your GIS360 harness: when a new pipe status, work correction type, or layer type is added, the agent must trace it through every switch, filter array, and display component — not just search for the literal string.

---
### `TODOS-format.md` — Structured backlog management

The format:

```markdown
### <Title>
**What:** one-line description
**Why:** concrete problem or value
**Context:** 3-month-later handoff detail
**Effort:** S/M/L/XL
**Priority:** P0/P1/P2/P3/P4
**Depends on:** <prerequisites>
```

**What's extractable:** the `Context` field is what makes this useful vs a standard TODO. "Enough detail that someone picking this up in 3 months understands the motivation, the current state, and where to start." For your harness, every finding that becomes a deferred TODO needs this level of context — otherwise the agent that runs the retro three weeks later has no idea why it was deferred.

The priority definitions map cleanly to your harness:

- **P0** = blocks next release (agent must fix before shipping)
- **P1** = this cycle (flag in completion report)
- **P2+** = deferred with context

---

### `greptile-triage.md` — External reviewer integration pattern

This is gstack's integration with Greptile (an external AI code review service). The pattern is more interesting than the specific tool:

**Per-project suppression history:**

```
~/.gstack/projects/{slug}/greptile-history.md
date | repo | type:fp|fix|already-fixed | file-pattern | category
```

Each triage outcome is recorded. Future reviews suppress known false positives. This is **reviewer memory** — the system learns which findings from this reviewer are noise in this codebase.

For your GIS360 harness: every automated review finding should be recorded with outcome. Over time, findings that are consistently marked false-positive for GIS360-specific patterns get suppressed automatically. This is Milestone 5 cross-session learning applied to the review layer.

**Two-tier reply escalation:**

- Tier 1 (first response) — friendly, evidence-included
- Tier 2 (re-flagged after prior reply) — firm, overwhelming evidence

The escalation detection before composing a reply prevents the agent from being either passive (ignoring repeated false positives) or aggressive (escalating prematurely).

---

### The 7 Specialist Files — The Review Army

Each specialist follows the same contract:

- **Scope condition** — when does this specialist run?
- **Output format** — structured JSON, one finding per line
- **NO FINDINGS** — explicit empty state
- **Categories** — specific, concrete, not vague

|Specialist|Scope Trigger|GIS360 Relevance|
|---|---|---|
|`maintainability`|Always-on|High — dead code, magic numbers, stale comments|
|`testing`|Always-on|High — missing negative paths, edge cases|
|`security`|Auth OR backend >100 lines|High — input validation, auth bypass|
|`performance`|Backend OR frontend|High — N+1 on spatial queries, missing indexes|
|`api-contract`|SCOPE_API=true|Medium — MVT tile API, work correction API|
|`data-migration`|SCOPE_MIGRATIONS=true|Medium — schema changes to pipe/spatial tables|
|`red-team`|diff >200 lines OR critical findings|High for major features|

**`red-team.md` is the most sophisticated.** It's explicitly NOT a checklist — it's adversarial analysis that runs after all other specialists to find what they missed. Five attack angles: happy path under load, silent failures, trust assumption exploits, edge case breaks, and cross-specialist gaps. The instruction "Think like an attacker, a chaos engineer, and a hostile QA tester simultaneously" gives the subagent a persona that changes its reasoning mode.

**`testing.md`** — the `fingerprint` JSON output pattern makes test findings directly actionable. The `test_stub` optional field (skeleton test code in the project's test framework) is excellent — findings come with their own fix templates.

**`data-migration.md`** is directly applicable to GIS360. The categories map perfectly: lock duration on spatial/geometry tables (PostGIS ALTER TABLE is expensive), backfill strategy for NOT NULL columns on large pipe datasets, multi-phase safety for schema changes that must coordinate with application deploys.

---

## Extracting the Review Army for Your Harness

Here's how this maps to your Milestone 1 and Milestone 6:

**For Milestone 1 (automated testing evaluation):** The two-pass review structure answers your MCP vs Computer Use question directly. Pass 1 (critical checks) maps to MCP-based testing — programmatic, deterministic, fast. Pass 2 specialist layer maps to semantic review that requires reading code, not just running it. The hybrid strategy is: MCP for execution validation, specialist review for code quality.

**For Milestone 6 (harness validation layer):**

Your harness validation chain after every implementation:

```
1. Test Failure Triage (from generate-test-failure-triage.ts)
   → classify in-branch vs pre-existing → STOP on in-branch

2. Pre-Landing Review (checklist.md)
   → Pass 1: CRITICAL checks
   → Pass 2: INFORMATIONAL checks
   → Fix-First heuristic → AUTO-FIX or ASK

3. Specialist Army (parallel subagents)
   → maintainability + testing (always)
   → security + performance (conditional)
   → data-migration (if schema changed)
   → api-contract (if API changed)
   → red-team (if diff > 200 lines)

4. Spec Compliance Check (your addition — gstack doesn't have this)
   → acceptance criteria from product knowledge → pass/fail per criterion

5. Completion Report
   → DONE / DONE_WITH_CONCERNS / BLOCKED
```

**GIS360-specific specialist categories to add:**

Since gstack's specialists are generic, your harness needs GIS360-specific additions within each:

- **Performance specialist**: spatial query plans (missing GIST indexes on geometry columns), MVT tile generation time, feature count thresholds that trigger rendering degradation
- **Data migration specialist**: PostGIS geometry type changes, coordinate system migrations, spatial index rebuild requirements
- **Testing specialist**: pipe geometry validation tests, MVT tile output verification, work correction state machine coverage
- **Security specialist**: spatial data access control (which users can see which pipe segments), work correction authorization

---

## Full Review Army Summary

|Pattern|Extractable?|Harness Application|
|---|---|---|
|Two-pass review (critical → informational)|✅|Milestone 1 testing strategy|
|Parallel specialist subagents|✅|Milestone 6 validation layer|
|Structured JSON finding output|✅|Finding pipeline with dedup via fingerprint|
|Fix-First Heuristic (auto-fix vs ask)|✅|Autonomy threshold for Milestone 6|
|Explicit suppression list|✅|GIS360-specific false positive suppression|
|Enum completeness trace (read, not grep)|✅|Semantic review requirement|
|Per-project reviewer memory (history file)|✅|Cross-session review learning for Milestone 5|
|Two-tier reply escalation|✅|External reviewer integration pattern|
|TODOS-format with Context field|✅|Deferred finding structure|
|Red team as adversarial persona|✅|Post-specialist gap analysis|
|NO FINDINGS explicit empty state|✅|Clean specialist output contract|
|`design-checklist.md`|❌|Not relevant to GIS360 backend|

---

## What autoplan actually is

It's not just a pipeline runner. It's an **autonomous decision engine** with an explicit theory of which decisions require humans and which don't. The architecture has five distinct components worth extracting separately.

---
### 1. The 6 Decision Principles — The Autonomy Engine

```
1. Choose completeness — cover more edge cases
2. Boil lakes — fix everything in blast radius if < 1 day CC effort
3. Pragmatic — cleaner of two equivalent options, 5 seconds choosing
4. DRY — reject duplicates of existing functionality
5. Explicit over clever — 10-line obvious fix > 200-line abstraction
6. Bias toward action — merge > review cycles > stale deliberation
```

With phase-specific tiebreakers:

- CEO phase: completeness + boil lakes dominate
- Eng phase: explicit + pragmatic dominate
- Design phase: explicit + completeness dominate

**This is the answer to your Milestone 6 autonomy question.** The harness doesn't need a model to decide — it needs a ranked set of principles it applies in order. When two options are viable, principle 1 wins. If still tied, principle 2. This is deterministic enough to be machine-executable, but flexible enough to handle real tradeoffs.

**For GIS360:** adapt the principles to your domain:

- P1 stays (completeness)
- P2: blast radius = files touched + direct importers of changed pipe/spatial models
- P3 stays (pragmatic)
- P4: GIS360-specific DRY — existing spatial utilities, pipe query patterns, MVT rendering helpers
- P5 stays (explicit)
- P6 stays (bias toward action)

---
### 2. Decision Classification — The Three-Tier Human Escalation Model

```
Mechanical → one clearly right answer → auto-decide silently
Taste      → reasonable people could disagree → auto-decide + surface at final gate
User Challenge → both models agree user's direction should change → NEVER auto-decide
```

**User Challenge** is the critical category. When both models (Claude + Codex subagent) independently recommend changing what the user explicitly asked for — merge features, split workflows, add/remove scope — this is never auto-decided. The user's original direction is the **default**. Models must make the case for change.

The format when surfacing a User Challenge:

```
What the user said: [original direction]
What both models recommend: [the change]
Why: [reasoning]
What context we might be missing: [explicit blind spots acknowledgment]
If we're wrong, the cost is: [downside of changing]
```

This is intellectually honest in a way most agent systems aren't. It acknowledges the models have blind spots, defaults to the human's stated intent, and only escalates when two independent models agree. For your harness: a single model recommendation on scope → auto-decide. Two independent signals (your harness's planning phase + spec compliance check) both flagging the same issue → User Challenge gate.

---

### 3. Sequential Execution With Phase Verification

```
Phase 0: Restore point + intake
Phase 1: CEO review (scope, strategy, premises)
Phase 2: Design review (UI scope only — skip otherwise)
Phase 3: Eng review (architecture, test plan, failure modes)
Phase 3.5: DX review (developer-facing scope only)
Phase 3.9: Completeness checklist
Phase 4: Final Approval Gate
```

**Sequential is mandatory** — each phase builds on the previous. Between phases: emit a transition summary and verify all required outputs exist before proceeding.

**Conditional phase execution** is the efficiency insight. Design review only runs if UI scope is detected. DX review only runs if developer-facing scope is detected. This maps directly to your preamble tier system from Batch 3 — inject only the context relevant to the task type.

**The completeness checklist before the gate** (lines 727-766) is the self-verification step. Before presenting the final gate, the agent runs through a structured checklist of required outputs. If any checkbox is missing: retry up to 2 times, then proceed with a warning. The explicit `Max 2 attempts — do not loop indefinitely` is your circuit breaker.

---
### 4. Dual Voices — Cross-Model Validation

Every phase runs both Claude (subagent) and Codex for independent analysis. The consensus table:

```
CEO Voices: Codex [summary], Claude subagent [summary], Consensus [X/6 confirmed]
```

When both models agree → high-confidence signal. When they disagree → surfaces as a Taste decision. When both disagree with the user → User Challenge.

**The cross-phase themes detection** is the most sophisticated part:

> For any concern that appeared in 2+ phases' dual voices independently: flag as a high-confidence signal.

A concern flagged by the CEO review AND independently by the Eng review carries more weight than either alone. For your harness: if your product knowledge check AND your code memory check both flag the same issue, that's a high-confidence signal worth escalating.

**For your harness:** you don't have Codex available as a second model in the same way, but the dual-voice pattern still applies. Your "second voice" can be: a separate Claude subagent given a different system prompt (skeptic vs builder), or your spec compliance check as an independent signal against your product knowledge layer.

---
### 5. The Restore Point Pattern

Phase 0, first action:

```bash
RESTORE_PATH=$HOME/.gstack/projects/$SLUG/${BRANCH}-autoplan-restore-${DATETIME}.md
# Write plan file's full contents to restore path
```

Before doing anything, snapshot the current state. If the run fails or the output is wrong, the user can restore to the pre-autoplan state. This is the cheapest possible crash safety — one file write at the start.

**For your harness:** before every task execution, snapshot the current state of the files the agent is about to modify. Not a git commit — a literal copy. Cheaper than a WIP commit, faster to restore.

---
### 6. The Final Approval Gate — Cognitive Load Management

```
0 user challenges → skip that section
0 taste decisions → skip that section
1-7 taste decisions → flat list
8+ taste decisions → group by phase + warning: "unusually high ambiguity"
```

The 8+ warning is a quality signal — if you're surfacing that many taste decisions, the requirement was probably underspecified or the plan is too large. This maps to your Milestone 2 spec quality measurement.

**Gate options:**

- A) Approve as-is
- B) Approve with overrides
- C) Interrogate (freeform question about any decision)
- D) Revise (re-run affected phases, max 3 cycles)
- E) Reject

Option D with `max 3 cycles` is your revision circuit breaker. If the plan needs more than 3 rounds of revision, something is fundamentally wrong with the requirement — escalate.

---
### 7. The "Full Depth Means Full Depth" Rule

> _"If you catch yourself writing fewer than 3 sentences for any review section, you are likely compressing."_

> _"`No issues found` is valid — but only after doing the analysis. State what you examined and why nothing was flagged."_

> _"`Skipped` is never valid for a non-skip-listed section."_

This is the anti-compression directive. Agent compression (summarizing instead of executing) is one of your six listed risks. gstack encodes the counter-pressure directly in the skill file: artifacts must exist on disk, sections can't be one-liners, "looks good" is not an output.

**For your harness:** every phase output must produce a concrete artifact — not a summary of what was considered, but the actual output (ASCII diagram, test plan, failure modes registry, spec compliance table). If the artifact doesn't exist, the phase didn't run.

---
## Autoplan → Your Harness Mapping

|Autoplan component|Your Milestone 6 equivalent|
|---|---|
|6 Decision Principles|Autonomy engine for task execution decisions|
|Decision classification (Mechanical/Taste/User Challenge)|Three-tier escalation model|
|Restore point (Phase 0)|Pre-execution snapshot before any file writes|
|Sequential phases with verification|Specify → Plan → Task → Implement pipeline|
|Conditional phase execution|Preamble tier by task type|
|Completeness checklist before gate|Pre-ship artifact verification|
|Dual voices → cross-phase themes|Spec compliance check + code memory check as independent signals|
|Final Approval Gate|Human-in-the-loop checkpoint for Milestone 6|
|`max 2 retry / max 3 revision cycles`|Circuit breakers at phase and plan level|
|Audit trail (every auto-decision logged)|Milestone 5 decision memory|
|Review log writes on completion|Timeline JSONL entries per phase|

---

## What autoplan reveals that the preamble files didn't

**The harness orchestration model is sequential, not parallel at the phase level.** Specialist subagents run in parallel within a phase (the review army), but phases are strictly sequential. This is the right design for your harness: you can parallelize within implementation (multiple file edits by subagents), but specify → plan → implement → validate must be sequential because each depends on the previous.

**The audit trail is the accountability layer.** Every auto-decision gets a row. This is what makes an autonomous system auditable after the fact — not just "what did it produce" but "why did it choose this over that." For your GIS360 harness, this audit trail also feeds your Milestone 5 cross-session learning: patterns in which principles get applied most often to GIS360 tasks inform your decision principle weighting.

**The gap your harness fills that autoplan doesn't:** autoplan operates on a plan document. Your harness needs to _create_ the plan from a natural language requirement — the four-phase pipeline (specify, plan, task, implement) from Milestone 2. Autoplan assumes the plan exists. Your harness starts one step earlier.

---

## What office-hours actually does

It converts a raw requirement (or idea) into a structured, approved design document that downstream skills consume automatically. It's the harness's **specify phase** in your four-phase pipeline. The HARD GATE is the most important line in the file:

> **"Do NOT invoke any implementation skill, write any code, scaffold any project, or take any implementation action. Your only output is a design document."**

This enforces the separation your harness needs between specification and implementation. No code gets written until the design doc is APPROVED. The approval gate in Phase 5 (A/Approve, B/Revise, C/Start over) is your spec sign-off checkpoint.

---

## Five Patterns Worth Extracting

### 1. The Six Forcing Questions → GIS360 Requirement Forcing Questions

The startup mode questions are Garry's specific YC lens (demand reality, status quo, desperate specificity, narrowest wedge, observation, future-fit). These are well-designed but not directly applicable to your context — you're not validating a startup idea, you're clarifying a development requirement.

**What's extractable is the structure, not the questions.** The forcing question pattern:

```
One question at a time via AskUserQuestion
Push until: specific, evidence-based, uncomfortable
Red flags: vague, hypothetical, category-level answers
After first answer: check language precision + hidden assumptions + real vs hypothetical
```

**For GIS360, your six forcing questions might be:**

```
Q1: Scope Reality
"What files or modules will this change touch? 
Name them. If you don't know yet, that's important."

Q2: Acceptance Criteria
"How will we know this is correct? What does passing look like — 
a test, a visual state, an API response, a user action?"

Q3: Regression Risk  
"What currently-working behavior could this break? 
What existing tests cover the adjacent code?"

Q4: Data Dependencies
"What pipe data, spatial data, or work correction state 
does this touch? Does it need a migration?"

Q5: Narrowest Version
"What's the smallest change that delivers the core value?
What can be deferred to a follow-up?"

Q6: Edge Cases
"What happens with empty geometry, null values, 
expired work corrections, or missing MVT tiles?"
```

Push until each answer is specific. "Various files" is not Q1. "It should work" is not Q2.

---

### 2. The Premise Challenge → Spec Validation Gate

Phase 3 is the most transferable pattern:

```
1. Is this the right problem? (different framing = simpler solution?)
2. What happens if we do nothing? (real pain or hypothetical?)
3. What existing code already partially solves this?
4. If it's a new artifact: how will it be distributed/deployed?
5. Does the diagnostic evidence support this direction?
```

Output as explicit PREMISES the requester must agree with before implementation starts:

```
PREMISES:
1. The pipe tracking query is the bottleneck, not the rendering layer — agree/disagree?
2. This can be implemented without a schema migration — agree/disagree?
3. The existing spatial utility functions can be reused — agree/disagree?
```

**Why this matters for your harness:** premise disagreement is the earliest possible detection of a misunderstood requirement. If the agent's premise about the data model is wrong, catching it here costs a 30-second clarification. Catching it after implementation costs a full rewrite. This is your Milestone 2 spec compliance problem solved at the intake layer.

---

### 3. Mandatory Alternatives with Effort/Risk Scoring

Phase 4 is non-negotiable:

```
APPROACH A: Minimal viable
  Effort: S | Risk: Low | Reuses: [existing patterns]
  
APPROACH B: Ideal architecture  
  Effort: L | Risk: Med | Reuses: [minimal]
  
APPROACH C: Creative/lateral (optional)
  Effort: M | Risk: Low | Reuses: [unexpected reuse]
```

Rules that matter:

- **One must be minimal viable** (fewest files, ships fastest)
- **One must be ideal architecture** (best long-term)
- **One can be creative/lateral** (unexpected framing)
- **STOP after presenting approaches** — do not proceed until user selects one

The `Reuses:` field is specifically valuable for GIS360 — it forces the agent to search for existing spatial utilities, pipe query helpers, and MVT rendering patterns before proposing new ones. Your DRY principle from autoplan's 6 decision principles, applied at spec time.

**Combined with autoplan's AUTO_DECIDE:** if running in autonomous mode, the agent applies decision principle 1 (completeness) to select the approach automatically, logs it as a Taste decision, and surfaces it at the final gate.

---

### 4. The Design Doc as Machine-Readable Spec

The design doc template is what makes the whole pipeline work. Every downstream skill reads it automatically. The key sections:

```markdown
## Problem Statement       ← what we're solving
## Demand Evidence         ← why it matters (startup) / acceptance criteria (GIS360)
## Status Quo              ← current behavior / existing code
## Target User & Narrowest Wedge  ← scope boundary
## Premises                ← validated assumptions
## Cross-Model Perspective ← second opinion (if ran)
## Approaches Considered   ← the alternatives
## Recommended Approach    ← selected approach with rationale
## Open Questions          ← unresolved items
## Success Criteria        ← how we know it's done
## Distribution Plan       ← how it gets deployed
## Dependencies            ← blockers and prerequisites
## The Assignment          ← concrete next action
```

**For your GIS360 harness**, adapt the template:

```markdown
## Problem Statement
## Acceptance Criteria          ← replaces "Demand Evidence"
## Current Behavior             ← replaces "Status Quo"  
## Scope Boundary               ← files, modules, data touched
## Premises                     ← validated assumptions
## Approaches Considered
## Recommended Approach
## Open Questions
## Success Criteria
## Regression Risk
## Data Dependencies
## The Assignment               ← first implementation task
```

**The `Supersedes:` field** creates a design lineage chain. When the same feature is revisited, the new doc references the old one. For GIS360, this is your architectural decision record — you can trace how the understanding of a feature evolved across multiple work sessions.

**Downstream discoverability** is the key infrastructure point:

```bash
# office-hours writes to:
~/.gstack/projects/$SLUG/$USER-$BRANCH-design-$DATETIME.md

# plan-eng-review, autoplan discover it via:
ls -t ~/.gstack/projects/$SLUG/*-design-*.md | head -1
```

No explicit handoff. The design doc is written to a known location and downstream skills find it automatically. For your harness: every phase writes artifacts to `projects/{repo}/{branch}/`, every subsequent phase reads from that location. The pipeline is implicit in the file system structure.

---

### 5. The Anti-Sycophancy Rules → Agent Requirement Validation Posture

The startup mode rules are written for Garry's YC partner persona, but the underlying principle is directly applicable:

```
Never say: "That's an interesting approach"
Never say: "There are many ways to think about this"
Never say: "That could work"

Always: take a position AND state what evidence would change it
Always: challenge the strongest version of the claim, not a strawman
```

Applied to your harness's specify phase: when a requirement is vague, the agent should not accept it and proceed — it should name the vagueness specifically and push for precision.

```
BAD: "Okay, I'll implement the pipe tracking improvement."
GOOD: "The requirement says 'improve performance' but doesn't specify: 
       is this query time, render time, or data load time? 
       Each has a different solution. I need a specific metric and target."
```

The `One question at a time` rule is equally important — batching clarifying questions overwhelms the requester and signals the agent doesn't know what it needs most. Ask the blocking question first. If that's answered, ask the next one.

---

## What office-hours reveals about the full pipeline

Reading it alongside autoplan, the complete requirement-to-implementation flow becomes clear:

```
office-hours (specify)
├── Context gathering (codebase + existing designs)
├── Mode detection (what kind of work is this?)
├── Forcing questions (one at a time until specific)
├── Premise challenge (validate assumptions)
├── Alternatives generation (minimal / ideal / creative)
├── [USER SELECTS APPROACH]
├── Design doc written to disk
└── [USER APPROVES]
        ↓
autoplan (plan)
├── Reads design doc automatically
├── CEO review (scope, strategy)
├── Eng review (architecture, test plan, failure modes)
├── Auto-decides using 6 principles
├── Surfaces taste decisions + user challenges
└── [USER APPROVES]
        ↓
[implement]
├── Continuous checkpointing
├── Specialist review army
├── Test failure triage
└── Spec compliance check (your addition)
        ↓
ship
└── Squash WIP → clean PR
```

**The gap your harness fills:** office-hours is interactive — it asks questions. Your harness needs to work from structured requirements (GitHub issues, JIRA tickets, PRD sections) rather than conversational input. But the **phases** are the same: context gathering → premise challenge → alternatives → approach selection → design doc → implementation. The forcing questions become extraction queries against the requirement document rather than interactive prompts.

---

## Full gstack Analysis Complete

Now that we've read all the preamble generators, the review army, autoplan, and office-hours, here's the synthesis of what gstack gives you vs what your harness needs to build:

**Directly extractable (high fidelity):**

- Preamble session initialization block
- Tiered preamble by task type
- Continuous checkpoint commit format with `[gstack-context]`
- Test failure triage protocol (branch diff → classify → triage)
- Review army specialist architecture (JSON finding output)
- Fix-First heuristic (auto-fix vs ask)
- Decision brief format (AskUserQuestion)
- Structured completion states (DONE/BLOCKED/NEEDS_CONTEXT)
- Premise challenge phase
- Mandatory alternatives with effort/risk
- Design doc as downstream-discoverable artifact
- Audit trail for every auto-decision
- Per-project suppression history for review findings

**Adapt for GIS360:**

- Six forcing questions → GIS360 requirement forcing questions
- Decision principles 1-6 → GIS360-weighted autonomy engine
- Design doc template → GIS360 spec template
- Confusion protocol categories → GIS360 high-stakes categories

**Build from scratch (gstack doesn't have it):**

- Product knowledge layer (Milestone 3)
- Spec compliance verification against acceptance criteria (Milestone 2)
- Hybrid retrieval for code memory (Milestone 4)
- Cross-session failure pattern aggregation (Milestone 5)
- Requirements extraction from structured documents (replaces interactive office-hours)

---
## `investigate/SKILL.md` — The Root Cause Protocol

### Structure

Five phases: Root Cause Investigation → Pattern Analysis → Hypothesis Testing → Implementation → Verification & Report

### The Iron Law

```
NO FIXES WITHOUT ROOT CAUSE INVESTIGATION FIRST.
Fixing symptoms creates whack-a-mole debugging.
```

This is the behavioral directive that prevents your harness from entering infinite retry loops. When a test fails or spec compliance check fails, the agent doesn't immediately try another fix — it investigates first.

---

### What's directly extractable

**Scope Lock after hypothesis formation:**

```bash
echo "<detected-directory>/" > $STATE_DIR/freeze-dir.txt
```

After forming a root cause hypothesis, the agent locks edits to the narrowest directory containing the affected files. This prevents the fix attempt from accidentally touching unrelated code. Combined with `REPO_MODE: collaborative`, this is your scope boundary enforcement — automatically scoped to the module being debugged.

**Pattern Analysis table:**

|Pattern|Signature|Where to look|
|---|---|---|
|Race condition|Intermittent, timing-dependent|Concurrent access to shared state|
|Nil/null propagation|NoMethodError, TypeError|Missing guards on optional values|
|State corruption|Inconsistent data, partial updates|Transactions, callbacks, hooks|
|Integration failure|Timeout, unexpected response|External API calls, service boundaries|
|Configuration drift|Works locally, fails in staging/prod|Env vars, feature flags, DB state|
|Stale cache|Shows old data, fixes on cache clear|Redis, CDN, browser cache|

For GIS360, extend this table with domain-specific patterns:

- **Spatial reference mismatch** — geometry operations failing because CRS doesn't match
- **Tile cache invalidation** — MVT tile showing stale pipe data after correction
- **Work correction state machine** — correction applied but downstream status not updated
- **Geometry precision** — floating point comparison failures on coordinate equality

**The 3-Strike Rule → circuit breaker:**

```
3 hypotheses tested, none match. STOP.
Options:
A) Continue — I have a new hypothesis: [describe]
B) Escalate — this needs someone who knows the system
C) Add logging and wait — instrument and catch it next time
```

This is your Milestone 5 circuit breaker, concrete and implemented. Three failed hypothesis attempts → mandatory human escalation. Option C (instrument and wait) is particularly valuable for intermittent bugs — the agent instruments rather than guessing.

**The Red Flags (anti-pattern detection):**

```
"Quick fix for now" → there is no "for now." Fix it right or escalate.
Proposing a fix before tracing data flow → you're guessing.
Each fix reveals a new problem elsewhere → wrong layer, not wrong code.
```

These are behavioral guardrails. For your harness: encode these as STOP conditions. If the agent says "for now" in its own reasoning trace, that's a signal to re-examine.

**Blast radius gate:**

```
Fix touches >5 files → AskUserQuestion about blast radius
A) Proceed — root cause genuinely spans these files
B) Split — fix critical path now, defer the rest
C) Rethink — more targeted approach exists
```

Five files is the threshold before mandatory human check-in. For GIS360, this maps well — a bug fix touching more than 5 files is almost certainly either a symptom fix or an architectural issue.

**Regression test requirement:**

```
Write a regression test that:
- FAILS without the fix (proves the test is meaningful)
- PASSES with the fix (proves the fix works)
```

This is the strongest version of test-driven debugging. The two-condition requirement prevents tests that vacuously pass. For your Milestone 1 testing evaluation: this is the spec-compliance verification applied to bug fixes.

**Structured debug report:**

```
SYMPTOM: what the user observed
ROOT CAUSE: what was actually wrong
FIX: what was changed, file:line
EVIDENCE: test output, reproduction confirmation
REGRESSION TEST: file:line of new test
RELATED: TODOS items, prior bugs in same area
STATUS: DONE / DONE_WITH_CONCERNS / BLOCKED
```

This report format feeds your timeline JSONL and learnings log. The `RELATED` field is especially valuable — it surfaces architectural patterns when the same area has recurring bugs.

**Investigation learning log:**

```bash
gstack-learnings-log '{
  "skill": "investigate",
  "type": "investigation",
  "key": "ROOT_CAUSE_KEY",
  "insight": "ROOT_CAUSE_SUMMARY",
  "confidence": 9,
  "source": "observed",
  "files": ["affected/file1.ts", "affected/file2.ts"]
}'
```

The `files` array is the key addition here — it wasn't in the other learning log calls. It enables future investigation searches to find prior bugs in the same files. For your GIS360 harness: when the agent investigates a bug in `pipe-tracking/geometry-validator.ts`, that file gets tagged in the investigation log. Next time a bug occurs in the same file, the preamble's learnings search surfaces the prior investigation automatically.

**GBrain context queries:**

```yaml
prior-investigations: filter type: timeline, content_contains: "investigate", limit: 5
project-learnings: learnings.jsonl, tail: 10
recent-eureka: eureka.jsonl, tail: 5
```

Prior investigations are retrieved by timeline search, not just keyword search. This means if the same module was investigated before, that context surfaces even if the current bug has different symptoms.

---
## `retro/SKILL.md` — The Measurement and Learning Loop

### What it is

A comprehensive weekly engineering retrospective that turns git history into structured metrics, trend data, and per-person feedback. It's the observability layer for your harness.

---

### What's directly extractable

**The metrics schema** is the most immediately applicable thing:

```json
{
  "commits": 47,
  "contributors": 3,
  "insertions": 3200,
  "deletions": 800,
  "test_ratio": 0.41,
  "active_days": 6,
  "sessions": 14,
  "deep_sessions": 5,
  "avg_session_minutes": 42,
  "loc_per_session_hour": 350,
  "feat_pct": 0.40,
  "fix_pct": 0.30,
  "peak_hour": 22,
  "ai_assisted_commits": 32
}
```

This is exactly your Milestone 1 baseline measurement dataset, automatically computed from git history. The `ai_assisted_commits` field (from `Co-Authored-By` trailers) is directly relevant — for your GIS360 harness, you'll want to track which commits were agent-produced vs human-produced, and what the quality difference looks like.

**Session detection with 45-minute gap threshold:**

```
Sessions detected via 45-minute gap between consecutive commits
Deep sessions: 50+ min
Medium: 20-50 min
Micro: <20 min (single-commit fire-and-forget)
```

For your harness performance measurement: session depth correlates with task complexity. Milestones 3-5 knowledge systems should reduce the number of micro-sessions (context-building commits) and increase deep sessions (sustained implementation).

**Hotspot analysis → architectural signal:**

```
Files changed 5+ times = churn hotspot
Recurring bugs in same files = architectural smell
```

The retro surfaces churn hotspots automatically. For GIS360, when `pipe-tracking/geometry-validator.ts` appears in 8 of the last 10 weekly retros' hotspot lists, that's a signal for Milestone 3 to specifically encode product knowledge for that module — it's an area the agent clearly struggles with.

**Fix ratio flag:**

```
If fix ratio exceeds 50% → flag as "ship fast, fix fast" pattern
→ may indicate review gaps
```

This is your Milestone 6 harness quality metric. If your autonomous system produces >50% fix commits, the validation layer (review army + spec compliance) isn't catching issues before implementation ships.

**Test health tracking:**

```json
"test_health": {
  "total_test_files": 47,
  "tests_added_this_period": 5,
  "regression_test_commits": 3,
  "test_files_changed": 8
}
```

Track regression test commits specifically (`test(qa):` prefix). Over time this measures whether your harness is improving test coverage or just shipping features without tests.

**Trend comparison between retros:**

```
                Last      Now       Delta
Test ratio:     22%  →   41%       ↑19pp
Fix ratio:      54%  →   30%       ↓24pp (improving)
LOC/hour:       200  →   350       ↑75%
Deep sessions:   3   →    5        ↑2
```

This is the measurement infrastructure your Milestone 6 benchmarks need. Run the retro before and after each milestone to measure improvement. `LOC/hour` measures throughput. `fix_ratio` measures quality. `deep_sessions` measures agent effectiveness on complex tasks.

**The `retro-context.md` pattern:**

```bash
[ -f ~/.gstack/retro-context.md ] && cat it → incorporate into narrative
```

User-authored non-git context (meeting notes, decisions, calendar events) gets merged into the retro. For your GIS360 harness, this is where you'd put sprint context — what requirements were assigned this week, what business priorities changed, what architectural decisions were made outside of code.

**AI-assisted commit tracking:**

```
Co-Authored-By trailers → count AI-assisted commits separately
Never include AI co-authors as team members
Track "AI-assisted commits" as separate metric
```

This is your harness attribution tracking. When Claude generates code that gets committed, the `Co-Authored-By: Claude <noreply@anthropic.com>` trailer marks it. The retro then tracks what percentage of total commits were AI-assisted, which is your Milestone 6 autonomy metric.

**The Global Retro mode** — cross-project analysis — is relevant if you run multiple GIS360-related repos. The `gstack-global-discover` binary finds all repos with AI coding sessions across all tools. For your harness measuring purposes, the per-repo breakdown shows where the harness is most effective vs where it's still struggling.

---
## Batch Summary

### investigate patterns to extract:

|Pattern|Harness application|
|---|---|
|Iron Law: investigate before fixing|Prevents retry-without-root-cause loops (Milestone 5)|
|Scope lock to narrowest directory|File scope enforcement during debugging|
|Pattern analysis table|GIS360-extended failure pattern recognition|
|3-strike rule → escalate|Circuit breaker (Milestone 5)|
|Blast radius gate at 5 files|Pre-fix scope check|
|Regression test: fail without / pass with|Strong test validation requirement|
|Structured debug report|Machine-readable investigation output|
|Investigation log with `files` array|Cross-session file-scoped memory (Milestone 5)|

### retro patterns to extract:

|Pattern|Harness application|
|---|---|
|JSON metrics schema|Milestone 1 baseline measurement dataset|
|Session detection (45-min gap)|Task complexity measurement|
|Hotspot analysis → architectural signal|Knowledge system prioritization (Milestones 3-4)|
|Fix ratio >50% flag|Harness quality metric (Milestone 6)|
|Test health tracking (regression commits)|Coverage improvement measurement|
|Trend comparison between retros|Milestone-to-milestone improvement measurement|
|AI-assisted commit tracking|Harness autonomy metric|
|`retro-context.md` non-git context|Sprint context injection|

---

## Your Harness Blueprint — Extracted from gstack

### Layer 1: Session Initialization (from preamble generators)

```
SESSION_ID = PID + timestamp
BRANCH, TASK_TYPE, HARNESS_MODE (autonomous/supervised)
Knowledge system health: PRODUCT_MEM, CODE_MEM, LEARNINGS
Checkpoint mode: continuous
Spawned session flag: SPAWNED_SESSION
Top-3 relevant learnings injected automatically
Timeline event: skill started
```

### Layer 2: Specification Phase (from office-hours)

```
Context gathering (codebase + prior designs on this branch)
GIS360 six forcing questions (one at a time)
Premise challenge (agree/disagree gate)
Mandatory alternatives (minimal / ideal / creative)
[USER or AUTO_DECIDE selects approach]
Design doc written to projects/{repo}/{branch}/
Downstream skills discover it automatically
```

### Layer 3: Planning Phase (from autoplan)

```
Restore point snapshot
Sequential phases: Product → Architecture → Test Plan
6 Decision Principles for auto-decisions
Three-tier decisions: Mechanical (silent) / Taste (gate) / User Challenge (never auto)
Dual-signal validation (spec compliance + code memory as independent checks)
Completeness checklist before final gate
Audit trail: every auto-decision logged
```

### Layer 4: Implementation (from continuous-checkpoint + scope enforcement)

```
Continuous WIP commits: [gstack-context] body
Decisions/Remaining/Tried fields per commit
Scope locked to blast radius (REPO_MODE enforcement)
Never git add -A
Progress summaries: done / next / surprises
Loop detection: same file/same diagnostic = STOP
```

### Layer 5: Validation (from review army + test failure triage)

```
Test failure triage:
  git diff → classify in-branch vs pre-existing → STOP on in-branch
Pre-landing review (two-pass: critical → informational)
Fix-First heuristic: AUTO-FIX or ASK
Parallel specialist subagents:
  maintainability + testing (always)
  security + performance (conditional)
  data-migration (if schema changed)
  GIS360 spatial specialist (new)
Spec compliance check (product knowledge → acceptance criteria)
```

### Layer 6: Completion (from completion-status + investigate + retro)

```
Debug report if fix required:
  symptom / root cause / fix / evidence / regression test
3-strike circuit breaker: STOP → escalate
Operational learning log (if durable insight found)
Timeline: skill completed, outcome, duration
Weekly retro:
  metrics JSON snapshot
  trend comparison vs prior week
  AI-assisted commit count
  fix ratio, test ratio, hotspot analysis
```

### What gstack doesn't provide (your genuine additions):

1. **Product knowledge layer** — requirements, design intent, business rules (Milestone 3)
2. **Spec compliance verification** — does output match acceptance criteria? (Milestone 2)
3. **Hybrid code retrieval** — graph + semantic + keyword (Milestone 4)
4. **Cross-session failure pattern aggregation** — Reflexion/ExpeL layer (Milestone 5)
5. **Requirements extraction from structured documents** — replaces interactive office-hours
6. **GIS360 domain specialists** — spatial, MVT, pipe tracking, work correction