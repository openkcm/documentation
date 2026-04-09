# Krypton — Business Case

## Executive Summary

Every organization that stores data faces the same fundamental tension: **encrypt everything to stay secure and compliant, without slowing down applications, losing control of keys, or getting locked into a single cloud vendor.**

Today's key management solutions force a compromise. Cloud-native KMS services (AWS KMS, Azure Key Vault) are fast to adopt but lock you in and hold your keys. Hardware Security Modules (Thales, Fortanix) give you control but are expensive and rigid. Open-source solutions (HashiCorp Vault) offer flexibility but lack the performance and structured key hierarchy that large organizations need.

**Krypton solves this.** It is the open-source cryptographic execution layer of OpenKCM — a vendor-neutral, KMIP-standard key management system that is Kubernetes-ready but not Kubernetes-bound. It runs on Kubernetes, bare-metal, VMs, or any container runtime, delivering sub-millisecond encryption, true customer key sovereignty, and a flexible key hierarchy that adapts to organizations of any size.

---

## The Problem

### 1. Vendor Lock-In Is the Norm

Every cloud KMS has a proprietary REST API. If you integrate your application with AWS KMS today, switching to Azure Key Vault or an on-premise HSM tomorrow means rewriting your encryption code, your key management logic, and your audit pipeline. Your keys — and by extension your data — are trapped in one provider's ecosystem.

### 2. Encryption Is Too Slow to Use Everywhere

Traditional KMS solutions require a network call for every encrypt/decrypt operation — 20 to 200ms round-trip. At scale (millions of operations per second), this makes encrypting 100% of data at rest impractical. Most organizations compromise by encrypting only "sensitive" fields, leaving the rest exposed.

### 3. Key Hierarchies Are Rigid

Existing solutions impose a fixed key structure. AWS KMS gives you a flat model (one CMK wrapping DEKs). Enterprise products offer two or three layers at most. But real organizations have complex needs:

- A multinational bank needs separate key chains per geographic region for data sovereignty.
- A platform hosting hundreds of services needs per-service isolation of encryption keys.
- A small startup just needs one level of key wrapping — nothing more.

No existing KMS lets you **choose the depth of your key hierarchy** to match your organizational structure.

### 4. "Trust Us" Is Not Enough

When a cloud provider says "your keys are safe with us," the customer has no mathematical guarantee. The provider holds the plaintext key material. A government subpoena, a rogue insider, or a breach at the provider level could expose customer data. Regulated industries (banking, healthcare, defense, public sector) cannot accept this.

### 5. Key Rotation Means Downtime

Security standards (PCI-DSS, NIST) require regular key rotation. With traditional KMS solutions, rotating a master key triggers a batch re-encryption of all dependent keys and data — a massive, risky operation that requires maintenance windows and careful coordination.

---

## The Solution: Krypton

Krypton is the cryptographic execution layer that manages the full lifecycle of encryption keys — generation, wrapping, rotation, revocation, and destruction — using the OASIS KMIP standard and a unique split-execution architecture.

### Core Value Propositions

#### 1. Zero Vendor Lock-In — KMIP Standard

Krypton speaks KMIP (Key Management Interoperability Protocol), the OASIS international standard for key management. It does not invent a proprietary API.

**What this means for the customer:**

- Any KMIP-compatible database (MongoDB, MariaDB, VMware) connects with **zero code changes** — just configuration.
- Switching from Krypton to another KMIP-compliant provider (Thales, Fortanix) requires changing **one hostname**.
- No vendor-specific classes, SDKs, or API adapters in application code.

> _"We didn't invent a new API for you to lock into. We use the OASIS KMIP standard — integrate once, work with any provider. Switch vendors by changing a hostname, not your code."_

#### 2. Encrypt Everything — No Performance Tax

Krypton's **Split-Execution Architecture** separates key operations into two components:

- **Krypton Core** (regional): Holds L2/L3 keys in secure memory. Performs wrapping and unwrapping. Connects to the customer's external L1 key.
- **Krypton Gateway** (edge, sidecar): Runs next to the application. Handles L4 Data Encryption Key operations locally in sub-millisecond time.

