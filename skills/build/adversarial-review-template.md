# Adversarial Review — Agent Prompt Template

Reusable template for launching adversarial Opus review agents against
System-tier production code. 

**When to use:**
- System-tier builds with production-facing code that will touch real
  data, run for extended periods, or ship to operators who depend on
  correctness.
- After applying round-1 fixes, to verify them AND find fresh issues —
  that's the round-2 template below.
- **Mid-operation emergent-bug investigation** — when a dry-run or canary
  surfaces a concurrency, runtime, or multi-process behavior that static
  code review wouldn't have caught. 

**When NOT to use:**
- Config-tier or trivial Skill-tier edits.
- Code that's already been exercised against real data and the concerns
  are about UX/ergonomics rather than correctness.
- When round-2 returned no CRITICAL and 0-1 MAJOR — stop there
  (diminishing returns, empirically supported).

**How to launch:** via the Agent tool with `subagent_type: general-purpose`
and `model: opus`. Pass the full prompt (filled-in template) via the
`prompt` parameter. The agent has NO conversation context beyond the
prompt and whatever files it reads — that's the point, fresh eyes.

---

## Round 1 — Initial adversarial review

Fill in the three bracketed sections before launching:
- `{{FILE_PATH}}` — absolute path to the file under review
- `{{FILE_SIZE}}` — approximate line count
- `{{CONTEXT_FILES}}` — bulleted list of files the agent should read to
  understand the design constraints (typically: build state file, any
  recipes or tech-notes the code depends on, sibling scripts it
  interacts with)
- `{{CATEGORY_RUBRIC}}` — the 6-12 category questions specific to THIS
  file's risk surface. See "Rubric categories" below for the
  language-level / domain-level / scope-level rubrics we've built up.

```
You are performing an adversarial code review with fresh eyes. You have
NO context about this project beyond what you read from files. This is
intentional — your value is the outside perspective.

**Mindset:** Pretend the code was written by a competent mid-level
engineer who got the basics right but missed something. Your job is to
find what they missed. Do not pad the review with generic observations
or low-value positives. Be specific, cite file:line, and focus on what
matters. If you find nothing CRITICAL, say so — don't manufacture
findings.

**File under review:** `{{FILE_PATH}}` (~{{FILE_SIZE}} lines)

**Context files (read as needed for design constraints):**
{{CONTEXT_FILES}}

**Adversarial rubric — check each category:**
{{CATEGORY_RUBRIC}}

**Output format (strict):**

## Adversarial Review: {{basename of FILE_PATH}}

**Overall Assessment:** [1 sentence — PASS / CONCERNS / FAIL + rationale]

### CRITICAL (must fix before [live run / canary / ship])
Each finding: `file:line` — description — why it's critical — recommended fix.
If none: state "None identified."

### MAJOR (should fix — fragile paths, observability gaps, silent-wrong-number risks)
Same format.

### MINOR (nice-to-have)
Same format.

### POSITIVE (max 3 bullets — only notable, non-obvious things done right)

### Questions for the author
Specific questions whose answers would change your assessment.

Cite file:line for every code finding. Your review is ADVISORY — the
shipping decision is the user's.
```

---

## Round 2 — Fix verification + fresh pass

Use this AFTER round-1 fixes have been applied. Two-part structure: Part
A verifies each round-1 finding was correctly addressed; Part B looks
at the current code with fresh eyes for regressions and new issues.

Additional placeholders beyond round 1:
- `{{ROUND_1_FINDINGS}}` — a list of round-1 findings with IDs
  (C1/C2/.../M1/M2/...) and brief fix-claim summaries. The agent will
  verify each one.
- `{{NEW_RUBRIC}}` — fresh questions specifically about whether the
  fixes introduced new bugs or interact poorly with each other.

```
You are performing a ROUND-2 adversarial code review on a file revised
in response to a round-1 review. Two parts:

**Part A — verify fixes:** For each round-1 finding, read the updated
code and confirm the fix actually addresses the issue. Flag incomplete,
wrong, or newly-buggy fixes.

**Part B — fresh adversarial pass:** Ignore what was fixed. Look at the
current code with fresh eyes. What did round-1 miss? What did the fixes
newly break? Mid-level-engineer mindset.

Do not pad. Cite file:line. If ready to ship, say so.

---

**File under review:** `{{FILE_PATH}}`

**Context files:**
{{CONTEXT_FILES}}

---

**Round-1 findings (verify each):**
{{ROUND_1_FINDINGS}}

---

**Adversarial rubric for Part B (fresh pass):**
{{NEW_RUBRIC}}

**Output format (strict):**

## Round-2 Adversarial Review: {{basename}}

### Part A — Fix Verification
| Finding | Fixed? | Notes |
|---|---|---|
| C1 | ✅/⚠️/❌ | ... |
| C2 | ✅/⚠️/❌ | ... |
...

### Part B — Fresh Findings

**Overall Assessment:** [1 sentence — READY TO SHIP / CONCERNS / REVISIT]

### NEW CRITICAL
If none: "None identified."

### NEW MAJOR
### NEW MINOR
### Questions for the author

Cite file:line. Advisory only.
```

