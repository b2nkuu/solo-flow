---
description: Mark a task done — record outcome and close the issue
argument-hint: "<issue-number>"
allowed-tools: [Bash]
---

# /solo:done

Finish a task: record an optional outcome line, set `completed`, flip status, close the issue.

## Input

`$ARGUMENTS` = issue number.

## Steps

### 1. Pre-flight + resolve repo

- `gh auth status` (stop on failure).
- Resolve repo as usual.

### 2. Fetch issue

```bash
gh issue view <n> --repo <owner/repo> --json number,title,body,labels,state
```

If already closed → `ℹ️  #<n> already closed.` and stop.

### 3. Ask once for outcome + acceptance + test plan (optional)

Parse both `## Acceptance` and `## Test Plan` from the body. For each section, list every `- [ ]` / `- [x]` item with an index. A section is "skippable" if it is missing or contains only the empty `- [ ]` placeholder — drop it from the prompt entirely. If both sections are skippable, jump straight to the outcome question.

Prompt in one block (omit any skippable section):

```
Acceptance items (<N>):
  1. [<x or space>] <item 1>
  2. [<x or space>] <item 2>
  …

Test Plan items (<M>):
  1. [<x or space>] <item 1>
  …

Tick all? [Y/edit/n]
One-line outcome (enter to skip):
```

`Tick all?` covers both sections as a single decision (the common case: the work is done, everything ticks). Read the responses (Y/edit/n is the first line, outcome is the second). If the user types nothing for outcome, skip the outcome step. Hold all decisions for step 4.

Tick handling:
- **Y** (default) — set every `- [ ]` under `## Acceptance` AND `## Test Plan` to `- [x]`. Already-ticked items stay ticked.
- **edit** — re-prompt separately so the user can tick a subset of each section:
  ```
  Acceptance tick which? (e.g. "1,3,4", "all", "none")
  Test plan tick which? (e.g. "1,2", "all", "none")
  ```
  Parse comma-separated indices per section. `all` ticks every item in that section; `none` leaves it. Invalid/out-of-range indices are ignored with a one-line warning.
- **n** — leave both sections unchanged.

If a section was skippable, treat it as if `none` was chosen for that section (no edits to it).

### 4. Apply body edits (outcome + acceptance + test plan)

Edit the body in one write that covers both changes (skip the write entirely if neither applies):

- If an outcome was provided, append under `## Notes`:
  ```
  - <YYYY-MM-DD>: [done] <outcome>
  ```
  (If `## Notes` is empty or contains only whitespace, put the bullet right under the heading.)
- If the acceptance decision was `Y` or `edit`, rewrite the `## Acceptance` block per step 3. Same for `## Test Plan`.

Use the same body-fetch / edit-in-place / `gh issue edit --body-file` pattern as `/solo:start`.

### 5. Update metadata

Set `completed: <YYYY-MM-DD>` inside the `<!-- solo:metadata ... -->` block (only if currently empty). Same edit pattern.

### 6. Flip status to done

Ensure label exists:

```bash
gh label create "status:done" --color "cccccc" --force --repo <owner/repo>
```

Remove the current `status:*` label and add `status:done`:

```bash
gh issue edit <n> --repo <owner/repo> \
  --remove-label "<current status:* if any>" \
  --add-label "status:done"
```

### 7. Close the issue

```bash
gh issue close <n> --repo <owner/repo>
```

### 8. Trunk-based merge hint

Read `trunk.name` from `.solo/config.yml` (default `main`) and the issue metadata `branch:` field.

**Guards — skip the PR offer entirely if any of these fail:**

- Not inside a git work tree (`git rev-parse --is-inside-work-tree` is not `true`).
- No `branch:` recorded in the issue metadata.
- Current local branch (`git branch --show-current`) does not match the recorded branch — the user is somewhere else, don't surprise them.
- Current branch IS the trunk (`git branch --show-current` == `trunk.name`) — there's nothing to PR.
- No `origin` remote exists (`git remote get-url origin` fails) — local-only repo, can't push.

If all guards pass, offer (don't force):

```
Branch <name> is on this issue. Open a PR to <trunk>? [Y/n]
```

On `y`:

```bash
git push -u origin <branch>
gh pr create --base <trunk> --head <branch> \
  --title "<issue title>" \
  --body "Closes #<n>"
```

If `gh pr create` fails because a PR already exists for the branch, surface the existing URL instead of erroring out:

```bash
gh pr view --json url -q .url 2>/dev/null
```

The principle: short-lived branches → merged back to trunk fast, not left dangling.

### 9. Confirm

```
🎉 Closed #<n> — <title>
```

If a PR was opened, add a second line: `🔀 PR: <url>`.
