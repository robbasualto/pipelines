# Tasks: Reusable CI/CD Workflow Library (v1)

## Review Workload Forecast

Note: this session's review budget is 800 lines (not the skill default 400); risk below is assessed against 800.

| Field | Value |
|-------|-------|
| Estimated changed lines | ~330-430 (5 new workflow YAMLs ~180-210 + README ~150-220) |
| 800-line budget risk | Low |
| Chained PRs recommended | No |
| Suggested split | Single PR |
| Delivery strategy | not provided this session — default `ask-on-risk` assumed |
| Chain strategy | pending |

Decision needed before apply: No
Chained PRs recommended: No
Chain strategy: pending
400-line budget risk: Low

### Suggested Work Units

| Unit | Goal | Likely PR | Focused test command | Runtime harness | Rollback boundary |
|------|------|-----------|----------------------|-----------------|-------------------|
| 1 | 5 reusable workflow files + README, single PR | PR 1 | `actionlint .github/workflows/*.yml` (or `yamllint`) | Scratch caller workflow at `@main` after the human access-setting task (Phase 1.2) | Revert PR; zero consumers exist yet, no external impact |

## Phase 1: Research & Prerequisites

- [x] 1.1 Research whether `aquasecurity/trivy-action@v0.36.0` exposes a `version:` input (check action.yml/release notes); result gates whether 2.4 declares `trivy-version`. [docker-build-workflow: Input Contract]
  - **Result**: CONFIRMED SUPPORTED. Fetched `action.yaml` directly from the `v0.36.0` tag (`https://raw.githubusercontent.com/aquasecurity/trivy-action/v0.36.0/action.yaml`) — it declares a `version` input (`description: 'Trivy version to use'`, `default: 'v0.70.0'`). `docker-build.yml` therefore declares `trivy-version` (default `"v0.70.0"`) and passes it through as the action's `version:` input.
- [ ] 1.2 HUMAN task (not automatable by apply): enable Settings → Actions → General → Access → "Accessible from repositories owned by the user" on `robbasualto/pipelines`. [workflow-consumption-contract: Repository Access Prerequisite]

## Phase 2: Reusable Workflow Files (2.1-2.3, 2.5 parallel; 2.4 waits on 1.1)

- [x] 2.1 Create `.github/workflows/go-build-test.yml`: inputs `go-version` (`"1.26.5"`), `working-directory` (`"."`), `contents: read`; checkout@v4, setup-go@v5, sequential `go build/vet/test ./...`, fail on first non-zero exit. [go-build-test-workflow]
- [x] 2.2 Create `.github/workflows/go-lint.yml`: inputs `golangci-lint-version` (`v2.12.2`), `working-directory`, `contents: read`; `gofmt -l` (fail if any file listed) then golangci-lint-action@v9.3.0. [go-lint-workflow]
- [x] 2.3 Create `.github/workflows/hadolint.yml`: input `dockerfile-path` (`"Dockerfile"`), no version input, `contents: read`; hadolint-action@v3.1.0. [hadolint-workflow]
- [x] 2.4 Create `.github/workflows/docker-build.yml`: inputs `image-tag` (required), `context`, `dockerfile-path`, `smoke-test-command` (`""`), plus `trivy-version` only if 1.1 confirms support; `contents: read`; checkout, build-push-action@v6 (`push: false`, no login/push step), job-level `env: IMAGE_REF: ${{ inputs.image-tag }}`, env-indirected optional smoke step (`if: inputs.smoke-test-command != ''`, `run: bash -c "$SMOKE_TEST_COMMAND"`), trivy-action@v0.36.0 report-only (`exit-code: "0"`, `continue-on-error: true`). [docker-build-workflow]
- [x] 2.5 Create `.github/workflows/gitleaks.yml`: optional secret `gh_token` (`required: false`) resolved as `GITHUB_TOKEN: ${{ secrets.gh_token || github.token }}`, `contents: read`; checkout (`fetch-depth: 0`), gitleaks-action@v3.0.0. [gitleaks-workflow]

## Phase 3: Documentation (after Phase 2 files exist)

- [x] 3.1 Write `README.md` per-workflow input/secret/permissions tables (source: design's Workflow Contracts table). [workflow-consumption-contract]
- [x] 3.2 Document `@v1` moving-tag policy plus manual tagging steps (annotated `vX.Y.Z`, then `git tag -f v1 && git push origin v1 --force`). [workflow-consumption-contract: Versioned Reference]
- [x] 3.3 Add worked go-hadolint-poc consumer example: `ci.yml` snippet with the full `needs:` graph from design's Data Flow and explicit `permissions:` per invoked workflow. [workflow-consumption-contract: Explicit Permissions and Secrets Propagation]
- [x] 3.4 Add Access-setting troubleshooting section referencing 1.2 and the opaque workflow-not-found/permission-failure scenario. [workflow-consumption-contract: Repository Access Prerequisite]

## Phase 4: Verification (sequential)

- [x] 4.1 Validate YAML syntax of all 5 workflow files locally (`actionlint` if available, else `yamllint` or `python -c "import yaml, sys; yaml.safe_load(open(sys.argv[1]))"` per file).
  - **Result**: Neither `actionlint` nor `yamllint` nor Python's `pyyaml` module was installed in this environment (checked first, none installed — no new tooling installed per constraint). Used Ruby's stdlib `YAML` (already present at `/usr/bin/ruby`) as the local syntax-validation fallback: `ruby -ryaml -e "YAML.load_file(ARGV[0])"` per file. All 5 workflow files parsed successfully (`OK: .github/workflows/{docker-build,gitleaks,go-build-test,go-lint,hadolint}.yml`).
- [ ] 4.2 HUMAN task, after 1.2 is done: push a scratch caller workflow (e.g. in go-hadolint-poc or a scratch branch) referencing these workflows at `@main` to prove cross-repo invocability. Cannot be proven before 1.2.
- [ ] 4.3 HUMAN task, after 4.2 passes: tag `v1.0.0`, then force-create the moving `v1` tag (Decision 5). Out of `sdd-apply`'s automated scope.
