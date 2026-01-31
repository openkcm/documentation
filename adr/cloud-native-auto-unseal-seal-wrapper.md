# ADR-203: Cloud-Native Auto-Unseal (Seal Wrapping)

| Status | Date | Document Type |
| :--- | :--- | :--- |
| **Active** | 2026-01-16 | Architecture Design Record |

## Context
While Shamir's Secret Sharing (ADR-202) provides the highest level of sovereignty for air-gapped or regulated deployments, it introduces a "Human Bottleneck." In a modern Cloud-Native environment (Kubernetes, Auto-Scaling Groups), pods and services may restart frequently due to updates, crashes, or rebalancing.

**The Problem:** If every pod restart requires 3 humans to manually input shard keys, the "Mean Time To Recovery" (MTTR) becomes hours instead of milliseconds. This invalidates the promise of high-availability SaaS.

**The Requirement:** We require a mechanism that allows the OpenKCM Crypto Core to **automatically decrypt its own MasterKey** upon startup, provided it is running in a trusted, authorized cloud environment.

## Decision
We will implement **Cloud-Native Auto-Unseal** (also known as "Seal Wrapping") by delegating the root trust to a trusted external Cloud KMS.

### The Mechanism: Envelope Encryption for Bootstrapping
Instead of splitting the MasterKey into human-held shards, the MasterKey is generated once and immediately **Wrapped (Encrypted)** by a specific key residing in the Cloud Provider's KMS (AWS KMS, Azure Key Vault, or GCP KMS).

* **The Anchor:** The Cloud KMS Key (e.g., `arn:aws:kms:us-east-1:123456789012:key/openkcm-boot-key`).
* **The Payload:** The Encrypted MasterKey Blob (stored in the OpenKCM Database).
* **The Identity:** The Cloud IAM Role assigned to the Virtual Machine or Pod (e.g., `role/openkcm-crypto-node`).

### The Boot Sequence
When a Crypto Core node starts up in "Auto-Unseal Mode":

1.  **Identity Verification:** The node proves its identity to the Cloud Provider (via AWS Instance Profile, Azure Managed Identity, or GCP Workload Identity).
2.  **Fetch Blob:** The node retrieves the *Encrypted MasterKey Blob* from its local configuration or database.
3.  **Remote Decrypt:** The node sends the blob to the Cloud KMS `Decrypt` endpoint.
    * *Security Check:* The Cloud Provider checks IAM policies: "Is this specific VM allowed to decrypt using the Boot Key?"
4.  **Load to Memory:** If authorized, the Cloud KMS returns the Plaintext MasterKey over the TLS connection. The node immediately loads it into `mlock` protected memory.
5.  **Ready State:** The service transitions to `Active` and begins processing requests.



### Supported Seal Providers
To ensure cloud-agnosticism, the Auto-Unseal interface must support pluggable "Seal Wrappers":

* **AWS KMS:** Uses `kms:Decrypt` with optional Encryption Context for tighter binding.
* **Azure Key Vault:** Uses Key Vault unwrapping.
* **GCP Cloud KMS:** Uses Cloud KMS asymmetric decryption.
* **HashiCorp Vault (Transit):** Allows an on-premise OpenKCM cluster to auto-unseal by calling a *central* upstream Vault service (acting as the KMS).
* **TPM (Trusted Platform Module):** For bare-metal gateway devices, allows unsealing only if the hardware signature matches (Secure Boot).

### Security Posture
In this model, the "Root of Trust" is effectively **Identity Access Management (IAM)**.
* If an attacker steals the database (the Encrypted MasterKey Blob), they cannot decrypt it because they are not running inside the authorized AWS Role.
* If an attacker steals the physical hard drive, they cannot decrypt it for the same reason.

## Consequences

### Positive (Pros)
* **Zero-Touch Operations:** Pods can crash, restart, and scale out horizontally without human intervention.
* **High Availability:** Recovery time is limited only by the network latency to the Cloud KMS (typically <50ms).
* **Cloud Native Alignment:** Leverages existing trusted infrastructure (IAM Roles) rather than inventing new authentication schemes.

### Negative (Cons) & Mitigations
* **Cloud Lock-In:** The deployment becomes tightly coupled to the specific cloud provider's KMS availability.
    * *Mitigation:* Use "Transit Unseal" (using an upstream Vault) to abstract the cloud provider if multi-cloud portability is required.
* **Compromised Identity:** If an attacker gains "Remote Code Execution" (RCE) on the pod, they inherit the IAM role and can request the MasterKey.
    * *Mitigation:* Strict `IPAddress` conditions in IAM policies and comprehensive CloudTrail logging of `Decrypt` events.

## References
* [ADR-201: MasterKey Immutability](masterkey-definition-and-immutability-standard.md)
* [ADR-202: Shamir Secret Sharing](masterkey-reconstruction-via-shamir-secret-sharing.md)
* [HashiCorp Vault: Auto Unseal](https://developer.hashicorp.com/vault/docs/concepts/seal#auto-unseal)