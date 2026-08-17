# Archive Report: Configurable Runner Input

**Change**: `configurable-runner-input`
**Archived**: 2026-08-16
**Artifact store**: OpenSpec + Engram
**Status**: Archived with warnings

## Executive Summary

The completed `configurable-runner-input` change is archived after delta synchronization and verification. Six delta specs were created as the initial OpenSpec source-of-truth specs, and the complete change folder was moved to `openspec/changes/archive/2026-08-16-configurable-runner-input/`.

All five reusable workflows remain generic: each exposes an optional string `runner` input defaulting to `ubuntu-latest` and uses `${{ inputs.runner }}` for its job runner. The confirmed `lab-runner` selector is caller-owned and documented only as a single-string example; no ARC label is embedded in workflow implementation.

## OpenSpec Synchronization

No pre-existing domain specs existed beyond `openspec/specs/.gitkeep`. The following six delta specs were copied as initial main specs without destructive replacement:

| Domain | Action | Result |
|---|---|---|
| `go-build-test-workflow` | Created | Optional runner selection with existing Go behavior preserved |
| `go-lint-workflow` | Created | Optional runner selection with existing lint behavior preserved |
| `hadolint-workflow` | Created | Optional runner selection with existing Dockerfile lint behavior preserved |
| `docker-build-workflow` | Created | Optional runner selection with `load: true`, `push: false`, smoke tests, and Trivy behavior preserved |
| `gitleaks-workflow` | Created | Optional runner selection with `gh_token` and `github.token` fallback preserved |
| `workflow-consumption-contract` | Created | Documentation, consumer validation, security, caller-owned selector, and outage boundaries |

## Archived Contents

- `exploration.md`
- `proposal.md`
- `specs/` with six delta specs
- `design.md`
- `tasks.md` — 8/8 implementation tasks checked
- `apply-progress.md`
- `verify-report.md`
- `archive-report.md`

The active change directory no longer exists. The archived task artifact contains no unchecked implementation tasks.

## Final Verification State

- Native SDD verification: **PASS WITH WARNINGS**.
- Requirements: **9/9**.
- Scenarios: **9/9**.
- Blockers: **0**.
- Critical findings: **0**.
- All five reusable workflows expose `runner: string`, `required: false`, `default: ubuntu-latest`, and exact `${{ inputs.runner }}` placement.
- README includes hosted and confirmed caller-owned `lab-runner` examples, DinD/buildx and host `curl` prerequisites, self-hosted security warnings, no automatic failover, and the GitHub Actions control-plane limitation.
- Consumer validation passed for nine calls, Go `1.26.0`, three Dockerfiles, permissions/secrets, external host-curl smoke tests, and local Docker/buildx smoke tests. Consumer files remained unchanged.

## Review Authority and Lineage

Native bounded review was approved for the frozen candidate under lineage `review-4c2435351313eba5`. Four review lenses completed with no findings, and the verification evidence passed.

A separate pre-commit gate validation returned `invalidated` because terminal receipt discovery observed unrelated-target scope. It is **not** recorded as pre-commit gate approval, was not reopened or rerun, and no commit is being made in this archive operation.

## Warnings and Limitations

- `actionlint` was unavailable; Ruby Psych YAML parsing and structural/compatibility assertions passed instead. This is a warning, not a workflow-aware lint pass.
- Live GitHub-hosted, ARC, and GitHub Actions control-plane execution was unavailable. Local Docker/buildx and host-curl behavior passed, but no live ARC execution is claimed.
- The consumer checkout retains its pre-existing staged `A .github/workflows/ci.yml` status; the validated consumer paths were unchanged by this work.
- No commit, push, or PR was performed.

## Engram Traceability

The archive report is persisted under `sdd/configurable-runner-input/archive-report`.

| Artifact | Engram observation |
|---|---:|
| `explore` | `#261` |
| `proposal` | `#262` |
| `spec` | `#263` |
| `design` | `#264` |
| `tasks` | `#266` |
| `apply-progress` | `#268` |
| `verify-report` | `#276` |
| Native review lineage | `review-4c2435351313eba5` (final-state review authority) |
