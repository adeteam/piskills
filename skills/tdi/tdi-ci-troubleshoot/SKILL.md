---
name: tdi-ci-troubleshoot
description: Diagnose TDI Bitbucket pipeline failures by mapping failures to setup/test/build stages and reproducing checks locally with repository-correct commands.
---

# TDI CI Troubleshoot

## Pipeline order

Bitbucket pipeline order is:

1. `./cicd/setup.sh`
2. `./cicd/test.sh`
3. `./cicd/build.sh`
4. `./cicd/upload.sh` (main/manual contexts)

## Critical caveat

- Current `cicd/test.sh` does **not** run backend unit tests (backend test block is commented out).
- Do not treat a green CI run as backend unit-test validation.

## Triage workflow

1. Identify failing stage and exact command/log section.
2. Reproduce locally from the matching working directory.
3. For backend changes, run module tests explicitly with repo wrappers:

```bash
cd src/<target-module>
../pip-ade-testing/test.sh
../pip-ade-testing/lint.sh
```

4. For frontend failures, reproduce from `ui/`:

```bash
yarn test-auto
yarn lint
yarn build
```

5. Summarize root cause, blast radius, and minimal fix.

## Output format

- Stage failed
- Reproduction command(s)
- Root cause
- Fix applied/proposed
- Follow-up checks
