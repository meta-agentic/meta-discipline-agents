---
name: agent-harness
description: "Use when building or reviewing everything that surrounds the model — the execution boundary, sandboxing, tool interfaces, memory and state persistence, verification loops, and context pipelines. Reach for it whenever an agent wrote outside its workspace, skipped a check it was told to run, lost its progress on a restart, or drowned in a context window full of material it never needed. Turns prompt-level rules into deterministic code. Emits a harness contract."
---

# Agent harness — model proposes, code disposes

**Agent = model + harness.** The model reasons and proposes; the harness decides whether the
proposal executes. Every property worth having — safety, cost, reproducibility, the right to
run unattended — belongs to the harness rather than the model. The work here is converting
rules that need watching into rules that cannot be broken.

One test decides what belongs in this file: *if the agent were actively trying to violate
this rule, would it succeed?* If yes, the rule is currently living in a prompt, where it is
a probability, and it belongs here, where it is not.
## Method

### 1. Build the execution boundary

The checkpoint between "model wants to do X" and "X happens". Three rules govern everything
crossing it.

**Resolve paths before acting on them.** Compare the fully resolved target against the
workspace root and refuse anything outside. One function applied without exception closes
every variant at once — relative traversal, symlinks, creative string construction. The
common near-miss is a config write that resolves to a credentials file one directory up.

**Allowlists, never blocklists.** A blocklist always has a gap, and finding gaps is what a
non-deterministic system is good at. An allowlist is complete by construction.

**A validation failure is a message, not a crash.** Return the refusal to the model as a
readable error it can route around; raising an exception turns a recoverable moment into a
dead run.

Treat tools as any hostile-input boundary: assume arguments are adversarial, validate them,
and return a structured result — success flag, message, payload, and an error code the
caller can branch on — rather than a bare string the next step parses by guesswork. Scope
each tool to least privilege; see [[skills/agent-safety/SKILL|agent-safety]].

### 2. Sandbox execution

Never run agent-generated code on the host. Not as a default, not for a quick test. The
`sandbox` key names the level: `path` validation is the minimum for local work, `container`
is the working default, a per-task microVM is the production standard, and `none` is invalid
whenever a code-execution or shell tool exists at all.

Abstract the execution backend on day one. Swapping local execution for a remote sandbox is
an afternoon with one interface and a week with scattered `subprocess` calls — identical
work, different timing. The same layer carries the cost levers: a per-execution timeout, a
`session-budget` ceiling, and cold-start-per-task unless measured latency justifies a warm
pool.

### 3. Persist state — three memories and a scratchpad

| Kind | Holds | Lifetime | In an estate |
|---|---|---|---|
| Short-term | The working conversation | One session | Transient |
| Episodic | What happened in past runs, and how it went | Cross-session | Captures to `raw/` |
| Semantic | Durable domain knowledge | Permanent | Promoted to `wiki/` |

Start simple — a message list, a small key-value file, a basic vector store — and add
machinery only when a specific failure demands it. A knowledge graph built before there is a
relational query to serve is complexity with no customer.

Three disciplines carry most of the value. **The scratchpad**: context is finite and a
long-horizon run degrades as it fills, so give the agent a file to externalize state into —
the plan before a large change, findings after each step. It doubles as the correction
surface, since editing its notes beats arguing in the transcript. Prompt it to re-read and
re-condense, or it only ever reacts to the last message. **Retrieval is not one dial**:
chunk along the document's own structure rather than by character count, run keyword and
dense search together because dense-only silently drops exact identifiers and error codes,
and re-rank a wide candidate set down to a few — a gap left by a missed chunk is a gap the
model fills by invention. **Forget on purpose**: time-weight importance so old, never-recalled
entries stop competing with current ones. The design question is never how to store more,
but what deserves to be remembered.

For durability, checkpoint after each step and make tool calls idempotent — check whether
the effect already exists before producing it again. A retried run that double-creates is
worse than one that failed cleanly.

### 4. Run verification unconditionally

Run checks **inside the harness**, after every mutation. This is the layer most often gotten
subtly wrong: if verification is a tool the model chooses to call, it will eventually be
skipped under context pressure or optimism, and that run is the one that ships. The harness
runs it because the harness controls when it runs.

Feed failures back as structured errors and block the completion signal until they clear. A
human should only see output that already passed. Decide pass/fail on exit status, not by
matching strings.

The honest limit: verification proves output is well-formed, not correct. It confirms the
code parses and lints; it cannot confirm the refactor was the right one. That gap is what
[[skills/agent-evaluation/SKILL|agent-evaluation]] closes.

### 5. Pipeline the context

Do not preload every specification, schema and style guide. A model carrying forty thousand
tokens of reference has already lost the task. Expose a catalog and a tool to fetch one on
demand — the progressive disclosure [[skills/agent-skills/SKILL|agent-skills]] applies to
procedures, applied to reference material. The agent pulls the schema when it writes a query,
not when it renames a variable.

