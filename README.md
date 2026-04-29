# Noor's Build Skill

A `/build` orchestrator skill for Claude Code, plus a companion `/post-mortem` skill that closes the learning loop. Together they walk the assistant — and you — through a structured build process, scaling the ceremony to match the size of what you're making, and feeding lessons from each build back into the system.

**What's in this repo:**
- `skills/build/` — the `/build` skill (`SKILL.md` + `adversarial-review-template.md`)
- `skills/post-mortem/` — the companion `/post-mortem` skill that the build skill invokes after weighty builds
- `.claude-plugin/plugin.json` — plugin manifest so Claude Code can install both skills as a single plugin

## What it does

Most builds fail in the same handful of ways: scope creep, missing reviews, half-thought-out designs, no plan, no QA, no record of what you actually shipped. This skill makes the assistant move through a predictable sequence — Scope → Design → Plan → Review → Build → QA → Ship → Monitor → Improve — and writes a state file as it goes, so you always know where you are and can pick up later.

The big idea: **scale the process to the work**. A 30-minute config tweak doesn't need a design review. A multi-week system rewrite absolutely does. The skill defines four tiers and routes you to the right amount of structure:

- **Config** — quick tweaks, < 1 hour, no state file
- **Skill** — new prompt or workflow, ~1-3 hours, lightweight design
- **Feature** — new functionality, half-day to a few days, full Plan + Review
- **System** — complex multi-component work, multi-day, includes Monitor phase for ongoing health

## How it works

Every build above Config tier writes a **state file** — a markdown doc that captures the tier, the design, the plan, the ship log, and a running checklist of what's done. The state file is the brain of the build, and it lives on disk, not in the conversation.

That single design choice unlocks the thing that makes this skill different: **builds can span multiple sessions.**

A System-tier build that takes a week across four sessions might go:

- **Session 1** — Scope, Design, Design Review → state file saved, session ends
- **Session 2** — Plan, Implementation Review → state file saved, session ends
- **Session 3** — Build steps 1-5 → state file saved, session ends
- **Session 4** — Build steps 6-10, QA, Ship → done

You don't have to finish everything in one giant context window. The skill writes to a state file on disk at each phase boundary (Scope, Design, Plan, Build, Ship), so once a phase is complete, it's safely persisted — you can clear context, walk away, come back tomorrow, hand the build off to a different conversation. On day four, you say `/build [name]` to your assistant — they read the state file and pick up cleanly.

**One caveat:** mid-phase work-in-progress lives in your active conversation until the phase commits to the state file. If you need to clear context *mid-phase*, ask your assistant to save current state first — otherwise that in-flight thinking is lost.

This is the **ralph loop** pattern: persistent state outside the conversation, the assistant looping back to it each session.

## Why this beats ad hoc building

Most of us, building with an AI assistant ad hoc. This skill forces gates that are easy to skip when you're heads-down in the work:

- **Background review agents launch automatically** at the right phase. They review with NO conversation context — fresh eyes — so they catch things the building assistant has gone blind to. This is genuinely different from "hey claude, review this."
- **QA runs automatically after Build** completes. Not "we'll test it later."
- **Post-mortem prompts** appear after weighty builds. The companion `/post-mortem` skill runs a structured retrospective — chronology → pivotal moments → patterns → recommendations — and recommendations route back into the system (skill edits, backlog entries) so the next build is smarter than this one. *("Backlog" = a document that tracks items to implement in future builds or that require further action — the place recommendations live until they're acted on.)*

Tier-scaling is the other half: you're not applying System-level ceremony to a 30-minute tweak, but you're also not winging a multi-day rewrite with no plan. The skill picks the right level of structure for the work in front of you.

## Who it's for

Anyone using Claude Code (or a similar AI assistant) for real, non-trivial work who wants a shared vocabulary with their assistant for *"what tier is this, what phase are we in, what's the gate before we move forward."*

## How to install

This repo is set up as a Claude Code plugin. In your Claude Code session, run:

```
/plugin marketplace add PolarisLibra/Noor-Build-Skill
/plugin install noor-build
```

Then run `/reload-plugins` (or restart Claude Code) and you're done. Both skills are installed.

**Heads up on naming:** Claude Code namespaces plugin-provided skills under the plugin name. So once installed, you invoke them as:

- `/noor-build:build` — to start a build
- `/noor-build:post-mortem` — to run a retrospective (the build skill will offer this for you after Ship)

Slash commands are the most reliable way to invoke. Natural-language triggers like "let's build something" may also fire depending on your assistant's setup, but for important builds, lead with the slash command.

The skills are self-describing: they tell the assistant when to ask questions, when to launch background review agents, when to pause for your approval, and when it's safe to keep moving.

Both skills install together as one plugin, but you don't have to use both. If you skip `/noor-build:post-mortem`, the Retrospective phase just becomes a manual exercise. The pair is designed to work together though, and the learning loop is most of the value.

### Manual install (if you don't want the plugin)

If you'd rather skip the plugin system and use the skills as `/build` and `/post-mortem` directly (no namespace prefix), copy the skill folders into your local `.claude/skills/`:

```
cp -r skills/build ~/.claude/skills/build
cp -r skills/post-mortem ~/.claude/skills/post-mortem
```

Then invoke as `/build` and `/post-mortem`.

## What the skill creates in your project

As you build, the skill writes a few files and folders in whatever directory you're working in:

- `build-state-[name].md` — one file per active build, lives in the project root
- `build-archive/` — completed and abandoned build state files move here after Ship
- `post-mortems/` — dated retrospective files from `/post-mortem`
- `backlog.md` — running list of recommendations and known limitations awaiting action

Nothing is hardcoded — these are just sensible defaults relative to your working directory. If you want them somewhere specific (a notes vault, a separate folder, etc.), edit the paths in the SKILL.md files.

## A note on naming

The skill refers to the assistant as **"Chief"** throughout — that's the name I use for my Claude Code assistant. The patterns hold regardless of what you call yours; mentally substitute whenever you read it.

## Adapt freely

Take what's useful, ignore what isn't, rename whatever feels off. If you build something good on top of this, I'd love to hear about it.  --Noor

## Credits

Built by Noor across many sessions of iterative collaboration with Claude (Anthropic's Claude Code). 

## License

MIT — see [LICENSE](LICENSE).
