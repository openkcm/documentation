# OpenKCM CMK Layer Elimination: Technical Argument for Review

**Date:** 2026-04-24
**Author:** Aysan
**Status:** Internal Review — Pre-PR
**Reviewers:** Architecture team, Tech Lead
**Related:** [CMK Elimination Index](CMK-ELIMINATION-INDEX.md) · [Platform Mesh Alignment Proposal](cmk-elimination-platform-mesh-alignment.md) · [ADR-105](../adr/cmk-as-controller-platform-mesh-native.md)

---

## Purpose

This document makes the case for eliminating the standalone CMK service and moving key governance natively into Platform Mesh. It is written for internal architecture review before opening a PR to the Platform Mesh team and before presenting to ApeiroRA security architect and requirements owner.

The argument is built from three sources: the ApeiroRA requirements epic, the Platform Mesh architecture documentation, and the showroom implementation evidence.

---

## The Requirements Already Specify This Architecture

The ApeiroRA epic defining OpenKCM's requirements states:

> *"enable OpenKCM via Platform Mesh (triggered by creation of account/tenant resource, this triggers the provisioning process in OpenKCM)"*

And the functional requirements are explicit:

> *REQ-001: System must automatically detect new tenant/account creation in KCP*
> *REQ-002: Platform Mesh must trigger OpenKCM CMK Service provisioning without user intervention*

This is not a vague integration requirement. It mandates that Platform Mesh account creation IS the trigger for OpenKCM provisioning. A standalone CMK service structurally cannot fulfill this — it would always require a polling mechanism or a custom event bridge sitting between Platform Mesh and CMK. A Kubernetes controller watching account CRDs fulfills it exactly, with no bridge required.

Furthermore, the Platform Mesh account model documentation explicitly names OpenKCM as the key management layer for Platform Mesh accounts:

> *"The platform provides comprehensive key management capabilities for accounts, leveraging OpenKCM (Key Chain Management) to offer flexible and robust data protection."*

OpenKCM is not described as a service that integrates with Platform Mesh from the outside. It is described as part of the Platform Mesh account model. The requirements and the Platform Mesh architecture documentation are already aligned on this. Our current standalone CMK architecture is the anomaly.

---

## What CMK Does Today and Why It Is Redundant

CMK is OpenKCM's governance layer — the "brain" that controls who can access what and under what conditions, without ever touching key material. It owns:

- Tenant registry (who exists, what they are subscribed to)
- L1 key governance (BYOK/HYOK — storing references to customer-managed external keys)
- Kill switch (instant global revocation via Orbital)
- Multi-party approval (Four-Eyes Principle for L1 linking and revocation)
- RBAC (L1-scoped permissions, cross-tenant isolation)
- Audit trail (tamper-evident compliance record for SOC2, TISAX, GDPR)
- Sovereign Portal UI (human interface for the above)

Platform Mesh natively provides every one of these:

| CMK Capability | Platform Mesh Native Equivalent |
| :--- | :--- |
| Tenant registry | Kubernetes CRDs + etcd — declarative, versioned, GitOps-native |
| Approval workflows (Four-Eyes) | ApprovalPolicy CRD — declarative multi-party approval |
| RBAC | Kubernetes RBAC + workspace isolation — instance-scoped permissions |
| Audit trail | Immutable Platform Mesh event system — richer than a PostgreSQL table |
| Portal UI | Luigi framework — same portal all Platform Mesh services use |
| Managed service lifecycle | MSP pattern — native to Platform Mesh, exactly the model CMK needs |
| Tenant auto-provisioning | Controller watching account CRDs — fulfills REQ-001 and REQ-002 natively |

The standalone CMK service exists because Platform Mesh was not mature enough when CMK was designed. That is no longer the case.

---

## The Showroom Proves It Is Possible

The showroom team (`showroom-msp-openkcm`) built a Kubernetes controller and UI demonstrating OpenKCM operating as a Platform Mesh native service. This is not theoretical — it is working code. What they built proves three things:

**1. The CRD model covers the CMK data model.** The types already defined — `Tenant`, `L1KeyReference`, `DomainKey`, `ServiceKey` — map directly to the CMK registry model. The data layer is solved.

