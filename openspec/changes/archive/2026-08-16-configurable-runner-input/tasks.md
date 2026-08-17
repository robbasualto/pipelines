# Tasks: Configurable Runner Input

## Review Workload Forecast

| Field | Value |
|---|---|
| Estimated changed lines | ~130–180 implementation lines (five YAML files plus README) |
| 400-line budget risk | Low |
| Chained PRs recommended | No |
| Suggested split | Single PR; keep the two work units as reviewable commits |
| Delivery strategy | ask-on-risk |
| Chain strategy | pending |

Decision needed before apply: No
Chained PRs recommended: No
Chain strategy: pending
400-line budget risk: Low

Below both budgets; no chaining decision is needed. The ARC gate was resolved by the confirmed `lab-runner` platform decision.

## Gate and Scope

The platform owner confirmed `lab-runner` as the ARC selector and its use as one `runs-on` string. The caller still owns repository runner-group access and actual ARC readiness; this change documents that prerequisite and does not perform a live ARC run. Exclude ARC infrastructure, Nexus, consumer edits, `go-hadolint-poc`, registry pushes, and control-plane replacement.

## Suggested Work Units

| Unit | Goal | Focused test command | Runtime harness | Rollback boundary |
|---|---|---|---|---|
| 1 | Add runner contracts and documentation | `actionlint .github/workflows/*.yml` plus structural checker | Scratch callers: omitted `runner` and explicit `ubuntu-latest`; no ARC run before gate | Revert the five workflow files and `README.md` only |
| 2 | Prove consumer compatibility without edits | Assertions plus `git -C /home/jotaese/Projects/Rightsizing-tps-final diff --exit-code` on consumer paths | Three Docker builds with DinD/buildx and host `curl` polling `/healthz` on ports 18081–18083 | Discard evidence; consumer files remain untouched |

## Phase 1: Gate and RED Baseline

- [x] 1.1 Confirm `lab-runner`, repository runner-group access, and single-string usability; the resolved decision permits the caller-owned selector in `README.md` while keeping workflows generic.
- [x] 1.2 After 1.1, prepare a temporary/out-of-tree RED checker for `.github/workflows/{go-build-test,go-lint,hadolint,docker-build,gitleaks}.yml`, `README.md`, and consumer snapshots; fail on missing/non-string input, changed contracts, hardcoded labels, or consumer edits.

## Phase 2: Implementation

- [x] 2.1 After 1.1, add identical optional `runner` (`required: false`, default `ubuntu-latest`) and exact `${{ inputs.runner }}` `runs-on` to the five paths; preserve inputs, permissions, actions, dependencies, secrets, Docker `load: true`/`push: false`, smoke command, and Gitleaks fallback.
- [x] 2.2 After 1.1, update `README.md` with hosted/default examples, confirmed `lab-runner` ARC example, access/labels, DinD/buildx, host `curl`, self-hosted trust/isolation/network/secret warnings, no failover, and control-plane limits.

## Phase 3: Verification and Evidence

- [x] 3.1 After 1.1, parse all five YAML files and run `actionlint`; if unavailable, run an installed YAML parser plus the structural checker and record the limitation.
- [x] 3.2 After 1.1, run compatibility assertions and both scratch caller scenarios; verify every original input/default/required flag, permissions, action/ref, dependency, Docker behavior, smoke data handling, and Gitleaks token fallback.
- [x] 3.3 After 1.1, inspect `/home/jotaese/Projects/Rightsizing-tps-final/.github/workflows/ci.yml` for all nine calls, Go `1.26.0`, `contents: read`, `needs`, `secrets.gh_token: ${{ secrets.GITHUB_TOKEN }}`, three Docker paths, and host-`curl` `/healthz` smoke commands; verify unchanged `/home/jotaese/Projects/Rightsizing-tps-final/docker/{checkout,inventory,payment}/Dockerfile` hashes and run all three smokes when prerequisites exist.
- [x] 3.4 After 1.1, record acceptance evidence, exact paths/diff, rollback boundary, and clean consumer status; do not commit or push. Acceptance requires five identical contracts, zero excluded-scope changes, and no unverified ARC execution claim.
