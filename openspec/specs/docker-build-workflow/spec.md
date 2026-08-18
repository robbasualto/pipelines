# Docker Build Workflow Specification

## Purpose

Define caller-selectable runner placement and optional registry publishing while preserving local image validation.

## Requirements

### Requirement: Input contract and backward-compatible local mode

The workflow MUST expose `on.workflow_call.inputs.runner` as a non-required string defaulting to `ubuntu-latest`, and its sole job MUST use `${{ inputs.runner }}` in `runs-on`. It MUST expose `push` as an optional boolean defaulting to `false` and `registry` as an optional string defaulting to `""`. It MUST retain required `image-tag`, all other existing inputs, `load: true`, caller `smoke-test-command`, report-only Trivy behavior, permissions, and action references.

When `push` is false, the workflow MUST build and load exactly `image-tag` locally with `push: false`, MUST skip registry login and push, and MUST preserve the existing smoke-test and Trivy behavior.

#### Scenario: Hosted default or explicit runner in local mode

- GIVEN a caller omits `runner` or supplies a hosted runner string
- WHEN the workflow builds an image
- THEN placement defaults or overrides as specified, the image is locally loaded, no registry push occurs, and any supplied smoke test runs unchanged.

### Requirement: Optional registry publishing

When `push` is true, the workflow MUST require a non-empty `registry` input and MUST construct `IMAGE_REF` as `${registry}/${image-tag}`. It MUST validate `NEXUS_DOCKER_PUSH_USERNAME` and `NEXUS_DOCKER_PUSH_PASSWORD` in the selected runner environment before building. It MUST build with `push: false` and `load: true`, run the smoke test and report-only Trivy scan against the fully qualified local image, then log in with the validated runner environment credentials via stdin and push `IMAGE_REF`.

The workflow MUST remain registry-agnostic: it MUST NOT declare credentials as workflow inputs or secrets and MUST NOT hard-code a registry host. Deployments MAY expose `NEXUS_REGISTRY_HOST` as the runner's registry-host contract/default reference; callers still provide the `registry` input explicitly when enabling push mode.

#### Scenario: Valid push mode

- GIVEN a caller sets `push: true`, supplies a non-empty `registry`, and selects a runner exposing both Nexus credential variables
- WHEN the workflow runs
- THEN it builds and loads `${registry}/${image-tag}`, smoke-tests and scans that local image, then logs in and pushes that exact reference.

#### Scenario: Invalid push configuration

- GIVEN `push: true` and either an empty `registry` or a missing runner credential variable
- WHEN the workflow starts
- THEN it fails validation before the image build and does not attempt registry login or push.

### Requirement: Image reference scope

The workflow MUST expose the mode-specific image reference as a job-level `env: IMAGE_REF` variable, scoped only within the single `docker-build` job. It MUST NOT expose `IMAGE_REF` as a `workflow_call` output.

#### Scenario: Smoke and scan use the same reference

- GIVEN a successful local build in either mode
- WHEN the smoke-test or Trivy step runs
- THEN it can read the exact local image reference from `IMAGE_REF`.
