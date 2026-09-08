---
name: tdi-backend-task
description: Implement backend changes in the TDI monorepo with submodule-aware edits, Django wrapper conventions, and repository-specific safety checks.
---

# TDI Backend Task

## Always-on repo constraints

- This repo is a monorepo with many Git submodules under `src/`.
- Most backend feature changes belong in a target submodule, not repository root.
- Use `bin/platform` as Django entrypoint when needed (not `manage.py`).

## Workflow

1. Locate the target backend module under `src/`.
2. Make focused edits in that submodule.
3. Keep changes scoped; avoid unrelated refactors.
4. Validate with repo-standard test/lint wrappers.
5. Confirm no accidental submodule pointer churn in the root repo.

## Validation (from module root)

```bash
../pip-ade-testing/test.sh
../pip-ade-testing/test.sh <test-target>
../pip-ade-testing/lint.sh
```

Do not assume direct `pytest`/`unittest` invocation is configured correctly in this repo.

## Safety notes

- Root commits track submodule refs; code usually belongs in submodule repos.
- Be cautious with scripts that reset submodules (for example `git_submodule_tocommit.sh`).
