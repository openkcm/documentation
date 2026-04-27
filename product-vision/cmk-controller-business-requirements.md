---
authors:
  - Aysan
status: Draft
last_updated: 2026-04-27
---

# CMK Controller: Business Requirements

## Purpose

The CMK Controller is the governance layer of OpenKCM — the component that controls who can access what and under what conditions. It replaces the standalone CMK service by delivering the same governance capabilities natively within Platform Mesh.

These are the business capabilities the CMK Controller must provide.

---

## Requirements

### 1. Tenant Onboarding

When a new account is created in Platform Mesh, OpenKCM must automatically provision that tenant without any manual intervention. The tenant must be ready to use encryption services immediately after account creation.

### 2. L1 Key Governance (BYOK / HYOK)

Customers who choose to bring their own key must be able to register their external key reference (AWS KMS, GCP KMS, Azure Key Vault, HSM, Vault) with OpenKCM. The platform validates the key is accessible and manages its full lifecycle from activation through to destruction.

### 3. Kill Switch

Customers must be able to instantly revoke their data access by revoking their L1 key. This is the customer's ultimate sovereignty control — once executed, no data encrypted under that key chain is accessible. Revocation must propagate globally within 2 minutes.

### 4. Key Suspension with Grace Period

If a customer's L1 key becomes unavailable or unreachable — due to misconfiguration, accidental unlinking, or an external key store issue — the system must enter a grace period (30, 60, or 90 days, configurable) during which new encryption is blocked but existing data remains readable. The customer is notified immediately and throughout the grace period to allow remediation. If no action is taken by the end of the grace period, the key transitions to permanently deactivated. Deliberate revocation (the kill switch) bypasses this grace period entirely.

### 5. Multi-Party Approval (Four-Eyes)

No single administrator can perform sensitive key operations alone. Any action that affects an L1 key — linking, revoking, destroying — must require approval from a minimum of two distinct authorized individuals. The person proposing the action cannot be one of the approvers.

### 6. Audit Trail

Every governance decision must be recorded in a tamper-evident, non-repudiable audit log — who proposed it, who approved it, when it happened, and what changed. This record must satisfy external auditors for SOC2, TISAX, and GDPR compliance.

### 7. Access Control

Key governance operations must be scoped to the tenant. An administrator for Tenant A must have zero visibility into or control over Tenant B's key references. Permissions must be enforceable at the individual key level, not just at the account level.

### 8. Customer Notifications

Customers must receive timely notifications for all significant key lifecycle events — key activation, suspension, grace period warnings, expiry, and revocation. Customers should never be surprised by a change in their key state.

### 9. Tenant Identity and Authentication

Each tenant must have its own identity configuration defining who is authorized to manage their keys. This includes integration with the customer's identity provider so that key governance operations are tied to verified corporate identities.

### 10. Governance Portal

Administrators must have a user interface to perform all of the above operations — onboarding tenants, managing key references, triggering and approving governance actions, and viewing the audit trail. This interface must be accessible within the Platform Mesh portal without requiring a separate login or context switch.

### 11. Tenant Suspension and Restoration

The platform must be able to temporarily suspend a tenant's account-level access — for compliance holds, investigations, or administrative reasons — and restore it when the issue is resolved. This is an administrative action taken by the platform operator, distinct from key-level suspension. Suspended tenants cannot perform new key operations but their existing data remains accessible.

### 12. Tenant Termination

When an account is permanently closed, OpenKCM must cleanly offboard the tenant, retiring their key references and ensuring no orphaned state remains in the system.
