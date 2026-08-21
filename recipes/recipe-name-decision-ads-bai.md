# Recipe: ADS Runtime + BAI (Decision Intelligence — Runtime with Business Automation Insights)

> **Recipe suffix**: `decision-ads-bai`  
> **Template**: `cp4ba-installations/templates26/cp4ba-cr-ref-decision-ads-bai.yaml`  
> **Config file**: `cp4ba-installations/configs26/env1-runtime-ads-bai.properties`  
> **CP4BA Version**: 26.0.0  
> **Deployment type**: Production  
> **Platform**: OpenShift Container Platform (OCP)

---

## Purpose

This recipe deploys an **ADS (Automation Decision Services) Runtime-only** environment with **Business Automation Insights (BAI)** added. It deploys only the Decision Runtime (no Designer, no BAS) but enables decision execution event collection, processing via Apache Flink, and analytics dashboards in OpenSearch.

This is the recommended pattern for **production execution namespaces** that require operational monitoring and analytics, but where decision authoring is performed in a separate dedicated authoring environment.

---

## CP4BA Capabilities Deployed

| Capability | Status | Description |
|---|---|---|
| **Foundation (CPFS / Zen / IAM)** | ✅ Active | Platform services, web console, identity management |
| **ADS Decision Designer** | ❌ Disabled | `decision_designer.enabled: false` |
| **ADS Decision Runtime** | ✅ Active | Executes deployed decision archives |
| **Business Automation Studio (BAS)** | ❌ Not included | Not needed for runtime-only |
| **BAI (Business Automation Insights)** | ✅ Active | Event processing, Flink, BPC dashboards |
| **OpenSearch** | ✅ Active | Event index and analytics |
| **Kafka** | ✅ Active | Event streaming bus |

---

## Deployment Patterns and Optional Components

```properties
CP4BA_INST_DEPL_PATTERNS="foundation,decisions_ads"
CP4BA_INST_OPT_COMPONENTS="ads_runtime,bai"
```

---

## Namespace

```
cp4ba-ads-bai-prod
```

---

## ADS Configuration Details

### Decision Designer (disabled)

```yaml
decision_designer:
  enabled: false    # runtime-only: Designer NOT deployed
```

### Decision Runtime

- **Enabled**: `true`
- **Replicas**: 2
- **Authentication mode**: `zen`
- **Archive storage type**: `fs` (PVC, 1Gi)
- **BAI event emitter**: `enabled: true` (BAI IS installed)

### BAI Event Emitter (active in this recipe)

```yaml
event_emitter:
  enabled: true    # BAI is installed
  kafka_topic: "ads-decision-execution-common-data"
  elasticsearch_index: "ads-decision-execution-common-data"
  allow_missing_events: true
  queue_capacity: 50000
  dequeur_threads: 1
```

> The value of `enabled` is driven by `CP4BA_INST_ADS_BAI_EMITTER_ENABLED` in the properties file. Set it to `true` when BAI is in optional components.

### Decision Runtime Service Resources

| Resource | Request | Limit |
|---|---|---|
| CPU | 500m | 2000m |
| Memory | 2Gi | 3Gi |
| Ephemeral Storage | 100Mi | 1000Mi |

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
| `ads` | ✅ | ADS Runtime is installed |
| `odm` | ❌ | ODM not installed |
| `bpmn` | ❌ | BAW not installed |

---

## Database Configuration

| Datasource key | Database Name | Schema/User | Purpose |
|---|---|---|---|
| `dc_ads_runtime_datasource` | `ads_bai_prod_adsruntimedb` | `adsrt` | ADS Runtime data |
| `dc_icn_datasource` | `ads_bai_prod_icn` | `icn` | IBM Content Navigator |

> The `dc_ads_designer_datasource` is NOT present (commented out) — this is a runtime-only deployment.

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

---

## Typical Deployment Topology

```
Authoring Namespace (cp4ba-ads-auth)          Prod Runtime Namespace (cp4ba-ads-bai-prod)
  ┌─────────────────────┐                        ┌─────────────────────────────┐
  │ ADS Decision        │  publish archives       │ ADS Decision Runtime        │
  │ Designer + BAS      │ ──────────────────────► │ + BAI (Flink+OpenSearch)    │
  └─────────────────────┘                        │ + Kafka event streaming     │
                                                  └─────────────────────────────┘
                                                           │
                                                           ▼
                                                  Business Performance Center
                                                  (decision execution dashboards)
```

---

## Installation Command

```bash
cd cp4ba-installations/scripts

_PTC=$(pwd)/../configs26

./cp4ba-one-shot-installation.sh \
  -c ${_PTC}/env1-runtime-ads-bai.properties \
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

- [ADS Decision Runtime parameters](https://www.ibm.com/docs/en/cloud-paks/cp-biz-automation/26.0.0?topic=parameters-decision-runtime)
- [BAI event processing parameters](https://www.ibm.com/docs/en/cloud-paks/cp-biz-automation/26.0.0?topic=baip-event-processing-parameters)
- [DICM overview](https://www.ibm.com/docs/en/cloud-paks/cp-biz-automation/26.0.0?topic=capabilities-decision-intelligence-client-managed-software)