**2. The controller pattern replaces the CMK service.** The tenant controller watches Tenant CRDs and reconciles tenant lifecycle using standard Kubernetes operator patterns — the same reconciliation loop that the MSP pattern mandates. No custom REST API required.

**3. The UI works inside Platform Mesh.** The showroom UI is wired to Luigi and runs inside the Platform Mesh portal. The CMK portal does not need to be a standalone service — it is a Platform Mesh micro-frontend.

The showroom work also reveals what is not yet done: the controller currently calls a mock OpenKCM REST API, not Krypton directly. The UI has a hard dependency on Luigi context injection. These are the integration gaps that CMK-as-Controller closes by making the controller a first-class Platform Mesh component rather than a bridge.

---

## L1 Key Model: Always Present, Either Platform or Customer

This is the key architectural decision that emerged from the internal review discussion.

**Every account has an L1 key reference. Always. No exceptions.**

When a tenant enables OpenKCM, the platform automatically provisions a platform-managed L1 key reference at account creation. The customer can replace it with their own key (BYOK/HYOK) at any time. The `L1KeyReference` field on the Tenant CRD is always populated — it is never empty.

This model resolves two concerns raised during review:

**The chain enforcement concern (team lead):** The key chain is always complete. Breaking it is always a deliberate act — an explicit `L1KeyRevocation` requiring Four-Eyes approval — never an accidental side effect of a misconfiguration. If the chain is intentionally broken, crypto stops. That is the point of the kill switch.

**The operational safety concern (architect):** DEKs are always decrypt-able because an L1 is always present at the account level. A database volume cannot be accidentally bricked by a missing root key because there is no state where the root key is missing. The platform-managed L1 is the safety net; BYOK/HYOK is the upgrade path.

| Mode | L1 owner | How it's set |
| :--- | :--- | :--- |
| **Platform-managed** | Provider (default) | Auto-provisioned at account creation |
| **Customer-managed (BYOK/HYOK)** | Customer's external KMS | Customer replaces the `L1KeyReference` on the Tenant CRD |

There is no mode with no L1. Tier 3 (no OpenKCM, no key governance) is a valid choice — but any account that enables OpenKCM always has an L1, whether platform-provisioned or customer-supplied.

**The product statement:** OpenKCM provisions an L1 key reference for every account at creation. The platform holds it by default. Customers can replace it with their own key at any time. The key chain is always complete. The kill switch always works.

---

## What Does NOT Change

This is the most important section for the security review.

**The Church and State separation (ADR-101) is fully preserved.** CMK-as-Controller owns governance only — it stores references, manages approvals, and broadcasts revocations. It never holds key material, never performs crypto operations. Krypton owns execution exactly as it does today.

| Principle | Today | CMK-as-Controller |
| :--- | :--- | :--- |
| CMK never holds key material | ✅ | ✅ Preserved |
| Krypton never stores policy | ✅ | ✅ Preserved |
| L1 keys stay in customer KMS | ✅ | ✅ Preserved |
| Kill switch via Orbital | ✅ | ✅ Preserved |
| Four-Eyes on L1 operations | ✅ | ✅ Via ApprovalPolicy |
| Audit trail for compliance | ✅ | ✅ Via Platform Mesh events |
| BYOK/HYOK support | ✅ | ✅ Via L1KeyReference CRD |

The propagation SLA from REQ-022 — configuration changes within 2 minutes globally — is well within what Orbital delivers. This is not a risk.

---

## What Gets Eliminated and Why Each Is Safe to Remove

**CMK API Server (REST/gRPC)** — replaced by Kubernetes API. All Platform Mesh services use KRM. There is no functional loss; it is a delivery mechanism change mandated by the Platform Mesh guiding principles: *"The API design is solely dedicated to a declarative model, with no support for imperative requests."* A custom REST API is architecturally inconsistent with Platform Mesh.

**CMK Registry Database (PostgreSQL)** — replaced by Tenant CRD in etcd. The Platform Mesh account model uses etcd as the source of truth for all service state. Keeping a separate PostgreSQL registry creates the dual-write problem: two sources of truth that must stay in sync. Eliminating it removes an entire class of consistency bugs.

