# Recipe: WFPS Runtime Foundation with PFS and BAI

## Overview

**Recipe Name**: Workflow Process Service Runtime Foundation with Process Federation Server and Business Automation Insights  
**Template File**: `cp4ba-cr-ref-wfps-pfs-bai-foundation.yaml`  
**Properties File**: `env1-runtime-wfps-pfs-bai.properties`  
**Purpose**: Foundation-only deployment for Workflow Process Service runtime with monitoring infrastructure

---

## Deployment Summary

### Patterns
- `foundation` - Base infrastructure services
- `workflow-process-service` - WFPS pattern (foundation only)

### Optional Components
- `bai` - Business Automation Insights
- `kafka` - Event streaming platform (required for BAI)
- `opensearch` - Analytics data storage and visualization

### License Type
- Production (configurable via `CP4BA_INST_LICENSE_TYPE`)

---

## Architecture

### Component Topology

```
┌─────────────────────────────────────────────────────────────┐
│                    User Access Layer                        │
│  - Zen UI (administration)                                  │
│  - BAI Dashboards (monitoring)                              │
│  - PFS UI (federated tasks) [separate deployment]           │
└─────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────┐
│                   Analytics & Events                        │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐       │
│  │    Kafka     │  │    Flink     │  │  OpenSearch  │       │
│  │  (Events)    │  │ (Processing) │  │  (Storage)   │       │
│  └──────────────┘  └──────────────┘  └──────────────┘       │
└─────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────┐
│                   Foundation Services                       │
│  ┌──────────────────────────────────────────────┐           │
│  │         Resource Registry                    │           │
│  │  - Service registration                      │           │
│  │  - Service discovery                         │           │
│  │  - Configuration management                  │           │
│  └──────────────────────────────────────────────┘           │
│  ┌──────────────────────────────────────────────┐           │
│  │      User Management Service (UMS)           │           │
│  │  - User authentication                       │           │
│  │  - User authorization                        │           │
│  │  - User profile management                   │           │
│  └──────────────────────────────────────────────┘           │
│  ┌──────────────────────────────────────────────┐           │
│  │      IAM (via CPFS)                          │           │
│  │  - Identity management                       │           │
│  │  - Access control                            │           │
│  │  - SSO integration                           │           │
│  └──────────────────────────────────────────────┘           │
│  ┌──────────────────────────────────────────────┐           │
│  │      Zen UI Framework                        │           │
│  │  - Common UI shell                           │           │
│  │  - Navigation                                │           │
│  │  - Administration                            │           │
│  └──────────────────────────────────────────────┘           │
└─────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────┐
│                   Infrastructure                            │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐       │
│  │  PostgreSQL  │  │   OpenLDAP   │  │   Storage    │       │
│  │  (Database)  │  │   (Users)    │  │  (RWX/RWO)   │       │
│  └──────────────┘  └──────────────┘  └──────────────┘       │
└─────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────┐
│              External WFPS Instances                        │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐       │
│  │   WFPS 1     │  │   WFPS 2     │  │   WFPS N     │       │
│  │ (Separate CR)│  │ (Separate CR)│  │ (Separate CR)│       │
│  └──────────────┘  └──────────────┘  └──────────────┘       │
│         ↓                  ↓                  ↓             │
│         └──────────────────┴──────────────────┘             │
│                            ↓                                │
│              ┌──────────────────────────┐                   │
│              │ Process Federation Server│                   │
│              │    (Separate CR)         │                   │
│              └──────────────────────────┘                   │
└─────────────────────────────────────────────────────────────┘
```

---

## Understanding Foundation-Only Deployment

### What is Foundation-Only?

This recipe deploys **only the foundation services** without any WFPS runtime instances. It provides:

1. **Shared Infrastructure**: Common services for multiple WFPS instances
2. **Monitoring Platform**: BAI for analytics across all WFPS instances
3. **Federation Support**: Foundation for PFS deployment
4. **IAM Services**: Centralized identity and access management

### Why Foundation-Only?

**Benefits**:
- **Separation of Concerns**: Foundation services separate from runtime
- **Shared Services**: Multiple WFPS instances share foundation
- **Independent Scaling**: Scale foundation and runtimes independently
- **Simplified Management**: Manage foundation separately
- **Cost Optimization**: Single foundation for multiple runtimes

**Use Cases**:
- Microservices architecture with multiple WFPS instances
- Multi-tenant WFPS deployments
- Separate development/test/production WFPS environments
- Large-scale WFPS deployments

