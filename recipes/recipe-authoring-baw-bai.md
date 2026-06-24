# Recipe: BAW Authoring with BAI

## Overview

**Recipe Name**: BAW Authoring with Business Automation Insights  
**Template File**: `cp4ba-cr-ref-authoring-baw-bai.yaml`  
**Properties File**: `env1-authoring-baw-bai.properties`  
**Purpose**: Full Business Automation Workflow authoring environment with real-time analytics and AI capabilities

---

## Deployment Summary

### Patterns
- `foundation` - Base infrastructure services
- `workflow` - Business Automation Workflow (authoring mode)
- `application` - Business Automation Studio

### Optional Components
- `baw_authoring` - BAW authoring capabilities (Process Designer, Case Builder)
- `bas` - Business Automation Studio
- `bai` - Business Automation Insights
- `kafka` - Event streaming platform (required for BAI)
- `workflow_assistant` - AI-powered workflow assistance (optional, requires GenAI)
- `workplace_assistant` - AI-powered workplace assistance (optional, requires GenAI)

### License Type
- Production or Non-Production (configurable via `CP4BA_INST_LICENSE_TYPE`)

---

## Architecture

### Component Topology

```
┌─────────────────────────────────────────────────────────────┐
│                    User Access Layer                        │
│  - Process Portal (task management)                         │
│  - Process Designer (BPMN authoring)                        │
│  - Case Builder (case management design)                    │
│  - Business Automation Navigator (content UI)               │
│  - Business Automation Studio (low-code development)        │
│  - BAI Dashboards (analytics and monitoring)                │
└─────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────┐
│                   Application Services                      │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐       │
│  │ BAW Authoring│  │     BAS      │  │     CPE      │       │
│  │   Server     │  │   Playback   │  │  (Content)   │       │
│  └──────────────┘  └──────────────┘  └──────────────┘       │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐       │
│  │   Navigator  │  │  App Engine  │  │   GraphQL    │       │
│  └──────────────┘  └──────────────┘  └──────────────┘       │
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
│  - Resource Registry                                        │
│  - User Management Service (UMS)                            │
│  - IAM (via CPFS)                                           │
│  - Zen UI Framework                                         │
└─────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────┐
│                   Infrastructure                            │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐       │
│  │  PostgreSQL  │  │   OpenLDAP   │  │   Storage    │       │
│  │  (Database)  │  │   (Users)    │  │  (RWX/RWO)   │       │
│  └──────────────┘  └──────────────┘  └──────────────┘       │
└─────────────────────────────────────────────────────────────┘
```

---

## CP4BA Capabilities Used

### 1. Business Automation Workflow (BAW) - Authoring Mode

**Configuration**:
```yaml
baw_configuration:
  - name: instance1
    capabilities: "workflow"
```

**Features Enabled**:
- **Process Designer**: Web-based BPMN 2.0 process modeling
- **Case Builder**: Dynamic case management design
- **Process Portal**: Task management and monitoring
- **Toolkit Development**: Reusable process components
- **Integration Designer**: Service integration authoring

**Databases**:
- Process Server Database (BPMDB): `${ENV}_baw_1`
- Design Object Store (DOS): `${ENV}_dos`
- Target Object Store (TOS): `${ENV}_tos`
- Document Object Store (DOCS): `${ENV}_docs`

**Key Configuration Parameters**:
```properties
CP4BA_INST_BAW_BPM_ONLY=false  # Includes Case management
CP4BA_INST_BAWAUTHORING_STORAGE_SIZE="20Gi"
CP4BA_INST_BAWAUTHORING_TASK_PRIORITIZATION_SERVICE_TOGGLE="true"
```

### 2. Business Automation Studio (BAS)

**Configuration**:
```yaml
bastudio_configuration:
  - name: bas
    replicas: 1
```

**Features Enabled**:
- **Application Designer**: Low-code application development
- **Playback Server**: Testing environment for applications
- **Decision Designer**: Business rules authoring
- **Workflow Designer**: Simplified workflow creation
- **Data Modeling**: Entity and relationship design

**Databases**:
- Playback Database (PBK): `${ENV}_appdb`
- Application Engine Database (AE): `${ENV}_aaedb`
- Automation Workstream Services (AWS): `${ENV}_awsdb`

