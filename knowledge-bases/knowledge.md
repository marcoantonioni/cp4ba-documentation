# CP4BA Knowledge Base — IBM Cloud Pak for Business Automation 26.0.0

> **Scope**: OpenShift containerized deployments only.  
> **Version**: CP4BA 26.0.0 / BAW 26.0.x / CPFS 4.x  
> **Disclaimer**: Templates in this repository are for educational/non-production purposes.

---

## Table of Contents

1. [Overview and Architecture](#1-overview-and-architecture)
2. [IBM Cloud Pak Foundational Services (CPFS)](#2-ibm-cloud-pak-foundational-services-cpfs)
3. [Foundation Pattern](#3-foundation-pattern)
4. [IBM Business Automation Workflow (BAW)](#4-ibm-business-automation-workflow-baw)
5. [IBM Workflow Process Service (WFPS)](#5-ibm-workflow-process-service-wfps)
6. [IBM Decision Intelligence Client Managed (ADS / DICM)](#6-ibm-decision-intelligence-client-managed-ads--dicm)
7. [IBM Operational Decision Manager (ODM)](#7-ibm-operational-decision-manager-odm)
8. [Business Automation Insights (BAI)](#8-business-automation-insights-bai)
9. [IBM Process Federation Server (PFS)](#9-ibm-process-federation-server-pfs)
10. [IBM Content Cortex / FileNet Content Manager (FNET / CORTEX)](#10-ibm-content-cortex--filenet-content-manager-fnet--cortex)
11. [Database Configuration](#11-database-configuration)
12. [Storage Configuration](#12-storage-configuration)
13. [LDAP and IAM Configuration](#13-ldap-and-iam-configuration)
14. [Deployment Patterns and Optional Components](#14-deployment-patterns-and-optional-components)
15. [Installation Tool and Scripts](#15-installation-tool-and-scripts)
16. [Deployment Recipes Catalogue](#16-deployment-recipes-catalogue)
17. [Custom Resource (CR) Structure Reference](#17-custom-resource-cr-structure-reference)

---

## 1. Overview and Architecture

IBM Cloud Pak for Business Automation (CP4BA) 26.0.0 is a containerized platform that bundles multiple business automation capabilities into a single Kubernetes-native deployment on Red Hat OpenShift.

### Key principles

- **ICP4ACluster CR** (`apiVersion: icp4a.ibm.com/v1`, `kind: ICP4ACluster`) is the single Custom Resource that drives all CP4BA capability deployments.
- **Patterns** (`sc_deployment_patterns`) activate capability groups: `foundation`, `workflow`, `workflow-process-service`, `decisions`, `decisions_ads`, `application`, `content`, `document_processing`.
- **Optional components** (`sc_optional_components`) enable specific sub-capabilities within an active pattern.
- **Deployment type** is always `Production` in the templates provided.
- **Deployment platform** is `OCP` (OpenShift Container Platform) for on-prem clusters, or `ROKS` for IBM Cloud.
- **Deployment profile size** (`sc_deployment_profile_size`): `small`, `medium`, or `large` — controls default resource requests/limits for all components.
- **Operator isolation model** used in this project: private catalog + operator co-located in the workload namespace (`CP4BA_AUTO_SEPARATE_OPERATOR=No`).

### Component relationship

```
OpenShift Cluster
└── Namespace: cp4ba-<env>
    ├── ICP4ACluster CR (icp4adeploy)
    ├── CP4BA Operator (ibm-cp4a-operator)
    ├── CPFS Foundational Services (Zen, IAM, License Service)
    ├── [Foundation] OpenSearch / Kafka (optional)
    ├── [workflow]   BAW Authoring or BAW Runtime
    ├── [workflow-process-service] WFPS Authoring
    ├── [decisions_ads] ADS Designer + ADS Runtime
    ├── [decisions]  ODM (DC, DSR, DSC, DR)
    ├── [application] Application Engine (AE / BAA)
    ├── [bai] BAI Flink + Opensearch + BPC
    └── Supporting: PostgreSQL StatefulSet, OpenLDAP pod
```

---

## 2. IBM Cloud Pak Foundational Services (CPFS)

**Version**: CPFS 4.x (continuous delivery)  
**Documentation**: https://www.ibm.com/docs/en/cloud-paks/foundational-services/4.x_cd

CPFS provides the platform-level services that all CP4BA capabilities depend on:

| Service | Description |
|---|---|
| **Zen / Cloud Pak for Data UI** | Web console and UI framework (`cpd-<namespace>.apps.<cluster>`) |
| **IAM (Identity and Access Management)** | Authentication and RBAC against LDAP/SCIM |
| **License Service** | Tracks license consumption metrics |
| **Cert Manager** | Certificate management for internal TLS |
| **MongoDB** | Internal data store for IAM and Zen |

### Relevant CR fields

```yaml
shared_configuration:
  sc_iam:
    default_admin_username: "cpadmin"   # IAM administrator mapped from LDAP
  encryption_key_secret: ibm-iaws-shared-key-secret
  image_pull_secrets:
    - ibm-entitlement-key
```

### SCIM configuration

All templates configure `scim_configuration_iam` inside `ldap_configuration` to define how LDAP attributes map to SCIM attributes (uid, cn, sn, mail, etc.), which enables IAM to synchronize users and groups from the external LDAP directory.

---

## 3. Foundation Pattern

**Pattern key**: `foundation`  
**Always present** in every deployment recipe.

The foundation pattern activates CPFS (Zen+IAM) and optionally OpenSearch and Kafka for event streaming.

### Optional components within Foundation

| Component key | Purpose |
|---|---|
| `opensearch` | OpenSearch cluster (used by BAI and PFS) |
| `kafka` | Kafka message bus (used by BAI event streaming) |
| `bai` | Business Automation Insights engine |
| `pfs` | Process Federation Server (federated task inbox) |

### Foundation-only recipe

The `cp4ba-cr-ref-foundation.yaml` template deploys CP4BA with only CPFS + optional OpenSearch and/or Kafka+BAI+PFS. No workflow, decisions, or content capabilities.

---

## 4. IBM Business Automation Workflow (BAW)

**Pattern key**: `workflow`  
**Documentation**: https://www.ibm.com/docs/en/baw/26.0.x

BAW provides BPMN process authoring (via Workflow Center) and runtime execution (via Workflow Server). In CP4BA, these are deployed as the `baw_authoring` component (authoring) or the `baw_configuration[]` section (runtime).

### BAW Authoring

**Optional component**: `baw_authoring`  
**CR section**: `workflow_authoring_configuration`

Workflow Center used to create, edit, and test process applications. Requires:
- `bastudio_configuration` — Business Automation Studio (BAS), the authoring IDE, including Playback Server
- `ecm_configuration` — FileNet Content Manager (CPE) for document/object store management
- `navigator_configuration` — IBM Content Navigator (ICN) UI for the content store

Key `workflow_authoring_configuration` fields:
```yaml
workflow_authoring_configuration:
  admin_user: "cp4admin"
  database:
    server_name: <db_host>
    database_name: <baw_db>
    port: "5432"
    type: postgresql
  case:
    datasource_configuration:
      dc_case_datasource: ...
      dc_case_datasource_tos: ...
```

### BAW Runtime

**CR section**: `baw_configuration[]` (array — supports multiple runtime instances)

Workflow Server for executing deployed process applications. Can be deployed in parallel with Workflow Center (authoring+runtime) or standalone. Each element in the array is an independent BAW runtime instance.

Key `baw_configuration[]` fields:
```yaml
baw_configuration:
  - name: bawins1
    admin_user: "cp4admin"
    database:
      server_name: <db_host>
      database_name: <baw_db>
      port: "5432"
      type: postgresql
    tls:
      tlsTrustList: []
    resources:
      limits:
        cpu: '3000m'
        memory: 3096Mi
```

### BAW + Application Engine (AE/BAA)

When the `application` pattern is added, Application Engine (BAA) is deployed alongside BAW. This enables the Application Designer and AppEngine Data Persistence optional components:

- **Optional components**: `app_designer`, `ae_data_persistence`
- **CR sections**: `application_engine_configuration[]`, `bastudio_configuration`

---

## 5. IBM Workflow Process Service (WFPS)

**Pattern key**: `workflow-process-service`  
**Optional component**: `wfps_authoring`  
**Documentation**: https://www.ibm.com/docs/en/cloud-paks/cp-biz-automation/26.0.0?topic=icpd-installing-cp4ba-workflow-process-service-runtime-production-deployment

WFPS is a lightweight workflow authoring and runtime environment focused on Cloud Native workflow processes. It is simpler than BAW and does not include Case Management or ECM.

### Key differences from BAW

| Feature | BAW | WFPS |
|---|---|---|
| BPMN Process Authoring | Yes (Workflow Center) | Yes (via BAS/BAStudio) |
| Case Management | Yes | No |
| ECM Integration | Yes (CPE + ICN) | No |
| Process Federation (PFS) | Optional | External (separate tool) |
| Pattern | `workflow` | `workflow-process-service` |

### CR section

```yaml
# WFPS uses bastudio_configuration for authoring IDE
bastudio_configuration:
  admin_user: "cp4admin"
  database:
    host: <db_host>
    name: <bas_db>
    port: "5432"
    type: postgresql
  playback_server:
    admin_user: "cp4admin"
    database:
      host: <db_host>
      name: <playback_db>
      port: "5432"
      type: postgresql
```

---

## 6. IBM Decision Intelligence Client Managed (ADS / DICM)

**Pattern key**: `decisions_ads`  
**Optional components**: `ads_designer`, `ads_runtime`, `bas` (required for authoring)  
**Documentation**: https://www.ibm.com/docs/en/cloud-paks/cp-biz-automation/26.0.0?topic=capabilities-decision-intelligence-client-managed-software

ADS (Automation Decision Services) is IBM's AI-powered decision management platform. It enables business users to create, test, and deploy decision models using natural language and machine learning.

### Components

| Component | Description | Authoring | Runtime only |
|---|---|---|---|
| **Decision Designer** | Web-based IDE for building decision services (connects to Git) | `enabled: true` | `enabled: false` |
| **Decision Runtime** | Executes deployed decision archives | `enabled: true` | `enabled: true` |
| **BAS / BAStudio** | Required for ADS Designer to function (provides authoring shell) | Yes | Not deployed |
| **Credentials Service** | Manages external credentials (Git, ML providers) | Yes | Yes |

### CR section `ads_configuration`

```yaml
ads_configuration:
  deployment_profile_size: "small"    # small | medium | large
  genai_secret_name: ads-genai-secret  # GenAI/WML integration

  decision_designer:
    enabled: true                      # false in runtime-only deployments
    admin_secret_name: ibm-dba-ads-designer-secret
    deployment_profile_size: "small"

  decision_runtime:
    enabled: true
    admin_secret_name: ibm-dba-ads-runtime-secret
    deployment_profile_size: "small"
    replica_count: 2
    authentication_mode: "zen"         # zen | basic | none
    archive_storage_type: "fs"         # fs (filesystem PVC)
    autoscaling:
      enabled: false

    # BAI event emitter
    event_emitter:
      enabled: false                   # true requires BAI
      kafka_topic: "ads-decision-execution-common-data"
      elasticsearch_index: "ads-decision-execution-common-data"
      allow_missing_events: true
      queue_capacity: 50000
      dequeur_threads: 1

  decision_runtime_service:
    stack_trace_enabled: true
    resources:
      requests:
        cpu: '500m'
        memory: '2Gi'
        ephemeral_storage: '100Mi'
      limits:
        cpu: '2000m'
        memory: '3Gi'
        ephemeral_storage: '1000Mi'
    persistence:
      use_dynamic_provisioning: true
      resources:
        requests:
          storage: "1Gi"
    rolling_update:
      max_unavailable: 1
      max_surge: 1
    tls:
      allow_self_signed: true
      verify_hostname: false
```

### Datasources for ADS

| Datasource key | Used by | Description |
|---|---|---|
| `dc_ads_designer_datasource` | ADS Designer | Schema for designer data (authoring only) |
| `dc_ads_runtime_datasource` | ADS Runtime | Schema for deployed decision archives |
| `dc_icn_datasource` | ICN (Navigator) | Required when bas/BAS is enabled |

### GenAI Integration

ADS supports GenAI features via WML (Watson Machine Learning). Configuration uses:
- `ads_genai_secret_name` — K8s Secret containing API key and endpoint
- The secret is pre-created with WML API key, URL, and project ID

### BAI emitter in ADS

Even without `bai` in optional components, the `event_emitter` section is present in the CR for future use. When `enabled: false`, it is silently ignored. When BAI is installed and `enabled: true`, ADS decision executions generate events published to Kafka and indexed in OpenSearch.

---

## 7. IBM Operational Decision Manager (ODM)

**Pattern key**: `decisions`  
**Documentation**: https://www.ibm.com/docs/en/cloud-paks/cp-biz-automation/26.0.0?topic=capabilities-operational-decision-manager

ODM is IBM's classic rule-based decision management platform. It enables business analysts to author, test, simulate, and execute business rules using Decision Center and Decision Server.

### Components

| Component | Description | Authoring | Runtime |
|---|---|---|---|
| **Decision Center (DC)** | Web-based rule authoring and governance IDE | `enabled: true` | `enabled: false` |
| **Decision Runner (DR)** | Test and simulate rules from Decision Center | `enabled: true` | `enabled: false` |
| **Decision Server Runtime (DSR)** | Executes deployed rulesets via REST/SOAP | `enabled: true` | `enabled: true` |
| **Decision Server Console (DSC)** | Management console for deployed rulesets | Always present | Always present |

### CR section `odm_configuration`

```yaml
odm_configuration:
  version: "26.0.0"
  deployment_profile_size: "small"
  debug: false

  audit_logging:
    enabled: false
    rolling_max_files: 5
    rolling_max_size: '20Mi'

  image:
    pullPolicy: IfNotPresent
    repository: cp.icr.io/cp/cp4a/odm

  dba:
    passwordSecretRef: ibm-odm-keystore-secret  # K8s Secret with keystore password

  decisionServerRuntime:
    enabled: true        # true in both authoring and runtime
    replicaCount: 1
    resources:
      requests: { cpu: 500m, memory: 2Gi }
      limits:   { cpu: 2000m, memory: 2Gi }

  decisionServerConsole:   # always enabled
    resources:
      requests: { cpu: 500m, memory: 2Gi }
      limits:   { cpu: 2000m, memory: 2Gi }

  decisionCenter:
    enabled: true          # false in runtime-only
    persistenceLocale: en_US
    replicaCount: 1
    disableAllAuthenticatedUser: false
    resources:
      requests: { cpu: 500m, memory: 2Gi }
      limits:   { cpu: 2000m, memory: 4Gi }

  decisionRunner:
    enabled: true          # false in runtime-only
    replicaCount: 1
    resources:
      requests: { cpu: 500m, memory: 2Gi }
      limits:   { cpu: 2000m, memory: 4Gi }

  customization:
    bai_kafka_topic:       # configure BAI topic name when BAI is used

  show_scim_connection: false   # enables SCIM import of users/groups in DC
```

### Datasources for ODM

| Datasource key | Used by | Present in |
|---|---|---|
| `dc_odm_datasource` | Shared DB config (server + port + name) | Both authoring and runtime |
| `dc_odm_decisioncenter_datasource` | Decision Center specific config | Authoring only |
| `dc_odm_decisionserver_datasource` | Decision Server specific config | Both authoring and runtime |

> **Note**: The runtime-only templates (`cp4ba-cr-ref-decision-odm.yaml`, `cp4ba-cr-ref-decision-odm-bai.yaml`) do **not** include `dc_odm_decisioncenter_datasource`, and Decision Center and Decision Runner are disabled via the values file.

### ODM and BAI

When the `bai` optional component is enabled alongside ODM:
- The `bai_configuration.odm.install` flag is set to `true`
- BAI consumes ODM rule execution events from Kafka
- The `odm_configuration.customization.bai_kafka_topic` can be set to override the default topic

---

## 8. Business Automation Insights (BAI)

**Optional component**: `bai`  
**Required infrastructure**: OpenSearch + Kafka (must also be listed in optional components)  
**Documentation**: https://www.ibm.com/docs/en/cloud-paks/cp-biz-automation/26.0.0?topic=baip-event-processing-parameters

BAI provides event processing, dashboards, and analytics for business automation events. It uses Apache Flink for stream processing, OpenSearch for event indexing, and Kafka for event ingestion.

### CR section `bai_configuration`

```yaml
bai_configuration:
  settings:
    egress: false     # enable egress for external Kafka

  business_performance_center:
    install: true
    all_users_access: true   # when false, only admins can view BPC

  # Event processors — enable only for installed components
  bpmn:
    install: false     # true if BAW workflow events needed
  bawadv:
    install: false     # true for BAW Advanced (Case) events
  content:
    install: false     # true for Content events (FNET/CPE)
  event_forwarder:
    install: false
  flink:
    create_route: true   # creates OpenShift route to Flink dashboard
  icm:
    install: false
  navigator:
    install: false
  ads:
    install: true      # true when ADS is installed
  odm:
    install: true      # true when ODM is installed
```

### BAI event processor matrix

| Processor | Enabled when... |
|---|---|
| `bpmn` | BAW workflow is deployed (BPMN events) |
| `bawadv` | BAW Advanced/Case events |
| `content` | FileNet/CPE content events |
| `navigator` | IBM Content Navigator events |
| `ads` | ADS decision execution events |
| `odm` | ODM rule execution events |

---

## 9. IBM Process Federation Server (PFS)

**Documentation**: https://www.ibm.com/docs/en/cloud-paks/cp-biz-automation/26.0.0?topic=deployments-installing-cp4ba-process-federation-server-production-deployment

PFS federates task inboxes from multiple BAW instances (and WFPS) into a single unified work portal. It is **not deployed via the ICP4ACluster CR** in these recipes — it is deployed using a separate external tool (`cp4ba-process-federation-server`).

### Key facts

- PFS uses OpenSearch to index and federate tasks
- The `pfs` optional component in `sc_optional_components` configures the PFS endpoint URLs within the CR, but the actual PFS pod lifecycle is managed externally
- BAW authoring and runtime environments can be registered with PFS for unified task federation

### In the context of this project

The `env1-runtime-os-bai-pfs.properties` config installs Foundation + OpenSearch + Kafka + BAI + PFS together. The PFS process is bootstrapped by the installation script using the external cp4ba-process-federation-server tooling.

---

## 10. IBM Content Cortex / FileNet Content Manager (FNET / CORTEX)

**Former name**: IBM FileNet Content Manager  
**Current name**: IBM Content Cortex  
**Pattern key**: `content`  
**Documentation**: https://www.ibm.com/docs/en/cloud-paks/cp-biz-automation/26.0.0?topic=deployments-installing-cp4ba-content-cortex-production-deployment

Content Cortex (FileNet) is the enterprise content management backbone. In BAW deployments it is used to store process documents, case files, and workflow attachments.

### Components

| Component | Description |
|---|---|
| **CPE (Content Platform Engine)** | Core FileNet engine: object stores, document lifecycle |
| **ICN (IBM Content Navigator)** | Web UI for browsing/managing content |
| **CSS (Content Search Services)** | Full-text indexing and search |
| **CMIS** | Content Management Interoperability Services |
| **TM (Task Manager)** | Task management integration |

### In BAW deployments

Even when the `content` pattern is not explicitly listed, BAW authoring deployments include:
- `ecm_configuration` — CPE configuration with FPOS object stores
- `navigator_configuration` — ICN configuration
- `dc_icn_datasource` — ICN database datasource

This is because BAW process applications require a content store for document attachments.

---

## 11. Database Configuration

All recipes use **PostgreSQL** as the database backend. EDB (Enterprise DB) support is deprecated and disabled (`CP4BA_INST_DB_USE_EDB=false`).

### PostgreSQL deployment model

| Parameter | Value |
|---|---|
| Image | `postgres:18.4` (OSS) |
| Deployment | StatefulSet in the same namespace as CP4BA |
| Port | `5432` |
| SSL | Enabled (SSL-only endpoint: `<cr-name>-ssl-rw` service) |
| Storage | 10Gi PVC |
| Admin password | `postgres` (demo only) |

### Database server naming convention

```
<db_cr_name_ssl>-rw.<namespace>.svc.cluster.local   # read-write (primary)
<db_cr_name_ssl>-r.<namespace>.svc.cluster.local    # read (replica)
```

### Database names convention

DB names are derived from the environment prefix (`CP4BA_INST_ENV_FOR_DB_PREFIX`, which is the `CP4BA_INST_ENV` with `-` replaced by `_`):

| Component | DB Name pattern | Example (env=`ads-auth`) |
|---|---|---|
| ICN | `<prefix>_icn` | `ads_auth_icn` |
| ADS Designer | `<prefix>_adsdesignerdb` | `ads_auth_adsdesignerdb` |
| ADS Runtime | `<prefix>_adsruntimedb` | `ads_auth_adsruntimedb` |
| ODM | `<prefix>_odmdb` | `odm_auth_odmdb` |
| BAW | `<prefix>_baw_1` | `baw_auth_baw_1` |
| BAS Playback | `<prefix>_appdb` | `baw_auth_appdb` |
| AEOS | `<prefix>_aeos` | `baw_auth_aeos` |

### DB Secrets

Each component has a dedicated K8s Secret containing the database credentials:

| Secret name | Used by |
|---|---|
| `ibm-ads-designer-database` | ADS Designer |
| `ibm-ads-runtime-database` | ADS Runtime |
| `ibm-odm-db-secret` | ODM (all components) |
| `ibm-odm-keystore-secret` | ODM keystore password |
| `<env>-bas-server-db-secret` | BAS/BAStudio |

### Connection pool settings

```properties
CP4BA_INST_DB_MAX_POOL_SIZE=100
CP4BA_INST_DB_MIN_POOL_SIZE=10
```

---

## 12. Storage Configuration

All recipes use **OCS External** (OpenShift Container Storage, external mode) storage classes:

| Storage class variable | Default value |
|---|---|
| `CP4BA_INST_SC_FILE` | `ocs-external-storagecluster-cephfs` |
| `CP4BA_INST_SC_BLOCK` | `ocs-external-storagecluster-ceph-rbd` |

### Storage class usage in CR

```yaml
shared_configuration:
  storage_configuration:
    sc_dynamic_storage_classname: "ocs-external-storagecluster-cephfs"
    sc_slow_file_storage_classname: "ocs-external-storagecluster-cephfs"
    sc_medium_file_storage_classname: "ocs-external-storagecluster-cephfs"
    sc_fast_file_storage_classname: "ocs-external-storagecluster-cephfs"
    sc_block_storage_classname: "ocs-external-storagecluster-ceph-rbd"
```

> All file storage classes point to the same CephFS class (slow/medium/fast differentiation is logical only in these configs).

---

## 13. LDAP and IAM Configuration

### LDAP

All recipes deploy a local **OpenLDAP** pod in the same namespace as CP4BA (unless `CP4BA_INST_LDAP=false`). This provides the user directory used by IAM.

Key LDAP parameters:

| Parameter | Description |
|---|---|
| `lc_selected_ldap_type` | `Custom` (generic LDAP) |
| `lc_ldap_ssl_enabled` | `false` (plaintext for demo) |
| `lc_ldap_user_name_attribute` | `*:cn` |
| `lc_ldap_group_membership_search_filter` | `(&(cn=%v)(objectclass=groupOfNames))` |
| `lc_ldap_recursive_search` | `true` |
| `lc_pagination_size` | `4500` |

### SCIM mapping (IAM)

The `scim_configuration_iam` block maps LDAP attributes to SCIM attributes used by CPFS IAM:

| SCIM attribute | LDAP attribute |
|---|---|
| `user_unique_id_attribute` | `uid` |
| `user_name_attribute` | `cn` |
| `user_display_name_attribute` | `cn` |
| `user_emails_attribute` | `mail` |
| `group_name_attribute` | `cn` |
| `group_members_attribute` | `member` |

### IAM Admin User

- IAM admin: `cpadmin` (mapped via `CP4BA_INST_IAM_ADMIN_USER`)
- Pak admin (BAW/BAS/etc.): `cp4admin` (mapped via `CP4BA_INST_PAKBA_ADMIN_USER`)
- Admin group: `AdminsGroup`

---

## 14. Deployment Patterns and Optional Components

### Patterns reference

| Pattern key | Capabilities activated |
|---|---|
| `foundation` | CPFS, Zen, IAM — always required |
| `workflow` | BAW Authoring or BAW Runtime |
| `workflow-process-service` | WFPS Authoring |
| `decisions` | ODM (DC, DSR, DSC, DR) |
| `decisions_ads` | ADS (Decision Designer + Decision Runtime) |
| `application` | Application Engine (BAA) + App Designer |
| `content` | FileNet CPE + ICN + CSS |
| `document_processing` | Document Processing / Capture |

### Optional components reference

| Component key | Requires pattern | Description |
|---|---|---|
| `baw_authoring` | `workflow` | BAW Workflow Center (authoring) |
| `bas` | `decisions_ads` or `workflow-process-service` | Business Automation Studio |
| `bai` | `foundation` | Business Automation Insights |
| `kafka` | `foundation` | Kafka message bus (for BAI) |
| `opensearch` | `foundation` | OpenSearch cluster |
| `pfs` | `foundation` | Process Federation Server |
| `ads_designer` | `decisions_ads` | ADS Decision Designer |
| `ads_runtime` | `decisions_ads` | ADS Decision Runtime |
| `decisionCenter` | `decisions` | ODM Decision Center (deprecated key, use CR flag) |
| `decisionRunner` | `decisions` | ODM Decision Runner (deprecated key, use CR flag) |
| `decisionServerRuntime` | `decisions` | ODM Decision Server Runtime (deprecated key) |
| `app_designer` | `application` | Application Designer |
| `ae_data_persistence` | `application` | Application Engine data persistence |
| `workflow_assistant` | `workflow` | Workflow AI assistant |
| `workplace_assistant` | `workflow` | Workplace AI assistant |

---

## 15. Installation Tool and Scripts

### Overview

The installation is driven by the script [`cp4ba-one-shot-installation.sh`](../cp4ba-installations/scripts/cp4ba-one-shot-installation.sh). This script:

1. Sources the `.properties` configuration file
2. Installs the CP4BA Case Package Manager (if `-m` flag is given)
3. Deploys the CP4BA Operator (CatalogSource + Subscription) in the target namespace
4. Creates all required K8s Secrets (DB, LDAP, ADS, ODM, etc.)
5. Deploys and configures the database (PostgreSQL StatefulSet)
6. Deploys and configures the LDAP server
7. Interpolates the CR template with values from the `.properties` file
8. Applies the ICP4ACluster CR to the cluster
9. Onboards IAM users and groups
10. Optionally configures the ZenService TLS certificate

### Usage

```bash
cd cp4ba-installations/scripts

./cp4ba-one-shot-installation.sh \
  -c <full-path-to-config.properties> \
  [-m]       # install CP4BA Case Package Manager (first time only)
  -v 26.0.0  # CP4BA version
  -k 26.0.0  # cert-kubernetes version
  [-o]       # skip operator installation (if already installed in namespace)
  [-x]       # enable trace/debug output
```

### Example commands per recipe

```bash
# Set the configs path
_PTC=$(pwd)/../configs26

# ADS Authoring
./cp4ba-one-shot-installation.sh -c ${_PTC}/env1-authoring-ads.properties -m -v 26.0.0 -k 26.0.0

# ADS Authoring + BAI
./cp4ba-one-shot-installation.sh -c ${_PTC}/env1-authoring-ads-bai.properties -m -v 26.0.0 -k 26.0.0

# ADS Runtime only
./cp4ba-one-shot-installation.sh -c ${_PTC}/env1-runtime-ads.properties -m -v 26.0.0 -k 26.0.0

# ADS Runtime + BAI
./cp4ba-one-shot-installation.sh -c ${_PTC}/env1-runtime-ads-bai.properties -m -v 26.0.0 -k 26.0.0

# ODM Authoring
./cp4ba-one-shot-installation.sh -c ${_PTC}/env1-authoring-odm.properties -m -v 26.0.0 -k 26.0.0

# ODM Authoring + BAI
./cp4ba-one-shot-installation.sh -c ${_PTC}/env1-authoring-odm-bai.properties -m -v 26.0.0 -k 26.0.0

# ODM Runtime only
./cp4ba-one-shot-installation.sh -c ${_PTC}/env1-runtime-odm.properties -m -v 26.0.0 -k 26.0.0

# ODM Runtime + BAI
./cp4ba-one-shot-installation.sh -c ${_PTC}/env1-runtime-odm-bai.properties -m -v 26.0.0 -k 26.0.0

# BAW Authoring
./cp4ba-one-shot-installation.sh -c ${_PTC}/env1-authoring-baw.properties -m -v 26.0.0 -k 26.0.0

# BAW Authoring + BAI
./cp4ba-one-shot-installation.sh -c ${_PTC}/env1-authoring-baw-bai.properties -m -v 26.0.0 -k 26.0.0

# BAW Authoring + BAI + AE
./cp4ba-one-shot-installation.sh -c ${_PTC}/env1-authoring-baw-bai-ae.properties -m -v 26.0.0 -k 26.0.0

# WFPS Authoring + BAI
./cp4ba-one-shot-installation.sh -c ${_PTC}/env1-authoring-wfps-pfs-bai.properties -m -v 26.0.0 -k 26.0.0

# BAW Runtime + BAI
./cp4ba-one-shot-installation.sh -c ${_PTC}/env1-runtime-baw-bai.properties -m -v 26.0.0 -k 26.0.0

# Foundation (OpenSearch only)
./cp4ba-one-shot-installation.sh -c ${_PTC}/env1-runtime-opensearch-foundation.properties -m -v 26.0.0 -k 26.0.0

# Foundation + OpenSearch + BAI + PFS
./cp4ba-one-shot-installation.sh -c ${_PTC}/env1-runtime-os-bai-pfs.properties -m -v 26.0.0 -k 26.0.0
```

### Key environment variables set by config files

| Variable | Description |
|---|---|
| `CP4BA_INST_NAMESPACE` | Target OCP namespace (e.g., `cp4ba-ads-auth`) |
| `CP4BA_INST_CR_TEMPLATE` | Path to the CR template YAML (relative to script) |
| `CP4BA_INST_DEPL_PATTERNS` | Comma-separated patterns list |
| `CP4BA_INST_OPT_COMPONENTS` | Comma-separated optional components list |
| `CP4BA_AUTO_SEPARATE_OPERATOR` | `No` = operator in same namespace |
| `CP4BA_AUTO_PRIVATE_CATALOG` | `Yes` = private CatalogSource |
| `CP4BA_INST_SC_FILE` | File storage class |
| `CP4BA_INST_SC_BLOCK` | Block storage class |
| `CP4BA_INST_DB_OSS_IMAGE` | PostgreSQL OSS image (`postgres:18.4`) |

---

## 16. Deployment Recipes Catalogue

The following table lists all 15 deployment recipes found in `cp4ba-installations/templates26/`, mapped to their config files and recipe documentation:

| # | Recipe Suffix | Template | Config File(s) | Namespace | Recipe File |
|---|---|---|---|---|---|
| 1 | `authoring-baw` | `cp4ba-cr-ref-authoring-baw.yaml` | `env1-authoring-baw.properties` | `cp4ba-baw-auth` | `recipe-name-authoring-baw.md` |
| 2 | `authoring-baw-bai` | `cp4ba-cr-ref-authoring-baw-bai.yaml` | `env1-authoring-baw-bai.properties` | `cp4ba-baw-bai-auth` | `recipe-name-authoring-baw-bai.md` |
| 3 | `authoring-baw-bai-ae` | `cp4ba-cr-ref-authoring-baw-bai-ae.yaml` | `env1-authoring-baw-bai-ae.properties` | `cp4ba-baw-bai-ae-auth` | `recipe-name-authoring-baw-bai-ae.md` |
| 4 | `authoring-wfps-bai` | `cp4ba-cr-ref-authoring-wfps-bai.yaml` | `env1-authoring-wfps-pfs-bai.properties` | `cp4ba-wfps-pfs-bai-auth` | `recipe-name-authoring-wfps-bai.md` |
| 5 | `authoring-decision-ads` | `cp4ba-cr-ref-authoring-decision-ads.yaml` | `env1-authoring-ads.properties` | `cp4ba-ads-auth` | `recipe-name-authoring-decision-ads.md` |
| 6 | `authoring-decision-ads-bai` | `cp4ba-cr-ref-authoring-decision-ads-bai.yaml` | `env1-authoring-ads-bai.properties` | `cp4ba-ads-bai-auth` | `recipe-name-authoring-decision-ads-bai.md` |
| 7 | `authoring-decision-odm` | `cp4ba-cr-ref-authoring-decision-odm.yaml` | `env1-authoring-odm.properties` | `cp4ba-odm-auth` | `recipe-name-authoring-decision-odm.md` |
| 8 | `authoring-decision-odm-bai` | `cp4ba-cr-ref-authoring-decision-odm-bai.yaml` | `env1-authoring-odm-bai.properties` | `cp4ba-odm-bai-auth` | `recipe-name-authoring-decision-odm-bai.md` |
| 9 | `baw-bai` | `cp4ba-cr-ref-baw-bai.yaml` | `env1-runtime-baw-bai.properties` | `cp4ba-baw-bai-prod` | `recipe-name-baw-bai.md` |
| 10 | `baw-bai-ae` | `cp4ba-cr-ref-baw-bai-ae.yaml` | *(derived from baw-bai)* | `cp4ba-baw-bai-ae-prod` | `recipe-name-baw-bai-ae.md` |
| 11 | `decision-ads` | `cp4ba-cr-ref-decision-ads.yaml` | `env1-runtime-ads.properties` | `cp4ba-ads-prod` | `recipe-name-decision-ads.md` |
| 12 | `decision-ads-bai` | `cp4ba-cr-ref-decision-ads-bai.yaml` | `env1-runtime-ads-bai.properties` | `cp4ba-ads-bai-prod` | `recipe-name-decision-ads-bai.md` |
| 13 | `decision-odm` | `cp4ba-cr-ref-decision-odm.yaml` | `env1-runtime-odm.properties` | `cp4ba-odm` | `recipe-name-decision-odm.md` |
| 14 | `decision-odm-bai` | `cp4ba-cr-ref-decision-odm-bai.yaml` | `env1-runtime-odm-bai.properties` | `cp4ba-odm-bai` | `recipe-name-decision-odm-bai.md` |
| 15 | `foundation` | `cp4ba-cr-ref-foundation.yaml` | `env1-runtime-opensearch-foundation.properties` / `env1-runtime-os-bai-pfs.properties` | `cp4ba-opensearch-prod` / `cp4ba-os-bai-pfs-prod` | `recipe-name-foundation.md` |

---

## 17. Custom Resource (CR) Structure Reference

The `ICP4ACluster` CR (`apiVersion: icp4a.ibm.com/v1`) has the following top-level sections:

```yaml
apiVersion: icp4a.ibm.com/v1
kind: ICP4ACluster
metadata:
  name: icp4adeploy
  namespace: cp4ba-<env>
  labels:
    app.kubernetes.io/instance: ibm-dba
    release: "26.0.0"
    cp4ba.ibm.com/backup-type: mandatory

spec:
  appVersion: "26.0.0"
  ibm_license: accept

  shared_configuration:        # Platform-wide settings
  ldap_configuration:          # LDAP server connection + SCIM mapping
  datasource_configuration:    # All database connections
  
  # Capability-specific sections (present only when pattern is active):
  workflow_authoring_configuration:   # BAW authoring
  baw_configuration: []               # BAW runtime (array)
  bastudio_configuration:             # BAS / WFPS authoring IDE
  ads_configuration:                  # ADS Designer + Runtime
  odm_configuration:                  # ODM components
  bai_configuration:                  # BAI event processing
  application_engine_configuration: [] # Application Engine (array)
  ecm_configuration:                  # FileNet CPE
  navigator_configuration:            # IBM Content Navigator
```

### Template variable substitution

The CR templates use `${VARIABLE_NAME}` placeholders that are substituted at install time from the `.properties` config files. The substitution is performed by the `cp4ba-one-shot-installation.sh` script using shell variable expansion (`envsubst` or equivalent).

Key variable categories:

| Prefix | Category |
|---|---|
| `CP4BA_INST_CR_*` | CR metadata (name, namespace) |
| `CP4BA_INST_DEPL_*` | Deployment configuration |
| `CP4BA_INST_SC_*` | Storage classes |
| `CP4BA_INST_DB_*` | Database server configuration |
| `CP4BA_INST_LDAP_*` | LDAP configuration |
| `CP4BA_INST_IAM_*` | IAM/admin users |
| `CP4BA_INST_BAW_*` | BAW-specific configuration |
| `CP4BA_INST_BAS_*` | BAS/BAStudio configuration |
| `CP4BA_INST_ADS_*` | ADS configuration |
| `CP4BA_INST_ODM_*` | ODM configuration |
| `CP4BA_INST_BAI_*` | BAI configuration |
| `CP4BA_BASE_VER` | CP4BA version (e.g., `26.0.0`) |
