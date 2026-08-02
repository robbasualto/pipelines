# gitleaks-workflow Specification

## Purpose

Reusable `workflow_call` workflow (`gitleaks.yml`) scanning repository content for committed secrets.

## Requirements

### Requirement: Secrets Contract

The workflow MUST declare an OPTIONAL `workflow_call` secret input `gh_token` (`required: false`). The workflow MUST internally resolve the token used by the scan action as `${{ secrets.gh_token || github.token }}`, so a caller that omits `gh_token` and/or omits `secrets: inherit` still receives a working default token rather than a hard failure.

#### Scenario: Caller omits gh_token on a pull_request event

- GIVEN a caller invokes the workflow on a `pull_request` event without passing `gh_token` and without `secrets: inherit`
- WHEN the workflow runs
- THEN it MUST still succeed, using the default `github.token` to resolve the PR diff

#### Scenario: Caller passes an explicit gh_token

- GIVEN a caller passes an explicit `gh_token` secret
- WHEN the workflow runs on a `pull_request` event
- THEN the scan uses the supplied `gh_token` to resolve the PR diff and completes

### Requirement: Secret Detection Enforcement

The workflow MUST fail the job when gitleaks detects one or more findings in scanned content.

#### Scenario: Clean history passes

- GIVEN no secrets are present in the scanned range
- WHEN the workflow runs
- THEN the job succeeds

#### Scenario: Secret detected fails the job

- GIVEN a matching secret pattern exists in the scanned range
- WHEN the workflow runs
- THEN the job MUST fail

### Requirement: Permissions Declaration

Callers MUST declare `contents: read` at minimum; on `pull_request` events using `secrets: inherit`, the caller's default `GITHUB_TOKEN` permissions apply as scoped by the caller workflow.

#### Scenario: Minimal permissions suffice for push scans

- GIVEN a caller job with `permissions: { contents: read }` on a `push` event
- WHEN the workflow runs
- THEN the scan completes without permission errors
