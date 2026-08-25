# Provenance

| Skill | Origin | License |
|-------|--------|---------|
| `agent-architecture` | first-party — authored in [meta-agentic/meta-os](https://github.com/meta-agentic/meta-os) and moved here when this pack was extracted | MIT |
| `agent-prompting` | first-party — authored for this pack | MIT |
| `agent-skills` | first-party — authored for this pack | MIT |
| `agent-harness` | first-party — authored for this pack | MIT |
| `agent-safety` | first-party — authored for this pack | MIT |
| `agent-evaluation` | first-party — authored for this pack | MIT |
| `loop-engineering` | first-party — authored for this pack | MIT |

All content is original prose and **public-safe by construction** — no instance data (repo
names, trackers, machine paths, promoted knowledge). Every estate-specific value is a
config key resolved from the instance's `.packs.yaml`, never a hardcoded name. No
third-party code, text, figures, or exercises are vendored.

## Grounding and attribution

The pack's *coverage map* — which decisions an agent engineering discipline has to have a
standard for — was drawn from two books by Antonio Gulli:

- ***Agentic Design Patterns: A Hands-On Guide to Building Intelligent Systems***
  (Springer, 2025; [publisher](https://link.springer.com/book/9783032014016)) — the pattern
  taxonomy. Its 21 patterns are the source of the grouping and the selection criteria
  summarized in `skills/agent-architecture/references/pattern-catalogue.md`.
- ***Building AI Agents: From Design Patterns to Production*** (draft, read with the
  author's permission; companion code at
  [agulli/atlas-agents](https://github.com/agulli/atlas-agents), MIT) — the operational
  discipline: the harness layers, the guardrail and threat model, trajectory evaluation,
  loop engineering, and the review-inversion argument.

**Concepts only.** No text, code, figure, table, or exercise from either book is copied,
quoted at length, or vendored here. Every skill is original prose written against standard
practice, and the parts that are this pack's own rather than either book's — the escalation
ladder as a single ordered gate, the rigor standards, the emitted ledgers, the autonomy
profiles, and the config schema — are the majority of the material. Where a named coinage
is load-bearing (for example *model proposes, code disposes* as the harness principle), it
is used as a named principle with the source credited rather than silently absorbed.

Both books are credited because the split between them is the pack's own spine: the first
teaches how to **write** agents, the second how to **run** them, and a discipline needs
both halves.

## Selection of coverage

The pack codifies **agent engineering as practised by an estate that runs agents, not just
writes them**, across the seven places where the practice needs a standard rather than a
preference:

- **shape** (`agent-architecture`) — whether to build an agent at all, and how much
  structure the task actually justifies;
- **instruction** (`agent-prompting`) — the prompt as a versioned, regression-tested
  artifact rather than prose someone edits on a Friday;
- **procedure** (`agent-skills`) — knowledge loaded on demand, authored so a machine that
  looks for shortcuts cannot find one;
- **enforcement** (`agent-harness`) — the deterministic layer that makes the other rules
  real;
- **permission** (`agent-safety`) — what the agent is capable of, before what it is
  inspected for;
- **grading** (`agent-evaluation`) — the trajectory, gated in CI;
- **operation** (`loop-engineering`) — the loop that runs to a checkable goal unattended,
  and the harness that learns from its own logs.

What is *not* here is deliberate. Framework tutorials, provider SDK differences, model
benchmarks and pricing tables all age in months and belong in documentation, not in a
discipline. Multimodal and voice interaction, managed-platform selection, and deployment
mechanics are adjacent engineering concerns rather than thin sections of this one. The pack
keeps what outlives the frameworks: the decisions, the rigor standards, and the artifacts a
reviewer can check.

## Dependency

None. This pack's skills stand alone and cite each other by wikilink rather than
duplicating; all seven are listed in `pack.yaml`'s `provides:` as the public surface other
packs may depend on.
