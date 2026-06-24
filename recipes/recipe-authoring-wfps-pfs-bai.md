# Recipe: WFPS Authoring with PFS and BAI

## Overview

**Recipe Name**: Workflow Process Service Authoring with Process Federation Server and Business Automation Insights  
**Template File**: `cp4ba-cr-ref-authoring-wfps-bai.yaml`  
**Properties File**: `env1-authoring-wfps-pfs-bai.properties`  
**Purpose**: Workflow Process Service authoring environment (using BAW authoring) with analytics for lightweight workflow development

---

## Deployment Summary

### Patterns
- `foundation` - Base infrastructure services
- `workflow-process-service` - Workflow Process Service (authoring mode)

### Optional Components
- `wfps_authoring` - WFPS authoring capabilities (uses BAW authoring)
- `bai` - Business Automation Insights
- `kafka` - Event streaming platform (required for BAI)
- `opensearch` - Analytics data storage and visualization

### License Type
- Production or Non-Production (configurable via `CP4BA_INST_LICENSE_TYPE`)

---

## Architecture

### Component Topology

```
┌─────────────────────────────────────────────────────────────┐
│                    User Access Layer                        │
│  - Process Designer (BPMN authoring via BAW)                │
│  - Process Portal (task management)                         │
│  - BAI Dashboards (analytics and monitoring)                │
└─────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────┐
│                   Authoring Services                        │
│  ┌──────────────────────────────────────────────┐           │
│  │     BAW Authoring (for WFPS development)     │           │
│  │  - Process Designer                          │           │
│  │  - Toolkit Development                       │           │
│  │  - Integration Designer                      │           │
│  │  - Testing Environment                       │           │
│  └──────────────────────────────────────────────┘           │
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

## Understanding WFPS Authoring

### What is WFPS Authoring?

**Workflow Process Service (WFPS)** is a lightweight workflow runtime service. However, **WFPS does not have its own authoring environment**. Instead:

- **Authoring**: Uses BAW authoring environment (Process Designer)
- **Runtime**: Deploys to WFPS runtime (lightweight execution)

### WFPS vs BAW

| Feature | BAW | WFPS |
|---------|-----|------|
| **Authoring** | Full Process Designer | Uses BAW authoring |
| **Runtime** | Full-featured | Lightweight |
| **Case Management** | Yes | No |
| **Content Integration** | Full FileNet | Minimal/External |
| **Footprint** | Larger | Smaller |
| **Use Case** | Complex workflows | Microservices workflows |
| **Deployment** | Monolithic | Multiple instances |

### Development Workflow

```
1. Design Process
   ↓
   [BAW Authoring Environment]
   - Create BPMN processes
   - Define services
   - Build toolkits
   ↓
2. Test Process
   ↓
   [BAW Authoring Environment]
   - Test in authoring server
   - Debug and refine
   ↓
3. Export Process Application
   ↓
   [Process Application Package]
   ↓
4. Deploy to WFPS Runtime
   ↓
   [WFPS Runtime Environment]
   - Lightweight execution
   - Microservices architecture
```

---

## CP4BA Capabilities Used

### 1. BAW Authoring (for WFPS Development)

**Configuration**:
```yaml
baw_configuration:
  - name: bas  # Note: Named 'bas' but provides BAW authoring
    capabilities: "workflow"
```

**Features Enabled**:
- **Process Designer**: BPMN 2.0 process modeling
- **Toolkit Development**: Reusable components
- **Integration Designer**: Service integration
- **Testing Environment**: Test processes before WFPS deployment
- **Process Portal**: Task management for testing

**Important Notes**:
- This is a **full BAW authoring environment**
- Used to create processes that will run on WFPS runtime
- Processes must be designed with WFPS limitations in mind:
  - No case management features
  - Minimal content integration
  - Lightweight services only

**Databases**:
- Process Server Database: `${ENV}_baw_1`
- Shared databases: BTS, IM, ZEN

**Key Configuration**:
```properties
CP4BA_INST_CUSTOM_XML_BAS_NAME_1="bas"
CP4BA_INST_LIBERTY_CUSTOM_XML_SECRET_NAME="my-liberty-custom-xml-secret"
CP4BA_INST_LOMBARDI_CUSTOM_XML_SECRET_NAME="my-lombardi-custom-xml-secret"
```

### 2. Business Automation Insights (BAI)

**Configuration**:
```yaml
bai_configuration:
  business_performance_center:
    all_users_access: true
  event_forwarder:
    enabled: true
