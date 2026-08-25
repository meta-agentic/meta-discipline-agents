---
name: agent-safety
description: "Use when deciding what an agent is permitted to do, and what stands between it and the user — input and output guardrails, tool permission scoping, the threat model for agents that read external data, and where a human gate belongs. Reach for it before granting any tool that writes, sends, spends, or deletes; before an agent processes fetched content (email, web pages, documents, other agents' output); and after any incident. Emits a threat and permission ledger."
---

# Agent safety — guardrails, permissions, and the threat model

Two failure classes needing different machinery. **Guardrails** stop bad content crossing
the boundary in either direction. **Permissions** decide what the agent is capable of at
all. Guardrails are inspection; permissions are architecture. A practitioner who builds only
guardrails ends up inspecting an agent for misuse of a capability it never needed.

Decides *what capability exists*; [[skills/agent-harness/SKILL|agent-harness]] then decides
how that capability is bounded in code.

## Method

### 1. Guard both directions, cheapest layer first

| Type | When | Catches |
|---|---|---|
| **Input** | Before the agent processes anything | Injections, jailbreaks, out-of-scope requests |
| **Output** | Before a response leaves the system | Leaked personal data, unsupported claims, harmful content |

Both are required; a system with one has a documented open side. Order them by cost: pattern
checks are free and run in a millisecond, a small classifier costs a fraction of a cent, and
the main agent is the most expensive component in the stack. Anything blocked early never
pays for the later layers.

That ordering also sets an architectural constraint: a model-as-judge check cannot sit
synchronously in front of every request — the latency and cost are real — so it belongs on
sampled traffic, high-risk paths, or after the fact.

Personal-data filtering on output is the one check with no performance argument against it.
It is nearly free and catches the category of leak with the most legal exposure. An agent
asked for an account balance that helpfully attaches every identifier it retrieved is not
hypothetical; it is the ordinary behavior of a system asked to be helpful with more context
than it should have had.

### 2. Model the threat where it actually arrives

The dangerous assumption is that the attack is in the user's message. Once an agent reads
mail, fetches pages, opens documents, or consumes another agent's output, **the attack
surface moves to the data**, and filters pointed at the user's prompt are looking the wrong
way.

- **Indirect prompt injection.** Instructions hidden in fetched content — invisible text, a
  markup comment, an unusual character range — read as part of the material the agent was
  asked to process. It does not experience this as an attack. Scan every piece of external
  content *before* it enters the context window.
- **The confused deputy.** The agent holds credentials and tool access the attacker does
  not. Nobody needs to breach your network if they can get your agent to act with its own
  authority on their behalf. This is what makes least privilege structural.
- **Non-human identity.** An agent with its own keys is an identity with no second factor,
  no manager who notices it behaving oddly, and no working hours. Compromised, it is an
  insider threat at machine speed.
- **Cascading compromise.** In a pipeline, one manipulated agent taints everything
  downstream, and corruption propagates faster than anyone reviews. Validate at every
  handoff, not only at the system boundary.

### 3. Scope permissions to least privilege

Five commitments, applied at the registry rather than in the prompt:

1. **Never trust agent output by default** — validate at every boundary it crosses.
2. **Least privilege per tool** — read-only unless writing is genuinely required. A
   summarization agent does not get a send capability; remove it, so the system prompt is
   not what stands between you and the incident.
3. **Scope identity per agent** — minimum credentials for that agent's job, not a shared
   service account with everything.
4. **Log every tool invocation** — the full trail, not just final outputs. An audit that
   records only answers cannot reconstruct what happened.
5. **Rate-limit high-risk actions** — a circuit breaker on outbound volume, spend, or
   deletion, because machine speed turns a small logic error into a large one before anyone
   reads an alert.

### 4. Place the human gate

Two rules place it. **Gate on irreversibility, not importance**: anything that mutates state
a user depends on, spends money, sends outbound communication, or cannot be undone gets a
checkpoint before execution — at every `autonomy` level, including `autonomous`. **Do not
let the agent widen its own permissions**: an agent requesting a broader capability mid-run
because it would be more efficient is describing a design change, and that is a conversation
with a human.

Place few gates and mean them. A gate approved reflexively has the cost of a gate and the
protection of none.

### 5. Start supervised, graduate on evidence

Every new tool, capability, codebase, or model version begins under supervision whatever
`autonomy` says. Watch the first several decisions to learn where the agent is *uncertain* —
that is what tells you which harness layer is missing. Correctness can be tested later;
uncertainty is the design signal. Graduating out is evidence-backed: the eval suite covers
the failure modes observed, and the harness now enforces what you were enforcing by hand.

## The rigor standard

