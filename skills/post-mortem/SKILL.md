---
name: post-mortem
description: Structured session retrospective — narrative, pivotal moments, patterns, and actionable recommendations
---

# Post-Mortem — Skill Definition

## Trigger

**Manual:** The user says `/post-mortem`, "let's do a post-mortem", "let's debrief this session", or similar.

**Chief flags candidates:** After sessions with multiple pivot points, repeated issues, significant design decisions, or builds that hit unexpected obstacles, Chief suggests: *"That was a meaty session — want to run /post-mortem?"* The user decides whether to proceed.

---

## What This Skill Does

Creates a dated, narrative-driven retrospective of the current session. Goes beyond "what broke" to capture the full story — how problems were encountered and solved, how the collaboration dynamic worked, what patterns emerged, and what concrete actions should follow.

The post-mortem is a learning document. Its goal is insight, not judgment.

---

## Execution Steps

### Step 0: Create the file

Create a timestamped post-mortem file:
- **Location:** `post-mortems/`. All post-mortems live in one place regardless of which project the session was about.
- **Filename:** `post-mortem-YYYY-MM-DD_HHMM.md`
- **Check the clock** (`date`) for accurate timestamp.

### Step 1: Write the chronology

Write a **strict chronological account** of the session. This is the core of the post-mortem — a dispassionate, detailed record of what happened in the order it happened. No thematic grouping, no imposed narrative arcs, no "acts." Just the events, decisions, and pivots as they occurred.

**How to write it:**
- Go through the conversation **linearly, start to finish.** Do not skip ahead, do not grep for keywords and construct a story from fragments. Read the sequence.
- Write in **chronological order.** Each paragraph or section should correspond to a phase of work in the order it happened. Use timestamps or sequence markers where possible.
- **Report what happened, not what it means.** Save interpretation for later steps. The chronology is facts: who said what, what was tried, what the result was, what happened next.
- **Include dead ends at full detail.** Failed approaches often contain the most important information. Don't compress "we tried several things and they didn't work" — lay out each attempt, why it failed, and what happened after.
- **Track who contributed what.** If the user went to another AI (CAI, Gemini, etc.), a friend, or an external source, note what they came back with and how it changed the direction. If Chief suggested something, note it. The post-mortem needs to capture the full collaboration, not just Chief's perspective.
- **Include the user's words when they signal something important.** Direct quotes that show frustration, breakthrough moments, or reframings are more valuable than Chief's paraphrase of them.

**Do NOT:**
- Impose a narrative structure (acts, themes, arcs) before you've written the chronology
- Start with lessons and then arrange facts to support them
- Summarize long stretches of iteration as "we went back and forth on X"
- Tell the story from Chief's perspective — tell it from the work's perspective
- Collapse multiple distinct attempts into one paragraph because they were "about the same thing"

**Length should match session weight.** A 30-minute fix gets a short chronology. A multi-hour session with many dead ends and pivots gets a long, detailed one. Use as much space as needed to capture what actually happened. No bullet-point summaries. Prose, in order, with detail.

**When working from JSONL session files:** Read the user messages linearly, not via keyword search. The sequence is the story. Grep can help you find a session, but it cannot help you understand one.

### Step 2: Identify pivotal moments

Identify 1-3 moments that shifted the trajectory of the session — where the problem was reframed, the approach fundamentally changed, or one person's intervention elevated the thinking.

For each pivotal moment:
- **What happened** — the specific thing that was said or done
- **What it changed** — how the session's direction shifted afterward
- **Why it mattered** — the downstream impact, what wouldn't have happened without this moment
- **The meta-lesson** — what this teaches about how the user and Chief work together

These are singular events, not recurring patterns. They may happen only once in a session but carry outsized importance. If no genuinely pivotal moments occurred, write *"None identified — [brief reason, e.g., straightforward execution]"* per the empty-sections rule. Not every session has them.

### Step 3: Identify negative patterns (2+ occurrences)

Look for behaviors or choices that caused problems and occurred at least twice during the session.

For each pattern:
- Describe the pattern and the specific instances
- Identify whether any existing skill, convention, or reference file *should have* prevented it
- If yes: why didn't it fire? (Not loaded? Too vague? Overridden by execution pressure?)
- If no: what *could* prevent it in the future?

Be honest but not punitive. The goal is accurate observation that leads to system improvement, not self-flagellation.

If no negative patterns occurred at 2+ occurrences, write *"None identified — [brief reason, e.g., no recurring issues in a short session]"* per the empty-sections rule.

### Step 4: Identify positive patterns (2+ occurrences)

Look for behaviors or choices that worked well and occurred at least twice.

For each pattern:
- Describe the pattern and the specific instances
- Assess whether it's already captured somewhere (CLAUDE.md, a skill, a convention)
- If not: is it worth turning into a rule, convention, or skill? Or is it situational?

