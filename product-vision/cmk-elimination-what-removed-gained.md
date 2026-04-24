# CMK Elimination: What's Removed, What's Gained

---

## What Gets Removed

| Component | What it did | Why it's gone |
| :--- | :--- | :--- |
| **CMK API Server** | Custom REST/gRPC interface for tenant and key operations | Platform Mesh and Kubernetes API replace it entirely |
| **CMK Registry Database** | Stored tenant metadata in PostgreSQL | Tenant CRDs in Kubernetes are the registry |
| **CMK Approval Engine** | Custom 4-eyes approval logic for key governance | Platform Mesh ApprovalPolicy does this natively |
| **CMK RBAC System** | Custom role definitions and access control | Kubernetes RBAC handles this |
| **CMK Audit Database** | Separate PostgreSQL for compliance logging | Platform Mesh audit trail is richer and already running |
| **CMK Tenant Manager** | Microservice managing tenant lifecycle | Replaced by a standard Kubernetes controller |
| **CMK Task Scheduler / Workers** | Background job execution for async operations | Kubernetes-native reconciliation loops replace this |
| **CMK UI → Luigi integration** | Portal wrapper for CMK governance UI | Platform Mesh portal already exposes this — work was never done and is now unnecessary |

**Net removal: 8 microservices, 2 databases, 15+ custom API endpoints, and an unbuilt UI integration.**

---

## What Gets Added

| Component | What it does |
| :--- | :--- |
| **CMK Controller** | A single lightweight Kubernetes controller that orchestrates L1 key lifecycle natively within Platform Mesh |
| **L1KeyValidation CRD** | Declarative health checks for L1 keys — replaces custom validation logic |
| **L1KeyRevocation CRD** | Explicit revocation workflow with multi-party approval and full audit trail built in |

---

## Before and After

| Aspect | Before (Legacy CMK) | After (CMK-as-Controller) |
| :--- | :--- | :--- |
| Services | 8 microservices | 1 controller |
| Databases | 2 (Registry + Audit) | 0 — state lives in Kubernetes |
| Approval engine | Custom, duplicates Platform Mesh | Platform Mesh ApprovalPolicy (native) |
| Audit trail | Separate database, manual export | Platform Mesh — integrated, immutable |
| UI | Requires Luigi integration (not built) | Platform Mesh portal (already exists) |
| Source of truth | PostgreSQL | Kubernetes CRDs |
| GitOps support | No | Yes |
| Annual infrastructure cost | $13K–29K | $0.5K–1K |

---

## What This Means for the Product

**Governance is native, not bolted on.** Tenant onboarding, key validation, and revocation are all first-class Platform Mesh operations — declarative, auditable, and consistent with how every other platform component works.

**Nothing is lost.** Every capability CMK provided is covered: approval workflows by ApprovalPolicy, audit by Platform Mesh trail, RBAC by Kubernetes native, tenant registry by CRDs.

**The integration debt stops here.** CMK was never fully integrated into Platform Mesh. Rather than completing an integration that would duplicate what already exists, we replace it with architecture that belongs there from the start.
