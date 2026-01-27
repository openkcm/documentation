---
authors:
  - Aysan
---

## Persona
**CMK API Server** - The OpenKCM Customer Managed Key API service that needs to automatically generate x509 client certificates for tenant authentication to external keystores. The CMK API Server integrates with OpenBao PKI service (running on Platform Mesh) to request, generate, and manage client certificates on-demand for secure keystore connections.

## Overview

As the CMK API Server, I need to integrate with the OpenBao PKI service running on Platform Mesh to automatically generate x509 client certificates for tenant authentication to external keystores. When tenants configure keystore connections that require client certificate authentication, I must request properly signed certificates from OpenBao PKI and configure them for secure mTLS connections to external keystores.

## Business Context

The CMK API Server needs automated certificate generation because:
- **Tenant Scaling**: Each tenant may need multiple certificates for different keystores
- **Security Requirements**: External keystores require mutual TLS authentication
- **Automation**: Certificate generation must be seamless during tenant onboarding
- **Certificate Lifecycle**: Automatic renewal and rotation without service disruption
- **Compliance**: All certificates must follow organizational PKI policies
- **Integration**: Leverage existing OpenBao PKI infrastructure on Platform Mesh

## User Stories

### Story 1: Initialize OpenBao PKI Plugin for Certificate Operations
**As the** CMK API Server  
**I want to** establish connection with OpenBao PKI service on Platform Mesh  
**So that** I can request x509 client certificates for tenant keystore authentication  

**Technical Journey:**
1. **Discover OpenBao PKI Service**: I discover OpenBao PKI through Platform Mesh service registry
   ```yaml
   # Service discovery for OpenBao PKI
   apiVersion: v1
   kind: Service
   metadata:
     name: openbao-pki-service
     namespace: platform-services
   spec:
     selector:
       app: openbao
       component: pki-engine
     ports:
     - name: vault-api
       port: 8200
       protocol: TCP
   ```

2. **Initialize PKI Plugin Configuration**: I configure my OpenBao PKI plugin
   ```json
   {
     "plugin_config": {
       "plugin_name": "openbao_pki_client",
       "vault_address": "https://openbao-pki-service.platform-services.svc.cluster.local:8200",
       "pki_mount_path": "pki_client_certs",
       "auth_method": "kubernetes",
       "service_account": "cmk-api-server",
       "namespace": "openkcm-system"
     }
   }
   ```

3. **Authenticate with OpenBao PKI**: I authenticate using Kubernetes service account
   ```python
   def authenticate_with_openbao():
       # Read Kubernetes service account token
       sa_token = read_service_account_token("/var/run/secrets/kubernetes.io/serviceaccount/token")
       
       # Authenticate with OpenBao using Kubernetes auth
       auth_response = vault_client.auth.kubernetes.login(
           role="cmk-api-server-role",
           jwt=sa_token
       )
       
       # Store vault token for subsequent operations
       self.vault_token = auth_response["auth"]["client_token"]
       return True
   ```

4. **Validate PKI Engine Access**: I verify I can access the PKI engine
   ```python
   def validate_pki_access():
       try:
           # Check PKI mount accessibility
           pki_config = vault_client.secrets.pki.read_config(
               mount_point="pki_client_certs"
           )
           
           # Verify required PKI role exists
           role_info = vault_client.secrets.pki.read_role(
               name="openkcm-client-cert-role",
               mount_point="pki_client_certs"
           )
           
           return True
       except Exception as e:
           log_error(f"PKI access validation failed: {e}")
           return False
   ```

5. **Initialize Certificate Request Templates**: I prepare certificate request templates
   ```json
   {
     "certificate_templates": {
       "thales_luna": {
         "common_name_pattern": "tenant-{tenant_id}-thales.keystore.internal",
         "key_type": "rsa",
         "key_bits": 4096,
         "ttl": "8760h"
       },
       "aws_kms": {
         "common_name_pattern": "tenant-{tenant_id}-aws.keystore.internal", 
         "key_type": "ec",
         "key_bits": 256,
         "ttl": "4380h"
       }
     }
   }
   ```

**Requirements:**
- Successful authentication with OpenBao PKI service
- PKI engine accessibility verified
- Certificate templates configured for different keystore types
- Plugin initialization completes within 30 seconds

### Story 2: Generate Client Certificate for Tenant Keystore Connection
**As the** CMK API Server  
**I want to** request x509 client certificates from OpenBao PKI when tenants configure keystores  
**So that** tenants can authenticate securely to their external keystores using mTLS  

