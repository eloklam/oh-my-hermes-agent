# Design rationale for Oh My Hermes

This document explains why the Hermes setup uses profiles, imported skills, and Kanban. It is not an implementation plan for a new tool.

## Design decisions

### 1. Profiles, not a new runtime

OMH uses named profiles. Hermes already has named profiles with isolated configs, sessions, memory, skills, and logs.

Therefore the setup is a profile mapping:

```text
orchestrator -> default
oracle       -> oracle
explorer     -> explorer
librarian    -> librarian
designer     -> designer
fixer        -> fixer
council      -> council
observer     -> observer
```

### 2. Kanban for durable multi-profile work

OMH work persists across multiple profile roles. Hermes-native equivalent is a Kanban board where each task is assigned to a profile.

Board:

```text
oh-my-hermes-agent
```

Dispatch command:

```bash
hermes kanban --board oh-my-hermes-agent dispatch
```

### 3. delegate_task is disabled in the OMH setup

Named OMH profile calls use Hermes Kanban so the real Hermes profile is spawned. Hermes `delegate_task` creates a generic temporary subagent and does not load `fixer`, `oracle`, `explorer`, `librarian`, `designer`, `council`, or `observer` profile context.

Therefore the `delegation` toolset is disabled for the default orchestrator and all OMH worker profiles. Re-enable it only for explicit debugging or generic throwaway subagent experiments.

Disable delegation for every OMH profile:

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

### 4. Skills are imported, not executed through external source runtimes

Skills compatible with Hermes are imported into:

```text
~/.hermes/skills/omh-imports/
```

Hermes loads them as native Hermes skills. External source runtime hooks are not used.

### 5. No external memory plugin

An external memory plugin is not migrated. Hermes native memory is used instead.

### 6. Secrets stay out of config.yaml

API keys belong in:

```text
~/.hermes/.env
```

Not in:

```text
~/.hermes/config.yaml
```

## Provider mapping example

This table shows how external source provider prefixes map to Hermes providers. It is illustrative, not a requirement.

| External prefix | Hermes provider |
| --- | --- |
| `openai/` | `openai-codex` |
| `anthropic/` | `anthropic` |
| `google/` | `google` |
| `fireworks-ai/` | `custom:fireworks-ai` |
| `kimi/` | `custom:kimi` |
| `reference-go/` | `custom:reference-go` |
| `cerebras/` | `custom:cerebras` |
| `deepseek/` | `custom:deepseek` |
| `groq/` | `custom:groq` |
| `x-ai/` | `custom:x-ai` |
| `perplexity/` | `custom:perplexity` |

Known compatibility note:

```text
openai/gpt-5.5-fast -> openai-codex/gpt-5.5
```

Reason: the ChatGPT Codex account rejects `gpt-5.5-fast`.

## Current Hermes config state this setup expects

Default profile:

```yaml
model:
  provider: openai-codex
  default: gpt-5.5
```

Worker profiles (explorer, librarian, designer, fixer, observer):

```yaml
model:
  provider: custom:fireworks-ai
  default: accounts/fireworks/routers/kimi-k2p6-turbo
```

Oracle and council:

```yaml
model:
  provider: openai-codex
  default: gpt-5.5
agent:
  service_tier: fast
```

Casual (not part of OMH worker delegation):

```yaml
model:
  provider: custom:fireworks-ai
  default: accounts/fireworks/routers/kimi-k2p6-turbo
agent:
  service_tier: fast
  reasoning_effort: medium
```

Source design variant mapping:

```text
orchestrator openai/gpt-5.5-fast      -> Hermes gpt-5.5 + service_tier=fast
oracle openai/gpt-5.5-fast xhigh      -> Hermes gpt-5.5 + service_tier=fast + reasoning_effort=xhigh
explorer/librarian/designer/fixer K2P6 thinking -> Hermes K2P6 + reasoning_effort=medium
council openai/gpt-5.5-fast           -> Hermes gpt-5.5 + service_tier=fast
observer disabled in source design    -> Hermes observer active on K2P6
```

Custom provider:

```yaml
providers:
  fireworks-ai:
    base_url: https://api.fireworks.ai/inference/v1
    key_env: FIREWORKS_AI_API_KEY
    default_model: accounts/fireworks/routers/kimi-k2p6-turbo
```

Secret:

```text
FIREWORKS_AI_API_KEY=...
```

stored in:

```text
~/.hermes/.env
```

## Parallel decomposition policy

Oh My Hermes orchestration should decompose non-trivial requests into independent workstreams before creating Kanban tasks. Multiple independent lanes should become parallel Kanban tasks, not one bundled fixer task.

Correct patterns:

- UI design + implementation -> designer task (parallel) + fixer task (depends on designer)
- Fix blockers + check model variants -> fixer task (parallel) + explorer task (parallel) + librarian task (parallel)
- Optional adversarial review -> oracle task as a dependent child after fixer completes

Use `hermes kanban --board oh-my-hermes-agent link <parent> <child>` to express dependencies so children wait until parents finish.

## Non-goals

This repo does not contain:

- a CLI
- a Python package
- a runtime wrapper
- a Hermes plugin
- an external plugin loader
- a memory plugin port

Hermes is the only runtime.
