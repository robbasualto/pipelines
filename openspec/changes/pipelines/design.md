# Design: Reusable CI/CD workflow library (v1)

## Technical Approach

Five `workflow_call`-only files in this repo's `.github/workflows/`, each a single job on `ubuntu-latest` with `timeout-minutes: 15`, minimal job-level `permissions`, and no app-specific logic. No `push`/`pull_request` triggers here — this repo has no code to CI. Consumers own the `needs:` graph and their own `permissions:`.

## Workflow Contracts

| File | Inputs (default) | Secrets | Permissions | Steps |
|---|---|---|---|---|
| `go-build-test.yml` | `go-version` (`"1.26.5"`), `working-directory` (`.`) | none | `contents: read` | checkout@v4, setup-go@v5, `go build/vet ./...`, `go test ./... -cover` |
| `go-lint.yml` | `go-version` (`"1.26.5"`), `golangci-lint-version` (`v2.12.2`), `working-directory` (`.`) | none | `contents: read` | checkout, setup-go, gofmt check, golangci-lint-action@v9.3.0 (`with.version`) |
| `hadolint.yml` | `dockerfile-path` (`Dockerfile`) | none | `contents: read` | checkout, hadolint-action@v3.1.0 |
| `docker-build.yml` | `image-tag` (required), `context` (`.`), `dockerfile-path` (`Dockerfile`), `smoke-test-command` (`""`) | none | `contents: read` | checkout, build-push-action@v6 (`push: false`), optional smoke, trivy-action@v0.36.0 (`exit-code: "0"`, `continue-on-error: true`) |
| `gitleaks.yml` | none | `gh_token` (optional) | `contents: read` | checkout (`fetch-depth: 0`), gitleaks-action@v3.0.0 |

## Architecture Decisions

