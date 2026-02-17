# ACD-102: OpenKCM CMK & Sovereign Portal

| Status | Date | Document Type |
| :--- | :--- | :--- |
| **Active** | 2026-02-02 | Architecture Concept Design |

## Overview
The **OpenKCM CMK (Customer Managed Key)** service is the authoritative governance engine and the "Brain" of the ecosystem. It provides the **Sovereign Portal**—the primary administrative interface where customers exercise direct ownership over their **L1 External Keys**.

To eliminate the risk of single-user compromise or rogue insider actions, the CMK Service houses the **Internal Workflow Management** engine. This engine enforces a strict **Multi-Party Authorization (MPA)** model, ensuring that no high-impact security decision is made by a single individual. By transforming administrative actions into a formal consensus process, OpenKCM shifts the liability from the provider to a verifiable, multi-actor business process.

## Business Value: The Governance "Safeguard"
Traditional key management relies on "Super Admins" who possess god-like powers over data. OpenKCM’s Workflow Management replaces this liability with **Consensus-Based Security**:

* **Insider Threat Mitigation:** Even if a high-privileged account is compromised, the attacker cannot revoke keys or change policies without a second authorized approval.
* **Operational Quality Control:** Workflows act as a "peer review" for security configurations, preventing accidental data lockouts caused by human error.
* **Regulatory Compliance:** Satisfies stringent requirements (e.g., TISAX, SOC2, GDPR) that mandate "Separation of Duties" and the "Four-Eyes Principle."

## The Sovereign Portal (UI/UX)
The Portal is the human-centric interface for the CMK, serving as a **Collaboration Hub** for security governance. It is the primary tool used by administrators to define the security parameters of the organization.

### Administrative Configuration
The **Administrator** uses the Portal UI to define the human framework of the trust model:
* **Actor Definition:** Explicitly onboarding and managing the identities authorized to participate in the security mesh.
* **Quorum Configuration:** Defining the mandatory "Four-Eyes" rules (e.g., "Any L1 unlinking requires one member of `Security_Ops` and one member of `Compliance_Legal`").
* **Notification Routing:** Setting up the communication channels (Slack, Email, PagerDuty) that alert actors when a proposal is awaiting their signature.

## Internal Workflow Management (MPA)
The CMK Service is the enforcement point for the "Four-Eyes Principle." Every sensitive administrative API call is intercepted and wrapped in a Workflow Object that tracks the "Lifecycle of a Decision."

### The "Propose-and-Approve" Business Flow
1.  **Initiation (The Proposer):** An administrator submits a change via the Portal (e.g., Linking a new L1 ARN).
    * *Business State:* The action is staged as a "Draft Policy" within the Tenant Schema.
2.  **Notification (The Alert):** The Workflow Engine identifies the required "Approver Pool" based on the Administrator's configuration.
3.  **Ratification (The Approver):** A second administrator logs into the Portal to review the pending request.
4.  **Execution (The Activation):** Only after the second signature is the state transitioned to `ACTIVE` and propagated via **Orbital**.

### Business Scenarios for Multi-Party Approval

| Action | Business Risk | Workflow Requirement |
| :--- | :--- | :--- |
| **Link L1 Key** | Grants platform access to data. | 2-Actor Approval (Security + Compliance) |
| **Revoke L1 Key** | Permanent Data Lockout (Denial of Service). | 2-Actor Approval (Senior Leadership) |
| **Actor Configuration** | Changing who can approve actions. | 2-Actor Approval (Super-Admin + Security) |
| **Modify Cedar Policy** | Changes who can see the keys. | 2-Actor Approval (Audit + Admin) |

## Components & Architecture
The CMK operates within the global Control Plane, interacting strictly with the **Active** node of the database cluster to maintain the integrity of the pending workflow state.

| Component | Role |
| :--- | :--- |
| **Workflow Engine** | Manages the state machine of pending decisions and enforces expiration rules for approvals. |
| **Policy Engine** | Consults **Cedar Policies** to ensure the Proposer and Approver belong to distinct, non-overlapping security groups as configured by the Admin. |
| **Audit Log** | Records the "Decision Chain," capturing the identities of both the requester and the final authorizer for every action. |

## Security & Governance Principles
* **Multi-Party Authorization (MPA):** No administrative action is executed with a single user's credentials. This is a hard-coded constraint in the CMK API.
* **Non-Repudiation:** Both the Proposer and the Approver are recorded, providing a "Paper Trail" that can be verified during external audits.
* **Sovereignty is Process-Driven:** Sovereignty is not just about the *key*, it is about the *process* used to manage it, defined and governed by the customer.

## Summary
The CMK service is the "Human Head" of the platform. By wrapping every cryptographic decision in an **Internal Workflow Management** system—configured directly by administrators via the Sovereign Portal—OpenKCM ensures that sovereignty is a rigorous, consensus-based business process.

**OpenKCM is the Value Engine that enables platforms to win bigger deals by mathematically and procedurally proving that no single person—not even a provider’s employee—can compromise the customer’s system alone.**