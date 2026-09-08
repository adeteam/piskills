---
name: tdi-frontend-api-model-factory
description: Implement frontend API integration in TDI using model factories, record classes, and model config registration in npm-ade-* modules. Use when adding backend-backed data access in Angular features.
---

# TDI Frontend API Model Factory Pattern

## Goal

Use the TDI data-store pattern for backend API access instead of ad-hoc HTTP calls.

## Core pattern

For API-backed entities in a module (`ui/modules/npm-ade-<domain>/`):

1. Create a model record + factory in `src/models/`.
2. Export it through `src/models/index.ts`.
3. Register the factory in `src/config/model.config.ts` (`ModelConfigDef.models`).
4. Use `store` APIs with `Factory.model` in components/services.

The store registration is wired through `MODEL_CONFIG` in app bootstrap, and `StoreService` loads all registered factories at startup.

## Step-by-step: add a new model

### 1) Create factory file in `src/models/<name>.factory.ts`

Template:

```ts
import { BaseEsRecord, BaseEsFactory } from '@ade/core/src/models/base.factory';

export class ExampleRecord extends BaseEsRecord {
  public id: string | number;
  public name: string;
}

export class ExampleFactory extends BaseEsFactory {
  public static model = 'chat-example';
  public static endpoint = '/api/chat/example/';
  public static recordClass = ExampleRecord;

  public static properties: object = Object.assign({
    id: { type: ['number', 'string', 'null'] },
    name: { type: ['string', 'null'] }
  }, (BaseEsFactory).properties);

  public static relationEndpoints: { [s: string]: object } = {
    'child': {
      model: 'chat-example-child',
      endpoint: '/api/chat/example/${id}/child/'
    }
  };
}
```

Notes:
- Keep model name stable and unique (`<domain>-<entity>` style used widely).
- Endpoint should match backend route style (typically trailing slash).
- Use `relationEndpoints` when backend exposes nested routes.

### 2) Export in module barrel

Update `src/models/index.ts`:

```ts
export * from './example.factory';
```

### 3) Register in module model config

Update `src/config/model.config.ts`:

```ts
import { ExampleFactory } from '../models';

export const ModelConfigDef = {
  models: [
    // ...existing
    ExampleFactory
  ]
};
```

### 4) Confirm config export exists

`src/config/index.ts` should export `model.config` (existing pattern in modules).

### 5) If this is a brand-new package/module

Also ensure app aggregation includes module models:
- `ui/src/app/config/model.config.ts` concatenates your module `ModelConfigDef.models`
- `ui/src/app/app.module.ts` imports the module package

(For normal feature work in an already-registered module, this is usually already done.)

## How to use factories in feature code

## Read operations

```ts
const item = await this.store.find(ExampleFactory.model, id);
const items = await this.store.findAll(ExampleFactory.model, { status: 'active' });
```

## Nested relation reads

```ts
const sub = await this.store.findSub(ExampleFactory.model, 'child', parentId, childId);
const subs = await this.store.findAllSub(ExampleFactory.model, 'child', parentId, { limit: 20 });
```

`'child'` must match a key in `relationEndpoints`.

## Child-entity pattern (generic)

Use `relationEndpoints` to map child entities under a parent slug path.

Pattern:

```ts
public static relationEndpoints: { [s: string]: object } = {
  'child': {
    model: 'domain-child',
    endpoint: '/api/domain/parent/${id}/child/'
  },
  'nested-child': {
    model: 'domain-nested-child',
    endpoint: '/api/domain/parent/${id}/group/${group_id}/nested-child/'
  }
};
```

Query pattern:

```ts
// direct child by id
const child = await this.store.findSub(ParentFactory.model, 'child', parentId, childId);

// child list
const children = await this.store.findAllSub(ParentFactory.model, 'child', parentId, { page_limit: 50 });

// nested child URL with additional slug params
const url = this.store.modelUrl(
  ParentFactory.model,
  { id: parentId, group_id: groupId },
  'nested-child'
);
```

Key rules:
- Relation name in code must match a `relationEndpoints` key exactly.
- URL placeholders (for example `${id}`, `${group_id}`) must be supplied in query params when building relation URLs.
- Keep relation endpoint naming stable; UI code depends on these keys.

## Mapper access

```ts
const mapper = this.store.getMapper(ExampleFactory.model);
```

## Create/update/delete via store

```ts
await this.store.getStorage().create(ExampleFactory.model, record);
await this.store.getStorage().update(ExampleFactory.model, id, payload);
await this.store.getStorage().destroy(ExampleFactory.model, id);
```

## When not to use model factories

Use `HttpClient` or transport service directly for:
- binary/file download/upload flows,
- one-off utility endpoints that are not entity models,
- endpoints with special request/response handling not suited to mapper flow.

For normal CRUD/list/detail data, prefer the factory + store pattern.

## Common failure modes

- Factory created but not exported in `src/models/index.ts`.
- Factory exported but not added to `src/config/model.config.ts`.
- Model name typo when calling `store.*`.
- Relation key mismatch between code and `relationEndpoints`.
- Endpoint path mismatch with backend route (often missing trailing slash).

## Validation checklist

- [ ] New factory compiles and is exported.
- [ ] Factory appears in module `ModelConfigDef.models`.
- [ ] Feature code uses `Factory.model` constants (no duplicated magic model strings).
- [ ] CRUD/list/relation flows work against backend endpoints.
- [ ] `yarn lint`, `yarn test-auto`, and `yarn build` pass in module and workspace as needed.
