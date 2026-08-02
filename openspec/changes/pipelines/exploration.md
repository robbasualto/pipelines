# Exploration: Reusable CI/CD workflow library

> Backfilled from Engram topic `sdd/pipelines/explore` (the explore phase agent had no write access).

## Current State

Source material is `go-hadolint-poc`, read in full (read-only; not modified).

`.github/workflows/ci.yml` (`push`, `pull_request`) — 5 jobs:

| Job | Content | Classification |
|---|---|---|
| `go-build-test` | checkout, `setup-go@v5` (go 1.26.5), `go build ./...`, `go vet ./...`, `go test ./... -cover` | Go-specific; only variable is go-version |
| `go-lint` | checkout, `setup-go@v5`, `gofmt -l .`, `golangci-lint-action@v9.3.0` (golangci-lint v2.12.2) | Go-specific; go-version + linter version |
| `hadolint` | checkout, `hadolint-action@v3.1.0` (`dockerfile: Dockerfile`) | Language-agnostic; only Dockerfile path |
| `docker-build` (`needs`: the 3 above) | checkout, `docker/build-push-action@v6` (context `.`, `push: false`, tag `go-hadolint-poc:ci`), smoke test asserting literal stdout `Hello, world!`, Trivy scan `aquasecurity/trivy-action@v0.36.0` report-only (`continue-on-error: true`, `exit-code: "0"`) | Generic structure, service-specific smoke assertion and hardcoded image name |
| `gitleaks` | checkout `fetch-depth: 0`, `gitleaks-action@v3.0.0` with `GITHUB_TOKEN` | Fully generic, no `needs` |

`.github/workflows/release.yml` (tags `v*`, `permissions: contents: write`) is a single sequential job duplicating ~80% of `ci.yml` inline, then `gh release create "$TAG" --generate-notes`. This duplication is the concrete driver for reuse.

`Dockerfile`: multi-stage `golang:1.26.5-bookworm` → `gcr.io/distroless/base-debian12:nonroot`, `ENTRYPOINT ["./server"]`. Confirms the smoke assertion is deterministic single-line stdout — inherently service-specific.

`pipelines` repo: empty except SDD scaffolding; `openspec/config.yaml` already declares hybrid store, no test runner, and rules for rollback plans, semver/ref strategy notes, RFC 2119 scenarios, per-workflow input/secret contracts, documented job dependency graphs, actionlint/yamllint as candidates, and "keep workflows minimal and composable; avoid embedding app-specific logic".

## Verified platform constraints (GitHub Actions, Aug 2026)

1. **`workflow_call` mechanics** — reusable workflows declare `on: workflow_call:` with `inputs`, `outputs`, `secrets`. Dedicated `workflow_call`-only files are the cleaner pattern for a library.
2. **Cross-repo access, private repos, same personal account** — a caller can `uses: robbasualto/pipelines/.github/workflows/<name>.yml@<ref>` only if the `pipelines` repo's Settings → Actions → General → Access is set to "Accessible from repositories owned by the user". This is a manual one-time repo setting, not expressible in YAML. Skipping it produces a runtime access error easy to misdiagnose as a syntax error.
3. **Nesting** — up to 10 total levels, no cycles. Supports an optional composed "meta" workflow later.
4. **Secrets** — `secrets: inherit` reaches a directly-called workflow only; nested levels must re-declare `inherit` or explicit mappings.
5. **Permissions** — `GITHUB_TOKEN` permissions can only be downgraded from caller to callee. A caller job omitting `permissions:` gives the callee restrictive defaults, which can fail validation before the run starts. Directly affects `contents: write` for `gh release create`.
6. **Composite actions vs reusable workflows** — composite actions are step-level (`inputs` only, no `needs`, no job-level `permissions`, no matrix). All 5 jobs are whole jobs with their own runner and dependency edges, so **full reusable workflows are the correct primitive**. A composite action may make sense later for a step-only concern (e.g. shared setup-go + cache).

## Approaches considered

