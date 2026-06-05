---
description: Walk through an issue's Test Plan, run or verify each item, tick what passes
argument-hint: "<issue-number>"
allowed-tools: [Bash]
---

# /solo:test

Walk through every item under `## Test Plan` on an issue, decide per item (run a command, manual verify, fail, or skip), and tick what passed back into the body.

Use this after the work is done but before `/solo:done` — it converts a list of intentions into a checked record of what was actually verified.

## Input

`$ARGUMENTS` = issue number (no `#` prefix needed; tolerate it if given).

## Steps

### 1. Pre-flight + resolve repo

- `gh auth status` (stop on failure).
- Resolve repo as usual (`.solo/config.yml` → `gh repo view` → ask).

### 2. Read the issue

```bash
gh issue view <n> --repo <owner/repo> --json number,title,body,labels,state
```

- If the issue is already closed → friendly note `ℹ️  #<n> is closed.` and continue (user may be re-verifying).
- Parse `## Test Plan`. If it is missing or contains only the empty `- [ ]` placeholder, stop with:
  ```
  ❌ #<n> has no test plan. Run /solo:plan to generate one, or edit the issue body to add items.
  ```

### 3. Show the plan

List every item with an index and current tick state:

```
🧪 #<n> <title> — Test Plan (<N> items)
   [1] [<x or space>] <item 1>
   [2] [<x or space>] <item 2>
   …
```

### 4. Walk items one at a time

For each item in order, do steps 4a–4c.

#### 4a. Suggest how to verify

Read the item text together with the issue title, `## What`, and `type:*` label. Output a single concrete suggestion:

- If the item names a concrete code path or behavior → propose a shell command (test runner, lint, type check, curl, etc.) that exercises it. Examples:
  - `Unit: serializer outputs correct columns` → `bun test src/serializer.test.ts` (use the project's actual test runner — read `package.json` / `Cargo.toml` / `pyproject.toml` if needed).
  - `Integration: hit endpoint with/without auth` → propose `curl` calls or the integration test file.
  - `Edge: empty users table → header only` → propose a test pattern flag or a `psql` snippet.
- If the item is inherently manual (UI clickthrough, observability check, design review) → output `(manual — verify yourself)` and skip the run option in the prompt.
- If you genuinely cannot infer a verification → output `(no suggestion — choose manual or skip)`.

Never invent file paths or commands that depend on tools the project clearly does not use.

#### 4b. Prompt

```
[<i>/<N>] <item text>
   suggested: <command or "(manual — verify yourself)">
   [r]un / [m]anual / [f]ail / [s]kip
```

Omit `r` from the choices when there is no command to run.

- **r** — execute the suggested command via the Bash tool. Stream the result. Treat exit code `0` as pass, anything else as fail. Capture the last ~10 lines of combined stdout/stderr for the summary.
- **m** — the user confirms the check passed manually (no command run). Mark pass.
- **f** — the user confirms it failed. Prompt once: `Failure note (enter to skip):`. Mark fail.
- **s** — leave the item exactly as it was; do not change its tick state.

If the user provides a different command (e.g. they paste `bun test --watch ...` instead of `r`), execute that command and treat the result the same as `r`.

#### 4c. Record the per-item result

In memory, hold a tuple `(index, decision, note?)` for each item. Do not write to GitHub yet — batch the body edit at the end so a mid-walk abort leaves the issue clean.

### 5. Apply ticks

Fetch the body, rewrite the `## Test Plan` block in place:

- For every item the walk marked `pass` (whether by `r` exit 0 or `m`), change `- [ ] ` to `- [x] `.
- For `fail`, force the item to `- [ ] ` (untick it even if it was previously ticked — failing now is the truth).
- For `skip`, leave the line unchanged.

Use the same body-fetch / edit-in-place / `gh issue edit --body-file` pattern as `/solo:start` and `/solo:done`.

### 6. Record a Notes entry

Append a single line under `## Notes` (one section edit, batched with step 5):

```
- <YYYY-MM-DD>: [test] <P> pass / <F> fail / <S> skip
```

If `<F> > 0`, also append a sub-line per failure with the user's note (or `(no note)`):

```
- <YYYY-MM-DD>: [test] 3 pass / 1 fail / 0 skip
  - failed [2]: <failure note>
```

### 7. Summary

```
🧪 #<n>: <P> pass / <F> fail / <S> skip
```

If `<F> > 0`, add a second line nudging next steps:

```
   ↳ /solo:note <n> "[blocked] <reason>" if you're stuck, or fix and rerun /solo:test <n>.
```

If `<F> == 0` and every item is now ticked, add:

```
✅ Test Plan complete — ready for /solo:done <n>.
```

## Design constraints

- **Never auto-run without `r`.** The user picks `r` explicitly per item; you do not chain runs across items.
- **Never tick fail items.** A failed step is the truth — surface it.
- **Batch the body write.** All ticks + the Notes line go in one `gh issue edit --body-file` call after the walk.
- **Read-only on issue body until the end.** This makes `Ctrl-C` mid-walk safe.
- **Don't change `status:*` labels.** This command verifies; it does not advance the workflow. Use `/solo:done` to close.
