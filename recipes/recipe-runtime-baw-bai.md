# Recipe: BAW Runtime with BAI

## Overview

**Recipe Name**: BAW Runtime with Business Automation Insights  
**Template File**: `cp4ba-cr-ref-baw-bai.yaml`  
**Properties File**: `env1-runtime-baw-bai.properties`  
**Purpose**: Production Business Automation Workflow runtime environment with real-time monitoring and analytics

---

## Deployment Summary

### Patterns
- `foundation` - Base infrastructure services
- `workflow` - Business Automation Workflow (runtime mode)

### Optional Components
- `workplace_assistant` - AI-powered workplace assistance (optional, requires GenAI)
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
│  - Process Portal (task management and execution)           │
│  - Business Automation Navigator (content access)           │
│  - BAI Dashboards (monitoring and analytics)                │
│  - Workplace Assistant (optional AI assistance)             │
└─────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────┐
│                   Runtime Services                          │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐       │
│  │ BAW Runtime  │  │     CPE      │  │  Navigator   │       │
│  │   Server     │  │  (Content)   │  │              │       │
│  └──────────────┘  └──────────────┘  └──────────────┘       │
│  ┌──────────────┐                                           │
│  │   GraphQL    │                                           │
│  └──────────────┘                                           │
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

### 1. Business Automation Workflow (BAW) - Runtime Mode

**Configuration**:
```yaml
baw_configuration:
  - name: baw1
    capabilities: "workflow"  # Runtime only, no authoring
```

**Features Enabled**:
- **Process Execution**: Run deployed process applications
- **Process Portal**: Task management and monitoring
- **Case Management**: Execute case operations
- **REST APIs**: Programmatic access to workflows
- **Integration Services**: Connect to external systems

**Databases**:
- Process Server Database (BPMDB): `${ENV}_baw_1`
- Design Object Store (DOS): `${ENV}_dos`
- Target Object Store (TOS): `${ENV}_tos`
- Document Object Store (DOCS): `${ENV}_docs`

**Key Differences from Authoring**:
- No Process Designer
- No Case Builder
- No toolkit development
- Optimized for execution performance
- Production-grade configuration

**Resource Configuration**:
```properties
# Runtime servers typically have higher resource allocation
CP4BA_INST_BAW_1_REPLICAS="1"  # Can be increased for HA
CP4BA_INST_BAW_1_LIMITS_CPU="4000m"
CP4BA_INST_BAW_1_LIMITS_MEMORY="8192Mi"
```

### 2. FileNet Content Manager (FNCM)

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
- **Content Platform Engine (CPE)**: Content repository for case documents
- **Business Automation Navigator**: User interface for content access
- **GraphQL API**: Modern API for content operations

**Object Stores**:
1. **Global Configuration Data (GCD)**: System configuration
2. **Design Object Store (DOS)**: Case designs (read-only in runtime)
3. **Target Object Store (TOS)**: Case runtime data
4. **Document Object Store (DOCS)**: Document storage
5. **Content Object Store**: General content storage

**Resource Configuration**:
```properties
CP4BA_INST_CPE_REPLICAS="1"
CP4BA_INST_CPE_RES_LIMITS_CPU="2"
CP4BA_INST_CPE_RES_LIMITS_MEM="6144Mi"
CP4BA_INST_CPE_RES_REQS_CPU="1"
CP4BA_INST_CPE_RES_REQS_MEM="3072Mi"
```

### 3. Business Automation Insights (BAI)

**Configuration**:
```yaml
bai_configuration:
  business_performance_center:
    all_users_access: true
  event_forwarder:
    enabled: true
```

**Features Enabled**:
- **Real-time Monitoring**: Live process execution tracking
- **Business Performance Center**: Production dashboards
- **Historical Analytics**: Trend analysis and reporting
- **Event Processing**: Apache Flink stream processing
- **Data Storage**: OpenSearch for analytics data

**Event Sources Configured**:
```properties
CP4BA_INST_BAI_BPMN="true"               # BPMN process events
CP4BA_INST_BAI_BPMN_TIMESERIES="true"    # Time series data
CP4BA_INST_BAI_CONTENT="true"            # Content events
CP4BA_INST_BAI_ICM="true"                # Case management events
CP4BA_INST_BAI_NAVIGATOR="true"          # Navigator events
CP4BA_INST_BAI_EGRESS="true"             # External event forwarding
```

