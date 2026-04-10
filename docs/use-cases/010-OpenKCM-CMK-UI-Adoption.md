---
authors:
  - Aysan
---

## Persona

**Platform UI Development Team & OpenKCM Team** — The teams responsible for integrating OpenKCM's Customer Managed Key (CMK) UI into the Platform Mesh user interface. The Platform Mesh team provides the shell, navigation, and microfrontend infrastructure; the OpenKCM team owns the CMK UI application (UI5-based).

## Overview

The CMK UI must be integrated into Platform Mesh so that customers see key management as part of a unified platform experience — not as a separate application. Due to technical constraints (UI5 framework, authentication context propagation, theme alignment), this integration follows a **three-stage approach**, each stage building on the previous one.

The CMK UI is currently built on **SAP UI5** and has been adjusted to align with the **Fiori Launchpad** design. Since Platform Mesh uses the **Luigi microfrontend framework**, a direct native integration is not possible out of the box — it requires planned effort at each stage.

## Integration Stages

### Stage 1: Self-Hosted CMK UI with Cross-Link (Accepted)

**Status:** Accepted — can proceed

The CMK UI is deployed and hosted independently. Platform Mesh provides a navigation entry (button or menu item) that **opens the CMK UI in a new browser window**. The user authenticates separately via Keycloak.

**What this delivers:**

- CMK UI is accessible from the platform
- No dependency on iframe or microfrontend integration
- Suitable for showroom and initial customer demos

**Limitations:**

- Separate browser window — user leaves the Platform Mesh context
- Separate authentication — security context is not propagated
- No unified look-and-feel

**Dependencies:**

- CMK UI deployed and reachable (currently deployed; auth fixes in progress)
- Platform Mesh navigation entry pointing to CMK UI URL

---

### Stage 2: iframe Integration within Platform Mesh

**Status:** Requires implementation

Platform Mesh embeds the CMK UI inside an **iframe** within its own page layout. The user navigates within Platform Mesh; clicking "CMK" loads the CMK UI inside the iframe. From the user's perspective, it looks like a single application.

**What this delivers:**

- Single-window experience — no pop-ups
- Platform Mesh controls the navigation shell
- Closer to a unified customer experience

**Requires:**

- **Security context propagation** — Platform Mesh must pass authentication tokens to the CMK UI iframe (SSO integration)
- **Theme alignment** — CMK UI is currently blue; Platform Mesh is white. The CMK UI theme must be adjusted to match Platform Mesh's visual identity
- **CORS and CSP configuration** — iframe embedding requires proper cross-origin policies

**Limitations:**

- Still not fully native — iframe has inherent limitations (scroll, resize, deep linking)
- Two separate applications running in one window

---

### Stage 3: Native Luigi Microfrontend Integration

**Status:** Requires significant effort — not yet planned in detail

The CMK UI is fully integrated as a **Luigi microfrontend** within Platform Mesh. The Luigi framework handles routing, navigation, and context. The CMK UI renders natively inside the Platform Mesh layout — same look-and-feel, same theme, same navigation hierarchy.

**What this delivers:**

- Fully unified user experience — indistinguishable from other Platform Mesh pages
- Single authentication flow
- Consistent theme, navigation, and notification system
- Deep linking and state management across platform

**Requires:**

- CMK UI adapted to Luigi microfrontend API
- Shared design tokens / theme system between CMK UI and Platform Mesh
- Notification system integration
- Potentially significant UI refactoring depending on UIX improvements (see open questions below)

---

## Current Deployment Status

| Component                            | Status                                                                                                                                                                                      |
| ------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| CMK UI                               | Deployed and running                                                                                                                                                                        |
| Keycloak login page                  | Working — login page renders                                                                                                                                                                |
| IDP / Authentication                 | **In progress** — auth back-and-forth broken; originally tested with mTLS in CT box, but client secrets against public IDPs were never tested; Chris, Philip, and Christian are fixing this |
| CMK backend (gRPC API)               | Running                                                                                                                                                                                     |
| Registry service (tenant management) | Running — used for L2 key registration and tenant onboarding                                                                                                                                |

