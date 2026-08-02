# docker-build-workflow Specification

## Purpose

Reusable `workflow_call` workflow (`docker-build.yml`) that builds a container image, optionally smoke-tests it, and reports (non-blocking) vulnerability scan results. It MUST NOT push images to any registry in v1.

## Requirements

### Requirement: Input Contract

The workflow MUST declare `workflow_call` inputs: `image-tag` (required), `dockerfile-path` (default `"Dockerfile"`), `context` (default `"."`), and `smoke-test-command` (string, default `""`). It MUST NOT declare any `push`, registry, or credentials input in v1.

`trivy-version` MUST be declared as an overridable input ONLY IF `aquasecurity/trivy-action@v0.36.0` supports a `version:` input (open question, to be confirmed during apply). If the action does not expose such an input, Trivy's version is fixed by the pinned action reference in the workflow file instead — the same treatment as hadolint's version in `hadolint-workflow`.

#### Scenario: Minimal invocation

- GIVEN a caller passes only `image-tag`
- WHEN the workflow runs
- THEN it builds using default `dockerfile-path`, `context`, skips smoke test, and scans with Trivy at whichever version resolution applies (input override if supported, else the pinned action default)

### Requirement: No Registry Push

The workflow MUST build the image locally only and MUST NOT push it to any container registry, and MUST NOT perform any registry login step.

#### Scenario: Build never pushes

- GIVEN any valid invocation
- WHEN the workflow runs
- THEN no registry login or push step executes, regardless of input values

### Requirement: Image Reference Availability

The workflow MUST expose the built image reference as a job-level `env: IMAGE_REF` variable (`IMAGE_REF: ${{ inputs.image-tag }}`), scoped only within the single `docker-build` job. Build, smoke-test, and scan all run in this same job. The workflow MUST NOT expose `IMAGE_REF` as a `workflow_call` output.

This is not a `workflow_call` output because with `push: false` hardcoded (no registry push in v1), the built image exists only in this job's local Docker daemon — there is nothing durable for a downstream job to reference. A `workflow_call` output would be misleading or non-functional regardless of the propagation mechanism used, since no other job could pull or reference the image it named.

#### Scenario: IMAGE_REF available within the job

- GIVEN a successful build
- WHEN the smoke-test or scan step within the same `docker-build` job runs
- THEN it can read `IMAGE_REF` from the job environment

#### Scenario: No cross-job output exists

- GIVEN a caller workflow defines a downstream job with `needs: docker-build`
- WHEN it inspects `needs.docker-build.outputs`
- THEN no `IMAGE_REF` (or equivalent) output is present

### Requirement: Optional Smoke Test

WHEN `smoke-test-command` is empty (the default), the workflow MUST skip the smoke-test step entirely. WHEN `smoke-test-command` is non-empty, the workflow MUST execute it as a shell command with the job-level `IMAGE_REF` env variable available, and the job MUST pass or fail based solely on the command's exit code — the workflow MUST NOT perform any stdout/stderr assertion itself.

#### Scenario: Empty command skips smoke test

- GIVEN `smoke-test-command` is not set
- WHEN the workflow runs
- THEN the smoke-test step is skipped and does not affect job result

#### Scenario: Non-empty command passes

- GIVEN `smoke-test-command` exits 0 against the job's `IMAGE_REF`
- WHEN the workflow runs
- THEN the job succeeds

#### Scenario: Non-empty command fails

- GIVEN `smoke-test-command` exits non-zero
- WHEN the workflow runs
- THEN the job MUST fail

### Requirement: Report-Only Vulnerability Scan

The workflow MUST run Trivy against the built image (at the overridden `trivy-version` if that input exists, else the pinned action default) and MUST NOT fail the job based on scan findings; results are informational only.

#### Scenario: Vulnerabilities found do not fail build

- GIVEN Trivy reports vulnerabilities in the built image
- WHEN the workflow runs
- THEN the job MUST still succeed if build and smoke-test steps succeeded

### Requirement: Permissions Declaration

The workflow requires no elevated permissions beyond `contents: read`. Callers MUST declare `contents: read`.

#### Scenario: Minimal permissions suffice

- GIVEN a caller job with `permissions: { contents: read }`
- WHEN the workflow runs
- THEN build, smoke test, and scan complete without permission errors
