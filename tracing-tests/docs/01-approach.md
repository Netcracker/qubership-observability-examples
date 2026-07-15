# Approaches to tracing validation in microservices

## The "lost traces" problem

In distributed systems, tracing can "get lost" for several reasons:
- A service is not instrumented
- A service is instrumented but does not propagate trace context
- Incompatible propagation formats between services
- Configuration errors (wrong endpoint, sampling)

The result is fragmented traces where part of the call chain is invisible, making debugging and root cause analysis impossible.

---

## Validation tests

### Core tests (detecting tracing loss)

| # | Test                                | What it detects                                                 | Severity |
|---|-------------------------------------|-----------------------------------------------------------------|-------------|
| 1 | Client span → Server span matching  | Services that receive requests but do not emit spans            | HIGH |
| 2 | Reject requests without trace_id    | Services that send requests without trace context               | HIGH |
| 3 | Span without parent_span_id = error | Services that tried to propagate context but did not find it    | MEDIUM |
| 4 | B3 propagation check                | Services that use the legacy B3 propagation protocol            | MEDIUM |


### Additional tests (instrumentation quality)

| # | Test | What it detects | Severity |
|---|------|--------------|-------------|
| 4 | Propagator format consistency | Format inconsistency (W3C vs B3 vs Jaeger) | HIGH |
| 5 | Required span attributes validation | Missing required attributes | MEDIUM |
| 6 | Trace depth anomaly detection | Suspiciously shallow traces | LOW |
| 7 | Hanging spans detection | Spans without end_time | MEDIUM |
| 8 | Duplicate span_id detection | Identifier collisions | LOW |
| 9 | Sampling decision propagation | Loss of the sampled=true flag | HIGH |
| 10 | Trace completeness ratio | % of complete vs fragmented traces | MEDIUM |

### Inventory-driven tests (reconciliation with the K8s API)

These work not on trace structure but reconcile the set of services/pods in Tempo against the K8s inventory. They catch "silent" services, typos in `service.name`, zombie sources, and internal contract violations.

| # | Test | What it detects | Severity |
|---|------|--------------|-------------|
| 11 | K8s pod → trace presence | Pods that are running but do not emit a single span during the window | MEDIUM |
| 12 | Trace → K8s pod existence | Traces from sources that do not exist in the cluster (typo in `service.name`, zombies) | LOW |
| 13 | `ENABLE_TRACING=true` contract | Pods that declare tracing support but emit nothing (strict form of Test 11) | HIGH |

Details: [tests/11-inventory-coverage.md](tests/11-inventory-coverage.md). Require read-only access to the K8s API (`pods`, `deployments`) in addition to the Tempo API.

---

## Detailed test descriptions

> **Organizing principle:** each test, documented as a runnable specification (run modes, algorithm, edge cases, report format), lives in a separate file `tests/NN-*.md`. Here we only provide short descriptions and links.

### Test 1: Client span → Server span matching

**Problem:** Service A calls service B, creates a client span, but service B does not create the corresponding server span.

**Check summary:** for every `kind=CLIENT` span in a trace there must exist a `kind=SERVER` span with `parent_span_id = client.span_id` from a different service. Absence means the `callee` is not instrumented, does not extract trace context, or loses it at an async boundary.

**Running without modifying microservices:** read-only access to the Tempo HTTP API; for smoke mode — access to Ingress. Detailed algorithm, false-positive filtering, the `missing_ratio` hypothesis heuristic, and report format are in [tests/01-client-server-matching.md](tests/01-client-server-matching.md).

---

### Test 2: Requests without trace_id

**Problem:** A service sends HTTP/gRPC requests without `traceparent`/`tracestate` headers.

**How to detect:**
- **Active method:** Middleware/sidecar checks incoming requests for the presence of trace headers
- **Passive method:** Server span without parent_span_id, even though the calling service is known

**Check configuration:**
```yaml
# Sidecar/Envoy configuration
validate_trace_headers:
  enabled: true
  action: log  # or reject
  exclude:
    - path: /health
    - path: /ready
    - source: external-gateway
```

---

### Test 3: Span without parent_span_id

**Problem:** A service creates a span but cannot find the parent context.

**When it is an error:**
- The request came from an internal service (not from the edge gateway)
- The request is not a scheduled job or async consumer

**When it is acceptable:**
- Root span from an edge gateway
- Cron job / scheduled task
- Kafka consumer (first span in a new chain)

