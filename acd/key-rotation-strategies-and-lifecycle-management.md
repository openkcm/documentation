# ACD-304: Key Rotation Strategies & Lifecycle Management

| Status | Date | Document Type |
| :--- | :--- | :--- |
| **Active** | 2026-01-14 | Architecture Concept Design |

## Overview
Key Rotation is the "heartbeat" of a healthy cryptographic system. In the OpenKCM ecosystem, rotation is not merely a compliance checkbox but a strategic capability that ensures **Perfect Forward Secrecy** and limits the "Blast Radius" of any potential compromise.

This document defines the lifecycle logic for the entire L1–L4 hierarchy. It establishes how OpenKCM handles the complex transition from "Old Keys" to "New Keys" without disrupting high-throughput SaaS operations, utilizing a philosophy of **"Lazy Re-encryption"** for data and **"Active Re-wrapping"** for keys.

## The Versioning Paradigm
In OpenKCM, a "Key" is not a static object; it is a **Versioned Chain**.
* **Active (Encryption) Version:** There is only *one* version used for *new* encryption operations (e.g., `v5`).
* **Retired (Decryption) Versions:** Previous versions (e.g., `v1`...`v4`) remain available strictly for *decryption* of existing data until they are formally archived or purged.

This allows the platform to rotate keys instantly without needing to re-encrypt terabytes of historical data immediately.

## Rotation Strategies by Layer

The rotation strategy differs for each layer of the hierarchy based on ownership and scope.

### L1 Rotation (Customer Driven)
The **L1 Root Key** lives in the customer's external KMS (AWS/Azure/GCP).
* **Trigger:** The customer rotates the key in their cloud console (or via policy).
* **OpenKCM Action:** OpenKCM detects the change (or receives a signal). It performs an **Active Re-wrap**: it fetches the L2 Tenant Key (currently wrapped by L1 `v1`), decrypts it, and immediately re-wraps it with L1 `v2`.
* **Impact:** Zero downtime. The internal L2 key material does not change, only its "envelope" changes.

### L2 & L3 Rotation (Internal Governance)
These keys (Tenant and Service keys) are managed entirely by OpenKCM.
* **Trigger:** Time-based (e.g., every 90 days) or Event-based (e.g., "Compromise Detected").
* **Strategy: Versioned Rolling.**
    1.  **Generate:** Create new random key material (`L2_v2`).
    2.  **Promote:** Update the **CMK Registry** to set `L2_v2` as the `Current` key for encryption.
    3.  **Retain:** Mark `L2_v1` as `Deprecated/Decrypt-Only`.
* **Crypto Core Behavior:** The Regional Core will immediately start using `L2_v2` for any new `Wrap` requests. It retains `L2_v1` in memory (or vault) to service `Unwrap` requests for older data.

### L4 Rotation (Ephemeral High-Velocity)
L4 keys are Data Encryption Keys (DEKs).
* **Trigger:** Per-Transaction or Per-Session.
* **Strategy: Disposable.**
    * L4 keys are rarely "rotated" in the traditional sense. They are generated once, used to encrypt a record/document, and then the *wrapped* L4 key is stored with the data.
    * To "rotate" an L4 key effectively means to **Re-encrypt the Data** (read record, decrypt, generate new L4, encrypt, save). This is an application-layer concern.

## Re-Encryption Strategies: Handling Historical Data

When a key rotates, what happens to the data encrypted by the old key? OpenKCM supports two distinct models.

### 1. Lazy Re-encryption (Default - High Performance)
* **Logic:** "New data uses new keys; old data stays on old keys."
* **Mechanism:** When L3 rotates from `v1` to `v2`, the system simply keeps `v1` available.
* **Pros:** Zero IOPS impact on the storage layer. Instant rotation.
* **Cons:** If `v1` was compromised, historical data is still vulnerable until re-encrypted.

### 2. Active Key Re-Wrapping (Security Hygiene)
* **Logic:** "Protect the keys that protect the data."
* **Mechanism:** When L2 rotates, we do not necessarily re-encrypt the data (L4). However, we **MUST** re-encrypt the **L3 Keys**.
    * The system takes all L3 Keys (wrapped by `L2_v1`), decrypts them, and re-wraps them with `L2_v2`.
    * Once finished, `L2_v1` can be destroyed.
* **Benefit:** This achieves **Cryptographic Erasure** of the old L2 version without needing to touch the petabytes of actual customer data (L4).

## The Lifecycle State Machine

| State | Capabilities | Transition Logic |
| :--- | :--- | :--- |
| **PRE_ACTIVE** | None | Key generated but not yet distributed to regions. |
| **ACTIVE** | Encrypt & Decrypt | The primary key for all new write operations. |
| **DEPRECATED** | Decrypt Only | Rotated out. Used only to read historical data. |
| **COMPROMISED** | Blocked | Emergency state. All operations rejected immediately. |
| **ARCHIVED** | Restore Only | Moved to cold storage (Glacier/Long-term). Removed from active cache. |
| **DESTROYED** | None | Cryptographically erased. Irreversible data loss for anything still using it. |

## Event-Driven "Emergency" Rotation
In the event of a suspected breach, the **Orbital** engine can trigger an **Emergency Rotation**:
1.  **Command:** Admin triggers "Rotate & Revoke" for Tenant X.
2.  **Broadcast:** Orbital notifies all Regional Cores.
3.  **Execution:**
    * New L2/L3 versions are generated immediately.
    * Old versions are marked `COMPROMISED` (or `DEPRECATED` depending on severity).
    * If `COMPROMISED`, the keys are purged from memory, rendering old data momentarily inaccessible until a controlled recovery/re-keying process is authorized.

## Summary
This document defines how OpenKCM manages the dimension of **Time** in cryptography. By decoupling **Key Rotation** (changing the L2/L3 master keys) from **Data Re-encryption** (rewriting L4 records), OpenKCM allows SaaS platforms to maintain military-grade security posture—rotating keys daily or hourly if needed—without incurring the massive performance penalty of rewriting databases.