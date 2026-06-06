---
description: Start a task — set in-progress and optionally create a branch (or run all planned issues through a Workflow)
argument-hint: "<issue-number> [--force] | workflow"
allowed-tools: [Bash, Workflow]
---

# /solo:start

Two shapes:

- `/solo:start <n>` — single-issue: flip status, create a branch, show goalposts. Steps 1–7 below.
- `/solo:start workflow` — batch: pick every `status:planned` issue in the current milestone and run them through a Claude Code Workflow (plan → implement in isolated worktrees → verify against Test Plan). Step 8 below.

## Input

`$ARGUMENTS` is one of:

- A single literal token `workflow` → batch mode (jump to step 8).
- An issue number (tolerate a leading `#`) optionally followed by flags:
  - `--force` — bypass the `size:xl` gate (see step 5).
  - Anything else → warn `⚠ unknown flag: <token> (ignored)` and continue.

Empty `$ARGUMENTS` → stop with `Usage: /solo:start <issue-number> | workflow`.

## Steps

### 1. Pre-flight + resolve repo

- `gh auth status` (stop on failure).
- Resolve repo as usual (`.solo/config.yml` → `gh repo view` → ask).

### 2. Read the issue

```bash
gh issue view <n> --repo <owner/repo> --json number,title,body,labels,state
```

- If the issue is already closed → stop with `❌ #<n> is closed.`
- If it already has `status:in-progress` → friendly no-op:
  ```
  ℹ️  #<n> already in progress.
  ```
  Skip status change but continue to branch creation (so the user can re-create a branch if needed).

### 3. Flip status to in-progress

Remove whichever `status:*` label is currently set (only one should be present per spec §4.4) and add `status:in-progress`. Ensure the label exists first (`gh label create "status:in-progress" --color "2ca02c" --force --repo <owner/repo>`).

```bash
gh issue edit <n> --repo <owner/repo> \
  --remove-label "<current status:* if any>" \
  --add-label "status:in-progress"
```

### 4. Update metadata

Fetch the body, find the `<!-- solo:metadata ... -->` comment, and set `started: <today>` (only overwrite if the field is empty). Write back:

```bash
gh issue view <n> --repo <owner/repo> --json body -q .body > /tmp/solo-body
# edit /tmp/solo-body in-place: replace ^started:.*$ with started: <YYYY-MM-DD>
gh issue edit <n> --repo <owner/repo> --body-file /tmp/solo-body
```

Preserve the rest of the body verbatim.

### 5. Branch creation (optional, trunk-based)

