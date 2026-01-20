# OpenKCM Architecture Design Record (ADR) Index

**OpenKCM (Open Key Chain Management)** is a dual-service architecture designed to transform data security from a compliance cost into a **primary revenue driver**. By decoupling **Customer Governance (CMK Service)** from **Infrastructure Operations (Crypto Service)**, the platform solves the "Trust Paradox": giving customers absolute sovereignty over their data while maintaining high-performance global operations.

### I. High-Level Strategy & Governance
*Decisions regarding the fundamental separation of powers and the "Trust Paradox."*

* [**ADR-101: The "Church & State" Separation of Governance and Execution**](separation-of-governance-cmk-and-execution-crypto.md)
  * **Context:** We decouple the **CMK Service** (Policy/Intent) from the **Crypto Service** (Execution).
  * **Decision:** Strict physical and logical isolation. The Control Plane (CMK) never possesses key material; the Execution Plane (Crypto) never stores persistent policy.
* [**ADR-102: Tiered Envelope Encryption (L1–L4)**](tiered-envelope-encryption-model.md)
  * **Context:** Balancing customer control with massive scale.
  * **Decision:** Adopt a 4-layer recursive wrap structure. L1 (Customer Root) wraps L2 (Tenant), which wraps L3 (Service), which wraps L4 (Data).
* [**ADR-103: Multi-Tenant Cryptographic Isolation**](multi-tenant-cryptographic-isolation-standard.md)
  * **Context:** Preventing cross-tenant data leakage in a shared SaaS environment.
  * **Decision:** Enforce "Hard" cryptographic headers. Every encrypted object must carry a versioned header identifying its Tenant ID, preventing accidental cross-decryption.

### II. Root of Trust & MasterKey Bootstrap
*Decisions regarding the startup sequence and the ultimate root of the internal security chain.*

* [**ADR-201: MasterKey Immutability & Volatile Memory**](masterkey-definition-and-immutability-standard.md)
  * **Context:** How to handle the internal "Key of Keys."
  * **Decision:** The MasterKey is never written to disk. It exists only in `mlock` protected RAM and must be reconstructed at every process restart.
* [**ADR-202: Shamir Secret Sharing (SSS) for Human Unsealing**](masterkey-reconstruction-via-shamir-secret-sharing.md)
  * **Context:** Emergency recovery when Cloud KMS is unavailable.
  * **Decision:** Split the MasterKey into $N$ shards with a threshold of $M$ required for reconstruction ($M-of-N$ scheme).
* *[*ADR-203: Cloud-Native Auto-Unseal (Seal Wrapping)**](cloud-native-auto-unseal-seal-wrapper.md)
  * **Context:** Enabling automated scaling without manual human intervention.
  * **Decision:** The MasterKey blob is encrypted by a cloud-native Transit Key (AWS KMS / Azure KeyVault) allowing the node to "boot and decrypt" itself automatically.
* [**ADR-204: MasterKey Storage and Shard Encryption Policy**](masterkey-storage-and-shard-encryption-policy.md)
  * *Context:* Policies for encrypting shards using different keystores (AWS, Azure, GCP, HSM).

### III. Internal Key Logic & Rotation (The IVK)
*Decisions regarding the "Internal Versioned Key" (IVK) which bridges the MasterKey to the Tenant Keys.*

* [**ADR-301: The IVK (Internal Versioned Key) Architecture**](ivk-architecture.md)
  * **Context:** Preventing a "God Key" scenario where one MasterKey encrypts billions of objects.
  * **Decision:** Introduce the **IVK** as an intermediate, rotatable key layer. The MasterKey wraps the IVK; the IVK wraps the L2 Tenant Keys.
* [**ADR-302: IVK Mapping Logic to L2 Tenant Keys**](ivk-mapping-logic-to-l2-tenant-keys.md)
  * *Context:* Technical implementation of how IVKs wrap per-tenant L2 keys.
* [**ADR-303: IVK Rotation Triggers & Grace Period Management**](ivk-rotation-triggers-and-grace-period-management.md)
  * *Context:* Logic for periodic rotation to new IVK versions for forward security.
* [**ADR-304: IVK Metadata Persistence & Cache-Policy**](ivk-metadata-persistence-and-cache-policy.md)
  * *Context:* Standards for storing and retrieving versioned IVK encrypted blobs.

