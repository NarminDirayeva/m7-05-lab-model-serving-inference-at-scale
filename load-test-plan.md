# Load Test Plan — Vision Moderation Service

---

## Tool Choice

**k6** (Grafana k6 v0.51+)

k6 is chosen over alternatives for two reasons. First, it supports both HTTP and gRPC natively and allows inline JavaScript scenario scripting, making it straightforward to mix synchronous single-image requests with async batch payloads in the same test run. Second, k6's built-in Prometheus remote-write output integrates directly with the existing observability stack — pass/fail thresholds can be evaluated in real time without post-processing. *(Locust was considered but discarded: Python's GIL limits sustained RPS without a distributed coordinator; vegeta lacks dynamic scenario logic; wrk2 cannot generate multi-part payloads.)*

---

## Test Phases

All durations and RPS targets are parameterized via environment variables (`K6_TARGET_RPS`, `K6_SOAK_DURATION`) to allow reuse in CI and pre-production environments.

| Phase | Duration | RPS ramp | Purpose |
|---|---|---|---|
| **Warmup** | 5 min | 0 → 50 RPS (linear) | Allow JIT compilation, Redis connection pool warm-up, and GPU CUDA context initialization; discard metrics from this phase |
| **Ramp-up** | 10 min | 50 → 300 RPS (linear) | Validate that autoscaler responds correctly; observe per-replica metric stabilization |
| **Sustained peak** | 30 min | 300 RPS (flat) | Primary SLO measurement window; must meet all pass/fail criteria below |
| **Spike** | 5 min | 300 → 500 RPS (step, instant) | Validate HPA scale-out behavior; confirm no SLO breach during scale-out lag |
| **Spike drain** | 5 min | 500 → 300 RPS (linear) | Confirm graceful scale-in; verify no request drops during pod termination |
| **Soak** | 4 h | 150 RPS (flat, 50% of peak) | Detect memory leaks, Redis connection exhaustion, log disk fill, and slow error accumulation over time |

**Total test runtime (excluding soak):** ~55 minutes  
**Soak run:** Scheduled separately overnight; triggered by the same k6 script with `K6_SOAK_DURATION=4h`.

---

## Traffic Shape

**Synchronous vs batch ratio:** 80% synchronous single-image requests / 20% batch requests (4–16 images per batch payload). This ratio reflects the observed production distribution from shadow-traffic analysis.

**Payload size distribution:**

| Payload type | Size range | % of requests |
|---|---|---|
| Small image (thumbnail) | 20–80 KB | 30% |
| Medium image (standard upload) | 80–300 KB | 55% |
| Large image (high-res upload) | 300 KB–1 MB | 15% |

Images are drawn from a pre-generated corpus of 10,000 synthetic test images stored in S3 and loaded into k6 SharedArray at startup to avoid per-VU disk I/O. Content-type is `multipart/form-data` for single requests and `application/json` with base64-encoded image array for batch.

**Concurrency model:** k6 arrival-rate executors (`constant-arrival-rate` for sustained phases, `ramping-arrival-rate` for ramp/spike). This decouples VU count from RPS and accurately models open-loop traffic, which is appropriate for an API that receives requests from many independent clients. Max VUs capped at 600 to prevent the test harness itself from becoming the bottleneck.

**Authentication:** Each VU uses a pre-issued JWT from a test-only issuer; tokens are rotated every 50 requests to exercise the token validation code path realistically.

---

## Pass/Fail Criteria

The k6 `thresholds` block enforces these criteria; the test exits with a non-zero status code if any threshold is breached.

| Metric | Threshold | Rationale |
|---|---|---|
| `http_req_duration{p(95)}` | ≤ 250 ms | Direct SLO latency_p95 objective |
| `http_req_duration{p(99)}` | ≤ 500 ms | Direct SLO latency_p99 objective |
| `http_req_failed` (rate) | ≤ 0.1% | 5xx + timeout rate; must stay below SLO error budget |
| `http_req_duration{p(50)}` | ≤ 120 ms | Median sanity check; regression canary |
| Autoscaler replica count at 500 RPS | ≥ 5 replicas within 90 s | Validates HPA scale-out meets spike SLO |
| Soak: `http_req_failed` rate drift | < 0.05% increase over 4 h baseline | Memory leak / connection exhaustion signal |
| Soak: `http_req_duration{p(95)}` drift | < 15 ms increase over 4 h baseline | Slow degradation signal |

---

## Bottleneck Checklist

The following metrics are actively monitored on each replica via Grafana dashboards and alerts during the test run:

**Compute (per pod)**
- CPU utilization (%) — should stay below 70% on non-inference cores; spike to 90% on pre/post-processing cores is acceptable
- GPU utilization (%) — target 75–90%; below 50% indicates under-batching; above 95% is a saturation warning
- GPU memory used (GB) — must stay below 20 GB / 24 GB to avoid OOM evictions
- JVM / Go GC pause duration (p99) — should be < 10 ms; longer pauses contribute to p99 latency

**Inference pipeline**
- Dynamic batch size distribution (histogram) — confirm batches of 8 are formed at 300 RPS; watch for batch=1 spikes indicating wait-window timeout triggers
- Batch wait time (ms) — should be < 15 ms at sustained load; consistently hitting the ceiling means the wait window is binding

**Downstream dependencies**
- Redis GET latency (p99, per AZ) — should be < 2 ms; > 5 ms triggers the feature-lookup budget overshoot
- Redis connection pool saturation (active / max) — must not exceed 80% of `max_active` connections
- Redis error rate — any errors here cause synchronous retries and add 8–16 ms to the request path

**Network and load balancer**
- Ingress bytes/s — validate the test harness is sending the expected payload size distribution
- Load balancer active connection count — watches for connection queue buildup at the LB layer
- TCP retransmit rate — elevated retransmits indicate network congestion between test harness and service

**Kubernetes / autoscaling**
- HPA desired vs current replica count — lag > 90 s during spike phase is a failure
- Pod restart count — any CrashLoopBackOff during the test is an automatic test failure
- Node GPU allocatable vs requested — confirms Karpenter is provisioning g5.xlarge nodes, not CPU-only nodes
