# ADR-304: IVK Metadata Persistence & Cache-Policy

| Status | Date | Document Type |
| :--- | :--- | :--- |
| **Active** | 2026-01-16 | Architecture Design Record |

## Context
The **Internal Versioned Key (IVK)** is the regional anchor for tenant isolation. Because the Crypto Core must handle thousands of tenant keys (L2) with sub-millisecond latency, we cannot rely solely on the MasterKey unsealing logic or remote database fetches for every operation.

We need a standardized way to persist IVK metadata (state, versions, and cryptobinding) in the **Key Storage Interface (KSI)** and a corresponding caching strategy in **mlock** memory. Without a strict policy, we risk "Key Drift" (where a node uses a stale IVK version) or "Memory Bloat" (where too many old IVK versions consume protected RAM).

## Decision
We will implement a **State-Aware Persistence Schema** for IVK metadata and a **Tiered Cache Policy** that prioritizes security and performance.



### IVK Metadata Schema
Every IVK stored in the KSI must be wrapped by the MasterKey and accompanied by a plaintext metadata header containing:
* **Version ID:** Unique monotonic integer (e.g., v1, v2).
* **Key State:** `ACTIVE` (Current write), `GRACE` (Read-only hot), `DEPRECATED` (Read-only cold), or `REVOKED` (Inaccessible).
* **Creation Timestamp:** UTC time of generation.
* **Hard Expiry:** The absolute date after which the IVK must be purged.
* **Integrity Hash:** An HMAC of the encrypted blob to prevent tampering in the KSI.

### Caching Tiers
The Crypto Core will manage IVKs across three distinct memory tiers:

* **Tier 1: Volatile Hot Cache (mlock RAM):** * Holds the `ACTIVE` and `GRACE` IVK versions in their plaintext ("Green") state.
    * Uses `mlock` to prevent swap-to-disk.
    * Provides <1ms latency for L2 unwrap operations.
* **Tier 2: Encrypted Local Cache (Regional SSD):**
    * Holds `DEPRECATED` IVK versions in their wrapped ("Red") state.
    * Allows for rapid "Fault-In" to memory without querying the remote KSI database.
* **Tier 3: Authoritative Store (KSI Database):**
    * The durable source of truth for all versions.
    * Accessed only during bootstrap or when a very old L2 key requires a "Deep Fetch."

### Cache Eviction & Refresh
* **Proactive Loading:** When a new IVK version is generated (ADR-303), it is immediately promoted to Tier 1.
* **Temporal Eviction:** Once the "Grace Period" expires, the IVK is evicted from Tier 1 (RAM) to Tier 2 (Disk).
* **Security Purge:** If the MasterKey is wiped from memory (e.g., system restart or tamper detection), the entire Tier 1 cache is instantly invalidated and zeroized.



## Consequences

### Positive (Pros)
* **Consistent Performance:** 99% of L2 unwrap operations hit the mlock cache, ensuring high-throughput KMIP processing.
* **Auditability:** The metadata schema provides a clear history of regional key lifecycles for compliance reporting.
* **Fault Tolerance:** Local disk caching (Tier 2) ensures that the cluster can remain operational even if the central KSI database is temporarily partitioned.

### Negative (Cons) & Mitigations
* **Metadata Overhead:** Storing detailed headers increases the storage footprint in the KSI.
    * **Mitigation:** Headers are lightweight and manageable even at the scale of millions of records.
* **Complexity:** Managing the transition between Active, Grace, and Deprecated states requires careful coordination.
    * **Mitigation:** The logic is encapsulated within the **IVK Manager** component of the Crypto Core, abstracting it from the KMIP execution logic.

## References
* [ADR-201: MasterKey Immutability & Volatile Memory](masterkey-definition-and-immutability-standard.md)
* [ADR-301: The IVK (Internal Versioned Key) Architecture](ivk-architecture.md)
* [ADR-303: IVK Rotation Triggers & Grace Period Management](ivk-rotation-triggers-and-grace-period-management.md)