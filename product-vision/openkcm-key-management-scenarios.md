---
authors:
  - Aysan
status: Draft
last_updated: 2026-04-27
---

# OpenKCM Key Management Scenarios: Platform-Managed vs. Customer-Managed Keys

## Overview

Every account on the platform gets encryption keys to encrypt their data. What differs is **who controls the root key** — and therefore who can exercise data sovereignty.

There are two distinct scenarios. They are not just configuration options; they represent fundamentally different trust models with different capabilities and responsibilities.

---

## Scenario A: Platform-Managed Encryption (Fallback)

### What it is

When an account is created and the customer does not bring their own key, the platform automatically provisions encryption using a Krypton-managed L1 root key. The customer's data is encrypted from day one without any action required.

### Who owns the key

The **platform** (Krypton) generates, stores, and rotates the L1 key material. The customer never sees or touches the key.

### What the customer gets

- Encryption at rest and in transit, immediately available
- No setup, no credentials, no key store required
- Key rotation handled automatically by the platform

### What the customer does NOT get

| Capability | Available? | Reason |
| :--- | :--- | :--- |
| Kill switch (instant data access revocation) | ❌ | Platform controls the key — customer cannot pull it |
| GDPR crypto-shredding | ❌ | Customer cannot destroy a key they do not own |
| Data sovereignty guarantee | ❌ | Data is only as sovereign as the platform's key management |
| Audit trail of key access | ❌ | Key operations are internal to the platform |
| Key portability | ❌ | Key material never leaves the platform |

### CMK controller involvement

**None.** The platform-managed L1 is provisioned and maintained entirely by Krypton as part of the account bootstrap. The CMK controller does not act on platform-managed keys.

The L1 RootKey CR exists in etcd to make the state visible and auditable, but it is owned and operated by the platform infrastructure, not by the CMK controller.

### When customers use this

- Customers who have no compliance requirement to hold their own key
- Accounts in early stages that intend to upgrade to BYOK/HYOK later
- Workloads where the platform's security posture is sufficient

---

## Scenario B: Customer-Managed Key (BYOK / HYOK)

### What it is

The customer provides their own L1 root key hosted in their own key store — AWS KMS, GCP KMS, Azure Key Vault, HashiCorp Vault / OpenBao, or an on-premises HSM (PKCS#11). The CMK controller validates the key, activates it, and from that point manages its full lifecycle.

**BYOK (Bring Your Own Key):** The customer's key is hosted in a cloud KMS the platform can reach and validate.

**HYOK (Hold Your Own Key):** The customer's key lives in an on-premises or air-gapped HSM. The platform validates it but the key material never leaves the customer's environment.

### Who owns the key

The **customer**. The platform holds only a reference (ARN, endpoint, path) — never the key material itself.

### What the customer gets

| Capability | Available? |
| :--- | :--- |
| Kill switch (instant global revocation within 2 minutes) | ✅ |
| GDPR crypto-shredding (destroy the key = destroy the data) | ✅ |
| Data sovereignty guarantee | ✅ |
| Full audit trail of all key lifecycle events | ✅ |
| Key portability (customer can take their key anywhere) | ✅ |
| Four-Eyes approval on sensitive operations | ✅ |

### CMK controller involvement

**Full ownership.** The CMK controller manages the entire lifecycle:

1. **Onboarding** — Customer provides a key reference on the L1 RootKey CR. The CMK controller calls Krypton to validate the key is accessible and the algorithm is supported.
2. **Activation** — Once validated, the controller transitions the L1 RootKey CR to `Active`. All L2/L3/L4 keys are now chained to the customer's key.
3. **Lifecycle management** — The controller enforces NIST SP 800-57 state transitions. Invalid transitions are rejected. Suspended keys allow decrypt but block encrypt.
4. **Kill switch** — The customer (or admin) creates an `L1KeyRevocation` CR. The controller enforces the Four-Eyes approval (minimum 2 distinct authorised approvers; proposer ≠ approver). Once approved, Krypton broadcasts revocation globally via Orbital with `PRIORITY_CRITICAL`. Full propagation within 2 minutes.
5. **Recovery** — If the customer's key store becomes unreachable, the controller surfaces a `ValidationFailed` condition and retries. Existing decrypt continues; new encrypt is blocked.

### When customers use this

- Customers with GDPR, TISAX, or SOC2 requirements that mandate key ownership
- Customers who need a data destruction guarantee (crypto-shredding)
- Customers operating in regulated industries (financial services, healthcare, public sector)
- Customers who want the kill switch as a contractual sovereignty guarantee

---

## Transition: Platform-Managed → Customer-Managed

A customer can upgrade from Scenario A to Scenario B at any time without disrupting running services.

**Flow:**
1. Customer configures their key reference on the L1 RootKey CR via the CMK MicroFrontend.
2. The CMK controller validates the new key via Krypton.
3. Krypton performs key re-wrapping: L2 keys are re-wrapped under the customer's new L1. This is transparent to services.
4. The old platform-managed key reference is discarded.
5. From this point, the customer has full sovereignty. The kill switch is now active.

**There is no service interruption during transition.**

Reversal (Customer-Managed → Platform-Managed) is technically possible but removes all sovereignty guarantees. This is a deliberate downgrade and must be treated as such in the MicroFrontend UX.

---

## Summary

| | Platform-Managed (Scenario A) | Customer-Managed (Scenario B) |
| :--- | :--- | :--- |
| Setup required | None | Customer provides key reference |
| Key ownership | Platform (Krypton) | Customer |
| Kill switch | ❌ | ✅ (2-min global SLA) |
| GDPR crypto-shredding | ❌ | ✅ |
| CMK controller manages | Nothing | Full L1 lifecycle |
| Suitable for | All accounts by default | Regulated / sovereignty-sensitive workloads |
| Upgrade path | → Scenario B at any time | ← Downgrade possible (loses sovereignty) |
