# TraceQL queries for tracing validation

## TraceQL overview

TraceQL is the Grafana Tempo query language for searching spans by attributes. Documentation: [Grafana Tempo TraceQL](https://grafana.com/docs/tempo/latest/traceql/)

---

## Queries by validation test

### Test 1: Client span → Server span matching

**Find all client spans:**
```traceql
{ span.kind = client }
```

**Find client spans for a specific service:**
```traceql
{ span.kind = client && resource.service.name = "order-service" }
```

**Matching verification requires programmatic analysis:**
```python
# Pseudo-code for analysis
for client_span in get_spans(kind="client"):
    server_spans = get_spans(
        parent_span_id=client_span.span_id,
        kind="server"
    )
    if not server_spans:
        report_issue(client_span, "missing_server_span")
```

---

### Test 2: Requests without trace_id

**Server spans without a parent (potentially without trace context):**
```traceql
{ span.kind = server && nestedSetParent = -1 }
```

**Exclude the edge gateway:**
```traceql
{ span.kind = server && nestedSetParent = -1 && resource.service.name != "api-gateway" }
```

**With a time filter (via Tempo API):**
```bash
curl "http://tempo:3200/api/search?q={span.kind=server && nestedSetParent=-1}&start=$(date -d '1 hour ago' +%s)&end=$(date +%s)"
```

---

### Test 3: Span without parent_span_id

**Root spans (first span in the chain):**
```traceql
{ nestedSetParent = -1 }
```

**Suspicious root spans (not from edge services):**
```traceql
{
  nestedSetParent = -1 &&
  resource.service.name !~ "gateway|ingress|edge"
}
```

---

### Test 4: Propagator format consistency

TraceQL cannot directly check the propagation format. This is verified at the level of:
- Middleware/sidecar logging
- Application logs
- Network capture

**Indirect check — spans with a broken parent:**
```traceql
{ status = error && name =~ ".*context.*" }
```

---

### Test 5: Required span attributes validation

**HTTP spans without method:**
```traceql
{ span.kind = server && span.http.method = "" }
```

**HTTP spans without status code:**
```traceql
{ span.kind = server && span.http.status_code = 0 }
```

**gRPC spans without service name:**
```traceql
{ span.rpc.system = "grpc" && span.rpc.service = "" }
```

**Database spans without db.system:**
```traceql
{ name =~ ".*query.*|.*SELECT.*" && span.db.system = "" }
```

**Spans without service.name in resource:**
```traceql
{ resource.service.name = "" }
```

---

### Test 6: Trace depth anomaly detection

**Traces with a small number of spans:**

TraceQL does not support aggregations. Use the Tempo API + post-processing:

```bash
# Fetch traces and count spans
curl "http://tempo:3200/api/search?q={resource.service.name=\"order-service\"}&limit=100" | \
  jq '.traces[] | {traceID, spanCount: .rootServiceName}'
```

**Programmatic analysis:**
```python
def check_trace_depth(trace_id, expected_min_depth=5):
    trace = tempo_client.get_trace(trace_id)
    actual_depth = calculate_depth(trace)
    if actual_depth < expected_min_depth:
        report_issue(trace_id, f"Shallow trace: {actual_depth} spans, expected >= {expected_min_depth}")
```

---

### Test 7: Hanging spans detection

**Spans with a very large duration (possibly not finished):**
```traceql
{ duration > 5m }
```

**Spans with unset status and a large duration:**
```traceql
{ status = unset && duration > 1m }
```

**Error spans with timeout:**
```traceql
{ status = error && name =~ ".*timeout.*" }
```

---

### Test 8: Duplicate span_id detection

TraceQL does not support GROUP BY. Use the Tempo API + post-processing:

```python
def find_duplicate_spans(trace_id):
    trace = tempo_client.get_trace(trace_id)
    span_ids = [span.span_id for span in trace.spans]
    duplicates = [id for id, count in Counter(span_ids).items() if count > 1]
    return duplicates
```

---

### Test 9: Sampling decision propagation

**Verification by comparing span counts:**

```python
def check_sampling_propagation(trace_id):
    trace = tempo_client.get_trace(trace_id)
    client_spans = [s for s in trace.spans if s.kind == "client"]
    server_spans = [s for s in trace.spans if s.kind == "server"]

    for client in client_spans:
        matching_server = find_matching_server(client, server_spans)
        if not matching_server:
            # The sampling flag may not have been propagated
            report_issue(client, "possible_sampling_loss")
```

---

### Test 10: Trace completeness ratio

**Query for collecting statistics:**

```python
def calculate_completeness_ratio(time_range):
    traces = tempo_client.search(time_range)
    complete = 0
    incomplete = 0

    for trace in traces:
        if is_complete(trace):
            complete += 1
        else:
            incomplete += 1

    return complete / (complete + incomplete)

def is_complete(trace):
    # Completeness criteria:
    # 1. A root span is present
    # 2. All client spans have matching server spans
    # 3. No orphan spans
    ...
```

---

## Useful debugging queries

### Find all spans for a service
```traceql
{ resource.service.name = "order-service" }
```

### Find error spans
```traceql
{ status = error }
```

### Find spans by HTTP endpoint
```traceql
{ span.http.route = "/api/orders/{id}" }
```

### Find slow spans
```traceql
{ duration > 1s }
```

### Find spans with a specific attribute
```traceql
{ span.custom.attribute = "value" }
```

### Combined query
```traceql
{
  resource.service.name = "payment-service" &&
  span.kind = server &&
  status = error &&
  duration > 500ms
}
```

---

## Grafana Dashboard examples

### Panel: Orphan spans by service

**Query:**
```traceql
{ nestedSetParent = -1 && resource.service.name != "api-gateway" } | count() by (resource.service.name)
```

**Visualization:** Bar chart

---

### Panel: Error rate by service

**Query:**
```traceql
{ status = error } | count() by (resource.service.name)
```

**Visualization:** Time series

---

### Panel: Missing attributes

**Query (multiple):**
```traceql
# Missing http.method
{ span.kind = server && span.http.method = "" } | count() by (resource.service.name)

# Missing http.status_code
{ span.kind = server && span.http.status_code = 0 } | count() by (resource.service.name)
```

**Visualization:** Table

---

## Alerting Rules

### Prometheus/Alertmanager

Tempo can generate metrics via metrics-generator:

```yaml
# prometheus-rules.yaml
groups:
  - name: tracing-validation
    rules:
      - alert: HighOrphanSpanRate
        expr: |
          sum(rate(tempo_spanmetrics_calls_total{span_status="ok", parent_span_id=""}[5m])) by (service_name)
          /
          sum(rate(tempo_spanmetrics_calls_total{span_status="ok"}[5m])) by (service_name)
          > 0.1
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "High orphan span rate for {{ $labels.service_name }}"
          description: "More than 10% of spans don't have parent spans"

      - alert: MissingServerSpans
        expr: |
          sum(rate(tempo_spanmetrics_calls_total{span_kind="client"}[5m])) by (service_name)
          -
          sum(rate(tempo_spanmetrics_calls_total{span_kind="server"}[5m])) by (service_name)
          > 100
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "Missing server spans for {{ $labels.service_name }}"
```

---

## Tempo API Reference

### Search traces
```bash
GET /api/search?q={query}&start={start}&end={end}&limit={limit}
```

### Get trace by ID
```bash
GET /api/traces/{traceID}
```

### Get trace in different format
```bash
GET /api/traces/{traceID}?accept=application/json
GET /api/traces/{traceID}?accept=application/protobuf
```

### Search tags
```bash
GET /api/search/tags
GET /api/v2/search/tags?scope=resource
GET /api/v2/search/tags?scope=span
```

### Tag values
```bash
GET /api/search/tag/{tagName}/values
```
