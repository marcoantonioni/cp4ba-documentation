# Recipe: BAW Authoring + BAI + Application Engine + ADS (Authoring)

> **Recipe suffix**: `authoring-baw-bai-ae-ads`  
> **Template**: [`cp4ba-cr-ref-authoring-baw-bai-ae-ads.yaml`](../cp4ba-installations/templates26/cp4ba-cr-ref-authoring-baw-bai-ae-ads.yaml)  
> **Config file**: [`env1-authoring-baw-bai-ae-ads.properties`](../cp4ba-installations/configs26/env1-authoring-baw-bai-ae-ads.properties)  
> **Disclaimer**: These configurations are not intended for production environments. The purpose is purely educational.

---

## Overview

This recipe deploys the most comprehensive CP4BA authoring environment, combining:

- **BAW Authoring** — Full Business Automation Workflow authoring (BPM + Case Management)
- **Business Automation Insights (BAI)** — Real-time analytics with Kafka + OpenSearch
- **Application Engine (AE)** — Low-code application authoring and runtime
- **ADS (Automation Decision Services)** — AI-infused decision authoring and runtime

This is the maximal authoring recipe, covering workflow processes, business applications, and intelligent decision automation in a single namespace.

---

## Deployment Details

| Property | Value |
|---|---|
| **Namespace** | `cp4ba-baw-bai-ae-ads-auth` |
| **CR Name** | `icp4adeploy` |
| **CR Kind** | `ICP4ACluster` |
| **Deployment Type** | `Production` |
| **Deployment Platform** | `OCP` |
| **Profile Size** | `small` |
| **CP4BA Version** | `26.0.0` |
| **License Type** | `production` |
| **FNCM License** | `production` |
| **BAW License** | `production` |

---

## CP4BA Patterns and Optional Components

```properties
CP4BA_INST_DEPL_PATTERNS=foundation,workflow,application,decisions_ads
CP4BA_INST_OPT_COMPONENTS=baw_authoring,bas,app_designer,ads_designer,ads_runtime,bai,pfs,kafka,opensearch,workflow_assistant,workplace_assistant
```

---

## Capabilities Deployed

### 1. Foundation
**Pattern**: `foundation`

Provides the baseline platform services:
- IBM Cloud Pak Foundational Services (CPFS/Zen): IAM, License Service, CP4D UI
- IBM Content Navigator (ICN): web portal for content and workflow
- Business Automation Studio (BAS): authoring IDE for processes, cases, decisions, and applications
- Resource Registry: etcd-based capability registry
- Business Team Service (BTS): team management for Case

### 2. Business Automation Workflow – Authoring
**Pattern**: `workflow` | **Optional component**: `baw_authoring`

Full BAW Authoring environment:
- **Workflow Center**: author, test, and publish BPMN processes and CMMN cases
- **FileNet Content Platform Engine (CPE)**: document/object store management
- **IBM Content Navigator (ICN)**: content browsing UI
- **Workplace / workflow_assistant / workplace_assistant**: AI-assisted task experience

**BAW databases** (PostgreSQL, SSL-enabled):

| DB Variable | Purpose |
|---|---|
| `CP4BA_INST_GCD_DB_NAME` = `baw_bai_ae_ads_auth_gcd` | FNCM Global Configuration DB |
| `CP4BA_INST_ICN_DB_NAME` = `baw_bai_ae_ads_auth_icn` | IBM Content Navigator |
| `CP4BA_INST_DOCS_DB_NAME` = `baw_bai_ae_ads_auth_bawdocs` | BAW documents object store |
| `CP4BA_INST_DOS_DB_NAME` = `baw_bai_ae_ads_auth_bawdos` | BAW design object store |
| `CP4BA_INST_TOS_DB_NAME` = `baw_bai_ae_ads_auth_bawtos` | BAW target object store |
| `CP4BA_INST_CONTENT_DB_NAME` = `baw_bai_ae_ads_auth_content` | Content object store |
| `CP4BA_INST_OS1_DB_NAME` = `baw_bai_ae_ads_auth_os1` | Additional object store |
| `CP4BA_INST_AWSDB_DB_NAME` = `baw_bai_ae_ads_auth_awsdb` | Advanced Work Services |
| `CP4BA_INST_AWSDOCS_DB_NAME` = `baw_bai_ae_ads_auth_awsdocs` | AWS documents |

