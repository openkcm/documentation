# ACD-303: KMIP Integration & Crypto Data Plane

| Status | Date | Document Type |
| :--- | :--- | :--- |
| **Active** | 2026-01-13 | Architecture Concept Design |

## Overview
This design document defines the standardized interface through which applications consume cryptographic services in the OpenKCM ecosystem. To ensure broad compatibility without vendor lock-in, OpenKCM adopts the **OASIS Key Management Interoperability Protocol (KMIP)** as its primary data plane API.

Both the **OpenKCM Crypto (Krypton)** and **OpenKCM Crypto (Krypton) Gateway** expose KMIP interfaces, but they serve distinct operational roles. This document details the **Split-Horizon Operations Model**, where high-frequency data plane operations are served at the Gateway, while complex lifecycle and wrapping operations are managed by the Core.



## The Split-Horizon Operations Model
OpenKCM divides KMIP responsibilities to balance sub-millisecond latency with strict governance and enhanced security isolation.

### Crypto (Krypton) Gateway: The Hot Path (L4 Data Plane)
The Gateway is optimized for speed and availability close to the workload. It is the **exclusive** handler for ephemeral Data Encryption Keys (DEKs).
* **Primary Operations:** `Create` (L4 DEK), `Get` (L4 DEK).
* **Behavior:** Local execution using cached L3 Service Keys.
* **Delegation:** All other operations (e.g., `Destroy`, `Revoke`, `Register`) received by the Gateway are securely proxied to the Crypto (Krypton).

### Crypto (Krypton): The Control Path & Key Hierarchy (L2/L3)
The Core serves as the authoritative engine for the key hierarchy and lifecycle management. It handles operations related to Key Encryption Keys (KEKs) and acts as the destination for delegated tasks.
* **Primary Operations:** `Encrypt` (Wrap), `Decrypt` (Unwrap), `Locate`, `Register`, `Activate`, `Destroy`.
* **Exclusion:** The Core does **not** serve `Create` or `Get` requests for L4 DEKs. These are served only by the Gateway to prevent the storage or transit of L4 keys within the central crypto data plane. This ensures that the most granular level of encryption material remains localized and isolated from the global governance hub.

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
* **Get:** Retrieves a stored key or its wrapped form. Access is strictly controlled by policy.
* **GetAttributes:** Fetches metadata (algorithm, state, owner, custom tags). Used for rotation logic and auditing.
* **SetAttribute:** Updates key metadata (e.g., renaming a key, setting an activation date).
* **Re-Key:** Rotates an existing key, generating a new version linked to the old one.
* **Destroy:** Permanently deletes key material from storage. Critical for GDPR compliance and crypto-shredding.
* **Activate / Deactivate:** Transitions the key state. Used for lifecycle control.

### Cryptographic Operations
In the OpenKCM context, these operations are primarily used for Envelope Encryption (Wrapping/Unwrapping).
* **Encrypt (Wrap):** Uses a KEK (L3 or L2) to encrypt a lower-level key or data.
* **Decrypt (Unwrap):** Uses a KEK to decrypt a wrapped DEK. This is the core function for allowing applications to access their data.

### Administrative / Management Operations
* **Check:** A lightweight test to verify if a key exists and is usable.
* **Archive:** Moves deactivated keys to cold storage (Optional/Advanced).
* **Poll:** Used for asynchronous operations (Optional/Advanced).

## Implementation Priority
To deliver immediate value, OpenKCM follows a phased implementation strategy for the KMIP server.

| Phase | Priority | Operations | Goal |
| :--- | :--- | :--- | :--- |
| **Phase 1** | **Essential** | `DiscoverVersions`, `Query`, `Locate`, `Create`, `Get`, `Encrypt`, `Decrypt`, `GetAttributes` | Enables full L4 key generation, wrapping/unwrapping, and basic retrieval. |
| **Phase 2** | **Lifecycle** | `Rekey`, `Destroy`, `Activate`, `Deactivate` | Adds robust rotation and state management capabilities. |
| **Phase 3** | **Advanced** | `Register`, `SetAttribute`, `Check` | Enables richer automation, external key imports, and health checks. |
| **Phase 4** | **Optional** | `Archive`, `Poll`, `Sign`, `Verify` | For large-scale compliance or PKI-integrated systems. |

## Architecture Flow Example
The following sequence illustrates a standard operational flow for an application integrating with OpenKCM via KMIP.



1.  **Handshake:** Client connects → `DiscoverVersions` → Negotiates protocol.
2.  **Discovery:** Client queries → `Query` / `Locate` → Discovers available algorithms and locates the correct Service Key (L3).
3.  **Key Gen (Gateway):** Client requests `Create` → Gateway generates L4 DEK locally.
4.  **Wrapping (Gateway):** Client sends DEK to be secured → `Encrypt` (using L3) → Gateway returns Wrapped DEK.
5.  **Storage (Client):** Client stores the Wrapped DEK alongside its encrypted data.
6.  **Access (Gateway):** Client reads data → Sends Wrapped DEK → `Decrypt` (using L3) → Gateway returns Plaintext DEK.
7.  **Rotation (Core):** Lifecycle policy triggers → `Rekey` → Core generates new version of L3 KEK.
8.  **Shredding (Core):** Data deletion request → `Destroy` → Core permanently erases key material.

## Summary
This document establishes the rules of engagement for the Data Plane. By standardizing on KMIP, OpenKCM ensures interoperability with existing hardware and software. By enforcing the **Core (Lifecycle)** vs. **Gateway (Hot Path)** split—and specifically excluding L4 keys from the Core—it guarantees that high-volume encryption never compromises the stability or security of the global governance layer.