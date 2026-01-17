# ACD-402: Global Resilience & Sovereign Reconciliation

| Status | Date | Document Type |
| :--- | :--- | :--- |
| **Active** | 2026-01-16 | Architecture Concept Design |

## Overview
**Global Resilience** in OpenKCM defines how the platform survives catastrophic infrastructure failures without compromising cryptographic sovereignty or data consistency. In a distributed key management system, "Resilience" often conflicts with "Security" (e.g., failing open vs. failing closed).

This document details the **Sovereign Reconciliation Strategy**, ensuring that regional Crypto Core clusters can operate autonomously during network partitions while guaranteeing that global governance intent (e.g., a "Revoke" command) is eventually and reliably enforced.

## The Resilience Philosophy: "Local Trust, Global Command"
OpenKCM is designed as a **Loose Federation of Autonomous Regions**.
* **The Brain (CMK Registry):** Lives in a primary region (e.g., `us-east-1`). It holds the "Desired State."
* **The Limbs (Crypto Cores):** Live in edge regions (e.g., `eu-central-1`, `ap-northeast-1`). They hold the "Actual State."

If the connection between the Brain and the Limbs is severed, the Limbs **continue to function** for existing authorized workloads but **fail safe** for lifecycle changes.



## Disaster Recovery Tiers

### Tier 1: Regional Autonomy (Network Partition)
*Scenario: The fiber cut between the US and Europe isolates the EU Crypto Core from the Global Registry.*
* **Read Operations (L4 Decrypt):** **Unaffected.** The EU Core has the L2 and IVK keys cached in its local secure memory and local KSI. It continues to serve KMIP traffic.
* **Write Operations (L4 Encrypt):** **Unaffected.** New DEKs can be generated locally.
* **Governance Operations (L2 Rotation/Revocation):** **Paused.** The local node cannot confirm the current status of the tenant's L1 key with the central registry. It defaults to the last known good state (cached policy) for a configurable TTL (e.g., 1 hour), after which it fails closed.

### Tier 2: Region Failure (Cluster Loss)
*Scenario: The entire `us-west-2` Crypto Core cluster is destroyed.*
* **Data Plane:** KMIP traffic fails over to the nearest healthy region (e.g., `us-east-1`).
* **Key Hydration:** The survivor region (`us-east-1`) queries the **Global Registry** for the tenant's metadata.
* **Re-Bootstrap:** The survivor region connects to the Customer's L1 KMS (which is multi-region by default in AWS/Azure) to unwrap the L2 key.
* **Result:** Zero data loss. The L2 key is mathematically identical, just loaded into a different region's memory.

### Tier 3: Global Registry Failure (Brain Death)
*Scenario: The central CMK database is corrupted or unreachable.*
* **Safety Protocol:** Regional Cores enter **"Standby Mode."**
* **Capabilities:** They continue to serve *existing* keys indefinitely based on their local snapshot.
* **Limitation:** No *new* tenants can be onboarded, and no *new* L2 keys can be generated.
* **Recovery:** The Registry is restored from an immutable, encrypted backup. **Orbital** then pushes a "State Resync" event to all regions to reconcile any drift.

## The Orbital Reconciliation Loop
Resilience is powered by **Orbital**, the state synchronization engine (ACD-201).

1.  **Snapshotting:** Every 5 minutes, the Regional Core takes a cryptographic snapshot of its active Key State (list of loaded L2s, active IVK versions).
2.  **Heartbeat:** This snapshot hash is sent to the Global Registry.
3.  **Diff Calculation:** The Registry compares the Regional Hash with its Authoritative Hash.
4.  **Remediation:** If a discrepancy is found (e.g., Region has an L2 that the Registry says should be revoked), Orbital issues a high-priority **"Force Sync"** command to the region.



## Sovereign Conflict Resolution
In a "Split-Brain" scenario where the Region thinks a key is valid but the Registry says it is revoked, **Security Trumps Availability.**

* **The Rule:** If the Region cannot reach the Registry to validate an L2 key's status after the TTL expires, the key is **Disabled**.
* **Why:** It is better to cause a service outage than to allow a revoked tenant to decrypt data. This preserves the "Sovereign Kill Switch" guarantee.

## Summary
ACD-402 defines a resilience model that prioritizes **Data Integrity and Sovereignty** over blind availability. By empowering regions to act autonomously for routine operations while strictly enforcing global governance for state changes, OpenKCM ensures that a localized failure never compromises the global trust model.