### 3. Business Automation Insights (BAI)
**Optional component**: `bai,kafka,opensearch`

Real-time analytics and monitoring:
- **Kafka** (Strimzi): event bus for business automation events
- **OpenSearch**: analytics data store and Kibana-compatible dashboards
- **BAI event processors**: BPMN (workflow), bawadv (case), ads (decision)

### 4. Application Engine (AE / App Designer)
**Pattern**: `application` | **Optional components**: `app_designer`

Low-code application building and runtime:
- **Application Designer**: visual app authoring inside BAS
- **Application Engine runtime**: executes deployed CP4BA applications
- **App databases** (PostgreSQL, SSL-enabled):

| DB Variable | Purpose |
|---|---|
| `CP4BA_INST_AE_DB_NAME` = `baw_bai_ae_ads_auth_aaedb` | Application Engine DB |
| `CP4BA_INST_AEOS_DB_NAME` = `baw_bai_ae_ads_auth_aeos` | App Engine Object Store |
| `CP4BA_INST_APP_DB_NAME` = `baw_bai_ae_ads_auth_appdb` | Application DB |

### 5. Automation Decision Services (ADS)
**Pattern**: `decisions_ads` | **Optional components**: `ads_designer,ads_runtime`

AI/ML-infused decision management:
- **Decision Designer**: DMN-based visual decision model authoring, integrated with Git
- **Decision Runtime**: high-throughput execution engine for deployed decision services
- **Credentials Service**: manages Git and ML provider credentials
- **REST API**: OpenAPI endpoint for invoking decisions

**ADS databases** (PostgreSQL, SSL-enabled):

| DB Variable | Purpose |
|---|---|
| `CP4BA_INST_ADS_DESIGNER_DB_NAME` = `baw_bai_ae_ads_auth_adsdesignerdb` | ADS Designer state store |
| `CP4BA_INST_ADS_RUNTIME_DB_NAME` = `baw_bai_ae_ads_auth_adsruntimedb` | ADS Runtime state store |

### 6. Process Federation Server (PFS)
**Optional component**: `pfs`

Federates task inboxes from this BAW authoring instance into a unified portal. Requires OpenSearch (already enabled via BAI).

---

## Storage Configuration

| Storage Class | Type | Default Value |
|---|---|---|
| File (RWX) | CephFS | `ocs-external-storagecluster-cephfs` |
| Block (RWO) | Ceph RBD | `ocs-external-storagecluster-ceph-rbd` |

---

## Database Configuration

- **Type**: PostgreSQL (OSS, `postgres:18.1` by default)
- **SSL**: Enforced (`CP4BA_INST_DB_ONLY_SSL=true`)
- **Instance count**: 1 PostgreSQL StatefulSet shared across all components
- **DB CR name**: `my-postgres-1-for-cp4ba-ssl`
- **SQL template**: `db-statements-ref-baw-authoring.sql`
- **TLS secret**: `my-db-tls-secret`

---

## LDAP and IAM Configuration

- **Local LDAP**: deployed in namespace (`CP4BA_INST_LDAP=true`)
- **IAM onboarding**: enabled (`CP4BA_INST_IAM=true`)
- **IAM admin user**: `cpadmin`
- **Pak admin user**: `cp4admin`
- **Admin group**: `AdminsGroup`
- **LDAP config file**: `_cfg-production-ldap-domain.properties`
- **LDAP type**: `Custom` (OpenLDAP-compatible)

---

## Operator Isolation

This deployment uses maximum namespace isolation:

```bash
CP4BA_AUTO_PRIVATE_CATALOG=Yes
CP4BA_AUTO_SEPARATE_OPERATOR=No
CP4BA_AUTO_ALL_NAMESPACES=No
CP4BA_AUTO_OPERATOR_NAMESPACE=cp4ba-baw-bai-ae-ads-auth
CP4BA_AUTO_CS_SERVICE_NAMESPACE=cp4ba-baw-bai-ae-ads-auth
```

