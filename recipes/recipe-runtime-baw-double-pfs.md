# Recipe: Multiple BAW Runtimes with PFS

## Overview

**Recipe Name**: Multiple BAW Runtime with Process Federation Server  
**Template File**: `cp4ba-cr-ref-baw-double-pfs.yaml`  
**Properties File**: `env1-runtime-baw-double-pfs.properties`  
**Purpose**: Multiple Business Automation Workflow runtime servers with Process Federation Server for unified task management

---

## Deployment Summary

### Patterns
- `foundation` - Base infrastructure services
- `workflow` - Business Automation Workflow (runtime mode, 2 instances)

### Optional Components
- `pfs` - Process Federation Server (deployed separately)

### License Type
- Non-Production or Production (configurable via `CP4BA_INST_LICENSE_TYPE`)

---

## Architecture

### Component Topology

```
┌─────────────────────────────────────────────────────────────┐
│                    User Access Layer                        │
│  - Process Federation Server (unified task list)            │
│  - Process Portal 1 (BAW Server 1)                          │
│  - Process Portal 2 (BAW Server 2)                          │
│  - Business Automation Navigator 1                          │
│  - Business Automation Navigator 2                          │
└─────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────┐
│                   Runtime Services                          │
│  ┌──────────────────────┐  ┌──────────────────────┐         │
│  │   BAW Server 1       │  │   BAW Server 2       │         │
│  │  (Full: Case+BPM)    │  │  (BPM+Content Only)  │         │
│  │  ┌────────────────┐  │  │  ┌────────────────┐  │         │
│  │  │ Process Portal │  │  │  │ Process Portal │  │         │
│  │  └────────────────┘  │  │  └────────────────┘  │         │
│  │  ┌────────────────┐  │  │  ┌────────────────┐  │         │
│  │  │   Navigator    │  │  │  │   Navigator    │  │         │
│  │  └────────────────┘  │  │  └────────────────┘  │         │
│  └──────────────────────┘  └──────────────────────┘         │
└─────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────┐
│                   Content Services                          │
│  ┌──────────────────────┐  ┌──────────────────────┐         │
│  │   CPE (Server 1)     │  │   CPE (Server 2)     │         │
│  │  - GCD (shared)      │  │  - Uses same GCD     │         │
│  │  - DOS               │  │  - No DOS            │         │
│  │  - TOS               │  │  - No TOS            │         │
│  │  - DOCS1             │  │  - DOCS2             │         │
│  └──────────────────────┘  └──────────────────────┘         │
└─────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────┐
│              Process Federation Server (PFS)                │
│  - Federates BAW Server 1                                   │
│  - Federates BAW Server 2                                   │
│  - Unified task list                                        │
│  - Cross-system search                                      │
│  - REST API                                                 │
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

### 1. Business Automation Workflow - Multiple Instances

This recipe deploys **two BAW runtime servers** with different configurations:

#### BAW Server 1 (Full Configuration)

**Configuration**:
```yaml
baw_configuration:
  - name: baw1
    capabilities: "workflow"
    case_configuration:
      enabled: true
```

**Features**:
- Full BPM capabilities
- Case management enabled
- Design Object Store (DOS)
- Target Object Store (TOS)
- Document Object Store (DOCS)
- Content Object Store
- Dedicated Navigator

**Databases**:
- Process Server: `${ENV}_baw_1`
- DOS: `${ENV}_dos_1`
- TOS: `${ENV}_tos_1`
- DOCS: `${ENV}_docs_1`
- Content: `${ENV}_content_1`

**Use Cases**:
- Application 1 workflows
- Case management operations
- Document-centric processes
- Full workflow capabilities

#### BAW Server 2 (Content Only)

**Configuration**:
```yaml
baw_configuration:
  - name: baw2
    capabilities: "workflow"
    case_configuration:
      enabled: false  # No case management
```

**Features**:
- BPM capabilities only
- No case management
- No DOS/TOS (uses Server 1's if needed)
- Document Object Store (DOCS)
- Content Object Store
- Dedicated Navigator

**Databases**:
- Process Server: `${ENV}_baw_2`
- DOCS: `${ENV}_docs_2`
- Content: `${ENV}_content_2`

**Use Cases**:
- Application 2 workflows
- Document workflows
- Lightweight processes
- Isolated application execution

**Key Configuration**:
```properties
# BAW Server 1
CP4BA_INST_BAW_1_NAME="baw1"
CP4BA_INST_BAW_1_CASE_ENABLED="true"

