---
authors:
  - Aysan
status: Draft
last_updated: 2026-04-29
audience: Showroom team (Igor), Platform Mesh team
---

# CMK MicroFrontend — Kickoff Specification

This document gives the showroom team everything needed to start building the CMK MicroFrontend as a native Platform Mesh micro-frontend (Luigi). It covers what exists today in the standalone CMK UI, what the new model looks like, the key concepts behind it, and the screens that need to be built.

---

## Context: What Is the CMK MicroFrontend?

OpenKCM is moving its governance UI from a standalone portal into Platform Mesh natively. Instead of a separate application, the CMK governance UI becomes a **Luigi micro-frontend** living inside the Platform Mesh portal — the same place customers manage everything else.

The customer should never need to leave the Platform Mesh portal to manage their encryption keys.

---

## What Exists Today: The Standalone CMK UI

The current CMK UI (`cmk-ui`) is a UI5 application with five sections:

### 1. Key Configurations
The main screen. Lists all key configurations for the tenant.

- Each configuration has a name, ID, and description
- Inside a configuration: a list of **keys** (BYOK, HYOK, or Platform-Managed) and a list of **systems** assigned to it
- Actions: Create configuration, create key (wizard), activate/deactivate key, assign/unassign systems

**Key Creation Wizard (inside a configuration):**
- Step 1: Select key type — BYOK, HYOK, or Managed
- Step 2: Select provider — AWS KMS, GCP KMS, Azure Key Vault, HashiCorp Vault, HSM
- Step 3: Enter key details — name, description, algorithm, region, backend endpoint
- Step 4: Review and confirm

**Key lifecycle states (what the customer sees):**
- Generated → Initializing → Processing → Active → Suspended → Deactivated

### 2. Systems
Lists all systems (workloads) connected to CMK for the tenant — databases, BTP subaccounts, apps.

- Filter by region, system type, assigned key configuration
- Click a system to see its details and switch which key configuration it uses
- A system points to a key configuration, not a key directly

### 3. Tasks
Four-Eyes approval workflows.

- Lists all pending and completed approval tasks
- Actions: view task details, approve, reject
- Each task shows: who proposed, what operation, what key, current approval status

### 4. User Groups
Role-based access control.

- Lists all user groups for the tenant
- Each group has members and assigned roles: Key Administrator, Tenant Administrator, Tenant Auditor
- Actions: create group, add/remove members, assign/revoke roles

### 5. Settings
Language selection only.

---

## What Changes in the New Model

The new CMK MicroFrontend is simpler and better aligned with how Platform Mesh works. The key architectural change is:

**Key Configurations + Systems → one unified Key Chain view**

In the old model, a "system" was a separate entity that pointed to a key configuration. In the new model, a workload (MongoDB, PostgreSQL) simply gets an **L3 ServiceKey CR** — that ServiceKey IS the binding between the workload and the key chain. No separate systems tab needed.

### Old vs New

| Old CMK UI | New CMK MicroFrontend |
| :--- | :--- |
| Key Configurations tab | Key Chain view (unified) |
| Systems tab | Gone — replaced by L3 ServiceKeys |
| Tasks tab | Approvals view |
| User Groups tab | Access view |
| Settings | Settings (unchanged) |
| 5 sections | 3 sections |

---

## The Key Chain Model Igor Needs to Understand

Every customer account has a key chain. It always has three levels:

```
L1 Root Key                          ← Customer's key (BYOK/HYOK) or Platform-managed
  └── L2 Domain Key                  ← Auto-provisioned by platform at account creation, read-only
        ├── L3 MongoDB Production    ← Customer's workload
        ├── L3 PostgreSQL Billing    ← Customer's workload
        └── + Add Service Key
```

**L1 — Root Key:**
- Two modes:
  - **Mode A (Platform-Managed):** Provisioned automatically, customer doesn't touch it, no kill switch
  - **Mode B (Customer-Managed / BYOK / HYOK):** Customer declares their own key from AWS KMS, GCP KMS, Azure Key Vault, HashiCorp Vault, or HSM. Full sovereignty, kill switch active.
- Customer can upgrade from Mode A → Mode B at any time
- Downgrading from Mode B → Mode A is possible but loses all sovereignty guarantees — must be clearly warned in the UI

**L2 — Domain Key:**
- Always auto-provisioned by the platform when the account is created
- Customer never creates or edits this
- Show it in the UI as read-only context

