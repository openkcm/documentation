---
status: Draft
last_updated: 2026-06-16
audience: Open Source Community, Contributors, Stakeholders
---

# OpenKCM Product Vision

## What is OpenKCM CMK?

OpenKCM CMK is an open source Customer Managed Key (CMK) layer for cloud-native platforms. It gives organizations cryptographic control over their own encryption keys — so that no platform operator, cloud provider, or managed service can access customer data without the customer's explicit consent.

OpenKCM CMK governs encryption keys. The actual cryptographic operations are performed by [Krypton](https://github.com/openkcm/krypton) — the crypto execution engine that OpenKCM CMK sits on top of.

---

## The Problem

When an organization runs workloads on a cloud platform, their data is typically encrypted — but by a key the platform controls. The organization trusts the platform not to look. That trust is implicit, unverifiable, and often not sufficient for regulated industries.

Customer Managed Keys solve this by inverting control: the customer owns the root key, the platform never has access to key material, and the customer can revoke access at any time — including instantly, for all workloads at once.

This is not a theoretical requirement. Defense agencies, government authorities, utilities, financial institutions, and any organization subject to data residency or sovereignty regulation needs this guarantee by law.

---

## Who OpenKCM CMK is for

**Sovereign and private cloud operators**
Organizations running their own infrastructure in a specific jurisdiction — air-gapped, on-premise, or in a regulated data center. Keys must never leave their environment. Examples: national defense agencies, government authorities, critical infrastructure operators.

**Managed service providers**
Companies offering services (databases, runtimes, platforms) to other organizations. They need to tell their customers: your data is encrypted under your key, not ours. OpenKCM CMK gives them the integration point to make that guarantee.

**Platform builders**
Teams building cloud-native platforms who want to offer CMK as a first-class capability without building key governance from scratch.

**Application and service developers**
Teams deploying workloads who should never need to think about key management — but whose applications must handle key revocation gracefully when the Red Button is pressed.

---

## Two Products

OpenKCM CMK is offered in two variants, both sharing the same CMK Core.

---

### OpenKCM CMK

The full-featured, standalone CMK product. Runs on any Kubernetes environment — no platform dependency required. Designed for organizations that operate their own infrastructure and need complete key governance under their own control.

**Target deployment:** Any Kubernetes cluster the customer operates themselves — on-premise, sovereign cloud, air-gapped, or any cloud provider.

**Key capabilities:**

- L1 Root Key registration and lifecycle management (enable, disable, rotate, delete)
- BYOK (Bring Your Own Key) and HYOK (Hold Your Own Key) — customer key material never touches the platform
- Pluggable keystore backends: OpenBao, AWS KMS, Azure Key Vault, GCP KMS, HSM via PKCS#11
- Kill switch (Red Button) — instant revocation of all workloads across the deployment
- Multi-party approval (Four-Eyes) — high-stakes operations require a second authorized approver
- Full audit trail — tamper-evident log of every key operation, exportable for compliance
- Tenant onboarding and offboarding
- Role-based access control via OIDC / RBAC
- Full CMK UI — own web interface, not embedded in any platform portal

**Key hierarchy:**

| Level | Name | Scope | Managed by |
|---|---|---|---|
| L1 | Root Key | Tenant | Customer — registered via OpenKCM CMK |
| L2 | Domain Key | Workspace | OpenKCM CMK — internal |
| L3 | Service Key | Service | Krypton — internal |
| L4 | Data Encryption Key | Workload | Krypton — internal |

The customer only ever registers and manages the L1 key. Everything below is derived and managed internally by OpenKCM CMK and Krypton.

---

### OpenKCM CMK Platform Mesh

A lightweight CMK integration embedded in the Platform Mesh portal. Designed for Platform Mesh operators and tenants who need customer-managed encryption without running a separate CMK product.

**Target deployment:** Platform Mesh — running as a Kubernetes controller in the platform-admin namespace, with cross-namespace RBAC authority over the entire workspace.

**Key capabilities:**

- L1 Root Key registration at account level — one registration covers all namespaces in the account
- BYOK and HYOK — same guarantee as OpenKCM CMK; OpenBao is the priority backend
- Kill switch — one action revokes access across all namespaces in the workspace
- Zero-touch encryption — workloads deployed from the marketplace are automatically encrypted without developer intervention
- Audit trail — via Platform Mesh audit infrastructure
- Access control — via Platform Mesh RBAC
- Lightweight embedded UI — microfrontend in the Platform Mesh portal via Luigi; no separate web interface

**What the security admin sees:**

The OpenKCM section appears at account level in the Platform Mesh portal — not inside a namespace. Each account has one workspace, and that workspace contains multiple namespaces. When OpenKCM is enabled, Platform Mesh automatically provisions one L2 Domain Key per namespace.

The security admin registers the organization's L1 root key at account level. She then binds that L1 key to the L2 Domain Keys that Platform Mesh has provisioned — one binding per namespace. Once bound, all workloads running in that namespace are encrypted under the L1→L2 key chain.

**One unified UI — access gated by key level**

All key levels (L1, L2, L3) are managed from a single account-level UI. What actions are available depends on the key level and the user's role:

| Key level | Generated by | Admin actions |
|---|---|---|
| L1 Root Key | Customer (via OpenKCM CMK) | Register, bind to L2, rotate, revoke, kill switch |
| L2 Domain Key | Platform Mesh (automatic) | View, bind L1 to L2 — cannot create or delete |
| L3 Service Key | Krypton (automatic) | View status — see open question below |

The security admin sees the full key chain for the account in one place. Actions are only available where the admin has authority — Platform Mesh-generated and Krypton-generated keys are visible but not creatable or deletable by the admin.

**What developers see:**

Nothing. Zero-touch encryption means workloads come up encrypted without any key configuration on the developer's part.

**L3 Service Key visibility — open question**

L3 keys are generated by Krypton automatically when a service is deployed, wrapped under the L2 of that namespace. Three options are under consideration for how L3 keys are handled in the UI:

- **Option A — L3 fully internal, never shown.** Krypton generates and manages L3 silently. The security admin only sees L1 and L2. The kill switch at L1 level cascades down through L2 and L3 automatically. Simplest model.

- **Option B — L3 visible but not manageable.** The account-level UI shows a breakdown: workspace → namespace → services → each with their L3 key status. Read-only. Useful for the security admin to verify that each service (MongoDB, Postgres, etc.) is actually encrypted. No actions at L3 level.

- **Option C — L3 has its own actions (selective revocation).** The kill switch can be triggered at L3 level — revoke only a specific service (e.g. MongoDB only), not the entire workspace. This requires UI at service level and introduces the question of who owns selective service revocation: OpenKCM CMK Platform Mesh or Krypton. This is an open architectural question to be resolved.

> **Current direction:** Option B for the initial release — L3 visible, not manageable. Option C is a future feature pending resolution of the selective revocation ownership question.

---

**Option A — L3 fully internal**

```
Account: ACME Corp
└── OpenKCM CMK Platform Mesh (account level)
        │
        ├── L1 Root Key: openbao/acme-l1-master   [Active]  ← admin registers + manages
        │
        ├── Namespace: default
        │       └── L2 Domain Key: acme-default-domain  [Bound]  ← admin binds L1→L2
        │
        ├── Namespace: db-a
        │       └── L2 Domain Key: acme-dba-domain      [Bound]
        │
        └── Namespace: db-b
                └── L2 Domain Key: acme-dbb-domain      [Bound]

L3 Service Keys and L4 Data Keys are managed internally by Krypton.
Admin never sees them. Kill switch at L1 cascades automatically.

Kill switch:
L1 revoked → L2 revoked → L3 revoked → all workloads inaccessible
```

---

**Option B — L3 visible, not manageable**

```
Account: ACME Corp
└── OpenKCM CMK Platform Mesh (account level)
        │
        ├── L1 Root Key: openbao/acme-l1-master   [Active]  ← admin registers + manages
        │
        ├── Namespace: default
        │       ├── L2 Domain Key: acme-default-domain  [Bound]  ← admin binds L1→L2
        │       └── Services (read-only):
        │               └── mongodb       → L3: acme-default-mongodb-key  [Active]
        │
        ├── Namespace: db-a
        │       ├── L2 Domain Key: acme-dba-domain      [Bound]
        │       └── Services (read-only):
        │               └── postgres      → L3: acme-dba-postgres-key     [Active]
        │
        └── Namespace: db-b
                ├── L2 Domain Key: acme-dbb-domain      [Bound]
                └── Services (read-only):
                        └── redis         → L3: acme-dbb-redis-key        [Active]

Admin can see all L3 keys and their status. No actions available at L3 level.
Kill switch at L1 still cascades automatically across all L2 and L3.
```

---

**Option C — L3 with selective revocation (future / open question)**

```
Account: ACME Corp
└── OpenKCM CMK Platform Mesh (account level)
        │
        ├── L1 Root Key: openbao/acme-l1-master   [Active]  ← admin registers + manages
        │
        ├── Namespace: default
        │       ├── L2 Domain Key: acme-default-domain  [Bound]
        │       └── Services:
        │               └── mongodb  → L3: acme-default-mongodb-key  [Active]  [Revoke]
        │
        ├── Namespace: db-a
        │       ├── L2 Domain Key: acme-dba-domain      [Bound]
        │       └── Services:
        │               └── postgres → L3: acme-dba-postgres-key     [Active]  [Revoke]
        │
        └── Namespace: db-b
                ├── L2 Domain Key: acme-dbb-domain      [Bound]
                └── Services:
                        └── redis    → L3: acme-dbb-redis-key        [Active]  [Revoke]

Admin can revoke individual service keys without triggering the full kill switch.
Example: revoke MongoDB only → MongoDB data inaccessible, Postgres and Redis unaffected.

⚠ Open question: who owns selective revocation — OpenKCM CMK Platform Mesh or Krypton?
```

**Platform Mesh account model:**

```
Account (e.g. ACME Corp)          ← administrative boundary, CMK governed here
    │
    └── Workspace: acme-prod
            │
            ├── Namespace: default        → Application
            ├── Namespace: db-a           → Database A
            ├── Namespace: db-b           → Database B
            └── Namespace: platform-admin → OpenKCM Controller
```

One L1 key at account level governs all workspaces and namespaces. Pressing the kill switch at account level locks everything — not just one namespace.

---

## What the two products share — CMK Core

Both products are built on **CMK Core** — the shared business logic library that contains:

- Key lifecycle state machine (NIST SP 800-57: PreActivation → Active → Suspended → Deactivated → Destroyed)
- L1 key validation logic
- Kill switch execution logic
- Tenant registry and onboarding/offboarding
- Audit trail logic
- Multi-party approval logic

CMK Core is deployment-agnostic. It has no dependency on Platform Mesh, any cloud provider, or any specific keystore. Both OpenKCM CMK and OpenKCM CMK Platform Mesh consume it as a shared library.

---

## What OpenKCM CMK does not do

- Perform cryptographic operations — that is Krypton's responsibility
- Store key material — keys live in the customer's own keystore (OpenBao, AWS KMS, HSM, etc.)
- Replace a secrets manager — OpenKCM CMK governs encryption keys, not application secrets
- Manage platform-controlled keys — platform-managed keys are out of scope; OpenKCM CMK only governs customer-owned keys

---

## Red Button / Kill Switch

The kill switch is a customer-initiated action that immediately revokes access to all data in the governed scope. It works by revoking the L1 root key — since all L2, L3, and L4 keys are derived from it, every encrypted workload becomes inaccessible within the propagation window.

This is a one-way, destructive action. It is not a pause — it is a cryptographic lock. Once executed, data cannot be recovered without a new key registration and an approval cycle.

For OpenKCM CMK: the kill switch scope is the tenant.
For OpenKCM CMK Platform Mesh: the kill switch scope is the account/workspace — all namespaces within it.

---

## BYOK and HYOK

OpenKCM CMK prioritizes **HYOK (Hold Your Own Key)**. The customer's L1 key material never touches the OpenKCM CMK platform. OpenKCM CMK holds only a reference — a pointer to the key in the customer's own keystore. At runtime, Krypton calls out to that keystore to use the key.

**BYOK (Bring Your Own Key)** is also supported — the customer imports key material into the platform's keystore. HYOK is the stronger sovereignty guarantee.

---

## Supported keystore backends

| Backend | Type | Status |
|---|---|---|
| OpenBao | Open source Vault fork | Priority |
| AWS KMS | Cloud KMS | Supported |
| Azure Key Vault | Cloud KMS | Supported |
| GCP KMS | Cloud KMS | Supported |
| HSM via PKCS#11 | Hardware Security Module | Supported |

---

## Non-functional requirements

Every OpenKCM CMK component — regardless of deployment variant — must meet the following bar:

- **Testability** — independently testable without a full stack; integration points abstracted
- **Observability** — structured logging, metrics, and tracing built in from day one
- **API stability** — published APIs are contracts; new capabilities extend, never break
- **Security by default** — authentication, authorization, and audit logging are requirements, not features added later
- **Operability** — operable by someone who did not build it; health endpoints, runbooks, graceful degradation

---

## Terminology

**OpenKCM**
The open source project and repository. The umbrella under which all components are developed and maintained — including OpenKCM CMK, OpenKCM CMK Platform Mesh, and Krypton.

**OpenKCM CMK**
The full-featured, standalone Customer Managed Key product. Runs on any Kubernetes environment with no platform dependency. Includes the full CMK UI, CMK Application, and CMK Core. Designed for organizations operating their own infrastructure who need complete key governance under their own control.

**OpenKCM CMK Platform Mesh**
The lightweight CMK layer for Platform Mesh. Includes an embedded microfrontend (via Luigi) and a CMK Controller for tenant onboarding and key lifecycle management within a Platform Mesh deployment. Relies on Platform Mesh for audit, RBAC, and notifications — does not require a separate CMK Application deployment.

**CMK Core**
The shared business logic library consumed by both OpenKCM CMK and OpenKCM CMK Platform Mesh. Contains the key lifecycle state machine (NIST SP 800-57), L1 key validation, kill switch execution logic, tenant registry, audit trail logic, and multi-party approval logic. Deployment-agnostic — no dependency on any platform or cloud provider.

**CMK Controller**
A Kubernetes controller that runs inside Platform Mesh (in the platform-admin namespace). Watches OpenKCM CRDs and reconciles key state across the workspace. Used exclusively by OpenKCM CMK Platform Mesh — not part of the standalone OpenKCM CMK deployment.

**Krypton**
The cryptographic execution engine. Performs key derivation, encryption, decryption, and KMIP services. Operates independently of OpenKCM CMK — it can run without CMK governance on top. OpenKCM CMK governs; Krypton executes.
