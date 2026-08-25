---
type: profile
tags: [pack, agents, profile, autonomy]
---
# Profile — supervised

**Every proposed action is shown to a human before it executes.** The agent reasons, plans
and proposes; a person approves each tool call that touches anything outside the context
window.

This is not a training-wheels mode you graduate out of once and for all. It is the correct
setting *per capability*, and you re-enter it every time one of these is true:

- a new codebase, estate, or data domain the agent has not worked in
- a newly registered tool, especially one that writes, deletes, sends, or spends
- any irreversible action class — migrations, infrastructure, auth, payments, outbound mail
- the first ten runs after a model change, since behavior moves when the model does

## What this profile sets

| Setting | Value | Why |
|---|---|---|
| Approval gate | Every tool call with a side effect | The point of the mode |
| `max-iterations` | Low (3–5) | You are reading each turn; long runs defeat supervision |
| Verification | Runs, and the human sees the result | You are calibrating what the harness catches |
| Escalation | Immediate — you are already there | No paging needed |

## What you are actually doing here

Watching the first ten decisions to find out where the agent is *uncertain*, not whether it
is *correct*. Correctness you can test later; uncertainty is what tells you which harness
layer is missing. Every approval you find yourself granting mechanically is a rule that
should move into code — see [[skills/agent-harness/SKILL|agent-harness]]. Every approval
that made you hesitate is an escalation condition worth encoding.

Leaving this profile is a decision with evidence behind it: the eval suite covers the
failure modes you saw, and the harness enforces the rules you were enforcing by hand.
