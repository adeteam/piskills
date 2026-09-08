---
name: tdi-frontend-dashboard-patterns
description: Build and register dashboard stats/components in TDI using module dashboard.config.ts, dashboard component exports, and app-level dashboard config aggregation.
---

# TDI Frontend Dashboard Patterns

## Goal

Add dashboard stats/cards and custom dashboard components using existing TDI wiring.

## Core pattern in this repo

Dashboard content is config-driven and aggregated:

1. Module-level config defines dashboard entries
   - `ui/modules/npm-ade-<module>/src/config/dashboard.config.ts`
2. App-level config aggregates module configs
   - `ui/src/app/config/dashboard.config.ts`
3. Core dashboard screen renders and filters by mode/permissions
   - `ui/modules/npm-ade-core/src/views/dashboard/dashboard.component.ts`

## TAC concrete example

### Module config
- `ui/modules/npm-ade-tac/src/config/dashboard.config.ts`

Defines:
- `dashboardStats` entries (name, icon, apiUrl, viewUrl, order, modes, permissions)
- `dashboardComponents` entries (component class, order, modes, permissions)

### Dashboard components used by TAC config
- `ui/modules/npm-ade-tac/src/components/dashboards/tools.component.ts`
- `ui/modules/npm-ade-tac/src/components/dashboards/agents.component.ts`
- exported from `ui/modules/npm-ade-tac/src/components/dashboards/index.ts`

### Component registration
TAC components module declares/exports dashboard components:
- `ui/modules/npm-ade-tac/src/components/components.module.ts`

## App aggregation requirement

If adding dashboard config to a module, ensure app config includes it:
- `ui/src/app/config/dashboard.config.ts` concatenates:
  - `ModuleDashboardConfigDef.dashboardStats`
  - `ModuleDashboardConfigDef.dashboardComponents`

If module is already listed there (like TAC), only module config updates are needed.

## How filtering works at runtime

Core dashboard uses `UiService` filtering:
- mode filtering (`modes`)
- permission filtering (`permissions`)

Relevant runtime path:
- `ui/modules/npm-ade-core/src/services/ui.service.ts`
- `modeDashboardComponents(...)`
- `modeDashboardStats(...)`

So new entries should include `modes`/`permissions` only when needed; otherwise defaults behave like existing entries.

## Adding a new dashboard stat

In module `dashboard.config.ts`, follow existing stat object shape:

```ts
{
  name: 'My Stat',
  icon: 'fa-star',
  apiUrl: '/api/<module>/my-stat/',
  viewUrl: '/<module>/my-view',
  order: 30,
  totalFormatter: 'none',
  modes: ['tdi'],
  permissions: []
}
```

Mirror formatter and ordering style used by neighboring stats.

## Adding a new dashboard component

1. Create component under module components dashboards folder.
2. Export it from `components/dashboards/index.ts`.
3. Declare/export it in module `components.module.ts` (follow existing dashboard component registration pattern).
4. Reference component class in module `dashboard.config.ts`:

```ts
{
  component: MyDashboardComponent,
  modes: ['tdi'],
  permissions: [],
  order: 2
}
```

## Component implementation patterns from TAC

- `ToolsDashboardComponent` extends `FormPaneComponent` and uses template-triggered modals.
- `AgentsDashboardComponent` extends `FormPaneComponent`, loads data via HTTP, and handles async readiness flags.

Use these as behavioral patterns instead of introducing a new dashboard framework.

## Checklist

- [ ] Dashboard stat/component added in module `dashboard.config.ts`.
- [ ] New dashboard component exported in `components/dashboards/index.ts`.
- [ ] Component declared/exported in module `components.module.ts`.
- [ ] App dashboard aggregation includes module config (if not already).
- [ ] `modes`/`permissions` align with existing module behavior.
