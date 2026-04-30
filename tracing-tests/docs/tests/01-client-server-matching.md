# Test 1: Client span → Server span matching

**Severity:** HIGH
**What it detects:** services that accept calls but do not create (or lose) server spans.
**Applicable to:** REST, gRPC, any synchronous RPC. For async (Kafka/RabbitMQ) — see the separate PRODUCER → CONSUMER matching test.

---

## 1. Black-box principle

The test must run against an existing cluster **without modifying microservices**. Only the following is required:

| Access | Why | How to obtain |
|---|---|---|
| Read-only to Tempo HTTP API (`/api/search`, `/api/traces/{id}`) | Source of all observations | `kubectl port-forward svc/tempo 3200:3200` or an in-cluster `Service` |
| (optional) HTTP access to Ingress / API Gateway | Only for synthetic mode | An existing public endpoint |
| (optional) read-only K8s API: `Service`, `Endpoints`, `Ingress` | Mapping `hostname → service` for culprit attribution | ServiceAccount with `get,list` on these resources |

No sidecars, init containers, or changes to service Helm charts are required. The validator lives as a separate pod or CLI binary.

---

## 2. What exactly we check

For each span with `kind = CLIENT` (HTTP/gRPC outgoing call) within the same `trace_id`, a span is expected with:

- `kind = SERVER`,
- `parent_span_id = <client.span_id>`,
- `resource.service.name != <client.resource.service.name>` (a foreign service, not itself).

The absence of such a server span = an indicator of one of the following problems:

1. The receiving service is **not instrumented** at all.
2. It is instrumented, but **does not extract** trace context from incoming headers (W3C `traceparent` is missing from the propagators).
3. Context is **lost on the async boundary** inside the receiver (hand-off to a thread pool / executor without context propagation).
4. **Propagation format mismatch**: A sends W3C, B reads only B3 (see Test 4).

The test **does not distinguish** the causes — it only localizes the `(caller, callee)` pair. Further diagnosis is manual.

---

## 3. Run modes

### Mode A — Offline (recommended as the baseline)

Analysis of traces already accumulated in Tempo over a window (hour / day / week).

**Pros:** nothing needs to be sent to the cluster; covers real production traffic; finds problems in all code paths that requests actually traversed.
**Cons:** does not find problems in dead endpoints; depends on traffic volume.

### Mode B — Synthetic / real-time

The validator itself generates a request to Ingress with a known `trace_id` and, after 10–30 s, checks the result in Tempo.

**Pros:** deterministic; works in smoke tests after deployment; covers endpoints without natural traffic.
**Cons:** you need to know the scenarios (which URLs to hit, which bodies to send); requires Ingress access; an alternative is to use existing tools (Tracetest, malabi) instead of your own runner.

In CI it makes sense to have **both**: smoke on critical flows (Mode B) + a nightly report over the past 24 hours (Mode A).

---

## 4. Algorithm (Mode A)

### Step 1. Obtain a pool of traces over the window

Tempo Search API:

```http
GET /api/search?q={span.kind=client}&start=<unix_ts>&end=<unix_ts>&limit=1000
```

The response contains an array `traces[]`, each with a `traceID`. For large windows — paginate by decreasing `end`.

### Step 2. For each trace_id fetch the full trace

```http
GET /api/traces/<traceID>
```

An OTLP structure is returned with `batches[].resource.attributes` (including `service.name`) and `batches[].scopeSpans[].spans[]` (with `spanId`, `parentSpanId`, `kind`, `attributes`).

### Step 3. Build per-trace indexes

```text
spans_by_id:        span_id          -> span (with denormalized service_name)
children_by_parent: parent_span_id   -> [span, ...]
```

### Step 4. For each client span, check for a server child

```text
for each c in spans where c.kind == CLIENT:
    candidates = children_by_parent[c.span_id]
    server_match = first s in candidates where
                       s.kind == SERVER
                       and s.service_name != c.service_name
    if server_match is missing:
        register issue(c, callee=resolve_callee(c))
```

### Step 5. Resolve the "callee" (culprit)

