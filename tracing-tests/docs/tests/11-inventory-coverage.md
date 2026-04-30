# Tests 11–13: Inventory coverage

**Status:** draft (ideas for further design).
**Severity:** Test 11 — MEDIUM, Test 12 — LOW, Test 13 — HIGH.
**Difference from Tests 1–10:** these tests do not work with trace structure; instead, they reconcile the **set of services in Tempo** against an external source of truth (K8s API). They catch problems that Tests 1–10 miss by construction.

---

## Motivation

Tests 1–10 analyze traces from the inside: "for each client span, is there a server span", "does the span have a parent", etc. They depend on actual traffic flowing through the system within the analysis window.

**What they miss:**
- A "silent" service that formally works but nobody calls it within the window. No spans — nothing to check.
- A service that writes traces but under the "wrong" `service.name` (typo, mismatch with the Deployment).
- Zombie sources: a deleted/renamed service whose traces are still arriving (old Collector buffer, dead staging environment).
- Violation of an internal contract: the service declares support for tracing (`ENABLE_TRACING=true`) but sends nothing.

Inventory tests close these gaps.

---

## Pod ↔ Tempo record mapping

**Recommended standard** (OpenTelemetry semantic conventions, no custom inventions):

| Resource attribute | Value |
|---|---|
| `service.name` | application name (usually = Deployment name) |
| `service.instance.id` | pod name (unique instance ID) |
| `k8s.pod.name` | pod name |
| `k8s.pod.uid` | pod UID (stable against recreation with the same name) |
| `k8s.namespace.name` | namespace |
| `k8s.deployment.name` | Deployment name |

**How to set them without modifying service code** (in decreasing order of intrusiveness):

1. **k8sattributes processor in OTel Collector** — enriches traces with k8s metadata on the Collector side based on the sender's source IP. Zero changes in the services. **This is the recommended option for a black-box approach.**
2. **Downward API + `OTEL_RESOURCE_ATTRIBUTES`** in the Deployment manifest (3 env vars, no code changes):
   ```yaml
   env:
     - name: POD_NAME
       valueFrom: { fieldRef: { fieldPath: metadata.name } }
     - name: POD_NAMESPACE
       valueFrom: { fieldRef: { fieldPath: metadata.namespace } }
     - name: OTEL_RESOURCE_ATTRIBUTES
       value: "k8s.pod.name=$(POD_NAME),k8s.namespace.name=$(POD_NAMESPACE),service.instance.id=$(POD_NAME)"
   ```
3. **OpenTelemetry Operator** with auto-instrumentation — a sidecar/init container automatically injects the attributes.

**Mapping in the test:**
- primary key: `(k8s.namespace.name, k8s.pod.name)` from spans ⇄ `(metadata.namespace, metadata.name)` from `kubectl get pods`;
- fallback when k8s attributes are absent: `service.name` from spans ⇄ Deployment name (more fragile, but works).

If not a single k8s attribute exists in the entire cluster — Inventory tests are **not applicable**. Enabling them = an implicit requirement for the cluster to have populated k8s resource attributes.

---

## Required access for the validator

In addition to what is required for Tests 1–10:

| Access | Why |
|---|---|
| Read-only K8s API: `pods`, `deployments`, `namespaces` | Inventory: list of entities, env vars, labels/annotations |
| (already present) read-only Tempo API | List of active `service.name` / `k8s.pod.name` within the window |

For the validator's ServiceAccount: `get,list,watch` on `pods` and `deployments` in the analyzed namespaces. No write permissions.

---

## Test 11: every pod → ≥1 record in Tempo

**What it detects:** services/pods that are running in the cluster but do not send a single span within the analysis window.

### Algorithm

1. Obtain the inventory from K8s:
   ```
   pods_in_cluster = { (ns, pod_name) } for all pods in Running state
                     excluding namespaces in namespace_blacklist
                     excluding pods matching pod_blacklist
   ```
