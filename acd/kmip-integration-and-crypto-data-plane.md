# ACD-303: KMIP Integration & Crypto Data Plane

| Status | Date | Document Type |
| :--- | :--- | :--- |
| **Active** | 2026-02-02 | Architecture Concept Design |

## Overview
This design document defines the standardized interface through which applications consume cryptographic services in the OpenKCM ecosystem. To ensure broad compatibility without vendor lock-in, OpenKCM adopts the **OASIS Key Management Interoperability Protocol (KMIP)** as its primary data plane API.

Both the **OpenKCM Crypto (Krypton)** and **OpenKCM Crypto (Krypton) Gateway** expose KMIP interfaces, but they serve distinct operational roles. This document details the **Split-Horizon Operations Model**, where high-frequency data plane operations are served at the Gateway, while complex lifecycle and wrapping operations are managed by the Core.

## The Split-Horizon Operations Model
OpenKCM divides KMIP responsibilities to balance sub-millisecond latency with strict governance and enhanced security isolation.

### Crypto (Krypton) Gateway: The Hot Path (L4 Authority)
The Gateway is optimized for speed and availability close to the workload. It acts as the **local authority** for the existence of L4 Data Keys.
* **Native Operations:** `Create` (L4), `Get` (L4), `Destroy` (L4), `Revoke` (L4).
* **Behavior:** It executes these operations against its local **Pluggable Internal Vault**.
* **Delegation:** For cryptographic sealing (Wrap/Unwrap), it relies on the Core. For all other KMIP verbs (e.g., `Register`, `Activate`), it proxies the request to the Core.

### Crypto (Krypton) Core: The Control Path & Sealing Engine (L2/L3)
The Core serves as the authoritative engine for the key hierarchy and the only place where L2/L3 keys are used.
* **Primary Operations:** `Encrypt` (Wrap), `Decrypt` (Unwrap), `Locate`, `Register`, `Activate`, `Check`.
* **Role:** It acts as the "Sealing Engine" for the Gateways. When a Gateway creates a key, it sends the bytes to the Core to be wrapped.
* **Exclusion:** The Core does **not** serve `Create` or `Get` requests for L4 DEKs directly to applications. This enforces the rule that L4 keys are edge-managed.

## Supported KMIP Operations Detail
OpenKCM implements a specific subset of the OASIS KMIP specification tailored for the L1-L4 hierarchy.

### Discovery and Session Operations
These operations handle the initial handshake and capability negotiation between the client and the OpenKCM node.
* **DiscoverVersions:** Negotiates the KMIP protocol version (1.x, 2.x). This is always the first handshake.
* **Query:** Returns server capabilities (supported operations, algorithms, profiles). Clients use this to adapt behavior.
* **Locate:** Searches for key objects based on attributes (e.g., Name, Group, UUID). Essential for dynamic lookups of specific Key IDs.

### Key Lifecycle Operations
These operations manage the state and existence of key material within the Registry.
* **Create:** Generates a new key (L4 at the Gateway, L2/L3 at the Core). Supports attributes like algorithm (AES), length (256), and usage masks.
* **Register:** Imports an externally generated key (e.g., a client-provided DEK).
* **Get:** Retrieves a stored key. The Gateway handles this for L4; the Core handles this for L2/L3 (internal use only).
* **GetAttributes:** Fetches metadata (algorithm, state, owner, custom tags). Used for rotation logic and auditing.
* **SetAttribute:** Updates key metadata (e.g., renaming a key, setting an activation date).
* **Re-Key:** Rotates an existing key, generating a new version linked to the old one.
* **Destroy:** Permanently deletes key material from storage. **Gateway handles L4 destruction locally.**
* **Revoke:** Transitions the key state to `Revoked`. **Gateway handles L4 revocation locally.**
* **Activate / Deactivate:** Transitions the key state. Used for lifecycle control.

### Cryptographic Operations
In the OpenKCM context, these operations are primarily used for Envelope Encryption (Wrapping/Unwrapping).
* **Encrypt (Wrap):** Uses a KEK (L3 or L2) to encrypt a lower-level key. **Always executed on the Core.**
* **Decrypt (Unwrap):** Uses a KEK to decrypt a wrapped DEK. **Always executed on the Core.**

### Administrative / Management Operations
* **Check:** A lightweight test to verify if a key exists and is usable.
* **Archive:** Moves deactivated keys to cold storage (Optional/Advanced).
* **Poll:** Used for asynchronous operations (Optional/Advanced).

## Implementation Priority
To deliver immediate value, OpenKCM follows a phased implementation strategy for the KMIP server.

| Phase | Priority | Operations | Goal |
| :--- | :--- | :--- | :--- |
| **Phase 1** | **Essential** | `DiscoverVersions`, `Query`, `Locate`, `Create`, `Get`, `Destroy`, `Revoke` | Enables full L4 lifecycle management at the Gateway. |
| **Phase 2** | **Core Logic** | `Encrypt`, `Decrypt`, `GetAttributes` | Enables the split-execution sealing logic between Gateway and Core. |
| **Phase 3** | **Advanced** | `Register`, `SetAttribute`, `Check`, `Activate` | Enables richer automation, external key imports, and health checks. |
| **Phase 4** | **Optional** | `Archive`, `Poll`, `Sign`, `Verify` | For large-scale compliance or PKI-integrated systems. |

## Architecture Flow Example
The following sequence illustrates a standard operational flow for an application integrating with OpenKCM via KMIP.

1.  **Handshake:** Client connects → `DiscoverVersions` → Negotiates protocol.
2.  **Key Gen (Gateway):** Client requests `Create` → Gateway generates L4 DEK locally.
    * *Internal Step:* Gateway calls Core `Encrypt` to wrap the new DEK with L3.
    * *Persistence:* Gateway stores Wrapped DEK in local Pluggable Vault.
3.  **Access (Gateway):** Client requests `Get (KeyID)` → Gateway fetches Wrapped DEK from local Vault.
    * *Internal Step:* Gateway calls Core `Decrypt` to unwrap the DEK.
    * *Response:* Gateway returns Plaintext DEK to client.
4.  **Revocation (Gateway):** Client requests `Revoke (KeyID)` → Gateway marks key as `Compromised` in local Vault. Subsequent `Get` requests fail immediately.
5.  **Shredding (Gateway):** Client requests `Destroy (KeyID)` → Gateway permanently deletes the blob from local Vault.

## Summary
This document establishes the rules of engagement for the Data Plane. By standardizing on KMIP, OpenKCM ensures interoperability with existing hardware and software. By enforcing the **Gateway (L4 Authority)** vs. **Core (L3 Authority)** split, it guarantees that:
1.  **High-Volume Lifecycle** (Create/Destroy) happens at the edge.
2.  **High-Security Unsealing** (Wrap/Unwrap) happens at the Core.