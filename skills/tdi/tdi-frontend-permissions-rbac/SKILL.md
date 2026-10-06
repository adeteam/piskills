---
name: tdi-frontend-permissions-rbac
description: Implement and troubleshoot TDI frontend permissions using CASL abilities, RBAC-loaded backend permissions, module ability rules, and permission-gated navigation/UI controls.
---

# TDI Frontend Permissions (CASL + RBAC)

## Goal

Add or modify frontend authorization in a way that matches TDI’s permission system.

Also load `/skill:tdi-backend-permissions-rbac` when permissions must protect backend resources. Frontend visibility checks are not a security boundary.

## Architecture summary

TDI frontend permissions are built from these layers:

1. **Base ability instance**
   - `@ade/app/src/app.ability.ts` provides a shared `Ability` instance.
2. **Module default rules**
   - Each module can register rules in `<module>.ability.ts` via `AuthenticationService.registerDefaultRules(...)`.
3. **Dynamic per-user rules**
   - `AuthenticationService.loadUserRules()` pulls user permissions from backend when RBAC is enforced.
4. **Runtime checks**
   - UI uses `authentication.can(action, subject, field?)`.

## Rule sources and loading flow

- Backend permission records come from `permission` model (`/api/wmsauth/permission/`).
- `PermissionRecord.toRules()` maps codename `subject_action` to CASL rules.
- `AuthenticationService.loadUserRules()`:
  - starts with default rules,
  - applies default rule builders,
  - loads backend permissions when RBAC enforce is on,
  - updates ability with final merged rules.

Important behavior:
- Superuser under RBAC enforce gets full allow (`CanRule(AllAction, AllSubject)`).
- If RBAC enforce is off, frontend opens all access.

## How to add permission-aware UI behavior

## 1) Gate actions in component/datatable code

Use `authentication.can(...)` checks near action definitions.

Examples in repo:
- datatable actions disabled when user lacks permission
- field-level checks like `can('edit', record, 'groups')`

Pattern:

```ts
if (!this.authentication.can('view', 'User')) {
  // hide/disable
}

const allowed = this.authentication.can('edit', userRecord, 'password');
```

Use a record object when conditions matter (`is_system`, `external`, etc.).

## 2) Gate nav/dashboard items

Nav config and dashboard config can include `permissions` entries.
- `UiService` filters items by mode + permission checks.
- If RBAC enforce is enabled, at least one configured permission must pass.

Legacy nav format seen in repo:

```ts
permissions: [
  ['User', 'view'],
  ['Group', 'view']
]
```

Typed format (`PermissionConfig`) is also supported in newer config paths:

```ts
permissions: [
  { subject: 'User', action: 'view' }
]
```

## 3) Add default module rules when needed

In `<module>.ability.ts`, register default rules in module constructor:

```ts
auth.registerDefaultRules(rules);
```

Use this for global module constraints that should apply independent of backend permission records.

## 4) Add dynamic rule builders for context-aware rules

For per-user rules (e.g., user can edit own profile fields), register builder:

```ts
auth.registerDefaultRuleBuilders((user) => [
  // build CanRule/CannotRule based on user
]);
```

See `auth.ability.ts` for concrete patterns.

## Naming and normalization notes

- CASL wrapper normalizes actions/subjects and supports aliases:
  - backend style: `add/view/edit/remove`
  - CASL style: `create/read/update/delete`
- Subject names are normalized by helper logic; keep naming consistent with existing models/records.

## Common failure modes

- Added backend permission but frontend never checks it.
- UI checks wrong subject/action casing or wrong subject name.
- Condition-based rules checked against string subject instead of record instance.
- Nav/dashboard permissions configured but mismatch expected tuple/object format in that config path.
- Expecting RBAC behavior while environment has RBAC enforce disabled.

## Implementation checklist

- [ ] Permission requirement mapped to explicit `action + subject`.
- [ ] Component action/buttons gated with `authentication.can(...)` where required.
- [ ] Nav/dashboard visibility permissions updated if feature should be menu-gated.
- [ ] Module default rules / dynamic builders updated when behavior is not purely backend-driven.
- [ ] Tested with RBAC enforce on and representative user roles.
- [ ] Ran `yarn lint`, `yarn test-auto`, and `yarn build` in module/workspace as needed.
