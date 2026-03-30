# Krypton — Target Audiences

## Audience 1: Engineers & Security Architects

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

---

## Audience 2: Enterprise Architects & CTOs

**Their question:** _"How does this fit a large organization with multiple departments, regions, and compliance requirements?"_

**The answer:**

Consider a multinational company with departments in different geographic regions, each with its own compliance requirements and data residency rules:

```
Corporate Root Key (L1 — in corporate HSM)
│
├── EU Region (L2)
│   ├── Finance Department (L3)
│   │   ├── SAP HANA (L4) — German data residency
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

---

## Audience 3: Decision Makers, Public Sector & EU Funding

**Their question:** _"How does this address data sovereignty and vendor independence?"_

**The answer:**

Krypton is built on a fundamental principle: **the underlying infrastructure is just compute.**

- The root key stays in the customer's own vault — not in a cloud provider's managed service.
- The key hierarchy can be replicated from one cloud provider to another (GCP → AWS → on-premise) because Krypton is provider-agnostic.
- The KMIP standard ensures no application code changes when moving between providers.
- The platform operator (whether SAP, a public cloud, or a sovereign cloud) **never holds plaintext key material**.

This directly addresses:

| Requirement               | How Krypton Delivers                                                                                |
| ------------------------- | --------------------------------------------------------------------------------------------------- |
| **GDPR / Data Residency** | Keys stay in specified regions; regional L2 isolation ensures data cannot be accessed from outside  |
| **DORA / NIS2**           | Full audit trail with correlation IDs; instant revocation capability; key rotation without downtime |
| **Schrems II**            | Customer controls the root key — no US provider can be compelled to decrypt EU data                 |
| **Digital Sovereignty**   | No dependency on any single cloud provider; open-source; OASIS standard                             |
| **EU Public Funding**     | Open-source project; not bound to commercial vendor; technology can be audited and verified         |

> _"Data sovereignty is not a feature we bolt on — it's the architecture. The root key never leaves your control, the infrastructure is just compute, and the protocol is an open standard. No vendor lock-in, no trust assumptions, no political dependencies."_
