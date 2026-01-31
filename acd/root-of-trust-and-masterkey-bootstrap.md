# ACD-302: Root of Trust & MasterKey Bootstrap

| Status | Date | Document Type |
| :--- | :--- | :--- |
| **Active** | 2026-01-13 | Architecture Concept Design |

## Overview
The **Root of Trust** is the foundational security pillar of the OpenKCM ecosystem. It defines how the platform establishes its identity, maintains multi-tenant isolation, and bootstraps its internal cryptographic engines across multi-cloud environments.

OpenKCM is designed to eliminate **"Blind Trust"** in the service provider. By utilizing a strictly hierarchical key model and pluggable unsealing mechanisms, the platform ensures that the ultimate root of authority always rests with the customer (via L1 keys) or a distributed set of stakeholders (via Shamir Secret Sharing).

## Layered Key Hierarchy Overview (L1 → L4)
OpenKCM employs a recursive envelope encryption model where each layer cryptographically binds the layer beneath it. This ensures that revoking access at a higher level instantly and mathematically invalidates all downstream data access.

### Layer Responsibilities & Storage

| Layer | Scope | Purpose | Storage / Keystore Provider |
| :--- | :--- | :--- | :--- |
| **L1** | Global / Root | **Customer Master Key (CMK)**: The ultimate root of trust. Customer-owned and managed. | External KMS/HSM (AWS, Azure, GCP, Thales, Fortanix) |
| **L2** | Tenant | **Tenant Encryption Key**: Provides mathematical siloing and isolation between unique customers. | Internal Secure Vault / OpenBao |
| **L2.1** | Tenant Version | **Versioned Tenant Keys**: Facilitates seamless key rotation, rollback, and per-resource versioning. | Internal Registry (Encrypted) |
| **L3** | Service | **Service Key**: Isolates specific application domains (e.g., "Payments", "PII-Vault", "Logs"). | Internal Secure Storage |
| **L4** | Workload | **Data Encryption Key (DEK)**: Ephemeral, short-lived keys used for per-record encryption/decryption. | Crypto (Krypton) Gateway Memory (KMIP) |



## MasterKey Management & System Bootstrapping
To reach an operational state, the **OpenKCM Crypto (Krypton)** must reconstruct its internal **MasterKey** to decrypt the system registry and access tenant keys. OpenKCM supports two primary bootstrap modes based on security requirements.

### Shamir Secret Sharing (SSS) – The "Four-Eyes" Strategy
In high-security, sovereign, or regulated deployments, the MasterKey is never stored in a single location.
* **Mechanism**: The MasterKey is split into **N shards**; a minimum threshold of **M shards** is required to reconstruct the key.
* **Stakeholder Control**: Shards are distributed across different cloud providers, regions, or physical stakeholders to prevent single-point compromise.
* **Recovery**: Requires a manual or orchestrated multi-party ceremony to bring the system online.

### Seal Auto-Unseal – The "Cloud-Native" Strategy
Designed for high-availability, automated containerized environments.
* **Mechanism**: The MasterKey is "sealed" (encrypted) by a trusted external KMS or an on-premise HSM.
* **Automated Flow**: At startup, the Crypto Service identifies itself via mTLS and IAM, requesting the KMS to unseal the MasterKey directly into secure memory.
* **Benefit**: Enables zero-touch recovery and seamless horizontal scaling.

## Keystore Integration Archetypes
OpenKCM abstracts complex provider APIs into a unified interface, supporting three primary integration patterns:

* **BYOK (Bring Your Own Key)**: Customer provides key material to be managed within the provider's KMS environment.
* **HYOK (Hold Your Own Key)**: Key material never leaves the customer's physical HSM; OpenKCM only sends "Wrap/Unwrap" requests to the hardware.
* **CYOK/MYOK (Control/Manage Your Own Key)**: Full lifecycle management where the customer maintains the key policy in their native cloud account (AWS/GCP/Azure).

## Security Controls & Operational Integrity

### Cryptographic Isolation
* **Mathematical Siloing**: Tenant A’s L2 key is wrapped by an L1 key that Tenant B cannot access. Cross-tenant leakage is physically impossible at the hardware level.
* **Zero-Knowledge Implementation**: OpenKCM Core handles encrypted key references; plaintext material for L1 and L2 is only present in volatile secure memory during active operations.

### Access & Separation of Duties
* **Governance vs. Execution**: CMK Service administrators (who manage policy) cannot access the Crypto Service memory where unsealed keys reside.
* **Four-Eyes Principle**: Optional SSS enforcement ensures that no single rogue administrator can bootstrap the system.

### Compliance & Audit
* **Non-Repudiable Logs**: Every key generation, rotation, or unwrap event generates a unique Correlation ID.
* **End-to-End Traceability**: Correlation IDs link internal application requests directly to the customer’s own CloudTrail or KMS logs, providing verifiable proof of access.

## Summary
This document establishes the **Sovereignty Firewall** of the OpenKCM platform. By enforcing a strict hierarchical model and supporting robust unsealing strategies, the platform ensures that:

1.  **Sovereignty is Absolute**: The customer’s L1 key remains the ultimate "Kill Switch."
2.  **Performance is Uncompromised**: The hierarchy allows for gateway-native L4 operations with sub-millisecond latency.
3.  **Liability is Minimized**: The provider shifts the risk of root-key compromise back to the hardware-backed customer account.

OpenKCM turns the **Root of Trust** into a **Strategic Value Engine**, allowing SaaS providers to win the most regulated enterprise deals by offering mathematical proof of data ownership.