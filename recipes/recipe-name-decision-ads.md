# Recipe: ADS Runtime (Decision Intelligence — Runtime Only)

> **Recipe suffix**: `decision-ads`  
> **Template**: `cp4ba-installations/templates26/cp4ba-cr-ref-decision-ads.yaml`  
> **Config file**: `cp4ba-installations/configs26/env1-runtime-ads.properties`  
> **CP4BA Version**: 26.0.0  
> **Deployment type**: Production  
> **Platform**: OpenShift Container Platform (OCP)

---

## Purpose

This recipe deploys an **ADS (Automation Decision Services) Runtime-only** environment. It deploys only the **Decision Runtime** (execution engine) without the Decision Designer (authoring IDE) or BAS.

This is the recommended pattern for **production execution namespaces** where decisions are deployed from a separate authoring environment. The runtime receives and executes decision archives published from an ADS Authoring instance.

---

## CP4BA Capabilities Deployed

| Capability | Status | Description |
|---|---|---|
| **Foundation (CPFS / Zen / IAM)** | ✅ Active | Platform services, web console, identity management |
| **ADS Decision Designer** | ❌ Disabled | `decision_designer.enabled: false` |
| **ADS Decision Runtime** | ✅ Active | Executes deployed decision archives |
| **Business Automation Studio (BAS)** | ❌ Not included | Not needed for runtime-only |
| **BAI (Business Automation Insights)** | ❌ Not included | BAI event emitter present but disabled |
| **OpenSearch / Kafka** | ❌ Not included | Not deployed in this recipe |

---

## Deployment Patterns and Optional Components

```properties
CP4BA_INST_DEPL_PATTERNS="foundation,decisions_ads"
CP4BA_INST_OPT_COMPONENTS="ads_runtime"
```

---

## Namespace

```
cp4ba-ads-prod
```

---

## ADS Configuration Details

### Decision Designer (disabled in runtime-only)

```yaml
decision_designer:
  enabled: false   # runtime-only: Designer NOT deployed
```

### Decision Runtime

- **Enabled**: `true`
- **Admin secret**: `ibm-dba-ads-runtime-secret`
- **Profile size**: `small`
- **Replicas**: 2
- **Authentication mode**: `zen`
- **Archive storage type**: `fs` (PVC, 1Gi)
- **Autoscaling**: disabled
- **BAI event emitter**: `enabled: false` (no BAI in this recipe)

### Key difference from ADS Authoring

In the runtime-only template, the `dc_ads_designer_datasource` is **commented out** — only `dc_ads_runtime_datasource` and `dc_icn_datasource` are active:

```yaml
datasource_configuration:
  ##  dc_ads_designer_datasource:    ← COMMENTED OUT (not needed)
  ##    ...

  dc_ads_runtime_datasource:         ← ACTIVE
    current_schema: "adsrt"
    database_name: "ads_prod_adsruntimedb"
    ...

  dc_icn_datasource:                 ← ACTIVE (required even in runtime)
    database_name: "ads_prod_icn"
    ...
```

### Decision Runtime Service Resources

| Resource | Request | Limit |
|---|---|---|
| CPU | 500m | 2000m |
| Memory | 2Gi | 3Gi |
| Ephemeral Storage | 100Mi | 1000Mi |

### Decision Selection Configuration

```yaml
decision_selection:
  threads: 1
  update_interval: 120000   # 2 minutes
  query_interval: 1000      # 1 second
  cache:
    config:
      expiry: ''
      resources: "<heap unit=\"entries\">100</heap>"
```

---

## Database Configuration

| Datasource key | Database Name | Schema/User | Purpose |
|---|---|---|---|
| `dc_ads_runtime_datasource` | `ads_prod_adsruntimedb` | `adsrt` | ADS Runtime data |
| `dc_icn_datasource` | `ads_prod_icn` | `icn` | IBM Content Navigator |

> The `dc_ads_designer_datasource` is NOT present in the runtime template — it is commented out.

**DB Server**: PostgreSQL 18.4 (OSS), SSL-only, port 5432  
**SQL template**: `db-statements-ref-ads.sql`

### DB Secrets

| Secret | Purpose |
|---|---|
| `ibm-ads-runtime-database` | PostgreSQL credentials for ADS Runtime |
| `ibm-dba-ads-runtime-secret` | ADS Runtime admin secret |

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

## Rolling Update Strategy

```yaml
rolling_update:
  max_unavailable: 1
  max_surge: 1
```

---

## TLS Configuration

```yaml
tls:
  allow_self_signed: true
  verify_hostname: false
```

---

## Typical Deployment Topology

```
Authoring Namespace (cp4ba-ads-auth)          Runtime Namespace (cp4ba-ads-prod)
  ┌─────────────────────┐                        ┌─────────────────────┐
  │ ADS Decision        │  publish archives       │ ADS Decision        │
  │ Designer            │ ──────────────────────► │ Runtime             │
  │ + BAS               │  (via Git + deploy)     │ (runtime-only)      │
  └─────────────────────┘                        └─────────────────────┘
```

---

## Installation Command

```bash
cd cp4ba-installations/scripts

_PTC=$(pwd)/../configs26

./cp4ba-one-shot-installation.sh \
  -c ${_PTC}/env1-runtime-ads.properties \
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
- [ADS shared parameters](https://www.ibm.com/docs/en/cloud-paks/cp-biz-automation/26.0.0?topic=parameters-shared-by-decision-designer-decision-runtime)
- [DICM overview](https://www.ibm.com/docs/en/cloud-paks/cp-biz-automation/26.0.0?topic=capabilities-decision-intelligence-client-managed-software)
