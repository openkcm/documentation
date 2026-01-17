# ACD-105: Sovereign Audit & Immutable Logging

| Status | Date | Document Type |
| :--- | :--- | :--- |
| **Active** | 2026-01-16 | Architecture Concept Design |

## Overview
**Sovereign Audit** is the mechanism by which OpenKCM proves its trustworthiness. In a cryptographic system where the provider claims "we cannot see your data," the only verification is an immutable, verifiable audit trail that links every operation back to an authorized identity.

This document defines the **Unified Audit Pipeline**, a system designed to capture, sign, and deliver high-fidelity security logs. It serves two masters:
1.  **The Platform Provider:** Who needs operational visibility and forensic capabilities.
2.  **The Sovereign Customer:** Who demands "Cryptographic Proof of Access" to verify that their L1 keys are only being used as contractually agreed.

## The "Glass House" Philosophy
OpenKCM operates on a principle of radical transparency.
* **Every Event is Logged:** From a high-level API call in the Portal to a low-level KMIP unwrap operation in the Crypto Core.
* **Every Log is Signed:** Logs are hashed and signed at the source (Regional Node) to prevent tampering during transit.
* **Every Access is Traceable:** A global `Correlation_ID` links the SaaS user's click to the specific millisecond the L1 Root Key was accessed in AWS KMS.

## The Audit Event Schema
To ensure interoperability with SIEMs (Splunk, Datadog, Sentinel), OpenKCM standardizes on a strict JSON schema.

| Field | Description | Example |
| :--- | :--- | :--- |
| `timestamp` | UTC Execution Time (Microsecond precision) | `2026-01-16T14:05:01.123456Z` |
| `correlation_id` | Global Trace ID | `req-abc-123-xyz` |
| `actor` | The Identity (User or Service) | `user:admin@tenant-a.com` or `spiffe://core/region-eu-1` |
| `action` | The Operation Performed | `kms:Decrypt`, `policy:Update` |
| `resource` | The Target Object | `key:L2-Tenant-A-v4` |
| `status` | Outcome (Success/Failure/Deny) | `ALLOW` |
| `signature` | Cryptographic seal of the log entry | `sha256:8f43...` |



## The Pipeline Architecture
The audit system is built on an asynchronous, reliable delivery model.

1.  **Generation (The Source):**
    * The **Crypto Core** generates a log entry for an L2 unwrap.
    * It appends the `Tenant_ID` and signs the payload with its local **Node Identity Key**.
2.  **Collection (The Orbital Agent):**
    * The local **Orbital Agent** buffers these logs to disk (handling network backpressure).
    * It batches and pushes them to the central **Governance Ingest**.
3.  **Validation & Routing (The Hub):**
    * The central ingestion service verifies the signature.
    * It routes the log to two destinations:
        * **Hot Storage:** OpenSearch / Postgres for immediate platform ops visibility.
        * **Cold Storage:** Immutable S3 Buckets (WORM compliant) for long-term retention.
4.  **Customer Export (The Sovereign Link):**
    * A dedicated "Log Streamer" filters events by `Tenant_ID`.
    * It pushes these filtered, signed logs directly to the customer’s own SIEM or S3 bucket (Bring Your Own Bucket).



## Sovereign Verification
Customers do not have to trust OpenKCM's logs blindly. Because every L2 unwrap requires a call to *their* external KMS (AWS/Azure), they can correlate:
* **OpenKCM Log:** "We accessed L1 key `arn:aws:kms...` at 14:05:01."
* **AWS CloudTrail:** "Principal `OpenKCM_Role` accessed key `arn:aws:kms...` at 14:05:01."

If these timestamps and correlation IDs match, the customer has **Cryptographic Proof** that the platform is operating correctly. If they diverge, the customer has evidence of tampering or unauthorized access.

## Summary
ACD-105 transforms logging from a debugging tool into a **Trust Product**. By providing signed, tamper-evident, and customer-exportable audit trails, OpenKCM allows enterprises to "Trust but Verify" the security of their sovereign data.