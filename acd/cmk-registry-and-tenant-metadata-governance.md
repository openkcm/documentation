# ACD-103: CMK Registry & Global Tenant Governance

| Status | Date | Document Type |
| :--- | :--- | :--- |
| **Active** | 2026-01-14 | Architecture Concept Design |

## Overview
The **CMK Registry** is the "Ledger of Trust" and the authoritative Source of Truth for the entire OpenKCM ecosystem. Situated within the **Governance Control Plane**, this registry manages the immutable lifecycle of tenant identities, their associated cryptographic root-key references (L1), and the logical mapping of application systems to those keys.

As the backend pillar of the **Value Engine**, the Registry ensures that the "Intent" of sovereignty defined by the customer in the Portal (ACD-102) is rigorously recorded, versioned, and synchronized across the global mesh via the Orbital reconciliation engine.

## Conceptual Role: The Governance Backbone
While the regional **Crypto Service** handles high-speed execution (the "Muscle"), the **CMK Registry** handles the global intent (the "Memory"). It defines *who* owns *what* key and *which* systems are legally permitted to use them.

### Governance vs. Execution
* **CMK Registry (The Intent):** Records the policy that "Tenant A" (L2 Key) is legally bound to an L1 key located at "AWS KMS ARN:1234".
* **OpenKCM Crypto Core (The Action):** Uses this metadata to perform the actual recursive unsealing (L4 → L3 → L2 → L1) required to activate tenant services.

## Data Architecture & Entities
The registry is built on a high-integrity relational model with strict **Row-Level Security (RLS)** to ensure metadata isolation between tenants.

### Core Entity Model
| Entity | Description | Strategic Metadata |
| :--- | :--- |:---|
| **Tenant Authority** | The primary administrative unit (e.g., Enterprise Customer). | `tenant_id`, `compliance_tier`, `org_name`, `status` |
| **L1 Sovereign Ref** | The pointer to the customer's external root key. | `key_provider` (AWS/Azure/GCP/HSM), `resource_uri` (ARN/URL), `region_affinity` |
| **System Scope** | The logical workload consuming cryptographic services (e.g., "Payments App"). | `system_id`, `service_account_id`, `kmip_profile` |
| **Governance Binding** | The active policy linking a System to a specific L1 Key. | `binding_id`, `l1_ref_id`, `system_id`, `rotation_schedule` |

## Governance Workflows

### The Onboarding Handshake (BYOK/HYOK)
The Registry orchestrates the "Handshake" process. When a customer inputs their L1 reference:
1.  **Validation:** The Registry triggers a metadata check via the **Keystore Interface (KSI)**.
2.  **Commit:** Upon validation, the L1 Reference is committed to the ledger as an "Inactive" asset.
3.  **Activation:** The customer explicitly binds the L1 Ref to a System, transitioning the state to "Active" and triggering synchronization.

### Policy-Driven Revocation (The Kill-Switch Backend)
The Registry serves as the backend logic for the **Global Kill-Switch**.
1.  **Status Change:** When a "Revoke" command is issued in the Portal, the Registry atomically updates the Tenant or L1 Key status to `REVOKED`.
2.  **Event Propagation:** This state change acts as a trigger for the **Orbital** engine to broadcast high-priority "Lock" directives to the execution plane.
3.  **Audit Trail:** The Registry records the exact timestamp and user identity responsible for the revocation for compliance purposes.

## Metadata Integrity & Security

### Zero-Knowledge Principle
The Registry strictly adheres to a **Zero-Knowledge** architecture regarding key material.
* **Stores:** Metadata, ARNs, URIs, Key IDs, and Policy Configurations.
* **Never Stores:** Plaintext L1, L2, or L3 key material.
* **Impact:** A full compromise of the CMK Registry database reveals *where* keys are located, but grants *zero ability* to decrypt any customer data.

### Isolation via Row-Level Security (RLS)
To prevent cross-tenant leakage in a multi-tenant SaaS environment:
* **Database Enforcement:** Every query is filtered at the database engine level by `tenant_id`.
* **Policy Guardrails:** Administrative access is governed by **Cedar Policies**, ensuring that even platform providers cannot query sensitive customer key mappings without explicit, audited "Break-Glass" authorization.

## Integration with Orbital (State Synchronization)
The Registry does not push configuration directly to hundreds of nodes. Instead, it provides the **"Desired State"** for the **Orbital Reconciliation Engine** (ACD-201).

* **Desired State (Registry):** "Tenant A is Active using Key X."
* **Actual State (Regional):** "Tenant A is currently using Key Y."
* **Reconciliation:** Orbital detects the drift and instructs the Regional Node to rotate to Key X to match the Registry's authoritative record.

## Summary
This document defines the **Source of Truth** for OpenKCM. By centralizing the "Intent" of key management while decentralizing the "Execution," the CMK Registry ensures that:
1.  **Governance is Centralized:** A single, immutable ledger for all global cryptographic relationships.
2.  **Execution is Distributed:** Zero-latency crypto operations occur at the edge, decoupled from the registry's availability.
3.  **Integrity is Absolute:** Metadata is isolated, auditable, and physically separated from the root-key material itself.

**OpenKCM is the Value Engine that enables platforms to win bigger deals, capture higher margins, and operate with zero-risk liability in a world that demands absolute data ownership.**