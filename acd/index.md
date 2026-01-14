# OpenKCM Architecture Concept Design (ACD) Index

This index defines the high-level conceptual frameworks and strategic paradigms that govern the OpenKCM ecosystem. These documents describe the fundamental "why" and "how" of the platform's core philosophies and component interactions.

### I. CMK Governance & Sovereign Management
* [**ACD-101: OpenKCM Platform Vision & Dual Service Model**](openkcm-strategic-vision.md)
  * **Concept**: The strategic decoupling of Customer Governance (CMK) from Infrastructure Execution (Crypto) to solve the "Trust Paradox" in SaaS.
* [**ACD-102: CMK Service Portal**](cmk-service-portal.md)
  * **Concept**: The interface for managing L1 External Keys and Customer Revocation.
* [**ACD-103: CMK Registry & Tenant Metadata Governance**](cmk-registry-and-tenant-metadata-governance.md)
  * **Concept**: The design of the global "Source of Truth" for tenant onboarding, L1 key linking, and system-wide resource state.
* [**ACD-104: Identity & Zero-Trust Authorization**](identity-and-zero-trust-authorization.md)
  * **Concept**: Integrating mTLS service identities and JWT-based human sessions with Cedar policies to enforce administrative and cryptographic silos.

### II. Regional Execution (The Crypto Layer)
* [**ACD-201: Orbital – State Synchronization & Task Reconciliation**](orbital–state-synchronization-and-task-reconciliation.md)
  * **Concept**: The distributed "nervous system" that bridges central governance and regional execution. It ensures that "Intent" defined in the CMK Registry is reliably translated into "Actual State" across global nodes using an asynchronous, idempotent task model via RabbitMQ.
* [**ACD-202: Crypto Core Regional Governance & Unsealing**](crypto-core-regional-governance-and-unsealing.md)
  * **Concept**: The regional authority responsible for the L1 → L2 → L3 unsealing sequence. It acts as the regional KMIP engine for L4 wrap/unwrap operations, manages long-term key lifecycles, and ensures regional nodes are properly provisioned.
* [**ACD-203: Crypto Edge – High-Performance Ephemeral Data Plane**](crypto-edge–high-performance-ephemeral-data-plane.md)
  * **Concept**: The high-speed, KMIP-compliant interface located at the workload edge. It dynamically manages ephemeral L4 Data Encryption Keys (DEKs) to provide sub-millisecond cryptographic operations while maintaining strict mTLS-based isolation for application services.

### III. Cryptographic Domain Concepts
* [**ACD-301: The Layered Key Hierarchy (L1 → L4)**](layered-key-hierarchy-(L1→L4).md)
  * **Concept**: The formal recursive unsealing sequence: L1 (External) $\rightarrow$ L2 (Tenant) $\rightarrow$ L3 (Service) $\rightarrow$ L4 (Data).
* [**ACD-302: Root of Trust & MasterKey Bootstrap**](root-of-trust-and-masterkey-bootstrap.md)
  * **Concept**: Conceptualizing the transition from a "Sealed" to "Unsealed" state via Shamir Secret Sharing (SSS) or Seal Auto-Unseal.
* [**ACD-303: KMIP Integration & Crypto Data Plane**](kmip-integration-and-crypto-data-plane.md)
  * **Concept**: Adhering to the OASIS KMIP standard to provide sub-millisecond encryption/decryption services at the Crypto and Crypto Edge.

### IV. Integration & Resilience
* **ACD-401: Pluggable Keystore Interface (KSI) Framework**
  * **Concept**: A driver-based approach for supporting heterogeneous external root-of-trust providers including AWS, GCP, Azure, and physical HSMs.
* **ACD-402: Global Resilience & Sovereign Reconciliation**
  * **Concept**: The high-level strategy for cross-region recovery, ensuring Orbital can re-hydrate regional state during disaster recovery events.