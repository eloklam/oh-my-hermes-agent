# Oh My Hermes Agent

> A port and improvement inspired by [oh-my-opencode-slim](https://github.com/alvinunreal/oh-my-opencode-slim).

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
6. **Cost-control and checkpointing** — checkpoint early, keep updates concise, and never block in long foreground polling loops. See [docs/swarm-design.md](docs/swarm-design.md) for the full rules.
7. **Board lifecycle** — reuse the default board; new boards are rare; no timestamped names; use tenants and workspaces for isolation. See [docs/swarm-design.md](docs/swarm-design.md).
8. **Session title hygiene** — set a stable title early (`YYYY-MM-DD | PROJECT | goal | area`); avoid repeated renaming. See [docs/swarm-design.md](docs/swarm-design.md).

## Suggested model mapping

Inspired by [oh-my-opencode-slim](https://github.com/alvinunreal/oh-my-opencode-slim) preset semantics.

| OMH Profile | Tier | Purpose |
| --- | --- | --- |
| `default` (orchestrator) | **Top** | Strongest all-around; implements *and* delegates |
| `oracle` | **Top** | Same model as orchestrator; `reasoning_effort: xhigh` |
| `explorer` | **Worker** | Cheap/fast scoped discovery |
| `librarian` | **Worker** | Same cheap model as explorer |
| `fixer` | **Worker** | Same cheap model as explorer |
| `designer` | **Creative** | UI/UX judgment; stronger than worker tier |
| `observer` | **Vision** | Vision-capable; disable if unused |
| `council` | **Consensus** | Strong synthesis; diversity across models matters |

```yaml
# Top tier — default (orchestrator)
provider: <your-orchestrator-provider>
model: <your-orchestrator-model>
agent:
  service_tier: fast
  reasoning_effort: medium

# Top tier — oracle (higher reasoning effort)
provider: <your-orchestrator-provider>
model: <your-orchestrator-model>
agent:
  service_tier: fast
  reasoning_effort: xhigh

# Worker tier — explorer, librarian, fixer
provider: <your-worker-provider>
model: <your-worker-model>
agent:
  service_tier: fast
  reasoning_effort: medium

# Creative tier — designer
provider: <your-creative-provider>
model: <your-creative-model>
agent:
  service_tier: fast
  reasoning_effort: medium

# Vision tier — observer (opt-in)
provider: <your-vision-provider>
model: <your-vision-model>
agent:
  service_tier: fast
  reasoning_effort: medium

# Consensus tier — council
provider: <your-review-provider>
model: <your-review-model>
agent:
  service_tier: fast
  reasoning_effort: medium
```

### Upstream preset examples

These are the actual model strings used by [oh-my-opencode-slim](https://github.com/alvinunreal/oh-my-opencode-slim) presets for reference. Replace them with your own provider/model choices; OMH does not endorse any specific vendor.

| Preset | Orchestrator / Default | Oracle | Explorer / Librarian | Designer | Fixer | Observer | Council |
|--------|------------------------|--------|----------------------|----------|-------|----------|---------|
| `openai` (default) | `openai/gpt-5.5` | `openai/gpt-5.5` high | `openai/gpt-5.4-mini` low | `openai/gpt-5.4-mini` medium | `openai/gpt-5.4-mini` low | (disabled) | (not preset) |
| `opencode-go` | `opencode-go/glm-5.1` | `opencode-go/deepseek-v4-pro` max | `opencode-go/minimax-m2.7` | `opencode-go/kimi-k2.6` medium | `opencode-go/deepseek-v4-flash` high | `opencode-go/kimi-k2.6` | `opencode-go/deepseek-v4-pro` high |
| `copilot` | `github-copilot/claude-opus-4.6` | `github-copilot/claude-opus-4.6` high | `github-copilot/grok-code-fast-1` low | `github-copilot/gemini-3.1-pro-preview` medium | `github-copilot/claude-sonnet-4.6` low | (disabled) | (not preset) |
| `kimi` | `kimi-for-coding/k2p5` | `kimi-for-coding/k2p5` high | `kimi-for-coding/k2p5` low | `kimi-for-coding/k2p5` medium | `kimi-for-coding/k2p5` low | (disabled) | (not preset) |
| `zai-plan` | `zai-coding-plan/glm-5` | `zai-coding-plan/glm-5` high | `zai-coding-plan/glm-5` low | `zai-coding-plan/glm-5` medium | `zai-coding-plan/glm-5` low | (disabled) | (not preset) |

> Sources: `src/cli/providers.ts`, `docs/opencode-go-preset.md`, `docs/thirty-dollars-preset.md`, `docs/authors-preset.md`.

Key take-aways from upstream:

- **Orchestrator is not a lightweight router** — upstream assigns it the same top-tier model as Oracle.
- **Worker triad shares one cheap model** — explorer, librarian, and fixer all map to the same low-cost tier.
- **Designer is a distinct tier** — upstream gives it a `medium` variant (not `low`) because UI/UX quality matters.
- **Observer is opt-in** — upstream disables it by default.
- **Council value = diversity** — benefit comes from comparing different model perspectives.

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
