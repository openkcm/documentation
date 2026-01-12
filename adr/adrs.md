---
authors:
  - Nicolae Nicora
---


# OpenKCM ADR Index

**OpenKCM (Open Key Chain Management)** is a dual-service architecture designed to transform data security from a compliance cost into a **primary revenue driver**. By decoupling **Customer Governance (CMK Service)** from **Infrastructure Operations (Crypto Service)**, the platform solves the "Trust Paradox": giving customers absolute sovereignty over their data while maintaining high-performance global operations.

### Group 1: High-Level Strategy & Governance
* [**ADR-001-Tiered-Envelope-Encryption-Model.md**](001-Tiered-Envelope-Encryption-Model.md)
    * *Context:* Defines the L1–L4 hierarchy where each parent key encrypts its child.
* **ADR-002-Separation-of-Governance-CMK-and-Execution-Crypto.md**
    * *Context:* Decouples customer trust (L1) from high-performance internal execution.
* **ADR-003-Multi-Tenant-Cryptographic-Isolation-Standard.md**
    * *Context:* Establishes mathematical siloing for shared infrastructure.
* **ADR-004-Compliance-Mapping-GDPR-DORA-Schrems-II.md**
    * *Context:* Bridges cryptographic features to specific regulatory mandates.

### Group 2: Root of Trust & MasterKey Lifecycle
* [**ADR-005-MasterKey-Definition-and-Immutability-Standard.md**](005-MasterKey-Definition-and-Immutability-Standard.md)
    * *Context:* Mandates that the MasterKey serves as the stable, non-rotating root of trust.
* **ADR-006-MasterKey-Reconstruction-via-Shamir-Secret-Sharing.md**
    * *Context:* Details splitting the MasterKey into N shards, requiring M for unsealing.
* **ADR-007-Auto-Unseal-Strategy-via-Cloud-KMS-Wrap.md**
    * *Context:* Details cloud-native automated decryption of the MasterKey blob at startup.
* **ADR-008-MasterKey-Storage-and-Shard-Encryption-Policy.md**
    * *Context:* Policies for encrypting shards using different keystores (AWS, Azure, GCP, HSM).

### Group 3: Internal Key Logic (IVK)
* [**ADR-009-IVK-Logical-Architecture-and-Versioning-Schema.md**](009-IVK-Logical-Architecture-and-Versioning-Schema.md)
    * *Context:* Establishes the IVK as the rotatable intermediary encrypted by the MasterKey.
* **ADR-010-IVK-Mapping-Logic-to-L2-Tenant-Keys.md**
    * *Context:* Technical implementation of how IVKs wrap per-tenant L2 keys.
* **ADR-011-IVK-Rotation-Triggers-and-Grace-Period-Management.md**
    * *Context:* Logic for periodic rotation to new IVK versions for forward security.
* **ADR-012-IVK-Metadata-Persistence-and-Cache-Policy.md**
    * *Context:* Standards for storing and retrieving versioned IVK encrypted blobs.

### Group 4: Security, Identity & Authorization
* **ADR-013-Mandatory-Mutual-TLS-mTLS-for-All-Service-Traffic.md**
    * *Context:* Requirement for x509 certificate-based authentication for all internal nodes.
* **ADR-014-Identity-Bound-Key-Authorization-Engine.md**
    * *Context:* **Authorization Core:** Validating that a caller identity (via mTLS certificate) is explicitly permitted to request keys for a specific Tenant ID.
* **ADR-015-Zero-Knowledge-Storage-Principles-for-Key-Metadata.md**
    * *Context:* Ensuring that persisted metadata is unusable without the reconstructed MasterKey in memory.
* **ADR-016-Four-Eyes-Principle-Enforcement-via-Quorum-Authorization.md**
    * *Context:* Requiring M distinct human/service approvals for high-risk admin tasks.
* **ADR-017-Secure-Bootstrapping-and-Secret-Injection-Policy.md**
    * *Context:* How initial secrets and shards are securely delivered to new nodes during scaling.

### Group 5: CMK Service (Customer Managed Key Layer)
* **ADR-018-Pluggable-KMS-Provider-Interface.md**
    * *Context:* Driver architecture for AWS KMS, GCP KMS, Azure Vault, and HSMs.
* **ADR-019-L1-Key-Reference-Storage-and-Metadata-Schema.md**
    * *Context:* Handling encrypted references without touching plaintext material.
* **ADR-020-Instant-Revocation-and-Kill-Switch-Mechanics.md**
    * *Context:* The "Cryptographic Erasure" flow: how L1 deletion invalidates the tenant chain.
* **ADR-021-BYOK-and-HYOK-Onboarding-Workflows.md**
    * *Context:* Procedures for integrating customer-provided keys into the L1 layer.

### Group 6: Crypto (Core & Edge Dual-Service Model)
* **ADR-022-Crypto-Core-Central-Orchestration-and-KEK-Lifecycle.md**
  * *Context:* Defines the Core service role in orchestrating L2 (Tenant) and L3 (Service) key lifecycles, and managing MasterKey unsealing via **Seal modes**.
* **ADR-023-Crypto-Edge-Workload-Orchestration-and-DEK-Lifecycle.md**
  * *Context:* Defines the Edge service role in orchestrating L4 (DEK) keys directly at the application workload level.
* **ADR-024-KMIP-Functional-Split-between-Core-and-Edge.md**
  * *Context:* Establishes that **Crypto Core** handles L3 (KEK) operations (excluding Create/Get), while **Crypto Edge** provides high-performance L4 operations (Create, Get, Encrypt, Decrypt).
* **ADR-025-Sub-Millisecond-Latency-KPIs-and-Measurement.md**
  * *Context:* Establishing performance baselines to ensure the Edge-level cryptographic operations do not impact application throughput.

### Group 7: Protocols & Communication
* **ADR-026-KMIP-Protocol-Adoption-and-Extensions.md**
    * *Context:* Standardization on Key Management Interoperability Protocol for client requests.
* **ADR-027-JSON-Web-Token-JWT-vs-Certificate-Identity.md**
    * *Context:* Justification for using mTLS certificates over JWTs for long-lived service identities.

### Group 8: Resilience & Global Consistency
* **ADR-028-Orbital-Library-Eventual-Consistency-Model.md**
    * *Context:* Logic for regional synchronization of key states and metadata.
* **ADR-029-Idempotent-Task-Execution-and-Retry-Backoff.md**
    * *Context:* Handling reconciliation during network partitions or cloud outages.
* **ADR-030-Multi-Region-State-Reconciliation-Flow.md**
    * *Context:* Procedures for cross-region consistency of tenant configurations.
* **ADR-031-Disaster-Recovery-and-Cold-Start-Procedures.md**
    * *Context:* Recovery workflows for MasterKey reconstruction in a total failure.

### Group 9: Observability & Audit
* **ADR-032-End-to-End-Cryptographic-Audit-Logging.md**
    * *Context:* Standards for non-repudiable logs that track every key access event.
* **ADR-033-Correlation-ID-Standard-for-KMS-Traceability.md**
    * *Context:* Linking internal app operations back to customer-controlled KMS logs.
* **ADR-034-Real-Time-Security-Alerting-on-Key-Revocation.md**
    * *Context:* Automated alerts for unauthorized access attempts or L1/L2 revocation.