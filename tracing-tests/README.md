# Tracing Validation

Documentation and approaches for tracing validation in microservices to identify services that "lose" trace context.

**Backend:** Grafana Tempo

---

## Quick Start

### 1. Understand the problem

Tracing gets lost when:
- A service is not instrumented
- A service does not propagate trace context
- Incompatible propagation formats
- Configuration errors

The result is fragmented traces and inability to debug.

### 2. Choose validation tests

| # | Test | Severity |
|---|------|----------|
| 1 | Client span → Server span matching | HIGH |
| 2 | Requests without trace_id | HIGH |
| 3 | Span without parent_span_id | MEDIUM |
| 4 | Propagator format consistency | HIGH |
| 5 | Required span attributes | MEDIUM |

Full list: [docs/01-approach.md](docs/01-approach.md)

### 3. Configure exclusions

Not all spans should be validated:
- Cron jobs (root span is allowed)
- Kafka consumers (new trace chain)
- Health checks (not traced)

Details: [docs/04-exclusions.md](docs/04-exclusions.md)

### 4. Run TraceQL queries

```traceql
# Find orphan server spans
{ span.kind = server && nestedSetParent = -1 && resource.service.name != "api-gateway" }

# Find spans missing required attributes
{ span.kind = server && span.http.method = "" }

# Find error spans
{ status = error }
```

All queries: [docs/03-tempo-queries.md](docs/03-tempo-queries.md)

---

## Documentation

| Document | Description |
|----------|-------------|
| [01-approach.md](docs/01-approach.md) | Catalog of tracing validation tests (overview + links to detailed files) |
| [02-architecture.md](docs/02-architecture.md) | Prototype architecture (offline/real-time) |
| [03-tempo-queries.md](docs/03-tempo-queries.md) | TraceQL queries for each test |
| [04-exclusions.md](docs/04-exclusions.md) | Exclusion rules |
| [05-implementation.md](docs/05-implementation.md) | Implementation: Go CLI, rollout phases, build-vs-buy |
| [06-test-framework-integration.md](docs/06-test-framework-integration.md) | Integration with the autotest framework (per-step assertions + aggregate via CLI) |
| [tests/01-client-server-matching.md](docs/tests/01-client-server-matching.md) | Detailed spec of Test 1 (runnable on an existing cluster) |
| [tests/11-inventory-coverage.md](docs/tests/11-inventory-coverage.md) | Tests 11–13: reconciliation with K8s API (pod↔trace, `ENABLE_TRACING` contract) |

---

## Expected outcome

After applying the approaches, teams receive a report:

```
Service Tracing Health Report
=============================
service-a: OK (all spans have parents, server spans match client spans)
service-b: WARN (missing server span for 3/10 client calls)
service-c: ERROR (no spans at all, likely not instrumented)
service-d: WARN (spans without required attributes: http.method)
```

---

## Operating modes

### Offline Analysis

Analysis of existing traces over a period:
- Daily/weekly health check
- Incident investigation
- Post-release quality assessment

### Real-time Validation

Validation during tests:
- Integration tests
- Smoke tests after deployment
- Validation of new services

---

## Industry sources

- [OpenTelemetry Distributed Tracing Best Practices](https://www.withcoherence.com/articles/opentelemetry-distributed-tracing-tutorial-and-best-practices)
- [Trace-based Testing OpenTelemetry Demo](https://opentelemetry.io/blog/2023/testing-otel-demo/)
- [Netflix Distributed Tracing Infrastructure](https://netflixtechblog.com/building-netflixs-distributed-tracing-infrastructure-bb856c319304)
- [Grafana Tempo TraceQL](https://grafana.com/docs/tempo/latest/traceql/)

---

## Existing tools

| Tool | Description |
|------|-------------|
| [Tracetest](https://tracetest.io/) | Trace-based testing with assertions |
| Jaeger Trace Quality Engine | Orphan span analysis (Jaeger only) |
| Tempo metrics-generator | Span metrics for monitoring |
