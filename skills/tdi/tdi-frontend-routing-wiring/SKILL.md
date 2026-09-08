---
name: tdi-frontend-routing-wiring
description: Wire new frontend routes in TDI Angular modules using existing layout, guard, and lazy-load patterns from npm-ade-* packages.
---

# TDI Frontend Routing Wiring

## Goal

Add routes using existing module routing conventions (do not invent new routing style).

## Existing patterns in repo

### Package-level routing module uses `RouterModule.forRoot(routes)`
Examples:
- `ui/modules/npm-ade-chat/src/chat.routing.ts`
- `ui/modules/npm-ade-admin/src/admin.routing.ts`
- `ui/modules/npm-ade-system/src/system.routing.ts`

Pattern:
- top-level route with `path: ''`
- layout component (`DefaultLayoutComponent`, `ContainerLayoutComponent`, or `EmptyLayoutComponent`)
- `canActivate`/`canDeactivate` guard sets aligned to module
- child routes lazy-loaded with string syntax (`'./views/x/x.module#XModule'`)

### Feature view routing uses `RouterModule.forChild(routes)`
Example:
- `ui/modules/npm-ade-chat/src/views/demo/demo.routing.ts`

Pattern:
- `path: ''` to local component
- feature-specific guards/data title when needed

## How to add a new routed feature

1. Create `views/<feature>/` with:
   - `<feature>.module.ts`
   - `<feature>.routing.ts`
   - `<feature>.component.ts/.html/.scss`
   - `index.ts`
2. In package routing file (for example `chat.routing.ts`), add a new child entry:

```ts
{
  path: 'chat/my-feature',
  loadChildren: './views/my-feature/my-feature.module#MyFeatureModule'
}
```

3. Reuse the same layout/guard block shape as sibling routes in that module.

## Adding a brand-new npm-ade module package route surface

Follow existing package root shape:
- `<module>.module.ts` imports:
  - `AppAbilityModule`
  - `<Module>AbilityModule`
  - `<Module>RoutingModule`
  - `CoreModule`
- `<module>.routing.ts` defines package routes with `RouterModule.forRoot`
- `src/index.ts` exports `<module>.module`
- `src/public_api.ts` exports `<module>.module`

Examples:
- `ui/modules/npm-ade-inventory/src/inventory.module.ts`
- `ui/modules/npm-ade-inventory/src/index.ts`
- `ui/modules/npm-ade-inventory/src/public_api.ts`

If introducing a new package, also wire app-level imports in:
- `ui/src/app/app.module.ts` (module import)
- `ui/src/app/config/model.config.ts` (model concat, if models provided)

## Guard/layout selection guidance (from existing usage)

- `DefaultLayoutComponent`: admin/system/search style pages
- `ContainerLayoutComponent`: chat demo feature pages
- `EmptyLayoutComponent`: chat widget/ui embed pages

Match neighboring routes in the same package instead of changing layout style.
