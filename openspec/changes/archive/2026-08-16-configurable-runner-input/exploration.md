## Exploration: Configurable runner input for reusable workflows

### Current State

Before this change, the `pipelines` repository was an infrastructure-as-code library whose five `workflow_call`-only workflows each contained one job, set `timeout-minutes: 15`, declared `contents: read`, and hardcoded `runs-on: ubuntu-latest`:

| Workflow | Existing contract | Runner-dependent behavior |
|---|---|---|
| `.github/workflows/go-build-test.yml` | `go-version`, `working-directory`; no secrets | Checkout, setup Go, build, vet, and test execute on the selected runner. |
| `.github/workflows/go-lint.yml` | `go-version`, `golangci-lint-version`, `working-directory`; no secrets | Checkout, setup Go, `gofmt`, and golangci-lint execute on the selected runner. |
| `.github/workflows/hadolint.yml` | `dockerfile-path`; no secrets | Checkout and the Hadolint action execute on the selected runner. |
| `.github/workflows/docker-build.yml` | Required `image-tag`; `context`, `dockerfile-path`, `smoke-test-command`, `trivy-version`; no secrets | `docker/build-push-action` uses `load: true`; the caller smoke command runs on the runner and may require Docker plus host `curl`. No push occurs. |
| `.github/workflows/gitleaks.yml` | Optional `runner` input; optional `gh_token` secret, falling back to `github.token` | Full-history checkout and Gitleaks execute on the selected runner. |

The backward-compatible contract is to add the same optional `runner` input to every workflow, with `type: string`, `required: false`, and default `ubuntu-latest`, then use `${{ inputs.runner }}` for that workflow's `runs-on`. GitHub's reusable-workflow syntax supports string `workflow_call` inputs and dynamic runner selection. Omitting the input therefore preserves today's behavior; supplying it changes only job placement. Existing inputs, permissions, secrets, action versions, job dependencies, and pass/fail behavior should remain unchanged.

The local consumer checkout is available at `/home/jotaese/Projects/Rightsizing-tps-final`. Its current `.github/workflows/ci.yml` invokes the library nine times: Go build/test, Go lint, three Hadolint jobs, three Docker-build jobs, and Gitleaks. It uses Go `1.26.0`, the three paths `docker/{checkout,inventory,payment}/Dockerfile`, `contents: read` on every caller job, and an explicit `GITHUB_TOKEN` to `gh_token` mapping for Gitleaks. Each Docker job passes an external HTTP smoke-test command that starts the image, publishes a unique port (`18081`–`18083`), and polls `/healthz` with host `curl`. The three Dockerfiles use static, non-root distroless runtime images, so `curl` must remain outside the image.

The implementation README documents the shared runner input, confirmed `lab-runner` caller example, ARC/self-hosted requirements, Docker-in-Docker, host `curl`, self-hosted security, and the limitation that ARC cannot repair GitHub Actions control-plane outages.

### Affected Areas

- `.github/workflows/go-build-test.yml` — add the optional runner input and replace the hardcoded job runner.
- `.github/workflows/go-lint.yml` — add the same runner contract without changing Go or lint inputs.
- `.github/workflows/hadolint.yml` — add the same runner contract without changing Dockerfile selection.
- `.github/workflows/docker-build.yml` — add the same runner contract; preserve local image loading, caller smoke commands, report-only Trivy behavior, and no-push behavior.
- `.github/workflows/gitleaks.yml` — add the runner input while preserving the optional `gh_token` secret and fallback behavior.
- `README.md` — document the default GitHub-hosted path, the confirmed caller-owned `lab-runner` ARC example, required runner labels/capabilities, Docker-in-Docker and host `curl`, security warnings, and the non-failover/control-plane boundary.
- `/home/jotaese/Projects/Rightsizing-tps-final/.github/workflows/ci.yml` — validation and downstream adoption target, not a file to modify during this exploration. An ARC rollout would need the same confirmed runner value on all nine reusable-workflow calls.
- `/home/jotaese/Projects/Rightsizing-tps-final/docker/{checkout,inventory,payment}/Dockerfile` — consumer fixtures proving that Docker builds and external host-level HTTP smoke tests are required; no changes are needed for this library contract.

### Approaches

1. **Per-workflow optional string selector (recommended)** — declare `runner` in each `workflow_call` contract and use it directly as that job's `runs-on` value.
   - Pros: exactly matches the requested API; preserves omitted-input callers; keeps the library label-agnostic; supports GitHub-hosted and a confirmed ARC label without embedding infrastructure policy; minimal change surface.
   - Cons: the input can express only one string selector, not an array of labels or a runner-group object; the caller must repeat the value for every job it wants on ARC.
   - Effort: Low.

