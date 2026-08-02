# workflow-consumption-contract Specification

## Purpose

Cross-cutting rules governing how any consumer repository references, versions, and authorizes the reusable workflows in `pipelines`.

## Requirements

### Requirement: Versioned Reference

Consumers MUST reference workflows as `uses: robbasualto/pipelines/.github/workflows/<name>.yml@v1`. `@v1` MUST be a moving major tag that only ever advances to non-breaking releases; the tag MUST NOT be force-moved onto a change that alters an existing input's meaning, removes an input/output, or changes pass/fail semantics.

#### Scenario: Consumer pins to v1

- GIVEN a consumer repo declares `uses: .../docker-build.yml@v1`
- WHEN a non-breaking fix is tagged as the new `v1`
- THEN the consumer's next run picks up the fix with no workflow-file change

#### Scenario: Breaking change requires new major

- GIVEN a change would alter an existing workflow's documented input/output contract
- WHEN it is released
- THEN it MUST ship under a new major tag (e.g. `@v2`), and `@v1` MUST NOT be moved onto it

### Requirement: Repository Access Prerequisite

Before any private consumer repository can invoke these workflows, the `pipelines` repository's GitHub Settings → Actions → Access MUST be set to allow "Accessible from repositories owned by the user." Consumers invoking before this is set MUST receive an opaque workflow-not-found/permission failure, not a silent no-op.

#### Scenario: Access not configured

- GIVEN the setting is not yet enabled
- WHEN a private consumer repo triggers a job using one of these workflows
- THEN the job fails with a workflow resolution/permission error

#### Scenario: Access configured

- GIVEN the setting is enabled
- WHEN any repository owned by the same account invokes a workflow at `@v1`
- THEN the workflow resolves and runs

### Requirement: Explicit Permissions and Secrets Propagation

Reusable workflows do NOT inherit a caller's default `permissions:`. Each consumer job MUST declare the `permissions:` block required by the specific workflow(s) it calls (minimum `contents: read` for all five; `gitleaks.yml` accepts an optional `gh_token` secret, falling back to `github.token` when the caller omits `secrets: inherit` or an explicit passthrough, so `pull_request`-triggered scans still succeed without one).

#### Scenario: Missing permissions block causes silent failure

- GIVEN a caller job omits `permissions:` entirely
- WHEN the workflow needs a token-gated operation
- THEN the job MUST fail with a 403/permission-denied error rather than degrade silently

#### Scenario: Correct propagation

- GIVEN a caller declares the documented `permissions:` and `secrets:` for the workflows it invokes
- WHEN the pipeline runs
- THEN no permission-related failures occur

### Requirement: Overridable Tool Versions

`golangci-lint-version` MUST have a pinned default and MUST be overridable per-call, so a consumer can bump this tool version without waiting for a new `pipelines` release/tag.

`trivy-version` MUST be overridable under the same terms ONLY IF `aquasecurity/trivy-action@v0.36.0` exposes a `version:` input (open question, to be confirmed during apply); otherwise Trivy's version is fixed by the pinned action reference and requires a `pipelines` release to change. `hadolint-version` is NOT part of this contract: `hadolint-action` has no version-of-the-binary input, so its version is always fixed by the pinned action reference (see `hadolint-workflow`).

#### Scenario: Consumer bumps golangci-lint version independently

- GIVEN a consumer passes a newer `golangci-lint-version` in its `with:` block
- WHEN the workflow runs
- THEN the newer version is used without any change to `pipelines`

#### Scenario: Trivy version bump depends on action support

- GIVEN `trivy-version` is confirmed as a supported input during apply
- WHEN a consumer passes a newer `trivy-version` in its `with:` block
- THEN the newer version is used without any change to `pipelines`; if the action does not support this input, no such override exists and a version bump requires a `pipelines` release
