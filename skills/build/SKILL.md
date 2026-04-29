---
name: build
description: Build process orchestrator — scope, design, plan, review, build, verify, ship, and self-improve
---

# Build — Skill Definition

## Purpose

The build process orchestrator manages the full lifecycle of Feature/System work: designed before built, reviewed before approved, verified before shipped, improved over time.

## Trigger

- The user says `/build`, "let's build something", "new build", or similar
- Chief recognizes that a request is Feature/System tier and proposes: *"This looks like a [tier]-tier build — want me to kick off /build?"*
- The user says `/build [name]` to resume an active build
- The user says `/build [name] polish/extend/fix/cancel` for iteration

Chief should never start coding on a Feature or System without engaging this process. Config and Skill tier work can proceed without `/build` if the scope is clear.

---

## Scope Tiers

| Tier | What it is | Examples | Phases |
|---|---|---|---|
| **Config** | Rule, setting, or convention change | CLAUDE.md rule, permission update, tag convention | Just do it → document. No state file. |
| **Skill** | New skill, reference file, self-contained capability | a note-creation skill, a code formatter | Scope → Design → Design Review → Build → Smoke Test → Ship → Retrospective |
| **Feature** | New capability wired into existing systems | an OAuth flow, a webhook handler, a CSV importer | Scope → [Target Audit] → Design → Plan → Design Review → Build → Verify → Ship → Retrospective |
| **System** | Multi-component build from scratch | a task management dashboard, a multi-stage data pipeline, this skill | Scope → [Target Audit] → Design → Architectural Review → Plan → Implementation Review → Build → Verify → Ship → Monitor → Retrospective |

*Note on review phase names:* "Design Review," "Architectural Review," and "Implementation Review" all use the same review agent (defined in Agent Prompts). The different names indicate which artifact is being reviewed — Design (Architectural) vs Plan (Implementation) — not three separate mechanisms.

Chief proposes tier during Scope. The user confirms or overrides. Tier can be upgraded or downgraded mid-build (see Iteration Paths).

---

## Orchestrator Routing

### Entry Routing

```
/build invoked
  |
  +-- No argument, no active builds → Scope phase (new build)
  |
  +-- No argument, 1 active build → Resume that build
  |
  +-- No argument, multiple active builds → List them, ask which one
  |
  +-- Named argument matches existing state file → Resume or iterate
  |
  +-- Named argument, no match → Scope phase (new build, pre-fill name)
  |
  +-- Named argument + keyword (polish/extend/fix/cancel) → Iteration path
```

### Active Build Discovery

Scan `build-state-*.md` files in the project root. Active = Status not `Abandoned`/`Paused`, no `## Ship Log` (or System tier with active Monitor issues). If multiple, list them and ask which one.

### Phase Routing

Read the `Tier:` line, check section headings in order for that tier, route to the first missing section. All sections present → build complete. If a heading exists but starts with `[INCOMPLETE]`, the phase is partially done — ask the user to resume or start fresh.

---

## Phases

### Scope

**Tiers:** All except Config

**What happens:** A short progressive interview to capture the build's intention and assign a tier.

**Scoping questions (progressive — stop when tier is clear):**

1. **"What are we making?"** — Capture in a sentence.
2. **"Is this the right problem to solve?"** — *"What alternative framings did we consider? Are we inheriting a problem statement from a prior context that still makes sense, or is it worth reframing?"* The highest-leverage decisions often come from reframing, not from better execution within the current frame. Skip only if the problem is genuinely undebatable (e.g., "fix this specific broken thing").
3. **"Does this involve code, or just text/config?"** — Text/config → likely Config tier. Chief says: *"This looks like a config change — no build process needed. I'll just make the change and document it."*
4. **"Is this adding something new, or changing how something existing works?"** — Changing existing → Feature at minimum.
5. **"How many systems does this touch?"** — Self-contained → Skill. Multiple systems → Feature or System.
6. **"What breaks if this goes wrong? Who sees it?"** — Only us, reversible → lighter process. Data loss possible, visible to others → full process.

Chief proposes tier and explains why. The user confirms or overrides.

**State file creation:** After tier is assigned, create `build-state-[kebab-case-name].md` with the Scope section using this template:

