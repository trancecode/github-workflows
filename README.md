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
| `go-analysis.yml` | gofmt, go vet, staticcheck, golangci-lint, blank lines |
| `go-build-test.yml` | go build and the test suite |
| `go-race.yml` | `go test -race` |
| `rust-ci.yml` | cargo fmt, clippy, test, build |
| `rust-docs.yml` | cargo doc, deployed to GitHub Pages |

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

   **`workflow_dispatch` only resolves workflows that already exist on the
   consumer's default branch.** To test a stub that is not on `main` yet, give
   the branch copy a `push:` trigger scoped to that branch instead, which is
   what the initial Go rollout used.
4. Merge to `main`.

## Recovery

If a change here breaks consumers, revert the commit on `main`. Every caller
heals on its next run.

This path is manual by necessity. `input-required-label.yml` and the other label
workflows are the autonomous agent's own control plane, so breaking them can
remove its ability to detect the work needed to fix them.


## Go workflow inputs

`go-analysis.yml` and `go-race.yml` take inputs for the two things that genuinely
vary across repositories: the apt packages needed to compile, and whether the
tests need a display.

Three further inputs exist so a repository can adopt the workflow before its
backlog is clear, rather than the whole adoption being blocked:

- `golangci-args` defaults to `--disable=staticcheck`, because staticcheck runs
  standalone and pinned in the same workflow. golangci-lint bundles it too but
  with the QF quickfix family enabled, so leaving both on reports one tool twice
  with different opinions. Note that golangci-lint's `unused` linter is
  staticcheck's `U1000` under another name, so excluding one usually means
  excluding both.
- `staticcheck-checks` defaults to `inherit`. A repository adopting staticcheck
  for the first time can narrow this, for example `inherit,-U1000,-SA1019`.
- `check-gofmt` and `check-blank-lines` can be turned off.

Every exclusion in a consumer stub should have a tracked issue against it. They
are adoption ramps, not settled policy.

## Required status checks

**A reusable workflow call reports its check as `caller-job / called-job`, not
`caller-job`.** Migrating a workflow that some repository requires as a status
check therefore breaks branch protection silently: the required context stops
existing, and the next pull request becomes unmergeable with no obvious cause.

This already happened twice here, to `analysis` and then to `build`. Before
migrating a workflow, check both places, because rulesets and classic branch
protection are separate:

```bash
gh api repos/OWNER/REPO/branches/main/protection/required_status_checks \
  -q '.contexts'
gh api repos/OWNER/REPO/rulesets
```

Then update the contexts in the same change, and verify against what is actually
reported:

```bash
gh api repos/OWNER/REPO/commits/$(gh api repos/OWNER/REPO/commits/main -q .sha)/check-runs \
  -q '.check_runs[].name'
```

## Why a workflow lives here

Not "how many repositories already copy it". That metric makes every new
capability look unjustified until it has already been duplicated a few times,
which is how the drift started in the first place.

The test is whether a **new** repository should get the capability for free.
`rust-ci.yml` and `rust-docs.yml` have exactly one consumer today and still
belong here: the next Rust repository should inherit working CI from a stub
rather than a copy that begins drifting the day it is made.

That principle also shapes the defaults. `rust-docs.yml` reads the rustdoc
redirect target from `Cargo.toml` rather than taking it as required
configuration, so a new crate needs nothing beyond its apt packages.

## Race cadence

Race builds roughly double test time, and Actions minutes are only billed for
private repositories. So public repositories and fast libraries run `-race` on
every push, while the private GUI repositories run it weekly.
