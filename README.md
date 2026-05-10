# Oh My Hermes Agent

A public profile and Kanban configuration pattern for running a lightweight multi-profile Hermes Agent workflow.

This repo is a documented setup and mapping record for Hermes users who want named-profile delegation via Kanban.

## What this is

- **Role mapping** from source design roles to Hermes profiles
- **Hermes config snippets** you can copy and adapt
- **Profile setup notes** for multi-profile workflows
- **Kanban multi-profile workflow mapping** notes
- **Design references** for attribution and future comparison


## Profiles

| Source role | Hermes profile | Purpose |
| --- | --- | --- |
| orchestrator | `default` | strategic coordinator and delegator |
| oracle | `oracle` | deep reasoning, adversarial review, architecture critique |
| explorer | `explorer` | broad codebase exploration and unknown discovery |
| librarian | `librarian` | evidence gathering and source-grounded docs |
| designer | `designer` | UI / UX review and edits |
| fixer | `fixer` | bounded implementation, bug fixing, tests |
| council | `council` | multi-model consensus and review |
| observer | `observer` | visual analysis for images, PDFs, screenshots |
| casual chat | `casual` | fast chat buddy for facts and casual conversation (not an OMH worker) |

## Quick start

1. **Install Hermes Agent** following the upstream Hermes documentation.
2. **Create profiles** for the roles you want:
   ```bash
   hermes profile create oracle
   hermes profile create explorer
   hermes profile create fixer
   # etc.
   ```
3. **Configure providers and models** per profile (see `configs/example.yaml` for a known-good example).
4. **Disable delegation** for every OMH profile so `delegate_task` is not used instead of named-profile Kanban:
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
5. **Create a Kanban board** (the default `oh-my-hermes-agent` board is assumed in docs):
   ```bash
   hermes kanban --board oh-my-hermes-agent create "My first task" --assignee fixer
   ```
6. **Dispatch** so the board scheduler spawns profile workers:
   ```bash
   hermes kanban --board oh-my-hermes-agent dispatch
   ```
7. **Verify** a profile has delegation disabled:
   ```bash
   hermes --profile fixer tools list | grep delegation
   ```
   Expected: a `✗ disabled delegation` line.

## Kanban delegation model

For non-trivial requests, the orchestrator must decompose work into independent lanes before creating Kanban tasks. Do not bundle unrelated work into a single fixer task.

### Dependency semantics

- **Truly parallel** tasks have no parent/child links and can run simultaneously.
- **Dependent** tasks are linked with `hermes kanban link <parent> <child>` so the child waits until the parent finishes.

Examples:

- Build app with UI/UX -> create designer task (parallel, no links). If implementation must wait for design decisions, create fixer task and link designer -> fixer.
- Fix blockers + research alternatives -> create fixer, explorer, and librarian tasks with **no links**; all run independently.
- Optional adversarial review -> create oracle task and link fixer -> oracle so oracle reviews after fixer finishes.

Use `hermes kanban --board oh-my-hermes-agent link <parent> <child>` to express dependencies so children wait until parents finish.

## No self-execution rule

The default orchestrator must not implement non-trivial work itself.

What the orchestrator should delegate via Kanban:

- GitHub PR and issue workflows
- Repository edits, file creation, or code changes
- Implementation, bug fixes, or tests
- Research, documentation, or evidence gathering
- Architecture or security review
- Config or profile changes

What the orchestrator may handle directly:

- Simple factual replies and quick lookups
- Tiny edits (single-line typos, trivial corrections)
- Direct checks and verification commands
- Clarification questions to the user

### Wrong vs correct

Wrong — one orchestrator doing everything itself:

```text
User: "Fix the auth bug and open a PR"
Orchestrator: (opens issue, writes fix, commits, opens PR, reviews it, merges)
```

Correct — parallel Kanban cards with the right profiles:

```text
User: "Fix the auth bug and open a PR"
Orchestrator:
  1. Create fixer task: "Fix auth bug in src/auth.py"
  2. Create explorer task: "Check for related auth issues" (parallel to fixer)
  3. Create librarian task: "Collect source-grounded references for auth fix" (parallel to fixer)
  4. Dispatch
  5. After fixer completes, create oracle task: "Review auth fix PR"
  6. Link fixer -> oracle
  7. Dispatch again
```

### Stalled-worker recovery

If a worker task is stalled (timed_out, crashed, or blocked), do not silently finish the work yourself.

Recovery options:

- Reclaim the task and re-queue it with a narrower scope
- Reassign to a different profile if the original profile is misconfigured
- Create a follow-up child task for the remaining work
- Block the task and ask the user for a decision if the stall reason is ambiguous

Example:

```bash
# Reclaim a stalled task so the dispatcher can retry it
hermes kanban --board oh-my-hermes-agent reclaim <task_id>
```

## Model / provider examples

The model and provider values in this repo reflect the author's current known-good setup. They are **examples**, not requirements. You can substitute any Hermes-supported provider and model.

```text
# Orchestrator
provider: openai-codex
model: gpt-5.5

# Worker profiles (explorer, librarian, designer, fixer, observer)
provider: custom:fireworks-ai
model: accounts/fireworks/routers/kimi-k2p6-turbo

# Oracle and council
provider: openai-codex
model: gpt-5.5
```

See `configs/example.yaml` for a full annotated example.

## Secrets and privacy

- `.env` is ignored by `.gitignore`.
- API keys belong in `~/.hermes/.env`, not in this repo.
- Do not commit `auth.json`, live profile configs with secrets, debug reports, or gateway logs.
- Use placeholder values in docs and example configs.

## Repository layout

```text
configs/example.yaml          # mapping and config notes
docs/swarm-design.md          # how Hermes profiles + Kanban form the OMH workflow
docs/design-rationale.md      # design rationale and parity notes
LICENSE                       # MIT License
CONTRIBUTING.md               # contribution guidelines
SECURITY.md                   # security policy
```

## Validation checklist

Before contributing, run these checks:

```bash
# No stale terminology (excludes these checklist files themselves)
git grep -n -i 'opencode' -- . ':!CONTRIBUTING.md' ':!README.md'
git grep -n -i '\bomo\b\|omo-slim' -- . ':!CONTRIBUTING.md' ':!README.md'
find . -iname '*opencode*' -not -path './README.md' -not -path './CONTRIBUTING.md' -print
find . -iname '*omo*' -not -path './README.md' -not -path './CONTRIBUTING.md' -print

# No real credentials (review any hits carefully)
git grep -n -E 'gho_|github_pat_|sk-[A-Za-z0-9]|BEGIN (RSA|OPENSSH|PRIVATE)|api[_-]?key\s*[:=]|token\s*[:=]|password\s*[:=]|secret\s*[:=]' -- .

# YAML is valid
python3 -c "import yaml; yaml.safe_load(open('configs/example.yaml'))"
```