### Architecture Pattern

```
Foundation Namespace (this recipe):
├── Foundation Services
├── BAI (monitoring all WFPS)
├── Kafka
└── OpenSearch

WFPS Namespace 1:
└── WFPS Instance 1 (uses foundation)

WFPS Namespace 2:
└── WFPS Instance 2 (uses foundation)

WFPS Namespace N:
└── WFPS Instance N (uses foundation)

PFS Namespace:
└── PFS (federates all WFPS)
```

---

## CP4BA Capabilities Used

### 1. Foundation Services

**Resource Registry**:
```yaml
resource_registry_configuration:
  replica_size: 1
```

**Features**:
- Service registration and discovery
- Configuration management
- Service health monitoring
- Dynamic service routing

**Configuration**:
```properties
CP4BA_INST_REGISTRY_REPLICA_SIZE="1"
```

**User Management Service (UMS)**:
- User authentication
- User authorization
- Profile management
- Integration with LDAP/IAM

**IAM Integration**:
- Identity management via CPFS
- Role-based access control
- SSO capabilities
- User onboarding

**Zen UI Framework**:
- Common UI shell
- Navigation framework
- Administration console
- Component integration

### 2. Business Automation Insights (BAI)

**Configuration**:
```yaml
bai_configuration:
  business_performance_center:
    all_users_access: true
  event_forwarder:
    enabled: true
```

**Features**:
- **Centralized Monitoring**: Monitor all WFPS instances
- **Real-time Analytics**: Live process metrics
- **Historical Analysis**: Trend analysis across instances
- **Event Processing**: Flink stream processing
- **Data Storage**: OpenSearch for analytics

**Event Sources**:
```properties
CP4BA_INST_BAI_BPMN="true"               # BPMN process events
CP4BA_INST_BAI_BPMN_TIMESERIES="true"    # Time series data
```

**Multi-Instance Support**:
- Collects events from all WFPS instances
- Unified dashboards across instances
- Cross-instance analytics
- Consolidated reporting

### 3. Apache Kafka

**Purpose**: Event streaming platform

**Configuration**:
- Deployed as part of BAI
- Handles events from all WFPS instances
- Provides event buffering and reliability
- Enables event replay

**Multi-Instance Support**:
- Single Kafka cluster for all WFPS
- Topic per WFPS instance
- Centralized event management

### 4. OpenSearch

**Purpose**: Analytics data storage

**Configuration**:
- Stores BAI event data from all WFPS
- Provides unified dashboards
- Enables cross-instance queries
- Supports historical analysis

---

## Database Configuration

### Database Architecture

**Minimal Database Footprint**:

```
PostgreSQL Server: my-postgres-1-for-cp4ba
├── wfps_pfs_bai_production_bts (Business Teams Service)
├── wfps_pfs_bai_production_im (Identity Management)
└── wfps_pfs_bai_production_zen (Zen Services)
```

**Note**: No WFPS runtime databases in this deployment. Each WFPS instance will have its own databases.

### Database Users

| Database | User | Purpose |
|----------|------|---------|
| bts | bts_user | Business Teams Service |
| im | im_user | Identity Management |
| zen | zen_user | Zen Services |

### External WFPS Database Configuration

**Each WFPS instance requires**:
```properties
# Set in WFPS instance properties
CP4BA_INST_DB_WFPS_EXT_1="true"  # Use external databases
```

**WFPS databases** (per instance):
- WFPS database
- BTS database (can be shared)
- IM database (can be shared)
- ZEN database (can be shared)

### Database Server Configuration

```properties
CP4BA_INST_DB_SERVER_TYPE="postgresql"
CP4BA_INST_DB_SERVER_PORT="5432"
CP4BA_INST_DB_STORAGE_SIZE="10Gi"
CP4BA_INST_DB_MAX_POOL_SIZE="100"
CP4BA_INST_DB_MIN_POOL_SIZE="10"
```

**Resource Allocation**:
```properties
CP4BA_INST_DB_REQS_CPU="2000m"
CP4BA_INST_DB_REQS_MEMORY="2048Mi"
CP4BA_INST_DB_LIMITS_CPU="4000m"
CP4BA_INST_DB_LIMITS_MEMORY="8192Mi"
```

---

## LDAP Configuration

### Embedded OpenLDAP

**Configuration**:
```properties
CP4BA_INST_LDAP=true
CP4BA_INST_LDAP_USE_VOLUME=false
```

