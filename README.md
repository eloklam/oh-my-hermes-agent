# Oh My Hermes Agent

> Formerly called **OMO-slim** — a lightweight multi-profile Hermes Agent workflow.

This repo documents how to run named-profile Kanban delegation in Hermes Agent.

## What this is

- **Role mapping** from design roles to Hermes profiles
- **Config snippets** you can copy and adapt (`configs/example.yaml`)
- **Kanban workflow rules** for multi-profile delegation

## Profiles

| Role | Hermes profile | Purpose |
| --- | --- | --- |
| orchestrator | `default` | routes work, never self-executes non-trivial tasks |
| oracle | `oracle` | adversarial review and architecture critique |
| explorer | `explorer` | broad codebase discovery |
| librarian | `librarian` | evidence gathering and source-grounded docs |
| designer | `designer` | UI / UX review |
| fixer | `fixer` | bounded implementation and tests |
| council | `council` | multi-model consensus |
| observer | `observer` | visual analysis |
| casual chat | `casual` | fast chat buddy (not an OMH worker) |

## Quick start

```bash
# 1. Create profiles
hermes profile create oracle
hermes profile create explorer
hermes profile create fixer

# 2. Disable generic delegation for every OMH profile
#    so workers use named-profile Kanban instead of delegate_task
hermes --profile default  tools disable delegation
hermes --profile fixer    tools disable delegation
hermes --profile explorer tools disable delegation
hermes --profile oracle   tools disable delegation

# 3. Create a Kanban board and dispatch
hermes kanban --board oh-my-hermes-agent create "My first task" --assignee fixer
hermes kanban --board oh-my-hermes-agent dispatch
```

## Core rules

1. **No self-execution** — the orchestrator must delegate non-trivial work via Kanban.
2. **Split before dispatch** — decompose requests into independent lanes; do not bundle unrelated work into one task.
3. **Link real dependencies** — use `hermes kanban link <parent> <child>` so children wait until parents finish.
4. **If a worker stalls, reclaim it** — do not silently finish the work yourself.
5. **Proactive reporting** — after monitoring or after any worker finishes, immediately report completions, blockers, and status transitions. Do not wait for the user to ask.

## Model examples

```yaml
# Orchestrator
provider: openai-codex
model: gpt-5.5

# Workers
provider: custom:<your-provider>
model: <your-worker-model>

# Oracle / council
provider: openai-codex
model: gpt-5.5
```

See `configs/example.yaml` for a full annotated example.

## Secrets and privacy

- `.env` is gitignored.
- API keys live in `~/.hermes/.env`, not in this repo.
- Use placeholder values in docs and example configs.

## Validation checklist

```bash
# No stale terminology in source files
git grep -n -i 'opencode' -- . ':!CONTRIBUTING.md' ':!README.md'
git grep -n -i '\bomo\b\|omo-slim' -- . ':!CONTRIBUTING.md' ':!README.md'

# No real credentials (review any hits)
git grep -n -E 'gho_|github_pat_|sk-[A-Za-z0-9]|BEGIN (RSA|OPENSSH|PRIVATE)|api[_-]?key\s*[:=]|token\s*[:=]' -- .

# YAML is valid
python3 -c "import yaml; yaml.safe_load(open('configs/example.yaml'))"
```

## License

MIT. See `LICENSE`.
