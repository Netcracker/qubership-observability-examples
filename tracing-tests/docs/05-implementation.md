# Validator implementation: form and language

## Decision

**Go CLI binary**, single codebase, launched in three modes via subcommands. Evolves from CLI → K8s CronJob → (optionally) long-running Deployment with a Prometheus exporter. No standalone microservice at the start.

---

## Deployment form

### One binary, three launch modes

Not three separate applications — **one binary with different subcommands**. Shared core (rules parsing, Tempo client, K8s client, exclusion engine), different entry points.

```bash
tracing-validator offline    --tempo ... --since 1h --rules all --out report.json
tracing-validator inventory  --tempo ... --kubeconfig ... --rules t11,t13
tracing-validator synthetic  --tempo ... --ingress ... --scenarios scenarios.yaml
tracing-validator test-run   --tempo ... --run-id $CI_PIPELINE_ID
tracing-validator serve      --tempo ... --kubeconfig ... --interval 15m   # Phase 2
```

### Evolution by phases

**Phase 1 (MVP, 1–2 weeks):** CLI binary. Launched in three ways without code changes:

| Where | How | Mode |
|---|---|---|
| Locally, dev/debug | `port-forward` to Tempo + run the binary | `offline`, `inventory` |
| K8s CronJob | Same image, `args: ["offline", "--since", "24h", ...]` | daily health check |
| CI step (GitLab/GitHub Actions) | Invoked from the pipeline after deployment | `synthetic`, `test-run` |

Output — JSON report. Non-zero exit code on issues with severity ≥ threshold (controlled via `--fail-on-severity` flag).

**Phase 2 (when a need for continuous monitoring appears):** same binary + `serve` subcommand:

```bash
tracing-validator serve --tempo ... --kubeconfig ... --interval 15m
```

Deployed as a Deployment. Exposes `/metrics` (Prometheus) with rule-specific counters:

```
tracing_validation_issues_total{rule, service, severity}
tracing_validation_completeness_ratio{service}
tracing_validation_orphan_spans_total{service}
```