When a server span is not found, we need to figure out **who was called** so that the issue is actionable. Sources, in priority order:

1. `peer.service` (an explicit hint from instrumentation, if present).
2. `server.address` + `server.port` (OTel semconv 1.21+).
3. Legacy names: `net.peer.name`, `net.peer.ip`, `http.host`, `http.url` hostname.
4. For gRPC: `rpc.service` complements the address.

Next, hostname → `Service` name in the cluster:

- If the hostname is cluster DNS (`*.svc.cluster.local`, or a short service name), extract the service name directly.
- Otherwise resolve via the K8s API: a `Service` with this ClusterIP / ExternalName.
- If resolution fails — this is an **external host** and falls under the "External API call" exclusion (see step 6).

### Step 6. Filtering out legitimate cases (False positive filter)

Without this filter the report would devolve into noise. Apply **before** registering an issue:

| Condition on client span | Why we skip |
|---|---|
| `db.system` is present | This is a DB client (Postgres, Redis, …). The DB does not write its own server span. |
| `messaging.system` is present | This is producer/consumer semantics, not CLIENT→SERVER. Checked by a separate test. |
| `peer.service` / hostname resolves to an external CIDR (not in the whitelist of cluster networks) | External API — instrumentation is outside our control. |
| `http.route` ∈ {`/health`, `/healthz`, `/ready`, `/live`, `/ping`, `/metrics`, `/actuator/*`} | Health/metrics — typically not traced (see `04-exclusions.md`). |
| Any rule from `exclusions:` in `docs/04-exclusions.md` fires | Explicitly configured exclusions. |
| The root span of the trace has `sampled=false` (the `01` flag is missing from traceparent) | No one was supposed to write spans in the first place. |
| `callee` ∈ `service_blacklist` | The service is explicitly marked as "not instrumented, not planned". |

### Step 7. Aggregation

A single uninstrumented service generates thousands of identical issues. Collapse by the key `(caller_service, callee_service, callee_endpoint)`:

```json
{
  "caller_service": "order-service",
  "callee_service": "notification-service",
  "callee_endpoint": "POST /notifications",
  "missing_server_span_count": 1247,
  "total_client_calls": 1247,
  "missing_ratio": 1.00,
  "sample_trace_ids": ["abc...", "def...", "..."]
}
```

`missing_ratio = 1.00` → the callee is almost certainly **not instrumented**.
`missing_ratio` around 0.05–0.20 → context is **lost** in some of the flows (async, retry, a specific endpoint).

### Step 8. Account for Tempo ingest lag

A server span may appear in Tempo **later** than the client span (different batches from different instances). To avoid counting this as missing:

- Analyze only traces whose root span ended `> 60 s` ago.
- Or: on the first "missing", mark as pending, re-check the trace after 30–60 s, and only then register an issue.

---

## 5. Algorithm (Mode B — synthetic)

### Step 1. Generate the trace context

```text
trace_id = 32 hex (random, non-zero)
span_id  = 16 hex (random)
traceparent = "00-{trace_id}-{span_id}-01"   // 01 = sampled, otherwise nothing will be recorded
tracestate  = ""                              // optional
```

### Step 2. Hit the Ingress

```http
GET https://api.staging.example/api/orders/123 HTTP/1.1
traceparent: 00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01
```

Scenario = ordered list of `(method, url, body, expected_chain)`.

### Step 3. Wait for ingest

Polling with back-off: 2 s, 5 s, 10 s, 20 s (max ≈ 60 s) — `GET /api/traces/{trace_id}`, until the trace appears and the number of spans stabilizes (two consecutive identical responses).

### Step 4. Apply the Mode A algorithm to a single trace

Plus an additional check against the scenario's `expected_chain`: if the expected chain was `gateway → order-service → payment-service → db`, but we got `gateway → order-service` — fail, indicating the link at which the break occurred.

---

## 6. Minimal run

