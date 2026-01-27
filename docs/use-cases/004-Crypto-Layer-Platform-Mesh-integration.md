---
authors:
  - Aysan
---

## Persona
**Platform Mesh Administrator** - An infrastructure administrator responsible for deploying and managing OpenKCM's Crypto Layer services within the Platform Mesh infrastructure. They ensure the Crypto Service is properly deployed, configured, and integrated with the mesh for tenant services like MongoDB.

## Overview

As a Platform Mesh Administrator, I need to deploy OpenKCM's Crypto Layer service within the Platform Mesh infrastructure so that tenant services (like MongoDB, PostgreSQL, microservices) can request encryption keys for data protection. The Crypto Layer must be highly available, scalable, and integrated with Platform Mesh's service discovery and networking.

## User Stories

### Story 1: Deploy Crypto Service in Platform Mesh
**As a** Platform Mesh Administrator  
**I want to** deploy the OpenKCM Crypto Service within Platform Mesh  
**So that** tenant services can access encryption key management capabilities  

**Deployment Journey:**
1. **Infrastructure Preparation**: I prepare the Platform Mesh environment
   - Kubernetes cluster with sufficient resources allocated
   - Network policies configured for Crypto Service communication
   - Storage provisioned for key metadata and configuration

2. **Crypto Service Deployment**: I deploy the Crypto Service components
   ```yaml
   # crypto-service-deployment.yaml
   apiVersion: apps/v1
   kind: Deployment
   metadata:
     name: openkcm-crypto-service
     namespace: openkcm-system
   spec:
     replicas: 3
     selector:
       matchLabels:
         app: openkcm-crypto-service
     template:
       spec:
         containers:
         - name: crypto-service
           image: openkcm/crypto-service:latest
           ports:
           - containerPort: 5696  # KMIP port
           - containerPort: 8443  # HTTPS management
           env:
           - name: POSTGRES_CONNECTION
             value: "postgresql://crypto-db:5432/openkcm"
           - name: TENANT_ISOLATION_ENABLED
             value: "true"
   ```

3. **Service Registration**: I register Crypto Service with Platform Mesh
   ```yaml
   # crypto-service.yaml  
   apiVersion: v1
   kind: Service
   metadata:
     name: openkcm-crypto-service
     namespace: openkcm-system
     annotations:
       platform-mesh.io/service-type: "encryption-service"
       platform-mesh.io/discovery-enabled: "true"
   spec:
     selector:
       app: openkcm-crypto-service
     ports:
     - name: kmip
       port: 5696
       protocol: TCP
     - name: management
       port: 8443
       protocol: TCP
   ```

4. **Health Checks**: I configure service health monitoring
   - Liveness probes for container health
   - Readiness probes for service availability
   - Platform Mesh health check integration

**Requirements:**
- Crypto Service deployed with high availability (3+ replicas)
- Service registered in Platform Mesh service discovery
- Health checks configured and reporting healthy status
- Network policies allow tenant service communication

### Story 2: Configure Crypto Service Integration with Platform Mesh
**As a** Platform Mesh Administrator  
**I want to** configure the Crypto Service for seamless Platform Mesh integration  
**So that** tenant services can discover and authenticate with the Crypto Service  

**Configuration Journey:**
1. **Service Discovery Setup**: I configure Platform Mesh service discovery
   ```yaml
   # service-discovery-config.yaml
   apiVersion: platform-mesh.io/v1
   kind: ServiceDiscovery
   metadata:
     name: openkcm-crypto-service
   spec:
     serviceName: openkcm-crypto-service
     namespace: openkcm-system
     endpoints:
       - protocol: KMIP
         port: 5696
         healthCheck: /health
       - protocol: HTTPS
         port: 8443
         healthCheck: /management/health
     tags:
       - "encryption"
       - "key-management" 
       - "kmip"
   ```

2. **Certificate Management**: I set up mTLS certificate provisioning
   ```yaml
   # crypto-service-certs.yaml
   apiVersion: cert-manager.io/v1
   kind: Certificate
   metadata:
     name: crypto-service-tls
     namespace: openkcm-system
   spec:
     secretName: crypto-service-tls
     issuerRef:
       name: platform-mesh-ca
       kind: ClusterIssuer
     dnsNames:
     - openkcm-crypto-service.openkcm-system.svc.cluster.local
     - crypto-service.platform-mesh.internal
   ```

