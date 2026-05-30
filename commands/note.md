---
description: Append a timestamped note to a task
argument-hint: "<issue-number> \"<note text>\""
allowed-tools: [Bash]
---

# /solo:note

Log a timestamped note on an issue without opening the browser.

## Input

`$ARGUMENTS` = `<issue-number> "<text>"`. Parse the leading integer; the rest is the note text (strip surrounding quotes).

## Steps

### 1. Pre-flight + resolve repo

- `gh auth status` (stop on failure).
- Resolve repo as usual.

### 2. Default storage = comment

Post the note as an issue comment:

```bash
gh issue comment <n> --repo <owner/repo> \
  --body "- <YYYY-MM-DD>: <text>"
```

The `<text>` is preserved verbatim. If it starts with `[decision]`, keep that prefix intact.

### 3. Mirror decisions into body Notes

If the text starts with `[decision]`, ALSO append the same line to the issue body's `## Notes` section so structured decisions survive in the parseable body:

1. Fetch body: `gh issue view <n> --repo <owner/repo> --json body -q .body > /tmp/solo-body`
2. Append a bullet under `## Notes`:
   - If `## Notes` already has content, add the new bullet at the end of that section (immediately before the next `---` or `<!-- solo:metadata` line, whichever comes first).
   - Format: `- <YYYY-MM-DD>: [decision] <rest of text>`
3. `gh issue edit <n> --repo <owner/repo> --body-file /tmp/solo-body`

For non-decision notes, **do not** edit the body — comment-only is sufficient.

### 4. Confirm

```
✅ Added note to #<n>
```

## Implementation note

Body-edit fragility (regex matching `## Notes` section) is the main risk here. If the body cannot be parsed safely (no `## Notes` heading, or no `<!-- solo:metadata` block), fall back to comment-only and warn the user once:

```
✅ Added comment to #<n> (body mirror skipped — issue not in solo format)
```