---

## Rubric categories — reusable building blocks

Mix-and-match these for the `{{CATEGORY_RUBRIC}}` slot, adding or
removing based on the file's risk surface. 

### Python / general
1. **Correctness** — regex anchoring, off-by-one, non-greedy quantifier
   pitfalls, recursion without depth limit, generator exhaustion.
2. **SQL correctness** — column count matches placeholders, FK
   relationships honored, WHERE clauses that accidentally become
   tautological, status-drift between SELECT and UPDATE (TOCTOU inside
   vs across transactions).
3. **Transaction semantics** — `with conn:` correctness, what happens
   on exception mid-transaction, idempotency under retry, per-row
   atomicity claims actually held.
4. **Idempotency** — will a second run produce the same result? What
   breaks the stability (ordering, timestamps, nondeterministic parse)?
5. **Failure handling** — all exception paths covered? off-by-one on
   attempt-count transitions to terminal states? what if a concurrent
   process already advanced the row?
6. **Observability** — progress prints flushed? silent failure modes
   (fallback path always "works" → operator can't see regressions)?
   summary output grep-friendly and reconcilable against inputs?
7. **Resource safety** — connection pooling, thread safety between
   main loop and background threads, signal-handler race conditions.
8. **Edge cases** — empty inputs, very first row, very last row,
   boundary values, non-ASCII, very large values, malformed data that
   parses as structurally valid but semantically wrong.
9. **Cross-stage integrity** — if this script depends on upstream
   scripts' outputs (S3 keys, DB state, ledger files), what if those
   assumptions are violated (empty S3 object, malformed ledger file,
   wrong stage)?

### Shell / bash
1. **Quoting + word-splitting** — `"$@"` vs `$@`, `printf '%q'` round-trip
   semantics, args with spaces/quotes/backticks/metas, variables in
   command strings passed to tmux/ssh/sudo.
2. **Race conditions on external state** — lockfile read-then-act
   windows, PID reuse, tmux session lifecycle races.
3. **Signal handling** — Ctrl-C behavior, trap ordering relative to
   critical sections (install trap BEFORE or AFTER the signal it
   references), signal propagation through exec/subshells.
4. **Error code hygiene** — `set -euo pipefail` interactions with
   pipelines that legitimately fail, `|| true` scope, exit codes
   consistent across commands.
5. **Portability** — macOS vs Linux (BSD vs GNU wc/awk/ps/date),
   POSIX sh vs bashism compatibility with shebang claim.
6. **Operational footguns** — error messages name the next action the
   operator should take? usage text reflects current reality?
7. **Idempotency of destructive ops** — running `stop` twice, running
   `start` after an ungraceful exit, what happens to stray state.

### Domain-specific (add as needed)
- **Read-only guarantees** for status/monitoring tools (`PRAGMA
  query_only`, no side effects anywhere).
- **Cost/quota correctness** for tools surfacing billing info (are we
  summing apples + oranges? are units consistent with upstream pricing
  model?).
- **Rate/throughput math** — denominators correct under "younger than
  window" edge cases, numerator vs denominator unit consistency.
- **Clock-skew defense** for any tool doing time-delta math across
  multiple hosts or storage layers.

---

## The stop-at-round-2 heuristic

| Round | Scripts reviewed | CRITICAL found | MAJOR found | MINOR found |
|---|---|---|---|---|
| Round 1 | 5 scripts | ≥2 per script | 6-10 per script | 7-8 per script |
| Round 2 | 5 scripts | 0 per script | 0-2 per script | 7-8 per script |

**Conclusion:** if round-2 returns 0 CRITICAL AND 0-1 MAJOR, STOP.
Round-3 expected return is pedantic MINORs that don't justify the
prompt cost. If round-2 returns 2+ MAJORs, apply those and consider
round-3.

**Don't confuse "nothing to fix" with "review complete."** A clean
round-1 return IS possible for simple files, but for System-tier
production code it's the exception, not the rule. Budget for at
least one round of real findings.

---

## Collaborating with the User on review results

Always present findings as a disposition table before touching code:

```markdown
| # | Finding | Recommended disposition |
|---|---|---|
| C1 | [description] | **ACCEPT.** [rationale] |
| M1 | [description] | **PARTIAL ACCEPT.** [what part, why partial] |
| M2 | [description] | **DEFER — canary validates this naturally.** |
| Nm1 | [description] | **DEFER (cosmetic).** |
```

The user scans, decides "accept your recommendations" or redirects specific
items. Then Chief applies, re-runs tests, re-launches round-2 review
if any MAJORs shifted. 