**LDAP Connection**:
```yaml
ldap_configuration:
  lc_selected_ldap_type: "Custom"
  lc_ldap_server: "openldap.${NAMESPACE}.svc.cluster.local"
  lc_ldap_port: "389"
  lc_ldap_base_dn: "dc=example,dc=com"
  lc_ldap_ssl_enabled: false
```

**Shared LDAP**:
- All WFPS instances use this LDAP
- Centralized user management
- Consistent authentication

### IAM Integration

```properties
CP4BA_INST_IAM=true
CP4BA_INST_IAM_ADMIN_USER="cpadmin"
CP4BA_INST_IAM_ADMIN_GROUP="AdminsGroup"
```

---

## Use Cases

### 1. Microservices WFPS Architecture

**Scenario**: Deploy multiple WFPS instances for different microservices

**Architecture**:
```
Foundation (this recipe):
└── Shared services + BAI

WFPS Instance 1: Order Processing
WFPS Instance 2: Customer Service
WFPS Instance 3: Inventory Management
WFPS Instance 4: Shipping

PFS: Unified task management
```

**Benefits**:
- Service isolation
- Independent scaling
- Shared monitoring
- Centralized management

### 2. Multi-Tenant WFPS Deployment

**Scenario**: Support multiple tenants with separate WFPS instances

**Architecture**:
```
Foundation (this recipe):
└── Shared services + BAI

WFPS Tenant A: Customer A workflows
WFPS Tenant B: Customer B workflows
WFPS Tenant C: Customer C workflows

PFS: Tenant-aware federation
```

**Benefits**:
- Tenant isolation
- Shared infrastructure
- Centralized monitoring
- Cost efficiency

### 3. Environment Separation

**Scenario**: Separate dev/test/prod WFPS environments

**Architecture**:
```
Foundation (this recipe):
└── Shared services + BAI

WFPS Dev: Development workflows
WFPS Test: Testing workflows
WFPS Prod: Production workflows

PFS: Environment-aware federation
```

**Benefits**:
- Environment isolation
- Shared monitoring
- Consistent infrastructure
- Simplified management

### 4. Large-Scale WFPS Deployment

**Scenario**: Deploy many WFPS instances for high volume

**Architecture**:
```
Foundation (this recipe):
└── Shared services + BAI

WFPS Pool 1: Instances 1-10
WFPS Pool 2: Instances 11-20
WFPS Pool 3: Instances 21-30

PFS: Load-balanced federation
```

**Benefits**:
- Horizontal scalability
- Shared monitoring
- Centralized management
- Cost optimization

---

## Deploying WFPS Instances

### WFPS Instance Deployment

**Separate Custom Resource** for each WFPS instance:

```yaml
apiVersion: icp4a.ibm.com/v1
kind: ICP4ACluster
metadata:
  name: wfps-demo-1
  namespace: wfps-instance-1
spec:
  shared_configuration:
    sc_deployment_patterns: "workflow-process-service"
    # Reference foundation services
    foundation_services:
      namespace: cp4ba-wfps-pfs-bai-production
      resource_registry_url: https://...
      ums_url: https://...
```

**Configuration**:
```properties
# WFPS instance properties
CP4BA_INST_CUSTOM_XML_WFPS_CR_NAME_1="wfps-demo-1"
CP4BA_INST_LIBERTY_CUSTOM_XML_SECRET_NAME_WFPS_1="my-liberty-custom-xml-secret-1"
CP4BA_INST_LOMBARDI_CUSTOM_XML_SECRET_NAME_WFPS_1="my-lombardi-custom-xml-secret-1"
```

### WFPS Instance Requirements

**Each WFPS instance needs**:
1. Own namespace (or shared namespace)
2. Own databases
3. Reference to foundation services
4. Process applications deployed
5. Custom XML configuration

---

## PFS Integration

### PFS Deployment

**Separate Custom Resource**:

```yaml
apiVersion: icp4a.ibm.com/v1
kind: ProcessFederationServer
metadata:
  name: pfs
  namespace: cp4ba-pfs
spec:
  pfs_configuration:
    replicas: 1
    federated_systems:
      - name: wfps1
        type: WFPS
        url: https://wfps1-route
        namespace: wfps-instance-1
      - name: wfps2
        type: WFPS
        url: https://wfps2-route
        namespace: wfps-instance-2
      # Add more WFPS instances
```

**Features**:
- Federates all WFPS instances
- Unified task list
- Cross-instance search
- Centralized task management