2. **Library-owned ARC routing or a structured multi-label selector** — hardcode an ARC label, or expand the input into arrays/groups and add fallback logic.
   - Pros: could centralize routing or express richer GitHub runner constraints.
   - Cons: invents/couples the library to infrastructure that has not been confirmed; breaks the requested string contract; increases consumer and validation complexity; automatic failover is not reliable for queued or control-plane failures and is explicitly out of scope.
   - Effort: Medium/High.

### Recommendation

Use Approach 1. Add the identical optional string input to all five workflow contracts and keep `ubuntu-latest` as the only library default. The confirmed `lab-runner` selector belongs in caller documentation/examples only; do not hardcode it, `self-hosted`, or any other ARC label in the reusable workflow YAML.

The confirmed ARC selector is `lab-runner`, and the platform decision confirms it is usable as one `runs-on` string. It belongs to the consumer/platform boundary: the caller must retain repository runner-group access and pass the confirmed value through `with.runner` on each intended call. The reusable library remains label-agnostic and embeds no ARC label in workflow YAML. The scalar contract still cannot encode multiple required labels or a runner-group object; if a future runner setup requires either, the API decision must be revisited rather than silently approximated.

No automatic failover should be designed or implied. A called job selects one runner before execution; this change does not add a second job, retry controller, watchdog, or fallback path. ARC provides execution capacity only and does not repair GitHub Actions scheduling/API/control-plane outages.

#### Consumer and README contract considerations

- A GitHub-hosted caller can omit `runner` and inherit `ubuntu-latest`; an explicit `runner: ubuntu-latest` example may demonstrate the override without asserting an ARC label.
- A self-hosted/ARC caller must provide the confirmed `lab-runner` selector, which must exist and be accessible to the repository. The reusable workflow YAML must remain generic; the caller owns the selector and its runner-group access.
- All jobs placed on the self-hosted selector must meet the action/tool prerequisites. In particular, each `docker-build` job needs a usable Docker daemon/buildx path (including the required privileged Docker-in-Docker setup where ARC provides DinD) and host `curl` for the consumer's external `/healthz` smoke tests. This change does not provision DinD, Docker, or `curl`.
- Self-hosted warnings must state that workflow code and caller-provided `smoke-test-command` execute on the selected machine. The consumer must control repository/PR trust, runner isolation, network exposure, and secret access; ARC is not a security boundary supplied by this library.
- The README must retain the existing `contents: read` and Gitleaks secret guidance. Adding `runner` must not alter permissions or secret propagation.

#### Validation strategy

1. Parse/lint all five workflow YAML files with available workflow-aware tooling (prefer `actionlint`; use a YAML parser as a secondary check).
2. Assert structurally for every workflow that `workflow_call.inputs.runner` is a non-required string defaulting to `ubuntu-latest`, and that the sole job uses the input in `runs-on`.
3. Diff-check that all pre-existing inputs, defaults, required flags, permissions, secrets, action references, dependencies, and Docker `push: false`/`load: true` behavior are unchanged.
4. Run a scratch caller that omits `runner` to cover the compatibility path and a separate caller that passes an ordinary known hosted value. Do not claim live ARC execution without a live ARC run.
5. Validate the consumer workflow's nine call sites: preserve Go `1.26.0`, all three Dockerfile paths, `needs` ordering, external host-curl smoke tests, `contents: read`, and the explicit Gitleaks token mapping. A caller adoption check must pass the confirmed selector to every intended call and verify Docker-in-Docker plus host `curl` on the actual runner.
6. Check README examples and warnings against the final input contract, including the confirmed selector example, no-failover statement, control-plane limitation, and security boundary.

### Risks

- **Caller ARC readiness:** confirmation of `lab-runner` as a single-string selector does not provision runner-group access, DinD/buildx, host `curl`, or live ARC execution.
- **Single-string limitation:** a scalar input cannot encode multiple required labels or a runner-group object. A future runner setup that requires either must trigger an API review rather than silent approximation.
- **Docker runner readiness:** an ARC runner without a reachable DinD Docker daemon/buildx setup or host `curl` will fail the consumer's Docker jobs even though the reusable workflow is syntactically valid.
- **Self-hosted execution risk:** caller code and smoke-test commands run with the permissions and network access of the selected machine; documentation cannot make an untrusted self-hosted runner safe.
- **No failover guarantee:** runner capacity loss, queueing, or a GitHub Actions control-plane outage is not repaired by this input and must not be presented as automatic fallback.
- **Documentation drift:** five workflow contracts and multiple consumer call sites must remain aligned; a missing `runner` on one call can silently leave a mixed hosted/ARC pipeline.

### Ready for Proposal

Yes. The recommended change is well-bounded and backward-compatible, and the platform decision confirms `lab-runner` as a usable single-string selector. No code or consumer files were modified during exploration.
