# ADR-501: Pluggable KMS Provider Interface

| Status | Date | Document Type |
| :--- | :--- | :--- |
| **Active** | 2026-01-17 | Architecture Design Record |

## Context
OpenKCM is designed to be **KMS-Agnostic**. Our core value proposition is that we can manage keys regardless of where the physical root of trust resides.
* **The Reality:** Every Cloud Provider (AWS, Azure, GCP) and HSM Vendor (Thales, Entrust) has a different API, authentication model, and error handling structure.
* **The Risk:** If we hardcode AWS KMS logic directly into the Crypto Core, we create vendor lock-in and make it impossible to deploy OpenKCM in "Sovereign Clouds" or On-Premise environments without massive code refactoring.

**The Requirement:** We need a unified abstraction layer that normalizes these disparate backends into a single, consistent interface for the OpenKCM application logic.

## Decision
We will implement the **Pluggable Keystore Interface (KSI)** as a strict Go Interface within the Crypto Core.



### Driver Implementation Strategy
We will implement specific drivers that satisfy this interface, loaded dynamically at runtime based on configuration.

1.  **AWS Driver:** Uses `aws-sdk-go-v2`. Maps `Encrypt` to `kms:Encrypt`. Authentication via IAM Roles for Service Accounts (IRSA).
2.  **Azure Driver:** Uses `azidentity` and Key Vault SDK. Maps `Encrypt` to `keyvault.Encrypt`. Authentication via Workload Identity.
3.  **HashiCorp Vault Driver:** Uses Transit Engine. Maps `Encrypt` to `/transit/encrypt`. Authentication via Kubernetes Auth Method.
4.  **PKCS#11 Driver:** (For HSMs). Uses low-level CGO bindings to talk to hardware.

### Error Normalization
The KSI layer acts as an "Error Translator."
* AWS might return `ThrottlingException`.
* Azure might return `429 Too Many Requests`.
* **KSI Result:** Both are translated to `openkcm.ErrRateLimitExceeded`, allowing the upper layers to implement a unified retry strategy (Exponential Backoff) without knowing the underlying provider.

## Consequences

### Positive (Pros)
* **Portability:** We can move a tenant from AWS to Azure by simply updating a config string (and migrating the key material), without changing application code.
* **Testability:** We can write a `MockKSI` driver for unit tests that simulates key operations in memory, making CI/CD fast and cost-free.
* **Future-Proof:** Adding support for "Oracle Cloud" or "Post-Quantum HSMs" only requires writing a new driver, not rewriting the core logic.

### Negative (Cons)
* **Lowest Common Denominator:** We can only expose features supported by *all* providers. (e.g., If AWS supports "Key Rotation" but a basic HSM doesn't, we have to build a software shim or disable that feature for HSM users).
* **Debugging Complexity:** An error in the abstraction layer can mask the true root cause from the underlying vendor driver.

## References
* [ACD-401: Pluggable Keystore Interface (KSI) Framework](../acd/pluggable-keystore-interface-ksi-framework.md)
* [AWS KMS API Reference](https://docs.aws.amazon.com/kms/latest/APIReference/)
* [Azure Key Vault REST API](https://learn.microsoft.com/en-us/rest/api/keyvault/)