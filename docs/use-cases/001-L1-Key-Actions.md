---
authors:
  - Aysan Mazloumi
---

## Persona
**Platform Users** - Anyone with access to an account in KCP who enables OpenKCM via Platform Mesh and accesses the OpenKCM UI for management of the keychain (e.g., switching from CSEK to CMK with HYOK or BYOK). These users interact with the CMK Layer of OpenKCM to configure and manage their customer-managed encryption keys.

## Overview

As a Platform User, I need to control my encryption keys to maintain data security and compliance. L1 key actions allow me to manage the lifecycle of my Customer Master Keys through the OpenKCM UI.

## User Stories

### Story 1: Enable My Encryption Key
**As a** Platform User  
**I want to** enable my L1 encryption key  
**So that** my applications can both encrypt new data and decrypt existing data  

**User Journey:**
1. I log into the OpenKCM UI through my KCP account
2. I navigate to my key management dashboard
3. I select my L1 key and click "Enable"
4. I receive confirmation that encryption and decryption operations are now active
5. My applications can now process data normally

### Story 2: Disable My Encryption Key
**As a** Platform User  
**I want to** disable my L1 encryption key  
**So that** I can prevent new data encryption while still accessing existing data  

**User Journey:**
1. I access the OpenKCM UI and go to key management
2. I select my active L1 key and click "Disable"
3. I confirm the action understanding it will stop new encryption
4. I receive confirmation that the key is now in read-only mode
5. My applications can decrypt existing data but cannot encrypt new data

## Requirements

### Functional Requirements

#### For Key Enable Action:
- **REQ-001**: Platform User must be authenticated through KCP account
- **REQ-002**: User must have permission to manage keys for their tenant
- **REQ-003**: System must validate that the L1 key exists and is accessible
- **REQ-004**: Enable action must propagate to all OpenKCM Crypto regions
- **REQ-005**: User must receive clear confirmation of successful enablement
- **REQ-006**: Both encryption and decryption operations must become available

#### For Key Disable Action:
- **REQ-007**: User must be able to disable an active key through the UI
- **REQ-008**: System must confirm the disable action with the user
- **REQ-009**: Disable must immediately prevent new encryption operations
- **REQ-010**: Existing decryption operations must continue to work
- **REQ-011**: User must be notified of the read-only state
- **REQ-012**: Applications must receive appropriate error messages for encryption attempts

### Non-Functional Requirements

#### Usability:
- **REQ-013**: Key actions must be completed within 30 seconds
- **REQ-014**: UI must provide clear status indicators for key states
- **REQ-015**: Error messages must be user-friendly and actionable
- **REQ-016**: Actions must be reversible (enable after disable)

#### Security:
- **REQ-017**: All key actions must be logged for audit purposes  
- **REQ-018**: User permissions must be validated before each action
- **REQ-019**: Key states must be consistent across all regions
- **REQ-020**: No sensitive key material should be displayed in the UI

#### Reliability:
- **REQ-021**: System must handle partial failures gracefully
- **REQ-022**: Failed actions must be retryable by the user
- **REQ-023**: Key state changes must be atomic (all-or-nothing)
- **REQ-024**: System must remain responsive during key operations

## Acceptance Criteria

### Enable Key Success Scenario:
- ✅ User can see key status change from "Disabled" to "Enabled"
- ✅ Applications can successfully encrypt new data
- ✅ Applications can decrypt existing data
- ✅ Action completes within 30 seconds
- ✅ Audit log records the enable action

### Disable Key Success Scenario:
- ✅ User can see key status change from "Enabled" to "Disabled"  
- ✅ New encryption attempts return clear error messages
- ✅ Existing data decryption continues to work
- ✅ User receives confirmation of read-only mode
- ✅ Action completes within 30 seconds

### Error Scenarios:
- ❌ If external keystore is unavailable → User sees retry option
- ❌ If user lacks permissions → Clear permission denied message
- ❌ If network issues occur → Automatic retry with user notification
- ❌ If key doesn't exist → Helpful error with next steps

## Business Value

### For Platform Users:
- **Data Control**: Immediate ability to control data encryption access
- **Compliance**: Meet regulatory requirements for data access management  
- **Security**: Prevent unauthorized encryption while maintaining data access
- **Operational Flexibility**: Manage encryption during maintenance or incidents

### For the Organization:
- **Customer Trust**: Users have direct control over their encryption keys
- **Regulatory Compliance**: Supports GDPR, DORA, and other data protection laws
- **Risk Management**: Ability to quickly restrict encryption in security incidents
- **Operational Efficiency**: Self-service key management reduces support overhead
