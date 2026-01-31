# ACD-101: OpenKCM Strategic Vision & Cryptographic Governance Model

| Status | Date | Document Type |
| :--- | :--- | :--- |
| **Active** | 2026-01-14 | Architecture Concept Design |

## Overview
**OpenKCM** is the **Value Engine** that enables platforms to win bigger deals, capture higher margins, and operate with zero-risk liability in a world that demands absolute data ownership.

In the modern sovereign-cloud era, traditional security is a cost center. OpenKCM transforms security into a **market differentiator** by solving the "Trust Paradox"—the conflict between providing total customer data ownership and the need for high-scale, cloud-native performance.

## Structural Pillar: The Governance & Execution Framework
To achieve absolute sovereignty without performance degradation, OpenKCM enforces a "Church and State" separation between the logic of access and the act of encryption.

### OpenKCM CMK: The Governance Control Plane
The **CMK** is the "Brain" of the ecosystem. It manages the **Intent** of security without ever possessing the material to execute it.
* **Sovereign Anchor:** Manages references to **L1 Root Keys** held exclusively in customer-owned environments (AWS KMS, Azure Key Vault, GCP KMS, or Private HSMs).
* **Security Barrier:** The CMK handles metadata and encrypted unseal requests but is physically and logically barred from touching plaintext key material.

### OpenKCM Crypto: The Execution Plane
The **Crypto** layer is the "Muscle." It consists of regional and gateway nodes designed for zero-latency cryptographic operations.
* **OpenKCM Crypto Core:** The regional authority that manages **L2 (Tenant)** and **L3 (Service)** keys. It performs the "Recursive Unsealing" required to activate a tenant's environment and executes stateless **L4 Wrap/Unwrap** operations via KMIP.
* **OpenKCM Crypto Gateway:** Distributed, high-speed interfaces that generate and manage **L4 (Data)** keys via KMIP. The Gateway operates close to the workload, ensuring sub-millisecond encryption.

## The Sovereignty Engine: Recursive Unsealing
OpenKCM enforces security through a tiered key hierarchy where each layer is mathematically bound to the one above it.

1.  **L1 (External)**: Customer-owned Root of Trust.
2.  **L2 (Tenant)**: Provides mathematical isolation between different chain trees for same customers.
3.  **L3 (Service)**: Provides isolation between specific application domains.
4.  **L4 (Data)**: Ephemeral keys used for per-record encryption, generated and stored at the Gateway or client side.

## Strategic Advantages

| Advantage | Business Meaning |
| :--- | :--- |
| **Market Expansion** | Opens Government, Finance, Defense, Healthcare, etc. |
| **Liability Shift** | No plaintext root keys → breach firewall | 
| **Pricing Leverage** | Full BYOK/HYOK + revocation as Platinum tier |
| **Competitive Moat** | True cryptographic isolation + multi-cloud native | 
| **Compliance Acceleration** | End-to-end traceability & audit logs |

## CMK Layer: Governance Capabilities
The CMK Layer is the **trust firewall** that allows enterprises to adopt SaaS with the same confidence as on-premise solutions.

* **Self-Service Onboarding**: Enterprises import keys or create them in their own cloud accounts without manual provider involvement.
* **Tenant-to-Key Mapping**: Securely binds tenant IDs to customer-specific KMS references (ARNs, URIs) with fine-grained authorization.
* **Pluggable Keystore Drivers**: Unified interface for AWS KMS, Azure Key Vault, GCP KMS, OpenBao, and private HSMs.
* **Revocation Kill-Switch**: Customer-initiated revocation instantly renders all tenant data inaccessible to the provider with no recovery path.

## The Crypto Layer: Operational Excellence
The Crypto Layer handles the "operational reality" of security—ensuring that billions of operations per day do not degrade the user experience.

### OpenKCM Crypto Core
* **Lifecycle Automation**: Handles rotation, versioning, and archiving of L2/L3 keys with zero downtime.
* **MasterKey Protection**: Reconstructs the internal MasterKey via Shamir Secret Sharing (SSS) or Seal auto-unseal into secure memory only.
* **Stateless KMIP Execution**: Performs Wrap/Unwrap operations for **L4 DEKs** against the **L3 KEK**. It transmits the result via mTLS but **does not store** the L4 key material.

### OpenKCM Crypto Gateway
* **Dedicated Gateway Service**: Deployed as a standalone service close to workloads, managing high-throughput operations.
* **Local Persistence & Caching**: Independently handles `Create` and `Get` operations for **L4 DEKs**, securely storing them in a local gateway vault to ensure resilience and performance.
* **KMIP/mTLS**: All communication uses standardized KMIP over mandatory mutual TLS.

## Strategic Outcome
**How Governance (CMK) & Operations (Crypto) Drive Enterprise Value**

| Strategic Dimension | CMK Service (Governance) | Crypto Service (Operations) | Combined Impact |
| :--- | :--- | :--- | :--- |
| **Market Access** | Unlocks regulated verticals via root-key control. | Enables high-throughput for AI/Real-time platforms. | Opens high-ACV deals previously blocked by sovereignty concerns. |
| **Trust & Retention** | Mathematically enforced revocation. | Zero-trust multi-tenancy (no cross-tenant leakage). | Reduces churn in regulated segments; customers stay longer. |


