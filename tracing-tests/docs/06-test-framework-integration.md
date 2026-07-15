# Test framework integration

## Context

The company is developing a wrapper on top of Playwright (the `testing-platform*` family of repositories). At the framework level, there is the ability to create traces and propagate `traceparent` into the application under test:

- **test scenario** = trace;
- **test step** = span;
- the framework acts as the root service of the trace, identifying itself with a name like `e2e-runner`.

The question arises: if the framework already creates spans for each "system call" step, then it knows about the expected trace structure. Some tracing validation checks can be done directly in the framework, rather than only in the Go CLI (`tracing-validator`).

---

## Responsibility split

The solution is **hybrid**, but not 50/50. Checks are split not by stage (before/after the run), but by **type**.

### Into the framework (inline, per-step)

Structural assertions at the level of "my step produced the expected server span":

- client span from the framework ⇒ server span in the named service within N seconds;
- span exists with the expected attributes (`http.route`, `http.status_code`);
- the call chain for this step has the expected depth (`gateway → service-a → service-b`).

Execution: immediately after the step runs, via polling of the Tempo search API.
Behavior: fail-fast — the test fails exactly on the problematic step, the stack trace points to the specific location in the scenario.
Code size: ~200 lines of TS. This is a thin layer of polling + fluent assertion API.

### Into the Go CLI (aggregate, post-run)

Everything that requires a top-down view or data sources outside of traces:

- **Inventory tests 11–13** — reconciliation with the K8s API; the framework does not need to know about K8s.
- **Aggregate rules for the run:** completeness ratio, propagator format consistency, orphan span counts, coverage of required attributes.
- **Cross-test analysis:** "in 3 out of 10 scenarios, context is lost on service Y" — visible only after the run.
- **The same rules** are later used in offline mode on production traffic — one codebase.

Execution: a separate CI step after all tests in the run complete.

---

## Why not "everything in the framework"

1. **Fragmented `testing-platform*` repositories.** If validation rules live in the framework core, they must be consistent across all wrappers and plugins. In practice, this means drift: one wrapper has updated to v2 rules, another is stuck on v1. With the Go CLI — one source of truth; wrappers only inject trace context and emit `test.run_id`. There is nothing to drift.

2. **Inventory tests 11–13 are not accessible to the framework by definition.** The check "the pod has `ENABLE_TRACING=true` but is not emitting spans" is about the cluster, not about the test run. The framework does not know about the cluster, and should not.

3. **Rules evolve.** Tracing policies, exclusions, thresholds change. In the Go CLI — a targeted change + redeploy. In the framework — a release + pinning the version in N repos + rollout.

4. **Autotest developers write in TS/JS.** They should not be forced to become experts in TraceQL and OTel semantics. The API must be simple: `await trace.expectCalled('order-service', 'POST /orders')`.

---

## Why not "everything in the Go CLI"

1. **Feedback loop.** If validation is only post-run, the developer sees "19 out of 20 scenarios passed, but trace-validator found 7 issues" — the link between issue and specific step is lost. A per-step assertion fails exactly where the problem is.
2. **Cutting off the cascade.** If step 3 out of 10 broke propagation, the next 7 steps will produce broken spans. Fail-fast saves CI time.
3. **Natural contextual information** (which step, which scenario) already exists in the framework. Reconstructing it from labels in the CLI is extra code.

---

## Attribute contract

So that the Go CLI can filter and attribute spans to a specific test run, the framework must set the following attributes. This is the **only formal contract** between the framework and the CLI.

| Attribute | Value | Why |
|---|---|---|
| `test.run_id` | UUID of the CI run (stable for the entire run) | Filter `{ resource.test.run_id = "..." }` in Tempo; isolation of test-run from prod traffic |
| `test.suite` | suite name | Attribution of issues to a suite |
| `test.name` | scenario name | Attribution to a scenario |
| `test.step` | step name/index | Localization down to the step in the aggregate report |
| `service.name` | stable framework name (`e2e-runner`) | So the CLI doesn't confuse the framework with a real service during matching |
| traceparent with `sampled=01` | always for test-run | Otherwise spans won't reach Tempo |

**Where to set:** Resource attributes in OTel SDK init, plus `test.step` as a span attribute of each step.

**Warning:** `test.run_id` **must** be the stable ID of the CI run itself (e.g., `$CI_PIPELINE_ID` + uuid), not regenerated in each step. Otherwise, the CLI will not be able to assemble the run into a single whole.

---

## Framework assertion API (sketch)

```typescript
import { trace } from '@testing-platform/tracing';

test('create order flow', async () => {
  await trace.step('create order', async () => {
    const res = await api.post('/orders', { itemId: 42 });

    // per-step assertion — simple fluent API
    await trace.expect()
      .serverSpanIn('order-service')
      .forClientCall('POST', '/orders')
      .toExistWithin('10s');

    // can chain: child call to payment-service
    await trace.expect()
      .serverSpanIn('payment-service')
      .asDescendantOf('order-service')
      .toExistWithin('10s');
  });
});
```

