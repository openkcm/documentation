# ADR-101: The "Church & State" Separation of Governance and Execution

| Status | Date | Document Type |
| :--- | :--- | :--- |
| **Active** | 2026-01-14 | Architecture Design Record |

## Context
OpenKCM functions as a "Value Engine" for sovereign SaaS, which introduces the **"Trust Paradox"**:

* **Sovereignty Requirement:** Customers demand absolute control over their L1 Root Keys and the ability to revoke access instantly (The "Kill Switch").
* **Performance Requirement:** SaaS applications require sub-millisecond encryption (L4) for billions of records across global regions.

**The Problem: Tight Coupling**
A monolithic architecture where the same service handles "Governance" (L1 Management) and "Execution" (L4 Encryption) creates unacceptable risks:
* **Privacy Violation:** If the Governance layer handles encryption, it effectively possesses the keys, negating the promise that "The Provider cannot see my data."
* **Blast Radius:** A DDoS attack on the public API (Governance) could paralyze the internal encryption engine (Execution).
* **Latency Physics:** A central Governance authority cannot physically service cryptographic requests from gateway locations (e.g., Sydney to N. Virginia) fast enough for real-time applications.
* **Residency Laws:** Data residency laws (GDPR, Schrems II) often mandate that key material *never* leaves a specific jurisdiction, while Governance metadata must be globally viewable.

## Decision
We will adopt a strict **Control Plane / Data Plane Separation** model, structurally decoupling the system into two distinct domains:

### OpenKCM CMK (Governance Plane)
**Role:** The single source of truth for "Who owns what." It manages the relationships between Tenants, Systems, and External L1 Keys without ever touching the key material itself.

* **Responsibilities:**
  * **Sovereign Portal:** UI/API for customer BYOK/HYOK onboarding and "Kill-Switch" revocation.
  * **Policy Registry:** Stores the immutable mapping of `Tenant_ID` $\to$ `L1_Key_ARN`.
  * **Orbital Command:** Publishes "Desired State" events (e.g., "Rotate Key", "Revoke Access") to the distributed mesh.
* **Security Boundary:**
  * **Zero Knowledge:** It stores *references* (ARNs, KeyIDs) but **never** accesses or stores plaintext key material.
  * **No Crypto Operations:** It does not expose KMIP or encryption endpoints.

### OpenKCM Crypto (Execution Plane)
**Role:** Distributed, autonomous regional nodes that execute the "Desired State." They perform the actual cryptographic lifting.

* **Responsibilities:**
  * **Recursive Unsealing:** Connects *directly* to the Customer's External KMS (AWS/Azure/GCP) to unwrap L2 Keys using the credentials provided by the CMK policy.
  * **Secure Memory:** Holds unsealed L2/L3 keys in `mlock` protected memory (RAM) only.
  * **KMIP Server:** Provides the high-throughput KMIP 1.4 interface for SaaS workloads.
  * **Regional Autonomy:** Can continue serving cached keys even if the CMK Governance Plane goes offline.
* **Security Boundary:**
  * **Ephemeral Existence:** If the process restarts, all keys are lost and must be re-fetched/re-unsealed.
  * **Isolation:** Never speaks to the CMK directly; receives instructions strictly via the Orbital Event Bus.

## Communication & The "Orbital" Link
To enforce this separation, we introduce **Orbital**, an asynchronous reconciliation engine.

* **CMK $\to$ Crypto:** **Asynchronous Events.** The CMK publishes a `TenantConfigurationUpdated` event. The Crypto node consumes it and aligns its local state.
* **Crypto $\to$ CMK:** **Status Reports.** The Crypto node publishes `HealthCheck` or `RotationComplete` events.
* **Exceptions (The Priority Lane):** "Revocation" events are tagged as `PRIORITY_CRITICAL` in Orbital to bypass standard queue backpressure, ensuring near-instant global lockout.

## Consequences

### Positive (Pros)
* **Liability Shift:** Because the CMK (Governance) never touches key material, the platform provider minimizes liability. We manage the *map*, not the *treasure*.
* **Residency Compliance:** We can deploy a Crypto Node in `eu-frankfurt` that *only* holds keys for German tenants. Even if the global CMK is in the US, the keys never leave Germany.
* **Performance:** Encryption happens at the Gateway (Crypto Node), millimeters away from the workload, while Governance happens centrally.
* **Resilience:** A failure in the Governance Portal does not stop the SaaS platform from encrypting/decrypting data (using cached keys).

### Negative (Cons) & Mitigations
* **Complexity of State:** "Eventual Consistency" means there is a non-zero lag between a user clicking "Save" and the key being active globally.
  * *Mitigation:* Orbital is tuned for sub-second propagation. The UI reflects "Pending" states until Crypto nodes confirm the `SyncAck`.
* **External KMS Dependency:** The Crypto Node needs direct network access to AWS KMS/Azure KV to unseal.
  * *Mitigation:* Use strict egress filtering and PrivateLink endpoints where possible to secure this traffic.

## Alternatives Considered
* **Centralized Proxy:** Routing all encryption calls through the CMK.
  * *Rejected:* Would introduce massive latency and violate data residency laws (keys moving across borders).
* **CMK-Side Unsealing:** The CMK unwraps the key and sends the plaintext to Crypto.
  * *Rejected:* Violates the "Zero Knowledge" principle. If the CMK database dump contains plaintext keys (or can derive them), we lose the "Trust Paradox" value proposition.

**Final Decision:** Strict **Governance vs. Execution** separation is the foundational architecture of OpenKCM.