```mermaid
graph TD
    %% --- Governance Control Plane ---
    subgraph GCP ["Governance Control Plane\nOpenKCM CMK\n(The Brain & Policy)"]
        CMK_API ["CMK API Server"]
        
        subgraph CMK_Plugins ["Plugins"]
            P_ID ["Identity"]
            P_KS ["Keystore"]
            P_SI ["SystemInfo"]
            P_NT ["Notify"]
        end

        L1_Ref [("L1 Key Pointer\n(Shadow Ref)")]

        %% Connections within GCP
        L1_Ref --> CMK_API
        CMK_API <--> CMK_Plugins
        P_KS --> P_ID
        P_KS --> P_SI
        P_KS --> P_NT
    end

    %% --- State Layer ---
    subgraph State ["State Layer: Postgres DB\n(Active-Standby, Schema-per-Tenant)"]
        DB_L1 [("L1 Key References")]
        DB_L2 [("L2 Key Metadata\n(Account)")]
        DB_L3 [("L3 Key Metadata\n(Service)")]
        
        %% Key Hierarchy
        DB_L1 <-- "Parent" --> DB_L2
        DB_L2 <-- "Parent" --> DB_L3
    end

    %% --- Execution Data Plane ---
    subgraph EDP ["Execution Data Plane\nOpenKCM Crypto\n(The Muscle & Gateway)"]
        Krypton ["Krypton"]
        
        subgraph Krypton_Plugins ["Plugins"]
            KP_KMS ["Key Material Storage"]
            KP_KS ["Keystore"]
        end
        
        KMIP_Int_K ["KMIP Interface"]
        
        KG ["Krypton Gateway\n(Edge Deployments)"]
        
        subgraph KG_Plugins ["Plugins"]
            KGP_KMS ["Key Material Storage"]
        end
        
        KMIP_Int_KG ["KMIP Interface"]

        %% Connections within EDP
        Krypton <--> Krypton_Plugins
        Krypton <--> KMIP_Int_K
        KG <--> KG_Plugins
        KG <--> KMIP_Int_KG
    end

    %% --- External Systems ---
    subgraph EXT_L1 ["External Physical Vaults\n(L1 Root of Trust)"]
        AWS_KMS ["AWS KMS"]
        Azure_KV ["Azure Vault"]
        GCP_KMS ["GCP KMS"]
        Thales ["Thales HSM"]
        HashiCorp_L1 ["HashiCorp"]
    end
    
    subgraph EXT_KM ["External Physical Vaults\n(Key Material Storages)"]
        OpenBao ["OpenBao"]
        HashiCorp_KM ["HashiCorp"]
    end

    %% --- Client Apps ---
    Client1 ["Client Apps & Services"]
    L4_Client [("L4 DEK\n(Client-Side)")]
    
    Client2 ["Client Apps & Services"]
    L4_Server [("L4 DEK\n(Server-Side)")]

    %% --- Cross-Layer Connections ---
    
    %% GCP to External & State
    P_KS ==>|Manage L1\n(via Plugins)| EXT_L1
    CMK_API <==>|"Read/Write Policy & Pointers"| State

    %% EDP to State & External
    Krypton <-->|"Fetch L2/L3 Key Metadata"| State
    KP_KS <==>|"Crypto Ops\n(Wrap/Unwrap L2)"| EXT_L1
    KP_KMS <==>|"Key Material Ops\n(L2 & L3 Key Store)"| EXT_KM
    KGP_KMS <==>|"Key Material Ops\n(L4 Key Store)"| EXT_KM

    %% Client Interactions
    Client1 -- "KMIP\nEncrypt/Decrypt Req" --> KMIP_Int_K
    KMIP_Int_K -.->|Generates| L4_Client
    L4_Client -.->|Held By| Client1

    Client2 -- "KMIP\nCreate/Get Req" --> KMIP_Int_KG
    KMIP_Int_KG -.->|Generates & Stores| L4_Server
    L4_Server -.->|Used By| Client2

    %% CMK to Krypton Link
    CMK_API <==>|"Orbital Link (gRPC)\nPolicy & Control"| Krypton

    %% Styling
    classDef controlPlane fill:#dbeafe,stroke:#2563eb,stroke-width:2px,color:black;
    classDef stateLayer fill:#e5e7eb,stroke:#6b7280,stroke-width:2px,color:black;
    classDef dataPlane fill:#dcfce7,stroke:#16a34a,stroke-width:2px,color:black;
    classDef externalVault fill:#fef3c7,stroke:#d97706,stroke-width:2px,color:black;
    classDef database fill:#f3f4f6,stroke:#4b5563,stroke-width:2px,color:black,shape:cylinder;
    classDef component fill:#ffffff,stroke:#9ca3af,stroke-width:1px,color:black;
    classDef client fill:#f9fafb,stroke:#9ca3af,stroke-width:1px,color:black;

    class GCP,CMK_API,CMK_Plugins,P_ID,P_KS,P_SI,P_NT controlPlane;
    class State stateLayer;
    class DB_L1,DB_L2,DB_L3 database;
    class EDP,Krypton,Krypton_Plugins,KP_KMS,KP_KS,KMIP_Int_K,KG,KG_Plugins,KGP_KMS,KMIP_Int_KG dataPlane;
    class EXT_L1,AWS_KMS,Azure_KV,GCP_KMS,Thales,HashiCorp_L1,EXT_KM,OpenBao,HashiCorp_KM externalVault;
    class L1_Ref,L4_Client,L4_Server component;
    class Client1,Client2 client;
```

## Summary
This document establishes OpenKCM as the **Strategic Growth Catalyst** for the modern sovereign-cloud era. By transforming security from a passive compliance cost into an active market differentiator, it empowers SaaS providers to:

* **Secure High-Value Enterprise Contracts** by satisfying the most stringent global data sovereignty and residency requirements.
* **Enhance Tiered Monetization** by positioning advanced cryptographic isolation and root-key control as premium, high-margin capabilities.
* **Minimize Operational Liability** by ensuring the provider never possesses plaintext root-key material, shifting ultimate access control to the customer.
* **Eliminate Performance Bottlenecks** through a decentralized, gateway-native cryptographic model designed for sub-millisecond global execution.