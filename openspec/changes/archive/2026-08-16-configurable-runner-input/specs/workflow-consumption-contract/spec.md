# Workflow Consumption Contract Specification

## Purpose

Define documentation, consumer validation, and the boundary for hosted or self-hosted runner selection.

## Requirements

### Requirement: Runner documentation and prerequisites

README MUST document the shared optional string input and `ubuntu-latest` default, hosted examples, and the confirmed caller-owned `lab-runner` ARC selector. It MUST state repository runner-group access, required runner capabilities, and that Docker-build callers provide DinD/buildx and host `curl`; the library provisions none and embeds no ARC selector.

#### Scenario: Consumer chooses infrastructure

- GIVEN a caller uses the default, a known hosted value, or the confirmed ARC value
- WHEN it configures `with.runner`
- THEN the README explains the resulting prerequisites and presents `lab-runner` only as the confirmed caller-owned selector, not as a library default.

### Requirement: Security and outage boundaries

README MUST warn that workflow code and caller-provided smoke commands execute on the selected self-hosted machine, requiring trust, isolation, network, and secret controls. It MUST state there is no automatic runner failover and ARC cannot repair GitHub Actions scheduling/API/control-plane outages.

#### Scenario: Runner unavailable

- GIVEN a selected runner is unavailable or the GitHub Actions control plane is impaired
- WHEN the workflow is requested
- THEN no alternate runner or repair path is implied or created by this change.

### Requirement: Consumer fact validation without consumer changes

Validation MUST inspect local `rightsizing-tps-final` facts for the `rightsizing-tps` consumer and its nine calls: one Go build/test, one Go lint, three Hadolint, three Docker-build, and one Gitleaks. It MUST verify Go `1.26.0`, three Dockerfiles, host-`curl` smoke commands, `contents: read`, needs/secrets, and Gitleaks token mapping; Dockerfiles and consumer workflow MUST NOT be modified.

#### Scenario: Existing consumer remains equivalent

- GIVEN the current local consumer and its three non-root distroless Dockerfiles
- WHEN validation compares the call graph and facts
- THEN all nine sites and required mappings remain accounted for, with zero consumer-file changes.

### Requirement: Confirmed ARC selector boundary

The platform owner has confirmed `lab-runner` as an accessible ARC label usable as one `runs-on` string. The selector MUST remain caller-owned: reusable workflow YAML MUST stay generic and MUST NOT embed `lab-runner`. Repository runner-group access and actual ARC readiness remain caller prerequisites. ARC/Kubernetes, Nexus, GitLab CI, DRP replacement, registry push, `go-hadolint-poc`, consumer-specific logic, image/port/service changes, and failover remain excluded.

#### Scenario: Caller selects the confirmed ARC runner

- GIVEN a caller has access to the `lab-runner` runner group
- WHEN it supplies `with.runner: lab-runner`
- THEN the reusable workflow receives one valid runner string, while the library remains generic and no live ARC execution is implied.
