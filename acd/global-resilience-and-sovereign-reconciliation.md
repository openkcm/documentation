# ACD-402: Global Resilience & Sovereign Reconciliation

| Status | Date | Document Type |
| :--- | :--- | :--- |
| **Active** | 2026-02-02 | Architecture Concept Design |

## Overview
**Global Resilience** in OpenKCM defines how the platform survives catastrophic infrastructure failures without compromising cryptographic sovereignty or data consistency.

The guiding principle is **"Radical Component Independence."** OpenKCM ensures that the **CMK Registry**, **CMK Server**, and **Krypton Core** operate as fully autonomous units. They do not depend on synchronous connectivity to each other to serve keys or process cryptographic operations.

## The Architecture of Independence
To achieve high availability, OpenKCM relies on **Orbital**, an asynchronous event mesh, to propagate state only when changes occur. There is no "chatty" API communication between components during normal operation.

### 1. The Autonomous Actors
Each component maintains its own local state and operates independently within its specific domain:

* **CMK Registry (The Architect):** Defines the **Intent** and **Metadata**.
  * It manages Tenant IDs and the **Key Definitions** (e.g., "Tenant X requires a 'System' L2 key").
  * It dictates *what* should exist, but never holds the actual key material.
* **CMK Server (The Sovereign Anchor):** Manages the **L1 Pointers**.
  * It holds the authoritative **Shadow References** (ARNs) required to access the customer's external Root of Trust.
  * It manages the Link/Unlink lifecycle events.
* **Krypton Core (The Builder):** Executes the **Implementation**.
  * It receives definitions from the Registry and pointers from the CMK Server.
  * It generates, rotates, and manages the actual **Key Material** (L2/L3 blobs).
  * It serves the data plane requests directly.

### 2. The Orbital Event Flows
Data is exchanged *only* when a state change occurs ("Propagation on Existence"). If the mesh is down, components continue operating on their last known good state.

| Source | Destination                           | Event Type | Purpose                                                                                                      | Trigger |
| :--- |:--------------------------------------| :--- |:-------------------------------------------------------------------------------------------------------------| :--- |
| **Registry** | **CMK Server** </br> **Krypton Core** | `Tenant.Define` | Informs about tenant exists and is valid.                                                                    | Tenant Onboarding / Offboarding |
| **Registry** | **Krypton Core**                      | `Key.Define` | Pushes **L2/L3 Metadata Definitions**. Instructs Krypton Core to generate/load material for specific scopes. | Service Onboarding / Offboarding |
| **CMK Server** | **Krypton Core**                      | `L1.Reference` | Pushes the **L1 Pointer** (ARN) and **Link/Unlink** instructions.                                            | Customer links a new External Key |

## Disaster Recovery Tiers

### Tier 1: Regional Autonomy (Network Partition)
*Scenario: A fiber cut isolates the Regional Krypton Core from the Global Orbital Mesh.*

* **Operational Status:** **Fully Functional.**
  * The **Krypton Core** serves requests using the L2/L3 definitions and L1 pointers it previously received and cached.
* **Cryptographic Lifecycle:** **Unaffected.**
  * The **Krypton Core** continues to **Rotate L2 & L3 keys** on schedule.
  * It communicates directly with the External L1 Provider (via Plugins) to wrap new keys.
* **Impact:** The Region cannot receive *new* Key Definitions (e.g., deploying a new Microservice requiring a new L3 key), but all existing services operate normally.

### Tier 2: Component Failure (Independent Survival)
*Scenario: The Global Registry crashes, but the CMK Server and Krypton Cores are healthy.*

* **CMK Server:** Continues to manage L1 Links for existing tenants.
* **Krypton Core:** Continues to serve encryption and rotate keys for existing definitions.
* **Registry:** The platform cannot **Define New Keys** or **Onboard New Tenants** until restored.

### Tier 3: Region Failure (Cluster Loss)
*Scenario: A catastrophic failure destroys an entire Regional Krypton Core.*

* **Failover:** Traffic routes to a secondary region.
* **Hydration:**
  1.  Survivor Core receives **Key Definitions** (Metadata) from **Registry** (via Orbital).
  2.  Survivor Core receives **L1 Pointer** from **CMK Server** (via Orbital).
* **Recovery:** The Survivor Core uses the Pointer to unlock the L2 structure and regenerates or retrieves the L3 material matching the Registry's definitions.

## Sovereign Conflict Resolution (Split-Brain)
In a distributed system, eventual consistency can lead to temporary conflicts. OpenKCM resolves these by checking against the specific authority for each state type.

1.  **Drift Detection:** The **Krypton Core** periodically emits a `State.Manifest` (containing active Tenant/Service IDs) via Orbital.
2.  **Definition Reconciliation (Registry):** The **CMK Registry** consumes the manifest. If it detects the **Krypton Core** is holding keys for a Service that was **Deleted** from the Registry, it issues a `Key.Purge` event.
3.  **Sovereign Reconciliation (CMK Server):** The **CMK Server** consumes the manifest. If it detects the **Krypton Core** is serving a Tenant that has been **Unlinked** (L1 Revoked), it issues a high-priority `L1.Revoke` event.
4.  **The Deadman Switch:** If a **Krypton Core** is isolated from the **CMK Server** (its Sovereign Anchor) for longer than the **Strict Sovereign TTL** (e.g., 24 hours), it voluntarily pauses key serving to prevent "Zombie Sovereignty."

## Summary
ACD-402 defines a resilience model based on **Asynchronous Independence**. By delegating the **Definition** of keys to the Registry, the **Sovereignty** to the CMK Server, and the **Generation** to the Krypton Core, OpenKCM guarantees that management outages never impact the security or availability of the active data plane.