```

**Features Enabled**:
- Real-time process monitoring
- Analytics dashboards
- Event processing with Flink
- Data storage in OpenSearch

**Event Sources**:
```properties
CP4BA_INST_BAI_BPMN="true"               # BPMN process events
CP4BA_INST_BAI_BPMN_TIMESERIES="true"    # Time series data
```

**Use Cases**:
- Monitor process development
- Analyze test executions
- Track performance metrics
- Validate process designs

### 3. Apache Kafka

**Purpose**: Event streaming for BAI

**Configuration**:
- Deployed as part of BAI
- Handles event ingestion
- Provides event buffering

### 4. OpenSearch

**Purpose**: Analytics data storage

**Configuration**:
- Stores BAI event data
- Provides dashboards
- Enables analytics queries

---

## Database Configuration

### Database Architecture

**Single PostgreSQL Server** with minimal databases:

```
PostgreSQL Server: my-postgres-1-for-cp4ba
├── wfps_authoring_pfs_bai_prod_baw_1 (BAW Authoring)
├── wfps_authoring_pfs_bai_prod_bts (Business Teams Service)
├── wfps_authoring_pfs_bai_prod_im (Identity Management)
└── wfps_authoring_pfs_bai_prod_zen (Zen Services)
```

### Database Users

| Database | User | Purpose |
|----------|------|---------|
| baw_1 | bawadmin | BAW Authoring Server |
| bts | bts_user | Business Teams Service |
| im | im_user | Identity Management |
| zen | zen_user | Zen Services |

**Important**: No content databases (DOS, TOS, DOCS) as WFPS doesn't use FileNet.

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
CP4BA_INST_LDAP_USE_PHPADMIN=false
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

```properties
CP4BA_INST_IAM=true
CP4BA_INST_IAM_ADMIN_USER="cpadmin"
CP4BA_INST_IAM_ADMIN_GROUP="AdminsGroup"
```

---

## Use Cases

### 1. WFPS Process Development

**Scenario**: Develop lightweight workflows for WFPS runtime

**Workflow**:
1. Access Process Designer via BAW authoring
2. Create BPMN processes with WFPS constraints:
   - No case management activities
   - No FileNet integration
   - Lightweight services only
   - REST API integrations
3. Test in authoring environment
4. Export process application
5. Deploy to WFPS runtime (separate deployment)

**Benefits**:
- Full authoring capabilities
- Integrated testing
- WFPS-optimized processes
- Microservices-ready workflows

### 2. Microservices Workflow Design

**Scenario**: Design workflows for microservices architecture

**Workflow**:
1. Design stateless processes
2. Use REST services for integration
3. Minimize database dependencies
4. Test scalability
5. Deploy to multiple WFPS instances

**Benefits**:
- Cloud-native design
- Horizontal scalability
- Container-friendly
- API-first approach

### 3. Process Testing and Validation

**Scenario**: Test processes before WFPS deployment

**Workflow**:
1. Develop process in authoring
2. Test with BAI monitoring
3. Analyze performance metrics
4. Optimize process design
5. Validate for WFPS deployment

**Benefits**:
- Early issue detection
- Performance validation
- Design optimization
- Deployment confidence

### 4. Training and Demonstrations

**Scenario**: Train developers on WFPS process design

**Workflow**:
1. Use embedded environment
2. Demonstrate WFPS constraints
3. Show best practices
4. Illustrate deployment process

**Benefits**:
- Self-contained environment
- Hands-on learning
- Best practices demonstration
- Deployment simulation

---

## WFPS Design Constraints

### What to Avoid

**Case Management Features**:
- ❌ Case activities
- ❌ Case properties
- ❌ Case folders
- ❌ Case roles

**FileNet Integration**:
- ❌ Content integration activities
- ❌ Document management
- ❌ Object store operations

**Heavy Components**:
- ❌ Complex human services
- ❌ Large data models
- ❌ Heavy integrations

### What to Use

**Lightweight Activities**:
- ✅ Service tasks
- ✅ Script tasks
- ✅ User tasks (simple)
- ✅ Gateway logic

**REST Integrations**:
- ✅ REST service calls
- ✅ External APIs
- ✅ Microservices integration

**Stateless Design**:
- ✅ Minimal process variables
- ✅ External data storage
- ✅ Event-driven patterns

---

## Resource Requirements

WARNING: The following are general indications and should be understood as a starting point from which to increase or decrease consumption based on the use case and related non-functional requirements.


### Minimum Resources

**For Small Profile**:
- **CPU**: 15-20 cores
- **Memory**: 48-64 GB
- **Storage**: 150-200 GB

### Component Resource Allocation

| Component | Replicas | CPU Request | Memory Request | CPU Limit | Memory Limit |
|-----------|----------|-------------|----------------|-----------|--------------|
| BAW Authoring | 1 | 2000m | 4096Mi | 4000m | 8192Mi |
| Flink | 1 | 1000m | 2048Mi | 2000m | 4096Mi |
| Kafka | 1 | 500m | 1024Mi | 1000m | 2048Mi |
| OpenSearch | 1 | 1000m | 2048Mi | 2000m | 4096Mi |
| PostgreSQL | 1 | 2000m | 2048Mi | 4000m | 8192Mi |
| OpenLDAP | 1 | 100m | 256Mi | 200m | 512Mi |

---

## Exporting for WFPS Runtime

### Export Process Application

1. **From Process Designer**:
   ```
   - Open Process Designer
   - Select Process Application
   - Create Snapshot
   - Export Snapshot
   - Download .twx file
   ```

2. **Prepare for WFPS**:
   ```
   - Verify no case management features
   - Verify no FileNet dependencies
   - Check service integrations
   - Validate process variables
   ```

### Deploy to WFPS Runtime

1. **WFPS Runtime Environment**:
   - Deploy WFPS runtime (separate recipe)
   - Configure WFPS instance
   - Set up databases

2. **Import Process Application**:
   ```
   - Access WFPS admin console
   - Import .twx file
   - Activate snapshot
   - Test deployment
   ```

3. **Verify Deployment**:
   ```
   - Test process execution
   - Verify integrations
   - Check performance
   ```

---

## Integration with PFS

### PFS Deployment

**Note**: PFS is listed in optional components but deployed separately.

**PFS Configuration**:
```yaml
apiVersion: icp4a.ibm.com/v1
kind: ProcessFederationServer
metadata:
  name: pfs
