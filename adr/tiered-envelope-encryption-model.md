# ADR-102: Tiered Envelope Encryption (L1–L4)

| Status | Date | Document Type |
| :--- | :--- | :--- |
| **Active** | 2026-01-14 | Architecture Design Record |

## Context
OpenKCM must solve the **"Trust Paradox"**: providing customers with absolute sovereignty over their data (via external keys) while ensuring the platform can perform high-speed cryptographic operations (billions of records) without latency bottlenecks.

A flat key model fails both requirements: it either exposes the master key to the provider (loss of sovereignty) or forces every database read to call an external KMS (latency unacceptable).

We require a **Multi-Layer Keychain Architecture** that supports:
* **Customer Sovereignty:** Top-level trust anchored in the customer's cloud (AWS/Azure/GCP).
* **Cryptographic Isolation:** Strict separation between Tenant A and Tenant B.
* **Performance:** Localized, low-latency encryption for data-at-rest.
* **Compliance:** Support for BYOK (Bring Your Own Key) and HYOK (Hold Your Own Key).

## Decision
We will implement a **Four-Level Hierarchical Keychain (L1–L4)** using **Envelope Encryption**. This structure ensures that "The Key protecting the Data" (L4) is never the same as "The Key protecting the Tenant" (L2), which is never the same as "The Key owned by the Customer" (L1).

### The Hierarchy Definition

| Layer | Key Name | Scope | Ownership | Persistence | Role |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **L1** | **Root Anchor** | Global / Tenant | **Customer** (External KMS) | **External Only** | The "Kill Switch." Revoking this mathematically locks the entire chain. |
| **L2** | **Tenant Key** | Tenant | **OpenKCM** (Platform) | **Encrypted (Red)** | The mathematical boundary of a tenant. Isolates Tenant A from Tenant B. |
| **L3** | **Service Key** | Service / Domain | **OpenKCM** (Platform) | **Encrypted (Red)** | Isolates "Billing" data from "Health" data within the same tenant. |
| **L4** | **Data Key (DEK)** | Record / Object | **OpenKCM** (Edge) | **Ephemeral / Wrapped** | The high-velocity key used to encrypt the actual database rows or files. |



### Core Concepts: Green vs. Red Keys
To explicitly define security boundaries, we classify key states into two categories:

* **🟥 Red Keys (Encrypted/Wrapped):**
    * **Definition:** Key material that is encrypted by its parent.
    * **Storage:** Safe to store in databases (PostgreSQL), logs, or backups.
    * **Example:** An L3 Key blob stored in the `keys` table, wrapped by the L2 Key.
* **🟩 Green Keys (Plaintext):**
    * **Definition:** Raw, usable cryptographic key material.
    * **Storage:** **STRICTLY FORBIDDEN** on disk.
    * **Memory:** Only exists in `mlock` (locked RAM) within the **Crypto Core** or **Crypto Edge** process memory.
    * **Lifecycle:** Generated on-the-fly or unwrapped just-in-time, then zeroed out immediately after use (or cached briefly in secure memory).

### The IVK (Internal Versioned Key) Logic
To manage the lifecycle of internal keys without forcing constant L1 interactions, we introduce the **IVK**.
* The **IVK** is a regionally rotated intermediate key managed by the OpenKCM Crypto Core.
* **Dual-Wrapping for Resilience:** The L2 Tenant Key is often persisted in two forms:
    1.  **Sovereign Binding:** Wrapped by the Customer's L1 (for recovery and provenance).
    2.  **Active Binding:** Wrapped by the Regional IVK (for high-speed rotation and internal management).

## MasterKey Lifecycle Strategy
The **OpenKCM Crypto Core** requires a "Key of Keys" to protect the IVKs and internal state. We support two bootstrapping modes:

1.  **Seal Auto-Unseal (Cloud Native):**
    * The internal MasterKey is encrypted by a cloud provider key (e.g., AWS KMS Key belonging to the *Provider*).
    * On startup, the node authenticates via IAM, calls `kms:Decrypt`, and loads the MasterKey into memory.
    * *Use Case:* High-scale, automated distinct deployments.

2.  **Shamir Secret Sharing (SSS) (Sovereign/On-Prem):**
    * The MasterKey is split into $N$ shards.
    * $M$ shards are required to reconstruct it ($M < N$).
    * Operators must manually input shards via the API to "Unseal" the node at boot.
    * *Use Case:* Air-gapped environments or ultra-high security zones where no single cloud provider is trusted.

## Key Workflow (The "Chain of Custody")
1.  **L4 Creation:** App requests encryption. **Crypto Edge** generates a random L4 DEK (Green).
2.  **L4 Use:** Edge encrypts data with L4 DEK.
3.  **L4 Wrapping:** Edge requests **Crypto Core** to wrap L4.
4.  **Recursive Unseal:**
    * Core checks memory for L3 (Green). If missing:
    * Checks memory for L2 (Green). If missing:
    * Calls **External L1** (AWS KMS) to unwrap L2 (Red $\to$ Green).
    * Unwraps L3 using L2 (Red $\to$ Green).
5.  **Completion:** Core wraps L4 using L3. Returns `L4_Wrapped_Blob` (Red) to the Edge.

## Consequences

### Positive (Pros)
* **Absolute Sovereignty:** If the customer disables their L1 Key in AWS, OpenKCM *cannot* decrypt the L2 key. The entire pyramid collapses, guaranteeing data lockout.
* **Granular Isolation:** Compromising a single L3 Service Key only exposes data for that specific service, not the whole tenant.
* **Performance:** High-volume encryption (L4) happens locally. Slow, expensive calls to External KMS (L1) happen rarely (only to unseal the L2).
* **Flexibility:** Supports BYOK (Customer L1) and standard SaaS (Provider L1) using the same logic.

### Negative (Cons) & Mitigations
* **Complexity:** Managing a 4-layer graph with rotation at every level is engineering-heavy.
    * *Mitigation:* Automated "Lifecycle Managers" in the Crypto Core handle rotation policies transparently.
* **Latency on Cold Start:** The first request requires a full recursive unseal (L1 call), which may take 200-500ms.
    * *Mitigation:* Aggressive caching of L2/L3 Green Keys in the Crypto Core memory (LRU Cache).
* **Key Loss Risk:** If the L2 key blob is corrupted and the L1 key is revoked, data is permanently lost ("Cryptographic Erasure").
    * *Acceptance:* This is a feature, not a bug. It is the definition of sovereignty.

## References
* [ACD-301: The Layered Key Hierarchy](../acd/layered-key-hierarchy-(L1→L4).md)
* [ACD-302: Root of Trust & MasterKey Bootstrap](../acd/root-of-trust-and-masterkey-bootstrap.md)