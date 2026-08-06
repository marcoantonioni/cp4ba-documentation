# Recipe: BAW Runtime + BAI

## Overview

| Attribute | Value |
|-----------|-------|
| **Template file** | [`cp4ba-cr-ref-baw-bai.yaml`](../cp4ba-installations/templates26/cp4ba-cr-ref-baw-bai.yaml) |
| **Primary config** | [`env1-runtime-baw-bai.properties`](../cp4ba-installations/configs26/env1-runtime-baw-bai.properties) |
| **CP4BA Version** | 26.0.0 |
| **Deployment type** | Production (Runtime mode) |
| **Platform** | OCP |
| **Profile size** | small |

## Purpose

This recipe deploys a **BAW Runtime production environment** — the execution environment for workflows and case management applications developed in a separate BAW Authoring environment. It includes:

- **Business Automation Workflow (BAW) Runtime**: One BAW runtime server with full BPM + Case capabilities, connected to FileNet Content Manager.
- **Business Automation Insights (BAI)**: Real-time monitoring and analytics.
- **Kafka**: Event streaming for BAI.
- **OpenSearch**: Analytics data store.
- **AI Assistants**: Workplace assistant and workflow assistant (configurable via GenAI).
- **ECM / Content Platform Engine**: FileNet CPE + ICN for document storage.
- **GraphQL API**: REST/GraphQL gateway to CPE.

This recipe is the **runtime counterpart** to the authoring recipes. It does **not** include:
- Business Automation Studio (BAS)
- Process Designer / Case Builder (authoring tools)
- Playback Server

## CP4BA Capabilities

### Patterns

```
foundation,workflow
```

### Optional Components

```
bai,kafka,opensearch,workflow_assistant,workplace_assistant
```

> `workflow_assistant` and `workplace_assistant` are enabled by `CP4BA_RUN_*` variables, which default to `CP4BA_INST_GENAI_ENABLED`.

## Capability Configuration Details

### Foundation (CPFS + Zen + IAM)

