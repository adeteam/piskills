---
name: tdi-frontend-datatable-patterns
description: Build and wire frontend datatables in TDI using existing DatatableWrapperComponent, dt-th column definitions, action menus, and inline editable column patterns.
---

# TDI Frontend Datatable Patterns

## Goal

Implement datatables using TDI's existing datatable stack and template patterns.

## Core existing pattern

1. Create a wrapper component that extends `DatatableWrapperComponent`.
2. Define datatable markup in `<name>.datatable.html` using `<datatable>` + `<th dt-th ...>`.
3. Pass `url`, `filters`, `modelFactory`, `modelMapper` into `<datatable>`.
4. Define action menus in wrapper component when needed.
5. Register/export datatable component through package `components.module.ts`.

## Concrete examples in repo

- Wrapper base:
  - `ui/modules/npm-ade-core/src/components/datatablewrapper.component.ts`
- Standard action-menu table:
  - `ui/modules/npm-ade-admin/src/components/datatables/user.datatable.ts`
  - `ui/modules/npm-ade-admin/src/components/datatables/user.datatable.html`
- Read-only table variant:
  - `ui/modules/npm-ade-chat/src/components/datatables/incident.datatable.ts`
  - `ui/modules/npm-ade-chat/src/components/datatables/incident.datatable.html`
- Inline editable checkbox column:
  - `ui/modules/npm-ade-admin/src/components/datatables/permission.datatable.ts`
  - `ui/modules/npm-ade-admin/src/components/datatables/permission.datatable.html`
- Action menu using emitted action:
  - `ui/modules/npm-ade-tac/src/components/datatables/task.datatable.ts`
  - `ui/modules/npm-ade-tac/src/components/datatables/task.datatable.html`

## Wrapper component shape

Pattern seen across modules:

```ts
@Component({
  selector: 'my-datatable',
  templateUrl: 'my.datatable.html'
})
export class MyDatatableComponent extends DatatableWrapperComponent {
  public actionMenus: any[] = [];

  public constructor(protected injection: InjectionService, protected injector: Injector) {
    super(injection, injector);
  }
}
```

If permission-gated actions are needed, follow `user.datatable.ts` style using `AuthenticationService` and `authentication.can(...)`.

## Template shape

Pattern:

```html
<datatable [responsive]="true"
           [url]="url"
           [filters]="filters"
           [modelFactory]="modelFactory"
           [modelMapper]="modelMapper"
           [options]="{'iDisplayLength':10}">
  <thead>
    <tr>
      <th dt-th field="id" class="control" [orderable]="false" [render]="formatter.none" [searchable]="false"></th>
      <th dt-th field="name">Name</th>
    </tr>
  </thead>
</datatable>
```

Follow existing column options from neighbors (`orderable`, `searchable`, `render`, width/style).

## Action menu pattern

- Define `actionMenus` in wrapper component.
- Pass `[menus]="menus"` in action column (`class="action"`).
- `DatatableWrapperComponent` auto-resolves modal actions when `modals` map contains matching key.

Existing menu types in code:
- route action string (for navigation)
- modal action (`modal: 'edit'`)
- callback/event action

## Inline editable column pattern

For editable cells, follow `permission.datatable.html` pattern:
- set `[editable]="true"`
- use `editableType="checkbox"` (or other supported type)
- optional `[editableDisabled]="editableDisabledExtractor"`
- `[editableInline]="true"`

Keep permission checks in template (`*ngIf="item | can: 'edit'"`) and component logic where needed.

## Wiring checklist

- [ ] Datatable component added to package `src/components/datatables/`.
- [ ] Export added to package datatables barrel (`src/components/datatables/index.ts`).
- [ ] Component declared/exported in package `components.module.ts`.
- [ ] Feature module imports correct `ComponentsModule` and uses selector in template.
- [ ] Inputs (`url`, `filters`, `modelFactory`, `modelMapper`) are provided by parent context.

## Notes

- Datatable internals rely on jQuery/DataTables integration in core component.
- Prefer existing wrapper/template patterns over introducing custom table stacks.
