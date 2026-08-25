---
name: agent-architecture
description: "Use before building or reviewing anything that calls a model in a loop: decides whether the task needs an agent at all, then what shape it takes — single loop, handoff router, state graph, or multi-agent crew — and which of the four reasoning patterns (ReAct / Chain-of-Thought / Reflection / Plan-and-Execute) fits. Also use when an existing agent thrashes, stalls, burns budget, or nobody can say why it did what it did. Emits an auditable agent decision record."
---

# Agent architecture — the shape decision

A practitioner reaching for this skill is about to commit to a structure, and structure is
the most expensive thing to change later. The work is to settle three questions in order —
*should this be an agent, how much structure does it justify, and how does it reason* — and
to write the answers down where a reviewer can challenge them.

This is the pack's entry point. It settles *shape*; the siblings fill that shape in — the
instruction layer ([[skills/agent-prompting/SKILL|agent-prompting]]), the procedure library
([[skills/agent-skills/SKILL|agent-skills]]), the deterministic enforcement
([[skills/agent-harness/SKILL|agent-harness]]), the permissions
([[skills/agent-safety/SKILL|agent-safety]]), the grading
([[skills/agent-evaluation/SKILL|agent-evaluation]]) and the loop around it all
([[skills/loop-engineering/SKILL|loop-engineering]]).

## Method

### 1. Apply the refusal test

Run this first, every time. Most proposed agents should not exist.

> **If the whole task draws as a flowchart whose branches never depend on model output, it
> is not an agent. Write the script.**

| Write a plain pipeline | Build an agent |
|---|---|
| Steps known in advance, and fixed | The next step depends on what the last one returned |
| Execution order must be guaranteed | The request is ambiguous and may need clarification |
| Latency or unit cost matters — every turn is an inference call | Which tools are needed isn't known until runtime |
| CRUD, ETL, formatting, deterministic validation | The work measurably improves under self-correction |

Failing this test and building anyway is the most expensive mistake in the domain, because
you pay for it on every run forever: slower, costlier, non-deterministic. **A "no" here is a
successful outcome of this skill, not a failure to deliver.**

Partial credit is the common answer. Hoist the deterministic parts into ordinary code and
leave the agent only the branches that genuinely need judgment. A small agent core wrapped
in a plain pipeline beats an agent that re-derives your business logic hourly.

### 2. Take the lowest rung of the ladder that fits

Each rung buys a capability and charges for it in latency, spend, and debuggability. Teams
routinely start three rungs too high and spend a week discovering it.

| Rung | Shape | Take it when | The tell that you climbed too early |
|---|---|---|---|
| 1 | **Single loop** — one agent, one tool set | One agent can carry the task end to end | — |
| 2 | **Handoff router** — a router plus narrow specialists, no shared state | Each request maps cleanly to exactly one specialist; sub-tasks are independent | You built a graph and every request visits exactly one node |
| 3 | **State graph** — nodes, edges, an explicit shared state object | The work branches, loops, retries, or needs a human checkpoint mid-flight | Your graph has no cycle and no conditional edge |
| 4 | **Multi-agent crew** — several agents with genuinely distinct roles | Roles are actually different jobs, and at least one is adversarial to another | Six agents whose system prompts differ by one adjective |

Two rules decide most of these. **A linear loop cannot retry** — the moment self-correction,
escalation, or a mid-flight human gate enters the requirements, there is nowhere in a flat
loop to put it, and that is the honest trigger for rung 3, not the size of the task.
**Count the coordination tax before adding an agent** — each one multiplies latency, spend,
and the surface where one bad output poisons everything downstream. Routing does not always
need a model at all: embedding similarity dispatches in a millisecond for a fraction of the
cost.

### 3. Choose the reasoning pattern

| Pattern | Loop shape | Reach for it when | Cost shape | Characteristic failure |
|---|---|---|---|---|
| **ReAct** | think → act → observe, one step at a time | Research, Q&A, exploration — the tool path is unknown in advance | One inference per step; grows with the task | Thrash: re-searching the same thing, converging on nothing |
| **Chain-of-Thought** | reason through fully, then act | Closed-form multi-step reasoning where the plan is derivable up front | Cheapest — one or two calls | A wrong premise early is never revisited |
| **Reflection** | generate → critique → revise | Writing and code generation — anywhere first drafts reliably improve | 2–3× per artifact | Sycophantic self-approval, or polishing forever |
| **Plan-and-Execute** | plan fully, then execute steps mechanically | Scope is clear up front, steps are near-independent, a reviewable plan has value | Plan + n steps, predictable | Brittle when reality diverges mid-plan |

Three rules outrank the table. **They compose** — Plan-and-Execute with a small ReAct loop
inside each step, closed by a Reflection pass, is a normal architecture; pick a primary
shape, then name the composites deliberately. **Plan-and-Execute needs a replan trigger**,
or it executes a plan it already knows is wrong. **Add a Reflection pass to code generation
by default**, and where being wrong is expensive — legal, medical, security, financial —
promote it to an explicit adversarial pass: one agent drafts, a second attacks, repeat until
the attacker runs out of objections or the round cap hits. An adversary that has never
blocked anything is not an adversary.

When those four don't settle it — the task is really about parallel decomposition,
prioritization under constraint, open-ended discovery, or retrieval quality — load
[[skills/agent-architecture/references/pattern-catalogue|references/pattern-catalogue.md]]:
21 patterns with a selection criterion and a cost for each.

A note on reasoning models: when the model does extended thinking internally, it wants one
large context and a plan, not a long chain of small ReAct turns. Use the graph as the
execution harness for the plan the model already produced, rather than as the reasoning
engine.

