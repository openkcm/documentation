---
authors:
  - Nicolae Nicora
---

# Separation of Governance (CMK) and Execution (Crypto)

**Status:** Accepted  
**Date:** 2026-01-12  

## Context
OpenKCM must support multi-region deployments and multi-cloud keystore integration while adhering to strict data residency requirements. 
We require a system that maintains a single source of truth for tenant governance without compromising the localized nature of cryptographic execution.

### The Problem: Tight Coupling
A monolithic approach where governance (tenant metadata, L1 references, job orchestration) and execution (L2/L3 key lifecycle, KMIP operations) are intertwined creates several critical risks:
* **Blast Radius:** A vulnerability or DoS in the KMIP-exposed execution layer could compromise global tenant metadata and governance integrity.
* **Latency & Throughput:** High-frequency L4 DEK operations would be gated by centralized governance checks, creating a performance bottleneck.
* **Compliance:** Strict data residency laws demand that cryptographic material (L2/L3 keys) be generated, stored, and used within specific geographic boundaries, while tenant governance remains globally consistent.
* **Scalability:** Execution load must scale independently without affecting the stability of the administrative control plane.

---

## Decision
Adopt a strict **Control Plane / Data Plane separation** model. We will decouple the **OpenKCM CMK** (Governance Plane) from the **OpenKCM Crypto** (Execution Plane).

### OpenKCM CMK (Governance / Control Plane)
**Role:** Central authority and single source of truth for tenant and key governance.
* **Responsibilities:**
  * Tenant onboarding, workspace linking, and lifecycle events.
  * L1 CMK configuration and key reference management (ARNs, URIs, aliases).
  * Mapping of "Systems" / applications to L1 keys.
  * UI/API surface for customer self-service (BYOK/HYOK onboarding, revocation simulation).
  * Policy enforcement for tenant ↔ system ↔ key relationships.
* **Security & Exposure:**
  * Does **not** handle L4 DEK keys or bulk encryption/decryption payloads.
  * Does **not** expose KMIP endpoints.
  * Communicates with the Crypto plane only via asynchronous job/task queues (message broker).

### OpenKCM Crypto (Execution / Data Plane)
**Role:** Regional cryptographic worker nodes that perform all sensitive key operations and serve high-volume requests.
* **Deployment Model:**
  * Regional clusters (e.g., `eu-west-1`, `us-east-1`) with localized PostgreSQL and optional OpenBao storage.
* **Responsibilities:**
  * Full **KMIP 1.4 server** implementation (Create, Get, Encrypt, Decrypt, Rotate).
  * Generation, storage, rotation, and retirement of **L2 Tenant Keys** and **L3 Service Keys**.
  * Envelope wrapping/unwrapping of L2/L3 keys using the MasterKey (SSS or Seal).
  * High-throughput L4 DEK generation via Crypto Edge nodes.
  * **Regional Autonomy:** Continues to serve crypto requests even if the CMK Governance plane is temporarily unreachable.
* **Security & Exposure:**
  * Exposes KMIP **only** to authorized Crypto Edge nodes via mandatory mTLS.
  * Never communicates directly with external KMS/HSM providers (delegated to the CMK plane).
  * Receives configuration and job updates asynchronously via a message broker.


## Authorization & Boundary Enforcement
To maintain strict separation, we enforce the following communication boundaries:
* **CMK → Crypto:** One-way, asynchronous job/task dispatch. No direct synchronous RPC allowed.
* **Crypto → CMK:** Strictly read-only status reporting (reconciliation feedback).
* **Crypto Edge → Core Crypto:** KMIP over mTLS only; authorization is strictly certificate-bound.
* **No Bypass:** No component is authorized to bypass the message broker for cross-plane communication.


## Consequences

### ✅ Positive (Pros)
* **High Availability & Regional Autonomy:** The Crypto plane continues serving encryption/decryption requests during CMK plane outages or maintenance.
* **Data Residency Compliance:** L2/L3 keys never leave their designated region—satisfies strict sovereign data laws.
* **Independent Scalability:** The execution plane can be scaled horizontally in high-demand regions without affecting the stability of the governance plane.
* **Blast Radius Containment:** A compromise of the KMIP-exposed execution plane cannot directly access or corrupt global tenant metadata or L1 references.

### ⚠️ Negative (Cons) & Mitigations
* **Eventual Consistency Delay:** Governance changes (e.g., tenant onboarding) are not instantaneous.
  * *Mitigation:* Orbital reconciliation is designed for low-latency sync (seconds to minutes); UI shows “pending” status.
* **Increased Infrastructure Complexity:** Requires a reliable message broker, regional DBs, and reconciliation logic.
  * *Mitigation:* Use proven components (NATS/RabbitMQ, PostgreSQL, Orbital) and automate via GitOps.
* **Operational Monitoring Overhead:** Need visibility into reconciliation lag and cross-plane health.
  * *Mitigation:* Implement end-to-end correlation IDs and real-time alerting on stalled tasks.


## Alternatives Considered
1. **Fully Centralized Model:** Rejected due to residency violations, latency bottlenecks, and single point of failure.
2. **Fully Decentralized:** Rejected as it is impossible to maintain consistent tenant-to-key mapping across regions.
3. **Synchronous RPC:** Rejected as it creates tight coupling, increases blast radius, and violates residency requirements.

**Final Path:** Strict asynchronous Control Plane / Data Plane separation with eventual consistency via Orbital. This enables OpenKCM to meet enterprise-grade compliance, scale, and security requirements while minimizing operational risk.