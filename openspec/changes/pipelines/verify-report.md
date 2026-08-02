# Verify Report: pipelines (Reusable CI/CD Workflow Library v1)

**Date**: 2026-08-02
**Verdict**: PASS WITH WARNINGS
**Batch verified**: apply batch 1 of N (12/12 automatable tasks; 3 human-only tasks correctly left open)

## Artifact Availability

- `proposal.md`, `design.md`, all 6 spec files, `tasks.md` — read from OpenSpec files in `openspec/changes/pipelines/`.
- `apply-progress` (Engram id #95) — read in full via `mem_get_observation`.
- Implementation — read all 5 `.github/workflows/*.yml` files plus `README.md` directly from the repo tree.

## Task Completeness

12/12 automatable tasks checked `[x]` in `tasks.md` and match the code. 3 tasks correctly left unchecked `[ ]`:
- 1.2 (enable repo Settings → Actions → Access) — human-only, not automatable.
- 4.2 (scratch cross-repo `@main` invocation test) — human-only, blocked on 1.2.
- 4.3 (tag `v1.0.0` / move `v1`) — human-only, blocked on 4.2.

These are not falsely marked done. They correctly block archive but do not indicate incomplete automated work.

## Test/Build Evidence

Per `openspec/config.yaml`: `testing.test_runner: none` (infra-as-code repo, no application code, `apply.tdd: false`, `verify.test_command: ""`). No runtime test suite applies.

YAML syntax re-validated independently this phase (all 5 files, `ruby -ryaml -e "YAML.load_file(ARGV[0])"`):

| File | Result |
|---|---|
| go-build-test.yml | OK |
| go-lint.yml | OK |
| hadolint.yml | OK |
| docker-build.yml | OK |
| gitleaks.yml | OK |

Confirmed `actionlint`, `yamllint`, and Python `pyyaml` remain unavailable in this environment (matches apply-progress's account — not a newly-introduced gap).

Independently re-verified the apply agent's Decision-1 research claim by fetching `action.yaml` directly from the pinned tag:
```
curl https://raw.githubusercontent.com/aquasecurity/trivy-action/v0.36.0/action.yaml
```
Confirmed: declares `version` input (`description: 'Trivy version to use'`, `default: 'v0.70.0'`), and the step's `with:` block passes `version: ${{ inputs.version }}` through to the underlying setup-trivy action. The apply agent's claim is **verified, not merely asserted** — this was independently reproducible from this vantage point (network access was available), not "unverifiable."

## Numbered Check Results

### 1. All 5 workflows trigger only on `workflow_call`
PASS — every file's `on:` block contains exactly one key, `workflow_call`. No `push`/`pull_request` trigger anywhere in `.github/workflows/`. Confirmed by direct read of all 5 files (each file's `on:` block, e.g. go-build-test.yml:3-13, gitleaks.yml:3-8).

### 2. docker-build.yml contract
- `push: false` hardcoded, no push/registry/credentials input declared: PASS — `docker-build.yml:45` `push: false`; inputs are only `image-tag`, `context`, `dockerfile-path`, `smoke-test-command`, `trivy-version` (lines 5-25); no login step exists.
- `IMAGE_REF` is job-level `env:`, not a `workflow_call` output: PASS — `docker-build.yml:34-35` `env: IMAGE_REF: ${{ inputs.image-tag }}`; no `outputs:` block under `workflow_call:` (lines 3-25).
- Smoke command via `env:` + `bash -c "$SMOKE_TEST_COMMAND"`, never inline-interpolated into `run:`: PASS — `docker-build.yml:49-53`, exact literal match to design's Interfaces/Contracts snippet. No other occurrence of `smoke-test-command` inside a `run:` block.
- `smoke-test-command` defaults to `""`, step skipped when empty: PASS — `docker-build.yml:21` `default: ""`; `docker-build.yml:50` `if: inputs.smoke-test-command != ''`.
- `trivy-version` input exists and is wired to `aquasecurity/trivy-action@v0.36.0`'s `version:` input: PASS — `docker-build.yml:22-25` declares the input (default `v0.70.0`); `docker-build.yml:60` `version: ${{ inputs.trivy-version }}` inside the `with:` block for `uses: aquasecurity/trivy-action@v0.36.0`. Apply agent's claim independently reconfirmed live this phase (see Test/Build Evidence above) — not an unverifiable assertion.

### 3. hadolint.yml has NO `hadolint-version` input
PASS — `hadolint.yml:3-9` declares only `dockerfile-path`. Matches spec's explicit statement that this was dropped as infeasible (hadolint-action bakes its binary in).

### 4. gitleaks.yml optional secret + fallback + fetch-depth
PASS — `gitleaks.yml:5-8` declares secret `gh_token` (`required: false`, not `GITHUB_TOKEN`); `gitleaks.yml:18` `env: GITHUB_TOKEN: ${{ secrets.gh_token || github.token }}`; `gitleaks.yml:20-23` checkout step with `fetch-depth: 0`.

### 5. go-build-test.yml / go-lint.yml inputs
PASS — `go-version` present on both (`go-build-test.yml:6-9`, `go-lint.yml:6-9`). `golangci-lint-version` present on go-lint.yml (`go-lint.yml:10-13`) and wired at `go-lint.yml:50` `version: ${{ inputs.golangci-lint-version }}` inside `golangci/golangci-lint-action@v9.3.0`'s `with:` block. gofmt check present at `go-lint.yml:38-45` (`gofmt -l .`, fails on non-empty output).

### 6. Input naming: `image-tag` / `context`
PASS — `docker-build.yml:6-13` declares `image-tag` and `context`; no `image-name` or `build-context` anywhere in the repo. Matches reconciled spec (`docker-build-workflow/spec.md:11`) and design's Workflow Contracts table.

### 7. README.md documentation completeness
PASS — README documents all 5 workflows with input/default/description tables, secrets, and required caller permissions (README.md:62-207). Includes a fully worked go-hadolint-poc `ci.yml` consumption example with the complete `needs:` graph and per-job `permissions:` (README.md:209-263), explicitly pinned to `@main` since no `v1` tag exists yet, and explicitly states "documentation only — go-hadolint-poc is not modified by this change" (README.md:211-213). No output tables exist for any workflow, but this correctly reflects that none of the 5 workflows declare `workflow_call` outputs — `docker-build.yml` explicitly documents why `IMAGE_REF` is deliberately not an output (README.md:151-154).

### 8. go-hadolint-poc untouched
PASS — `git status --short` in `/home/jotaese/Projects/go-hadolint-poc` shows only `M .atl/.skill-registry.cache.json`, `M .atl/skill-registry.md` (unrelated `chore: refresh skill registry` housekeeping, already committed as `be5d66b` prior to this session) and an untracked `openspec/changes/nexus-docker-registry/` directory (a separate, unrelated SDD change in progress). Nothing in this working tree relates to the `pipelines` change.

### 9. Human-only tasks correctly left open
PASS — `tasks.md` lines 31, 52, 53 show 1.2, 4.2, 4.3 as `- [ ]` (unchecked), each explicitly annotated "HUMAN task" / not automatable. Not falsely marked done.

## Design Deviation Note (not in the numbered checklist, flagged proactively)

`docker-build.yml:47` adds `load: true` to the `docker/build-push-action@v6` step. This is not named in `design.md`'s Workflow Contracts table or Interfaces/Contracts snippet. Assessed as **WARNING, not CRITICAL**: without `load: true`, a buildx build with `push: false` leaves the image only in the builder's cache, not in the local Docker daemon — `IMAGE_REF` would then be unreachable by `docker run` in the smoke-test step, which would make the spec's "IMAGE_REF available within the job" scenario (docker-build-workflow spec.md:37-41) and "Non-empty command passes" scenario (spec.md:59-63) unsatisfiable in practice. The addition is necessary for the documented behavior to actually work, does not reintroduce any push/registry surface, and does not contradict any MUST in the spec — but `design.md` should be updated to name it explicitly so the design document stays a complete, accurate description of the shipped contract.

## Spec Compliance Matrix (summary)

| Capability | Requirements | Status |
|---|---|---|
| `go-build-test-workflow` | Input Contract, Build/Vet/Test Sequence, Permissions Declaration | PASS (3/3) |
| `go-lint-workflow` | Input Contract, Formatting and Lint Checks, Permissions Declaration | PASS (3/3) |
| `hadolint-workflow` | Input Contract, Lint Enforcement, Permissions Declaration | PASS (3/3) |
| `docker-build-workflow` | Input Contract, No Registry Push, Image Reference Availability, Optional Smoke Test, Report-Only Vulnerability Scan, Permissions Declaration | PASS (6/6) |
| `gitleaks-workflow` | Secrets Contract, Secret Detection Enforcement, Permissions Declaration | PASS (3/3) |
| `workflow-consumption-contract` | Versioned Reference, Repository Access Prerequisite, Explicit Permissions and Secrets Propagation, Overridable Tool Versions | PASS (4/4, documentation-level; runtime cross-repo scenarios remain unproven pending human tasks 1.2/4.2) |

Note per the skill's rule ("a spec scenario is compliant only when a covering test passed at runtime"): most scenarios above are structurally/statically compliant by inspection, consistent with this repo's declared `testing.test_runner: none` and infra-as-code nature — there is no code path to unit-test. The `workflow-consumption-contract` scenarios that require a real cross-repo GitHub Actions run (Access Prerequisite, Versioned Reference resolution) are explicitly unprovable until human tasks 1.2 and 4.2 are done; this is a known, correctly-flagged gap, not a defect.

## Issues

**CRITICAL**: None.

**WARNING**:
1. `docker-build.yml`'s `load: true` on the build-push-action step is undocumented in `design.md` (see Design Deviation Note above). Functionally necessary and spec-compliant, but the design artifact should be updated for accuracy.
2. `workflow-consumption-contract` runtime scenarios (cross-repo resolution, Access Prerequisite behavior) remain unproven — correctly blocked on human tasks 1.2/4.2, not a code defect.

**SUGGESTION**:
1. Consider installing `actionlint` (via a package manager, not committed to the repo) for a stronger schema-level check than plain YAML syntax parsing, before the scratch cross-repo test (task 4.2).

## Overall Verdict

**PASS WITH WARNINGS.**

Code, spec, and documentation are aligned for all 5 workflows and the consumption contract. All 7 architecture decisions in `design.md` are implemented as decided, with one undocumented-but-correct addition (`load: true`). All 9 specifically-requested checks pass. No CRITICAL issues.

**Archive readiness**: NOT ready to archive yet — by design. The 3 human-only follow-up tasks (1.2 repo Access setting, 4.2 scratch cross-repo test, 4.3 `v1.0.0`/`v1` tagging) remain open and are prerequisites the orchestrator/user must complete outside this automated pipeline before `sdd-archive` runs. This verify report evaluates documentation/spec/code alignment as requested; it does not assert the change is otherwise archive-ready.
