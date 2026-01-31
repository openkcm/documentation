# ADR-604: Sub-Millisecond Latency KPIs and Measurement

| Status | Date | Document Type |
| :--- | :--- | :--- |
| **Active** | 2026-01-17 | Architecture Design Record |

## Context
The **OpenKCM Crypto (Krypton) Gateway** is marketed as a "Zero-Latency" component. If encryption adds noticeable delay to the application's critical path, developers will bypass it, compromising security.
* **The Promise:** Encryption should be "effectively free" (< 1ms).
* **The Reality:** Without strict KPIs and granular observability, latency creeps in due to Garbage Collection (GC), network jitter, or lock contention.

**The Requirement:** We must define exactly what "Sub-Millisecond" means, how to measure it without the Observer Effect, and how to alert when we breach it.

## Decision
We define the **"Golden Signal"** for Cryptography as the **P99 Latency of L4 Operations** measured at the SDK/Sidecar boundary.

### The KPIs
| Metric | Target | Failure Threshold | Rationale |
| :--- | :--- | :--- | :--- |
| **P50 (Median)** | **< 0.2ms** | > 0.5ms | Most ops are just AES-NI instructions in local memory. |
| **P99 (Tail)** | **< 1.0ms** | > 5.0ms | Accounts for occasional GC pauses or L3 cache misses. |
| **P99.9 (Deep Tail)** | **< 10ms** | > 50ms | Accounts for the rare "Cold Start" where L3 must be fetched from Core. |
| **Availability** | **99.99%** | < 99.9% | Gateway must survive Core outages. |

### Measurement Strategy
We will implement **Client-Side Histogram Metrics** in the OpenKCM SDK and Sidecar.
* **Why Client-Side?** Measuring at the server (Gateway) misses the network hop from App-to-Sidecar. The Application SDK's view is the only one that matters to the user.
* **Buckets:** Exponential buckets optimized for low latency: `[0.1ms, 0.25ms, 0.5ms, 1ms, 2.5ms, 5ms, 10ms, Inf]`.

### Alerting Logic
* **"Slow Burn":** If P99 > 1ms for 5 minutes -> **Warning** (Check CPU throttling).
* **"Fast Burn":** If P99 > 5ms for 1 minute -> **Critical** (Possible L3 Cache Stampede).

## Consequences

### Positive (Pros)
* **Trust:** We can prove to customers that security is not slowing them down using their own Datadog/Prometheus dashboards.
* **Early Detection:** Tracking P99.9 allows us to catch "Micro-Bursts" or lock contention issues before they affect the median user.

### Negative (Cons)
* **Metric Volume:** High-cardinality histograms (per tenant, per operation) can be expensive to ingest.
    * *Mitigation:* We aggregate metrics at the Sidecar level before pushing to the central observability platform, dropping per-request tracing unless an error occurs (Head-Based Sampling).

## References
* [Google SRE Book: Golden Signals](https://sre.google/sre-book/monitoring-distributed-systems/)
* [ACD-203: Crypto (Krypton) Gateway Performance](../acd/crypto-gateway–high-performance-ephemeral-data-plane.md)