**Flink Configuration**:
```properties
CP4BA_INST_BAI_FLINK_ROUTE="true"
CP4BA_INST_BAI_FLINK_ADDITIONAL_TASK_MGRS=1
CP4BA_INST_BAI_END_AGGR_DELAY=10000
```

### 4. Apache Kafka

**Purpose**: Event streaming platform for BAI

**Configuration**:
- Deployed as part of BAI
- Handles event ingestion from BAW runtime
- Provides event buffering and reliability
- Enables event replay for analytics

### 5. OpenSearch

**Purpose**: Analytics data storage and visualization

**Configuration**:
- Stores BAI event data
- Provides Kibana-like dashboards
- Enables custom analytics queries
- Supports historical data analysis

### 6. Workplace Assistant (Optional)

**Purpose**: AI-powered assistance for end users

**Configuration**:
```properties
CP4BA_RUN_WORKPLACE_AGENT="${CP4BA_INST_GENAI_ENABLED}"
```

**Features** (when enabled):
- Intelligent task recommendations
- Natural language queries
- Process guidance
- Contextual help

---

## Database Configuration

### Database Architecture

**Single PostgreSQL Server** with multiple databases:

```
PostgreSQL Server: my-postgres-1-for-cp4ba
├── baw_bai_production_baw_1 (BAW Process Server)
├── baw_bai_production_dos (Design Object Store)
├── baw_bai_production_tos (Target Object Store)
├── baw_bai_production_docs (Document Object Store)
├── baw_bai_production_gcd (Global Configuration Data)
├── baw_bai_production_content (Content Object Store)
└── baw_bai_production_icndb (Navigator)
```

### Database Users

| Database | User | Purpose |
|----------|------|---------|
| baw_1 | bawadmin | BAW Process Server |
| dos | bawdos | Design Object Store |
| tos | bawtos | Target Object Store |
| docs | bawdocs | Document Object Store |
| gcd | gcd | Global Configuration Data |
| content | content | Content Object Store |
| icndb | icn | Navigator |

### Database Server Configuration

```properties
CP4BA_INST_DB_SERVER_TYPE="postgresql"
CP4BA_INST_DB_SERVER_PORT="5432"
CP4BA_INST_DB_STORAGE_SIZE="10Gi"  # Increase for production
CP4BA_INST_DB_MAX_POOL_SIZE="100"
CP4BA_INST_DB_MIN_POOL_SIZE="10"
```

**Production Resource Allocation**:
```properties
CP4BA_INST_DB_REQS_CPU="2000m"
CP4BA_INST_DB_REQS_MEMORY="2048Mi"
CP4BA_INST_DB_LIMITS_CPU="4000m"
CP4BA_INST_DB_LIMITS_MEMORY="8192Mi"
```

**SSL Configuration** (Recommended for Production):
```properties
CP4BA_INST_DB_ONLY_SSL="true"  # Enable SSL for production
```

---

## LDAP Configuration

### Production LDAP Options

**Option 1: Embedded OpenLDAP** (Testing/Demo):
```properties
CP4BA_INST_LDAP=true
CP4BA_INST_LDAP_USE_VOLUME=false
```

**Option 2: External LDAP** (Production):
```properties
CP4BA_INST_LDAP=false
# Configure external LDAP connection
```

**LDAP Connection**:
```yaml
ldap_configuration:
  lc_selected_ldap_type: "Custom"  # or "Microsoft Active Directory"
  lc_ldap_server: "ldap.company.com"
  lc_ldap_port: "636"  # SSL port
  lc_ldap_base_dn: "dc=company,dc=com"
  lc_ldap_ssl_enabled: true
  lc_ldap_ssl_secret_name: "ldap-ssl-cert"
```

### IAM Integration

**User Onboarding**:
```properties
CP4BA_INST_IAM=true
CP4BA_INST_IAM_ADMIN_USER="cpadmin"
CP4BA_INST_IAM_ADMIN_GROUP="AdminsGroup"
```

---

## Storage Configuration

### Storage Classes

**File Storage (RWX)**:
```properties
CP4BA_INST_SC_FILE="ocs-external-storagecluster-cephfs"
```

