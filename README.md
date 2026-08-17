# pipelines

Reusable GitHub Actions CI/CD workflow library for repositories owned by
`robbasualto`. This repository contains no application code — it exposes
`workflow_call`-only workflows that consumer repositories invoke from their
own `ci.yml`.

Every workflow is a single job that defaults to `ubuntu-latest`, with
`timeout-minutes: 15`, minimal job-level `permissions`, and no app-specific
logic. `permissions:`
are **not** inherited from the caller — each consumer job must declare the
permissions block documented below for every workflow it invokes.

## Runner selection

Each reusable workflow exposes the same optional `runner` input:

| Input | Type | Required | Default | Description |
|---|---|---|---|---|
| `runner` | `string` | No | `ubuntu-latest` | A hosted runner name or caller-owned self-hosted/ARC selector |

Omitting `runner` is backward-compatible and keeps the existing
`ubuntu-latest` behavior. An explicit hosted runner selection is also
supported:

```yaml
jobs:
  go-build-test:
    permissions:
      contents: read
    uses: robbasualto/pipelines/.github/workflows/go-build-test.yml@v1.0.1
    with:
      runner: ubuntu-latest
```

For the confirmed ARC integration, a caller may pass the single-string
selector `lab-runner`:

```yaml
with:
  runner: lab-runner
```

`lab-runner` is caller-owned, not a library default or embedded workflow
label. The caller's repository must have access to the corresponding ARC
runner group and label. The selector must resolve to a runner with the
capabilities required by every action and command in the called workflow.
This input accepts one string only; it cannot express an array of labels or a
runner-group object, so the caller must provide a selector that is valid for
the target runner setup and repeat it on each reusable-workflow call that
should use that runner.

Self-hosted and ARC runners must provide the action/runtime prerequisites
themselves. In particular, each `docker-build.yml` job needs a reachable
Docker daemon and buildx; where ARC uses Docker-in-Docker, the caller's
runner setup must provide the required privileged DinD arrangement. Consumer
smoke tests that call an external service also need host `curl`; the
distroless images do not provide it. This library provisions neither ARC,
DinD, Docker, `curl`, nor any runner or other infrastructure.

Treat self-hosted execution as a security boundary owned by the caller:
workflow code and caller-provided commands such as `smoke-test-command` run
on the selected machine. Use trusted workflow and pull-request sources,
adequate runner isolation, controlled network access, and least-privilege
secret exposure. ARC does not make untrusted code safe, and this library does
not add those controls.

There is no automatic GitHub-hosted-to-ARC failover. A selected runner that
is unavailable does not cause this library to choose another infrastructure
class. ARC supplies runner capacity but cannot recover GitHub Actions
scheduling, API, or control-plane outages.

## Repository Visibility

