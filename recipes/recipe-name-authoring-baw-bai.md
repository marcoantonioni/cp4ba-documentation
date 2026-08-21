# Recipe: BAW Authoring + BAI

## Overview

| Attribute | Value |
|-----------|-------|
| **Template file** | [`cp4ba-cr-ref-authoring-baw-bai.yaml`](../cp4ba-installations/templates26/cp4ba-cr-ref-authoring-baw-bai.yaml) |
| **Primary config** | [`env1-authoring-baw-bai.properties`](../cp4ba-installations/configs26/env1-authoring-baw-bai.properties) |
| **Alternate config (GenAI + PFS)** | [`env1-authoring-baw-pfs-genai.properties`](../cp4ba-installations/configs26/env1-authoring-baw-pfs-genai.properties) |
| **CP4BA Version** | 26.0.0 |
| **Deployment type** | Production (Authoring mode) |
| **Platform** | OCP |
| **Profile size** | small |

## Purpose

This recipe deploys a **complete BAW Authoring environment** that extends the basic authoring recipe with:
- **Business Automation Insights (BAI)**: Real-time event processing, Flink-based analytics, and OpenSearch dashboards
- **Kafka**: Event streaming infrastructure for BAI
- **OpenSearch**: Analytics data store
- **PFS (Process Federation Server)**: For federated task management (enabled by default)
- **AI Assistants** (optional via GenAI variant)

Two config variants are provided:

- **`env1-authoring-baw-bai`**: Full BAW Authoring + BAI + PFS stack, GenAI disabled by default.
- **`env1-authoring-baw-pfs-genai`**: Same capabilities plus GenAI/watsonx.ai integration activated.

## CP4BA Capabilities

### Patterns

```
foundation,workflow
```

### Optional Components

```
baw_authoring,bas,bai,pfs,kafka,opensearch,workflow_assistant,workplace_assistant
```

> `workflow_assistant` and `workplace_assistant` are enabled only when `CP4BA_INST_GENAI_ENABLED=true`.

## Capability Configuration Details

### Foundation (CPFS + Zen + IAM)

