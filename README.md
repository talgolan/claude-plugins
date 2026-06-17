# talgolan/claude-plugins

The dedicated **Claude Code plugin marketplace** for Tal Golan's plugins. This
repo holds only the catalog (`.claude-plugin/marketplace.json`); each plugin
lives in its own repo and is referenced by github source.

## Plugins in this marketplace

| Plugin | Repo | What it does |
|---|---|---|
| `smoke-test-plugin` | `talgolan/smoke-test-plugin` | Scaffold an executable smoke-test framework into any project (`/smoke-init`, `/smoke-add`, `/smoke-sync`). |
| `session-continuity` | `talgolan/session-continuity` | Cross-session project memory (`SESSION_PRIMER.md` + `LEARNINGS.md`) with fire-before-action hooks. |
| `memory-audit-plugin` | `talgolan/memory-audit-plugin` | Audit a project memory dir for rot; propose approval-gated fixes (`/memory-audit`). |

## Install

```
/plugin marketplace add talgolan/claude-plugins
/plugin install smoke-test-plugin@talgolan
/plugin install session-continuity@talgolan
/plugin install memory-audit-plugin@talgolan
```

Update the catalog later:

```
/plugin marketplace update talgolan
```

## Maintaining the catalog

- The marketplace `name` is `talgolan` (the namespace users install from:
  `<plugin>@talgolan`).
- Each `plugins[]` entry uses a github source pointing at the plugin's own repo.
  Adding a plugin = add one entry here + bump nothing else (versions live in each
  plugin's `plugin.json`).
- Plugins do NOT carry their own `marketplace.json` — this repo is the single
  catalog. (Migrated 2026-06 from per-repo marketplaces; see MIGRATION-RUNBOOK.md.)

## Status

⚠️ NOT YET LIVE on GitHub. This repo + the rename of `smoke-test-skill` →
`smoke-test-plugin` + publishing `memory-audit-plugin` are executed via
`MIGRATION-RUNBOOK.md` in a dedicated session.
