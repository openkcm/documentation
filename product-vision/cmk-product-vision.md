---
status: Draft
last_updated: 2026-06-24
audience: Open Source Community, Contributors, Stakeholders
---

# OpenKCM Product Vision

## What is OpenKCM CMK?

OpenKCM CMK is an open source Customer Managed Key (CMK) layer for cloud-native platforms. It gives organizations cryptographic control over their own encryption keys — so that no platform operator, cloud provider, or managed service can access customer data without the customer's explicit consent.

OpenKCM CMK governs encryption keys. The actual cryptographic operations are performed by [Krypton](https://github.com/openkcm/krypton) — the crypto execution engine that OpenKCM CMK sits on top of.

---

## The Problem

When an organization runs workloads on a cloud platform, their data is typically encrypted — but by a key the platform controls. The organization trusts the platform not to look. That trust is implicit, unverifiable, and often not sufficient for regulated industries.

Most organizations also operate across multiple cloud providers, on-premise systems, and HSMs — each with its own key management interface. Keys end up scattered with no unified view, no single audit trail, and no single point of revocation.

OpenKCM CMK solves both: it inverts control so the customer owns the root key and the platform never touches key material, and it acts as a single control plane across all keystores — AWS KMS, Azure Key Vault, GCP KMS, OpenBao, HSMs — managed from one UI, one audit trail, one kill switch.

When access needs to be revoked, it is immediate. No waiting periods, no platform approval, no delay — the customer triggers the kill switch and all governed workloads become inaccessible instantly. Most commercial KMS products enforce mandatory delays of 7 to 90 days before key destruction. OpenKCM CMK does not.

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

| | OpenKCM CMK | OpenKCM CMK Platform Mesh |
|---|---|---|
| Deployment | Any Kubernetes cluster | Platform Mesh only |
| UI | Full standalone web interface | Embedded microfrontend in Platform Mesh portal |
| L2 provisioning | Customer via UI | CMK Controller (automatic, one per account/workspace) |
| L3 management | Customer via UI | Security admin via UI |
| L4 management | Customer via UI or consuming service via KMIP | Consuming service via KMIP only |
| Tenant management | Customer via UI | Platform Mesh account model |
| Audit | Built-in audit trail | Via Platform Mesh audit infrastructure |
| RBAC | OIDC / RBAC | Via Platform Mesh OpenFGA |
| Kill switch scope | Tenant | Org (Model 1) or account (Model 2) — see deployment models |

Full product details:
- [OpenKCM CMK — Standalone](cmk-standalone-vision.md)
- [OpenKCM CMK — Platform Mesh](cmk-platform-mesh-vision.md)

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
For OpenKCM CMK Platform Mesh: the kill switch scope depends on the deployment model — org level (all accounts in the organization) when enabled at org level, or account level (all namespaces in that account) when enabled at account level.

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
A Kubernetes controller that watches OpenKCM CRDs and reconciles key state. Placement depends on the deployment model — at org level when OpenKCM is enabled at org level (governs all accounts), or at account level when enabled at account level (governs that account only). Used exclusively by OpenKCM CMK Platform Mesh — not part of the standalone OpenKCM CMK deployment.

**Krypton**
The cryptographic execution engine. Performs key derivation, encryption, decryption, and KMIP services. Operates independently of OpenKCM CMK — it can run without CMK governance on top. OpenKCM CMK governs; Krypton executes.
