# OpenKCM Product Vision: CMK Governance → Platform Mesh Native

**Date:** 2026-04-24
**Author:** Aysan
**Status:** Proposal — Open for Review
**Audience:** Platform Mesh team (Mirza), OpenKCM architects

---

## Why This Document Exists

Following our sync with the showroom team on Igor's development (UI and controller for OpenKCM), we observed that Platform Mesh natively provides capabilities that the CMK service was built to deliver. This document proposes consolidating some of key governance natively into Platform Mesh.

This is a product vision proposal. We are opening it as a PR to get alignment from the Platform Mesh team.

---

## What CMK Is — For Context

CMK (Customer Master Key service) is OpenKCM's governance layer for customer's key configuration and key binding (L1>L2). It never touches key material, but it controls **who can access what and under what conditions**.

Specifically, CMK is responsible for:

| Capability | What it means in practice |
| :--- | :--- |
| **Tenant Registry** | Authoritative record of which tenants exist and what they are subscribed to |
| **L1 Key Governance** | Storing references (ARNs) to customer-managed external keys; validating they are accessible |
| **Kill Switch** | Instant global revocation of a tenant's data access by revoking their L1 key reference |
| **Multi-Party Approval (Four-Eyes)** | No single admin can link or revoke a key alone — requires two distinct authorized approvers |
| **RBAC** | L1-scoped permissions preventing cross-tenant privilege escalation |
| **Audit Trail** | Tamper-evident record of every governance decision for SOC2, TISAX, GDPR compliance |
| **Sovereign Portal UI** | Human interface for administrators to perform the above operations |

The **separation between CMK (governance) and Krypton (crypto execution)** is a hard architectural principle — CMK never holds key material, Krypton never stores policy. This separation is not changing. What we are proposing is that the governance capabilities CMK provides move into Platform Mesh natively, rather than existing as a standalone service.

---

## The Observation That Triggered This Proposal

The showroom team built a Kubernetes controller and UI that demonstrates OpenKCM operating as a Platform Mesh native service. Looking at what they built, we observed:

**The controller** (`showroom-msp-openkcm`) watches Tenant CRDs and reconciles tenant lifecycle — the same thing CMK's Tenant Manager does, but using Kubernetes-native patterns instead of a custom REST API.

**The UI** is wired directly to Platform Mesh via Luigi, consuming the same portal infrastructure that all other Platform Mesh services use — instead of a standalone CMK portal.

**The CRD types** already defined (Tenant, L1KeyReference, DomainKey, ServiceKey) map directly to CMK's registry model.

This means the showroom work is not just a demo — it is a proof that the entire CMK governance model can be expressed as Platform Mesh CRDs and controllers. The standalone CMK service is not adding capability; it is duplicating what Platform Mesh already provides.

---

## The Proposal

**Consolidate the standalone CMK service into Platform Mesh as a controller (CMK-as-Controller).**

### What this looks like

| CMK (today) | CMK-as-Controller (proposed) |
| :--- | :--- |
| Custom REST/gRPC API | Kubernetes API — same as all Platform Mesh services |
| PostgreSQL tenant registry | Tenant CRD in etcd |
| PostgreSQL audit database | Platform Mesh audit trail |
| Custom approval engine | Platform Mesh native approval mechanism |
| Custom RBAC | Kubernetes RBAC + Platform Mesh policies |
| Standalone CMK portal | Platform Mesh portal (Luigi) |
| 8 microservices | 1 controller |

### What does NOT change

- **The Church & State separation (ADR-101) is preserved.** CMK-as-Controller still owns governance only — no key material, no crypto operations. Krypton still owns execution.
- **L1 key sovereignty is preserved.** Customer-managed keys (BYOK/HYOK) remain in the customer's external KMS. The controller stores only references (ARNs), never material.
- **The kill switch works.** L1KeyRevocation CRD triggers immediate revocation broadcast via Orbital, same as today. This is a security-critical capability and must be validated.
- **Multi-party approval is preserved.** The Four-Eyes Principle moves from CMK's custom workflow engine to Platform Mesh's native approval mechanism — same governance semantics, native implementation.
- **Audit trail is preserved.** Platform Mesh immutable event system replaces CMK's audit database. All governance decisions remain traceable for compliance.

---

## The Requirements Themselves Mandate This Direction

This is not just an architectural preference — the original ApeiroRA epic that defines OpenKCM's requirements explicitly states:

> *"enable OpenKCM via Platform Mesh (triggered by creation of account/tenant resource, this triggers the provisioning process in OpenKCM)"*

And in the functional requirements:
> *REQ-001: System must automatically detect new tenant/account creation in KCP*
> *REQ-002: Platform Mesh must trigger OpenKCM CMK Service provisioning without user intervention*

**A standalone CMK service can never fulfill these requirements cleanly.** It would always require polling, bridging, or a custom event listener sitting between Platform Mesh and CMK. A native controller watching Kubernetes CRDs fulfills them perfectly — tenant creation in Platform Mesh IS the trigger, and the controller reconciles it automatically.

The requirements don't describe a CMK service that integrates with Platform Mesh. They describe a CMK service that IS Platform Mesh. Our proposal is simply making the architecture match what the requirements already specify.

The requirements also define the propagation SLA explicitly:
> *REQ-022: Configuration changes must propagate to all regions within 2 minutes*

This is well within what Orbital can deliver, and removes the sub-second ambiguity we identified as a risk. The SLA is confirmed.

