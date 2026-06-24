---
status: Draft
last_updated: 2026-06-24
audience: Open Source Community, Contributors, Stakeholders
---

# OpenKCM CMK — Platform Mesh

## Overview

OpenKCM CMK Platform Mesh is a lightweight CMK integration embedded in the Platform Mesh portal. Designed for Platform Mesh operators and tenants who need customer-managed encryption without running a separate CMK product.

OpenKCM is a **service provider** on Platform Mesh. It publishes its key management API through the Platform Mesh provider-consumer model. When an operator enables OpenKCM from the marketplace — at either org level or account level — an APIBinding is created and key lifecycle operations become available in that workspace without any direct access to the OpenKCM provider infrastructure.

**Target deployment:** Platform Mesh — running as a Kubernetes controller, with placement at org level or account level depending on the chosen deployment model (see Deployment Models below).

---

## Platform Mesh architecture — what OpenKCM builds on

Platform Mesh is built on **kcp**, a Kubernetes control plane without container scheduling. Every account maps to an isolated kcp workspace. OpenKCM integrates with four platform primitives:

| Primitive | What it does | How OpenKCM uses it |
|---|---|---|
| **APIExport** | Provider publishes its service API | OpenKCM publishes the CMK API (key registration, binding, kill switch) from its provider workspace |
| **APIBinding** | Consumer subscribes to a provider API | When an account enables OpenKCM from the marketplace, it creates an APIBinding — the CMK API appears in their workspace |
| **Virtual workspaces** | Aggregated view of all bound consumer objects | The CMK Controller watches all consumer workspaces from one virtual endpoint — no per-workspace polling |
| **OpenFGA (ReBAC)** | Relationship-based authorization derived from org hierarchy | Role separation between security admin and developer is handled automatically — no custom role logic needed in OpenKCM |

**Placement and governance scope:** Where OpenKCM is placed determines what it can govern. At org level, it has a single aggregated view across all accounts and workspaces — required for cross-workspace kill switch and single pane of glass L1 management (Model 1). At account level, it governs only that account's workspace — the account holder sees and controls only their own keys, and cross-account visibility is intentionally absent (Model 2). Both placements are valid; the customer chooses by where they enable OpenKCM from the marketplace.

---

## Key capabilities

- L1 Root Key registration at org level — one registration covers all accounts and all their workspaces
- BYOK and HYOK — customer key material never touches the platform; OpenBao is the priority backend
- Kill switch — one action revokes access across all accounts and all their workspaces in the organization
- Zero-touch encryption — workloads deployed from the marketplace are automatically encrypted without developer intervention
- Audit trail — via Platform Mesh audit infrastructure
- Access control — via Platform Mesh OpenFGA (no custom role logic in OpenKCM)
- Lightweight embedded UI — microfrontend in the Platform Mesh portal via Luigi; no separate web interface

---

## Key hierarchy

| Level | Name | Provisioned by | Admin actions |
|---|---|---|---|
| L1 | Root Key | Security admin (via OpenKCM UI at org level) | Register, rotate, revoke, kill switch |
| L2 | Domain Key | CMK Controller — automatic, one per account/workspace | View only — bind L1→L2 done by security admin at org level; cannot create or delete |
| L3 | Service Key | Security admin (via OpenKCM UI) | Create, view status — see open question below |
| L4 | Data Encryption Key | Consuming service (via Krypton Gateway KMIP) | Not visible to admin |

### How L2 is provisioned

L2 provisioning is triggered by an APIBinding being created in an account's workspace. The CMK Controller — watching all consumer workspaces via the virtual workspace endpoint — detects the new binding and calls the Krypton gRPC API to provision one L2 domain key for that account. One account = one L2. L2 is not provisioned per namespace — it is provisioned once at account level and covers all namespaces within that account.

**Model 1 (org-level enablement):** OpenKCM enabled at org level → CMK Controller provisions one L2 for every existing account in the org. Every new account created afterwards gets an L2 automatically.

**Model 2 (account-level enablement):** Each account enables OpenKCM independently → CMK Controller scoped to that account provisions one L2 for that account only.

The security admin does not create L2 keys — they only bind an L1 key to the existing L2. An unbound account is not customer-governed — it uses platform-managed encryption until the admin binds an L1 key.

### How L4 is handled

