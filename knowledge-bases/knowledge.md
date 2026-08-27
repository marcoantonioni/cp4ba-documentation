# CP4BA v26.0.0 — Knowledge Base

> **Scope**: IBM Cloud Pak for Business Automation (CP4BA) v26.0.0 — OpenShift containerized deployments only.  
> **Sources**: `cp4ba-installations/configs26`, `cp4ba-installations/templates26`, `references/cp4ba-yamls`, `cp4ba-installations/scripts`.

---

## Table of Contents

1. [Architecture Overview](#1-architecture-overview)
2. [CP4BA Foundation Pattern](#2-cp4ba-foundation-pattern)
3. [Business Automation Workflow (BAW)](#3-business-automation-workflow-baw)
4. [Workflow Process Service (WFPS)](#4-workflow-process-service-wfps)
5. [Process Federation Server (PFS)](#5-process-federation-server-pfs)
6. [IBM FileNet Content Manager / Content Cortex (FNCM / CORTEX)](#6-ibm-filenet-content-manager--content-cortex-fncm--cortex)
7. [Decision Intelligence – ADS (DICM / ADS)](#7-decision-intelligence--ads-dicm--ads)
8. [Operational Decision Manager (ODM)](#8-operational-decision-manager-odm)
9. [Business Automation Insights (BAI)](#9-business-automation-insights-bai)
10. [Application Engine (AE / App Designer)](#10-application-engine-ae--app-designer)
11. [IBM Robotic Process Automation (RPA)](#11-ibm-robotic-process-automation-rpa)
12. [IBM Cloud Pak Foundational Services (CPFS)](#12-ibm-cloud-pak-foundational-services-cpfs)
13. [Storage Classes](#13-storage-classes)
14. [Database Configuration](#14-database-configuration)
15. [LDAP and IAM Configuration](#15-ldap-and-iam-configuration)
16. [Deployment Patterns and Optional Components](#16-deployment-patterns-and-optional-components)
17. [Installation Tooling – cp4ba-one-shot-installation.sh](#17-installation-tooling--cp4ba-one-shot-installationsh)
18. [Recipe Catalogue](#18-recipe-catalogue)
19. [Reference Topology Files (ibm_cp4a_cr_production_FC_*)](#19-reference-topology-files-ibm_cp4a_cr_production_fc_)

---

## 1. Architecture Overview

CP4BA v26.0.0 is deployed as an OpenShift operator-managed workload. The central Custom Resource kind is `ICP4ACluster` (API version `icp4a.ibm.com/v1`). A single CR configures all enabled capabilities through the `spec` section.

**Key architectural decisions in these recipes:**

| Concern | Choice |
|---|---|
| Operator scope | Namespace-scoped (private catalog, `CP4BA_AUTO_PRIVATE_CATALOG=Yes`) |
| Operator placement | Same namespace as workloads (`CP4BA_AUTO_SEPARATE_OPERATOR=No`) |
| Multi-AZ | Disabled (`sc_is_multiple_az: false`) |
| Egress | Not restricted (`sc_restricted_internet_access: false`) |
| FIPS | Disabled |
| Image repository | `cp.icr.io` |
| Image pull secret | `ibm-entitlement-key` |
| Root CA secret | `icp4a-root-ca` |
| Ingress | OCP Routes (not Nginx ingress) |
| License type | `production` (all recipes) |

---

## 2. CP4BA Foundation Pattern

**Pattern name**: `foundation`

The Foundation pattern is the baseline for every CP4BA deployment. It installs the core shared services required by all capabilities.

### Foundation Components

| Component | Description |
|---|---|
| IBM Cloud Pak foundational services (CPFS/Zen) | License service, IAM, Identity Provider bridge, CP4D control-plane (Zen) |
| Business Automation Navigator (ICN) | Central web UI for accessing case, content and workflow |
| Business Automation Studio (BAS / BAStudio) | Authoring environment for processes, cases, decisions, and applications |
| Resource Registry | etcd-based service registry for capability discovery |
| Business Team Service (BTS) | Team management service used by Case management |
| Business Automation Machine Learning (BAML) | Optional ML service |

### Foundation CR Structure

```yaml
spec:
  shared_configuration:
    sc_deployment_patterns: "foundation"
    sc_optional_components: ""   # extend per recipe
    sc_deployment_type: "Production"
    sc_deployment_platform: "OCP"
    sc_deployment_profile_size: "small|medium|large"
    sc_image_repository: cp.icr.io
    storage_configuration:
      sc_dynamic_storage_classname: "<file-sc>"
      sc_block_storage_classname:   "<block-sc>"
    sc_iam:
      default_admin_username: "cpadmin"
    sc_content_initialization: true
    sc_content_verification: false
    encryption_key_secret: ibm-iaws-shared-key-secret
```

---

## 3. Business Automation Workflow (BAW)

**Pattern name**: `workflow`  
**Optional components**: `baw_authoring` (authoring only), `workflow_assistant`, `workplace_assistant`

BAW provides Business Process Management (BPM) and Case Management capabilities. It runs on top of IBM FileNet Content Manager and uses multiple PostgreSQL databases.

### BAW Modes

| Mode | Pattern | Optional Component | Use-case |
|---|---|---|---|
| Runtime only | `workflow` | — | Production execution environment |
| Authoring + Runtime | `workflow` | `baw_authoring` | Full dev/test authoring environment |

### BAW Database Requirements

Each BAW instance requires multiple databases on PostgreSQL:

| DB Variable | Purpose |
|---|---|
| `CP4BA_INST_GCD_DB_NAME` | FNCM Global Configuration Database |
| `CP4BA_INST_ICN_DB_NAME` | IBM Content Navigator |
| `CP4BA_INST_DOCS_DB_NAME` | BAW Documents object store |
| `CP4BA_INST_DOS_DB_NAME` | BAW Design object store |
| `CP4BA_INST_TOS_DB_NAME` | BAW Target object store |
| `CP4BA_INST_CONTENT_DB_NAME` | Content object store |
| `CP4BA_INST_OS1_DB_NAME` | Additional object store (OS1) |
| `CP4BA_INST_AWSDB_DB_NAME` | Advanced Work Services DB |

### BAW CR Key Sections

```yaml
spec:
  baw_configuration:
    - name: "bawins1"
      workflow_authoring_configuration:  # present only in authoring mode
        ...
      database:
        type: postgresql
        ssl_enabled: true
      content_integration:
        domain_name: "P8Domain"
```

### BAW License Types

- `sc_deployment_baw_license`: `production` | `non-production` | `user`
- `sc_deployment_fncm_license`: `production` | `non-production` | `concurrent-user` | `authorized-user` | `user`

---

## 4. Workflow Process Service (WFPS)

**Pattern name**: `workflow-process-service`  
**Optional component**: `wfps_authoring`

WFPS is a lightweight workflow runtime aimed at cloud-native scenarios. It provides a simplified process execution engine without Case Management or FNCM dependencies.

### WFPS Authoring Mode

When `wfps_authoring` is included:
- Business Automation Studio (BAS) is deployed
- BAI/Kafka/OpenSearch can be optionally added for insights

### WFPS CR Key Sections

```yaml
spec:
  workflow_process_service_configuration:
    server:
      - name: "wfps-authoring"
        ...
```

### Key Recipe

- **authoring-wfps-bai**: WFPS Authoring + BAI + PFS (namespace: `cp4ba-wfps-pfs-bai-auth`)

---

## 5. Process Federation Server (PFS)

**Pattern/Optional component**: `pfs` (requires `foundation` or `workflow`)

PFS federates multiple BAW/WFPS instances into a unified task list accessible from a single portal (Workplace or Federated Portal). It indexes tasks in OpenSearch/Elasticsearch.

### PFS Dependencies

- OpenSearch (or Elasticsearch) for task index store
- Kafka (event streaming for task events)
- BAI (for task event routing)

### PFS Custom Resource

PFS can also be deployed as a **separate CR** of kind `ProcessFederationServer`:

```yaml
apiVersion: icp4a.ibm.com/v1
kind: ProcessFederationServer
metadata:
  name: pfs
spec:
  pfs_configuration:
    service:
      ...
    federated_data_repository:
      ...
```

### PFS Configuration in ICP4ACluster

```yaml
spec:
  pfs_configuration:
    pfs:
      service:
        ...
```

---

## 6. IBM FileNet Content Manager / Content Cortex (FNCM / CORTEX)

**Pattern name**: `content`  
**Optional components**: `cmis`, `css`, `tm`, `ier`, `iccsap`

FNCM provides enterprise content management. In CP4BA it underpins BAW Case Management.

### FNCM Components

| Component | Description |
|---|---|
| CPE (Content Platform Engine) | Core content repository engine |
| CSS (Content Search Services) | Full-text search index |
| CMIS | CMIS protocol endpoint |
| GraphQL | GraphQL API gateway for content |
| IER (IBM Enterprise Records) | Records management |
| TM (Task Manager) | Task management integration |
| ICCSAP | SAP integration connector |

### FNCM CR Key Sections

```yaml
spec:
  ecm_configuration:
    fncm_secret_name: "ibm-fncm-secret"
    cpe:
      datavolume:
        existing_pvc_for_cpe_cfgstore: ...
    css:
      ...
    navigator_configuration:
      ...
```

### FNCM Database Requirements

| DB Variable | Purpose |
|---|---|
| `CP4BA_INST_GCD_DB_NAME` | Global Configuration Database (GCD) |
| `CP4BA_INST_ICN_DB_NAME` | Navigator DB |
| `CP4BA_INST_OS1_DB_NAME` | First Object Store |

---

## 7. Decision Intelligence – ADS (DICM / ADS)

**Pattern name**: `decisions_ads`  
**Optional components**: `ads_designer`, `ads_runtime`

IBM Decision Intelligence Client Managed Software (ADS) provides a low-code/no-code decision authoring environment and a high-throughput runtime for AI-infused business decisions.

### ADS Components

| Component | Description |
|---|---|
| Decision Designer | Authoring environment for decision models (DMN-based) |
| Decision Runtime | Execution engine for deployed decision services |
| REST API service | OpenAPI endpoint for invoking decisions |
| Credentials Service | Manages external service credentials (e.g., ML model endpoints) |
| Git Service | Version control for decision projects |
| Parsing Service | Natural-language parse for decision tables |
| Run Service | Orchestrates decision execution |

### ADS Modes

| Mode | Patterns | Optional Components |
|---|---|---|
| Authoring + Runtime | `foundation,decisions_ads` | `ads_designer,ads_runtime,bas` |
| Runtime only | `foundation,decisions_ads` | `ads_runtime` |

### ADS CR Key Sections

```yaml
spec:
  ads_configuration:
    decision_designer:
      ...
    decision_runtime:
      ...
    rest_api:
      ...
```

### ADS Database Requirements

| DB Variable | Purpose |
|---|---|
| `CP4BA_INST_ADS_DESIGNER_DB_NAME` | ADS Designer state store |
| `CP4BA_INST_ADS_RUNTIME_DB_NAME` | ADS Runtime state store |

---

## 8. Operational Decision Manager (ODM)

**Pattern name**: `decisions`  
**Optional components**: `decisionCenter`, `decisionRunner`, `decisionServerRuntime`

ODM provides a rule-based decision management system based on ILOG JRules / Business Rules.

### ODM Components

| Component | Description |
|---|---|
| Decision Server Runtime | Executes rule sets |
| Decision Server Console | Management console for rule sets |
| Decision Center | Authoring environment for business rules |
| Decision Runner | Test-and-simulate runner for rules |

### ODM Modes

| Mode | Patterns | Optional Components | Use-case |
|---|---|---|---|
| Authoring (full) | `foundation,decisions` | — | Includes all ODM components |
| Runtime only | `foundation,decisions` | — | Minimal, no Decision Center |

### ODM CR Key Sections

```yaml
spec:
  odm_configuration:
    decisionCenter:
      enabled: true
      ...
    decisionServerRuntime:
      enabled: true
      ...
    decisionServerConsole:
      enabled: true
    decisionRunner:
      enabled: true
```

---

## 9. Business Automation Insights (BAI)

**Optional component**: `bai` (requires `foundation`)

BAI provides real-time process analytics via event streaming (Kafka) and an OpenSearch-backed dashboard (Kibana-compatible).

### BAI Dependencies

| Component | Purpose |
|---|---|
| Kafka (via Strimzi) | Event ingestion pipeline |
| OpenSearch | Analytics data store and search |
| BAI Processing | Event processor and aggregator |

### BAI Configuration

```yaml
spec:
  bai_configuration:
    ...
```

When `bai` is listed as optional component, the installer also enables `kafka` and `opensearch` to provide the underlying event bus and data store.

---

## 10. Application Engine (AE / App Designer)

**Pattern name**: `application`  
**Optional components**: `app_designer`, `ae_data_persistence`

The Application Engine provides a low-code application builder and runtime. App Designer is the authoring surface (integrated in BAStudio).

### AE CR Key Sections

```yaml
spec:
  application_engine_configuration:
    - name: "aae"
      database:
        ...
```

### AE Database Requirements

| DB Variable | Purpose |
|---|---|
| `CP4BA_INST_AE_DB_NAME` | Application Engine DB |
| `CP4BA_INST_AEOS_DB_NAME` | App Engine Object Store |
| `CP4BA_INST_APP_DB_NAME` | Application DB |

---

## 11. IBM Robotic Process Automation (RPA)

RPA in CP4BA v26 is deployed **outside** the `ICP4ACluster` CR. It uses its own operator and a separate Microsoft SQL Server database.

### RPA Architecture

```
OCP Namespace: cp4ba-rpa
 ├─ ICP4ACluster CR          (foundation pattern only – provides IAM/LDAP/ICN context)
 ├─ IBM MQ Operator           (v3.9 channel)
 ├─ IBM RPA Operator          (v3.3 channel)
 │   └─ RpaServer CR          (instance: rpa / tenant: ibm)
 └─ Microsoft SQL Server pod  (mssql/server:2025-latest)
```

### RPA Operator Versions

| Component | Channel | Starting CSV | Image |
|---|---|---|---|
| IBM MQ | `v3.9` | — | `icr.io/cpopen/ibm-mq-operator-catalog@sha256:...` |
| IBM RPA | `v3.3` | `ibm-automation-rpa.v3.3.0` | `icr.io/cpopen/ibm-rpa-operator-catalog:latest` |

### RPA Database – Microsoft SQL Server

RPA requires **5 SQL Server databases**:

| Database | Purpose |
|---|---|
| `address` | Address book / user directory |
| `automation` | Automation assets (scripts, bots) |
| `knowledge` | Knowledge base for NLP |
| `wordnet` | WordNet lexical database |
| `audit` | Audit and tracking |

Connection strings use format:
```
Data Source=<service>.<ns>.svc.cluster.local\<instance>,<port>;
Initial Catalog=<db>;User ID=<user>;Password=<pwd>;
Connect Timeout=30;Encrypt=False;...
```

### RPA CP4BA CR (Foundation only)

The `ICP4ACluster` CR for RPA uses only the `foundation` pattern with no optional components:

```yaml
spec:
  shared_configuration:
    sc_deployment_patterns: "foundation"
    sc_optional_components: ""
```

This provides the Zen/IAM framework that RPA integrates with for SSO.

---

## 12. IBM Cloud Pak Foundational Services (CPFS)

CPFS v4.x is the shared services layer for all CP4BA capabilities. In these recipes, it is deployed **namespace-scoped** (private catalog pattern).

### CPFS Key Services

| Service | Purpose |
|---|---|
| License Service | Tracks capacity-based license usage |
| IAM (Keycloak-based) | Identity and access management |
| Zen (CPD) | Cloud Pak control plane UI |
| cert-manager | TLS certificate management |
| Postgres (CPFS internal) | Stores BTS, ICN, Zen, IM state |

### CPFS Namespace Isolation Variables

```bash
CP4BA_AUTO_NAMESPACE="${CP4BA_INST_NAMESPACE}"
CP4BA_AUTO_OPERATOR_NAMESPACE="${CP4BA_INST_NAMESPACE}"
CP4BA_AUTO_CS_SERVICE_NAMESPACE="${CP4BA_INST_NAMESPACE}"
CP4BA_AUTO_PRIVATE_CATALOG="Yes"
CP4BA_AUTO_SEPARATE_OPERATOR="No"
CP4BA_AUTO_ALL_NAMESPACES="No"
```

---

## 13. Storage Classes

All recipes use OCS (OpenShift Container Storage) storage classes. Both file and block storage are required.

### Default Storage Classes

| Type | Storage Class | Used for |
|---|---|---|
| File (RWX) | `ocs-external-storagecluster-cephfs` | Most CP4BA PVCs |
| Block (RWO) | `ocs-external-storagecluster-ceph-rbd` | Block-requiring PVCs |

### Alternative Storage Classes (commented in configs)

```bash
# NFS
CP4BA_INST_SC_FILE="managed-nfs-storage"
CP4BA_INST_SC_BLOCK="managed-nfs-storage"

# OCS internal
CP4BA_INST_SC_FILE="ocs-storagecluster-cephfs"
CP4BA_INST_SC_BLOCK="ocs-storagecluster-ceph-rbd"
```

### CR Storage Section

```yaml
spec:
  shared_configuration:
    storage_configuration:
      sc_dynamic_storage_classname: "${CP4BA_INST_SC_FILE}"
      sc_slow_file_storage_classname: "${CP4BA_INST_SC_FILE}"
      sc_medium_file_storage_classname: "${CP4BA_INST_SC_FILE}"
      sc_fast_file_storage_classname: "${CP4BA_INST_SC_FILE}"
      sc_block_storage_classname: "${CP4BA_INST_SC_BLOCK}"
```

---

## 14. Database Configuration

### PostgreSQL (Primary – CP4BA components)

All CP4BA capabilities (BAW, ADS, ODM, AE, ICN, BTS) use PostgreSQL. The tooling deploys a dedicated PostgreSQL StatefulSet per deployment.

#### Key Variables

| Variable | Default | Description |
|---|---|---|
| `CP4BA_INST_DB` | `true` | Deploy local PostgreSQL pod |
| `CP4BA_INST_DB_USE_EDB` | `false` | Use EDB operator (deprecated) |
| `CP4BA_INST_DB_SERVER_TYPE` | `postgresql` | DB type |
| `CP4BA_INST_DB_SERVER_PORT` | `5432` | Port |
| `CP4BA_INST_DB_ONLY_SSL` | `true` | Enforce SSL-only connections |
| `CP4BA_INST_DB_STORAGE_SIZE` | `10Gi` | PVC size per DB instance |
| `CP4BA_INST_DB_OSS_IMAGE` | `postgres:18.1` | OSS PostgreSQL image |
| `CP4BA_INST_DB_INSTANCES` | `1` | Number of DB server instances |

#### SSL Configuration

```bash
CP4BA_INST_DB_POSTGRES_CR_SSL_TEMPLATE="../templates-db-configs/pg-ssl.yaml"
CP4BA_INST_DB_POSTGRES_CONF_SSL_TEMPLATE="../templates-db-configs/postgresql_ssl.conf"
```

SSL service name pattern: `<cr-name>-ssl-rw.<namespace>.svc.cluster.local`

#### Foundation DB (common to all deployments)

| DB Variable | Purpose |
|---|---|
| `CP4BA_INST_ICN_DB_NAME` | IBM Content Navigator |
| `CP4BA_INST_DB_BTS_USER/PWD` | Business Team Service |
| `CP4BA_INST_DB_IM_USER/PWD` | Identity Management |
| `CP4BA_INST_DB_ZEN_USER/PWD` | Zen/CPD control plane |

### Microsoft SQL Server (RPA only)

RPA requires SQL Server (`mssql/server:2025-latest`) with TCP port `1433` (NodePort `31433` for external access).

---

## 15. LDAP and IAM Configuration

### LDAP

All deployments optionally install a local OpenLDAP pod. The LDAP configuration is externalized to `_cfg-production-ldap-domain.properties`.

```yaml
spec:
  ldap_configuration:
    lc_selected_ldap_type: "Custom"
    lc_ldap_server: "<host>"
    lc_ldap_port: "<port>"
    lc_ldap_ssl_enabled: false
    lc_ldap_user_name_attribute: "*:cn"
    lc_ldap_group_membership_search_filter: "(&(cn=%v)(objectclass=groupOfNames))"
    custom:
      lc_user_filter: "(&(cn=%v)(objectclass=person))"
      lc_group_filter: "(&(cn=%v)(objectclass=groupOfNames))"
```

### SCIM Configuration (for IAM sync)

```yaml
scim_configuration_iam:
  user_unique_id_attribute: "uid"
  user_name_attribute: "cn"
  group_object_class_attribute: "groupOfNames"
```

### IAM Admin Defaults

| Variable | Default |
|---|---|
| `CP4BA_INST_IAM_ADMIN_USER` | `cpadmin` |
| `CP4BA_INST_IAM_ADMIN_GROUP` | `AdminsGroup` |
| `CP4BA_INST_PAKBA_ADMIN_USER` | `cp4admin` |

---

## 16. Deployment Patterns and Optional Components

### Patterns

| Pattern | Provides |
|---|---|
| `foundation` | CPFS, Zen, ICN, BAStudio, ResourceRegistry, BTS |
| `workflow` | BAW (BPM + Case), FNCM/CPE, ECM |
| `workflow-process-service` | WFPS (lightweight BPM, no Case) |
| `application` | Application Engine (low-code apps) |
| `decisions` | ODM (rule-based decisions) |
| `decisions_ads` | ADS (AI/ML-infused decisions) |
| `content` | Full FNCM (CPE, CSS, CMIS, GraphQL, IER, TM) |
| `document_processing` | Document Processing (ADP) |

### Optional Components Reference

| Optional Component | Pattern Required | Description |
|---|---|---|
| `baw_authoring` | `workflow` | BAW authoring mode |
| `bas` | `foundation` | Business Automation Studio |
| `bai` | `foundation` | Business Automation Insights |
| `pfs` | `workflow` or `workflow-process-service` | Process Federation Server |
| `kafka` | `foundation` | Kafka event bus (Strimzi) |
| `opensearch` | `foundation` | OpenSearch cluster |
| `ads_designer` | `decisions_ads` | ADS decision authoring |
| `ads_runtime` | `decisions_ads` | ADS decision runtime |
| `app_designer` | `application` | Application Designer (in BAStudio) |
| `ae_data_persistence` | `application` | App Engine data persistence |
| `wfps_authoring` | `workflow-process-service` | WFPS authoring mode |
| `workflow_assistant` | `workflow` | Workflow AI assistant |
| `workplace_assistant` | `workflow` | Workplace AI assistant |
| `cmis` | `content` | CMIS protocol endpoint |
| `css` | `content` | Content Search Services |
| `tm` | `content` | Task Manager |
| `ier` | `content` | IBM Enterprise Records |

---

## 17. Installation Tooling – cp4ba-one-shot-installation.sh

Located at: `cp4ba-installations/scripts/cp4ba-one-shot-installation.sh`

This is the **only supported installation script** for all recipes. It is a single-shot orchestrator that reads a `.properties` configuration file and performs end-to-end deployment.

### Usage

```bash
./cp4ba-one-shot-installation.sh \
  -c <full-path-to-config-file> \
  [-t] \
  [-p <cert-kubernetes-scripts-folder>] \
  [-m] \
  [-v <case-package-manager-version>] \
  [-k <cert-kubernetes-version>] \
  [-d <target-folder-for-case-package-manager>] \
  [-o] \
  [-x]
```

### Options

| Flag | Argument | Description |
|---|---|---|
| `-c` | `<full-path>` | **(Required)** Full path to the `.properties` configuration file |
| `-t` | — | Test configuration only (dry run), exits without installing |
| `-p` | `<folder>` | Use previously installed CP4BA Case Manager scripts folder (e.g., `<path>/cert-kubernetes/scripts`). Mutually exclusive with `-m` |
| `-m` | — | Install a fresh Case Package Manager |
| `-v` | `<version>` | Case Package Manager version (e.g., `5.1.0`). Optional; installs latest if omitted |
| `-k` | `<version>` | cert-kubernetes version |
| `-d` | `<folder>` | Target folder for Case Package Manager download. Mandatory when `-m` is set |
| `-o` | — | Skip operator installation |
| `-x` | — | Enable trace/debug output |

### Prerequisites

The script requires sibling repositories cloned alongside the installation repo:

```
parent-folder/
  cp4ba-installations/        ← this repo
  cp4ba-casemanager-setup/    ← Case Manager setup scripts
  cp4ba-idp-ldap/             ← LDAP/IAM setup scripts
  cp4ba-utilities/            ← Utilities (TLS, etc.)
  cp4ba-logger/               ← Logging library
```

### Typical Invocation Examples

```bash
# Test configuration only
./cp4ba-one-shot-installation.sh \
  -c ../configs26/env1-authoring-baw.properties \
  -t

# Full install with fresh Case Package Manager
./cp4ba-one-shot-installation.sh \
  -c ../configs26/env1-authoring-baw.properties \
  -m \
  -d /opt/cp4ba-cmgr

# Full install using existing Case Package Manager
./cp4ba-one-shot-installation.sh \
  -c ../configs26/env1-authoring-baw.properties \
  -p /opt/cp4ba-cmgr/cert-kubernetes/scripts

# Install with trace enabled, skip operators
./cp4ba-one-shot-installation.sh \
  -c ../configs26/env1-runtime-baw-bai.properties \
  -p /opt/cp4ba-cmgr/cert-kubernetes/scripts \
  -o -x
```

---

## 18. Recipe Catalogue

The following recipes are available as individual files in the `recipes/` folder.

| Recipe File | Config File | Namespace | Patterns | Key Optional Components |
|---|---|---|---|---|
| `recipe-name-authoring-baw.md` | `env1-authoring-baw.properties` | `cp4ba-baw-auth` | `foundation,workflow` | `baw_authoring,bas,workflow_assistant,workplace_assistant` |
| `recipe-name-authoring-baw-bai.md` | `env1-authoring-baw-bai.properties` | `cp4ba-baw-bai-auth` | `foundation,workflow` | `baw_authoring,bas,bai,pfs,kafka,opensearch,workflow_assistant,workplace_assistant` |
| `recipe-name-authoring-baw-bai-ae.md` | `env1-authoring-baw-bai-ae.properties` | `cp4ba-baw-bai-ae-auth` | `foundation,workflow,application` | `baw_authoring,bas,app_designer,bai,pfs,kafka,opensearch,workflow_assistant,workplace_assistant` |
| `recipe-name-authoring-baw-bai-ae-ads.md` | `env1-authoring-baw-bai-ae-ads.properties` | `cp4ba-baw-bai-ae-ads-auth` | `foundation,workflow,application,decisions_ads` | `baw_authoring,bas,app_designer,ads_designer,ads_runtime,bai,pfs,kafka,opensearch,workflow_assistant,workplace_assistant` |
| `recipe-name-authoring-decision-ads.md` | `env1-authoring-ads.properties` | `cp4ba-ads-auth` | `foundation,decisions_ads` | `ads_designer,ads_runtime,bas` |
| `recipe-name-authoring-decision-ads-bai.md` | `env1-authoring-ads-bai.properties` | `cp4ba-ads-bai-auth` | `foundation,decisions_ads` | `ads_designer,ads_runtime,bas,bai` |
| `recipe-name-authoring-decision-odm.md` | `env1-authoring-odm.properties` | `cp4ba-odm-auth` | `foundation,decisions` | *(none)* |
| `recipe-name-authoring-decision-odm-bai.md` | `env1-authoring-odm-bai.properties` | `cp4ba-odm-bai-auth` | `foundation,decisions` | `bai` |
| `recipe-name-authoring-wfps-bai.md` | `env1-authoring-wfps-pfs-bai.properties` | `cp4ba-wfps-pfs-bai-auth` | `foundation,workflow-process-service` | `wfps_authoring,bai,kafka,opensearch` |
| `recipe-name-baw-bai.md` | `env1-runtime-baw-bai.properties` | `cp4ba-baw-bai-prod` | `foundation,workflow` | `bai,kafka,opensearch,workflow_assistant,workplace_assistant` |
| `recipe-name-baw-bai-ae.md` | `env1-runtime-baw-bai-ae.properties` | `cp4ba-baw-bai-ae-prod` | `foundation,workflow,application` | `bai,kafka,opensearch,workflow_assistant,workplace_assistant,app_designer` |
| `recipe-name-decision-ads.md` | `env1-runtime-ads.properties` | `cp4ba-ads-prod` | `foundation,decisions_ads` | `ads_runtime` |
| `recipe-name-decision-ads-bai.md` | `env1-runtime-ads-bai.properties` | `cp4ba-ads-bai-prod` | `foundation,decisions_ads` | `ads_runtime,bai` |
| `recipe-name-decision-odm.md` | `env1-runtime-odm.properties` | `cp4ba-odm` | `foundation,decisions` | *(none)* |
| `recipe-name-decision-odm-bai.md` | `env1-runtime-odm-bai.properties` | `cp4ba-odm-bai` | `foundation,decisions` | `bai` |
| `recipe-name-foundation.md` | `env1-runtime-opensearch-foundation.properties` / `env1-runtime-os-bai-pfs.properties` | `cp4ba-opensearch-prod` / `cp4ba-os-bai-pfs-prod` | `foundation` | `opensearch` / `opensearch,kafka,bai,pfs` |
| `recipe-name-rpa.md` | `env1-runtime-rpa.properties` | `cp4ba-rpa` | `foundation` (+RPA operator) | *(none in ICP4ACluster)* |

---

## 19. Reference Topology Files (ibm_cp4a_cr_production_FC_*)

These files in `references/cp4ba-yamls/` define the canonical CP4BA topology templates from IBM. They serve as the foundation for all custom recipes.

| File | Topology | Key Capabilities |
|---|---|---|
| `ibm_cp4a_cr_production_FC_foundation.yaml` | Foundation only | ICN, BAStudio, RR, BTS |
| `ibm_cp4a_cr_production_FC_workflow.yaml` | Full BAW runtime | BAW, FNCM/CPE, ICN, ECM, BAI, BAML |
| `ibm_cp4a_cr_production_FC_workflow_authoring.yaml` | BAW authoring | BAW authoring, FNCM/CPE, ICN |
| `ibm_cp4a_cr_production_FC_decisions.yaml` | ODM | decisionServerRuntime, Console, decisionCenter, decisionRunner |
| `ibm_cp4a_cr_production_FC_decisions_ads.yaml` | ADS | decision_designer, decision_runtime, rest_api, credentials_service, git_service, parsing_service, run_service, decision_runtime_service |
| `ibm_cp4a_cr_production_FC_content.yaml` | Full FNCM | CPE, CSS, CMIS, GraphQL, ES, TM, IER, ICCSAP |
| `ibm_cp4a_cr_production_FC_application.yaml` | Application Engine | AE, App Designer |
| `ibm_cp4a_cr_production_FC_process_federation_server.yaml` | Standalone PFS | ProcessFederationServer CR, pfs_configuration, OpenSearch |
| `ibm_cp4a_cr_production_FC_workflow_process_service_authoring.yaml` | WFPS Authoring | wfps_authoring, BAStudio |

---

*Last updated: 2025 — Sources: CP4BA v26.0.0 deployment recipes and reference configurations.*
