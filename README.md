# Flutter skills for Claude Code

Claude Code skills for setting up Flutter apps with a feature-first clean architecture.

| Skill | What it does |
| --- | --- |
| `flutter-init` | Folder structure (core/ + features/), DI, bootstrap, error/result types, "add a feature" scaffold. Runs the others. |
| `flutter-config` | Environments via `--dart-define-from-file`, typed AppConfig, flavors, Dio ApiClient + interceptors, storage, constants. |
| `flutter-theme` | Design tokens, light/dark ThemeExtension, fonts (incl. RTL), text scale, spacing, ThemeCubit. |
| `flutter-routing` | go_router with typed AppRoute enum, auth guard, StatefulShellRoute bottom nav, enum conventions. |
| `flutter-components` | Shared component library (AppButton, AppTextField, AppImage, AppStatusView, …). |

## Install

Link the skills into your personal skills folder so edits here take effect right away:

```sh
for s in skills/*; do ln -sfn "$PWD/$s" ~/.claude/skills/"$(basename "$s")"; done
```

Or copy them into a project's `.claude/skills/` to use them only there.
