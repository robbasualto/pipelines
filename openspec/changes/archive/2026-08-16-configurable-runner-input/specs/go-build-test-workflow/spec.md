# Go Build Test Workflow Specification

## Purpose

Define caller-selectable runner placement without changing the Go build/test contract.

## Requirements

### Requirement: Optional runner selection

The workflow MUST expose `on.workflow_call.inputs.runner` as a non-required string defaulting to `ubuntu-latest`, and its sole job MUST use `${{ inputs.runner }}` in `runs-on`. Existing inputs, permissions, checkout, setup, build, vet, and test behavior MUST remain unchanged.

#### Scenario: Default and explicit placement

- GIVEN a caller omits `runner` or supplies a hosted runner string
- WHEN the reusable workflow is invoked
- THEN the job resolves to `ubuntu-latest` or the supplied string respectively, with the existing Go checks unchanged.
