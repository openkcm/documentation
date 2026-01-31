---
authors:
  - Aysan
---

## Persona
**Platform User** - An enterprise customer who needs encryption services for their applications. They enable OpenKCM from the Platform marketplace and expect their tenant environment to be automatically provisioned with proper cryptographic isolation, without needing to understand the technical details of tenant management.

## Overview

As a Platform User, I need to enable OpenKCM encryption services from the Platform marketplace for my organization. Once enabled, the system should automatically create an isolated tenant environment with proper encryption boundaries. The OpenKCM Controller handles all technical provisioning behind the scenes, implementing Multi-Tenant Cryptographic Isolation (MTCI) standards without requiring my intervention.

## User Stories

### Story 1: Enable OpenKCM from Platform Marketplace
**As a** Platform User  
**I want to** enable OpenKCM encryption services from the Platform marketplace  
**So that** my applications can use customer-controlled encryption with automatic tenant setup  

**User Journey:**
1. **Access Platform Marketplace**: I log into my Platform account
   - Organization: "ACME Corporation"
   - Account ID: "acme-corp-12345"
   - Region: "eu-west-1"

2. **Browse Security Services**: I navigate to encryption and security services
3. **Select OpenKCM**: I find "OpenKCM - Customer Managed Key Service" in the marketplace
   ```
   OpenKCM Service Details:
   ✓ Customer-controlled encryption keys
   ✓ Multi-tenant isolation
   ✓ KMIP protocol support
   ✓ Compliance ready (SOC2, GDPR, PCI DSS)
   ✓ Automatic service integration
   ```

4. **Enable Service**: I click "Enable OpenKCM" for my account
   - Service enables immediately in the marketplace
   - I receive confirmation: "OpenKCM is being provisioned for ACME Corporation"

5. **Automatic Provisioning Starts**: Behind the scenes, OpenKCM Controller detects my enablement
   ```json
   {
     "event": "service_enabled",
     "customer": "acme-corp-12345", 
     "service": "openkcm",
     "region": "eu-west-1",
     "timestamp": "2026-01-15T10:30:00Z"
   }
   ```

6. **Provisioning Completion**: I receive notification within 3 minutes
   ```
   "OpenKCM is ready for ACME Corporation!
   
   ✓ Tenant environment created
   ✓ Encryption keys configured  
   ✓ Services can now request encryption keys
   ✓ CMK UI available at: https://cmk.openkcm.io/tenant/acme-corp-12345"
   ```

**Requirements:**
- Simple one-click enablement from marketplace
- Automatic tenant provisioning within 3 minutes
- Clear confirmation when service is ready
- CMK UI immediately accessible after provisioning

### Story 2: Automatic Tenant Creation (Behind the Scenes)
**As a** Platform User  
**I want to** have my tenant automatically created with proper isolation  
**So that** I can start using encryption immediately without technical setup  

**What Happens Automatically (OpenKCM Controller Actions):**
1. **Event Detection**: OpenKCM Controller detects my service enablement
2. **Tenant Provisioning**: Controller creates my isolated tenant environment:
   ```
   Tenant: acme-corp-12345
   ├── L2 Tenant Key: L2-acme-corp-12345-primary (unique to my org)
   ├── Database Isolation: PostgreSQL RLS configured
   ├── Certificate Template: mTLS certs for my services
   └── Audit Stream: Dedicated logging for my operations
   ```

3. **Multi-Tenant Cryptographic Isolation**: My tenant gets mathematically isolated encryption:
   - Unique L2 Tenant Encryption Key (never shared with other customers)
   - Database-level isolation preventing cross-tenant access
   - Cryptographic context binding preventing data relocation

4. **Service Integration Setup**: My services can immediately use encryption:
   - MongoDB, PostgreSQL, microservices can request keys
   - mTLS certificates automatically issued for my services
   - Crypto Service routes my requests to isolated boundaries

**My Experience:**
- I don't need to understand L2 keys or MTCI standards
- My tenant is ready to use within minutes
- All technical complexity handled automatically
- I can focus on my applications, not key management infrastructure

**Requirements:**
- Zero technical configuration required from me
- Tenant isolation implemented per MTCI standard automatically
- All my services can immediately access encryption capabilities
- Complete audit trail maintained for compliance

### Story 3: Access My Tenant Management Interface
**As a** Platform User  
**I want to** access the CMK UI to manage my organization's encryption settings  
**So that** I can configure customer-managed keys and monitor my encryption usage  

**User Journey:**
1. **Access CMK UI**: I click the provided link to access my tenant
   - URL: `https://cmk.openkcm.io/tenant/acme-corp-12345`
   - Single sign-on through my Platform account

2. **View My Tenant Dashboard**: I see my organization's encryption environment
   ```
   ACME Corporation - Encryption Dashboard
   
   Tenant Status: Active
   Region: eu-west-1
   Services Using Encryption: 0 (ready for integration)
   
   Key Management:
   ├── Master Key: Platform-Managed (default)
   ├── Tenant Isolation: Active ✓
   └── Service Integration: Ready ✓
   ```

3. **Configure Customer-Managed Keys** (Optional): I can upgrade to BYOK/HYOK
   - Option to configure AWS KMS (BYOK)
   - Option to configure on-premises HSM (HYOK)
   - Keep platform-managed keys (default)

4. **Monitor Service Integration**: I see which services are using encryption
   ```
   Service Integration Status:
   
   MongoDB Clusters: 0 connected (ready)
   PostgreSQL Instances: 0 connected (ready)  
   Microservices: 0 connected (ready)
   
   To enable encryption for your services, follow the integration guide.
   ```

