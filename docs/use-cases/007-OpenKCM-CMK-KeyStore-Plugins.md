---
authors:
  - Aysan
---

## Persona
**Enterprise Security Administrator** - A security professional responsible for managing their organization's encryption keys and compliance requirements. They need to connect their existing external keystores (AWS KMS, Fortanix, Thales, OpenBao) to OpenKCM's CMK service to maintain control over their Customer Master Keys while leveraging OpenKCM's encryption services.

## Overview

As an Enterprise Security Administrator, I need to integrate my organization's existing external keystores with OpenKCM's CMK service. This allows me to maintain our Customer Master Keys (L1) in our trusted keystore infrastructure while enabling OpenKCM to derive tenant and service keys for our Platform Mesh applications. The CMK Plugins provide secure connectors to various keystore providers without exposing our master keys.

## Business Context

Organizations choose external keystores for various reasons:
- **Regulatory Compliance**: Some regulations require on-premises key control
- **Existing Infrastructure**: Organizations have invested in HSMs or cloud KMS
- **Security Policies**: Corporate security mandates specific keystore vendors
- **Multi-Cloud Strategy**: Keys managed independently from application infrastructure
- **Audit Requirements**: Centralized key governance across all systems

## User Stories

### Story 1: Configure AWS KMS Plugin for BYOK
**As an** Enterprise Security Administrator  
**I want to** connect my AWS KMS to OpenKCM using the AWS KMS Plugin  
**So that** my Customer Master Keys remain in AWS while OpenKCM can derive encryption keys  

**Configuration Journey:**
1. **Access CMK Plugin Management**: I log into OpenKCM CMK UI
   - Navigate to "External Keystores" → "Plugin Configuration"
   - Select "Add New Keystore" → "AWS KMS"

2. **Configure AWS KMS Connection**: I provide AWS KMS details
   ```json
   {
     "plugin_type": "aws_kms",
     "plugin_name": "primary_aws_kms",
     "configuration": {
       "region": "us-east-1",
       "kms_key_id": "arn:aws:kms:us-east-1:123456789012:key/12345678-1234-1234-1234-123456789012",
       "role_arn": "arn:aws:iam::123456789012:role/OpenKCM-KMS-Access",
       "external_id": "openkcm-external-id-12345"
     }
   }
   ```

3. **Set Up IAM Permissions**: I configure AWS IAM for OpenKCM access
   ```json
   {
     "Version": "2012-10-17",
     "Statement": [
       {
         "Effect": "Allow",
         "Action": [
           "kms:Decrypt",
           "kms:DescribeKey",
           "kms:GenerateDataKey"
         ],
         "Resource": "arn:aws:kms:us-east-1:123456789012:key/12345678-1234-1234-1234-123456789012",
         "Condition": {
           "StringEquals": {
             "kms:ViaService": "openkcm.us-east-1.amazonaws.com"
           }
         }
       }
     ]
   }
   ```

4. **Test Connection**: I validate the plugin configuration
   - Click "Test Connection" in CMK UI
   - System performs test encryption/decryption with AWS KMS
   - Validates IAM permissions and key accessibility

5. **Activate Plugin**: I enable the AWS KMS plugin for my tenant
   ```json
   {
     "status": "connection_successful",
     "plugin_status": "active",
     "l1_key_location": "aws_kms",
     "key_operations_ready": true
   }
   ```

6. **Verify L1 Key Usage**: OpenKCM confirms it can derive keys from my AWS KMS
   - L2 tenant keys encrypted using my AWS KMS Customer Master Key
   - All key derivation operations use AWS KMS for L1 key access
   - Audit logs show successful integration with AWS KMS

**Requirements:**
- AWS KMS plugin connects within 5 minutes
- IAM role validation ensures secure access
- Test operations confirm key accessibility
- All KMS operations logged for audit compliance

### Story 2: Configure Fortanix DSM Plugin for Cloud HSM
**As an** Enterprise Security Administrator  
**I want to** connect my Fortanix Data Security Manager to OpenKCM  
**So that** my keys remain in Fortanix HSM while enabling OpenKCM encryption services  

**Configuration Journey:**
1. **Select Fortanix Plugin**: I choose Fortanix DSM from available plugins
   - Plugin Type: "Fortanix Data Security Manager"
   - Integration Mode: "Cloud HSM with API Access"

