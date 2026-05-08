---
date: 2026-05-07
tags: [framework, bmad, agent-harness, flow]
source: ""
status: developing
---

# BMAD — Flow of Development

The full 9-phase flow as I've reconstructed it. Companion to [[BMAD - Summary]].

---

## The Flow

```
INTAKE → CONTEXT → SPEC → VALIDATE → IMPLEMENT → REVIEW → PRESENT → DECIDE
```

---

## Phase 1: Intake & Routing

**Source:** quick-dev step-01

The first question is always: what kind of work is this?

**Check in this order:**

1. Is there an existing spec with a status? Route directly to the right phase.
2. Is this an epic story? Load or compile the epic context file.
3. Is this freeform? Load relevant planning artifacts selectively.

**Multi-goal check:** Does the intent contain more than one independently shippable deliverable? If yes, halt and force a split decision before anything else. Deferred goals go to `deferred-work.md`.

**Route to one of two paths:**

```
Zero blast radius (no plausible unintended consequences)
    → One-shot path (faster, lighter review)

Everything else
    → Plan-code-review path (full 5-phase flow)
```

**Blast radius is not LLM judgment alone.** For your harness, compute it: count callers of changed functions, count modules touched, count boundary crossings. Use `analyze_sources.py` logic as a model. If any metric exceeds threshold, force the full path.

---

## Phase 2: Context Loading

**Source:** distillator + generate-project-context + compile-epic-context

Before writing a single line of spec, load the right context. The order matters:

```
Always loaded (every task):
  project-context.md          ← unobvious rules, tech stack, anti-patterns

Loaded for epic stories:
  epic-N-context.md           ← compiled from PRD + arch + UX, cached
  previous done story         ← Code Map + Design Notes + Change Log only

Loaded for freeform:
  selective planning artifacts ← only what's relevant to this intent
```

**Cache invalidation:** Before loading `epic-N-context.md`, check if any planning artifact is newer than the cache. If stale, recompile via subagent using compile-epic-context rules: scope aggressively, describe by purpose not by source, target 800-1500 tokens, never quote verbatim.

**Context budget check:** Run token estimation (analyze_sources.py logic) before loading. If total context would exceed 15K tokens, compress first using distillator's 7-step pipeline. Never flood the context window with raw planning docs.

---

## Phase 3: Spec Generation

**Source:** quick-dev step-02 + spec-template + validate-prd

Generate the spec using this structure:

```
<frozen-after-approval>
  ## Intent
    Problem: one to two sentences
    Approach: one to two sentences

  ## Boundaries
    Always:    invariant rules
    Ask First: decisions requiring human approval
    Never:     out of scope + forbidden approaches

  ## I/O & Edge-Case Matrix
    | Scenario | Input | Expected Output | Error Handling |
</frozen-after-approval>

## Code Map          ← agent-populated: file paths + roles
## Tasks & Acceptance ← checkboxes + Given/When/Then ACs
## Spec Change Log   ← append-only, starts empty
## Design Notes      ← only when non-obvious
## Verification      ← CLI commands to confirm
```

**Codebase investigation:** Use subagents for exploration. Instruct them to return distilled summaries only — never raw file dumps. This prevents context snowballing.

**Token check:** Spec must stay 900-1600 tokens.

- Below 900: too ambiguous, keep writing
- Above 1600: split or accept risk (human decides)
- Above 1600 and human accepts: flag it at every subsequent step

**Self-review before human sees it:** Check against the Ready for Development standard:

- Every task has a file path and specific action
- Tasks ordered by dependency
- All ACs use Given/When/Then
- No placeholders or TBDs

---

## Phase 4: Spec Validation

**Source:** validate-prd adapted for specs (not PRDs)

Run these checks automatically before showing the human. Each check is isolated, auto-proceeds on pass, halts on failure:

|Check|What it catches|Severity threshold|
|---|---|---|
|Template variables|Unfilled `{placeholder}`|Any → Critical|
|Density|Filler phrases, hedging, wordiness|>5 → Warning|
|Measurability|ACs without Given/When/Then, vague quantifiers|>3 → Warning|
|Traceability|ACs that don't trace to Intent section (orphan ACs)|Any → Critical|
|Implementation leakage|HOW in ACs instead of WHAT|>2 → Warning|
|SMART scoring|Score each AC on Specific/Measurable/Attainable/Relevant/Traceable|<3 on any → Flag|
|Completeness|Code Map populated, Verification section present|Missing → Warning|

**Auto-fixes** (no human needed): template variables with obvious values, formatting issues, density violations.

