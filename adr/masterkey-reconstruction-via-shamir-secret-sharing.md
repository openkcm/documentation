# ADR-202: MasterKey Reconstruction via Shamir Secret Sharing

| Status | Date | Document Type |
| :--- | :--- | :--- |
| **Active** | 2026-01-16 | Architecture Design Record |

## Context
The **MasterKey** is the root of trust for the OpenKCM Crypto (Krypton). While Cloud-Native deployments can use "Auto-Unseal" (ADR-203), high-security Sovereign deployments (Banking, Defense, Offline/Air-Gapped) cannot rely on a single external Cloud Provider (AWS/Azure) to bootstrap the trust chain.

**The Risk:** If we rely on a single administrator to hold a recovery key, that administrator becomes a single point of failure (bus factor) and a high-value target for coercion (insider threat).

**The Requirement:** We require a **Quorum-Based Unsealing Mechanism**. The system must require the cooperation of multiple distinct human operators to reconstruct the MasterKey in memory, without any single operator ever possessing the full key.

## Decision
We will implement **Shamir's Secret Sharing (SSS)** algorithm to split the MasterKey into multiple shards.

### The Splitting Logic (Initialize)
During the cluster initialization ("Key Ceremony"), the MasterKey is generated in memory and immediately split into $N$ shards, with a reconstruction threshold of $M$ ($M \le N$).

* **Algorithm:** Shamir's Secret Sharing (GF($2^8$)).
* **Parameters:**
    * $N$ (Total Shards): e.g., 5.
    * $M$ (Threshold): e.g., 3.
* **Distribution:** Each shard is output *once* to the operator (via PGP-encrypted email, QR Code, or Hardware Token) and then deleted from the server's memory.
* **Persistence:** The server stores **Encrypted Shard Blobs** (Red Keys) in the database to validate the shards later, but it *cannot* decrypt them without the human inputs.



### The Unsealing Logic (Reconstruct)
When the Crypto (Krypton) service restarts in "Sealed Mode":
1.  **State:** The service enters a `Sealed` state. API returns `503 Service Unavailable (Sealed)`.
2.  **Input:** Operators visit the `SysAdmin Portal` or use the CLI to submit their individual shards.
    * Operator A submits Shard #1.
    * Operator B submits Shard #3.
    * Operator C submits Shard #5.
3.  **Reconstruction:** Once the $M$-th shard is received, the Crypto (Krypton) combines them in memory to mathematically reconstruct the MasterKey.
4.  **Verification:** The derived MasterKey is hashed and compared to a stored `MasterKey_Hash`.
5.  **Activation:** If valid, the MasterKey is locked into RAM (`mlock`), and the service transitions to `Active`.

### Shard Protection
To prevent "Copy-Paste" attacks where a shard is stolen from a laptop:
* **Encryption:** Shards distributed to humans are typically encrypted with the Operator's PGP Public Key.
* **Hardware Tokens (Optional):** For "Platinum" tier, shards are written to YubiKeys or Smart Cards, requiring physical possession to submit.

## Consequences

### Positive (Pros)
* **No Single Point of Compromise:** An attacker must coerce/compromise $M$ distinct administrators to steal the MasterKey.
* **Resilience:** If $N-M$ administrators lose their keys or leave the company, the remaining $M$ can still recover the system.
* **Sovereignty:** This mechanism works perfectly in an air-gapped bunker with no internet connection to AWS/Azure.

### Negative (Cons) & Mitigations
* **Operational Friction:** Restarting a cluster (e.g., after an OS patch) requires coordinating 3 humans to wake up and input keys.
    * *Mitigation:* Use this mode strictly for "Root CA" or "High-Security" clusters. Use Auto-Unseal for standard SaaS web tiers.
* **Ceremony Complexity:** The initial generation "Key Ceremony" must be strictly audited.
    * *Mitigation:* Provide a specific `openkcm init --pgp-keys=...` CLI tool to automate the secure distribution.

## References
* [ADR-201: MasterKey Immutability](masterkey-definition-and-immutability-standard.md)
* [Shamir, A. (1979). "How to share a secret"](https://dl.acm.org/doi/10.1145/359168.359176)