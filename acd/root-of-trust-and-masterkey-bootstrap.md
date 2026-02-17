# ACD-302: Root of Trust & MasterKey Bootstrap

| Status | Date | Document Type |
| :--- | :--- | :--- |
| **Active** | 2026-02-02 | Architecture Concept Design |

## Overview
The **Root of Trust** is the foundational security pillar of the OpenKCM ecosystem. It defines how the platform establishes its identity, maintains multi-tenant isolation, and bootstraps its internal cryptographic engines across multi-cloud environments.

OpenKCM is designed to eliminate **"Blind Trust"** in the service provider. By utilizing a strictly hierarchical key model and pluggable unsealing mechanisms, the platform ensures that the ultimate root of authority always rests with the customer (via L1 keys) or a distributed set of stakeholders (via Shamir Secret Sharing).

## Layered Key Hierarchy Overview (L1 → L4)
OpenKCM employs a recursive envelope encryption model where each layer cryptographically binds the layer beneath it. This ensures that revoking access at a higher level instantly and mathematically invalidates all downstream data access.

### Layer Responsibilities & Storage

| Layer | Scope | Purpose | Owner | Storage Location |
| :--- | :--- | :--- | :--- | :--- |
| **L1** | Global | **Customer Master Key (CMK)**: The ultimate kill-switch. Never leaves the customer's external hardware. | **Customer** | External KMS/HSM (AWS, Azure, Thales) |
| **L2** | Tenant | **Tenant Root**: Provides mathematical siloing between customers. Derived only after L1 handshake. | **Core** | Regional Core Vault (Encrypted) / Core RAM (Plaintext) |
| **L3** | Service | **Service Key (KEK)**: Isolates specific domains (e.g., "Payments", "Logs"). Used to wrap L4s. | **Core** | Regional Core Vault (Encrypted) / Core RAM (Plaintext) |
| **L4** | Workload | **Data Key (DEK)**: Ephemeral keys for high-speed local encryption. | **Gateway** | **Gateway Pluggable Internal Vault** (Ciphertext) |

## MasterKey Management (Platform Bootstrap)
Before the **OpenKCM Crypto (Krypton) Core** can process any tenant data, it must first "Unlock Itself." This process relies on the **MasterKey**—the cryptographic root of the Regional Core itself (distinct from Customer L1 keys).

OpenKCM supports two primary bootstrap modes for the Regional Core:

### 1. Shamir Secret Sharing (SSS) – The "Four-Eyes" Strategy
In high-security, sovereign, or regulated deployments, the MasterKey is never stored in a single location.
* **Mechanism**: The MasterKey is split into **N shards** (e.g., 5); a minimum threshold of **M shards** (e.g., 3) is required to reconstruct the key.
* **Stakeholder Control**: Shards are distributed across different cloud providers, regions, or physical stakeholders to prevent single-point compromise.
* **Recovery**: Requires a manual or orchestrated multi-party ceremony to bring the Regional Core online.

### 2. Auto-Unseal – The "Cloud-Native" Strategy
Designed for high-availability, automated containerized environments where manual intervention is not feasible.
* **Mechanism**: The MasterKey is "sealed" (encrypted) by a trusted Cloud KMS (e.g., AWS KMS) or an on-premise HSM.
* **Automated Flow**: At startup, the Core identifies itself via mTLS/IAM, requests the external KMS to unseal the MasterKey, and boots into memory.
* **Benefit**: Enables zero-touch recovery and seamless horizontal scaling of Core nodes.

## Keystore Integration Archetypes
OpenKCM abstracts complex provider APIs into a unified **Sovereign Link** interface, supporting three primary integration patterns for the L1 Root:

* **BYOK (Bring/Link Your Own Key)**: Customer provides key material (or a reference to it) to be managed within the provider's KMS environment. The Core "Links" to this key for L2 unsealing.
* **HYOK (Hold Your Own Key)**: Key material never leaves the customer's physical HSM; OpenKCM sends "Wrap/Unwrap" requests over the network to the hardware anchor.
* **CYOK (Control Your Own Key)**: Full lifecycle management where the customer maintains the key policy in their native cloud account, and OpenKCM operates as a delegated principal.

## Security Controls & Operational Integrity

### Cryptographic Isolation
* **Mathematical Siloing**: Tenant A’s L2 key is wrapped by an L1 key that Tenant B cannot access. Cross-tenant leakage is physically impossible at the hardware level.
* **Split-Execution Memory**: The **Gateway** never holds L2 or L3 keys. The **Core** never persists L4 keys. This limits the blast radius of any single component compromise.

### Access & Separation of Duties
* **Governance vs. Execution**: CMK Service administrators (who manage policy in the Control Plane) cannot access the Crypto (Krypton) Core memory where unsealed keys reside.
* **Four-Eyes Principle**: Optional SSS enforcement ensures that no single rogue administrator can bootstrap the Regional Core or export the MasterKey.

### Compliance & Audit
* **Non-Repudiable Logs**: Every key generation, rotation, or unwrap event generates a unique Correlation ID.
* **End-to-End Traceability**: Correlation IDs link internal application requests directly to the customer’s own CloudTrail or KMS logs, providing verifiable proof that **only authorized L1 access occurred**.

## Summary
ACD-302 establishes the **Sovereignty Firewall** of the OpenKCM platform. By enforcing a strict distinction between the **Platform Root** (MasterKey) and the **Tenant Root** (L1), the architecture ensures that:

1.  **Sovereignty is Absolute**: The customer’s L1 key remains the ultimate "Kill Switch."
2.  **Platform is Resilient**: The Core can be bootstrapped automatically (Auto-Unseal) or securely (Shamir) depending on the threat model.
3.  **Liability is Minimized**: The provider shifts the risk of root-key compromise back to the hardware-backed customer account.