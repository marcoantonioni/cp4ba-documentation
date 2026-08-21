# Recipe: ODM Runtime (Operational Decision Manager — Runtime Only)

> **Recipe suffix**: `decision-odm`  
> **Template**: `cp4ba-installations/templates26/cp4ba-cr-ref-decision-odm.yaml`  
> **Config file**: `cp4ba-installations/configs26/env1-runtime-odm.properties`  
> **CP4BA Version**: 26.0.0  
> **Deployment type**: Production  
> **Platform**: OpenShift Container Platform (OCP)

---

## Purpose

This recipe deploys an **IBM Operational Decision Manager (ODM) Runtime-only** environment. It deploys only the execution components: Decision Server Runtime (DSR) and Decision Server Console (DSC). Decision Center and Decision Runner are **disabled**.

This is the recommended pattern for **production execution namespaces** where business rules are authored in a separate ODM Authoring environment and rulesets are published to this runtime for execution.

---

## CP4BA Capabilities Deployed

| Capability | Status | Description |
|---|---|---|
| **Foundation (CPFS / Zen / IAM)** | ✅ Active | Platform services, web console, identity management |
| **ODM Decision Center (DC)** | ❌ Disabled | `decisionCenter.enabled: false` |
| **ODM Decision Runner (DR)** | ❌ Disabled | `decisionRunner.enabled: false` |
| **ODM Decision Server Runtime (DSR)** | ✅ Active | Executes deployed rulesets |
| **ODM Decision Server Console (DSC)** | ✅ Active | Management console |
| **BAI (Business Automation Insights)** | ❌ Not included | |
| **BAS / BAStudio** | ❌ Not included | ODM does not require BAS |

---

## Deployment Patterns and Optional Components

```properties
CP4BA_INST_DEPL_PATTERNS="foundation,decisions"
CP4BA_INST_OPT_COMPONENTS=""
```

> **Note**: ODM components are activated by the `decisions` pattern. DC and DR are explicitly disabled within `odm_configuration`.

---

## Namespace

```
cp4ba-odm
```

---

## ODM Configuration Details

### Components matrix (runtime-only)

| Component | Enabled | Replicas |
|---|---|---|
| Decision Center (DC) | ❌ `false` (no authoring) | — |
| Decision Runner (DR) | ❌ `false` (no testing) | — |
| Decision Server Runtime (DSR) | ✅ `true` | 1 |
| Decision Server Console (DSC) | ✅ (always) | 1 |

### CR configuration (runtime-only)

```yaml
odm_configuration:
  version: "26.0.0"
  deployment_profile_size: "small"
  debug: false

  dba:
    passwordSecretRef: ibm-odm-keystore-secret

  decisionServerRuntime:
    enabled: true           # RUNTIME: enabled
    replicaCount: 1
    resources:
      requests: { cpu: 500m, memory: 2Gi }
      limits:   { cpu: 2000m, memory: 2Gi }

  decisionServerConsole:    # always present
    resources:
      requests: { cpu: 500m, memory: 2Gi }
      limits:   { cpu: 2000m, memory: 2Gi }

  decisionCenter:
    enabled: false          # RUNTIME: disabled — no DC
    # ... (fields present but DC will not start)

  decisionRunner:
    enabled: false          # RUNTIME: disabled — no DR
    # ... (fields present but DR will not start)
```

---

## Key difference from ODM Authoring

### Datasource difference

In the **runtime-only** template, `dc_odm_decisioncenter_datasource` is **NOT present**:

```yaml
datasource_configuration:
  dc_odm_datasource:              ← ACTIVE (shared config)
    dc_database_type: "postgresql"
    ...

  # dc_odm_decisioncenter_datasource  ← NOT PRESENT (no Decision Center)

  dc_odm_decisionserver_datasource:  ← ACTIVE (runtime only)
    database_type: "postgresql"
    ...
```

---

## Database Configuration

| Datasource key | Database Name | User | Purpose |
|---|---|---|---|
| `dc_odm_datasource` | `odm_odmdb` | `odm` | Shared DB config (server, port, name) |
| `dc_odm_decisionserver_datasource` | `odm_odmdb` | `odm` | Decision Server Runtime config |

> `dc_odm_decisioncenter_datasource` is **absent** in the runtime template — Decision Center is not deployed.

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

---

## Typical Deployment Topology

```
Authoring Namespace (cp4ba-odm-auth)          Runtime Namespace (cp4ba-odm)
  ┌──────────────────────┐                       ┌──────────────────────┐
  │ ODM Decision Center  │  deploy rulesets       │ ODM Decision Server  │
  │ + Decision Runner    │ ─────────────────────► │ Runtime              │
  │ + DSR + DSC          │  (via DC export)       │ + DSC                │
  └──────────────────────┘                       └──────────────────────┘
```

---

## Installation Command

```bash
cd cp4ba-installations/scripts

_PTC=$(pwd)/../configs26

./cp4ba-one-shot-installation.sh \
  -c ${_PTC}/env1-runtime-odm.properties \
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
- [Configuring ODM](https://www.ibm.com/docs/en/cloud-paks/cp-biz-automation/26.0.0?topic=resource-configuring-operational-decision-manager)
- [ODM overview](https://www.ibm.com/docs/en/cloud-paks/cp-biz-automation/26.0.0?topic=capabilities-operational-decision-manager)
