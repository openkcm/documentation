---
authors:
  - Aysan
status: Draft
last_updated: 2026-04-30
audience: Product leadership, investors, ApeiroRA stakeholders
---

# OpenKCM UI Strategy: From Standalone to Embedded

---

## The Question This Document Answers

> *"We invested in the CMK UI. Why doesn't it fit anymore? And how will OpenKCM have a UI for open source users if we're moving to Platform Mesh?"*

This document answers both questions directly.

---

## Why the Legacy CMK UI No Longer Fits

### 1. The CMK UI was built before Krypton existed

The standalone CMK UI was built when there was no crypto execution layer. At the time, CMK was the entire product — governance, key management, tenant registry — all in one place with its own REST API, its own database, its own portal.

This was the right decision at the time. You build with what you have.

Then Krypton came.

### 2. Krypton changed the architecture

Krypton introduced a fundamentally different model:

| CMK UI model (old) | Krypton model (new) |
| :--- | :--- |
| Key Configurations (flat) | Tiered key hierarchy: L1 → L2 → L3 → L4 |
| Systems assigned to key configs | ServiceKey CRs per workload |
| Custom REST API | KMIP protocol + Kubernetes CRDs |
| PostgreSQL as source of truth | etcd as source of truth |
| Standalone portal | Platform Mesh native |

The CMK UI understands "key configurations" and "systems." Krypton uses L1 root keys, L2 domain keys, L3 service keys, and L4 data encryption keys. These are not the same model. The old UI cannot represent the new architecture — it would require a complete rewrite to make it work with Krypton.

### 3. Rebuilding the CMK UI for Krypton would cost more than building new

Adapting the legacy CMK UI to the Krypton model means:
- Rewriting the key creation wizard to understand L1/L2/L3
- Rebuilding the system assignment model as ServiceKey CRs
- Replacing the REST API client with Kubernetes API calls
- Replacing the flat key list with a hierarchical key chain view
- Completing the Luigi integration that was never finished

At that point, you have rebuilt the entire UI from scratch — inside a legacy codebase, against a UI framework (UI5) that was chosen for a different context. That is more expensive and more risky than building the MicroFrontend correctly from the start.

---

## The CMK UI Investment Was Not Wasted

The CMK UI is the most complete specification of what OpenKCM's governance layer needs to do.

Every screen, every workflow, every edge case that had to be thought through to build it — the key creation wizard, the Four-Eyes approval flow, the system assignment model, the lifecycle states, the RBAC model — is now the blueprint for the new MicroFrontend.

**We didn't waste the investment. We used it to discover exactly what needs to be built.**

That is product development. The CMK UI answered the question: *what does an OpenKCM governance UI actually need?* The MicroFrontend answers the question: *how do we build that correctly?*

---

## Open Source Still Gets a UI

Moving to a Platform Mesh native MicroFrontend does not mean open source users have no UI. It means they get a better one.

### The MicroFrontend is portable

The CMK MicroFrontend is a React application. Luigi is the shell that hosts it inside the Platform Mesh portal — but the UI itself is framework-agnostic. The same application can be:

- **Embedded in Platform Mesh** via Luigi for ApeiroRA customers
- **Shipped as a standalone portal** for open source users who don't have Platform Mesh

One codebase, two deployment modes.

### The old CMK UI could not serve open source users either

The legacy CMK UI talked to the standalone CMK REST API. That API is tied to the old architecture — PostgreSQL registry, custom approval engine, 8 microservices. An open source user deploying OpenKCM would have to run all 8 services just to get the UI working. That is not a viable open source deployment model.

The MicroFrontend talks to the CMK Controller — one Kubernetes controller. That is a viable open source deployment model.

---

## The Timeline of Decisions

Understanding why each decision was made requires understanding the sequence:

| When | What happened | Why |
| :--- | :--- | :--- |
| Early | CMK UI built as standalone portal | No Platform Mesh, no Krypton — standalone was the only option |
| Later | Krypton designed with L1-L4 tiered model | Crypto architecture established, KMIP, Kubernetes-native |
| Later | CMK UI becomes misaligned with Krypton | Different data model, different API, different deployment target |
| Later | Platform Mesh matures | Native CRDs, Luigi, approval mechanisms, MSP pattern available |
| Now | CMK MicroFrontend direction decided | Build on the right foundation — portable, Krypton-native, open source ready |

At no point was the CMK UI a mistake. Each decision was correct given what was known at the time. Krypton changed the architecture. Platform Mesh changed the deployment model. The right response is to adapt.

---

## Summary

| Question | Answer |
| :--- | :--- |
| Was the CMK UI investment wasted? | No — it defined the product and is the blueprint for the MicroFrontend |
| Why doesn't the legacy UI fit? | Krypton introduced a tiered key model the old UI cannot represent |
| What about open source users? | The MicroFrontend is portable — it can be deployed standalone, outside Platform Mesh |
| Is the new UI a replacement? | Yes — same governance capabilities, built on the right architecture |
| Is this more investment? | No — rebuilding the legacy UI for Krypton would cost more than building new |
