---
name: loop-engineering
description: "Use when an agent should run to a goal without a human re-prompting it after every failure — designing the trigger, the machine-checkable termination condition, structured failure feedback, context compaction, futility detection and escalation. Also covers running unattended (daemon loops, watchdogs, graceful shutdown, durable execution) and the self-improving harness. Reach for it when you catch yourself pasting an error back to an agent for the fourth time. Emits a loop charter."
---

# Loop engineering — the system is the loop

If you are reading an error, pasting it back, and waiting — then reading the next error —
you are not operating an autonomous system. You *are* its trigger, its verifier, its memory,
and its escalation path. The practitioner's job here is to remove themselves from those four
jobs, deliberately and in that order.

This is the third of three disciplines, and it stacks rather than replaces:

| Discipline | Where the loop lives | What it optimizes |
|---|---|---|
| **Prompting** | The human is the loop | The words |
| **Harness engineering** | The environment supports the loop | The workspace, boundary, verification |
| **Loop engineering** | The system *is* the loop | The termination condition and everything driving toward it |

A loop wrapped around a weak prompt in an unstructured environment automates the production
of poor output, faster. Get [[skills/agent-prompting/SKILL|agent-prompting]] and
[[skills/agent-harness/SKILL|agent-harness]] right first.

## Method

### 1. Design the verifier first

The most useful observation in this skill: in a well-designed loop the model is rarely the
constraint. Generation is cheap, fast and endlessly willing. The constraint is whatever
decides *whether the work is actually done*.

Get the verifier right and a mid-tier model grinds its way to a correct answer. Get it wrong
and the strongest model available loops forever, convinced it finished hours ago — or worse,
optimizes precisely against the flaw in your measure. Harden the check before upgrading the
generator, essentially always.

### 2. Write a termination condition a script could decide

Where most loops die, and they die at design time. The test is mechanical: *could a shell
script decide, with no judgment call, whether this condition holds?* "Improve the codebase"
fails — the agent can never know it is finished, so it never finishes. "The suite passes and
the linter is clean" succeeds. If the condition needs human judgment, fix the goal before
writing any of the loop.

### 3. Choose the trigger

A schedule, a failing pipeline, an opened pull request, a queue message. The mechanics are
ordinary; the intent is not. A daemon's trigger starts a *task*. An engineered loop's
trigger starts a *campaign* that does not stop until the goal is verifiably met or the loop
gives up on purpose.

### 4. Feed lessons forward, not transcripts

When an iteration fails, do not hand back a raw error dump. Process it: what was attempted,
how it failed, and what earlier attempts already ruled out. A dump makes the agent re-derive
the diagnosis every round; a summary lets it spend reasoning on what to try differently.
Attempt six should be smarter than attempt one, not merely further from the top of the
context window.

Compact on the same schedule. A loop running for hours overflows any context window if
history accumulates — failed attempts survive as one line each, not as transcripts. This is
the scratchpad from [[skills/agent-harness/SKILL|agent-harness]], run on a cadence.

### 5. Detect futility, not just death

A watchdog catches a *dead* agent. The harder and more expensive case is a live one making
no progress — retrying the same failing call, circling the same approach, billing the whole
time. Fingerprint each attempt (action, arguments, failure) and treat a repeat as a stop
condition: pause, persist state, escalate.

The loop must be able to give up deliberately. When it exhausts `max-iterations`, breaches
`session-budget`, or trips the futility guard, it persists state and pages `escalation`. A
loop with nowhere to page does not escalate — it stalls silently, which is the failure that
costs most before anyone notices.

### 6. Make it survive unattended operation

What kills long-running agents, in rough order of frequency: **unhandled exceptions** — catch
broadly in the main loop, log, move the failed item aside, continue; **memory growth** —
reset per-iteration state, or an agent fine for an hour exhausts a container overnight;
**network hangs** — set explicit timeouts, since an agent blocked forever is alive to the OS
and dead to you; **stuck states** — only completion-rate tracking catches these, which is why
step 5 is not optional.

Three primitives cover the rest. A **watchdog** is a separate process — an agent cannot watch
itself — monitoring liveness *and* progress, restarting with exponential backoff so a crash
loop does not become a rate-limit incident. **Graceful shutdown** catches the termination
signal, finishes or safely abandons the current item, flushes state, exits within the grace
period; without it the in-flight task fails silently on every deploy. **Durable execution**
means checkpointing after each step and idempotent tool calls, so a restart resumes rather
than repeats.

### 7. Let the harness learn — under gates

Once the loop runs itself, the next move is a harness that reads its own logs. Four
mechanisms are ordinary engineering: **critic-rewritten prompts**, where a cheaper model
reviews failure logs on a schedule and proposes a sharper instruction; **tool synthesis**,
where repeated deterministic computation becomes a registered function; **experience
distillation**, where a hard-won success is compressed into a reusable procedure so the next
similar task starts where the last finished — this is what makes a skill library compound,
and [[skills/agent-skills/SKILL|agent-skills]] owns the authoring and the bounded edits; and
**learned routing**, where success rates per model per task class stop being static config.

Their shared shape: **failure is inevitable, and the engineering goes into never failing the
same way twice.** Prevention by rigid rules loses to a non-deterministic system that finds a
new gap weekly; adaptation does not.

One firm boundary. A harness that rewrites its own prompts and mints its own tools is
self-modifying. Every mutation is logged, versioned, and gated by the same eval suite a
human's change would face — because when someone asks why behavior changed on a given date,
"it tuned itself" is only acceptable if you can produce the diff.

### 8. Spend your remaining attention where it scales

