# ACD-202: OpenKCM Crypto (Krypton) – Regional Governance & KMIP Unsealing

| Status | Date | Document Type |
| :--- | :--- | :--- |
| **Active** | 2026-01-31 | Architecture Concept Design |

## Overview
**OpenKCM Crypto (Krypton)** is the regional "Execution Authority" and cryptographic backbone of the ecosystem. While the central CMK Service manages global governance, Krypton is the only component responsible for the **physical custody and execution** of encryption operations.

Its primary mission is to maintain the **Sovereign Chain of Trust** by recursively unsealing keys from the customer's Hardware Root (L1) down to the Data Encryption Key (L4). It serves as the "Sovereign Vault" that holds the intermediate keys (L2/L3) in secure memory, ensuring they are never exposed to the network or the ephemeral Gateways.

## The Regional Component Stack
Krypton is a high-integrity cluster designed for **Zero-Trust Execution** and **Vendor Neutrality**:

* **Krypton Core (The Orchestrator):** The Go-based engine that receives lifecycle commands from **Orbital** and exposes the KMIP interface for cryptographic operations. It is the authoritative "Brain" of the region.
* **Pluggable Internal Vault:** A strictly defined **Storage Plugin Interface** that abstracts the physical persistence of secrets. The platform creates no vendor lock-in; customers or platform engineers can select the storage backend that fits their compliance needs. The core orchestrates the encryption logic, while the selected plugin handles the secure storage of the encrypted **L2/L3 Blobs**.

## The Cryptographic Hierarchy (L1 → L4)
Krypton enforces a strict hierarchy where every layer protects the one below it.

| Layer | Name | Location | Role |
| :--- | :--- | :--- | :--- |
| **L1** | **Hardware Root** | **Customer HSM/KMS** | The physical anchor (e.g., AWS KMS ARN). Krypton never sees this private key; it only calls the API to `Decrypt` the L2 blob. |
| **L2** | **Tenant Root** | **Krypton Secure Memory** | The "Sovereign Link." Unsealed via L1 and held in RAM to verify tenant activity. |
| **L3** | **Key Encryption Key (KEK)** | **Krypton Secure Memory** | Intermediate keys used to wrap/unwrap L4 keys. **These never leave the Core.** |
| **L4** | **Data Encryption Key (DEK)** | **Application / Gateway** | The ephemeral keys encrypting actual data. Managed by the Gateway but wrapped by the Core. |

## Communication Protocols

### Northbound: Orbital Integration
* **Protocol:** **RabbitMQ (MQTP)** or **gRPC** for asynchronous lifecycle tasks (Link/Unlink).
* **Function:** Synchronizing the "Desired State" (Global Registry) with the "Actual State" (Regional Vault).

### Southbound: KMIP & Gateway Delegation
* **Protocol:** **OASIS KMIP** over mTLS.
* **Function:** Serving as the authoritative engine for L4 wrapping operations.
* **Delegation:** The **Krypton Gateway** (ACD-203) delegates all *Unwrap* operations to the Core. The Core validates the request, performs the transform using the cached L3 key, and returns the result.

## The Sovereign Workflow

### 1. The "Sovereign Link" (Activation)
This process turns a static database record into a live encryption service.
1.  **Command:** Orbital sends a `LINK_KEY` task with the L1 Reference.
2.  **Handshake:** Krypton authenticates to the Customer's External KMS (AWS/Azure) and requests decryption of the L2 Blob.
3.  **Activation:** The decrypted **L2 Key** is loaded into **Secure Memory**. The tenant is now "Active."

### 2. The Execution Loop (KMIP Unsealing)
1.  **Request:** A Gateway sends a **Ciphertext L4 DEK** to the Core.
2.  **Validation:** The Core checks if the Tenant's L2 Key is loaded (Active).
3.  **Execution:** The Core uses the **L3 KEK** (derived from L2) to unwrap the L4 DEK.
4.  **Response:** The **Plaintext L4 DEK** is returned to the Gateway.

### 3. The "Kill-Switch" (Revocation)
1.  **Command:** Orbital broadcasts an `UNLINK_KEY` task (Lane 0 Priority).
2.  **Action:** Krypton **zeroizes the memory regions** holding the L2 and L3 keys.
3.  **Outcome:** The "Chain of Trust" is broken instantly. All subsequent KMIP requests fail because the Core no longer possesses the keys required to unwrap L4 data.

## Regional Autonomy & Persistence
To ensure resilience, Krypton operates with **Cached Autonomy**.

* **Regional Persistence:** The Core uses the configured **Storage Plugin** to persist the *encrypted* state of the key hierarchy. If the entire cluster restarts, it retrieves the blobs via the plugin and contacts the Customer's External L1 KMS to re-establish trust.
* **Fail-Safe:** While it can continue serving existing keys during a control plane outage, it enforces a **"Fail-Closed"** posture for revocation. If the L1 link is severed (e.g., customer disables AWS KMS key), Krypton eventually fails to re-wrap its internal vault, naturally terminating access.

## Summary
**ACD-202** defines the regional "Root of Authority" for OpenKCM. By serving as the **KMIP Engine** that protects the **L3 KEK**, the **OpenKCM Crypto (Krypton)** ensures that:

1.  **Sovereignty is Maintained:** The L1-to-L4 chain is mathematically isolated and can be severed instantly.
2.  **Storage is Flexible:** The architecture supports **Plug-and-Play Backends**, allowing implementations to be selected via the **Internal Vault Plugin** based on deployment needs.
3.  **Governance is Enforced:** No data key can be unwrapped without passing through the policy checks of the Core.