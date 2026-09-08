---
name: tdi-frontend-form-patterns
description: Build frontend forms in TDI using existing pane/widget form abstractions, model-backed submit flows, and validation/error handling patterns.
---

# TDI Frontend Form Patterns

## Goal

Implement forms using existing base form abstractions instead of ad-hoc submit logic.

## Existing form abstractions

Core base classes:
- `ui/modules/npm-ade-core/src/panes/form.component.ts` (`FormPaneComponent`)
- `ui/modules/npm-ade-core/src/panes/modelform.component.ts` (`ModelFormPaneComponent`)

Chat widget base:
- `ui/modules/npm-ade-chat/src/widgets/base-model.component.ts` (`BaseModelWidgetComponent`)

These already handle:
- form creation/reload lifecycle
- submit state and alert messages
- create/update/delete routing through store/modelFactory
- backend error message extraction

## Concrete usage examples

- `ui/modules/npm-ade-chat/src/views/widget/contact-info.component.ts`
- `ui/modules/npm-ade-chat/src/views/widget/picklist-values.component.ts`

Observed pattern in those components:
1. set `modelFactory` (`@Input()` defaults to a factory)
2. set `modelMapper` in constructor using `this.store.getMapper(Factory.model)`
3. fetch backing item in `refreshParams(...)` using `this.store.find(...)`
4. implement `createFormControls()` returning field config arrays
5. implement `getFormData()` mapping form values to backend payload shape
6. optionally override `actionAfterSubmittingSuccess(...)` for post-submit behavior

## Recommended implementation shape

```ts
export class MyFormComponent extends BaseModelWidgetComponent {
  @Input()
  public modelFactory = MyFactory;

  public constructor(protected injection: InjectionService, protected injector: Injector) {
    super(injection, injector);
    this.modelMapper = this.store.getMapper(MyFactory.model);
  }

  public async refreshParams(queryParams: any): Promise<void> {
    this.item = await this.store.find(MyFactory.model, queryParams['id']);
  }

  public createFormControls(): object {
    return {
      name: ['', Validators.required]
    };
  }

  public getFormData(): object {
    const data = this.form.value;
    return { name: data.name };
  }
}
```

## Validation patterns already in use

- Angular `Validators` and `RxwebValidators` are both used.
- Conditional validation pattern exists in `contact-info.component.ts`.

## Submit/CRUD behavior to rely on

`ModelFormPaneComponent` resolves create/update/delete automatically based on:
- `item.id` presence
- configured HTTP method
- `modelFactory` / `modelMapper` / record class

Prefer this built-in flow over custom `HttpClient` submit logic for model-backed forms.

## When direct HTTP is acceptable

For non-model workflows (special endpoints, binary flows), direct `HttpClient` usage exists in codebase; keep those cases explicit.
