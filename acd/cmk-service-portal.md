# ACD-102: OpenKCM CMK & Sovereign Portal

| Status | Date | Document Type |
| :--- | :--- | :--- |
| **Active** | 2026-01-14 | Architecture Concept Design |

## Overview
The **OpenKCM CMK** (Customer Managed Key) service is the authoritative governance engine and the "Brain" of the ecosystem. It provides the **Sovereign Portal**—the primary administrative interface where customers exercise direct ownership over their **L1 External Keys** (AWS KMS, Azure Key Vault, GCP KMS, or On-prem HSMs).

This service acts as the **Cryptographic Gateway**, ensuring that while OpenKCM manages the operational complexity of the key hierarchy, the **sovereignty** and the **legal right to decrypt** remain exclusively within the customer’s controlled keystore. As a vital pillar of the **Value Engine**, it transforms passive key storage into an active, manageable, and auditable strategic asset that enables zero-risk liability.

## The Sovereign Portal (UI/UX)
The Portal is the human-centric interface for the CMK, designed for Security Administrators to oversee and audit their organization's global cryptographic footprint across multi-cloud environments.

### Core Capabilities
* **External Key Onboarding:** A streamlined, wizard-driven workflow to link cloud-native key references (ARNs, URIs, or Aliases) to the OpenKCM mesh.
* **Real-time Status Dashboard:** A "single pane of glass" providing live telemetry on the connection status between regional execution nodes and external customer keystores.
* **The Sovereign Kill-Switch:** A high-visibility, authoritative control to revoke the platform's access to the L1 key, instantly locking dependent data globally.
* **Audit & Usage Transparency:** Integrated immutable logs documenting every request made by the platform to the customer's root key for wrapping or unwrapping operations.

## Key Management Logic & Workflows
The CMK manages the metadata and the "Handshake" logic for L1 keys. Adhering to the **Governance & Execution Framework**, it never stores plaintext key material; it only manages the authority and references required to invoke external providers.

### The BYOK/HYOK Handshake
1.  **Selection:** The administrator selects the provider (e.g., AWS KMS) and identifies the specific Key ID.
2.  **Validation:** The **Keystore Interface (KSI)** performs a non-destructive cryptographic verification (a "noop" operation) to confirm connectivity and permissions.
3.  **Linkage:** Upon successful validation, the service establishes a secure binding between the internal `L2Ref` and the `L1_External_Ref` within the specific tenant's boundary.

### Sovereign Revocation (The Kill-Switch)
When an administrator initiates a revocation in the Portal:
1.  **State Update:** The L1 key status is marked as `REVOKED/UNLINK` in the central Registry.
2.  **Orbital Dispatch:** The **Orbital engine** broadcasts a high-priority, low-latency event to all regional **OpenKCM Crypto Core** nodes.
3.  **Memory Purge & Re-encryption:** Regional nodes immediately drop any active L1-derived sessions. The L2 Key (and downstream L3 keys) are not deleted, but the L2 Key is transitioned to a state where it is kept encrypted using the **IVK (Internal Versioned Key)**. This ensures that while the structure remains, it is mathematically locked and cannot be activated without the L1 Key.
4.  **Verification:** The Portal reflects a "Locked" status, providing verifiable proof that the data silo is secure.

## Components & Architecture
The CMK is a composite service operating within the global Control Plane.

| Component | Role |
| :--- | :--- |
| **Management API** | The gRPC/REST interface for all Portal actions and automated CI/CD key-management integrations. |
| **Registry Service** | The authoritative system of record for mappings between Tenants and their L1 Root Key metadata. |
| **Keystore Interface (KSI)** | A driver-based abstraction layer that standardizes interactions with AWS, Azure, GCP, and physical HSMs. |
| **Session Manager** | Manages secure, administrative sessions gated by the **Envoy Gateway** with mandatory multi-factor authentication. |

## Security & Governance Principles
* **Zero-Knowledge Metadata:** The service maintains the location and policy of the keys, but never the key material itself.
* **Separation of Concerns:** Portal administrators manage the *policies* and *lifecycles* of the keys but are technically incapable of performing *cryptographic operations* on application data—those are reserved for regional execution nodes.
* **Non-Repudiation:** Every administrative action is digitally signed and logged, creating a tamper-proof trail for compliance frameworks such as SOC2, HIPAA, and GDPR.

## Summary
The CMK service is the "Human Head" of the platform, making **Customer-Managed Keys** operational at scale. By centralizing L1 management into a unified portal, OpenKCM allows enterprises to maintain total sovereignty over their data regardless of which cloud provider holds the physical bits.

**OpenKCM is the Value Engine that enables platforms to win bigger deals, capture higher margins, and operate with zero-risk liability in a world that demands absolute data ownership.**