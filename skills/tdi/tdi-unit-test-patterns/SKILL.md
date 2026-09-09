---
name: tdi-unit-test-patterns
description: Write or modify Python and Django unit tests in TDI using repository mocking conventions. Use when a task explicitly requests backend unit tests, mocks, patches, or temporary Django settings.
---

# TDI Unit Test Patterns

## Mocking style

Always use `with` context managers for temporary mocks and settings overrides. Do not use `patch`, `patch.object`, or `override_settings` as function, method, or class decorators.

Keep each context manager scoped to only the code that needs the altered behavior. Patch the name where the system under test looks it up, not necessarily where the object was originally defined.

### `patch`

```python
def test_sends_notification(self):
    with patch("example.services.send_notification") as send_notification:
        result = run_service()

    send_notification.assert_called_once_with(result)
```

### `patch.object`

```python
def test_uses_client(self):
    with patch.object(client, "fetch", return_value={"status": "ok"}) as fetch:
        result = service.execute()

    fetch.assert_called_once_with()
    self.assertEqual(result, {"status": "ok"})
```

### `override_settings`

```python
def test_feature_enabled(self):
    with override_settings(FEATURE_ENABLED=True):
        result = feature_status()

    self.assertTrue(result)
```

### Multiple temporary overrides

Nest context managers when their scopes differ. Combine them in one `with` statement when they apply to the same operation.

```python
def test_sends_when_enabled(self):
    with override_settings(FEATURE_ENABLED=True), patch(
        "example.services.send_notification"
    ) as send_notification:
        run_service()

    send_notification.assert_called_once_with()
```

## Repository constraints

- Do not add unit tests unless the user explicitly requests them.
- Follow the surrounding test module's base classes, assertions, fixtures, and naming conventions.
- Run backend tests through `../pip-ade-testing/test.sh` from the target module root; do not assume direct `pytest` or `unittest` invocation is configured correctly.
