# Fork Binary Builds Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add and run a manually dispatched GitHub Actions workflow that builds RTK packages for seven platform/package targets without invoking upstream release automation.

**Architecture:** A standalone `.github/workflows/build-binaries.yml` copies the proven build and packaging jobs from the existing reusable release workflow but stops after artifact upload. The workflow is committed on the feature branch, copied to `develop` through a targeted cherry-pick, and dispatched from `develop` with `source_ref` set to the branch, tag, or SHA to build.

**Tech Stack:** GitHub Actions, Rust stable toolchain, Cargo, cargo-deb, cargo-generate-rpm, GitHub CLI

## Global Constraints

- The workflow must be manually dispatched with `workflow_dispatch`.
- The operator dispatches from a ref containing the workflow and selects the source revision through the required `source_ref` input.
- Build `x86_64-apple-darwin`, `aarch64-apple-darwin`, `x86_64-unknown-linux-musl`, `aarch64-unknown-linux-gnu`, `x86_64-pc-windows-msvc`, Debian, and RPM artifacts.
- Use read-only repository permissions.
- Do not create or move tags, create a GitHub Release, update `latest`, notify Discord, update Homebrew, or expose repository secrets to the selected source revision.
- Use `fail-fast: false` for the target matrix.
- Prefix repository shell commands with `rtk`.

---

### Task 1: Add the artifact-only build workflow

**Files:**
- Create: `.github/workflows/build-binaries.yml`

**Interfaces:**
- Consumes: required workflow input `source_ref` and repository variable `RTK_TELEMETRY_URL`.
- Produces: GitHub Actions artifacts named `rtk-<rust-target>`, `rtk-deb`, and `rtk-rpm`.

- [ ] **Step 1: Create the workflow**

Create a `Build Fork Binaries` workflow with:

```yaml
on:
  workflow_dispatch:
    inputs:
      source_ref:
        description: Branch, tag, or SHA to build
        required: true

permissions:
  contents: read
```

Copy the existing release workflow's `build`, `build-deb`, and `build-rpm`
jobs, including target setup, release builds, archive creation, and
`actions/upload-artifact@v4`. Omit the `release`, `notify-discord`, and
`homebrew` jobs.

- [ ] **Step 2: Check workflow structure**

Run:

```bash
rtk rg -n "workflow_dispatch|contents: read|target:|Upload artifact|Upload DEB|Upload RPM|action-gh-release|create-github-app-token" .github/workflows/build-binaries.yml
```

Expected: the trigger, read permission, five Rust targets, and three upload
step names are present; `action-gh-release` and `create-github-app-token` are
absent.

- [ ] **Step 3: Parse the workflow**

Run:

```bash
rtk ruby -e 'require "yaml"; YAML.load_file(".github/workflows/build-binaries.yml", aliases: true); puts "workflow yaml: ok"'
```

Expected: `workflow yaml: ok`.

- [ ] **Step 4: Commit the workflow**

Stage `.github/workflows/build-binaries.yml`, inspect the cached diff, and
commit:

```bash
rtk git add .github/workflows/build-binaries.yml
rtk git --no-pager diff --cached
rtk git commit -m "Add fork binary build workflow"
```

### Task 2: Publish the workflow to the default branch

**Files:**
- Modify: Git history for `develop`

**Interfaces:**
- Consumes: the Task 1 workflow commit.
- Produces: `.github/workflows/build-binaries.yml` on `origin/develop`, making manual dispatch available in GitHub.

- [ ] **Step 1: Record the workflow commit and verify branch state**

Run:

```bash
FORK_WORKFLOW_COMMIT=$(rtk git rev-parse HEAD)
test -n "$FORK_WORKFLOW_COMMIT"
rtk git status --short --branch
rtk git fetch origin develop
```

Expected: the feature branch is clean and `origin/develop` is available.

- [ ] **Step 2: Update local develop and cherry-pick the workflow commit**

Run:

```bash
rtk git switch develop
rtk git merge --ff-only origin/develop
rtk git cherry-pick "$FORK_WORKFLOW_COMMIT"
```

Expected: `develop` receives only the workflow commit, without the feature
implementation or design documents.

- [ ] **Step 3: Push the default branch**

Run:

```bash
rtk git push origin develop
```

Expected: `origin/develop` advances to the workflow commit.

### Task 3: Trigger and verify the feature build

**Files:**
- No file changes

**Interfaces:**
- Consumes: `build-binaries.yml` from the default branch and source ref `feat/codex-hook`.
- Produces: a GitHub Actions run with five target jobs, one DEB job, and one RPM job.

- [ ] **Step 1: Dispatch the workflow**

Run:

```bash
rtk gh workflow run build-binaries.yml --ref develop -f source_ref=feat/codex-hook
```

Expected: GitHub accepts the dispatch.

- [ ] **Step 2: Resolve the new run**

Run:

```bash
FORK_BUILD_RUN_ID=$(rtk gh run list --workflow build-binaries.yml --branch develop --event workflow_dispatch --limit 1 --json databaseId --jq '.[0].databaseId')
test -n "$FORK_BUILD_RUN_ID"
```

Expected: one new `workflow_dispatch` run appears.

- [ ] **Step 3: Verify job scheduling**

Run:

```bash
rtk gh run view "$FORK_BUILD_RUN_ID"
```

Expected: build jobs are scheduled for all five Rust targets plus DEB and RPM.

- [ ] **Step 4: Restore the working branch**

Run:

```bash
rtk git switch feat/codex-hook
rtk git status --short --branch
```

Expected: the feature branch is restored and clean, with its local design and
workflow commits ahead of `origin/feat/codex-hook` until explicitly pushed.
