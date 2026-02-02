# ACD-101: OpenKCM CMK (Customer Master Key) – Governance Control Plane

| Status | Date | Document Type |
| :--- | :--- | :--- |
| **Active** | 2026-02-02 | Architecture Concept Design |

## Overview
The **OpenKCM CMK (Customer Master Key) Service** is the centralized Governance Control Plane of the OpenKCM ecosystem. It serves as the authoritative "Brain" of the platform, decoupling the **Policy of Sovereignty** (who owns the data) from the **Execution of Cryptography** (how data is encrypted).

While the distributed **Crypto (Krypton)** nodes handle the physical wrapping and unwrapping of keys, the CMK Service manages the metadata, identities, and permissions that authorize those operations. It creates a unified **Single Pane of Glass** for managing data sovereignty across multi-cloud and hybrid environments.

## Strategic Value Proposition
The CMK Service is designed to solve the "Governance Gap" in modern cloud usage, where organizations lose visibility and control over their encryption roots.

1.  **Absolute Sovereignty (The Kill Switch):** The CMK manages the secure pointer to the customer's external **Hardware Root of Trust (L1)**. By deleting this pointer in the CMK, an administrator can instantaneously revoke data access globally, enforcing a "Digital Lockout" independent of the underlying cloud provider.
2.  **Unified Governance:** It replaces fragmented key management with a single, abstract governance layer. It ensures that the **Identity** of a tenant and their **Permission** to exist are defined once and synchronized to all connected regions via **Orbital**.
3.  **Auditability:** It provides a tamper-evident audit log of every governance decision—onboarding tenants, linking keys, or modifying access policies—generating the artifacts required for SOC2, TISAX, and GDPR compliance.

## Architecture: The "Brain" vs. The "Muscle"
OpenKCM enforces a strict separation of duties to minimize risk.

* **The CMK Service (The Brain):**
  * **Responsibility:** Governance, Identity, Access Policy, Audit.
  * **Data Access:** **Zero Access.** The CMK Service never sees, holds, or processes the L2/L3 keys or the L4 data keys. It only manages the *metadata* of trust.
  * **Storage:** Relational Database (PostgreSQL) storing Tenant Contexts, L1 ARNs, and Policy Documents.

* **The Crypto Service (The Muscle):**
  * **Responsibility:** Encryption, Decryption, Key Generation.
  * **Data Access:** High Access. It holds the cryptographic material in secure memory to perform operations authorized by the CMK.

## Core Capabilities

### 1. Global Tenant Registry ("The Census")
The CMK Service maintains the authoritative list of **Tenant Contexts**.
* **Function:** It defines which tenants exist, which services they are subscribed to, and which geographic regions they are allowed to operate in.
* **Security:** If a workload attempts to request a key for a tenant not listed in the CMK Registry, the request is rejected at the edge.

### 2. Sovereign Linking Engine (BYOK/HYOK)
The CMK Service orchestrates the lifecycle of the **Sovereign Link**.
* **L1 Pointer:** It stores the reference (ARN/URL) to the customer’s external L1 Key (AWS KMS, Azure Vault, or HSM).
* **Validation:** It periodically probes the external L1 key to ensure it is active and accessible, updating the "Sovereignty Health" status on the dashboard.
* **Revocation:** It broadcasts `UNLINK` commands to the **Orbital** event mesh when a customer initiates a revocation, ensuring rapid propagation to all execution nodes.

### 3. Native RBAC & Access Control
The CMK Service implements a granular Role-Based Access Control (RBAC) system specifically designed for **Sovereign Key Management**.
* **L1-Scoped Permissions:** Access rights are defined in relation to specific **L1 Keys** or **Tenant Contexts**. This ensures that an administrator for the "Finance" tenant cannot view or modify the L1 links of the "HR" tenant.
* **Group Management:** Users are organized into functional groups (e.g., `EU_Key_Admins`, `Global_Auditors`). Permissions to Link, Unlink, or Rotate specific L1 keys are assigned to these groups, preventing privilege escalation across sovereign boundaries.
* **Isolation of Duty:** The system enforces strict isolation, ensuring that the ability to manage the L1 Lifecycle (Linking/Unlinking) can be separated from the ability to manage platform configuration.

### 4. Internal Workflow Management (Multi-Party Authorization)
To prevent insider threats, the CMK Service includes a workflow engine for administrative actions.
* **Four-Eyes Principle:** Critical actions (like Unlinking an L1 Key) can be configured to require approval from two distinct administrators.
* **Decision Lifecycle:** The CMK tracks the state of every administrative request from `DRAFT` → `PENDING_APPROVAL` → `ACTIVE`.

## Integration Interfaces
The CMK Service exposes strictly versioned interfaces for platform integration:
* **Management API (gRPC/REST):** For the Sovereign Portal and CLI tools.
* **Identity Provider (OIDC):** For federating corporate identities (Okta, AD) into platform roles.
* **Orbital Uplink:** For publishing state changes to the regional Crypto nodes.

## Summary
**ACD-101** defines the **OpenKCM CMK Service** as the governance anchor of the platform. By centralizing the "Who, What, and Where" of encryption while delegating the "How," it allows organizations to scale their cloud operations without ever losing their grip on the "Keys to the Kingdom."