---

## Custom XML Configuration

This recipe supports custom Liberty and Lombardi XML configuration:

| Variable | Value |
|---|---|
| `CP4BA_INST_CUSTOM_XML_FOLDER_NAME` | `templates-custom-xml` |
| `CP4BA_INST_LIBERTY_CUSTOM_XML_SECRET_NAME` | `my-liberty-custom-xml-secret` |
| `CP4BA_INST_LOMBARDI_CUSTOM_XML_SECRET_NAME` | `my-lombardi-custom-xml-secret` |
| `CP4BA_INST_LIBERTY_CUSTOM_XML_TEMPLATE_NAME` | `liberty-custom-xml-template-sample-custom-db` |
| `CP4BA_INST_LOMBARDI_CUSTOM_XML_TEMPLATE_NAME` | `lombardi-custom-xml-template-sample-document` |

---

## Installation Command

Use the `cp4ba-one-shot-installation.sh` script from the `cp4ba-installations/scripts` directory.

### Prerequisites

Clone the required sibling repositories alongside this project:

```bash
git clone https://github.com/marcoantonioni/cp4ba-casemanager-setup
git clone https://github.com/marcoantonioni/cp4ba-idp-ldap
git clone https://github.com/marcoantonioni/cp4ba-utilities
git clone https://github.com/marcoantonioni/cp4ba-logger
```

### First-time installation (installs Case Package Manager)

```bash
cd cp4ba-installations/scripts

./cp4ba-one-shot-installation.sh \
  -c ../configs26/env1-authoring-baw-bai-ae-ads.properties \
  -m \
  -d /opt/cp4ba-cmgr
```

### Subsequent installations (reuse existing Case Package Manager)

```bash
cd cp4ba-installations/scripts

./cp4ba-one-shot-installation.sh \
  -c ../configs26/env1-authoring-baw-bai-ae-ads.properties \
  -p /opt/cp4ba-cmgr/cert-kubernetes/scripts
```

### Test configuration only (dry run)

```bash
cd cp4ba-installations/scripts

./cp4ba-one-shot-installation.sh \
  -c ../configs26/env1-authoring-baw-bai-ae-ads.properties \
  -t
```

### With trace enabled

```bash
cd cp4ba-installations/scripts

./cp4ba-one-shot-installation.sh \
  -c ../configs26/env1-authoring-baw-bai-ae-ads.properties \
  -p /opt/cp4ba-cmgr/cert-kubernetes/scripts \
  -x
```

---

## Post-Installation Access

| Service | URL Pattern |
|---|---|
| CP4BA Console (Zen) | `https://cpd-cp4ba-baw-bai-ae-ads-auth.apps.<cluster-domain>` |
| Workflow Center (BAW Authoring) | `https://cpd-cp4ba-baw-bai-ae-ads-auth.apps.<cluster-domain>/bas/...` |
| App Designer | Accessible from BAStudio |
| ADS Decision Designer | Accessible from BAStudio |
| BAI / Business Performance Center | `https://cpd-cp4ba-baw-bai-ae-ads-auth.apps.<cluster-domain>/bai/...` |

---

## Reference

- [CP4BA BAW Authoring parameters](https://www.ibm.com/docs/en/cloud-paks/cp-biz-automation/26.0.0?topic=parameters-business-automation-workflow-authoring)
- [CP4BA Application Engine parameters](https://www.ibm.com/docs/en/cloud-paks/cp-biz-automation/26.0.0?topic=foundation-application-engine)
- [CP4BA ADS documentation](https://www.ibm.com/docs/en/cloud-paks/cp-biz-automation/26.0.0?topic=capabilities-decision-intelligence-client-managed-software)
- [CP4BA BAI documentation](https://www.ibm.com/docs/en/cloud-paks/cp-biz-automation/26.0.0?topic=baip-event-processing-parameters)
- [CP4BA v26.0.0 documentation](https://www.ibm.com/docs/en/cloud-paks/cp-biz-automation/26.0.0)
