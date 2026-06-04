# Metrics Implementation

How to instrument a connector for Prometheus / OpenTelemetry, following the conventions enforced by the `inorbit-connector` framework. **Read this before writing any `meter.create_*` or `metric.add` call.**

> The conventions here assume the framework provides: single `inorbit_connector` exporter namespace, `get_connector_meter()` wrapper, canonical `inorbit.connector.upstream.http.*` family with separate request/error counters, `record_upstream_http_request()` / `record_upstream_http_error()` helpers, and `EndpointMapper` / `PathTemplater` normalizers. Check your installed `inorbit-connector` version's release notes — don't backport these conventions blindly to older framework versions.

## TL;DR

```python
# 1. Get a vendor-namespaced meter — prefix is added automatically
from inorbit_connector.metrics import get_connector_meter
meter = get_connector_meter("acme")   # use your connector_type

# 2. Declare domain instruments at module scope. Names do NOT include the vendor prefix.
mission_failures = meter.create_counter(
    "mission.failures",
    unit="1",
    description="Mission executions that ended in failure (attribute: reason)",
)

# 3. Record upstream HTTP calls via the framework helpers. Success and error
#    are SEPARATE metrics. Pick the helper that fits the code path.
from inorbit_connector.metrics.http import (
    EndpointMapper,
    record_upstream_http_request,
    record_upstream_http_error,
)

_endpoint = EndpointMapper([
    ("/api/v1/missions", "missions"),
    ("/api/v1/status",   "status"),
])

async def _request(self, method, path):
    start = time.perf_counter()
    try:
        resp = await self._http.request(method, path)
        if resp.status_code >= 400:
            record_upstream_http_error(
                vendor="acme", method=method, endpoint=_endpoint(path),
                error_kind=f"http_{resp.status_code // 100}xx",
                duration_seconds=time.perf_counter() - start,
            )
        else:
            record_upstream_http_request(
                vendor="acme", method=method, endpoint=_endpoint(path),
                duration_seconds=time.perf_counter() - start,
            )
        return resp
    except httpx.TimeoutException:
        record_upstream_http_error(
            vendor="acme", method=method, endpoint=_endpoint(path),
            error_kind="timeout",
            duration_seconds=time.perf_counter() - start,
        )
        raise
    except httpx.ConnectError:
        record_upstream_http_error(
            vendor="acme", method=method, endpoint=_endpoint(path),
            error_kind="connect_error",
            duration_seconds=time.perf_counter() - start,
        )
        raise
```

That's the whole pattern. The rest of this document explains why, and what to avoid.

## What you get for free (framework-level)

If your connector subclasses `FleetConnector` or `Connector` and opts in via the `metrics:` block in YAML config, the framework emits:

| Metric | Type | Attributes | Meaning |
|---|---|---|---|
| `inorbit_connector_up` | Gauge | — | 1 while the connector's main thread is alive |
| `inorbit_connector_session_connected` | Gauge | `robot_id` | 1 when the MQTT session to InOrbit is connected |
| `inorbit_connector_execution_loop_ticks_total` | Counter | — | Successful run-loop iterations |
| `inorbit_connector_execution_loop_errors_total` | Counter | — | Exceptions caught in the run loop |
| `inorbit_connector_upstream_http_requests_total` | Counter | `vendor`, `method`, `endpoint` | Successful upstream HTTP calls (emitted by `record_upstream_http_request`) |
| `inorbit_connector_upstream_http_errors_total` | Counter | `vendor`, `method`, `endpoint`, `error_kind` | Failed upstream HTTP calls (emitted by `record_upstream_http_error`) |
| `inorbit_connector_upstream_http_duration_seconds` | Histogram | `vendor`, `method`, `endpoint` | Latency of all upstream HTTP calls (recorded on both paths) |
| `calls_publish_*_total` (8 of them) | Counter | `robot_id` | SDK-level publish call counts (pose, map, odometry, …) |

Every metric also carries Resource attributes set by `setup_prometheus_metrics`: `service.name=inorbit_connector`, `service.instance.id`, `service.version`, `inorbit.connector.type`, `inorbit.connector.id`, plus any `account_id` / `location_id` from `ConnectorRootConfig`. The reference collector promotes these to series labels — use them as filter dimensions in PromQL.