3. **Network Policies**: I configure secure network access
   ```yaml
   # crypto-service-network-policy.yaml
   apiVersion: networking.k8s.io/v1
   kind: NetworkPolicy
   metadata:
     name: crypto-service-access
     namespace: openkcm-system
   spec:
     podSelector:
       matchLabels:
         app: openkcm-crypto-service
     ingress:
     - from:
       - namespaceSelector:
           matchLabels:
             platform-mesh.io/tenant: "true"
       ports:
       - protocol: TCP
         port: 5696  # KMIP access for tenant services
   ```

4. **Configuration Validation**: I verify the integration
   - Service discoverable through Platform Mesh DNS
   - mTLS certificates properly provisioned and valid
   - Network policies allow authorized tenant service access
   - Crypto Service registers successfully with service mesh

**Requirements:**
- Service discoverable via Platform Mesh service registry
- mTLS certificates automatically provisioned and rotated
- Network policies enforce secure tenant service access
- Integration validated through connectivity tests

### Story 3: Configure Multi-Tenant Isolation for Crypto Service
**As a** Platform Mesh Administrator  
**I want to** configure tenant isolation within the Crypto Service  
**So that** each tenant's encryption keys are completely isolated from other tenants  

**Isolation Configuration Journey:**
1. **Tenant Database Setup**: I configure PostgreSQL with Row-Level Security
   ```sql
   -- Enable Row-Level Security for tenant isolation
   ALTER TABLE key_metadata ENABLE ROW LEVEL SECURITY;
   
   -- Create tenant isolation policy
   CREATE POLICY tenant_isolation ON key_metadata
     USING (tenant_id = current_setting('app.current_tenant'));
   
   -- Create tenant-specific database roles
   CREATE ROLE tenant_a_crypto;
   GRANT SELECT, INSERT, UPDATE ON key_metadata TO tenant_a_crypto;
   ```

2. **Tenant Certificate Provisioning**: I set up tenant-specific mTLS certificates
   ```yaml
   # tenant-certificates.yaml
   apiVersion: platform-mesh.io/v1
   kind: TenantCertificate
   metadata:
     name: tenant-certificates
   spec:
     tenants:
     - name: "tenant-a"
       services:
         - mongodb-cluster-tenant-a
         - redis-cluster-tenant-a
       certificateTemplate:
         subject:
           organizationalUnit: ["tenant-a"]
         extensions:
           tenantId: "tenant-a"
   ```

3. **Tenant Routing Configuration**: I configure request routing based on certificates
   ```yaml
   # tenant-routing.yaml
   apiVersion: platform-mesh.io/v1
   kind: TenantRouting
   metadata:
     name: crypto-service-routing
   spec:
     service: openkcm-crypto-service
     routing:
       - match:
           certificate:
             subject:
               organizationalUnit: "tenant-a"
         route:
           tenant: "tenant-a"
           database: "tenant_a_keys"
   ```

4. **Isolation Validation**: I test tenant boundaries
   - Verify tenant-a services can only access tenant-a keys
   - Confirm cross-tenant requests are blocked
   - Validate certificate-based tenant identification
   - Test database-level isolation with RLS policies

**Requirements:**
- PostgreSQL RLS enforces database-level tenant isolation
- mTLS certificates identify tenant context for each request
- Crypto Service routes requests to correct tenant data
- Cross-tenant access attempts are blocked and logged

### Story 4: Monitor and Scale Crypto Service Deployment
**As a** Platform Mesh Administrator  
**I want to** monitor Crypto Service performance and scale based on demand  
**So that** encryption key operations maintain performance SLAs across all tenants  

**Monitoring and Scaling Journey:**
1. **Metrics Collection**: I configure comprehensive monitoring
   ```yaml
   # crypto-service-monitoring.yaml
   apiVersion: monitoring.coreos.com/v1
   kind: ServiceMonitor
   metadata:
     name: crypto-service-metrics
   spec:
     selector:
       matchLabels:
         app: openkcm-crypto-service
     endpoints:
     - port: management
       path: /metrics
       interval: 30s
   ```

2. **Performance Dashboards**: I set up operational visibility
   - Key operation latency (p50, p95, p99)
   - Request rate by tenant and operation type
   - Error rates and failure modes
   - Resource utilization (CPU, memory, network)
   - Database connection pool status