**Resource Configuration**:
```properties
CP4BA_INST_BAS_1_REPLICAS="1"
CP4BA_INST_BAS_1_LIMITS_CPU="5000m"
CP4BA_INST_BAS_1_LIMITS_MEMORY="3096Mi"
CP4BA_INST_BAS_1_LOGS_TRACE="*=info"
```

### 3. FileNet Content Manager (FNCM)

**Configuration**:
```yaml
ecm_configuration:
  cpe:
    replica_count: 1
  navigator:
    replica_count: 1
  graphql:
    enabled: true
```

**Components**:
- **Content Platform Engine (CPE)**: Core content repository
- **Business Automation Navigator**: Web UI for content access
- **GraphQL API**: Modern API for content operations

**Object Stores**:
1. **Global Configuration Data (GCD)**: System configuration
2. **Design Object Store (DOS)**: Case designs and templates
3. **Target Object Store (TOS)**: Case runtime data
4. **Document Object Store (DOCS)**: Document storage
5. **Custom Object Stores**: Application-specific content (OS1, OS2)
6. **Content Object Store**: General content storage
7. **Case History Object Store (CHOS)**: Case audit trail

**Resource Configuration**:
```properties
CP4BA_INST_CPE_REPLICAS="1"
CP4BA_INST_CPE_RES_LIMITS_CPU="2"
CP4BA_INST_CPE_RES_LIMITS_MEM="6144Mi"
```

### 4. Business Automation Insights (BAI)

**Configuration**:
```yaml
bai_configuration:
  business_performance_center:
    all_users_access: true
  event_forwarder:
    enabled: true
```

**Features Enabled**:
- **Real-time Analytics**: Live process monitoring
- **Business Performance Center**: Dashboards and reports
- **Process Mining**: Process discovery and analysis
- **Event Processing**: Apache Flink stream processing
- **Data Storage**: OpenSearch for analytics data

**Event Sources Configured**:
```properties
CP4BA_INST_BAI_BPC_WORKFORCE="true"      # Workforce analytics
CP4BA_INST_BAI_BPC_ALLUSER="true"        # All users access
CP4BA_INST_BAI_BPMN="true"               # BPMN process events
CP4BA_INST_BAI_BPMN_TIMESERIES="true"    # Time series data
CP4BA_INST_BAI_CONTENT="true"            # Content events
CP4BA_INST_BAI_ICM="true"                # Case management events
CP4BA_INST_BAI_NAVIGATOR="true"          # Navigator events
```

**Flink Configuration**:
```properties
CP4BA_INST_BAI_FLINK_ROUTE="true"
CP4BA_INST_BAI_FLINK_ADDITIONAL_TASK_MGRS=1
CP4BA_INST_BAI_END_AGGR_DELAY=10000
```

### 5. Apache Kafka

**Purpose**: Event streaming platform for BAI

**Configuration**:
- Deployed as part of BAI
- Handles event ingestion from all sources
- Provides event buffering and reliability

### 6. OpenSearch

**Purpose**: Analytics data storage and visualization

**Configuration**:
- Stores BAI event data
- Provides Kibana-like dashboards
- Enables custom analytics queries

### 7. AI Assistants (Optional)

**Workflow Assistant**:
```properties
CP4BA_RUN_AUTHORING_AGENT="${CP4BA_INST_GENAI_ENABLED}"
```

**Workplace Assistant**:
```properties
CP4BA_RUN_WORKPLACE_AGENT="${CP4BA_INST_GENAI_ENABLED}"
```

**GenAI Configuration** (requires Watson X):
```properties
CP4BA_INST_BAS_GENAI_ENABLED="false"  # Set to true to enable
CP4BA_INST_BAS_GENAI_WX_APIKEY=""     # Watson X API key
CP4BA_INST_BAS_GENAI_WX_PRJ_ID=""     # Watson X project ID
CP4BA_INST_BAS_GENAI_WX_URL_PROVIDER="https://us-south.ml.cloud.ibm.com"
```

---

## Database Configuration

### Database Architecture

