# ACD-203: Crypto (Krypton) Gateway – High-Performance Ephemeral Data Plane

| Status | Date | Document Type |
| :--- | :--- | :--- |
| **Active** | 2026-01-13 | Architecture Concept Design |

## Overview
The **OpenKCM Crypto (Krypton) Gateway** is the distributed execution component designed to run physically close to customer workloads, whether in a public cloud VPC, an on-premise data center, or a metro-gateway location. It serves as the ultra-low-latency interface for application encryption operations, prioritizing speed and availability above all else.

To achieve sub-millisecond latency while maintaining strict global governance, the Crypto (Krypton) Gateway operates on a **Split-Function Model**: it autonomously handles high-volume "Hot Path" data plane operations locally, while securely delegating complex "Control Path" lifecycle management to the regional **OpenKCM Crypto (Krypton)**.

## The Split-Function Philosophy
Unlike traditional monolithic KMS appliances, the OpenKCM architecture recognizes that data encryption requires different operational characteristics than key management policy.

### The Hot Path (Local Execution)
99% of cryptographic traffic consists of creating new Data Encryption Keys (DEKs) for new data or retrieving existing DEKs to read data. The Gateway optimizes these operations for speed by executing them entirely within its local boundary, utilizing local entropy and secure local storage.

### The Control Path (Delegated Governance)
Operations that alter the fundamental state, policy, or lifecycle of a key require global consensus and audit trails. The Gateway acts as a secure, mTLS-authenticated proxy for these requests, tunneling them back to the **Crypto (Krypton)** which holds the authoritative registry.

## Architectural Components
The Crypto (Krypton) Gateway acts as a specialized, highly efficient forwarder with local caching capabilities.

* **KMIP Listener (mTLS):** The standardized front-door that accepts incoming connections from workload clients, enforcing strict mutual TLS authentication based on SPIFFE IDs or x509 certificates.
* **Local Key Engine:** A high-performance cryptographic module responsible for generating high-entropy ephemeral L4 keys.
* **Gateway Vault (Local Persistence):** A lightweight, secure local storage mechanism (e.g., an embedded OpenBao instance or secure enclave) used to persist wrapped L4 keys close to the workload for rapid retrieval.
* **Core Uplink Client (gRPC):** Manages the persistent, long-haul mTLS connection back to the regional Crypto (Krypton) for carrying delegated control traffic and receiving L3 Service Key updates.

## Operational Workflows

### KMIP Create & Get (The Hot Path)
These operations are executed locally to ensure maximum throughput.
1.  **Workload Request:** An application requests a new L4 DEK via KMIP `Create`.
2.  **Local Generation:** The Gateway utilizes local hardware entropy to generate the key material.
3.  **L3 Wrapping:** The Crypto (Krypton) Gateway wraps the new L4 DEK using the locally cached **L3 Service Key** (which was previously unsealed by the Core).
4.  **Local Persistence:** The wrapped L4 DEK is stored in the **Gateway Vault**.
5.  **Response:** The plaintext L4 key and its ID are returned to the client.

### Delegation Workflow (The Control Path)
When the Gateway receives a request for a non-local operation (e.g., `KMIP Destroy`, `Revoke`, or complex queries):
1.  **Identification:** The Gateway parses the KMIP header and identifies the operation as "Control Path."
2.  **Tunneling:** The Gateway opens a secure gRPC tunnel to the **OpenKCM Crypto (Krypton)**, forwarding the client's request context.
3.  **Core Execution:** The Core executes the command against the persistent central Registry and authoritative Vault.
4.  **Response Relay:** The Core's response is passed back through the Gateway to the client, preserving the synchronous KMIP session illusion.

## Resilience & Autonomy
The Crypto (Krypton) Gateway is designed for **Partition Tolerance**. If the network link to the regional Crypto (Krypton) is severed:
* **Read Operations Continue:** The Gateway can continue to serve `KMIP Get` requests for any L4 keys already cached in its local Gateway Vault, using its cached L3 key for unwrapping.
* **Create Operations Continue:** The Gateway can continue to generate and locally store new L4 keys.
* **Control Operations Pause:** Delegated operations (like lifecycle changes) will fail safely until connectivity to the Core is restored, ensuring governance state never diverges.

## Summary
This document defines the **Crypto (Krypton) Gateway** as the high-speed tactical layer of OpenKCM. By restricting its autonomous scope to L4 `Create` and `Get` operations while providing secure local persistence, it ensures that:
1.  **Performance is Maximized:** Encryption latency is dictated by local network physics, not wide-area network round trips.
2.  **Governance is Centralized:** 100% of significant lifecycle changes are enforced by the authoritative Crypto (Krypton).
3.  **Resilience is localized:** Workloads can continue to encrypt and decrypt data even during regional control plane outages.