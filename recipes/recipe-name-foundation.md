# Recipe: Foundation

## Overview

| Attribute | Value |
|-----------|-------|
| **Template file** | [`cp4ba-cr-ref-foundation.yaml`](../cp4ba-installations/templates26/cp4ba-cr-ref-foundation.yaml) |
| **Primary config** | [`env1-runtime-opensearch-foundation.properties`](../cp4ba-installations/configs26/env1-runtime-opensearch-foundation.properties) |
| **Alternate config** | [`env1-runtime-os-bai-pfs.properties`](../cp4ba-installations/configs26/env1-runtime-os-bai-pfs.properties) |
| **CP4BA Version** | 26.0.0 |
| **Deployment type** | Production |
| **Platform** | OCP |
| **Profile size** | small |

## Purpose

This recipe deploys the **Foundation layer** of CP4BA, which provides all the shared infrastructure services (IBM Cloud Pak Foundational Services, Zen UI, IAM, OpenSearch, and optionally Kafka, BAI, and PFS) without deploying any workflow runtime.

It is typically used in two scenarios:

1. **Pure OpenSearch Foundation** (`env1-runtime-opensearch-foundation.properties`): Provides a minimal foundation with just OpenSearch enabled — useful as shared analytics infrastructure.
2. **OpenSearch + BAI + PFS Foundation** (`env1-runtime-os-bai-pfs.properties`): Foundation with Business Automation Insights, Kafka, and Process Federation Server — useful as a shared monitoring/federation platform that WfPS or BAW runtime deployments can connect to.

## CP4BA Capabilities

### Patterns

```
foundation
```

### Optional Components

**Variant 1 — Pure OpenSearch:**
```
opensearch
```

**Variant 2 — OpenSearch + BAI + PFS:**
```
opensearch,kafka,bai,pfs
```

## Capability Configuration Details

### Foundation (CPFS + Zen + IAM)

Always enabled. Provides:
- IBM Cloud Pak Foundational Services (CPFS) — license metering, common UI, identity services
- Zen platform UI — the central CP4BA dashboard
- IAM (Identity and Access Management) — integrated with LDAP
- Resource Registry — service discovery
- BTS (Business Teams Service) — team management

Configured via `shared_configuration`:
- `sc_deployment_license`: `production`
- `sc_iam.default_admin_username`: IAM platform administrator
- `sc_content_initialization`: `true` — runs initialisation jobs
- LDAP integration with custom OpenLDAP type

**Resource Registry replica size** is controlled by `CP4BA_INST_REGISTRY_REPLICA_SIZE` (default: `1`).

### OpenSearch

Enabled via optional component `opensearch`. Provides the full-text search and analytics backend used by BAI and other components.

Configured within `shared_configuration.opensearch_configuration` in the `baw-bai-ae` template variant (in the foundation template the opensearch is configured automatically when the `opensearch` optional component is declared).

### BAI (Business Automation Insights) — Variant 2 only

Enabled via optional component `bai`. Provides real-time event processing and dashboards.

Key `bai_configuration` settings:
```yaml
bai_configuration:
  business_performance_center:
    install: false             # Workforce insights dashboards
  bpmn:
    install: false             # BPMN process analytics
  bawadv:
    install: false
  content:
    install: false
  event_forwarder:
    install: false
  flink:
    additional_task_managers: 1
    create_route: true
```

> **Note**: In the foundation-only recipe the BAI event sources (bpmn, content, icm, etc.) are not applicable since no workflow or content runtime is deployed. The BAI infrastructure is provisioned and ready to receive events from external systems.

### Kafka — Variant 2 only

Enabled via optional component `kafka`. Provides the event streaming bus required by BAI.

### PFS (Process Federation Server) — Variant 2 only

Enabled via optional component `pfs`. Process Federation Server is deployed as a standalone service alongside the ICP4ACluster CR using the `cp4ba-process-federation-server` companion tooling.

PFS configuration variables:
```bash
CP4BA_INST_PFS=true
CP4BA_INST_PFS_NAME="pfs-demo"
CP4BA_INST_PFS_REPLICAS=2
CP4BA_INST_PFS_RES_REQS_CPU="1000m"
CP4BA_INST_PFS_RES_REQS_MEMORY="1024Mi"
CP4BA_INST_PFS_RES_LIMITS_REQS_CPU="2000m"
CP4BA_INST_PFS_RES_LIMITS_REQS_MEMORY="4Gi"
```

## Database Configuration

This recipe uses a minimal set of databases (foundation services only):

| Component | Purpose |
|-----------|---------|
| BTS DB | Business Teams Service |
| IM DB | Identity Management |
| Zen DB | Zen platform services |
| Model Gateway DB | AI Model Gateway |

SQL template used: `db-statements-ref-wfps.sql`

```bash
CP4BA_INST_DB_1_TEMPLATE="../templates-sql/db-statements-ref-wfps.sql"
```

## Infrastructure

| Component | Configuration |
|-----------|--------------|
| PostgreSQL | 1 instance, SSL-enabled, in-namespace |
| OpenLDAP | 1 instance, in-namespace |
| Storage (File) | `ocs-external-storagecluster-cephfs` |
| Storage (Block) | `ocs-external-storagecluster-ceph-rbd` |
| DB Storage | 10 Gi |

## LDAP / IAM

- LDAP type: Custom (OpenLDAP-compatible)
- User filter: `(&(cn=%v)(objectclass=person))`
- Group filter: `(&(cn=%v)(objectclass=groupOfNames))`
- IAM admin user: `cpadmin`
- Pak admin user: `cp4admin`

## Installation Command

### Variant 1 — Pure OpenSearch Foundation

```bash
_PTC=/home/$USER/cp4ba-projects/cp4ba-installations/configs26
_VV=26.0.0
_KK=26.0.0
CONFIG_FILE=${_PTC}/env1-runtime-opensearch-foundation.properties
./cp4ba-one-shot-installation.sh -c ${CONFIG_FILE} -m -v ${_VV} -k ${_KK}
```

### Variant 2 — OpenSearch + BAI + PFS

```bash
_PTC=/home/$USER/cp4ba-projects/cp4ba-installations/configs26
_VV=26.0.0
_KK=26.0.0
CONFIG_FILE=${_PTC}/env1-runtime-os-bai-pfs.properties
./cp4ba-one-shot-installation.sh -c ${CONFIG_FILE} -m -v ${_VV} -k ${_KK}
```

### With existing Case Package Manager

```bash
./cp4ba-one-shot-installation.sh \
  -c ${CONFIG_FILE} \
  -p /your-folder/ibm-cp-automation-26.0.0/.../cert-kubernetes/scripts
```

### Skip operator installation (operators already present)

```bash
./cp4ba-one-shot-installation.sh -c ${CONFIG_FILE} -m -v ${_VV} -k ${_KK} -o
```

## References

- [CP4BA 26.0.0 Foundation](https://www.ibm.com/docs/en/cloud-paks/cp-biz-automation/26.0.0)
- [CPFS 4.x Documentation](https://www.ibm.com/docs/en/cloud-paks/foundational-services/4.x_cd)
- [PFS Production Deployment](https://www.ibm.com/docs/en/cloud-paks/cp-biz-automation/26.0.0?topic=deployments-installing-cp4ba-process-federation-server-production-deployment)
