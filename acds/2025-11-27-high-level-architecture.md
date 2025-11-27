---
authors:
  - NicolaeNicora # Change to your own handle. Add yourself to "authors.yml" if necessary.
---

# High Level Architecture

## 1. Introduction

OpenKCM (Open Key Chain Management) is a **multi-tenant, multi-cloud key management platform** designed for:

- End-to-end key lifecycle management
- Strong tenant isolation
- Integration with external KMS/HSMs
- Flexible key usage patterns (BYOK, HYOK, CYOK/MYOK)
- Automated and manual MasterKey recovery

The platform supports both **internal keys** and **crypto edge ephemeral keys**, enabling services and applications to securely perform encryption and decryption operations without direct access to root key material.

---

## 2. Layered Key Hierarchy Overview

OpenKCM employs a **hierarchical key model**:

```
L1 → (L2 → L2.1) → L3 → L4
```

### **Layer Responsibilities**

| Layer | Scope | Purpose | Storage/Keystore | Notes |
|-------|-------|---------|----------------|-------|
| **L1** | Global / Tenant Root | Customer Master Key (CMK) | External KMS/HSM (AWS, Azure, GCP, Vault, HSM) | Root of trust. Supports BYOK/HYOK/CYOK. |
| **L2** | Tenant | Tenant Encryption Keys | Internal Vault or External KMS | Encrypts L2.1 / L3 keys. Rotated per tenant policy. |
| **L2.1** | Tenant Versioned | Per-version or per-resource tenant keys | Internal Keystore | Provides key versioning and rollback capabilities. |
| **L3** | Service | Service/Application-specific keys | Internal Keystore | Encrypts L4 keys used by client services. |
| **L4** | Crypto Edge | Client ephemeral keys | Ephemeral storage at client | Short-lived keys for application use. Accessed via KMIP. |

---

## 3. Key Lifecycle & Management

### **Key Generation**
- L1 generated externally or provided by customer (BYOK/HYOK/CYOK)
- L2/L2.1 generated internally and wrapped by L1
- L3 generated at service level and wrapped by L2
- L4 generated at edge level on behind of customer and wrapped by L3

### **Key Rotation**
- **L1**: Rotated per external KMS/HSM policy. Manual rotation supported for BYOK.
- **L2**: Rotated on tenant lifecycle events or scheduled rotation.
- **L2.1**: Rotated per version or resource-specific requirements.
- **L3**: Rotated on service deployment or security events.
- **L4**: Ephemeral, rotated per request from client side.

### **Key Expiration and Revocation**
- Expired keys marked for deprecation.
- Revoke access triggers re-encryption of dependent layers.
- Audit logs maintained for compliance.

---

## 4. Keystore Integration

OpenKCM abstracts multiple external and internal keystore implementations:

| Keystore Type | Supported Layers | Notes |
|---------------|----------------|------|
| AWS KMS | L1, L2 | Supports BYOK, automatic rotation, IAM integration |
| Azure Key Vault | L1, L2 | Key Vault policies control access |
| GCP KMS | L1, L2 | Can integrate with internal L2/L3 keys |
| Vault/OpenBao | L1-L3 | Internal multi-tenant keys, HSM integration optional |
| HSMs (Thales, nShield, IBM) | L1 | Required for HYOK deployments, offline key storage |
| Internal Keystore | L2.1, L3, L4 | Managed internally, supports key versioning |

---

## 5. Master Key Management

### 5.1 Shamir Secret Sharing (SSS)
- MasterKey split into **N shards**; **M shards required** to reconstruct.
- Each shard encrypted using a separate keystore.
- Stored in the database or distributed secure storage.
- Recovery:
    1. Retrieve M shards
    2. Decrypt each shard
    3. Reconstruct MasterKey
- **Use Case:** High-security, multi-cloud resilience, disaster recovery.

### 5.2 Seal Auto-Unseal
- MasterKey encrypted using a single KMS/HSM (AWS, OpenBao).
- System requests decryption at startup.
- **Use Case:** Automated cloud-native environments, containerized deployments.

---

## 6. Encryption Flow (L1 → L4)

1. **L1** encrypts **L2 keys**
2. **L2** encrypts **L2.1** and/or **L3 keys**
3. **L3** encrypts **L4 keys**
4. **L4** used for ephemeral encrypt/decrypt at crypto edge
5. Data flow:
    - Client services interact with L4 via **KMIP operations**:
        - `Create/Get`
        - `Encrypt/Decrypt`
    - L4 keys are ephemeral, rotated per request/session

---

## 7. Security Controls

- **Access Control**
    - Tenant isolation enforced at L2/L2.1
    - Service-level access restricted via IAM/ACL
- **Audit Logging**
    - Key creation, rotation, and access logged
    - Cloud-native logging integrated with KMS/HSM
- **Separation of Duties**
    - Different layers of keys have distinct operators
    - L1/HSM operators cannot access L4 ephemeral keys

---

## 8. Monitoring & Compliance

- **Key usage metrics**
    - Access frequency, failed attempts, rotation status
- **Rotation compliance**
    - Automated reporting for rotation schedules
- **Audit**
    - FIPS 140-2 compliance for HSMs
    - PCI DSS / SOC2 audit readiness

---

## 9. Operational Considerations

- **Deployment**
    - L1 requires secure keystore connectivity
    - L2/L3 keys provisioned via internal vault
    - L4 keys generated at client or service edge
- **Backup**
    - Encrypted backups of all internal keys
    - Shard backups for MasterKey recovery (SSS)
- **Disaster Recovery**
    - Multi-cloud support ensures resilience for L1
    - Shards distributed across regions for SSS

---

## 10. Use Cases

1. **Multi-Tenant SaaS**
    - Each tenant has independent L2/L2.1 keys
    - Tenant data encrypted end-to-end
2. **Cloud Migration**
    - Keys can be moved between cloud KMS providers
3. **High-Security Workloads**
    - L1 in HSM (HYOK), MasterKey protected via SSS
4. **Edge-Compute Applications**
    - L4 ephemeral keys used for per-request encryption

---

## 11. Summary

OpenKCM provides:

- A hierarchical, multi-layer key management system
- Flexible integration with external keystores (BYOK/HYOK/CYOK)
- MasterKey lifecycle via **SSS or Seal**
- Tenant and service isolation
- KMIP-based crypto edge for ephemeral key usage
- Auditability, rotation, and disaster recovery mechanisms