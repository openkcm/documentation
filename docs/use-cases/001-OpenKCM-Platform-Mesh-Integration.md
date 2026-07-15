---
authors:
  - Aysan
last_updated: 2026-07-15
---

## Persona
**Platform Users** - Anyone with access to an account in a platform who enables OpenKCM and uses it to manage their customer encryption keys (e.g., configuring HYOK or BYOK). These users interact with OpenKCM to configure their key hierarchy, while Krypton operates as the integrated backend that executes all crypto operations.

## Overview

As a Platform User, I need to enable OpenKCM for my account through a platform. This integration provisions OpenKCM and automatically configures Krypton as the backend. Platform Users configure and manage their Customer Managed Keys (CMK) through OpenKCM.

## User Stories

### Story 1: Enable OpenKCM for My Account
**As a** Platform User  
**I want to** enable OpenKCM for my account  
**So that** I can manage my customer encryption keys  

**User Journey:**
1. I create a new account in my platform
2. The platform automatically detects the new account and provisions OpenKCM
3. Krypton is automatically configured and integrated as the crypto backend
4. I receive notification that OpenKCM is ready
5. I can now configure my encryption keys through OpenKCM

### Story 2: Configure My Customer-Managed Encryption
**As a** Platform User  
**I want to** set up my Customer Managed Keys in OpenKCM  
**So that** I have full sovereignty over my encryption keys  

**User Journey:**
1. I log into OpenKCM through my account
2. I see the key configuration dashboard
3. I choose my preferred key management approach (BYOK or HYOK)
4. I configure my external key store (AWS KMS, Azure Key Vault, etc.)
5. OpenKCM validates the configuration with Krypton
6. I receive confirmation that my customer-managed encryption is active
7. I can monitor and manage my keys through OpenKCM

### Story 3: Configure BYOK (Bring Your Own Key)
**As a** Platform User  
**I want to** use my existing keys from my cloud provider  
**So that** I can maintain consistency with my existing security infrastructure  

**User Journey:**
1. I open the key configuration in OpenKCM
2. I select "Bring Your Own Key (BYOK)"
3. I provide my existing key ARN/URI from AWS KMS, Azure Key Vault, or GCP KMS
4. I configure the necessary permissions and trust relationships
5. OpenKCM validates my key configuration through Krypton
6. My applications begin using my own keys for encryption (handled by Krypton)
7. I can monitor my key status and usage in OpenKCM

### Story 4: Set up HYOK (Hold Your Own Key)
**As a** Platform User with strict compliance requirements  
**I want to** keep my encryption keys in my own HSM or on-premises systems  
**So that** I maintain complete control and meet regulatory requirements  

**User Journey:**
1. I choose "Hold Your Own Key (HYOK)" in OpenKCM during setup
2. I configure connection to my HSM or on-premises key management system
3. I provide necessary certificates and connection details
4. Krypton establishes secure communication with my key store
5. I verify in OpenKCM that keys never leave my controlled environment
6. My data is encrypted using keys that remain under my physical control
7. I can monitor the connection status and key health in OpenKCM

## Requirements

### Functional Requirements

#### For OpenKCM Service Enablement:
- **REQ-001**: System must automatically detect new account creation in KCP
- **REQ-002**: The platform must trigger OpenKCM provisioning without user intervention
- **REQ-003**: Krypton must be automatically configured and integrated
- **REQ-004**: User must receive clear notification when OpenKCM is ready
- **REQ-005**: OpenKCM must become accessible once provisioning completes
- **REQ-006**: User must be able to access key configuration through OpenKCM

#### For Customer-Managed Key Configuration:
- **REQ-007**: User must be able to configure CMK through OpenKCM without technical expertise
- **REQ-008**: OpenKCM must coordinate seamlessly with Krypton
- **REQ-009**: System must validate external key store connectivity before activation
- **REQ-010**: Rollback capability must be available if configuration fails
- **REQ-011**: Key configuration changes must propagate to Krypton automatically
- **REQ-012**: Configuration status must be visible to the user throughout the process

#### For External Key Store Integration:
- **REQ-013**: Support for AWS KMS, Azure Key Vault, GCP KMS integration
- **REQ-014**: Support for HSM and on-premises key management systems
- **REQ-015**: Secure credential management for external system access
- **REQ-016**: Real-time validation of key accessibility and permissions
- **REQ-017**: Automated trust relationship configuration where possible
- **REQ-018**: Clear error messages for configuration issues

### Non-Functional Requirements

#### Performance:
- **REQ-019**: Provisioning must complete within 5 minutes for standard setup
- **REQ-020**: Key configuration changes must not cause application downtime
- **REQ-021**: External key store validation must complete within 30 seconds
- **REQ-022**: Configuration changes must propagate to all regions within 2 minutes

#### Security:
- **REQ-023**: All external key store credentials must be encrypted at rest
- **REQ-024**: Integration must use least-privilege access principles
- **REQ-025**: All configuration actions must be logged for audit
- **REQ-026**: Multi-factor authentication required for HYOK setup
- **REQ-027**: Network connections to external systems must be encrypted

#### Reliability:
- **REQ-028**: Provisioning failures must be automatically retried
- **REQ-029**: Partial failures must not leave account in inconsistent state
- **REQ-030**: External key store outages must not prevent data access
- **REQ-031**: Configuration must be backed up and recoverable

## Acceptance Criteria

### Successful Provisioning:
- ✅ New account automatically triggers OpenKCM setup
- ✅ User receives notification when setup is complete
- ✅ OpenKCM is accessible within 5 minutes of account creation
- ✅ Configuration wizard guides user through setup options

### Successful External Key Store Integration:
- ✅ User can connect to AWS KMS, Azure Key Vault, or GCP KMS
- ✅ System validates permissions and connectivity before activation
- ✅ BYOK setup completes with user's existing keys
- ✅ HYOK setup maintains keys in user's controlled environment
- ✅ Dashboard shows real-time key status and health

### Error Scenarios:
- ❌ If provisioning fails → Automatic retry with user notification
- ❌ If external key store is unreachable → Clear error with resolution steps
- ❌ If configuration fails → Automatic rollback to previous state
- ❌ If permissions are insufficient → Detailed permission requirements shown

## Business Value

### For Platform Users:
- **Zero-Touch Setup**: Automatic provisioning when creating new accounts
- **Compliance Ready**: Easy path to meet regulatory requirements
- **Vendor Flexibility**: Use any supported key management system
- **Data Sovereignty**: Full control over encryption keys and data access

### For the Organization:
- **Reduced Onboarding Friction**: Automated setup increases adoption
- **Security by Default**: All data encrypted from day one
- **Compliance Enablement**: Easy path for customers to meet regulations
- **Competitive Advantage**: Superior encryption options vs. competitors
- **Customer Retention**: Strong security builds trust and reduces churn

## Integration Dependencies

### Platform Requirements:
- **Account Lifecycle Events**: Detection of account creation and deletion
- **Service Discovery**: Registration of OpenKCM services in mesh
- **Network Policies**: Secure communication between services
- **Resource Management**: CPU, memory, and storage provisioning

### External System Requirements:
- **Cloud KMS APIs**: Integration with AWS, Azure, GCP key management
- **HSM Connectivity**: PKCS#11 and proprietary HSM interfaces
- **Identity Integration**: SSO and authentication with existing systems
- **Monitoring Integration**: Health checks and alerting for key services
