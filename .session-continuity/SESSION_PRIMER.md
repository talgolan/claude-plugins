# Session primer — claude-plugins marketplace migration

## What this repo is

`talgolan/claude-plugins` — the dedicated marketplace catalog (marketplace
`name` field = `talgolan`, so installs are `<plugin>@talgolan`). Holds
`.claude-plugin/marketplace.json` listing 3 plugins by github source. Each
plugin lives in its own repo; the per-repo marketplace.json files were removed
so this catalog is the single source of truth.

## Current state (2026-06-17)

Migration runbook (`MIGRATION-RUNBOOK.md`) executed inside the itb container.
Phases A–D + dir rename DONE. Phase E pending (HOST-ONLY). Phase F pending.

| Phase | What | Status |
|---|---|---|
| A | publish `talgolan/claude-plugins` (this repo), marketplace.json at root | ✅ done |
| B | publish `talgolan/memory-audit-plugin`, in-repo marketplace.json removed | ✅ done |
| C | remove `talgolan/session-continuity` in-repo marketplace.json (on **main**) | ✅ done |
| D | rename `smoke-test-skill`→`smoke-test-plugin` (repo+plugin.json+pkg+bun.lock+README), drop marketplace.json, 67 tests pass, local dir renamed | ✅ done |
| E | repoint HOST itb install: remove old `talgolan` mkt, add `talgolan/claude-plugins`, reinstall plugins | ⏳ **HOST-ONLY — do on host CC session** |
| F | update itb prose refs `smoke-test-skill`→`smoke-test-plugin` | ⏳ pending (non-blocking) |

## Outstanding items

1. **Phase E (host)** — the host `talgolan` marketplace is still sourced from
   `talgolan/smoke-test-skill` (renamed; its marketplace.json deleted, so it
   will fail on refresh). On the HOST Claude Code session run:
   ```
   /plugin marketplace remove talgolan
   /plugin marketplace add talgolan/claude-plugins
   /plugin install smoke-test-plugin@talgolan
   /plugin install memory-audit-plugin@talgolan
   ```
   Do NOT reinstall session-continuity under talgolan — it's installed from its
   own `talgolan/session-continuity` marketplace and stays there.
2. **Phase F** — prose/memory mentions of `smoke-test-skill` in itb docs +
   project memory dir (see runbook Phase F list). Non-blocking accuracy fixes.
3. Local dir `smoke-test-skill` was renamed to `smoke-test-plugin` in the
   container working tree (`/home/assistant/TG/smoke-test-plugin`).

## Gotchas hit this run

- session-continuity clone was checked out on feature branch
  `spec/improvement-roadmap`, not `main`. The `git rm` commit first landed on
  the feature branch; default branch is `main`, which marketplaces read. Fixed
  by cherry-picking the removal commit onto `main` and pushing main. Feature
  branch still carries the removal (harmless).
- `bun install` does NOT rewrite the embedded workspace `name` in `bun.lock`;
  had to edit line 6 by hand to `smoke-test-plugin`.
- gh token scopes: `repo, workflow, gist, read:org` — **no `delete_repo`**, so
  the runbook's delete-based rollback would fail. Forward path needs no delete.

## Repos (all public, talgolan)

- `claude-plugins` — this catalog. marketplace.json at root `.claude-plugin/`.
- `memory-audit-plugin` — `/memory-audit` auditor.
- `smoke-test-plugin` — renamed from smoke-test-skill. `/smoke-init|add|sync`.
- `session-continuity` — repo keeps no `-plugin` suffix; plugin name `session-continuity`.
