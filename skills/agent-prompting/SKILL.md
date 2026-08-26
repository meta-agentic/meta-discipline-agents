---
name: agent-prompting
description: "Use when writing, restructuring, debugging, or reviewing an agent's system prompt — and whenever an agent drifts out of role, ignores a rule buried in its instructions, produces output downstream code can't parse, or starts behaving differently after an untracked prompt edit. Structures the prompt in four layers (identity, instructions, context, output contract), treats persona and reasoning budget as measured engineering choices, and puts the prompt under version control with regression tests. Emits a prompt version record."
---

# Agent prompting — the prompt as versioned code

A chat prompt is a conversation. An agent's system prompt is the operating instruction for
an autonomous process, and it fails the way software fails: silently, in production, in a
branch nobody tested. The work here is not better prose — it is giving the prompt a
structure that can be diffed, a persona that has been measured, and a test suite that fails
the build.

It sits between two neighbours. The shape decision settles what the agent *is*; this settles
what it is *told*; and any rule that has to hold rather than merely persuade gets handed on
to the harness, where it becomes code.
## Method

### 1. Stack the four layers, in order

Keeping them separate is what makes a failure diagnosable — when identity, rules, runtime
data and output format are one block, you cannot tell which rotted.

| Layer | Answers | Contains | Fails as |
|---|---|---|---|
| **1 · Identity** | Who am I, and where do I stop? | Role, standing commitments, explicit out-of-scope boundaries | Drift — useful at everything, reliable at nothing |
| **2 · Instructions** | What are the rules and the workflow? | Numbered procedure, tool rules, error handling, escalation | Skipped steps, invented recovery |
| **3 · Context** | What is true right now? | Injected runtime state: user, session, retrieved facts, prior summary | Stale or unbounded context |
| **4 · Output contract** | How must the answer be shaped? | Exact schema, required fields, what must never appear | Downstream parsers break on a change nobody made deliberately |

### 2. Assemble, don't concatenate

Layers 1, 2 and 4 are static and versioned; layer 3 is built per request. A single builder
function composing them in fixed order is the difference between a prompt you can test and a
string you can only observe.

Past roughly 800 tokens, structure stops being optional: long unstructured prose gets
treated as tone, while numbered rules and explicit do/don't lists get treated as a contract.
If instructions have grown past a few thousand tokens, that is not a prompting problem — move
procedures into [[skills/agent-skills/SKILL|agent-skills]].

### 3. Choose the persona by measurement

Role assignment moves real behavior: reasoning depth, willingness to commit, hedging,
verbosity and hallucination rate all shift with it. Because the difference is measurable,
the choice belongs in the eval suite, not in a kickoff meeting. Run variants against the
same cases and score quality, tool-call efficiency and hallucination rate *together* — the
intuitive favourite frequently loses, because a persona that produces confident, fluent
output produces more unsupported claims with it.

Where the output shape is complex, or the model repeats one specific mistake, use two or
three worked examples instead of another paragraph. Prose explains *why* a rule exists;
examples specify *what the result looks like*.

### 4. Match the reasoning budget to the task

| Task shape | Budget | Reasoning |
|---|---|---|
| Security review, architecture, hard debugging | High | The task has a right answer that checking will find |
| Implementation, refactoring | Medium | Enough to plan, not enough to spiral |
| Summarizing, drafting, formatting | Low | Extra deliberation mostly adds hedging |
| Classification, routing, extraction | Off | Latency is the product; there is nothing to deliberate |

**Maximum effort is not maximum quality.** Past the task's actual difficulty, more
deliberation makes a model second-guess correct conclusions. Verify the budget against the
eval suite like any other parameter.

### 5. Version it and gate it

Version the prompt with the code that uses it — a version string and changelog in the file,
so a running agent can report which prompt it is on. Every change gets a regression run
before it ships, asserting on behavior rather than prose: which tools must be called, which
keywords must appear, which must *not*, the maximum tool-call count, and whether an
out-of-scope request is refused. Gate on `eval-pass-rate` and `eval-hallucination-rate`; the
harness for that is [[skills/agent-evaluation/SKILL|agent-evaluation]].

Test the unhappy path. A suite where every case is well-formed and similarly phrased passes
at 100% and tells you nothing.

### 6. Move safety-critical rules out