**L3 — Service Keys:**
- One per workload (MongoDB, PostgreSQL, any service that needs encryption)
- Customer names them and manages their lifecycle
- States: Generated → Active → Deactivated
- Customer can create, activate, deactivate, delete

**Kill Switch:**
- Only available in Mode B (customer-managed L1)
- Transitions L1 from Active → Deactivated
- Requires Four-Eyes approval — cannot be done by one person alone
- Once fired: all L2/L3/L4 keys in the chain become inaccessible within 2 minutes
- Data is not deleted — it is cryptographically locked

**Key States the UI must show clearly:**

| State | Encrypt | Decrypt | What customer sees |
| :--- | :--- | :--- | :--- |
| Active | ✅ | ✅ | Green — all good |
| Suspended | ❌ | ✅ | Orange — grace period running, X days left |
| Deactivated | ❌ | ❌ | Red — data inaccessible |
| Destroyed | ❌ | ❌ | Red — key material gone permanently |

---

## Screens to Build

### Screen 1: Key Chain (Main Screen)

The default landing screen. Shows the full key chain for the tenant.

**What it shows:**
- L1 Root Key — provider (AWS KMS / Vault / HSM / Platform-managed), ARN/endpoint, current state badge
- L2 Domain Key — auto-provisioned, read-only, state badge
- L3 Service Keys — list of workloads, each with name, state badge, actions

**Actions from this screen:**
- Declare L1 (if Mode A — upgrade to BYOK/HYOK)
- Replace L1 (if Mode B — change key reference, requires Four-Eyes)
- Trigger Kill Switch (Mode B only — requires Four-Eyes, prominent but not accidental)
- Add Service Key (create new L3)
- Activate / Deactivate / Delete a Service Key

**Mode A → Mode B upgrade:**
- Clear CTA: "Bring your own key" 
- Multi-step wizard: choose provider → enter key reference/credentials → confirm
- Warning screen before confirming: explain what changes (sovereignty gained, platform no longer holds the key)

**Mode B → Mode A downgrade:**
- Available but must show a strong warning: "You will lose the kill switch and full data sovereignty"
- Requires Four-Eyes approval

**Grace period indicator (Suspended state):**
- Show days remaining prominently on the L1 card
- Link to notification history

### Screen 2: Approvals

All Four-Eyes approval requests — pending and history.

**What it shows:**
- Pending requests (need action)
- Completed requests (history)
- Each request: operation type, proposed by, proposed at, current approvers, status

**Actions:**
- Approve a pending request
- Reject a pending request
- Propose a new operation (e.g., kill switch, L1 replacement, key destroy)
- Rule enforced: proposer cannot be one of the approvers

### Screen 3: Access

Who can manage keys for this tenant.

**What it shows:**
- User groups and their assigned roles
- Roles: Key Administrator, Tenant Administrator, Tenant Auditor

**Actions:**
- Create group
- Add/remove members
- Assign/revoke roles

---

## What Igor Does NOT Need to Build

- The audit trail UI — this comes from Platform Mesh natively
- Notification delivery — Platform Mesh handles delivery; OpenKCM publishes events
- L2 Domain Key management — read-only, auto-provisioned
- Authentication / SSO — Platform Mesh handles this via Luigi context injection

---

## Integration Notes for Luigi

- The MicroFrontend runs inside the Platform Mesh portal via Luigi
- Tenant context (tenant ID, user identity, roles) is injected by Luigi — do not re-implement auth
- Navigation registration follows the Platform Mesh MSP pattern
- Four-Eyes approval integrates with Platform Mesh's native approval mechanism (confirmed, tracked in [platform-mesh/backlog#267](https://github.com/platform-mesh/backlog/issues/267))
- Notifications published by OpenKCM are delivered to customers through Platform Mesh

---

## Screenshots of the Existing CMK UI

> **TODO:** Add screenshots of the following screens from `cmk-ui` for reference:
> - Key Configurations list view
> - Key Configuration detail view (with keys and systems)
> - Key Creation Wizard (all 4 steps)
> - Tasks / approval workflow view
> - User Groups view

These screenshots will help the showroom team understand what the current UX looks like and what to improve or replicate in the new MicroFrontend.

---

## Open Product Decisions

