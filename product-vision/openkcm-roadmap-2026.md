---
authors:
  - Aysan
status: Draft
last_updated: 2026-04-29
---

# OpenKCM Roadmap 2026 — Revised

This roadmap reflects the updated product strategy announced April 2026: **the standalone CMK service will not be developed further. CMK governance moves natively into Platform Mesh as the CMK Controller.** Krypton development continues unchanged.

---

## Strategy Change Summary

| Before | After |
| :--- | :--- |
| Standalone CMK service (8 microservices) | CMK Controller (1 Kubernetes controller) |
| CMK UI as standalone portal with Luigi integration | CMK MicroFrontend natively in Platform Mesh portal |
| CMK ↔ Krypton integration via custom API bridge | CMK Controller calls Krypton API directly |
| PostgreSQL for tenant registry and audit | Kubernetes CRDs (etcd) + Platform Mesh audit trail |

Krypton development — KMIP server, MasterKey management, KMS backend plugins, IVK system — is unaffected by this change. The crypto execution layer stays as planned.

---

## Timeline Overview

```
2026
│
├─ Q1 (Jan-Mar) ──── ✅ INVESTIGATION & DESIGN (COMPLETE)
│   ├── ✅ Low-Level Design (LLD) for Krypton
│   ├── ✅ KMIP Protocol POC
│   ├── ✅ Architecture decisions finalized (ADR-101 through ADR-604)
│   ├── ✅ CMK Layer v0.1.0 → v0.7.0 (16 releases)
│   ├── ✅ Orbital v0.4.0 → v0.5.1
│   └── ✅ Plugin architecture redesigned (gRPC → Go interfaces)
│
├─ Q2 (Apr-Jun) ──── 🔄 KRYPTON CORE + CMK CONTROLLER FOUNDATION
│   │
│   ├── Krypton Core
│   │   ├── KMIP Server implementation
│   │   ├── KeyChain & Key lifecycle (L2-L4)
│   │   ├── MasterKey management (Seal + Shamir SSS)
│   │   ├── Static MasterKey provider
│   │   ├── CLI tool (krypton command)
│   │   └── AWS KMS plugin (Seal provider)
│   │
│   └── CMK Controller (replaces standalone CMK)
│       ├── Harden showroom controller (crash recovery, retry, idempotency)
│       ├── Freeze Krypton API contract
│       ├── Tenant CRD reconciliation (auto-provision on account creation)
│       ├── L1 key validation via Krypton
│       └── Showroom demo with MongoDB (Tariq & Priya story)
│
├─ Q3 (Jul-Sep) ──── 🎯 CRYPTO LAYER MVP + CMK MICROFRONTEND
│   │
│   ├── Krypton
│   │   ├── KMIP + crypto ops production-ready
│   │   ├── OpenBao keystore plugin
│   │   └── IVK rotation (automatic L2.x rotation)
│   │
│   └── CMK MicroFrontend (Platform Mesh native, Luigi)
│       ├── Key Chain view (L1 → L3 ServiceKeys)
│       ├── L1 declaration — BYOK / HYOK onboarding
│       ├── Kill switch — state transition with Four-Eyes approval
│       └── Approvals view (pending / approve / reject)
│
├─ Q4 (Oct-Dec) ──── CMK CONTROLLER + KRYPTON FULL INTEGRATION & HARDENING
│   │
│   ├── Krypton
│   │   ├── Seal / Auto-Unseal (AWS, Azure, GCP, HSM)
│   │   ├── HA & disaster recovery
│   │   ├── mTLS authentication
│   │   ├── MongoDB KMIP integration validated
│   │   └── Multi-tenant key isolation production-ready
│   │
│   └── CMK Controller
│       ├── L1 (CMK Controller) ↔ L2-L4 (Krypton) full integration
│       ├── Kill switch — global propagation within 2 minutes (REQ-022)
│       ├── Key suspension + grace period enforcement
│       ├── Tenant termination with grace period
│       ├── Four-Eyes approval via Platform Mesh (platform-mesh/backlog#267)
│       ├── Access control view in CMK MicroFrontend
│       ├── Audit trail integration
│       └── Production-ready deployment
```

