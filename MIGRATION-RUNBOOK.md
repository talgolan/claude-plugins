# Migration runbook — dedicated `talgolan/claude-plugins` marketplace

Execute in a fresh session. Goal: stand up a dedicated marketplace repo, rename
`smoke-test-skill` → `smoke-test-plugin`, publish `memory-audit-plugin`, and
repoint the itb install at the new catalog. All steps are doc-verified against
https://code.claude.com/docs/en/plugin-marketplaces and `.../plugins-reference`.

## ⚠️ Executing this runbook INSIDE itb (real-world test, 2026-06-17)

The operator plans to run this migration from inside an itb container as a
dogfood test. Container execution changes four things — read before starting:

1. **Launch itb in the PARENT `TG/` dir, not one plugin dir.** itb mounts only
   the project you launch it in, at `/home/assistant/<projectName>`. To touch all
   four repos (claude-plugins, memory-audit-plugin, smoke-test-skill,
   session-continuity-plugin) you need them all visible. Launch:
   `cd ~/active_development/TG && ITB_PROFILE= itb run .` → inside, the work dir is
   `/home/assistant/TG/` and the four repos are subdirs there. EVERY `cd` path in
   the phases below becomes `/home/assistant/TG/<repo>` (NOT `~/active_development/TG/<repo>`).

2. **`gh` must be authed IN the container.** itb forwards a token only when
   `ITB_GH_TOKEN` is set at launch (entrypoint writes `~/.config/gh/hosts.yml`
   from it; default host github.com). BEFORE launching, on the HOST:
   `export ITB_GH_TOKEN="$(gh auth token)"` then `itb run .`. FIRST step inside the
   container: `gh auth status` — must show authenticated to github.com. If not,
   the `gh repo create/rename` calls fail; fix the token forward before proceeding.
   Token needs `repo` + `delete_repo` scopes (delete only if you use rollback).

3. **Phase E is HOST-ONLY — do NOT run it inside itb.** `/plugin marketplace
   add/remove/install` configures the HOST Claude Code plugin store
   (`~/.claude/plugins/`). Running it inside the container does nothing to the
   host install. Exit itb and run Phase E in your normal host Claude Code session.

4. **`claude plugin validate` may be absent in-container.** If the `claude` CLI
   isn't installed in the container, skip the validate step (Phase B) or run it
   on the host; the `zsh -n` syntax check still works in-container.

Everything else (Phases A-D, F git work) runs fine inside itb once gh is authed
and you've launched from the parent dir.

## Context (state at handoff, 2026-06-17)

- **Local dirs already prepared** under `~/active_development/TG/`:
  - `claude-plugins/` — the dedicated marketplace repo (this dir). Has
    `.claude-plugin/marketplace.json` (catalog of all 3 plugins, github sources)
    + `README.md`. NOT a git repo yet, NOT on GitHub.
  - `memory-audit-plugin/` — complete plugin (plugin.json, skills/memory-audit/
    SKILL.md, scripts/memory-audit.zsh, README, LICENSE). NOT a git repo yet.
    Still carries an in-repo `.claude-plugin/marketplace.json` that must be
    DELETED (catalog now centralized) — see Phase C.
  - `smoke-test-skill/` — live published repo `talgolan/smoke-test-skill`.
    Carries an in-repo `.claude-plugin/marketplace.json` (the OLD `talgolan`
    marketplace) that must be deleted. plugin.json name = `smoke-test-skill`.
  - `session-continuity-plugin/` — live repo `talgolan/session-continuity`
    (note: repo name has NO `-plugin` suffix; plugin.json name = `session-continuity`).
    Carries an in-repo marketplace.json to delete.
- **itb install today**: `~/.claude/plugins/known_marketplaces.json` has a
  `talgolan` marketplace sourced from `github:talgolan/smoke-test-skill`;
  `installed_plugins.json` has key `smoke-test-skill@talgolan` v0.7.1.