2. Obtain active pods from Tempo within the window:
   ```
   pods_in_tempo = distinct (k8s.namespace.name, k8s.pod.name)
                   from spans where start_time in [window_start, window_end]
   ```
3. Compute `pods_in_cluster \ pods_in_tempo` — pods without traces.

### Filtering out legitimate cases

Without this, the test catches half of the cluster:

| Condition | Why we skip it |
|---|---|
| Pod was created less than `min_pod_age` ago (e.g., 10 minutes) | Just started, hasn't had time to process anything yet. |
| Pod label/annotation explicitly marks "no tracing" (`tracing.company.io/enabled=false` or similar) | Explicit opt-out. |
| Pod is in `system_namespaces` (`kube-system`, `istio-system`, monitoring operators) | Infrastructure, usually without tracing. |
| Pod matches `pod_blacklist` from the config | Explicit exclusion (legacy, third-party images). |
| This is a CronJob/Job that has not completed its first run within the window | Naturally no traffic. |
| Ready, but traffic policy is `0% traffic` (canary/stopped) | No incoming traffic = no traces. |

### Output (issue)

```json
{
  "rule": "inventory_pod_to_trace",
  "severity": "WARNING",
  "namespace": "payments",
  "pod": "notification-service-7d8f6-xk2lm",
  "deployment": "notification-service",
  "pod_age": "4h12m",
  "last_span_seen": null,
  "hypothesis": "service not instrumented or not sending to Collector"
}
```

### Limitations

- Does not distinguish "not instrumented" vs. "no traffic within the window". For a stricter check — see **Test 13**.
- Requires k8s resource attributes (see mapping). Without them the test degenerates into comparing `service.name` ↔ Deployment.

---

## Test 12: Tempo has no records from services outside k8s

**What it detects:** traces from sources that do not exist in the cluster.

### Potential causes of this rule firing (for the inverse `pods_in_tempo \ pods_in_cluster`)

1. **Typo in `service.name`:** the service is running but sends spans with `service.name=notificaton-service` (with a typo) — no Deployment matches.
2. **Zombie source:** the service has been removed from the cluster, but a Collector/agent somewhere is still buffering old spans, or staging from another environment sends to the same Tempo.
3. **Renaming:** a Deployment has been renamed, the new `service.name` has not yet been set in the instrumentation.
4. **External source:** another cluster / dev laptop / test environment writes to this Tempo due to a configuration error.
5. **A pod died just now, but the analysis window still includes its spans:** legitimate race condition.

### Algorithm

```
services_in_tempo  = distinct service.name (or (ns, pod_name)) within the window
services_in_k8s    = all Deployment.name (or (ns, Pod.name)) at the current moment,
                     plus pod_names of pods that were alive within the window
                     (via K8s events / audit log, if available)

suspicious = services_in_tempo \ services_in_k8s
           \ known_external_sources   # whitelist ingress-gateway, edge
```

### Filtering out legitimate cases

| Condition | Why we skip it |
|---|---|
| `service.name` ∈ `external_sources_whitelist` | CDN, Envoy edge, partner integrations. |
| Pod was alive within the window but deleted before the check | Race condition, not an issue. Check via pod history (K8s events API). |
| For Tests 1–10 this service acts as a "correct caller", i.e., is connected to the rest of the topology | Likely a typo rather than an external zombie — record separately with `hypothesis=typo_in_service_name`. |

### Output

```json
{
  "rule": "inventory_trace_to_pod",
  "severity": "INFO",
  "service_name_in_tempo": "notificaton-service",
  "span_count_in_window": 3421,
  "closest_k8s_deployment": "notification-service",
  "edit_distance": 1,
  "hypothesis": "likely typo in service.name (edit distance 1 to existing deployment)"
}
```

### Limitations

- Low severity (severity INFO) — rarely blocks anything, but indicates drift between the inventory and reality.
- Requires a stable `external_sources_whitelist`, otherwise noisy.

---

## Test 13: `ENABLE_TRACING=true` → traces are mandatory

