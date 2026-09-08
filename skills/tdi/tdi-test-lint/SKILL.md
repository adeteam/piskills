---
name: tdi-test-lint
description: Run and troubleshoot backend and frontend validation in TDI using the correct module-root wrappers and UI workspace commands.
---

# TDI Test & Lint

## Backend (run from module root, for example `src/pip-ade-ai`)

Run tests:

```bash
../pip-ade-testing/test.sh
../pip-ade-testing/test.sh tests.ai
../pip-ade-testing/test.sh tests.ai.tool.test_salesforce_case
```

Run lint:

```bash
../pip-ade-testing/lint.sh
# optional
../pip-ade-testing/lint.sh --include-tests=true
```

## Why these wrappers

- Do not assume `pytest` is available/configured.
- `test.sh` sets `TEST_ROOT` and Django settings via the testing wrapper platform script.
- Direct `python -m unittest` often fails due to unconfigured Django settings.

## Frontend (`ui/`)

```bash
yarn test-auto
yarn lint
yarn build
```

## CI caveat

Current CI does not execute backend unit tests in `cicd/test.sh`; run backend tests locally for backend changes.
