---
name: tdi-backend-permissions-rbac
description: Register, enforce, and troubleshoot TDI backend RBAC permissions for Django REST Framework and Elasticsearch-backed resources. Use when protecting API actions, adding permission records, diagnosing unexpected access, or aligning backend permissions with frontend CASL subjects.
---

# TDI Backend Permissions and RBAC

## Goal

Protect backend resources with explicit TDI permissions. Do not treat permission registration, frontend hiding, or the global employee RBAC as endpoint-level authorization.

Also load `/skill:tdi-backend-task` for submodule and validation rules. Load `/skill:tdi-frontend-permissions-rbac` when the permission also controls frontend navigation, views, links, or actions. Load `/skill:tdi-backend-websockets` when the resource has a WebSocket endpoint.

## Architecture summary

TDI backend authorization has distinct layers:

1. **Permission definitions** under `<app>/django/permissions/` register selectable permission records.
2. **`post_migrate` synchronization** creates records in Django `auth_permission`, `SystemUserPermission`, and `SystemGroupPermission`.
3. **Global DRF permission/RBAC configuration** authenticates and applies default RBAC rules.
4. **Resource-specific enforcement** checks the permission codename required by each endpoint action.
5. **Frontend CASL checks** hide UI elements but are not a backend security boundary.

These layers are complementary. A permission definition alone does not protect an endpoint.

## Inspect before implementing

Check all of the following in the actual checkout:

- `etc/settings/std/auth.py` for `AUTH_ENFORCE_RBAC`
- `etc/settings/std/api.py` for `DEFAULT_PERMISSION_CLASSES` and `DEFAULT_RBAC_CLASSES`
- `src/pip-ade-core/src/core/permissions/rbac.py`
- `src/pip-ade-core/src/core/rbacs/`
- `src/pip-ade-core/src/core/signals/permission.py`
- `src/pip-ade-core/src/core/services/permission_service.py`
- `src/pip-ade-wmsauth/src/wmsauth/services/authorization_service.py`
- Neighboring `<app>/django/permissions/` definitions
- The target viewset's supported actions and any custom endpoints

Do not assume the current global RBAC enforces application permission codenames. In common TDI configurations, `RuckusEmployeeRbac` grants broad access to tenant-1 employees. An explicit resource check is still required when every user must possess a named permission.

## 1. Register permission definitions

Create the standard package layout in the owning backend submodule:

```text
src/pip-ade-<domain>/src/<app>/django/
├── __init__.py
└── permissions/
    ├── __init__.py
    └── <resource>.py
```

Define only actions the resource actually supports:

```python
DEFINITION = [
    {
        "app": "example",
        "model": "ExampleResource",
        "code": "exampleresource_add",
        "description": "Ability to add example resources"
    },
    {
        "app": "example",
        "model": "ExampleResource",
        "code": "exampleresource_view",
        "description": "Ability to view example resources"
    }
]
```

The permission discovery service recursively scans installed apps for `django.permissions` modules and their `DEFINITION` values.

### Codename rules

Permission codenames follow:

```text
<subject>_<action>
```

The frontend splits the codename at the first underscore. Keep the subject portion free of underscores. Follow existing compact lowercase names for multiword subjects when coordinating with frontend CASL, for example:

```text
jiracodingtask_view
licenseactivation_create
```

Use the exact same subject in frontend checks:

```ts
authentication.can('view', 'jiracodingtask')
```

The `model` value remains the exact Python model class name, such as `JiraCodingTask`.

Common action names are:

- `add` for create
- `view` for list/retrieve
- `edit` for update/partial update
- `remove` or the established module convention for destroy

Confirm neighboring definitions before choosing between legacy aliases such as `delete` and `remove`.

## 2. Enforce permissions on REST endpoints

Do not stop after adding `DEFINITION`. Registration makes permissions assignable; it does not necessarily make a viewset require them.

For strict resource authorization, create an endpoint permission class that:

1. Preserves the existing global `RbacPermission` behavior.
2. Honors `AUTH_ENFORCE_RBAC`.
3. Preserves superuser access.
4. Maps each supported DRF action to an explicit codename.
5. Denies unknown or unsupported actions by default.

Pattern:

```python
from django.conf import settings

from core.permissions import RbacPermission
from wmsauth.services import authorization


class ExampleResourcePermission(RbacPermission):
    def has_permission(self, request, view):
        if not super().has_permission(request, view):
            return False
        if not settings.AUTH_ENFORCE_RBAC or request.user.is_superuser:
            return True

        if view.action == 'create':
            return authorization(request).can('exampleresource_add')
        if view.action in ('list', 'retrieve'):
            return authorization(request).can('exampleresource_view')
        if view.action in ('update', 'partial_update'):
            return authorization(request).can('exampleresource_edit')
        if view.action == 'destroy':
            return authorization(request).can('exampleresource_remove')

        return False
```

Attach it explicitly:

```python
class ExampleResourceViewSet(RestViewSet):
    _model = ExampleResource
    permission_classes = [ExampleResourcePermission]
```

### Important behavior

