# ACD-402: Global Resilience & Sovereign Reconciliation

| Status | Date | Document Type |
| :--- | :--- | :--- |
| **Active** | 2026-02-02 | Architecture Concept Design |

## Overview
**Global Resilience** in OpenKCM defines how the platform survives catastrophic infrastructure failures without compromising cryptographic sovereignty.

This document clarifies the **Split-Lifecycle Responsibility** model. While the Global Registry orchestrates the "Birth" and "Death" of tenants, the Regional Krypton Core is the sole authority for their "Life" (Rotation and Maintenance).

## The Resilience Philosophy: "Registry Orchestrates, Core Maintains"
OpenKCM divides responsibilities to ensure that routine security maintenance (Rotation) is never blocked by global control plane availability.

### 1. The CMK Registry (Lifecycle Orchestrator)
* **Role:** The "Conductor" of the platform.
* **Responsibility:** **Tenant Lifecycle Triggers (Create/Delete).**
  * **Onboarding:** When a new Tenant is defined, the Registry instructs the Core to "Initialize L2/L3 Structure" (Create).
  * **Offboarding:** When a Tenant is removed, the Registry instructs the Core to "Purge L2/L3 Structure" (Delete).
  * **Scope:** It defines *who* exists (Tenant IDs) and *where* they exist (Region Allow-List).
* **Constraint:** The Registry triggers the *existence* of tenants but does not manage key versioning, rotation schedules, or L1 Pointers.

### 2. The Krypton Core (Maintenance Authority)
* **Role:** The "Engine" of the platform.
* **Responsibility:** **Key Maintenance & Rotation.**
  * Once a tenant is initialized, the Core owns the key timeline.
  * **Rotation:** The Core's internal scheduler triggers L2 & L3 rotation based on policy (e.g., "Rotate every 90 days").
  * **Persistence:** The Core persists the rotated blobs in its local Regional Vault via Plugins.

## Disaster Recovery Tiers

### Tier 1: Regional Autonomy (Network Partition)
*Scenario: The fiber cut between the Global Registry (CMK) and the Regional Core (e.g., EU-Central) isolates the region.*

* **Tenant Lifecycle (Create/Delete):** **Paused.**
  * The Region cannot receive instructions to onboard new tenants.
  * Existing tenants cannot be offboarded (keys remain active until connectivity restores).
* **Key Maintenance (Rotation):** **Unaffected.**
  * The **Krypton Core** continues to execute scheduled rotations for L2 and L3 keys.
  * It interacts directly with the **External L1 Provider** (via Plugins) to wrap new versions.
  * **Zero Dependency:** Rotation succeeds even if the Registry is unreachable for days.
* **L4 Operations:** **Unaffected.**
  * The Gateway continues to function using the Core's local state.

### Tier 2: Region Failure (Cluster Loss)
*Scenario: The entire `us-west-2` Regional Core is destroyed.*

* **Failover:** Traffic routes to a secondary region (e.g., `us-east-1`).
* **Tenant Hydration:** The survivor region queries the **CMK Registry** to get the **Tenant Definition** (Identity & Region Scope).
* **State Recovery:** The survivor region reconstructs the L2/L3 hierarchy using the definition provided by the Registry and the material available in the shared/replicated storage.

### Tier 3: Global Registry Failure (Brain Death)
*Scenario: The central CMK database is corrupted or unreachable.*

* **Impact on Crypto:** **Zero.**
  * Regional Cores function normally for all existing tenants.
  * **L2/L3 Rotations continue on schedule** because the Registry is not involved in rotation.
* **Impact on Management:**
  * **Frozen State:** No new tenants can be created. No existing tenants can be deleted.
  * The platform is statically locked but cryptographically active and secure.

## Summary
ACD-402 defines a resilience model where **Availability of Management** (Registry) is decoupled from **Availability of Security** (Core). By isolating **Rotation** as a purely regional function, OpenKCM ensures that the most critical security operation—keeping keys fresh—continues even during total global control plane failure.