**Single PostgreSQL Server** with multiple databases:

```
PostgreSQL Server: my-postgres-1-for-cp4ba
├── baw_auth_bai_baw_1 (BAW Process Server)
├── baw_auth_bai_dos (Design Object Store)
├── baw_auth_bai_tos (Target Object Store)
├── baw_auth_bai_docs (Document Object Store)
├── baw_auth_bai_gcd (Global Configuration Data)
├── baw_auth_bai_os1 (Custom Object Store 1)
├── baw_auth_bai_os2 (Custom Object Store 2)
├── baw_auth_bai_content (Content Object Store)
├── baw_auth_bai_chos (Case History Object Store)
├── baw_auth_bai_icndb (Navigator)
├── baw_auth_bai_appdb (Playback/Application)
├── baw_auth_bai_aaedb (Application Engine)
└── baw_auth_bai_awsdb (Automation Workstream Services)
```

### Database Users

Each database has a dedicated user (following CP4BA best practices):

| Database | User | Purpose |
|----------|------|---------|
| baw_1 | bawadmin | BAW Process Server |
| dos | bawdos | Design Object Store |
| tos | bawtos | Target Object Store |
| docs | bawdocs | Document Object Store |
| gcd | gcd | Global Configuration Data |
| os1 | os | Custom Object Store |
| content | content | Content Object Store |
| chos | chos | Case History Object Store |
| icndb | icn | Navigator |
| appdb | pbk | Playback/Application |
| aaedb | app | Application Engine |
| awsdb | aws | Workstream Services |

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

**SSL Configuration**:
```properties
CP4BA_INST_DB_ONLY_SSL="false"  # Can be set to true for production
```

---

## LDAP Configuration

### Embedded OpenLDAP

**Configuration**:
```properties
CP4BA_INST_LDAP=true  # Deploys local OpenLDAP
CP4BA_INST_LDAP_USE_VOLUME=false  # Uses Secret for LDIF
CP4BA_INST_LDAP_USE_PHPADMIN=false  # No phpLDAPadmin UI
```

**LDAP Structure**:
```
dc=example,dc=com
├── ou=Users
│   ├── cn=cp4admin (CP4BA admin)
│   ├── cn=cpadmin (IAM admin)
│   └── cn=user1, user2, ... (test users)
└── ou=Groups
    ├── cn=AdminsGroup
    ├── cn=DevelopersGroup
    └── cn=UsersGroup
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

### IAM Integration

**User Onboarding**:
```properties
CP4BA_INST_IAM=true  # Enables IAM user onboarding
CP4BA_INST_IAM_ADMIN_USER="cpadmin"
CP4BA_INST_IAM_ADMIN_GROUP="AdminsGroup"
```

**SCIM Configuration**:
Maps LDAP attributes to IAM user properties for synchronization.

---

## Storage Configuration

### Storage Classes

**File Storage (RWX)**:
```properties
CP4BA_INST_SC_FILE="ocs-external-storagecluster-cephfs"
```

Used for:
- Shared configuration files
- Log files
- Content storage
- Backup storage

**Block Storage (RWO)**:
```properties
CP4BA_INST_SC_BLOCK="ocs-external-storagecluster-ceph-rbd"
```

Used for:
- Database storage
- High-performance workloads
- Kafka persistent volumes

### Storage Allocation

**BAW Authoring**:
```properties
CP4BA_INST_BAWAUTHORING_STORAGE_SIZE="20Gi"
CP4BA_INST_BAWAUTHORING_STORAGE_DYNAMIC_PROVISIONING="true"
```

**CPE Data Volumes**:
```properties
CP4BA_INST_CPE_DATAVOLUME_STORAGE_SIZE_SMALL="10Gi"
CP4BA_INST_CPE_DATAVOLUME_STORAGE_SIZE_MEDIUM="20Gi"
CP4BA_INST_CPE_DATAVOLUME_STORAGE_SIZE_LARGE="50Gi"
```

---

## Network Configuration

### Service Exposure

**Routes Created**:
- Process Portal: `https://cpd-${NAMESPACE}.apps.${CLUSTER_DOMAIN}`
- Process Designer: Accessed via Process Portal
- Case Builder: Accessed via Process Portal
- Navigator: `https://cpd-${NAMESPACE}.apps.${CLUSTER_DOMAIN}/icn`
- Business Automation Studio: `https://cpd-${NAMESPACE}.apps.${CLUSTER_DOMAIN}/bas`
- BAI Dashboards: `https://cpd-${NAMESPACE}.apps.${CLUSTER_DOMAIN}/bai`

