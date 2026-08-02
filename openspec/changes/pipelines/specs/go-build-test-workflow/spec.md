# go-build-test-workflow Specification

## Purpose

Reusable `workflow_call` workflow (`go-build-test.yml`) that builds, vets, and tests a Go module for any consumer repository.

## Requirements

### Requirement: Input Contract

The workflow MUST declare `workflow_call` inputs `go-version` (string, pinned default) and `working-directory` (string, default `"."`).

#### Scenario: Defaults used

- GIVEN a caller invokes the workflow with no inputs
- WHEN the job runs
- THEN it sets up the pinned default `go-version` and uses `working-directory: "."`

#### Scenario: Override respected

- GIVEN a caller passes a custom `go-version`
- WHEN the job runs
- THEN the specified Go toolchain version is installed instead of the default

### Requirement: Build/Vet/Test Sequence

The workflow MUST run `go build ./...`, `go vet ./...`, then `go test ./...` in that order inside `working-directory`, and MUST fail the job on the first non-zero exit.

#### Scenario: All steps pass

- GIVEN valid Go source and passing tests
- WHEN the workflow runs
- THEN the job succeeds after all three steps complete

#### Scenario: A step fails

- GIVEN a failing test or vet violation
- WHEN the workflow runs
- THEN the job MUST fail and MUST NOT run subsequent unrelated steps as if it passed

### Requirement: Permissions Declaration

The workflow requires no elevated `permissions:` beyond `contents: read`. Callers MUST declare at least `contents: read` in the invoking job.

#### Scenario: Minimal permissions suffice

- GIVEN a caller job with `permissions: { contents: read }`
- WHEN the workflow runs
- THEN checkout and build succeed without permission errors
