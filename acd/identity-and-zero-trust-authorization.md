# ACD-104: Identity & Zero-Trust Authorization

| Status | Date | Document Type |
| :--- | :--- | :--- |
| **Active** | 2026-01-31 | Architecture Concept Design |

## Overview
The **Identity & Authorization** component establishes the security perimeter of the OpenKCM ecosystem. To ensure defense-in-depth and separation of concerns, OpenKCM implements a strict **Two-Level Authorization Strategy**.

1.  **Level 1 (The Edge):** Infrastructure & API Access Control enforced by **Envoy ExtAuthZ** using **Cedar Policies**.
2.  **Level 2 (The Core):** Functional RBAC & Data Sovereignty enforced by the **CMK Service** using internal Group/Role definitions managed via the **Sovereign Portal**.



## Level 1: Edge Authorization (Envoy + Cedar)
The first line of defense occurs at the network ingress. No request reaches the application logic unless it passes this gate.

### Mechanism
* **Enforcement Point:** The **Envoy Gateway** acts as the policy enforcement point (PEP).
* **Decision Engine:** Envoy delegates the decision to an **ExtAuthZ** service, which evaluates **Cedar Policies**.
* **Scope:** This level determines **"API Access."** It answers: *Is Principal X allowed to call RPC method Y?*

### The Policy Check
When a request arrives (e.g., `POST /RegisterTenant`):
1.  **Identity Extraction:** Envoy validates the mTLS certificate or OIDC Token.
2.  **Policy Query:** The ExtAuthZ service queries the Cedar engine:
    > *"Does User 'alice@corp.com' have 'Action::Call_RegisterTenant' on 'Resource::GlobalRegistry'?"*
3.  **Outcome:**
    * **Deny:** Connection is dropped with `403 Forbidden`. The CMK Service never sees the traffic.
    * **Allow:** Request is forwarded to the backend.

## Level 2: Application RBAC & Group Management
Once a request passes the edge, the **CMK Service** performs the second, fine-grained authorization check based on business logic and functional roles.

### Mechanism
* **Enforcement Point:** The **CMK API Server** (Application Code).
* **Logic:** Role-Based Access Control (RBAC) rooted in the **Tenant Schema**.
* **Scope:** This level determines **"Functional capabilities."** It answers: *Does User X belong to a Group that has the rights to perform this specific business workflow?*

### Group Management (via CMK UI)
Unlike the low-level Cedar policies, Group Management is a human-centric administrative task performed within the **Sovereign Portal (CMK UI)**.

* **Administrator Action:** An Admin logs into the Portal to create Groups (e.g., "Security_Admins", "Audit_Viewers") and assign Users to them.
* **Storage:** These associations are stored in the **Tenant’s Private Schema**.
* **Evaluation:** When the API receives a request, it looks up the user's Group assignments in the database to validate they have the specific permission (e.g., `CAN_APPROVE_WORKFLOW`, `CAN_REVOKE_KEY`).

## The Complete Authorization Flow

1.  **Ingress:** User calls `RotateKey()`.
2.  **Level 1 Check (Envoy/Cedar):**
    * *Check:* "Is this user allowed to talk to the Key Management API?"
    * *Result:* **PASS**. (User is a valid employee).
3.  **Forwarding:** Request reaches CMK Service.
4.  **Level 2 Check (CMK RBAC):**
    * *Check:* "Does this user belong to the 'Crypto_Ops' group for 'Tenant_A'?"
    * *Result:* **PASS**. (User is assigned to the correct group in the UI).
5.  **Execution:** The key rotation logic proceeds.

## Security Outcomes

### Defense in Depth
If an attacker bypasses the Envoy gateway (e.g., via a misconfiguration), the **Level 2 RBAC** in the application still prevents unauthorized access. Conversely, if the application has a bug, the **Level 1 Cedar Policy** restricts which APIs can even be reached.

### Separation of Configuration
* **Platform Engineers** manage **Level 1 (Cedar)** to secure the infrastructure APIs.
* **Tenant Administrators** manage **Level 2 (Groups)** via the UI to secure their own business operations.

## Summary
ACD-104 defines a robust authorization pipeline. **Level 1 (Envoy/Cedar)** acts as the rigorous "Bouncer" at the door, ensuring only valid principals enter the building. **Level 2 (CMK RBAC)** acts as the "Internal ID Badge," ensuring users can only access the specific rooms and workflows (Groups) defined by their administrators in the **Sovereign Portal**.