2. **Configure Fortanix Connection**: I provide Fortanix DSM details
   ```json
   {
     "plugin_type": "fortanix_dsm",
     "plugin_name": "fortanix_production_hsm",
     "configuration": {
       "dsm_endpoint": "https://sdkms.fortanix.com",
       "app_uuid": "550e8400-e29b-41d4-a716-446655440000",
       "api_key": "fortanix_api_key_encrypted_blob",
       "group_id": "openkcm-key-group",
       "key_name": "customer-master-key-acme-corp"
     }
   }
   ```

3. **Set Up Fortanix Permissions**: I configure DSM access controls
   ```json
   {
     "group_permissions": {
       "group_name": "openkcm-key-group",
       "permissions": [
         "ENCRYPT",
         "DECRYPT", 
         "DERIVE_KEY",
         "EXPORT_KEY"
       ],
       "applications": ["openkcm-crypto-service"],
       "key_usage_policies": {
         "max_operations_per_hour": 10000,
         "allowed_algorithms": ["AES-256-GCM"]
       }
     }
   }
   ```

4. **Test Fortanix Integration**: I validate HSM connectivity
   ```python
   # Test operations performed by plugin
   test_operations = {
     "connectivity": "test_dsm_connection()",
     "key_access": "test_key_operations(customer_master_key)",
     "derive_test": "test_l2_key_derivation()",
     "audit_test": "verify_hsm_audit_logs()"
   }
   ```

5. **Activate Fortanix Plugin**: System confirms HSM integration ready
   ```json
   {
     "plugin_status": "active",
     "hsm_connectivity": "connected",
     "key_operations": "verified",
     "fips_compliance": "level_3_validated"
   }
   ```

**Requirements:**
- Fortanix DSM API integration working within 10 minutes
- HSM key operations tested and verified
- FIPS Level 3 compliance maintained
- Rate limiting respects HSM capacity

### Story 3: Configure Thales Luna HSM Plugin for On-Premises HYOK
**As an** Enterprise Security Administrator  
**I want to** connect my on-premises Thales Luna HSM to OpenKCM  
**So that** my Customer Master Keys never leave my data center while enabling cloud encryption  

**Configuration Journey:**
1. **Select Thales Luna Plugin**: I choose on-premises HSM integration
   - Plugin Type: "Thales Luna Network HSM"
   - Connection Method: "PKCS#11 Interface"

2. **Configure HSM Network Connection**: I provide Luna HSM details
   ```json
   {
     "plugin_type": "thales_luna",
     "plugin_name": "datacenter_primary_hsm",
     "configuration": {
       "hsm_ip_addresses": ["192.168.1.100", "192.168.1.101"],
       "partition_label": "openkcm_partition",
       "slot_id": 1,
       "client_certificate": "thales_client_cert.pem",
       "client_private_key": "thales_client_key.pem",
       "ca_certificate": "thales_ca_cert.pem"
     }
   }
   ```

3. **Set Up HSM Partitioning**: I configure dedicated HSM partition for OpenKCM
   ```bash
   # Commands run on Luna HSM
   lunacm
   > partition create -partition openkcm_partition -domain example.com
   > partition assignPed -partition openkcm_partition
   > clientIdentificationPolicy create -partition openkcm_partition -policy openkcm_policy
   ```

4. **Configure Network Security**: I set up secure communication
   ```yaml
   # Network configuration for HSM access
   hsm_network:
     encryption: "TLS 1.3"
     authentication: "mutual_tls"
     network_policies:
       - source: "openkcm-crypto-service"
         destination: "192.168.1.100:1792"
         protocol: "PKCS#11"
     firewall_rules:
       - allow: "openkcm_service_ips"
       - deny: "all_other_traffic"
   ```

5. **Test HSM Operations**: I validate on-premises connectivity
   ```python
   # HSM connectivity tests
   hsm_tests = {
     "network_connectivity": "ping_hsm_appliances()",
     "pkcs11_interface": "test_pkcs11_operations()",
     "key_generation": "test_master_key_access()",
     "latency_check": "measure_hsm_response_times()"
   }
   ```

