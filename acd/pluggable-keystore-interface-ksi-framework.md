# ACD-401: Pluggable Keystore Interface (KSI) Framework

| Status | Date | Document Type |
| :--- | :--- | :--- |
| **Active** | 2026-01-16 | Architecture Concept Design |

## Overview
The **Pluggable Keystore Interface (KSI)** is the infrastructure abstraction layer of the OpenKCM ecosystem. It serves as the "Universal Translator" that decouples the high-level cryptographic operations of the Crypto Core (L2/L3 management) from the low-level API specificities of physical storage providers.

In a multi-cloud or hybrid world, a sovereign platform cannot be tightly coupled to a single vendor's API (e.g., exclusively using AWS KMS APIs). The KSI ensures that OpenKCM can operate identically whether it is deployed on AWS, Azure, Google Cloud, or an air-gapped bare-metal cluster backed by Hardware Security Modules (HSMs).

## The Driver Architecture
The KSI operates on a strictly defined **Driver Model**. The OpenKCM Crypto Core interacts exclusively with the generic KSI Interface, while interchangeable drivers handle the translation to the backend provider.



### The Interface Contract
The KSI defines a Go interface that every driver must implement. This contract guarantees uniform behavior for all cryptographic storage operations:
* `Put(keyID, blob, metadata)`: Persist an encrypted object.
* `Get(keyID)`: Retrieve an encrypted object and its metadata.
* `Delete(keyID)`: Permanently remove an object (Crypto-Shredding).
* `Health()`: Verify connectivity and permissions with the backend.

### Identity Federation (Credential-less Access)
To maintain the **Zero-Trust** security posture, the KSI framework avoids storing long-lived credentials (like AWS Access Keys or Service Principals) within the application. Instead, it relies on **Workload Identity Federation**:
1.  **OIDC Exchange:** The OpenKCM node presents its signed Kubernetes Service Account Token (JWT) to the external provider (e.g., AWS IAM or Azure AD).
2.  **Ephemeral Access:** The provider validates the OIDC signature and issues a temporary, scoped access token valid for the duration of the operation or session.

## Supported Provider Tiers
OpenKCM categorizes drivers into tiers based on their capabilities and support level.

### Tier 1: Cloud-Native Roots (Managed)
*Drivers that interface with hyperscale cloud KMS providers. Used primarily for wrapping the MasterKey or storing L2 Sovereign Keys.*
* **AWS KMS:** Uses the AWS SDK v2; maps OpenKCM Tenants to KMS Tags or Aliases.
* **Azure Key Vault:** Uses the Azure SDK for Go; maps Tenants to distinct Key Vaults or Key Tags.
* **Google Cloud KMS:** Uses the GCP KMS Client; supports Cloud EKM integration.

### Tier 2: Enterprise & Hybrid (Sovereign)
*Drivers for on-premise, air-gapped, or highly regulated deployments.*
* **HashiCorp Vault:** Interfaces with the Transit and KV engines. Supports Namespacing for tenant isolation.
* **Thales / Fortanix (KMIP/PKCS#11):** Generic drivers for connecting to legacy hardware HSMs or dedicated crypto appliances.

### Tier 3: Development & Testing
* **Encrypted SQL:** Stores key blobs in PostgreSQL (protected by a MasterKey). Used for metadata-heavy environments or where external KMS costs are prohibitive.
* **In-Memory:** Ephemeral storage for unit testing and CI/CD pipelines.

## Security & Isolation
The KSI acts as the final gatekeeper before data hits the disk.

* **Encryption at Rest:** The KSI mandates that *no* key material (L2 or L3) is ever sent to the driver in plaintext. All payloads passed to `Put()` must already be wrapped by the parent key (MasterKey or L1).
* **Metadata Binding:** The driver is responsible for persisting the **Authenticated Additional Data (AAD)** alongside the ciphertext. For example, in AWS, the `Tenant_ID` is written as an encryption context tag.

## Summary
This document defines how OpenKCM achieves **Infrastructure Independence**. By abstracting the storage layer into a Pluggable Keystore Interface, the platform ensures that:
1.  **Portability is Absolute:** A tenant's key hierarchy can be migrated from AWS to Azure to On-Prem simply by changing the KSI driver configuration.
2.  **Lock-in is Eliminated:** The application logic never depends on proprietary vendor APIs.
3.  **Security is Uniform:** The enforcement of strict interface contracts ensures that a security policy defined in the Governance layer is respected regardless of the underlying hardware.