These are unresolved questions that need answers before or during implementation. They affect the scope and behaviour of the MicroFrontend.

### 1. Should L2 Domain Key be shown in the UI?

The current showroom UI shows the DomainKey in the header bar (e.g. "ig-1 / Active"). L2 is auto-provisioned by the platform and the customer never touches it. Showing it adds complexity without adding customer value — it is internal plumbing.

**Options:**
- Remove L2 from the UI entirely — show only L1 and L3
- Keep L2 as read-only context in the header (current approach)

**Recommendation:** Remove it. The customer cares about L1 (do I own my key?) and L3 (which workloads are protected?). L2 is invisible infrastructure.

---

### 2. Should L3 ServiceKeys have Activate / Deactivate controls?

The current showroom UI has Activate and Deactivate buttons on each L3 ServiceKey card.

**Decision:** Keep Activate / Deactivate at L3 for workload-level control. However, the chain is hierarchical — if L1 is Deactivated or Suspended, all L3s become inaccessible regardless of their individual state. L1 always overrides.

**UI behaviour:**
- L3 can be individually activated or deactivated by the customer
- When L1 is Deactivated or Suspended, all L3 cards must visually reflect the locked state (greyed out, locked icon) — individual L3 controls are disabled
- The customer should clearly understand that restoring access requires fixing L1 first, not individual L3s

---

### 3. Should L1 have its own dedicated view?

**Decision:** Yes — L1 gets a separate dedicated screen.

**Rationale:** L1 management is fundamentally different from L3 management. L1 is rare, high-stakes, and requires Four-Eyes approval. L3 is frequent, low-stakes, and self-service. Mixing them on one screen makes the kill switch feel too casual and buries the L1 configuration.

**Navigation structure:**
- **Key Chain view** — overview screen: shows L1 status card + L3 ServiceKey list. Clicking L1 navigates to the L1 detail screen.
- **L1 detail screen** — dedicated screen for: backend config (AWS KMS / Vault / HSM), key reference/ARN, rotation policy, Mode A→B upgrade, kill switch, grace period status.
- **L3 management** — stays inline in the Key Chain view (create, activate, deactivate, delete).

---

### 4. Which operations require Four-Eyes approval, and should Approvals be a separate view?

**Context:** This decision affects both the governance model and the navigation structure.

**L3 deactivation impact:** Deactivating an L3 ServiceKey makes an entire workload (e.g. MongoDB) inaccessible. This is intentional, scoped, and reversible — the customer can reactivate it themselves. The blast radius is limited to one workload, not the entire tenant.

**L1 revocation impact:** Revoking L1 makes all data for the entire tenant inaccessible. It is irreversible without a new approval cycle. The blast radius is total.

**Option A — Approvals only at L1 (recommendation):**
- L3 operations are self-service, no approval needed — reversible and scoped
- L1 operations always require Four-Eyes — irreversible and total
- In this case: Approvals view can be blended into the L1 detail screen, no separate navigation item needed
- Navigation: Key Chain / L1 Detail (includes approvals) / Access

**Option B — Approvals at both L1 and L3:**
- L3 deactivation also requires Four-Eyes — adds friction but prevents accidental outages
- In this case: Approvals must be a separate view to show both L1 and L3 pending requests
- Navigation: Key Chain / Approvals / Access

**Recommendation:** Option A. L3 deactivation is an operational action the customer should be able to do quickly without waiting for a second approver. The reversibility makes the risk acceptable. Reserve Four-Eyes for the irreversible, tenant-wide L1 operations only. This also simplifies the navigation — Approvals lives inside the L1 detail screen, not as a standalone section.

| Operation | Option A | Option B |
| :--- | :--- | :--- |
| L1 — declare / replace / kill switch / destroy | ✅ Approval required | ✅ Approval required |
| L3 — activate / deactivate / delete | ❌ Self-service | ✅ Approval required |
| Approvals as separate nav item | ❌ Blended into L1 view | ✅ Separate view |

---

## Kill Switch: Functional Requirements

> **⚠️ Proposal — Needs Team Discussion**
> The requirements below are a first proposal based on internal architecture and tech lead review. They have not been formally agreed. This section should be discussed and validated with the team before implementation begins.

---

### 1. Kill Switch Is a State Transition, Not a Delete

The kill switch is an explicit, approved state transition on the Tenant CRD:

```
Active → Deactivated
```