| # | Decision | Choice | Rejected | Rationale |
|---|---|---|---|---|
| 1 | Tool-version inputs | Only where the wrapped action exposes a version input: `golangci-lint-version` yes; **`hadolint-version` and `trivy-version` dropped** | Version input per action | Step-level `uses:` cannot be expression-interpolated, so action refs are literal. hadolint-action bakes its hadolint image; trivy-action's `version` input is unverified. Bumping those = library release. |
| 2 | `IMAGE_REF` export | Job-level `env: IMAGE_REF: ${{ inputs.image-tag }}` | Step output; `$GITHUB_ENV`; `workflow_call` output | Job-level env is the only form visible inside the caller-supplied shell string; the image is never pushed, so no other job can consume it — a workflow output would be a lie. |
| 3 | Smoke-command injection | Pass via `env:`, run `bash -c "$SMOKE_TEST_COMMAND"` | Inline `${{ inputs.smoke-test-command }}` in `run:` | Inline interpolation executes at template-expansion time; env indirection keeps caller input as data. |
| 4 | gitleaks token | `secrets: gh_token: required: false`, resolved `GITHUB_TOKEN: ${{ secrets.gh_token \|\| github.token }}` | Require `secrets: inherit` | `github.token` always exists in the called workflow, so a caller forgetting `inherit` cannot reproduce go-hadolint-poc bug `5ecbb2c`. |
| 5 | Release/tagging | **Manual**: annotated `vX.Y.Z`, then `git tag -f v1 && git push origin v1 --force` after validation | Release automation in this repo | Automation would need self-CI (explicit non-goal) and would force-move `v1` without human validation; single maintainer, rare releases. Revisit at second consumer. |
| 6 | Consumption example | Yes — worked go-hadolint-poc example in `README.md`, documentation only | No example / editing go-hadolint-poc | The manual access prerequisite fails opaquely; a concrete snippet is the cheapest diagnosis aid. go-hadolint-poc is NOT modified by this change. |
| 7 | `runs-on` | Hardcoded `ubuntu-latest` | `runner` input | No self-hosted fleet; unvalidated surface. |
| 8 | `build-push-action` image visibility | `load: true` alongside `push: false` | `push: false` alone | With `push: false` only, the built image stays in the buildx cache, invisible to the `docker run` smoke-test step. `load: true` loads it into the local Docker daemon so `IMAGE_REF` (decision #2) is actually runnable. |

## Data Flow

    consumer ci.yml (push / pull_request)
      ├─ uses go-build-test.yml@v1 ─┐
      ├─ uses go-lint.yml@v1        ├─ needs ─→ uses docker-build.yml@v1
      ├─ uses hadolint.yml@v1 ──────┘              (image-tag, smoke-test-command)
      └─ uses gitleaks.yml@v1  (independent, no needs)

## File Changes

| File | Action | Description |
|---|---|---|
| `.github/workflows/go-build-test.yml` | Create | Go build/vet/test |
| `.github/workflows/go-lint.yml` | Create | gofmt + golangci-lint |
| `.github/workflows/hadolint.yml` | Create | Dockerfile lint |
| `.github/workflows/docker-build.yml` | Create | Build, optional smoke, report-only Trivy |
| `.github/workflows/gitleaks.yml` | Create | Secret scan |
| `README.md` | Create | Per-workflow input/secret/permissions tables, `@v1` policy, manual tagging steps, consumer example, access-setting troubleshooting |

## Interfaces / Contracts

```yaml
# docker-build.yml — injection-safe optional smoke step
- name: Smoke test
  if: inputs.smoke-test-command != ''
  env:
    SMOKE_TEST_COMMAND: ${{ inputs.smoke-test-command }}
  run: bash -c "$SMOKE_TEST_COMMAND"
```

## Testing Strategy

| Layer | What | Approach |
|---|---|---|
| Static | YAML/schema validity | `actionlint` locally before commit (config.yaml apply rule) |
| Integration | Cross-repo invocability | Temporary caller workflow on a scratch branch after the access setting is enabled |
| Acceptance | Contract completeness | Manual review against spec scenarios; no test runner in this repo |

## Threat Matrix

| Boundary | Applicability | Design response | Planned RED test |
|---|---|---|---|
| Caller-supplied shell (`smoke-test-command`) | Applicable | Env indirection (Decision 3); empty ⇒ step skipped; failure ⇒ non-zero exit fails the job | Caller value containing `"` / `$(...)` / `}}` must be executed as data, never expanded |
| Documentation-like paths | N/A | No file classification or execution routing |
| Git repository selection | N/A | `actions/checkout` only; no `git -C` |
| Commit state | N/A | Library never commits |
| Push state | Applicable (docs only) | Tag push is a documented manual step (Decision 5), never automated | None — no code path |
| PR commands | N/A | No `gh`/PR automation in v1 |

## Migration / Rollout

1. ~~Manual prerequisite: Settings → Actions → General → Access → "Accessible from repositories owned by the user".~~ **Superseded during verification**: this repo's default branch is `master`, not `main` (inherited from local `init.defaultBranch`) — fixed below. More importantly, the `access_level: user` policy only permits **private-to-private** sharing per GitHub's own REST API docs ("access allows sharing across user owned private repositories only"). `go-hadolint-poc`, the first real consumer, is **public** — a public caller does not qualify under that policy regardless of the setting. Cross-repo invocation failed instantly (`referenced_workflows: []`) until `pipelines` itself was made **public**, which requires no Access setting at all. Decision: keep `pipelines` public going forward; the private+access-level path only applies if every future consumer is also private.
2. Workflows live on `master` (this repo's actual default branch); validate with a scratch caller using `@master`, not `@main`. (Confirmed working: a disposable branch in `go-hadolint-poc` calling `hadolint.yml@master` succeeded end-to-end.)
3. Tag `v1.0.0`, then create moving `v1`. `@master` remains valid after tagging (v1 is additive, not a replacement); `@master` should still be avoided by consumers post-v1 for stability, but is not "unsupported."
4. Rollback: re-point `v1` to the previous tag; never force-move `v1` onto a breaking change.

## Open Questions

- [x] Confirm during apply whether `trivy-action@v0.36.0` exposes a `version` input — **confirmed yes** (fetched `action.yaml` from the pinned tag directly), `trivy-version` is implemented as an input in `docker-build.yml`.
- [ ] Proposal question round (smoke scope, gitleaks ownership, `@v1` vs SHA pinning) remains unanswered; design assumes the proposal's defaults.
