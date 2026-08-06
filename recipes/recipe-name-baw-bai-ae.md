# Recipe: BAW Runtime + BAI + Application Engine + CMIS

## Overview

| Attribute | Value |
|-----------|-------|
| **Template file** | [`cp4ba-cr-ref-baw-bai-ae.yaml`](../cp4ba-installations/templates26/cp4ba-cr-ref-baw-bai-ae.yaml) |
| **Primary config** | --THIS SECTION MUST BE UPDATED-- *(no dedicated v26 config file was found; closest is `env1-runtime-baw-bai.properties` but without AE specifics)* |
| **CP4BA Version** | 26.0.0 |
| **Deployment type** | Production (Runtime mode) |
| **Platform** | OCP |
| **Profile size** | small |

## Purpose

This recipe is the **most complete BAW Runtime** topology. It extends [recipe-name-baw-bai.md](recipe-name-baw-bai.md) by adding:

- **Application Engine (AE) with data persistence** (`application_engine_configuration`): A persistent Application Engine instance backed by an object store, enabling deployed Business Automation Applications to maintain state.
- **CMIS (Content Management Interoperability Services)**: A standards-based content access protocol gateway to CPE, useful for legacy or third-party integrations.
- **Application pattern**: Activates the `application` CP4BA pattern alongside `workflow`.
- **OpenSearch embedded configuration** (`sc_deployment_ocp_platform` + `opensearch_configuration.route_host`).

Use this recipe when deploying a **production BAW Runtime** that also needs to:
- Host and run Business Automation Applications (from BAS)
- Expose content via CMIS protocol
- Provide persistent application storage

## CP4BA Capabilities

### Patterns

```
foundation,workflow,application
```

> Note: The template header declares `patterns: foundation,workflow` but the structure includes `application_engine_configuration` making it effectively a `workflow,application` combined topology. The `application` pattern must be present in `CP4BA_INST_DEPL_PATTERNS` to activate AE.

### Optional Components

```
bai,kafka,opensearch,workflow_assistant,workplace_assistant,ae_data_persistence
```

> `cmis` should be added to optional components when CMIS gateway is needed. The template includes `ecm_configuration.cmis` block.

## Capability Configuration Details

### Foundation (CPFS + Zen + IAM)

