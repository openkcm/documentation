# ACD-201: Orbital – State Synchronization & Task Reconciliation

| Status | Date | Document Type |
| :--- | :--- | :--- |
| **Active** | 2026-01-31 | Architecture Concept Design |

## Overview
**Orbital** is the distributed "Nervous System" of OpenKCM. In a complex, multi-regional Sovereign Cloud, business intent must be synchronized across different layers of the infrastructure to ensure consistency and security.

Orbital functions as the **Strategic Event Mesh** that connects:
1.  **The Governance Layer to the Execution Layer (Vertical):** Managing the full **Key Linking and Unlinking of L2 Keys to L1 Roots** (Activation and Revocation) on the Crypto (Krypton) nodes.
2.  **The Global Registry to Regional CMKs (Horizontal A):** Orchestrating the lifecycle of tenants across multiple geographic management planes.
3.  **The Global Registry to Regional Cryptos (Horizontal B):** Orchestrating the lifecycle of tenant enclaves across multiple geographic execution planes.



## Strategic Capabilities

### 1. Vertical Synchronization: The "Sovereign Link" (L2 to L1)
**Flow:** **Regional CMK (Governance) ↔ Regional Crypto (Execution)**

This is the local security loop. It translates a regional governance decision into a cryptographic reality within the specific data center.

* **L2 Link (Activation):**
    * **Trigger:** An administrator in the Regional Portal approves a workflow to "Activate Tenant Keys."
    * **Action:** Orbital delivers a signed command to the local **Crypto (Krypton)** node to cryptographically bind the Tenant's **L2 Key** with the Hardware **L1 Root Key**.
    * **Business Value:** Turns a static "Tenant Context" into a live, usable encryption service securely anchored to local hardware.
* **L2 Unlink (The Sovereign Kill-Switch):**
    * **Trigger:** A security event, contract termination, or explicit "Revoke" request.
    * **Action:** Orbital broadcasts a high-priority **Unlink** event. The Crypto layer immediately **drops the L2 key from secure memory** and severs the connection to the L1 Root.
    * **Business Value:** Guarantees that data access is physically terminated the moment the business decision is made.

### 2. Horizontal Synchronization A: Global Governance
**Flow:** **Global Registry ↔ Regional CMK Deployments**

This vector ensures that the "Management Surface" is consistent globally.

* **Tenant Metadata Sync:**
    * **Action:** Orbital pushes `SYNC_TENANT_METADATA` events to Regional CMK clusters.
    * **Business Value:** Ensures that when an admin logs into the "EU-Germany" portal, they see the exact same tenant configuration and labels as they do in the "US-East" portal.
* **Global Admin Identity:**
    * **Action:** Syncs RBAC definitions for administrators.
    * **Business Value:** "Manage Once, Access Anywhere."

### 3. Horizontal Synchronization B: Global Execution
**Flow:** **Global Registry ↔ Regional Crypto (Krypton) Nodes**

This vector manages the physical existence of the tenant on the encryption infrastructure globally. It allows the Global Registry to directly orchestrate the lifecycle of secure memory enclaves without relying on regional management layers.

* **Global Context Creation:**
    * **Trigger:** New Tenant Onboarding in Global Registry.
    * **Action:** Orbital broadcasts a `CREATE_TENANT_CONTEXT` event directly to **ALL** authorized Regional Crypto Nodes.
    * **Business Value:** Pre-provisions the secure infrastructure instantly across the globe.
* **Global Offboarding (The Ultimate Wipe):**
    * **Trigger:** Contract Termination.
    * **Action:** Orbital broadcasts a `DESTROY_TENANT_CONTEXT` command to every Crypto Node in the fleet.
    * **Business Value:** Guarantees **Global Data Sanitization** by ensuring the memory footprint is physically wiped from every data center simultaneously.

## Hybrid Communication Architecture

To support these three vectors, Orbital employs a **Hybrid Transport Model**.

### Path A: The Message Fabric (RabbitMQ)
Used for **All Three Vectors** when durability is required.
* **Usage:** Tenant Creation, Context Destruction, Metadata Sync.
* **Business Guarantee:** **"Global Consistency."** If a region is temporarily isolated, the command is queued and auto-reconciled later.

### Path B: The Direct Control Link (gRPC)
Used for **Vertical Synchronization** (Vector 1) and Telemetry.
* **Usage:** Real-time **L2 to L1 Linking**, Unlinking, Health Checks, Audit Streaming.
* **Business Guarantee:** **"Instant Responsiveness."** Immediate execution of critical security actions.

## Priority Lanes (Quality of Service)
Orbital prioritizes security-critical events over routine synchronization.

| Lane | Priority | Business Use Case | Guarantee |
| :--- | :--- | :--- | :--- |
| **Lane 0** | **Critical** | **L2 Unlink (Vector 1) / Global Offboarding (Vector 3).** | **Broadcast.** Sent via all channels to ensure immediate execution and global termination. |
| **Lane 1** | High | **Context Creation (Vector 3) / L2 Link (Vector 1).** | **Priority.** Ensures new customers are ready to transact immediately. |
| **Lane 2** | Standard | **Metadata Sync (Vector 2).** | **Reliable.** Keeps tenant names and labels in sync. |

## Summary
**ACD-201** establishes Orbital as the **Synchronization Engine** of OpenKCM. By explicitly defining vectors from the **Global Registry** to both the **Regional Management Plane (CMK)** and the **Regional Data Plane (Crypto)**, it ensures that:

1.  **Management is Global:** Admins see a unified view of the world.
2.  **Infrastructure is Global:** Tenant enclaves are provisioned everywhere instantly.
3.  **Sovereignty is Local:** Actual **L2 to L1 Key Linking and Unlinking** remains a local action.