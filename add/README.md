---
authors:
  - Nicolae Nicora
---

# OpenKCM Architecture Design Document (ADD) Index

### I. Core Infrastructure & Execution Services
* **ADD-101: CMK Control Plane & Registry** – Registry database schemas, Governance API specification, and tenant onboarding state machines.
* **ADD-102: Crypto Core Engine (Regional)** – Regional execution logic, MasterKey memory management, and L1→L3 unsealing sequences.
* **ADD-103: Crypto Gateway Module (Sidecar)** – High-throughput L4 DEK lifecycle, local caching strategies (LRU), and application sidecar design.
* **ADD-104: Sovereign Portal & BYOK Workflows** – Frontend-to-Backend contracts for the "Kill-Switch" and key linkage Wizards.

### II. Key Lifecycle & Root of Trust
* **ADD-201: Hierarchical Envelope Encryption** – Technical specification for L1→L4 wrapping formats and AES-GCM header structures.
* **ADD-202: MasterKey Unsealing State Machine** – Shamir (M,N) reconstruction protocols and Seal/Auto-unseal initialization flows.
* **ADD-203: IVK & L2 Rotation Logic** – Orchestration of background "Lazy Re-encryption," Internal Versioned Key (IVK) management, and transition states.

### III. Protocols & Secure Communications
* **ADD-301: KMIP 1.4 Implementation Profile** – Supported operation mapping (Get/Create/Destroy) and Protobuf/TTL message formats.
* **ADD-302: mTLS & Identity Mapping** – Certificate parsing logic, SPIFFE ID validation, and SubjectCN-to-TenantID authorization binding.
* **ADD-303: Secure Memory & Zeroization** – Go implementation details for `runtime/secret` handling, `mlock` OS calls, and memory purging routines.

### IV. Global Connectivity & Data Integrity
* **ADD-401: Orbital Library & Task Dispatch** – Message broker topology and idempotent reconciliation logic using the `orbital` library. Defines `TaskRequest` and `TaskResponse` Protobufs for "at-least-once" delivery.
* **ADD-402: Regional Keystore & RLS Schema** – PostgreSQL Row-Level Security (RLS) definitions and regional metadata persistence specifications.
* **ADD-403: Pluggable Keystore Interface (KSI)** – Driver specifications for adapting external L1 providers (AWS KMS, Azure KV, GCP KMS) into a unified internal interface.

### V. Audit & Observability
* **ADD-501: Cryptographic Audit & Traceability** – Non-repudiable log formats, Merkle Tree signing chains, and Cloud KMS correlation ID tracking.
* **ADD-502: Metrics & Health Introspection** – Prometheus metric definitions for cryptographic operations, latency histograms, and "Checker" sidecar health probes.