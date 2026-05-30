---
description: Review inbox and plan tasks (set priority, size, status)
allowed-tools: [Bash]
---

# /solo:plan

Turn raw captures from the Inbox into planned work. Be efficient — batch the decisions.

## Steps

### 1. Pre-flight + resolve repo

- `gh auth status` (stop on failure).
- Resolve repo as usual.

### 2. Fetch inbox

```bash
gh issue list --repo <owner/repo> --state open \
  --label "status:inbox" --limit 100 \
  --json number,title,labels
```

If empty: print `📭 Inbox is empty.` and stop.

### 3. Present the batch

Render a numbered list of inbox items with their current type:

```
📋 Inbox (<count>):
  [1] #<n> [<type>] <title>
  [2] #<n> [<type>] <title>
  …
```

Then ask the user once for a batch decision. Accept either:

- **One-line shorthand per item:** `1: high m` (priority size), `2: low xs milestone:v1`, `3: skip` (leave in inbox).
- **Bulk:** `all: med m` to apply the same priority/size to every item.

Allowed values:

- Priority: `high`, `med` (alias `medium`), `low`
- Size: `xs`, `s`, `m`, `l`, `xl`
- Milestone: `milestone:<name>` (optional, per item). The milestone must already exist — do not auto-create.

**Trunk-based sizing nudge:** if the user assigns `size:xl` to any item, warn before confirming:

```
⚠ #<n> is size:xl — too large for a short-lived branch.
   Consider breaking it into smaller issues (size ≤ l) so each can ship in ≤ 2 days.
   Plan it anyway? [y/N]
```

`xl` is allowed (it's just a flag for "needs breakdown"), but solo-flow follows trunk-based development — long-lived branches are an anti-pattern.

### 4. Summarize and confirm

Before applying any changes, print:

```
About to apply:
  #<n>  +priority:high  +size:m   inbox → planned
  #<n>  +priority:low   +size:xs  +milestone:v1   inbox → planned
  #<n>  (skipped)
Confirm? [y/N]
```

Only `y` proceeds. Anything else aborts with no changes.

### 5. Apply

For each non-skipped item:

1. Ensure each needed label exists:
   ```bash
   gh label create "priority:high"   --color "d62728" --force --repo <owner/repo>
   gh label create "priority:medium" --color "ff9900" --force --repo <owner/repo>
   gh label create "priority:low"    --color "9e9e9e" --force --repo <owner/repo>
   gh label create "size:xs" --color "c4b5fd" --force --repo <owner/repo>
   gh label create "size:s"  --color "a78bfa" --force --repo <owner/repo>
   gh label create "size:m"  --color "8b5cf6" --force --repo <owner/repo>
   gh label create "size:l"  --color "7c3aed" --force --repo <owner/repo>
   gh label create "size:xl" --color "6d28d9" --force --repo <owner/repo>
   gh label create "status:planned" --color "fbbf24" --force --repo <owner/repo>
   ```
   (Only create the ones you actually need; the `--force` keeps it idempotent.)
2. Apply labels + status swap in one edit:
   ```bash
   gh issue edit <n> --repo <owner/repo> \
     --add-label "priority:<x>,size:<y>,status:planned" \
     --remove-label "status:inbox" \
     [--milestone "<name>"]
   ```

### 6. Confirm

```
✅ Planned <count> items.
```