**Block Storage (RWO)**:
```properties
CP4BA_INST_SC_BLOCK="ocs-external-storagecluster-ceph-rbd"
```

### Storage Allocation

**CPE Data Volumes**:
```properties
CP4BA_INST_CPE_DATAVOLUME_STORAGE_SIZE_SMALL="10Gi"
CP4BA_INST_CPE_DATAVOLUME_STORAGE_SIZE_MEDIUM="20Gi"
CP4BA_INST_CPE_DATAVOLUME_STORAGE_SIZE_LARGE="50Gi"
```

---

## Use Cases

### 1. Production Workflow Execution

**Scenario**: Execute business processes in production

**Workflow**:
1. Deploy process applications from authoring environment
2. Users access Process Portal
3. Execute workflows and complete tasks
4. Monitor performance via BAI
5. Handle exceptions and errors

**Benefits**:
- Optimized for performance
- High availability options
- Production monitoring
- Scalable architecture

### 2. Case Management Operations

**Scenario**: Manage dynamic cases in production

**Workflow**:
1. Create cases from templates
2. Add documents to cases
3. Execute case activities
4. Collaborate on cases
5. Track case progress
6. Close cases

**Benefits**:
- Flexible case handling
- Document integration
- Role-based access
- Audit trail

### 3. Business Process Monitoring

**Scenario**: Monitor and analyze process performance

**Workflow**:
1. Access BAI dashboards
2. View real-time metrics
3. Analyze process bottlenecks
4. Track SLA compliance
5. Generate reports
6. Identify optimization opportunities

**Benefits**:
- Real-time visibility
- Historical analysis
- Performance metrics
- Compliance tracking

### 4. Task Management

**Scenario**: Manage and complete work items

**Workflow**:
1. Users access Process Portal
2. View assigned tasks
3. Complete tasks
4. Reassign or delegate tasks
5. Track task history

**Benefits**:
- Centralized task list
- Priority management
- Collaboration features
- Mobile access

---

## Resource Requirements

WARNING: The following are general indications and should be understood as a starting point from which to increase or decrease consumption based on the use case and related non-functional requirements.

### Production Resources

**For Small Profile** (Minimum):
- **CPU**: 15-20 cores
- **Memory**: 48-64 GB
- **Storage**: 150-200 GB

**For Medium Profile** (Recommended):
- **CPU**: 30-40 cores
- **Memory**: 96-128 GB
- **Storage**: 300-500 GB

### Component Resource Allocation

| Component | Replicas | CPU Request | Memory Request | CPU Limit | Memory Limit |
|-----------|----------|-------------|----------------|-----------|--------------|
| BAW Server | 2 | 2000m | 4096Mi | 4000m | 8192Mi |
| CPE | 2 | 1000m | 3072Mi | 2000m | 6144Mi |
| Navigator | 2 | 500m | 1024Mi | 1000m | 2048Mi |
| Flink | 2 | 1000m | 2048Mi | 2000m | 4096Mi |
| Kafka | 3 | 500m | 1024Mi | 1000m | 2048Mi |
| OpenSearch | 3 | 1000m | 2048Mi | 2000m | 4096Mi |
| PostgreSQL | 1 | 2000m | 4096Mi | 4000m | 8192Mi |

---

## High Availability Configuration

### Multiple Replicas

**BAW Runtime**:
```yaml
baw_configuration:
  - name: baw1
    replicas: 2  # or more for HA
```

**CPE**:
```yaml
ecm_configuration:
  cpe:
    replica_count: 2
```

**Navigator**:
```yaml
ecm_configuration:
  navigator:
    replica_count: 2
```

### Database HA

**Options**:
1. PostgreSQL with replication
2. External managed database (RDS, Azure Database, etc.)
3. Database clustering (RAC for Oracle, Always On for SQL Server)

### Storage HA

**Requirements**:
- Replicated storage backend
- Multi-zone storage classes
- Regular backups

---

## Maintenance and Operations

### Backup Strategy

**Daily Backups**:
```bash
# Database backup
oc exec postgresql-pod -- pg_dumpall > backup-$(date +%Y%m%d).sql

# Content backup
# Use CPE backup utilities

# Configuration backup
oc get icp4acluster -o yaml > cr-backup-$(date +%Y%m%d).yaml
```