Always enabled. See [knowledge base §4](../knowledge-bases/knowledge.md#4-shared-configuration-concepts).

Additional in this template vs. the base runtime recipe:
```yaml
shared_configuration:
  sc_deployment_ocp_platform:         # OCP-specific platform detail
  opensearch_configuration:
    route_host:                       # Custom OpenSearch route hostname (optional)
```

### BAW Runtime

Same as [recipe-name-baw-bai.md §BAW Runtime](recipe-name-baw-bai.md#baw-runtime), with the addition of:
- `federation_config`: PFS federation configuration within each BAW instance
- `appengine`: Per-BAW-instance Application Engine reference
- Extended probe configuration (`probe` block) for startup/liveness/readiness

Key additions to `baw_configuration[0]`:
```yaml
- name: "baw1"
  ...
  federation_config:
    # PFS federation settings
  appengine:
    # AE connection settings
  probe:
    startupProbe:
      ...
    livenessProbe:
      ...
    readinessProbe:
      ...
```

### ECM / Content Platform Engine (CPE) + ICN + CMIS

This template adds `cmis` to `ecm_configuration`:

```yaml
ecm_configuration:
  fncm_secret_name: ibm-fncm-secret
  cpe:
    # replica, resources, object stores...
  cmis:
    # CMIS gateway configuration
  graphql:
    # GraphQL API settings
```

Also, `navigator_configuration` includes extended production settings:
```yaml
navigator_configuration:
  ban_secret_name: ibm-ban-secret
  icn_production_setting:
    # Production-specific ICN settings
  monitor_enabled: false
  logging_enabled: false
  probe:
    # Startup/liveness/readiness probes
```

### BAI (Business Automation Insights)

Same as [recipe-name-baw-bai.md §BAI](recipe-name-baw-bai.md#bai-business-automation-insights).

### BAML

Same as [recipe-name-baw-bai.md §BAML](recipe-name-baw-bai.md#baml).

### Application Engine (AE) with Data Persistence

Adds a dedicated `application_engine_configuration` section:

```yaml
application_engine_configuration:
  - name: workspace
    admin_secret_name: icp4adeploy-workspace-aae-app-engine-admin-secret
    admin_user: "cp4admin"
    database:
      # PostgreSQL connection for AE database
    env:
      # AE environment configuration
    session:
      # Session persistence settings
    data_persistence:
      # Object store reference for application data
```

Uses `AEOS` (Application Engine Object Store) for persistent data storage.

### AI Assistants

Same as [recipe-name-baw-bai.md §AI Assistants](recipe-name-baw-bai.md#ai-assistants).

## Key Differences from recipe-name-baw-bai

| Feature | BAW+BAI (Runtime) | BAW+BAI+AE (Runtime) |
|---------|-------------------|---------------------|
| Patterns | `foundation,workflow` | `foundation,workflow,application` |
| CMIS | ✗ | ✓ |
| Application Engine | ✗ | ✓ (persistent) |
| `application_engine_configuration` | ✗ | ✓ |
| AE Object Store | ✗ | ✓ (`AEOS`) |
| OpenSearch route config | ✗ | ✓ |
| Extended probes | basic | extended |

## Database Configuration

Uses BAW Runtime SQL template: `db-statements-ref-baw.sql`

All databases from the base runtime recipe plus:

| Database | Variable | Purpose |
|----------|----------|---------|
| `*_aeos` | `CP4BA_INST_AEOS_DB_NAME` | Application Engine object store |
| `*_aaedb` | `CP4BA_INST_AE_DB_NAME` | Application Engine database |

CPE object stores include BAWINS1AEOS (Application Engine OS) and BAWINS1APP (Application OS) in addition to the standard set.

## Infrastructure

| Component | Configuration |
|-----------|--------------|
| PostgreSQL | 1 instance, in-namespace |
| OpenLDAP | 1 instance, in-namespace |
| Storage (File) | `ocs-external-storagecluster-cephfs` |
| Storage (Block) | `ocs-external-storagecluster-ceph-rbd` |
| DB Storage | 10 Gi |
| BAW File Store | 20 Gi (dynamic) |

## Installation Command

--THIS SECTION MUST BE UPDATED-- *(No dedicated `env1-runtime-baw-bai-ae.properties` config file exists in configs26. The template exists and defines the full topology. A config file must be created by copying `env1-runtime-baw-bai.properties` and adding the AE variables.)*

As a reference, the closest applicable command would be:

```bash
_VV=26.0.0
_KK=26.0.0
_PTC=/home/$USER/cp4ba-projects/cp4ba-installations/configs26
# CONFIG_FILE must be created with CP4BA_INST_CR_TEMPLATE pointing to cp4ba-cr-ref-baw-bai-ae.yaml
# and AE persistence variables enabled
CONFIG_FILE=${_PTC}/env1-runtime-baw-bai-ae.properties
./cp4ba-one-shot-installation.sh -c ${CONFIG_FILE} -m -v ${_VV} -k ${_KK}
```

Key additional variables needed in the config:
```bash
# AE data persistence
CP4BA_INST_AE_PERSISTENCE_ENABLE="true"
CP4BA_INST_AE_OS_NAME="AEOS"
CP4BA_INST_AE_SERVER_ENV_TYPE="production"
CP4BA_INST_AE_NAME="workspace"
CP4BA_INST_AE_SECRET_NAME="icp4adeploy-workspace-aae-app-engine-admin-secret"
CP4BA_INST_AE_DB_TYPE="postgresql"

# Patterns must include application
CP4BA_INST_DEPL_PATTERNS="foundation,workflow,application"
CP4BA_INST_OPT_COMPONENTS="bai,kafka,opensearch,workflow_assistant,workplace_assistant,ae_data_persistence"

# Template reference
CP4BA_INST_CR_TEMPLATE="templates26/cp4ba-cr-ref-baw-bai-ae.yaml"
```

## References

- [BAW Runtime Parameters](https://www.ibm.com/docs/en/cloud-paks/cp-biz-automation/26.0.0?topic=parameters-business-automation-workflow-runtime-workstream-services)
- [Application Engine Parameters](https://www.ibm.com/docs/en/cloud-paks/cp-biz-automation/26.0.0?topic=parameters-application-engine)
- [BAI Event Processing Parameters](https://www.ibm.com/docs/en/cloud-paks/cp-biz-automation/26.0.0?topic=baip-event-processing-parameters)
- [Content Cortex (FNCM) Production Deployment](https://www.ibm.com/docs/en/cloud-paks/cp-biz-automation/26.0.0?topic=deployments-installing-cp4ba-content-cortex-production-deployment)
- [CP4BA 26.0.0 Documentation](https://www.ibm.com/docs/en/cloud-paks/cp-biz-automation/26.0.0)
