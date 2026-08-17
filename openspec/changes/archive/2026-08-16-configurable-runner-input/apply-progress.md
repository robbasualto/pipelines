# Apply Progress: Configurable Runner Input

**Batch**: Corrective rerun 2; cumulative progress from apply batch 1 retained
**Mode**: Standard (`strict_tdd: false`; infra-as-code repository with no application test runner)
**Delivery**: Single reviewable work unit; forecast is below the 800-line session budget and no chain is required.

## Correction Scope

- Reconciled proposal, exploration, design, and workflow-consumption specification wording with the confirmed `lab-runner` platform decision and its single-string `runs-on` usability.
- Preserved the architectural boundary: `lab-runner` is caller-owned and appears only in documentation/examples; all five reusable workflow YAML files remain generic with `default: ubuntu-latest` and `${{ inputs.runner }}`.
- Corrected the Gitleaks README input table to include `runner` while retaining the optional `gh_token` and `github.token` fallback behavior.
- No consumer checkout, ARC infrastructure, Nexus, `go-hadolint-poc`, registry behavior, or control-plane logic was changed.

## Completed Tasks

- [x] 1.1 Platform decision confirmed: `lab-runner` is the caller-owned ARC selector and is usable as the requested single string. Repository runner-group access remains a caller prerequisite and was documented; no live ARC execution was claimed.
- [x] 1.2 Ran the temporary inline structural/compatibility checker against all five workflows, README requirements, and consumer snapshots.
- [x] 2.1 Added the identical optional `runner` input (`string`, `required: false`, `default: ubuntu-latest`) and changed only each job's `runs-on` to `${{ inputs.runner }}`.
- [x] 2.2 Documented omission compatibility, explicit `ubuntu-latest`, confirmed `lab-runner`, caller-owned access and single-string limits, runner capabilities, DinD/buildx, host `curl`, security boundaries, no failover, and GitHub Actions control-plane limits.
- [x] 3.1 Parsed all five workflow YAML files with Ruby Psych and ran structural assertions. `actionlint` was unavailable in the environment.
- [x] 3.2 Verified preserved workflow contracts and README requirements; the checker covered all pre-existing inputs/defaults/required flags, permissions, action references, dependencies, Docker flags, smoke command handling, and Gitleaks fallback. Omitted and explicit hosted runner scenarios were structurally covered by the default and `runner: ubuntu-latest` contract assertions.
- [x] 3.3 Validated the read-only consumer checkout: nine reusable calls, Go `1.26.0`, three Dockerfile paths, `contents: read`, Docker `needs`, explicit Gitleaks token mapping, and host-`curl` `/healthz` smoke commands. Docker/buildx and host `curl` were available; all three local image builds and health checks passed. The local Docker daemon is available, and the existing `github-runner` container exposed a nested Docker daemon (`docker info` server `29.2.1`); no live ARC runner/control-plane path was available.
- [x] 3.4 Recorded exact focused checks, runtime results, consumer hashes/status, scope, and rollback boundary. No consumer files were edited, and no commit or push was performed.

## Files Changed

| File | Action | What Was Done |
|---|---|---|
| `.github/workflows/go-build-test.yml` | Modified | Added `runner` input and dynamic job runner. |
| `.github/workflows/go-lint.yml` | Modified | Added `runner` input and dynamic job runner. |
| `.github/workflows/hadolint.yml` | Modified | Added `runner` input and dynamic job runner. |
| `.github/workflows/docker-build.yml` | Modified | Added `runner` input and dynamic job runner; preserved local load/no-push behavior. |
| `.github/workflows/gitleaks.yml` | Modified | Added `runner` input and dynamic job runner; preserved token fallback. |
| `README.md` | Modified | Added runner contract, hosted/ARC usage, prerequisites, security, and outage boundaries. |
| `openspec/changes/configurable-runner-input/tasks.md` | Modified | Marked all apply tasks complete and recorded the confirmed platform decision. |
| `openspec/changes/configurable-runner-input/proposal.md` | Modified | Reconciled the confirmed selector and caller-owned/generic workflow boundary. |
| `openspec/changes/configurable-runner-input/exploration.md` | Modified | Reconciled current state, recommendation, validation, and risk wording. |
| `openspec/changes/configurable-runner-input/design.md` | Modified | Recorded confirmed ARC readiness without embedding the selector in workflows. |
| `openspec/changes/configurable-runner-input/specs/workflow-consumption-contract/spec.md` | Modified | Replaced the stale selector gate with the confirmed caller contract. |
| `openspec/changes/configurable-runner-input/apply-progress.md` | Modified | Merged prior evidence with this corrective rerun and its focused checks. |