6. **Activate Thales Plugin**: System confirms HSM ready for key operations
   ```json
   {
     "plugin_status": "active",
     "hsm_appliances": ["192.168.1.100:online", "192.168.1.101:standby"],
     "partition_status": "authenticated",
     "fips_level": "4_validated",
     "latency_ms": 45
   }
   ```

**Requirements:**
- On-premises HSM connectivity established securely
- PKCS#11 interface working correctly
- Network latency under 50ms for key operations
- HSM high availability with failover tested

### Story 4: Configure OpenBao Plugin for Open Source Key Management
**As an** Enterprise Security Administrator  
**I want to** connect my OpenBao deployment to OpenKCM  
**So that** I can use open-source key management with OpenKCM encryption services  

**Configuration Journey:**
1. **Select OpenBao Plugin**: I choose the open-source Vault-compatible option
   - Plugin Type: "OpenBao (Vault-Compatible)"
   - Authentication Method: "AppRole with JWT"

2. **Configure OpenBao Connection**: I provide OpenBao cluster details
   ```json
   {
     "plugin_type": "openbao",
     "plugin_name": "openbao_cluster_production",
     "configuration": {
       "vault_address": "https://openbao.company.internal:8200",
       "auth_method": "approle",
       "role_id": "openkcm-role-id-12345",
       "secret_id_path": "/etc/openkcm/openbao-secret-id",
       "mount_path": "openkcm-keys",
       "key_name": "customer-master-key"
     }
   }
   ```

3. **Set Up OpenBao Policies**: I configure access policies for OpenKCM
   ```hcl
   # OpenBao policy for OpenKCM integration
   path "openkcm-keys/data/customer-master-key" {
     capabilities = ["read"]
   }
   
   path "openkcm-keys/encrypt/customer-master-key" {
     capabilities = ["update"]
   }
   
   path "openkcm-keys/decrypt/customer-master-key" {
     capabilities = ["update"]
   }
   
   path "auth/token/lookup-self" {
     capabilities = ["read"]
   }
   ```

4. **Configure AppRole Authentication**: I set up service authentication
   ```bash
   # Configure AppRole for OpenKCM service
   vault auth enable approle
   vault write auth/approle/role/openkcm-service \
     token_policies="openkcm-policy" \
     token_ttl=1h \
     token_max_ttl=4h \
     bind_secret_id=true
   ```

5. **Test OpenBao Integration**: I validate the open-source integration
   ```python
   # OpenBao integration tests
   vault_tests = {
     "authentication": "test_approle_login()",
     "key_access": "test_transit_operations()",
     "policy_enforcement": "verify_access_controls()",
     "token_renewal": "test_token_lifecycle()"
   }
   ```

6. **Activate OpenBao Plugin**: System confirms Vault compatibility
   ```json
   {
     "plugin_status": "active",
     "vault_version": "compatible",
     "auth_status": "authenticated",
     "policies_applied": "verified",
     "transit_engine": "enabled"
   }
   ```

**Requirements:**
- OpenBao API compatibility verified
- AppRole authentication working correctly
- Transit secret engine configured for key operations
- Policy enforcement preventing unauthorized access

## Plugin Management and Operations

### Story 5: Monitor and Manage Plugin Health
**As an** Enterprise Security Administrator  
**I want to** monitor the health and performance of my external keystore plugins  
**So that** I can ensure reliable key operations and proactive issue resolution  

**Management Journey:**
1. **Plugin Health Dashboard**: I monitor all configured keystores
   ```
   External Keystore Status Dashboard
   
   AWS KMS (primary_aws_kms):           ✅ Healthy (2ms avg latency)
   Fortanix DSM (fortanix_production):  ✅ Healthy (5ms avg latency)  
   Thales Luna (datacenter_primary):    ⚠️  Warning (48ms avg latency)
   OpenBao (openbao_cluster):          ✅ Healthy (3ms avg latency)
   ```

2. **Performance Monitoring**: I track key operation metrics
   ```json
   {
     "plugin_metrics": {
       "aws_kms": {
         "operations_per_hour": 2450,
         "success_rate": 99.8,
         "avg_latency_ms": 2.1,
         "error_rate": 0.2
       },
       "thales_luna": {
         "operations_per_hour": 890,
         "success_rate": 99.9,
         "avg_latency_ms": 48.2,
         "error_rate": 0.1
       }
     }
   }
   ```

