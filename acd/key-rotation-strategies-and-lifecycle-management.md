# ACD-304: Key Rotation Strategies & Lifecycle Management

| Status | Date | Document Type |
| :--- | :--- | :--- |
| **Active** | 2026-02-02 | Architecture Concept Design |

## Overview
Key Rotation is the "heartbeat" of a healthy cryptographic system. In the OpenKCM ecosystem, rotation is not merely a compliance checkbox but a strategic capability that ensures **Perfect Forward Secrecy** and limits the "Blast Radius" of any potential compromise.

This document defines the lifecycle logic for the entire L1–L4 hierarchy. It establishes how OpenKCM handles the complex transition from "Old Keys" to "New Keys" without disrupting high-throughput SaaS operations, utilizing a philosophy of **"Lazy Re-encryption"** for data and **"Active Re-wrapping"** for keys.



## The Versioning Paradigm
In OpenKCM, a "Key" is not a static object; it is a **Versioned Chain**.
* **Active Version (Encryption):** There is only *one* version used for *new* encryption operations (e.g., `L3_v5`).
* **Retired Versions (Decryption):** Previous versions (e.g., `L3_v1`...`v4`) remain available strictly for *decryption* of existing data until they are formally archived or purged.

This allows the platform to rotate keys instantly without needing to re-encrypt terabytes of historical data immediately.

## Rotation Strategies by Layer

The rotation strategy differs for each layer of the hierarchy based on ownership and scope.

### L1 Rotation (Customer Driven)
The **L1 Root Key** lives in the customer's external KMS (AWS/Azure/GCP).
* **Trigger:** The customer rotates the key in their cloud console (or via automated policy).
* **OpenKCM Action:** OpenKCM detects the change (or receives a signal via the **CMK Service**).
* **Mechanism (Active Re-wrap):**
    1.  The **Core** fetches the encrypted L2 Blob (currently wrapped by `L1_v1`).
    2.  The Core sends it to the External KMS to `Decrypt` (using `L1_v1`).
    3.  The Core sends the plaintext L2 back to the External KMS to `Encrypt` (using `L1_v2`).
    4.  The new L2 Blob is saved to the Registry.
* **Impact:** Zero downtime. The internal L2 key material does not change, only its "envelope" changes.

### L2 & L3 Rotation (Core Governance)
These keys (Tenant and Service keys) are managed entirely by the **OpenKCM Core**.
* **Trigger:** Time-based (e.g., every 90 days) or Event-based (e.g., "Compromise Detected").
* **Strategy: Versioned Rolling.**
    1.  **Generate:** Core generates new random key material (`L3_v2`).
    2.  **Promote:** Core updates the Registry to set `L3_v2` as `State=Active`.
    3.  **Retain:** Core marks `L3_v1` as `State=Deactivated` (Decrypt-Only).
    4.  **Distribute:** **Orbital** pushes the new `L3_v2` to all connected **Gateways**.
* **Gateway Behavior:** The Gateway immediately uses `L3_v2` for new `Create` operations. It keeps `L3_v1` in cache to service `Get` requests for older records.

### L4 Rotation (Ephemeral High-Velocity)
L4 keys are Data Encryption Keys (DEKs) managed by the **Gateway**.
* **Trigger:** Per-Transaction or Per-Session.
* **Strategy: Disposable.**
    * L4 keys are rarely "rotated" in the traditional sense. They are generated once, used to encrypt a record, and then stored.
    * **Data Rotation:** To "rotate" an L4 key effectively means to **Re-encrypt the Data** (Application reads record, decrypts, Gateway generates *new* L4, Application encrypts, saves). This is an application-layer concern.
    * **Envelope Rotation:** The Gateway *can* re-wrap an existing L4 key with a new L3 key (without decrypting the data itself) if the L4 is stored in the **Gateway Pluggable Vault**.

## Re-Encryption Strategies: Handling Historical Data

When a key rotates, what happens to the data encrypted by the old key? OpenKCM supports two distinct models.

### 1. Lazy Re-encryption (Default - High Performance)
* **Logic:** "New data uses new keys; old data stays on old keys."
* **Mechanism:** When L3 rotates from `v1` to `v2`, the Gateway simply keeps `v1` available in cache.
* **Pros:** Zero IOPS impact on the storage layer. Instant rotation.
* **Cons:** If `v1` is compromised, historical data is still vulnerable until re-encrypted.

### 2. Active Key Re-Wrapping (Security Hygiene)
* **Logic:** "Protect the keys that protect the data."
* **Mechanism:** When L2 rotates, we do not touch the data (L4). Instead, we re-encrypt the **Intermediate Keys (L3)**.
    * The Core identifies all L3 Keys wrapped by `L2_v1`.
    * The Core decrypts them and immediately re-wraps them with `L2_v2`.
    * Once finished, `L2_v1` is destroyed.
* **Benefit:** This achieves **Cryptographic Erasure** of the old L2 version without needing to touch the petabytes of actual customer data (L4).

## The Lifecycle State Machine

| State | KMIP Equivalent | Capabilities | Transition Logic |
| :--- | :--- | :--- | :--- |
| **PRE_ACTIVE** | `Pre-Active` | None | Key generated in Core but not yet distributed to Gateways. |
| **ACTIVE** | `Active` | Encrypt (Wrap) & Decrypt (Unwrap) | The primary key for all new write operations. |
| **DEPRECATED** | `Deactivated` | Decrypt (Unwrap) Only | Rotated out. Used only to read historical data. |
| **COMPROMISED** | `Compromised` | Blocked | Emergency state. All operations rejected immediately. |
| **DESTROYED** | `Destroyed` | None | Cryptographically erased. Irreversible data loss. |

## Event-Driven "Emergency" Rotation
In the event of a suspected breach, the **Orbital** engine can trigger an **Emergency Rotation**:
1.  **Command:** Admin triggers "Rotate & Revoke" for Tenant X via **CMK Portal**.
2.  **Broadcast:** Orbital notifies all Regional Cores and Gateways.
3.  **Execution:**
    * New L2/L3 versions are generated immediately.
    * Old versions are marked `COMPROMISED`.
    * **Gateways** flush their caches of the compromised key ID.
    * **Result:** Any attempt to decrypt data protected by the compromised key fails instantly across the entire mesh.

## Summary
ACD-304 defines how OpenKCM manages the dimension of **Time** in cryptography. By decoupling **Key Rotation** (changing the L2/L3 master keys in the Core) from **Data Re-encryption** (rewriting L4 records at the Edge), OpenKCM allows SaaS platforms to maintain military-grade security posture—rotating keys daily or hourly if needed—without incurring the massive performance penalty of rewriting databases.