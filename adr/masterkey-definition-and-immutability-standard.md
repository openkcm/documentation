# ADR-201: MasterKey Immutability & Volatile Memory

| Status | Date | Document Type              |
| :--- | :--- |:---------------------------|
| **Active** | 2026-01-14 | Architecture Design Record |

## Context
The **MasterKey** is the root of the internal trust hierarchy for the OpenKCM Crypto Core. It is the "Key of Keys" used to wrap the Internal Versioned Keys (IVKs), which in turn wrap the Tenant Keys (L2).

**The Risk:** If the MasterKey is compromised, the entire region's data is compromised.
**The Constraint:** To prevent offline attacks, forensic recovery from stolen hard drives, or swap-file analysis, the MasterKey must **never** be persisted to non-volatile storage (Disk, SSD, Flash) in plaintext.

We require a strategy that ensures:
1.  **Immutability:** The key cannot be changed once loaded (except via full rotation).
2.  **Volatility:** The key exists *only* in RAM and vanishes instantly upon power loss or process termination.
3.  **Resilience:** The system can recover the key after a restart using either automated (Cloud) or manual (Sovereign) methods.

## Decision
We will implement a **Strict Volatile Memory Standard** coupled with a **Dual-Mode Bootstrap Strategy**.

### Volatile Memory & Core Dump Protection
The MasterKey is a "Green Key" (Plaintext) only when it resides in the active memory of the `crypto-core` process. To protect this memory:

* **`mlock` (Memory Locking):** The process must immediately invoke the `mlock()` (or `VirtualLock` on Windows) system call on the memory buffer holding the MasterKey. This prevents the OS from swapping this memory page to disk (swap file) during high-pressure scenarios.
* **Core Dump Disable:** The process must set the `RLIMIT_CORE` to `0` at startup to prevent the OS from writing the process memory to disk in the event of a crash.
* **Zeroization:** On `SIGTERM` or `SIGINT`, the application must intercept the signal and overwrite the MasterKey memory address with zeros before exiting.

### Bootstrap Mode A: Shamir Secret Sharing (SSS)
*Designed for Sovereign, Air-Gapped, or Multi-Cloud deployments.*

The MasterKey is mathematically split into $N$ shards using Shamir's Secret Sharing ($M$-of-$N$ threshold).
* **Storage:** Each shard is encrypted by a *different* external provider (AWS, Azure, GCP, HSM) to prevent any single cloud provider from deriving the master key.
* **Reconstruction:** On startup, the service pauses and waits for a "Key Ceremony" where operators (or automated sidecars) submit $M$ shards via the API.
* **Use Case:** Banking, Defense, and scenarios requiring independence from a single cloud vendor.

### Bootstrap Mode B: Cloud Seal (Auto-Unseal)
*Designed for Cloud-Native SaaS and Kubernetes.*

The MasterKey is generated once and immediately encrypted (Sealed) by a trusted Cloud KMS (e.g., AWS KMS Key `alias/openkcm-boot`).
* **Storage:** The *Encrypted* MasterKey Blob is stored in the local PostgreSQL database.
* **Auto-Unseal:** On startup, the service authenticates via IAM (Instance Profile / Workload Identity), calls `kms:Decrypt` on the blob, and loads the resulting plaintext directly into `mlock` memory.
* **Use Case:** High-scale, stateless auto-scaling groups where manual intervention is impossible.

## Architecture Diagrams

![crypto-masterkey.png](.images/crypto-masterkey.png)

### SSS Mode (Sovereign Reconstruction)
```mermaid
flowchart LR
    MK[MasterKey (RAM)]
    SSS[Shamir Reconstruction]
    
    subgraph "Encrypted Shards (Persistence)"
        S1[Shard 1 (AWS Encrypted)]
        S2[Shard 2 (Azure Encrypted)]
        S3[Shard 3 (GCP Encrypted)]
        S4[Shard 4 (HSM Encrypted)]
    end

    S1 -->|Decrypt & Submit| SSS
    S2 -->|Decrypt & Submit| SSS
    S3 -->|Decrypt & Submit| SSS
    S4 -->|Decrypt & Submit| SSS
    
    SSS -->|M-of-N Threshold Met| MK
    style MK fill:#b7f7b7,stroke:#2d7a2d,stroke-width:2px
```

### Seal Mode (Cloud Native)
```mermaid
flowchart LR
    MK[MasterKey (RAM)]
    KMS[Cloud KMS / HSM]
    DB[(Encrypted Blob)]

    DB -->|Fetch Blob| KMS
    KMS -->|Decrypt (IAM Auth)| MK
    style MK fill:#b7f7b7,stroke:#2d7a2d,stroke-width:2px
```

## Consequences

### ✅ Positive (Pros)
* **Defense Against Forensics:** Stealing the physical server or snapshotting the disk yields only encrypted blobs (Red Keys), never the MasterKey.
* **Operational Flexibility:** We can support a Bank (using SSS with smart cards) and a Startup (using AWS Auto-Unseal) with the same binary.
* **Cloud Agnostic:** SSS Mode allows us to survive the total outage of AWS by relying on shards stored in Azure/GCP.

### ⚠️ Negative (Cons) & Mitigations
* **Operational Overhead (SSS):** SSS requires manual intervention or complex orchestration to reboot a cluster.
  * *Mitigation:* Use "Seal Mode" for standard tiers and reserve "SSS Mode" for Platinum/Sovereign tiers.
* **Memory Pressure:** Locking memory (`mlock`) reduces the available RAM for other processes.
  * *Mitigation:* The MasterKey is small (256-bit + overhead). The impact is negligible, but `ulimit` settings in Kubernetes must allow for memory locking.

## References
* [Shamir, A. (1979). "How to share a secret"](https://dl.acm.org/doi/10.1145/359168.359176)
* [HashiCorp Vault: Seal/Unseal Concepts](https://developer.hashicorp.com/vault/docs/concepts/seal)