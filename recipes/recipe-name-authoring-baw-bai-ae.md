# Recipe: BAW Authoring + BAI + Application Engine

## Overview

| Attribute | Value |
|-----------|-------|
| **Template file** | [`cp4ba-cr-ref-authoring-baw-bai-ae.yaml`](../cp4ba-installations/templates26/cp4ba-cr-ref-authoring-baw-bai-ae.yaml) |
| **Primary config** | [`env1-authoring-baw-bai-ae.properties`](../cp4ba-installations/configs26/env1-authoring-baw-bai-ae.properties) |
| **CP4BA Version** | 26.0.0 |
| **Deployment type** | Production (Authoring mode) |
| **Platform** | OCP |
| **Profile size** | small |

## Purpose

This recipe extends the [BAW Authoring + BAI](recipe-name-authoring-baw-bai.md) recipe by adding:

- **Application Engine (AE) with data persistence**: A dedicated Application Engine instance with its own database-backed object store, enabling persistent application state for Business Automation Applications deployed from BAS.
- **Application pattern**: Activates the `application` CP4BA pattern alongside `workflow`.
- **`app_designer`**: Application Designer component for building low-code applications.

Use this recipe when you need the full BAW Authoring + BAI stack **and** want to deploy and test Business Automation Applications with full persistence (not just ephemeral playback).

## CP4BA Capabilities

### Patterns

```
foundation,workflow,application
```

### Optional Components

```
baw_authoring,bas,app_designer,bai,pfs,kafka,opensearch,workflow_assistant,workplace_assistant
```

## Capability Configuration Details

### Foundation (CPFS + Zen + IAM)

Always enabled. See [knowledge base §4](../knowledge-bases/knowledge.md#4-shared-configuration-concepts).

### BAW Authoring

Full BAW Authoring with BPM + Case + Content + AE integration. The template includes `appengine` embedded configuration within `workflow_authoring_configuration.appengine`.

### BAS (Business Automation Studio)

Full BAS setup with Playback Server. See [recipe-name-authoring-baw.md §BAS](recipe-name-authoring-baw.md#bas-business-automation-studio).

### ECM / Content Platform Engine (CPE) + ICN

Identical setup as [recipe-name-authoring-baw.md §ECM](recipe-name-authoring-baw.md#ecm--content-platform-engine-cpe).

### BAI (Business Automation Insights)

Same as [recipe-name-authoring-baw-bai.md §BAI](recipe-name-authoring-baw-bai.md#bai-business-automation-insights).

### BAML

Present for task prioritization. See [recipe-name-authoring-baw-bai.md §BAML](recipe-name-authoring-baw-bai.md#baml-business-automation-machine-learning).

### Application Engine (AE) with Data Persistence

This recipe enables `ae_data_persistence` implicitly via `app_designer` optional component and adds a dedicated `application_engine_configuration` section to the CR.

```yaml
application_engine_configuration:
  - name: workspace
    admin_secret_name: icp4adeploy-workspace-aae-app-engine-admin-secret
    admin_user: "cp4admin"
    database:
      # PostgreSQL connection for AE persistence
    env:
      # Environment-specific settings
    session:
      # Session configuration
    data_persistence:
      # Persistence settings referencing AEOS object store
```

Key configuration variables:
```bash
CP4BA_INST_AE_PERSISTENCE_ENABLE="true"           # vs. "false" in base recipe
CP4BA_INST_AE_OS_NAME="${AEOS_OBJSTORE_NAME}"      # references BAWINS1AEOS
CP4BA_INST_AE_SERVER_ENV_TYPE="development"
CP4BA_INST_AE_NAME="workspace"
CP4BA_INST_AE_SECRET_NAME="icp4adeploy-workspace-aae-app-engine-admin-secret"
CP4BA_INST_AE_DB_TYPE="postgresql"
```

The Application Engine uses the `AEOS` object store backed by the `*_aeos` database.

### PFS (Process Federation Server)

Same as [recipe-name-authoring-baw-bai.md §PFS](recipe-name-authoring-baw-bai.md#pfs-process-federation-server).

## Key Differences from recipe-name-authoring-baw-bai

| Feature | BAW+BAI | BAW+BAI+AE |
|---------|---------|-----------|
| Patterns | `foundation,workflow` | `foundation,workflow,application` |
| `app_designer` | ✗ | ✓ |
| Application Engine | ephemeral | persistent (`ae_data_persistence=true`) |
| AE Object Store | ✗ | ✓ (`AEOS`) |
| AE Database | ✗ | ✓ (`*_aeos`) |
| `application_engine_configuration` | ✗ | ✓ |

## Database Configuration

Uses BAW Authoring SQL template: `db-statements-ref-baw-authoring.sql`

All databases from the base recipe plus:

| Database | Variable | Purpose |
|----------|----------|---------|
| `*_aeos` | `CP4BA_INST_AEOS_DB_NAME` | Application Engine object store |
| `*_aaedb` | `CP4BA_INST_AE_DB_NAME` | Application Engine database |

CPE object stores include BAWINS1AEOS (Application Engine Object Store) and BAWINS1APP (Application Object Store).

BAI event emitter:
```bash
CP4BA_INST_BAI_OBJECTSTORE_CONTENT_EVENT_ENABLED="true"
```

## Infrastructure

| Component | Configuration |
|-----------|--------------|
| PostgreSQL | 1 instance, SSL-enabled, in-namespace |
| OpenLDAP | 1 instance, in-namespace |
| Storage (File) | `ocs-external-storagecluster-cephfs` |
| Storage (Block) | `ocs-external-storagecluster-ceph-rbd` |
| DB Storage | 10 Gi |
| BAW File Store | 20 Gi (dynamic) |

Network policy: uses `allow-all` template by default:
```bash
CP4BA_INST_NP_TEMPLATE_1="../templates-networkpolicies/mutually-exclusive/my-network-policy-sample-allow-all.yaml"
```

## Installation Command

```bash
_PTC=/home/$USER/cp4ba-projects/cp4ba-installations/configs26
_VV=26.0.0
_KK=26.0.0
CONFIG_FILE=${_PTC}/env1-authoring-baw-bai-ae.properties
./cp4ba-one-shot-installation.sh -c ${CONFIG_FILE} -m -v ${_VV} -k ${_KK}
```

### With GenAI

```bash
export CP4BA_INST_GENAI_ENABLED="true"
export CP4BA_INST_GENAI_WX_APIKEY="<your-ibm-cloud-api-key>"
export CP4BA_INST_GENAI_WX_PRJ_ID="<your-watsonx-project-id>"

./cp4ba-one-shot-installation.sh -c ${CONFIG_FILE} -m -v ${_VV} -k ${_KK}
```

## References

- [BAW Authoring Parameters](https://www.ibm.com/docs/en/cloud-paks/cp-biz-automation/26.0.0?topic=parameters-business-automation-workflow-authoring)
- [Application Engine Parameters](https://www.ibm.com/docs/en/cloud-paks/cp-biz-automation/26.0.0?topic=foundation-application-engine)
- [BAI Event Processing Parameters](https://www.ibm.com/docs/en/cloud-paks/cp-biz-automation/26.0.0?topic=baip-event-processing-parameters)
- [CP4BA 26.0.0 Documentation](https://www.ibm.com/docs/en/cloud-paks/cp-biz-automation/26.0.0)
- [IBM BAW Documentation](https://www.ibm.com/docs/en/baw/26.0.x)
