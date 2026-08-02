# go-lint-workflow Specification

## Purpose

Reusable `workflow_call` workflow (`go-lint.yml`) enforcing Go formatting and static-analysis standards.

## Requirements

### Requirement: Input Contract

The workflow MUST declare `workflow_call` inputs `golangci-lint-version` (pinned default, overridable) and `working-directory` (default `"."`).

#### Scenario: Consumer bumps tool version without a new release

- GIVEN a caller passes a newer `golangci-lint-version`
- WHEN the workflow runs
- THEN it installs the specified version instead of the pinned default, with no change required in `pipelines`

### Requirement: Formatting and Lint Checks

The workflow MUST run `gofmt -l` and MUST fail if it reports any file, then MUST run `golangci-lint run` at the pinned/overridden version and MUST fail on any reported issue.

#### Scenario: Clean code passes

- GIVEN correctly formatted, lint-clean Go source
- WHEN the workflow runs
- THEN both checks pass and the job succeeds

#### Scenario: Formatting violation fails the job

- GIVEN a file not matching `gofmt` output
- WHEN the workflow runs
- THEN the job MUST fail before or independent of the `golangci-lint` step result

### Requirement: Permissions Declaration

The workflow requires no elevated permissions beyond `contents: read`. Callers MUST declare `contents: read`.

#### Scenario: Minimal permissions suffice

- GIVEN a caller job with `permissions: { contents: read }`
- WHEN the workflow runs
- THEN linting completes without permission errors
