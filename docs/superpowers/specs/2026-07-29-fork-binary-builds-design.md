# Fork Binary Builds

## Goal

Provide an on-demand GitHub Actions workflow that builds distributable RTK
binaries from any branch, tag, or commit in this fork without invoking the
upstream release process.

## Workflow

Add `.github/workflows/build-binaries.yml` with a `workflow_dispatch` trigger
and a required `source_ref` input. The operator dispatches from a branch that
contains the workflow, then selects the branch, tag, or commit to build through
`source_ref`.

The workflow uses the same target matrix and packaging conventions as
`.github/workflows/release.yml`:

- `x86_64-apple-darwin`
- `aarch64-apple-darwin`
- `x86_64-unknown-linux-musl`
- `aarch64-unknown-linux-gnu`
- `x86_64-pc-windows-msvc`
- Debian package
- RPM package

Each matrix job checks out the selected revision, installs the required Rust
target and native cross-compilation tools, builds in release mode, packages the
binary, and uploads it with `actions/upload-artifact`.

## Boundaries

The workflow has read-only repository permissions and does not:

- create or move tags;
- create a GitHub Release;
- update the `latest` tag;
- notify Discord;
- update Homebrew;
- require GitHub App or upstream repository secrets.

Fork telemetry remains optional. The build receives only the public repository
variable `RTK_TELEMETRY_URL`; repository secrets are deliberately withheld
because the selected `source_ref` can run arbitrary build scripts. Checkout also
disables persisted Git credentials so selected source code cannot reuse the
workflow token after checkout.

## Failure Behavior

Matrix jobs use `fail-fast: false`, allowing independent platforms to finish
when another target fails. Each target uploads only its own completed package.
The workflow run reports failure when any required target or package job fails.

## Verification

Validate the workflow syntax locally where tooling permits, push it to the
fork's default branch so GitHub exposes `workflow_dispatch`, dispatch from that
branch with `source_ref=feat/codex-hook`, wait for completion, confirm that all
seven build jobs succeeded, confirm each job's `Record source revision` step
printed the checked-out source SHA, and verify that all seven artifacts were
uploaded. GitHub's run `headSha` identifies the dispatch branch, not the
selected `source_ref`.