---

## Milestones

| Milestone | Target | Description | Status |
| :--- | :--- | :--- | :--- |
| 🔬 LLD Complete | Mar 2026 | Low-Level Design finalized | ✅ Done |
| 📋 Strategy Pivot | Apr 2026 | CMK standalone stopped, CMK Controller announced | ✅ Done |
| 🚀 Showroom Demo | Jun 2026 | CMK Controller + Krypton + MongoDB on Platform Mesh | 🔄 In Progress |
| 🎯 Crypto Layer MVP | Aug 2026 | Production-ready KMIP server with multi-tenancy | Planned |
| 🖥️ CMK MicroFrontend | Sep 2026 | Key Chain UI live in Platform Mesh portal | Planned |
| 🔗 Full Chain (L1-L4) | Nov 2026 | CMK Controller + Krypton integrated end-to-end | Planned |

---

## What Is No Longer Being Built

The following CMK standalone work is stopped. Engineers previously working on these items should redirect to CMK Controller and CMK MicroFrontend.

| Stopped | Replaced by |
| :--- | :--- |
| CMK standalone service development | CMK Controller |
| CMK UI Luigi integration (standalone) | CMK MicroFrontend (Platform Mesh native) |
| CMK PostgreSQL registry & audit DB | Kubernetes CRDs + Platform Mesh audit trail |
| CMK custom approval engine | Platform Mesh approval mechanism |
| CMK custom RBAC | Kubernetes RBAC |

---

## Krypton: Unchanged Work (Q2–Q4)

Krypton development continues as planned. No changes.

### Q2 — MasterKey: Seal Providers (Epic #26)

| # | Title |
| --- | --- |
| 34 | OpenBao/Vault provider |
| 35 | AWS KMS provider |
| 36 | GCP KMS provider |
| 37 | Azure Key Vault provider |
| 38 | Common Seal utilities & patterns |
| 39 | Configuration parsing & provider factory |
| 40 | IVK layer integration & algorithm agnosticism |
| 41 | Security hardening & zeroization |
| 42 | Comprehensive testing suite |

### Q2–Q3 — MasterKey: Shamir SSS (Epic #27)

| # | Title |
| --- | --- |
| 44 | SSS Generator CLI Tool |
| 45 | Shard Encryption Workflow (KMS/HSM Integration) |
| 49 | AWS KMS Decryption Client |
| 50 | GCP KMS Decryption Client |
| 51 | HashiCorp/OpenBao Vault Decryption Client |
| 52 | HSM (PKCS#11) Decryption Client |
| 53 | mTLS-backed REST Decryption Client |

### Q2 — KMIP & Crypto Core (Epic #61)

| # | Title |
| --- | --- |
| 61 | Crypto Core & Edge Services Using KMIP 1.4 |
| 23 | Internal Versioned Key (IVK) Management |
| 60 | IVK Management (Per-Version Algorithm & Automatic L2.x Rotation) |

---

## CMK Controller: New Work (Q2–Q4)

### Q2 — Foundation

- Harden showroom controller for production (crash recovery, retry logic, idempotency, observability)
- Freeze Krypton API contract — controller calls Krypton directly, no mock
- Tenant CRD reconciliation — auto-provision on Platform Mesh account creation (fulfills REQ-001, REQ-002)
- L1 key validation flow — controller calls Krypton to validate customer key reference
- Showroom demo app — end-user app querying MongoDB (demo story: Priya + Tariq)

### Q3 — CMK MicroFrontend

- Key Chain view — L1 → L3 ServiceKeys in one screen
- BYOK / HYOK onboarding — customer declares L1 key reference
- Kill switch — approved state transition (Active → Deactivated)
- Approvals view — Four-Eyes approval via Platform Mesh (#267)

### Q4 — Full Integration & Hardening

- L1 revocation → Krypton broadcast → global propagation ≤ 2 minutes (REQ-022)
- Key suspension with grace period (30 / 60 / 90 days configurable)
- Tenant termination with grace period
- Access control view in CMK MicroFrontend
- Audit trail integration with Platform Mesh
- End-to-end test suite
- Production-ready deployment
