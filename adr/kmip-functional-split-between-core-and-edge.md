# ADR-603: KMIP Functional Split between Core and Gateway

| Status | Date | Document Type |
| :--- | :--- | :--- |
| **Active** | 2026-01-17 | Architecture Design Record |

## Context
In the OpenKCM ecosystem, we must support two conflicting operational profiles:
1.  **High-Frequency Data Plane:** Applications need to encrypt/decrypt millions of data records (L4 DEKs) per minute with sub-millisecond latency.
2.  **High-Assurance Control Plane:** The platform must manage the lifecycle of Master Keys (L2/L3) with strict auditability, policy checks, and consistency, even if it takes 100-200ms.

**The Problem:** A monolithic KMIP server cannot satisfy both.
* If we process L4 operations in the central Core, network latency (Core-to-App) destroys application throughput.
* If we push full L2/L3 management to the Gateway, we expand the attack surface and lose centralized governance (Split-Brain).

**The Requirement:** We need a strict functional partition of the KMIP specification that optimizes the "Gateway" for speed and the "Core" for security.

## Decision
We will enforce a **Split-Horizon KMIP Router** logic at the application SDK and Gateway level. The OpenKCM ecosystem will not be a single KMIP server, but a distributed mesh where specific operations are routed to specific nodes.

### 1. The Gateway Scope (Hot Path)
The **Crypto (Krypton) Gateway** is authoritative *only* for ephemeral **L4 Data Encryption Keys (DEKs)**.
* **Supported Operations:**
    * `Create` (L4 Only)
    * `Get` (L4 Only - from local cache/storage)
    * `Encrypt` (Wrap L4 using cached L3)
    * `Decrypt` (Unwrap L4 using cached L3)
* **Behavior:** These operations execute locally in the Gateway process memory. They **never** trigger a synchronous call to the Core (unless the L3 cache is empty).

### 2. The Core Scope (Control Path)
The **Crypto (Krypton)** is authoritative for **L2 (Tenant)** and **L3 (Service)** Keys and all Lifecycle events.
* **Supported Operations:**
    * `Register` (Importing keys)
    * `Rekey` (Rotation)
    * `Activate` / `Revoke` / `Destroy`
    * `Locate` (Global search)
    * `GetAttribute` / `SetAttribute`
* **Behavior:** These operations involve database writes (KSI) and Orbital synchronization. They are strictly forbidden at the Gateway.

### 3. The Proxy Mechanism
If the Crypto (Krypton) Gateway receives a Control Path request (e.g., `Destroy Key`), it does not reject it outright. Instead, it acts as a **Transparent Proxy**:
1.  **Identify:** Gateway parser detects `Operation=Destroy`.
2.  **Authenticate:** Gateway validates the mTLS identity of the caller.
3.  **Forward:** Gateway tunnels the raw KMIP packet to the upstream Crypto (Krypton) via a persistent mTLS connection.
4.  **Relay:** Core executes the logic and returns the response; Gateway passes it back to the client.
    *This preserves the illusion of a single endpoint for the application developer.*

## Consequences

### Positive (Pros)
* **Performance:** 99.9% of traffic (L4 ops) hits the Gateway and returns in <1ms.
* **Security:** L2/L3 key material never leaves the Core's secure vault (except when L3 is pushed to Gateway memory). The MasterKey never leaves the Core.
* **Governance:** Complex state changes (Rotation, Revocation) are serialized through the Core, preventing "Split-Brain" states where two Edges think the key has different states.

### Negative (Cons)
* **Router Complexity:** The Gateway must implement a smart KMIP packet inspector to decide whether to `Execute` locally or `Proxy` upstream.
* **Partial Failure Modes:** If the link between Gateway and Core is down, `Create L4` works (Availability), but `Rotate L3` fails (Consistency). This is an acceptable trade-off for the "AP" (Availability/Partition) design of the Data Plane.

## References
* [ACD-303: KMIP Integration & Crypto Data Plane](../acd/kmip-integration-and-crypto-data-plane.md)
* [ACD-203: Crypto (Krypton) Gateway – High-Performance Ephemeral Data Plane](../acd/crypto-gateway–high-performance-ephemeral-data-plane.md)
* [ADR-602: Crypto (Krypton) Gateway Workload Orchestration](crypto-gateway-workload-orchestration-and-dek-lifecycle.md)