**Certificate Generation Journey:**
1. **Receive Tenant Keystore Configuration**: I receive tenant keystore setup request
   ```json
   {
     "tenant_id": "acme-corp-12345",
     "keystore_config": {
       "type": "thales_luna_hsm",
       "connection_endpoints": ["192.168.1.100:1792"],
       "authentication_method": "client_certificate",
       "certificate_required": true
     }
   }
   ```

2. **Prepare Certificate Request**: I build certificate request based on keystore type
   ```python
   def prepare_certificate_request(tenant_id, keystore_type):
       template = self.certificate_templates[keystore_type]
       
       cert_request = {
           "common_name": template["common_name_pattern"].format(tenant_id=tenant_id),
           "alt_names": [f"tenant-{tenant_id}.internal"],
           "key_type": template["key_type"],
           "key_bits": template["key_bits"],
           "ttl": template["ttl"],
           "format": "pem_bundle"
       }
       
       return cert_request
   ```

3. **Request Certificate from OpenBao PKI**: I call OpenBao PKI to generate certificate
   ```python
   def generate_client_certificate(tenant_id, keystore_type):
       # Prepare certificate request
       cert_request = self.prepare_certificate_request(tenant_id, keystore_type)
       
       # Submit certificate generation request to OpenBao PKI
       response = vault_client.secrets.pki.generate_certificate(
           name="openkcm-client-cert-role",
           extra_params=cert_request,
           mount_point="pki_client_certs"
       )
       
       return {
           "certificate": response["data"]["certificate"],
           "private_key": response["data"]["private_key"],
           "ca_chain": response["data"]["ca_chain"],
           "serial_number": response["data"]["serial_number"]
       }
   ```

4. **Store Certificate Securely**: I encrypt and store the certificate bundle
   ```python
   def store_certificate_bundle(tenant_id, keystore_type, cert_bundle):
       # Encrypt private key using tenant's L2 key
       encrypted_private_key = encrypt_with_tenant_key(
           tenant_id=tenant_id,
           data=cert_bundle["private_key"]
       )
       
       # Store in secure certificate database
       certificate_record = {
           "tenant_id": tenant_id,
           "keystore_type": keystore_type,
           "certificate": cert_bundle["certificate"],
           "encrypted_private_key": encrypted_private_key,
           "ca_chain": cert_bundle["ca_chain"],
           "serial_number": cert_bundle["serial_number"],
           "created_at": datetime.utcnow(),
           "expires_at": extract_expiry_date(cert_bundle["certificate"])
       }
       
       return self.db.certificates.insert(certificate_record)
   ```

5. **Configure Keystore Connection**: I configure the certificate for keystore authentication
   ```python
   def configure_keystore_authentication(tenant_id, keystore_type, cert_bundle):
       # Create keystore connection configuration
       keystore_config = {
           "tenant_id": tenant_id,
           "keystore_type": keystore_type,
           "authentication": {
               "method": "client_certificate",
               "certificate_path": f"/certs/{tenant_id}-{keystore_type}.crt",
               "private_key_path": f"/private/{tenant_id}-{keystore_type}.key",
               "ca_chain_path": f"/ca/{keystore_type}-ca-chain.pem"
           }
       }
       
       # Test mTLS connection to keystore
       connection_test = test_keystore_connection(keystore_config)
       
       if connection_test.success:
           # Activate keystore connection
           activate_keystore_connection(tenant_id, keystore_type)
       else:
           raise KeystoreConnectionError(f"Certificate authentication failed: {connection_test.error}")
   ```

6. **Schedule Certificate Renewal**: I set up automatic renewal before expiry
   ```python
   def schedule_certificate_renewal(tenant_id, keystore_type, cert_expiry):
       renewal_time = cert_expiry - timedelta(days=30)  # Renew 30 days before expiry
       
       renewal_job = {
           "tenant_id": tenant_id,
           "keystore_type": keystore_type,
           "renewal_time": renewal_time,
           "job_type": "certificate_renewal"
       }
       
       self.scheduler.add_job(renewal_job)
   ```

**Requirements:**
- Certificate generation completes within 2 minutes
- Generated certificates successfully authenticate to target keystore
- Private keys encrypted using tenant-specific encryption
- Certificate renewal scheduled automatically

### Story 3: Handle Certificate Renewal and Rotation
**As the** CMK API Server  
**I want to** automatically renew client certificates before they expire  
**So that** tenant keystore connections remain active without interruption  