---

## Deployment Steps

### Prerequisites

1. **OpenShift Cluster**: Version 4.12+
2. **IBM Entitlement Key**: Valid entitlement
3. **Namespace**: Created for foundation
4. **Storage**: File (RWX) and Block (RWO) storage classes

### Installation Process

#### Phase 1: Deploy Foundation

1. **Prepare Configuration**:
   ```bash
   vi env1-runtime-wfps-pfs-bai.properties
   
   # Key settings:
   # CP4BA_INST_DEPL_PATTERNS="foundation,workflow-process-service"
   # CP4BA_INST_OPT_COMPONENTS="bai,kafka,opensearch"
   # CP4BA_INST_CR_TEMPLATE="templates25.0.1/cp4ba-cr-ref-wfps-pfs-foundation.yaml"
   ```

2. **Generate Custom Resource**:
   ```bash
   ./generate-cr.sh env1-runtime-wfps-pfs-bai.properties
   ```

3. **Deploy Infrastructure**:
   ```bash
   ./deploy-database.sh
   ./deploy-ldap.sh
   ```

4. **Apply Foundation CR**:
   ```bash
   oc apply -f ${OUTPUT_FOLDER}/icp4adeploy-cr.yaml -n ${NAMESPACE}
   ```

5. **Monitor Foundation Deployment**:
   ```bash
   oc logs -f -l name=ibm-cp4a-operator -n ${NAMESPACE}
   oc get icp4acluster -n ${NAMESPACE}
   ```

6. **Wait for Foundation to be Ready**:
   ```bash
   oc wait --for=condition=ready pod -l app=resource-registry -n ${NAMESPACE}
   oc wait --for=condition=ready pod -l app=ums -n ${NAMESPACE}
   ```

#### Phase 2: Deploy WFPS Instances

1. **For Each WFPS Instance**:
   ```bash
   # Create namespace
   oc create namespace wfps-instance-1
   
   # Prepare WFPS configuration
   vi wfps-instance-1.properties
   
   # Generate WFPS CR
   ./generate-wfps-cr.sh wfps-instance-1.properties
   
   # Deploy WFPS databases
   ./deploy-wfps-database.sh wfps-instance-1
   
   # Apply WFPS CR
   oc apply -f wfps-instance-1-cr.yaml -n wfps-instance-1
   ```

2. **Monitor WFPS Deployments**:
   ```bash
   oc get pods -n wfps-instance-1
   oc get pods -n wfps-instance-2
   ```

#### Phase 3: Deploy PFS

1. **Prepare PFS Configuration**:
   ```bash
   vi pfs-configuration.properties
   
   # Configure all WFPS instances
   ```

2. **Generate PFS CR**:
   ```bash
   ./generate-pfs-cr.sh pfs-configuration.properties
   ```

3. **Apply PFS CR**:
   ```bash
   oc apply -f pfs-cr.yaml -n cp4ba-pfs
   ```

4. **Configure Federation**:
   ```bash
   # Register all WFPS instances with PFS
   ```

### Verification

1. **Check Foundation**:
   ```bash
   oc get pods -n ${FOUNDATION_NAMESPACE}
   # Verify all foundation services running
   ```

2. **Check WFPS Instances**:
   ```bash
   oc get pods -n wfps-instance-1
   oc get pods -n wfps-instance-2
   # Verify all WFPS instances running
   ```

3. **Check PFS**:
   ```bash
   oc get pods -n cp4ba-pfs
   # Verify PFS running and federating
   ```

4. **Check BAI**:
   ```bash
   # Access BAI dashboards
   # Verify events from all WFPS instances
   ```

---

## Resource Requirements

WARNING: The following are general indications and should be understood as a starting point from which to increase or decrease consumption based on the use case and related non-functional requirements.

### Foundation Resources

**For Small Profile**:
- **CPU**: 10-15 cores
- **Memory**: 32-48 GB
- **Storage**: 100-150 GB

### Component Resource Allocation

| Component | Replicas | CPU Request | Memory Request | CPU Limit | Memory Limit |
|-----------|----------|-------------|----------------|-----------|--------------|
| Resource Registry | 1 | 500m | 1024Mi | 1000m | 2048Mi |
| UMS | 1 | 500m | 1024Mi | 1000m | 2048Mi |
| Zen | 1 | 500m | 1024Mi | 1000m | 2048Mi |
| Flink | 1 | 1000m | 2048Mi | 2000m | 4096Mi |
| Kafka | 1 | 500m | 1024Mi | 1000m | 2048Mi |
| OpenSearch | 1 | 1000m | 2048Mi | 2000m | 4096Mi |
| PostgreSQL | 1 | 2000m | 2048Mi | 4000m | 8192Mi |
| OpenLDAP | 1 | 100m | 256Mi | 200m | 512Mi |

