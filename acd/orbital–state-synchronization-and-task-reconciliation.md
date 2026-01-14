# ACD-201: Orbital – State Synchronization & Task Reconciliation

| Status | Date | Document Type |
| :--- | :--- | :--- |
| **Active** | 2026-01-13 | Architecture Concept Design |

## Overview
**Orbital** is the distributed "nervous system" of OpenKCM, acting as the architectural framework responsible for synchronizing governance intent across the entire global mesh. It bridges the central **CMK Control Plane** with decentralized **Regional Crypto Execution Planes** to maintain a consistent security posture even in partitioned or multi-cloud environments.



## The Multi-Tier Mesh Architecture
Based on the latest mesh design, Orbital operates across two primary synchronization tiers to ensure global reach and regional autonomy:

* **Global-to-Regional Sync:** The **CMK Global/Mesh Registry** synchronizes high-level tenant and policy data with regional nodes via **mTLS** and **TLS + IAS JWT**.
* **Regional-to-Node Sync:** Each regional node orchestrates tasks for its local **OpenKCM Crypto Core** and manages the policy distribution to distributed **OpenKCM Crypto Edge** components.

## The Reconciliation Philosophy: Desired vs. Actual State
Orbital manages the lifecycle of system state through a continuous "Desired vs. Actual" reconciliation loop.

* **Desired State (Governance):** The authoritative configuration defined in the **Registry Service** and stored in the central **PostgreSQL** database.
* **Actual State (Execution):** The real-world status of keys and cryptographic silos as reported by regional **OpenKCM Crypto Core** nodes.

### The Convergence Loop
1.  **Detection:** The **Registry Service** and **OpenKCM Controller** monitor the **Registry** for deltas between the intended policy and the current state.
2.  **Task Generation:** The **CMK Tenant Manager** creates discrete, idempotent tasks (e.g., linking an L1 key to an L2 tenant).
3.  **Dispatch:** The **CMK API Server** publishes these tasks to the **RabbitMQ Message Broker** using the **MQTP/MQTT** protocol.
4.  **Regional Execution:** The regional **OpenKCM Crypto Core** consumes the task, performing the necessary unwrap/wrap operations or schema updates.
5.  **State Reporting:** Upon completion, the regional node updates the central registry, moving the "Actual State" into alignment with the "Desired State".



## Communication Architecture

### Task Dispatch (RabbitMQ & MQTP)
* **Protocols:** Tasks are transmitted via **RabbitMQ** using **MQTP** for reliable regional delivery.
* **Durability:** All tasks are persistent within the broker, ensuring they are retained if a regional node is temporarily offline.
* **Idempotency:** Tasks are versioned and unique; regional nodes can re-execute them without creating duplicate key material or corrupted states.

### Priority Lanes
Orbital enforces a tiered delivery system to handle critical security events:
* **Lane 0 (Immediate):** Revocation events and global "Kill-Switch" triggers initiated via the **CMK UI/Portal**.
* **Lane 1 (High):** New tenant onboarding and initial L1-L2 key linking managed by the **Tenant Manager**.
* **Lane 2 (Normal):** Routine key rotations and metadata synchronization.

## Resilience & Feedback Loops

### Regional Autonomy
OpenKCM prioritizes **Availability and Partition Tolerance (AP)**. If a region is isolated from the CMK Mesh, the **OpenKCM Crypto Core** continues to serve existing keys from its local **OpenBao** instance. Once the partition heals, Orbital "re-hydrates" the region by re-playing missed tasks in sequence.

### The "Checker" Component
Every regional cluster includes a **Checker** service that verifies the health of critical components.
* **Envoy & OPA Integration:** The Checker utilizes **Envoy** proxies and **Open Policy Agent (OPA)** for secure health introspection.
* **Feedback Signal:** If the Checker detects a failure (e.g., an unreachable HSM or Vault instance), it signals Orbital to flag the region as "Out of Sync" in the governance dashboard.

## Security Outcomes
* **Auditability:** Every task is tagged with a **Correlation ID**, enabling end-to-end traceability from a portal action to regional execution.
* **Sovereignty Enforcement:** The asynchronous nature of Orbital ensures that a customer-initiated **Revocation** is eventually consistent across all global nodes, even those currently experiencing network lag.

## Summary
This document establishes the **Operational Integrity** of the platform. By treating state synchronization as a continuous, mesh-aware reconciliation loop, Orbital ensures that OpenKCM remains:
1.  **Reliable:** Tasks are persisted and guaranteed for delivery.
2.  **Scalable:** The mesh architecture supports thousands of regional nodes managed by a central governance point.
3.  **Autonomous:** Regions can function independently while waiting for the latest state updates.