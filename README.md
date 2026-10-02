# going-dev/.github

Shared CI and Renovate config for going-dev repositories. The public org
profile lives in [`profile/README.md`](profile/README.md).

Releases are cut by semantic-release on merge to `main` (see
[`release.yaml`](.github/workflows/release.yaml)). Pin reusable workflows by full
commit SHA with the tag as a trailing comment, so Renovate can bump both.

In `.github/workflows/`, `.yml` files are reusable workflows, called with
`workflow_call`. `.yaml` files run on this repository's own events, and some of
them call the `.yml` ones.

## gh-aw compile

[`gh-aw-compile.yml`](.github/workflows/gh-aw-compile.yml) regenerates a
repository's gh-aw lock files and generated assets on its own Renovate
branches, and pushes them back with the `going-dev-gh-aw-compiler` app. It
fetches pinned `going-dev/doc-review-agent` imports into the compiler cache,
compiles with a read-only token and no secrets, and pushes only the files
gh-aw generates.

Add this caller as `.github/workflows/gh-aw-compile.yaml`:

```yaml
name: gh-aw compile

on:
  push:
    # gh-aw and doc-review-agent branches only. Action-only bumps need no
    # recompile. A push to a listed branch runs that branch's copy of this file,
    # which can point uses: anywhere, so only writers can trigger it.
    branches:
      - "renovate/github-gh-aw-**"
      - "renovate/going-dev-doc-review-agent-**"
    paths:
      - ".github/aw/actions-lock.json"
      - ".github/workflows/*.md"
      - ".github/workflows/gh-aw-compile.yaml"

permissions: {}

jobs:
  compile:
    permissions:
      contents: read
    uses: going-dev/.github/.github/workflows/gh-aw-compile.yml@<sha> # vX.Y.Z
    with:
      # renovate: datasource=github-releases depName=github/gh-aw
      gh-aw-version: v0.89.21
    secrets:
      GH_APP_ID: ${{ secrets.GH_APP_ID }}
      GH_APP_PRIVATE_KEY: ${{ secrets.GH_APP_PRIVATE_KEY }}
```

Keep `gh-aw-version` in the caller. A Renovate gh-aw bump has to change a file
the push trigger matches, or the recompile never runs. Any other gh-aw pin in
the calling repository, such as a compile check in its own `validate.yml`,
carries the same annotation, so one Renovate PR moves them together.

The repository also needs:

- the `going-dev-gh-aw-compiler` app installed, and access to the org-level
  `GH_APP_ID` and `GH_APP_PRIVATE_KEY` secrets;
- the Renovate preset below.

## PR hygiene

Two checks on pull request titles, both skipped for PRs opened by
`github-actions`, `dependabot`, `renovate`, `scf-autopilot` and
`going-dev-gh-aw-compiler`:

- [`pr-jira-check.yml`](.github/workflows/pr-jira-check.yml) requires a Jira
  key such as `SRE-123`.
- [`semantic-pr.yml`](.github/workflows/semantic-pr.yml) requires a
  Conventional Commits header. The squash-merge commit takes the PR title, and
  semantic-release reads it.

Add these callers as `.github/workflows/pr-jira-check.yaml` and
`.github/workflows/semantic-pr.yaml`:

```yaml
name: PR Jira ticket check

on:
  pull_request:
    types: [opened, reopened, edited, synchronize]

permissions: {}

jobs:
  jira-check:
    permissions: {}
    uses: going-dev/.github/.github/workflows/pr-jira-check.yml@<sha> # vX.Y.Z
```

```yaml
name: Semantic PR

on:
  pull_request_target:
    types: [opened, edited, reopened, synchronize]

permissions: {}

jobs:
  semantic-pr:
    permissions:
      pull-requests: read
    uses: going-dev/.github/.github/workflows/semantic-pr.yml@<sha> # vX.Y.Z
```

Both also work from a `pull_request` trigger. They only read the event payload
and never check out the PR's code. The tradeoff is that the PR's own copy of the
caller runs, so a PR can edit its gate.

A called job reports its check as `<caller job> / <called job>`. These report
as `jira-check / check` and `semantic-pr / check`. Require those names in the
repository's ruleset or branch protection.

## Renovate preset

[`renovate/gh-aw.json`](renovate/gh-aw.json) holds the Renovate rules every gh-aw
repository needs:

- generated files are left alone;
- gh-aw bumps are batched weekly;
- `# renovate:` annotated versions in workflows are tracked;
- the SHA-pinned doc-review-agent import is tracked.

```json
{
  "extends": [
    "config:recommended",
    "github>going-dev/.github//renovate/gh-aw"
  ]
}
```

Preset rules apply before a repository's own `packageRules`, and a later
matching rule wins. A repository that automerges GitHub Actions updates must
exclude `going-dev/.github` in its own rules. Otherwise a bump of the workflow
that receives the compiler app key merges without review.
