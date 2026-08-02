# hadolint-workflow Specification

## Purpose

Reusable `workflow_call` workflow (`hadolint.yml`) linting a Dockerfile for best-practice violations.

## Requirements

### Requirement: Input Contract

The workflow MUST declare `workflow_call` input `dockerfile-path` (default `"Dockerfile"`). Hadolint's version is NOT a caller-facing input: `hadolint-action` bakes its hadolint binary into the action itself, so the only pinnable surface is the action reference (`hadolint/hadolint-action@vX`) declared in the workflow file — an implementation detail, not part of the input contract. Bumping the effective hadolint version requires a `pipelines` release.

#### Scenario: Default dockerfile-path used

- GIVEN a caller omits `dockerfile-path`
- WHEN the workflow runs
- THEN it lints `Dockerfile` at the repository root

### Requirement: Lint Enforcement

The workflow MUST lint the file at `dockerfile-path` and MUST fail the job if hadolint reports any error-level finding.

#### Scenario: Clean Dockerfile passes

- GIVEN a Dockerfile with no error-level findings
- WHEN the workflow runs
- THEN the job succeeds

#### Scenario: Violation fails the job

- GIVEN a Dockerfile with an error-level hadolint finding
- WHEN the workflow runs
- THEN the job MUST fail and report the finding

### Requirement: Permissions Declaration

The workflow requires no elevated permissions beyond `contents: read`. Callers MUST declare `contents: read`.

#### Scenario: Minimal permissions suffice

- GIVEN a caller job with `permissions: { contents: read }`
- WHEN the workflow runs
- THEN the lint step completes without permission errors