**You don't declare any of these yourself.** They're emitted by the framework on your behalf. Your job is to add domain instruments for things the framework doesn't know about (upstream API health, robot domain state, etc.).

## Two layers, two patterns

### Layer 1: domain instruments (vendor-specific)

Things only your connector knows: battery level on this fleet's API, mission queue depth, command latency to the upstream FMS. These get their own metric names with a vendor prefix.

**Get a meter via `get_connector_meter`, not `get_meter` directly:**

```python
# inorbit_connector.metrics
def get_connector_meter(connector_type: str) -> PrefixedMeter:
    """Wraps inorbit_edge.metrics.get_meter() with a PrefixedMeter that
    prepends `<connector_type>.` to every instrument name on creation."""
```

```python
# your_connector/src/metrics.py
from inorbit_connector.metrics import get_connector_meter

meter = get_connector_meter("acme")   # MUST match CONNECTOR_TYPE on your config

mission_queue_depth = meter.create_up_down_counter(
    "mission.queue_depth",            # NO "acme." prefix — wrapper adds it
    unit="1",
    description="Missions currently queued (attribute: priority)",
)
```

This exports as `inorbit_connector_acme_mission_queue_depth` in Prometheus. The vendor prefix comes from the wrapper. Two connector types declaring `mission.queue_depth` get distinct metric descriptors (`..._acme_...` vs `..._otto_...`) — no descriptor collision.

**Do not use** `inorbit_edge.metrics.get_meter("inorbit_acme_connector")` directly. The CI lint flags it. Reasons:
- Manual prefix means manual discipline; the wrapper makes the prefix structural.
- Direct `get_meter` is allowed for framework code only.

### Layer 2: canonical metrics (cross-vendor)

For dimensions every connector cares about — upstream HTTP health is the canonical example — emit through framework-provided helpers that own the metric name AND the attribute schema.

**Upstream HTTP** uses three framework instruments and two helpers — successes and errors are separate counters, with a shared duration histogram:

```python
# Declared by the framework (don't redeclare):
#   inorbit.connector.upstream.http.requests   (Counter, success path)
#   inorbit.connector.upstream.http.errors     (Counter, error path)
#   inorbit.connector.upstream.http.duration   (Histogram, both paths)

from inorbit_connector.metrics.http import (
    record_upstream_http_request,   # success path
    record_upstream_http_error,     # error path
)

# success
record_upstream_http_request(
    vendor="acme",                  # = your connector_type
    method="GET",                   # GET / POST / PUT / PATCH / DELETE / OPTIONS / HEAD
    endpoint="missions",            # MUST be normalized — see below
    duration_seconds=0.142,
)

# error (timeout, connect failure, non-2xx response, etc.)
record_upstream_http_error(
    vendor="acme",
    method="GET",
    endpoint="missions",
    error_kind="timeout",           # bounded enum: timeout / connect_error / http_4xx / http_5xx / other
    duration_seconds=0.142,
)
```

**Why split:** the success counter's descriptor stays at three attributes (`vendor`, `method`, `endpoint`) and only the error counter carries the additional `error_kind` label. Failures are rare, so the error descriptor's label space grows slowly. Total HTTP throughput is `requests + errors`. Error rate is `errors / (requests + errors)`.

The attribute schemas are frozen by the framework. You cannot add keys. If you need more dimensions for vendor-specific debugging, add a separate domain counter under Layer 1.

Cross-vendor queries work without metric-name fan-out (PromQL — metrics land in GCP as the Prometheus descriptor type via the `googlemanagedprometheus` exporter):

```promql
# Total upstream errors by vendor and kind
sum by (vendor, error_kind) (rate(inorbit_connector_upstream_http_errors_total[5m]))

# Error rate across all vendors
sum(rate(inorbit_connector_upstream_http_errors_total[5m]))
/
(sum(rate(inorbit_connector_upstream_http_requests_total[5m]))
 + sum(rate(inorbit_connector_upstream_http_errors_total[5m])))
```

## Endpoint cardinality is the #1 footgun

