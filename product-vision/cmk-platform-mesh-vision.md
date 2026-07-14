---
status: Draft
last_updated: 2026-07-14
audience: Open Source Community, Contributors, Stakeholders
---

# OpenKCM CMK — Platform Mesh

## Platform dependency — Model 1 not yet deliverable

> **As of 2026-07-09 (confirmed with Platform Mesh team):**
> kcp does not track the relationship between accounts and organizations as a structural platform primitive. The workspace tree is a visual representation only — there is no org-level API that OpenKCM can build on for cross-account governance.
>
> **Model 1 (org-level OpenKCM) is blocked** until Platform Mesh builds an org-level provider API or dedicated virtual workspace. This document describes both models as target architecture. **Model 2 (account-level) is the only currently deliverable model.**

---

## Overview

OpenKCM CMK Platform Mesh is a lightweight CMK integration embedded in the Platform Mesh portal. Designed for Platform Mesh operators and tenants who need customer-managed encryption without running a separate CMK product.

OpenKCM is a **service provider** on Platform Mesh. It publishes its key management API through the Platform Mesh provider-consumer model. When an operator enables OpenKCM from the marketplace at account level — an APIBinding is created and key lifecycle operations become available in that workspace without any direct access to the OpenKCM provider infrastructure.

**Target deployment:** Platform Mesh — running as a Kubernetes controller at account level (Model 2, current) or org level (Model 1, future — see platform dependency above).

---

## Platform Mesh architecture — what OpenKCM builds on

Platform Mesh is built on **kcp**, a Kubernetes control plane without container scheduling. Every account maps to an isolated kcp workspace. OpenKCM integrates with four platform primitives:

| Primitive | What it does | How OpenKCM uses it |
|---|---|---|
| **APIExport** | Provider publishes its service API | OpenKCM publishes the CMK API (key registration, binding, kill switch) from its provider workspace |
| **APIBinding** | Consumer subscribes to a provider API | When an account enables OpenKCM from the marketplace, it creates an APIBinding — the CMK API appears in their workspace |
| **Virtual workspaces** | Aggregated view of all bound consumer objects | The CMK Controller watches all consumer workspaces from one virtual endpoint — no per-workspace polling |
| **OpenFGA (ReBAC)** | Relationship-based authorization within an account | OpenKCM defines its own role model via Platform Mesh provider permissions — Platform Mesh is not opinionated about authorization |

**Placement and governance scope:** Where OpenKCM is placed determines what it can govern. At account level (Model 2 — current), it governs only that account's workspace — the account holder sees and controls only their own keys, and cross-account visibility is intentionally absent. At org level (Model 1 — future, platform dependency not yet met), it would have a single aggregated view across all accounts — required for cross-workspace kill switch and single pane of glass L1 management. Model 1 requires Platform Mesh to build an org-level provider API first.

---

## Key capabilities

- L1 Root Key registration at account level — the account holder registers their own L1 key and binds it to their L2
- BYOK and HYOK — customer key material never touches the platform; OpenBao is the priority backend
- Kill switch — one action revokes access to all namespaces and services within the account
- Zero-touch encryption — workloads deployed from the marketplace are automatically encrypted without developer intervention
- Audit trail — via Platform Mesh audit infrastructure
- Access control — via Platform Mesh provider permissions (OpenKCM defines its own role model)
- Lightweight embedded UI — microfrontend in the Platform Mesh portal via Luigi; no separate web interface

---

## Key hierarchy

| Level | Name | Provisioned by | Admin actions |
|---|---|---|---|
| L1 | Root Key | Key Admin (via OpenKCM UI at account level) | Register, bind to L2, trigger kill switch |
| L2 | Domain Key | CMK Controller — automatic, one per account/workspace | View only — cannot create or delete |
| L3 | Service Key | Account Encryption Admin (via OpenKCM UI, per namespace) | Create, view status |
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

| | Model 1 (org level) ⚠ future | Model 2 (account level) ✓ current |
|---|---|---|
| Controller scope | All accounts in the org | One account only |
| L2s governed | N — one per account | 1 |
| L1 → L2 bindings | N — one per account | 1 |
| Kill switch scope | All accounts in the org | That account only |
| Who operates OpenKCM | Customer (= platform provider = org owner) | Account holder independently |
| Platform dependency | Requires org-level provider API — not yet available in Platform Mesh | None — available today |