Reviewing less is not reviewing nothing. In priority order: **specs, goals and termination
conditions**, where judgment gets encoded and an error is multiplied by every iteration;
**eval diffs**, not results, because a weakened threshold is how a system rots while
dashboards stay green; **escalations**, each one the system reporting it hit something
outside its encoded judgment, and the most actionable queue you have; and
**self-modifications**, every rewritten prompt, synthesized tool and distilled skill.

One rule for everything else: **review the disagreements, not the output.** Where a builder
and an adversarial reviewer agree, the work is almost always fine. Where they disagree is the
most information-dense artifact the system produces — which is also why an adversary that has
never blocked anything is not an adversary, just a cost.

## The rigor standard

- **The verifier is named and justified.** A loop whose charter cannot say what decides
  "done", and why that is trusted more than the agent, will not stop.
- **Termination is script-decidable.** If a human must look at the result to know whether
  the loop is finished, the human is still in the loop.
- **Failures are compacted into lessons.** Feeding transcripts forward buys context
  exhaustion and pays for the same diagnosis repeatedly.
- **Futility is a stop condition**, distinct from death and more expensive to miss.
- **Escalation has a destination and carries state.** Paging nowhere, or paging without the
  state to resume, is stalling with extra steps.
- **Unattended means watchdog, timeouts, graceful shutdown and checkpoints** — all four, or
  the loop is a script that has not crashed yet.
- **Tools are idempotent across restarts**, or recovery duplicates side effects.
- **Self-modification is logged, versioned, eval-gated and human-reviewed**, without
  exception.
- **`autonomy: autonomous` is earned**, against the prerequisites in
  `profiles/autonomous.md`, not selected.

## Checkable output

A campaign ships a **loop charter**, written before the loop runs and read against every
escalation:

```markdown
# Loop charter — nightly dependency triage
- **Trigger:** cron 02:00 UTC, plus advisory-feed webhook
- **Termination condition:** every advisory in the feed window has a written verdict
  AND `pytest -q` exits 0  ·  Checkable by script: yes
- **Verifier:** the test suite plus a schema check on the verdict file — trusted over
  the agent because neither reads the agent's reasoning
- **Bounds:** max 12 iterations · budget $2.00 · wall-clock 40m
- **Feedback:** failures compacted to one line each (advisory id, approach, why it
  failed); ruled-out approaches carried forward
- **Compaction:** every 4 iterations, transcripts → lesson lines
- **Futility guard:** same (tool, args, error) fingerprint 3× → pause, persist, escalate
- **Escalation:** to #agents-oncall, carrying the scratchpad and the last 3 lesson lines
- **Unattended:** watchdog yes · timeouts 30s per call · shutdown handled (SIGTERM) ·
  checkpoints every advisory · idempotent tools verified (post_issue dedupes on title)
- **Self-modification:** none enabled
- **Autonomy:** semi-autonomous — post_issue gated; prerequisites for autonomous not
  yet met (no consensus judge on the verdict check)
```

The charter fails review when the verifier is unnamed, when termination needs judgment, or
when unattended operation is claimed with any of the four primitives missing.

## Anti-patterns

- **Being the loop and calling it automation** — the trigger, the verifier, the memory and
  the escalation path, all unpaid, at two in the morning.
- **"Improve the codebase" as a goal.** It has no stopping condition, so the loop either
  runs forever or stops arbitrarily and calls that done.
- **Upgrading the model to fix a loop that will not converge**, when the verifier is what is
  letting it declare victory.
- **Pasting the raw traceback forward** so attempt six re-derives what attempt two already
  learned.
- **Accumulating history until the context window decides the outcome.**
- **A watchdog that only checks liveness**, blind to the healthy process burning budget on
  one repeated failure.
- **A loop with no escalation target**, which does not fail loudly — it goes quiet, and quiet
  looks like working.
- **Skipping graceful shutdown** because deploys are rare, losing the in-flight task every
  time they are not.
- **A self-improving harness with no diff**, leaving you unable to explain your own system
  on the day it matters.
- **Reading every green result and no escalations** — attention spent where the system is
  already confident, and withheld where it admitted uncertainty.

## Configure

Reads `packs.agents.config` (`scripts/packs.sh config agents`), config-first with these
fallbacks:

| Key | Meaning | Default |
|-----|---------|---------|
| `max-iterations` | Hard cap on loop turns before abort and escalate | `10` |
| `session-budget` | Spend ceiling in USD, enforced between turns | `5.00` |
| `escalation` | Where a stalled or budget-exceeded loop pages a human | — |
| `trace-home` | Where charters, lesson lines and escalation state are written | — |
| `autonomy` | Selects the profile; `autonomous` makes every bound a prerequisite | `semi-autonomous` |
| `skill-review` | Gates distilled skills produced by step 7 | `required` |

## Related

[[skills/agent-harness/SKILL|agent-harness]] — supplies the scratchpad, checkpoints and
trace this skill runs on. [[skills/agent-evaluation/SKILL|agent-evaluation]] — the evaluator
problem there is this skill's verifier problem, and it gates every self-modification.
[[skills/agent-skills/SKILL|agent-skills]] — receives distilled experience as reviewable
procedures. [[skills/agent-architecture/SKILL|agent-architecture]] — required the termination
predicate this skill designs.

*Grounding:* loop engineering, the self-improving harness and the review-inversion argument
from *Building AI Agents: From Design Patterns to Production* (Gulli); goal-setting,
monitoring and exception-recovery patterns from *Agentic Design Patterns* (Springer 2025).
Concepts only — see `PROVENANCE.md`.