```bash
# 1. Access to Tempo (no changes in the cluster)
kubectl -n observability port-forward svc/tempo 3200:3200

# 2. Offline report for the last hour
tracing-validator offline \
    --tempo http://localhost:3200 \
    --since 1h \
    --exclusions ./exclusions.yaml \
    --rules client_server_matching \
    --out report.json

# 3. Synthetic smoke-test
tracing-validator synthetic \
    --tempo http://localhost:3200 \
    --ingress https://api.staging.example \
    --scenarios ./scenarios.yaml \
    --wait 60s
```

The only requirement for the microservices themselves is that **they already send OTLP to Tempo**. If they do not — Test 1 will detect exactly that: the service will show up as a `callee` without a single server span.

---

## 7. Report format (Test 1 issue)

```json
{
  "rule": "client_server_matching",
  "severity": "ERROR",
  "caller_service": "order-service",
  "callee_service": "notification-service",
  "callee_endpoint": "POST /notifications",
  "missing_count": 1247,
  "total_count": 1247,
  "missing_ratio": 1.00,
  "first_seen": "2026-04-21T08:14:32Z",
  "last_seen": "2026-04-21T09:14:01Z",
  "sample_trace_ids": [
    "4bf92f3577b34da6a3ce929d0e0e4736",
    "8a3b2e1d9f6c4b2a7e5d8c9b1a2f3e4d"
  ],
  "hypothesis": "callee is likely not instrumented (missing_ratio = 1.00)"
}
```

The `hypothesis` field is a heuristic based on `missing_ratio` and the presence of spans from `callee_service` in other traces:

| `missing_ratio` | Spans from callee exist anywhere | Hypothesis |
|---|---|---|
| ≈ 1.00 | nowhere | `callee` is not instrumented |
| ≈ 1.00 | present in other traces | `callee` does not extract trace context (incompatible propagator or none at all) |
| 0.05–0.50 | present | context is lost on some code paths (async, retry, specific endpoint) |
| < 0.05 | present | likely ingest lag / sampling — lower severity to WARNING |

---

## 8. Edge cases

| Case | Handling |
|---|---|
| HTTP client retries (one client span, 3 attempts) | A single server span is enough — this is OK. |
| Sampling: client recorded, server not | If the root has `sampled=false`, discard the whole trace. If `sampled=true` but server is missing — that is actually Test 9 (sampling propagation), but manifests as Test 1. Mark with a separate flag `possibly_sampling_loss`. |
| Async boundary (Kafka between A and B) | `kind=PRODUCER` on A, `kind=CONSUMER` on B. **Do not confuse with CLIENT→SERVER** — a separate matching via span links / `messaging.message_id`. |
| Self-call (a service calls itself) | The condition `s.service_name != c.service_name` cuts off the legitimate match. Remove it for self-call: allow a server span from the same service if `peer.service` points to it. |
| The Ingress controller writes a client span, and the backend behind it writes a server span with a different `trace_id` | This is a broken propagator on the ingress (Test 4). Register as a Test 1 issue with `hypothesis = "trace_id mismatch — broken propagator at caller"`. |
| A service mesh (Istio/Linkerd) adds its own spans | Between the client (application) and server (application), an extra layer of mesh spans appears. The algorithm must walk the `parent → child` chain rather than strictly one level — look for the nearest SERVER descendant from a different service. |
| Cron / Kafka consumer as root | `kind != CLIENT` — the algorithm does not touch them (they are checked by Tests 2, 3). |

---

## 9. Relation to other tests

- **Test 2** (requests without trace_id) — complementary: Test 1 asks "I have a client, where is the server?", Test 2 asks "I have a server without a parent — who called it?". They often catch the same thing from different sides.
- **Test 4** (propagator format) — the main cause of Test 1 firing when `missing_ratio ≈ 1` and spans from the callee exist somewhere else.
- **Test 9** (sampling propagation) — produces false positives in Test 1 if `sampled=false` is not filtered out.

---

## 10. What this test does not cover

- Instrumentation performance (overhead) — a separate task.
- Correctness of attributes — Test 5.
- Correctness of span timing (clock skew) — a separate test.
- Trace completeness from a business-logic standpoint (whether all expected steps were present) — Test 6 / 10.
