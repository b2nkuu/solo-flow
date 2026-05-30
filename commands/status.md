---
description: Show overall project status
allowed-tools: [Bash]
---

# /solo:status

Snapshot of the whole project — counts by status and type, plus what needs attention.

## Steps

### 1. Pre-flight + resolve repo

- `gh auth status` (stop on failure).
- Resolve repo as usual.

### 2. Fetch open issues

```bash
gh issue list --repo <owner/repo> --state open --limit 500 \
  --json number,title,labels
```

### 3. Fetch closed-last-7d count

```bash
SINCE=$(date -v-7d +%Y-%m-%d)
gh issue list --repo <owner/repo> --state closed \
  --search "closed:>=$SINCE" --limit 200 \
  --json number
```

### 4. Bucket open issues

For each open issue, find its `status:*` label and `type:*` label and increment counters.

### 5. Render

```
📊 <owner/repo>

Open: <total>
  status:inbox        <n>
  status:planned      <n>
  status:in-progress  <n>
  status:blocked      <n>

By type:
  feature   <n>
  bug       <n>
  task      <n>
  research  <n>
  idea      <n>

Closed last 7d: <n>
```

### 6. Attention warnings

After the snapshot, surface anything noteworthy (each as a single line, only if true):

- `⚠ Inbox is large (<n>) — try /solo:plan` if inbox > 10.
- `⚠ <n> tasks blocked` if blocked > 0 — also hint `/solo:today` to see them.
- `⚠ No tasks in progress` if in-progress is 0 and planned > 0.
- `⚠ <n> stale in-progress (>= <max> days)` — count in-progress issues whose metadata `started:` is older than `trunk.max_branch_age_days` (default `2`). Skip if `max_branch_age_days` is `0`. This is the trunk-based-development health signal.

Omit the warnings block entirely if nothing applies.
