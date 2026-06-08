# Capacity Plan — Vision Moderation Service

---

## 1. Latency Budget Breakdown

Target: **p95 end-to-end latency ≤ 250 ms** (external SLA boundary)

| Stage | Budget (ms) | Notes |
|---|---|---|
| Network in | 5 | Client → load balancer → pod ingress; assumes co-located clients or same-region CDN |
| Auth + routing | 3 | JWT validation (in-memory JWKS cache) + L7 routing rule lookup |
| Payload parse | 4 | JSON envelope decode + base64 image decode; ~200 KB median image |
| Feature lookup (Redis) | 8 | Single GET for account-level policy flags; Redis Cluster, same AZ, p99 < 2 ms; budget covers retries |
| Pre-processing | 20 | Resize → normalize → tensor conversion on CPU (SIMD); batched across concurrent requests |
| Model inference | 170 | **GPU path (see §2)**; ResNet-50-scale vision model, batch=8, A10G GPU; includes CUDA kernel launch overhead |
| Post-processing | 10 | Softmax → threshold check → label mapping → audit-log write (async fire-and-forget) |
| Serialization | 5 | JSON response marshal; response body ~0.5 KB |
| Network out | 5 | Pod egress → load balancer → client |
| **Headroom** | **20** | Buffer for GC pauses, cold Redis misses, occasional batch size=1 inference penalty |
| **Total** | **250** | ✅ Headroom is positive (+20 ms) |

**Work shown:**  
5 + 3 + 4 + 8 + 20 + 170 + 10 + 5 + 5 = **230 ms** assigned  
250 − 230 = **20 ms headroom** (8% of budget)

---

## 2. CPU vs GPU Decision

**Decision: GPU (NVIDIA A10G, 24 GB VRAM)**

A vision moderation model (ResNet-50 or equivalent, ~25 M parameters) requires ~0.8 ms per image on an A10G at batch=8, versus ~28 ms per image on a modern CPU core (e.g., c6i.4xlarge). At the target 300 RPS sustained load, CPU inference would consume ≈ 300 × 28 ms = 8.4 core-seconds per second, requiring ~9 dedicated vCPUs per replica just for inference — making the latency budget impossible to meet without a prohibitively large fleet.

With the A10G, a single GPU can sustain ~1,000 inferences/second at batch=8 (170 ms budget comfortably met), giving **~125 requests/replica/second** after accounting for pre/post-processing overhead. One `g5.xlarge` instance (4 vCPU, 16 GB RAM, 1× A10G) covers the full inference path within budget.

| | CPU (c6i.4xlarge) | GPU (g5.xlarge) |
|---|---|---|
| vCPU / GPU | 16 vCPU / — | 4 vCPU / 1× A10G |
| Inference latency (batch=8) | ~224 ms | ~12 ms |
| Per-replica throughput | ~35 RPS | ~125 RPS |
| On-demand price (us-east-1) | ~$0.68/hr | ~$1.006/hr |
| Monthly cost per replica | ~$490 | ~$725 |

*Prices sourced from AWS public pricing page (approximate, ±30%); verify at [https://aws.amazon.com/ec2/pricing/on-demand/](https://aws.amazon.com/ec2/pricing/on-demand/).*

GPU costs ~48% more per replica but delivers ~3.6× the throughput, yielding a **2.4× better cost-per-request**. GPU is the correct choice.

---

## 3. Replica Sizing

**Per-replica throughput:** 125 RPS (GPU, g5.xlarge)  
**Headroom factor:** 1.30 (30% buffer above peak demand)

| Scenario | Target RPS | Raw replicas needed | +30% headroom → replicas | Monthly cost (on-demand) | Spike strategy |
|---|---|---|---|---|---|
| Sustained | 300 | 300 ÷ 125 = 2.4 → **3** | ceil(3 × 1.3) = **4** | 4 × $725 = **$2,900** | — |
| Spike | 500 | 500 ÷ 125 = 4.0 → **4** | ceil(4 × 1.3) = **6** | 6 × $725 = **$4,350** | Autoscaling (see below) |

**Spike strategy: Autoscaling (Kubernetes HPA + Karpenter)**  
- Baseline fleet: **4 replicas** (covers sustained 300 RPS with headroom)  
- Scale-out trigger: CPU utilization > 60% **or** custom metric `inference_queue_depth > 20` for 30 seconds
- Target ceiling: **6 replicas** for 500 RPS spike
- Scale-out time: Karpenter node provisioning ~60–90 s; pre-warmed node pool reduces this to ~20 s
- Warm overprovisioning **not** chosen: g5.xlarge instances are expensive to idle; autoscaling is cost-efficient for spikes that last > 2 minutes. Sub-2-minute spikes are absorbed by the 30% headroom already built into the 4-replica baseline.
- Queue-shedding fallback: if replica count hits ceiling and queue depth exceeds 50, return HTTP 429 with `Retry-After: 5s` — preserves SLO for existing in-flight requests.

---

## 4. Batching Decision

**Decision: Enable dynamic batching**  
**Max batch size: 8**  
**Max wait window: 15 ms**

Dynamic batching is enabled because the A10G GPU is severely underutilized at batch=1 (kernel launch overhead dominates; effective throughput drops to ~40 RPS per GPU). Batching up to 8 images amortizes CUDA kernel dispatch and tensor memory transfer costs, pushing throughput to ~125 RPS per replica while keeping inference latency well within budget. At batch=8, measured inference time on an A10G for a ResNet-50-scale model is approximately 12 ms, compared to ~8 ms at batch=1 — a modest 4 ms penalty that is easily absorbed by the 170 ms inference budget slot.

The **15 ms wait window** is chosen to be no more than 8.7% of the total 170 ms inference budget, ensuring that even a request that arrives just after a batch closes will still be dispatched within the latency envelope. At 300 RPS sustained, the average inter-arrival time is ~3.3 ms, meaning a batch of 8 typically fills within ~26 ms — the wait window will rarely be the binding constraint. At 500 RPS spike, a full batch of 8 fills in ~16 ms, so the wait window is almost never reached. If a batch does not fill within 15 ms, it is dispatched immediately regardless of size, preventing starvation of low-traffic periods or isolated requests.