| Operation    | Krypton            | AWS KMS           | HashiCorp Vault    |
| ------------ | ------------------ | ----------------- | ------------------ |
| Create DEK   | **<0.5ms** (local) | 20-50ms (network) | 50-200ms (network) |
| Encrypt data | **<0.5ms** (local) | 20-50ms (network) | 50-200ms (network) |
| Decrypt data | **<1ms** (cached)  | 20-50ms (network) | 50-200ms (network) |

> _"Encrypt 100% of your data at rest with sub-millisecond overhead. No performance compromises. No excuses not to encrypt."_

#### 3. Your Keys, Your Rules — Mathematically Guaranteed

The customer's root key (L1) **never leaves the customer's own HSM or cloud KMS**. Krypton holds only a reference (an ARN, a URL) — never the plaintext key material.

- Krypton calls the customer's KMS to wrap/unwrap — but cannot see the L1 key.
- If the customer disables L1 in their own console, Krypton **cannot decrypt anything**. Instantly. Globally.
- No amount of internal access, root privileges, or legal compulsion at the platform operator level can override this.

> _"We don't ask you to trust us. Your root key stays in your vault. We hold a reference, not the key. If we can't call your KMS, we can't decrypt. That's not a promise — it's math."_

#### 4. Flexible Key Hierarchy — Fits Any Organization

**This is what makes Krypton fundamentally different from other KMS solutions.**

Unlike traditional platform KMS approaches (which impose a fixed L2→L3→L4 chain) or AWS KMS (flat CMK→DEK), Krypton lets the organization **define its own keychain depth**:

**Large Enterprise (e.g., multinational bank with compliance per country):**

```
L1 (Customer Root — in customer's HSM)
  └─ L2 (Regional Key — EU)
       └─ L3 (Department Key — Risk Analytics)
            └─ L4 (Service Key — PostgreSQL cluster)
                 └─ DEK (Data Encryption Key — per-record)
  └─ L2 (Regional Key — APAC)
       └─ L3 (Department Key — Trading)
            └─ ...
```

Each department, each geographic region, each deployment gets its own **sub-keychain**, isolated from the others. The security administrator of the Risk Analytics department in the EU region cannot see or affect the Trading department in APAC.

**Small Company (e.g., a startup with one service):**

```
L1 (Customer Root)
  └─ L2 (Single key — wraps everything)
       └─ DEK (Data Encryption Key)
```

Just two levels. No unnecessary complexity.

**The customer decides:**

- How many layers the keychain has
- What each layer represents (region, department, service, environment)
- Which keys are isolated from which

> _"Krypton doesn't impose a key hierarchy. You design the keychain that matches your organization — from a two-level setup for a startup to a deep, geographically partitioned hierarchy for a multinational enterprise."_

#### 5. Top-Down Revocation — Cut Off Any Branch

Because the keychain is hierarchical and keys at each level are wrapped by the level above, revoking a key at **any level** instantly makes everything below it unreadable:

| Action                         | Effect                                                                                                             |
| ------------------------------ | ------------------------------------------------------------------------------------------------------------------ |
| Disable **L1**                 | **Global kill switch** — all data across all regions, all departments, all services becomes unreadable. Instantly. |
| Revoke **L2** (regional key)   | All departments and services in that region are locked out. Other regions are unaffected.                          |
| Revoke **L3** (department key) | One department's data is shredded. Other departments continue operating.                                           |
| Revoke **L4/DEK**              | A single service's or record's data is cryptographically destroyed.                                                |

The key material is never exposed downstream. When you revoke a key, the layers below **do not have the material to recover**. Even if someone created a secret using a key in that branch, they cannot decrypt it after revocation.

> _"Revoke from the top, shred to the bottom. One action, cryptographic certainty. No key material leaks, no cleanup needed, no trust required."_

#### 6. Provider-Agnostic L1 — Bring Any Root Key

Krypton's pluggable Keystore Interface (KSI) supports any L1 provider:

