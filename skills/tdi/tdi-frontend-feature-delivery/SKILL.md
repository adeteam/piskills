---
name: tdi-frontend-feature-delivery
description: Build frontend features in TDI Angular workspaces using module-specific package structure, routing/module wiring, barrel exports, and repo-standard validation commands.
---

# TDI Frontend Feature Delivery

## Scope

Use this skill when adding or changing frontend features in TDI.

TDI UI has two layers:
- App workspace: `ui/`
- Feature/shared packages: `ui/modules/npm-ade-*` (Angular library packages)

Most feature work belongs in the appropriate `npm-ade-*` package.

## Choose the correct module first

1. Identify domain ownership (`admin`, `chat`, `inventory`, `system`, etc.).
2. Implement in `ui/modules/npm-ade-<domain>/` when the feature is domain-specific.
3. Only change `ui/src` app-level code when truly app-shell behavior is needed.

## Package structure conventions (module packages)

Inside `ui/modules/npm-ade-<domain>/src`, follow existing patterns:
- `views/<feature>/` for routed UI features
- `components/` for reusable presentational pieces
- `services/` for module services
- `models/` for module model factories/types
- `index.ts` barrel exports in subfolders
- `public_api.ts` for package public surface

For routed features, use the existing trio:
- `<feature>.component.ts/.html/.scss`
- `<feature>.module.ts`
- `<feature>.routing.ts`
- `index.ts` export file

## Routing/module wiring standards

1. Add feature module and route in the owning package routing module (for example `chat.routing.ts`).
2. Preserve existing lazy-loading style in this repo (string-based `loadChildren` in Angular 9 projects where currently used).
3. Use `RouterModule.forChild(...)` in feature routing modules.
4. Keep guard/layout behavior consistent with neighboring routes.

For route/layout/guard wiring details, use `/skill:tdi-frontend-routing-wiring`.
For component/view scaffolding details, use `/skill:tdi-frontend-component-scaffold`.
For model-backed reactive forms, use `/skill:tdi-frontend-form-patterns`.
For datatable implementation patterns, use `/skill:tdi-frontend-datatable-patterns`.
For pane implementation patterns, use `/skill:tdi-frontend-pane-patterns`.
For modal implementation patterns, use `/skill:tdi-frontend-modal-patterns`.
For dashboard stats/components/config wiring, use `/skill:tdi-frontend-dashboard-patterns`.
For backend data/API integration details, use `/skill:tdi-frontend-api-model-factory`.
For frontend authorization and permission-gated UI behavior, use `/skill:tdi-frontend-permissions-rbac`.

## Service and export standards

When adding services:
1. Add service file under `src/services/`.
2. Register provider in `src/services/service.module.ts` (if module uses this pattern).
3. Export through `src/services/index.ts`.
4. Export package entry points through `src/index.ts` and `src/public_api.ts` when feature should be externally consumable.

## Styling standards

- Use SCSS (workspace schematics default to `styleext: "scss"`).
- Keep feature styles local to component SCSS unless intentionally shared.
- Reuse existing skin/theming patterns where applicable.

## Validation workflow

### Module-level validation (preferred while iterating)

From `ui/modules/npm-ade-<domain>/`:

```bash
yarn install --non-interactive
yarn lint
yarn build
```

If install fails with registry/auth errors, run `npm run artifactregistry-login` first, then re-run install.

### App/workspace validation (before handoff)

From `ui/`:

```bash
yarn install --non-interactive
yarn lint
yarn build
```

If install fails with registry/auth errors, run `npm run artifactregistry-login` first, then re-run install.

## Delivery checklist

- [ ] Feature implemented in correct `npm-ade-*` package.
- [ ] Routing/module wiring follows adjacent patterns.
- [ ] Barrels/public exports updated where needed.
- [ ] Workspace-level lint/tests/build pass for cross-module confidence.
- [ ] No unrelated formatting/churn changes.
