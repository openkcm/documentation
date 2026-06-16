---
status: Draft
last_updated: 2026-06-16
audience: Open Source Community, Contributors, Stakeholders
---

# OpenKCM CMK Platform Mesh — Tenancy Model

## Overview

This document defines the tenancy model for OpenKCM CMK Platform Mesh. It describes the entities, their relationships, and the key binding model that governs how customer-owned encryption keys are applied across a Platform Mesh deployment.

---

## Entity Relationship Model

```
+------------------+
|     Account      |
+------------------+
| AccountID        |
| Name             |
| Status           |
+------------------+
        | 1
        |
        | 1
+------------------+
|    Workspace     |
+------------------+
| WorkspaceID      |
| AccountID        |
| Status           |
+------------------+
        | 1
        |
        | N
+------------------+
|    Namespace     |
+------------------+
| NamespaceID      |
| WorkspaceID      |
| Name             |
| Status           |
+------------------+
        | 1
        |
        | 1
+----------------------+
|   L2 Domain Key      |
+----------------------+
| L2KeyID              |
| NamespaceID          |
| Status               |
| ProvisionedBy        | ← Platform Mesh (automatic)
+----------------------+
        | 1
        |
        | 1 (key binding — admin assigns)
+----------------------+
|   L1 Root Key        |
+----------------------+
| L1KeyID              |
| AccountID            |
| Name                 |
| Status               |
| KeystoreType         | ← OpenBao, AWS KMS, Azure KV, HSM...
| KeystoreRef          | ← reference only, no key material stored
| RegisteredBy         | ← Security Admin
+----------------------+
        | 1
        |
        | N
+----------------------+
|  External Keystore   |
+----------------------+
| KeystoreID           |
| Type                 |
| URL                  |
| Status               |
+----------------------+


+------------------+         +------------------+
|    Namespace     |         |   L2 Domain Key  |
+------------------+         +------------------+
        | 1                          | 1
        |                            |
        | N                          | N
+------------------+         +------------------+
|    Service       |         |   Key Binding    |
+------------------+         +------------------+
| ServiceID        |         | BindingID        |
| NamespaceID      |         | L2KeyID          |
| Name             |         | L1KeyID          |
| Status           |         | BoundBy          | ← Security Admin
+------------------+         | BoundAt          |
        | 1                  +------------------+
        |
        | 1
+------------------+
|   L3 Service Key |
+------------------+
| L3KeyID          |
| ServiceID        |
| L2KeyID          |
| Status           |
| GeneratedBy      | ← Krypton (automatic)
+------------------+


+------------------+
|  Security Admin  |
+------------------+
| UserID           |
| AccountID        |
| Role             |
+------------------+
        |
        | registers L1 keys
        | binds L1 → L2
        | triggers kill switch
```

---

## Entities

### Account
The top-level administrative boundary. Represents one customer organization. One account has exactly one workspace. The account is the tenant — pressing the kill switch at account level locks the entire account.

### Security Admin
A user with account-level permissions in Platform Mesh. Responsible for registering L1 root keys, binding them to L2 domain keys, and triggering the kill switch. The security admin operates from the account-level OpenKCM CMK UI.

### CMK Controller
A Kubernetes controller running in the platform-admin namespace of the workspace. Watches OpenKCM CRDs and reconciles key state across all namespaces. One controller per Platform Mesh installation — shared across all accounts.

### Workspace
The deployment environment within an account. One account has one workspace. Contains multiple namespaces where applications and services run.

### Namespace
A Kubernetes namespace within the workspace. Each namespace has exactly one L2 Domain Key, provisioned automatically by Platform Mesh when OpenKCM CMK Platform Mesh is enabled.

### L1 Root Key
A customer-owned root encryption key. Registered by the security admin at account level. Key material never leaves the customer's external keystore — OpenKCM CMK holds only a reference. One account can have multiple L1 root keys. Each L1 key is backed by one external keystore.

### External Keystore
The customer's own key management system where L1 key material lives. Supported backends: OpenBao, AWS KMS, Azure Key Vault, GCP KMS, HSM via PKCS#11. OpenKCM CMK never stores key material — it holds a pointer and calls the keystore at runtime via Krypton.

### Key Binding
The association between an L1 Root Key and an L2 Domain Key. Created by the security admin. One L2 Domain Key is bound to exactly one L1 Root Key. One L1 Root Key can be bound to multiple L2 Domain Keys (multiple namespaces can share the same L1 key). Different namespaces can be bound to different L1 keys — the admin decides.

### L2 Domain Key
Provisioned automatically by Platform Mesh when OpenKCM CMK Platform Mesh is enabled — one per namespace. The security admin cannot create or delete L2 keys. The admin can bind an L1 key to an L2 key. All services in a namespace are encrypted under that namespace's L2 key.

### L3 Service Key
Generated automatically by Krypton when a service is deployed in a namespace. Wrapped under the L2 Domain Key of that namespace. One L3 key per service instance. Not created or managed by the security admin.

### Service
An application or workload running inside a namespace (e.g. MongoDB, Postgres, Redis). Has one L3 Service Key. Encrypted automatically — no key configuration required from the developer.

---

## Key Binding Rules

| Rule | Description |
|---|---|
| One L2 per namespace | Platform Mesh provisions exactly one L2 Domain Key per namespace |
| Admin registers L1 | The security admin registers one or more L1 keys at account level |
| Admin binds L1 → L2 | The security admin decides which L1 key governs which namespace |
| One L1 per L2 | Each L2 Domain Key is bound to exactly one L1 Root Key at a time |
| One L1 → many L2s | One L1 Root Key can be bound to multiple namespaces |
| Different L1s per namespace | Different namespaces can be governed by different L1 keys |
| Unbound namespace | A namespace with no L1 binding is not customer-governed — it uses platform-managed encryption until the admin binds an L1 key |