OpenKCM on Platform Mesh covers two deployment scenarios. **Model 2 is the currently deliverable model.** Model 1 is documented as target architecture pending Platform Mesh building an org-level provider primitive.

### How the self-selection works (target state)

| Where OpenKCM is enabled | Deployment model | Status |
|---|---|---|
| **Account level** | Enterprise / managed platform | ✓ Deliverable today |
| **Org level** | Air-gapped / sovereign | ⚠ Blocked — requires Platform Mesh org-level provider API |

> **Open question (confirmed with Platform Mesh team 2026-07-09):** kcp does not expose an org-level structural primitive today. The workspace tree is visual only. Platform Mesh would need to build a dedicated org-level provider API or virtual workspace for Model 1 to be possible. Timeline unknown.

---

### Model 1 — Org level enablement (air-gapped / sovereign) ⚠ Future state

> **Platform dependency not yet met.** This model requires an org-level provider API from Platform Mesh that does not exist today. Documented here as target architecture only.

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

### Model 2 — Account level enablement (enterprise / managed platform) ✓ Current

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

Platform Mesh is **not opinionated about authorization** — it delegates role definition to the provider via the provider permissions feature. OpenKCM defines its own role model using this feature.

> **As of 2026-07-09 (confirmed with Platform Mesh team):** OpenKCM uses provider permissions to define custom roles. The Platform Mesh authorization UI epic (ships this year) will allow the OpenKCM UI to show/hide actions based on role once it ships. Until then: implement functionality fully, apply role enforcement to UI once the epic ships.

All roles are scoped to **account level**. There are no cross-account roles.

| Role | Scope | What they can do |
|---|---|---|
| Key Admin | Account | Registers L1 key, binds L1→L2, triggers kill switch, sees full key hierarchy (L1, L2, all namespaces, all L3 keys) |
| Account Encryption Admin | Namespace | Manages L3 service key for their namespace only — no visibility into other namespaces, L1, or L2 |
| Developer | Namespace (application only) | Deploys services — zero-touch encryption, no key visibility at any level |

OpenKCM uses Platform Mesh provider permissions to enforce role boundaries. Custom role logic lives in OpenKCM, not in Platform Mesh.

---

## Platform Mesh account model

```
Account (e.g. ACME Corp)             ← OpenKCM scoped here (account level)
    │
    ├── CMK Controller                ← account level, watches this account's workspace
    │
    └── Workspace: acme-prod         ← one L2 Domain Key auto-provisioned here
            │
            ├── Namespace: default        → Application workloads
            ├── Namespace: db-a           → Database A (L3 key, managed by Encryption Admin for db-a)
            └── Namespace: db-b           → Database B (L3 key, managed by Encryption Admin for db-b)
```

One account has one workspace. The CMK Controller provisions one L2 domain key per account workspace automatically. The Key Admin registers an L1 key at account level and binds it to the account's L2 key. Account Encryption Admins create L3 service keys for each service within their namespace. Different namespaces are governed by different Account Encryption Admins — no cross-namespace visibility.

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
| Revoke L1 Root Key | That account | All namespaces and services in the account become inaccessible |
| Kill switch (account level) | That account | All namespaces, all services inaccessible — cascades through L2 → L3 → L4 automatically |

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
│       ├── Account workspace: acme-prod  ← OpenKCM enabled here at account level
│       │       └── OpenKCM APIBinding → CMK API available in this workspace
│       └── Virtual workspace: aggregated view of this account's bound objects
│
├── OpenKCM provider cluster (account level)
│       ├── CMK Controller  ← watches this account's workspace, provisions L2, manages lifecycle
│       ├── CMK UI (microfrontend via Luigi, embedded in Platform Mesh portal)
│       └── OpenBao binding ← provider-to-provider, key material stays in customer keystore
│
└── Account workspace: acme-prod
        ├── L2 Domain Key CR  ← auto-provisioned by CMK Controller
        ├── L3 Service Key CRs ← created by Account Encryption Admin via OpenKCM UI per namespace
        └── Workload namespaces → KMIP endpoint + key ID injected into pods at deploy time
```
