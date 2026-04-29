# CMK Integration: Executive Summary

**Date:** 2026-04-22
**Decision:** Consolidate legacy CMK service into Platform Mesh as CMK-as-Controller
**Replacement:** CMK-as-Controller (Platform Mesh native)

---

## The Insight

You asked: "Can we do this without CMK at all? The requirements should be possible via CMK OpenKCM solution—something like a controller responsible for onboarding L1 to Platform Mesh."

**Answer:** Yes. And it's simpler than any intermediate solution.

The standalone CMK service is **redundant overhead**. Platform Mesh already provides:

- **Policy engine** (ApprovalPolicy CRDs)
- **RBAC integration** (Kubernetes native)
- **Audit trail** (immutable event system)
- **Workflow execution** (operator pattern)

The missing piece was just a **controller** to orchestrate L1 key onboarding. That's exactly what we're building.

---

## What Changes

### Before: Legacy CMK (8 services)

```
CMK API Server (REST/gRPC)
  ↓ reads/writes
CMK Registry DB (PostgreSQL) [Tenants, Approvals, Policies]
  ↓ syncs to
Kubernetes API [Tenants CRD]
  ↓ publishes
Orbital Mesh
  ↓ executes
Krypton
```

**Problems:**

- Tenant state exists in 2 places (CMK DB + Kubernetes)
- Approvals handled twice (CMK approval engine + Platform Mesh)
- Audit trail fragmented (CMK audit DB + Kubernetes events)
- 8 services + 2 databases to operate

### After: CMK-as-Controller (1 service)

```
User applies L1KeyRevocation CRD to Kubernetes API
  ↓
Kubernetes API server (stores in etcd)
  ↓
CMK Controller watches (reconciliation loop)
  ↓
Controller:
  • Checks Platform Mesh ApprovalPolicy
  • Calls Krypton to validate/revoke
  • Records audit event
  • Updates CRD status
  ↓
Done (all state in Kubernetes)
```

**Benefits:**

- **Single source of truth:** Kubernetes CRDs
- **No separate databases:** All in etcd
- **Native approval:** Platform Mesh handles workflows
- **Integrated audit:** Platform Mesh audit trail
- **Single service:** 1 controller (minimal, standard K8s)

---

## What Transitions into Platform Mesh

| Component                 | Reason                                   |
| :------------------------ | :--------------------------------------- |
| **CMK API Server**        | All operations via Kubernetes API        |
| **CMK Registry Database** | Tenant CRD is the registry               |
| **CMK Approval Engine**   | Platform Mesh ApprovalPolicy replaces    |
| **CMK RBAC System**       | Kubernetes RBAC + Platform Mesh policies |
| **CMK Audit Database**    | Platform Mesh audit trail used           |
| **CMK Tenant Manager**    | Kubernetes controller handles            |
| **CMK Task Scheduler**    | Kubernetes CronJobs + controllers        |
| **CMK Task Workers**      | Reconciliation loops replace             |

**Result:** 8 microservices → 1 controller

---

## What Gets Added

| Component               | Purpose                                           |
| :---------------------- | :------------------------------------------------ |
| **CMK Controller**      | Orchestrates L1 lifecycle (runs in Platform Mesh) |
| **L1KeyValidation CRD** | Declarative L1 key health checks                  |
| **L1KeyRevocation CRD** | Multi-party approval workflow with audit          |

---

## Three Layers (Simplified Model)

```
┌────────────────────────────────────────────────────┐
│ Kubernetes API (Source of Truth)                   │
│ • Tenant CRD                                       │
│ • L1KeyReference CRD                               │
│ • L1KeyValidation CRD (NEW)                        │
│ • L1KeyRevocation CRD (NEW)                        │
│ • All versioned & auditable                        │
└────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────┐
│ Platform Mesh (Governance)                         │
│ • ApprovalPolicy (multi-party workflows)           │
│ • Audit Trail (immutable events)                   │
│ • RBAC (Kubernetes native)                         │
│ • CMK Controller (reconciliation loop)             │
└────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────┐
│ Krypton (Enforcement)                              │
│ • Validates L1 key accessibility                   │
│ • Generates/wraps L2-L4 keys                       │
│ • Executes revocation commands                     │
│ • Via Orbital event mesh                           │
└────────────────────────────────────────────────────┘
```

---

## Workflow: L1 Key Lifecycle

### Onboarding Tenant with L1 Key

```yaml
# 1. User applies Tenant CRD
apiVersion: governance.openkcm.io/v1alpha1
kind: Tenant
metadata:
  name: acme-corp
spec:
  l1KeyReference:
    backend: aws_kms
    awsConfig:
      keyId: "arn:aws:kms:us-west-2:123456:key/abc"

# 2. CMK Controller (auto):
#    - Creates L1KeyValidation CRD
#    - Calls Krypton to validate key
#    - Records audit event: "L1_VALIDATED"
#    - Sets .status.ready = true

kubectl get tenant acme-corp -o jsonpath='{.status}'
# { "ready": true, "l1KeyValidated": true, ... }
```

### Revoking Tenant