**Human gate triggers**: orphan ACs (intent is unclear), measurability failures on critical paths, SMART score <3 on >30% of ACs.

Failures append to the validation report. The report travels with the spec through the rest of the flow.

---

## Phase 5: Human Checkpoint

**Source:** quick-dev step-02 CHECKPOINT 1

Present the spec with:

- Validation report summary (pass/warning/critical per check)
- SMART score table for ACs
- Token count
- Clickable path to the spec file

**Decision:**

- `[A] Approve` → re-read spec from disk, check for external edits, freeze the `<frozen-after-approval>` block, set status `ready-for-dev`
- `[E] Edit` → apply changes, re-run validation, return to checkpoint

**Once approved:** Everything inside `<frozen-after-approval>` is locked. The agent cannot touch it. Only the human can change it.

---

## Phase 6: Implementation

**Source:** quick-dev step-03

```
1. Capture baseline_commit → spec frontmatter (critical for diff-based review)
2. Set status: in-progress
3. Load context: files from spec frontmatter context: field
4. Hand spec to subagent for implementation
5. Self-check: every task [x] before leaving this phase
```

**Path formatting rule:** All markdown links in the spec use spec-file-relative paths (clickable in VS Code). All terminal output uses CWD-relative `path:line` format (clickable in terminal).

**No push, no remote ops during implementation.**

---

## Phase 7: Adversarial Review

**Source:** quick-dev step-04 + adversarial reviewer skills

Three reviewers in parallel, each with strictly different context:

```
Blind Hunter (bmad-review-adversarial-general)
  Gets:    diff only
  Method:  attitude-driven skepticism, find at least 10 issues
  Output:  markdown list
  Purpose: finds what spec-aware reviewers rationalize away

Edge Case Hunter (bmad-review-edge-case-hunter)
  Gets:    diff + codebase read access
  Method:  exhaustive path enumeration, two-pass validation
  Output:  structured JSON {location, trigger_condition, guard_snippet, potential_consequence}
  Purpose: finds unhandled boundary conditions in changed paths only

Acceptance Auditor (inline)
  Gets:    diff + spec + context docs listed in spec frontmatter
  Method:  check every AC, check every Always/Never boundary
  Output:  markdown list of violations
  Purpose: finds where implementation diverges from stated intent

Domain Reviewer (GIS360-specific — not in BMAD, you build this)
  Gets:    diff + GIS360 domain knowledge distillate
  Method:  check spatial precision, projection consistency, MVT tile validity,
           pipe geometry integrity, work correction chain
  Output:  structured JSON (same format as Edge Case Hunter)
  Purpose: finds domain-specific failures the generic reviewers miss
```

**Subagent isolation is mandatory.** Each reviewer gets exactly its specified context and nothing more. No cross-contamination. If subagents aren't available, generate review prompt files and halt for human to run in separate sessions.

**Deduplication + Classification:**

Merge all findings, deduplicate, then classify each:

```
intent_gap  → Root cause is inside <frozen-after-approval>
              Action: revert code, loop back to human to resolve intent
              
bad_spec    → Root cause is outside <frozen-after-approval>
              Action: before reverting — extract KEEP instructions
                      revert code
                      read Spec Change Log, respect all logged constraints
                      append change-log entry: finding + amendment + bad-state-avoided + KEEP instructions
                      re-run implementation (Phase 6), then re-run review (Phase 7)
              
patch       → Trivially fixable without human input
              Action: auto-fix immediately
              
defer       → Pre-existing issue not caused by this change
              Action: append to deferred-work.md
              
reject      → Noise
              Action: drop silently
```

**Cascade rule:** If `intent_gap` or `bad_spec` exist, `patch` and `defer` are moot — code is being re-derived anyway. Don't waste tokens processing them.

**Circuit breaker:** Loop counter increments on every `bad_spec` loopback. At 5, HALT and escalate to human. At this point, classify the failure type using the correct-course taxonomy:

```
Technical limitation discovered during implementation  → Direct Adjustment
New requirement emerged                                → Direct Adjustment or MVP Review
Misunderstanding of original requirements              → intent_gap → human resolves
Strategic pivot                                        → Major scope → PM + Architect
Failed approach requiring different solution           → Rollback or Direct Adjustment
```

Minor scope (fix within current plan) → agent handles. Moderate scope (backlog reorganization) → notify human. Major scope (fundamental replan) → HALT, escalate.

---

## Phase 8: Human Review

**Source:** checkpoint-preview

Present the change organized by concern, not by file:

