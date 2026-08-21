# Recipe: ODM Authoring + BAI (Operational Decision Manager — Authoring with Business Automation Insights)

> **Recipe suffix**: `authoring-decision-odm-bai`  
> **Template**: `cp4ba-installations/templates26/cp4ba-cr-ref-authoring-decision-odm-bai.yaml`  
> **Config file**: `cp4ba-installations/configs26/env1-authoring-odm-bai.properties`  
> **CP4BA Version**: 26.0.0  
> **Deployment type**: Production  
> **Platform**: OpenShift Container Platform (OCP)

---

## Purpose

This recipe deploys a **full IBM Operational Decision Manager (ODM) Authoring** environment with **Business Automation Insights (BAI)** added. It includes all four ODM components (Decision Center, Decision Runner, Decision Server Runtime, Decision Server Console) and enables ODM rule execution event processing through Apache Flink and OpenSearch.

This is the recommended setup for teams that want to author business rules AND gain operational visibility through dashboards and analytics.

---

## CP4BA Capabilities Deployed

| Capability | Status | Description |
|---|---|---|
| **Foundation (CPFS / Zen / IAM)** | ✅ Active | Platform services, web console, identity management |
| **ODM Decision Center (DC)** | ✅ Active | Web IDE for authoring and governing business rules |
| **ODM Decision Runner (DR)** | ✅ Active | Test/simulate rulesets from Decision Center |
| **ODM Decision Server Runtime (DSR)** | ✅ Active | Executes deployed rulesets |
| **ODM Decision Server Console (DSC)** | ✅ Active | Management console for deployed rulesets |
| **BAI (Business Automation Insights)** | ✅ Active | Event processing, Flink, BPC dashboards |
| **OpenSearch** | ✅ Active | Event index and analytics storage (via BAI) |
| **Kafka** | ✅ Active | Event streaming bus |
| **BAS / BAStudio** | ❌ Not included | ODM does not require BAS |

---

## Deployment Patterns and Optional Components

```properties
CP4BA_INST_DEPL_PATTERNS="foundation,decisions"
CP4BA_INST_OPT_COMPONENTS="bai"
```

> **Note**: OpenSearch and Kafka are implicit dependencies of the `bai` component and are automatically deployed.

---

## Namespace

```
cp4ba-odm-bai-auth
```

---

## ODM Configuration Details

All four ODM components are active (same as the ODM Authoring recipe without BAI):

| Component | Enabled | Replicas |
|---|---|---|
| Decision Center (DC) | ✅ `true` | 1 |
| Decision Runner (DR) | ✅ `true` | 1 |
| Decision Server Runtime (DSR) | ✅ `true` | 1 |
| Decision Server Console (DSC) | ✅ (always) | 1 |

### ODM BAI Kafka Topic

ODM can emit rule execution events to Kafka for BAI processing. The topic is configured via:

```yaml
odm_configuration:
  customization:
    bai_kafka_topic:   # leave empty to use default, or set custom topic name
```

---

## BAI Configuration

```yaml
bai_configuration:
  settings:
    egress: false
  business_performance_center:
    install: true
    all_users_access: true
  bpmn:
    install: false
  bawadv:
    install: false
  content:
    install: false
  event_forwarder:
    install: false
  flink:
    create_route: true
  icm:
    install: false
  navigator:
    install: false
  ads:
    install: false   # ADS not installed
  odm:
    install: true    # ODM events enabled
```

### BAI components activated

| Processor | Enabled | Reason |
|---|---|---|
| `odm` | ✅ | ODM is installed |
| `ads` | ❌ | ADS not installed |
| `bpmn` | ❌ | BAW not installed |
| `content` | ❌ | FileNet not installed |

---

## Database Configuration

ODM authoring uses three datasource entries — all pointing to the same database:

| Datasource key | Database Name | User | Purpose |
|---|---|---|---|
| `dc_odm_datasource` | `odm_bai_auth_odmdb` | `odm` | Shared DB config |
| `dc_odm_decisioncenter_datasource` | `odm_bai_auth_odmdb` | `odm` | Decision Center config |
| `dc_odm_decisionserver_datasource` | `odm_bai_auth_odmdb` | `odm` | Decision Server Runtime config |

**DB Server**: PostgreSQL 18.4 (OSS), SSL-only, port 5432  
**SQL template**: `db-statements-ref-odm.sql`

### DB Secrets

| Secret | Purpose |
|---|---|
| `ibm-odm-db-secret` | PostgreSQL credentials for ODM |
| `ibm-odm-keystore-secret` | ODM keystore password |

---

## Resource Summary

| Component | CPU Request | CPU Limit | Mem Request | Mem Limit |
|---|---|---|---|---|
| Decision Center | 500m | 2000m | 2Gi | 4Gi |
| Decision Runner | 500m | 2000m | 2Gi | 4Gi |
| Decision Server Runtime | 500m | 2000m | 2Gi | 2Gi |
| Decision Server Console | 500m | 2000m | 2Gi | 2Gi |

---

## Storage

| Class variable | Value |
|---|---|
| File (CephFS) | `ocs-external-storagecluster-cephfs` |
| Block (Ceph RBD) | `ocs-external-storagecluster-ceph-rbd` |

---

## LDAP / IAM

- Local OpenLDAP pod deployed in-namespace
- IAM admin: `cpadmin`
- Pak admin: `cp4admin`

---

## Installation Command

```bash
cd cp4ba-installations/scripts

_PTC=$(pwd)/../configs26

./cp4ba-one-shot-installation.sh \
  -c ${_PTC}/env1-authoring-odm-bai.properties \
  -m \
  -v 26.0.0 \
  -k 26.0.0
```

**Flags**:
- `-c` — path to the config properties file
- `-m` — install CP4BA Case Package Manager (first time only)
- `-v` — CP4BA version
- `-k` — cert-kubernetes version
- `-o` — (optional) skip operator installation
- `-x` — (optional) enable trace output

---

## References

- [ODM configuration parameters](https://www.ibm.com/docs/en/cloud-paks/cp-biz-automation/26.0.0?topic=parameters-operational-decision-manager)
- [ODM decision management event processing walkthrough](https://www.ibm.com/docs/en/cloud-paks/cp-biz-automation/26.0.0?topic=events-decision-management-event-processing-walkthrough)
- [BAI event processing parameters](https://www.ibm.com/docs/en/cloud-paks/cp-biz-automation/26.0.0?topic=baip-event-processing-parameters)
- [ODM overview](https://www.ibm.com/docs/en/cloud-paks/cp-biz-automation/26.0.0?topic=capabilities-operational-decision-manager)
