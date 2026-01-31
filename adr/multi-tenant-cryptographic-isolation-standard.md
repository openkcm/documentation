# ADR-103: Multi-Tenant Isolation & Pluggable Storage Strategy

| Status | Date | Document Type |
| :--- | :--- | :--- |
| **Active** | 2026-01-16 | Architecture Design Record |

## Context
OpenKCM must guarantee strict multi-tenant isolation across its entire topology: the **Governance Plane (CMK)**, the **Execution Core**, and the **Execution Gateway**.

**The Risk:** Hardcoding storage backends (e.g., "Always use Vault" or "Always use Memory") creates vendor lock-in and limits deployment flexibility. Some customers require L4 keys to vanish instantly upon process termination (Memory), while others require local survivability (Encrypted Disk).

**The Requirement:** We require a **Unified Pluggable Storage Strategy** that enforces isolation while allowing the underlying storage medium to be swapped based on the environment (Cloud vs. On-Prem vs. Gateway).

## Decision
We will implement a **Layered Isolation Strategy** utilizing **PostgreSQL Schemas** for Governance and **Pluggable Storage Interfaces (PSI)** for both Crypto (Krypton) and Crypto (Krypton) Gateway.

### 1. Governance Isolation (CMK Service)
The OpenKCM CMK Registry (Governance Layer) will utilize a **Schema-per-Tenant** architecture within PostgreSQL.

* **Implementation:**
  * Every Tenant is assigned a dedicated PostgreSQL Schema (e.g., `tenants.t_acme_corp`).
  * The schema serves as a "Virtual Database." All governance tables exist identically within each schema.
  * **Connection Routing:** The Database Abstraction Layer identifies the tenant context and sets the `search_path` to that specific schema.
* **Benefits:** Atomic lifecycle management (Drop Schema) and prevention of SQL injection cross-tenant leakage.

### 2. Crypto (Krypton) Isolation (L2/L3 Keys)
The OpenKCM Crypto (Krypton) (Regional Execution Layer) will utilize a **Core Key Storage Interface (KSI)** to manage persistent L2/L3 keys.

**The Interface Contract:**
The Core logic talks to a `CoreStorageProvider` interface. The implementation maps the OpenKCM `Tenant_ID` to the native isolation construct of the backend.

**Supported Core Plugins:**

| Plugin Implementation | Isolation Mechanism | Use Case |
| :--- | :--- | :--- |
| **HashiCorp Vault / OpenBao** | **Namespaces**. Maps `TenantID` $\to$ `ns/openkcm/<tenant>/`. | High-Security Enterprise / Banking. |
| **Cloud KMS (AWS/Azure)** | **Tags & IAM**. Maps `TenantID` $\to$ Resource Tags. | Cloud-Native SaaS. |
| **Encrypted SQL Blob** | **Table Partitioning**. Maps `TenantID` $\to$ Partitioned Tables. | Simplified / Developer Mode. |

### 3. Crypto (Krypton) Gateway Isolation (L4 Keys)
The OpenKCM Crypto (Krypton) Gateway (Sidecar/SDK Layer) will utilize an **Gateway Key Storage Interface (EKSI)** to manage ephemeral L4 keys.

**The Interface Contract:**
The Gateway Sidecar does not assume it is running in a specific environment (Pod, VM, Lambda). It delegates L4 key caching/storage to a `EdgeStorageProvider` plugin.

**Supported Gateway Plugins:**

| Plugin Implementation | Storage Behavior | Use Case |
| :--- | :--- | :--- |
| **Volatile Memory (Default)** | **`mlock` RAM**. Keys exist only in locked process memory. Lost on restart. | Highest Security / Ephemeral Containers. |
| **Encrypted Filesystem** | **Local Disk / Tmpfs**. Keys are wrapped and written to a local volume. | Survivability across process restarts (e.g., VM reboot). |
| **Shared Cache (Redis)** | **Remote Cache**. Keys are encrypted and stored in a shared Redis cluster. | High Availability for stateless web farm fleets. |

**Gateway Isolation Logic:**
Regardless of the plugin, the Gateway enforces **Namespace Isolation**:
* If using **Memory**: Separate maps for separate Tenants (if serving multi-tenant traffic).
* If using **Filesystem**: Separate sub-directories per Tenant ID (`/data/keys/<tenant_id>/`).
* If using **Redis**: Key prefixes (`openkcm:cache:<tenant_id>:key_xyz`).

## Consequences

### Positive (Pros)
* **Environment Adaptability:** We can deploy the Gateway Sidecar in a "Zero-Trust" bank environment (using the **Memory Plugin** so no keys ever touch disk) or in a "High-Resilience" IoT environment (using the **Filesystem Plugin** to survive reboots).
* **Vendor Agnosticism:** The Core is not locked into HashiCorp Vault. Customers can bring their own storage backend.
* **Granular Control:** We can configure the Core to use Vault for storage while configuring the Gateway to use purely volatile RAM, optimizing for both persistence security and runtime speed.

### Negative (Cons) & Mitigations
* **Configuration Complexity:** Operators must explicitly configure storage drivers for both Core (`config.core.storage`) and Gateway (`config.gateway.storage`).
  * *Mitigation:* Ship with sensible defaults: `Core=Vault` and `Gateway=Memory`.
* **Consistency Challenges:** Implementing the `EdgeStorageProvider` for distributed systems (like Redis) introduces eventual consistency risks.
  * *Mitigation:* The Gateway interface treats all L4 keys as immutable or short-lived, reducing sync issues.

## References
* [ADR-101: Separation of Governance and Execution](separation-of-governance-cmk-and-execution-crypto.md)
* [HashiCorp Vault: Secrets Engines](https://developer.hashicorp.com/vault/docs/secrets)