# ACD-203: OpenKCM Crypto (Krypton) Gateway – High-Performance Ephemeral Data Plane

| Status | Date | Document Type |
| :--- | :--- | :--- |
| **Active** | 2026-01-31 | Architecture Concept Design |

## Overview
The **OpenKCM Crypto (Krypton) Gateway** is the distributed execution component designed to run physically close to customer workloads (e.g., Sidecar, DaemonSet). It serves as the ultra-low-latency interface for application encryption operations, prioritizing throughput and connection management.

To achieve hyperscale performance while maintaining **Sovereign Isolation**, the Gateway operates on a strict **Split-Execution Model**:

1.  **L4 Lifecycle Management (The Hot Path):** The Gateway natively handles **KMIP Create, Get, Destroy, and Revoke** for L4 Data Keys by interacting with its local **Pluggable Internal Vault** and the Core.
2.  **Delegated Execution (The Control Path):** **All other KMIP operations** (e.g., `Activate`, `Register`, `Locate`, `Query`, `Check`) are immediately forwarded to the **Krypton Core** for authoritative execution.

## The Split-Function Philosophy
Unlike traditional monolithic KMS appliances, OpenKCM separates high-volume *L4 Data Key* operations from global *Governance* operations.

### The Hot Path (L4 Operations)
The Gateway is the authority for the **existence** of L4 keys. It generates them, stores them (encrypted) in its Pluggable Vault, and deletes them upon request. However, it relies on the **Krypton Core** to perform the cryptographic **Sealing (Wrap/Unwrap)** of those keys using the L3 Root.

### The Control Path (Everything Else)
Operations that require complex queries, global registry lookups, or operations on L2/L3 keys are outside the Gateway's scope. It acts as a transparent, mTLS-authenticated proxy for these requests, tunneling them directly to the Regional Core.

## Architectural Components
The Krypton Gateway acts as a specialized, cryptographic forwarder with flexible storage.

* **KMIP Listener (mTLS):** The standardized front-door that accepts incoming connections from workload clients.
* **Local RNG Engine:** Generates high-entropy L4 keys locally.
* **Pluggable Internal Vault:** A strictly defined **Storage Plugin Interface** used to persist, update, and delete **Wrapped L4 Keys**. *Note: This Vault holds the lock (Ciphertext), but never the key (L3).*

## Operational Workflows

### 1. KMIP Create (Generate & Seal)
* **Action:** Gateway generates **Plaintext L4** locally.
* **Delegation:** Gateway sends Plaintext L4 to Core → Core wraps with L3 → Returns **Ciphertext L4**.
* **Persistence:** Gateway stores **Ciphertext L4** in **Pluggable Internal Vault**.
* **Response:** Gateway returns Plaintext L4 and KeyID to client (then wipes memory).

### 2. KMIP Get (Fetch & Verify)
* **Action:** Gateway retrieves **Ciphertext L4** from **Pluggable Internal Vault**.
* **Check:** If Key State is `Revoked` in local metadata, return Error.
* **Delegation:** Gateway sends Ciphertext L4 to Core → Core verifies L2/L3 state → Returns **Plaintext L4**.
* **Response:** Gateway returns Plaintext L4 to client.

### 3. KMIP Destroy (Delete)
* **Action:** Application requests destruction of an L4 KeyID.
* **Execution:** Gateway locates the key in the **Pluggable Internal Vault**.
* **Deletion:** Gateway permanently removes the **Ciphertext L4** blob and metadata from the storage backend.
* **Response:** Gateway returns `Success`. *Note: Since the blob is gone, the key is cryptographically erased.*

### 4. KMIP Revoke (Update State)
* **Action:** Application requests revocation of an L4 KeyID.
* **Execution:** Gateway updates the metadata of the key in the **Pluggable Internal Vault**, setting `State = Revoked`.
* **Enforcement:** Any subsequent `Get` request will be blocked by the Gateway's local state check.
* **Response:** Gateway returns `Success`.

### 5. The Control Path (All Other Operations)
* **Scope:** `Activate`, `Register`, `Locate`, `Check`, `Query`, `Discovery`, etc.
* **Execution:**
    1.  Gateway parses KMIP header.
    2.  Gateway identifies operation is **NOT** Create/Get/Destroy/Revoke.
    3.  Gateway tunnels the request to **Krypton Core**.
    4.  Core executes logic against global registry.
    5.  Response relayed to client.

## Resilience vs. Sovereignty
This architecture makes a deliberate trade-off to prioritize **Sovereignty** over offline autonomy.

* **If Network to Core is Severed:**
    * **Create/Get Fail:** Gateway cannot contact Core to Wrap/Unwrap.
    * **Destroy/Revoke Succeed (Conditionally):** If the Pluggable Vault is local/available, the Gateway *can* destroy or revoke keys locally even if the Core is offline, as these are storage operations.
    * **Delegated Ops Fail:** Tunnel is down.
    * **Benefit:** The system fails closed for Access (Get), but remains open for Cleanup (Destroy).

## Summary
ACD-203 defines the **Krypton Gateway** as a specialized L4 Manager:

1.  **L4 Authority:** It fully manages the lifecycle (`Create`, `Get`, `Revoke`, `Destroy`) of Data Keys via its **Pluggable Internal Vault**.
2.  **Cryptographic Dependency:** It depends on the Core solely for the **L3 Wrap/Unwrap** functions.
3.  **Proxy for Governance:** It transparently forwards all other complex requests to the Core.