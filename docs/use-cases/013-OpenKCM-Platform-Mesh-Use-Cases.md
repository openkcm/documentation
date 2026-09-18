---
status: Proposal- can be confirmed after review
version: 1
authors:
  - Product Management
audience: OpenKCM team
---

# OpenKCM — Platform Mesh Use Cases

## Purpose

This document covers the use cases for OpenKCM specifically within a Platform Mesh deployment. It is a companion to `openkcm-use-cases.md`, which covers the broader portfolio independent of Platform Mesh.

Use cases are presented from the perspective of each role involved — what they experience, what they control, and what is transparent to them.

---

## Role Definitions

These role definitions align with Platform Mesh's own terminology as defined in the [Account Model documentation](https://documentation.apeirora.eu/best-practices/platform-mesh/account-model).

**Service Consumer**
An organization with an account on Platform Mesh. They consume services from the marketplace and own their workspace.

**Service Provider (MSP)**
Another organization on the same Platform Mesh installation, operating in their own isolated workspace/account. The MSP publishes their service via APIExport and the service consumer binds to it via APIBinding within the same KCP instance. They build and operate the service on their own infrastructure. They may integrate with OpenKCM/Krypton to offer encryption as part of their service offering.

**Root Key Authority**
A role within the service consumer's organization responsible for configuring encryption through the administration layer UI. The Root Key Authority is the primary initiating human role for L1 registration and binding — they initiate these actions, which then go through an approval workflow where other authorized approvers may also interact. They register the L1 key and authorize its binding to an encryption boundary. In some cases (UC1) they also define the boundary — in others (UC3) the boundary is revealed through encryption activity and the Root Key Authority only authorizes the L1 binding.

---

## Key Concept: Encryption Boundary

When this document refers to an **encryption boundary**, it means the **L2 boundary** — the logical scope under which a single L2 key is derived and all data within that scope is protected.

An encryption boundary is an isolated logical scope under which one L2 key is derived. Key material follows the data — wherever the data lives, the key lives. The service consumer does not need to declare data residency requirements explicitly. In cases where an encryption boundary spans more than one region or location, the administration layer ensures that the L2 key material does not cross regional boundaries.

An encryption boundary can span:
- More than one service or system instance
- More than one region

**Types of encryption boundary:**

- **EB-1 — Service instance** — one service instance (e.g. one database) = one encryption boundary. The most granular option.
- **EB-2 — Namespace-scoped boundary** — all services in a namespace share one encryption boundary. Only valid when the namespace maps to a single runtime/data boundary. In Platform Mesh, namespaces are virtual and can span multiple regions — this boundary type must not be used when the namespace spans more than one region.
- **EB-3 — System** — a named logical group registered in the administration layer, defined by the service consumer or service provider (depending on the use case), spanning multiple services within the governed scope — potentially across multiple locations within the same compliance region.

These boundary types apply across all use cases. The use case variants reference them by type.

---

## Key Concept: Region

When this document refers to a **region**, it means a **compliance region** — a governance and data residency boundary (e.g. Germany, EU, US). Encryption boundaries must not cross compliance regions.

Within a compliance region, data may be distributed across multiple locations (e.g. Berlin and Dresden within Germany) for geo-redundancy or performance. An encryption boundary can include all these locations — the boundary spans them as long as they remain within the same compliance region. Key material follows the data and does not cross the compliance region boundary.

---

## What OpenKCM Is

**OpenKCM** = Administration layer + OpenKCM Runtime.

**OpenKCM Runtime** contains Krypton as the key-hierarchy engine. Services and platforms integrate with the OpenKCM Runtime — not with Krypton directly. Krypton is the engine behind that boundary.

**Krypton** is the platform-neutral key-hierarchy and cryptographic engine inside the OpenKCM Runtime. It executes all key operations. By itself it has no governance model, no approval workflows, no business-level audit trail, and no permission management. It is a framework, not a product.

**The administration layer** is the governance component — the component that makes Krypton a product. It provides the business-level audit trail, the managed L1-to-system coupling workflow, and the visual administration layer for the service consumer's encryption landscape. The service consumer brings their own IDP; the administration layer maps that IDP's roles to key governance operations independently of the platform provider.

---

## Use Cases

### UC1 — Pure PaaS
*A service consumer (developer team) implements their own application on their account. They define the encryption boundary themselves — there is no tenant concept here.*

**Who is involved:**
- **Root Key Authority** — registers the L1 key (HYOK) and links it to the encryption boundary through the administration layer
- **Service consumer** — operates the workload (e.g. a self-deployed database instance) on a cluster, either their own or one provisioned by the platform

These can be the same person or different people within the same organization.

**What they experience:**
The Root Key Authority registers their L1 key (HYOK — the key stays in their own external keystore, outside OpenKCM entirely) through the administration layer. The Root Key Authority defines the encryption boundary — the type is chosen from EB-1, EB-2, or EB-3 depending on how the application is structured. A default boundary type can be configured — if no boundary is explicitly specified, the administration layer applies the default. The default boundary policy must itself be configured and approved by the Root Key Authority or governance policy in advance. The Root Key Authority can override the default per boundary. Once the boundary is defined, the Root Key Authority explicitly links the L1 key to it through the administration layer. The linking goes through an approval workflow — once approved, the L2 key for that boundary is wrapped with the service consumer's L1 key.