**Ingress Configuration**:
```properties
CP4BA_INST_PLATFORM="OCP"  # Uses OpenShift Routes
sc_ingress_enable: false   # Not using Ingress (ROKS only)
```

### Network Policies

**Configuration**:
```properties
CP4BA_INST_NP_DEPLOY="false"  # Network policies not deployed by default
sc_generate_sample_network_policies: false
```

For production, enable network policies and customize as needed.

---

## Security Configuration

### Certificates

**Root CA**:
```yaml
root_ca_secret: icp4a-root-ca
```

**ZenService Certificate** (Optional):
```properties
CP4BA_INST_ZS_CONFIGURE=true
CP4BA_INST_ZS_SOURCE_SECRET="letsencrypt-certs"
CP4BA_INST_ZS_SOURCE_NAMESPACE="openshift-config"
CP4BA_INST_ZS_TARGET_SECRET="my-letsencrypt"
```

### Secrets

**Key Secrets Created**:
- `ibm-entitlement-key`: IBM Container Registry access
- `ldap-bind-${ENV}`: LDAP bind credentials
- `icp4a-root-ca`: Root CA certificate
- `ibm-iaws-shared-key-secret`: Encryption key
- `${ENV}-bas-server-db-secret`: BAS database credentials
- Database secrets for each component

### Custom XML

**Liberty Custom XML**:
```properties
CP4BA_INST_LIBERTY_CUSTOM_XML_SECRET_NAME="my-liberty-custom-xml-secret"
CP4BA_INST_LIBERTY_CUSTOM_XML_TEMPLATE_NAME="liberty-custom-xml-template-sample-custom-db"
```

**Lombardi Custom XML**:
```properties
CP4BA_INST_LOMBARDI_CUSTOM_XML_SECRET_NAME="my-lombardi-custom-xml-secret"
CP4BA_INST_LOMBARDI_CUSTOM_XML_TEMPLATE_NAME="lombardi-custom-xml-template-sample-document"
```

---

## Use Cases

### 1. Process Development

**Scenario**: Develop and test BPMN workflows

**Workflow**:
1. Access Process Designer via Process Portal
2. Create BPMN process models
3. Define service integrations
4. Create user interfaces
5. Test in authoring environment
6. Deploy to runtime environment

**Benefits**:
- Full authoring capabilities
- Integrated testing
- Version control
- Collaboration features

### 2. Case Management Design

**Scenario**: Design dynamic case management solutions

**Workflow**:
1. Access Case Builder
2. Define case types and properties
3. Create case stages and activities
4. Configure roles and permissions
5. Design case UI
6. Test case scenarios
7. Deploy to production

**Benefits**:
- Visual case design
- Flexible case structures
- Integration with content
- Role-based access

### 3. Low-Code Application Development

**Scenario**: Build business applications with BAS

**Workflow**:
1. Access Business Automation Studio
2. Create application using App Designer
3. Define data models
4. Create workflows
5. Design user interfaces
6. Test in Playback Server
7. Deploy to Application Engine

**Benefits**:
- Rapid development
- No coding required
- Reusable components
- Integrated testing

### 4. Process Analytics and Monitoring

**Scenario**: Monitor and analyze process performance

**Workflow**:
1. Access BAI dashboards
2. View real-time process metrics
3. Analyze bottlenecks
4. Track SLA compliance
5. Generate reports
6. Identify improvement opportunities

**Benefits**:
- Real-time visibility
- Historical analysis
- Custom dashboards
- Process mining

### 5. Training and Demonstrations

**Scenario**: Train users and demonstrate capabilities

**Workflow**:
1. Use embedded LDAP with test users
2. Demonstrate process authoring
3. Show case management
4. Illustrate low-code development
5. Display analytics dashboards