**Don't skip this step.** Learning from success prevents regression — if you only track failures, you'll avoid past mistakes but may drift away from approaches that were already working.

If no positive patterns occurred at 2+ occurrences, write *"None identified — [brief reason, e.g., short session without repeats]"* per the empty-sections rule.

### Step 5: Recommendations

Write up concrete, actionable recommendations. Each should specify:
- **What** to do (design a system, create a skill, add information, modify a process)
- **Where** it would live (which file, skill, or convention)
- **Why** it matters (what problem it prevents or what positive pattern it reinforces)
- **Implementation hint** — enough for the user to judge whether to do it now or backlog it. Is this a one-line edit to an existing file? A new file? A skill modification? A build? The user needs to know the size of the work at a glance so they can say "just do it" or "backlog it."

Recommendation types include:
- Design a system that does X
- Create a skill that does Y
- Add Z information to [specific file]
- Ensure that [trigger] loads [context file]
- Modify [skill] to include [step]
- Add a convention to CLAUDE.md
- Route [feedback] to [permanent home]

**Document and recommend only.** Don't execute recommendations during the post-mortem. They require the user's judgment and may need their own design process.

If no recommendations emerge, write *"None — [brief reason, e.g., the session worked cleanly and no process gaps surfaced]"* per the empty-sections rule. Don't invent recommendations to fill the section.

### Step 6: Review with the user and backlog

After writing the post-mortem, **show it to the user** — either present it in conversation or point them to the file. Then ask:

*"Want to add any of these recommendations to the backlog?"*

For each recommendation the user approves:
- Add it to `backlog.md`
- Include a link back to the post-mortem file for context: `(see: post-mortems/post-mortem-YYYY-MM-DD_HHMM.md, Recommendation #N)`
- This ensures the backlog item is traceable — anyone picking it up later can read the full reasoning

The user may also:
- Modify a recommendation before adding it
- Reject some recommendations — that's fine, just don't add them
- Want to execute one immediately — that's a separate conversation, not part of the post-mortem

---

## Output Format

```markdown
# Post-Mortem: [Session Topic]

**Date:** YYYY-MM-DD
**Duration:** ~Xh
**Participants:** the user + Chief
**Outcome:** [1-2 sentence summary of what was accomplished]

---

## 1. Chronology
[Detailed chronological account — prose, in order, no thematic grouping. Dead ends at full detail. Who contributed what. Use as much space as needed.]

---

## 2. Pivotal Moments
[1-3 moments with What happened / What it changed / Why it mattered / Meta-lesson — OR "None identified — [reason]" per the empty-sections rule]

---

## 3. What Was Created/Modified
[Lists of created, modified, deleted items]

---

## 4. Negative Patterns (2+ occurrences)
[Pattern, instances, what could prevent it — OR "None identified — [reason]"]

---

## 5. Positive Patterns (2+ occurrences)
[Pattern, instances, worth preserving as what — OR "None identified — [reason]"]

---

## 6. Recommendations
[Numbered, actionable items with what/where/why — OR "None — [reason]"]
```

After writing, present to the user and ask which recommendations to add to the backlog.

---

## Rules

- **The chronology is the foundation.** Everything else flows from it. If the chronology is wrong or incomplete, the patterns and recommendations will be wrong too. Spend most of your effort here. Read the conversation linearly — do not grep-and-assemble.
- **Empty sections are explicit, not blank.** If a section has nothing substantive to capture (no pivotal moments, no patterns at 2+ occurrences, no recommendations), write *"None identified — [brief reason]"* rather than leaving it blank or padding with weak content. Explicit "none" signals the question was asked and genuinely answered. This keeps lightweight post-mortems clean without making them feel performative.
- **Chronology before interpretation.** Write the full chronology FIRST. Only then identify patterns and recommendations. If you start with lessons and arrange facts to support them, you'll miss what actually happened. (Origin: 2026-04-02 — Chief grep'd JSONL for keywords, constructed a thematic narrative, and missed the entire multi-AI collaboration arc, the EPS→SVG dead end, and the actual breakthrough moment. The user's own account was dramatically more accurate.)
- **Be honest, not brutal.** Accurate observation that leads to improvement. Not punishment.
- **Track all contributors.** This isn't just Chief's log. If the user consulted CAI, Gemini, friends, or external resources, capture what each contributed. The collaboration dynamic across all participants is often the real story.
- **Dead ends deserve full detail.** Don't compress failed approaches into "we tried several things." Each dead end has a specific failure mode that teaches something. Lay out the sequence: what was tried, why it failed, what happened next.
- **Don't skip positive patterns.** They're as valuable as negative ones.
- **Don't execute recommendations.** Document them. The user decides what to act on.
- **Route this post-mortem's own learnings.** After the user reviews the post-mortem and decides on actions, route the decisions per the normal information routing system (CLAUDE.md, skill files, project notes, etc.). The post-mortem is an analysis document, not a permanent home for rules.
