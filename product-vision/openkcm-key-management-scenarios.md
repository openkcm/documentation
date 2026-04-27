---
authors:
  - Aysan
status: Draft
last_updated: 2026-04-27
---

# OpenKCM Key Management Scenarios: Platform-Managed vs. Customer-Managed Keys

## Overview

Every account that enables OpenKCM gets an L1 key reference automatically provisioned at account creation. What differs is **who owns the key material behind that reference** — and therefore who can exercise full data sovereignty.

There are two modes. They are not just configuration options; they represent fundamentally different trust models with different capabilities and responsibilities. Every account has an L1 — there is no mode with no key governance.

---

## Mode A: Platform-Managed L1 (Default)

### What it is

When an account enables OpenKCM, the platform automatically provisions an L1 key reference at account creation. The customer's services can immediately request encryption keys without any setup. The CMK controller manages the lifecycle of this platform-held key.

### Who owns the key

**The platform (provider).** Krypton generates and manages the L1 key material. The customer never sees or touches the key material itself.

### What the customer gets

- Encryption keys available immediately on account creation
- No setup, no credentials, no external key store required
- Key rotation handled automatically by the platform

### What the customer does NOT get

| Capability | Available? | Reason |
| :--- | :--- | :--- |
| Kill switch (instant data access revocation) | ❌ | Platform controls the key — customer cannot pull it |
| GDPR crypto-shredding | ❌ | Customer cannot destroy a key they do not own |
| Full data sovereignty guarantee | ❌ | Data is only as sovereign as the platform's key management |
| Four-Eyes approval on key operations | ❌ | No customer-owned key to govern |
| Key portability | ❌ | Key material stays within the platform |

### CMK controller involvement

**None.** The platform-managed L1 is provisioned and maintained entirely by Krypton as part of account bootstrap. The CMK controller only governs customer-declared key references — there is nothing for it to govern here.

Key governance capabilities (kill switch, Four-Eyes, audit trail, crypto-shredding) are unlocked when the customer brings their own key in Mode B.

### When customers use this

- Accounts with no compliance requirement to hold their own key
- Accounts in early stages that intend to upgrade to BYOK/HYOK later
- Workloads where the platform's security posture is sufficient

---

## Mode B: Customer-Managed Key (BYOK / HYOK)

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
| Full data sovereignty guarantee | ✅ |
| Full audit trail of all key lifecycle events | ✅ |
| Key portability (customer can take their key anywhere) | ✅ |
| Four-Eyes approval on sensitive operations | ✅ |

### CMK controller involvement

**Full ownership.** The CMK controller manages the entire lifecycle:

1. **Onboarding** — Customer provides a key reference on the L1 RootKey CR. The CMK controller calls Krypton to validate the key is accessible and the algorithm is supported.
2. **Activation** — Once validated, the controller transitions the L1 RootKey CR to `Active`. All L2/L3/L4 keys are now chained to the customer's key.
3. **Lifecycle management** — The controller enforces NIST SP 800-57 state transitions. Invalid transitions are rejected. Suspended keys allow decrypt but block encrypt.
4. **Kill switch** — The customer (or admin) submits an explicit approved revocation request requiring Four-Eyes approval (minimum 2 distinct authorised approvers; proposer ≠ approver). Once approved, Krypton broadcasts revocation globally with `PRIORITY_CRITICAL`. Full propagation within 2 minutes.
5. **Recovery** — If the customer's key store becomes unreachable, the controller surfaces a `ValidationFailed` condition and retries. Existing decrypt continues; new encrypt is blocked.

### When customers use this

- Customers with GDPR, TISAX, or SOC2 requirements that mandate key ownership
- Customers who need a data destruction guarantee (crypto-shredding)
- Customers operating in regulated industries (financial services, healthcare, public sector)
- Customers who want the kill switch as a contractual sovereignty guarantee

---

## Transition: Platform-Managed → Customer-Managed

A customer can upgrade from Mode A to Mode B at any time without disrupting running services.

**Flow:**
1. Customer configures their key reference on the L1 RootKey CR via the CMK MicroFrontend.
2. The CMK controller validates the new key via Krypton.
3. Krypton performs key re-wrapping: L2 keys are re-wrapped under the customer's new L1. This is transparent to services.
4. The old platform-managed key reference is retired.
5. From this point, the customer has full sovereignty. The kill switch is now active.

**There is no service interruption during transition.**

Reversal (Customer-Managed → Platform-Managed) is technically possible but removes all sovereignty guarantees. This is a deliberate downgrade and must be treated as such in the MicroFrontend UX.

---

## Summary

| | Platform-Managed (Mode A) | Customer-Managed (Mode B) |
| :--- | :--- | :--- |
| Setup required | None — auto-provisioned at account creation | Customer provides key reference |
| Key ownership | Platform (provider) | Customer |
| Kill switch | ❌ | ✅ (2-min global SLA) |
| GDPR crypto-shredding | ❌ | ✅ |
| Four-Eyes governance | ❌ | ✅ |
| Audit trail | Platform-internal | ✅ Customer-visible |
| CMK controller manages | Nothing — Krypton owns it | Full customer L1 lifecycle |
| Suitable for | All accounts by default | Regulated / sovereignty-sensitive workloads |
| Upgrade path | → Mode B at any time | ← Downgrade possible (loses sovereignty) |
