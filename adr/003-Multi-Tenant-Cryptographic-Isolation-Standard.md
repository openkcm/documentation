---
authors:
  - Nicolae Nicora
---

# Multi-Tenant Cryptographic Isolation Standard

**Status:** Accepted  
**Date:** 2026-01-12


## Context
OpenKCM is designed as a multi-tenant platform where a single regional cluster (**OpenKCM Crypto**) serves thousands of distinct customers. Logical isolation (database row-level security or namespace isolation) is insufficient to prevent cross-tenant data access if the application layer is compromised.

We require **Mathematical Siloing**—where data belonging to Tenant A is cryptographically inaccessible to Tenant B, even if Tenant B gains access to the storage layer or the shared execution environment.


## Decision
Implement a mandatory **Multi-Tenant Cryptographic Isolation (MTCI)** standard based on a unique Key-Per-Tenant (KPT) model. Central to this decision is the role of the **L2 Tenant Encryption Key** as the primary isolation boundary.

### Unique Tenant Entropy (L2 Boundary)
Every tenant onboarded to OpenKCM must have at least one dedicated **L2 Tenant Encryption Key**.

* **Uniqueness:** No two tenants shall ever share the same L2 key material.
* **Multi-Key Support:** A single tenant **MAY possess multiple L2 keys**. This supports:
  * **Geographic Sovereignty:** Separate L2 keys for different regional data silos (e.g., `L2-EU`, `L2-US`).
  * **Departmental Isolation:** Dedicated L2 keys for specific internal business units (e.g., `L2-Finance`, `L2-HR`).
  * **Rotation Windows:** Overlapping L2 versions during background re-encryption.
* **Derivation:** All L2 keys must be generated using a FIPS-compliant CSPRNG within the secure memory of the Core Crypto service.
* **Isolation Role:** Each L2 key acts as the "Master KEK" (Key Encryption Key) for its specific branch of the cryptographic tree.


### Envelope Encryption Standards
Isolation is enforced through strict envelope encryption. Even if a tenant has multiple L2 keys, each branch remains mathematically isolated:
1.  **L1 → L2:** Every L2 key is wrapped by the Customer's L1 CMK (via AWS KMS, Azure KV, etc.).
2.  **L2 → L3:** Application-specific L3 keys are wrapped by a specific chosen L2 key.
3.  **L3 → L4:** Data Encryption Keys (DEKs) are wrapped by the L3 key.

### Cryptographic Context Injection
All KMIP operations (Encrypt/Decrypt) must include a **Cryptographic Context** (Authenticated Additional Data - AAD):
* **Tenant Binding:** The `Tenant_ID` and the specific `L2_Key_ID` are injected into the AEAD (AES-GCM or Poly1305) tag calculation.
* **Enforcement:** Decryption will fail if the provided `Tenant_ID` or `L2_Key_ID` does not match the material used during the original encryption, preventing "ciphertext relocation" attacks.

## Technical Requirements

### Metadata Segregation
To support multiple L2 keys, the metadata storage must maintain a strict lineage:
* **Composite Indexing:** Every managed object is indexed by `[Tenant_ID][L2_Key_ID][Object_UUID]`.
* **Row-Level Security (RLS):** PostgreSQL RLS policies must enforce that the Execution Plane can only query objects belonging to the `Tenant_ID` authenticated via mTLS.

### Authorization Enforcement
The **OpenKCM Crypto** execution plane must verify ownership at every KMIP request:
1.  **Identity Extraction:** Extract `Tenant_ID` from the client's mTLS x.509 certificate.
2.  **Access Verification:** Verify that the requested L3 or L4 key descends from an L2 key owned by that specific `Tenant_ID`.
3.  **Operation Execution:** The request is rejected if the hierarchy traversal attempts to cross a tenant boundary.

## Consequences

### ✅ Positive (Pros)
* **Granular Sovereignty:** Tenants can "shred" specific business units or regions by revoking only the associated L2 key or its L1 parent.
* **Mathematical Certainty:** Cross-tenant leakage is physically impossible; a compromise of one tenant's keys does not provide access to any other tenant's material.
* **Regulatory Compliance:** Directly addresses GDPR and DORA requirements for data segregation and localized cryptographic control.

### ⚠️ Negative (Cons) & Mitigations
* **Management Complexity:** Supporting multiple L2 keys per tenant increases the complexity of the key discovery logic.
  * *Mitigation:* Use KMIP `Locate` operations with specific attributes to filter the correct L2 branch automatically.
* **Database Growth:** Each new L2 branch adds significant metadata overhead.
  * *Mitigation:* Implement automated scrubbing for retired/expired L2 hierarchies once all data has been rotated.
