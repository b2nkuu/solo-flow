---
description: Resume a blocked task
argument-hint: "<issue-number> [\"<resolution>\"]"
allowed-tools: [Bash]
---

# /solo:unblock

Resume a previously blocked task. Restore it to `in-progress` if it was already started, otherwise to `planned`.

## Input

`$ARGUMENTS` = `<issue-number>` and optionally `"<resolution>"`. Resolution is optional — if missing, do not prompt.

## Steps

### 1. Pre-flight + resolve repo

- `gh auth status` (stop on failure).
- Resolve repo as usual.

### 2. Read the issue

```bash
gh issue view <n> --repo <owner/repo> --json number,title,body,labels
```

- If the issue does not have `status:blocked`, print `ℹ️  #<n> is not blocked.` and stop.

### 3. Decide restore status

Parse the `<!-- solo-flow:metadata ... -->` block from the body and read the `started:` field.

- If `started:` has a value → restore to `status:in-progress`.
- Else → restore to `status:planned`.

Ensure the target label exists:

```bash
gh label create "status:in-progress" --color "2ca02c" --force --repo <owner/repo>
# or
gh label create "status:planned" --color "fbbf24" --force --repo <owner/repo>
```

Swap labels:

```bash
gh issue edit <n> --repo <owner/repo> \
  --remove-label "status:blocked" \
  --add-label "<status:in-progress | status:planned>"
```

### 4. Log an `[unblocked]` note

Post a comment:

```bash
gh issue comment <n> --repo <owner/repo> \
  --body "- <YYYY-MM-DD>: [unblocked] <resolution-or-empty>"
```

If no resolution was given, the body is `- <YYYY-MM-DD>: [unblocked]` (no trailing text).

### 5. Confirm

```
▶️ Resumed #<n>
```
