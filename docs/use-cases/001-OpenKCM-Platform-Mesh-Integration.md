---
authors:
  - Aysan Mazloumi
---

## Persona
**Platform Users** - Anyone with access to an account in KCP who enables OpenKCM via Platform Mesh and accesses the OpenKCM CMK UI for management of their keychain (e.g., configuring CMK with HYOK or BYOK). These users interact exclusively with the CMK Layer of OpenKCM through the web interface, while the Crypto Service operates as an integrated backend service.

## Overview

As a Platform User, I need to enable OpenKCM encryption services for my account/tenant through Platform Mesh. This integration provisions the CMK Service with its web UI and automatically configures the backend Crypto Service. Platform Users work exclusively with Customer Managed Keys (CMK) through the CMK UI interface.

## User Stories

### Story 1: Enable OpenKCM for My Account
**As a** Platform User  
**I want to** enable OpenKCM encryption services for my account  
**So that** I can manage my customer encryption keys through the CMK UI  

**User Journey:**
1. I create a new account or tenant in KCP
2. Platform Mesh automatically detects the new tenant resource
3. The system triggers OpenKCM CMK Service provisioning
4. The backend Crypto Service is automatically configured and integrated
5. I receive notification that OpenKCM CMK UI is ready
6. I can now access the CMK web interface to configure my encryption keys

### Story 2: Configure My Customer-Managed Encryption
**As a** Platform User  
**I want to** set up my Customer Managed Keys through the CMK UI  
**So that** I have full sovereignty over my encryption keys  

**User Journey:**
1. I log into the OpenKCM CMK UI through my KCP account
2. I see the key configuration dashboard
3. I choose my preferred key management approach (BYOK or HYOK)
4. I configure my external key store (AWS KMS, Azure Key Vault, etc.)
5. The CMK Service coordinates with the backend Crypto Service
6. I receive confirmation that my customer-managed encryption is active
7. I can monitor and manage my keys through the CMK UI

### Story 3: Configure BYOK (Bring Your Own Key)
**As a** Platform User  
**I want to** use my existing encryption keys from my cloud provider  
**So that** I can maintain consistency with my existing security infrastructure  

**User Journey:**
1. I access the OpenKCM CMK UI configuration wizard
2. I select "Bring Your Own Key (BYOK)"
3. I provide my existing key ARN/URI from AWS KMS, Azure Key Vault, or GCP KMS
4. I configure the necessary permissions and trust relationships through the UI
5. The CMK Service validates my key configuration with the backend Crypto Service
6. My applications begin using my own keys for encryption (handled by Crypto Service)
7. I can monitor my key status and usage in the CMK UI dashboard

### Story 4: Set up HYOK (Hold Your Own Key)
**As a** Platform User with strict compliance requirements  
**I want to** keep my encryption keys in my own HSM or on-premises systems  
**So that** I maintain complete control and meet regulatory requirements  

**User Journey:**
1. I choose "Hold Your Own Key (HYOK)" in the CMK UI during setup
2. I configure connection to my HSM or on-premises key management system
3. I provide necessary certificates and connection details through the CMK interface
4. The CMK Service establishes secure communication with my key store via the Crypto Service
5. I verify through the CMK UI that keys never leave my controlled environment
6. My data is encrypted using keys that remain under my physical control
7. I can monitor the connection status and key health in the CMK dashboard

## Requirements

### Functional Requirements

#### For OpenKCM Service Enablement:
- **REQ-001**: System must automatically detect new tenant/account creation in KCP
- **REQ-002**: Platform Mesh must trigger OpenKCM CMK Service provisioning without user intervention
- **REQ-003**: Backend Crypto Service must be automatically configured and integrated
- **REQ-004**: User must receive clear notification of CMK UI availability
- **REQ-005**: OpenKCM CMK UI must become accessible once provisioning completes
- **REQ-006**: User must be able to access CMK configuration wizard through the web interface

#### For Customer-Managed Key Configuration:
- **REQ-007**: User must be able to configure CMK through the CMK UI without technical expertise
- **REQ-008**: CMK Service must coordinate seamlessly with backend Crypto Service
- **REQ-009**: System must validate external key store connectivity before activation
- **REQ-010**: Rollback capability must be available if configuration fails
- **REQ-011**: Key configuration changes must propagate to Crypto Service automatically
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
- **REQ-020**: CSEK to CMK migration must not cause application downtime
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
- **REQ-029**: Partial failures must not leave tenant in inconsistent state
- **REQ-030**: External key store outages must not prevent data access
- **REQ-031**: Configuration must be backed up and recoverable

## Acceptance Criteria

### Successful Provisioning:
- ✅ New tenant automatically triggers OpenKCM setup
- ✅ User receives email notification when setup is complete
- ✅ OpenKCM UI is accessible within 5 minutes of account creation
- ✅ Default CSEK encryption is active immediately
- ✅ Configuration wizard guides user through setup options

### Successful CSEK to CMK Migration:
- ✅ User can initiate migration through UI without technical knowledge
- ✅ All existing data is re-encrypted with new key hierarchy
- ✅ Applications continue working without modification during migration
- ✅ User receives confirmation when migration is complete
- ✅ Key management dashboard shows new CMK status

### Successful External Key Store Integration:
- ✅ User can connect to AWS KMS, Azure Key Vault, or GCP KMS
- ✅ System validates permissions and connectivity before activation
- ✅ BYOK setup completes with user's existing keys
- ✅ HYOK setup maintains keys in user's controlled environment
- ✅ Dashboard shows real-time key status and health

### Error Scenarios:
- ❌ If provisioning fails → Automatic retry with user notification
- ❌ If external key store is unreachable → Clear error with resolution steps
- ❌ If migration fails → Automatic rollback to previous state
- ❌ If permissions are insufficient → Detailed permission requirements shown

## Business Value

### For Platform Users:
- **Zero-Touch Setup**: Automatic provisioning when creating new accounts
- **Gradual Adoption**: Start with default encryption, upgrade when ready
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

### Platform Mesh Requirements:
- **Tenant Lifecycle Events**: Detection of account/tenant creation and deletion
- **Service Discovery**: Registration of OpenKCM services in mesh
- **Network Policies**: Secure communication between services
- **Resource Management**: CPU, memory, and storage provisioning

### External System Requirements:
- **Cloud KMS APIs**: Integration with AWS, Azure, GCP key management
- **HSM Connectivity**: PKCS#11 and proprietary HSM interfaces
- **Identity Integration**: SSO and authentication with existing systems
- **Monitoring Integration**: Health checks and alerting for key services