Always enabled. See [knowledge base §4](../knowledge-bases/knowledge.md#4-shared-configuration-concepts).

### BAW Runtime

Configured via `baw_configuration` (an array supporting multiple BAW instances, one per entry):

```yaml
baw_configuration:
  - name: "baw1"
    capabilities:
      - workflow
      - workstreams
    host_federated_portal: false
    admin_user: "cp4admin"
    database:
      host: <DB_SERVER>
      name: <ENV>_baw_1
      port: "5432"
      type: postgresql
    env_type: "production"
    replicas: 1
    resources:
      # CPU and memory limits/requests
    jms:
      # JMS configuration
    storage:
      use_dynamic_provisioning: true
      size_for_filestore: "20Gi"
    tls:
      # TLS settings
    content_integration:
      # CPE integration
    case:
      # Case management object stores
    event_emitter:
      # BAI event emitter settings
    case_history_emitter:
      # Case history BAI events
    business_event:
      # Business event configuration
    environment_config:
      # GenAI CSP headers
    federation_config:
      # PFS federation (optional)
```

Key runtime variables:
```bash
CP4BA_INST_BAW_1=true
CP4BA_INST_BAW_1_NAME="baw1"
CP4BA_INST_BAW_1_REPLICAS="1"
CP4BA_INST_BAW_1_LIMITS_CPU="5000m"
CP4BA_INST_BAW_1_LIMITS_MEMORY="3096Mi"
CP4BA_INST_BAW_1_DB_SECRET="${CP4BA_INST_ENV}-baw1-server-db-secret"
```

### ECM / Content Platform Engine (CPE) + ICN

Configured via `ecm_configuration` and `navigator_configuration`. Same object store topology as the authoring recipes (BAWINS1DOS, BAWINS1DOCS, BAWINS1TOS, BAWINS1CONTENT, MYOS1, AEOS).

CPE resource settings:
```bash
CP4BA_INST_CPE_REPLICAS="1"
CP4BA_INST_CPE_RES_LIMITS_CPU="2"
CP4BA_INST_CPE_RES_LIMITS_MEM="6144Mi"
CP4BA_INST_CPE_RES_REQS_CPU="1"
CP4BA_INST_CPE_RES_REQS_MEM="3072Mi"
```

### BAI (Business Automation Insights)

```yaml
bai_configuration:
  business_performance_center:
    install: false           # Workforce Insights not enabled in runtime variant
  bpmn:
    install: true
    force_opensearch_timeseries: true
    end_aggregation_delay: 10000
  bawadv:
    install: false
  content:
    install: true
  event_forwarder:
    install: false
  flink:
    additional_task_managers: 1
    create_route: true
  icm:
    install: true
  navigator:
    install: true
  ads:
    install: false
  odm:
    install: false
```

### BAML

```yaml
baml_configuration:
  workforce_insights:
    replicas: 1
  intelligent_task_prioritization:
    retrain_model_schedule: "*/30 * * * *"
```

### AI Assistants

Controlled by:
```bash
CP4BA_RUN_AUTHORING_AGENT="${CP4BA_INST_GENAI_ENABLED}"
CP4BA_RUN_WORKPLACE_AGENT="${CP4BA_INST_GENAI_ENABLED}"
```

## Database Configuration

Uses BAW Runtime SQL template: `db-statements-ref-baw.sql`

| Database | Variable | Purpose |
|----------|----------|---------|
| `*_baw_1` | `CP4BA_INST_BAW_1_DB_NAME` | BAW runtime server 1 |
| `*_gcd` | `CP4BA_INST_GCD_DB_NAME` | FileNet GCD |
| `*_icn` | `CP4BA_INST_ICN_DB_NAME` | Content Navigator |
| `*_bawdocs` | `CP4BA_INST_DOCS_DB_NAME` | BAW DOCS object store |
| `*_bawdos` | `CP4BA_INST_DOS_DB_NAME` | BAW DOS object store |
| `*_bawtos` | `CP4BA_INST_TOS_DB_NAME` | BAW TOS / Case object store |
| `*_content` | `CP4BA_INST_CONTENT_DB_NAME` | Content object store |
| `*_chos` | `CP4BA_INST_CHOS_DB_NAME` | Case history object store |

Note: This recipe does **not** include Playback Server DB (`*_appdb`) or AE DB (`*_aaedb`) as there is no BAS.

BAI event emitter:
```bash
CP4BA_INST_BAI_EVENT_EMITTER_UNIQUE_ID="EEID1"
CP4BA_INST_BAI_EVENT_EMITTER_DATE_SQL="20240301T000000Z"
CP4BA_INST_BAI_OBJECTSTORE_CONTENT_EVENT_ENABLED="true"
```

## Infrastructure

| Component | Configuration |
|-----------|--------------|
| PostgreSQL | 1 instance, non-SSL (`CP4BA_INST_DB_ONLY_SSL=false`) as default, in-namespace |
| OpenLDAP | 1 instance, in-namespace |
| Storage (File) | `ocs-external-storagecluster-cephfs` |
| Storage (Block) | `ocs-external-storagecluster-ceph-rbd` |
| DB Storage | 10 Gi |
| BAW File Store | 20 Gi (dynamic) |

> **Note**: In `env1-runtime-baw-bai.properties`, `CP4BA_INST_DB_ONLY_SSL="false"` — the DB service used is still the SSL-named service endpoint (`-ssl-rw`), but the config allows non-SSL fallback. Adjust to `true` for production hardening.

Network policy: uses `allow-all` template by default:
```bash
CP4BA_INST_NP_TEMPLATE_1="../templates-networkpolicies/mutually-exclusive/my-network-policy-sample-allow-all.yaml"
```

## Installation Command

```bash
_VV=26.0.0
_KK=26.0.0
_PTC=/home/$USER/cp4ba-projects/cp4ba-installations/configs26
CONFIG_FILE=${_PTC}/env1-runtime-baw-bai.properties
./cp4ba-one-shot-installation.sh -c ${CONFIG_FILE} -m -v ${_VV} -k ${_KK}
```

### Skip operators (if already installed)

```bash
./cp4ba-one-shot-installation.sh -c ${CONFIG_FILE} -m -v ${_VV} -k ${_KK} -o
```

### With GenAI

```bash
export CP4BA_INST_GENAI_ENABLED="true"
export CP4BA_INST_GENAI_WX_APIKEY="<your-ibm-cloud-api-key>"
export CP4BA_INST_GENAI_WX_PRJ_ID="<your-watsonx-project-id>"

./cp4ba-one-shot-installation.sh -c ${CONFIG_FILE} -m -v ${_VV} -k ${_KK}
```

## References

- [BAW Runtime Parameters](https://www.ibm.com/docs/en/baw/26.0.x)
- [BAI Event Processing Parameters](https://www.ibm.com/docs/en/cloud-paks/cp-biz-automation/26.0.0?topic=baip-event-processing-parameters)
- [Content Cortex (FNCM) Production Deployment](https://www.ibm.com/docs/en/cloud-paks/cp-biz-automation/26.0.0?topic=deployments-installing-cp4ba-content-cortex-production-deployment)
- [CP4BA 26.0.0 Documentation](https://www.ibm.com/docs/en/cloud-paks/cp-biz-automation/26.0.0)