3. **Handle Plugin Failures**: I manage keystore connectivity issues
   ```python
   # Automated failover and alerting
   def handle_plugin_failure(plugin_name, error_type):
       if error_type == "connectivity_lost":
           # Switch to backup plugin if configured
           activate_backup_plugin(plugin_name)
           
       elif error_type == "authentication_failed":
           # Refresh credentials and retry
           refresh_plugin_credentials(plugin_name)
           
       # Alert security team
       send_alert(f"Plugin {plugin_name} experiencing {error_type}")
   ```

4. **Plugin Rotation and Updates**: I manage plugin lifecycle
   - Rotate authentication credentials on schedule
   - Update plugin versions for security patches
   - Test plugin connectivity before deploying changes
   - Maintain plugin backup configurations

**Requirements:**
- Real-time plugin health monitoring
- Automated failover for plugin failures
- Performance metrics and alerting
- Plugin lifecycle management capabilities

## Technical Architecture

### Plugin Architecture Overview:
```
OpenKCM CMK Service
├── Plugin Manager
│   ├── AWS KMS Plugin
│   ├── Fortanix DSM Plugin
│   ├── Thales Luna Plugin
│   └── OpenBao Plugin
├── Plugin Interface (Standardized API)
├── Connection Pool Manager
├── Credential Manager (Encrypted Storage)
└── Health Monitor & Alerting

External Keystores
├── AWS KMS (Cloud)
├── Fortanix DSM (Cloud HSM)
├── Thales Luna HSM (On-Premises)
└── OpenBao (Self-Managed)
```

### Key Operation Flow with Plugins:
```
Service requests encryption key
        ↓
OpenKCM Crypto Service identifies tenant L1 key location
        ↓
Plugin Manager routes to appropriate keystore plugin
        ↓
Plugin authenticates with external keystore
        ↓
External keystore performs L1 key operation
        ↓
Plugin returns result to OpenKCM
        ↓
OpenKCM derives L2/L3/L4 keys using L1 result
        ↓
Ephemeral key provided to service
```

## Requirements

### Functional Requirements:
- **REQ-001**: Support AWS KMS, Fortanix, Thales, and OpenBao keystores
- **REQ-002**: Each plugin must provide secure authentication mechanisms
- **REQ-003**: Plugin failures must not prevent access to L1 keys
- **REQ-004**: All plugin operations must be logged for audit compliance

### Performance Requirements:
- **REQ-005**: Plugin operations complete within 100ms (95th percentile)
- **REQ-006**: Support 1000+ key operations per second per plugin
- **REQ-007**: Plugin failover completes within 30 seconds
- **REQ-008**: Connection pooling optimizes keystore access

### Security Requirements:
- **REQ-009**: L1 Customer Master Keys never leave external keystores
- **REQ-010**: Plugin credentials encrypted at rest and in transit
- **REQ-011**: Mutual authentication between plugins and keystores
- **REQ-012**: Plugin access logs integrated with tenant audit streams

## Success Criteria

### Integration Success:
- ✅ All four keystore types (AWS, Fortanix, Thales, OpenBao) integrate successfully
- ✅ Plugin configuration completed through CMK UI
- ✅ Key operations work reliably with each external keystore
- ✅ Plugin failures handled gracefully with appropriate failover

### Security Success:
- ✅ L1 keys remain in customer-controlled external keystores
- ✅ Plugin authentication prevents unauthorized access
- ✅ All keystore operations maintain complete audit trails
- ✅ Compliance requirements met for each keystore type

### Operational Success:
- ✅ Plugin health monitoring provides real-time visibility
- ✅ Performance meets SLAs across all keystore integrations
- ✅ Plugin lifecycle management (updates, rotation) works smoothly
- ✅ Documentation enables customers to self-configure plugins

## Business Value

- **Customer Choice**: Support for multiple keystore vendors and deployment models
- **Regulatory Compliance**: Enables HYOK and BYOK scenarios for different regulations
- **Existing Investment Protection**: Leverages customers' existing keystore infrastructure
- **Risk Distribution**: Reduces vendor lock-in by supporting multiple keystore options
- **Hybrid Deployment**: Supports both cloud and on-premises key management requirements
- **Open Standards**: OpenBao support demonstrates commitment to open-source solutions