# BAW Server 2
CP4BA_INST_BAW_2_NAME="baw2"
CP4BA_INST_BAW_2_CASE_ENABLED="false"
```

### 2. Process Federation Server (PFS)

**Deployment**: Separate Custom Resource (ProcessFederationServer kind)

**Configuration**:
```yaml
apiVersion: icp4a.ibm.com/v1
kind: ProcessFederationServer
metadata:
  name: pfs
spec:
  pfs_configuration:
    replicas: 1
    federated_systems:
      - name: baw1
        type: BAW
        url: https://baw1-route
      - name: baw2
        type: BAW
        url: https://baw2-route
```

**Features**:
- **Unified Task List**: Single view of tasks from both BAW servers
- **Cross-System Search**: Search tasks across all federated systems
- **Saved Searches**: User-defined search criteria
- **REST API**: Programmatic access to federated tasks
- **Process Visibility**: View processes across systems

**Federated Systems**:
1. BAW Server 1 (baw1)
2. BAW Server 2 (baw2)

**Benefits**:
- Single point of access for all tasks
- Consistent user experience
- Centralized task management
- Cross-application visibility

**Resource Configuration**:
```properties
CP4BA_INST_PFS_REPLICAS="1"
CP4BA_INST_PFS_LIMITS_CPU="2000m"
CP4BA_INST_PFS_LIMITS_MEMORY="4096Mi"
```

### 3. FileNet Content Manager (FNCM)

**Shared Components**:
- **Global Configuration Data (GCD)**: Shared by both BAW servers
- **GraphQL API**: Shared content API

**Per-Server Components**:
- **CPE Instances**: Separate CPE for each BAW server
- **Navigator Instances**: Dedicated Navigator for each server
- **Object Stores**: Separate object stores per server

**Configuration**:
```yaml
ecm_configuration:
  cpe:
    - name: cpe1
      replica_count: 1
      object_stores:
        - dos1
        - tos1
        - docs1
        - content1
    - name: cpe2
      replica_count: 1
      object_stores:
        - docs2
        - content2
  navigator:
    - name: icn1
      replica_count: 1
    - name: icn2
      replica_count: 1
```

---

## Database Configuration

### Database Architecture

**Single PostgreSQL Server** with databases for both BAW servers:

```
PostgreSQL Server: my-postgres-1-for-cp4ba
├── BAW Server 1 Databases
│   ├── baw_prod_double_pfs_baw_1 (Process Server 1)
│   ├── baw_prod_double_pfs_dos_1 (Design Object Store 1)
│   ├── baw_prod_double_pfs_tos_1 (Target Object Store 1)
│   ├── baw_prod_double_pfs_docs_1 (Document Object Store 1)
│   ├── baw_prod_double_pfs_content_1 (Content Object Store 1)
│   └── baw_prod_double_pfs_icndb_1 (Navigator 1)
├── BAW Server 2 Databases
│   ├── baw_prod_double_pfs_baw_2 (Process Server 2)
│   ├── baw_prod_double_pfs_docs_2 (Document Object Store 2)
│   ├── baw_prod_double_pfs_content_2 (Content Object Store 2)
│   └── baw_prod_double_pfs_icndb_2 (Navigator 2)
└── Shared Databases
    └── baw_prod_double_pfs_gcd (Global Configuration Data)
```

### Database Users

**BAW Server 1**:
| Database | User | Purpose |
|----------|------|---------|
| baw_1 | bawadmin1 | Process Server 1 |
| dos_1 | bawdos1 | Design Object Store 1 |
| tos_1 | bawtos1 | Target Object Store 1 |
| docs_1 | bawdocs1 | Document Object Store 1 |
| icndb_1 | icn1 | Navigator 1 |

**BAW Server 2**:
| Database | User | Purpose |
|----------|------|---------|
| baw_2 | bawadmin2 | Process Server 2 |
| docs_2 | bawdocs2 | Document Object Store 2 |
| icndb_2 | icn2 | Navigator 2 |

**Shared**:
| Database | User | Purpose |
|----------|------|---------|
| gcd | gcd | Global Configuration Data |

### Database Server Configuration

```properties
CP4BA_INST_DB_SERVER_TYPE="postgresql"
CP4BA_INST_DB_SERVER_PORT="5432"
CP4BA_INST_DB_STORAGE_SIZE="20Gi"  # Larger for multiple servers
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

