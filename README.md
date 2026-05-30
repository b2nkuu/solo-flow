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

## Design philosophy

- **Frictionless capture.** One line in, one line out.
- **Single source of truth.** GitHub Issues hold the real state.
- **Terminal-first.** No browser needed for daily work.
- **Small surface area.** A few commands, each doing one thing well.
- **Trunk-based.** Short-lived branches, merged back fast (see above).
- **Solo by design.** No team features, no assignment juggling.

## License

MIT — see [LICENSE](./LICENSE).
