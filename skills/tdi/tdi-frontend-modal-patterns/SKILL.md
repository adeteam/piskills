---
name: tdi-frontend-modal-patterns
description: Build and wire frontend modals in TDI using existing BaseModal/FormModal/ModelFormModal patterns, ng-template modal structure, and parent-view integration conventions.
---

# TDI Frontend Modal Patterns

## Goal

Implement modals using TDI’s existing modal hierarchy and wiring patterns.

## Core modal bases in repo

- `BaseModalComponent` (`@ade/core/src/modals/base.component.ts`)
- `FormModalComponent` (`@ade/core/src/modals/form.component.ts`)
- `ModelFormModalComponent` (`@ade/core/src/modals/modelform.component.ts`)

Use:
- `BaseModalComponent` for non-form modals (viewer/terminal/confirm UI)
- `FormModalComponent` for custom submit logic not tied to model factory
- `ModelFormModalComponent` for model-backed create/edit/delete

## Concrete examples

### Model-backed form modal
- `ui/modules/npm-ade-admin/src/views/user/modals/create.component.ts`
- `ui/modules/npm-ade-admin/src/views/user/modals/edit.component.ts`
- templates: corresponding `*.component.html`

### Custom form modal (non-model submit)
- `ui/modules/npm-ade-chat/src/components/modals/reset.component.ts`

### Non-form modal
- `ui/modules/npm-ade-tac/src/components/modals/terminal.component.ts`

## Required template structure

Current pattern uses `ng-template` referenced by base class `@ViewChild('modal')`:

```html
<ng-template #modal>
  <div class="modal-header">...</div>
  <div class="modal-body">...</div>
</ng-template>
```

Keep this structure; `show()` opens this template via `BsModalService`.

## Model form modal pattern

Typical steps:
1. Extend `ModelFormModalComponent`.
2. Define `alertSubject`.
3. Set mapper(s) in constructor using `this.store.getMapper(...)`.
4. Implement `createFormControls()` and `getFormData()`.
5. Optional `@Input() item` for edit/delete scenarios.

Observed in admin modals:
- create modal sets `this.modelMapper = this.store.getMapper(UserFactory.model)`
- edit modal hydrates controls from `item`

## Parent integration patterns

### Direct modal usage in parent template
Example:
- `ui/modules/npm-ade-admin/src/views/user/user.component.html`
- `ui/modules/npm-ade-admin/src/views/user/view.component.html`

Pattern:
- declare modal component with template ref (`#editModal`)
- wire `(onSave)` to reload pane/datatable
- call `modal.show()` from parent TS (via `@ViewChild`) or action decorators

### Datatable action menu + modal map
When used with datatable wrapper:
- pass `[modals]="{ 'edit': editModal, 'delete': deleteModal }"`
- menu entries in datatable wrapper use `modal: 'edit'`

`DatatableWrapperComponent` resolves and calls `setModel(item)` + `show()`.

## Registration pattern

Declare modal components in owning feature/module declarations.

Example:
- `ui/modules/npm-ade-admin/src/views/user/user.module.ts`
  declares create/edit/delete/change-password modals.

Optional barrel exports (commonly used):
- `ui/modules/npm-ade-admin/src/views/user/modals/index.ts`

## Modal behavior conventions seen in repo

- Header close button calls `close()`.
- Form submit binds `(submit)="submit()"` or button click calls `submit()`.
- Alerts use `<alert [subject]="alertSubject" ...>` in form modals.
- `messageDuration`/`errorMessageDuration` adjusted per modal where needed.

## Do not introduce a new modal stack

Use existing ngx-bootstrap + base modal classes and ng-template pattern already used across modules.
