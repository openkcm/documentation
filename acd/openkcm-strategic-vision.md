# ACD-101: OpenKCM Strategic Vision & Cryptographic Governance Model

| Status | Date | Document Type |
| :--- | :--- | :--- |
| **Active** | 2026-01-31 | Architecture Concept Design |

## Overview
**OpenKCM** is the **Value Engine** that enables platforms to win bigger deals, capture higher margins, and operate with zero-risk liability in a world that demands absolute data ownership.

In the modern sovereign-cloud era, traditional security is a cost center. OpenKCM transforms security into a **market differentiator** by solving the "Trust Paradox"—the conflict between providing total customer data ownership and the need for high-scale, cloud-native performance.

## Structural Pillar: The Governance & Execution Framework
To achieve absolute sovereignty without performance degradation, OpenKCM enforces a strict separation of duties between the logic of access (**Governance**) and the act of encryption (**Execution**).

### OpenKCM CMK: The Governance Control Plane
The **CMK** is the "Brain" of the ecosystem. It manages the **Intent** of security without ever possessing the material to execute it.
* **Sovereign Anchor:** Manages **Shadow References** to **L1 Root Keys** held exclusively in customer-owned environments (AWS KMS, Azure Key Vault, GCP KMS, or Private HSMs).
* **Security Barrier:** The CMK handles metadata, policy (Cedar), and encrypted unseal requests but is physically and logically barred from touching plaintext key material.
* **Active-Standby State:** Writes all policy and configuration to the **Active** node of the Schema-per-Tenant Postgres cluster, ensuring high availability and strict consistency.

### OpenKCM Crypto: The Execution Plane
The **Crypto** layer is the "Muscle." It consists of regional and gateway nodes designed for zero-latency cryptographic operations.
* **OpenKCM Crypto (Krypton):** The regional authority that manages the lifecycle of **L2 (Tenant/Account)** and **L3 (App/Service)** keys. It performs the "Recursive Unsealing" required to activate a tenant's environment and executes stateless **L4 Wrap/Unwrap** operations via KMIP.
* **OpenKCM Crypto (Krypton) Gateway:** Distributed, high-speed interfaces that handle the "Hot Path" of **L4 DEK** generation and management via KMIP. The Gateway operates close to the tenant workloads, ensuring sub-millisecond encryption.

## The Sovereignty Engine: Recursive Unsealing & Shadow Keys
OpenKCM enforces security through a tiered key hierarchy where each layer is mathematically bound to the one above it, and where the provider holds only **pointers**, not secrets.

1.  **L1 (External)**: Customer-owned Root of Trust (Physical Key). OpenKCM holds only a **Shadow Reference** (ARN).
2.  **L2 (Tenant)**: Intermediate key providing mathematical isolation. Stored as an encrypted blob in a external vault, unwrappable *only* by the L1.
3.  **L3 (Service)**: Provides isolation between specific application domains.
4.  **L4 (DEK)**: Ephemeral Data Encryption Key used for per-record encryption, generated at the Gateway and often managed by the client.


![high-level-architecture.png](.images/high-level-architecture.png)

## Strategic Advantages

| Advantage | Business Meaning | Technical Enabler |
| :--- | :--- | :--- |
| **Market Expansion** | Opens Gov, Finance, Defense, Healthcare, etc. | **L1 Root Control** (BYOK/HYOK) |
| **Liability Shift** | No plaintext root keys → breach firewall | **Shadow Key Pattern** (Zero-Knowledge DB) |
| **Pricing Leverage** | Revocation as Platinum tier | **Sovereign Kill-Switch** |
| **Competitive Moat** | True cryptographic isolation | **Schema-per-Tenant** Architecture |
| **Compliance Speed** | End-to-end traceability | **Sovereign Audit**  |

## CMK Layer: Governance Capabilities
The CMK Layer is the **trust firewall** that allows enterprises to adopt SaaS with the same confidence as on-premise solutions.

* **Self-Service Onboarding**: Enterprises import keys or create them in their own cloud accounts without manual provider involvement.
* **Tenant-to-Key Mapping**: Securely binds tenant IDs to customer-specific KMS references (ARNs, URIs) within dedicated **Tenant Schemas**.
* **OpenKCM Plugin Architecture**: Unified interface for **Identity** (OIDC/SPIFFE), **Keystore** (AWS/Azure/Vault), **SystemInfo**, and **Notification** plugins.
* **Revocation Kill-Switch**: Customer-initiated revocation instantly renders all tenant data inaccessible globally.

## The Crypto Layer: Operational Excellence
The Crypto Layer handles the "operational reality" of security—ensuring that billions of operations per day do not degrade the user experience.

### OpenKCM Crypto (Krypton)
* **MasterKey Protection**: Reconstructs the internal MasterKey via Shamir Secret Sharing (SSS) or Seal auto-unseal into secure memory only.
* **Lifecycle Automation**: Handles rotation, versioning, and archiving of L2/L3 keys.
* **Root of Trust Bootstrap**: Initializes via the **Keystore Plugin**, loading the necessary credentials to unseal the L2 keys from the Shadow References.
* **High-Performance KMIP**: Performs high-speed Wrap/Unwrap operations for **L4 DEKs** against the **L3 KEK**.

### OpenKCM Crypto (Krypton) Gateway
* **Dedicated Gateway Service**: Deployed as a standalone service close to workloads, managing high-throughput operations.
* **Local Persistence & Caching**: Independently handles `Create` and `Get` operations for **L4 DEKs**, securely storing them in a local gateway vault to ensure resilience and performance.
* **KMIP/mTLS**: All communication uses standardized KMIP over mandatory mutual TLS.

## Strategic Outcome
**How Governance (CMK) & Operations (Crypto) Drive Enterprise Value**

| Strategic Dimension | CMK Service (Governance) | Crypto Service (Operations)                         | Combined Impact |
| :--- | :--- |:----------------------------------------------------| :--- |
| **Market Access** | Unlocks regulated verticals via root-key control. | Enables high-throughput for real-time platforms.    | Opens high-ACV deals previously blocked by sovereignty concerns. |
| **Trust & Retention** | Mathematically enforced revocation. | Zero-trust multi-tenancy (no cross-tenant leakage). | Reduces churn in regulated segments; customers stay longer. |


## Summary
This document establishes OpenKCM as the **Strategic Growth Catalyst** for the modern sovereign-cloud era. By transforming security from a passive compliance cost into an active market differentiator, it empowers SaaS providers to:

* **Secure High-Value Enterprise Contracts** by satisfying the most stringent global data sovereignty and residency requirements.
* **Enhance Tiered Monetization** by positioning advanced cryptographic isolation and root-key control as premium, high-margin capabilities.
* **Minimize Operational Liability** by ensuring the provider never possesses plaintext root-key material (via the **Shadow Key Pattern**), shifting ultimate access control to the customer.
* **Eliminate Performance Bottlenecks** through a decentralized, gateway-native cryptographic model designed for sub-millisecond global execution.