## Work Unit Evidence

| Evidence | Exact result |
|---|---|
| Focused test command and exact result | `ruby - <<'RUBY' ...` structural checker: **PASS** — five YAML contracts parsed, five exact dynamic runners, preserved input snapshots, README boundaries, and no workflow ARC label. `git diff --check -- .github/workflows README.md`: **PASS**. |
| Corrective artifact-alignment check | Corrected `ruby - <<'RUBY' ...` structural/compatibility checker: **PASS** — all five workflow contracts parsed, runner defaults and expressions verified, pre-existing contracts preserved, README Gitleaks `runner` row and token fallback verified, and no workflow embeds `lab-runner`. |
| Workflow lint command and exact result | `actionlint .github/workflows/*.yml`: unavailable (`command not found`). Ruby Psych `5.2.2` parser plus structural assertions: **PASS**. |
| Consumer validation command and exact result | Read-only Ruby consumer checker: **PASS** — 9 calls (1 Go build/test, 1 Go lint, 3 Hadolint, 3 Docker-build, 1 Gitleaks), Go `1.26.0`, three Dockerfiles, `contents: read`, `needs`, token mapping, host curls; all 9 can pass `runner: lab-runner`. `git -C /home/jotaese/Projects/Rightsizing-tps-final diff --exit-code -- .github/workflows/ci.yml docker/checkout/Dockerfile docker/inventory/Dockerfile docker/payment/Dockerfile`: exit 0. Consumer remains unchanged; its pre-existing staged `A .github/workflows/ci.yml` status is retained. |
| Runtime harness command and exact result | Prior valid evidence retained: `docker buildx build --load ...` plus `docker run --publish 18081/18082/18083 ...` and host `curl --fail .../healthz`: **PASS** for checkout, inventory, and payment. This corrective scope is SDD/README-only, so Docker execution was not repeated. Local Docker daemon, buildx `0.36.1`, host curl `8.21.0`, and nested `github-runner` Docker server `29.2.1` were available. Live ARC scheduling/API/control-plane execution was unavailable and is not claimed. |
| Consumer fixture hashes | `ci.yml` `e7043eeb975988ad484204ff8289fdf12adc2982fe2591efe1ee4d83af685a71`; checkout `2058de29595600ae3ce5e707d5fdc92bac5a82414f35cb52722b14f61ccf5075`; inventory `eb0b8bc2382ca0a67103f2a72d66df8eab5b09e0187569b6879ed9a4d587d47d`; payment `757f0cd9f0c664e41cc1dd50248426c1ce694c7f180ba1cc33fa69ec1e468baa`; hashes were identical before and after apply. Consumer status retained pre-existing staged `A .github/workflows/ci.yml`; apply did not edit the checkout. |
| Rollback boundary | For this correction, revert only the reconciled proposal, exploration, design, workflow-consumption spec, and Gitleaks README table edits; reusable workflow implementation and consumer files remain untouched. Full feature rollback remains limited to the five `.github/workflows/{go-build-test,go-lint,hadolint,docker-build,gitleaks}.yml` runner-input changes, the runner-selection README section, and this change's SDD artifacts. |

## Deviations and Limitations

- Earlier SDD artifacts contained stale selector wording. The confirmed platform decision permits the clearly labeled caller-owned `lab-runner` example in README; no workflow embeds that label and no consumer code was added.
- The Gitleaks README table now lists the shared `runner` input and still documents the optional `gh_token` with its `github.token` fallback.
- `actionlint` and a Python YAML package were unavailable; Ruby Psych and exact structural assertions were used instead.
- No GitHub-hosted, ARC, or GitHub Actions control-plane execution was possible from this local environment. Local Docker/DinD capability and consumer smoke behavior were validated without fabricating ARC success.

## Status

All 8 tasks complete. Ready for `sdd-verify`.