| Provider             | Type        | Status     |
| -------------------- | ----------- | ---------- |
| AWS KMS              | Cloud       | Supported  |
| Azure Key Vault      | Cloud       | Supported  |
| Google Cloud KMS     | Cloud       | Supported  |
| Thales Luna HSM      | Hardware    | Supported  |
| Securosys HSM        | Hardware    | Supported  |
| OpenBao (Vault fork) | Open-source | Supported  |
| Custom HSM           | Hardware    | Via plugin |

**Switch L1 providers at any time.** If you're on AWS KMS today and want to move to an on-premise HSM tomorrow, change the plugin configuration. The entire key hierarchy below L1 remains intact — no re-encryption, no downtime.

**Keystore Deployment Models:**

Krypton supports three deployment models for the customer's L1 key, matching different trust and operational preferences:

| Model                         | Who Manages L1                                                                    | Customer Control                                                   | Use Case                                                        |
| ----------------------------- | --------------------------------------------------------------------------------- | ------------------------------------------------------------------ | --------------------------------------------------------------- |
| **Provider Managed**          | Platform operator provisions and manages the L1 key on behalf of the customer     | Lowest — customer trusts the operator                              | Customers who want zero operational overhead                    |
| **BYOK (Bring Your Own Key)** | Customer generates L1 in their own KMS, imports it into Krypton's keystore        | Medium — customer controls generation, operator holds wrapped copy | Customers who want key provenance but accept managed operations |
| **HYOK (Hold Your Own Key)**  | Customer's L1 stays in the customer's own HSM/KMS; Krypton only holds a reference | Highest — L1 never leaves customer's vault                         | Regulated industries, data sovereignty, zero-trust requirements |

The **HYOK** model is Krypton's default and recommended mode — it provides the mathematical guarantee that the platform operator can never access plaintext key material.

> _"The underlying infrastructure is just compute. Bring your root key from AWS today, move it to an on-premise HSM tomorrow. We don't care which provider you use — and neither should you."_

#### 7. Zero-Downtime Key Rotation

Krypton's IVK (Internal Versioned Key) mechanism enables **lazy re-wrapping**:

1. New key version activates **instantly** — zero downtime.
2. Old keys re-wrap themselves **passively** as they're accessed.
3. No batch job, no IOPS spike, no maintenance window.

Krypton maps directly to the customer-facing key lifecycle operations:

| Customer Operation | Krypton Action                                                         | NIST SP 800-57 State    |
| ------------------ | ---------------------------------------------------------------------- | ----------------------- |
| **ENABLE**         | Key created and activated — ready for wrap/unwrap                      | Pre-Active → Active     |
| **ROTATE**         | New IVK version issued; old version lazy-re-wraps on access            | Active (new version)    |
| **DISABLE**        | Key suspended — decrypt-only, no new encryptions                       | Active → Deactivated    |
| **DELETE**         | Key material destroyed — all dependent data cryptographically shredded | Deactivated → Destroyed |

Every state transition is logged with a correlation ID, and the customer can trigger any of these through the CMK control plane or directly via Krypton's API.

> _"Rotate keys daily if you want. Zero downtime, zero batch jobs. Old keys re-encrypt silently in the background."_

#### 8. FIPS 140-2/3 Compliant Cryptography

Krypton's cryptographic operations use FIPS 140-2 validated libraries (and are designed for FIPS 140-3 readiness). All key generation, wrapping, and encryption use NIST-approved algorithms (AES-256-GCM, RSA-OAEP, HKDF).

- **In FIPS mode**, Krypton restricts itself to FIPS-approved algorithms only — no fallback to non-compliant ciphers.
- **L1 providers** (AWS KMS, Azure Key Vault, Thales Luna HSM) are themselves FIPS 140-2 Level 2/3 certified, extending the compliance chain from root to leaf.
- **Audit evidence**: Krypton logs which algorithm and key size was used for every operation, providing the documentation trail that auditors require.

This matters for regulated industries (banking, healthcare, defense, public sector) where FIPS compliance is not optional — it is a procurement prerequisite.