**What it detects:** violation of the company's internal contract. The service has declared support for tracing (via env var) but sends nothing — an explicit instrumentation or configuration bug, not "legitimate silence".

This is the **strict form of Test 11**: there "no traces" is a warning (may be legitimate), here it is an error (contract violated).

### Precondition

Company standard: a service supporting tracing reads the `ENABLE_TRACING` variable (`true`/`false`). Presence of the variable with value `true` in the Pod spec = contract "I send traces".

### Algorithm

1. Collect from the K8s API:
   ```
   tracing_enabled_pods = {
       (ns, pod_name) for all Running pods
       where any container has env ENABLE_TRACING == "true"
                                (or resolved ConfigMap/Secret ref == "true")
   }
   ```
   Important nuance: `ENABLE_TRACING` can come from a ConfigMap/Secret via `envFrom` or `valueFrom.configMapKeyRef` — it needs to be resolved, otherwise coverage is incomplete.
2. Obtain `pods_in_tempo` for the window (as in Test 11).
3. Issue = `tracing_enabled_pods \ pods_in_tempo` taking into account `min_pod_age` and other filters from Test 11.

### Output

```json
{
  "rule": "enable_tracing_contract_violation",
  "severity": "ERROR",
  "namespace": "payments",
  "pod": "notification-service-7d8f6-xk2lm",
  "deployment": "notification-service",
  "enable_tracing_source": "env[ENABLE_TRACING]=true (direct)",
  "pod_age": "6h30m",
  "last_span_seen": null,
  "hypothesis": "contract violation: ENABLE_TRACING=true but no spans in window"
}
```

### Potential causes of this rule firing

1. The service is genuinely not instrumented (the OTel SDK was forgotten, though the env var was set in advance).
2. Instrumentation exists, but the SDK does not read this specific env var (a different flag internally).
3. The service cannot reach the Collector (network policy, incorrect `OTEL_EXPORTER_OTLP_ENDPOINT`).
4. Sampling ratio = 0 on this service.
5. A custom filter in the application drops all spans.

The hypothesis can be refined via cross-checks: inspect Collector logs for rejected spans from this pod IP, check network reachability, etc. — this goes beyond the black-box test, but is useful for a runbook.

### Limitations

- Works only if the company actually follows the `ENABLE_TRACING` standard. If the standard is young and not widely adopted — lower severity to WARNING and apply only to a whitelist of services.
- Does not cover the case "instrumentation is broken but ENABLE_TRACING is not set" — for that, Test 11 or Test 1 is needed.

---

## Relation to other tests

| With what | How it relates |
|---|---|
| Test 1 (client → server) | Complementary. Test 1 — "callee does not respond with a span when there is a call". Tests 11/13 — "the service sends nothing at all". Intersection: if a service is fully uninstrumented, Test 1 will point at it as a missing callee, and Test 13 — as a contract violation. Test 13 is **more actionable**: an explicit cause (contract violated), without hypotheses. |
| Test 4 (propagator consistency) | Orthogonal. |
| Exclusion rules from `04-exclusions.md` | Apply to Tests 11/13 first and foremost: `service_blacklist` = `pod_blacklist`, `namespace_blacklist` — a new configuration field. |

---

## Open questions

1. **What analysis window?** For rare crons (once a day), a window < 24 h will produce false positives for Test 11. Options: per-service window, or a filter by `schedule` from the CronJob spec.
2. **How to tell `ENABLE_TRACING=true` from a ConfigMap vs. inline?** Technically — resolve references via the K8s API. Practically — is inline enough, or is full resolving needed? A question for the company's real-world practice.
3. **Is the K8s events API needed?** For Test 12 it is desirable to know the history of pod deletions within the window (not just the current snapshot). Alternative — run the validator more frequently and store pod snapshots yourself.
4. **Is correlation with Collector metrics also needed?** If the Collector reports `otelcol_receiver_accepted_spans{service_name=X} > 0`, but nothing is stored in Tempo — this is an issue of a different level (exporter pipeline), not about the service itself.

These questions will be resolved when the draft moves to a runnable specification.
