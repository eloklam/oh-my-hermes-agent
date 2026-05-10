# OMH multi-profile workflow design

This document describes how OMH behavior maps to Hermes primitives.

It is not an implementation of a new swarm runtime.

## Runtime boundary

Canonical runtime:

```bash
hermes
hermes --profile <role>
hermes kanban --board oh-my-hermes-agent ...
```

Nothing in this repo should wrap or replace Hermes.

## Named-profile delegation

OMH `@agent` delegation maps to Hermes Kanban plus Hermes profiles. If a task should go to `@fixer`, `@oracle`, `@explorer`, `@librarian`, `@designer`, `@council`, or `@observer`, create a Kanban task assigned to that profile and dispatch the board.

Use Kanban when:

- the user asks for a named specialist
- the task should preserve profile context/state
- the work may outlive the current turn
- dependencies, retries, logs, or audit trail matter
- multiple workers should fan out/fan in

## Decomposition rule

The default orchestrator must decompose non-trivial requests into independent workstreams before creating Kanban tasks. Do not bundle multiple independent lanes into a single fixer task.

Examples of correct decomposition:

- Build a feature with UI/UX:
  - designer task (parallel)
  - fixer task that depends on designer (linked)
- Fix blockers and research alternatives:
  - fixer task (parallel)
  - explorer task (parallel)
  - librarian task (parallel)
- Optional adversarial review:
  - fixer task
  - oracle task that depends on fixer

Use `hermes kanban --board oh-my-hermes-agent link <parent> <child>` to express dependencies.

## No self-execution rule

The default orchestrator must not implement non-trivial work itself. It is a delegator, not an executor.

Work that must be delegated via Kanban:

- GitHub PR and issue workflows
- Repository edits, file creation, or code changes
- Implementation, bug fixes, or tests
- Research, documentation, or evidence gathering
- Architecture or security review
- Config or profile changes

Work the orchestrator may handle directly:

- Simple factual replies and quick lookups
- Tiny edits (single-line typos, trivial corrections)
- Direct checks and verification commands
- Clarification questions to the user

### Wrong vs correct

Wrong — orchestrator does issue + PR + review in one turn:

```text
User: "Document the new rate limiter and open a PR"
Orchestrator: (creates issue, writes doc, commits, opens PR, self-reviews, merges)
```

Correct — parallel cards with real profiles:

```text
User: "Document the new rate limiter and open a PR"
Orchestrator:
  - fixer task: "Write rate limiter docs in docs/rate-limiter.md"
  - explorer task: "Check for existing rate limiter references" (parallel to fixer)
  - librarian task: "Collect source-grounded evidence for rate limiter design" (parallel to fixer)
  - oracle task: "Review rate limiter docs PR" (depends on fixer)
  - link fixer -> oracle
  - dispatch
```

### Stalled-worker recovery

If a worker task stalls (timed_out, crashed, blocked), do not silently finish the work yourself.

Recovery options, in order:

1. Reclaim the task so the dispatcher can retry it with the same or narrower scope.
2. Reassign to a different profile if the original profile is misconfigured.
3. Create a follow-up child task for the remaining work.
4. Block the task and ask the user for a decision if the stall reason is ambiguous.

Example reclaim:

```bash
hermes kanban --board oh-my-hermes-agent reclaim <task_id>
```

## delegate_task is not part of this OMH setup

Hermes `delegate_task` is disabled for the default orchestrator and all OMH worker profiles. It does not load a named Hermes profile, so it must not be used for specialist delegation.

If temporarily re-enabled for debugging, use it only when:

- the subtask is small
- the parent should wait for a short result
- durability and named profile state are not needed
- the user did not request a named specialist

Do not leave it enabled in normal OMH operation. It creates generic temporary subagents and can be confused with real `@fixer`/`@oracle` profile routing.

Properties:

- temporary child agent
- isolated context
- not persisted as profile state
- cancelled if parent turn is interrupted

## Disable delegation for all OMH profiles

Run these once per profile to remove the `delegate_task` tool:

```bash
hermes --profile default  tools disable delegation
hermes --profile fixer    tools disable delegation
hermes --profile explorer tools disable delegation
hermes --profile librarian tools disable delegation
hermes --profile designer tools disable delegation
hermes --profile oracle   tools disable delegation
hermes --profile observer tools disable delegation
hermes --profile council  tools disable delegation
```

Verify:

