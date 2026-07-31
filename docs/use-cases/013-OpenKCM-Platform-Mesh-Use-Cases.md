---
authors:
  - Aysan Mazloumi
last_updated: 2026-07-31
---

# OpenKCM on Platform Mesh — Use Cases

## Overview

OpenKCM integrates with Platform Mesh as a key management service provider. When an account enables OpenKCM, the customer registers their root key once — and from that point, all encryption in their account is governed by that key. The platform cannot access customer data without the customer's explicit consent.

There are three use cases OpenKCM supports depending on how workloads are structured within the account.

---

## Pattern 1 — Single Namespace Encryption

**Who it is for:** A team running their own services in a single namespace with no shared dependencies.

**What it looks like:** A team deploys MongoDB in their namespace. OpenKCM automatically provisions encryption for that namespace. The team registers their service key and MongoDB encrypts all data transparently — no key configuration required from developers.

**The value:** Zero-touch encryption. Developers deploy services without thinking about key management. The security admin registers the root key once and the rest is automatic. If the security admin disables the root key, all data in the namespace becomes instantly inaccessible.

---

## Pattern 2 — Cross-Namespace System Encryption

**Who it is for:** Organizations running shared services (databases, message queues) that serve multiple teams or departments in different namespaces.

**What it looks like:** A MongoDB service runs in one namespace and serves both an HR team and a Finance team in separate namespaces. Each team's data is encrypted with its own service key. The HR team's data and the Finance team's data are cryptographically isolated from each other — even though they use the same database service.

The service team that runs MongoDB manages the encryption keys for each tenant they serve. HR and Finance teams do not manage keys — they just use the service.

**The value:** Shared infrastructure, isolated encryption. One database service can serve multiple teams without those teams sharing cryptographic boundaries. Each team's data can be independently suspended or revoked without affecting others.

---

## Pattern 3 — Cross-Account Service Consumption

**Who it is for:** Organizations consuming services from other accounts or from managed service providers on Platform Mesh.

**What it looks like:** A service running in one account is consumed by teams in another account. Encryption keys are scoped to the consuming account — the service provider never has access to the consumer's key material.

**The value:** Cryptographic sovereignty across account boundaries. The consumer holds their root key, and even the service provider cannot access their data.

**Status:** This pattern is planned for a future release. Current delivery focuses on Patterns 1 and 2.

---

## What the Customer Manages

In all patterns, the customer's active role is minimal:

- **Register the root key once** — connect to their own external keystore (AWS KMS, Azure Key Vault, OpenBao, or HSM). The key never leaves their keystore.
- **Trigger the kill switch if needed** — disabling the root key instantly makes all data inaccessible across all services and namespaces.

