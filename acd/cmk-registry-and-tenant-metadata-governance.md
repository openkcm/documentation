# ACD-103: CMK Registry & Global Tenant Governance

| Status | Date | Document Type |
| :--- | :--- | :--- |
| **Active** | 2026-01-31 | Architecture Concept Design |

## Overview
The **CMK Registry** is the "Front Door" and **Business Inventory System** of the OpenKCM ecosystem. While the CMK Service (ACD-102) acts as the secure execution environment for sovereign operations, the Registry provides the essential **Global Discovery** and **Governance Topology**.

It serves as the **Global Switchboard**, answering the three fundamental business questions required to operate a Sovereign Cloud at scale:
1.  **Who are our customers?** (Tenant Lifecycle)
2.  **What L2 Keys/Accounts do they own?** (System Inventory)
3.  **Who is authorized to manage them?** (Global Identity Mapping)

By decoupling **Governance (Registry)** from **Sovereign Execution (CMK Service)**, OpenKCM enables a "Single Pane of Glass" management model without violating the physical isolation requirements of tenant data.

## Strategic Role: The "Global Switchboard"
In a sovereign cloud, data must be physically isolated to meet strict compliance mandates (e.g., GDPR, TISAX). However, fragmented management leads to operational failure. The Registry solves this by providing:

* **Unified Visibility:** Administrators access a single global estate regardless of whether tenant keys reside in AWS, Azure, or private HSMs.
* **Routing Intelligence:** The Registry directs management traffic to the correct **Sovereign Vault** based on the tenant's region and identity context.
* **Metadata Separation:** It holds the "Map" to the inventory but never the "Keys" to the vault. This separation significantly lowers the total risk profile of the Control Plane.

## Core Business Capabilities

### 1. Tenant Lifecycle Management (The Customer)
The Registry is the System of Record for the "Customer Entity."
* **Onboarding:** Provisions the logical identity required for billing and RBAC integration.
* **State Management:** Tracks the operational readiness of the customer (e.g., `Provisioning`, `Active`, `Blocked`).
* **Data Residency Enforcement:** Anchors the tenant to a specific legal jurisdiction (Region Affinity) before any cryptographic assets are created.

### 2. System Inventory (The L2 Key Layer)
The Registry maintains a comprehensive catalog of all **Systems**, which represent the **L2 Keys** (also referred to as **Accounts** or **Tenants** within different customer ecosystems).
* **L2 Key Definition:** Allows developers to register the primary cryptographic account (e.g., "Payments Account") independently of the underlying hardware or cloud provider.
* **L1 Key Claims:** Provides high-level visibility into which L2 systems have successfully established a root-of-trust connection to a customer-managed L1 key.
* **Organizational Tagging:** Supports labels and metadata to map L2 key usage to specific cost centers or business units.

### 3. Global Identity & Access (The Principal)
The Registry serves as the central directory for **Principals** (Users and Service Accounts).
* **Identity Normalization:** Connects to corporate Identity Providers (OIDC/SPIFFE) to resolve external identities into the OpenKCM context.
* **Tenant-to-User Mapping:** Defines the high-level relationship of which administrators are permitted to access which L2 key environments.

## Governance Use Cases

### 1. Managed Service Provider (MSP) Efficiency
* **Challenge:** Managing encryption accounts for 50 diverse enterprise clients across different regions.
* **Registry Solution:** The MSP admin logs into the Registry and views 50 isolated L2 systems from one dashboard. They initiate a global compliance report, and the Registry orchestrates the request across 50 sovereign schemas.

### 2. Strategic Data Sovereignty
* **Scenario:** A customer demands that their "Account" data never leaves a specific German data center.
* **Action:** The Admin configures the System/L2 in the Registry with `Region: DE-Frankfurt`.
* **Result:** The Registry ensures all subsequent CMK Service workflows and data storage are physically restricted to that specific sovereign zone.

### 3. Security Posture Auditing
* **Scenario:** A CISO needs to know if all "Production Accounts" are correctly protected.
* **Action:** A query to the Registry's Inventory Service filtered by labels (`env=prod`).
* **Result:** Immediate visibility into the global security posture without needing the authority to access the actual encryption keys.

## Risk & Liability Model
The Registry is architected as a **Minimal Liability Layer**.

| Data Class | Stored in Registry? | Business Risk |
| :--- | :--- | :--- |
| **Tenant/User Metadata** | Yes | Low (Operational/Privacy) |
| **L2 System Inventory** | Yes | Low (Metadata/Configuration) |
| **Encryption Keys** | **NO** | **None** |
| **Workflow State (MPA)** | **NO** (In CMK) | **None** |
| **Sovereign Policies** | **NO** (In CMK) | **None** |

**Strategic Implication:** A breach of the Registry provides an attacker with a list of L2 accounts but **zero ability to decrypt data** or bypass the Multi-Party Authorization workflows enforced by the CMK Service.

## Summary
**ACD-103** defines the CMK Registry as the **Intelligence and Routing Layer** of OpenKCM. It provides the essential operational cohesion that allows a fragmented, multi-tenant L2 key infrastructure to function as a seamless, enterprise-grade SaaS platform.

**It enables organizations to scale security operations globally while maintaining absolute local sovereignty over every individual account.**