# Design: Configurable Runner Input

## Technical Approach

Add one identical optional scalar input to each reusable workflow and change only that job's runner expression. Keep the library label-agnostic: omitted input resolves to `ubuntu-latest`; callers may provide a valid single-string selector. Extend README usage and boundaries, but do not modify consumers or infrastructure.

## Architecture Decisions

| Decision | Choice | Rejected alternative | Rationale |
|---|---|---|---|
| Runner API | `type: string`, `required: false`, `default: ubuntu-latest` in every `workflow_call.inputs` | Arrays, runner groups, or library-owned labels | Matches the requested contract and preserves omitted callers; valid only if the confirmed ARC selector works as one string. |
| Placement | Replace only `runs-on: ubuntu-latest` with `runs-on: ${{ inputs.runner }}` | Failover, retries, ARC/Kubernetes changes | Placement is the only behavior change; capacity and control-plane recovery are outside this library. |
| ARC readiness | Use the confirmed caller-owned `lab-runner` selector while keeping reusable workflow YAML generic | Embedding `self-hosted`, `lab-runner`, or another label in library workflows | Records the confirmed single-string selector without coupling the library to caller infrastructure or claiming live ARC execution. |

## Data Flow

Caller `with.runner` (or default) → reusable input → sole job `runs-on` → existing steps and caller smoke command on that runner.

The exact YAML addition, repeated unchanged in all five files, is:

```yaml
runner:
  description: "Runner label or hosted runner name."
  type: string
  required: false
  default: ubuntu-latest
```

Insert it under `on.workflow_call.inputs`. In all five workflow files, replace only the existing job's `runs-on` value with `${{ inputs.runner }}`. Preserve existing inputs, permissions, secrets, action versions, dependencies, `push: false`, `load: true`, `SMOKE_TEST_COMMAND`, and Gitleaks token fallback.

## File Changes

| File | Action | Description |
|---|---|---|
| Five `.github/workflows/*.yml` files | Modify | Add the identical input and replace one `runs-on` expression per file. |
| `README.md` | Modify | Add runner contract, examples, prerequisites, security, and outage limits. |
| `openspec/changes/configurable-runner-input/design.md` | Create | This implementation and validation design. |
| `/home/jotaese/Projects/Rightsizing-tps-final/...` | No change | Consumer validation target only. |

README sections: runner selection; omitted/default hosted example; explicit `runner: ubuntu-latest`; **confirmed ARC** using the caller-owned `lab-runner` selector; repository runner-group access and required capabilities; Docker DinD/buildx and host `curl`; self-hosted trust/isolation/network/secret warnings; no automatic failover; and the fact that ARC cannot repair GitHub Actions scheduling/API/control-plane outages. State that no ARC label is embedded or provisioned in the reusable workflows.

## Interfaces / Contracts

The contract is `with.runner` on each reusable workflow. `rightsizing-tps-final/.github/workflows/ci.yml` remains nine calls: one Go build/test, one Go lint, three Hadolint, three Docker-build, and one Gitleaks. Validate Go `1.26.0`, three Dockerfile paths, `contents: read`, `needs`, `secrets.gh_token: ${{ secrets.GITHUB_TOKEN }}`, and external host-`curl` `/healthz` smoke commands. Its non-root distroless Dockerfiles require host tooling; no consumer modification is permitted.

## Testing Strategy

| Layer | What to test | Approach |
|---|---|---|
| Structural | Five identical inputs; five jobs use the input; no hardcoded ARC label | YAML parse plus type/default/required and exact `runs-on` assertions; allow the confirmed `lab-runner` example in documentation only. |
| Compatibility | Existing contracts | Compare snapshots for inputs/defaults/required flags, permissions, secrets, action refs, dependencies, Docker flags, smoke command, and Gitleaks fallback. Scratch callers cover omission and explicit `ubuntu-latest`; no ARC run is claimed. |
| Consumer | Nine calls and fixtures | Inspect checkout, Go `1.26.0`, three Dockerfiles, ports/host `curl`, needs/secrets/mapping; assert consumer diff and Dockerfile hashes are unchanged. |
| Workflow lint | Syntax and Actions semantics | Run `actionlint` when available. If unavailable, run an installed YAML parser plus the structural assertions and report that action-specific lint was unavailable. |

The platform owner has confirmed that `lab-runner` is an accessible ARC label usable as one `runs-on` string. The caller still owns repository runner-group access and actual runner readiness. The reusable workflow YAML MUST remain generic and MUST NOT embed `lab-runner`. No changes to ARC infrastructure, Nexus, consumer, `go-hadolint-poc`, registry behavior, or control plane.

## Threat Matrix

| Boundary | Applicability and response |
|---|---|
| Documentation-like paths | N/A — README is documentation only; no executable-file classification. |
| Git repository selection | N/A — no `git -C` or repository selection changes. |
| Commit state | N/A — no commit automation. |
| Push state | N/A — no push or refspec changes; Docker remains no-push. |
| PR commands | N/A — no PR automation. |

The runner/shell boundary is validated separately: caller commands remain data passed through `SMOKE_TEST_COMMAND`, execute only on the selected runner, and must not be interpolated into workflow YAML. RED coverage should fail on a missing input, non-string/default mismatch, altered existing contract, hardcoded ARC label, or changed consumer file.

## Migration / Rollout

No migration required. Omitted callers remain hosted-compatible. The confirmed selector is caller opt-in; rollback is reverting the five workflow and README changes.

## Open Questions

- [x] Platform owner: confirm `lab-runner` as an accessible ARC label and one-string `runs-on` usability.
- [x] Keep the selector caller-owned and claim no live ARC execution without a live run.
