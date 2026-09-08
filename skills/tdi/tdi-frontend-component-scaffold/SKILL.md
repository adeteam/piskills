---
name: tdi-frontend-component-scaffold
description: Scaffold and wire new Angular components/views in TDI using existing npm-ade-* file layout, module declarations, and export-barrel patterns.
---

# TDI Frontend Component Scaffold

## Goal

Create new components/views following existing file layout and module wiring patterns.

## Existing scaffold patterns

### Routed view folder pattern
Concrete example:
- `ui/modules/npm-ade-chat/src/views/demo/`
  - `demo.component.ts/.html/.scss`
  - `demo.module.ts`
  - `demo.routing.ts`
  - `index.ts`

### Feature module pattern
Example file:
- `ui/modules/npm-ade-chat/src/views/demo/demo.module.ts`

Pattern:
- imports Angular/common/forms modules
- imports shared component modules via `.forRoot()` where used in neighbors
- declares feature component(s)
- imports feature routing module

### Reusable components module pattern
Examples:
- `ui/modules/npm-ade-admin/src/components/components.module.ts`
- `ui/modules/npm-ade-remote/src/components/components.module.ts`

Pattern:
- `ComponentsModule` with declarations + exports
- optional `forRoot()` returning `ModuleWithProviders<ComponentsModule>`

## How to add a new routed view

1. Add folder `src/views/<feature>/`.
2. Create component files + module + routing + index.
3. Add route lazy-load entry in package routing file.
4. Keep selector/style conventions aligned with sibling files.

Minimal `index.ts` pattern (seen broadly):

```ts
export * from './my-feature.module';
```

## How to add a reusable component (non-route)

1. Add component under `src/components/...`.
2. Declare/export it in package `components.module.ts`.
3. If consumed by other modules in-package, ensure importing module pulls `ComponentsModule`.

## Package export surfaces

When feature should be externally consumable from package root, ensure exports are consistent:
- `src/index.ts`
- `src/public_api.ts`

Example pattern:
- `ui/modules/npm-ade-inventory/src/index.ts`
- `ui/modules/npm-ade-inventory/src/public_api.ts`

Both export the package module.

## Do not introduce new scaffolding conventions

Before adding structure, check sibling feature folders in same module and mirror their shape/import style.
