---
type: index
tags: [os, skills, pack, agents]
---
# agents pack — skills

Each skill is an *executable discipline*: a method + a standard of rigor + a checkable
artifact. They compose in the order an agent is actually built.

**[[skills/agent-architecture/SKILL|agent-architecture]] is the entry point** — it decides
whether to build an agent at all and what shape it takes. Three skills fill that shape in:
**agent-prompting** (the instruction layer), **agent-skills** (the procedure library), and
**agent-harness** (the code that makes rules deterministic instead of hopeful).
**agent-safety** decides what the thing is permitted to do; **agent-evaluation** decides
whether it works; **loop-engineering** is what lets it run without you. No skill restates
another — each cites its siblings by wikilink.

| Skill | Discipline | Checkable output |
|-------|------------|------------------|
| [[skills/agent-architecture/SKILL\|agent-architecture]] | The shape decision: the refusal test, the escalation ladder (loop → handoff → graph → crew), reasoning-pattern selection, loop invariants | **agent decision record** |
| [[skills/agent-prompting/SKILL\|agent-prompting]] | The prompt as versioned code: four layers, measured personas, reasoning budget, regression testing | **prompt version record** |
| [[skills/agent-skills/SKILL\|agent-skills]] | Declarative procedures: Clarify–Execute–Verify, descriptions as activation indexes, bundled scripts, skills-are-code security | **skill review record** |
| [[skills/agent-harness/SKILL\|agent-harness]] | Model proposes, code disposes: execution boundary, sandbox, persistence and memory, verification loops, context pipelines | **harness contract** |
| [[skills/agent-safety/SKILL\|agent-safety]] | Guardrails and permissions: layered inspection, the fetched-data threat model, zero-trust scoping, where the human gate goes | **threat & permission ledger** |
| [[skills/agent-evaluation/SKILL\|agent-evaluation]] | Trajectory-level grading: strong cases, judge discipline, CI gates, permanent red-team cases | **eval report** |
| [[skills/loop-engineering/SKILL\|loop-engineering]] | The system as the loop: termination conditions, structured feedback, futility detection, unattended operation, the self-improving harness | **loop charter** |

Progressive disclosure: `agent-architecture` carries
[[skills/agent-architecture/references/pattern-catalogue|references/pattern-catalogue.md]] —
21 patterns with a selection criterion and a cost for each — loaded only when the four
primary reasoning patterns don't settle the shape.

Config knobs in `pack.yaml`; autonomy profiles in `profiles/` (`supervised` ·
`semi-autonomous` · `autonomous`). See `README.md`.

<!--
  Required by the meta-os convention that every folder carries its own _index.md.
  Add a row when you add a skill — this table is what an agent reads on entering the
  folder, so a missing row means a skill nobody finds.
-->
