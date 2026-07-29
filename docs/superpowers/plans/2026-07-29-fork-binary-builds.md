# Fork Binary Builds Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add and run a manually dispatched GitHub Actions workflow that builds RTK packages for seven platform/package targets without invoking upstream release automation.

**Architecture:** A standalone `.github/workflows/build-binaries.yml` copies the proven build and packaging jobs from the existing reusable release workflow but stops after artifact upload. The workflow is committed on the feature branch, copied to `develop` through a targeted cherry-pick, and dispatched against `feat/codex-hook`.

**Tech Stack:** GitHub Actions, Rust stable toolchain, Cargo, cargo-deb, cargo-generate-rpm, GitHub CLI

## Global Constraints

- The workflow must be manually dispatched with `workflow_dispatch`.
- The operator dispatches from a branch that contains the workflow, then selects the source revision through the required `source_ref` input.
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
- Consumes: the required `source_ref` workflow input and the optional repository variable `RTK_TELEMETRY_URL`.
- Produces: GitHub Actions artifacts named `rtk-<rust-target>`, `rtk-deb`, and `rtk-rpm`.

- [ ] **Step 1: Create the workflow**

Create a `Build Fork Binaries` workflow with:

```yaml
on:
  workflow_dispatch:
    inputs:
      source_ref:
        description: Branch, tag, or commit SHA to build
        required: true
        type: string

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
rtk rg -n "workflow_dispatch|source_ref|contents: read|persist-credentials: false|target:|Upload artifact|Upload DEB|Upload RPM|RTK_TELEMETRY_TOKEN|action-gh-release|create-github-app-token" .github/workflows/build-binaries.yml
```

Expected: the trigger, required `source_ref` input, read permission, five Rust
targets, and three upload step names are present; `action-gh-release`,
`create-github-app-token`, and `RTK_TELEMETRY_TOKEN` are absent; every checkout
uses `persist-credentials: false`.

- [ ] **Step 3: Parse the workflow**

Run:

```bash
rtk ruby -e 'require "yaml"; YAML.load(File.read(".github/workflows/build-binaries.yml")); puts "workflow yaml: ok"'
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
FORK_WORKFLOW_COMMIT=$(rtk git log --format=%H --max-count=1 -- .github/workflows/build-binaries.yml)
test -n "$FORK_WORKFLOW_COMMIT"
rtk git status --short --branch
rtk git fetch origin develop
```

Expected: the workflow-specific commit is recorded and `origin/develop` is
available. If later hardening changes also modify `.github/workflows/build-binaries.yml`,
record that latest workflow commit instead of assuming the current branch `HEAD`
is workflow-only.

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
- Consumes: `build-binaries.yml` from the dispatch branch and source ref `feat/codex-hook`.
- Produces: a GitHub Actions run with five target jobs, one DEB job, and one RPM job.

- [ ] **Step 1: Dispatch the workflow**

Run:

```bash
rtk gh workflow run build-binaries.yml --ref develop -f source_ref=feat/codex-hook
```

Expected: GitHub accepts the dispatch. Dispatching with
`--ref feat/codex-hook` fails unless that ref already contains the workflow.

- [ ] **Step 2: Resolve the new run**

Run:

```bash
FORK_BUILD_RUN_ID=$(rtk gh run list --workflow build-binaries.yml --branch develop --limit 1 --json databaseId --jq '.[0].databaseId')
test -n "$FORK_BUILD_RUN_ID"
```

Expected: one new `workflow_dispatch` run appears for the dispatch branch.

- [ ] **Step 3: Verify job scheduling**

Run:

```bash
rtk gh run view "$FORK_BUILD_RUN_ID"
```

Expected: build jobs are scheduled for all five Rust targets plus DEB and RPM.

- [ ] **Step 4: Wait for completion and verify artifacts**

Run:

```bash
rtk gh run watch "$FORK_BUILD_RUN_ID" --exit-status
rtk gh run view "$FORK_BUILD_RUN_ID" --json jobs,headBranch,headSha,conclusion
rtk gh run download "$FORK_BUILD_RUN_ID" --dir /tmp/rtk-fork-build-artifacts
rtk find /tmp/rtk-fork-build-artifacts -maxdepth 2 -type f
```

Expected: the run succeeds, all five matrix builds plus DEB and RPM complete,
the `Record source revision` step in every job prints the resolved source SHA
for the selected `source_ref`, and all seven artifacts are present. The GitHub
run `headSha` identifies the dispatch branch, not necessarily the selected
`source_ref`, so do not use `headSha` as proof of the source revision.

- [ ] **Step 5: Restore the working branch**

Run:

```bash
rtk git switch feat/codex-hook
rtk git status --short --branch
```

Expected: the feature branch is restored and clean, with its local design and
workflow commits ahead of `origin/feat/codex-hook` until explicitly pushed.
