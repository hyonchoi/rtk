# Fork Binary Builds

## Goal

Provide an on-demand GitHub Actions workflow that builds distributable RTK
binaries from any branch, tag, or commit in this fork without invoking the
upstream release process.

## Workflow

Add `.github/workflows/build-binaries.yml` with a `workflow_dispatch` trigger.
The operator dispatches from a ref containing the workflow and passes the
branch, tag, or commit to build through the required `source_ref` input. Every
checkout uses that input, keeping workflow discovery separate from build source
selection.

The workflow uses the same target matrix and packaging conventions as
`.github/workflows/release.yml`:

- `x86_64-apple-darwin`
- `aarch64-apple-darwin`
- `x86_64-unknown-linux-musl`
- `aarch64-unknown-linux-gnu`
- `x86_64-pc-windows-msvc`
- Debian package
- RPM package

Each matrix job checks out `source_ref`, installs the required Rust
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

The fork telemetry URL remains optional. The artifact-only workflow does not
expose repository secrets to the selected source revision.

## Failure Behavior

Matrix jobs use `fail-fast: false`, allowing independent platforms to finish
when another target fails. Each target uploads only its own completed package.
The workflow run reports failure when any required target or package job fails.

## Verification

Validate the workflow syntax locally where tooling permits, push it to the
fork's default branch so GitHub exposes `workflow_dispatch`, dispatch it from
that branch with `source_ref=feat/codex-hook`, and confirm that all seven build
jobs build the requested source revision.