> _"FIPS compliance is not a checkbox we add later. The crypto primitives are FIPS from day one — AES-256-GCM, RSA-OAEP, NIST-approved algorithms only. No shortcuts, no fallbacks."_

#### 9. End-to-End Audit Logging

Every key operation in Krypton produces a structured, immutable audit event with a **correlation ID** that traces the action from the customer's API call through the CMK control plane, Krypton Core, and down to the Gateway.

| Audit Dimension    | What Is Logged                                                        |
| ------------------ | --------------------------------------------------------------------- |
| **Who**            | Authenticated identity (mTLS cert DN, OIDC subject, API key hash)     |
| **What**           | Operation type (Create, Activate, Revoke, Destroy, Get, Wrap, Unwrap) |
| **Which key**      | Key ID, key level (L1–L4), key version, parent key reference          |
| **When**           | Timestamp with microsecond precision (UTC)                            |
| **Where**          | Krypton Core region, Gateway instance, source IP                      |
| **Result**         | Success / failure with error code; NIST state transition              |
| **Correlation ID** | End-to-end trace ID linking the request across all components         |

Audit logs are designed for integration with SIEM systems (Splunk, Elastic, Azure Sentinel) and satisfy:

- **DORA Article 11** — ICT-related incident detection and reporting
- **NIS2 Article 23** — significant incident notification with forensic evidence
- **PCI-DSS Requirement 10** — track and monitor all access to cryptographic keys
- **GDPR Article 30** — records of processing activities involving encrypted data

> _"Every key operation is logged — who did what, to which key, when, from where, and whether it succeeded. One correlation ID traces the action end-to-end. Your auditors will thank you."_

#### 10. Red Button Dry-Run — Test Before You Shred

The top-down revocation capability (§5) is powerful — but irreversible actions demand confidence. Krypton provides a **dry-run mode** for the Red Button:

1. **Simulate revocation**: The customer selects a key level (L1, L2, or L3) and triggers a dry-run.
2. **Impact report**: Krypton returns a detailed blast radius — which services, how many DEKs, which tenants, and which data stores would be affected.
3. **No actual revocation**: No keys are disabled. No data becomes inaccessible. The operation is read-only.
4. **Confidence to act**: When the real incident hits, the customer already knows exactly what will happen.

This directly addresses the requirement for **Red Button testing capability** — the ability to validate the kill switch without triggering it.

> _"Test the kill switch before you need it. Dry-run any revocation — see the blast radius, confirm the scope, act with certainty when it matters."_

---

## Target Audiences

### Audience 1: Engineers & Security Architects

**Their question:** _"Is this just another hype product, or is there something substantial?"_

**The answer — through a concrete example:**

A security engineer at a multi-tenant platform (like Platform Mesh) needs to encrypt MongoDB data at rest. Today, they configure MongoDB to connect to Krypton Gateway via KMIP over mTLS:

```yaml
security:
  enableEncryption: true
  kmip:
    serverName: krypton.platform.example.com
    port: 5696
    clientCertificateFile: /certs/client.pem
    serverCAFile: /certs/ca.pem
```

MongoDB starts, requests an L4 DEK from Krypton via KMIP `Create`, receives a plaintext key, and encrypts all data at rest. The keychain behind this is:

```
Customer's AWS KMS (L1) → Tenant Root Key (L2) → Service KEK (L3) → MongoDB DEK (L4)
```

Now, an incident: the L3 key for this service is suspected compromised. Per NIST SP 800-57 compliance, the key is marked as **Suspended** — it can still **decrypt** existing data (allowing safe read operations) but can **no longer encrypt** new data. The security engineer rotates to a new L3, and new L4 DEKs are generated under the new chain. The old data remains readable during migration.

If the situation escalates and the L2 tenant key is revoked, **all services under that tenant** lose access immediately — MongoDB can no longer start, no data can be read or written. The customer can restore by re-enabling the key in their own KMS console.

**Key technical takeaways for engineers:**