### IV: Security, Identity & Authorization
* **ADR-401: Mandatory Mutual TLS mTLS for All Service Traffic**
    * *Context:* Requirement for x509 certificate-based authentication for all internal nodes.
* **ADR-402: Identity Bound Key Authorization Engine**
    * *Context:* **Authorization Core:** Validating that a caller identity (via mTLS certificate) is explicitly permitted to request keys for a specific Tenant ID.
* **ADR-403: Zero Knowledge Storage Principles for Key Metadata**
    * *Context:* Ensuring that persisted metadata is unusable without the reconstructed MasterKey in memory.
* **ADR-404: Four Eyes Principle Enforcement via Quorum Authorization**
    * *Context:* Requiring M distinct human/service approvals for high-risk admin tasks.
* **ADR-405: Secure Bootstrapping and Secret Injection Policy**
    * *Context:* How initial secrets and shards are securely delivered to new nodes during scaling.

### V: CMK Service (Customer Managed Key Layer)
* [**ADR-501: Pluggable KMS Provider Interface**](pluggable-kms-provider-interface.md)
    * *Context:* Driver architecture for AWS KMS, GCP KMS, Azure Vault, and HSMs.
* **ADR-502: L1 Key Reference Storage and Metadata Schema**
    * *Context:* Handling encrypted references without touching plaintext material.
* **ADR-503: Instant Revocation and Kill Switch Mechanics**
    * *Context:* The "Cryptographic Erasure" flow: how L1 deletion invalidates the tenant chain.
* **ADR-504: BYOK and HYOK Onboarding Workflows**
    * *Context:* Procedures for integrating customer-provided keys into the L1 layer.

### VI: Crypto (Core & Edge Dual-Service Model)
* [**ADR-601: Crypto Core Central Orchestration and KEK Lifecycle**](crypto-core-central-orchestration-and-kek-lifecycle.md)
  * *Context:* Defines the Core service role in orchestrating L2 (Tenant) and L3 (Service) key lifecycles, and managing MasterKey unsealing via **Seal modes**.
* [**ADR-602: Crypto Edge Workload Orchestration and DEK Lifecycle**](crypto-edge-workload-orchestration-and-dek-lifecycle.md)
  * *Context:* Defines the Edge service role in orchestrating L4 (DEK) keys directly at the application workload level.
* [**ADR-603: KMIP Functional Split between Core and Edge**](kmip-functional-split-between-core-and-edge.md)
  * *Context:* Establishes that **Crypto Core** handles L3 (KEK) operations (excluding Create/Get), while **Crypto Edge** provides high-performance L4 operations (Create, Get, Encrypt, Decrypt).
* [**ADR-604: Sub Millisecond Latency KPIs and Measurement**](sub-millisecond-latency-kpis-and-measurement.md)
  * *Context:* Establishing performance baselines to ensure the Edge-level cryptographic operations do not impact application throughput.

### VII: Protocols & Communication
* **ADR-701: KMIP Protocol Adoption and Extensions**
    * *Context:* Standardization on Key Management Interoperability Protocol for client requests.
* **ADR-702: JSON Web Token JWT vs Certificate Identity**
    * *Context:* Justification for using mTLS certificates over JWTs for long-lived service identities.

### VIII: Resilience & Global Consistency
* **ADR-801: Orbital Library Eventual Consistency Model**
    * *Context:* Logic for regional synchronization of key states and metadata.
* **ADR-802: Idempotent Task Execution and Retry Backoff**
    * *Context:* Handling reconciliation during network partitions or cloud outages.
* **ADR-803: Multi Region State Reconciliation Flow**
    * *Context:* Procedures for cross-region consistency of tenant configurations.
* **ADR-804: Disaster Recovery and Cold Start Procedures**
    * *Context:* Recovery workflows for MasterKey reconstruction in a total failure.

### IX: Observability & Audit
* **ADR-901: End to End Cryptographic Audit Logging**
    * *Context:* Standards for non-repudiable logs that track every key access event.
* **ADR-902: Correlation ID Standard for KMS Traceability**
    * *Context:* Linking internal app operations back to customer-controlled KMS logs.
* **ADR-903: Real Time Security Alerting on Key Revocation**
    * *Context:* Automated alerts for unauthorized access attempts or L1/L2 revocation.