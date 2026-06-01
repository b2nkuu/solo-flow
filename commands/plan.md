---
description: Review inbox and plan tasks (set priority, size, status)
allowed-tools: [Bash]
---

# /solo:plan

Turn raw captures from the Inbox into planned work. Be efficient — batch the decisions.

Also handles milestone management (`/solo:plan milestone …`) and backfill of legacy issues missing a milestone.

## Sub-commands

If `$ARGUMENTS` starts with `milestone`, jump to the **Milestone sub-command** section. Otherwise run the default Inbox flow below.

## Steps

### 1. Pre-flight + resolve repo

- `gh auth status` (stop on failure).
- Resolve repo as usual.
- Read `.solo/config.yml`: `milestone.current` (string) and `milestone.required` (default `false`).

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
- Milestone: `milestone:<name>` (optional, per item). The milestone must already exist — do not auto-create. Use `/solo:plan milestone create <name>` to make a new one.

**Milestone default:** if the user omits `milestone:<name>` for an item, attach `milestone.current` (if set and still open). Items with no milestone after this default are allowed only when `milestone.required: false`; otherwise re-ask for the missing items.

**Trunk-based sizing nudge:** if the user assigns `size:xl` to any item, warn before confirming:

```
⚠ #<n> is size:xl — too large for a short-lived branch.
   Consider breaking it into smaller issues (size ≤ l) so each can ship in ≤ 2 days.
   Plan it anyway? [y/N]
```

`xl` is allowed (it's just a flag for "needs breakdown"), but solo follows trunk-based development — long-lived branches are an anti-pattern.

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

### 7. Backfill legacy issues (offered only when `milestone.required: true`)

After the inbox pass, check for open issues with no milestone (any status). If any exist:

```bash
gh issue list --repo <owner/repo> --state open --limit 200 \
  --search "no:milestone" --json number,title,labels
```

Present them and offer a batch assignment:

```
🧭 <count> open issues have no milestone:
  [1] #<n> [<status>] <title>
  …
Assign milestone? (name, "all:<name>", "skip", or enter to defer)
```

On a name (e.g. `v0.4`) or `all:<name>`, apply via `gh issue edit <n> --milestone "<name>"`. Verify the milestone exists and is open first — if not, prompt to create it (same flow as the sub-command below).

Skip this step entirely when `milestone.required: false`.

## Milestone sub-command

Invoked as `/solo:plan milestone <action> [args]`. Actions:

### `list`

```bash
gh api "repos/<owner/repo>/milestones?state=open" \
  --jq '.[] | "\(.number)\t\(.title)\t\(.open_issues)/\(.open_issues + .closed_issues)"'
```

Render:

```
📦 Milestones (open):
  #<num> <title>  (<closed>/<total> done)  [current]
  …
```

Mark the milestone matching `milestone.current` with `[current]`.

### `create <name>`

```bash
gh api repos/<owner/repo>/milestones -f title="<name>"
```

Then ask `Set as current? [Y/n]`. On `y` (default), update `.solo/config.yml` `milestone.current:` in place.

If the API returns "already_exists", surface that and offer to just set it as current.

### `current [<name>]`

- No arg → print `milestone.current` from config.
- With a name → verify it is an open milestone, then update `.solo/config.yml` `milestone.current:` in place.

### `close <name>`

```bash
MS_NUMBER=$(gh api "repos/<owner/repo>/milestones?state=open" \
  --jq '.[] | select(.title=="<name>") | .number')
gh api -X PATCH "repos/<owner/repo>/milestones/$MS_NUMBER" -f state=closed
```

If `<name>` was `milestone.current`, clear it in config (set to empty string) and remind the user that new captures need a current milestone (block if `milestone.required: true`). Suggest running `/solo:release` instead if they wanted to cut a release.