**Certificate Renewal Journey:**
1. **Monitor Certificate Expiration**: I continuously check certificate expiry dates
   ```python
   def monitor_certificate_expiration():
       # Check all certificates expiring within 30 days
       expiring_certs = self.db.certificates.find({
           "expires_at": {"$lte": datetime.utcnow() + timedelta(days=30)},
           "status": "active"
       })
       
       for cert in expiring_certs:
           self.schedule_certificate_renewal(
               cert["tenant_id"], 
               cert["keystore_type"]
           )
   ```

2. **Generate New Certificate**: I request new certificate from OpenBao PKI
   ```python
   def renew_certificate(tenant_id, keystore_type):
       # Generate new certificate with same parameters
       new_cert_bundle = self.generate_client_certificate(tenant_id, keystore_type)
       
       # Store new certificate with "pending" status
       new_cert_id = self.store_certificate_bundle(
           tenant_id, keystore_type, new_cert_bundle, status="pending"
       )
       
       return new_cert_id
   ```

3. **Test New Certificate**: I validate new certificate works with keystore
   ```python
   def test_new_certificate(tenant_id, keystore_type, new_cert_id):
       # Retrieve new certificate bundle
       new_cert = self.db.certificates.find_one({"_id": new_cert_id})
       
       # Test keystore connection with new certificate
       test_config = self.build_keystore_config(tenant_id, keystore_type, new_cert)
       
       connection_result = test_keystore_connection(test_config)
       
       if connection_result.success:
           return True
       else:
           # Mark certificate as failed
           self.db.certificates.update_one(
               {"_id": new_cert_id},
               {"$set": {"status": "failed", "error": connection_result.error}}
           )
           return False
   ```

4. **Perform Certificate Rotation**: I switch to new certificate seamlessly
   ```python
   def rotate_certificate(tenant_id, keystore_type, new_cert_id):
       # Mark old certificate as "rotating"
       old_cert = self.db.certificates.find_one({
           "tenant_id": tenant_id,
           "keystore_type": keystore_type,
           "status": "active"
       })
       
       if old_cert:
           self.db.certificates.update_one(
               {"_id": old_cert["_id"]},
               {"$set": {"status": "rotating"}}
           )
       
       # Activate new certificate
       self.db.certificates.update_one(
           {"_id": new_cert_id},
           {"$set": {"status": "active"}}
       )
       
       # Update keystore connection to use new certificate
       self.update_keystore_connection(tenant_id, keystore_type, new_cert_id)
       
       # Keep old certificate for grace period (7 days)
       self.schedule_certificate_cleanup(old_cert["_id"], days=7)
   ```

**Requirements:**
- Certificate renewal starts 30 days before expiry
- New certificates tested before activation
- Certificate rotation happens without service interruption
- Old certificates maintained during grace period

### Story 4: Revoke and Cleanup Certificates
**As the** CMK API Server  
**I want to** revoke certificates when tenants disconnect keystores or certificates are compromised  
**So that** unused certificates don't pose security risks  

**Certificate Revocation Journey:**
1. **Handle Tenant Keystore Disconnection**: I revoke certificates when keystore removed
   ```python
   def handle_keystore_disconnection(tenant_id, keystore_type):
       # Find active certificate for this keystore
       active_cert = self.db.certificates.find_one({
           "tenant_id": tenant_id,
           "keystore_type": keystore_type,
           "status": "active"
       })
       
       if active_cert:
           # Revoke certificate in OpenBao PKI
           self.revoke_certificate_in_pki(active_cert["serial_number"])
           
           # Mark certificate as revoked in database
           self.db.certificates.update_one(
               {"_id": active_cert["_id"]},
               {"$set": {"status": "revoked", "revoked_at": datetime.utcnow()}}
           )
   ```

2. **Revoke Certificate in OpenBao PKI**: I call OpenBao to revoke certificate
   ```python
   def revoke_certificate_in_pki(serial_number):
       try:
           # Revoke certificate in OpenBao PKI
           vault_client.secrets.pki.revoke_certificate(
               serial_number=serial_number,
               mount_point="pki_client_certs"
           )
           
           log_info(f"Certificate {serial_number} revoked in OpenBao PKI")
           return True
           
       except Exception as e:
           log_error(f"Failed to revoke certificate {serial_number}: {e}")
           return False
   ```

