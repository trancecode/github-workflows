# github-workflows

Reusable GitHub Actions workflows shared by the managed repositories.

Before this repository existed, each workflow was copied into every repository at
setup time and never re-synchronized, so the copies drifted. One of them,
`monthly-audit.yml`, silently reported zero merged pull requests in every private
repository for months because its `permissions:` block omitted `pull-requests`.
Eleven separate autonomous audit sessions rediscovered that bug and reached four
different conclusions about it.

Design and rationale live in
[`claude_code_environment/docs/plans/2026-07-31-shared-workflows-migration-design.md`](https://github.com/herve-quiroz/claude_code_environment/blob/main/docs/plans/2026-07-31-shared-workflows-migration-design.md).

## Workflows

| Workflow | Purpose |
|---|---|
| `check-go-version.yml` | Files an issue when a newer Go release is available |
| `commit-validation.yml` | Enforces commit message conventions |
| `failing-checks-label.yml` | Maintains the `failing checks` label on pull requests |
| `input-required-label.yml` | Maintains the `input required` label on issues and pull requests |
| `issue-dependencies.yml` | Maintains `ready` and `blocked` from native issue dependencies |
| `merge-conflict-label.yml` | Maintains the `merge conflict` label on pull requests |
| `monthly-audit.yml` | Opens the monthly workflow audit issue |

## Calling a workflow

Callers keep a stub that supplies the trigger and the permissions. The trigger
must live in the calling repository, so the stub cannot be eliminated entirely.

```yaml
name: Monthly Workflow Audit

on:
  schedule:
    - cron: '0 0 1 * *'
  workflow_dispatch:

permissions:
  issues: write
  contents: read
  pull-requests: read

jobs:
  audit:
    uses: trancecode/github-workflows/.github/workflows/monthly-audit.yml@main
```

Two rules govern the stub:

* **Permissions must be granted here.** A called workflow's `GITHUB_TOKEN` can
  only be equal to or more restrictive than the caller's, so a scope the stub
  omits cannot be granted centrally.
* **Triggers stay per-repository.** Stubs deliberately preserve whatever triggers
  that repository already used. `nrg` validates commits on push as well as on
  pull requests; most repositories only validate pull requests. The reusable
  workflow handles both.

Context resolves to the caller: `github.repository`, `actions/checkout` and
`secrets.GITHUB_TOKEN` all refer to the repository doing the calling.

## Pinning

Callers pin `@main`. A change reaches every repository on its next run.

This is deliberate. The workflows where a break would hurt most are the ones that
never change, and `@main` gives the best recovery story: one revert here heals
every consumer at once.

## Development loop

1. Branch here, edit, push. No caller is affected.
2. `lint.yml` runs actionlint on the pull request.
3. For a risky change, push a branch to a low-traffic consumer with its stub
   re-pinned to your branch, then run it directly:

   ```bash
   gh workflow run monthly-audit.yml \
     -R herve-quiroz/lockstep \
     --ref my-test-branch
   ```

   Delete the branch afterwards. This matters most for the scheduled workflows,
   where the alternative is waiting a month for signal.
4. Merge to `main`.

## Recovery

If a change here breaks consumers, revert the commit on `main`. Every caller
heals on its next run.

This path is manual by necessity. `input-required-label.yml` and the other label
workflows are the autonomous agent's own control plane, so breaking them can
remove its ability to detect the work needed to fix them.
