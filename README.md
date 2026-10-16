# claude-skills

My Claude Code skills, packaged as a plugin marketplace. Each area is a plugin, so a
project installs only what it needs.

| Plugin | Area | Skills |
| --- | --- | --- |
| [`flutter`](plugins/flutter) | Mobile | `flutter-init`, `flutter-config`, `flutter-theme`, `flutter-routing`, `flutter-components` |
| `backend` | Backend | _planned_ |
| `web` | Web | _planned_ |
| `admin` | Admin dashboards | _planned_ |

## Install

In Claude Code:

```
/plugin marketplace add <github-user>/claude-skills      # or a local path to this repo
/plugin install flutter@claude-skills
```

Skills from a plugin are namespaced, e.g. `/flutter:flutter-config`. Update later with
`/plugin marketplace update claude-skills`.

## Layout

```
.claude-plugin/marketplace.json     # lists the plugins
plugins/
  <plugin>/
    .claude-plugin/plugin.json      # name, version, description
    skills/<skill-name>/SKILL.md    # one folder per skill
```

## Adding a plugin (e.g. backend)

1. Create `plugins/backend/.claude-plugin/plugin.json` with `name`, `version`, `description`.
2. Add skills under `plugins/backend/skills/<skill-name>/SKILL.md`.
3. Add an entry to `plugins` in `.claude-plugin/marketplace.json`.

## Adding a skill to an existing plugin

Create `plugins/<plugin>/skills/<skill-name>/SKILL.md` with `name` and `description`
frontmatter, and bump the plugin's `version` in `plugin.json`.
