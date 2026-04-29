---
authors:
  - Aysan
status: Draft
last_updated: 2026-04-27
---

# CMK Controller: Visual Overview

---

## 1. Architecture: How It Works

Who owns what and how the pieces connect.

```mermaid
%%{init: {'theme': 'base', 'themeVariables': {'fontSize': '18px', 'fontFamily': 'arial'}}}%%
flowchart LR
    Admin(("👤 Admin"))

    subgraph PM ["  Platform Mesh  "]
        direction LR
        MFE(["CMK MicroFrontend (Luigi)"])
        L1[/"L1 RootKey CR"/]
        L2["L2 DomainKey CR"]
        L3["L3 ServiceKey CRs"]
        SVC[("Services")]
        MFE --> L1 --> L2 --> L3 --> SVC
    end

    subgraph OKCM ["  OpenKCM  "]
        direction LR
        CC["CMK Controller"]
        KR["Krypton"]
        KGW["Krypton Gateway"]
        L4["L4 DEK"]
        CC -->|"Call API"| KR
        KR --> KGW
        KR -->|"generates"| L4
    end

    subgraph EXT ["  Customer Key Backends  "]
        direction LR
        AWS(["AWS KMS"]) ~~~ AZ(["Azure Key Vault"]) ~~~ GCP(["GCP KMS"]) ~~~ HSM(["🔒 HSM"]) ~~~ VAULT(["Vault / OpenBao"])
    end

    Admin -->|"manages via"| MFE
    L1 -->|"watch CRs"| CC
    KR -->|"validate"| EXT

    style PM fill:#dbeafe,stroke:#3b82f6
    style OKCM fill:#f5f3ff,stroke:#8b5cf6
    style EXT fill:#f0fdf4,stroke:#22c55e
    style Admin fill:#fef9c3,stroke:#eab308
    style L1 fill:#bfdbfe,stroke:#2563eb
    style MFE fill:#93c5fd,stroke:#1d4ed8
    style HSM fill:#bbf7d0,stroke:#16a34a
    style CC fill:#ddd6fe,stroke:#7c3aed
    style KR fill:#ddd6fe,stroke:#7c3aed
    style KGW fill:#ddd6fe,stroke:#7c3aed
    style L4 fill:#ede9fe,stroke:#7c3aed
```

---

## 2. Customer Journey: Two Modes

How a customer goes from account creation to full sovereignty.

```mermaid
flowchart LR
    A[Account Created\nin Platform Mesh] --> B{Customer brings\nown key?}

    B -->|No| C[Mode A\nPlatform-Managed L1]
    C --> D[Krypton provisions\nL1 internally]
    D --> E[Encryption available\nimmediately]
    E --> F[No kill switch\nNo sovereignty]

    B -->|Yes| G[Mode B\nCustomer-Managed L1\nBYOK / HYOK]
    G --> H[Customer declares\nL1 reference on\nTenant CRD]
    H --> I[CMK Controller\ncalls Krypton\nto validate]
    I -->|Valid| J[Key transitions\nto Active]
    J --> K[Full sovereignty\nKill switch active\nFour-Eyes enabled]

    F -.->|upgrade at any time| G

    style C fill:#fff3e0,stroke:#FF9800
    style G fill:#e8f5e9,stroke:#4CAF50
    style F fill:#fff3e0,stroke:#FF9800
    style K fill:#e8f5e9,stroke:#4CAF50
```

---

## 3. Key Lifecycle: NIST States

How a key moves through its lifecycle and what each state means for the customer.

```mermaid
stateDiagram-v2
    [*] --> Active : L1 declared & validated

    Active --> Suspended : L1 unavailable or unlinked\n(accidental / external issue)
    Suspended --> Active : Customer re-links L1\n(within grace period)
    Suspended --> Deactivated : Grace period expires\n(30 / 60 / 90 days)\nno action taken

    Active --> Deactivated : Kill switch executed\n(Four-Eyes approved)
    Deactivated --> Destroyed : Explicit approved destroy

    state Active {
        Encrypt: ✅ Encrypt allowed
        Decrypt: ✅ Decrypt allowed
    }

    state Suspended {
        note: Customer notified immediately\nGrace period running
        Encrypt2: ❌ Encrypt blocked
        Decrypt2: ✅ Decrypt allowed
    }

    state Deactivated {
        Encrypt3: ❌ Encrypt blocked
        Decrypt3: ❌ Decrypt blocked
    }

    state Destroyed {
        Encrypt4: ❌ Encrypt blocked
        Decrypt4: ❌ Decrypt blocked
        note2: Key material gone
    }
```

---

## 4. Before vs. After

What gets replaced and what stays.

```mermaid
flowchart LR
    subgraph Before ["Before: Standalone CMK"]
        B1[CMK API Server]
        B2[CMK Registry\nPostgreSQL]
        B3[CMK Approval Engine]
        B4[CMK RBAC]
        B5[CMK Audit DB\nPostgreSQL]
        B6[CMK Tenant Manager]
        B7[CMK Task Scheduler]
        B8[CMK Portal\nstandalone]
        B9[Event Bridge\nto Platform Mesh]
    end

    subgraph After ["After: CMK Controller"]
        A1[CMK Controller\n1 Kubernetes controller]
        A2[Tenant CRD\netcd]
        A3[Platform Mesh\nApproval]
        A4[Kubernetes RBAC]
        A5[Platform Mesh\nAudit Trail]
        A6[Platform Mesh\nPortal - Luigi]
    end

    Before -->|replaced by| After

    style Before fill:#ffebee,stroke:#F44336
    style After fill:#e8f5e9,stroke:#4CAF50
```

---

## 5. CMK MicroFrontend: Proposed UI Structure

Three sections replace five — Key Configurations and Systems are blended into a single Key Chain view.

```mermaid
%%{init: {'theme': 'base', 'themeVariables': {'fontSize': '18px', 'fontFamily': 'arial'}}}%%
flowchart TD
    subgraph NAV ["  CMK MicroFrontend Navigation  "]
        direction LR
        S1["1. Key Chain"]
        S2["2. Approvals"]
        S3["3. Access"]
    end

    subgraph KC ["  Key Chain View  "]
        direction TB
        L1["L1 Root Key\nAWS KMS / arn:aws:kms:..."]
        L2["L2 Domain Key\nauto-provisioned by platform"]
        L3A["L3 MongoDB Production"]
        L3B["L3 PostgreSQL Billing"]
        L3C["+ Add Service Key"]
        L1 --> L2
        L2 --> L3A
        L2 --> L3B
        L2 --> L3C
    end

    subgraph AP ["  Approvals View  "]
        direction TB
        AP1["Pending Four-Eyes requests"]
        AP2["Propose / Approve / Reject"]
        AP1 --- AP2
    end

    subgraph AC ["  Access View  "]
        direction TB
        AC1["Groups & Roles"]
        AC2["Who can manage keys\nfor this tenant"]
        AC1 --- AC2
    end

    S1 --> KC
    S2 --> AP
    S3 --> AC

    style NAV fill:#dbeafe,stroke:#3b82f6
    style KC fill:#f5f3ff,stroke:#8b5cf6
    style AP fill:#fef9c3,stroke:#eab308
    style AC fill:#f0fdf4,stroke:#22c55e
    style L1 fill:#bfdbfe,stroke:#2563eb
    style L2 fill:#ddd6fe,stroke:#7c3aed
    style L3A fill:#ede9fe,stroke:#7c3aed
    style L3B fill:#ede9fe,stroke:#7c3aed
    style L3C fill:#f3f4f6,stroke:#9ca3af
```
