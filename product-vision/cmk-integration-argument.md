# CMK Layer Elimination: Architecture Argument

**Date:** 2026-04-24
**Purpose:** Structured argument for eliminating the standalone CMK service in favour of CMK-as-Controller (Platform Mesh native)
**Related:** [CMK Integration Index](CMK-INTEGRATION-INDEX.md) · [ADR-105](../adr/cmk-as-controller-platform-mesh-native.md) · [Executive Summary](cmk-integration-executive-summary.md)

---

## Core Position

The CMK layer became redundant the moment Platform Mesh matured. Every capability CMK provides — approval workflows, RBAC, audit trail, tenant registry — Platform Mesh already does natively. We have been running a parallel system that duplicates all of it.

---

## Argument 1: We Were Solving the Same Problem Twice

CMK maintains its own approval engine, its own RBAC, its own audit database, and its own tenant registry. Platform Mesh provides all of these too. Every time a tenant is onboarded or a key is revoked, the same operation runs in two places and must stay in sync. That synchronisation gap is where bugs live and where operational overhead compounds over time.

---

## Argument 2: The Separation of Concerns Is Cleaner Without CMK

With the CMK layer removed, every component owns a single, unambiguous responsibility:

| Layer | Responsibility |
|---|---|
| **Kubernetes CRDs** | Source of truth — tenant state, key references, revocations |
| **Platform Mesh** | Governance — approval policies, audit trail, RBAC |
| **Krypton** | Enforcement — crypto operations, L1 validation, revocation execution |

CMK sits awkwardly between these layers today, owning nothing cleanly and duplicating everything.

---

## Argument 3: Completing the CMK Integration Would Make Things Worse, Not Better

This is the point that makes elimination the only rational path. CMK is not currently integrated with Platform Mesh — and finishing that integration would require work that proves our own argument against it.

The showroom team built the most advanced integration attempt to date (`showroom-msp-openkcm`). What they produced makes the integration gap explicit:

**The controller talks to CMK via a custom REST client, not Kubernetes primitives.**
The tenant controller (`internal/controller/governance/tenant_controller.go`) calls `r.APIClient.CreateTenant()` — an HTTP call to the OpenKCM REST API. This is a bridge layer, not a native Platform Mesh integration. The controller is a wrapper around CMK's own API.

**A mock API had to be built because the real integration is not stable enough to test against.**
`internal/mockapi/server.go` exists specifically because the real Krypton/CMK API cannot be used directly in development or demo scenarios. The showroom requires a fake CMK service to demonstrate the flow at all.

**The UI is wired to Luigi and cannot function without it.**
`ui/src/App.tsx` calls `subscribeToLuigi()` on startup and fails with an explicit error if the Luigi portal has not injected its `crdGatewayApiUrl` context. To ship CMK as a properly integrated Platform Mesh service, the CMK UI must be wrapped as a Luigi micro-frontend — portal registration, context injection, navigation wiring, and authentication delegation. That is significant engineering investment to surface tenant onboarding and key governance capabilities that **Platform Mesh's UI already provides natively**. We would be building a Luigi wrapper for functionality that already exists in the portal.

---

## Argument 4: The Showroom Work Is Not Wasted — It Proves the Right Architecture

The controller pattern the showroom team built is correct. Kubernetes operator, CRD-driven reconciliation, multi-cluster manager — all of it stays. The only change is the direction of the dependency: instead of the controller calling CMK's REST API, it calls Krypton directly, with Platform Mesh as the governance layer.

The CRD types (`Tenant`, `L1KeyReference`), the reconciliation loop structure, and the multi-cluster manager setup are all promoted directly into the CMK-as-Controller design. The showroom work is the foundation, not a sunk cost.

---

## Argument 5: Concrete Cost — 8 Services to 1 Controller

### What is eliminated

| Component | Replaced by |
|---|---|
| CMK API Server (REST/gRPC) | Kubernetes API |
| CMK Registry Database (PostgreSQL) | Tenant CRD in etcd |
| CMK Approval Engine | Platform Mesh ApprovalPolicy |
| CMK RBAC System | Kubernetes RBAC |
| CMK Audit Database (PostgreSQL) | Platform Mesh audit trail |
| CMK Tenant Manager | CMK Controller reconciliation loop |
| CMK Task Scheduler | Kubernetes CronJobs |
| CMK Task Workers | Controller reconciliation loops |
| CMK UI → Luigi integration work | Platform Mesh portal (already exists) |

### Cost impact

| Year | CMK cost | Controller cost | Savings |
|---|---|---|---|
| 1 | $13K–29K | $0.5K–1K | **$12K–28K** |
| 2 | $13K–29K | $0.5K–1K | **$12K–28K** |
| 3 | $13K–29K | $0.5K–1K | **$12K–28K** |
| 5-year total | $65K–145K | $2.5K–5K | **$62K–142K** |

Additional savings: engineering time that would have gone into the Luigi integration is redirected to shipping the controller.

---

## Addressing Pushback

| Pushback | Response |
|---|---|
| "What CMK features would we lose?" | Every CMK feature is covered: approval by ApprovalPolicy, audit by Platform Mesh trail, RBAC by Kubernetes native. Nothing is lost. |
| "Is this risky given the showroom work?" | The showroom talks to a mock API — there is no live traffic to migrate. We are replacing an incomplete integration before it ships, not after. |
| "Why not just finish the Luigi integration?" | We would be building a portal wrapper for capabilities the portal already has. That is the definition of redundant work. |
| "Why not CMK v2, audit-only?" | An audit-only CMK still requires its own database and sync logic. Full elimination via the controller is simpler than a slimmed-down version. We evaluated this — see [ADR CMK v2](../adr/cmk-v2-lean-audit-compliance-layer.md). |

---

## Summary

| Aspect | Legacy CMK | CMK-as-Controller |
|---|---|---|
| Services | 8 microservices | 1 controller |
| Databases | 2 (Registry + Audit) | 0 (state in etcd) |
| API endpoints | 15+ custom | 0 (Kubernetes API) |
| Approval engine | Custom, duplicates Platform Mesh | Platform Mesh ApprovalPolicy |
| Audit trail | Separate PostgreSQL | Platform Mesh integrated |
| UI integration | Luigi wrapper (not yet built) | Platform Mesh portal (exists) |
| Source of truth | PostgreSQL | Kubernetes CRDs |
| GitOps support | No | Yes |
| Annual cost | $13K–29K | $0.5K–1K |

Eliminating CMK produces a leaner, more maintainable architecture where every layer owns exactly one responsibility, governance is native to the platform, and the engineering investment goes into shipping rather than duplicating what already exists.
