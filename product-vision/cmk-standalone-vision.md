---
status: Draft
last_updated: 2026-06-24
audience: Open Source Community, Contributors, Stakeholders
---

# OpenKCM CMK — Standalone

## Overview

OpenKCM CMK is the full-featured, standalone Customer Managed Key product. It runs on any Kubernetes environment with no platform dependency. Designed for organizations that operate their own infrastructure and need complete key governance under their own control.

This is an open source product. The customer — whether a government authority, a large enterprise, or a managed service provider — downloads, deploys, and operates it themselves. There is no vendor running it on their behalf, no managed service, no external dependency. The customer owns the deployment end to end.

**Target deployment:** Any Kubernetes cluster the customer operates themselves — on-premise, sovereign cloud, air-gapped, or any cloud provider.

---

## Key capabilities

- L1 Root Key registration and lifecycle management (enable, disable, rotate, delete)
- BYOK (Bring Your Own Key) and HYOK (Hold Your Own Key) — customer key material never touches the platform
- Pluggable keystore backends: OpenBao, AWS KMS, Azure Key Vault, GCP KMS, HSM via PKCS#11
- Kill switch (Red Button) — instant revocation of all workloads across the deployment
- Multi-party approval (Four-Eyes) — high-stakes operations require a second authorized approver
- Full audit trail — tamper-evident log of every key operation, exportable for compliance
- Tenant onboarding and offboarding
- Region and tenant management — region → tenant → key hierarchy ❓ — not confirmed
- Role-based access control via OIDC / RBAC
- Full CMK UI — own web interface, not embedded in any platform portal

---

## Key hierarchy

Krypton (crypto layer) knows nothing by default. Every key at every level must be explicitly registered via the Krypton API. The CMK UI is the management surface for the full key hierarchy — there is no platform controller to automate any part of this.

| Level | Name | Scope | Managed by |
|---|---|---|---|
| L1 | Root Key | Tenant | Customer — registered via OpenKCM CMK UI |
| L2 | Domain Key | Domain | Customer via OpenKCM CMK UI ❓ — in Platform Mesh this is auto-provisioned by CMK Controller; in standalone there is no controller, so manual creation is assumed but not confirmed |
| L3 | Service Key | Service | Customer via OpenKCM CMK UI ❓ — assumed manual creation by security admin, but whether this is via UI, CLI, or API is not confirmed |
| L4 | Data Encryption Key | Workload | Consuming service via Krypton Gateway KMIP (when KMIP-capable) ❓ — or customer via UI when no KMIP service present; both paths assumed, neither confirmed as built |

> **Open question:** L2, L3, and L4 management in standalone is not yet validated with Krypton or engineering. The table above reflects the current assumption — all levels are manually managed by the customer via UI when no consuming service handles it via KMIP. This needs to be confirmed before any UI work starts.

The standalone UI is expected to support all four key levels (L1–L4) ❓ — the full scope of the standalone UI is not yet confirmed. Unlike Platform Mesh — where L2 is provisioned automatically and L4 is handled by consuming services via KMIP — standalone cannot make either assumption. The customer may be the only actor managing the full key chain.

### L4 — two modes

When a KMIP-capable consuming service is present (e.g. MongoDB, PostgreSQL), it manages L4 directly via the Krypton Gateway KMIP interface:
- **Option A:** the consuming service provides its own DEKs; Krypton wraps them under L3 and returns the wrapped blob. The consuming service owns and stores the wrapped DEK.
- **Option B:** the consuming service requests Krypton to create, manage, and store the DEKs entirely.

When no KMIP-capable consuming service is present, the customer manages L4 through the CMK UI. The UI registers the DEK, wraps it under the appropriate L3 key, and the key material can be injected into workloads via Kubernetes Secrets.

---

## Region and tenant model

The standalone deployment is organized as:

```
Region
└── Tenant
        └── L1 Root Key
                └── L2 Domain Key (one per domain boundary)
                        └── L3 Service Key (one per service)
                                └── L4 Data Encryption Key (one per workload)
```

A default region and default tenant are provisioned at deployment time so the customer has a starting point ❓ — whether this auto-provisioning is actually implemented or just assumed needs confirmation. The customer then creates their own domain boundaries (L2) and service keys (L3) according to their own business structure — the platform makes no assumptions about domain organization ❓ — this flexibility is assumed but the actual onboarding flow is not defined.

Multiple tenants within a region are supported ❓ — not confirmed. Multiple regions are supported for multi-region sovereign deployments ❓ — not confirmed.

> **Open question:** The region and tenant model described here is a proposal, not a confirmed design. Default provisioning, multi-tenant support, and multi-region support all need to be validated with engineering before being treated as committed scope.

---

## What the security admin manages

The security admin is the single operator of the CMK UI. The responsibilities below reflect the current assumption — not all of these are confirmed as built or in scope.

- Register one or more L1 root keys (pointing to external keystores — no key material imported)
- Define domain boundaries and create L2 domain keys ❓
- Define service keys (L3) within each domain ❓
- Register L4 DEKs for workloads where no consuming service handles this via KMIP ❓
- Bind L1 → L2 to bring domains under customer-managed encryption ❓
- Trigger the kill switch at tenant level when required
- Manage tenant onboarding and offboarding ❓
- Manage regions if running a multi-region deployment ❓

---

## Kill switch

Triggering the kill switch at tenant level revokes the L1 root key. Since all L2, L3, and L4 keys are derived from L1, every encrypted workload in the tenant becomes inaccessible within the propagation window.

