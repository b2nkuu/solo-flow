---
description: Initialize solo-flow for this repository (labels + config)
allowed-tools: [Bash]
---

# /solo:init

One-time, idempotent setup for a project: create the label taxonomy and write `.solo/config.yml`.

## Steps

### 1. Pre-flight

```bash
gh auth status
```

- If `gh` is not installed: stop with `https://cli.github.com` hint.
- If auth fails: stop with `gh auth login` hint.

### 2. Resolve / confirm repo

- If `.solo/config.yml` already exists with a real `repo:` field → confirm it: `Use repo <owner/repo>? [Y/n]`.
- Else try `gh repo view --json nameWithOwner -q .nameWithOwner`. If found → confirm.
- Else ask the user for `owner/repo`.

### 3. Create labels (idempotent)

Create the full taxonomy with `--force` so reruns are safe. Run them as a batch:

```bash
REPO=<owner/repo>

# Type
gh label create "type:feature"  --color "1f77b4" --force --repo $REPO
gh label create "type:bug"      --color "d62728" --force --repo $REPO
gh label create "type:task"     --color "2ca02c" --force --repo $REPO
gh label create "type:idea"     --color "9467bd" --force --repo $REPO
gh label create "type:research" --color "17becf" --force --repo $REPO

# Priority
gh label create "priority:high"   --color "d62728" --force --repo $REPO
gh label create "priority:medium" --color "ff9900" --force --repo $REPO
gh label create "priority:low"    --color "9e9e9e" --force --repo $REPO

# Status
gh label create "status:inbox"       --color "ededed" --force --repo $REPO
gh label create "status:planned"     --color "fbbf24" --force --repo $REPO
gh label create "status:in-progress" --color "2ca02c" --force --repo $REPO
gh label create "status:blocked"     --color "8b0000" --force --repo $REPO
gh label create "status:done"        --color "cccccc" --force --repo $REPO

# Size
gh label create "size:xs" --color "c4b5fd" --force --repo $REPO
gh label create "size:s"  --color "a78bfa" --force --repo $REPO
gh label create "size:m"  --color "8b5cf6" --force --repo $REPO
gh label create "size:l"  --color "7c3aed" --force --repo $REPO
gh label create "size:xl" --color "6d28d9" --force --repo $REPO
```

### 4. Write `.solo/config.yml`

If the file does not exist, create it. If it does, **do not overwrite** — print `ℹ️  .solo/config.yml already exists, keeping it.` instead.

Detect the trunk branch name (default `main`, fallback `master` if `main` doesn't exist locally):

```bash
git symbolic-ref refs/remotes/origin/HEAD 2>/dev/null | sed 's|refs/remotes/origin/||'
# fallback if not in a git repo: "main"
```

Use the detected name as `trunk.name`.

```yaml
# solo-flow configuration
version: 1

repo: "<owner/repo>"

defaults:
  inbox_label: "status:inbox"
  capture_type_guessing: true

trunk:
  name: "<detected-or-main>"
  max_branch_age_days: 2          # /solo:today warns when an in-progress branch is older

branch:
  enabled: true
  pattern: "{type}/{issue}-{slug}"
  branch_from_trunk: true          # always branch from trunk, not current

note:
  storage: "comment"
  decision_prefix: "[decision]"

display:
  today_suggested_limit: 5
  date_format: "%Y-%m-%d"
```

Create `.solo/` directory if missing.

### 5. Optional milestones

Ask once: `Create initial milestones? (comma-separated names, or enter to skip):`

For each name, run:

```bash
gh api repos/<owner/repo>/milestones -f title="<name>" 2>/dev/null || true
```

(Errors swallowed — duplicates are not fatal.)

### 6. Confirm

```
✅ solo-flow initialized for <owner/repo>
   labels: 18 created/updated
   config: .solo/config.yml
   [milestones: <list>]
```

Drop the milestones line if none were created.