- KMIP — no proprietary API, no vendor-specific code
- Sub-millisecond L4 operations at the Gateway
- NIST-compliant key lifecycle (Pre-Active → Active → Suspended → Deactivated → Compromised → Destroyed)
- Krypton handles the key chain; the service handles its own data cleanup (re-encryption on rotation)

### Audience 2: Enterprise Architects & CTOs

**Their question:** _"How does this fit a large organization with multiple departments, regions, and compliance requirements?"_

**The answer:**

Consider a multinational company with departments in different geographic regions, each with its own compliance requirements and data residency rules:

```
Corporate Root Key (L1 — in corporate HSM)
│
├── EU Region (L2)
│   ├── Finance Department (L3)
│   │   ├── HANA Database (L4) — German data residency
│   │   └── PostgreSQL (L4) — EU GDPR scope
│   └── R&D Department (L3)
│       └── MongoDB (L4) — R&D test data
│
├── US Region (L2)
│   ├── Sales Department (L3)
│   │   └── CRM Database (L4)
│   └── Operations (L3)
│       └── Redis Cache (L4)
│
└── APAC Region (L2)
    └── Manufacturing (L3)
        └── IoT Data Store (L4)
```

Each department's sub-keychain is **isolated**. The EU Finance team's keys are independent from APAC Manufacturing. A compromise in one branch cannot propagate to another. And yet, a single corporate root key governs the entire structure — the CISO can revoke everything from the top if needed.

Krypton creates and manages this **full hierarchical organization of keys** — as deep or as flat as needed. For a small company, just L1 → L2 → DEK. For an enterprise, as many layers as their organizational structure demands.

**Key takeaways for enterprise architects:**

- Key hierarchy mirrors organizational structure (regions, departments, services)
- Geographic isolation of key chains for data residency compliance
- Centralized governance (CMK control plane) with decentralized execution (Krypton Core per region)
- Scales from startup to multinational without architectural changes

### Audience 3: Decision Makers, Public Sector & EU Funding

**Their question:** _"How does this address data sovereignty and vendor independence?"_

**The answer:**

Krypton is built on a fundamental principle: **the underlying infrastructure is just compute.**

- The root key stays in the customer's own vault — not in a cloud provider's managed service.
- The key hierarchy can be replicated from one cloud provider to another (GCP → AWS → on-premise) because Krypton is provider-agnostic.
- The KMIP standard ensures no application code changes when moving between providers.
- The platform operator (whether a managed cloud, a public cloud, or a sovereign cloud) **never holds plaintext key material**.

This directly addresses:

| Requirement               | How Krypton Delivers                                                                                |
| ------------------------- | --------------------------------------------------------------------------------------------------- |
| **GDPR / Data Residency** | Keys stay in specified regions; regional L2 isolation ensures data cannot be accessed from outside  |
| **DORA / NIS2**           | Full audit trail with correlation IDs; instant revocation capability; key rotation without downtime |
| **Schrems II**            | Customer controls the root key — no US provider can be compelled to decrypt EU data                 |
| **Digital Sovereignty**   | No dependency on any single cloud provider; open-source; OASIS standard                             |
| **EU Public Funding**     | Open-source project; not bound to commercial vendor; technology can be audited and verified         |

> _"Data sovereignty is not a feature we bolt on — it's the architecture. The root key never leaves your control, the infrastructure is just compute, and the protocol is an open standard. No vendor lock-in, no trust assumptions, no political dependencies."_

---

## Customer Perspective

### Why Customers Need Krypton

Customer Managed Keys (CMK) is not a nice-to-have — it is the only means for customers to keep control over their data when they move to the cloud. In regulated industries, deals cannot be won without CMK or a credible roadmap to provide it. Krypton is the cryptographic execution layer that makes CMK real.

A **Red Button** describes what a customer ultimately wants: a button they can press to cryptographically lock all related business data. Not even the platform operator's technical staff can access the data anymore. The system becomes unavailable — a database will shut down if the customer key is missing. This is not something customers will do regularly in production, but what matters is that **they have the choice to do so**.