**Every minute of every day, somewhere, a connector author writes:**

```python
record_upstream_http_request(vendor="acme", method="GET", endpoint=path, duration_seconds=0.1)
                                                                  ^^^^
```

where `path` is `"/api/v1/missions/abc-1234-def/result"`. The descriptor's `endpoint` label then accumulates one new value per UUID forever. Stackdriver cannot un-add a label value. The metric is irreversibly polluted.

**Always normalize before passing.** Use one of the two framework helpers:

### EndpointMapper — explicit prefix table (preferred for stable APIs)

```python
from inorbit_connector.metrics.http import EndpointMapper

_endpoint = EndpointMapper([
    ("/api/v1/missions",      "missions"),
    ("/api/v1/mission_queue", "mission_queue"),
    ("/api/v1/status",        "status"),
    # ...one entry per route family
])
_endpoint("/api/v1/missions/abc-123/result")   # → "missions"
_endpoint("/api/v1/unknown")                    # → "other"
```

The mapper is declared once (typically as a module-level constant in your API client). Add a row when you add an endpoint. Unknown paths collapse to `"other"`.

### PathTemplater — automatic ID masking (preferred for evolving APIs)

```python
from inorbit_connector.metrics.http import PathTemplater

_endpoint = PathTemplater()
_endpoint("missions/abc-123-def/result")   # → "missions/{id}/result"
_endpoint("orders/42")                      # → "orders/{id}"
_endpoint("status")                         # → "status"
```

Catches UUIDs, long hex strings, and numeric segments. Less label-stable across API surface changes — when in doubt prefer the mapper.

**Never pass raw paths.** The runtime guard in `record_upstream_http_request` / `record_upstream_http_error` logs a warning if your endpoint contains both a slash and a digit, but it doesn't block. Lint catches the literal-path case at CI time. Discipline catches the rest.

## Cardinality discipline (general)

Every attribute key on every counter must come from a bounded enum. Bounded = "I could write down all possible values on a Post-it." Unbounded = "this value contains user input, timestamps, exception messages, or upstream IDs."

| Attribute style | Bounded? | Notes |
|---|---|---|
| `method` ∈ `{GET, POST, PUT, ...}` | ✓ | HTTP method enum |
| `endpoint` ∈ `{missions, status, ...}` (after normalizer) | ✓ | Normalized via EndpointMapper |
| `error_kind` ∈ `{timeout, connect_error, http_4xx, http_5xx, other}` | ✓ | Framework convention on `record_upstream_http_error` |
| `result` ∈ `{success, failure, missing, ...}` | ✓ | Vendor-defined enum |
| `robot_id` (varying per call) | ✓ | Bounded by fleet size, low growth |
| `endpoint` = raw URL path | ✗ | Use EndpointMapper |
| `error_message` (exception text) | ✗ | Bucket by exception class instead |
| `status_text` (server-returned phrase) | ✗ | Use status code or class |
| `request_id` / `trace_id` | ✗ | Belongs in logs / traces, not metrics |

**Why this matters more than you think:** GCP Cloud Monitoring caps custom metrics at 200,000 active time series per descriptor, and a label key that gets a high-cardinality value once **stays on the descriptor permanently**. There's no `UPDATE METRIC DESCRIPTOR DROP LABEL` — your only fix is `delete + recreate`, which destroys historical data. Cardinality mistakes are not recoverable in place. Stop the leak at the call site.

## Histograms cost ~18× more than counters

A counter emits one value per series per scrape. A histogram with OTEL's **default bucket boundaries** emits **16 bucket counts + `_sum` + `_count` = 18 values** per series per scrape. Stackdriver ingests every one of those. The cost scales with the bucket count.

### The shape of the cost

```
ingested_volume ∝
    (number of attribute combinations)
  × (bucket count + 2)                          # +2 for sum and count
  × (scrapes per period)
```

**Worked example.** A connector records `api_request_duration_seconds` with three attributes — `endpoint` (8 values), `method` (5 values), `outcome` (7 values) — using the OTEL default histogram. Scrape interval is 30s.

