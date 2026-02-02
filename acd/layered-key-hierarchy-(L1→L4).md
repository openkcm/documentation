# ACD-301: The Layered Key Hierarchy (L1 → L4)

| Status | Date | Document Type |
| :--- | :--- | :--- |
| **Active** | 2026-02-02 | Architecture Concept Design |

## Overview
The **Layered Key Hierarchy** is the core cryptographic data structure of the OpenKCM ecosystem. It implements a strict **Recursive Envelope Encryption** model designed to solve the tension between **Data Sovereignty** (Customer Control) and **Hyperscale Performance** (Provider Speed).

Instead of using a single key to encrypt millions of records, OpenKCM organizes keys into a strict dependency tree. Each layer protects the confidentiality of the layer immediately below it. This ensures that the physical possession of encrypted data (L4) is useless without the entire chain of upstream keys (L3 → L2 → L1) remaining intact and active in the Regional Core.

## The Hierarchy Definitions
OpenKCM standardizes four distinct levels of keys, each with a specific scope, lifecycle, and "Owner."

![key-chain-representation.png](.images/key-chain-representation.png)

### L1: The Root of Trust (External)
* **Scope:** The entire Tenant or Organization.
* **Owner:** **The Customer (Sovereign).**
* **Storage:** External KMS (AWS KMS, Azure Key Vault, GCP KMS) or physical HSM.
* **Purpose:** The "Master Key" that wraps the L2. It **never** leaves the customer's controlled environment. The customer grants OpenKCM permission to `Unwrap` using this key, a permission they can revoke (Cut the Link) at any time.

### L2: The Tenant Key (Intermediate)
* **Scope:** A specific Tenant Context within the Regional Core.
* **Owner:** **OpenKCM Crypto (Krypton) Core.**
* **Storage:** Encrypted (wrapped by L1) in the Regional Core Vault.
* **Location:** **Core RAM Only.**
* **Purpose:** The mathematical root of multi-tenancy. Even if the database is compromised, Tenant A's L2 key cannot be loaded into memory without a real-time call to Tenant A's external L1.

### L3: The Service / KEK (Intermediate)
* **Scope:** A logical domain, service, or compliance boundary (e.g., "Payments Service," "EU-Log-Archive").
* **Owner:** **OpenKCM Crypto (Krypton) Core.**
* **Storage:** Encrypted (wrapped by L2) in the Regional Core Vault.
* **Location:** **Core RAM Only.**
* **Constraint:** **Never leaves the Core.** It is never sent to the Gateway or the Application.
* **Purpose:** Acts as the **Key Encryption Key (KEK)** for high-volume operations. The Core uses this key to wrap/unwrap the millions of L4 keys sent to it by Gateways.

### L4: The Data Encryption Key / DEK (Leaf)
* **Scope:** A single record, file, object, or session.
* **Owner:** **Krypton Gateway / Application.**
* **Storage:** Encrypted (Wrapped by L3) in the **Gateway's Pluggable Internal Vault**.
* **Location:** Ephemeral RAM (Plaintext) during use; Pluggable Vault (Ciphertext) at rest.
* **Purpose:** High-speed bulk encryption. Millions of L4 keys can exist, all protected by a single L3 key held in the Core.

## The Recursive Unsealing Flow
The system "boots" a tenant's context through a specific sequence, moving from **Slow/Sovereign** (L1) to **Fast/Local** (L4).

1.  **L1 Handshake (The Link):** The **Krypton Core** presents the Encrypted L2 Blob to the customer's external KMS. The KMS decrypts it (validating the customer's policy) and returns the **Plaintext L2** into the Core's secure memory.
2.  **L3 Derivation:** The Core uses the L2 key to decrypt the specific **L3 Service Keys** required for the active services. **The L3 keys remain locked in the Core.**
3.  **L4 Generation (Gateway):** A **Krypton Gateway** receives a request. It generates a high-entropy **Plaintext L4** locally.
4.  **L4 Sealing (Remote Wrap):** The Gateway sends the Plaintext L4 to the Core via gRPC. The Core wraps it with the cached **L3 Key** and returns the **Ciphertext L4**.
5.  **L4 Persistence:** The Gateway stores the **Ciphertext L4** in its local **Pluggable Internal Vault**.

## Rotation & Lazy Re-Wrapping
Key rotation in OpenKCM follows a "Lazy" strategy to ensure performance stability.

* **L3 Rotation Logic:** When an L3 key is rotated (e.g., `L3_v1` → `L3_v2`), the Core does *not* immediately force a re-encryption of all millions of L4 keys.
* **Legacy Support:** The Core retains `L3_v1` in a "Decrypt-Only" state.
* **L4 Validity:** Existing L4 keys (wrapped by `L3_v1`) remain valid. The Gateway sends the `Ciphertext L4` + `Key Version ID` to the Core. The Core identifies that it was wrapped by `L3_v1`, unwraps it using the old key, and returns the plaintext.
* **Opportunistic Upgrade:** Upon access, the Gateway *may* request the Core to re-wrap the L4 key with the *current* active version (`L3_v2`), effectively migrating the data key to the new hierarchy without decrypting the underlying customer data.

## Mathematical Siloing & Multi-Tenancy
This hierarchy enforces **"Mathematical Isolation."** In a traditional SaaS, a software bug might allow Tenant B to read Tenant A's data. In OpenKCM, even if the software fails, the cryptography holds.

* To read Data (L4), you need the **Plaintext L4**.
* To get Plaintext L4, you must send **Ciphertext L4** to the Core.
* The Core will only unwrap if it holds **Tenant A's L3 Key**.
* The Core only holds L3 if it successfully loaded **Tenant A's L2 Key**.
* The Core could only load L2 if **Tenant A's External L1** allowed it.
* **Therefore: Access to Data = Access to L1 Root.**

## Crypto-Shredding & Revocation
The hierarchy enables instant, verifiable data destruction at different blast radii.

### 1. Granular Shredding (L4 Deletion)
* **Action:** Gateway deletes a specific **Ciphertext L4** from its Pluggable Vault.
* **Result:** A single record or file becomes permanently unreadable.
* **Authority:** Gateway (Local).

### 2. Domain Shredding (L3 Destruction)
* **Action:** An administrator explicitly **Deletes** a specific L3 Key Version (e.g., `L3_v1`) from the Core.
* **Result:** Since `L3_v1` is permanently removed from memory and storage, the millions of L4 keys wrapped by that specific version can never be unwrapped again.
* **Distinction:** This is distinct from *Rotation* (which keeps the old key). Shredding is a destructive, irreversible compliance action.
* **Authority:** Core (Regional).

### 3. Sovereign Revocation (L1 Kill-Switch)
* **Action:** Customer disables their **L1 Key** in AWS/Azure/GCP.
* **Result:** The Core can no longer unwrap the L2 key.
* **Outcome:** The entire Tenant Context in the Core expires. The Gateway's entire Pluggable Vault (holding millions of L4s) becomes a graveyard of useless ciphertext.
* **Authority:** Customer (Global).

## Summary
This document defines the data structure that makes OpenKCM possible. The **L1 → L4 Hierarchy** allows the platform to balance two opposing requirements:
1.  **Sovereignty:** The customer retains ultimate control via the L1.
2.  **Speed:** The application enjoys local speed via the L4.
3.  **Security:** The **Split-Execution** model ensures the expensive remote calls (L1) happen rarely, while the cheap local calls (L4) happen frequently, without ever exposing the Root Keys to the edge.