L4 (Data Encryption Keys) are managed entirely by consuming services (e.g. MongoDB, PostgreSQL) via the Krypton Gateway KMIP interface. Platform Mesh injects the KMIP endpoint and the relevant key ID into workload pods as environment variables or Kubernetes Secrets at deploy time — this is what makes encryption zero-touch for developers. L4 is not visible in the admin UI. The kill switch at L1 cascades automatically through L2, L3, and L4 — the admin does not need to act at L4 level.

---

## Deployment models

The tenant boundary is always the account/workspace. L2 is always one per account. These two facts are constant across both deployment models.

**The difference between the two models is how many L2s the CMK Controller governs** — and therefore how many accounts fall under a single security admin's key governance.

| | Model 1 (org level) | Model 2 (account level) |
|---|---|---|
| Controller scope | All accounts in the org | One account only |
| L2s governed | N — one per account | 1 |
| L1 → L2 bindings | N — one per account | 1 |
| Kill switch scope | All accounts in the org | That account only |
| Who operates OpenKCM | Customer (= platform provider = org owner) | Account holder independently |

OpenKCM on Platform Mesh covers two deployment scenarios. The deployment model is **self-selecting** — it is determined by where the customer enables OpenKCM from the marketplace, not by a configuration flag or a separate product. This gives customers full flexibility to choose their own sovereignty model.

### How the self-selection works

| Where OpenKCM is enabled | Deployment model | Who operates OpenKCM |
|---|---|---|
| **Org level** | Air-gapped / sovereign | Customer (= platform provider = org owner) |
| **Account level** | Enterprise / managed platform | Account holder independently |

The customer makes this choice once at enablement time. Platform Mesh APIExport/APIBinding supports enablement at different levels of the hierarchy — OpenKCM publishes two APIExports, one at org level and one at account level. The customer binds to whichever fits their sovereignty requirement.

> **Open question (for RFC meeting):** Can one OpenKCM deployment serve both models simultaneously via two APIExports — one at org level, one at account level? To be validated with the Platform Mesh team.

---

### Model 1 — Org level enablement (air-gapped / sovereign)

The customer enables OpenKCM from the marketplace at **org level**. OpenKCM operates across the entire organization. The org-level super admin has full cross-workspace visibility and control — single pane of glass, cross-workspace kill switch, full governance. The platform provider and the account holder are the same entity.

```
Customer enables OpenKCM at org level
        │
        ▼
CMK Controller at org level
└── Provisions L2 automatically for every account in the org
└── Provisions L2 for every new account created afterwards
└── Super admin sees all accounts, all keys, full kill switch
└── No external party in the loop
```

**L2 provisioning trigger:** OpenKCM enabled at org level → CMK Controller provisions one L2 for every existing account. New account created → L2 provisioned automatically.

---

### Model 2 — Account level enablement (enterprise / managed platform)

The customer enables OpenKCM from the marketplace at **account level**. OpenKCM operates within that account only. The account holder manages their own keys independently — the platform provider has zero visibility into that account's key governance. Different accounts can each enable their own OpenKCM instance independently.

```
Account Holder A enables OpenKCM at account level
        │
        ▼
CMK Controller scoped to Account A
└── Provisions L2 for Account A only
└── Account Holder A sees only their own keys
└── Platform provider has no visibility into Account A's key governance

Account Holder B enables OpenKCM at account level (independently)
        │
        ▼
CMK Controller scoped to Account B
└── Provisions L2 for Account B only
└── Completely isolated from Account A
```

**L2 provisioning trigger:** Account enables OpenKCM → CMK Controller provisions one L2 for that account.

**The sovereignty guarantee:** the platform provider operates the infrastructure but is completely outside the key governance boundary. Each account holder owns and operates their own OpenKCM instance.

---

**The critical boundary in Model 2:** even though the platform provider operates Platform Mesh, they cannot see key material, key bindings, or trigger key operations for any account that has enabled OpenKCM at account level. OpenFGA enforces this — account-level enablement means the account holder owns the OpenKCM workspace, not the platform provider.

---

## Role separation — how authorization works

Platform Mesh uses **OpenFGA** (relationship-based access control) to derive permissions from the organizational hierarchy. OpenKCM does not need to build custom role logic — the platform handles it.

