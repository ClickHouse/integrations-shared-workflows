# integrations-shared-workflows

Reusable GitHub Actions workflows shared across ClickHouse integrations repos
(language clients, connectors, …). Source repos call these via `workflow_call`
so the logic lives in one place and per-repo specifics are passed as inputs.

## Workflows

### `claude-pr-triage.yml` — Triage PRs with Claude

Classifies each PR (category + low/medium/high risk) against a rubric the caller
supplies, then applies `triage:*` / `risk:*` labels and upserts a single sticky
comment. Claude is read-only on PR state; a follow-up workflow step performs all
label/comment writes from Claude's validated JSON output, so a prompt injection
in a PR cannot mislabel or post a tampered comment.

For the judgement criteria, each caller passes its own rubric via
the `triage_instructions` input, so the same workflow serves repos with very
different risk surfaces.

| Input | Required | Default | Purpose |
|---|---|---|---|
| `triage_instructions` | yes | — | The repo's rubric only: category meanings, High/Medium/Low risk rules, optional reviewer-action policy. The method, Concerns guidance, schema, and comment format come from the skeleton. |
| `categories` | no | `bugfix,feature,refactor,perf,deps,docs,tests,infra` | Allowed `category` values; drives both the JSON-schema enum and validation. |
| `model` | no | action default | Claude model override (e.g. `claude-opus-4-8`). |
| `max_turns` | no | `15` | Max agent turns. |
| `pr_number` | no | from event | PR to triage; forward your `workflow_dispatch` input here. `pull_request` callers can omit it. |

**Secret:** `ANTHROPIC_API_KEY` (required), passed via `secrets: inherit`.

This workflow processes untrusted PR content, so it's hardened against secret
exfiltration: the Claude step has no network tools and no arbitrary Bash (only
read-only `gh` reads), can't write files or post comments, and never executes
the PR's code. The workflow references only `ANTHROPIC_API_KEY` and the
`permissions`-scoped `GITHUB_TOKEN`, and GitHub injects a secret into a step only
where it's explicitly referenced — so `secrets: inherit` doesn't expose anything
else, and a per-job `environment` would add no isolation here. Rotate the key if
you ever suspect compromise.

**Quick Start:**

 * Add `ANTHROPIC_API_KEY` secret.
 * Add `triage:` and `risk:` labels.
 * Copy paste caller example and provide the category and risk descriptions for your repo.

See [`examples/caller-claude-pr-triage.yml`](examples/caller-claude-pr-triage.yml)
for a copy-paste caller.

See [here](https://github.com/ClickHouse/clickhouse-cs/blob/main/.github/workflows/claude-pr-triage.yml) for the live workflow in the .NET repo.

### `cross-repo-bug-relay.yml` — Relay issues/PRs to a central repo

Called from source repos on `issues` / `pull_request` events to copy the item
into a central cross-repo-investigation repo (issues labelled `relayed`, PRs
`relayed-pr`). See the header of the workflow file for usage; `central_repo` is
configurable and defaults to the language-client pipeline.

## Conventions

- Pin reusable-workflow refs to `@main` (or a tag/SHA) in callers.