**Variants:**
The Root Key Authority can define the encryption boundary as any of the three types defined above — EB-1 (instance), EB-2 (namespace), or EB-3 (System).

**Note:** The L1-to-boundary coupling is a deliberate business decision that must go through an approval workflow. Without the administration layer, this coupling can be done by anyone with cluster access via a Kubernetes resource — with no approval, no audit record, and no governance.

---

### UC2 — Service Consumer Building an Application
*A service consumer is building their own application (e.g. pet shop) on Platform Mesh and using marketplace services as building blocks.*

**Who is involved:**
- **Root Key Authority** — registers the L1 key (HYOK) and links it to the encryption boundary
- **Service consumer** — builds and runs the application, uses services from the marketplace
- **Service provider** — offers a managed service (e.g. MongoDB) on the marketplace

**What they experience:**
The Root Key Authority registers their L1 key (HYOK) through the administration layer. The service consumer builds their application using marketplace services. When ordering a service instance (e.g. a database) from the marketplace, the service consumer specifies the encryption boundary context (EB-1, EB-2, or EB-3) as a parameter to the service instance at creation time. The marketplace service (e.g. MongoDB) uses this context to request the L3 key from the OpenKCM Runtime.

**Variants:**

- **UC2.1 — Manual L3 configuration / non-native integration** — the service has NOT integrated with OpenKCM (vanilla service). The service consumer manually creates the L3 key and provides the key reference to the service configuration. The encryption boundary context EB-1, EB-2, or EB-3 is specified by the consumer. This variant is operationally heavy and is not the target architecture — it is an interim path until the service provider integrates natively with OpenKCM.

- **UC2.2 — OpenKCM-native service integration** — the service (e.g. Krypton-native DB) has integrated with the OpenKCM Runtime. The service automatically requests an L3 key from the OpenKCM Runtime, propagating the encryption boundary context as a parameter from the service consumer. The Root Key Authority only needs to link their L1.

Both variants apply to all three boundary types (EB-1, EB-2, EB-3).

**Note:** The service provider does not decide the encryption boundary — they receive it as a parameter. The Root Key Authority always holds the L1 kill switch regardless of variant.

---

### UC3 — Service Consumer Subscribing to a SaaS Application
*A service consumer goes to the marketplace, subscribes to a SaaS application (e.g. pet shop), and gets a tenant. The encryption boundary is always EB-3 — the tenant. It appears in the administration layer when something is actually encrypted for that tenant — not necessarily before. The service consumer does not define the boundary.*

**Who is involved:**
- **Root Key Authority** — links their L1 key to the tenant that appears in the administration layer
- **Service consumer** — subscribes to the SaaS application, receives a tenant
- **Service provider (pet shop)** — multi-tenant SaaS application, manages the service

**From the service consumer's perspective:**
The service consumer subscribes to the SaaS application from the marketplace. They receive a tenant. This tenant is the encryption boundary EB-3 — it appears in the administration layer when the first encryption activity happens for that tenant. The Root Key Authority authorizes the L1 binding to this tenant — they do not define the boundary, it is revealed through encryption activity. From that point, all data in this tenant is encrypted under the service consumer's key. They hold the kill switch.

**From the service provider's (pet shop) perspective:**
The pet shop is a multi-tenant SaaS application. When a service consumer subscribes, a tenant is created. The tenant is the encryption boundary (EB-3). The pet shop then handles its own services using UC2 patterns — either provider managed L3 or consumer managed L3 — always passing EB-3 (the tenant) as the boundary context. One administration layer governs all tenants.

**Audit trail:**
- **Provider/internal audit logs** — operational logs of key operations (wrap, unwrap, create). These are Krypton-level logs for provider operations.
- **Customer/external audit trail** — the governance trail visible to the service consumer. It must show: L1 binding, approval events, revocation, all affected encryption boundaries, and which Root Key Authority and approvers authorized each decision. This is the compliance-grade audit trail produced by the administration layer — not the operational logs.

**Note:** The provider does the work, but the service consumer holds the key. Each consumer's L1 never touches the provider's infrastructure. The provider can operate the service for all consumers but cannot decrypt any consumer's data.

---

### UC4 — Sovereign / Air-Gapped
*UC4 is a deployment profile, not a separate functional use case. UC1, UC2, and UC3 all apply within a sovereign deployment — the difference is that the entire stack (Platform Mesh, Krypton, administration layer, marketplace services) runs within the service consumer's own perimeter. One organization may operate the platform, the MSPs, and consume services — all within the same sovereign boundary. No external dependencies, no internet calls, no provider cooperation outside the perimeter.*

**Why this matters:** The strongest form of sovereignty OpenKCM enables. Not "your key is in a region near you" but "your key is on your hardware, in your building, under your control."

---