A rule in the prompt is probabilistic; a rule in code is deterministic. Path validation,
tool allowlists, spend ceilings and mandatory verification do not belong in layer 2 at all.
The signal is unmistakable: if you find yourself strengthening wording — adding capitals,
adding "ALWAYS", adding "you MUST" — the rule is in the wrong layer of the system.

## The rigor standard

- **Four layers, kept separate.** A prompt that mixes them cannot have a failure attributed
  to a layer, which means it cannot be debugged, only rewritten.
- **No unversioned change reaches production.** A prompt without a version and changelog is
  a behavior surface nobody can attribute an incident to.
- **Every change runs regression before it ships**, and a regression is read as a blocker,
  not as noise.
- **Safety-critical rules live in the harness.** Instruction is persuasion; code is
  enforcement. Anything that must hold under adversarial pressure holds in code or not at
  all.
- **Persona is chosen against a measured alternative.** An unmeasured persona is a taste
  claim wearing an engineering claim's clothes.
- **The reasoning budget is per task class**, never a global default, and never assumed
  monotonic in quality.
- **Context injection is bounded.** A full transcript pasted into layer 3 is a retrieval
  step that was skipped.

## Checkable output

A prompt change ships a **prompt version record**, kept beside the prompt and updated in the
same commit:

```markdown
# Prompt version record — atlas-triage
- **Version:** 2.3  ·  **Previous:** 2.2
- **Changed:** layer 2 — split the escalation rule into three numbered conditions
- **Reason:** three production cases escalated on ambiguity instead of asking a
  clarifying question; the single combined rule was read as either/or
- **Regression:** triage-suite — 58/60 (was 57/60), hallucination 3% (was 4%)
- **Persona:** "senior support engineer" — chosen over "helpful assistant" on a
  measured 11-point drop in unsupported claims at equal quality
- **Reasoning budget:** low — classification task, latency is the product
- **Rules moved to the harness this revision:** refund ceiling (was "never exceed
  $50" in layer 2, now enforced in the tool)
```

The record fails review when the regression line is missing, when a persona change carries
no measurement, or when the reason names a preference rather than an observed failure.

## Anti-patterns

- **Escalating the wording instead of moving the rule.** Each capital letter added to a
  safety instruction is evidence the instruction cannot carry the weight.
- **The 5,000-token system prompt.** Every rule was reasonable when added; the aggregate is
  a document in which step two gets skipped because it sits on line 400.
- **Editing the prompt to fix one production case**, shipping it without a regression, and
  resurrecting a bug fixed two months ago.
- **A suite of happy-path cases.** It measures how well you imagine your users, not how the
  agent behaves.
- **Persona chosen for tone.** The fluent, confident voice usually costs accuracy, and the
  cost is invisible until it is measured.
- **Maxing the reasoning budget on everything** because more thinking sounds better —
  paying latency and tokens to make a model hedge on a question it had right.
- **Pasting the whole conversation into layer 3** and calling it memory. That is
  [[skills/agent-harness/SKILL|agent-harness]]'s job, and it involves forgetting.

## Configure

Reads `packs.agents.config` (`scripts/packs.sh config agents`), config-first with these
fallbacks:

| Key | Meaning | Default |
|-----|---------|---------|
| `eval-pass-rate` | Minimum regression pass rate below which a prompt change is blocked | `0.90` |
| `eval-hallucination-rate` | Maximum tolerated hallucination rate for the same gate | `0.05` |
| `trace-home` | Where regression output and version records are written | — |

Under `autonomy: autonomous` a prompt change is a self-modification of an unattended system:
it carries the same review requirement as any other, per `profiles/autonomous.md`.

## Related

[[skills/agent-harness/SKILL|agent-harness]] — receives every rule that must be enforced
rather than instructed. [[skills/agent-skills/SKILL|agent-skills]] — where instructions go
once the prompt outgrows them. [[skills/agent-evaluation/SKILL|agent-evaluation]] — owns the
suite this skill's regression step runs against.

*Grounding:* the four-layer architecture and prompt-regression discipline from *Building AI
Agents: From Design Patterns to Production* (Gulli); advanced prompting techniques from
*Agentic Design Patterns* (Springer 2025). Concepts only — see `PROVENANCE.md`.
