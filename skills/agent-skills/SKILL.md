---
name: agent-skills
description: "Use when authoring, reviewing, refactoring, or securing a declarative skill (a SKILL.md an agent loads on demand) — and whenever an agent skips a step of a documented procedure, takes a shortcut it was told not to, activates the wrong skill for a task, or a system prompt has grown into an unreadable wall of standing instructions. Covers the Clarify–Execute–Verify structure, writing descriptions as activation indexes, bundled scripts, and the security posture that treats a skill as executable code. Emits a skill review record."
---

# Agent skills — procedures the agent loads when it needs them

A skill is a procedure written for a machine that will find every shortcut and take it.
That sentence explains every convention below: write a procedure that holds under pressure,
make it findable by the task that needs it, and treat the file as what it is — executable
behavior that gets reviewed before it ships.

The problem it solves is accumulation. Standing instructions pile up until an agent skips
step two of a five-step procedure because it was buried on line 400. The fix is not a bigger
context window; it is a library loaded on demand, which is where instructions go once the
prompt has outgrown them.
## Method

### 1. Write the description as an activation index

At rest, only frontmatter — name and description — sits in context. That costs almost
nothing, so a library can be large; when a task matches a description, the body loads.

The description is therefore matched *against the task*, not read as a summary. Write it in
the words a person would actually use, including their verbs and nouns. Too vague and the
wrong skill activates; too specific and a legitimate trigger is missed. This is the highest
-leverage line in the file, and routing degrades as the library grows: past a few dozen
skills, choosing among them becomes a leading source of error. Two skills whose descriptions
could both plausibly match one task is a design bug in the pair.

### 2. Structure the body as Clarify → Execute → Verify

**Clarification gates** — list the facts the agent must hold before the first irreversible
step: which environment, which target, whether a backup exists. If any is missing it stops
and asks. Forbid inferring them from context; inference is exactly what produces a confident
production migration that was meant for staging.

**Rationalizations** — write down the excuses the agent will actually generate, each with
its rebuttal, so the shortcut is refuted before it is thought. This section feels strange to
write once and obvious forever after. Every excuse found in a trace belongs here afterwards.

**Verification** — define what proves the task is done, and state plainly that a command
exiting zero is not proof. Query the resulting state and confirm it. Skills that demand
evidence compose: one skill's verified output becomes the next one's trusted input.

### 3. Apply the writing rules

| Rule | Why |
|---|---|
| **Match specificity to fragility** | Robust tasks get goals; brittle ones get the exact command with exact flags and an instruction not to embellish |
| **Provide defaults, not menus** | One clear default and one explicit escape hatch. Five options asks the agent to make a judgment you already made |
| **Plan → validate → execute** | Build validation into the procedure rather than trusting the agent to check itself |
| **Procedure, not data** | A skill distilled from a successful run carries the method and none of that run's values |
| **One skill, one job** | A skill that searches *and* summarizes is a hidden agent — split it and let the loop orchestrate |

### 4. Constrain bundled scripts

Models are unreliable at generating exact deterministic logic on demand; when a step needs
precision, ship a script. Three constraints: **no interactive prompts** — a script waiting
on `[y/N]` hangs the agent forever, so take flags; **emit JSON, not formatted tables** —
the caller parses; **use distinct, documented exit codes** so the caller can branch without
scraping text.

**Pin every version.** An unpinned package-runner invocation is a live dependency on
whatever is published under that name today. Agents also invent plausible package names and
attackers register them, so an improvised unpinned command is a supply-chain opening.

### 5. Treat the file as code

A SKILL.md is executable instruction: anyone who can write to the skills directory can
redirect the agent. Version-control skills alongside the code they operate on; review
changes in pull requests, because a changed SKILL.md is changed behavior; restrict write
access by filesystem permission or CI gate; audit skills from outside sources by reading the
whole body, not the frontmatter; and sandbox the loader so path validation stops an agent
from talking it into reading elsewhere — see [[skills/agent-harness/SKILL|agent-harness]].

This applies with full force to skills an agent wrote itself. Distilling a hard-won success
into a reusable skill is among the highest-return moves available — the lesson is paid for
once instead of re-taught every session — but merging it unread automates the injection of
unreviewed behavior into every future run. `skill-review: advisory` is a sandbox setting,
not a production one.

### 6. Improve without rewriting

