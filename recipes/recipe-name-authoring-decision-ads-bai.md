# Recipe: ADS Authoring + BAI (Decision Intelligence — Authoring with Business Automation Insights)

> **Recipe suffix**: `authoring-decision-ads-bai`  
> **Template**: `cp4ba-installations/templates26/cp4ba-cr-ref-authoring-decision-ads-bai.yaml`  
> **Config file**: `cp4ba-installations/configs26/env1-authoring-ads-bai.properties`  
> **CP4BA Version**: 26.0.0  
> **Deployment type**: Production  
> **Platform**: OpenShift Container Platform (OCP)

---

## Purpose

This recipe deploys an **ADS (Automation Decision Services) Authoring** environment with **Business Automation Insights (BAI)** enabled. It extends the base ADS Authoring recipe by adding event processing and analytics capabilities, enabling decision execution events to be emitted to Kafka and indexed in OpenSearch for dashboards and monitoring.

---

## CP4BA Capabilities Deployed

| Capability | Status | Description |
|---|---|---|
| **Foundation (CPFS / Zen / IAM)** | ✅ Active | Platform services, web console, identity management |
| **ADS Decision Designer** | ✅ Active | Web IDE for authoring decision models and rules |
| **ADS Decision Runtime** | ✅ Active | Executes deployed decision archives |
| **Business Automation Studio (BAS)** | ✅ Active | Authoring shell required by ADS Designer |
| **BAI (Business Automation Insights)** | ✅ Active | Event processing, Flink, Business Performance Center |
| **OpenSearch** | ✅ Active | Event index and analytics storage |
| **Kafka** | ✅ Active | Event streaming bus for BAI pipeline |

---

## Deployment Patterns and Optional Components

```properties
CP4BA_INST_DEPL_PATTERNS="foundation,decisions_ads"
CP4BA_INST_OPT_COMPONENTS="ads_designer,ads_runtime,bas,bai"
```

---

## Namespace

```
cp4ba-ads-bai-auth
```

---

## ADS Configuration Details

### Decision Designer

- **Enabled**: `true`
- **Admin secret**: `ibm-dba-ads-designer-secret`
- **Profile size**: `small`

### Decision Runtime

- **Enabled**: `true`
- **Replicas**: 2
- **Authentication mode**: `zen`
- **Archive storage type**: `fs` (PVC 1Gi)
- **BAI event emitter**: configured and active when BAI is installed

### BAI Event Emitter (active)

```yaml
event_emitter:
  enabled: true     # BAI IS installed in this recipe
  kafka_topic: "ads-decision-execution-common-data"
  elasticsearch_index: "ads-decision-execution-common-data"
  allow_missing_events: true
  queue_capacity: 50000
  dequeur_threads: 1
```

> The `enabled` field value is driven by `CP4BA_INST_ADS_BAI_EMITTER_ENABLED` in the properties file. When `bai` is in optional components this should be set to `true`.

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
    install: true    # ADS events enabled
  odm:
    install: false
```

### BAI components activated

| Processor | Enabled | Reason |
|---|---|---|
| `ads` | ✅ | ADS is installed |
| `bpmn` | ❌ | BAW not installed |
| `bawadv` | ❌ | BAW Advanced not installed |
| `content` | ❌ | FileNet not installed |
| `odm` | ❌ | ODM not installed |

---

## Business Automation Studio (BAS)

Same configuration as the base ADS Authoring recipe:

```yaml
bastudio_configuration:
  admin_user: "cp4admin"
  database:
    host: <postgresql-host>
    name: ads_bai_auth_baw_1
    port: "5432"
    type: postgresql
  playback_server:
    admin_user: "cp4admin"
    database:
      host: <postgresql-host>
      name: ads_bai_auth_appdb
      port: "5432"
      type: postgresql
```

---

## Database Configuration

| Datasource | Database Name | Schema/User | Purpose |
|---|---|---|---|
| `dc_ads_designer_datasource` | `ads_bai_auth_adsdesignerdb` | `adsdes` | ADS Designer data |
| `dc_ads_runtime_datasource` | `ads_bai_auth_adsruntimedb` | `adsrt` | ADS Runtime data |
| `dc_icn_datasource` | `ads_bai_auth_icn` | `icn` | IBM Content Navigator (BAS dependency) |
| *(bastudio DB)* | `ads_bai_auth_baw_1` | `bawadmin` | BAS main database |
| *(playback DB)* | `ads_bai_auth_appdb` | `pbk` | BAS Playback Server |

**DB Server**: PostgreSQL 18.4 (OSS), SSL-only, port 5432  
**SQL template**: `db-statements-ref-ads.sql`

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
  -c ${_PTC}/env1-authoring-ads-bai.properties \
  -m \
  -v 26.0.0 \
  -k 26.0.0
```

**Flags**:
- `-c` — path to the config properties file
- `-m` — install a fresh CP4BA Case Package Manager (use only on first deployment)
- `-v` — CP4BA version
- `-k` — cert-kubernetes version
- `-o` — (optional) skip operator installation if already deployed
- `-x` — (optional) enable trace output

---

## References

- [ADS Decision Designer parameters](https://www.ibm.com/docs/en/cloud-paks/cp-biz-automation/26.0.0?topic=parameters-decision-designer)
- [ADS Decision Runtime parameters](https://www.ibm.com/docs/en/cloud-paks/cp-biz-automation/26.0.0?topic=parameters-decision-runtime)
- [BAI event processing parameters](https://www.ibm.com/docs/en/cloud-paks/cp-biz-automation/26.0.0?topic=baip-event-processing-parameters)
- [DICM overview](https://www.ibm.com/docs/en/cloud-paks/cp-biz-automation/26.0.0?topic=capabilities-decision-intelligence-client-managed-software)
