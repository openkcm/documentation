# ACD-104: Identity & Zero-Trust Authorization

| Status | Date | Document Type |
| :--- | :--- | :--- |
| **Active** | 2026-01-13 | Architecture Concept Design |

## Overview
The **Identity & Zero-Trust Authorization** component establishes the intelligent security perimeter of the OpenKCM ecosystem. It codifies a **Zero-Trust Architecture (ZTA)** where no entity—service or human—is trusted by default. By integrating **mutual TLS (mTLS)** for cryptographic service identity and the **Cedar Policy Language** for high-performance authorization, OpenKCM ensures that every cryptographic operation is verified and scoped strictly to a tenant's mathematical silo.

## The Identity Framework
OpenKCM utilizes a dual-identity model to manage both autonomous service traffic and human administrative actions.

### Service Identity (mTLS)
All internal communication (e.g., between the CMK API and regional nodes) requires **Mutual TLS (mTLS)**.
* **Certificate Gateway:** Traffic is funneled through an Envoy-based Certificate Gateway that requires x509 Client Certificates.
* **Trusted CA:** Identity is verified against a list of Trusted CA Certificates.
* **Automatic Enrichment:** The gateway filters and enriches requests with identity metadata before forwarding them to internal services.

### Human & Administrative Identity (OIDC/JWT)
Administrative access to the CMK Portal is managed via external Identity Providers (IDPs).
* **Session Gateway:** A dedicated Session/JWT Gateway (Envoy) handles authentication, callbacks, and logout flows.
* **JWT Validation:** The **Session Manager** interacts with a **Valkey** cache to manage session states and check for token revocation.
* **IDP Integration:** OpenKCM connects to external IDPs to retrieve OIDC provider details via gRPC and mTLS.



## Cedar: The Authorization Engine
OpenKCM leverages the **Cedar Policy Language** to separate authorization logic from application code, providing the expressiveness needed for complex multi-tenant requirements.

### Policy Evaluation
* **ExtAuthZ Service:** An External Authorization service acts as the bridge between incoming requests and the policy store.
* **Cedar Policies:** Authorization decisions are evaluated against a central repository of Cedar Policies.
* **Fine-Grained Context:** The engine enriches requests with gRPC and mTLS context to make real-time "Allow/Deny" decisions.

## Enforcement via Envoy & ExtAuthZ
Authorization is enforced at the network level, ensuring that unauthorized requests never reach the sensitive **CMK API Server** or **Registry Service**.

### The Enforcement Loop
1.  **Request Capture:** The **Envoy Proxy** (acting as a Gateway) intercepts all incoming traffic from consumers and external systems.
2.  **Identity Extraction:** Envoy extracts the **mTLS certificate** (for services) or **JWT token** (for users).
3.  **ExtAuthZ Trigger:** Envoy calls the **ExtAuthZ** service via gRPC, passing the identity and request metadata.
4.  **Policy Query:** The ExtAuthZ service queries the **Cedar Policy** engine to validate the request against defined rules.
5.  **Decision Delivery:** If authorized, the request is forwarded to the appropriate internal component (e.g., CMK API Server or Tenant Manager); otherwise, it is blocked.



## Security Outcomes

### Mathematical Multi-Tenancy
The authorization layer serves as the logical gatekeeper for cryptographic silos. Even if a user gains access to the portal, Cedar ensures they can only interact with resources tagged with their specific `tenant_id`.

### Separation of Duties (SoD)
The platform enforces a strict SoD model through component isolation:
* **Governance Operators:** Manage the Registry and Cedar policies but lack the cryptographic identity to execute regional tasks.
* **Regional Crypto Nodes:** Have identities permitted to perform encryption/decryption tasks but are restricted from modifying the global CMK Registry.

## Compliance & Auditability
* **Verifiable Policy Store:** Cedar policies are human-readable, allowing auditors to verify access rules without inspecting low-level source code.
* **End-to-End Audit:** Every authorization decision is traceable from the initial Envoy gateway through the ExtAuthZ service to the final database record.
* **Centralized Logging:** All administrative and service actions are captured in the **PostgreSQL** database for long-term audit retention.

## Summary
This document establishes the **Intelligent Perimeter** of OpenKCM. By combining mTLS for service identity and Cedar for granular policy, the platform ensures that:

* **Identity is Cryptographic:** No request is anonymous; every service is verified via x509 certificates.
* **Access is Scoped:** The **ExtAuthZ** service ensures that actions are strictly limited by policy.
* **Trust is Never Assumed:** The gateway architecture ensures that every entry point is explicitly secured by Envoy.