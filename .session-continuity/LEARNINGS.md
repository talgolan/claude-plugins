# Learnings — claude-plugins migration

## 0. Same-session `/plugin marketplace remove` + re-`add` leaves a stale in-memory plugin index → installs fail with "not found"

**Symptom:** Phase E on host. `/plugin marketplace remove talgolan` then
`/plugin marketplace add talgolan/claude-plugins` both succeeded. Then
`/plugin install smoke-test-plugin@talgolan` → "Plugin smoke-test-plugin not
found in marketplace talgolan". Same for memory-audit-plugin.
`/plugin marketplace update talgolan` did NOT fix it.

**What was verified correct on disk (ruled out as cause):**
- `marketplace.json` schema matches official docs exactly: per-plugin
  `"source": {"source": "github", "repo": "owner/repo"}` is the correct form.
- `claude plugin validate .` passes (only a missing-description warning).
- Cached `~/.claude/plugins/marketplaces/talgolan/.claude-plugin/marketplace.json`
  is byte-identical to repo HEAD, lists all 3 plugin names exactly.
- `known_marketplaces.json` talgolan → `talgolan/claude-plugins`, freshly fetched.
- Plugin sub-repos reachable (`git ls-remote` OK).
- The 374KB `plugin-catalog-cache.json` is the unrelated *official discover*
  catalog (has `models`, official plugins) — never holds a user marketplace; red herring.

**Cause:** The running CC session rebuilt the on-disk marketplace cache but did
not rebuild its in-memory marketplace→plugin index for the re-added name.

**Fix:** Restart Claude Code, then `/plugin install <name>@talgolan`. (Unverified
this session — left for next session; if it still fails fresh, it's a genuine CC
bug with re-added marketplaces, not a repo-side problem.)

**Lesson:** When repointing a marketplace, prefer restarting CC over
remove+re-add within one session. If you must do it live and installs 404,
restart before assuming the catalog is broken.

## 1. A plugin clone may be checked out on a non-default branch — verify before `git rm` of catalog files

**Symptom:** Ran `git rm .claude-plugin/marketplace.json` + commit + push in
`session-continuity-plugin`; push succeeded but the file was still present on
GitHub's default branch, so a marketplace refresh would still see it.

**Cause:** The local clone was on feature branch `spec/improvement-roadmap`, not
`main`. The removal commit landed on the feature branch. Marketplaces read the
**default branch** (`main`), where the file was untouched.

**Fix:** Cherry-picked the isolated removal commit onto `main`
(`git checkout main; git reset --hard origin/main; git cherry-pick <sha>`) and
pushed main. Left the feature branch as-is.

**Lesson:** Before any catalog/manifest mutation in a plugin repo, check
`git branch --show-current` vs `gh repo view --json defaultBranchRef`. If they
differ, target the default branch explicitly — don't assume the clone is on it.

## 2. `bun install` does not rewrite the embedded workspace name in bun.lock

After renaming `package.json` `"name"`, `bun install` reported OK but
`bun.lock` line 6 (`workspaces.""."name"`) still held the old name. Had to edit
bun.lock by hand. When renaming a bun project, grep bun.lock and patch the
workspace name directly.

## 3. Phase E (`/plugin ...`) only ever affects the store of the CC instance running it

The migration ran inside the itb container. The container's own
`~/.claude/plugins/` is a *separate* store from the host's. Running `/plugin`
commands in-container would repoint the container store, not the host install
the operator cares about. Verified by reading the in-container
`known_marketplaces.json` — it had its own `talgolan` entry. Phase E is
host-only by construction, not just convention.