1. **One reusable workflow per concern (recommended)** — granular files, consumers compose their own `ci.yml`/`release.yml`.
   - Pros: matches config.yaml composability rule; narrow independently versionable contracts; piecemeal adoption (a future non-Go service pulls only hadolint/docker-build/gitleaks); small blast radius.
   - Cons: consumers write more `uses:` wiring; the `needs:` graph is re-declared per consumer.
   - Effort: Medium.
2. **One monolithic parametrized `ci.yml` with boolean feature switches** — consumer calls once.
   - Pros: one-line consumer wiring.
   - Cons: violates the repo's own "composable / no app-specific logic" rules; forces every future stack into one job shape; input surface grows per adopted stack; large contract, harder partial rollback.
   - Effort: Low upfront, High long-term.
3. **Hybrid: granular + an optional composed orchestrator workflow** chaining them via nested `workflow_call`.
   - Pros: building blocks plus a low-friction common case.
   - Cons: extra artifact versioned in lockstep with its dependencies; more design surface for iteration one.
   - Effort: Medium-High.

**Recommendation**: Approach 1 now; defer Approach 3's composed wrapper until a second real consumer proves the common-case shape. Reject Approach 2.

## First-cut input/secret surface

- `go-build-test` / `go-lint`: `go-version` (required, no default — versions age out and services diverge), `golangci-lint-version` (pinned default), `working-directory` (default `.`). No secrets.
- `hadolint`: `dockerfile-path` (default `Dockerfile`). No secrets.
- `docker-build`: `image-name` (required, must not default to a hardcoded name), `context`/`dockerfile-path`, `image-tag`, smoke-test contract (the one genuinely service-specific behavior — must be caller-supplied, not baked in), Trivy severity/exit-code defaults preserving report-only behavior. No registry secrets while `push: false`.
- `gitleaks`: no required inputs; needs `GITHUB_TOKEN` via `secrets: inherit` or an explicit mapping (see `go-hadolint-poc` commit `5ecbb2c`).
- A consumer `release.yml` composes the same granular workflows plus a final `gh release create` job with explicit `permissions: contents: write`.

## Versioning / consumption strategy

| Strategy | Pros | Cons |
|---|---|---|
| `@main` | Zero friction, always latest | Any bad edit breaks every consumer instantly; no rollback boundary |
| `@v1` moving major tag (**recommended**) | Consumers control upgrade timing; standard convention; supports documented breaking-change process; satisfies config.yaml ref-strategy rule | Requires tagging discipline in `pipelines` |
| Full SHA pin | Maximum reproducibility/supply-chain safety | Poor ergonomics for a single-maintainer private repo; no untrusted publisher to defend against yet |

Recommendation: tag-pinned `@v1`; cut `v1.0.0` once validated against `go-hadolint-poc`; rollback = re-pin to the previous tag. `@main` acceptable only during first bring-up before `v1` exists.

## Risks surfaced

- Manual, non-YAML Actions access setting on `pipelines` is a hard prerequisite; must be an explicit task, not automatable by `sdd-apply`.
- Smoke-test genericization is the highest-risk design decision — real services may need multi-line output, HTTP health checks, exit-code-only checks, or nothing.
- `permissions:` propagation for the future release flow silently 403s if mis-declared; must be documented explicitly in design.
- Temptation to skip tagging discipline with a single consumer removes the rollback boundary.
- Action versions (`golangci-lint-action@v9.3.0` / v2.12.2, `hadolint-action@v3.1.0`, `trivy-action@v0.36.0`) are copied as-is; propose/design must decide pin-internally vs expose-as-input.

## Open questions left for the proposal

1. Combine `go-build-test` + `go-lint` into one reusable workflow, or keep two (affects GitHub Checks granularity)?
2. Does `docker-build` need registry push in v1, or is push deferred? (Recommendation: defer — no current job pushes.)

## Ready for Proposal

Yes — both source workflows read in full, platform constraints verified, decomposition and ref strategy recommended.