This repository is **public**, deliberately. GitHub's private-repo
`Settings → Actions → General → Access` policy only allows `access_level:
user` to grant **"sharing across user owned private repositories only"**
(per GitHub's REST API docs) — it does not cover a public caller invoking a
private repo's reusable workflows. `go-hadolint-poc`, the first real
consumer, is public, so keeping `pipelines` private would silently block it
(instant failure, `workflow was not found`, no job ever created). Making
`pipelines` public removes the access check entirely: any repository,
regardless of owner or visibility, can invoke these workflows.

If a future consumer set is private-only, private + the `user` access level
is a valid alternative — but the moment any public repo needs to call in,
`pipelines` itself must be public too.

## Versioning

Consumers reference workflows as:

```yaml
uses: robbasualto/pipelines/.github/workflows/<name>.yml@v1.0.1
```

`v1.0.1` is the current immutable release and the first published release
containing the configurable `runner` input. Pin it when reproducibility and
an explicit release boundary matter.

`@v1` is a moving major tag that only ever advances to non-breaking releases.
It is never force-moved onto a change that alters an existing input's meaning,
removes an input/output, or changes pass/fail semantics. A breaking change
ships under a new major tag (e.g. `@v2`); `@v1` keeps pointing at the last
compatible release. This README does not assume that `@v1` has advanced to
`v1.0.1`; moving a major tag requires an explicit maintainer release action.

### Manual tagging steps (maintainer only)

1. Validate the change (YAML syntax + scratch caller invocation).
2. Create an annotated release tag:
   ```bash
   git tag -a vX.Y.Z -m "vX.Y.Z"
   git push origin vX.Y.Z
   ```
3. As a separate, explicit maintainer release action, optionally move the
   moving major tag onto it (only for non-breaking releases). Do not perform
   this step merely because an immutable release was published:
   ```bash
   git tag -f v1 vX.Y.Z
   git push origin v1 --force
   ```
4. Rollback: re-point `v1` at the previous compatible tag. Never force-move
   `v1` onto a breaking change.

## Workflows

### `go-build-test.yml`

Builds, vets, and tests a Go module.

| Input | Default | Description |
|---|---|---|
| `go-version` | `"1.26.5"` | Go toolchain version to install |
| `working-directory` | `"."` | Directory containing the Go module |

- Secrets: none
- Permissions required by the caller: `contents: read`
- Steps: `actions/checkout@v4` → `actions/setup-go@v5` → `go build ./...` →
  `go vet ./...` → `go test ./... -cover`. Fails on the first non-zero exit.

```yaml
jobs:
  go-build-test:
    permissions:
      contents: read
    uses: robbasualto/pipelines/.github/workflows/go-build-test.yml@v1.0.1
```

### `go-lint.yml`

Enforces Go formatting and static-analysis standards.

| Input | Default | Description |
|---|---|---|
| `go-version` | `"1.26.5"` | Go toolchain version to install |
| `golangci-lint-version` | `"v2.12.2"` | golangci-lint version, overridable per-call |
| `working-directory` | `"."` | Directory containing the Go module |

- Secrets: none
- Permissions required by the caller: `contents: read`
- Steps: checkout, setup-go, `gofmt -l` (fails if any file is listed),
  `golangci/golangci-lint-action@v9.3.0` at the pinned/overridden version.

```yaml
jobs:
  go-lint:
    permissions:
      contents: read
    uses: robbasualto/pipelines/.github/workflows/go-lint.yml@v1.0.1
    with:
      golangci-lint-version: v2.13.0 # optional override, no pipelines release needed
```

### `hadolint.yml`

Lints a Dockerfile for best-practice violations.

| Input | Default | Description |
|---|---|---|
| `dockerfile-path` | `"Dockerfile"` | Path to the Dockerfile to lint |

- Secrets: none
- Permissions required by the caller: `contents: read`
- Steps: checkout, `hadolint/hadolint-action@v3.1.0`.
- No version input: `hadolint-action` bakes its hadolint binary into the
  action itself, so the only pinnable surface is the action reference in
  this repository. Bumping the effective hadolint version requires a
  `pipelines` release.

```yaml
jobs:
  hadolint:
    permissions:
      contents: read
    uses: robbasualto/pipelines/.github/workflows/hadolint.yml@v1.0.1
```

### `docker-build.yml`

Builds a container image locally, optionally smoke-tests it, and reports
(non-blocking) vulnerability scan results. **Never pushes to any registry.**

| Input | Default | Description |
|---|---|---|
| `image-tag` | *(required)* | Tag applied to the locally built image; also exposed as the job-level `IMAGE_REF` env var |
| `context` | `"."` | Docker build context |
| `dockerfile-path` | `"Dockerfile"` | Path to the Dockerfile to build |
| `smoke-test-command` | `""` | Optional shell command to smoke-test the built image. Skipped entirely when empty |
| `trivy-version` | `"v0.70.0"` | Trivy version for the report-only scan (see note below) |

- Secrets: none
- Permissions required by the caller: `contents: read`
- Steps: checkout, `docker/build-push-action@v6` (`push: false`, `load: true`
  — no login/push step exists in this workflow), optional smoke-test step,
  `aquasecurity/trivy-action@v0.36.0` (`exit-code: "0"`, `continue-on-error: true`).
- **`IMAGE_REF` is a job-level `env` var, not a `workflow_call` output.**
  Since `push: false` is hardcoded, the built image only exists in this
  job's local Docker daemon — there is nothing durable for a downstream
  job to reference. A `workflow_call` output would be misleading.
- The smoke-test command is passed via `env: SMOKE_TEST_COMMAND` and
  executed as `bash -c "$SMOKE_TEST_COMMAND"` — never inlined directly into
  `run:` — so caller-supplied values containing `"`, `$(...)`, or `}}` are
  always treated as data, never expanded at template-render time.
- The job's pass/fail is driven only by the build and (if set) the
  smoke-test exit code. Trivy findings are informational only and never
  fail the job.

**`trivy-version` note**: confirmed during apply that
`aquasecurity/trivy-action@v0.36.0`'s `action.yaml` (fetched directly from
the pinned tag) declares a `version` input (default `v0.70.0`), so this
input is safe to expose and override per-call without a `pipelines`
release.