## Use Cases

### 1. Multi-Application Deployment

**Scenario**: Deploy multiple workflow applications with isolation

**Architecture**:
- Application 1 on BAW Server 1
- Application 2 on BAW Server 2
- PFS provides unified task access

**Benefits**:
- Application isolation
- Independent scaling
- Separate maintenance windows
- Resource allocation per application

**Example**:
```
BAW Server 1: HR Processes
├── Employee Onboarding
├── Leave Requests
└── Performance Reviews

BAW Server 2: Finance Processes
├── Purchase Orders
├── Invoice Processing
└── Expense Reports

PFS: Unified Task List
└── All tasks from both applications
```

### 2. Workload Segregation

**Scenario**: Separate high-volume and low-volume processes

**Architecture**:
- BAW Server 1: High-volume, simple processes
- BAW Server 2: Low-volume, complex processes with case management
- PFS for unified access

**Benefits**:
- Performance optimization
- Resource allocation based on workload
- Reduced contention
- Better scalability

### 3. Multi-Tenant Architecture

**Scenario**: Support multiple business units or customers

**Architecture**:
- BAW Server 1: Tenant A
- BAW Server 2: Tenant B
- PFS with tenant-aware filtering

**Benefits**:
- Data isolation
- Independent configurations
- Separate SLAs
- Tenant-specific customizations

### 4. Phased Migration

**Scenario**: Migrate from legacy system gradually

**Architecture**:
- BAW Server 1: New processes
- BAW Server 2: Migrated legacy processes
- PFS provides unified interface

**Benefits**:
- Gradual migration
- Parallel operation
- Risk mitigation
- User continuity

### 5. Development and Testing Isolation

**Scenario**: Separate development/test from production-like environment

**Architecture**:
- BAW Server 1: Production-like testing
- BAW Server 2: Development and experimentation
- PFS for integrated testing

**Benefits**:
- Safe experimentation
- Realistic testing
- Minimal production impact
- Integrated validation

---

## PFS Configuration

### Federated System Setup

**BAW Server 1 Configuration**:
```yaml
federated_systems:
  - name: baw1
    type: BAW
    url: https://baw1-cpd-namespace.apps.cluster.com
    rest_url: https://baw1-cpd-namespace.apps.cluster.com/rest/bpm/wle/v1
    authentication:
      type: UMS
      secret: baw1-auth-secret
```

**BAW Server 2 Configuration**:
```yaml
federated_systems:
  - name: baw2
    type: BAW
    url: https://baw2-cpd-namespace.apps.cluster.com
    rest_url: https://baw2-cpd-namespace.apps.cluster.com/rest/bpm/wle/v1
    authentication:
      type: UMS
      secret: baw2-auth-secret
```

### PFS Features

**Task Federation**:
- Aggregates tasks from all federated systems
- Real-time synchronization
- Consistent task representation

**Search Capabilities**:
- Cross-system search
- Saved searches
- Advanced filters
- Custom queries

**REST API**:
```
GET /rest/bpm/federated/v1/tasks
GET /rest/bpm/federated/v1/tasks/{taskId}
POST /rest/bpm/federated/v1/tasks/{taskId}/actions
GET /rest/bpm/federated/v1/searches
```

---

## Resource Requirements

WARNING: The following are general indications and should be understood as a starting point from which to increase or decrease consumption based on the use case and related non-functional requirements.

### Total Resources

**For Small Profile**:
- **CPU**: 25-35 cores
- **Memory**: 80-100 GB
- **Storage**: 250-350 GB

### Component Resource Allocation

