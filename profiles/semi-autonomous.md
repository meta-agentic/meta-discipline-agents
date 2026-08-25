---
type: profile
tags: [pack, agents, profile, autonomy]
---
# Profile — semi-autonomous

**The agent runs freely inside its tool constraints and produces reviewable output; a human
approves before anything is committed or mutated.** This is the default, and it is where
most production agent work should live: near-full speed, with the irreversible step still
gated.

The distinction that matters is not "how much does the agent do" but "what can it do
without asking". Reading, searching, analyzing, generating a diff, running tests — all
unattended. Committing, merging, deploying, sending, spending, deleting — gated.

## What this profile sets

| Setting | Value | Why |
|---|---|---|
| Approval gate | Mutations and irreversible actions only | Reads and drafts are cheap to undo |
| `max-iterations` | Moderate (10–20) | Long enough to finish, short enough to notice a stall |
| Verification | Automatic, unconditional, in the harness | The human reviews verified output, not raw output |
| Escalation | On stall, budget breach, or repeated failure | Nobody is watching turn by turn |

## The failure mode specific to this profile

The gate erodes. Approvals become reflexive, the diff gets skimmed, and the mode quietly
becomes `autonomous` without anyone deciding to change it. Two defenses: keep the gated
action list explicit in the harness contract rather than in a habit, and read the
disagreements — where the builder and the adversarial reviewer differ — instead of reading
every green diff. See [[skills/loop-engineering/SKILL|loop-engineering]] on what still
deserves human eyes.
