# semantic-release-issue-4290

Minimal reproduction for the bug reported in [semantic-release/semantic-release#4290](https://github.com/semantic-release/semantic-release/issues/4290): when the release workflow is triggered via `workflow_run` (separate from the `check` workflow), `semantic-release` always detects `main` instead of the actual branch that triggered the push (e.g. `beta`), breaking prerelease generation.

## Branch structure

This repo uses two branches to document the problem and its solution:

### `main`

Stable branch. Contains the first standard release (`v1.0.0`) and serves as the "everything works normally" baseline before introducing the prerelease scenario that triggers the bug.

### `beta`

Test branch where the problem was reproduced and iterated on until finding a real fix. The commit history documents the process:

1. **`feat: test beta release`**
   First attempt: `Release` workflow triggered by `workflow_run` pointing at the `Check` workflow, with an explicit checkout of the `head_sha`/`head_branch` corresponding to `beta`. Despite the correct checkout, `semantic-release` reported the run as if it were on `main` and produced a stable release instead of a `beta` prerelease.

2. **`feat: test beta release with env variables`**
   Workaround attempt: manually overriding `GITHUB_REF` and `GITHUB_SHA` via `env:` in the release job, to force `env-ci` (which `semantic-release` relies on to detect the current branch) to read the correct branch. GitHub Actions silently discards any attempt to override default environment variables (`GITHUB_*`/`RUNNER_*`), so the job kept resolving `main` regardless of the assigned value. This was confirmed by the run itself: `semantic-release` tried to create the release against `main` again and failed with `fatal: tag 'v1.1.0' already exists`, because that tag had already been created in the previous attempt (also incorrectly resolved as `main`).

3. **`feat: test beta release in same job`**
   The fix that worked: dropping the `workflow_run` split and running `check` and `release` as jobs within the **same workflow**, chained with `needs: check`. Since there's no independent run triggered by `workflow_run`, the context (`GITHUB_REF`) correctly reflects the branch that triggered the push, and `semantic-release` detects `beta` as a prerelease without any workaround.

## Root cause

`env-ci` (used internally by `semantic-release`) determines the current branch by reading `GITHUB_REF` on GitHub Actions. When a workflow is triggered by `workflow_run`, that run lives in an execution context separate from the original push: even if the `checkout` step targets the correct commit/branch, `GITHUB_REF` still points to the context of the run triggered by `workflow_run` (typically the default branch), not the branch that triggered the original workflow. Since `GITHUB_*` variables can't be overridden from within a workflow, there's no way to correct this from inside the release job when using `workflow_run`.

## Solution

Avoid `workflow_run` for this use case. Alternatives that preserve correct branch detection:

- **A single workflow with chained jobs** (`needs:`) — the option used in this repo.
- **Reusable workflows** (`workflow_call`) if `check` and `release` need to stay in separate files: an orchestrator workflow triggers both via `uses: ./.github/workflows/...`, preserving the original push's context.

## References

- [Original issue on semantic-release/semantic-release](https://github.com/semantic-release/semantic-release/issues/4290)
