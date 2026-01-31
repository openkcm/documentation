
# OpenKCM MVP Deliverables - June 2026

## Architecture Overview

The system is designed with a specific fallback mechanism when the CMK (L1) layer is not linked. In this scenario, the internal versioned key serves as the anchor within the Crypto layer. This enables parallel development within two logical systems: **Crypto Layer** and **CMK Layer**.

---

## CMK Service MVP
 
**Timeline:** January - June 2026

### MVP CMK Goals by End of June 2026:
- ✅ **CMK UI integrated in Platform** - Seamless user interface within platform dashboard
- ✅ **Platform users can enable OpenKCM** - One-click automated deployment
- ✅ **AWS KMS plugin** - Users can connect their AWS KMS keys
- ✅ **Upload L1 keys from AWS KMS** via CMK UI - Import customer master keys
- ✅ **View and enable L1 keys on specific tenants** - Tenant-key assignment interface
- ✅ **Tenant management** - Create, manage, assign keys to tenants

### CMK Dashboard - Key Management


### CMK Core Features:
- **Platform Integration** - Embedded UI with SSO
- **AWS KMS Plugin** - Connect to customer's AWS accounts
- **Key Import Workflow** - Retrieve and display AWS KMS keys
- **Tenant-Key Assignment** - Assign specific keys to specific tenants
- **Key Management Dashboard** - View all imported and platform keys
- **One-Click Enablement** - Automated service deployment from platform

### CMK Independence:
- **No Crypto Layer dependency** - CMK operates independently
- **Standalone key management** - Manages L1 keys without crypto integration
- **Platform-focused** - UI and tenant management primary goals
- **Future integration ready** - Architecture supports eventual Crypto connection

---

## Crypto Service MVP

 **Timeline:** January - June 2026

### MVP Crypto Goals by End of June 2026:
- ✅ **KMIP server implementation** - Basic encrypt/decrypt protocol for MongoDB
- ✅ **Simple tenant key management** - Generate/store keys per tenant
- ✅ **MongoDB integration** - Transparent database encryption via KMIP
- ✅ **Basic tenant isolation** - Separate key storage per tenant
- ✅ **Key generation service** - Create encryption keys for MongoDB collections
- ✅ **Health monitoring** - Service status and basic metrics

### Crypto Service - MongoDB Encryption Independent

```
MongoDB Database
        ↓ (KMIP Protocol)
Crypto KMIP Server
├── Tenant Key Generation
├── Simple Key Storage  
├── Basic Tenant Isolation
└── MongoDB Integration
```

### MongoDB Use Case (Independent Operation):
1. **MongoDB needs encryption** for a collection
2. **MongoDB calls Crypto** via KMIP protocol
3. **Crypto generates tenant-specific key** (no CMK involvement)
4. **Crypto returns key** to MongoDB via KMIP response
5. **MongoDB encrypts collection** transparently
6. **No external dependencies** - fully self-contained

### Crypto Core Features:
- **KMIP Protocol Server** - Handle MongoDB key requests
- **Autonomous Key Generation** - Create AES-256 keys for database encryption
- **Tenant Database** - Simple tenant → keys mapping storage
- **MongoDB Direct Integration** - KMIP-based transparent encryption
- **Basic Audit Logging** - Key operation tracking for compliance
- **Service Health Monitoring** - Status checks and basic metrics

### Crypto Independence:
- **No CMK dependency** - Generates and manages own keys
- **MongoDB-focused** - Direct integration with database encryption
- **Self-contained** - All encryption capabilities built-in
- **Scalable foundation** - Architecture ready for future CMK integration

---

## Parallel Development Strategy

### Independent Delivery Goals:
- **CMK Team** delivers customer key management and platform integration
- **Crypto Team** delivers working MongoDB encryption capabilities
- **No cross-team dependencies** during MVP phase
- **Both teams deliver complete, usable functionality**

### Future Integration (Post-MVP):
- **Q3 2026:** Crypto Layer integrates with CMK L1 keys
- **Q4 2026:** Advanced key hierarchy (L1 → L2 → L3 → L4)
- **Q1 2027:** Full OpenKCM integration with external keystores

### Success Criteria:

#### CMK Success:
- ✅ Platform users can enable OpenKCM seamlessly
- ✅ Users can import and manage AWS KMS keys
- ✅ Tenant-key assignments work correctly
- ✅ UI integrated smoothly within platform experience

#### Crypto Success:
- ✅ MongoDB databases encrypt transparently using Crypto keys
- ✅ KMIP protocol handles all standard operations
- ✅ Tenant isolation prevents cross-tenant key access
- ✅ Service operates reliably with proper monitoring

### Business Value:
- **Immediate Customer Value** - Working encryption and key management
- **Parallel Team Productivity** - No blocking dependencies
- **Risk Mitigation** - Each team delivers independently
- **Future-Ready Architecture** - Integration path clearly defined
- **Compliance Foundation** - Audit logging and security controls in place