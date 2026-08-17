# Proposal: Configurable Runner Input

## Intent

Let reusable workflows run on hosted or caller-selected infrastructure. Omitted `runner` remains `ubuntu-latest`.

## Scope

### In Scope
- Add identical optional string input `runner` (default `ubuntu-latest`) to all five workflows and use it for `runs-on`.
- Preserve existing contracts, Docker `load: true`/no-push, and caller smoke tests.
- Document hosted/ARC examples, label requirements, DinD, host `curl`, self-hosted security, no failover, and control-plane limits.
- Validate `rightsizing-tps-final` facts without changing that consumer or Dockerfiles.

### Out of Scope
- ARC/Kubernetes, Nexus, GitLab CI, DRP/control-plane replacement, registry push, `go-hadolint-poc`, commits, pushes, library-owned labels, consumer-specific logic, image names, ports, service names, or failover.

## Capabilities

### New Capabilities
- None; this is a backward-compatible modification.

### Modified Capabilities
- `go-build-test-workflow`, `go-lint-workflow`, `hadolint-workflow`: runner selection without changing checks.
- `docker-build-workflow`: runner selection; preserve local load, smoke tests, report-only Trivy, and no push.
- `gitleaks-workflow`: runner selection; preserve optional `gh_token` fallback.
- `workflow-consumption-contract`: propagation, prerequisites, security, refs, outage boundaries.

## Approach

Use the scalar input and keep the library label-agnostic. The platform owner has confirmed `lab-runner` as an accessible single-string ARC selector for caller use. README may show that confirmed caller-owned example, while the reusable workflow YAML remains generic and embeds no ARC label.

## Affected Areas

| Area | Impact | Description |
|---|---|---|
| `.github/workflows/*.yml` | Modified | Five contracts and runner expressions |
| `README.md` | Modified | Usage, prerequisites, security, limitations |
| `rightsizing-tps-final` | Unchanged | Validation target only |

## Risks

| Risk | Likelihood | Mitigation |
|---|---|---|
| Caller ARC access or runner readiness is incomplete | Med | The caller must provide repository runner-group access and a usable `lab-runner` selector with the required capabilities |
| ARC Docker jobs lack DinD/buildx or host `curl` | Med | Verify actual runner prerequisites |
| Self-hosted commands expose host resources | Med | Document trust, isolation, network, and secret controls |
| Outage is mistaken for failover | Med | Document no fallback or outage repair |

## Rollback Plan

Revert the five workflow and README changes. Omitted inputs remain compatible; released consumers can pin the prior ref/tag.

## Dependencies

- Confirmed platform decision: `lab-runner` is usable as the caller's single-string ARC selector; repository runner-group access remains a caller prerequisite.
- Consumer-side DinD/buildx and host `curl`; this change provisions neither.

## Success Criteria

- [x] All five workflows declare the optional string input and use it for `runs-on`.
- [x] Existing contracts and Docker behavior remain unchanged.
- [x] README covers usage, prerequisites, security, and outage boundaries without embedding a library-owned label.
- [x] Consumer facts and nine call sites are validated without changes.
- [x] ARC label gate is confirmed before apply; no live ARC execution is claimed.

## Proposal question round

Confirmed: `lab-runner` is the caller-owned ARC selector, usable as one string; caller access and security/DinD prerequisites remain required.