### Business Requirements

These are the real-world scenarios that drive adoption:

| #       | Business Requirement                 | What the Customer Needs                                                                                                                                     | How Krypton Delivers                                                                                                                                |
| ------- | ------------------------------------ | ----------------------------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------- |
| **BR1** | **Block access to data at any time** | No local authority should be able to force the platform operator or the underlying hyperscaler to provide customer business data.                           | L1 root key stays in the customer's own vault. Disable L1 → all data is instantly unreadable. No override possible.                                 |
| **BR2** | **Protection against cyber-attacks** | In case of a serious cyber-attack with an unknown security issue, the customer has the choice to lock all data — accepting that systems become unavailable. | Top-down revocation at any key level (§5). Instant cryptographic lockdown. No key material to leak.                                                 |
| **BR3** | **Country retreat**                  | If the primary key is not available, the data is unusable (crypto shredded). Emergency exit from a geographic region.                                       | Regional L2 key isolation. Revoke one region's L2 → that region's data is cryptographically destroyed while other regions continue operating.       |
| **BR4** | **Loss of trust in provider**        | The customer can lock the data and move on — without depending on the provider's cooperation.                                                               | HYOK model: L1 never leaves the customer's vault. The platform operator holds a reference, not the key. Customer revokes → provider cannot decrypt. |
| **BR5** | **Compliance**                       | Industry and regional regulations (GDPR, DORA, NIS2, PCI-DSS, Schrems II) require cryptographic key control, audit trails, and data residency.              | FIPS 140-2/3 crypto (§8), E2E audit logging with correlation IDs (§9), regional key isolation, NIST SP 800-57 key lifecycle.                        |

### Functional Requirements

These are the specific capabilities customers expect from the key management system:

