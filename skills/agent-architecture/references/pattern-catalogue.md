---
type: reference
tags: [pack, agents, patterns]
---
# Pattern catalogue — the 21 selection criteria

Loaded on demand from [[skills/agent-architecture/SKILL|agent-architecture]] when the four
reasoning patterns in the main table don't settle the shape. Each row is *when to reach for
it* and *what it costs you* — the compositional vocabulary of agent design, not an
implementation guide. Frameworks change; these choices don't.

Grounded in the pattern taxonomy of *Agentic Design Patterns* (Gulli, Springer 2025);
original wording, and the cost/failure columns are this pack's.

## Structural patterns — how work is decomposed

| Pattern | Reach for it when | Cost / failure mode |
|---|---|---|
| **Prompt chaining** | A task is too big for one prompt and splits into distinct stages, with tool calls or state between them | Each link is a failure point; errors compound silently down the chain |
| **Routing** | Input varies enough that different requests need different handling | A misroute is invisible — the wrong specialist answers confidently. Route on embeddings when the model adds nothing |
| **Parallelization** | Several sub-tasks are genuinely independent and results merge afterwards | Only pays when the branches don't need each other; a false independence assumption corrupts the merge |
| **Planning** | The request needs a sequence of interdependent steps to reach one synthesized outcome | The plan is made before reality is known — pair with a replan trigger |
| **Multi-agent collaboration** | The task decomposes into sub-tasks needing genuinely different expertise or an adversarial second opinion | Coordination tax: latency, spend, and a wider surface for one bad output to poison the rest |
| **Prioritization** | The agent juggles multiple competing goals under a resource constraint | Without an explicit ranking the agent optimizes whatever it saw last |

## Capability patterns — how the agent reaches the world

| Pattern | Reach for it when | Cost / failure mode |
|---|---|---|
| **Tool use** | The agent needs anything beyond its own weights: live data, private systems, exact computation, side effects | Every tool is attack surface and permission scope — see [[skills/agent-safety/SKILL|agent-safety]] |
| **Knowledge retrieval (RAG)** | Answers must rest on specific, current, or proprietary material, ideally with citations | Retrieval quality *is* answer quality; a dense-only retriever silently drops exact identifiers |
| **Model Context Protocol** | Tools must be shared across frameworks or discovered without redeploying | Overkill for a fixed handful of functions — direct function calling is fine there |
| **Inter-agent communication (A2A)** | Agents built on different stacks or owned by different teams must collaborate | A remote agent is an untrusted boundary; validate at every handoff |
| **Memory management** | The agent must hold context across turns, track multi-step progress, or recall preferences | Unmanaged memory becomes noise that misleads worse than silence — see [[skills/agent-harness/SKILL|agent-harness]] |

## Reasoning patterns — how the agent thinks

| Pattern | Reach for it when | Cost / failure mode |
|---|---|---|
| **Reasoning techniques** (CoT, self-consistency, tree search, ReAct) | One pass can't get there, and the reasoning path itself matters | Deeper reasoning is not free and not monotonic — past a budget it hedges rather than improves |
| **Reflection** | Output quality matters more than speed and cost, and first drafts reliably improve | Self-critique degenerates into self-approval; use a separate critic where objectivity matters |
| **Goal setting and monitoring** | The agent must pursue a high-level objective over many steps without supervision | A goal a machine can't check is not a goal — see [[skills/loop-engineering/SKILL|loop-engineering]] |
| **Exploration and discovery** | The solution space isn't defined and the point is to surface unknown unknowns | Unbounded by nature; needs an explicit budget and a stopping rule more than most |
| **Learning and adaptation** | The environment shifts and yesterday's behavior degrades | Every adaptation changes future behavior, so every one is a reviewable, eval-gated change |

## Operational patterns — how it survives contact with production

| Pattern | Reach for it when | Cost / failure mode |
|---|---|---|
| **Exception handling and recovery** | Anywhere real: tools time out, APIs return nonsense, inputs are malformed | A `try/except` around the whole loop is not recovery — model failures as routes, not crashes |
| **Human-in-the-loop** | Errors carry safety, legal, or financial weight, or the judgment is genuinely ambiguous | A gate everyone clicks through is not a gate; place few and mean them |
| **Guardrails / safety** | The output reaches users, systems, or your reputation | Guardrails run in code, not in the prompt — [[skills/agent-safety/SKILL|agent-safety]] |
| **Evaluation and monitoring** | Always, before deploy and continuously after | Behavior drifts when the model changes underneath you — [[skills/agent-evaluation/SKILL|agent-evaluation]] |
| **Resource-aware optimization** | Budget, latency, or constrained hardware bind the design | Big model for planning, small model for execution; a single model tier for everything overpays or underperforms |

## Using this catalogue

Two failure modes it exists to prevent:

1. **Naming a pattern instead of choosing one.** Reaching for "multi-agent" because the
   phrase fits the meeting is not a decision. The decision is in the middle column, and it
   must be true of *this* task.
2. **Stacking patterns without counting.** Each one added is latency, spend, and another
   place to debug. The architecture that survives is the smallest set whose selection
   criteria you can actually defend in the decision record.
