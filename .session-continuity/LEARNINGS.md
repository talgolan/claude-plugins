# Learnings — claude-plugins migration

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