```bash
hermes --profile fixer tools list | grep delegation
```

Expected: a `✗ disabled delegation` line.

## Durable profile workflow

OMH durable workflow maps to Hermes Kanban plus Hermes profiles.

Each task is assigned to a Hermes profile:

```text
explorer   discovery
librarian  evidence and docs
fixer      bounded implementation
oracle     review
designer   UI/UX lane
council    multi-model consensus lane
observer   vision lane
casual     fast chat buddy, not part of serious OMH worker delegation
```

Default dependency shape:

```text
explorer   ─╢
                ─╮> fixer ─> oracle
librarian ─╯
```

Canonical commands with explicit links:

```bash
# Create tasks
EXP=$(hermes kanban --board oh-my-hermes-agent create "Explore implementation surface: mission" --assignee explorer --workspace dir:/repo --json | jq -r '.id')
LIB=$(hermes kanban --board oh-my-hermes-agent create "Collect source-grounded references: mission" --assignee librarian --workspace dir:/repo --json | jq -r '.id')
FIX=$(hermes kanban --board oh-my-hermes-agent create "Implement bounded changes: mission" --assignee fixer --workspace dir:/repo --json | jq -r '.id')
ORA=$(hermes kanban --board oh-my-hermes-agent create "Final adversarial review: mission" --assignee oracle --workspace dir:/repo --json | jq -r '.id')

# Link dependencies: explorer and librarian parent fixer; fixer parents oracle
hermes kanban --board oh-my-hermes-agent link "$EXP" "$FIX"
hermes kanban --board oh-my-hermes-agent link "$LIB" "$FIX"
hermes kanban --board oh-my-hermes-agent link "$FIX" "$ORA"

# Dispatch
hermes kanban --board oh-my-hermes-agent dispatch
```

## Cost-control and checkpointing rules

To prevent avoidable cost and poor UX during multi-profile Kanban execution:

1. **No long blocking polling loops** — The orchestrator must never run long foreground wait/poll loops that prevent replying to the user. Use `kanban_heartbeat` every few minutes for genuinely long operations, and keep the user updated instead of blocking silently.
2. **Checkpoint after each meaningful batch or blocker** — After each batch of work or any high-risk blocker, stop and give a status checkpoint instead of silently continuing the entire task graph.
3. **No auto-cascade of oracle/GPT review after blockers** — Do not automatically spawn oracle or GPT review follow-ups after a blocker resolves. Ask the user or give a checkpoint first, unless they explicitly told you to fully auto-run and accepted the cost.
4. **One deliberate review gate** — Oracle/deep-review is valuable and correct. Avoid automatic repeated cascades after blockers without checkpointing. Use one deliberate review gate or explicit user-approved follow-up, rather than chaining multiple expensive reviews silently. For normal verification, prefer deterministic local checks or lower-cost worker profiles.
5. **Dry-run before dispatch on busy boards** — Before dispatching on a busy board, use dry-run and ensure unrelated ready tasks are not spawned accidentally.
6. **Immediate stop on cost/stuck complaints** — If the user complains about cost or stuck behavior, immediately stop, block, or reclaim the relevant running task before explaining.
7. **Concise updates with time limits** — Keep user-facing updates concise. Do not wait more than a few minutes without checkpointing.

## Update reporting rules

The orchestrator must proactively report Kanban worker outcomes to the user. Do not wait for the user to ask "what happened" or "any updates."

- After monitoring the board or after any worker finishes, immediately summarize completions, blockers, and status transitions.
- When a worker completes, report its summary, changed files, and any follow-up tasks created.
- When a worker blocks, report the blocker reason and ask for the specific decision needed.
- When a worker stalls (timed_out, crashed, reclaimed), report the stall and the recovery action taken.
- Keep reports concise but specific: name task IDs, profiles involved, and concrete artifacts.

## Config location

Main Hermes config:

```text
~/.hermes/config.yaml
```

Profile configs:

```text
~/.hermes/profiles/oracle/config.yaml
~/.hermes/profiles/explorer/config.yaml
~/.hermes/profiles/librarian/config.yaml
~/.hermes/profiles/designer/config.yaml
~/.hermes/profiles/fixer/config.yaml
~/.hermes/profiles/council/config.yaml
~/.hermes/profiles/observer/config.yaml
~/.hermes/profiles/casual/config.yaml
```

Secrets:

```text
~/.hermes/.env
```

Do not put API keys in `config.yaml`.
