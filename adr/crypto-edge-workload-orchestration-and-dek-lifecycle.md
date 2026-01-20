# ADR-602: Crypto Edge Workload Orchestration and DEK Lifecycle

| Status | Date | Document Type |
| :--- | :--- | :--- |
| **Active** | 2026-01-17 | Architecture Design Record |

## Context
The **OpenKCM Crypto Edge** is the "Fast Twitch Muscle" of the platform. Unlike the Core, which manages thousands of stable KEKs (L3), the Edge deals with **millions of ephemeral DEKs (L4)** every minute.

**The Problem:**
1.  **Latency:** If the Edge calls the Core for every single data record, the network round-trip (20-50ms) kills application throughput.
2.  **Scale:** A single application pod can generate 10,000 requests/sec. The Core cannot handle this write volume globally.
3.  **Consistency:** We need to ensure that an L4 key generated at the Edge is cryptographically bound to the correct L3 key without needing a synchronous database write.

## Decision
We will implement an **Autonomous Edge Orchestrator** that manages a **High-Velocity L4 Lifecycle** using local caching and asynchronous batching.



### The Edge Orchestration Model
The Edge operates on a "Lease & Batch" philosophy.

1.  **L3 Caching (The Lease):**
    * On startup (or first request), the Edge requests the **L3 Service Key** from the Core.
    * The Core returns the *Wrapped L3*. The Edge cannot unwrap it (it has no MasterKey).
    * *Correction:* The Edge establishes an **mTLS Session** with the Core. The Core sends the **Plaintext L3 Key** over the secure channel to the Edge's memory.
    * The Edge caches this L3 key for a short TTL (e.g., 5 minutes).

2.  **L4 Generation (The Hot Path):**
    * **Request:** App calls `CreateKey(TenantA)`.
    * **Action:** Edge generates 32 bytes of random entropy (AES-256).
    * **Wrap:** Edge uses the cached L3 key to encrypt this new L4 locally (AES-GCM).
    * **Response:** Returns `Wrapped_L4_Blob` to the App.
    * **Latency:** < 0.5ms (Zero network calls).

3.  **L4 Decryption (The Read Path):**
    * **Request:** App calls `DecryptKey(Wrapped_L4_Blob)`.
    * **Action:** Edge unwraps the blob using the cached L3 key.
    * **Response:** Returns `Plaintext_L4`.

### The "No-State" Principle
The Edge is **Stateless**. It does *not* store L4 keys in a local database.
* **Storage:** The Application is responsible for storing the `Wrapped_L4_Blob` alongside its data (in MySQL/S3).
* **Recovery:** If the Edge crashes, it simply restarts, re-fetches the L3 key from the Core, and resumes unwrapping the blobs presented by the Application.

## Consequences

### Positive (Pros)
* **Infinite Scalability:** Throughput is limited only by local CPU (AES-NI instructions), not by database IOPS or network bandwidth.
* **Simplicity:** No distributed consensus (Raft/Paxos) is needed at the Edge because the Edge doesn't persist state.
* **Resilience:** If the Core goes down, the Edge can continue to serve requests for the duration of its L3 Cache TTL (e.g., 5-60 mins), allowing the system to ride out control plane outages.

### Negative (Cons)
* **Revocation Lag:** If an L3 key is revoked at the Core, the Edge might continue using its cached copy for up to 5 minutes.
    * *Mitigation:* Orbital pushes a "Revocation Event" to the Edge to bust the cache immediately (Eventual Consistency < 1s).
* **Key Exposure:** The L3 key exists in the Edge's memory. If the Edge pod is compromised (memory dump), the L3 is exposed.
    * *Mitigation:* The L3 key is typically scoped to a specific service (not the whole tenant), limiting the blast radius.

## References
* [ACD-203: Crypto Edge – High-Performance Ephemeral Data Plane](../acd/crypto-edge–high-performance-ephemeral-data-plane.md)
* [ADR-601: Crypto Core Central Orchestration](crypto-core-central-orchestration-and-kek-lifecycle.md)