## Tenant Onboarding & Controller

The **Platform Mesh team** has offered to help prototype the integration:

- They will create the **controller** and **microfrontend** for Platform Mesh
- They will handle **tenant onboarding** by calling the **gRPC API of the registry service** to create tenants
- This is a **prototype** — after handover, ownership transfers to the OpenKCM team to continue, improve, and maintain

The registry service remains in place for tenant management, L2 key registration, and related operations. It is part of the CMK layer and is accessed via gRPC.

## Open Questions & Risks

### UIX Improvements (Peter / Matthias)

Peter is acting as requirement engineer with Matthias and driving UIX changes to the CMK UI:

| Change                                                                       | Status                            |
| ---------------------------------------------------------------------------- | --------------------------------- |
| Key configuration tiles redesign                                             | In discussion                     |
| Notification mechanism change                                                | Pushed back — needs clarification |
| SAP-specific elements (SAP Help, SAP Fiori Launchpad, potentially SAP Jewel) | Requested — not yet confirmed     |

**Critical constraint:** SAP-specific elements **must not** enter the OpenKCM repository. The UI must remain **common for both** SAP internal use and OpenKCM open-source. SAP-specific features must be layered separately (e.g., at build time or via configuration), not baked into the core CMK UI.

**Key question:** Will these UIX improvements break the previous conclusion that CMK UI (UI5) can be integrated into Platform Mesh via Luigi? The current expectation is "non-breaking — mostly realignment," but **this is not yet confirmed** because the speaker has not been involved in those decisions yet.

**Action required:**

- Align with Peter on whether UIX changes affect Luigi compatibility
- Ensure SAP specifics are layered, not merged into the OpenKCM codebase
- A workshop/meeting in late April (Walldorf) should be used to clarify this face-to-face

### Authentication Integration

- Stage 1 works with separate Keycloak login
- Stage 2+ requires security context propagation from Platform Mesh to CMK UI
- The session manager was tested with mTLS in CT box but **not with client secrets against public IDPs** — this needs to be resolved before Stage 2

## Adoption Requirements (by Stage)

| Requirement                        | Stage 1    | Stage 2    | Stage 3    |
| ---------------------------------- | ---------- | ---------- | ---------- |
| CMK UI deployed and reachable      | Required   | Required   | Required   |
| Platform Mesh navigation entry     | Required   | Required   | Required   |
| Security context propagation (SSO) | Not needed | Required   | Required   |
| Theme alignment (blue → white)     | Not needed | Required   | Required   |
| iframe embedding (CORS/CSP)        | Not needed | Required   | Not needed |
| Luigi microfrontend adaptation     | Not needed | Not needed | Required   |
| Shared notification system         | Not needed | Not needed | Required   |
| SAP-specific elements layered out  | Required   | Required   | Required   |

## Success Criteria

### Stage 1 Success

- ✅ CMK UI accessible from Platform Mesh via navigation link
- ✅ User can authenticate via Keycloak and manage keys
- ✅ Suitable for showroom demos

### Stage 2 Success

- ✅ CMK UI renders inside Platform Mesh (iframe) — no separate window
- ✅ SSO tokens propagated — user logs in once
- ✅ Theme visually aligned with Platform Mesh

### Stage 3 Success

- ✅ CMK UI fully integrated as Luigi microfrontend
- ✅ Navigation, theme, notifications, and deep linking unified
- ✅ Indistinguishable from native Platform Mesh pages

## Business Value

- **Seamless User Experience**: Unified interface reduces user friction and training needs
- **Operational Efficiency**: Integrated workflows improve productivity
- **Consistent Brand Experience**: Maintains platform's professional appearance
- **Reduced Development Overhead**: Shared components and patterns reduce maintenance
- **Improved Feature Adoption**: Familiar interface encourages key management usage
- **Future-Ready Architecture**: Established integration patterns support future services
