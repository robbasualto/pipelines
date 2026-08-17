# Go Lint Workflow Specification

## Purpose

Define caller-selectable runner placement without changing linting behavior.

## Requirements

### Requirement: Optional runner selection

The workflow MUST expose `on.workflow_call.inputs.runner` as a non-required string defaulting to `ubuntu-latest`, and its sole job MUST use `${{ inputs.runner }}` in `runs-on`. Existing Go, golangci-lint, directory, permission, action, and pass/fail contracts MUST remain unchanged.

#### Scenario: Default and explicit placement

- GIVEN a caller omits `runner` or supplies a hosted runner string
- WHEN the reusable workflow is invoked
- THEN the job resolves to `ubuntu-latest` or the supplied string respectively, with formatting and golangci-lint checks unchanged.