- **gh**: authenticated as `talgolan` (keyring). `gh repo rename`/`create` available.

## Naming decisions (locked)

- Dedicated marketplace repo: **`talgolan/claude-plugins`**.
- Marketplace `name` field stays **`talgolan`** (so installs are `<plugin>@talgolan`).
- Rename: **`smoke-test-skill` → `smoke-test-plugin`** (repo + plugin.json name).
- `session-continuity` keeps its name (already correct).
- Each plugin repo loses its in-repo `marketplace.json`; the catalog is centralized.

## Safe execution order (nothing breaks until the rename, which GitHub redirects)

GitHub auto-creates a redirect on `gh repo rename`, so old `talgolan/smoke-test-skill`
references keep resolving after the rename. Creating the two new repos is purely
additive. The only fragile moment is removing the old in-repo marketplace.json,
covered by repointing itb in Phase E.

---

### Phase A — publish the dedicated marketplace repo (additive, safe)

```sh
cd ~/active_development/TG/claude-plugins
git init -b main
git add -A
git commit -m "feat: dedicated talgolan marketplace catalog (3 plugins)"
gh repo create talgolan/claude-plugins --public --source=. --remote=origin --push
```

VERIFY:
```sh
gh repo view talgolan/claude-plugins --json name,visibility
# marketplace.json must be at repo root .claude-plugin/marketplace.json
gh api repos/talgolan/claude-plugins/contents/.claude-plugin/marketplace.json --jq '.name'
```

---

### Phase B — publish memory-audit-plugin (additive, safe)

First DELETE its in-repo marketplace.json (catalog is centralized now):
```sh
cd ~/active_development/TG/memory-audit-plugin
rm .claude-plugin/marketplace.json
```

Validate the plugin before publishing:
```sh
claude plugin validate . --strict     # expect: passes (only `name` is required)
zsh -n scripts/memory-audit.zsh && echo "script syntax OK"
```

Publish:
```sh
git init -b main
git add -A
git commit -m "feat: memory-audit-plugin — read-only memory-dir auditor (/memory-audit)"
gh repo create talgolan/memory-audit-plugin --public --source=. --remote=origin --push
```

VERIFY:
```sh
gh repo view talgolan/memory-audit-plugin --json name
gh api repos/talgolan/memory-audit-plugin/contents/.claude-plugin/plugin.json --jq '.name'
```

---

### Phase C — clean session-continuity's in-repo marketplace.json

```sh
cd ~/active_development/TG/session-continuity-plugin
git rm .claude-plugin/marketplace.json
git commit -m "chore: remove per-repo marketplace.json — catalog moved to talgolan/claude-plugins"
git push
```
(plugin.json stays; only the marketplace.json is removed.)

---

### Phase D — rename smoke-test-skill → smoke-test-plugin

```sh
cd ~/active_development/TG/smoke-test-skill

# 1. Rename on GitHub (creates a redirect from the old name).
gh repo rename smoke-test-plugin -R talgolan/smoke-test-skill --yes

# 2. Point the local clone at the new URL.
git remote set-url origin https://github.com/talgolan/smoke-test-plugin.git

# 3. Update the plugin's identity + remove its in-repo marketplace.json.
#    - .claude-plugin/plugin.json: "name": "smoke-test-skill" -> "smoke-test-plugin"
#    - package.json: "name" field if it says smoke-test-skill
#    - README.md: title + any install commands referencing the old name
#    - rm .claude-plugin/marketplace.json   (catalog centralized)
```

Edit those files (use Edit, not sed, to keep it reviewable), then:
```sh
rm .claude-plugin/marketplace.json
git add -A
git commit -m "chore: rename smoke-test-skill -> smoke-test-plugin; drop per-repo marketplace.json"
git push
# Optionally rename the LOCAL dir too, for consistency:
cd .. && mv smoke-test-skill smoke-test-plugin
```