**TraceQL query:**
```
{ span.kind = server && parent_span_id = "" && resource.service.name != "edge-gateway" }
```

---

### Test 4: Propagator format consistency

**Problem:** Service A uses W3C Trace Context, service B uses B3 headers.

**Formats:**
| Format | Headers | Example |
|--------|---------|--------|
| W3C Trace Context | `traceparent`, `tracestate` | `traceparent: 00-trace_id-span_id-01` |
| B3 Single | `b3` | `b3: trace_id-span_id-1-parent_span_id` |
| B3 Multi | `X-B3-TraceId`, `X-B3-SpanId`, etc. | Multiple headers |
| Jaeger | `uber-trace-id` | `trace_id:span_id:parent_id:flags` |

**Recommendation:** Standardize on W3C Trace Context with a fallback to B3 for legacy.

---

### Test 5: Required span attributes validation

**Required attributes (semantic conventions):**

| Attribute | Span type | Description |
|---------|------------|----------|
| `service.name` | All | Service name |
| `span.kind` | All | CLIENT, SERVER, PRODUCER, CONSUMER, INTERNAL |
| `http.method` | HTTP | GET, POST, etc. |
| `http.url` or `http.route` | HTTP | URL or route template |
| `http.status_code` | HTTP | 200, 404, 500, etc. |
| `rpc.system` | gRPC | "grpc" |
| `rpc.service` | gRPC | Service name |
| `rpc.method` | gRPC | Method name |
| `db.system` | Database | "postgresql", "mysql", etc. |
| `db.statement` | Database | SQL query (sanitized) |

**TraceQL for finding spans without attributes:**
```
{ span.kind = server && http.method = "" }
```

---

### Test 6: Trace depth anomaly detection

**Problem:** A trace contains 1-2 spans where a chain of 5-10 is expected.

**How to detect:**
1. Establish a baseline depth for typical user flows
2. Compare the depth of a specific trace against the baseline
3. Abnormally shallow traces → possible loss of spans

**Metric:**
```
trace_depth_ratio = actual_depth / expected_depth
if trace_depth_ratio < 0.5 → WARNING
```

---

### Test 7: Hanging spans detection

**Problem:** A span started but did not finish (no `end_time`).

**Causes:**
- Application crash
- Timeout without proper cleanup
- Bug in instrumentation

**TraceQL:**
```
{ status = unset && duration > 60s }
```

---

### Test 8: Duplicate span_id detection

**Problem:** Two spans with the same `span_id` in the same trace.

**Causes:**
- Bug in ID generation
- Replay attack
- Incorrect SDK usage

**How to detect:**
```sql
SELECT trace_id, span_id, COUNT(*)
FROM spans
GROUP BY trace_id, span_id
HAVING COUNT(*) > 1
```

---

### Test 9: Sampling decision propagation

**Problem:** Parent span has `sampled=true`, but the child span is not recorded.

**W3C Trace Context:**
```
traceparent: 00-{trace_id}-{span_id}-{flags}
                                      ^^ 01 = sampled
```

**How to detect:**
- Compare the number of client spans with the number of server spans
- If client spans > server spans when sampled=true → propagation problem

---

### Test 10: Trace completeness ratio

**Tracing quality metric:**
```
completeness_ratio = traces_with_full_chain / total_traces
```

**How to define a "complete" trace:**
1. Has a root span (entry point)
2. Every client span has a corresponding server span
3. No orphan spans (spans without a parent, except the root)

**Target:** completeness_ratio > 95%

---

## Test applicability matrix

| Service type | T1 | T2 | T3 | T4 | T5 | T6 | T7 | T8 | T9 | T10 | T11 | T12 | T13 |
|-------------|----|----|----|----|----|----|----|----|----|----|----|----|----|
| REST API | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| gRPC Service | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| Kafka Consumer | ✓ | - | - | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| Cron Job | - | - | - | ✓ | ✓ | - | ✓ | ✓ | - | - | - | ✓ | ✓ |
| Edge Gateway | ✓ | - | - | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| Database | ✓ | N/A | N/A | N/A | ✓ | N/A | ✓ | ✓ | N/A | N/A | N/A | N/A | N/A |

**Legend:**
- ✓ — test applicable
- `-` — test not applicable (exclusion)
- N/A — not relevant for this type

*Test 11 is not applicable to Cron Job — the absence of spans between runs is legitimate. Test 13 is applicable to Cron Job if `ENABLE_TRACING=true` is declared: the contract holds for all runs within the window.*