spec:
  pfs_configuration:
    federated_systems:
      - name: wfps1
        type: WFPS
        url: https://wfps1-route
```

**Benefits**:
- Unified task list across WFPS instances
- Cross-system search
- Centralized task management

---

## Maintenance and Operations

### Backup Strategy

**Databases**:
```bash
# Backup BAW authoring database
pg_dump baw_1 > baw-authoring-backup.sql

# Backup shared databases
pg_dump bts > bts-backup.sql
pg_dump im > im-backup.sql
pg_dump zen > zen-backup.sql
```

**Process Applications**:
- Export snapshots regularly
- Version control .twx files
- Document dependencies

### Monitoring

**Development Monitoring**:
- Track process development activity
- Monitor test executions
- Review BAI metrics
- Check resource utilization

**BAI Dashboards**:
- Process execution metrics
- Test coverage
- Performance trends
- Error rates

---

## Best Practices

### WFPS Process Design

1. **Keep It Simple**: Design lightweight processes
2. **Stateless**: Minimize process state
3. **REST APIs**: Use REST for integrations
4. **External Data**: Store data externally
5. **Event-Driven**: Use events for communication

### Development Workflow

1. **Version Control**: Use snapshots effectively
2. **Testing**: Test thoroughly in authoring
3. **Documentation**: Document WFPS constraints
4. **Validation**: Validate before WFPS deployment

### Performance

1. **Optimize Processes**: Keep processes efficient
2. **Minimize Variables**: Reduce process variables
3. **Efficient Services**: Optimize service calls
4. **Caching**: Use caching where appropriate

---

## Troubleshooting

### Common Issues

**1. Process Won't Deploy to WFPS**:
- Check for case management features
- Verify no FileNet dependencies
- Review service integrations
- Validate process structure

**2. Performance Issues in Authoring**:
- Check resource allocation
- Review database performance
- Optimize process design
- Monitor BAI metrics

**3. BAI Not Receiving Events**:
- Verify Kafka is running
- Check event emitter configuration
- Review Flink logs
- Test event generation

---

## Migration to WFPS Runtime

### Preparation Checklist

- [ ] Process application exported
- [ ] No case management features
- [ ] No FileNet dependencies
- [ ] Services are WFPS-compatible
- [ ] Process variables minimized
- [ ] Integration tested
- [ ] Documentation complete

### Deployment Steps

1. Deploy WFPS runtime environment
2. Configure WFPS databases
3. Import process application
4. Configure integrations
5. Test thoroughly
6. Deploy to production

---

## Related Recipes

- **[WFPS Runtime Foundation](recipe-runtime-wfps-pfs-bai-foundation.md)**: WFPS runtime environment
- **[BAW Authoring with BAI](recipe-authoring-baw-bai.md)**: Full BAW authoring
- **[BAW Runtime with Double PFS](recipe-runtime-baw-double-pfs.md)**: Multi-server with PFS

---

## References

- [CP4BA 25.0.1 Documentation](https://www.ibm.com/docs/en/cloud-paks/cp-biz-automation/25.0.1)
- [WFPS Documentation](https://www.ibm.com/docs/en/cloud-paks/cp-biz-automation/25.0.1?topic=ipd-installing-cp4ba-workflow-process-service-runtime-production-deployment)
- [BAW Authoring Parameters](https://www.ibm.com/docs/en/cloud-paks/cp-biz-automation/25.0.1?topic=parameters-business-automation-workflow-authoring)
- [WFPS Design Guidelines](https://www.ibm.com/docs/en/baw/25.0.x)

---

*Last Updated: 2026-06-23*