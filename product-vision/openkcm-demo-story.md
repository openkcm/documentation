---
authors:
  - Aysan
status: Draft
last_updated: 2026-04-28
audience: Platform Mesh team , ApeiroRA stakeholders
---

# OpenKCM Demo Story: Data Sovereignty in Action

---

## The Setup

**Acme Corp** is a financial services company running on ApeiroRA Platform Mesh. They store sensitive customer records — names, account numbers, transaction history — in a MongoDB database running in their Platform Mesh account.

They have chosen **Mode B: Customer-Managed Key (BYOK)**. Their L1 root key lives in their AWS KMS account. They own it. The platform never touches the key material.

Two people are on stage for this demo:

| Persona | Role | What they see |
| :--- | :--- | :--- |
| **Priya** | Acme's IT Security Admin | The CMK MicroFrontend in Platform Mesh portal |
| **Tariq** | Acme's Developer | His app, querying MongoDB |

---

## Act 1: Everything Works

Tariq opens his application. He queries a customer record. The data loads instantly.

Behind the scenes:
- Tariq's app calls Krypton Gateway
- Krypton fetches the L4 DEK, derived from the chain: L1 (Acme's AWS key) → L2 (Domain Key) → L3 (MongoDB ServiceKey) → L4 (DEK)
- MongoDB record is decrypted and returned

**Tariq sees:** His customer data. Normal. Nothing remarkable.

Priya opens the CMK MicroFrontend. She sees the key chain:

```
L1 Root Key — AWS KMS / arn:aws:kms:eu-west-1:...    [Active]
  └── L2 Domain Key (platform-managed)               [Active]
        └── L3 MongoDB Production                    [Active]
```

Everything is green. Acme's data is protected and accessible.

---

## Act 2: The Acquisition

Acme Corp receives an acquisition offer. Legal counsel advises: before any due diligence begins, Acme must ensure the acquirer cannot access customer data without explicit consent.

Priya needs to pull the kill switch.

She goes to the CMK MicroFrontend and triggers a revocation request on the L1 Root Key. She enters a reason: *"Pre-acquisition data lockdown — legal instruction ref. ACQ-2026-004."*

Because this is a sensitive key operation, **Four-Eyes approval is required**. Priya cannot approve her own request. A second authorized person — Acme's Compliance Officer — receives a notification and logs in to approve.

Two approvals. Done.

The CMK Controller sees the approved revocation, calls Krypton, and broadcasts the revocation globally with `PRIORITY_CRITICAL`.

**Within 2 minutes**, every regional Krypton node has invalidated Acme's L1 key chain.

---

## Act 3: The Kill Switch Fires

Tariq refreshes his application.

He tries to load a customer record.

**Nothing loads.**

Not a crash. Not a 500 error. Just: *"Data not accessible."*

He tries again. Same result. He calls Priya.

Priya opens the CMK MicroFrontend. The key chain now shows:

```
L1 Root Key — AWS KMS / arn:aws:kms:eu-west-1:...    [Deactivated]
  └── L2 Domain Key (platform-managed)               [Deactivated]
        └── L3 MongoDB Production                    [Deactivated]
```

She tells Tariq: *"Yes. This is intentional. The data is locked. No one can read it — including us."*

The MongoDB database still exists. The records are still on disk. But without Acme's L1 key, the L4 DEKs cannot be derived. The data is **cryptographically inaccessible** — not deleted, but permanently unreadable to anyone who does not hold the L1 key.

Acme holds the L1 key. The acquirer does not.

---

## What This Demonstrates

| Capability | What the demo shows |
| :--- | :--- |
| **Kill switch** | One approved action locks all data globally within 2 minutes |
| **Four-Eyes approval** | No single admin can pull the kill switch alone |
| **Full chain enforcement** | L1 revocation cascades to L2, L3, L4 — no data escapes |
| **GDPR crypto-shredding** | Data is not deleted — it is rendered permanently unreadable |
| **End-user impact** | The developer feels it immediately — not an abstract concept |
| **Audit trail** | Every step — proposal, approval, revocation — is recorded and tamper-evident |
| **Customer sovereignty** | The platform cannot read Acme's data either — the key is Acme's |

---

## Act 4: Restoration (Optional)

If the acquisition does not proceed, Acme can restore access. Priya submits a re-activation request. Four-Eyes approval fires again. Once approved, the CMK Controller calls Krypton to re-validate the L1 key. The chain is restored.

Tariq refreshes his app. The data is back.

**The data was never gone. It was just locked.**

---

## What Needs to Be Built for This Demo

| Component | Status |
| :--- | :--- |
| CMK Controller — watches CRDs, calls Krypton | Showroom (needs hardening) |
| L1 Key Chain (L1 → L2 → L3) | Showroom CRDs exist |
| Kill switch — state transition on Tenant CRD | Needs implementation |
| Four-Eyes approval | Platform Mesh confirmed — [platform-mesh/backlog#267](https://github.com/platform-mesh/backlog/issues/267) |
| CMK MicroFrontend (Key Chain view) | Needs implementation |
| End-user app showing data lockout | Needs a simple demo app |
| Audit trail view | Depends on Platform Mesh audit capability (Q2) |