| Role | Scope | What they can do in OpenKCM |
|---|---|---|
| Super Admin | Org level — air-gapped model | Full access — all accounts, all keys, cross-workspace kill switch |
| Platform Operator | Org level — enterprise model | Infrastructure health only — no key material, no key operations |
| Security Admin | Org level | Register L1 keys, bind L1→L2, trigger kill switch, view full key chain across all accounts |
| Workspace Admin | Workspace level | View key status for their workspace — no key management actions |
| Developer | Namespace level | Nothing — zero-touch encryption; no key visibility |

OpenFGA derives the permission boundary from the organizational relationship graph automatically. No per-user configuration needed in OpenKCM.

---

## Platform Mesh account model

```
Organization (e.g. ACME Corp)        ← OpenKCM lives here (org level)
    │
    ├── CMK Controller                ← org level, watches all account workspaces
    │
    └── Account: acme-prod           ← workspace, APIBinding to OpenKCM here
            │
            ├── Namespace: default        → Application workloads
            ├── Namespace: db-a           → Database A
            └── Namespace: db-b           → Database B
```

One organization has one or more accounts. Each account maps to a kcp workspace. The CMK Controller provisions one L2 domain key per account workspace automatically. The security admin registers one or more L1 keys at org level and binds each L1 key to the relevant account workspaces.

One L1 key can be bound to multiple workspaces. Different workspaces can be bound to different L1 keys. Pressing the kill switch at org level locks everything — all workspaces, all namespaces, all workloads, regardless of which L1 key they are bound to.

---

## OpenBao — provider-to-provider integration

OpenKCM does not connect to OpenBao directly as an external dependency. On Platform Mesh, OpenBao is itself a **service provider** — it publishes its API through the same APIExport/APIBinding model. OpenKCM is a **composing provider** — it binds to OpenBao's API to store key material, then exposes the CMK API to end consumers.

The security admin never touches OpenBao directly. They register their L1 key in the OpenKCM UI — OpenKCM resolves the OpenBao binding underneath. This is the HYOK guarantee: key material stays in OpenBao (customer-controlled), OpenKCM holds only the reference.

---

## What the security admin sees

The security admin operates from the OpenKCM CMK UI — a microfrontend embedded in the Platform Mesh portal. In Model 1 (org-level enablement), they see the full key chain for the entire organization in one place. In Model 2 (account-level enablement), each account's security admin sees only their own account's key chain.

On enabling OpenKCM from the Platform Mesh marketplace, the CMK Controller automatically provisions L2 domain keys for all relevant account workspaces. These L2 keys appear in the UI immediately. The admin then registers their L1 root key and binds it to each account's L2 key. They also create L3 service keys for each service that needs to be encrypted under that account's L2.

Actions are only available where the admin has authority — CMK Controller-provisioned L2 keys are visible but cannot be created or deleted by the admin.

---

## What developers see

Nothing. Zero-touch encryption means workloads come up encrypted without any key configuration on the developer's part. The developer deploys to a namespace — Platform Mesh injects the KMIP endpoint and key ID into the pod at deploy time. Encryption is handled automatically once the security admin has bound an L1 key to that workspace.

---

## L3 Service Key visibility — open question

L3 keys are service-level keys, one per service (e.g. MongoDB, PostgreSQL), created by the security admin via the OpenKCM UI and wrapped under the L2 of that account. Three options are under consideration for how L3 is surfaced in the UI:

- **Option A — L3 fully internal, never shown.** The security admin creates L3 keys via the UI but they are not displayed after creation. The kill switch at L1 cascades down through L2 and L3 automatically. Simplest model.

- **Option B — L3 visible and manageable.** The org-level UI shows a breakdown: account → services → each with their L3 key status. The security admin creates and views L3 keys. Useful for verifying that each service is actually encrypted.

- **Option C — L3 with selective revocation.** In addition to Option B, the kill switch can be triggered at L3 level — revoke only a specific service, not the entire account. Open architectural question: who owns selective service revocation — OpenKCM CMK Platform Mesh or Krypton?

> **Current direction:** Option B for the initial release — L3 visible and manageable by the security admin. Option C is a future feature pending resolution of the selective revocation ownership question.

### Option A — L3 fully internal

```
Organization: ACME Corp
└── OpenKCM CMK (org level)
        │
        ├── L1 Root Key: openbao/acme-l1-master   [Active]  ← admin registers + manages
        │
        ├── Account Workspace: acme-prod
        │       └── L2 Domain Key: acme-prod-domain  [Bound]  ← admin binds L1→L2
        │
        └── Account Workspace: acme-dev
                └── L2 Domain Key: acme-dev-domain   [Bound]

L3 and L4 keys exist but are never shown in the UI.
Kill switch: L1 revoked → L2 revoked → L3 revoked → all workloads inaccessible.
```