```
1. Orientation
   → Intent summary (from spec, verbatim)
   → Surface area stats: files changed · modules touched · ~lines of logic
                         · boundary crossings · new public interfaces
   
2. Walkthrough
   → 2-5 concern groups (design intents, not file names)
   → Each concern: why this approach, what it achieves
   → Stops: path:line + ≤15 word framing
   → Human can "dig into [area]" for correctness-focused deep dive

3. Detail Pass
   → Risk spots ordered by blast radius (not diff order)
   → Risk taxonomy: [auth] [public API] [schema] [billing] [infra]
                    [security] [config] [spatial] [MVT] [pipe-data]
   → Spec Change Log entries surfaced (machine hardening findings)

4. Testing
   → Manual observations only (not duplicating automated tests)
   → For each: what to do, what to expect
   → GIS360 UI changes → Computer Use verification suggestions
   → GIS360 API changes → MCP test suggestions
```

**Early exit at any step:** if the human signals a decision, jump to the decision gate.

---

## Phase 9: Decision Gate

**Source:** checkpoint-preview step-05

Three options:

**Approve** → Generate Suggested Review Order (concern-based, entry point first, peripherals last), mark spec `done`, commit, open spec in editor.

**Rework** → Classify failure: approach wrong (loop to Phase 1) / spec wrong (loop to Phase 3, amend frozen block if intent_gap) / implementation wrong (loop to Phase 6).

**Discuss** → Open conversation, return to decision after.

---

## The Complete Flow

```
Phase 1: Intake & Routing
    ↓
Phase 2: Context Loading (always-loaded + selective + cache check)
    ↓
Phase 3: Spec Generation (subagent exploration → spec template → self-review)
    ↓
Phase 4: Spec Validation (8 automated checks, auto-fix what you can)
    ↓
Phase 5: Human Checkpoint (approve freezes intent, edit loops back)
    ↓
Phase 6: Implementation (baseline_commit → subagent → self-check)
    ↓
Phase 7: Adversarial Review (4 reviewers → deduplicate → classify → act)
    ↓ (bad_spec loops back to Phase 6, max 5 times)
    ↓ (intent_gap loops back to Phase 5)
    ↓ (circuit breaker → correct-course if >5 loops)
Phase 8: Human Review (concern walkthrough → risk spots → testing suggestions)
    ↓
Phase 9: Decision Gate (approve / rework / discuss)
    ↓
Done: committed, spec done, deferred items logged
```

---

## The State Machine

Every spec tracks status through this progression:

```
draft → ready-for-dev → in-progress → in-review → done
```

Status never regresses. If a `bad_spec` loop sends you back to implementation, status stays `in-progress`. If an `intent_gap` sends you back to planning, status goes back to `draft` only if the human amends the frozen block.

The Spec Change Log is append-only throughout. Every loopback adds an entry. By the time the spec reaches `done`, the Change Log is a complete audit trail of every mistake made and corrected.

---

## What BMAD gets right that you must keep

1. **Frozen/mutable spec split** — human intent is structurally protected from agent modification. Non-negotiable.
    
2. **Context isolation per reviewer** — each reviewer gets exactly its specified context. Non-negotiable.
    
3. **Append-only Spec Change Log** — prevents the agent from re-deriving the same bad state. Non-negotiable.
    
4. **Just-in-time step loading** — don't front-load context you won't need until later. Non-negotiable.
    
5. **Concern-based human review** — organize by design intent not by file. Non-negotiable.
    

---

## What BMAD gets wrong that you must fix

1. **Replace project-context.md flat file with the six-layer product knowledge system** — BMAD's entire knowledge layer is one file loaded on every task regardless of relevance. Your harness needs selective retrieval.
    
2. **Add the Domain Reviewer (GIS360-specific)** — none of BMAD's three reviewers know about spatial operations, MVT rendering, or pipe geometry. You need a fourth reviewer that does.
    
3. **Compute blast radius, don't estimate it** — BMAD's one-shot vs full-path routing relies on LLM judgment. Your harness should compute it from the dependency graph.
    
4. **Add staleness detection to all cached artifacts** — epic context, project context, distillates. BMAD has no invalidation mechanism.
    
5. **Wire the inner-to-outer loop** — when the circuit breaker trips (5 `bad_spec` loops), automatically classify the failure type and route to the correct-course equivalent. BMAD leaves this gap — the agent gets stuck at step-04 with no path forward.
    
6. **Cross-task learning** — the Spec Change Log prevents re-deriving the same bad state within a task. It doesn't prevent the same class of mistake on the next task. Your Dynamic Memory (Milestone 5) fills this gap.