When a skill underperforms, the instinct is to regenerate it wholesale; a full rewrite
discards what the file already got right and re-introduces fixed problems. **Bound the
edit** to a handful of lines per revision — unbounded rewriting on the evidence of one bad
run overshoots the way an oversized learning rate does. **Keep a rejected-edit log** so the
same broken idea is not re-proposed next month. **Protect the durable lessons** in a section
ordinary edits may not overwrite, or long-run learning gets flattened by whichever revision
came last. Validate every proposed edit against held-out cases before accepting it.

## The rigor standard

- **The description is an activation index and is tested as one** — against real phrasings,
  and against the rest of the library for collision.
- **Clarification gates precede every irreversible step**, and nothing required is inferred
  from context.
- **"The command succeeded" is never verification.** Evidence means the resulting state was
  queried and matched.
- **Fragile operations get exact commands.** A high-level goal in front of a production
  migration is an invitation to improvise.
- **Scripts are non-interactive, machine-readable, and pinned.** Any one of the three
  missing makes the script unusable by the caller it was written for.
- **A distilled skill carries no data from the run it came from.**
- **Agent-authored skills go through human review** wherever `skill-review: required`, which
  is everywhere that is not a sandbox.
- **Edits are bounded and validated**, and a rejected edit is recorded with its reason.

## Checkable output

Authoring or revising a skill ships a **skill review record**:

```markdown
# Skill review record — deploy-checklist
- **Author:** agent-distilled from trace raw/agents/2026-08-19-deploy
- **Activation:** tested against 7 phrasings ("ship it", "deploy to prod",
  "cut a release"…), 7/7 matched; collides with: none (checked `release-notes`)
- **Clarify:** target environment · change ticket · rollback verified
- **Rationalizations:** 4 covered — incl. "the pipeline is green so the gate is
  redundant"
- **Verify:** health endpoint returns the new build SHA; exit code alone rejected
- **Scripts:** scripts/preflight.py — non-interactive, JSON out, exit 0/1/2/3
  documented; pinned tool versions
- **Security:** write access = repo maintainers · reviewed in PR #48 · no
  external content ingested
- **Edit bound:** 6 lines changed · rejected-edit log updated (1 entry: "skip
  preflight when hotfix" — rejected, hotfixes are when it matters most)
- **Validation:** 9 held-out cases — 9/9
```

The record fails review when activation was not tested, when an agent-authored skill has no
PR link, or when the edit is unbounded.

## Anti-patterns

- **The description written as a summary.** "Does database things" is a sentence about the
  file; the frontmatter needs a sentence about the *task*.
- **Two skills with overlapping triggers**, shipped separately and each surprised when the
  other activates.
- **Suggestions dressed as a procedure.** Without rationalizations, an agent under pressure
  reasons its way around every step that costs it time.
- **A menu of five options**, deferring to the agent a judgment the author already had the
  context to make.
- **The fossilized skill** — distilled from a real run with that run's figures inside it,
  confidently reporting a quarter that already ended.
- **The interactive script** that hangs a loop forever on a prompt no one will ever answer.
- **`uvx tool` without a version**, on a name the agent may have invented.
- **Auto-merging what `/learn` produced.** The distillation is the cheap part; the review is
  what makes it trustworthy.
- **Rewriting a skill from scratch each time it underperforms**, so it never accumulates.

## Configure

Reads `packs.agents.config` (`scripts/packs.sh config agents`), config-first with these
fallbacks:

| Key | Meaning | Default |
|-----|---------|---------|
| `skill-review` | `required` (agent-authored skills land through a PR) \| `advisory` (sandbox only) | `required` |
| `trace-home` | Where skill review records and rejected-edit logs are written | — |
| `sandbox` | Bounds what a skill's bundled scripts may execute | `container` |

## Related

[[skills/agent-prompting/SKILL|agent-prompting]] — the prompt shrinks as this library grows.
[[skills/agent-harness/SKILL|agent-harness]] — owns the loader, its path sandboxing, and the
progressive disclosure this skill applies to procedures.
[[skills/loop-engineering/SKILL|loop-engineering]] — experience distillation is where
agent-authored skills come from, and why review matters.

*Grounding:* the Agent Skills protocol, progressive disclosure and the Clarify/Execute/Verify
structure from *Building AI Agents: From Design Patterns to Production* (Gulli); bounded-edit
and rejected-edit discipline from the skill-optimization literature surveyed there. Concepts
only — see `PROVENANCE.md`.
