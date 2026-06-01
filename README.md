# solo

Lightweight task management for solopreneurs using GitHub Issues, from inside Claude Code.

Capture an idea, work it, ship it, reflect on the week — all without opening the GitHub web UI.

## Why

You already use GitHub. You're working alone. You don't need a Jira clone — you need to add a task in 5 seconds, know what to do next in 5 seconds, and close it in 5 seconds. solo keeps the data in Issues (single source of truth) and gives you a small set of slash commands that do exactly that.

## Prerequisites

- [Claude Code](https://docs.claude.com/en/docs/claude-code)
- [`gh` CLI](https://cli.github.com) installed and authenticated:

  ```bash
  gh auth status   # must pass
  # if not:
  gh auth login
  ```

- A GitHub repository for the project.

## Install

Clone or symlink this directory and load it as a Claude Code plugin. Once installed, `/solo:*` commands appear in the slash menu.

## Quickstart

```bash
cd path/to/your-repo
```

In Claude Code:

```
/solo:init                                   # one-time: create labels + .solo/config.yml
/solo:capture "fix login redirect bug"       # captures to Inbox as type:bug
/solo:today                                  # see in-progress, suggested, blocked
/solo:start 42                               # flip to in-progress + create branch
/solo:note 42 "[decision] use JWT for mobile"
/solo:done 42                                # close + record outcome
```

You don't need `/solo:init` to start capturing — `/solo:capture` works out of the box and creates any missing labels on demand.

## Commands

| Command | Purpose | Args |
|---|---|---|
| `/solo:capture` | Capture a task or idea to the Inbox | `"text"` |
| `/solo:today` | Show today's focus list | — |
| `/solo:start` | Mark in-progress + create branch | `<issue#>` |
| `/solo:done` | Record outcome + close | `<issue#>` |
| `/solo:note` | Append a timestamped note | `<issue#> "text"` |
| `/solo:block` | Mark blocked with a reason | `<issue#> "reason"` |
| `/solo:unblock` | Resume a blocked task | `<issue#> ["resolution"]` |
| `/solo:plan` | Triage the Inbox into planned work | — |
| `/solo:plan milestone` | Manage milestones (list, create, current, close) | `<action> [name]` |
| `/solo:release` | Tag from trunk, generate notes, close milestone | `[--dry-run]` |
| `/solo:week` | Past 7 days summary | — |
| `/solo:status` | Project snapshot | — |
| `/solo:init` | Idempotent setup (labels + config) | — |

## How it models work

- **State** is stored as GitHub Issue labels — no separate database.
- **Status** is mutually exclusive via `status:inbox`, `status:planned`, `status:in-progress`, `status:blocked`, `status:done` (the last one also closes the issue).
- **Type** is one of `type:feature`, `type:bug`, `type:task`, `type:idea`, `type:research`.
- **Priority** is `priority:high|medium|low`. **Size** is `size:xs|s|m|l|xl`. Set during `/solo:plan`.
- **Decisions** are notes prefixed `[decision]` — posted as comments and mirrored into the issue body's `## Notes` section so they stay parseable.

## Configuration

`.solo/config.yml` is created by `/solo:init`:

```yaml
version: 1
repo: "owner/repo"

defaults:
  inbox_label: "status:inbox"
  capture_type_guessing: true

branch:
  enabled: true
  pattern: "{type}/{issue}-{slug}"

note:
  storage: "comment"
  decision_prefix: "[decision]"

display:
  today_suggested_limit: 5
  date_format: "%Y-%m-%d"
```

No secrets — auth is entirely via `gh`.

## Natural-language mode

A bundled skill (`solo-assistant`) lets Claude proactively suggest the right `/solo:*` command when you talk about tasks in plain English ("I should add dark mode", "what's next", "done with #45"). It always confirms before mutating GitHub state.

## Trunk-based development

solo assumes [trunk-based development](https://trunkbaseddevelopment.com/) — a single long-lived `trunk` branch (`main` by default) and short-lived, small feature branches that merge back fast. The plugin bakes the principles in:

- **Branch from trunk, always.** `/solo:start` updates `trunk` and branches off it — never off another feature branch.
- **Short-lived branches.** Target ≤ 2 days. `/solo:today` warns when an in-progress task is older than `trunk.max_branch_age_days` (configurable).
- **Small scope.** `/solo:plan` and `/solo:start` flag `size:xl` and suggest breakdown before work begins.
- **Ship fast.** `/solo:done` offers to push the branch and open a PR back to trunk in one step.
- **One issue → one branch → one PR.** No bundling of unrelated work.
- **Feature flags > long branches.** For multi-week work, gate behind a flag and keep merging to trunk.

Configure the trunk name and branch-age threshold in `.solo/config.yml`:

```yaml
trunk:
  name: "main"
  max_branch_age_days: 2
```

## Releases & milestones

solo treats a release as a **snapshot of trunk** — a git tag plus a GitHub Release. No release branches. Hotfixes are new commits on trunk with a new patch tag. Multi-week work that can't ship goes behind a feature flag, not a long-lived branch.

**Milestones group issues by intended release.** GitHub Milestones are the source of truth — solo just keeps `milestone.current` in `.solo/config.yml` pointing at the active one, so `/solo:capture` can attach it automatically.

### Flow

```
/solo:plan milestone create "v0.4"     # open a new milestone, set as current
/solo:capture "add dark mode"          # → attached to v0.4
…
/solo:release --dry-run                # preview the tag + notes
/solo:release                          # tag, push, release, close v0.4, open v0.5
```

`/solo:release` will:

1. Refuse to run unless you're on trunk, working tree is clean, and trunk is in sync with `origin`.
2. Suggest the next version (patch bump from the latest tag, or `release.initial_version` if none).
3. Generate notes from issues closed since the previous tag, grouped by type (`Features`, `Fixes`, `Other`).
4. Tag, push, create a GitHub Release, close the chosen milestone, and offer to open the next one.

### Strict mode (opt-in)

By default `milestone.required: false` — milestones are optional and `/solo:release` only warns about closed issues that slipped through without one.

Flip the flag when you want stricter discipline:

```yaml
milestone:
  current: "v0.4"
  required: true
```

With `required: true`:

- `/solo:capture` refuses to create an issue when no active milestone is set.
- `/solo:plan` offers a backfill pass for any open issues missing a milestone.
- `/solo:today` warns loudly about issues without a milestone.
- `/solo:release` blocks if the milestone still has unfinished issues, or if any closed issue since the last tag has no milestone.

### Migration

If you're upgrading an existing solo project:

1. Re-run `/solo:init` — it adds the `release:` and `milestone:` config blocks without touching the rest.
2. Create your first milestone: `/solo:plan milestone create v0.1`.
3. Use `/solo:plan` to backfill milestones on existing open issues.
4. Flip `milestone.required: true` once your backlog is clean.

## Design philosophy

- **Frictionless capture.** One line in, one line out.
- **Single source of truth.** GitHub Issues hold the real state.
- **Terminal-first.** No browser needed for daily work.
- **Small surface area.** A few commands, each doing one thing well.
- **Trunk-based.** Short-lived branches, merged back fast (see above).
- **Tag, don't branch.** Releases are snapshots of trunk, not parallel lines of history.
- **Solo by design.** No team features, no assignment juggling.

## License

MIT — see [LICENSE](./LICENSE).
