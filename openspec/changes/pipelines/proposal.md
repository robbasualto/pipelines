# Proposal: Reusable CI/CD workflow library (v1)

## Intent

`go-hadolint-poc` holds all CI inline and `release.yml` re-implements ~80% of `ci.yml`. Every future service would copy the same five jobs. Create the first `workflow_call` library in `pipelines` so services consume versioned pipelines instead of duplicating them.

## Scope

### In Scope
- Five reusable workflows: `go-build-test.yml`, `go-lint.yml`, `hadolint.yml`, `docker-build.yml`, `gitleaks.yml`.
- Documented input/secret/permissions contract per workflow.
- Consumption + versioning docs (`@v1` moving major tag) and repo README.
- Manual prerequisite: Actions access "Accessible from repositories owned by the user" on `pipelines`.

### Out of Scope
- Registry push and registry credentials (v2).
- Migrating `go-hadolint-poc` (follow-up change, after `v1.0.0`).
- A composed "meta" orchestrator workflow (wait for a second consumer).
- actionlint/yamllint self-CI for this repo.

## Decisions (exploration open questions)

1. **Split, not combined**: `go-build-test` and `go-lint` stay separate — preserves per-check GitHub UI granularity and narrow input surfaces; accepts duplicated checkout/setup-go.
2. **No push in v1**: `docker-build.yml` is build + optional smoke test + report-only Trivy, `push: false`, and ships no stub `push` input.
3. **Smoke test = optional caller command**: input `smoke-test-command` (default `""`, step skipped when empty); image ref exported as `IMAGE_REF`; pass/fail is exit code only, no stdout assertion in the library. Consumer passes `test "$(docker run --rm "$IMAGE_REF")" = "Hello, world!"`.
4. **Action versions are inputs with pinned defaults** (golangci-lint, hadolint, Trivy).

## Capabilities

### New Capabilities
- `go-build-test-workflow`: Go build/vet/test contract.
- `go-lint-workflow`: gofmt + golangci-lint contract.
- `hadolint-workflow`: Dockerfile lint contract.
- `docker-build-workflow`: image build, optional smoke test, report-only scan.
- `gitleaks-workflow`: secret-scan contract.
- `workflow-consumption-contract`: ref/versioning strategy, cross-repo access, secrets/permissions propagation.

### Modified Capabilities
- None.

## Approach

Granular one-workflow-per-concern (exploration Approach 1), each `workflow_call`-only, with no app-specific logic. Consumers compose their own `ci.yml`/`release.yml`, re-declaring the `needs:` graph and `permissions:` explicitly.

## Affected Areas

| Area | Impact | Description |
|---|---|---|
| `.github/workflows/*.yml` | New | Five reusable workflows |
| `README.md` | New | Usage + versioning docs |
| GitHub repo settings | Modified | Actions access scope (manual) |
| `go-hadolint-poc` | Unchanged | Migrated later |

## Risks

| Risk | Likelihood | Mitigation |
|---|---|---|
| Access setting forgotten → opaque runtime failure | High | Prerequisite task + README troubleshooting |
| Smoke contract too narrow for future services | Med | Optional, exit-code-only, caller-owned |
| `permissions:` mis-propagation → silent 403 | Med | Document required caller `permissions:` |
| Consumers drift to `@main` | Med | `@v1` documented as only supported ref |

## Rollback Plan

Nothing consumes these workflows yet: v1 rollback is reverting the files, zero consumer impact. After tagging, consumers roll back by re-pinning `@v1` to the previous tag; the moving `v1` tag is never force-moved onto a breaking change.

## Dependencies

- Manual one-time Actions access setting on `pipelines`.
- `go-hadolint-poc` as validation consumer before tagging `v1.0.0`.

## Success Criteria

- [ ] Five `workflow_call` workflows with documented inputs/secrets/permissions.
- [ ] Each invocable from a private repo owned by the same account.
- [ ] No hardcoded service name, image name, or stdout assertion in the library.
- [ ] Versioning/rollback documented; `v1.0.0` tagged after validation.

## Proposal question round

Executor cannot prompt the user directly. Assumptions needing confirmation:
1. Per-check UI granularity worth duplicated setup-go cost (decision 1)?
2. Exit-code-only smoke testing acceptable, or must v1 support HTTP health checks (decision 3)?
3. Ship `gitleaks.yml` in v1, or own secret scanning centrally later?
4. `@v1` moving tag acceptable, or is SHA pinning required by policy?
