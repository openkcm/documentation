# ADR-301: The IVK (Internal Versioned Key) Architecture

| Status | Date | Document Type |
| :--- | :--- |:---|
| **Active** | 2026-01-14 | Architecture Design Record |

## Context
OpenKCM Crypto requires a secure internal mechanism to protect **tenant-specific L2 keys**, which define the cryptographic boundary separating one customer's data from another.

These L2 keys **must never be stored in plaintext** and must be encrypted under a stable internal root. However, relying solely on a single MasterKey creates a "God Key" risk: if the MasterKey is compromised or needs rotation, billions of downstream keys must be re-encrypted immediately, causing massive downtime.

**The Requirement:** We need an intermediate layer that:
1.  **Isolates the MasterKey:** Prevents the MasterKey from directly encrypting millions of tenant keys.
2.  **Enables Rotation:** Allows the platform to rotate its internal protection (IVK) without forcing a re-encryption of customer data (L4).
3.  **Supports Availability:** Ensures L2 keys can be decrypted even if the Customer's L1 key is momentarily unreachable (for internal management operations).

To satisfy these requirements, OpenKCM introduces the **Internal Versioned Key (IVK)**.

> **Strategic Constraint:** The **MasterKey MUST NEVER rotate.** It acts as the immutable, static root of trust for the lifetime of the cluster. All rotation complexity is offloaded to the IVK layer.

## Decision
We introduce the **Internal Versioned Symmetric Key (IVK)** as the mandatory intermediate layer between the MasterKey and Tenant Keys.

### The IVK Definition
* **Algorithm:** Symmetric AES-256-GCM.
* **Scope:** Cluster-wide (Regional). One active IVK protects all tenants in that region.
* **Storage:** Stored as an **Encrypted Blob** in the database, wrapped by the MasterKey.
* **Role:** Exclusively used to **wrap and unwrap L2 Tenant Keys**.

### The Rotation Logic (Lazy Re-Encryption)
* **Versioning:** The system maintains a chain of IVKs (`IVK_v1`, `IVK_v2`, ...).
* **Active State:** Only the *latest* version (`IVK_vCurrent`) is used to encrypt *new* L2 keys.
* **Legacy State:** All previous versions (`IVK_vOld`) are retained in the database (encrypted by MasterKey) to allow decryption of older L2 keys.
* **Benefit:** When we rotate the IVK, we do **not** need to re-encrypt all existing L2 keys immediately. We simply start using the new IVK for new tenants, and slowly migrate old tenants in the background.

### Dual-Wrapping of L2 Keys
Every **L2 Tenant Key** is stored with **two distinct wrappings** to satisfy both Sovereignty and Availability:
1.  **Sovereign Binding:** Wrapped by the **Customer L1** (External KMS). *Used for Recovery & Provenance.*
2.  **Active Binding:** Wrapped by the **Regional IVK**. *Used for High-Speed Operational Access.*

## Architecture Overview

### Key Hierarchy
The IVK sits as the "Gearbox" between the static MasterKey engine and the dynamic Tenant wheels.

```mermaid
flowchart TD
    MK[<b>MasterKey</b><br/>Immutable Root<br/>(RAM Only)]
    IVK[<b>Internal Versioned Key 'IVK'</b><br/>Rotatable Intermediate<br/>(AES-256-GCM)]
    L2[<b>L2 Tenant Key</b><br/>Isolation Boundary]
    L3[<b>L3 Service Key</b><br/>Domain Isolation]
    L4[<b>L4 Data Key</b><br/>Ephemeral DEK]

    MK -->|Wraps| IVK
    IVK -->|Wraps| L2
    L2 -->|Wraps| L3
    L3 -->|Wraps| L4
    
    style MK fill:#b7f7b7,stroke:#2d7a2d,stroke-width:2px
    style IVK fill:#f9f,stroke:#333,stroke-width:2px
```

## Storage & Protection Model

| Key Layer | Stored As | Encryption Parent | Rotation Strategy | Failure Impact |
| :--- | :--- | :--- | :--- | :--- |
| **MasterKey** | SSS Shards / Sealed Blob | External Cloud/HSM | **NEVER** | Total Region Loss (Recovery Impossible) |
| **IVK (vN)** | Encrypted Blob | MasterKey | **Monthly / On-Demand** | Unable to decrypt L2 keys wrapped by vN |
| **L2 Tenant** | Encrypted Blob | **IVK (Active)** | Yearly / Breach | Loss of single Tenant |
| **L3 Service** | Encrypted Blob | L2 Tenant | Monthly | Loss of single Service |
| **L4 Data** | Wrapped Ciphertext | L3 Service | Ephemeral | Loss of single Record |

## IVK Lifecycle Workflows

### A. IVK Generation & Rotation
When the `Rotate_IVK` command is issued (e.g., every 30 days):
1.  **Generate:** Crypto Core generates a new random AES-256 key (`IVK_v2`).
2.  **Wrap:** `IVK_v2` is wrapped by the MasterKey.
3.  **Persist:** The wrapped blob is stored in the `system_keys` table.
4.  **Promote:** The Registry marks `IVK_v2` as `ACTIVE`. `IVK_v1` becomes `DEPRECATED` (decrypt-only).

### B. L2 Tenant Key Access
When an application needs to use Tenant A's keys:
1.  **Fetch:** Core retrieves the encrypted L2 blob for Tenant A.
2.  **Identify:** The blob header says: *"Wrapped by IVK_v1"*.
3.  **Unwrap IVK:** Core uses MasterKey to unwrap `IVK_v1`.
4.  **Unwrap L2:** Core uses `IVK_v1` to unwrap L2.
5.  **Cache:** The L2 plaintext is cached in secure memory (LRU) for subsequent operations.

## Consequences

### ✅ Positive (Pros)
* **Forward Secrecy:** If `IVK_v1` is compromised, we can rotate to `IVK_v2`. While `v1` data is at risk, all *future* data is secure.
* **Operational Velocity:** Rotating the IVK takes milliseconds (1 database write). Rotating a MasterKey without an IVK would take days (decrypting/re-encrypting millions of rows).
* **Availability:** We don't need to call AWS KMS (L1) for every single L2 access, avoiding throttling limits and network latency.

### ⚠️ Negative (Cons) & Mitigations
* **Complexity:** We must manage a "Keyring" of historical IVKs.
  * *Mitigation:* Automated background jobs prune IVKs that are no longer protecting any active L2 keys.
* **Double Encryption Overhead:** L2 keys are double-wrapped (once by L1, once by IVK).
  * *Mitigation:* The storage cost is negligible (kilobytes). The CPU cost is minimal compared to the network latency saved.

## Alternatives Considered
* **Direct MasterKey Wrapping:**
  * *Rejected:* Rotating the MasterKey would require downtime and massive re-encryption.
* **Direct L1 Dependency (No IVK):**
  * *Rejected:* If AWS KMS is down or slow, the entire SaaS platform halts. IVK allows us to cache L2 keys independently of L1 availability (once initially unsealed).

**Final Decision:** The **IVK** is the standard internal mechanism for protecting Tenant L2 Keys.