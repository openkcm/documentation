# ADR-303: IVK Rotation Triggers & Grace Period Management

| Status | Date | Document Type |
| :--- | :--- | :--- |
| **Active** | 2026-01-16 | Architecture Design Record |

## Context
The **Internal Versioned Key (IVK)** acts as the regional wrapper for all L2 Tenant Keys. To maintain Perfect Forward Secrecy and minimize the "Blast Radius" of a potential internal compromise, these keys must rotate frequently.

However, rotating a regional root key that protects thousands of tenant keys cannot be an atomic, blocking operation. If the system immediately purged the old IVK, all L2 keys not yet re-wrapped would become instantly inaccessible, causing a massive service outage.

We require a state-machine based approach to rotation that allows for a "Grace Period" where multiple IVK versions coexist in memory to facilitate non-disruptive, lazy re-wrapping.

## Decision
We will implement a **Three-Phase IVK Lifecycle** governed by specific temporal and event-based triggers.



## Rotation Triggers
An IVK rotation (the generation of a new version) is triggered by three conditions:

* **Scheduled (Time-based):** Default rotation occurs every 30 days. This is the primary hygiene mechanism.
* **Emergency (Event-based):** Triggered manually via the CMK Portal or Orbital if an administrator suspects a regional node compromise.
* **Volume (Usage-based):** Triggered after an IVK has performed $2^{32}$ wrap operations to prevent cryptographic wear-out and side-channel analysis.

## The Version State Machine
At any given time, a Regional Crypto Core cluster manages three states of IVKs:

### Active Version (The "Write" Key)
* **Definition:** The single latest version ($IVK_{v(n)}$).
* **Behavior:** Used for all *new* L2 key generations and all *re-wrap* operations.
* **Memory:** Must be present in `mlock` RAM.

### Grace Version (The "Hot" Read Key)
* **Definition:** The immediately preceding version ($IVK_{v(n-1)}$).
* **Behavior:** Used only for *decryption* of existing L2 keys. When an L2 key wrapped by this version is accessed, it is decrypted and immediately re-wrapped with the **Active Version** (Lazy Re-wrap).
* **Memory:** Stays in `mlock` RAM for 24 hours (default) after a rotation to ensure sub-millisecond access for active tenants.

### Deprecated Version (The "Cold" Read Key)
* **Definition:** Any version older than $v(n-1)$.
* **Behavior:** These keys are purged from `mlock` memory but remain in the **Key Storage Interface (KSI)**.
* **Access:** If a tenant key is requested that hasn't been touched in weeks, the Core must "Fault In" the Deprecated IVK from the KSI, unwrap it using the MasterKey, and then perform the L2 unwrap.



## Lazy Re-Wrap Logic
We will not perform a "Batch Re-wrap" of all L2 keys upon IVK rotation. Instead:
1.  **Access:** Application requests Tenant A's L2.
2.  **Detection:** Core detects that L2 is wrapped by $IVK_{v(n-1)}$.
3.  **Promotion:** Core unwraps L2, then immediately wraps it with $IVK_{v(n)}$.
4.  **Update:** The new encrypted blob is persisted to the KSI.
5.  **Completion:** Future requests for Tenant A now use the Active IVK directly.

## Consequences

### Positive (Pros)
* **Zero Downtime:** Rotation is completely transparent to the application.
* **Controlled Load:** Cryptographic overhead is spread out over time based on actual usage rather than a massive background job.
* **Security Resilience:** An emergency rotation immediately switches the "Write" key, ensuring any new keys created post-incident are protected by fresh material.

### Negative (Cons) & Mitigations
* **Memory Overhead:** Maintaining multiple IVKs in `mlock` RAM increases the memory footprint.
    * **Mitigation:** The "Grace Period" is configurable; high-memory environments can shorten the window to 1 hour.
* **Storage Fragmentation:** The database will contain L2 blobs wrapped by various IVK versions.
    * **Mitigation:** The **Orbital Scrubber** runs a low-priority background task to identify "Stale" L2s (wrapped by versions older than 90 days) and force a re-wrap during off-peak hours.

## References
* [ADR-301: The IVK (Internal Versioned Key) Architecture](ivk-architecture.md)
* [ADR-302: IVK Mapping Logic to L2 Tenant Keys](ivk-mapping-logic-to-l2-tenant-keys.md)
* [ACD-304: Key Rotation Strategies & Lifecycle Management](../acd/key-rotation-strategies-and-lifecycle-management.md)