3. **Auto-scaling Configuration**: I configure horizontal pod autoscaling
   ```yaml
   # crypto-service-hpa.yaml
   apiVersion: autoscaling/v2
   kind: HorizontalPodAutoscaler
   metadata:
     name: crypto-service-hpa
   spec:
     scaleTargetRef:
       apiVersion: apps/v1
       kind: Deployment
       name: openkcm-crypto-service
     minReplicas: 3
     maxReplicas: 20
     metrics:
     - type: Pods
       pods:
         metric:
           name: kmip_requests_per_second
         target:
           type: AverageValue
           averageValue: "1000"
   ```

4. **Alerting Setup**: I configure operational alerts
   - High latency alerts (>100ms for key operations)
   - Error rate alerts (>1% failure rate)
   - Resource exhaustion alerts
   - Tenant isolation breach alerts
   - Certificate expiration warnings

**Requirements:**
- Comprehensive metrics collection and visualization
- Auto-scaling based on key operation load
- Proactive alerting for performance and security issues
- Operational runbooks for common scenarios

## Technical Architecture

### Crypto Service Deployment Architecture:
```
Platform Mesh Cluster
├── openkcm-system namespace
│   ├── Crypto Service Pods (3+ replicas)
│   ├── PostgreSQL Database (RLS enabled)
│   ├── Certificate Manager
│   └── Monitoring Stack
├── Tenant Namespaces
│   ├── MongoDB Services (with tenant certificates)
│   ├── Other Database Services
│   └── Microservices
└── Platform Mesh Infrastructure
    ├── Service Discovery
    ├── Load Balancing
    ├── Network Policies
    └── Certificate Authority
```

### Service Communication Flow:
```
Tenant Service (MongoDB) → Platform Mesh Service Discovery
                        ↓
                    Crypto Service Endpoint Resolution
                        ↓
                    mTLS Connection Establishment
                        ↓
                    KMIP Key Request (with tenant certificate)
                        ↓
                    Tenant Validation & Key Derivation
                        ↓
                    Encrypted Key Response
```

## Requirements

### Deployment Requirements:
- **REQ-001**: Crypto Service must deploy with high availability (3+ replicas)
- **REQ-002**: Service must register with Platform Mesh service discovery
- **REQ-003**: PostgreSQL database must be configured with Row-Level Security
- **REQ-004**: mTLS certificates must be automatically provisioned and rotated

### Performance Requirements:
- **REQ-005**: Service must handle 10,000+ key requests per second
- **REQ-006**: Key operations must complete within 50ms (95th percentile)
- **REQ-007**: Auto-scaling must respond to load within 2 minutes
- **REQ-008**: Service startup time must be under 60 seconds

### Security Requirements:
- **REQ-009**: Tenant isolation must be enforced at network and database levels
- **REQ-010**: All service communications must use mTLS encryption
- **REQ-011**: Certificate-based tenant identification must be validated
- **REQ-012**: Security policies must prevent cross-tenant access

### Operational Requirements:
- **REQ-013**: Comprehensive monitoring and alerting must be configured
- **REQ-014**: Service must integrate with Platform Mesh logging infrastructure
- **REQ-015**: Deployment must support rolling updates with zero downtime
- **REQ-016**: Backup and disaster recovery procedures must be documented

## Success Criteria

### Deployment Success:
- ✅ Crypto Service deployed and running with high availability
- ✅ Service registered and discoverable through Platform Mesh
- ✅ Multi-tenant isolation configured and validated
- ✅ Network policies and certificates properly configured

### Integration Success:
- ✅ Tenant services (MongoDB, etc.) can discover and connect to Crypto Service
- ✅ KMIP protocol communication working correctly
- ✅ mTLS authentication and tenant identification working
- ✅ Database-level tenant isolation enforced

### Operational Success:
- ✅ Monitoring dashboards show service health and performance
- ✅ Auto-scaling responds appropriately to load changes
- ✅ Alerts fire correctly for various failure scenarios
- ✅ Service meets performance SLAs under production load

## Business Value

- **Scalable Encryption**: Centralized key management scales across all tenant services
- **Operational Efficiency**: Automated deployment and management through Platform Mesh
- **Security Compliance**: Proper tenant isolation and audit trails for regulatory requirements
- **Developer Experience**: Transparent integration allows services to add encryption easily
- **Cost Optimization**: Shared infrastructure with auto-scaling optimizes resource usage
- **Platform Integration**: Native Platform Mesh integration provides seamless service discovery