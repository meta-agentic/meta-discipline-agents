---
type: profile
tags: [pack, agents, profile, autonomy]
---
# Profile — autonomous

**The loop runs to its termination condition without a human in the path.** Reserved for
narrow, well-understood workflows with a machine-checkable definition of done — nightly
suite runs, routine dependency updates, scheduled report generation, bounded refactors.

This profile is *earned*, not chosen. The prerequisites are not advisory:

1. An eval suite that covers the failure modes, gating deploys in CI —
   [[skills/agent-evaluation/SKILL|agent-evaluation]].
2. A termination condition a shell script can decide with no judgment call —
   [[skills/loop-engineering/SKILL|loop-engineering]].
3. Deterministic enforcement in the harness, not the prompt —
   [[skills/agent-harness/SKILL|agent-harness]].
4. A sandbox (`microvm`, or `container` at minimum) for anything that executes code.
5. A configured `escalation` target. A loop with nowhere to page does not escalate; it
   stalls silently, which is the expensive failure.
6. Least-privilege tool scoping and an audit log of every invocation —
   [[skills/agent-safety/SKILL|agent-safety]].

## What this profile sets

| Setting | Value | Why |
|---|---|---|
| Approval gate | None in the happy path | That is the definition |
| `max-iterations` | Explicit and enforced, however high | Unbounded is never a valid value |
| `session-budget` | Explicit and enforced in code | Nobody is watching the spend either |
| Futility detection | Required | A live agent repeating one failure costs more than a dead one |
| Self-modification | Logged, versioned, eval-gated, human-reviewed | Behavior that changes future behavior always gets read |

## The honest framing

Running unattended is not a statement about the agent's competence. It is a statement about
the harness's coverage: you are asserting that everything the agent could do wrong is
either impossible, caught by verification, or bounded by budget. Autonomy is what the
harness *buys you*, and you do not get to claim it without having paid.
