# IBM Cloud Pak for Business Automation (CP4BA) - Knowledge Base

## Version Information
- **CP4BA Version**: 25.0.1
- **Documentation Date**: 2026-06-23
- **Purpose**: Educational and reference documentation for CP4BA deployment configurations

---

## Table of Contents

1. [Overview](#overview)
2. [CP4BA Architecture](#cp4ba-architecture)
3. [Core Components](#core-components)
4. [Deployment Patterns](#deployment-patterns)
5. [Optional Components](#optional-components)
6. [Infrastructure Requirements](#infrastructure-requirements)
7. [Database Configuration](#database-configuration)
8. [LDAP and IAM Integration](#ldap-and-iam-integration)
9. [Security and Certificates](#security-and-certificates)
10. [Deployment Topologies](#deployment-topologies)
11. [Recipe Templates](#recipe-templates)

---

## Overview

IBM Cloud Pak for Business Automation (CP4BA) is a comprehensive platform that provides integrated capabilities for business automation, including workflow management, content management, decision automation, and document processing. CP4BA 25.0.1 runs on Red Hat OpenShift Container Platform and leverages Kubernetes for orchestration.

### Key Features
- **Multi-pattern deployment**: Support for foundation, workflow, content, application, decisions, and document processing patterns
- **Containerized architecture**: All components run as containers in OpenShift
- **Scalable and resilient**: Kubernetes-native design with high availability options
- **Integrated security**: Built-in integration with IBM Cloud Pak Foundational Services (CPFS) for IAM
- **Flexible licensing**: Support for production and non-production licenses

---

## CP4BA Architecture

### High-Level Architecture

CP4BA follows a layered architecture:

```
┌─────────────────────────────────────────────────────────────┐
│                    User Interface Layer                     │
│  (Business Automation Navigator, Workplace, Process Portal) │
└─────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────┐
│                   Application Services Layer                │
│  (BAW, WFPS, Content Services, Decision Services, BAS)      │
└─────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────┐
│                    Foundation Services Layer                │
│  (Resource Registry, UMS, IAM, Zen, Kafka, OpenSearch)      │
└─────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────┐
│                   Infrastructure Layer                      │
│  (OpenShift, Storage, Databases, LDAP, Networking)          │
└─────────────────────────────────────────────────────────────┘
```

### Custom Resource (CR) Structure

CP4BA deployments are defined using Kubernetes Custom Resources:

- **ICP4ACluster**: Main CR for CP4BA deployments (workflow, content, application patterns)
- **ProcessFederationServer**: Separate CR for PFS deployments
- **Cartridge**: CR for individual capability cartridges

---

## Core Components

### 1. IBM Business Automation Workflow (BAW)

**Purpose**: Comprehensive workflow and case management solution combining BPM and Case capabilities.

**Key Features**:
- Process Designer for BPMN-based workflows
- Case Builder for dynamic case management
- Process Portal for task management
- Integration with FileNet Content Manager
- Support for both authoring and runtime environments

**Deployment Modes**:
- **Authoring**: Full development environment with Process Designer and Case Builder
- **Runtime**: Production environment for executing workflows and cases

**Components**:
- **Workflow Server**: Core BPM engine
- **Process Portal**: User interface for task management
- **Process Designer**: Authoring tool (authoring mode only)
- **Case Builder**: Case management authoring (authoring mode only)
- **Performance metrics**: Business Automation Insights

**Database Requirements**:
- Process Server database (BPMDB)
- Content Platform Engine databases (DOS, DOCS, TOS for case management)

### 2. IBM Workflow Process Service (WFPS)

**Purpose**: Lightweight workflow runtime service for executing processes without full BAW capabilities.

**Key Features**:
- Executes workflows created in BAW authoring environment
- Lighter footprint than full BAW runtime
- Suitable for microservices architectures
- Can be deployed multiple times for different applications

**Deployment Modes**:
- **Authoring**: Development environment (typically uses BAW authoring)
- **Runtime**: Production execution environment

**Components**:
- **WFPS Runtime**: Workflow execution engine
- **REST APIs**: For process interaction
- **Task Management**: Basic task handling

**Database Requirements**:
- Business Automation Workflow database (minimal schema)
- Shared databases (BTS, IM, ZEN)

### 3. IBM FileNet Content Manager (FNCM/Content Cortex)

**Purpose**: Enterprise content management system for storing, managing, and securing business content.

**Key Features**:
- Content Platform Engine (CPE) for content storage
- Business Automation Navigator (BAN/ICN) for user interface
- GraphQL API for modern integrations
- Content Search Services (CSS) for full-text search
- External Share for secure external collaboration

**Components**:
- **Content Platform Engine (CPE)**: Core content repository
- **Business Automation Navigator (Navigator/ICN)**: Web-based UI
- **GraphQL**: Modern API layer
- **Content Search Services (CSS)**: Search indexing (optional)
- **Content Management Interoperability Services (CMIS)**: Standards-based API (optional)

**Object Stores**:
- **GCD (Global Configuration Data)**: System configuration
- **Design Object Store (DOS)**: Case management designs
- **Target Object Store (TOS)**: Case management runtime data
- **Custom Object Stores**: Application-specific content

**Database Requirements**:
- GCD database
- Object Store databases (DOS, TOS, custom OS)
- ICN database

### 4. IBM Business Automation Studio (BAS)

**Purpose**: Unified authoring environment for creating automation artifacts.

**Key Features**:
- Low-code/no-code development
- Integration with multiple CP4BA capabilities
- Playback server for testing
- Application deployment

**Components**:
- **Studio UI**: Web-based authoring interface
- **Playback Server**: Testing environment
- **Application Engine**: Runtime for studio applications

**Database Requirements**:
- Playback database (PBK)
- Application Engine database (AE)
- Automation Workstream Services database (AWS)

### 5. IBM Business Automation Insights (BAI)

**Purpose**: Real-time business analytics and monitoring for automation processes.

**Key Features**:
- Real-time event processing with Apache Flink
- Business performance dashboards
- Process mining capabilities
- Integration with Kafka for event streaming
- OpenSearch for data storage and visualization

**Components**:
- **Flink**: Stream processing engine
- **Kafka**: Event streaming platform
- **OpenSearch**: Data storage and visualization
- **BAI Management**: Configuration and administration

**Event Sources**:
- BPMN processes (BAW/WFPS)
- Case management (ICM)
- Content events (FNCM)
- Business decisions (ODM)
- Workflow operations

**Database Requirements**:
- Uses OpenSearch for data persistence
- No traditional RDBMS required

### 6. IBM Process Federation Server (PFS)

**Purpose**: Federated task management across multiple workflow systems.

**Key Features**:
- Unified task list from multiple BAW/WFPS instances
- Cross-system process visibility
- Saved searches and filters
- REST API for task operations

**Deployment**:
- Separate Custom Resource (ProcessFederationServer kind)
- Can federate multiple BAW and WFPS instances
- Requires configuration of federated systems

**Components**:
- **PFS Server**: Federation engine
- **REST API**: Task management interface
- **Elasticsearch**: Search and indexing (embedded)

---

## Deployment Patterns

CP4BA supports multiple deployment patterns that can be combined:

### 1. Foundation Pattern

**Purpose**: Base infrastructure services required by all other patterns.

**Includes**:
- Resource Registry
- User Management Service (UMS)
- IAM integration with CPFS
- Zen UI framework
- Shared services

**When to Use**:
- Required for all CP4BA deployments
- Can be deployed standalone for WFPS foundation

### 2. Workflow Pattern

**Purpose**: Business process management and workflow automation.

**Includes**:
- Business Automation Workflow (BAW) runtime or authoring
- Workflow Process Service (WFPS)
- Integration with Content services
- Process Portal

**Sub-patterns**:
- `workflow-authoring`: BAW authoring
- `workflow-process-service-authoring`: WFPS authoring (uses BAW authoring)
- `workflow`: BAW runtime
- `workflow-process-service`: WFPS runtime

### 3. Content Pattern

**Purpose**: Enterprise content management.

**Includes**:
- Content Platform Engine (CPE)
- Business Automation Navigator (ICN)
- GraphQL API
- Optional: CSS, CMIS, External Share

### 4. Application Pattern

**Purpose**: Low-code application development and deployment.

**Includes**:
- Business Automation Studio (BAS)
- Application Engine
- Application Designer (optional)

### 5. Decisions Pattern

**Purpose**: Business rules and decision management.

**Includes**:
- Operational Decision Manager (ODM)
- Decision Center
- Decision Server Runtime
- Decision Runner

### 6. Decisions_ADS Pattern

**Purpose**: AI-powered decision services.

**Includes**:
- Automation Decision Services (ADS)
- ADS Designer
- ADS Runtime

### 7. Document Processing Pattern

**Purpose**: Intelligent document processing with AI.

**Includes**:
- Document Processing Designer
- Document Processing Runtime
- AI services integration

---

## Optional Components

### Business Automation Insights (BAI)
- Real-time analytics and monitoring
- Requires Kafka and OpenSearch
- Integrates with workflow, content, and decision services

### Kafka
- Event streaming platform
- Required for BAI
- Can be used for custom integrations

### OpenSearch
- Search and analytics engine
- Required for BAI
- Provides Kibana-like dashboards

### Process Federation Server (PFS)
- Federated task management
- Separate CR deployment
- Connects to multiple workflow systems

### Workflow Assistant
- AI-powered workflow assistance
- Requires GenAI configuration
- Integrates with Watson X

### Workplace Assistant
- AI-powered workplace assistance
- Requires GenAI configuration
- Enhances user experience

### Content Search Services (CSS)
- Full-text search for content
- Integrates with CPE
- Uses embedded search engine

### Task Manager (TM)
- Standalone task management
- Alternative to Process Portal
- Lightweight task handling

---

## Infrastructure Requirements

### OpenShift Platform

**Supported Versions**:
- OpenShift Container Platform 4.12+
- Red Hat OpenShift on IBM Cloud (ROKS)
- Azure Red Hat OpenShift (ARO)
- Red Hat OpenShift Service on AWS (ROSA)

**Platform Types**:
- `OCP`: Standard OpenShift
- `ROKS`: IBM Cloud OpenShift
- `ARO`: Azure OpenShift
- `ROSA`: AWS OpenShift

### Storage Classes

CP4BA requires two types of storage:

**File Storage** (RWX - ReadWriteMany):
- Used for: Shared configuration, logs, content storage
- Examples: NFS, CephFS, IBM Cloud File Storage
- Parameters:
  - `sc_dynamic_storage_classname`
  - `sc_slow_file_storage_classname`
  - `sc_medium_file_storage_classname`
  - `sc_fast_file_storage_classname`

**Block Storage** (RWO - ReadWriteOnce):
- Used for: Databases, high-performance workloads
- Examples: Ceph RBD, IBM Cloud Block Storage
- Parameter: `sc_block_storage_classname`

### Resource Profiles

CP4BA supports three deployment profile sizes:

**Small**:
- Development and testing
- Minimal resource allocation
- Single replica for most components

**Medium**:
- Small production workloads
- Moderate resource allocation
- Limited high availability

**Large**:
- Production workloads
- Full resource allocation
- High availability configuration

### Namespace Configuration

**Deployment Isolation Levels**:

1. **Maximum Isolation** (Recommended for demos/education):
   - Operators in deployment namespace
   - Private catalog in deployment namespace
   - All components in single namespace

2. **Shared Operators**:
   - Operators in separate namespace
   - Multiple deployments share operators
   - Catalog in operator namespace

3. **Cluster-wide**:
   - Operators watch all namespaces
   - Single operator installation
   - Multiple deployments across namespaces

---

## Database Configuration

### Supported Database Types

**PostgreSQL** (Recommended):
- Version: 13+, 16+ recommended
- Deployment options:
  - EDB Postgres (deprecated in v25)
  - Open-source PostgreSQL
  - External managed PostgreSQL
- SSL support: Optional but recommended

**Oracle**:
- Version: 19c+
- Enterprise Edition required for some features
- RAC support available

**Microsoft SQL Server**:
- Version: 2019+
- Enterprise Edition recommended
- Always On support available

**IBM Db2**:
- Version: 11.5+
- HADR support available
- Native integration

### Database Architecture

**Shared vs. Dedicated Databases**:

1. **Single Database Server**:
   - Multiple databases on one server
   - Separate users per component
   - Suitable for development/testing

2. **Multiple Database Servers**:
   - Dedicated servers per capability
   - Better isolation and performance
   - Recommended for production

**Database Naming Convention**:
```
${ENV_PREFIX}_${COMPONENT}_${INSTANCE}

Examples:
- baw_auth_bai_baw_1 (BAW database)
- baw_auth_bai_gcd (GCD database)
- baw_auth_bai_dos (Design Object Store)
```

### Database Users and Permissions

**Key Principle**: Each CP4BA component requires a dedicated database user.

**Common Database Users**:
- `bawadmin`: BAW Process Server
- `bawdocs`: BAW DOCS object store
- `bawdos`: BAW DOS object store
- `bawtos`: BAW TOS object store
- `gcd`: Global Configuration Data
- `icn`: Business Automation Navigator
- `content`: Content object stores
- `ae`: Application Engine
- `pbk`: Playback database
- `app`: Application database
- `aws`: Automation Workstream Services
- `bts_user`: Business Teams Service
- `im_user`: Identity Management
- `zen_user`: Zen services

**Important Restrictions**:
- Cannot share database users between BAW and Workstream Services
- Cannot share database users between different components
- Can share database server but must use different users

### SSL/TLS Configuration

**Database SSL Options**:
- `CP4BA_INST_DB_ONLY_SSL="false"`: Non-SSL connections
- `CP4BA_INST_DB_ONLY_SSL="true"`: SSL-only connections

**SSL Configuration Files**:
- `postgresql.conf`: PostgreSQL configuration
- `postgresql_ssl.conf`: SSL-enabled configuration
- `pg_hba.conf`: Host-based authentication

---

## LDAP and IAM Integration

### LDAP Configuration

**Supported LDAP Types**:
- IBM Security Directory Server (SDS)
- Microsoft Active Directory (AD)
- OpenLDAP
- Custom LDAP

**LDAP Deployment Options**:

1. **Embedded LDAP** (Development/Testing):
   - OpenLDAP pod deployed in CP4BA namespace
   - Pre-populated with test users
   - Automatic configuration
   - Not for production use

2. **External LDAP** (Production):
   - Corporate LDAP/AD server
   - Manual configuration required
   - SSL/TLS recommended

**LDAP Configuration Parameters**:
```yaml
ldap_configuration:
  lc_selected_ldap_type: "Custom"  # or "IBM Security Directory Server", "Microsoft Active Directory"
  lc_ldap_server: "ldap.example.com"
  lc_ldap_port: "389"  # or 636 for SSL
  lc_bind_secret: "ldap-bind-secret"
  lc_ldap_base_dn: "dc=example,dc=com"
  lc_ldap_ssl_enabled: false
  lc_ldap_user_name_attribute: "*:cn"
  lc_ldap_group_base_dn: "dc=example,dc=com"
```

**SCIM Configuration for IAM**:
- Maps LDAP attributes to IAM user properties
- Required for user onboarding
- Configures user and group synchronization

### IAM Integration

**IBM Cloud Pak Foundational Services (CPFS)**:
- Provides Identity and Access Management
- Integrates with LDAP for user authentication
- Manages roles and permissions
- Provides Zen UI framework

**User Onboarding Process**:
1. Users defined in LDAP
2. SCIM configuration maps LDAP to IAM
3. Users onboarded to CPFS IAM
4. Roles assigned in CP4BA components

**Default Admin Configuration**:
- IAM admin user: `cpadmin` (configurable)
- IAM admin group: `AdminsGroup`
- CP4BA admin user: `cp4admin` (configurable)

---

## Security and Certificates

### Certificate Management

**Root CA Certificate**:
- Secret name: `icp4a-root-ca`
- Used for internal service communication
- Automatically generated if not provided
- Signs all internal certificates

**External TLS Certificates**:
- Optional custom certificates for routes
- Secret name: configurable (e.g., `external_tls_certificate_secret`)
- Used for external access to CP4BA services

**ZenService Certificate**:
- Custom certificate for Zen UI entry point
- Recommended for production
- Can use Let's Encrypt or corporate CA
- Configuration:
  ```
  CP4BA_INST_ZS_SOURCE_SECRET="letsencrypt-certs"
  CP4BA_INST_ZS_SOURCE_NAMESPACE="openshift-config"
  CP4BA_INST_ZS_TARGET_SECRET="my-letsencrypt"
  ```

### Secrets Management

**Key Secrets**:
- `ibm-entitlement-key`: IBM Container Registry access
- `ldap-bind-secret`: LDAP bind credentials
- `icp4a-root-ca`: Root CA certificate
- `ibm-iaws-shared-key-secret`: Encryption key
- Database secrets: Per-component database credentials
- Custom XML secrets: Liberty and Lombardi customizations

### Network Policies

**Network Policy Support**:
- Starting v25.0.0: Operators don't create network policies automatically
- Optional: Generate sample network policy templates
- Parameter: `sc_generate_sample_network_policies: true/false`

**Network Policy Templates**:
- Allow-all policy (for testing)
- Restrictive policies (for production)
- Component-specific policies
- Custom policies

### Security Context

**RunAsUser**:
- Optional for non-OCP platforms
- Not supported on OCP and ROKS
- Parameter: `sc_run_as_user`

**Seccomp Profiles**:
- Default: `RuntimeDefault` (OCP 4.11+)
- Options: `RuntimeDefault`, `Localhost`, `Unconfined`
- Custom profiles must exist on worker nodes

**FIPS Mode**:
- Optional FIPS 140-2 compliance
- Parameter: `enable_fips: true/false`
- Requires FIPS-enabled OpenShift

---

## Deployment Topologies

### Single Instance Deployments

**BAW Authoring**:
```
Foundation + BAW Authoring + Content + BAS + BAI
├── 1 BAW Authoring Server
├── Content Platform Engine
├── Business Automation Navigator
├── Business Automation Studio
└── Business Automation Insights
```

**BAW Runtime**:
```
Foundation + BAW Runtime + Content + BAI
├── 1 BAW Runtime Server
├── Content Platform Engine
├── Business Automation Navigator
└── Business Automation Insights
```

**WFPS Runtime**:
```
Foundation + WFPS Runtime + BAI
├── 1 WFPS Runtime Server
└── Business Automation Insights
```

### Multi-Instance Deployments

**Multiple BAW Servers with PFS**:
```
Foundation + Multiple BAW + PFS
├── BAW Server 1 (with Content)
├── BAW Server 2 (with Content)
├── Process Federation Server
└── Shared Content Services
```

**WFPS with PFS**:
```
Foundation + WFPS + PFS (separate CR)
├── WFPS Runtime (ICP4ACluster CR)
└── Process Federation Server (ProcessFederationServer CR)
```

### Separation of Concerns

**Authoring vs. Runtime**:
- Authoring: Development environment
- Runtime: Production environment
- Separate namespaces recommended
- Different resource allocations

**Multi-Environment Strategy**:
```
Development Namespace:
└── BAW Authoring + BAS + Content

Test Namespace:
└── BAW Runtime + Content

Production Namespace:
├── BAW Runtime 1 + Content
├── BAW Runtime 2 + Content
└── PFS
```

---

## Recipe Templates

This section provides an overview of the deployment recipes available in this repository. Each recipe is a complete, tested configuration for a specific CP4BA deployment topology.

### Recipe 1: BAW Authoring with BAI

**File**: `cp4ba-cr-ref-authoring-baw-bai.yaml`  
**Properties**: `env1-authoring-baw-bai.properties`

**Purpose**: Full Business Automation Workflow authoring environment with Business Automation Insights for development and testing.

**Patterns**: `foundation,workflow,application`

**Optional Components**: `baw_authoring,bas,bai,kafka,workflow_assistant,workplace_assistant`

**Key Capabilities**:
- BAW Process Designer for BPMN workflows
- Case Builder for case management
- Business Automation Studio for low-code development
- Business Automation Insights for real-time analytics
- Content management with FileNet
- AI assistants (optional, requires GenAI configuration)

**Use Cases**:
- Development environment for workflow applications
- Process and case design
- Testing and validation
- Training and demonstrations

**Infrastructure**:
- 1 PostgreSQL database server
- 1 OpenLDAP server (embedded)
- Storage: File (RWX) and Block (RWO)
- Profile size: Small (configurable)

**See**: [Recipe Details](../recipes/recipe-authoring-baw-bai.md)

---

### Recipe 2: BAW Runtime with BAI

**File**: `cp4ba-cr-ref-baw-bai.yaml`  
**Properties**: `env1-runtime-baw-bai.properties`

**Purpose**: Production Business Automation Workflow runtime environment with monitoring and analytics.

**Patterns**: `foundation,workflow`

**Optional Components**: `workplace_assistant,bai,kafka,opensearch`

**Key Capabilities**:
- BAW runtime for executing workflows and cases
- Process Portal for task management
- Business Automation Insights for monitoring
- Content management with FileNet
- Workplace assistant (optional)

**Use Cases**:
- Production workflow execution
- Case management operations
- Business process monitoring
- Performance analytics

**Infrastructure**:
- 1 PostgreSQL database server
- 1 OpenLDAP server (embedded)
- Storage: File (RWX) and Block (RWO)
- Profile size: Small (configurable)

**See**: [Recipe Details](../recipes/recipe-runtime-baw-bai.md)

---

### Recipe 3: BAW Runtime with Double PFS

**File**: `cp4ba-cr-ref-baw-double-pfs.yaml`  
**Properties**: `env1-runtime-baw-double-pfs.properties`

**Purpose**: Multiple BAW runtime servers with Process Federation Server for federated task management.

**Patterns**: `foundation,workflow`

**Optional Components**: `pfs`

**Key Capabilities**:
- 2 BAW runtime servers
- Process Federation Server for unified task list
- Content management (BAW 1 with full content, BAW 2 with content only)
- Federated process visibility

**Use Cases**:
- Multi-application workflow deployment
- Federated task management across systems
- Scalable workflow architecture
- Application isolation with shared services

**Infrastructure**:
- 1 PostgreSQL database server
- 1 OpenLDAP server (embedded)
- Storage: File (RWX) and Block (RWO)
- Profile size: Small (configurable)
- PFS deployed separately (ProcessFederationServer CR)

**Configuration Notes**:
- BAW 1: Full configuration with case management
- BAW 2: Content only, no case management
- Both BAW instances have separate navigators
- PFS federates both BAW instances

**See**: [Recipe Details](../recipes/recipe-runtime-baw-double-pfs.md)

---

### Recipe 4: WFPS Authoring with PFS and BAI

**File**: `cp4ba-cr-ref-authoring-wfps-bai.yaml`  
**Properties**: `env1-authoring-wfps-pfs-bai.properties`

**Purpose**: Workflow Process Service authoring environment (uses BAW authoring) with analytics.

**Patterns**: `foundation,workflow-process-service`

**Optional Components**: `wfps_authoring,bai,kafka,opensearch`

**Key Capabilities**:
- BAW authoring for process design
- WFPS runtime configuration
- Business Automation Insights
- Lightweight workflow execution

**Use Cases**:
- Microservices-based workflow development
- Lightweight process authoring
- WFPS application development
- Testing WFPS deployments

**Infrastructure**:
- 1 PostgreSQL database server
- 1 OpenLDAP server (embedded)
- Storage: File (RWX) and Block (RWO)
- Profile size: Small (configurable)

**See**: [Recipe Details](../recipes/recipe-authoring-wfps-pfs-bai.md)

---

### Recipe 5: WFPS Runtime Foundation with PFS and BAI

**File**: `cp4ba-cr-ref-wfps-pfs-bai-foundation.yaml`  
**Properties**: `env1-runtime-wfps-pfs-bai.properties`

**Purpose**: Foundation-only deployment for Workflow Process Service runtime with monitoring.

**Patterns**: `foundation,workflow-process-service`

**Optional Components**: `bai,kafka,opensearch`

**Key Capabilities**:
- Foundation services for WFPS
- Business Automation Insights
- Minimal footprint deployment
- Support for external WFPS instances

**Use Cases**:
- WFPS runtime foundation
- Shared services for multiple WFPS instances
- Monitoring infrastructure
- Lightweight workflow execution environment

**Infrastructure**:
- 1 PostgreSQL database server
- 1 OpenLDAP server (embedded)
- Storage: File (RWX) and Block (RWO)
- Profile size: Small (configurable)
- WFPS instances deployed separately

**Configuration Notes**:
- Foundation-only CR
- WFPS instances use separate CRs
- PFS deployed separately (ProcessFederationServer CR)
- Shared BAI for all WFPS instances

**See**: [Recipe Details](../recipes/recipe-runtime-wfps-pfs-bai-foundation.md)

---

## Recipe Comparison Matrix

| Feature | BAW Auth | BAW Runtime | BAW Double PFS | WFPS Auth | WFPS Foundation |
|---------|----------|-------------|----------------|-----------|-----------------|
| **Pattern** | workflow | workflow | workflow | wfps | wfps |
| **Mode** | Authoring | Runtime | Runtime | Authoring | Runtime |
| **Process Designer** | ✓ | ✗ | ✗ | ✓ | ✗ |
| **Case Builder** | ✓ | ✗ | ✗ | ✗ | ✗ |
| **BAS** | ✓ | ✗ | ✗ | ✗ | ✗ |
| **BAI** | ✓ | ✓ | ✗ | ✓ | ✓ |
| **Content** | ✓ | ✓ | ✓ | ✗ | ✗ |
| **PFS** | ✗ | ✗ | ✓ | ✗ | ✗ |
| **Multiple Servers** | ✗ | ✗ | ✓ (2) | ✗ | ✗ |
| **AI Assistants** | Optional | Optional | ✗ | ✗ | ✗ |
| **Use Case** | Development | Production | Multi-app | Development | Foundation |

---

## Best Practices

### Deployment Planning

1. **Start with Foundation**: Always select foundation pattern capabilities first
2. **Separate Environments**: Use different namespaces for dev/test/prod
3. **Resource Planning**: Size appropriately based on workload
4. **Storage Selection**: Choose appropriate storage classes
5. **Database Strategy**: Plan database topology early

### Security

1. **Use SSL/TLS**: Enable SSL for databases and LDAP in production
2. **Custom Certificates**: Use corporate CA certificates
3. **Network Policies**: Implement restrictive network policies
4. **Secrets Management**: Use external secrets management (e.g., Vault)
5. **RBAC**: Implement proper role-based access control

### High Availability

1. **Multiple Replicas**: Configure multiple replicas for critical components
2. **Database HA**: Use database clustering (RAC, Always On, HADR)
3. **Storage HA**: Use replicated storage solutions
4. **Load Balancing**: Configure proper load balancing
5. **Backup Strategy**: Implement regular backup procedures

### Performance

1. **Resource Limits**: Set appropriate CPU and memory limits
2. **Database Tuning**: Optimize database configuration
3. **Storage Performance**: Use fast storage for databases
4. **Monitoring**: Implement comprehensive monitoring with BAI
5. **Scaling**: Plan for horizontal and vertical scaling

### Maintenance

1. **Version Control**: Track all configuration changes
2. **Documentation**: Document customizations and configurations
3. **Testing**: Test all changes in non-production first
4. **Backup**: Regular backups of databases and configurations
5. **Upgrade Planning**: Plan and test upgrades carefully

---

## Troubleshooting

### Common Issues

**Pod Startup Failures**:
- Check resource availability
- Verify storage class configuration
- Review pod logs
- Check secrets and configmaps

**Database Connection Issues**:
- Verify database credentials
- Check network connectivity
- Validate SSL configuration
- Review database logs

**LDAP Integration Problems**:
- Verify LDAP connectivity
- Check bind credentials
- Validate LDAP filters
- Review SCIM configuration

**Performance Issues**:
- Check resource utilization
- Review database performance
- Analyze network latency
- Monitor storage I/O

### Diagnostic Commands

```bash
# Check pod status
oc get pods -n <namespace>

# View pod logs
oc logs <pod-name> -n <namespace>

# Describe pod for events
oc describe pod <pod-name> -n <namespace>

# Check custom resource status
oc get icp4acluster -n <namespace>
oc describe icp4acluster <cr-name> -n <namespace>

# View operator logs
oc logs -l name=ibm-cp4a-operator -n <namespace>

# Check storage
oc get pvc -n <namespace>
oc get pv

# Network connectivity test
oc exec <pod-name> -n <namespace> -- curl <service-url>
```

---

## References

### Official Documentation

- [CP4BA 25.0.1 Documentation](https://www.ibm.com/docs/en/cloud-paks/cp-biz-automation/25.0.1)
- [BAW Documentation](https://www.ibm.com/docs/en/baw/25.0.x)
- [FileNet Content Manager](https://www.ibm.com/docs/en/cloud-paks/cp-biz-automation/25.0.1?topic=deployments-installing-cp4ba-filenet-content-manager-production-deployment)
- [WFPS Documentation](https://www.ibm.com/docs/en/cloud-paks/cp-biz-automation/25.0.1?topic=ipd-installing-cp4ba-workflow-process-service-runtime-production-deployment)
- [PFS Documentation](https://www.ibm.com/docs/en/cloud-paks/cp-biz-automation/25.0.1?topic=deployments-installing-cp4ba-process-federation-server-production-deployment)
- [CPFS Documentation](https://www.ibm.com/docs/en/cloud-paks/foundational-services/4.x_cd)

### Installation Guides

- [Multi-Pattern Production Deployment](https://www.ibm.com/docs/en/cloud-paks/cp-biz-automation/25.0.1?topic=deployments-installing-cp4ba-multi-pattern-production-deployment)
- [Multiple Versions on Same Cluster](https://www.ibm.com/docs/en/cloud-paks/cp-biz-automation/25.0.1?topic=deployments-installing-multiple-versions-cp4ba-same-openshift-cluster)

---

## Glossary

- **BAI**: Business Automation Insights
- **BAS**: Business Automation Studio
- **BAW**: Business Automation Workflow
- **BPM**: Business Process Management
- **BPMN**: Business Process Model and Notation
- **CPE**: Content Platform Engine
- **CPFS**: Cloud Pak Foundational Services
- **CR**: Custom Resource
- **CSS**: Content Search Services
- **FNCM**: FileNet Content Manager
- **GCD**: Global Configuration Data
- **IAM**: Identity and Access Management
- **ICN**: IBM Content Navigator (Business Automation Navigator)
- **ICM**: IBM Case Manager
- **LDAP**: Lightweight Directory Access Protocol
- **ODM**: Operational Decision Manager
- **PFS**: Process Federation Server
- **SCIM**: System for Cross-domain Identity Management
- **TLS**: Transport Layer Security
- **UMS**: User Management Service
- **WFPS**: Workflow Process Service

---

## Document History

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | 2026-06-23 | Bob (AI Assistant) | Initial comprehensive knowledge base creation |

---

*This knowledge base is maintained for educational purposes and should be adapted for specific production requirements.*