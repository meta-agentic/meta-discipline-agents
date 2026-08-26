---
name: agent-evaluation
description: "Use when building, extending, or reviewing an agent's eval suite, and whenever an agent passes its tests but fails in production, gets the right answer by an expensive or erratic route, or starts behaving differently after a model or prompt change. Grades the trajectory rather than only the final answer, sets up judge-based scoring and CI gates, and treats red-team cases as permanent. Emits an eval report."
---

# Agent evaluation — grade the path, not just the destination

Grading an agent the way you would grade a chatbot — check the final answer, move on —
measures the one thing cheapest to get right by accident. An agent can reach a correct
answer through fifteen erratic tool calls instead of three, exhaust its budget doing it, and
still score full marks. The route is where the failure lives, so the work is to make the
route gradeable and then gate on it.

That depends on one thing the harness owes this skill: a persisted trajectory. Without a
recorded path there is nothing to grade but the output, which is chatbot evaluation again
under a different name.
## Method

### 1. Write the cases before the agent

Evals are a specification, not a post-launch health check. Writing them first forces the
question that spec-writing otherwise skips: what does "working correctly" actually mean
here? A behavior you cannot write a case for is one you do not yet understand well enough to
implement — the test telling you something at the cheapest possible moment.

### 2. Make each case assert on the trajectory

A case with only an expected output is a weak case.

| Assertion | Catches |
|---|---|
| **Required content** | The answer missing the substance it was asked for |
| **Forbidden content** | Confident drift into the wrong topic, or a refusal where none belongs |
| **Required tools** | The agent answering from memory when it was supposed to check |
| **Forbidden tools** | Reaching for a capability this task must never involve |
| **Maximum tool calls** | The expensive, erratic route to a correct answer |
| **Latency and cost ceilings** | Regressions invisible in output quality |
| **A reference fact** | The semantic check keyword matching cannot make |

Measure the same six on every run: correctness, tool efficiency, recovery when something
broke, hallucination rate, latency, cost. A suite reporting only a pass rate hides four of
the six ways an agent gets worse.

### 3. Grade nuance with a judge, under constraints

Keyword assertions break in both directions — a correct answer phrased without the expected
words fails, a wrong answer containing them passes. For cases needing semantic judgment,
grade with a model under three constraints:

**Reasoning before score, always.** A judge asked for a number first produces a number
anchored to nothing, then rationalizes it. Requiring reasoning first produces a grade
grounded in the content. The order is the whole technique.

**Ground the judge in reference facts**, not its own opinion of quality, and have it report
hallucination separately from correctness.

**Use a panel where it matters.** A single judge does not reliably agree with *itself* on
borderline cases; an odd-numbered panel at majority vote is what `judge: consensus` is for —
worth its cost on release gates and safety cases, wasteful on obvious ones.

**The evaluator is the design problem.** This generalizes past evaluation: any loop
optimizing against a measure optimizes against *exactly* that measure, loopholes included. A
weak evaluator is not fooled occasionally; it is exploited systematically. Effort spent
making the judge harder to satisfy pays better than effort spent making the generator
smarter.

### 4. Gate the build

A suite that runs when someone remembers is a dashboard. Wired into CI with thresholds it is
a regression net: assert on `eval-pass-rate` and `eval-hallucination-rate`, and let a deploy
that regresses either fail the build exactly as a compile error would.

Run it continuously, not once before launch. Behavior drifts when the model is updated
underneath you, when a prompt changes, and when real inputs diverge from the cases you
imagined — and you do not control when the first of those happens.

### 5. Keep red-team cases permanently

Jailbreak attempts, injection payloads, empty and malformed inputs, hostile phrasings,
character-set stress — these enter the suite and stay. A safety regression is always a deploy
blocker, never a flake to re-run. Every production incident ends by adding the case that
would have caught it, which is what makes the suite appreciate rather than decay.

### 6. Review the suite's diff, not its results

The suite protects the system, so changes *to it* deserve more scrutiny than its output. A
lowered threshold, a deleted red-team case, or a quietly loosened assertion is how a system
rots while every dashboard stays green. Results are for machines to gate on; diffs are for
humans to read.

## The rigor standard

