# Contributing to Oh My Hermes Agent

This repository is **docs and configuration only**. It is not a runtime, package, or plugin.

## What to contribute

- Documentation improvements
- Config example corrections
- Profile setup notes
- Kanban workflow clarifications

## What not to contribute without discussion

- New runtime wrappers or CLI tools
- Python packages or helper modules
- Hermes plugins
- Large automation scripts

## Before opening a PR

1. Validate YAML if you edited `configs/example.yaml`:
   ```bash
   python3 -c "import yaml; yaml.safe_load(open('configs/example.yaml'))"
   ```
2. Check for stale terminology (these commands exclude the checklist files themselves):
   ```bash
   git grep -n -i 'opencode' -- . ':!CONTRIBUTING.md' ':!README.md'
   git grep -n -i '\bomo\b\|omo-slim' -- . ':!CONTRIBUTING.md' ':!README.md'
   find . -iname '*opencode*' -not -path './README.md' -not -path './CONTRIBUTING.md' -print
   find . -iname '*omo*' -not -path './README.md' -not -path './CONTRIBUTING.md' -print
   ```
3. Confirm no credentials, tokens, or auth files are committed.
4. Keep changes small and explicit.