- Assigning `permission_classes` replaces DRF's configured default list for that view. Subclassing and calling `RbacPermission.has_permission()` preserves the platform baseline before applying the stricter resource check.
- Do not rely only on `_rbac = PermissionRbac()` without tracing how default RBAC classes are combined. Default and view RBACs may be combined with OR semantics.
- Read-only endpoints generally need only `view`; do not register misleading add/edit/remove permissions.
- Custom viewset actions must be mapped explicitly. Fail closed for actions that are not recognized.
- Keep authorization at the endpoint even when every corresponding UI control is hidden.

## 3. Nested and object-level resources

For nested endpoints:

- Require the child resource's action permission.
- Validate that the child belongs to the parent identifier in the URL.
- Do not assume possession of the parent ID proves access.
- Add an object-level check or RBAC filter when users may access only a subset of records.

A global action permission controls access to the resource type; it does not automatically establish row-level ownership or tenancy.

## 4. WebSocket endpoints

DRF permission classes do not run for Django Channels consumers.

If a protected REST resource is also streamed over WebSockets:

1. Reject unauthenticated users before accepting.
2. Apply the same named permission as the REST read endpoint.
3. Validate parent-child/resource ownership.
4. Accept only after all checks succeed.
5. Use `4401` for unauthenticated and `4403` for forbidden connections.
6. Run synchronous database and permission work through `sync_to_async`.

Follow `/skill:tdi-backend-websockets` for the full consumer lifecycle.

## 5. Materialize permission records

Permission records are created by the post-migrate synchronization hook. After adding definitions, run from the TDI root:

```bash
bin/platform migrate --noinput
```

Verify all three stores:

```bash
bin/platform shell -c "
from django.contrib.auth.models import Permission
from wmsauth.models import SystemGroupPermission, SystemUserPermission
codes = ['exampleresource_add', 'exampleresource_view']
print(list(Permission.objects.filter(codename__in=codes).values_list('codename', flat=True)))
print(list(SystemGroupPermission.objects.filter(codename__in=codes).values_list('codename', flat=True)))
print(list(SystemUserPermission.objects.filter(codename__in=codes).values_list('codename', flat=True)))
"
```

If definitions are discoverable but rows are absent, the migration/post-migrate synchronization has not run against that database.

## 6. Session and process lifecycle

TDI's authorization service caches permission codenames in the user's session.

After adding or changing assignments:

- Log out and back in, or otherwise clear/reload the permission session.
- Restart backend workers after changing Python view or permission classes.
- Restart ASGI workers after changing WebSocket consumers.
- Confirm the running service uses the checkout/container that was modified.

A frontend rebuild does not reload backend Python code. A backend restart does not rebuild frontend static assets.

## 7. Validate effective access

Do not validate only by looking at the permissions table or by observing hidden UI.

Test representative users with `AUTH_ENFORCE_RBAC=True`:

- user without permission receives HTTP 403
- user with `view` can list and retrieve
- user with `view` but not `add` cannot create
- user with `add` has only the intended create behavior
- superuser behavior matches platform policy
- unsupported actions remain unavailable
- nested resource ownership is enforced
- WebSocket connection is rejected without read permission

Inspect the user's direct and group permissions because effective access is their union.

For an isolated diagnostic, instantiate the endpoint permission against the real user and action, or call the actual API. Prefer an actual HTTP request for final verification because it exercises URL routing, authentication, sessions, and the loaded application process.

## Common failure modes

- Permission definitions were added, but `bin/platform migrate --noinput` was not run.
- Permission rows exist, but the endpoint never checks their codenames.
- Global `RuckusEmployeeRbac` still grants the user access.
- The permission class replaced global defaults without preserving baseline checks.
- The backend process still has the old viewset loaded.
- The user's session contains stale permission assignments.
- Frontend subject and backend codename subject do not match.
- UI controls are hidden, but direct API or WebSocket access remains open.
- REST is protected, but the equivalent WebSocket is not.
- A list endpoint is protected while a nested or custom endpoint bypasses the check.

## Validation commands

From the backend module root:

```bash
../pip-ade-testing/lint.sh
```

From the TDI root:

```bash
bin/platform check
bin/platform migrate --noinput
```

Use the repository test wrapper only when tests are requested:

```bash
../pip-ade-testing/test.sh <test-target>
```

## Checklist

- [ ] Permission definitions exist under the owning app's `django/permissions/` package.
- [ ] Codenames use frontend-compatible `<subject>_<action>` naming.
- [ ] Definitions are discoverable by `PermissionService.get_app_permissions()`.
- [ ] Permission rows exist in all three permission tables after migration.
- [ ] Every supported REST action has an explicit permission check.
- [ ] Unknown actions fail closed.
- [ ] Global baseline RBAC remains enforced.
- [ ] Nested resource relationships and object access are validated.
- [ ] Related WebSocket endpoints mirror read authorization.
- [ ] Backend processes are restarted after Python changes.
- [ ] Session-cached permissions are refreshed after assignment changes.
- [ ] Unauthorized API requests return 403 even without frontend involvement.
- [ ] Backend lint and Django checks pass.
