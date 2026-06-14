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

### 3. Resolve milestone to close

- If `milestone.current` is set, fetch it via `gh api repos/<owner/repo>/milestones?state=open` and match by title.
- If not set or not found in open milestones, list open milestones and let the user pick one (or enter to skip closing a milestone).

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

User can accept the suggestion (enter) or type any value. Validate it matches `^v?\d+\.\d+\.\d+$`. Re-ask on bad input.

Render the final tag using `release.tag_pattern` with `{version}` replaced by the user's input (strip a leading `v` from input first so `v{version}` doesn't double up).

### 5. Collect issues for notes

Find issues closed since the previous tag (or since repo start if no previous tag):

```bash
PREV=<latest tag or empty>
SINCE=$(git log -1 --format=%cI ${PREV:+$PREV})   # ISO date of prev tag commit, or empty

# fetch closed issues, then filter by closedAt > SINCE
gh issue list --repo <owner/repo> --state closed --limit 500 \
  --json number,title,labels,closedAt,milestone
```

In memory: keep issues with `closedAt > SINCE` (or all closed issues if no previous tag). Then **scope to the chosen milestone by default**: keep only issues whose `milestone.title == <chosen milestone>`. If no milestone was chosen in Step 3, keep all closed-since-prev-tag issues (nothing to scope to).

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
  Milestone:  <milestone title> (will close) | (none)
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

If a milestone was chosen, close it:

```bash
MS_NUMBER=$(gh api "repos/<owner/repo>/milestones?state=open" \
  --jq '.[] | select(.title=="<milestone>") | .number')
gh api -X PATCH "repos/<owner/repo>/milestones/$MS_NUMBER" -f state=closed
```

### 9. Open the next milestone

**Skip this entire step (no prompt, no suggest, no config update) when `milestone.required: false`.** Soft-milestone flow treats milestones as optional, so auto-suggesting the next one feels like coercion. Step 10 will also omit the "Next milestone:" line accordingly.

Only when `milestone.required: true`, after closing, ask:

```
Open next milestone? [Y/n] (suggest: v<next>)
```

Suggested name: increment the closed milestone's last numeric segment by one (e.g. `v0.3` → `v0.4`, `v0.3.0` → `v0.4.0`). If no milestone was closed, suggest `release.initial_version`.

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
| `milestone.required: true` + unfinished issues in milestone | Stop |
| `milestone.required: true` + closed issues since last tag without milestone | Stop |
| `milestone.required: false` + same conditions | Warn, proceed on confirm |
| `milestone.required: false` | Omit step 9 entirely (no next-milestone prompt, suggest, or config update) |
| Tag already exists locally or remotely | Stop with hint to pick a new version |

## Notes

- Solo follows trunk-based development: no release branches. Hotfix = new commit on trunk + new patch tag. If the user asks for a release branch, explain why solo does not support it (link to README's Releases section).
- Never `git push --force`. Never delete tags.