| Component | Instances | CPU Request | Memory Request | CPU Limit | Memory Limit |
|-----------|-----------|-------------|----------------|-----------|--------------|
| BAW Server 1 | 1 | 2000m | 4096Mi | 4000m | 8192Mi |
| BAW Server 2 | 1 | 2000m | 4096Mi | 4000m | 8192Mi |
| CPE 1 | 1 | 1000m | 3072Mi | 2000m | 6144Mi |
| CPE 2 | 1 | 1000m | 3072Mi | 2000m | 6144Mi |
| Navigator 1 | 1 | 500m | 1024Mi | 1000m | 2048Mi |
| Navigator 2 | 1 | 500m | 1024Mi | 1000m | 2048Mi |
| PFS | 1 | 1000m | 2048Mi | 2000m | 4096Mi |
| PostgreSQL | 1 | 2000m | 4096Mi | 4000m | 8192Mi |

---

## Maintenance and Operations

### Backup Strategy

**Per-Server Backups**:
```bash
# Backup BAW Server 1 databases
pg_dump baw_1 > baw1-backup.sql
pg_dump dos_1 > dos1-backup.sql
pg_dump tos_1 > tos1-backup.sql

# Backup BAW Server 2 databases
pg_dump baw_2 > baw2-backup.sql
```

**PFS Backup**:
```bash
# Backup PFS configuration
oc get processfederationserver -o yaml > pfs-backup.yaml

# Backup PFS data (if persistent)
# PFS uses embedded Elasticsearch
```

### Monitoring

**Per-Server Monitoring**:
- Monitor each BAW server independently
- Track resource utilization per server
- Monitor database connections per server

**PFS Monitoring**:
- Monitor federation health
- Track federated system availability
- Monitor task synchronization
- Check search performance

### Scaling

**Scale Individual Servers**:
```yaml
baw_configuration:
  - name: baw1
    replicas: 2  # Scale BAW Server 1
  - name: baw2
    replicas: 2  # Scale BAW Server 2
```

**Scale PFS**:
```yaml
pfs_configuration:
  replicas: 2  # Scale PFS for HA
```

---

## Troubleshooting

### Common Issues

**1. PFS Cannot Connect to BAW Server**:
- Verify BAW server URLs
- Check authentication secrets
- Test network connectivity
- Review PFS logs

**2. Tasks Not Appearing in PFS**:
- Verify federation configuration
- Check BAW server REST API
- Review synchronization logs
- Test direct BAW access

**3. Performance Issues**:
- Check resource allocation
- Monitor database performance
- Review network latency
- Analyze PFS search performance

**4. Authentication Failures**:
- Verify UMS configuration
- Check LDAP connectivity
- Review SSO settings
- Test user credentials

---

## Best Practices

### Application Distribution

1. **Logical Separation**: Group related processes on same server
2. **Load Balancing**: Distribute workload evenly
3. **Resource Allocation**: Allocate resources based on workload
4. **Isolation**: Keep critical processes isolated

### PFS Configuration

1. **Saved Searches**: Create common saved searches for users
2. **Performance**: Optimize search queries
3. **Monitoring**: Monitor federation health
4. **Failover**: Configure PFS HA for production

### Database Management

1. **Connection Pools**: Size appropriately per server
2. **Maintenance**: Schedule maintenance per server
3. **Backups**: Backup each server's databases separately
4. **Monitoring**: Monitor database performance per server

---

## Related Recipes

- **[BAW Authoring with BAI](recipe-authoring-baw-bai.md)**: Development environment
- **[BAW Runtime with BAI](recipe-runtime-baw-bai.md)**: Single server runtime
- **[WFPS Runtime Foundation](recipe-runtime-wfps-pfs-bai-foundation.md)**: WFPS with PFS

---

## References

- [CP4BA 25.0.1 Documentation](https://www.ibm.com/docs/en/cloud-paks/cp-biz-automation/25.0.1)
- [Multiple BAW Instances](https://www.ibm.com/docs/en/cloud-paks/cp-biz-automation/25.0.1?topic=cbawrws-optional-configuring-multiple-instances-business-automation-workflow-workstream-services)
- [PFS Documentation](https://www.ibm.com/docs/en/cloud-paks/cp-biz-automation/25.0.1?topic=deployments-installing-cp4ba-process-federation-server-production-deployment)
- [PFS with BAW](https://www.ibm.com/docs/en/baw/25.0.x?topic=ic-installing-process-federation-server-business-automation-workflow-containers)

---

*Last Updated: 2026-06-23*