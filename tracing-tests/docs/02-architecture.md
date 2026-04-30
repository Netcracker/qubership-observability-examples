# Tracing validation prototype architecture

## Overview

The tracing validation system analyzes data from Grafana Tempo and identifies microservice instrumentation issues.

---

## System components

```
┌─────────────────┐     ┌──────────────────┐     ┌─────────────────┐
│   Test Runner   │────▶│  Grafana Tempo   │────▶│  Trace Analyzer │
│  (generates     │     │  (tracing        │     │  (validates     │
│   test traffic) │     │   backend)       │     │   trace rules)  │
└─────────────────┘     └──────────────────┘     └─────────────────┘
         │                       │                        │
         │                       │                        │
         ▼                       ▼                        ▼
┌─────────────────┐     ┌──────────────────┐     ┌─────────────────┐
│   Staging Env   │     │   Tempo HTTP     │     │   Report        │
│  (microservices │     │   API / TraceQL  │     │   Generator     │
│   under test)   │     │                  │     │   (JSON/HTML)   │
└─────────────────┘     └──────────────────┘     └─────────────────┘
```

---

## Two operating modes

### A) Offline Analysis (post-hoc)

Analysis of existing traces over a given period.

**Use cases:**
- Regular (daily/weekly) tracing health-check
- Incident investigation
- Post-release instrumentation quality assessment

**Process:**
```
1. Query traces from Tempo for the period (e.g., the last 24 hours)
2. For each trace, apply the set of validation rules
3. Aggregate results by service
4. Generate a report
```

**Input data:**
- Time range (start, end)
- List of services to analyze (optional)
- Set of validation rules

**Output data:**
- List of issues per service
- Quality metrics (completeness ratio, error rate)

---

### B) Real-time Validation (during test)

Tracing validation during test execution.

**Use cases:**
- Integration tests
- Smoke tests after deployment
- Validation of new services before production

**Process:**
```
1. Test Runner generates a request with a known trace_id
2. The request flows through the chain of services
3. Poll Tempo until spans appear
4. Compare the expected trace structure against the actual one
5. Fail the test if the structure does not match
```

**Input data:**
- Test request with injected trace context
- Expected trace structure (list of services in the call chain)
- Timeout for awaiting spans

**Output data:**
- PASS/FAIL for each test
- Details of discrepancies (missing spans, wrong attributes)

---

## Detailed component description

### Test Runner

Generates HTTP/gRPC requests with manual trace context injection.

**Functionality:**
- Generation of unique trace_id values
- Injection of W3C Trace Context headers
- Sending requests to the entry point
- Saving the mapping: trace_id → expected structure

**HTTP request example:**
```http
GET /api/orders/123 HTTP/1.1
Host: api-gateway.staging
traceparent: 00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01
tracestate:
```

**Code example (Python):**
```python
import uuid
import requests

def generate_trace_context():
    trace_id = uuid.uuid4().hex
    span_id = uuid.uuid4().hex[:16]
    return {
        'traceparent': f'00-{trace_id}-{span_id}-01',
        'tracestate': ''
    }, trace_id

def send_test_request(url):
    headers, trace_id = generate_trace_context()
    response = requests.get(url, headers=headers)
    return trace_id, response
```

---

### Tempo Query Client

Interacts with Grafana Tempo via HTTP API.

**Endpoints:**
| Endpoint | Description |
|----------|-------------|
| `GET /api/traces/{traceID}` | Get trace by ID |
| `GET /api/search` | Search traces by attributes |
| `GET /api/search/tags` | List available tags |
| `GET /api/v2/search/tags` | List tags with scope |

**Trace request example:**
```bash
curl -s "http://tempo:3200/api/traces/4bf92f3577b34da6a3ce929d0e0e4736" | jq
```

**Response example:**
```json
{
  "batches": [
    {
      "resource": {
        "attributes": [
          {"key": "service.name", "value": {"stringValue": "api-gateway"}}
        ]
      },
      "scopeSpans": [
        {
          "spans": [
            {
              "traceId": "4bf92f3577b34da6a3ce929d0e0e4736",
              "spanId": "00f067aa0ba902b7",
              "parentSpanId": "",
              "name": "GET /api/orders/{id}",
              "kind": 2,
              "startTimeUnixNano": "1699900000000000000",
              "endTimeUnixNano": "1699900000500000000"
            }
          ]
        }
      ]
    }
  ]
}
```