```yaml
jobs:
  docker-build:
    needs: [go-build-test, go-lint, hadolint]
    permissions:
      contents: read
    uses: robbasualto/pipelines/.github/workflows/docker-build.yml@v1.0.1
    with:
      image-tag: go-hadolint-poc:ci
      smoke-test-command: "docker run --rm go-hadolint-poc:ci --version"
```

### `gitleaks.yml`

Scans repository content for committed secrets.

| Input | Default | Description |
|---|---|---|
| `runner` | `ubuntu-latest` | Hosted runner name or caller-owned self-hosted/ARC selector |

| Secret | Required | Description |
|---|---|---|
| `gh_token` | No | Token used to resolve PR diffs. Falls back to `github.token` when omitted, so a caller that forgets `secrets: inherit` still gets a working default rather than a hard failure |

- Permissions required by the caller: `contents: read` (minimum; on
  `pull_request` events using `secrets: inherit`, the caller's default
  `GITHUB_TOKEN` permissions apply as scoped by the caller workflow)
- Steps: checkout (`fetch-depth: 0`), `gitleaks/gitleaks-action@v3.0.0`.
  Internally resolves `GITHUB_TOKEN: ${{ secrets.gh_token || github.token }}`.
- Fails the job when gitleaks reports one or more findings.

```yaml
jobs:
  gitleaks:
    permissions:
      contents: read
    uses: robbasualto/pipelines/.github/workflows/gitleaks.yml@v1.0.1
    # secrets: inherit  # optional — omitting it still works via github.token fallback
```

## Worked example: `go-hadolint-poc` consumer (illustrative only)

This snippet shows how `go-hadolint-poc` **would** consume these workflows.
It is documentation only — `go-hadolint-poc` is not modified by this
change, and the example pins to the current immutable runner-capable release
`@v1.0.1`.

```yaml
# go-hadolint-poc/.github/workflows/ci.yml (illustrative — not applied)
name: CI

on:
  push:
  pull_request:

jobs:
  go-build-test:
    permissions:
      contents: read
    uses: robbasualto/pipelines/.github/workflows/go-build-test.yml@v1.0.1

  go-lint:
    permissions:
      contents: read
    uses: robbasualto/pipelines/.github/workflows/go-lint.yml@v1.0.1

  hadolint:
    permissions:
      contents: read
    uses: robbasualto/pipelines/.github/workflows/hadolint.yml@v1.0.1

  docker-build:
    needs: [go-build-test, go-lint, hadolint]
    permissions:
      contents: read
    uses: robbasualto/pipelines/.github/workflows/docker-build.yml@v1.0.1
    with:
      image-tag: go-hadolint-poc:${{ github.sha }}
      smoke-test-command: "docker run --rm go-hadolint-poc:${{ github.sha }} --version"

  gitleaks:
    permissions:
      contents: read
    uses: robbasualto/pipelines/.github/workflows/gitleaks.yml@v1.0.1
```

Notes on this graph:

- `docker-build` declares `needs: [go-build-test, go-lint, hadolint]` so the
  image is only built after source quality gates pass.
- `gitleaks` has no `needs:` — it scans independently and in parallel.
- Every job declares its own `permissions:` block; none of these workflows
  inherit the caller's default permissions.

### Troubleshooting: workflow not found / permission error

If this example (or any invocation of a workflow in this repository) fails
with a workflow resolution or permission error, check the
[Repository Access Prerequisite](#repository-access-prerequisite) first —
this is the most common cause of an opaque failure before the first
successful cross-repo invocation.