**CMK Approval Engine** — replaced by Platform Mesh ApprovalPolicy CRD. The Four-Eyes Principle semantics are preserved. The implementation moves from custom code to a platform-native feature that is reusable across all governance operations.

**CMK RBAC** — replaced by Kubernetes RBAC with workspace isolation. The Platform Mesh account model provides hierarchical isolation — each workspace is an independent control plane. L1-scoped permissions map to workspace-scoped RBAC rules.

**CMK Audit Database (PostgreSQL)** — replaced by Platform Mesh immutable event system. The requirements need a tamper-evident, non-repudiable audit trail (REQ-025). The Platform Mesh event system satisfies this. A separate PostgreSQL database is not more compliant — it is more fragile.

**CMK Sovereign Portal (standalone)** — replaced by Platform Mesh portal via Luigi. REQ-005 requires the OpenKCM UI to be accessible through the user's account. The Platform Mesh portal already fulfills this. A standalone portal requires users to context-switch out of Platform Mesh to manage their keys — the opposite of what REQ-001 through REQ-006 describe.

**CMK UI → Luigi integration work** — this work was never completed. The showroom proves it is needed (the UI fails without Luigi context). Eliminating CMK as a standalone service means this work is no longer required — the UI becomes a native Platform Mesh micro-frontend from the start.

---

## The Cost of Doing Nothing

If we continue with standalone CMK, we must:

1. **Complete the Luigi integration** — the showroom UI cannot ship without it. This is significant engineering work: portal registration, context injection, navigation wiring, authentication delegation. We would be building a Luigi wrapper for capabilities the Platform Mesh portal already has.

2. **Build and maintain the event bridge** — REQ-001 and REQ-002 require Platform Mesh account creation to trigger CMK provisioning. Without a native controller, this requires a custom event listener permanently sitting between Platform Mesh and CMK. This is infrastructure debt that grows with every Platform Mesh update.

3. **Keep two sources of truth in sync** — CMK PostgreSQL registry and Kubernetes CRDs must remain consistent. Every Platform Mesh API change risks breaking the sync. This is where production incidents come from.

4. **Pay for infrastructure that duplicates what we already have** — 8 microservices, 2 databases, 15+ custom API endpoints. Estimated $12K–28K per year, plus operational overhead.

The cost of doing nothing is not zero. It is continuous investment in an integration that proves our own argument against itself.

---

## Open Questions for the Architecture Review

The L1 key model (always present, platform or customer) is resolved — see section above. These are the remaining items to validate before opening the PR:

**Q1: ApprovalPolicy CRD** — Is this production-ready in Platform Mesh today? This is the single biggest dependency. If it does not exist or is not stable, the governance model cannot ship. We need a concrete answer before going to Mirza.

**Q2: Controller production hardening** — The showroom controller is demonstration code. It needs crash recovery, retry logic, idempotency guarantees, and observability before it can be considered production-ready. Is the team aligned on this scope?

**Q3: Krypton API contract** — The controller will call Krypton directly (not via a mock). Does a stable, versioned Krypton API exist? If not, freezing it is a prerequisite.

**Q4: Multi-cluster reconciliation** — The showroom uses `sigs.k8s.io/multicluster-runtime`. Is this the agreed approach for cross-workspace controller patterns in OpenKCM, or is there a preferred alternative?

**Q5: MFA for HYOK** — REQ-026 requires multi-factor authentication for HYOK setup. Does Platform Mesh handle this through its identity layer, or must the controller implement it independently?

---

## Summary

The product vision is: **OpenKCM key governance belongs in Platform Mesh, not in a standalone service.**

This is not a preference — it is what the requirements specify (REQ-001, REQ-002), what the Platform Mesh account model documentation describes (OpenKCM as the key management layer for accounts), and what the showroom implementation demonstrates (a working controller and UI running natively on Platform Mesh).

The standalone CMK service is the consequence of building before Platform Mesh was mature. The maturity is now there. Continuing to build and maintain CMK as a standalone service means permanently paying for an integration that duplicates what Platform Mesh already provides — while never fully satisfying the requirements that mandate native integration in the first place.

The architecture review should focus on the five open questions above. If those are resolved, the path to the Platform Mesh alignment PR and the conversation with head architecture of ApeiroRA is clear.