```markdown
# Build: [Human-Readable Name]
Tier: [Skill | Feature | System]
Status: Active
Started: [YYYY-MM-DD]
Last updated: [YYYY-MM-DD]
Depends on: [none | build-state-X.md, build-state-Y.md]

## Scope
Date: [YYYY-MM-DD]

**What:** [1-2 sentence description]
**Why:** [What problem does this solve]
**Problem framing:** [What alternative framings were considered? Was this the right problem to solve, or an inherited framing? If no reframing surfaced, state "N/A — problem was clearly scoped."]
**Success criteria:**
- [Measurable criterion 1]
- [Measurable criterion 2]
- [Measurable criterion 3]

**Tier:** [tier] — [1-sentence justification]
**Constraints:**
- [Constraint 1]
- [Constraint 2]

**Touches:**
- [System/file 1]
- [System/file 2]

**Risk if it goes wrong:** [1 sentence]
**Security/privacy:** [Does this touch credentials, API keys, personal data, or auth? If yes, how are they managed? If no, state "N/A".]
```

**About `Depends on:`** — lists builds whose Ship must complete before this build's Build phase starts (e.g., backup pipeline before BD pipeline). Chief flags unmet dependencies at Build-phase start. Most builds: `Depends on: none`.

The Scope section must be comprehensive enough for a review agent to understand the full intention without conversation context. If it's not, that's a signal to improve it before proceeding.

---

### Target Audit

**Tiers:** Feature, System — **only when modifying existing skills or systems**

**What happens:** Before designing the new capability, audit the existing skill/system it will integrate into. The goal is to catch contradictions, stale instructions, or ambiguous flows BEFORE the new design gets layered on top.

**When to run:** If the Scope's "Touches" list includes modifying an existing skill file or wiring into an existing system. If the build is entirely new (no existing system to integrate into), skip this phase — but **still add a Target Audit section to the state file with "N/A — new build, no existing system" as the body** so phase routing doesn't get confused about whether the phase was attempted or just skipped.

**How:** Launch a background agent to read the target skill/system files and check for:
- Internal contradictions
- Dead or superseded instructions
- Stale references (file paths, API methods, tool names)
- Ambiguous flow that the new build would make worse
- Gaps that should be resolved before adding complexity

**After the agent returns:** Fix any issues found in the existing skill BEFORE starting Design. This prevents the new design from inheriting or compounding existing problems.

**Append to state file:**

```markdown
## Target Audit
Date: [YYYY-MM-DD]

**Target:** [skill/system being modified]
**Issues found:** [count]
**Fixed before Design:** [list of fixes, or "None needed"]
```

---

### Design

**Tiers:** All except Config

**Tier check before starting:** Is this still the tier we assigned in Scope, or has scope grown/shrunk enough to warrant Upgrade Tier / Downgrade Tier? One-line check at the top of Design.

**What happens:** Architecture, access patterns, integration points, failure modes. Depth scales with tier.

**Stay in design mode when the user is there.** When they're exploring options or asking "what do you think?", research references and discuss tradeoffs — don't jump to execution. Only ask "want me to build it?" once the design conversation has concluded.

**Deliberate over design docs.** Explain changes before editing. Design docs warrant more deliberation than code fixes.

**For Feature/System tiers:** Chief recommends entering plan mode before starting Design to prevent premature coding. Plan mode is a convention (not mechanically enforced) — if it's skipped, note that in the state file. The real approval gate is exiting plan mode before Build. See Plan Mode Protocol section for details.

**Skill tier Design covers:**
- Trigger: what invokes this
- Inputs: what data it reads, what files it loads
- Process: numbered steps
- Output: what it produces
- Edge cases: what could go wrong and how it's handled

**Feature/System tier Design covers all of the above, plus:**
- Architecture: how this fits into existing systems
- Access patterns: what operations, how often, risk if one goes wrong
- Integration points: what existing systems it connects to and how
- Failure modes table: failure, impact, mitigation
- Data flow: input → processing → output
- Open questions: resolved during design or flagged for the user

**System tier may additionally use research agents** (optional) to explore reference implementations, design patterns, or prior art. Research findings are summarized in a Research sub-section within Design.

**For builds with user-facing categories, labels, or data shapes:** do a "name the things" pass with the user before coding. Design the taxonomy / data model / vocabulary before building UI. Start with the user's mental model, not the technically precise one. Iterating these mid-build causes migrations and rework.

### Design Principles

These are the principles Chief designs to, and the same principles the review agent checks against. Applying them during Design reduces review-cycle churn.

