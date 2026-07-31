---
authors:
  - Aysan Mazloumi
last_updated: 2026-07-31
status: Proposal — incomplete, pending alignment
---

## Persona
**Platform User (Security Admin)** — A user with account-level access on Platform Mesh who enables OpenKCM and registers the organisation's root key. They are responsible for connecting OpenKCM to their external keystore and managing the L1 key lifecycle. Sensitive L1 operations require approval from a second authorised person — the security admin can configure which operations require approval and who the approvers are.

## Overview

As a Platform User, I need to enable OpenKCM for my account and register my root key so that all encryption in my account is governed by a key I control. OpenKCM is available as a Managed Service Provider (MSP) on the Platform Mesh marketplace. Once enabled, OpenKCM provisions the necessary components and connects to my external keystore. I manage my root key (L1).

## User Stories

### Story 1: Enable OpenKCM for My Account
**As a** Platform User  
**I want to** enable OpenKCM for my account  
**So that** I can manage my customer encryption keys  

**User Journey:**
1. I find OpenKCM in the Platform Mesh marketplace
2. I enable OpenKCM for my account
3. Krypton is deployed and ready to handle encryption operations
4. The OpenKCM UI is accessible for key configuration

### Story 2: Register My Root Key (BYOK — Bring Your Own Key)
**As a** Platform User  
**I want to** connect my existing key from my cloud provider to OpenKCM  
**So that** all encryption in my account uses a key I own and control  

**User Journey:**
1. I open the OpenKCM UI
2. I select "Bring Your Own Key (BYOK)"
3. I provide my key ARN/URI from AWS KMS, Azure Key Vault, or GCP KMS and the required access credentials
4. I confirm the registration — OpenKCM shows the key on my dashboard and confirms successful registration or returns an error if the keystore could not be reached
5. In the OpenKCM UI I can see the L1 key status (active/inactive)

### Story 3: Register My Root Key (HYOK — Hold Your Own Key)
**As a** Platform User with strict compliance requirements  
**I want to** connect my on-premises HSM or key management system to OpenKCM  
**So that** my key material never leaves my own infrastructure  

**User Journey:**
1. I open the OpenKCM UI
2. I select "Hold Your Own Key (HYOK)"
3. I provide the connection details and certificates for my HSM or on-premises KMS
4. I confirm the registration — OpenKCM shows the key on my dashboard and confirms successful registration or returns an error if the keystore could not be reached
5. In the OpenKCM UI I can see the L1 key status (active/inactive)

### Story 4: Bind the Root Key to a Namespace or System
**As a** Platform User (Security Admin)  
**I want to** bind my registered root key to a namespace or system  
**So that** encryption for that namespace or system is governed by my key  

**User Journey:**
1. I open the OpenKCM UI
2. I see my registered L1 key and the namespaces or systems in my account that are not yet bound
3. I select a namespace or system and assign my L1 key to it
4. The binding enters a pending state if a second approver is required (see Story 5)
5. Once approved, the namespace or system is active — all services in it encrypt under my key
6. The binding is visible in the OpenKCM UI as part of the key hierarchy overview

### Story 5: Configure Approval Workflow for L1 Key Operations
**As a** Platform User (Security Admin)  
**I want to** configure which L1 key operations require a second approver  
**So that** no single person can unilaterally rotate, revoke, or trigger the kill switch on the root key  

**User Journey:**
1. I open the OpenKCM UI and go to approval settings
2. I select which operations require a second approval — e.g. L1 key revocation, kill switch, key rotation, key binding
3. I assign one or more users as approvers for these operations — the initiator cannot approve their own request
4. When I or another user initiates a sensitive L1 operation, it enters a pending state
5. The assigned approver receives a notification and reviews the request
6. The operation only executes after the approver confirms
7. Both the initiation and the approval are recorded in the audit log

## Requirements

### Functional Requirements

- **REQ-001**: User must be able to enable OpenKCM for their account from the Platform Mesh marketplace
- **REQ-002**: Once enabled, OpenKCM must be provisioned and the OpenKCM UI must be accessible without further user intervention
- **REQ-003**: Krypton must be deployed and ready to handle encryption operations as part of provisioning
- **REQ-004**: User must be able to register an L1 key from AWS KMS, Azure Key Vault, GCP KMS, OpenBao, or an HSM via PKCS#11
- **REQ-005**: Before activating the key, OpenKCM must perform a test call to the external keystore to confirm it is reachable and the credentials are valid
- **REQ-006**: The L1 key status (active/inactive) and details of L1 key.
- **REQ-007**: All key registration and binding actions must be logged for audit
- **REQ-008**: All connections between Krypton and the external keystore must be encrypted using TLS
- **REQ-009**: The L1 key reference and keystore connection details must be recoverable if OpenKCM is restarted or redeployed
- **REQ-010**: The security admin must be able to bind a registered L1 key to a namespace or system from the OpenKCM UI on account level.
- **REQ-011**: The security admin must be able to configure which L1 operations (registration, binding, rotation, revocation, kill switch) require approval from a second authorised user before execution
- **REQ-012**: The security admin must be able to assign one or more users as approvers for L1 operations
- **REQ-013**: A sensitive L1 operation must not execute until the required approval is given — the initiator cannot be their own approver — it must remain in a pending state until a different authorised user approves
- **REQ-014**: Both the initiation and the approval of any L1 operation must be recorded in the audit log with user identity and timestamp

### Non-Functional Requirements

- **REQ-015**: OpenKCM provisioning must complete within 5 minutes of being enabled
- **REQ-016**: The test call to the external keystore must complete within 30 seconds
- **REQ-017**: Key registration must not cause downtime for any running services

## Acceptance Criteria

### Successful Provisioning:
- ✅ User enables OpenKCM from the marketplace and the OpenKCM UI becomes accessible
- ✅ Krypton is running and able to serve key operations

### Successful Key Registration:
- ✅ User can register an L1 key from any supported keystore (AWS KMS, Azure Key Vault, GCP KMS, OpenBao, HSM)
- ✅ OpenKCM confirms keystore reachability before activating the key
- ✅ L1 key status is visible in the OpenKCM UI
- ✅ All registration actions appear in the audit log

### Successful Key Binding:
- ✅ Security admin can bind the L1 key to a namespace or system from the OpenKCM UI
- ✅ Binding requires approval if configured
- ✅ Binding is visible in the key hierarchy overview

### Error Scenarios:
- ❌ If provisioning fails → user receives an error with the reason
- ❌ If the external keystore is unreachable during the test call → clear error shown, key not activated
- ❌ If credentials are invalid → clear error with description of what is missing
