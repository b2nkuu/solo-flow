---
description: Mark a task blocked with a reason
argument-hint: "<issue-number> \"<reason>\""
allowed-tools: [Bash]
---

# /solo:block

Mark a task blocked and log the reason.

## Input

`$ARGUMENTS` = `<issue-number> "<reason>"`. Parse the leading integer; the rest (strip quotes) is the reason. If no reason is supplied, ask once: `Reason for blocking?`.

## Steps

### 1. Pre-flight + resolve repo

- `gh auth status` (stop on failure).
- Resolve repo as usual.

### 2. Flip status to blocked

Ensure label exists:

```bash
gh label create "status:blocked" --color "8b0000" --force --repo <owner/repo>
```

Find the current `status:*` label on the issue (`gh issue view <n> --json labels`) and swap it:

```bash
gh issue edit <n> --repo <owner/repo> \
  --remove-label "<current status:*>" \
  --add-label "status:blocked"
```

### 3. Log the reason as a `[blocked]` note

Post a comment (same pattern as `/solo:note`):

```bash
gh issue comment <n> --repo <owner/repo> \
  --body "- <YYYY-MM-DD>: [blocked] <reason>"
```

This is **not** a `[decision]` note, so body mirroring is not required.

### 4. Confirm

```
⏸ Blocked #<n> — <reason>
```
