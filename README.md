# pi-skills

Shared Pi skills repository for team-wide workflows.

## Structure

- `skills/common/` - Skills that apply across all repositories
- `skills/tdi/` - Skills specific to the TDI codebase
- `skills/other-repo/` - Skills specific to another codebase

## Use from a project

Add this package path in `.pi/settings.json`:

```json
{
  "packages": ["/opt/piskills"]
}
```

Or install as a project package:

```bash
pi install -l /opt/piskills
```
