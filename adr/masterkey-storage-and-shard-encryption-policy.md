# ADR-204: MasterKey Storage and Shard Encryption Policy

| Status | Date | Document Type |
| :--- | :--- | :--- |
| **Active** | 2026-01-16 | Architecture Design Record |

## Context
The **MasterKey** is the root of trust for the OpenKCM Crypto Core. While previous records define how the key is reconstructed or auto-unsealed, a strict policy is required to govern the **persistence of the encrypted root material** and the **encryption standards for human-held shards**.

The Risk: If the encrypted MasterKey blob or the Shamir Secret Sharing (SSS) shards are stored with weak encryption or inadequate metadata, the system becomes vulnerable to offline brute-force attacks or "shard substitution" attacks where an attacker replaces a legitimate shard with one they control.

The Requirement: Every piece of the root secret—whether a sealed blob or a Shamir shard—must be protected by high-entropy encryption and bound to a specific cluster identity.

## Decision
We will implement a mandatory **Root Persistence & Shard Protection Standard** using AES-256-GCM and Participant Public Key Encryption (PPKE).

### MasterKey Blob Storage (Seal Mode)
When operating in **Auto-Unseal Mode**, the encrypted MasterKey blob is stored in the regional database.
* **Encryption:** The blob must be encrypted using **AES-256-GCM**.
* **Binding:** The `Cluster_ID` and `Region_ID` must be included as **Authenticated Additional Data (AAD)**. This prevents a database backup from one region being used to bootstrap a cluster in another region.
* **Metadata:** The storage record must include the `KMS_Key_Version` and `Timestamp` of the seal operation.

### Shard Encryption Policy (SSS Mode)
In **Shamir Secret Sharing Mode**, shards are never distributed to humans in plaintext. Every shard generated during the "Key Ceremony" must be encrypted for a specific recipient.

* **Mandatory PPKE:** Each shard must be wrapped using the **RSA-4096** or **Ed25519** public key of the designated shard holder.
* **Shard Identity:** Every shard must contain a signed header identifying the **Shard Index**, the **Cluster Fingerprint**, and the **Threshold (M)** required for reconstruction.
* **Storage of Shard Blobs:** While humans hold the "Green" unwrap material, the Crypto Core stores the **Wrapped Shard Blobs** in its local database. This allows the system to verify the integrity of a submitted shard before attempting reconstruction.

### Key Ceremony Logs
Every time the MasterKey is generated, split, or rotated, a **Ceremony Manifest** is created:
* **Integrity:** The manifest contains a hash of the *Encrypted* shards (not the plaintext).
* **Audit:** It records which public keys were used to encrypt each shard, providing a non-repudiable trail of who participated in the root bootstrap.



## Consequences

### Positive (Pros)
* **Offline Security:** Even if the database is leaked, the MasterKey remains secure as shards are encrypted for specific individuals and the Seal Blob is bound to the Cloud KMS.
* **Integrity Guarantee:** AAD prevents moving root secrets between environments (e.g., from Production to Development).
* **Non-Repudiation:** The use of participant public keys ensures that only authorized, identified individuals can provide the material required to unseal the cluster.

### Negative (Cons) & Mitigations
* **Public Key Management:** Requires operators to maintain their own PGP or SSH keys for the unsealing ceremony.
    * *Mitigation:* The CMK Sovereign Portal provides a "Public Key Registry" where admins can upload their keys during onboarding.
* **Rotation Complexity:** Rotating shard holders requires a new Key Ceremony.
    * *Acceptance:* This is an intentional security gate to ensure the "Root of Trust" is always held by current, authorized personnel.

## References
* [ADR-201: MasterKey Immutability & Volatile Memory](masterkey-definition-and-immutability-standard.md)
* [ADR-202: Shamir Secret Sharing for Human Unsealing](masterkey-reconstruction-via-shamir-secret-sharing.md)
* [ADR-203: Cloud-Native Auto-Unseal](cloud-native-auto-unseal-seal-wrapper.md)