---

## Kill Switch Scope

| Action | Scope | Effect |
|---|---|---|
| Revoke L1 Root Key | All namespaces bound to that L1 | All services in those namespaces become inaccessible |
| Kill switch (account level) | Entire account | All namespaces, all services inaccessible — regardless of which L1 they are bound to |

If an account has multiple L1 keys bound to different namespaces, revoking one L1 key affects only the namespaces bound to it. The full account kill switch revokes everything.

---

## Lifecycle States

### L1 Root Key lifecycle

```
Registered → Active → Suspended → Revoked
                ↑          │
                └──────────┘ (re-activated by admin)
```

- **Registered** — key reference created in OpenKCM CMK, not yet validated
- **Active** — key reachable and validated by Krypton; workloads are encrypted
- **Suspended** — key temporarily unavailable; grace period begins; workloads inaccessible after grace period expires
- **Revoked** — kill switch executed; all bound workloads inaccessible; irreversible without new key registration and approval

### L2 Domain Key lifecycle

Managed by Platform Mesh and OpenKCM CMK Controller. Follows the state of the bound L1 key — if L1 is revoked, L2 is revoked automatically.

### L3 Service Key lifecycle

Managed by Krypton. Created when a service is deployed, revoked when the L2 above it is revoked.

---

## Deployment Scenarios

### Scenario 1 — SaaS Tenant on shared Platform Mesh

A customer organization (e.g. ACME Corp) runs workloads on a shared Platform Mesh installation operated by someone else. They are one of many tenants on the platform.

- OpenKCM CMK Platform Mesh is the right product
- UI sits at **account level** — the tenant manages their own keys within their account
- Kill switch scope: the account — does not affect other tenants
- CMK Controller is shared platform infrastructure, but key governance is per-tenant

```
Platform Mesh (shared installation)
│
├── Account: ACME Corp    ← OpenKCM CMK UI here
│       └── Workspace → Namespaces → Services
│
├── Account: VW           ← OpenKCM CMK UI here
│       └── Workspace → Namespaces → Services
│
└── Account: Siemens      ← OpenKCM CMK UI here
        └── Workspace → Namespaces → Services
```

---

### Scenario 2 — Air-gapped / sovereign deployment

An organization runs their own Platform Mesh installation in their own data center. They are both the platform operator and the only tenant. No shared platform, no other tenants.

- **OpenKCM CMK (standalone) is the right product** — not the Platform Mesh lightweight variant
- The lightweight UI is designed for tenant-level governance within a shared platform; at deployment/operator level it effectively becomes the full CMK feature set
- Standalone gives the operator full governance: own UI, multi-party approval, full audit trail, platform-level kill switch — without depending on a shared platform portal
- The operator manages keys for their entire installation, not just one account within it

```
Air-gapped deployment (single organization)
│
├── Platform Mesh (operator-owned)
│       └── Account: Operator Org
│               └── Workspace → Namespaces → Services
│
└── OpenKCM CMK (standalone)   ← full product, own UI, platform-level governance
```

---

## Open Questions

**1. SaaS Tenant — is the current UI scope sufficient?**
The current OpenKCM CMK Platform Mesh UI design sits at account level. For the SaaS Tenant scenario this is correct. But does Platform Mesh support account-level microfrontend registration today, or does the UI need to move there from namespace level first?

---

**2. Air-gapped — standalone or extend the lightweight UI?**
The current direction is to use OpenKCM CMK (standalone) for air-gapped deployments rather than extending the lightweight UI one level up to platform level. The alternative — extending the lightweight UI to platform level — risks replicating the full CMK feature set inside the Platform Mesh portal, eliminating the "lightweight" distinction.

> **Proposal:** Air-gapped / sovereign deployments use **OpenKCM CMK (standalone)**, not OpenKCM CMK Platform Mesh. The lightweight variant is explicitly scoped to the SaaS Tenant scenario. This keeps a clean product boundary and avoids feature creep in the Platform Mesh integration.

---

**3. Platform-level kill switch in air-gapped deployments**
In an air-gapped deployment, who triggers the kill switch — the platform operator or a designated security admin? Is there a platform-level role in Platform Mesh that maps to this, or does the standalone CMK product own this entirely?

> **Proposal:** In air-gapped deployments using OpenKCM CMK (standalone), the kill switch is owned entirely by the standalone product — no dependency on Platform Mesh roles. The designated security admin in the standalone UI triggers it, subject to multi-party approval. Platform Mesh roles are irrelevant in this scenario.

---

**4. Multi-party approval — SaaS Tenant**
For the SaaS Tenant scenario, does Platform Mesh provide a native four-eyes approval mechanism that OpenKCM CMK Platform Mesh can integrate with, or does the lightweight variant skip multi-party approval entirely and rely on the tenant's own processes?

---

**5. L3 visibility scope**
Which L3 option (A, B, or C) applies to each deployment scenario? The air-gapped operator may have different requirements from a SaaS tenant.

> **Proposal:** SaaS Tenant → Option B (L3 visible, not manageable) for initial release. Air-gapped / standalone → Option C (selective revocation) as a target feature, since sovereign operators are more likely to need granular service-level revocation. Option C remains an open architectural question pending resolution of who owns selective revocation — OpenKCM CMK or Krypton.