**Benefits**:
- Self-contained environment
- No external dependencies
- Quick setup
- Reproducible demos

---

## Deployment Steps

### Prerequisites

1. **OpenShift Cluster**:
   - Version 4.12 or higher
   - Sufficient resources (CPU, memory, storage)
   - Storage classes configured

2. **IBM Entitlement Key**:
   - Valid entitlement for CP4BA
   - Created as secret `ibm-entitlement-key`

3. **Namespace**:
   - Created: `cp4ba-baw-auth-bai` (or custom name)
   - Labeled appropriately

### Installation Process

1. **Prepare Configuration**:
   ```bash
   # Edit properties file
   vi env1-authoring-baw-bai.properties
   
   # Update key parameters:
   # - CP4BA_INST_NAMESPACE
   # - CP4BA_INST_SC_FILE
   # - CP4BA_INST_SC_BLOCK
   # - License type
   ```

2. **Generate Custom Resource**:
   ```bash
   # Use cp4ba-installations tool to interpolate template
   ./generate-cr.sh env1-authoring-baw-bai.properties
   ```

3. **Deploy Database**:
   ```bash
   # Deploy PostgreSQL
   ./deploy-database.sh
   
   # Wait for database to be ready
   oc wait --for=condition=ready pod -l app=postgresql -n ${NAMESPACE}
   ```

4. **Deploy LDAP**:
   ```bash
   # Deploy OpenLDAP
   ./deploy-ldap.sh
   
   # Wait for LDAP to be ready
   oc wait --for=condition=ready pod -l app=openldap -n ${NAMESPACE}
   ```

5. **Apply Custom Resource**:
   ```bash
   # Apply the generated CR
   oc apply -f ${OUTPUT_FOLDER}/icp4adeploy-cr.yaml -n ${NAMESPACE}
   ```

6. **Monitor Deployment**:
   ```bash
   # Watch operator logs
   oc logs -f -l name=ibm-cp4a-operator -n ${NAMESPACE}
   
   # Check CR status
   oc get icp4acluster -n ${NAMESPACE}
   oc describe icp4acluster icp4adeploy -n ${NAMESPACE}
   ```

7. **Onboard IAM Users**:
   ```bash
   # Run IAM onboarding script
   ./onboard-iam-users.sh
   ```

8. **Configure ZenService Certificate** (Optional):
   ```bash
   # Apply custom certificate
   ./configure-zen-certificate.sh
   ```

### Verification

1. **Check Pod Status**:
   ```bash
   oc get pods -n ${NAMESPACE}
   ```

2. **Access Process Portal**:
   ```bash
   # Get route
   oc get route cpd -n ${NAMESPACE}
   
   # Access: https://cpd-${NAMESPACE}.apps.${CLUSTER_DOMAIN}
   ```

3. **Login**:
   - Username: `cp4admin`
   - Password: `dem0s` (or configured password)

4. **Verify Components**:
   - Process Portal accessible
   - Process Designer opens
   - Case Builder accessible
   - Navigator works
   - BAS accessible
   - BAI dashboards display

---

## Resource Requirements

WARNING: The following are general indications and should be understood as a starting point from which to increase or decrease consumption based on the use case and related non-functional requirements.

### Minimum Resources

**For Small Profile**:
- **CPU**: 20-30 cores
- **Memory**: 64-96 GB
- **Storage**: 200-300 GB

### Component Resource Allocation

| Component | Replicas | CPU Request | Memory Request | CPU Limit | Memory Limit |
|-----------|----------|-------------|----------------|-----------|--------------|
| BAW Server | 1 | 2000m | 4096Mi | 4000m | 8192Mi |
| CPE | 1 | 1000m | 3072Mi | 2000m | 6144Mi |
| Navigator | 1 | 500m | 1024Mi | 1000m | 2048Mi |
| BAS | 1 | 2000m | 2048Mi | 5000m | 3096Mi |
| Flink | 1 | 1000m | 2048Mi | 2000m | 4096Mi |
| Kafka | 1 | 500m | 1024Mi | 1000m | 2048Mi |
| OpenSearch | 1 | 1000m | 2048Mi | 2000m | 4096Mi |
| PostgreSQL | 1 | 2000m | 2048Mi | 4000m | 8192Mi |
| OpenLDAP | 1 | 100m | 256Mi | 200m | 512Mi |

