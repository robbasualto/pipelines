```yaml
schema: gentle-ai.verify-result/v1
evidence_revision: sha256:7f1aaea136fe8aa2d90013c5768a4b038af7f66e3cacb82bd8d039559a1478b4
verdict: pass_with_warnings
blockers: 0
critical_findings: 0
requirements: 9/9
scenarios: 9/9
test_command: "ruby -"
test_exit_code: 0
test_output_hash: sha256:a17033f2ef1d217233e37b2678fd1265a08c7f381d77a279d5a8736c835b0757
build_command: "printf '%s\\n' 'No build command configured (infra-as-code repository)'"
build_exit_code: 0
build_output_hash: sha256:6e05aba1b76f1c02305f9176c8488e8e0e87dce2471a3d51d77bfe583afc5a9d
```

## Verification Report

**Change**: `configurable-runner-input`  
**Version**: N/A (active delta)  
**Mode**: Standard (`strict_tdd: false`)  
**Native attempt**: ordinal 3, generation 3; no additional native attempt started.

### Executive result

**PASS WITH WARNINGS**. The five reusable workflows implement the identical optional string runner contract, preserve their existing workflow behavior, and the README and consumer facts satisfy the active proposal/spec/design acceptance boundary. The warnings are limited to unavailable `actionlint` and the inability to execute GitHub-hosted/ARC control-plane jobs locally; local Docker/buildx plus all three host-curl health checks passed.

### Completeness

| Metric | Result |
|---|---:|
| Tasks total | 8 |
| Tasks complete | 8 |
| Tasks incomplete | 0 |
| Review budget | 800 changed lines |
| Implementation/documentation diff | 102 changed lines (94 additions, 8 deletions) |

All task checkboxes in `tasks.md` are checked. Standard verification was used because strict TDD is disabled and this is an infrastructure-as-code repository with no application test runner or coverage command.

### Commands and evidence

| Check | Exact command/result |
|---|---|
| YAML syntax | `ruby -e 'require "yaml"; ... YAML.parse_file(path)' .github/workflows/*.yml` — exit 0; Ruby 3.4.10, Psych 5.2.2; all five files parsed. |
| Structural/compatibility/consumer checker | `ruby - <<'RUBY' ... RUBY` — exit 0; output hash `sha256:a17033f2ef1d217233e37b2678fd1265a08c7f381d77a279d5a8736c835b0757`; all workflow contracts, README requirements, nine consumer calls, mappings, hashes, and scalar `lab-runner` caller addition passed. |
| Formatting | `git diff --check` — exit 0. |
| Workflow-aware lint | `actionlint .github/workflows/*.yml` — exit 127, `command not found`; not a PASS. |
| Consumer worktree check | `git -C /home/jotaese/Projects/Rightsizing-tps-final diff --exit-code -- .github/workflows/ci.yml docker/checkout/Dockerfile docker/inventory/Dockerfile docker/payment/Dockerfile` — exit 0. Consumer status contains pre-existing staged `A .github/workflows/ci.yml`; no verification change was made. |
| Local consumer runtime | `docker buildx build --load ...` for each of checkout, inventory, payment, followed by the consumer-equivalent `docker run --publish` and host `curl --fail .../healthz` polling — exit 0 for all three; Docker server 29.7.2, buildx 0.36.1, curl 8.21.0. Temporary images/containers were cleaned up. |
| Build | No build command is configured for this infra-as-code repository; no application build was claimed. |

### Workflow contract verification

All five `.github/workflows/*.yml` files expose `workflow_call.inputs.runner` with `type: string`, `required: false`, and `default: ubuntu-latest`; each sole job uses exactly `${{ inputs.runner }}` in `runs-on`. No workflow contains `lab-runner`, `self-hosted`, or another ARC label, and none hardcodes `runs-on: ubuntu-latest`.

Existing contracts were preserved: input names/defaults/required flags, `contents: read`, action references, one-job/no-dependency structure, `timeout-minutes: 15`, Gitleaks optional `gh_token` and `${{ secrets.gh_token || github.token }}` fallback, full-history checkout, Docker `load: true`/`push: false`, report-only Trivy, and caller smoke command via `SMOKE_TEST_COMMAND` and `bash -c`.

### README verification

README covers omission/backward compatibility, explicit `ubuntu-latest`, the confirmed caller-owned `lab-runner` selector, repository runner-group/label access, required runner capabilities, DinD/buildx, host `curl`, self-hosted trust/isolation/network/secret controls, no automatic failover, and ARC's inability to repair GitHub Actions scheduling/API/control-plane outages. It states that the library provisions neither infrastructure nor an ARC label.

### Consumer verification