3. **Clean Up Certificate Files**: I securely delete certificate data
   ```python
   def cleanup_certificate_data(certificate_id):
       cert_record = self.db.certificates.find_one({"_id": certificate_id})
       
       if cert_record:
           # Securely delete private key from memory/storage
           secure_delete(cert_record["encrypted_private_key"])
           
           # Remove certificate files from filesystem
           cert_files = [
               f"/certs/{cert_record['tenant_id']}-{cert_record['keystore_type']}.crt",
               f"/private/{cert_record['tenant_id']}-{cert_record['keystore_type']}.key"
           ]
           
           for file_path in cert_files:
               if os.path.exists(file_path):
                   secure_file_delete(file_path)
           
           # Update database record
           self.db.certificates.update_one(
               {"_id": certificate_id},
               {"$set": {"status": "cleaned_up", "cleaned_at": datetime.utcnow()}}
           )
   ```

**Requirements:**
- Certificate revocation completes within 1 minute
- Revoked certificates added to CRL in OpenBao PKI  
- Certificate data securely deleted from all locations
- Revocation events logged for audit compliance

## Technical Architecture

### CMK API Server Certificate Plugin Architecture:
```
CMK API Server
├── Certificate Plugin Manager
│   ├── OpenBao PKI Client
│   ├── Certificate Request Builder
│   ├── Certificate Storage Manager
│   └── Certificate Lifecycle Manager
├── Encrypted Certificate Database
├── Certificate Renewal Scheduler
└── Keystore Connection Manager

Platform Mesh Integration
├── OpenBao PKI Service (platform-services namespace)
├── Kubernetes Service Account Authentication
├── Service Discovery for OpenBao endpoint
└── Network Policies for secure communication

External Keystore Integration
├── Thales Luna HSM (mTLS with generated certificates)
├── AWS KMS (mTLS with generated certificates)  
├── Fortanix DSM (mTLS with generated certificates)
└── Other Keystores (mTLS authentication)
```

### Certificate Request Flow:
```
Tenant configures keystore connection
        ↓
CMK API Server receives keystore configuration
        ↓
CMK API Server prepares certificate request for keystore type
        ↓
CMK API Server calls OpenBao PKI to generate certificate
        ↓
OpenBao PKI signs certificate and returns bundle
        ↓
CMK API Server encrypts and stores certificate securely
        ↓
CMK API Server configures keystore connection with certificate
        ↓
CMK API Server tests mTLS connection to external keystore
        ↓
CMK API Server schedules certificate renewal
```

## Requirements

### Integration Requirements:
- **REQ-001**: Successfully authenticate with OpenBao PKI using Kubernetes service account
- **REQ-002**: Generate x509 client certificates for different keystore types
- **REQ-003**: Configure mTLS connections to external keystores using generated certificates
- **REQ-004**: Automatically renew certificates before expiration

### Performance Requirements:
- **REQ-005**: Certificate generation completes within 2 minutes
- **REQ-006**: Support 100+ certificate operations per hour
- **REQ-007**: Certificate renewal without keystore connection interruption
- **REQ-008**: Certificate revocation completes within 1 minute

### Security Requirements:
- **REQ-009**: Private keys encrypted using tenant-specific encryption keys
- **REQ-010**: Certificate data never logged in plaintext
- **REQ-011**: Revoked certificates properly handled in PKI CRL
- **REQ-012**: Complete audit trail for all certificate operations

## Success Criteria

### Integration Success:
- ✅ CMK API Server successfully connects to OpenBao PKI service
- ✅ Certificate generation works for all supported keystore types  
- ✅ Generated certificates successfully authenticate to external keystores
- ✅ Certificate renewal happens automatically without service disruption

### Security Success:
- ✅ Private keys remain encrypted and secure throughout lifecycle
- ✅ Certificate revocation properly invalidates access
- ✅ mTLS connections established successfully with all keystore types
- ✅ Complete audit trail maintained for compliance

### Operational Success:
- ✅ Certificate operations meet performance requirements
- ✅ Automatic renewal prevents certificate expiry issues
- ✅ Certificate lifecycle management works reliably
- ✅ Error handling and recovery procedures work correctly

## Business Value

- **Automated Security**: Seamless mTLS authentication to external keystores
- **Scalability**: Automated certificate management for thousands of tenants
- **Compliance**: Complete certificate lifecycle audit trails
- **Reliability**: Automatic renewal prevents service disruptions
- **Integration**: Leverages existing OpenBao PKI infrastructure
- **Operational Efficiency**: Eliminates manual certificate management overhead