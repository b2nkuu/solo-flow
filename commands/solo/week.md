---
description: Summarize the past week's completed work and decisions
allowed-tools: [Bash]
---

# /solo:week

Summarize the past 7 days as a reflection tool. Readable, not a report generator.

## Steps

### 1. Pre-flight + resolve repo

- `gh auth status` (stop on failure).
- Resolve repo as usual.

### 2. Compute date window

```bash
SINCE=$(date -v-7d +%Y-%m-%d)   # macOS
# Linux fallback: SINCE=$(date -d '7 days ago' +%Y-%m-%d)
TODAY=$(date +%Y-%m-%d)
```

### 3. Fetch closed-this-week

```bash
gh issue list --repo <owner/repo> --state closed \
  --search "closed:>=$SINCE" --limit 200 \
  --json number,title,labels,closedAt,body
```

### 4. Fetch in-progress carryover

```bash
gh issue list --repo <owner/repo> --state open \
  --label "status:in-progress" --limit 100 \
  --json number,title,labels
```

### 5. Extract decisions

For each closed issue's `body`, scan `## Notes` for lines matching `- <date>: [decision] <text>` where date is within the window. Collect them.

(Optionally also scan comments via `gh issue view <n> --json comments` — but only if cheap; skip if more than ~30 closed issues to avoid rate-limit noise.)

### 6. Render

```
🗓  Week of <SINCE> → <TODAY>

Completed (<total>):
  feature (<count>):
    #<n> <title>
    …
  bug (<count>):
    #<n> <title>
    …
  task / research / idea (<count>): …

In Progress carryover (<count>):
  #<n> <title>
  …

Decisions (<count>):
  - <date> #<n>: <decision text>
  …
```

Omit any section that is empty. If everything is empty, print:

```
🗓  Week of <SINCE> → <TODAY>
   Nothing closed this week.
```