- **Non-fragile:** What happens when a dependency is unavailable? Design for graceful degradation.
- **Simplicity over complexity:** Fewer moving parts unless removing them loses capabilities.
- **Safe access patterns:** Additive operations should be additive; destructive ones should be gated.
- **Failure is visible:** Breaks surface clearly, not silently.
- **Existing patterns preferred:** Match codebase conventions rather than inventing new ones.
- **Proportional to the problem:** Solution sized correctly — neither over- nor under-engineered.
- **Reversible where possible:** Mistakes can be undone.
- **Security:** Credentials in env vars (not tracked files), no personal data exposed, gitignore implications considered, auth follows established patterns.
- **Operational characteristics** *(especially for data-heavy or long-running builds):* performance tuning choices (journal mode, batch sizing, index strategy), resource budgets (time/memory/disk/thermal/bandwidth/rate limits), observability (progress visible mid-run? logs unbuffered?), long-running handling (checkpointing, resumability), failure recovery (what's the retry story?).

**Append to state file:**

```markdown
## Design
Date: [YYYY-MM-DD]

### Architecture
[How this fits into existing systems]

### Access Patterns
[Operations, frequency, risk]

### Integration Points
- [System 1]: [how it connects]

### Failure Modes
| Failure | Impact | Mitigation |
|---------|--------|------------|
| [failure] | [impact] | [mitigation] |

### Edge Cases
- [edge case and how handled]

### Open Questions
- [resolved or flagged]
```

Skill tier uses a lighter version (just Trigger, Inputs, Process, Output, Edge Cases).

---

### Plan

**Tiers:** Feature, System

**Tier check before starting:** Is this still the tier we assigned in Scope (or upgraded to during Design), or has planning revealed it's actually bigger/smaller? Upgrade/Downgrade Tier now if needed, before Plan locks in.

**What happens:** Break the design into numbered implementation steps. Stay in plan mode.

**The plan must be visible.** Not in Chief's head. Written out so the user can review and redirect before code is written.

**Append to state file:**

```markdown
## Plan
Date: [YYYY-MM-DD]

### Implementation Steps
1. [Step] — [files, changes, effort]
   - Dependencies: [none | step N]
   - Parallelizable: [yes | no]
2. [Step]
   ...

### File Inventory
Files to create: [list]
Files to modify: [list]
Files to read (reference): [list]

### Risk Checkpoints
- After step [N]: [what to verify]
```

**File inventory threshold:** If files being modified exceed what's listed in the File Inventory (unexpected files), pause and discuss with the user. The risk is touching unexpected files, not touching many files.

---

### Design Review

**Tiers:** Skill, Feature, System

**What happens:** Launch a background agent to review the design and/or plan with fresh eyes. The agent has NO conversation context — only the state file and rubric. This is intentional.

**When it runs:**
- Skill tier: once, after Design (reviews Design against Scope). Agent review is cheap and consistently catches real issues — a real skill-tier build proved this when the agent found a blocking CLAUDE.md gap that was invisible during heads-down building.
- Feature tier: once, after Plan (reviews both Design and Plan)
- System tier: twice — Architectural Review after Design, Implementation Review after Plan

**CRITICAL: Chief launches the Design Review automatically.** After completing the phase that precedes Design Review for this tier — **Design** for Skill tier, **Plan** for Feature/System tier — immediately launch the review agent as the next action. Do not wait for the user to ask. For Feature/System, this happens inside plan mode; findings are presented, incorporated, THEN plan mode is exited for approval.

**Naming note:** "Design Review" (Skill/Feature), "Architectural Review" (System, after Design), and "Implementation Review" (System, after Plan) all use the SAME review agent defined in Agent Prompts below. The different names describe WHICH ARTIFACT the review focuses on, not different mechanisms. The agent's rubric adapts per-phase.

**The review runs inside plan mode for Feature/System** (it's read-only; plan mode stays active until findings are incorporated). **Skill tier has no plan mode** — the review still runs read-only, and findings are incorporated before Build starts, but there's no plan-mode gate to exit. Findings are reviewed with the user and incorporated into the Design or Plan before Build begins.

**Re-review after substantial revisions.** If a review leads to significant changes (rescoping, failure-mode redesigns, plan restructuring), run another review before approving. Multiple review rounds are normal for non-trivial builds, not exceptional — three review rounds on a single build can catch real issues each time.

**Agent prompt — see Agent Prompts section below.**

**After the agent returns:**
1. Present findings to the user
2. For each finding, decide together: Accept / Reject / Defer
3. Incorporate accepted changes into the Design or Plan sections
4. Append the review results to the state file

**Append to state file:**

```markdown
## Design Review
Date: [YYYY-MM-DD]
Review type: [Architectural | Implementation | Combined]

### Agent Assessment
[Agent's full output]

### Disposition
| Finding | Decision | Rationale |
|---------|----------|-----------|
| [finding] | [Accepted/Rejected/Deferred] | [why] |

### Changes Made
- [Change incorporated into Design or Plan]
```

For System tier, the second review uses the heading `## Implementation Review`.

---

### Build

**Tiers:** All except Config

**What happens:** Write the code. Exit plan mode first (Feature/System).

**Exiting plan mode is the conscious gate.** It means: design is done, plan is approved, reviews are incorporated, time to execute. Chief should not exit plan mode until the user has seen the review results and approved.

**During Build:**
- Follow the plan step by step (Feature/System) or the design (Skill)
- Test incrementally — not "build everything then test at the end"
- If files being modified go beyond the File Inventory, pause: *"Scope is growing — we're touching files not in the plan. Continue or re-plan?"*
- Agents can parallelize independent build steps

**Append to state file:**

```markdown
## Build Log
Date started: [YYYY-MM-DD]

### Step 1: [name from Plan]
- Status: [Complete | In Progress | Blocked]
- Files changed: [list]
- Notes: [decisions, deviations]

### Emergent Decisions
[Decisions made during Build that weren't in the Plan. For each:
- **Context:** what situation prompted the decision
- **Options considered:** alternatives evaluated
- **Decision:** what was chosen
- **Rationale:** why, and any trade-offs accepted

Note: existing state files' "Deviation Log" sections remain as-is; new builds use this structure. Post-mortem Step 1 reads this section as primary chronology input at Ship.]
```

Skill tier (no Plan) uses free-form entries:

```markdown
## Build Log
Date: [YYYY-MM-DD]

- Created `[file]`: [what it does]
- Modified `[file]`: [what changed]
- [Decision]: [rationale]
```

---

### Smoke Test

**Tiers:** Skill only

**What happens:** Quick inline verification by Chief (not an agent). Does it run? Does it produce expected output? Does the obvious edge case work?

**Append to state file:**

```markdown
## Smoke Test
Date: [YYYY-MM-DD]

- [Test 1]: [PASS | FAIL] — [what was tested]
- [Test 2]: [PASS | FAIL]

Issues found:
- [Issue]: [resolved | deferred to backlog]
```

---

### Verify

**Tiers:** Feature, System

**What happens:** Launch a QA agent to test the build against everything specified in the state file.

**CRITICAL: Chief launches QA automatically after Build is complete.** When the last build step is done, immediately launch the QA agent as the next action — do not wait for the user to ask. Present findings when the agent returns.

**For builds that modified an existing skill or system, QA should spot-check internal consistency of the modified target.** E.g., grep for orphaned references, read linked sections as a cluster to confirm they agree, verify no heading-routing ambiguity remains. Target Audit catches issues BEFORE layering new design on top; this coherence check catches issues AFTER the edits land. Different failure modes — post-build coherence issues can be invisible to Design Review.

**Agent prompt — see Agent Prompts section below.**

**After the agent returns:**
- If PASS → proceed to Ship
- If CONCERNS → present to the user, decide together
- If FAIL → loop back to Build with specific issues. If failure reveals a design flaw, loop back to Design.

Re-verification appends a new dated sub-section (doesn't overwrite the original).

**Append to state file:**

```markdown
## Verification
Date: [YYYY-MM-DD]

### QA Agent Report
**Overall: [PASS | FAIL | CONCERNS]**

#### Success Criteria Check
| Criterion | Result | Notes |
|-----------|--------|-------|
| [from Scope] | PASS/FAIL | [detail] |

#### Issues
- [Issue]: [Critical | Minor]

### Disposition
[Proceed to Ship | Loop back to Build | Loop back to Design]
```

---

### Ship

**Tiers:** All except Config

**What happens:** Document what was built, update system files, run the post-ship checklist.

**Before shipping, review relevant open items in `backlog.md`** for post-mortem recommendations from prior builds that might apply to this one.

**Post-ship checklist (mandatory):**

- [ ] Project documentation updated (CLAUDE.md, README, technical notes — wherever your project keeps them)
- [ ] backlog.md updated (if known limitations)
- [ ] Security check: no credentials in tracked files
- [ ] Security check: gitignore updated if new sensitive files created
- [ ] Security check: no personal data in config or permissions
- [ ] State file status set to "Complete"
- [ ] **Ask the user: "Want me to commit the build files?"** If yes → stage specific build files (not `git add -A`), commit with descriptive message, remind them to push if they want remote backup. If no → note in Ship Log that commit was deferred; the user may commit manually or batch with other work later. Do NOT auto-commit without asking.

**Append to state file:**

```markdown
## Ship Log
Date: [YYYY-MM-DD]

### What was built
[1-3 sentence summary]

### Files created
- [file]: [purpose]

### Files modified
- [file]: [what changed]

### Documentation updated
- [doc]: [what was added/changed]

### Known limitations
- [limitation] — logged to backlog.md
```

**Archive:** After shipping, completed state files move to `build-archive/`.

**Completion signal (mandatory):** After the Ship checklist passes and the state file is archived, Chief explicitly closes the build:

*"✅ Build complete: [name]. Exiting build process."*

This is the "build mode off" signal — it tells the user the build is done and we're back to normal conversation. Don't skip it, even if the transition feels obvious.

---

### Monitor

**Tiers:** System only

**What happens:** Define health checks at ship time that catch degradation before the user hits it. This phase is **mandatory for System-tier builds** — not "define checks if you feel like it." A System-tier build without Monitor ships blind.

**Each health check must specify:**
1. **What gets monitored** — the specific thing being checked (e.g., "backup ran successfully," "API endpoint responds under 500ms," "row counts within expected range")
2. **Thresholds** — what counts as healthy vs degraded vs failed (e.g., "healthy = backup within last 48h; degraded = 48-96h stale; failed = >96h stale")
3. **Fix iteration trigger** — what level of degradation prompts action (e.g., "degraded for >3 consecutive days → Fix iteration")
4. **Check frequency** — when the check fires (e.g., "session start," "a scheduled cron," "manual invocation")

**Append to state file:**

```markdown
## Monitor
Setup date: [YYYY-MM-DD]

### Health Checks
- **[Check 1]:**
  - What: [what gets monitored]
  - Thresholds: healthy = [...] / degraded = [...] / failed = [...]
  - Fix trigger: [what prompts Fix iteration]
  - Frequency: [when check fires]
  - How to run: [specific command or invocation]

### Monitor Log
- [YYYY-MM-DD]: [All pass | Degraded: X | Failed: Y — description]
```

When a Monitor check fails or stays degraded past its Fix trigger, it feeds back as a Fix iteration.

---

### Retrospective

**Tiers:** All except Config

**What happens:** After shipping, Chief asks the user a single question: *"Want to run /post-mortem?"*

- **If yes** → invoke the `/post-mortem` skill. Its output drives improvements (skill edits, backlog entries) via post-mortem Step 6.
- **If no** → note the reason in the Ship Log (e.g., "straightforward build, no friction to dissect") and close the build.

This is how `/build` improves itself over time. The user decides per-build whether post-mortem is warranted — proportional to the build's weight and complexity.

---

## Plan Mode Protocol

Feature/System tiers: Chief recommends entering plan mode before Design, staying in it through Design Review. The exit is the gate — only exit after the review agent has run and findings have been incorporated. This is a convention, not mechanically enforced; if skipped, note in the state file. Real safety net is the review agent + the user's approval. Skill/Config tiers: no plan mode; the conversation is the thinking phase.

---

## Iteration Paths

### Polish
**Trigger:** "tweak X", "adjust the feel", "small change to X"
**Re-entry:** Build phase
**Process:** Change → smoke test → append to Build Log. No agent review. Lightest path.

### Extend
**Trigger:** "add Y to X", "X needs a new capability"
**Re-entry:** Design phase (scoped to the extension only)
**Process:** Design new piece → Plan → Design Review → Build → Verify → update Ship Log

### Fix
**Trigger:** "X is broken", "X isn't working right"
**Re-entry:** Verify phase (reproduce first — never guess-fix)
**Process:** Reproduce bug → Build fix → Verify fix resolves it + no regression

**Hardening-build discipline** (for Fix iterations that might reveal structural issues):
- **Step back before fixing.** Ask: *"Is this a symptom of a structural issue, or truly a surface-level fix?"* If you can't tell, that's itself a signal — slow down, audit the system, don't rush to a fix that papers over the real problem.
- **QA the existing system FIRST** (before designing the fix). The findings reveal what the fix should actually address.
- **Three-layer audit sequence:** Target Audit (symptoms) → big-picture design review (structural causes) → Plan (tactical). Each layer catches what the previous missed.

### Cancel
**Trigger:** "drop this build", "we're not doing X anymore"
**Process:** Set Status to `Abandoned` in header. Add brief reason. Archive state file to `build-archive/`.

### Upgrade Tier
**Trigger:** "this is bigger than we thought"
**Process:** Update Tier in state file header. Add any missing phase sections for the new tier. Continue from current position.

### Downgrade Tier
**Trigger:** "this is simpler than we thought"
**Process:** Update Tier in state file header. Skip remaining phases that don't apply to the new tier. Continue.

All iteration appends to the existing state file — never overwrites completed sections.

---

## State File Convention

- **Naming:** `build-state-[kebab-case-name].md`. On collision, add a disambiguator (e.g., `build-state-tasks-v2.md`).
- **Location:** the project root while active; `build-archive/` when completed or abandoned.
- **Header format:** see Scope section template.
- **Corruption recovery:** if a section heading exists but contents are malformed (session crash mid-write), treat as incomplete. Ask the user: resume from last complete section or start the phase fresh.

---

## Agent Prompts

### Design Review Agent

**Single agent, tier-specific focus.** Used across all three review phase names in the skill — "Design Review" (Skill/Feature tier), "Architectural Review" (System, after Design), and "Implementation Review" (System, after Plan). These are not three different mechanisms — they are three phase labels for the same agent, whose rubric adapts based on which artifact (Design or Plan) just completed.

**Review-agent ROI note.** For Feature and System-tier builds, assume the review agent will find real issues — budget for revision cycles, don't treat reviews as rubber stamps. Past builds consistently show agents catching gaps invisible to in-conversation review. If the agent returns "None identified" across all categories, that's useful signal — but it's the exception, not the default.

**Launch as a background agent. Pass the full state file contents in the prompt.**

```
You are reviewing a build design with fresh eyes. You have NO context about this project beyond what is in the state file below. This is intentional — your value is the outside perspective.

Read the full state file, then evaluate the most recent phase against the Scope section.

[IF ARCHITECTURAL REVIEW — after Design:]
Focus on the DESIGN section. Check for:
1. GAPS — Requirements in Scope not addressed by Design
2. FAILURE MODES — Are mitigations sufficient or hand-waving?
3. SCOPE CREEP — Complexity beyond what Scope requires
4. SIMPLER ALTERNATIVES — Fewer moving parts, same outcome?
5. MISSING EDGE CASES — Empty inputs, unavailable dependencies, unexpected state
6. EXISTING SYSTEM RISK — Could this break something that works?

[IF IMPLEMENTATION REVIEW — after Plan:]
Focus on the PLAN section against both Scope and Design. Check for:
1. DEPENDENCY ORDERING — Steps in the right order?
2. MISSING STEPS — Work implied by Design not in Plan?
3. TESTABILITY — Can each step be verified independently?
4. PARALLELIZATION — Independent steps correctly identified?
5. FILE INVENTORY — Matches what steps describe?
6. RISK CHECKPOINT PLACEMENT — In the right places?

Apply the Design Principles defined in the Design phase of this skill:
- Non-fragile: What happens when a dependency is unavailable?
- Simplicity over complexity: Fewer moving parts unless removing them loses capabilities.
- Safe access patterns: Additive operations should be additive.
- Failure is visible: Breaks surface clearly, not silently.
- Existing patterns preferred: Match codebase conventions.
- Proportional to the problem: Solution sized correctly.
- Reversible where possible: Mistakes can be undone.
- Security: Credentials in env vars, no personal data exposed, gitignore implications, auth follows established patterns.
- Operational characteristics (esp. data-heavy or long-running builds): performance tuning (journal mode, batch sizing, index strategy), resource budgets (time/memory/disk/thermal/bandwidth/rate limits), observability (progress visible? logs unbuffered?), long-running handling (checkpointing, resumability), failure recovery.

Output format (for each category below, write findings or "None identified" — do not pad with low-value observations):

### [Architectural | Implementation] Review

**Overall Assessment:** [1-2 sentences]

**Gaps Found:**
**Failure Mode Concerns:**
**Scope Creep Flags:**
**Simpler Alternatives:**
**Missing Edge Cases:**
**Risk to Existing Systems:**
**Plan Issues** (if implementation review):

**Spot anything we haven't considered? Any other suggestions?** [Open-ended observations]

**Recommendation:** [Proceed | Revise Design | Revise Plan | Rethink Approach]

Your recommendation is ADVISORY — the user makes the final decision.

--- STATE FILE ---
[paste full state file contents here]
--- END STATE FILE ---
```

### QA Verification Agent

**Launch as a background agent after Build is complete. Pass the full state file contents.**

```
You are a QA reviewer for a completed build. You have NO context beyond what is in the state file below. This is intentional — your value is objective verification.

Read the full state file. Verify that what was BUILT matches what was DESIGNED and meets the SUCCESS CRITERIA in SCOPE.

Your process:
1. Check each success criterion in Scope — does the Build Log demonstrate it was met?
2. Check each failure mode in Design — does the build handle it?
3. Check each edge case in Design — was it addressed?
4. Check the deviation log — were deviations justified? Do they introduce new risks?
5. Check file inventory — were files missed or unexpected files created?
6. If you can access the filesystem: verify files exist, read them, check they match described behavior
7. Security check: any credentials in tracked files? Personal data exposed?

Output format:

### QA Verification Report

**Overall: [PASS | FAIL | CONCERNS]**

#### Success Criteria Check
| Criterion | Result | Notes |
|-----------|--------|-------|
| [from Scope] | PASS/FAIL | [evidence] |

#### Failure Mode Coverage
| Failure Mode | Handled? | Notes |
|-------------|----------|-------|
| [from Design] | Yes/No/Partial | [detail] |

#### Issues
**Critical (must fix before Ship):**
- [issue]

**Minor (can defer to backlog):**
- [issue]

**Spot anything we haven't considered? Any other suggestions?**
[Open-ended observations]

**Recommendation:** [PASS | FAIL — fix issues listed above | CONCERNS — review with the user]

Your recommendation is ADVISORY — the user makes the final decision.

--- STATE FILE ---
[paste full state file contents here]
--- END STATE FILE ---
```

---

## Edge Cases

| Situation | Response |
|---|---|
| The user disagrees with tier | Override and adjust. The user's call. |
| Tier mismatch discovered mid-build | Upgrade or downgrade — update header, adjust phases. |
| Scope grows beyond File Inventory | Pause: "We're touching files not in the plan. Continue or re-plan?" |
| Agent returns "Rethink Approach" | Present to the user: accept / proceed anyway / discuss. Advisory, not a block. |
| Verification fails | Loop to Build with specific fixes. If design flaw revealed → loop to Design. |
| Multiple active builds | List and ask which one. If builds touch the same files, flag overlap. |
| Stale build (7+ days) | Nudge: "Build [name] hasn't been touched in [N] days. Resume, pause, or abandon?" |
| Session ends mid-phase | State file has progress. `[INCOMPLETE]` marker. Resume next session. |
| Config reveals itself as bigger | Offer to create a state file and upgrade tier. |
| Agent dispatch fails | Inline review using same checklist. Note reduced value in state file. Flag to the user. |
| State file corruption | Detect malformed sections. Resume from last complete section. Ask the user. |

---

## Rules

Invariants that must hold across all phases. Other guidance (tier proposal, plan mode convention, checklist enforcement, etc.) lives in the relevant phase sections; this list is for the non-negotiables.

1. **State file is append-only.** Never overwrite completed sections. Iteration appends new dated sub-sections.
2. **Agents receive only the state file.** No conversation context. Fresh eyes by design.
3. **Known limitations always go to `backlog.md`** during Ship.
4. **The build process improves itself.** After every Ship (except Config), the user decides whether to run `/post-mortem`. Recommendations route via the post-mortem skill's Step 6 — skill edits applied directly OR backlog entries with backlinks. Process changes to `/build` itself are Polish iterations on this skill.
5. **Announce phase transitions during Build.** When moving between Plan steps or Build passes, Chief narrates explicitly: *"Step N complete, moving to Step N+1."*
