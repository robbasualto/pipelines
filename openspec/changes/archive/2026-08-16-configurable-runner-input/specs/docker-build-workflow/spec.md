# Docker Build Workflow Specification

## Purpose

Define caller-selectable runner placement while preserving local image validation.

## Requirements

### Requirement: Optional runner selection with unchanged Docker contract

The workflow MUST expose `on.workflow_call.inputs.runner` as a non-required string defaulting to `ubuntu-latest`, and its sole job MUST use `${{ inputs.runner }}` in `runs-on`. It MUST retain required `image-tag`, all existing inputs, `load: true`, `push: false`, caller `smoke-test-command`, report-only Trivy behavior, permissions, and action references.

#### Scenario: Hosted default or explicit runner

- GIVEN a caller omits `runner` or supplies a hosted runner string
- WHEN the workflow builds an image
- THEN placement defaults or overrides as specified, the image is locally loaded, no registry push occurs, and any supplied smoke test runs unchanged.
