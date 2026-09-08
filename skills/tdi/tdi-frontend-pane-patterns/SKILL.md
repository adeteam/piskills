---
name: tdi-frontend-pane-patterns
description: Build and wire frontend pane components in TDI using existing ModelFormPane/FormPane patterns, lazy-load providers, and parent view wiring conventions.
---

# TDI Frontend Pane Patterns

## Goal

Implement pane components using existing pane abstractions and wiring patterns from current npm-ade modules.

## Core pane bases in repo

- `FormPaneComponent` (`@ade/core/src/panes/form.component.ts`)
- `ModelFormPaneComponent` (`@ade/core/src/panes/modelform.component.ts`)

Most editable/detail panes in admin/tac/chat extend `ModelFormPaneComponent`.

## Concrete examples

- Admin user panes:
  - `ui/modules/npm-ade-admin/src/views/user/panes/details.component.ts`
  - `ui/modules/npm-ade-admin/src/views/user/panes/permission.component.ts`
  - `ui/modules/npm-ade-admin/src/views/user/panes/group.component.ts`
- Parent pane host:
  - `ui/modules/npm-ade-admin/src/views/user/view.component.html`
  - `ui/modules/npm-ade-admin/src/views/user/view.component.ts`

## Pane component shape

Observed pattern:

1. `@Component` selector `*-pane`, template in same folder.
2. Extend `ModelFormPaneComponent`.
3. Register lazy provider when used with `<lazyload>`:

```ts
providers: [{
  provide: LazyLoadableComponent,
  useExisting: MyPaneComponent
}]
```

4. Inputs usually include:
- `@Input() lazy = false`
- `@Input() item` (record)

5. Optional `reload()` override calls `super.reload()` then lazy/table refresh logic.
6. Optional `lazyload()` computes URLs via `store.modelUrl(...)`.

## Common pane responsibilities

### Editable detail pane
(Example: `details.component.ts`)
- build form controls from `item`
- rely on `submitField(...)` for inline field updates
- use `item | can: 'edit':'field'` checks in template

### Datatable-backed pane
(Examples: group/permission panes)
- compute relation URL(s) via `store.modelUrl(model, {id}, relation)`
- pass URL to datatable child
- use `ViewChild` to trigger datatable reload when needed

## Parent view wiring pattern

In parent view template:
- render panes inside sections (often with `<lazyload>` wrappers)
- pass `[item]` and `[lazy]="true"`
- keep references (`#detailsPane`, `#groupPane`) when parent needs to trigger reloads

Example: `ui/modules/npm-ade-admin/src/views/user/view.component.html`

## Registration pattern

For routed feature modules, declare panes in the feature module declarations.

Example:
- `ui/modules/npm-ade-admin/src/views/user/user.module.ts`
  declares all pane components.

Optional barrel exports (commonly used):
- `ui/modules/npm-ade-admin/src/views/user/panes/index.ts`

## Do not introduce new pane framework

Follow sibling pane style in the same module:
- same base class
- same lazy provider pattern
- same form/datatable integration approach
- same permission checks style in templates