### Option B — L3 visible and manageable

```
Organization: ACME Corp
└── OpenKCM CMK (org level)
        │
        ├── L1 Root Key: openbao/acme-l1-master   [Active]  ← admin registers + manages
        │
        ├── Account Workspace: acme-prod
        │       ├── L2 Domain Key: acme-prod-domain  [Bound]  ← admin binds L1→L2
        │       └── Services:
        │               ├── mongodb   → L3: acme-prod-mongodb-key  [Active]  ← admin creates
        │               └── postgres  → L3: acme-prod-postgres-key [Active]  ← admin creates
        │
        └── Account Workspace: acme-dev
                ├── L2 Domain Key: acme-dev-domain   [Bound]
                └── Services:
                        └── redis     → L3: acme-dev-redis-key     [Active]  ← admin creates

Admin can see and create L3 keys. No delete or revoke actions at L3 level.
Kill switch at L1 still cascades automatically across all L2 and L3.
```

### Option C — L3 with selective revocation (future / open question)

```
Organization: ACME Corp
└── OpenKCM CMK (org level)
        │
        ├── L1 Root Key: openbao/acme-l1-master   [Active]
        │
        ├── Account Workspace: acme-prod
        │       ├── L2 Domain Key: acme-prod-domain  [Bound]
        │       └── Services:
        │               ├── mongodb   → L3: acme-prod-mongodb-key  [Active]  [Revoke]
        │               └── postgres  → L3: acme-prod-postgres-key [Active]  [Revoke]
        │
        └── Account Workspace: acme-dev
                ├── L2 Domain Key: acme-dev-domain   [Bound]
                └── Services:
                        └── redis     → L3: acme-dev-redis-key     [Active]  [Revoke]

Admin can revoke individual service keys without triggering the full kill switch.
Example: revoke MongoDB only → MongoDB data inaccessible, Postgres and Redis unaffected.

⚠ Open question: who owns selective revocation — OpenKCM CMK Platform Mesh or Krypton?
```

---

## Kill switch scope

| Action | Scope | Effect |
|---|---|---|
| Revoke L1 Root Key | All workspaces bound to that L1 | All services in those workspaces become inaccessible |
| Kill switch (org level) | Entire organization | All workspaces, all namespaces, all services inaccessible — regardless of which L1 they are bound to |

---

## Integration path — how OpenKCM connects to Platform Mesh

OpenKCM uses the **multicluster-runtime** integration path. The CMK Controller is a custom controller with its own reconciliation logic — not a passive sync agent. This is required because:

- Kill switch propagation is cross-workspace and must cascade through L2 → L3 → L4 in a controlled sequence
- L2 provisioning requires calling the Krypton gRPC API, not just syncing CRD state
- Authorization decisions (which admin can act on which workspace) require custom logic on top of OpenFGA

The CMK Controller connects to Platform Mesh via outbound HTTPS to kcp — no inbound access to the OpenKCM provider cluster is required.

---

## Deployment scenario

```
Platform Mesh
│
├── kcp (control plane)
│       ├── Organization workspace: root:acme    ← OpenKCM lives here
│       │       └── OpenKCM APIBinding → CMK API available to all accounts
│       └── Virtual workspace: aggregated view of all bound account objects
│
├── OpenKCM provider cluster (org level)
│       ├── CMK Controller  ← watches virtual workspace, provisions L2, manages lifecycle
│       ├── CMK UI (microfrontend via Luigi, embedded in Platform Mesh portal)
│       └── OpenBao binding ← provider-to-provider, key material stays in customer keystore
│
├── Account workspace: acme-prod
│       ├── L2 Domain Key CR  ← auto-provisioned by CMK Controller
│       ├── L3 Service Key CRs ← created by security admin via OpenKCM UI
│       └── Workload namespaces → KMIP endpoint + key ID injected into pods at deploy time
│
└── Account workspace: acme-dev
        ├── L2 Domain Key CR  ← auto-provisioned by CMK Controller
        ├── L3 Service Key CRs ← created by security admin via OpenKCM UI
        └── Workload namespaces → KMIP endpoint + key ID injected into pods at deploy time
```