- **Both directions are guarded.** Input-only or output-only is a half-built boundary, and
  which half is missing determines only which incident you get.
- **External content is scanned before it enters context.** Filtering the user's prompt
  while trusting fetched mail is the 2026 version of validating the form but not the API.
- **Capability is removed, not discouraged.** A tool the agent does not have cannot be
  talked into use; a tool it has and is told not to use is one persuasive document away.
- **Credentials are per agent.** A shared service account makes the audit log unable to
  answer who did it.
- **Every tool invocation is logged**, not just outcomes.
- **Irreversible actions are gated at every autonomy level.** `autonomous` describes the
  happy path, not the destructive one.
- **The agent never acquires capability at runtime.**
- **High-risk actions have a circuit breaker** with a threshold someone chose deliberately.
- **A judge-based check is never the synchronous front door** — that is a latency and cost
  decision disguised as a safety one.

## Checkable output

A deployment ships a **threat and permission ledger**, written before launch and revisited
after every incident:

```markdown
# Threat & permission ledger — atlas-triage
## Tools
| Tool | Scope | Reversible? | Gate | Rate limit |
|---|---|---|---|---|
| read_advisory | read-only, feed only | yes | none | 60/min |
| search_codebase | read-only, ./workspace | yes | none | 120/min |
| post_issue | create only, one repo | NO | human approval | 5/hour, breaker at 10/day |

## Identity
- Credentials held: advisory feed token (ro), repo issues token (create-only)
- Shared with: none  ·  Audit log: raw/agents/tools.jsonl  ·  Retention: 90 days

## External content
- Ingested from: advisory feed, linked CVE pages
- Scanned before context entry: yes — invisible-text and instruction-pattern scan
- Agent-to-agent input validated at handoff: n/a (single agent)

## Guardrails
- Input: pattern scan → intent classifier → agent
- Output: personal-data filter → unsupported-claim check
- Judge-based checks: sampled at 10%, plus every run that proposes an issue

## Autonomy
- Level: semi-autonomous  ·  Supervised period: 14 runs; 3 near-misses, all on
  advisory pages containing quoted attacker instructions
- Human gates: post_issue  ·  Circuit breakers: 10 issues/day, $2.00/session

## Reviewed: 2026-08-20 · after incident: none
```

The ledger fails review when a tool's reversibility is unstated, when external content has
no scan, or when a gate exists with no breaker behind it.

## Anti-patterns

- **Filtering the user and trusting the data.** The prompt box stopped being the attack
  surface the moment the agent could read anything.
- **The permissive service account** shared across agents, so the audit trail can say what
  happened but never who.
- **A tool retained "in case it's useful later"** — capability with no current task is
  attack surface with no owner.
- **The system prompt as permission model**: telling an agent not to use a tool it holds.
- **Granting a capability mid-run** because the agent argued it would be more efficient.
- **Gates everywhere**, approved reflexively, so the one that mattered was clicked through
  like the rest.
- **Judge-model checks on the synchronous path**, discovered in the latency graph rather
  than the design review.
- **Jumping to the configured autonomy level on a new capability** and calling the
  supervised period a formality.
- **Logging outcomes only**, so an incident review reconstructs intent from final answers.

## Configure

Reads `packs.agents.config` (`scripts/packs.sh config agents`), config-first with these
fallbacks:

| Key | Meaning | Default |
|-----|---------|---------|
| `autonomy` | `supervised` \| `semi-autonomous` \| `autonomous` — the blast-radius knob | `semi-autonomous` |
| `sandbox` | Bounds code execution; `none` is invalid with a code or shell tool | `container` |
| `session-budget` | Spend ceiling, doubling as the cost circuit breaker | `5.00` |
| `trace-home` | Where the tool audit log and this ledger are written | — |
| `escalation` | Where a gate or breaker pages a human | — |

Each `profiles/<autonomy>.md` states what that level requires; `autonomous` lists the
prerequisites that must be met before it may be selected at all.

## Related

[[skills/agent-harness/SKILL|agent-harness]] — makes structural what this skill would
otherwise only inspect. [[skills/agent-evaluation/SKILL|agent-evaluation]] — red-team cases
are permanent, and a safety regression always blocks.
[[skills/agent-skills/SKILL|agent-skills]] — a writable skill directory is itself an
injection vector. [[skills/loop-engineering/SKILL|loop-engineering]] — escalations are the
backlog of missing guardrails.

*Grounding:* the agentic threat model, layered guardrails and zero-trust scoping from
*Building AI Agents: From Design Patterns to Production* (Gulli); guardrail and
human-in-the-loop patterns from *Agentic Design Patterns* (Springer 2025); the OWASP
Foundation's agentic top-ten as a coverage checklist. Concepts only — see `PROVENANCE.md`.