| #       | Functional Requirement                                                                                            | How Krypton Delivers                                                                                                                                                                |
| ------- | ----------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **FR1** | **Customer Managed Keys (L1) are master keys** — the customer fully controls the root of the key hierarchy        | L1 is always external (customer's HSM or cloud KMS). Krypton holds a reference, never plaintext material. Attach tenants and services to this master key via the key hierarchy.     |
| **FR2** | **Red Button workflow** — an instant, auditable process to lock all data, not dependent on a single administrator | Top-down revocation (§5) at L1 or L2 level. Workflow requires authenticated identity (mTLS/OIDC). Every action logged with correlation ID.                                          |
| **FR3** | **Onboarding** — registering a customer's key into the system must be a secure, auditable process                 | CMK control plane handles L1 registration. Supports Provider Managed, BYOK, and HYOK onboarding flows (§6). Each onboarding event is audit-logged.                                  |
| **FR4** | **ENABLE / ROTATE / DISABLE / DELETE** — full key lifecycle management                                            | Maps directly to NIST SP 800-57 states: Pre-Active → Active → Deactivated → Destroyed (§7). IVK mechanism enables zero-downtime rotation.                                           |
| **FR5** | **Test the Red Button** — validate the kill switch without triggering it                                          | Red Button dry-run mode (§10): simulate revocation, receive blast radius report (affected services, DEKs, tenants), no actual revocation.                                           |
| **FR6** | **FIPS compliance** — cryptographic operations must use validated, certified algorithms                           | FIPS 140-2 validated libraries, FIPS 140-3 readiness. AES-256-GCM, RSA-OAEP, HKDF. FIPS mode restricts to approved algorithms only (§8).                                            |
| **FR7** | **Choice between keystores** for the master key                                                                   | Three deployment models (§6): **Provider Managed** (operator manages L1), **BYOK** (customer generates, imports to Krypton), **HYOK** (L1 stays in customer's vault — recommended). |
| **FR8** | **End-to-end audit logs** — from key management down to the persistence layer                                     | Structured, immutable audit events with correlation IDs across the full chain: who, what, which key, when, where, result (§9). SIEM-ready.                                          |

### What This Means for Platform Services

The benefit for services that integrate with Krypton (databases, caches, storage layers): **you do not have to implement CMK yourself.** Krypton handles the key hierarchy, the lifecycle, the audit trail, and the customer's key sovereignty — transparently. Services connect via KMIP and get encryption keys. The CMK complexity is abstracted away.

Of course, end-to-end testing is essential to ensure the full chain works correctly — from the customer's L1 key through Krypton Core and Gateway down to the service's encrypted data at rest.

---

## Competitive Positioning

| Capability                   | Krypton                                        | AWS KMS          | Thales CipherTrust | Fortanix DSM           | HashiCorp Vault        |
| ---------------------------- | ---------------------------------------------- | ---------------- | ------------------ | ---------------------- | ---------------------- |
| **Protocol**                 | KMIP (OASIS standard)                          | Proprietary REST | KMIP               | KMIP                   | Proprietary + KMIP     |
| **Vendor lock-in**           | None                                           | AWS-only         | Hardware-bound     | Multi-cloud            | Multi-cloud            |
| **Encryption latency**       | <0.5ms (Gateway)                               | 20-50ms          | 10-30ms (HSM)      | 50-300ms               | 50-200ms               |
| **Customer key sovereignty** | Mandatory (L1 always external)                 | Optional         | HSM-controlled     | Optional               | Optional               |
| **Flexible keychain depth**  | Unlimited layers                               | Flat (CMK→DEK)   | 2-3 layers         | 2-3 layers             | Flat                   |
| **Top-down revocation**      | Any level, instant                             | Key policy only  | Manual             | Via API                | Manual                 |
| **Zero-downtime rotation**   | Lazy re-wrapping (IVK)                         | Not exposed      | Manual             | Requires re-encryption | Requires re-encryption |
| **Cryptographic shredding**  | Per-record (L4 Revoke)                         | Bulk delete      | Manual             | Via API                | Manual deletion        |
| **Multi-cloud L1**           | AWS, Azure, GCP, HSM, OpenBao                  | AWS only         | HSM only           | Multi-cloud            | Multi-cloud            |
| **Open source**              | Yes                                            | No               | No                 | No                     | BSL (not OSS)          |
| **Kubernetes-ready**         | Yes (Operator + sidecar); also bare-metal, VMs | No               | No                 | Partial                | Yes                    |
| **FIPS 140-2/3**             | Yes (FIPS-validated crypto libs)               | Yes              | Yes (HSM)          | Yes                    | Enterprise only        |
| **E2E audit logging**        | Correlation IDs across full chain              | CloudTrail only  | Partial            | Via API                | Partial                |
| **Red Button dry-run**       | Yes (impact simulation)                        | No               | No                 | No                     | No                     |

---

## Demo Scenario: Platform Mesh + MongoDB

### Phase 1: Setup — "Show It Works"

**Story:** A security engineer on the Platform Mesh platform decides to enable encryption at rest for MongoDB using Krypton.

1. **Before encryption:** Show that MongoDB data is stored in plaintext on disk.
2. **Enable Krypton:** Configure MongoDB to connect to Krypton Gateway via KMIP.
3. **Restart MongoDB:** MongoDB requests a DEK from Krypton, encrypts all data.
4. **Verify encryption:** Show that the same data on disk is now ciphertext.
5. **Show the keychain:** Display the L1 → L2 → L3 → L4 (DEK) hierarchy that Krypton created. The customer's external key (L1) wraps L2, L2 wraps L3, L3 wraps the DEK that MongoDB uses.

**Takeaway:** Zero application code changes. KMIP standard. Transparent encryption.

### Phase 2: Incident — "Show It Protects"

**Story:** A key in the chain is suspected compromised.

1. **Suspend the L3 key:** Mark it as compromised/suspended in Krypton.
2. **NIST-compliant behavior:** MongoDB can still **read** (decrypt) existing data — because NIST SP 800-57 allows decryption on compromised keys for continuity. But it **cannot write** new encrypted data with the compromised key.
3. **Rotate:** Issue a new L3 key. New DEKs are generated under the new chain.
4. **Revoke the old L3:** The old branch is cut. Data encrypted under the old DEKs is no longer accessible once the old key is destroyed.

**Takeaway:** Krypton stops the usage of a compromised key. The service is responsible for data cleanup (re-encryption), which is the same regardless of which KMS is used.

### Phase 3: Kill Switch — "Show It Shreds"

**Story:** Complete lockdown required.

1. **Disable L1** (or L2): In the customer's AWS KMS console (or Krypton CMK portal).
2. **Immediate effect:** Krypton Core can no longer unseal L2. All KMIP requests fail.
3. **MongoDB cannot start:** No DEK available. Service is down.
4. **Recovery:** Customer re-enables L1. Krypton re-unseals. MongoDB restarts with same keys.

**Takeaway:** One action, global effect. The customer — not the platform operator — holds the kill switch.

---

## Business Value Summary

### For the Customer

| Value                         | Proof Point                                                                           |
| ----------------------------- | ------------------------------------------------------------------------------------- |
| **No vendor lock-in**         | KMIP standard; switch KMS providers by changing one hostname                          |
| **Full data sovereignty**     | L1 key never leaves customer's vault; mathematical guarantee                          |
| **Encrypt everything**        | Sub-millisecond performance; no "too slow to encrypt" excuse                          |
| **Compliance ready**          | FIPS 140-2/3 crypto; NIST key lifecycle; GDPR cryptographic erasure; DORA audit trail |
| **Organizational fit**        | Keychain depth matches org structure — startup to multinational                       |
| **Instant incident response** | Top-down revocation at any level; cryptographic shredding                             |

### For the Platform Operator

| Value                       | Proof Point                                                               |
| --------------------------- | ------------------------------------------------------------------------- |
| **Multi-cloud portability** | Same Krypton instance works on AWS, GCP, Azure, on-premise                |
| **Reduced liability**       | Never holds customer key material; cannot be compelled to decrypt         |
| **Operational simplicity**  | Kubernetes-ready operator; automated key rotation; no maintenance windows |
| **Open-source trust**       | Code is auditable; no "trust us" black box                                |

### For EU / Public Sector Funding

| Value                     | Proof Point                                                        |
| ------------------------- | ------------------------------------------------------------------ |
| **Digital sovereignty**   | No dependency on US hyperscalers for key material                  |
| **Open-source**           | Publicly funded technology that benefits the community             |
| **Standards-based**       | OASIS KMIP; NIST SP 800-57 key lifecycle                           |
| **Provider independence** | Replicate key infrastructure across providers without code changes |

---

## What Krypton Does NOT Do

Transparency is important for a credible business case:

- **Krypton does not handle data re-encryption on key rotation.** When a DEK is rotated, it is the service's responsibility to re-encrypt data with the new key. This is the same with any KMS — it is not a Krypton limitation but an industry-wide reality.
- **Krypton standalone requires a solution for secure L1 onboarding.** The CMK control plane handles L1 key registration and linkage (including Provider Managed, BYOK, and HYOK onboarding flows). Running Krypton independently (without CMK) requires defining how the L1 key is securely brought into the keychain. This is an active area of development.
- **Krypton does not replace the application's encryption logic.** For databases that support KMIP (MongoDB, MariaDB), encryption is transparent. For databases that don't, integration requires a bridge layer (OpenBao, sidecar proxy, or application-level envelope encryption).

---

## Conclusion

Krypton brings the full body of cryptography to organizations of any size:

- **Small companies** get simple, standards-based encryption with a two-level keychain and zero vendor lock-in.
- **Large enterprises** get a deep, organizationally structured key hierarchy with geographic isolation, department-level key chains, and top-down revocation.
- **Platform operators** get a managed control plane for keys that makes the underlying infrastructure irrelevant — deploy on AWS today, move to GCP tomorrow, go on-premise next year.
- **Public sector and sovereignty-conscious organizations** get mathematical proof that keys stay in their control, built on open-source technology and international standards.

The keychain is the product. The flexibility of the keychain is what makes Krypton different. And the customer — not the provider — decides how deep it goes.