Scraped by the existing Prometheus. Alerts and dashboards sit on top — the rules are already sketched in [03-tempo-queries.md](03-tempo-queries.md#alerting-rules).

**Phase 3 (only if it's actually needed):** UI / report history / separate storage. **Skipped by default.** Grafana + Tempo + Prometheus cover most use cases.

---

## Why CLI rather than a microservice from day one

1. **"Standalone microservice"** = Deployment + Service + RBAC + CI + monitoring of the validator itself. For a task that initially = "hit Tempo and the K8s API, compare, print JSON", that's overengineering.

2. **A CLI turns into a CronJob without rewriting** — just change the launch command. The reverse isn't as easy: if you start from a stateful service, the CLI mode for a developer has to be bolted on separately.

3. **The validator holds no state.** All data sources are Tempo and the K8s API. A stateful service is justified only if you need report history or a UI. Phase 3 can deliver that — but only when the demand appears.

4. **Easier to test.** A CLI binary with deterministic output (JSON report) is trivially testable: plug in a fake Tempo response, compare the report with the expected one. For a service with HTTP endpoints it's harder.

---

## Why Go

The only truly significant technical discriminator is **`client-go`**. Inventory tests (11–13) require:

- listing Pods with namespace/label filters;
- resolving `envFrom` / `valueFrom.configMapKeyRef` (Test 13 needs the final env values, not references);
- optionally, watching pod events for Test 12.

In Go it's a single import (`k8s.io/client-go`) and ready-made typed structs. In Python (`kubernetes-client`) it works, but noticeably slower and with a less convenient object model. In Java it's heavier to deploy, and startup time is critical for a CronJob.

Additional upsides of Go for this profile:

- Native k8s ecosystem: Tempo, OTel Collector, every k8s operator — all in Go.
- Small static binary (~20–30 MB), ideal for distribution via container / CronJob.
- Concurrency out of the box — polling Tempo and K8s in parallel without extra infrastructure.
- Fast startup (< 100 ms), important for a CronJob on a frequent schedule.

**When Go would be the wrong choice:** if the company has no Go expertise at all and none is expected to appear. But in a k8s environment that's unlikely — Go will come in handy for other tools anyway.

---

## What to build ourselves vs what to take off the shelf

### Build our own

- **Offline trace analysis against rules (Tests 1–10)** — rules are specific to your policies and `exclusions.yaml`. There's nothing off the shelf that produces exactly this report.
- **Inventory tests (11–13)** — your specifics. `ENABLE_TRACING` is an internal contract, nobody else has it.
- **Report aggregator in your format** (JSON / HTML / Markdown for tickets).
- **Exclusion engine** (`docs/04-exclusions.md`).

### Use existing tools

| Task | Off-the-shelf tool |
|---|---|
| Synthetic smoke tests (Test 1 Mode B) — traceparent injection, Tempo polling, assertions | [Tracetest](https://tracetest.io/). Handles it out of the box. Don't duplicate. |
| Enriching spans with k8s attributes (`k8s.pod.name` etc.) | OTel Collector `k8sattributes processor`. See [tests/11-inventory-coverage.md](tests/11-inventory-coverage.md). |
| Prometheus metrics from spans | Tempo `metrics-generator`. |
| Dashboards / alerting | Grafana + Alertmanager. Rules — in [03-tempo-queries.md](03-tempo-queries.md). |
| Per-step assertions from tests | Built into the autotest framework. See [06-test-framework-integration.md](06-test-framework-integration.md). |

---

## Honest question: do we actually need to build our own?

This is worth running as a filter **before** starting work.

**You can get away without your own binary** if:
- Your tests are mostly Tests 1, 2, 4, 5.
- They are expressible via TraceQL queries + Prometheus alerting (drafts are already in [03-tempo-queries.md](03-tempo-queries.md)).
- You need per-span alerting, not an aggregated per-service report.

Then validation = a set of alerting rules and Grafana dashboards. Zero custom code.

**Your own binary is justified** if:
- You need programmatic logic that TraceQL can't express (Test 1 matching with callee resolution, Tests 11–13 inventory).
- You need an **aggregated per-service report** with heuristics (`missing_ratio`, hypothesis generation).
- You need integration with an internal inventory (K8s, CMDB).
- You need a unified entry point for every mode (offline / inventory / synthetic / test-run).

**Deciding argument:** Inventory tests 11–13 can't be expressed in TraceQL in principle — the source of truth there is not Tempo, but the K8s API. At the very least, your own tool is needed for those. And once you've committed, it's logical to cover the rest of the rules with it too.

---

## Project layout (sketch)

```
tracing-validator/
├── cmd/
│   └── tracing-validator/
│       └── main.go               # cobra root + subcommand registration
├── internal/
│   ├── tempo/                    # Tempo HTTP API client
│   │   ├── client.go
│   │   └── search.go
│   ├── k8s/                      # K8s API client (inventory)
│   │   ├── pods.go
│   │   └── env_resolver.go       # resolves envFrom / configMapKeyRef
│   ├── model/                    # OTLP trace / span structs
│   ├── rules/                    # implementation of Tests 1–13
│   │   ├── t01_client_server_matching.go
│   │   ├── t05_required_attributes.go
│   │   ├── t11_pod_to_trace.go
│   │   ├── t12_trace_to_pod.go
│   │   ├── t13_enable_tracing_contract.go
│   │   └── rule.go               # shared Rule interface
│   ├── exclusions/               # engine from 04-exclusions.md
│   ├── report/                   # JSON / HTML / Markdown writers
│   └── commands/                 # subcommand handlers
│       ├── offline.go
│       ├── inventory.go
│       ├── synthetic.go
│       ├── testrun.go
│       └── serve.go              # Phase 2
├── configs/
│   ├── exclusions.example.yaml
│   └── scenarios.example.yaml
├── Dockerfile
└── deploy/
    ├── cronjob.yaml              # example CronJob for daily offline
    └── deployment.yaml           # Phase 2: Deployment + Service + ServiceMonitor
```

**Shared Rule interface:**

```go
type Rule interface {
    Name() string                  // "client_server_matching"
    Severity() Severity            // ERROR / WARNING / INFO
    AppliesTo() []ServiceType      // REST, gRPC, Kafka consumer, ...
    Validate(ctx context.Context, t *Trace, ex *ExclusionEngine) []Issue
}
```

Inventory rules (T11–T13) get an additional interface with the K8s client:

```go
type InventoryRule interface {
    Rule
    ValidateInventory(ctx context.Context, tempo TempoClient, k8s K8sClient, ex *ExclusionEngine) []Issue
}
```

This gives a clean split: "rule over a trace" vs "rule over inventory", and it's easy to add new ones.

---

## Environment requirements

### For Phase 1 (CLI)

| What | Why |
|---|---|
| Read-only HTTP access to Tempo (`/api/search`, `/api/traces/{id}`) | All modes |
| Read-only K8s API: `get,list` on `pods`, `deployments`, `configmaps` (for env resolution) | Inventory mode (Tests 11–13) |
| HTTP access to Ingress / API Gateway | Synthetic mode |
| ServiceAccount (if run as a CronJob) | With the minimal RBAC rights above |

### Additionally for Phase 2 (serve)

| What | Why |
|---|---|
| `Service` + `ServiceMonitor` (Prometheus Operator) | Metrics collection |
| PodDisruptionBudget (optional) | For HA |

---

## What NOT to do

Anti-patterns easy to avoid at the start:

1. **Designing it as a microservice from the start.** First prove that a CLI is not enough.
2. **Building our own web UI.** Grafana is already there. Dashboards on TraceQL + Prometheus metrics give 90% of the needed visualization.
3. **Keeping rules in code and exclusions in code.** Rules — in code (they require programmatic logic). Exclusions — in a YAML config (they change often, don't require a redeploy).
4. **Duplicating Tracetest.** Synthetic mode with assertions is its core job. Don't rewrite it.
5. **Tying into Tempo's internal storage format.** Public HTTP API only. Otherwise any Tempo upgrade breaks the validator.
6. **Writing multiple languages in the same codebase.** One binary, one language, one ci/cd pipeline.

---

## Open questions for the next step

1. **Binary / repository name.** Currently in the text — `tracing-validator`. Confirm whether it fits the company namespace.
2. **Where to run the Phase 1 CronJob.** In the same namespace as Tempo, or in a separate `observability-validator`?
3. **Output format for CI.** Is JSON enough, or is JUnit XML needed for GitLab/GitHub test reports integration?
4. **Rule versioning.** If rule T14 is added — is it optional (opt-in via `--rules`) or enabled by default? Recommendation: opt-in for the first 2 releases, then enabled by default.
5. **Report history.** Do we keep report.json as a CI artifact, or also separately (object storage)?

These get resolved when moving from the design doc to actual development.