### 6. Write the contract down

The five layers are only real once they are stated in a contract the agent loads every
session and a reviewer can read. That artifact is the checkable output below.

## The rigor standard

- **Enforcement lives in code, not instruction.** Anything safety-critical stated only in
  the prompt is unenforced, however firmly it is worded.
- **Agent-generated code never runs on the host.** `sandbox: none` alongside a code or
  shell tool is a rejection, not a trade-off.
- **Commands are allowlisted.** A blocklist is a claim to have imagined every dangerous
  command.
- **Every write resolves its path first**, against a root, with no exceptions carved for
  convenience.
- **Verification is harness-run and unconditional**, and pass/fail comes from exit status.
- **An invalid proposal returns an error the model can read.** A harness that crashes on bad
  input trades a recoverable turn for a dead run.
- **Long runs checkpoint, and tools are idempotent** — otherwise a restart repeats side
  effects instead of resuming.
- **Tools return structured results.** A bare string cannot distinguish empty result from
  failure from refusal, so the caller guesses.
- **The execution backend is abstracted** from day one, or changing sandbox means touching
  agent logic.

## Checkable output

A harness ships a **harness contract**, kept in the repository it operates on and loaded
every session — one of the few artifacts always worth human eyes, per
[[skills/loop-engineering/SKILL|loop-engineering]]:

```markdown
# Harness contract — atlas-triage
## Identity
Triage assistant operating on ./workspace/ only.

## Capabilities
MAY: read any file under ./workspace/ · search the advisory feed ·
     run `python -m pytest ./workspace/tests/ -v` · write under ./workspace/output/
MUST NOT: write outside ./workspace/ · run unlisted commands ·
     make network calls beyond the advisory feed · delete without confirmation

## Boundary
- Sandbox: container  ·  Path validation root: ./workspace/
- Allowlisted commands: pytest, ruff, git status
- Tool permissions: read_advisory (ro) · search_codebase (ro) · post_issue (IRREVERSIBLE, gated)

## Persistence
- Trace: raw/agents  ·  Scratchpad: ./workspace/output/scratchpad.md  ·  Checkpoint: every step
- Memory: short-term messages · episodic → raw/agents · semantic → wiki/security

## Verification (harness-run, unconditional)
- After every write: syntax check, then ruff
- Done requires: ruff clean AND the advisory list fully assessed

## Output contract
Every mutating response reports: files changed and why · the command that verifies it ·
assumptions a human should check
```

A rule absent from this contract, or not enforceable by the code behind it, is not a rule.

## Anti-patterns

- **The strongly worded system prompt** standing in for a permission check — persuasion
  where enforcement was needed.
- **"Just this once" host execution** for a quick test, on the run that turns out not to be
  a test.
- **A blocklist of dangerous commands**, complete right up until the model composes one that
  is not on it.
- **Verification exposed as a tool.** Given the choice, a model under context pressure
  eventually skips the check, and it will be the run that matters.
- **Grepping command output for "error"** instead of reading the exit status, so a run that
  prints the word passes and a silent failure does not.
- **Crashing on a bad proposal**, discarding a run that one readable error message would
  have saved.
- **The context dump** — every schema and style guide preloaded, the original task buried
  under reference material.
- **Memory that only accumulates.** Without decay, stale facts outnumber current ones and
  mislead worse than silence.
- **`subprocess.run` scattered through the codebase**, making the sandbox a rewrite rather
  than a config change.

## Configure

Reads `packs.agents.config` (`scripts/packs.sh config agents`), config-first with these
fallbacks:

| Key | Meaning | Default |
|-----|---------|---------|
| `sandbox` | `none` \| `path` \| `container` \| `microvm` — where agent-generated code executes | `container` |
| `session-budget` | Per-session spend ceiling in USD, enforced between turns | `5.00` |
| `max-iterations` | Hard cap on loop turns before abort and escalate | `10` |
| `trace-home` | Where trajectory logs, scratchpads and checkpoints are written | — |

Under `autonomy: autonomous`, `sandbox: none` and `path` are both out of range — host
execution is not an option at that level (`profiles/autonomous.md`).

## Related

[[skills/agent-prompting/SKILL|agent-prompting]] — sends every safety-critical rule here.
[[skills/agent-safety/SKILL|agent-safety]] — decides *what capability exists* before this
skill decides how it is bounded. [[skills/agent-evaluation/SKILL|agent-evaluation]] — closes
the gap between well-formed and correct. [[skills/loop-engineering/SKILL|loop-engineering]] —
consumes the trace and checkpoints this layer produces.

*Grounding:* the harness layers and the *model proposes, code disposes* principle from
*Building AI Agents: From Design Patterns to Production* (Gulli), used as a named principle
with attribution; memory-management and RAG patterns from *Agentic Design Patterns*
(Springer 2025). Concepts only — see `PROVENANCE.md`.
