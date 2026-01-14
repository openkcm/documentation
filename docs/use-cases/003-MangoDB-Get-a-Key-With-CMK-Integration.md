---
authors:
  - Aysan 
---

## Persona
**Tenant Administrator** - A user who manages MongoDB in Platform Mesh and needs to set up end-to-end encryption using their own L1 Customer Master Key through OpenKCM.

## Overview

As a Tenant Administrator, I need to enable OpenKCM for my tenant, configure my L1 Customer Master Key through the CMK UI, and ensure my MongoDB service can request encryption keys to protect database records with my customer-controlled encryption.

## User Stories

### Story 1: Enable OpenKCM for My Tenant
**As a** Tenant Administrator  
**I want to** enable OpenKCM services for my tenant account  
**So that** I can manage customer-controlled encryption for my MongoDB data  

**User Journey:**
1. I create or access my tenant account in KCP (Platform Mesh)
2. Platform Mesh detects my tenant and automatically provisions OpenKCM services
3. I receive notification that OpenKCM CMK UI is available
4. I can now access the CMK web interface to configure my encryption keys
5. Backend Crypto Service is automatically configured and ready for MongoDB integration

**Requirements:**
- Platform Mesh automatically provisions OpenKCM when tenant is created
- CMK UI becomes accessible within 5 minutes of tenant creation
- Crypto Service is configured and running for my tenant
- I receive clear notification when services are ready

### Story 2: Upload My L1 Customer Master Key
**As a** Tenant Administrator  
**I want to** configure my L1 Customer Master Key in the CMK UI  
**So that** all my data encryption uses my own customer-controlled key  

**User Journey:**
1. I log into the OpenKCM CMK UI with my tenant credentials
2. I navigate to "Master Key Configuration" section
3. I choose my key management option:
   - **Option A (BYOK)**: I provide my AWS KMS key ARN and configure access permissions
   - **Option B (HYOK)**: I configure connection to my on-premises HSM with endpoint details
4. I test the connection to verify my L1 key is accessible
5. I save the configuration and see my L1 key status as "Active"
6. System confirms my L1 key is ready for encryption operations

**Requirements:**
- CMK UI supports both BYOK (AWS KMS, Azure Key Vault) and HYOK (HSM) configurations
- Connection test validates L1 key accessibility before saving
- Configuration changes take effect within 2 minutes
- Clear status indicators show L1 key state (Inactive/Active/Error)
- All configuration changes are logged for audit

### Story 3: MongoDB Requests Encryption Key from OpenKCM
**As a** MongoDB service in my tenant  
**I want to** request encryption keys from OpenKCM Crypto Service  
**So that** I can encrypt database records using the tenant's L1 Customer Master Key  

**Technical Journey:**
1. **Data Write Request**: Application sends customer data to MongoDB
   ```json
   {
     "tenant_id": "my-tenant",
     "collection": "customer_data",
     "document": {
       "customer_id": "12345",
       "name": "John Doe",
       "email": "john@company.com"
     }
   }
   ```

2. **MongoDB Authentication**: I authenticate with OpenKCM Crypto Service
   - Use mTLS certificate provided by Platform Mesh
   - Certificate includes my tenant scope and MongoDB service identity

3. **Key Request**: I request encryption key via KMIP protocol
   ```
   KMIP Request:
   - Operation: Get Encryption Key
   - Tenant: my-tenant
   - Service: mongodb-prod
   ```

4. **Key Derivation**: Crypto Service processes my request
   - Validates I'm authorized for "my-tenant" 
   - Derives ephemeral encryption key from the configured L1 Customer Master Key
   - Returns fresh encryption key for this operation

5. **Data Encryption**: I encrypt the sensitive data
   ```python
   # Receive encryption key from OpenKCM
   encryption_key = crypto_response.key
   
   # Encrypt sensitive fields
   encrypted_doc = {
     "customer_id": "12345",  # Not encrypted (for indexing)
     "name": encrypt(encryption_key, "John Doe"),
     "email": encrypt(encryption_key, "john@company.com")
   }
   
   # Store encrypted document
   db.customer_data.insert_one(encrypted_doc)
   
   # Discard encryption key (ephemeral)
   secure_delete(encryption_key)
   ```