It is not a separate operation, not a deletion of the L1 key reference, and not an unlink. The L1 key reference remains on the Tenant CRD — it is the *state* that changes. The UI must reflect this — the kill switch is a governance action, not a destructive one.

---

### 2. Unlink Is Not a Supported Operation

Customers cannot simply remove or unlink their L1 key reference. There is no "remove key" button. The supported operations are:

- **Replace** — swap one L1 reference for another (requires Four-Eyes)
- **Suspend** — graceful degradation with grace period (see below)
- **Revoke (kill switch)** — immediate, approved, permanent transition to Deactivated

If a customer wants to stop using their key, they use the kill switch. If they want to switch keys, they replace the reference. Unlinking is not an option.

---

### 3. Kill Switch vs Suspension — The UI Must Make This Distinction Clear

These are two fundamentally different states and the UI must never confuse them:

| | Suspension | Kill Switch |
| :--- | :--- | :--- |
| **Trigger** | L1 key becomes unreachable or unavailable | Explicit approved revocation request |
| **How it happens** | Automatic — system detects the issue | Manual — customer initiates |
| **Encrypt** | ❌ Blocked | ❌ Blocked |
| **Decrypt** | ✅ Allowed | ❌ Blocked |
| **Grace period** | ✅ 30 / 60 / 90 days configurable | ❌ Bypassed entirely |
| **Reversible** | ✅ Re-link the key to restore Active | ❌ No — requires new approval cycle |
| **Four-Eyes** | ❌ Not required | ✅ Required |
| **Intent** | Accidental / external issue | Deliberate sovereignty action |

The customer must never mistake a suspension for a kill switch or vice versa. Visual language, copy, and colour must be distinct.

---

### 4. Kill Switch UI Flow

The UI flow for triggering the kill switch:

1. Customer clicks "Trigger Kill Switch" on the L1 detail screen
2. **Warning screen** — full-page confirmation, not a small dialog:
   - "This will make all data for this tenant permanently inaccessible"
   - "Decrypt will be blocked — running services will stop working"
   - "This action requires approval from a second authorised person"
   - "This cannot be undone without a new approval cycle"
   - Requires the customer to type a confirmation phrase (e.g. the tenant name)
3. Customer submits — creates a pending approval request
4. **Pending state shown in UI** — kill switch is proposed but not yet active
5. Second authorised person (not the proposer) approves
6. **Propagation window** — 2-minute SLA. UI shows a propagation status indicator: "Revocation in progress — propagating to all regions"
7. **Deactivated state** — L1 card turns red, all L3 cards locked, clear message: "Data access revoked. All encryption and decryption is blocked."

---

### 5. Can the Kill Switch Be Cancelled?

**Before approval:** Yes — the proposer or an admin can cancel the pending request. The key remains Active.

**After approval but during propagation:** No — once approved, revocation is in flight. It cannot be cancelled.

**After propagation completes:** No — requires a new approval cycle to restore (re-link or replace L1).

The UI must clearly show which cancellation window is still open.

---

### 6. Grace Period UI Requirements

When L1 enters Suspended state (key becomes unreachable):

- L1 card shows orange/amber state — clearly different from red (Deactivated)
- Grace period countdown shown prominently: "X days remaining to restore access"
- Encrypt blocked indicator on all L3 cards
- Decrypt still allowed — running services continue to work
- Notification history accessible from the L1 card
- Clear CTA: "Restore access" — re-links the key and returns to Active

When grace period expires with no action:
- Automatic transition to Deactivated
- Decrypt also blocked from this point
- UI shows: "Grace period expired. Data access is now blocked."

---

### 7. What the UI Shows During the 2-Minute Propagation Window

After kill switch is approved, before propagation completes:

- L1 card shows "Revocation propagating..." with a progress indicator
- L3 cards show a transitioning state — not yet locked but not fully active
- A banner: "Kill switch approved. Propagating to all regions — this may take up to 2 minutes."
- Once propagation is confirmed complete: full Deactivated state shown

---

## Open Questions for the Showroom Team

1. Which component library are we targeting — UI5 Web Components (React), or something else?
2. Is there a Platform Mesh MicroFrontend starter template we should use?
3. What is the Luigi context injection API we should rely on for tenant ID and user roles?
4. Are there existing Platform Mesh MicroFrontends we can use as reference implementations?


