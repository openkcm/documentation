# CMK Integration: Product Vision Index

**Last Updated:** 2026-04-24
**Status:** Approved
**Owner:** Product Management

---

## Documents

| Document | Purpose | Audience | Read Time |
| :--- | :--- | :--- | :--- |
| [Internal Review Argument](cmk-integration-internal-review.md) | Full technical argument for architect and tech lead review — grounded in ApeiroRA requirements and Platform Mesh docs | Architecture team, Tech Lead | 15 min |
| [Platform Mesh Alignment Proposal](cmk-integration-platform-mesh-alignment.md) | PR document for Platform Mesh team — explains CMK, the proposal, and the 6 questions we need answered | Platform Mesh team, Mirza | 15 min |
| [Architecture Argument](cmk-integration-argument.md) | Why we are eliminating the CMK layer — structured argument covering duplication, integration cost, and platform alignment | Management, Leadership | 10 min |
| [Executive Summary](cmk-integration-executive-summary.md) | What changes, what gets eliminated, cost impact, and high-level timeline | Leadership, Product | 5 min |
| [What's Removed, What's Gained](cmk-integration-what-removed-gained.md) | Before/after comparison of the architecture | Product, Engineering leads | 15 min |
| [ADR-105: CMK-as-Controller](../adr/cmk-as-controller-platform-mesh-native.md) | The architecture decision record — owned by engineering | Architects, Engineers | 20 min |

---

## The Decision in One Paragraph

The standalone CMK service duplicates capabilities that Platform Mesh already provides natively — approval workflows, RBAC, audit trail, and tenant registry. Completing the CMK integration would require building a Luigi UI wrapper for functionality the Platform Mesh portal already has. Instead, CMK governance and L1 key management move into Platform Mesh as a native controller. The result is a simpler architecture, a single source of truth, and a saving of $12K–28K per year.

---

## Key Decisions

| Decision | Chosen path | Why |
| :--- | :--- | :--- |
| Full elimination vs. CMK v2 (audit-only) | Full elimination | CMK v2 still required its own database and sync logic — not simpler enough |
| Where governance lives | Platform Mesh ApprovalPolicy | Already exists, reusable, declarative, with integrated audit trail |
| Where L1 validation lives | Krypton | Krypton holds all backend libraries (AWS, GCP, Azure, HSM); keeps crypto at the crypto layer |
| Source of truth | Kubernetes CRDs | Single source of truth, GitOps-native, no separate database |

---

## Cost Impact

| | CMK (current) | CMK-as-Controller | Savings |
| :--- | :--- | :--- | :--- |
| Annual | $13K–29K | $0.5K–1K | **$12K–28K** |
| 5-year | $65K–145K | $2.5K–5K | **$62K–142K** |

Plus: engineering time previously allocated to Luigi integration is redirected to shipping the controller.