VERIFY:
```sh
gh repo view talgolan/smoke-test-plugin --json name      # resolves
gh api repos/talgolan/smoke-test-plugin/contents/.claude-plugin/plugin.json --jq '.name'  # smoke-test-plugin
```

CAUTION: renaming the plugin `name` changes the install key. itb's current
`smoke-test-skill@talgolan` install becomes stale — Phase E reinstalls it as
`smoke-test-plugin@talgolan`.

---

### Phase E — repoint the itb install at the new marketplace

The old `talgolan` marketplace (sourced from the renamed smoke repo, now missing
its marketplace.json) must be replaced with the dedicated one.

In a Claude Code session (these are `/plugin` slash commands, not shell):
```
/plugin marketplace remove talgolan
/plugin marketplace add talgolan/claude-plugins
/plugin install smoke-test-plugin@talgolan
/plugin install memory-audit-plugin@talgolan
```
(session-continuity: reinstall only if it was installed before — check
`installed_plugins.json`.)

VERIFY (shell):
```sh
cat ~/.claude/plugins/known_marketplaces.json | python3 -m json.tool | grep -A4 talgolan
# source.repo should now be talgolan/claude-plugins
cat ~/.claude/plugins/installed_plugins.json | python3 -m json.tool | grep -i "smoke-test\|memory-audit"
# expect smoke-test-plugin@talgolan (NOT smoke-test-skill@talgolan)
```
VERIFY (functional): in itb, run `/smoke-init --list` or `/smoke-add` to confirm
the renamed plugin's commands still resolve; run `/memory-audit` to confirm the
new plugin loads.

---

### Phase F — update itb prose references (non-blocking, do after E works)

These are doc/memory mentions of the old name — update for accuracy:
```
~/active_development/TG/itb/.session-continuity/SESSION_PRIMER.md
~/active_development/TG/itb/.session-continuity/LEARNINGS.md
~/active_development/TG/itb/docs/superpowers/plans/2026-06-03-dev-build-loop.md
~/active_development/TG/itb/docs/superpowers/plans/2026-06-15-model-backend-auth.md
```
And the itb-project memory dir mentions:
```
~/.claude/projects/-Users-tal-golan-active-development-TG-itb/memory/feedback_smoke_test_mandatory.md
.../feedback_bash_cwd_resets.md
.../project_smoke_skill_daemon_manual_backport.md
```
`smoke-test-skill` → `smoke-test-plugin`. (Historical plan docs may keep the old
name as point-in-time accurate — judgment call; primer/memory should be current.)

---

## Rollback

- New repos (claude-plugins, memory-audit-plugin): `gh repo delete` if needed.
- Repo rename: `gh repo rename smoke-test-skill -R talgolan/smoke-test-plugin --yes`
  to revert; GitHub redirect from the reverted name re-establishes.
- itb install: re-add the old marketplace `gh:talgolan/smoke-test-skill` and
  reinstall `smoke-test-skill@talgolan` (the redirect keeps the old source URL
  working).

## Post-migration cleanup checklist

- [ ] `talgolan/claude-plugins` live, marketplace.json at root `.claude-plugin/`.
- [ ] `talgolan/memory-audit-plugin` live, no in-repo marketplace.json.
- [ ] `talgolan/smoke-test-plugin` live (renamed), plugin.json name updated, no in-repo marketplace.json.
- [ ] `talgolan/session-continuity` — in-repo marketplace.json removed.
- [ ] itb: known_marketplaces `talgolan` → source `talgolan/claude-plugins`.
- [ ] itb: installed `smoke-test-plugin@talgolan` + `memory-audit-plugin@talgolan`.
- [ ] `/smoke-*` and `/memory-audit` resolve in an itb session.
- [ ] itb primer/memory prose updated to the new name.
- [ ] Local dir `smoke-test-skill` renamed to `smoke-test-plugin` (optional, for consistency).
