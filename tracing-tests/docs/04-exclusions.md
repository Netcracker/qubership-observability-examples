# Exclusion rules for tracing validation

## Overview

Not all services and spans must undergo the same validation. Some patterns are acceptable exclusions.

---

## Exclusion types

### 1. Scheduled Jobs / Cron Tasks

**Why excluded:** A Cron job runs on a schedule rather than in response to an external request. A root span without parent_span_id is the expected behavior.

**How to identify:**
- Service name contains "job", "cron", "scheduler"
- Span name contains "scheduled", "cron", "job"
- Special attribute: `span.kind = internal` or a custom tag

**Configuration:**
```yaml
exclusions:
  - type: scheduled_job
    match:
      - resource.service.name =~ ".*-job$"
      - resource.service.name =~ ".*-scheduler$"
      - span.name =~ "^cron:.*"
    skip_rules:
      - span_without_parent
      - requests_without_trace_id
```

---

### 2. Message Consumers (Kafka, RabbitMQ)

**Why excluded:** A consumer may process a message that was sent long ago. The trace context from the producer may be unavailable or intentionally not used (a new trace chain).

**How to identify:**
- Span kind = CONSUMER
- Presence of messaging.* attributes
- Service name contains "consumer", "listener"

**Configuration:**
```yaml
exclusions:
  - type: message_consumer
    match:
      - span.kind = consumer
      - span.messaging.system != ""
    skip_rules:
      - span_without_parent  # Allowed for async processing
    still_validate:
      - required_attributes  # messaging.* attributes must be present
      - client_server_matching  # Downstream calls must be linked
```

**Important:** A consumer span must become the parent for downstream calls. Ensure that the context is propagated further.

---

### 3. External API Calls (Incoming)

**Why excluded:** External systems (partners, legacy clients) may not support trace context.

**How to identify:**
- The request comes through the edge gateway
- Source IP is not from the internal network
- No traceparent header

**Configuration:**
```yaml
exclusions:
  - type: external_incoming
    match:
      - resource.service.name = "api-gateway"
      - resource.service.name = "ingress"
      - span.http.client_ip !~ "10\\..*|192\\.168\\..*"
    skip_rules:
      - requests_without_trace_id
    action: create_new_trace  # Gateway must create a new trace
```

---

### 4. Health Checks / Readiness Probes

**Why excluded:** Health check requests typically:
- Have no trace context
- Must not create spans (spam in the tracing backend)

**How to identify:**
- HTTP path: /health, /healthz, /ready, /live, /ping
- User-Agent: kube-probe, Kubernetes-probe

**Configuration:**
```yaml
exclusions:
  - type: health_check
    match:
      - span.http.route =~ "^/(health|healthz|ready|live|ping)$"
      - span.http.user_agent =~ ".*kube-probe.*"
    action: skip_tracing  # Do not create spans at all
```

**Recommendation:** Configure the SDK/instrumentation to exclude health check endpoints:

```python
# Python OpenTelemetry
from opentelemetry.instrumentation.flask import FlaskInstrumentor

def request_hook(span, environ):
    if environ.get('PATH_INFO') in ['/health', '/ready']:
        span.set_attribute('tracing.excluded', True)

FlaskInstrumentor().instrument(request_hook=request_hook)
```

---

### 5. Internal Metrics / Admin Endpoints

**Why excluded:** Prometheus scrape, admin endpoints — internal traffic.