---

## Why Platform Mesh Is the Right Home for This

Platform Mesh already provides every building block CMK was built to deliver:

| CMK capability | Platform Mesh equivalent |
| :--- | :--- |
| Tenant registry | Kubernetes CRDs + etcd as source of truth |
| Approval workflows / Four-Eyes | Platform Mesh native approval mechanism |
| RBAC | Kubernetes RBAC + workspace isolation |
| Audit trail | Immutable Platform Mesh event system |
| Portal UI | Luigi framework (already used by showroom) |
| Managed service lifecycle | Managed Service Provider pattern (native to Platform Mesh) |

The Platform Mesh design principles — declarative APIs via Kubernetes Resource Model, MSP pattern, uniform API layer — are exactly the model CMK should have been built on. The showroom proves it is possible. The CMK standalone service is the legacy of building before Platform Mesh was mature enough.

---

## What We Need From Platform Mesh

We need clarity on the following before we can commit to this direction.

### Question 1: Multi-Party Approval — Does Platform Mesh have a native mechanism and is it production-ready?

We need multi-party approval (Four-Eyes Principle) for L1 key linking and revocation. This is a hard compliance requirement (SOC2, TISAX, GDPR).

**We need to know:**
- Does Platform Mesh have a native multi-party approval mechanism today as a shipped feature?
- Does it support: minimum approver count, role-based approver pools, separation of proposer and approver?
- Is it stable enough to build against, or is it still evolving?

**Status: ✅ Agreed** — Platform Mesh has confirmed role-based approvals will be provided. Tracked in [platform-mesh/backlog#267](https://github.com/platform-mesh/backlog/issues/267).

### Question 2: Audit Trail — Is it sufficient for compliance?

We need a tamper-evident, non-repudiable audit trail of every governance decision (who proposed, who approved, when, what changed). This must satisfy external auditors for SOC2, TISAX, and GDPR.

**We need to know:**
- What does the Platform Mesh audit trail actually record?
- Is it immutable? What prevents tampering?
- Can it be queried and exported for compliance reporting?

### Question 3: L1 Revocation — Can we guarantee propagation within 2 minutes globally?

The kill switch is our most critical security feature. When a customer revokes their L1 key, every regional Krypton node must stop serving data for that tenant. Our requirements (REQ-022) define the SLA as **2 minutes for propagation to all regions**. Today this goes via Orbital with `PRIORITY_CRITICAL` tagging.

**We need to know:**
- Can Platform Mesh guarantee propagation of a revocation event to all regions within 2 minutes?
- Does Platform Mesh have a priority lane for security-critical events, or does everything go through the same queue?

### Question 4: RBAC — Can we express L1-scoped permissions?

CMK's RBAC is scoped to individual L1 keys and tenants — not just workspace-level. An admin for tenant A must have zero visibility into tenant B's L1 key references.

**We need to know:**
- Can Platform Mesh RBAC enforce permissions at the CRD instance level (not just resource type level)?
- Can we express: "User X can approve revocations for Tenant A but not Tenant B"?

### Question 5: Roadmap alignment

We want to make sure this direction is compatible with where Platform Mesh is going, not just where it is today.

**We need to know:**
- Is OpenKCM operating as a Platform Mesh native MSP (controller + CRDs) aligned with the Platform Mesh roadmap?
- Are there planned changes to the account model, approval mechanism, or audit system that would affect this design?

### Question 6: MFA for HYOK setup

Our requirements (REQ-026) mandate multi-factor authentication for HYOK setup — this is a hard compliance requirement. HYOK means a customer is connecting their on-premises HSM or private key store, which is the highest-risk onboarding operation.

**We need to know:**
- Does Platform Mesh handle MFA natively through its identity layer?
- Or is this something OpenKCM must implement independently on top of Platform Mesh authentication?

---

## What We Are NOT Asking Platform Mesh To Do

To be clear about scope:

- We are not asking Platform Mesh to own crypto operations — Krypton stays as-is
- We are not asking Platform Mesh to store key material — ever
- We are not asking Platform Mesh to replace Orbital — event propagation stays
- We are not asking Platform Mesh to change its architecture for us — we adapt to it

We are asking whether Platform Mesh's existing capabilities are sufficient for us to build on top of, and whether this direction is strategically aligned.

---

## Risks We Have Already Identified

We are not coming to this conversation blind. The analysis we have done surfaced the following risks that we own and are responsible for resolving:

| Risk | Our mitigation |
| :--- | :--- |
| Showroom controller is demo code, not production-hardened | We will harden it — crash recovery, retry logic, observability |
| Mock API exists because real Krypton API isn't frozen | We will freeze the Krypton API contract early |
| Luigi integration not fully built | We treat this as critical path, not afterthought |
| Eventual consistency lag in revocation | We will define and measure an SLA; Orbital priority lane must be confirmed |
| No end-to-end test suite yet | We will build it — knowledge transfer doc already lists the test cases |

---

## Summary

The product vision is: **OpenKCM governance belongs in Platform Mesh, not in a standalone service.** The showroom work proves the architecture is viable. The CMK standalone service is integration debt that grows the longer we carry it.

We are proposing this as a PR to start the conversation. We are not asking for immediate approval — we are asking for feedback on the six questions above so we can make an informed decision together.

**The single most important question is Question 1 — whether Platform Mesh has a production-ready native approval mechanism. Everything else can be resolved with engineering work. That one depends on Platform Mesh.**