5. **View Usage Analytics**: I can track my encryption operations
   - Key requests per day/month
   - Service-level encryption usage
   - Compliance status and audit logs

**Requirements:**
- CMK UI accessible immediately after tenant creation
- Clear visibility into tenant status and capabilities
- Option to configure customer-managed keys
- Usage analytics and monitoring available

### Story 4: Integrate My Services with Encryption
**As a** Platform User  
**I want to** enable encryption for my existing Platform services  
**So that** my data is automatically protected with my tenant's encryption keys  

**User Journey:**
1. **Service Discovery**: My existing services automatically discover OpenKCM
   - MongoDB clusters in my namespace detect available encryption
   - Services receive mTLS certificates for OpenKCM integration

2. **Automatic Encryption Enablement**: My services start using encryption transparently
   ```python
   # My MongoDB service automatically encrypts data
   # No code changes required in my applications
   
   # Before: Plain text storage
   customer_record = {"name": "John Doe", "email": "john@example.com"}
   
   # After: Automatic encryption (behind the scenes)
   # MongoDB requests key from OpenKCM using my tenant's L2 key
   # Data encrypted with ephemeral keys derived from my tenant boundary
   ```

3. **Monitor Encryption Usage**: I see encryption activity in CMK UI
   ```
   Service Integration Update:
   
   MongoDB Clusters: 2 connected, encrypting data ✓
   ├── customer-db: 1,250 operations/day
   └── analytics-db: 890 operations/day
   
   Encryption Status: All sensitive data protected
   Performance Impact: <5% overhead
   ```

4. **Compliance Reporting**: I can generate reports showing encryption coverage
   - Which services are using encryption
   - What data is protected by customer-controlled keys
   - Audit trail for compliance requirements

**Requirements:**
- Services integrate with encryption automatically
- No application code changes required
- Clear visibility into which data is encrypted
- Compliance reporting available for audits

## Technical Flow (Behind the Scenes)

### Platform User Enablement to Service Integration:
```
Platform User enables OpenKCM in marketplace
                ↓
OpenKCM Controller detects enablement event
                ↓
Controller creates tenant with unique L2 key (MTCI standard)
                ↓
PostgreSQL RLS configured for tenant isolation
                ↓
mTLS certificate templates created for tenant services
                ↓
Crypto Service routing configured for tenant requests
                ↓
Platform User receives "Ready" notification
                ↓
Platform User accesses CMK UI
                ↓
Platform User's services automatically discover OpenKCM
                ↓
Services request encryption keys using tenant's L2 boundary
                ↓
Data encrypted with mathematically isolated keys
```

### Multi-Tenant Isolation Implementation:
```
Customer: ACME Corp (acme-corp-12345)
├── L2 Tenant Key: Unique to ACME Corp only
├── Database Isolation: RLS prevents cross-tenant access
├── Service Certificates: Scoped to ACME Corp tenant only
└── Audit Logs: Dedicated stream for ACME Corp operations

Customer: Beta Corp (beta-corp-67890)  
├── L2 Tenant Key: Unique to Beta Corp only (different from ACME)
├── Database Isolation: RLS prevents access to ACME data
├── Service Certificates: Scoped to Beta Corp tenant only
└── Audit Logs: Dedicated stream for Beta Corp operations
```

## Requirements

### User Experience Requirements:
- **REQ-001**: One-click enablement from Platform marketplace
- **REQ-002**: Automatic tenant provisioning within 3 minutes
- **REQ-003**: CMK UI accessible immediately after provisioning
- **REQ-004**: Services integrate with encryption automatically

### Technical Requirements (Automated by Controller):
- **REQ-005**: Each tenant gets unique L2 Tenant Encryption Key
- **REQ-006**: PostgreSQL RLS enforces tenant isolation
- **REQ-007**: mTLS certificates scoped to tenant boundaries
- **REQ-008**: Crypto Service routes requests to correct tenant

### Security Requirements:
- **REQ-009**: Multi-Tenant Cryptographic Isolation (MTCI) standard implemented
- **REQ-010**: Cross-tenant access mathematically impossible
- **REQ-011**: All tenant operations logged for audit compliance
- **REQ-012**: Tenant boundaries validated automatically

## Success Criteria

### User Experience Success:
- ✅ Platform Users can enable OpenKCM with one click
- ✅ Tenant environment ready within 3 minutes
- ✅ CMK UI accessible and easy to navigate
- ✅ Services integrate with encryption automatically

### Security Success:
- ✅ Each customer gets mathematically isolated tenant
- ✅ L2 Tenant Keys unique per tenant (MTCI compliance)
- ✅ Cross-tenant access prevention verified through testing
- ✅ Complete audit trails for all tenant operations

### Integration Success:
- ✅ Existing services (MongoDB, etc.) can use encryption immediately
- ✅ No application code changes required
- ✅ Performance impact minimal (<5% overhead)
- ✅ Compliance reporting available for customer audits

## Business Value

- **Effortless Onboarding**: Customers get enterprise encryption with one click
- **Zero Configuration**: Technical complexity handled automatically
- **Immediate Value**: Services can use encryption within minutes of enablement
- **Mathematical Isolation**: Each customer's data cryptographically separated
- **Compliance Ready**: Audit trails and reporting built-in from day one
- **Scalable Growth**: Controller handles thousands of customer tenants automatically
- **Cost Effective**: Shared infrastructure with isolated tenant boundaries