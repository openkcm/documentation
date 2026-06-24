---
status: Draft
last_updated: 2026-06-24
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

+----------------------+
|   L2 Domain Key      |
+----------------------+
| L2KeyID              |
| WorkspaceID          | ← one per account/workspace, not per namespace
| Status               |
| ProvisionedBy        | ← CMK Controller (automatic, via Krypton API)
+----------------------+
        | 1
        |
        | 1 (key binding — admin assigns)
+----------------------+
|   L1 Root Key        |
+----------------------+
| L1KeyID              |
| OrgID / AccountID    | ← org level (Model 1) or account level (Model 2)
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
| CreatedBy        | ← Security Admin (via OpenKCM CMK UI)
+------------------+


+------------------+
|  Security Admin  |
+------------------+
| UserID           |
| OrgID / AccountID| ← org level (Model 1) or account level (Model 2)
| Role             |
+------------------+
        |
        | registers L1 keys
        | binds L1 → L2
        | creates L3 keys
        | triggers kill switch
```

---

## Entities

### Account
The top-level administrative boundary. Represents one customer organization. One account has exactly one workspace. The account is the tenant — pressing the kill switch at account level locks the entire account.

### Security Admin
A user responsible for registering L1 root keys, binding them to L2 domain keys, creating L3 service keys, and triggering the kill switch. In Model 1 (org-level enablement), the security admin operates at org level and has visibility across all accounts. In Model 2 (account-level enablement), the security admin operates within their own account only.

### CMK Controller
A Kubernetes controller that watches OpenKCM CRDs and reconciles key state. In Model 1, it runs at org level and governs all accounts in the organization. In Model 2, it is scoped to a single account. One controller per OpenKCM deployment.

### Workspace
The deployment environment within an account. One account has one workspace. Contains multiple namespaces where applications and services run.

### Namespace
A Kubernetes namespace within the workspace. Applications and services run here. Namespace-level encryption separation is achieved through L3 Service Keys — one per service — all wrapped under the account's single L2 Domain Key.

### L1 Root Key
A customer-owned root encryption key. Registered by the security admin at org level (Model 1) or account level (Model 2). Key material never leaves the customer's external keystore — OpenKCM CMK holds only a reference. One L1 key can be bound to multiple accounts (Model 1). Each L1 key is backed by one external keystore.

### External Keystore
The customer's own key management system where L1 key material lives. Supported backends: OpenBao, AWS KMS, Azure Key Vault, GCP KMS, HSM via PKCS#11. OpenKCM CMK never stores key material — it holds a pointer and calls the keystore at runtime via Krypton.

### Key Binding
The association between an L1 Root Key and an L2 Domain Key. Created by the security admin. One L2 Domain Key is bound to exactly one L1 Root Key. One L1 Root Key can be bound to multiple L2 Domain Keys (multiple accounts can share the same L1 key). Different accounts can be bound to different L1 keys — the admin decides.

### L2 Domain Key
Provisioned automatically by the CMK Controller when OpenKCM is enabled on Platform Mesh — **one per account/workspace**. The CMK Controller calls Krypton explicitly via API to create it. The security admin cannot create or delete L2 keys. The admin binds an L1 key to the account's L2 key. All services across all namespaces in that account are encrypted under the single L2 key.

### L3 Service Key
Created by the security admin via the OpenKCM CMK UI. Wrapped under the L2 Domain Key of that account. One L3 key per service instance. Not provisioned automatically — the security admin defines the service boundaries.

### Service
An application or workload running inside a namespace (e.g. MongoDB, Postgres, Redis). Has one L3 Service Key. Encrypted automatically — no key configuration required from the developer.

---

## Key Binding Rules

| Rule | Description |
|---|---|
| One L2 per account | CMK Controller provisions exactly one L2 Domain Key per account/workspace via Krypton API |
| Admin registers L1 | The security admin registers one or more L1 keys at org level (Model 1) or account level (Model 2) |
| Admin binds L1 → L2 | The security admin decides which L1 key governs which account |
| One L1 per L2 | Each L2 Domain Key is bound to exactly one L1 Root Key at a time |
| One L1 → many L2s | One L1 Root Key can be bound to multiple accounts (Model 1) |
| Different L1s per account | Different accounts can be governed by different L1 keys |
| Admin creates L3 | The security admin creates one L3 Service Key per service via the OpenKCM CMK UI |
| Unbound account | An account with no L1 binding is not customer-governed — it uses platform-managed encryption until the admin binds an L1 key |

---

## Kill Switch Scope

| Action | Scope | Effect |
|---|---|---|
| Revoke L1 Root Key | All accounts bound to that L1 | All services in those accounts become inaccessible |
| Kill switch — Model 1 (org level) | Entire organization | All accounts, all namespaces, all services inaccessible — regardless of which L1 they are bound to |
| Kill switch — Model 2 (account level) | That account only | All namespaces and services in that account become inaccessible |

In Model 1, if different accounts are bound to different L1 keys, revoking one L1 affects only the accounts bound to it. The org-level kill switch revokes everything across all accounts.

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

Managed by the CMK Controller. Follows the state of the bound L1 key — if L1 is revoked, L2 is revoked automatically.

### L3 Service Key lifecycle

Created by the security admin via the OpenKCM CMK UI. Revoked when the L2 above it is revoked.

---

## Deployment Scenarios

These scenarios map directly to the two deployment models defined in `cmk-platform-mesh-vision.md`. The self-selecting mechanism — where the customer enables OpenKCM from the marketplace at org level or account level — determines which scenario applies.

### Scenario 1 — Enterprise / managed platform (Model 2: account-level enablement)

A customer organization (e.g. ACME Corp) runs workloads on a shared Platform Mesh installation operated by a platform provider. They are one of many tenants on the platform. Each account holder enables OpenKCM independently at account level and manages their own keys in isolation.

- OpenKCM CMK Platform Mesh is the right product
- CMK Controller scoped to that account — one L2 provisioned for that account
- Kill switch scope: that account only — does not affect other tenants
- Platform provider has no visibility into any account's key governance

```
Platform Mesh (shared installation)
│
├── Account: ACME Corp    ← enables OpenKCM at account level, manages own keys
│       └── Workspace → L2 → Namespaces → L3 per service
│
├── Account: VW           ← enables OpenKCM at account level, independently
│       └── Workspace → L2 → Namespaces → L3 per service
│
└── Account: Siemens      ← enables OpenKCM at account level, independently
        └── Workspace → L2 → Namespaces → L3 per service
