---
description: Show today's focus — in-progress, suggested, and blocked tasks
allowed-tools: [Bash]
---

# /solo:today

Show a focused, scannable list of what to work on today.

## Steps

### 1. Pre-flight + resolve repo

- Run `gh auth status`. On failure: stop, tell user to run `gh auth login`.
- Resolve repo: `.solo/config.yml` `repo:` field → fallback `gh repo view --json nameWithOwner -q .nameWithOwner` → ask user.
- Read `display.today_suggested_limit` from `.solo/config.yml` (default `5`).

### 2. Fetch open issues

One call, then group in memory:

```bash
gh issue list --repo <owner/repo> --state open --limit 200 \
  --json number,title,labels,body
```

### 3. Group by status

For each issue, look at its `labels[].name` array and bucket by which `status:*` label it carries:

- `status:in-progress` → **In Progress**
- `status:planned` → **Suggested Next**
- `status:blocked` → **Blocked**
- (Issues with `status:inbox` or no status label are excluded from `/solo:today`.)

### 4. Order Suggested Next

Sort by priority then size:

- Priority order: `priority:high` > `priority:medium` > `priority:low` > (no priority label)
- Size order: `size:xs` < `size:s` < `size:m` < `size:l` < `size:xl` < (no size label, sorted last)

Show only the top N (`display.today_suggested_limit`, default 5).

### 5. Extract blocked reasons

For each Blocked issue, scan the `body` for the most recent `[blocked]` note under `## Notes` (format: `- <date>: [blocked] <reason>`). If none in body, fetch comments and find the latest `[blocked]` line:

```bash
gh issue view <n> --repo <owner/repo> --json comments -q '.comments[].body'
```

Use the reason text (everything after `[blocked] `).

### 6. Render

Use this exact shape (omit a group entirely if empty; if all three are empty, print `No active tasks. Try /solo:plan or /solo:capture.`):

```
📅 Today (<YYYY-MM-DD>)

In Progress (<count>):
  #<n> [<priority>][<size>] <title>
  …

Suggested Next (<count>):
  #<n> [<priority>][<size>] <title>
  …

Blocked (<count>):
  #<n> ⏸ <reason>
  …
```

For the `[priority]` and `[size]` brackets in-line, use short forms: `high`/`med`/`low` and `xs`/`s`/`m`/`l`/`xl`. If a label is missing, render `-` (e.g. `[-][m]`).

Keep output compact — no extra blank lines, no headers other than the ones above.

### 7. Stale-branch warning (trunk-based)

After rendering, for each **In Progress** issue parse the `started:` field from its `<!-- solo-flow:metadata -->` block. If `started` is more than `trunk.max_branch_age_days` (default `2`) days ago, append a one-line warning under the In Progress group:

```
  ⚠ #<n> in progress for <X>d — ship or break it down
```

This is the trunk-based-development nudge: short-lived branches only. Don't show the warning if `trunk.max_branch_age_days` is set to `0` (disabled).