**Backup Retention**:
- Daily: 7 days
- Weekly: 4 weeks
- Monthly: 12 months

### Monitoring

**Health Checks**:
```bash
# Pod health
oc get pods -n ${NAMESPACE}

# CR status
oc describe icp4acluster -n ${NAMESPACE}

# Resource utilization
oc adm top pods -n ${NAMESPACE}
```

**BAI Monitoring**:
- Process execution metrics
- Task completion rates
- SLA compliance
- Error rates
- Performance trends

### Performance Tuning

**Database Tuning**:
- Optimize connection pools
- Tune PostgreSQL parameters
- Regular VACUUM and ANALYZE

**BAW Tuning**:
- Adjust thread pools
- Optimize cache settings
- Configure JVM parameters

**Content Tuning**:
- Optimize object store configuration
- Configure cache settings
- Tune search indexing

### Troubleshooting

**Common Issues**:

1. **Performance Degradation**:
   - Check resource utilization
   - Review database performance
   - Analyze BAI metrics
   - Check network latency

2. **Task Delays**:
   - Check BAW server logs
   - Verify database connections
   - Review thread pool settings

3. **BAI Data Gaps**:
   - Check Kafka connectivity
   - Verify Flink jobs
   - Review event emitter configuration

---

## Security Best Practices

### Production Security

1. **Enable SSL/TLS**:
   - Database connections
   - LDAP connections
   - External routes

2. **Network Policies**:
   - Implement restrictive policies
   - Limit pod-to-pod communication
   - Control external access

3. **Secrets Management**:
   - Use external secrets manager
   - Rotate credentials regularly
   - Encrypt sensitive data

4. **Access Control**:
   - Implement RBAC
   - Use least privilege principle
   - Regular access reviews

5. **Audit Logging**:
   - Enable comprehensive logging
   - Monitor security events
   - Regular log reviews

---

## Scaling Strategy

### Horizontal Scaling

**Increase Replicas**:
```yaml
baw_configuration:
  - name: baw1
    replicas: 3  # Scale based on load
```

**Load Balancing**:
- OpenShift routes handle load balancing
- Session affinity for stateful operations

### Vertical Scaling

**Increase Resources**:
```properties
CP4BA_INST_BAW_1_LIMITS_CPU="8000m"
CP4BA_INST_BAW_1_LIMITS_MEMORY="16384Mi"
```

### Auto-scaling

**Horizontal Pod Autoscaler**:
```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: baw-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: baw-server
  minReplicas: 2
  maxReplicas: 5
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
```

---

## Migration from Authoring

### Process Application Deployment

1. **Export from Authoring**:
   - Use Process Designer
   - Export process applications
   - Include all dependencies

2. **Import to Runtime**:
   - Access Process Admin Console
   - Import process applications
   - Activate snapshots

3. **Validate**:
   - Test process execution
   - Verify integrations
   - Check permissions

### Case Solution Deployment

1. **Export from Authoring**:
   - Use Case Builder
   - Export case solutions
   - Include all artifacts

2. **Deploy to Runtime**:
   - Use Case Configuration Tool
   - Deploy case solutions
   - Configure security

3. **Validate**:
   - Create test cases
   - Verify workflows
   - Check document access

---

## Related Recipes

- **[BAW Authoring with BAI](recipe-authoring-baw-bai.md)**: Development environment
- **[BAW Runtime with Double PFS](recipe-runtime-baw-double-pfs.md)**: Multi-server deployment
- **[WFPS Runtime Foundation](recipe-runtime-wfps-pfs-bai-foundation.md)**: Lightweight runtime

---

## References

- [CP4BA 25.0.1 Documentation](https://www.ibm.com/docs/en/cloud-paks/cp-biz-automation/25.0.1)
- [BAW Runtime Parameters](https://www.ibm.com/docs/en/cloud-paks/cp-biz-automation/25.0.1?topic=parameters-business-automation-workflow-runtime-workstream-services)
- [BAI Configuration](https://www.ibm.com/docs/en/cloud-paks/cp-biz-automation/25.0.1?topic=insights-configuring-business-automation)
- [Production Deployment Guide](https://www.ibm.com/docs/en/cloud-paks/cp-biz-automation/25.0.1?topic=deployments-installing-cp4ba-multi-pattern-production-deployment)

---

*Last Updated: 2026-06-23*