```yaml
# 1. User requests revocation
apiVersion: governance.platform.mesh/v1alpha1
kind: L1KeyRevocation
metadata:
  name: acme-corp-revoke
spec:
  tenantID: acme-corp
  reason: "Customer request"

# 2. Platform Mesh ApprovalPolicy (auto):
#    - Requires 2 approvals (security-admin + compliance-officer)
#    - Creates approval notifications
#    - Blocks until approved

# 3. Approvers approve (via kubectl or GitOps)

# 4. CMK Controller (auto):
#    - Detects both approvals
#    - Calls Krypton to revoke L1 key
#    - Records audit: "L1_REVOKED"
#    - Sets .status.state = PROPAGATED

kubectl get l1keyrevocation acme-corp-revoke -o jsonpath='{.status}'
# { "state": "PROPAGATED", "approvals": [...] }
```

---

## Cost Reduction

| Component                             | Annual Cost       | Status          |
| :------------------------------------ | :---------------- | :-------------- |
| CMK API Server                        | $5K-10K           | REMOVE          |
| CMK Registry DB                       | $2K-5K            | REMOVE          |
| CMK Audit DB                          | $2K-5K            | REMOVE          |
| CMK Services (task scheduler/workers) | $4K-9K            | REMOVE          |
| **Total Removed**                     | **$13K-29K**      | **CONSOLIDATE** |
| CMK Controller (new)                  | $0.5K-1K          | ADD             |
| **Net Savings**                       | **$12K-28K/year** | **REDUCE COST** |

Plus: **Reduced operational overhead** (no separate database operations, monitoring, backups)

---

## Implementation Timeline

| Phase                          | Timeline          | Deliverables                                |
| :----------------------------- | :---------------- | :------------------------------------------ |
| **Design**                     | Done (2026-04-22) | ADR-105, implementation guides              |
| **Phase 1: Deploy Controller** | Weeks 1-2         | CMK Controller running alongside legacy CMK |
| **Phase 2: Migrate Tenants**   | Weeks 3-6         | 100% of tenants on CRD-based model          |
| **Phase 3: Retire Legacy CMK** | Weeks 7-12        | Decommission CMK services & databases       |
| **Complete**                   | End of Week 12    | Platform Mesh-native governance             |

---

## Architecture Comparison

| Aspect                     | Legacy CMK            | CMK-as-Controller        |
| :------------------------- | :-------------------- | :----------------------- |
| **Services**               | 8 microservices       | 1 controller             |
| **Databases**              | 2 (Registry + Audit)  | 0 (all in Kubernetes)    |
| **API Endpoints**          | 15+ custom endpoints  | 0 (use Kubernetes API)   |
| **Approval Engine**        | Custom implementation | Platform Mesh native     |
| **Audit Trail**            | Separate DB           | Platform Mesh integrated |
| **RBAC**                   | Custom role system    | Kubernetes RBAC          |
| **Source of Truth**        | PostgreSQL            | Kubernetes CRDs          |
| **GitOps Support**         | No                    | Yes                      |
| **Operational Complexity** | High                  | Low                      |
| **Annual Cost**            | $13K-29K baseline     | $0.5K-1K baseline        |

---

## Key Design Decisions

### Decision 1: Consolidate CMK into Platform Mesh

Instead of "CMK v2" (audit-only), build CMK-as-Controller (fully operational).

**Rationale:**

- Audit-only still requires database & sync logic
- Controller-based is simpler (uses existing Platform Mesh infrastructure)
- Kubernetes-native is more maintainable
- Future-proof: scales with Platform Mesh

### Decision 2: Use Platform Mesh ApprovalPolicy

Don't build custom approval workflows in controller.

**Rationale:**

- Platform Mesh already has robust policy engine
- Reusable across all governance operations
- Integrates with RBAC & audit trail
- Declarative & auditable

### Decision 3: Krypton Owns L1 Validation

Controller orchestrates, Krypton validates.

**Rationale:**

- Krypton has client libraries for all backends (AWS, GCP, Azure, HSM, Vault)
- Validation happens at crypto layer (where it belongs)
- Controller stays lightweight
- Separation of concerns

---

## Next Steps

1. **Review ADR-105** – CMK-as-Controller architecture document
2. **Review CMK Controller Implementation Guide** – Code structure and Go examples
3. **Approve timeline** – Weeks 1-12 migration schedule
4. **Kick off Phase 1** – Deploy CMK Controller alongside legacy CMK
5. **Monitor parallel operation** – Verify CRD reconciliation works
6. **Migrate first batch** – Test with 10 non-critical tenants
7. **Expand migration** – Roll out to all tenants
8. **Retire CMK** – Decommission legacy services

---

## FAQ

**Q: Do we lose any compliance features?**
A: No. Platform Mesh audit trail is richer than CMK audit DB. All events are immutable & signed.

**Q: What about existing CMK integrations?**
A: Krypton already integrates with Platform Mesh. No external integrations on CMK service layer.

**Q: Is this the final architecture?**
A: Yes. Kubernetes-native governance is the target end state. CMK-as-Controller achieves it.

**Q: Can we go back to CMK if needed?**
A: Yes, until Week 12. After that, legacy CMK is retired.

---

## Summary

By adopting **CMK-as-Controller**, OpenKCM achieves:

✅ **Simpler architecture** – 1 controller instead of 8 services
✅ **Single source of truth** – Kubernetes CRDs
✅ **Native governance** – Platform Mesh integrates all policy & audit
✅ **Cost savings** – $12K-28K/year + operational reduction
✅ **Future-proof** – Fully Kubernetes-native, scales with platform

The result is a **lean, maintainable, production-grade governance system** where OpenKCM operates as a first-class Platform Mesh service, not a layered external system.