### 4. Bound the loop

Ship none of these unbounded, whatever the shape.

| Invariant | Config key | What its absence looks like in production |
|---|---|---|
| Hard iteration cap | `max-iterations` | Infinite recursion; the same output emitted six times |
| Budget ceiling with a kill | `session-budget` | A runaway run discovered on the invoice |
| Explicit termination predicate | — | Confident stops on half-finished work |
| Persisted reasoning trace | `trace-home` | "It did something weird once" and no way to find out why |
| No-progress detector | — | Same tool, same args, three turns running |
| Approval gate on irreversible acts | `autonomy` | Email sent, record deleted, money moved |

Reversibility is the axis that matters most: prefer tools that are idempotent or undoable,
and gate everything else. The cap is the floor of loop discipline, not the ceiling — the
full treatment is [[skills/loop-engineering/SKILL|loop-engineering]].

### 5. Record the decision

Write the agent decision record below before any code exists. A design that cannot fill its
lines is not ready to build.

## The rigor standard

- **The refusal test is recorded, not just performed.** A design with no written Gate 0
  verdict cannot be challenged later, and "we considered it" is not a verdict.
- **The rung must be justified by the design's own description.** A graph with no cycle, a
  crew with no adversarial role, or a router in front of one specialist is a rung claimed
  and not earned.
- **The reasoning pattern follows task structure, never framework default.** If the answer
  to "why ReAct" is "that's what the example used", the choice has not been made yet.
- **Plan-and-Execute without a replan trigger is incomplete**, not merely rigid.
- **Every loop is bounded before it ships** — `max-iterations` and `session-budget` are
  code, not intentions, and there is no valid value of *unbounded*.
- **Termination is a predicate, not the model's opinion.** "It said Final Answer" is an
  observation about text, not a check.
- **Irreversible actions are gated at every `autonomy` level**, including `autonomous`.
- **No single provider is the only path.** A hardcoded model string makes a vendor outage a
  total outage.

## Checkable output

A shape decision ships an **agent decision record** to `trace-home`, promoted to the
instance's `wiki/` once the design settles. It is what a reviewer reads instead of
re-litigating the design:

```markdown
# Agent decision record — nightly dependency triage
- **Gate 0:** hybrid — the fetch/parse/report path is deterministic and stays in code;
  only "is this CVE relevant to us" needs judgment
- **Rung:** single loop — rejected rung 3 (no retry requirement, no human checkpoint)
- **Pattern:** ReAct + Reflection pass on the final summary — tool path is unknown per CVE
- **Tools:** read_advisory (ro) · search_codebase (ro) · post_issue (IRREVERSIBLE, gated)
- **Memory:** short-term messages · episodic → raw/agents · semantic → wiki/security
- **State:** typed object — {advisories, assessed, findings, cursor}
- **Invariants:** max 12 iterations · budget $2.00 · terminates when every advisory is
  assessed · trace to raw/agents · approval gate on post_issue
- **Replan trigger:** n/a (not Plan-and-Execute)
- **Autonomy:** semi-autonomous — eval suite covers 14 triage cases, harness enforces
  read-only codebase access
- **Review by:** first model upgrade, or 2026-11-01
```

The record fails review when a line is absent, when the rung is not justified against the
one below it, or when an invariant names an intention rather than a config value.

## Anti-patterns

- **Skipping Gate 0 because the request said "agent".** The word in the ticket is not the
  analysis, and the analysis is the cheapest hour in the project.
- **Choosing the shape from the framework you already installed.** Tooling is not a
  requirement; picking a graph library first and finding a use for the graph second is how
  a bicycle acquires a jet engine.
- **Adding agents to add capability.** More brains are not smarter — a narrow brain is
  harder to fool. Two agents with different jobs beat six with overlapping ones.
- **Routing with a model where embeddings would do**, paying inference latency on every
  request for a decision a similarity score already made.
- **Treating the iteration cap as the whole of loop discipline.** A capped loop that repeats
  one failing action ten times has burned the budget and learned nothing.
- **Deferring the decision record until "the design settles".** It settles by being written;
  an undocumented architecture is re-argued at every review.
- **A Reflection pass that only ever approves.** Self-critique with no rejection is cost
  with no signal — see the adversarial framing in step 3.

## Configure

Reads `packs.agents.config` (`scripts/packs.sh config agents`), config-first with these
fallbacks:

| Key | Meaning | Default |
|-----|---------|---------|
| `max-iterations` | Hard cap on loop turns before abort and escalate | `10` |
| `session-budget` | Per-session spend ceiling in USD, enforced in code | `5.00` |
| `autonomy` | `supervised` \| `semi-autonomous` \| `autonomous` — sets where approval gates sit | `semi-autonomous` |
| `trace-home` | Where the decision record and trajectory logs are written | — |

Under `autonomy: autonomous` the invariants in step 4 stop being defaults and become
prerequisites — see `profiles/autonomous.md`.

## Related

[[skills/agent-harness/SKILL|agent-harness]] — the shape decided here is only real once the
harness enforces it. [[skills/loop-engineering/SKILL|loop-engineering]] — owns the
termination condition this skill only requires to exist.
[[skills/agent-evaluation/SKILL|agent-evaluation]] — the eval suite is what lets a rung or
an autonomy level be raised on evidence.

*Grounding:* pattern taxonomy from *Agentic Design Patterns* (Gulli, Springer 2025);
operational discipline from *Building AI Agents: From Design Patterns to Production*.
Concepts only — see `PROVENANCE.md`.
