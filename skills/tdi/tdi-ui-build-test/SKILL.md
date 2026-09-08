---
name: tdi-ui-build-test
description: Install, test, lint, and build the TDI Angular workspace using repo-standard commands and CI-compatible setup. Use for frontend validation and release readiness.
---

# TDI UI Build & Test

## Scope

Use for frontend work in `ui/` and for reproducing CI-style UI checks.

## Install/auth path

From repo root:

```bash
cd ui
yarn install --non-interactive
```

If dependency install fails with private registry/auth errors, run:

```bash
npm run artifactregistry-login
yarn install --non-interactive
```

Notes:
- CI commonly runs artifact registry login before install.
- Local development may not require it if credentials are already available.

## Main validation commands

From `ui/`:

```bash
yarn test-auto
yarn lint
yarn build
```

## Production build behavior

Use the production wrapper script when requested:

```bash
./build.sh
```

Notes:
- `build.sh` uses `ng build --preserve-symlinks`
- `--aot` is intentionally disabled in this flow

## Troubleshooting checklist

1. Confirm dependency install succeeded (and run `npm run artifactregistry-login` only if auth errors occur).
2. Re-run failing command directly (`yarn test-auto`, `yarn lint`, or `yarn build`).
3. Report failures with command, package/module path, and first actionable stack trace.
