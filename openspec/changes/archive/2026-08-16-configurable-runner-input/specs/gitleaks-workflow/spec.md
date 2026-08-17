# Gitleaks Workflow Specification

## Purpose

Define caller-selectable runner placement without changing secret scanning.

## Requirements

### Requirement: Optional runner selection with token fallback

The workflow MUST expose `on.workflow_call.inputs.runner` as a non-required string defaulting to `ubuntu-latest`, and its sole job MUST use `${{ inputs.runner }}` in `runs-on`. The optional `gh_token` secret, `github.token` fallback, permissions, full-history checkout, action reference, and scan behavior MUST remain unchanged.

#### Scenario: Default or explicit runner and omitted token

- GIVEN a caller omits `runner` and may omit `gh_token`, or supplies a hosted runner string
- WHEN the workflow runs
- THEN placement resolves correctly and token omission still uses `github.token` while the existing full-history scan remains intact.