- **Trajectory assertions are mandatory.** Output-only grading cannot distinguish a good
  agent from a lucky one.
- **Six metrics, every run.** Correctness alone hides cost, latency, efficiency and
  hallucination regressions.
- **Forbidden tools and forbidden content are specified**, not just required ones — most
  dangerous failures are things that should never have happened.
- **The judge reasons before it scores**, and grades against reference facts rather than
  taste.
- **Borderline safety and release calls go to a panel.** One judge's vote on an ambiguous
  case is a coin weighted by phrasing.
- **The suite gates CI with explicit thresholds.** Run manually, it measures nothing that
  can block anything.
- **Adversarial cases are permanent**, and their regressions are never re-run away.
- **The suite is written before the agent**, because it is the specification.
- **A threshold change or case removal is reviewed as a diff** with a stated reason.

## Checkable output

Each run ships an **eval report** to `trace-home`; the CI gate reads it:

```markdown
# Eval report — atlas-triage @ 2.3
- **Suite:** 60 cases (14 red-team) · **Run:** 2026-08-20T09:14Z · **Model:** claude-opus-5
- **Pass rate:** 96.7% (gate 0.90) · **Hallucination:** 3.3% (gate 0.05)
- **Tool efficiency:** median 3 calls (budget 6) · **p95 latency:** 11.2s · **Cost/case:** $0.031
- **Recovery:** 5 induced failures, 5 handled without escalation
- **Judge:** consensus(3) · disagreement on 2 cases (tc-031, tc-044)
- **Failures:** tc-018 — cited an advisory it never fetched (forbidden-content assert)
                tc-052 — 9 tool calls, exceeded max_tool_calls=6
- **Suite diff since last run:** +2 red-team cases (invisible-text injection); no
  thresholds changed
- **Verdict:** pass
```

A red verdict blocks the deploy. A suite diff with no accompanying reason blocks the merge.

## Anti-patterns

- **The 92% that means nothing.** A keyword pass rate over a well-formed suite, reported as
  readiness, while the agent takes five times the necessary calls to get there.
- **Evals written after launch**, so they document the behavior that exists rather than the
  behavior intended.
- **Asking the judge for a score first.** The number arrives before the reasoning and the
  reasoning becomes its justification.
- **A judge graded on vibes** rather than reference facts, rewarding fluency over
  correctness.
- **One judge deciding a release**, on exactly the borderline cases where it disagrees with
  itself.
- **Re-running a failed red-team case until it passes** and calling it flaky.
- **Loosening a threshold to unblock a deploy** — the fastest way to make every future
  dashboard meaningless.
- **Evaluating once and shipping**, then meeting model drift as an incident rather than a
  failed build.
- **Grading only what the agent said**, never what it did, in a system whose whole value is
  what it does.

## Configure

Reads `packs.agents.config` (`scripts/packs.sh config agents`), config-first with these
fallbacks:

| Key | Meaning | Default |
|-----|---------|---------|
| `eval-pass-rate` | Minimum suite pass rate below which a deploy fails | `0.90` |
| `eval-hallucination-rate` | Maximum tolerated hallucination rate | `0.05` |
| `judge` | `single` \| `consensus` (odd-numbered panel, majority vote) | `single` |
| `trace-home` | Where eval reports and trajectories are written | — |

Under `autonomy: autonomous` this suite is a prerequisite rather than a practice — the loop
has no human in the path, so the gate is the only thing that stops a regression shipping
(`profiles/autonomous.md`).

## Related

[[skills/agent-harness/SKILL|agent-harness]] — supplies the trajectory this skill grades,
and stops at well-formed where this skill starts at correct.
[[skills/agent-prompting/SKILL|agent-prompting]] — prompt regression runs against this
suite. [[skills/agent-safety/SKILL|agent-safety]] — supplies the red-team cases.
[[skills/loop-engineering/SKILL|loop-engineering]] — the evaluator problem here is the
verifier problem there, at a different scale.

*Grounding:* trajectory evaluation, judge discipline and CI gating from *Building AI Agents:
From Design Patterns to Production* (Gulli); evaluation and monitoring patterns from
*Agentic Design Patterns* (Springer 2025). Concepts only — see `PROVENANCE.md`.
