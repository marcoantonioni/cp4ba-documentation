# Recipe: ODM Authoring (Operational Decision Manager — Full Authoring)

> **Recipe suffix**: `authoring-decision-odm`  
> **Template**: `cp4ba-installations/templates26/cp4ba-cr-ref-authoring-decision-odm.yaml`  
> **Config file**: `cp4ba-installations/configs26/env1-authoring-odm.properties`  
> **CP4BA Version**: 26.0.0  
> **Deployment type**: Production  
> **Platform**: OpenShift Container Platform (OCP)

---

## Purpose

This recipe deploys a **full IBM Operational Decision Manager (ODM) Authoring** environment. It includes all four ODM components: Decision Center (authoring IDE), Decision Runner (test/simulate), Decision Server Runtime (execution engine), and Decision Server Console (management).

This is the recommended setup for business rule authoring teams who need to create, govern, and test business rules before deploying them to production.

---

## CP4BA Capabilities Deployed

| Capability | Status | Description |
|---|---|---|
| **Foundation (CPFS / Zen / IAM)** | ✅ Active | Platform services, web console, identity management |
| **ODM Decision Center (DC)** | ✅ Active | Web IDE for authoring and governing business rules |
| **ODM Decision Runner (DR)** | ✅ Active | Test/simulate rulesets from Decision Center |
| **ODM Decision Server Runtime (DSR)** | ✅ Active | Executes deployed rulesets |
| **ODM Decision Server Console (DSC)** | ✅ Active | Management console for deployed rulesets |
| **BAI (Business Automation Insights)** | ❌ Not included | Not in this recipe |
| **BAS / BAStudio** | ❌ Not included | ODM does not require BAS |

---

## Deployment Patterns and Optional Components

```properties
CP4BA_INST_DEPL_PATTERNS="foundation,decisions"
CP4BA_INST_OPT_COMPONENTS=""
```

> **Note**: ODM components are activated by the `decisions` pattern. The individual ODM components (DC, DR, DSR, DSC) are controlled by flags within `odm_configuration`, not by optional components.

---

## Namespace

```
cp4ba-odm-auth
```

---

## ODM Configuration Details

### All components enabled (authoring mode)

| Component | Enabled | Replicas |
|---|---|---|
| Decision Center (DC) | ✅ `true` | 1 |
| Decision Runner (DR) | ✅ `true` | 1 |
| Decision Server Runtime (DSR) | ✅ `true` | 1 |
| Decision Server Console (DSC) | ✅ (always) | 1 |

### CR configuration excerpt

```yaml
odm_configuration:
  version: "26.0.0"
  deployment_profile_size: "small"
  debug: false

  audit_logging:
    enabled: false
    rolling_max_files: 5
    rolling_max_size: '20Mi'

  dba:
    passwordSecretRef: ibm-odm-keystore-secret

  decisionServerRuntime:
    enabled: true
    replicaCount: 1
    resources:
      requests: { cpu: 500m, memory: 2Gi }
      limits:   { cpu: 2000m, memory: 2Gi }

  decisionServerConsole:
    resources:
      requests: { cpu: 500m, memory: 2Gi }
      limits:   { cpu: 2000m, memory: 2Gi }

  decisionCenter:
    enabled: true       # authoring: true
    persistenceLocale: en_US
    replicaCount: 1
    disableAllAuthenticatedUser: false
    resources:
      requests: { cpu: 500m, memory: 2Gi }
      limits:   { cpu: 2000m, memory: 4Gi }

  decisionRunner:
    enabled: true       # authoring: true
    replicaCount: 1
    resources:
      requests: { cpu: 500m, memory: 2Gi }
      limits:   { cpu: 2000m, memory: 4Gi }

  show_scim_connection: false   # set true to enable SCIM import in DC
```

### Resource summary

| Component | CPU Request | CPU Limit | Mem Request | Mem Limit |
|---|---|---|---|---|
| Decision Center | 500m | 2000m | 2Gi | 4Gi |
| Decision Runner | 500m | 2000m | 2Gi | 4Gi |
| Decision Server Runtime | 500m | 2000m | 2Gi | 2Gi |
| Decision Server Console | 500m | 2000m | 2Gi | 2Gi |

---

## Database Configuration

ODM authoring uses three datasource entries:

| Datasource key | Database Name | User | Purpose |
|---|---|---|---|
| `dc_odm_datasource` | `odm_auth_odmdb` | `odm` | Shared DB config (server, port, name) |
| `dc_odm_decisioncenter_datasource` | `odm_auth_odmdb` | `odm` | Decision Center specific config |
| `dc_odm_decisionserver_datasource` | `odm_auth_odmdb` | `odm` | Decision Server Runtime config |

> All three datasources point to the **same database** (`odmdb`). ODM uses schemas/tablespaces to separate data internally.

**DB Server**: PostgreSQL 18.4 (OSS), SSL-only, port 5432  
**SQL template**: `db-statements-ref-odm.sql`

### DB Secrets

| Secret | Purpose |
|---|---|
| `ibm-odm-db-secret` | PostgreSQL credentials for ODM |
| `ibm-odm-keystore-secret` | ODM keystore password |

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
- SCIM connection to Decision Center: disabled by default (`show_scim_connection: false`)
  - Set `CP4BA_INST_ODM_ENABLE_SCIM_DEC_CENTER=true` to enable user/group import in DC

---

## Probe Configuration

```yaml
readinessProbe:
  initialDelaySeconds: 5
  periodSeconds: 5
  failureThreshold: 45

livenessProbe:
  initialDelaySeconds: 300
  periodSeconds: 30
  failureThreshold: 4

startupProbe:
  initialDelaySeconds: 15
  periodSeconds: 20
  failureThreshold: 30
```

---

## Installation Command

```bash
cd cp4ba-installations/scripts

_PTC=$(pwd)/../configs26

./cp4ba-one-shot-installation.sh \
  -c ${_PTC}/env1-authoring-odm.properties \
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