Always enabled. See [knowledge base §4](../knowledge-bases/knowledge.md#4-shared-configuration-concepts).

### BAW Authoring

Full BAW Authoring with BPM + Case + Content integration. Configured via `workflow_authoring_configuration`.

Key settings include:
- Content integration: CPE for document storage, Case object stores (DOS, DOCS, TOS, CONTENT)
- Federation config: PFS integration for federated task view
- Application Engine embedded inside authoring
- Custom XML secrets for Liberty and Lombardi customization
- GenAI CSP headers configured for watsonx domains

```yaml
workflow_authoring_configuration:
  content_integration:
    # CPE integration settings
  case:
    # Case object store settings
  appengine:
    # Embedded Application Engine
  storage:
    use_dynamic_provisioning: true
    size_for_filestore: "20Gi"
  federation_config:
    # PFS federation settings
  business_event:
    # BAI business event emitter settings
```

### BAS (Business Automation Studio)

Identical to the base authoring recipe. See [recipe-name-authoring-baw.md](recipe-name-authoring-baw.md).

### ECM / Content Platform Engine (CPE) + ICN

Enabled as part of the workflow pattern. Identical object store topology as the base authoring recipe.

### BAI (Business Automation Insights)

Enabled via optional component `bai`. Configured via `bai_configuration`:

```yaml
bai_configuration:
  business_performance_center:
    install: true                           # Workforce Insights dashboards
    workforce_insights_secret: custom-bpc-workforce-secret

  bpmn:
    install: true                           # BPMN process analytics
    force_opensearch_timeseries: true
    end_aggregation_delay: 10000

  bawadv:
    install: false                          # BAW Advanced analytics
  content:
    install: true                           # Content events analytics
  event_forwarder:
    install: false
  flink:
    additional_task_managers: 1
    create_route: true
  icm:
    install: true                           # Case management analytics
  navigator:
    install: true                           # Navigator analytics
  ads:
    install: false
  odm:
    install: false
```

### BAML (Business Automation Machine Learning)

```yaml
baml_configuration:
  workforce_insights:
    replicas: 1
    resources:
      limits:
        cpu: '1'
        memory: 1024Mi
      requests:
        cpu: '1'
        memory: 1024Mi
  intelligent_task_prioritization:
    retrain_model_schedule: "*/30 * * * *"
```

### PFS (Process Federation Server)

Process Federation Server is deployed via the `cp4ba-process-federation-server` companion tooling using the following configuration:

```bash
CP4BA_INST_PFS=true
CP4BA_INST_PFS_NAME="pfs-demo"
CP4BA_INST_PFS_NAMESPACE="${CP4BA_INST_NAMESPACE}"
CP4BA_INST_PFS_REPLICAS=2
CP4BA_INST_PFS_RES_REQS_CPU="1000m"
CP4BA_INST_PFS_RES_REQS_MEMORY="1024Mi"
CP4BA_INST_PFS_RES_LIMITS_REQS_CPU="2000m"
CP4BA_INST_PFS_RES_LIMITS_REQS_MEMORY="4Gi"
```

### AI Assistants (GenAI + PFS variant only)

See [recipe-name-authoring-baw.md §AI Assistants](recipe-name-authoring-baw.md) for details.

## Database Configuration

Uses BAW Authoring SQL template: `db-statements-ref-baw-authoring.sql`

Full database set (same as base authoring recipe plus AE persistence databases):

| Database | Variable | Purpose |
|----------|----------|---------|
| `*_baw_1` | `CP4BA_INST_BAS_1_DB_BAW_NAME` | BAW authoring / BAS |
| `*_appdb` | `CP4BA_INST_APP_DB_NAME` | Playback Server |
| `*_aaedb` | `CP4BA_INST_AE_DB_NAME` | Application Engine |
| `*_aeos` | `CP4BA_INST_AEOS_DB_NAME` | Application Engine object store |
| `*_gcd` | `CP4BA_INST_GCD_DB_NAME` | FileNet GCD |
| `*_icn` | `CP4BA_INST_ICN_DB_NAME` | Content Navigator |
| `*_bawdocs` | `CP4BA_INST_DOCS_DB_NAME` | BAW DOCS object store |
| `*_bawdos` | `CP4BA_INST_DOS_DB_NAME` | BAW DOS object store |
| `*_bawtos` | `CP4BA_INST_TOS_DB_NAME` | BAW TOS / Case object store |
| `*_content` | `CP4BA_INST_CONTENT_DB_NAME` | Content object store |
| `*_chos` | `CP4BA_INST_CHOS_DB_NAME` | Case history object store |
| `*_os1` | `CP4BA_INST_OS1_DB_NAME` | Additional object store |

BAI event emitter configuration:
```bash
CP4BA_INST_BAI_EVENT_EMITTER_UNIQUE_ID="EEID1"
CP4BA_INST_BAI_EVENT_EMITTER_DATE_SQL="20240301T000000Z"
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

### Standard BAW Authoring + BAI

```bash
_PTC=/home/$USER/cp4ba-projects/cp4ba-installations/configs26
_VV=26.0.0
_KK=26.0.0
CONFIG_FILE=${_PTC}/env1-authoring-baw-bai.properties
./cp4ba-one-shot-installation.sh -c ${CONFIG_FILE} -m -v ${_VV} -k ${_KK}
```

### GenAI + PFS Variant

```bash
export CP4BA_INST_GENAI_ENABLED="true"
export CP4BA_INST_GENAI_WX_APIKEY="<your-ibm-cloud-api-key>"
export CP4BA_INST_GENAI_WX_PRJ_ID="<your-watsonx-project-id>"

_PTC=/home/$USER/cp4ba-projects/cp4ba-installations/configs26
_VV=26.0.0
_KK=26.0.0
CONFIG_FILE=${_PTC}/env1-authoring-baw-pfs-genai.properties
./cp4ba-one-shot-installation.sh -c ${CONFIG_FILE} -m -v ${_VV} -k ${_KK}
```

## References

- [BAW Authoring Parameters](https://www.ibm.com/docs/en/cloud-paks/cp-biz-automation/26.0.0?topic=parameters-business-automation-workflow-authoring)
- [BAI Event Processing Parameters](https://www.ibm.com/docs/en/cloud-paks/cp-biz-automation/26.0.0?topic=baip-event-processing-parameters)
- [CP4BA 26.0.0 Documentation](https://www.ibm.com/docs/en/cloud-paks/cp-biz-automation/26.0.0)
- [IBM BAW Documentation](https://www.ibm.com/docs/en/baw/26.0.x)
- [PFS Production Deployment](https://www.ibm.com/docs/en/cloud-paks/cp-biz-automation/26.0.0?topic=deployments-installing-cp4ba-process-federation-server-production-deployment)
