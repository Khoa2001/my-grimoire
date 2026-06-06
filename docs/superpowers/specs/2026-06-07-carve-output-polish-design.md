# Carve Output Polish — Design Spec

## Problem

`/carve` plan output is verbose. Three sources of noise:
1. `Reason:` label — content duplicates the commit message
2. `Files:` label + full paths — long, redundant label, paths unreadably deep
3. Actions block — always visible, takes 6 lines, users learn it once

## Design

### Display format (Step 4 of SKILL.md)

**Before:**
```
── Carve plan ──────────────────────────────────────────
[1] feat(auth): add login flow
    Reason: Login feature implementation
    Files:
      lib/features/auth/presentation/views/login_view.dart
      lib/features/auth/presentation/bloc/auth_bloc.dart
────────────────────────────────────────────────────────
Total: 3 groups · 6 files

Actions:
  [commit all]       — commit all groups in order
  [commit 1 2]       — commit specific groups (space or comma separated)
  [skip 2]           — exclude group from plan
  [edit 2: message]  — change a group's commit message
  [stop]             — stop, leave everything dirty
```

**After:**
```
── Carve plan ──────────────────────────────────────────
[1] feat(auth): add login flow
    views/login_view.dart  bloc/auth_bloc.dart

[2] fix(ui): correct button padding
    widgets/app_button.dart

[3] chore: update dependencies
    pubspec.yaml  pubspec.lock
────────────────────────────────────────────────────────
3 groups · 6 files  (? for help)
```

### Rules

- **No `Reason:` line.** Commit message conveys intent.
- **Files: last 2 path segments**, space-separated on one line per group (or wrapped to next line if >3 files). No `Files:` label.
- **Footer:** `N groups · M files  (? for help)` — drop the `Total:` prefix.
- **Actions menu hidden.** Only shown when user types `?`.

### `?` help text

```
Actions:
  commit all       — commit all groups in order
  commit 1 2       — commit specific groups (space or comma separated)
  skip 2           — exclude group from plan
  edit 2: message  — change a group's commit message
  stop             — stop, leave everything dirty
```

## Scope

Single file: `~/.claude/skills/carve/SKILL.md`
- Modify Step 4 format example
- Add `?` handler in Step 5
