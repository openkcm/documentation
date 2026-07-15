---
authors:
  - Aysan
date: 2026-07-15
last_updated: 2026-07-15
status: Draft
---

# OpenKCM — Platform Mesh Deployment Model

## Overview

OpenKCM is deployed at **account level** on Platform Mesh. Each account enables OpenKCM independently, manages its own L1 key, and runs its own CMK Controller scoped to that account. The platform provider has zero visibility into any account's key governance.

---

## Use Case — Shared Platform Provider: Multiple Independent Organizations as Tenants

**Who:** A company offering a managed Platform Mesh installation to multiple independent organizations (e.g. a managed service provider or systems integrator offering Platform Mesh as a service).

**The situation:** The platform provider operates the infrastructure. Each tenant is a fully independent organization with its own account on Platform Mesh. Each organization has strict requirements that the platform provider cannot access their data or their key governance. The platform provider and the tenant are different legal entities.

**Why the keychain matters:** Each organization enables OpenKCM at account level independently. Each organization's CMK Controller is scoped entirely to their own account. The platform provider has zero visibility into any tenant's key governance — this is enforced by the Platform Mesh account model and OpenFGA, not just policy.

**The critical distinction from a single-operator model:** The platform provider does not touch OpenKCM at all — they provide the infrastructure; each tenant brings or enables their own OpenKCM. "Cannot read tenant data" is a structural guarantee of the deployment topology, not a product feature the platform provider configures.

**What OpenKCM provides (per tenant):**
- Each tenant enables OpenKCM at their account level — CMK Controller scoped to their account only
- Tenant registers their own L1 (BYOK or HYOK) — platform provider has no reference to it
- L2 provisioned for that account, L3 per service — fully isolated from all other tenants
- Kill switch scope: that account only — does not affect other tenants or the platform provider
- Audit logs belong to the tenant — platform provider cannot access them

**Deployment model:** Each tenant runs their own CMK Controller scoped to their account, L1 in each tenant's own keystore.

**L4 path:** Tenant's choice — each organization manages their own L4 independently. KMIP self-registration for standard workloads. UI-managed L4 for tenants with per-customer data isolation requirements within their own services.

**The one-liner:** *A platform provider whose tenants are independent organizations — each owns their own keys, and the provider is structurally excluded from key governance.*
