# OpenKCM Architecture Concept Design (ACD) Index

This index defines the high-level conceptual frameworks and strategic paradigms that govern the OpenKCM ecosystem. These documents articulate the "Value Engine" philosophy, explaining how OpenKCM solves the "Trust Paradox" by structurally decoupling Governance (Intent) from Execution (Action).

### I. Strategic Core & Governance Control Plane
* [**ACD-101: OpenKCM Strategic Vision & Cryptographic Governance Model**](openkcm-strategic-vision.md)
  * **Concept**: Defines OpenKCM as a "Value Engine" that separates Global Policy (CMK) from Regional Execution (Crypto). It establishes the business case for "Liability Shift" and the technical foundation for solving the Trust Paradox.
* [**ACD-102: OpenKCM CMK & Sovereign Portal**](cmk-service-portal.md)
  * **Concept**: The "Human Head" of the platform. Covers the Sovereign Portal, the administrative workflows for BYOK/HYOK, and the "Sovereign Kill-Switch" that allows customers to revoke L1 keys and instantly lock global data.
* [**ACD-103: CMK Registry & Global Tenant Governance**](cmk-registry-and-tenant-metadata-governance.md)
  * **Concept**: The "Ledger of Trust" and Source of Truth. It manages the immutable intent of the system, mapping Tenants to L1 References and defining the "Desired State" for the Orbital engine.
* [**ACD-104: Identity & Zero-Trust Authorization**](identity-and-zero-trust-authorization.md)
  * **Concept**: The security mesh integrating mTLS service identities and JWT-based human sessions. It defines how Cedar policies enforce strict isolation between the Governance and Execution planes.
* [**ACD-105: Sovereign Audit & Immutable Logging**](sovereign-audit-and-immutable-logging.md)
  * **Concept**: The architecture for high-volume, tamper-proof logging. Defines how the platform generates signed "Proofs of Access" for every cryptographic operation and separates the Log Plane from the Control Plane to satisfy strict compliance (SOC2, WORM).

### II. The Execution Plane (Regional & Gateway)
* [**ACD-201: Orbital – State Synchronization & Task Reconciliation**](orbital–state-synchronization-and-task-reconciliation.md)
  * **Concept**: The distributed nervous system bridging the CMK (Intent) and Crypto (Action). It utilizes an asynchronous event bus to broadcast priority revocation events and reconcile configuration drift across the global mesh.
* [**ACD-202: OpenKCM Crypto Core – Regional Governance & KMIP Unsealing**](crypto-core-regional-governance-and-unsealing.md)
  * **Concept**: The regional authority responsible for the "Recursive Unsealing" of the L1 $\rightarrow$ L2 $\rightarrow$ L3 chain. It acts as the stateless execution engine for L4 Wrap/Unwrap operations without persisting data keys.
* [**ACD-203: OpenKCM Crypto Gateway – High-Performance Ephemeral Data Plane**](crypto-gateway–high-performance-ephemeral-data-plane.md)
  * **Concept**: The forward-deployed interface located at the workload gateway. It manages ephemeral L4 Data Encryption Keys (DEKs) to deliver sub-millisecond cryptographic performance while maintaining strict isolation.

### III. Cryptographic Mechanics & Standards
* [**ACD-301: The Layered Key Hierarchy (L1 → L4)**](layered-key-hierarchy-(L1→L4).md)
  * **Concept**: The formal definition of "Mathematical Sovereignty." Details the dependency chain: L1 (External Root) $\rightarrow$ L2 (Tenant) $\rightarrow$ L3 (Service) $\rightarrow$ L4 (Data Record).
* [**ACD-302: Root of Trust & MasterKey Bootstrap**](root-of-trust-and-masterkey-bootstrap.md)
  * **Concept**: The bootstrapping protocol for the internal Crypto Core, utilizing Shamir Secret Sharing (SSS) or Seal Auto-Unseal mechanisms to establish the initial secure memory space.
* [**ACD-303: KMIP Integration & Crypto Data Plane**](kmip-integration-and-crypto-data-plane.md)
  * **Concept**: The interface standard. How OpenKCM utilizes the OASIS KMIP protocol over mTLS to standardize cryptographic requests from SaaS applications.
* [**ACD-304: Key Rotation Strategies & Lifecycle Management**](key-rotation-strategies-and-lifecycle-management.md)
  * **Concept**: The automated logic for time-based and event-based rotation. Defines "Lazy Re-encryption" vs. "Deep Re-keying" strategies to ensure security hygiene without forcing application downtime.

### IV. Ecosystem Integration & Resilience
* [**ACD-401: Pluggable Keystore Interface (KSI) Framework**](pluggable-keystore-interface-ksi-framework.md)
  * **Concept**: The abstraction layer allowing OpenKCM to support heterogeneous external providers (AWS KMS, Azure Key Vault, GCP KMS, Thales, Fortanix) through a unified driver model.
* [**ACD-402: Global Resilience & Sovereign Reconciliation**](global-resilience-and-sovereign-reconciliation.md)
  * **Concept**: The strategy for cross-region disaster recovery, ensuring that Orbital can re-hydrate regional node state from the central Registry without compromising cryptographic boundaries.
* **ACD-403: Application Integration Patterns & SDKs**
  * **Concept**: The developer experience manual. Defines the standard patterns for integrating SaaS workloads with OpenKCM, including the use of Client SDKs, Sidecars, and local caching strategies for L4 keys.