This is irreversible without a new key registration and approval cycle. Multi-party approval (Four-Eyes) is required before the kill switch executes.

---

## Deployment scenario

```
Sovereign / air-gapped deployment
│
├── OpenKCM CMK (standalone)
│       ├── CMK UI  ← full web interface, operator-owned
│       ├── CMK Application
│       └── CMK Core
│
└── Krypton
        ├── Krypton Core (gRPC API — used by CMK for L1/L2/L3 management)
        └── Krypton Gateway (KMIP — used by consuming services for L4)
```

The customer operates both OpenKCM CMK and Krypton. No dependency on any external platform or managed service.

---

## Sovereign cloud use case

A sovereign cloud customer — a government authority, defense agency, or regulated enterprise — controls their own infrastructure entirely. They decide where deployments live, what crosses borders, and who has physical access. No cloud provider, no managed service, no external dependency is acceptable.

### Target customers

**National defense and intelligence agencies**
Organizations where data sovereignty is a legal and operational non-negotiable. Key material must remain within national borders, under direct physical control. Any third-party dependency in the encryption path is unacceptable.

**Government authorities and public sector**
Ministries, regulatory bodies, and public administration operating under data residency laws (e.g. GDPR, national cybersecurity frameworks). Required to demonstrate cryptographic control over citizen and state data.

**Critical infrastructure operators**
Energy grids, water systems, transportation networks, telecommunications. Subject to NIS2, KRITIS, or equivalent national frameworks requiring strong data protection and operational independence from foreign cloud providers.

**Regulated financial institutions**
Banks, insurers, and financial market infrastructures operating under strict data residency and auditability requirements. Need to demonstrate to regulators that encryption keys are under their exclusive control.

**Managed service providers offering sovereign cloud**
Companies building sovereign cloud platforms for the above customers. They need CMK as a first-class capability to offer their own customers the same guarantees — your data is encrypted under your key, not ours.

---

**Deployment model:**

The customer deploys Krypton in each of their own data centers. They deploy OpenKCM CMK once and connect it to all their Krypton instances. All key management happens from one UI — but key material never leaves the data center it was created in.

```
OpenKCM CMK (one instance — e.g. central data center)
    │
    ├── Connected to: Krypton — Frankfurt data center  (EU keys stay here)
    └── Connected to: Krypton — Virginia data center   (US keys stay here)
```

The customer registers their Krypton deployments in OpenKCM CMK once. From that point, every time they create a domain key or service key, they select which deployment it lives on. OpenKCM CMK talks to the right Krypton instance. The key material never moves.

**Why this model satisfies sovereign cloud requirements:**

- Key material stays physically in the customer's own server room — never crosses to a third party
- No cloud provider dependency — no AWS, no Azure, no GCP in the data path
- No routing layer deciding where keys live — the customer decides explicitly, at creation time
- One UI, one audit trail, one kill switch — across all deployments
- Kill switch is immediate — no mandatory delays, no platform approval required

**What the admin does:**

1. Deploys Krypton in each data center
2. Registers each Krypton deployment in OpenKCM CMK (name + endpoint)
3. Registers L1 root key (pointing to their HSM or OpenBao instance in that data center)
4. Creates domain keys (L2) — selecting which deployment each domain lives on
5. Creates service keys (L3) within each domain
6. From that point — workloads in that domain are encrypted, kill switch is armed

**Multi-region key visibility:**

The admin sees all keys across all connected deployments in one place. The region is just a label for which Krypton endpoint a key lives on — not a routing abstraction, not a tenant boundary. The customer defined those regions themselves by choosing where to deploy Krypton.

---

## Enterprise use case

A large enterprise — a manufacturer, financial institution, or technology company — typically runs workloads across multiple environments: on-premise data centers, one or more cloud providers, and potentially air-gapped systems. They need encryption key governance across all of these from one place, without handing control to any single vendor.

OpenKCM CMK is open source. The enterprise downloads it, deploys it on their own infrastructure, and operates it themselves. No vendor has access to their deployment. No managed service is involved. The enterprise's own security team owns the full stack.

### Target customers

**Large enterprises running hybrid infrastructure**
Organizations running workloads across on-premise, cloud, and air-gapped environments simultaneously. They need one key management control plane that spans all of it — without being locked into a single cloud provider's KMS.

**Enterprises with strict compliance requirements**
Organizations subject to ISO 27001, SOC 2, PCI-DSS, HIPAA, or similar frameworks that require demonstrable control over encryption keys, a tamper-evident audit trail, and the ability to revoke access immediately.

**Organizations replacing fragmented KMS setups**
Enterprises currently managing keys separately in AWS KMS, Azure Key Vault, and on-premise HSMs — with no unified view, no single audit trail, and no single kill switch. OpenKCM CMK connects to all of them from one UI.

**Teams building internal platforms**
Platform engineering teams within large organizations who want to offer encryption key governance as a self-service capability to internal development teams — without building key management from scratch.

### Why self-deployment matters

The enterprise deploys and operates OpenKCM CMK themselves. This means:

- **No vendor lock-in** — the software is open source; the enterprise can fork it, extend it, or replace it
- **No external access** — no vendor has visibility into the enterprise's key operations or audit trail
- **Full control over upgrades** — the enterprise decides when to upgrade, not a managed service provider
- **Runs anywhere** — on their own Kubernetes cluster, on any cloud, or air-gapped; no infrastructure dependency