6. **Audit Logging**: All operations are logged
   - MongoDB logs: Successful data encryption for tenant
   - Crypto Service logs: Encryption key provided to MongoDB for my-tenant
   - CMK Service logs: L1 key used for encryption key derivation

**Requirements:**
- MongoDB authenticates successfully with Crypto Service using tenant-scoped certificates
- Encryption key requests complete within 100ms
- Each encryption operation uses a fresh ephemeral key
- Keys are derived from the tenant's configured L1 Customer Master Key
- All key operations are logged for audit compliance
- MongoDB can only access keys for its authorized tenant

## End-to-End Flow

```
1. Tenant Administrator → Enable OpenKCM (Platform Mesh provisions services)
                    ↓
2. Tenant Administrator → Configure L1 Key in CMK UI (BYOK/HYOK)
                    ↓
3. L1 Customer Master Key → Active and available for encryption operations
                    ↓
4. MongoDB → Request encryption key from Crypto Service
                    ↓
5. Crypto Service → Derive ephemeral key from L1 Customer Master Key
                    ↓
6. MongoDB → Encrypt data using derived key, store encrypted data
                    ↓
7. Data encrypted at rest using customer's L1 Key
```

## Requirements

### Functional Requirements:
- **REQ-001**: Platform Mesh must automatically provision OpenKCM services for new tenants
- **REQ-002**: CMK UI must support L1 key configuration (BYOK and HYOK)
- **REQ-003**: System must validate L1 key connectivity before activation
- **REQ-004**: MongoDB must authenticate with Crypto Service using mTLS
- **REQ-005**: Crypto Service must derive encryption keys from configured L1 key
- **REQ-006**: MongoDB must encrypt sensitive data using OpenKCM-provided keys
- **REQ-007**: All operations must maintain tenant isolation

### Performance Requirements:
- **REQ-008**: OpenKCM provisioning completes within 5 minutes
- **REQ-009**: L1 key configuration takes effect within 2 minutes
- **REQ-010**: Encryption key requests complete within 100ms
- **REQ-011**: Database performance impact stays under 20%

### Security Requirements:
- **REQ-012**: L1 key material never leaves customer's keystore (HYOK) or cloud KMS (BYOK)
- **REQ-013**: Ephemeral encryption keys are never persisted
- **REQ-014**: Cross-tenant key access is prevented and logged
- **REQ-015**: All key operations are audited with full traceability

## Success Criteria

### Setup Phase Success:
- ✅ OpenKCM services provisioned automatically when tenant created
- ✅ Tenant can access CMK UI and configure L1 Customer Master Key
- ✅ L1 key connection tested and confirmed active
- ✅ MongoDB service registered and authenticated with Crypto Service

### Operational Phase Success:
- ✅ MongoDB successfully requests encryption keys from Crypto Service
- ✅ Data encrypted using keys derived from tenant's L1 Customer Master Key
- ✅ Tenant isolation maintained (MongoDB only accesses own tenant's keys)
- ✅ Performance requirements met for all key operations

### Security Phase Success:
- ✅ All sensitive data encrypted at rest using customer-controlled L1 key
- ✅ Ephemeral keys used for each operation (no key reuse)
- ✅ Complete audit trail from L1 key usage to data encryption
- ✅ Customer maintains full control over their encryption keys

## Business Value

- **Complete Customer Control**: Tenant owns and controls their L1 encryption key
- **Seamless Integration**: MongoDB encryption works transparently with minimal setup
- **Regulatory Compliance**: Customer-controlled encryption meets compliance requirements
- **Data Security**: All database records protected with customer's own keys
- **Operational Simplicity**: Automated provisioning with simple UI configuration
- **Audit Ready**: Full traceability from key configuration to data encryption