- Attribute combinations: 8 × 5 × 7 = **280**
- Series per combination: 16 buckets + sum + count = **18**
- Scrapes per 30 days: 86,400
- Data points: 280 × 18 × 86,400 ≈ **435M / month**

A counter with the same attribute set emits 280 × 1 × 86,400 ≈ 24M points per month — **~18× less**. This is why a single histogram can dominate the ingestion cost of an entire connector even when "everything else looks fine".

### Sizing histograms

Three knobs, in order of leverage:

1. **Pick explicit bucket boundaries matched to your latency distribution.** OTEL's default 15-boundary set is generic — covers microseconds to minutes. HTTP calls to a vendor API typically live in 50ms–5s; 5–7 boundaries are enough to power useful p50/p95/p99 alerts:

    ```python
    duration = meter.create_histogram(
        "api.duration",
        unit="s",
        description="...",
        explicit_bucket_boundaries=(0.05, 0.1, 0.25, 0.5, 1.0, 2.5, 5.0),
    )
    ```

    Cuts the bucket-multiplier from 18 to **9**. ~2× cost reduction with one argument.

2. **Drop unnecessary attribute dimensions from the histogram.** Latency-by-`outcome` is rarely a useful question — when failures happen you look at the errors counter, not the p99-of-timeouts. The canonical framework histogram intentionally omits `error_kind` for this reason. Dropping a 7-value attribute that contributes nothing to alerting cuts cost ~7×.

3. **Question whether you need a histogram at all.** "Is this thing slow?" is sometimes better answered by a simple `request_count`-vs-`errors_count` ratio than by p99 latency. Don't reach for a histogram by reflex.

Combined, the connector in the worked example above can typically reach **~10–20MB/month** for the same metric — comfortably alongside the rest of its counters.

### Rules of thumb

| Pattern | Cost | Notes |
|---|---|---|
| Counter, 1 attribute (e.g., `robot_id`) | Cheapest | Use freely |
| Counter, 3-4 attributes | Cheap | Bounded enums only |
| Histogram, default buckets, 1-2 attributes | Moderate | Acceptable when you actually need latency percentiles |
| Histogram, default buckets, 3+ attributes | **Expensive** | This is where the surprise bills live |
| Histogram, custom 5-7 buckets, 1-2 attributes | Cheap | Recommended for HTTP latency |
| Histogram with high-cardinality label | **Catastrophic** | The 200K-series cap kicks in quickly |

If you find yourself writing a 3-attribute histogram with default buckets, pause. Either narrow the bucket boundaries, or split one dimension off into a separate counter.

## When to declare a new domain metric

Before adding a `meter.create_counter("foo.bar", ...)`:

1. **Is the framework already emitting it?** Check the table at the top of this file. Don't redeclare `up` or `execution_loop_ticks`.
2. **Does a canonical helper cover it?** If it's an HTTP call, use `record_upstream_http_request` / `record_upstream_http_error` — don't roll your own `acme_api_requests` counter.
3. **Does the metric answer an alerting question?** "Will an oncall engineer page someone if this metric does X?" If no, prefer logs.
4. **Are all attribute values bounded?** See the table above.
5. **Will two future readers of the code know what the metric means?** If the description is "I don't know, just gather everything", you don't need this metric yet.

When all five answers are clear, declare it on the connector's meter via `get_connector_meter`. Keep the count small — most connectors land below 10 domain instruments.

## Lifecycle and placement

**Module-level declaration.** Put instrument creation at module load time, not inside `__init__` or per-call. Same pattern the SDK uses:

```python
# your_connector/src/metrics.py
meter = get_connector_meter("acme")

mission_failures = meter.create_counter("mission.failures", ...)
api_circuit_breaker_opens = meter.create_counter("api.circuit_breaker_opens", ...)
```

Then import the instruments by name where you need them:

```python
# your_connector/src/connector.py
from .metrics import mission_failures

async def _handle_mission(self, m):
    try:
        await self._executor.run(m)
    except MissionError as e:
        mission_failures.add(1, {"reason": e.category})
        raise
```

**Observable gauges** (callback-evaluated at scrape time) need to close over connector state. Register them once during `_connect()`:

