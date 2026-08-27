# Recipe: RPA (Robotic Process Automation)

> **Recipe suffix**: `rpa`  
> **Template**: [`cp4ba-cr-ref-rpa.yaml`](../cp4ba-installations/templates26/cp4ba-cr-ref-rpa.yaml)  
> **Config file**: [`env1-runtime-rpa.properties`](../cp4ba-installations/configs26/env1-runtime-rpa.properties)  
> **Disclaimer**: These configurations are not intended for production environments. The purpose is purely educational.

---

## Overview

This recipe deploys IBM Robotic Process Automation (RPA) on OpenShift. RPA is **not deployed via the standard `ICP4ACluster` CR** like other CP4BA capabilities. Instead, it uses:

1. A minimal `ICP4ACluster` CR (foundation only) — to provide the IAM/Zen context for SSO
2. A dedicated **IBM MQ Operator** — for RPA's internal messaging
3. A dedicated **IBM RPA Operator** — which manages the `RpaServer` CR (instance + tenant)
4. A **Microsoft SQL Server** pod — as the RPA database backend

---

## Deployment Details

| Property | Value |
|---|---|
| **Namespace** | `cp4ba-rpa` |
| **CR Name** | `icp4adeploy` |
| **CR Kind** | `ICP4ACluster` |
| **Deployment Type** | `Production` |
| **Deployment Platform** | `OCP` |
| **Profile Size** | `small` |
| **CP4BA Version** | `26.0.0` |
| **License Type** | `production` |

---

## CP4BA Patterns and Optional Components

```properties
CP4BA_INST_DEPL_PATTERNS=foundation
CP4BA_INST_OPT_COMPONENTS=
```

> The `ICP4ACluster` CR in this recipe activates **only the Foundation pattern** (no workflow, no decisions, no content). This provides the Zen/IAM infrastructure that RPA integrates with for Single Sign-On.

---

## Capabilities Deployed

### 1. Foundation (CP4BA CR)
**Pattern**: `foundation`

Provides the baseline IAM/Zen platform:
- IBM Cloud Pak Foundational Services (CPFS/Zen): IAM, License Service, CP4D UI
- Resource Registry (etcd-based)

**Foundation databases** (PostgreSQL, SSL-enabled):

| DB Variable | Purpose |
|---|---|
| `CP4BA_INST_ICN_DB_NAME` = `rpa_icn` | IBM Content Navigator |
| `CP4BA_INST_DB_BTS_USER` | Business Team Service |
| `CP4BA_INST_DB_IM_USER` | Identity Management |
| `CP4BA_INST_DB_ZEN_USER` | Zen/CPD control plane |

### 2. IBM RPA (separate operator)
**Operator channel**: `v3.3`  
**Starting CSV**: `ibm-automation-rpa.v3.3.0`  
**Operator image**: `icr.io/cpopen/ibm-rpa-operator-catalog:latest`

The RPA Operator manages the full RPA lifecycle including:
- RPA Server (robot execution engine)
- Tenant management
- Script repository
- Attended / unattended automation control plane

**RPA instance configuration:**

| Parameter | Value |
|---|---|
| Instance name | `rpa` |
| Tenant name | `ibm` |
| Tenant owner | `cp4admin` |
| Tenant owner email | `cp4admin@vuxprod.net` |

### 3. IBM MQ (separate operator)
**Operator channel**: `v3.9`  
**Operator image**: `icr.io/cpopen/ibm-mq-operator-catalog@sha256:0a0bc44cde96ff5e855b2276c32e0abad79ec3c2fbbc95bdc9426d0ac046b5a6`

MQ is required by RPA for internal asynchronous messaging between components.

---

## RPA Database — Microsoft SQL Server

RPA uses **Microsoft SQL Server** (not PostgreSQL). A SQL Server pod is deployed in the same namespace.

| Parameter | Value |
|---|---|
| Image | `mcr.microsoft.com/mssql/server:2025-latest` |
| SQL instance name | `SQLEXPRESS` |
| Service account | `ibm-cp4ba-anyuid` |
| DB secret | `rpa-mssql` |
| DB admin user | `sa` |
| DB admin password | `dem0s-dem0s` |
| TCP port | `1433` |
| NodePort (external) | `31433` |
| PVC name | `mssql-data` |
| PVC size | `8Gi` |
| Deployment name | `rpa-mssql` |

### SQL Server Databases

RPA requires **5 SQL Server databases**:

| Database | Connection string variable | Purpose |
|---|---|---|
| `address` | `CP4BA_INST_RPA_DB_CONN_PARAMS_ADDRESS` | Address book / user directory |
| `automation` | `CP4BA_INST_RPA_DB_CONN_PARAMS_AUTOMATION` | Automation assets (scripts, bots) |
| `knowledge` | `CP4BA_INST_RPA_DB_CONN_PARAMS_KNOWLEDGE` | Knowledge base for NLP |
| `wordnet` | `CP4BA_INST_RPA_DB_CONN_PARAMS_WORDNET` | WordNet lexical database |
| `audit` | `CP4BA_INST_RPA_DB_CONN_PARAMS_AUDIT` | Audit trail |