---

## Maintenance and Operations

### Backup Strategy

**Databases**:
```bash
# Backup PostgreSQL
oc exec postgresql-pod -- pg_dumpall > backup.sql
```

**Content**:
- Backup CPE object stores
- Backup FileNet configuration

**Configuration**:
- Export Custom Resource
- Backup secrets and configmaps

### Monitoring

**Health Checks**:
```bash
# Check pod health
oc get pods -n ${NAMESPACE}

# Check CR status
oc get icp4acluster -n ${NAMESPACE}

# View operator logs
oc logs -l name=ibm-cp4a-operator -n ${NAMESPACE}
```

**BAI Monitoring**:
- Access BAI dashboards
- Monitor process metrics
- Check event processing
- Review Flink jobs

### Troubleshooting

**Common Issues**:

1. **Pod Startup Failures**:
   - Check resource availability
   - Verify storage class
   - Review pod logs

2. **Database Connection Issues**:
   - Verify database is running
   - Check credentials
   - Test connectivity

3. **LDAP Issues**:
   - Verify LDAP pod is running
   - Check bind credentials
   - Test LDAP queries

4. **BAI Not Receiving Events**:
   - Check Kafka is running
   - Verify event emitter configuration
   - Review Flink logs

---

## Customization Options

### Scaling

**Increase Replicas**:
```yaml
baw_configuration:
  - name: instance1
    replicas: 2  # Increase for HA
```

**Resource Adjustments**:
```properties
CP4BA_INST_BAS_1_LIMITS_CPU="8000m"
CP4BA_INST_BAS_1_LIMITS_MEMORY="6144Mi"
```

### Additional Object Stores

Add custom object stores for specific applications:
```properties
CP4BA_INST_CPE_OBJSTORAGE_CUSTOM1="CUSTOM1"
CP4BA_INST_CPE_OBJSTORAGE_CUSTOM1_CONN_NAME="CUSTOM1_connection"
```

### GenAI Integration

Enable AI assistants:
```properties
CP4BA_INST_GENAI_ENABLED=true
CP4BA_INST_BAS_GENAI_WX_APIKEY="your-api-key"
CP4BA_INST_BAS_GENAI_WX_PRJ_ID="your-project-id"
```

### External Services

**External Database**:
```properties
CP4BA_INST_DB=false
# Configure external database connection
```

**External LDAP**:
```properties
CP4BA_INST_LDAP=false
# Configure external LDAP connection
```

---

## Migration Path

### From Development to Production

1. **Export Artifacts**:
   - Export process applications
   - Export case solutions
   - Export BAS applications

2. **Prepare Production Environment**:
   - Deploy runtime recipe (recipe-runtime-baw-bai.md)
   - Configure production databases
   - Set up production LDAP

3. **Import and Deploy**:
   - Import process applications
   - Deploy case solutions
   - Deploy BAS applications

4. **Validate**:
   - Test workflows
   - Verify case operations
   - Check integrations

---

## Related Recipes

- **[BAW Runtime with BAI](recipe-runtime-baw-bai.md)**: Production runtime environment
- **[BAW Runtime with Double PFS](recipe-runtime-baw-double-pfs.md)**: Multi-server deployment
- **[WFPS Authoring](recipe-authoring-wfps-pfs-bai.md)**: Lightweight workflow authoring

---

## References

- [CP4BA 25.0.1 Documentation](https://www.ibm.com/docs/en/cloud-paks/cp-biz-automation/25.0.1)
- [BAW Authoring Parameters](https://www.ibm.com/docs/en/cloud-paks/cp-biz-automation/25.0.1?topic=parameters-business-automation-workflow-authoring)
- [BAI Configuration](https://www.ibm.com/docs/en/cloud-paks/cp-biz-automation/25.0.1?topic=insights-configuring-business-automation)
- [BAS Documentation](https://www.ibm.com/docs/en/cloud-paks/cp-biz-automation/25.0.1?topic=studio-business-automation)

---

*Last Updated: 2026-06-23*