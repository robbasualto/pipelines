# Hadolint Workflow Specification

## Purpose

Define caller-selectable runner placement without changing Dockerfile linting.

## Requirements

### Requirement: Optional runner selection

The workflow MUST expose `on.workflow_call.inputs.runner` as a non-required string defaulting to `ubuntu-latest`, and its sole job MUST use `${{ inputs.runner }}` in `runs-on`. The Dockerfile input, permissions, action reference, and lint behavior MUST remain unchanged.

#### Scenario: Default and explicit placement

- GIVEN a caller omits `runner` or supplies a hosted runner string
- WHEN the reusable workflow is invoked
- THEN the job resolves to `ubuntu-latest` or the supplied string respectively, and the requested Dockerfile is linted as before.
