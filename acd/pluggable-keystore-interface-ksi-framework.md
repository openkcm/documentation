# ACD-401: OpenKCM Plugin Architecture & Abstraction Layers

| Status | Date | Document Type |
| :--- | :--- | :--- |
| **Active** | 2026-02-02 | Architecture Concept Design |

## Overview
The **OpenKCM Plugin Architecture** is the infrastructure abstraction layer of the ecosystem. It serves as the "Universal Translator" that decouples the high-level cryptographic operations of the **Governance Control Plane (CMK)** and **Execution Plane (Crypto/Gateway)** from the low-level API specificities of physical providers.

This framework ensures that OpenKCM can operate identically whether deployed on AWS, Azure, Google Cloud, or an air-gapped bare-metal cluster backed by Hardware Security Modules (HSMs).

## The Plugin Ecosystem
As defined in the architecture, the plugin system is composed of four distinct pillars that exist within the **CMK API Server** and the **Krypton** components.

| Plugin Type | Responsibility | Implementation Context |
| :--- | :--- | :--- |
| **Identity** | Authenticating the OpenKCM node to the environment (e.g., OIDC, IAM). | CMK, Krypton |
| **Keystore (KSI)** | Persisting encrypted key blobs (L2/L3) and L1 Pointers. | CMK, Krypton |
| **SystemInfo** | Reporting health, versioning, and capabilities. | CMK, Krypton, Gateway |
| **Notify** | Broadcasting governance events (Link/Unlink) to the Orbital mesh. | CMK |

## The Driver Architecture by Component

The specific role of the plugins changes depending on which component they are serving (The Brain, The Muscle, or The Edge).

### 1. CMK API Server (The Brain)
The CMK uses plugins to manage the **Sovereign Link** to the customer's external root.

* **Role:** Manage L1 Key Pointers (Shadow References).
* **Primary Plugin:** **Identity & Keystore**.
* **Target Backend:** **External Physical Vaults (L1 Root of Trust)**.
    * *Cloud Providers:* AWS KMS, Azure Key Vault, GCP KMS.
    * *HSM Providers:* Thales HSM, HashiCorp Vault.
* **Function:** The CMK does not hold keys; it uses the plugin to validating the existence and accessibility of the external L1 key.

### 2. Krypton Core (The Muscle)
The Regional Core uses plugins to manage the internal **L2 and L3 Key Hierarchy**.

* **Role:** Key Material Storage (L2/L3) & Crypto Operations (Wrap/Unwrap).
* **Primary Plugin:** **Key Material Storage / Keystore**.
* **Target Backend:**
    * *Internal Vaults:* Encrypted Storage (e.g., PostgreSQL, internal secure storage).
    * *External Unseal:* Connects to L1 Vaults (AWS/Azure) strictly for the **L2 Unsealing** operation.
* **Function:** Handles the encryption and decryption of the intermediate keys (L2/L3) that form the chain of trust.

### 3. Krypton Gateway (The Edge)
The Gateway uses plugins to manage the high-velocity **L4 Data Encryption Keys**.

* **Role:** L4 Key Store (Creation & Retrieval).
* **Primary Plugin:** **Key Material Storage**.
* **Target Backend:** **External Physical Vaults (Key Material Storages)**.
    * *Examples:* **OpenBao**, **HashiCorp Vault**, Redis, or Encrypted SQL.
* **Function:** The Gateway generates L4 keys locally and persists them (wrapped by L3) into the local pluggable vault. It never speaks to the L1 Root directly.

## Supported Provider Tiers

### Tier 1: Cloud-Native Roots (L1 / MasterKey)
*Drivers that interface with hyperscale cloud KMS providers. Used by CMK and Krypton Core for the Sovereign Link.*
* **AWS KMS:** Maps OpenKCM Tenants to KMS Keys via ARN.
* **Azure Key Vault:** Maps Tenants to Key Vault keys.
* **Google Cloud KMS:** Supports Cloud EKM integration.

### Tier 2: Enterprise & Hybrid (HSM / On-Prem)
*Drivers for air-gapped or highly regulated deployments.*
* **HashiCorp Vault:** Interfaces with Transit and KV engines.
* **Thales / Fortanix (KMIP/PKCS#11):** Connects to legacy hardware HSMs.
* **OpenBao:** Supported for CYOK/MYOK scenarios.

### Tier 3: Operational Storage (L4 / Cache)
*High-performance drivers used by the **Crypto Gateway**.*
* **OpenBao / HashiCorp Vault:** Secure persistence for L4 blobs.
* **Redis / Encrypted SQL:** Alternative backends for high-speed L4 retrieval.

## Security & Isolation
The Plugin Architecture acts as the final gatekeeper before data hits the wire or disk.

* **Encryption at Rest:** The framework enforces that **no key material** (L2, L3, or L4) is ever sent to a `Keystore` driver in plaintext. All payloads passed to `Put()` must already be wrapped.
* **Identity Federation:** To maintain **Zero-Trust**, the Identity plugins utilize workload identity (e.g., Kubernetes Service Account tokens) to authenticate with external vaults (AWS/Azure) without storing static credentials.

## Summary
ACD-401 defines how OpenKCM achieves **Infrastructure Independence**. By abstracting the storage and identity layers through the `Identity`, `Keystore`, `SystemInfo`, and `Notify` plugins, the platform ensures that:
1.  **Portability:** The platform can run on any cloud or on-prem hardware.
2.  **Sovereignty:** L1 keys remain in the customer's external vault, accessed only via the abstract plugin interface.
3.  **Flexibility:** The Gateway can swap its storage backend (e.g., from OpenBao to Redis) without changing the application code.