### Connection String Format

```
Data Source=rpa-mssql-service.<namespace>.svc.cluster.local\SQLEXPRESS,1433;
Initial Catalog=<db>;
User ID=sa;Password=dem0s-dem0s;
Connect Timeout=30;Encrypt=False;TrustServerCertificate=False;
ApplicationIntent=ReadWrite;MultiSubnetFailover=False
```

---

## Supporting PostgreSQL (Foundation only)

A small PostgreSQL StatefulSet is deployed for the Foundation databases (ICN, BTS, IM, Zen):

| Parameter | Value |
|---|---|
| DB CR name | `my-postgres-1-for-cp4ba-ssl` |
| SQL template | `db-statements-ref-zenbtsim.sql` |
| Storage size | `10Gi` |
| SSL-only | `true` |
| OSS image | `postgres:18.4` |

---

## Storage Configuration

| Storage Class | Type | Default Value |
|---|---|---|
| File (RWX) | CephFS | `ocs-external-storagecluster-cephfs` |
| Block (RWO) | Ceph RBD | `ocs-external-storagecluster-ceph-rbd` |

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

```bash
CP4BA_AUTO_PRIVATE_CATALOG=Yes
CP4BA_AUTO_SEPARATE_OPERATOR=No
CP4BA_AUTO_ALL_NAMESPACES=No
CP4BA_AUTO_OPERATOR_NAMESPACE=cp4ba-rpa
CP4BA_AUTO_CS_SERVICE_NAMESPACE=cp4ba-rpa
```

---

## RPA SMTP Configuration

RPA requires an SMTP server for notifications and tenant owner email delivery:

| Parameter | Value |
|---|---|
| `CP4BA_INST_RPA_SMTP_USER` | `cp4admin` |
| `CP4BA_INST_RPA_SMTP_PASSWORD` | `dem0s` |

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
  -c ../configs26/env1-runtime-rpa.properties \
  -m \
  -d /opt/cp4ba-cmgr
```

### Subsequent installations (reuse existing Case Package Manager)

```bash
cd cp4ba-installations/scripts

./cp4ba-one-shot-installation.sh \
  -c ../configs26/env1-runtime-rpa.properties \
  -p /opt/cp4ba-cmgr/cert-kubernetes/scripts
```

### Test configuration only (dry run)

```bash
cd cp4ba-installations/scripts

./cp4ba-one-shot-installation.sh \
  -c ../configs26/env1-runtime-rpa.properties \
  -t
```

### With trace enabled

```bash
cd cp4ba-installations/scripts

./cp4ba-one-shot-installation.sh \
  -c ../configs26/env1-runtime-rpa.properties \
  -p /opt/cp4ba-cmgr/cert-kubernetes/scripts \
  -x
```

---

## Architecture Diagram

```
Namespace: cp4ba-rpa
┌─────────────────────────────────────────────────────────┐
│                                                         │
│  ICP4ACluster CR (icp4adeploy)                          │
│  ├── Foundation pattern only                            │
│  │   ├── CPFS (Zen + IAM + License)                    │
│  │   └── Resource Registry                              │
│  └── PostgreSQL StatefulSet (Foundation DBs)            │
│                                                         │
│  IBM MQ Operator (channel: v3.9)                        │
│  └── MQ instance (for RPA messaging)                    │
│                                                         │
│  IBM RPA Operator (channel: v3.3)                       │
│  └── RpaServer CR                                       │
│      ├── Instance: rpa                                  │
│      └── Tenant: ibm (owner: cp4admin)                  │
│                                                         │
│  Microsoft SQL Server pod (mssql:2025-latest)           │
│  ├── DB: address                                        │
│  ├── DB: automation                                     │
│  ├── DB: knowledge                                      │
│  ├── DB: wordnet                                        │
│  └── DB: audit                                          │
│                                                         │
│  OpenLDAP pod                                           │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

---

## Post-Installation Access

| Service | URL Pattern |
|---|---|
| CP4BA Console (Zen) | `https://cpd-cp4ba-rpa.apps.<cluster-domain>` |
| IBM RPA Control Center | Accessible from Zen / RPA operator route |

---

## Reference

- [IBM RPA v30.0.x documentation](https://www.ibm.com/docs/en/rpa/30.0.x)
- [CP4BA v26.0.0 documentation](https://www.ibm.com/docs/en/cloud-paks/cp-biz-automation/26.0.0)
- [CPFS Foundational Services v4.x](https://www.ibm.com/docs/en/cloud-paks/foundational-services/4.x_cd)
