---
name: tdi-env-setup
description: Set up or repair the local TDI development environment with the repo-pinned Python/Poetry toolchain and install flow. Use when onboarding, rebuilding venvs, or troubleshooting dependency setup.
---

# TDI Environment Setup

## Scope

Use this skill when preparing a local TDI workspace from scratch or after dependency/toolchain drift.

## Required toolchain

- Python `3.10`
- Poetry `2.3.3`
- Pinned packaging tools used in READMEs/CI:
  - `pip==26.0.1`
  - `poetry==2.3.3`
  - `poethepoet==0.44.0`
  - `wheel==0.46.3`
  - `setuptools==82.0.1`

## Standard install flow (repo root)

```bash
poetry lock --regenerate
poetry install --with test-dev --with mod-dev
poetry run poe recurse-install-mod-dev
```

## Notes and gotchas

- `poetry/install_nested_mods.sh` expects `yq`.
- Nested module lockfiles may be regenerated; expect lockfile churn.
- In this monorepo, most feature/backend work belongs in a submodule under `src/`.

## Verification

- Confirm Poetry environment is healthy:

```bash
poetry env info
```

- Confirm Django wrapper is available:

```bash
bin/platform --help
```