### Per-WFPS Instance Resources

**Each WFPS instance requires**:
- **CPU**: 2-4 cores
- **Memory**: 4-8 GB
- **Storage**: 10-20 GB

---

## Maintenance and Operations

### Backup Strategy

**Foundation Backups**:
```bash
# Backup foundation databases
pg_dump bts > bts-backup.sql
pg_dump im > im-backup.sql
pg_dump zen > zen-backup.sql

# Backup foundation CR
oc get icp4acluster -o yaml > foundation-backup.yaml

# Backup BAI configuration
# BAI uses OpenSearch for data
```

**WFPS Instance Backups**:
```bash
# Backup each WFPS instance separately
# Each instance has own databases and configuration
```

### Monitoring

**Foundation Monitoring**:
- Monitor foundation service health
- Track resource utilization
- Review BAI metrics
- Check Kafka performance

**Cross-Instance Monitoring**:
- BAI dashboards show all WFPS instances
- Unified metrics across instances
- Cross-instance analytics
- Consolidated reporting

### Scaling

**Scale Foundation**:
```yaml
resource_registry_configuration:
  replica_size: 2  # Scale for HA
```

**Scale WFPS Instances**:
- Add more WFPS instances as needed
- Scale individual instances independently
- Foundation supports many instances

**Scale BAI**:
```yaml
bai_configuration:
  flink:
    replicas: 2
  kafka:
    replicas: 3
  opensearch:
    replicas: 3
```

---

## Best Practices

### Foundation Management

1. **High Availability**: Deploy foundation with HA
2. **Monitoring**: Comprehensive monitoring of foundation
3. **Backup**: Regular backups of foundation
4. **Updates**: Update foundation separately from WFPS
5. **Security**: Secure foundation services

### WFPS Instance Management

1. **Isolation**: Keep instances isolated
2. **Naming**: Use consistent naming convention
3. **Documentation**: Document each instance purpose
4. **Monitoring**: Monitor via BAI
5. **Lifecycle**: Manage instance lifecycle independently

### BAI Configuration

1. **Event Collection**: Configure event collection per instance
2. **Dashboards**: Create dashboards per instance and aggregate
3. **Retention**: Configure data retention policies
4. **Performance**: Monitor BAI performance
5. **Alerts**: Set up alerts for critical metrics

---

## Troubleshooting

### Common Issues

**1. WFPS Cannot Connect to Foundation**:
- Verify foundation services are running
- Check network connectivity
- Review service URLs
- Test authentication

**2. BAI Not Receiving Events from WFPS**:
- Verify Kafka connectivity
- Check event emitter configuration
- Review Flink logs
- Test event generation

**3. PFS Cannot Federate WFPS**:
- Verify WFPS URLs
- Check authentication
- Review PFS logs
- Test WFPS REST API

**4. Performance Issues**:
- Check foundation resource utilization
- Monitor database performance
- Review Kafka performance
- Analyze BAI metrics

---

## Related Recipes

- **[WFPS Authoring](recipe-authoring-wfps-pfs-bai.md)**: WFPS development environment
- **[BAW Runtime with Double PFS](recipe-runtime-baw-double-pfs.md)**: Multi-server BAW with PFS
- **[BAW Runtime with BAI](recipe-runtime-baw-bai.md)**: BAW runtime with monitoring

---

## References

- [CP4BA 25.0.1 Documentation](https://www.ibm.com/docs/en/cloud-paks/cp-biz-automation/25.0.1)
- [WFPS Documentation](https://www.ibm.com/docs/en/cloud-paks/cp-biz-automation/25.0.1?topic=ipd-installing-cp4ba-workflow-process-service-runtime-production-deployment)
- [Foundation Services](https://www.ibm.com/docs/en/cloud-paks/cp-biz-automation/25.0.1?topic=parameters-shared-configuration)
- [PFS Documentation](https://www.ibm.com/docs/en/cloud-paks/cp-biz-automation/25.0.1?topic=deployments-installing-cp4ba-process-federation-server-production-deployment)

---

*Last Updated: 2026-06-23*