The read-only `/home/jotaese/Projects/Rightsizing-tps-final/.github/workflows/ci.yml` contains exactly nine calls: one Go build/test, one Go lint, three Hadolint, three Docker-build, and one Gitleaks. Both Go calls use `1.26.0`; all nine jobs declare `contents: read`; the three Docker jobs retain their five-job `needs` graph; Gitleaks maps `secrets.GITHUB_TOKEN` to `gh_token`; and Docker calls retain the three Dockerfile paths and external host-curl `/healthz` smoke commands on ports 18081–18083. The three Dockerfiles remain Go 1.26.0 builder/non-root distroless images. An in-memory caller-side compatibility check confirms `runner: lab-runner` is a valid single-string addition for all nine calls without writing the consumer.

Consumer fixture SHA-256 values observed:

| File | SHA-256 |
|---|---|
| `.github/workflows/ci.yml` | `e7043eeb975988ad484204ff8289fdf12adc2982fe2591efe1ee4d83af685a71` |
| `docker/checkout/Dockerfile` | `2058de29595600ae3ce5e707d5fdc92bac5a82414f35cb52722b14f61ccf5075` |
| `docker/inventory/Dockerfile` | `eb0b8bc2382ca0a67103f2a72d66df8eab5b09e0187569b6879ed9a4d587d47d` |
| `docker/payment/Dockerfile` | `757f0cd9f0c664e41cc1dd50248426c1ce694c7f180ba1cc33fa69ec1e468baa` |

### Spec compliance matrix

| Requirement | Scenario | Covering evidence | Result |
|---|---|---|---|
| Go build/test runner selection | Default or explicit placement | Runtime Ruby structural checker + YAML parse | ✅ COMPLIANT |
| Go lint runner selection | Default or explicit placement | Runtime Ruby structural checker + YAML parse | ✅ COMPLIANT |
| Hadolint runner selection | Default or explicit placement | Runtime Ruby structural checker + YAML parse | ✅ COMPLIANT |
| Docker runner and unchanged contract | Hosted default or explicit runner | Runtime Ruby compatibility checker + local build/smoke harness | ✅ COMPLIANT |
| Gitleaks runner and token fallback | Default/explicit runner and omitted token | Runtime Ruby compatibility checker | ✅ COMPLIANT |
| Runner documentation/prerequisites | Consumer chooses infrastructure | Runtime README assertion checker | ✅ COMPLIANT |
| Security/outage boundaries | Runner unavailable or control plane impaired | Runtime README assertion checker | ✅ COMPLIANT |
| Consumer fact validation | Existing consumer remains equivalent | Runtime consumer checker + three local Docker/curl runs + diff/hash checks | ✅ COMPLIANT |
| Confirmed ARC selector boundary | Caller selects `lab-runner` | Runtime scalar/call-site compatibility assertion; no live ARC claim | ✅ COMPLIANT |

**Compliance summary**: 9/9 requirements and 9/9 scenarios covered by passing verification checks. GitHub Actions scheduling itself was not locally executable; this is recorded as a warning rather than fabricated ARC evidence.

### Design coherence

| Decision | Result | Evidence |
|---|---|---|
| Identical scalar API in all five workflows | ✅ Followed | Exact input contract assertions passed. |
| Only `runs-on` placement changes | ✅ Followed | Workflow diff contains only runner input additions and runner expressions. |
| Caller-owned ARC selector; generic library YAML | ✅ Followed | No workflow ARC/self-hosted label; README-only `lab-runner` example. |
| Preserve Docker/smoke/Gitleaks behavior | ✅ Followed | Compatibility assertions and local consumer runtime passed. |
| No consumer/infrastructure/failover changes | ✅ Followed | Consumer diff exit 0; no ARC live execution or failover was claimed. |

### Acceptance mapping

1. **Five optional runner inputs** — PASS: all five contracts are identical and dynamic.
2. **Existing behavior** — PASS: inputs, flags, permissions, secrets, refs, dependencies, timeout, Docker flags, smoke handling, and token fallback preserved.
3. **Documentation** — PASS: all required hosted, ARC, prerequisite, security, failover, and control-plane statements present.
4. **Consumer validation** — PASS: nine calls, Go 1.26.0, three Dockerfiles, permissions, needs, token mapping, smoke commands, unchanged hashes, and local runtime validated; live ARC remains unverified.
5. **Structural/tooling checks** — PASS with warning: Ruby Psych and assertions passed; `git diff --check` passed; `actionlint` unavailable.
6. **Scope** — PASS: no source or consumer files modified by verification; no commit or push.

### Issues

**CRITICAL**: None.  
**WARNING**:
- `actionlint` is unavailable (`command not found`); YAML parsing and targeted structural assertions are the substitute.
- No GitHub-hosted, ARC, or GitHub Actions control-plane execution is possible from this local environment. Local Docker/DinD-equivalent capability and host-curl behavior passed, but this is not live ARC proof.
- The consumer repository retains a pre-existing staged `A .github/workflows/ci.yml` status; the targeted worktree diff is clean and verification did not alter it.
**SUGGESTION**: Install `actionlint` and run the reusable workflows through an actual GitHub Actions caller when ARC access is available.

### Verdict

**PASS WITH WARNINGS** — implementation and acceptance evidence satisfy the active SDD artifacts; only workflow-aware lint and live GitHub/ARC execution remain unavailable locally.