```

---

### Scenario 2 — Air-gapped / sovereign deployment (Model 1: org-level enablement)

An organization runs their own Platform Mesh installation. They are both the platform operator and the org owner. They enable OpenKCM at org level — the CMK Controller governs all accounts across the entire organization from one place.

- OpenKCM CMK Platform Mesh is the right product — org-level enablement
- CMK Controller at org level — provisions L2 for every account automatically
- Security admin has single pane of glass across all accounts
- Kill switch scope: entire organization — all accounts, all namespaces, all services
- No external platform provider in the loop

```
Air-gapped deployment (single organization owns and operates everything)
│
├── Org level: OpenKCM enabled here → CMK Controller governs all accounts
│
├── Account: prod    → L2 auto-provisioned → security admin binds L1 → L3 per service
├── Account: dev     → L2 auto-provisioned → security admin binds L1 → L3 per service
└── Account: staging → L2 auto-provisioned → security admin binds L1 → L3 per service
```

---

## Open Questions

**1. Two APIExports — can one OpenKCM deployment serve both models?**
Can a single OpenKCM deployment publish two APIExports — one at org level and one at account level — so customers can choose their model at enablement time without deploying a separate instance? To be validated with the Platform Mesh team.

---

**2. Multi-party approval — Platform Mesh integration**
Does Platform Mesh provide a native four-eyes approval mechanism that OpenKCM CMK Platform Mesh can integrate with, or does the lightweight variant rely on the tenant's own processes?

---

**3. L3 visibility scope**
Which L3 option (A, B, or C) applies to each deployment scenario?

> **Current direction:** Option B for initial release — L3 visible and manageable by the security admin in both scenarios. Option C (selective service-level revocation) is a future feature pending resolution of who owns selective revocation — OpenKCM CMK or Krypton.