```python
# your_connector/src/metrics.py
from opentelemetry.metrics import Observation

def register_battery_gauge(robot) -> None:
    def _cb(_options):
        return [
            Observation(robot.battery_level(rid), {"robot_id": rid})
            for rid in robot.known_robot_ids()
        ]
    meter.create_observable_gauge(
        "battery.level",
        callbacks=[_cb],
        unit="1",
        description="Battery level per robot (0.0–1.0)",
    )
```

Callback should be cheap and side-effect-free — it runs on every Prometheus scrape (typically every 30s).

## CaC integration

Domain metrics don't appear in `cac/data_sources.yaml`. They're for operational health, not InOrbit dashboards. If a quantity belongs on the InOrbit robot screen (e.g., battery level), publish it via `publish_key_values` AND optionally also as a metric. The two surfaces serve different audiences.

## Common mistakes

| Mistake | Fix |
|---|---|
| `meter.create_counter("acme.api.errors")` while using `get_connector_meter("acme")` | Drop the `acme.` from the instrument name — wrapper adds it |
| `record_upstream_http_request/error(..., endpoint=path)` with raw `path` | Normalize via `EndpointMapper` or `PathTemplater` first |
| Adding new attributes to canonical HTTP metrics | Forbidden — declare a separate Layer 1 counter |
| Calling `record_upstream_http_request` on a non-2xx response | That's an error — use `record_upstream_http_error` with `error_kind="http_4xx"` (or `5xx`) |
| Using `inorbit_edge.metrics.get_meter(...)` directly in a vendor package | Use `get_connector_meter(connector_type)` |
| Manual `MeterProvider` / `PrometheusMetricReader` setup in a connector | Use `MetricsConfig` in YAML; framework wires the provider |
| Domain metric name without vendor prefix (`api.errors`) | Wrapper handles the prefix — verify you're calling `get_connector_meter(...)` |
| `attributes={"error": str(e)}` | Bucket by exception class: `{"error_kind": type(e).__name__}` |
| Battery emitted as 0–100 instead of 0–1 | `battery_value / 100.0` — same as `publish_key_values` |

## Querying

The reference collector ships metrics to GCP via the `googlemanagedprometheus` exporter, which writes them as the Prometheus descriptor type (`prometheus.googleapis.com/...`). **PromQL is the query surface** — in the Cloud Monitoring console and in Grafana (Managed Service for Prometheus datasource). Descriptor names carry a type suffix (`.../inorbit_connector_up/gauge`); in PromQL you query by the bare wire name.

The metric names exported by this framework:

- Framework: `inorbit_connector_up`, `inorbit_connector_session_connected`, `inorbit_connector_execution_loop_*`
- SDK: `calls_publish_*_total`
- Canonical: `inorbit_connector_upstream_http_requests_total`, `inorbit_connector_upstream_http_errors_total`, `inorbit_connector_upstream_http_duration_*`
- Vendor domain: `inorbit_connector_<vendor>_*`

Cross-connector aggregation works on every metric **except** Vendor domain (where per-vendor names are intentional — `mir.api.errors` and `otto.api.errors` aren't the same thing). Slicing labels: `connector_type` and `customer` (promoted by the collector), `instance` (= connector_id, from the `prometheus_target` monitored resource).

Example queries:

```promql
# Are any connectors down?
min by (connector_type, instance) (inorbit_connector_up) == 0

# Upstream API errors per vendor, broken down by failure mode
sum by (vendor, error_kind) (rate(inorbit_connector_upstream_http_errors_total[5m]))

# Error rate (errors / total) per vendor
sum by (vendor) (rate(inorbit_connector_upstream_http_errors_total[5m]))
/
(sum by (vendor) (rate(inorbit_connector_upstream_http_requests_total[5m]))
 + sum by (vendor) (rate(inorbit_connector_upstream_http_errors_total[5m])))

# Specific vendor: missions failing
sum by (reason) (rate(inorbit_connector_acme_mission_failures_total[5m]))
```

## Further reading

- `MetricsConfig` reference: [framework-api.md](framework-api.md)
- GCP Cloud Monitoring quotas: https://docs.cloud.google.com/monitoring/quotas
