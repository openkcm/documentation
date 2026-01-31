# ACD-202: OpenKCM Crypto Core – Regional Governance & KMIP Unsealing

| Status | Date | Document Type |
| :--- | :--- | :--- |
| **Active** | 2026-01-13 | Architecture Concept Design |

## Overview
The **OpenKCM Crypto Core** is the regional "Authority" and cryptographic backbone of the OpenKCM ecosystem. While the central CMK Service manages global governance and tenant metadata, the OpenKCM Crypto Core is responsible for the localized execution of those policies and the lifecycle of the regional key hierarchy.

Its primary mission is twofold:
1. **Recursive Unsealing**: Using a customer’s external **L1 Root Key** to unlock the regional **L2 (Tenant)** and **L3 (Key Encryption Key - KEK)** layers.
2. **KMIP Execution**: Serving as the regional KMIP engine that performs cryptographic operations for customer **L4 Data Encryption Keys (DEKs)**, specifically wrapping and unwrapping them against the **L3 KEK**.

## The Regional Component Stack
The OpenKCM Crypto Core is a high-integrity cluster of components designed for resilience and security:

* **OpenKCM Crypto (Core Service):** The Go-based orchestration engine that performs **KMIP operations** regarding customer **L4 (DEK)** material, handling the Wrap and Unwrap logic against the **L3 (KEK)**.
* **Internal Vault (OpenBao):** The regional secure storage for L2 and L3 intermediate keys. It is protected by the internal MasterKey and accessible only via internal mTLS.
* **Checker:** A specialized sidecar service that provides health introspection, OPA-based policy verification, and cluster status reporting.



## Communication Protocol: KMIP & mTLS
The **OpenKCM Crypto Core** utilizes the KMIP protocol as its primary operational interface for both internal orchestration and high-performance data plane requests.

* **KMIP Protocol**: The Core exposes the **OASIS KMIP** standard. This enables a standardized approach for clients (or the Crypto Gateway) to request that L4 DEKs be wrapped or unwrapped by the regional L3 KEK.
* **Identity-Based mTLS**: Every KMIP request is gated by **mutual TLS (mTLS)**. This ensures that only authorized workloads can request the Core to perform operations using a specific tenant's L3 key.

## The Wrapping & Unwrapping Workflow (L1 → L4)
The Core manages the cryptographic chain of custody from the cloud-native root to the individual data record:

1.  **L1 Handshake**: **OpenKCM Crypto** calls the customer's external KMS to unwrap the **L2 Tenant Key**.
2.  **L3 Activation**: The L2 key is used to unwrap/unlock the **L3 KEK (Key Encryption Key)** stored within the Internal Vault.
3.  **L4 KMIP Operation**: When an application requires data access, it sends the wrapped **L4 DEK** to the Core via KMIP. The Core unwraps the L4 DEK using the **L3 KEK** and returns the material (or performs the transform) according to the policy.



## Regional Autonomy & Persistence
To ensure zero-downtime operations, the OpenKCM Crypto Core is designed to operate independently of the central CMK Control Plane:

* **Regional Persistence**: The Core relies on its internal **Vault (OpenBao)** to maintain the state of the regional key hierarchy. It can restart and re-establish the L1-L3 chain using the customer's external key without contacting the central Registry.
* **Task Reconciliation**: It synchronizes with the **Orbital** framework to receive global policy updates or rotation commands, ensuring the "Actual State" of regional KMIP operations matches the "Desired State" in the CMK.

## Summary
This document defines the regional "Root of Authority" for OpenKCM. By serving as the KMIP engine for **L4 DEK** operations against the **L3 KEK**, the **OpenKCM Crypto Core** ensures that:
1.  **Sovereignty is Maintained**: The L1-L4 chain remains intact and mathematically isolated.
2.  **Performance is Standardized**: KMIP provides a high-throughput, industry-standard interface for data operations.
3.  **Governance is Localized**: Regional nodes handle the heavy lifting of cryptographic transforms while remaining under global administrative control.