**Under the hood of `toExistWithin()`:**

1. Take the `trace_id` of the current step from the OTel context.
2. Poll `GET /api/traces/{trace_id}` with back-off 1s → 2s → 5s until the deadline.
3. Apply local matching (the same logic as in the Go CLI for Test 1, but only for a single expected span).
4. On expiration — fail with a detailed message: "server span `POST /orders` in `order-service` not found within 10s; the last trace snapshot contains: ...".

The API surface is intentionally narrow: the framework should not try to express all 13 rules. Only basic structural expectations for a step.

---

## CI flow

```
┌──────────────────────────────────────────────────────────────┐
│ CI pipeline                                                  │
│                                                              │
│ 1. deploy to staging                                         │
│                                                              │
│ 2. run e2e tests                                             │
│    ├─ each step injects traceparent with test.run_id=$RUN_ID │
│    ├─ per-step assertions (trace.expect...toExistWithin)     │
│    │  └─ fail-fast: fails on the problematic step            │
│    └─ CI step fails if at least one test failed              │
│                                                              │
│ 3. tracing-validator test-run --run-id=$RUN_ID               │
│    ├─ fetches all spans with test.run_id=$RUN_ID             │
│    ├─ applies Tests 1–13 to this subset                      │
│    └─ writes report.json                                     │
│                                                              │
│ 4. (optional) tracing-validator offline --since 1h           │
│    └─ in parallel on production traffic                      │
└──────────────────────────────────────────────────────────────┘
```

**Go CLI command for test-run mode:**

```bash
tracing-validator test-run \
    --tempo https://tempo.staging \
    --run-id $CI_PIPELINE_ID \
    --rules t1,t4,t5,t9,t10 \
    --out report.json \
    --fail-on-severity ERROR
```

Inventory tests (11–13) in test-run mode are usually **not** applicable: they are about cluster state, not about a specific run. They are run as a separate nightly CronJob against the entire staging environment.

---

## Responsibilities by repository

| Layer | Repository (approximately) | Responsibility |
|---|---|---|
| Framework core | `testing-platform-core` | OTel SDK init; setting contract attributes (`test.run_id`, `test.suite`, etc.); injecting `traceparent` into the framework's HTTP clients; fluent assertion API (`trace.expect()...`) |
| Wrappers / plugins | `testing-platform-*` | **Nothing trace-specific.** Use the core API as a black box |
| Report storage | separate repo or artifact | Store `report.json` from the CLI, aggregate across runs |
| Validator | `tracing-validator` (Go CLI) | All rules T1–T13, exclusion engine, reports |

Key principle: **any new validation rule is added to the Go CLI**, not to the framework. The framework changes only when the attribute contract changes — which should happen rarely.

---

## What not to build into the framework

Common mistakes:

1. **Implementing a TraceQL-like DSL in TS.** Overengineering. If an assertion is more complex than "span exists as a descendant of X" — it is an aggregate rule, its place is in the CLI.
2. **Polling via WebSocket stream.** Tempo search REST + back-off is sufficient.
3. **Keeping a trace snapshot in the framework's state.** Each assertion is independent, reads a fresh trace by `trace_id`.
4. **Checking cross-test invariants** ("no service should appear orphan in more than 5% of runs"). This is purely aggregate logic, its place is in the CLI.
5. **Embedding business assertions into trace checks.** "The order was created in the DB" is an API/DB assertion, not about tracing. Don't mix.

---

## Open questions

1. **Minimum version of Tempo/OTel Collector.** Filtering by `resource.test.run_id` requires that this be a resource attribute, not a span attribute — otherwise the TraceQL filter will be inefficient. You need to verify that this is actionable in your current version of Tempo.
2. **Sampling for test-run.** Forcing always-on for traces with `test.run_id` — this is either an OTel Collector `tail_sampling` policy, or the framework sets `01` in `traceparent`. The latter is simpler and more reliable.
3. **Isolation of test traffic in Tempo.** Ideally, staging Tempo accepts both prod traffic and test runs. To keep test runs from polluting the aggregate statistics of Tests 1–10 in offline mode, filter them by `test.run_id != ""`.
4. **What to do with flaky assertions?** If `toExistWithin(10s)` periodically fails due to ingest lag — increase the timeout to 30–60s or add retries. Do not add "soft fail" — this dilutes the signal.
5. **How to reuse the CLI rules inside the framework for per-step assertions?** One option — extract the rules into a YAML specification, parse from both tools. Overhead upfront, may be useful later if there are many rules.