solo follows [trunk-based development](https://trunkbaseddevelopment.com/): branches are **short-lived** (target ≤ 2 days), scoped small, and merged back to `trunk` fast.

**Size gate:** if the issue is labeled `size:xl`, stop and warn:

```
⚠ #<n> is size:xl — too large for a short-lived branch.
   Break it down with /solo:plan first, or override with --force.
```

Only continue past the gate if the user passes `--force` in `$ARGUMENTS` or explicitly confirms.

Read TBD config from `.solo/config.yml`:

- `branch.enabled` (default `true`)
- `branch.pattern` (default `{type}/{issue}-{slug}`)
- `trunk.name` (default `main`)
- `trunk.max_branch_age_days` (default `2`)

Pre-branch hygiene:

- `git rev-parse --is-inside-work-tree` must return `true` — else skip branch creation, print `(not a git repo — skipped branch)`.
- Branch **from** `trunk.name`, not from whatever the current branch is. First: `git fetch origin <trunk>` then `git switch <trunk> && git pull --ff-only`. If that fails (uncommitted changes, divergence), ask the user before continuing.
- After trunk is up to date, check `git status --porcelain`:
  - Empty → create the branch automatically.
  - Non-empty → ask: `Working tree is dirty. Create branch anyway? [y/N]`. Only proceed on `y`.

Build the branch name:

- `{type}` = the type stripped of `type:` prefix (e.g. `feature`).
- `{issue}` = the issue number.
- `{slug}` = the title lowercased, non-alphanumeric → `-`, collapse repeats, trimmed to ~40 chars.

```bash
git switch -c <branch-name>
```

Record the branch name in metadata `branch:` field (same body-edit flow as step 4).

### 6. Confirm

```
✅ Started #<n> — <title>
🌿 Branch: <branch-name>
```

If branch was skipped, drop the second line and add `(no branch created)`.

### 7. Show goalposts

Parse the issue body for the `## Acceptance` and `## Test Plan` sections. Print them as-is to set context for the work — checked items keep their `[x]`, unchecked stay `[ ]`. Skip a section entirely if it is missing or contains only the empty `- [ ]` placeholder. If both are absent, skip the whole step (no blank output).

```
📋 Acceptance:
   - [ ] <item 1>
   - [ ] <item 2>
   …

🧪 Test Plan:
   - [ ] <item 1>
   - [ ] <item 2>
   …
```

Read-only — does not modify the issue.

### 8. Batch workflow mode (`/solo:start workflow`)

Reached only when `$ARGUMENTS` is the single token `workflow`. Steps 1–7 above do **not** run in this mode — there is no single issue to flip or branch from. The whole flow lives below.

#### 8a. Pre-flight + resolve repo

Same as step 1 (`gh auth status`, resolve repo from `.solo/config.yml`). Also read:

- `milestone.current` (optional)
- `trunk.name` (default `main`)
- `workflow.max_parallel` (default `4`)
- `workflow.max_retries` (default `3`)

#### 8b. Build the source list

Fetch open issues once:

```bash
gh issue list --repo <owner/repo> --state open --limit 200 \
  --json number,title,body,labels,milestone
```

Filter to issues that carry `status:planned`. Then:

- If `milestone.current` is set → keep only issues whose `milestone.title == milestone.current`.
- Otherwise → keep all planned issues.

#### 8c. Refusals (before any mutation)

- Empty source list → stop with:
  ```
  No planned issues to run.
  ```
- Any issue carries `size:xl` → list them and stop:
  ```
  ⚠ size:xl in batch:
     #<n> <title>
     …
     Break them down with /solo:plan first.
  ```
- Any issue's body is missing a non-empty `## Acceptance` or `## Test Plan` section → list them and stop:
  ```
  ⚠ Missing AC or Test Plan:
     #<n> <title> (no AC)
     #<m> <title> (no Test Plan)
     …
     Run /solo:plan to fill them in.
  ```

Refusals abort the whole batch — no partial flips, no partial branches.

#### 8d. Confirm

Show the batch and ask once:

```
🤖 Workflow batch — <N> issue(s):
   #<n1> [<priority>][<size>] <title>
   #<n2> [<priority>][<size>] <title>
   …
   parallel: <min(N, workflow.max_parallel)>   retries: <workflow.max_retries>
   milestone filter: <milestone.current or "none">
Start? [y/N]
```

Anything other than `y` (case-insensitive, accept `yes`/`ครับ`/`ใช่`) → abort.

#### 8e. Sync trunk

Once for the whole batch:

```bash
git fetch origin <trunk>
git switch <trunk>
git pull --ff-only
```

Fail-fast if trunk can't be brought clean — ask the user before any worktree is created.

#### 8f. Invoke the Workflow tool

One Workflow call. Pass the source list, repo, trunk name, and config knobs via `args`. The script runs **one pipeline per issue, all in parallel** (capped at `workflow.max_parallel`). Each pipeline does:

1. **Claim** — flip the issue's status `planned` → `in-progress` and stamp `started: <today>` in metadata. This is the atomic flip — a workflow killed before claim leaves the issue as `planned`; after claim leaves it as `in-progress`. Build the branch name (`{type}/{issue}-{slug}`) and create an isolated worktree off `trunk` (use the agent's `isolation: 'worktree'` option for the implement agent — solo's per-issue parallelism is the canonical case for it).
2. **Plan agent** — reads `## What` + `## Acceptance` + `## Test Plan` for this issue, returns `{ subtasks: [{ id, summary, covers: [<acceptance-index>...] }] }` via schema. Reject + retry once if any Acceptance item is uncovered; fail this issue's pipeline on second miss.
3. **Implement agent** — runs in the worktree (isolation guarantees no cross-issue file fights). Works the subtasks serially, commits per subtask. Returns the worktree path + commit list.
4. **Verify agent** — walks the `## Test Plan` in the spirit of `/solo:test`: per item, propose a check, run it (Bash inside the worktree), return `{ index, passed, evidence }`. Then:
   - All passed → tick `- [ ]` → `- [x]` for every Test Plan item in the issue body, append `- <date>: [workflow] all <N> test items passed` to `## Notes`, leave `status:in-progress`. Pipeline returns success with the branch name and worktree path so the user can switch into it.
   - Any failed → loop back to Implement with only the failing items as the next subtask list. Cap at `workflow.max_retries`. On cap, return failure with the unfinished items + last evidence.

Issues are independent — one pipeline failing does **not** kill the others. Use `parallel()` (or `pipeline()` driven by the source array) at the script top so each issue runs end-to-end as soon as a slot frees up.

#### 8g. Summary

After the Workflow returns, print a per-issue summary:

```
🤖 Workflow batch finished — <P> green / <F> red

Green:
   #<n> <title>  →  <branch>  (worktree: <path>)
   …

Red:
   #<n> <title>  →  <branch>  (worktree: <path>)
      ✗ test [2] <item>: <one-line evidence>
   …

Next: switch into a green branch and /solo:done <n>, or fix a red and re-run /solo:start workflow.
```

Re-running `/solo:start workflow` after partial failure is safe: claimed issues are now `status:in-progress`, so they fall out of the source filter and won't be re-claimed. To retry a red issue, manually flip it back to `status:planned` (or fix on the existing worktree and `/solo:test <n>` directly).

Config knobs (read from `.solo/config.yml`, all optional):

```yaml
workflow:
  max_parallel: 4           # cap concurrent pipelines (per-issue)
  max_retries: 3            # implement→verify loop cap per issue
  plan_model: sonnet        # override planning agent model
  implement_model: sonnet   # override implementation agent model
  verify_model: haiku       # override verify agent model
```
