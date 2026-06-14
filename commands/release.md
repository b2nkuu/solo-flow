---
description: Cut a release from trunk — tag, GitHub Release, close milestone
argument-hint: "[--dry-run]"
allowed-tools: [Bash]
---

# /solo:release

Cut a release from trunk in one command. Tag, generate notes from closed issues, close the milestone, open the next one. Trunk-based: no release branch, ever.

## Input

`$ARGUMENTS` — optional. Recognized:

- `--dry-run` — print the preview and stop. No tag, no push, no GitHub Release, no milestone changes.
- `--include-all-closes` — restore previous Step 5 behavior: include every issue closed since the previous tag in Notes, regardless of milestone. Default scopes Notes to the chosen milestone only.

Anything else: ignore (no version flags, no `--milestone`, no `--patch` — keep the surface tiny).

## Steps

### 1. Pre-flight

- `gh auth status` (stop on failure).
- Resolve repo as usual.
- Read `.solo/config.yml`: `trunk.name` (default `main`), `release.tag_pattern` (default `v{version}`), `release.initial_version` (default `0.1.0`), `milestone.current`, `milestone.required` (default `true`).

### 2. Trunk-based invariants

All must pass — no overrides:

- Inside a git work tree (`git rev-parse --is-inside-work-tree` is `true`).
- `origin` remote exists.
- Current branch == `trunk.name`. If not: stop with `❌ /solo:release must run on <trunk>. git switch <trunk>.`
- Working tree clean (`git status --porcelain` empty). If not: stop with `❌ working tree dirty. commit or stash first.`
- Local trunk in sync with `origin/<trunk>` (`git fetch origin <trunk>` then compare `HEAD` vs `origin/<trunk>`). If behind or diverged: stop with `❌ <trunk> not in sync with origin. git pull first.`
- **No open PR closes an issue in the chosen milestone.** After Step 3 has resolved the milestone, list open PRs and scan each body for `Closes #<n>` (case-insensitive, also matches `Fixes #<n>` / `Resolves #<n>`). For every `<n>` referenced, look up the issue and check `milestone.title == <chosen milestone>`. If any match, stop with:
  ```
  ❌ open PRs still close issues in milestone <name>:
       PR #<pr> closes #<n> <issue title>
       …
     Merge or detach them before releasing.
  ```
  This prevents silent ship failure where `/solo:done` closed the issue but the PR carrying the code is still open — trunk has no code, but Notes would announce it shipped. Skip this guard if no milestone was chosen.

> Note: this guard runs after Step 3 (milestone resolution) even though it is listed here for grouping with the other invariants. Implementations may evaluate it immediately after the milestone is known.

### 3. Resolve milestone to close

- If `milestone.current` is set, fetch it via `gh api repos/<owner/repo>/milestones?state=open` and match by title.
- If not set or not found in open milestones, list open milestones and let the user pick one (or enter to skip closing a milestone).

Capture the milestone's `number`, `title`, `open_issues`, and `closed_issues` from the API response. Reuse `number` in Step 8 (close API call) and the issue counts in Step 6 (preview progress line) — do **not** re-query the API by title later.

A release without a milestone is allowed (e.g. an emergency patch). Just skip the close-milestone step later.

### 4. Compute next version

Find the latest tag matching `release.tag_pattern`:

```bash
git tag --list "v*" --sort=-v:refname | head -n 1
```

- No tag yet → suggest `release.initial_version`.
- Has tag → suggest patch bump (e.g. `v0.3.1` → `v0.3.2`).

Always ask:

```
Next version (latest: <tag-or-none>, suggest: <suggested>):
```

User can accept the suggestion (enter) or type any value. Validate it matches `^v?\d+\.\d+\.\d+$` — pre-release suffixes (`-rc1`, `-beta`, etc.) are not accepted. Re-ask on bad input with: `❌ pre-release suffixes not supported. Use plain semver (e.g. v0.4.0).`

Render the final tag using `release.tag_pattern` with `{version}` replaced by the user's input (strip a leading `v` from input first so `v{version}` doesn't double up).

### 5. Collect issues for notes

Find issues closed since the previous tag (or since repo start if no previous tag):

```bash
PREV=<latest tag or empty>
SINCE=""
[ -n "$PREV" ] && SINCE=$(git log -1 --format=%cI "$PREV")   # ISO date of prev tag commit, or empty when no prior tag

# fetch closed issues, then filter by closedAt > SINCE (only when SINCE is non-empty)
gh issue list --repo <owner/repo> --state closed --limit 500 \
  --json number,title,labels,closedAt,milestone
```

In memory: if `SINCE` is non-empty, keep issues with `closedAt > SINCE`. If `SINCE` is empty (no previous tag), keep **all** closed issues — the latent bug to avoid is letting `SINCE` default to `HEAD`'s commit date, which would filter every issue out.

Then **scope to the chosen milestone by default**: keep only issues whose `milestone.title == <chosen milestone>`. If no milestone was chosen in Step 3, keep every issue that passed the SINCE filter (nothing to scope to).

If `--include-all-closes` was passed, skip the milestone-scope filter and keep every closed-since-prev-tag issue (previous behavior).

Group the remaining issues by `type:*` label:

- `type:feature` → **Features**
- `type:bug` → **Fixes**
- everything else (`type:task`, `type:idea`, `type:research`, no type) → **Other**

