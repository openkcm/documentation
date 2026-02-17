# ACD-103: CMK Registry & Global Tenant Governance

| Status | Date | Document Type |
| :--- | :--- | :--- |
| **Active** | 2026-02-02 | Architecture Concept Design |

## Overview
The **CMK Registry** is the "Front Door" and **Business Inventory System** of the OpenKCM ecosystem. While the **CMK Server** (ACD-101) acts as the secure execution environment for sovereign operations, the Registry provides the essential **Global Discovery** and **Governance Topology**.

It serves as the **Global Switchboard**, answering the fundamental business questions required to operate a Sovereign Cloud at scale:
1.  **Who are our customers?** (Tenant Lifecycle)
2.  **What Systems (L2 Keys) do they own?** (Account Inventory)
3.  **What Services (L3 Keys) are running?** (Application Inventory)
4.  **Who is authorized to manage them?** (Global Identity Mapping)

By decoupling **Global Visibility (Registry)** from **Sovereign Execution (CMK Server)**, OpenKCM enables a "Single Pane of Glass" management model without violating the physical isolation requirements of tenant data.

## Strategic Role: The "Global Switchboard"
In a sovereign cloud, data must be physically isolated to meet strict compliance mandates (e.g., GDPR, TISAX). However, fragmented management leads to operational failure. The Registry solves this by providing:

* **Unified Visibility:** Administrators access a single global estate regardless of whether tenant keys reside in AWS, Azure, or private HSMs.
* **Routing Intelligence:** The Registry directs management traffic to the correct **CMK Server** based on the tenant's region and identity context.
* **Metadata Separation:** It holds the "Map" to the inventory but never the "Keys" to the vault. This separation significantly lowers the total risk profile of the Control Plane.

## Core Business Capabilities

### 1. Tenant Lifecycle Management (The Customer)
The Registry is the System of Record for the "Customer Entity."
* **Onboarding:** Provisions the logical identity required for billing and RBAC integration.
* **State Management:** Tracks the operational readiness of the customer (e.g., `Provisioning`, `Active`, `Blocked`, `Suspended`).
* **Data Residency Enforcement:** Anchors the tenant to a specific legal jurisdiction (Region Affinity) before any cryptographic assets are created.

### 2. System Inventory (The L2 Key Layer)
The Registry maintains a comprehensive catalog of all **Systems**, which represent the **L2 Keys** (often mapped to "Accounts" or "Tenants" in the customer's view).
* **L2 System Definition:** Allows developers to define the primary cryptographic scope (e.g., "HR System," "Payments Account") independently of the underlying hardware or cloud provider.
* **Operational State:** Tracks the high-level status of the System (e.g., `Active`, `Decommissioned`) to drive UI visibility and reporting.
* **Organizational Tagging:** Supports labels and metadata to map L2 Systems to specific cost centers, business units, or compliance tiers.

### 3. Service Inventory (The L3 Key Layer)
The Registry acts as the granular catalog for **Services**, which represent the **L3 Keys** (the actual working keys used by applications).
* **Service Definition:** Defines the specific application scopes (e.g., "Checkout Microservice," "Audit Logger," "Database Encryption") that exist within a System.
* **Hierarchy Enforcement:** Strictly maps every Service (L3) to a parent System (L2), ensuring that applications inherit the sovereignty and isolation properties of their parent account.
* **Authorization Scope:** Acts as the reference point for RBAC, defining which identities are allowed to request keys for a specific Service (e.g., "Only the PaymentGateway App can access the 'Payments' L3 Service").

### 4. Global Identity & Access (The Principal)
The Registry serves as the central directory for **Principals** (Users and Service Accounts).
* **Identity Normalization:** Connects to corporate Identity Providers (OIDC/SPIFFE) to resolve external identities into the OpenKCM context.
* **Tenant-to-User Mapping:** Defines the high-level relationship of which administrators are permitted to view or manage which L2 System environments.
* **Role Assignment:** Maps Global Roles (e.g., "Auditor," "Admin") to specific Principals within the context of a Tenant.

## Risk & Liability Model
The Registry is architected as a **Minimal Liability Layer**.

| Data Class | Stored in Registry? | Business Risk |
| :--- | :--- | :--- |
| **Tenant/User Metadata** | **Yes** | Low (Operational/Privacy) |
| **L2 System Inventory** | **Yes** | Low (Metadata/Configuration) |
| **L3 Service Inventory** | **Yes** | Low (Metadata/Configuration) |
| **Encryption Keys** | **NO** | **None** |
| **L1 Key Pointers (ARNs)** | **NO** (In CMK Server) | **None** |
| **Sovereign Policies** | **NO** (In CMK Server) | **None** |

**Strategic Implication:** A breach of the Registry provides an attacker with a list of Systems and Services but **zero ability to decrypt data**, access keys, or bypass the sovereign controls enforced by the CMK Server.

## Summary
**ACD-103** defines the CMK Registry as the **Intelligence and Routing Layer** of OpenKCM. It provides the essential operational cohesion that allows a fragmented, multi-tenant L2/L3 key infrastructure to function as a seamless, enterprise-grade SaaS platform.

**It enables organizations to scale security operations globally while maintaining absolute local sovereignty over every individual account and service.**