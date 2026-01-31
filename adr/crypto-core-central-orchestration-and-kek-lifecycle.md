# ADR-601: Crypto (Krypton) Central Orchestration and KEK Lifecycle

| Status | Date | Document Type |
| :--- | :--- | :--- |
| **Active** | 2026-01-17 | Architecture Design Record |

## Context
The **OpenKCM Crypto (Krypton)** is the "Execution Engine" of the platform. While the Governance Plane (CMK) defines *who* can access keys, the Core is responsible for the actual cryptographic heavy lifting:
* Recursively unsealing the key hierarchy (L1 -> IVK(L2) -> L2 -> L3).
* Managing the lifecycle of **Key Encryption Keys (KEKs)** (L2 and L3).
* Serving high-speed KMIP operations to the Crypto (Krypton) Gateway.

**The Problem:** Without a central orchestration logic, the Core becomes a simple "dumb keystore." We need an intelligent supervisor that ensures keys are rotated on schedule, cached effectively for performance, and purged instantly upon revocation.

## Decision
We will implement the **Crypto (Krypton) Orchestrator** as the central runtime responsible for managing the **KEK Lifecycle State Machine**.



### The Orchestrator's Responsibilities
1.  **Lazy Hydration (On-Demand Loading):**
    * The Core does *not* load all 10,000 tenant keys at startup.
    * Instead, it waits for a KMIP request. If the required L2/L3 key is not in the `mlock` cache, the Orchestrator initiates a "Hydration Workflow" to fetch the encrypted blob from the KSI, unwrap it (via IVK or L1), and hot-load it into memory.
2.  **Active Rotation Management:**
    * The Orchestrator runs a background ticker to check the `RotateAfter` timestamp of all active L3 keys.
    * When a key expires, it generates a new version (L3_v2), wraps it with the parent L2, and persists it to the KSI *without* blocking incoming traffic.
3.  **Revocation Enforcement:**
    * Upon receiving a `RevokeTenant` signal from Orbital, the Orchestrator instantly evicts all associated L2 and L3 keys from the `mlock` cache, effectively killing the tenant's ability to decrypt data.

### The KEK Lifecycle State Machine
Every L2 and L3 key managed by the Core transitions through strict states:
* **Pending:** Generated but not yet persisted.
* **Active:** Loaded in memory (`mlock`) and valid for encryption/decryption.
* **Expired:** Valid for decryption only (Grace Period).
* **Revoked:** Forbidden for all operations. Removed from memory.

## Consequences

### Positive (Pros)
* **Memory Efficiency:** "Lazy Hydration" ensures that we only spend RAM on active tenants. A tenant that hasn't accessed the system in months consumes zero memory.
* **Security:** Keys are only "Green" (plaintext) when necessary. Inactive keys stay "Red" (encrypted) on disk.
* **Resilience:** The Orchestrator handles transient failures (e.g., KSI temporary outage) with internal retries during the hydration phase.

### Negative (Cons)
* **Cold Start Latency:** The first request for a cold tenant pays the "Hydration Penalty" (fetching + unwrapping), adding ~50-200ms latency.
    * *Mitigation:* A "Pre-Warm" API allows the CMK Service to signal the Core to load keys for a high-priority tenant *before* traffic hits.

## References
* [ACD-202: OpenKCM Crypto (Krypton) – Regional Governance](../acd/crypto-core-regional-governance-and-unsealing.md)
* [ADR-304: IVK Metadata Persistence & Cache-Policy](ivk-metadata-persistence-and-cache-policy.md)