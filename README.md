# meta-os agents pack

The **agent engineering discipline** for a [meta-os](https://github.com/meta-agentic/meta-os)
Agentic OS instance: how to decide whether to build an agent, what shape it takes, and what
has to be true before it is allowed to run without you. A pack is a codified discipline — a
method, a standard of rigor, and portability across estates (see the framework's
`systems/pack-strategy.md`).

| Skill | What it drives | Checkable output |
|-------|----------------|------------------|
| `skills/agent-architecture/` | The shape decision: the agent-vs-pipeline refusal test, the escalation ladder (single loop → handoff → state graph → crew), reasoning-pattern selection, and the invariants that bound any loop | **agent decision record** |
| `skills/agent-prompting/` | The prompt as versioned code: the four layers (identity, instructions, context, output contract), persona as a measured choice, reasoning budgets, prompt regression | **prompt version record** |
| `skills/agent-skills/` | Declarative procedures loaded on demand: Clarify–Execute–Verify, activation-index descriptions, bundled-script rules, and the posture that treats a `SKILL.md` as executable code | **skill review record** |
| `skills/agent-harness/` | Everything around the model: execution boundary, sandbox, memory and durable state, unconditional verification, context pipelines | **harness contract** |
| `skills/agent-safety/` | What the agent may do: layered guardrails, the threat model for agents that read external data, least-privilege tool scoping, where a human gate belongs | **threat & permission ledger** |
| `skills/agent-evaluation/` | Grading the trajectory, not just the answer: strong case design, judge discipline, CI gates, permanent red-team cases | **eval report** |
| `skills/loop-engineering/` | Running to a goal without a human in the path: machine-checkable termination, structured feedback, futility detection, unattended operation, the self-improving harness | **loop charter** |

Every skill is parameterized against instance config (`autonomy`, `sandbox`,
`max-iterations`, `session-budget`, eval thresholds, `trace-home`, `escalation`) — they carry
the method, not anyone's stack — and each emits a named ledger with a defined *failing*
reading, so a reviewer can audit a run against the discipline's own standard instead of
taking it on trust.

## The two ideas the pack is built on

**Most proposed agents should not exist.** The first gate in the first skill is a refusal
test, and a "no" is a successful outcome. Agents are slower, costlier and non-deterministic,
and you pay for all three on every run forever.

**Agent = model + harness.** The model reasons; the harness decides whether the proposal
executes. Every property worth having — safety, cost control, reproducibility, the right to
run unattended — is a property of the harness. The recurring move across all seven skills is
to take a rule out of the prompt, where it is probabilistic, and put it in code, where it is
not.

## Why this is a discipline, not a prompt pile

The three-part test every pack must pass (`systems/pack-strategy.md`):

- **Recognizable** — an engineer who has shipped an agent and then operated it reads the
  escalation ladder, the harness layers and the loop charter as how the work actually goes.
- **Portable** — nothing is welded to a framework or a provider. The autonomy level is one
  config line (`autonomy: supervised | semi-autonomous | autonomous`), and every budget,
  threshold, sandbox level and path resolves from instance config.
- **Checkable** — every skill emits a ledger with an explicit failing reading: a loop with
  no verifier named, a tool granted more scope than its task, an eval threshold lowered
  without a reviewed diff, a skill merged unread.

## Mount it

From your instance root (see the framework's `systems/packs.md`):

```bash
scripts/packs.sh add agents
```

Skills land in the instance's union `skills/` and project-local `.claude/skills/`.
This pack ships no hooks and no agents.

## Provenance

First-party and original. Concepts are grounded in Antonio Gulli's two books —
*Agentic Design Patterns* (Springer, 2025) for the pattern taxonomy and
*Building AI Agents: From Design Patterns to Production* for the operational discipline —
with attribution and no text or code reproduced. See `PROVENANCE.md`.