---

### Trace Analyzer

Applies validation rules to traces.

**Rule structure:**
```python
class ValidationRule:
    name: str
    description: str
    severity: Literal['ERROR', 'WARNING', 'INFO']
    applicable_to: List[str]  # service types

    def validate(self, trace: Trace) -> List[ValidationResult]
```

**Rule example (Client-Server matching):**
```python
class ClientServerMatchingRule(ValidationRule):
    name = "client_server_matching"
    description = "Client span must have a matching server span"
    severity = "ERROR"

    def validate(self, trace: Trace) -> List[ValidationResult]:
        results = []
        client_spans = [s for s in trace.spans if s.kind == SpanKind.CLIENT]

        for client_span in client_spans:
            server_span = find_server_span(trace, client_span.span_id)
            if not server_span:
                results.append(ValidationResult(
                    rule=self.name,
                    severity=self.severity,
                    span_id=client_span.span_id,
                    service=client_span.service_name,
                    message=f"Missing server span for client call to {client_span.attributes.get('http.url')}"
                ))

        return results
```

---

### Report Generator

Generates reports in various formats.

**Formats:**
- JSON — for programmatic processing
- HTML — for visual analysis
- Markdown — for inclusion in documentation/tickets

**JSON report example:**
```json
{
  "report_time": "2024-01-15T10:30:00Z",
  "period": {
    "start": "2024-01-14T00:00:00Z",
    "end": "2024-01-15T00:00:00Z"
  },
  "summary": {
    "total_traces_analyzed": 10000,
    "completeness_ratio": 0.87,
    "services_with_issues": 3,
    "total_issues": 45
  },
  "services": [
    {
      "name": "order-service",
      "status": "OK",
      "issues": []
    },
    {
      "name": "payment-service",
      "status": "WARNING",
      "issues": [
        {
          "rule": "required_attributes",
          "severity": "WARNING",
          "count": 12,
          "message": "Missing http.status_code in 12 spans"
        }
      ]
    },
    {
      "name": "notification-service",
      "status": "ERROR",
      "issues": [
        {
          "rule": "client_server_matching",
          "severity": "ERROR",
          "count": 33,
          "message": "Missing server spans for 33/100 client calls"
        }
      ]
    }
  ]
}
```

---

## CI/CD integration

### Option 1: Post-deployment validation

```yaml
# .gitlab-ci.yml
tracing-validation:
  stage: post-deploy
  script:
    - sleep 60  # Wait for traces to be ingested
    - python validate_tracing.py --trace-id $SMOKE_TEST_TRACE_ID
  allow_failure: true  # Non-blocking initially
```

### Option 2: Scheduled job

```yaml
# Kubernetes CronJob
apiVersion: batch/v1
kind: CronJob
metadata:
  name: tracing-health-check
spec:
  schedule: "0 8 * * *"  # Daily at 8:00
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: analyzer
            image: tracing-analyzer:latest
            args:
              - --mode=offline
              - --period=24h
              - --output=/reports/daily-report.json
```

---

## Scaling

### For small systems (< 100 services)

- A single analyzer instance
- Direct requests to the Tempo API
- JSON reports

### For large systems (> 100 services)

- Streaming analysis via Tempo metrics-generator
- Prometheus metrics for real-time monitoring
- Grafana dashboards for visualization
- Alertmanager for notifications

**Metrics to export:**
```
tracing_validation_issues_total{service="...", rule="...", severity="..."}
tracing_completeness_ratio{service="..."}
tracing_orphan_spans_total{service="..."}
```

---

## Security

### Tempo access

- Use a service account with read-only access
- Limit request time range (no more than 24h at once)
- Rate limiting on the analyzer side

### Report data

- Do not include sensitive data from span attributes
- Sanitize SQL queries and URLs with parameters
- Do not log full trace_id values in public reports
