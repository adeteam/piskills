---
name: org-pr-review
description: Perform a structured pull request review with risk analysis, testing recommendations, and actionable feedback. Use when reviewing changes in any repository.
---

# Org PR Review

## Goal

Produce a consistent, high-signal code review summary.

## Workflow

1. Identify scope and changed components.
2. Check correctness, edge cases, and failure modes.
3. Check tests and suggest missing coverage.
4. Check readability and maintainability.
5. Provide prioritized findings:
   - Critical
   - Important
   - Nice-to-have

## Output format

- **Summary**
- **Findings** (with file paths)
- **Test recommendations**
- **Merge readiness**
