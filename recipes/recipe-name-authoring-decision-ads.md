# Recipe: ADS Authoring (Decision Intelligence — Authoring)

> **Recipe suffix**: `authoring-decision-ads`  
> **Template**: `cp4ba-installations/templates26/cp4ba-cr-ref-authoring-decision-ads.yaml`  
> **Config file**: `cp4ba-installations/configs26/env1-authoring-ads.properties`  
> **CP4BA Version**: 26.0.0  
> **Deployment type**: Production  
> **Platform**: OpenShift Container Platform (OCP)

---

## Purpose

This recipe deploys an **ADS (Automation Decision Services) Authoring** environment. It provides both the Decision Designer (web-based IDE) and the Decision Runtime (execution engine) within the same namespace, backed by Business Automation Studio (BAS/BAStudio) as the authoring shell.

This is the recommended starting point for teams building AI-powered decision services with IBM Decision Intelligence Client Managed Software (DICM / ADS).

---

## CP4BA Capabilities Deployed

| Capability | Status | Description |
|---|---|---|
| **Foundation (CPFS / Zen / IAM)** | ✅ Active | Platform services, web console, identity management |
| **ADS Decision Designer** | ✅ Active | Web IDE for authoring decision models and rules |
| **ADS Decision Runtime** | ✅ Active | Executes deployed decision archives |
| **Business Automation Studio (BAS)** | ✅ Active | Authoring shell required by ADS Designer |
| **BAI (Business Automation Insights)** | ❌ Not included | BAI event emitter is present but disabled |
| **OpenSearch / Kafka** | ❌ Not included | Not deployed in this recipe |

---

## Deployment Patterns and Optional Components

```properties
CP4BA_INST_DEPL_PATTERNS="foundation,decisions_ads"
CP4BA_INST_OPT_COMPONENTS="ads_designer,ads_runtime,bas"
```

---

## Namespace

```
cp4ba-ads-auth
```

---

## ADS Configuration Details

### Decision Designer

- **Enabled**: `true`
- **Admin secret**: `ibm-dba-ads-designer-secret`
- **Profile size**: `small`
- Connects to BAS for the authoring UI shell
- Supports Git integration for storing decision projects (requires `CP4BA_INST_GIT_ENABLED=true` + token)
- Supports GenAI/WML integration via `ads-genai-secret` (pre-created secret)

### Decision Runtime

- **Enabled**: `true` (required for authoring environment — test/preview)
- **Admin secret**: `ibm-dba-ads-runtime-secret`
- **Profile size**: `small`
- **Replicas**: 2
- **Authentication mode**: `zen` (uses CPFS/Zen SSO)
- **Archive storage type**: `fs` (filesystem PVC, 1Gi)
- **BAI event emitter**: `enabled: false` (no BAI in this recipe)
- **Autoscaling**: disabled

### BAI Event Emitter (pre-configured, disabled)

```yaml
event_emitter:
  enabled: false   # BAI is not installed — will be silently ignored
  kafka_topic: "ads-decision-execution-common-data"
  elasticsearch_index: "ads-decision-execution-common-data"
  allow_missing_events: true
  queue_capacity: 50000
  dequeur_threads: 1
```

### Decision Runtime Service Resources

| Resource | Request | Limit |
|---|---|---|
| CPU | 500m | 2000m |
| Memory | 2Gi | 3Gi |
| Ephemeral Storage | 100Mi | 1000Mi |

---

## Business Automation Studio (BAS)

BAS is the authoring IDE that hosts ADS Designer. It includes a Playback Server for testing decision services.

```yaml
bastudio_configuration:
  admin_user: "cp4admin"
  database:
    host: <postgresql-host>
    name: ads_auth_baw_1
    port: "5432"
    type: postgresql
  playback_server:
    admin_user: "cp4admin"
    database:
      host: <postgresql-host>
      name: ads_auth_appdb
      port: "5432"
      type: postgresql
  resources:
    bastudio:
      limits:
        cpu: '5000m'
        memory: 3096Mi
```

---

## Database Configuration

| Datasource | Database Name | Schema/User | Purpose |
|---|---|---|---|
| `dc_ads_designer_datasource` | `ads_auth_adsdesignerdb` | `adsdes` | ADS Designer data |
| `dc_ads_runtime_datasource` | `ads_auth_adsruntimedb` | `adsrt` | ADS Runtime data |
| `dc_icn_datasource` | `ads_auth_icn` | `icn` | IBM Content Navigator (required by BAS) |
| *(bastudio DB)* | `ads_auth_baw_1` | `bawadmin` | BAS main database |
| *(playback DB)* | `ads_auth_appdb` | `pbk` | BAS Playback Server |

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
- Users from `_cfg-production-ldap-domain.properties` onboarded to IAM
- IAM admin: `cpadmin`
- Pak admin: `cp4admin`

---

## GenAI Integration (optional)

ADS includes a pre-configured GenAI secret reference (`ads-genai-secret`) for watsonx.ai integration. To activate:

```bash
export CP4BA_INST_ADS_GENAI_APIKEY="<your-ibm-cloud-api-key>"
export CP4BA_INST_ADS_GENAI_PRJ_ID="<your-watsonx-project-id>"
export CP4BA_INST_ADS_GENAI_ML_URL="https://us-south.ml.cloud.ibm.com"
```

---

## Installation Command

```bash
cd cp4ba-installations/scripts

_PTC=$(pwd)/../configs26

./cp4ba-one-shot-installation.sh \
  -c ${_PTC}/env1-authoring-ads.properties \
  -m \
  -v 26.0.0 \
  -k 26.0.0
```

**Flags**:
- `-c` — path to the config properties file
- `-m` — install a fresh CP4BA Case Package Manager (use only on first deployment)
- `-v` — CP4BA version
- `-k` — cert-kubernetes version
- `-o` — (optional) skip operator installation if already deployed in namespace
- `-x` — (optional) enable trace output

---

## References

- [ADS Decision Designer parameters](https://www.ibm.com/docs/en/cloud-paks/cp-biz-automation/26.0.0?topic=parameters-decision-designer)
- [ADS Decision Runtime parameters](https://www.ibm.com/docs/en/cloud-paks/cp-biz-automation/26.0.0?topic=parameters-decision-runtime)
- [ADS shared parameters](https://www.ibm.com/docs/en/cloud-paks/cp-biz-automation/26.0.0?topic=parameters-shared-by-decision-designer-decision-runtime)
- [DICM overview](https://www.ibm.com/docs/en/cloud-paks/cp-biz-automation/26.0.0?topic=capabilities-decision-intelligence-client-managed-software)