### 6. Preview

```
📦 Release preview

  Tag:        <new-tag>
  Trunk:      <trunk> @ <short-sha>
  Milestone:  <milestone title> (<closed>/<total> closed, will close) | (none)
  Previous:   <prev-tag or "no prior release">

Notes:
  ## Features
  - #42 add dark mode
  ## Fixes
  - #51 fix login redirect
  ## Other
  - #48 docs cleanup

⚠ <N> closed issues since <prev-tag> are not attached to milestone <milestone>:
  - #50 chore: bump deps
  - #53 fix typo
```

Compute `<closed>` and `<total>` from the counts captured in Step 3 (`closed_issues` and `closed_issues + open_issues`). Example: `Milestone: v0.4 (3/5 closed, will close)`.

The ⚠ block only appears when a milestone was chosen AND there are orphan issues. **Orphan = closed since previous tag AND (`milestone == null` OR `milestone.title != <chosen milestone>`)**. Enumerate orphans independently of the `--include-all-closes` flag — the warning reflects repo hygiene, not what ends up in Notes.

**Block conditions** (stop with error, do not proceed):

- `milestone.required: true` AND any open issue exists with `milestone == <chosen milestone>` AND `status:in-progress` or `status:blocked`:
  ```
  ❌ milestone <name> has N unfinished issues:
       #<n> [<status>] <title>
       …
     Finish or move them before releasing.
  ```
- `milestone.required: true` AND the ⚠ "closed issues without milestone" list above is non-empty:
  ```
  ❌ N closed issues since <prev-tag> have no milestone:
       #<n> <title>
       …
     Assign them via /solo:plan first.
  ```

If `milestone.required: false`, those become warnings only — proceed.

### 7. Confirm or dry-run

- `--dry-run` → stop here. Print `(dry-run — nothing pushed.)` and exit.
- Else: `Proceed? [y/N]` — only `y` proceeds.

### 8. Execute

Write the notes body to a temp file. Then:

```bash
git tag -a <new-tag> -m "<new-tag>"
git push origin <new-tag>

gh release create <new-tag> \
  --repo <owner/repo> \
  --target <trunk> \
  --title "<new-tag>" \
  --notes-file <tempfile>
```

If a milestone was chosen, close it using the `number` captured in Step 3 (no re-query by title):

```bash
gh api -X PATCH "repos/<owner/repo>/milestones/$MS_NUMBER" -f state=closed
```

### 9. Open the next milestone

**Skip this entire step (no prompt, no suggest, no config update) when `milestone.required: false`.** Soft-milestone flow treats milestones as optional, so auto-suggesting the next one feels like coercion. Step 10 will also omit the "Next milestone:" line accordingly.

Only when `milestone.required: true`, after closing, ask:

```
Open next milestone? [Y/n] (suggest: v<next>)
```

Suggested name: **minor bump** of the closed milestone (e.g. `v0.3` → `v0.4`, `v0.3.0` → `v0.4.0`). Concretely: for `vX.Y` → `vX.(Y+1)`; for `vX.Y.Z` → `vX.(Y+1).0`. Patch milestones (e.g. `v0.3.1`) are not auto-suggested — patch tags belong to the hotfix flow (see Notes) and do not create or close a milestone. If no milestone was closed, suggest `release.initial_version`.

User can accept, type a custom name, or `n` to skip.

If accepted:

```bash
gh api repos/<owner/repo>/milestones -f title="<name>" 2>/dev/null || true
```

Then update `.solo/config.yml` `milestone.current:` to the new name (or empty string if skipped). Edit in place — don't rewrite the rest of the file.

### 10. Confirm

```
🚀 Released <new-tag>
   URL: <release URL from gh>
   Closed milestone: <name>           (omit line if none)
   Next milestone: <name> (current)   (omit line if skipped OR if step 9 was skipped because milestone.required: false)
```

## Guards summary

| Condition | Action |
|---|---|
| Not on trunk | Stop |
| Dirty work tree | Stop |
| Trunk not in sync with origin | Stop |
| Open PR body has `Closes #<n>` for an issue in the chosen milestone | Stop (list offending PRs) |
| `milestone.required: true` + unfinished issues in milestone | Stop |
| `milestone.required: true` + closed issues since last tag without milestone | Stop |
| `milestone.required: false` + same conditions | Warn, proceed on confirm |
| `milestone.required: false` | Omit step 9 entirely (no next-milestone prompt, suggest, or config update) |
| Tag already exists locally or remotely | Stop with hint to pick a new version |

**Notes scope** (Step 5): Release Notes are scoped to the chosen milestone by default — only closed-since-prev-tag issues with `milestone.title == <chosen milestone>` appear. Pass `--include-all-closes` to fall back to the old behavior (every closed-since-prev-tag issue, regardless of milestone). The ⚠ orphan list in the preview is independent of this flag.

## Notes

- Solo follows trunk-based development: no release branches. Hotfix = new commit on trunk + new patch tag. Patch tags (e.g. `v0.3.1`) ship through the hotfix flow and do **not** create or close a milestone — milestones track minor/major scope only (`x.(y+1).0`). If the user asks for a release branch, explain why solo does not support it (link to README's Releases section).
- Never `git push --force`. Never delete tags.
