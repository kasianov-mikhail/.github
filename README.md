Shared [reusable workflows](https://docs.github.com/en/actions/sharing-automations/reusing-workflows) for kasianov-mikhail repositories.

## Update flow

```
┌─────────┐       push        ┌─────────┐
│ .github │ ──────────────▶   │  scout  │
└─────────┘   Notify Scout    └─────────┘
              (workflow_dispatch)   │
                                   │ tests pass on main
                                   │ (repository_dispatch)
                                   ▼
                              ┌─────────┐
                              │scout-ip │
                              └─────────┘
                                   │
                                   │ update Package.resolved
                                   │ debounce 15 min
                                   ▼
                              ┌──────────┐
                              │TestFlight│
                              └──────────┘
```

## 🔧 Auto Fix

Runs Claude Code to diagnose a workflow failure and create a fix PR.

```yaml
name: Auto Fix

on:
  workflow_run:
    workflows: [CI]  # name of the workflow to monitor
    types: [completed]

permissions:
  contents: write
  pull-requests: write
  actions: read
  id-token: write

jobs:
  fix:
    if: github.event.workflow_run.conclusion == 'failure'
    uses: kasianov-mikhail/.github/.github/workflows/auto-fix.yml@main
    with:
      workflow_name: CI
      run_id: ${{ github.event.workflow_run.id }}
      head_branch: ${{ github.event.workflow_run.head_branch }}
    secrets:
      ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
      GH_PAT: ${{ secrets.GH_PAT }}
```

## 🔀 Resolve Conflicts

Resolves merge conflicts on a PR using Claude Code.
The caller is responsible for finding conflicted PRs and passing each PR number.

```yaml
name: Resolve Conflicts

on:
  push:
    branches: [main]

permissions:
  contents: write
  pull-requests: write
  actions: read
  id-token: write

jobs:
  find:
    runs-on: ubuntu-latest
    outputs:
      prs: ${{ steps.check.outputs.prs }}
    steps:
      - name: Find conflicted PRs
        id: check
        env:
          GH_TOKEN: ${{ secrets.GH_PAT }}
        run: |
          sleep 30
          PRS=$(gh pr list --repo ${{ github.repository }} --json number,mergeable --jq '[.[] | select(.mergeable == "CONFLICTING") | .number]')
          echo "prs=$PRS" >> $GITHUB_OUTPUT

  resolve:
    needs: find
    if: needs.find.outputs.prs != '[]'
    strategy:
      matrix:
        pr: ${{ fromJson(needs.find.outputs.prs) }}
    uses: kasianov-mikhail/.github/.github/workflows/resolve-conflicts.yml@main
    with:
      pr: ${{ matrix.pr }}
    secrets:
      ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
      GH_PAT: ${{ secrets.GH_PAT }}
```

## Required secrets

- `ANTHROPIC_API_KEY` — for Claude Code
- `GH_PAT` — personal access token with `repo` and `workflow` scopes
