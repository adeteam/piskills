---
name: tdi-service-construction
description: Build TDI service classes using the repository singleton factory pattern with settings passed as constructor kwargs. Use when creating or refactoring service modules.
---

# TDI Service Construction

## Goal

Create services that follow the TDI singleton access pattern and initialize from Django settings via constructor kwargs.

## Required pattern

1. Provide a module-level factory function (for example `myservice()`) that returns a singleton instance.
2. Store singleton state on `ClassName._instance`.
3. Instantiate the service once and pass settings-derived values as kwargs.
4. Keep settings-to-kwargs mapping in the factory function.
5. Keep constructor focused on assigning/configuring instance state from kwargs.

## Reference examples

- `src/pip-ade-chat/src/chat/services/azureopenai_service.py`
- `src/pip-ade-coveo/src/coveo/services/coveo_service.py`
- `src/pip-ade-flexera/src/flexera/services/flexera_service.py`

## Template

```python
from django.conf import settings


def myservice():
    if MyService._instance is None:
        MyService._instance = MyService(
            endpoint=settings.MY_SERVICE_ENDPOINT,
            api_key=settings.MY_SERVICE_API_KEY,
            timeout=settings.MY_SERVICE_TIMEOUT,
        )
    return MyService._instance


class MyService:
    _instance = None

    def __init__(self, **kwargs):
        self._endpoint = kwargs["endpoint"]
        self._api_key = kwargs["api_key"]
        self._timeout = kwargs.get("timeout", 30)
```

## Design checklist

- [ ] Factory function exists and is the normal entrypoint.
- [ ] `Class._instance` singleton guard is implemented.
- [ ] Constructor receives settings-derived kwargs from the factory.
- [ ] Constructor state assignment is explicit and minimal.
- [ ] Expensive clients/connections are lazily created where possible.

## Test checklist

- [ ] Reset `Class._instance = None` in test setup/teardown.
- [ ] Verify factory returns same instance across calls.
- [ ] Verify constructor kwargs mapping from settings is correct.
- [ ] Verify lazy client initialization behavior where applicable.

## Avoid

- Instantiating service directly throughout code instead of using the factory.
- Spreading settings lookups across business methods.
- Reinitializing clients on every call unless required.
