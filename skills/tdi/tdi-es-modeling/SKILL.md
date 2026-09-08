---
name: tdi-es-modeling
description: Design and update Elasticsearch model schemas in TDI using explicit top-level fields and typed embedded objects. Use when adding or changing ES-backed model structures.
---

# TDI ES Modeling

## Core conventions

1. Keep source-of-truth/top-level fields explicit on the model.
2. Avoid unstructured catch-all `ObjectField` payloads for typed data.
3. Put reusable embedded model classes in an `embedded/` package under the module model package, for example:

```text
src/<module>/src/<app>/models/embedded/
```

4. Export embedded classes via `embedded/__init__.py`.
5. Import embedded classes from the parent model, for example:

```python
from .embedded import InfoX
```

6. Use a top-level `info = Info...()` embedded object for generated/runtime metadata.
7. Prefer schema-first fields (`KeywordField`, `TextField`, `BooleanField`, etc.) over dynamic object blobs.
8. If arrays contain structured objects, define a dedicated embedded class and use `ArrayField(EmbeddedClass())`.

## Implementation checklist

- [ ] Top-level fields are explicit and typed.
- [ ] Reusable embedded models live in `models/embedded/`.
- [ ] `embedded/__init__.py` exports are updated.
- [ ] Parent model imports embedded classes from `.embedded`.
- [ ] No `ArrayField(ObjectField())` for known structured payloads.
- [ ] `info` embedded object exists when storing runtime/generated metadata.

## Review output

When done, summarize:
- schema changes,
- migration/reindex implications,
- compatibility risks for downstream queries.
