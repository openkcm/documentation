# ACD-105: Sovereign Audit & Immutable Logging

| Status | Date | Document Type |
| :--- | :--- | :--- |
| **Active** | 2026-01-31 | Architecture Concept Design |

## Overview
**Sovereign Audit** is the mechanism by which OpenKCM converts "Security Operations" into "Business Trust." In a Sovereign Cloud environment, it is not enough for a provider to claim they cannot access customer data; they must prove it.

This document defines the **Unified Audit Pipeline**, a strategic capability designed to capture, seal, and deliver immutable proof of every system interaction. It serves two critical business functions:
1.  **Operational Transparency:** Providing the platform provider with the visibility needed to maintain service health.
2.  **Sovereign Verification:** Giving the customer the "Cryptographic Proof" required to satisfy regulators (GDPR, TISAX) that their data isolation contract is being honored.



## The "Glass House" Philosophy
OpenKCM operates on a principle of radical transparency to eliminate the "Trust Gap" inherent in SaaS platforms.

* **Total Capture:** Every significant business event—from onboarding a new tenant to rotating a cryptographic key—is recorded.
* **Tamper-Evidence:** Logs are digitally signed at the source. Any attempt to alter a log entry to cover tracks invalidates the signature, providing immediate forensic evidence of tampering.
* **Global Traceability:** Every action is tagged with a unique "Correlation ID," allowing a customer to trace a specific user click in the Portal all the way down to the specific millisecond a key was accessed in the vault.

## The Business Data Model (The Audit Schema)
To ensure that audit logs are useful for compliance and forensics, OpenKCM standardizes the "Who, What, Where, and When" of every event. This structured approach allows logs to be instantly ingested by enterprise tools without complex translation.

| Business Context | Description | Value to Customer |
| :--- | :--- | :--- |
| **The Actor** | The User or Service Identity. | Answers "Who performed this action?" (e.g., `admin@tenant-a`). |
| **The Action** | The specific operation. | Answers "What happened?" (e.g., `Key Rotation`, `User Login`, `Workflow Approval`). |
| **The Resource** | The target object. | Answers "What was affected?" (e.g., `Production-Database-Key`). |
| **The Context** | The Tenant ID and Trace ID. | Ensures the log is routed strictly to the correct customer's sovereign silo. |
| **The Outcome** | Success or Failure (with Reason). | Provides forensic details on *why* an access attempt was denied (e.g., `MFA Failed`). |

## The Pipeline Strategy: Vendor Neutrality (OTEL)
OpenKCM builds its audit pipeline on the **OpenTelemetry (OTEL)** industry standard. This is a strategic decision to avoid "Vendor Lock-In."

Instead of forcing customers to use a proprietary logging tool, the platform supports a **"Bring Your Own Observability"** model:
1.  **Universal Compatibility:** Because the platform speaks the global standard language of observability, it can connect to *any* enterprise SIEM (Splunk, Datadog, Sentinel, Dynatrace) out of the box.
2.  **Flexible Routing:** The system can simultaneously send operational metrics to the provider (for health checks) while routing sensitive security logs exclusively to the customer's private storage.

## Sovereign Routing & Storage
The architecture respects the physical boundaries of the customer's data.

* **Platform Operations:** The provider receives only anonymized health data required to keep the lights on.
* **Sovereign Storage:** Sensitive audit logs containing user identities and key usage patterns are routed directly to the **Tenant's Private Storage** (Sovereign Bucket or Private Schema).
* **Compliance Archiving:** The system supports "Write Once, Read Many" (WORM) storage, ensuring that audit trails satisfy legal retention requirements (e.g., 7 years for banking logs).

## Sovereign Verification (The "Trust but Verify" Model)
This architecture allows customers to independently audit the platform provider.

Because every key operation in OpenKCM eventually triggers a call to the customer's own cloud infrastructure (e.g., AWS KMS or Azure Key Vault), the customer can perform a **"Double-Entry Bookkeeping"** check:
* **The OpenKCM Log says:** "We accessed your key at 12:00:01."
* **The Cloud Provider Log says:** "OpenKCM accessed your key at 12:00:01."

If these records match, the customer has mathematical proof that the system is behaving correctly. If they diverge, the customer has immediate proof of unauthorized activity.

## Summary
ACD-105 transforms logging from a technical overhead into a **Strategic Trust Product**. By delivering signed, standardized, and customer-owned audit trails, OpenKCM allows enterprises to move sensitive workloads to the cloud with the confidence that they retain absolute oversight over every access event.