**How to identify:**
- HTTP path: /metrics, /admin/*, /actuator/*
- Source: prometheus, grafana-agent

**Configuration:**
```yaml
exclusions:
  - type: metrics_endpoint
    match:
      - span.http.route =~ "^/(metrics|admin|actuator)/.*"
    action: skip_tracing
```

---

### 6. Batch Processing Jobs

**Why excluded:** A batch job may process data without an incoming request.

**How to identify:**
- Long-running process
- Span duration > 1 minute
- Service name contains "batch", "etl", "import", "export"

**Configuration:**
```yaml
exclusions:
  - type: batch_job
    match:
      - resource.service.name =~ ".*-batch$"
      - resource.service.name =~ ".*-etl$"
    skip_rules:
      - span_without_parent
      - trace_depth_anomaly  # Batch may have a simple structure
```

---

## Configuration format

### Full structure

```yaml
# tracing-validation-config.yaml
version: "1.0"

# Global settings
settings:
  default_severity: WARNING
  trace_completeness_threshold: 0.95
  max_allowed_orphan_ratio: 0.05

# Exclusion rules
exclusions:
  - name: scheduled-jobs
    type: scheduled_job
    description: "Cron jobs and scheduled tasks"
    match:
      any:  # OR condition
        - resource.service.name =~ ".*-job$"
        - resource.service.name =~ ".*-cron$"
        - span.name =~ "^scheduled:.*"
    skip_rules:
      - span_without_parent
      - requests_without_trace_id

  - name: kafka-consumers
    type: message_consumer
    description: "Kafka consumer services"
    match:
      all:  # AND condition
        - span.kind = consumer
        - span.messaging.system = "kafka"
    skip_rules:
      - span_without_parent
    still_validate:
      - required_attributes
      - downstream_propagation

  - name: external-api-gateway
    type: external_incoming
    description: "External requests via API Gateway"
    match:
      all:
        - resource.service.name = "api-gateway"
        - span.kind = server
        - nestedSetParent = -1
    skip_rules:
      - requests_without_trace_id
    metadata:
      expected_action: "create_new_trace"

  - name: health-endpoints
    type: health_check
    description: "Kubernetes probes"
    match:
      any:
        - span.http.route =~ "^/(health|healthz|ready|live|ping)$"
        - span.http.route = "/actuator/health"
    action: skip_entirely  # Do not analyze these spans

# Service whitelist (only these services are validated)
# If not specified — all services are validated
service_whitelist:
  - order-service
  - payment-service
  - user-service
  - notification-service

# Service blacklist (these services are not validated)
service_blacklist:
  - legacy-monolith  # Not instrumented, no plans to instrument
  - third-party-adapter  # External code, cannot modify
```

---

## Whitelist vs Blacklist

### Whitelist approach

**When to use:**
- A new system with few services
- Strict control is desired
- Gradual rollout of validation

**Pros:**
- Explicit — we know exactly what is validated
- No false positives from new services

**Cons:**
- Needs to be updated when services are added
- A new service may be forgotten

```yaml
service_whitelist:
  - order-service
  - payment-service
  # New services must be added manually
```

---

### Blacklist approach

**When to use:**
- A large system with many services
- Most services must be validated
- Exclusions are rare

**Pros:**
- New services are validated automatically
- Less maintenance

**Cons:**
- False positives from new/test services
- Must remember to add legacy to the blacklist

```yaml
service_blacklist:
  - legacy-monolith
  - test-service
  - mock-*  # Pattern matching
```

---

### Recommendation

**Combined approach:**

```yaml
# Whitelist for production-critical services (strict validation)
strict_validation:
  services:
    - payment-service
    - order-service
  rules: all

# Default validation for the rest (relaxed)
default_validation:
  exclude:
    - "*-mock"
    - "*-test"
    - legacy-*
  rules:
    - required_attributes
    - client_server_matching

# Blacklist for full exclusion
skip_entirely:
  - third-party-adapter
  - monitoring-agent
```

---

## Applying exclusions in code

### Python example

```python
class ExclusionMatcher:
    def __init__(self, config: dict):
        self.exclusions = config.get('exclusions', [])

    def should_skip(self, span: Span, rule: str) -> bool:
        for exclusion in self.exclusions:
            if self._matches(span, exclusion['match']):
                if rule in exclusion.get('skip_rules', []):
                    return True
        return False

    def _matches(self, span: Span, match_config: dict) -> bool:
        if 'any' in match_config:
            return any(self._evaluate(span, cond) for cond in match_config['any'])
        if 'all' in match_config:
            return all(self._evaluate(span, cond) for cond in match_config['all'])
        return self._evaluate(span, match_config)

    def _evaluate(self, span: Span, condition: str) -> bool:
        # Parse condition like "resource.service.name =~ .*-job$"
        field, operator, value = parse_condition(condition)
        actual_value = get_span_field(span, field)

        if operator == '=':
            return actual_value == value
        elif operator == '!=':
            return actual_value != value
        elif operator == '=~':
            return re.match(value, actual_value) is not None
        elif operator == '!~':
            return re.match(value, actual_value) is None

        return False
```

### Usage

```python
exclusion_matcher = ExclusionMatcher(load_config())

def validate_span(span: Span, rules: List[ValidationRule]) -> List[ValidationResult]:
    results = []
    for rule in rules:
        if exclusion_matcher.should_skip(span, rule.name):
            continue
        results.extend(rule.validate(span))
    return results
```

---

## Logging exclusions

It is important to log when an exclusion is applied, for auditing:

```python
def validate_with_logging(span: Span, rules: List[ValidationRule]) -> List[ValidationResult]:
    results = []
    for rule in rules:
        if exclusion_matcher.should_skip(span, rule.name):
            logger.debug(
                f"Skipping rule '{rule.name}' for span {span.span_id} "
                f"(service: {span.service_name}, exclusion matched)"
            )
            continue
        results.extend(rule.validate(span))
    return results
```

**Monitoring metric:**
```
tracing_validation_exclusions_total{service="...", rule="...", exclusion_type="..."}
```
