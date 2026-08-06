# Recipe: WfPS Authoring + BAI + PFS

## Overview

| Attribute | Value |
|-----------|-------|
| **Template file** | [`cp4ba-cr-ref-authoring-wfps-bai.yaml`](../cp4ba-installations/templates26/cp4ba-cr-ref-authoring-wfps-bai.yaml) |
| **Primary config** | [`env1-authoring-wfps-pfs-bai.properties`](../cp4ba-installations/configs26/env1-authoring-wfps-pfs-bai.properties) |
| **CP4BA Version** | 26.0.0 |
| **Deployment type** | Production (Authoring mode) |
| **Platform** | OCP |
| **Profile size** | small |

## Purpose

This recipe deploys a **Workflow Process Service (WfPS) Authoring** environment — the lightweight BAW alternative when you need to author and execute workflow processes **without** Case Management (ICM) or FileNet Content Manager (ECM).

The recipe combines:
- **WfPS Authoring** (`wfps_authoring`): Authoring mode of Workflow Process Service, which internally uses a BAW Authoring server (`bastudio_configuration` + `workflow_authoring_configuration`) without ECM/Case object stores.
- **Business Automation Insights (BAI)**: Real-time event processing and analytics.
- **Kafka**: Event streaming for BAI.
- **OpenSearch**: Analytics data store.
- **Process Federation Server (PFS)**: Federated task list view deployed via companion tooling.

This topology is ideal for:
- Teams using WfPS (lightweight processes, no case management)
- Environments where ECM is not required
- Lower footprint than full BAW deployments
- Monitoring WfPS processes via BAI dashboards

## CP4BA Capabilities

### Patterns

```
foundation,workflow-process-service
```

### Optional Components

```
wfps_authoring,bai,kafka,opensearch
```

> `workflow_assistant` and `workplace_assistant` are controlled by `CP4BA_RUN_AUTHORING_AGENT` / `CP4BA_RUN_WORKPLACE_AGENT` which default to the value of `CP4BA_INST_GENAI_ENABLED`.

## Capability Configuration Details

### Foundation (CPFS + Zen + IAM)

Always enabled. See [knowledge base §4](../knowledge-bases/knowledge.md#4-shared-configuration-concepts).

### WfPS Authoring (`wfps_authoring`)

Activates WfPS Authoring mode. This uses:
- `bastudio_configuration` — BAS authoring studio connected to a BAW Authoring server
- `workflow_authoring_configuration` — BAW Authoring configuration (WfPS scope, no Case/ECM)

**Important distinctions from full BAW Authoring:**
- No Case Builder / Case configuration
- No FileNet CPE / ECM object stores
- No `initialize_configuration` / `datasource_configuration` for ECM
- Database footprint is significantly smaller (only BAW + CPFS shared databases)
- No `ecm_configuration` or `navigator_configuration` blocks (no ICN)

BAW Authoring / BAS configuration (same top-level keys as full BAW):
```yaml
bastudio_configuration:
  admin_user: "cp4admin"
  database:
    host: <DB_SERVER>
    name: <ENV>_baw_1
    port: "5432"
    type: postgresql
  playback_server:
    admin_user: "cp4admin"
    database:
      name: <ENV>_appdb
  resources:
    bastudio:
      limits:
        cpu: '5000m'
        memory: 3096Mi

workflow_authoring_configuration:
  storage:
    use_dynamic_provisioning: true
    size_for_filestore: "20Gi"
  environment_config:
    show_task_prioritization_service_toggle: true
    always_run_task_prioritization_service: false
  custom_xml_secret_name: my-liberty-custom-xml-secret
  lombardi_custom_xml_secret_name: my-lombardi-custom-xml-secret
```

### BAI (Business Automation Insights)

Same as the BAW+BAI recipe. Key configuration:

```yaml
bai_configuration:
  business_performance_center:
    install: true
    workforce_insights_secret: custom-bpc-workforce-secret
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

PFS is deployed via the `cp4ba-process-federation-server` companion repository, running alongside the `ICP4ACluster` CR:

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

### AI Assistants

AI assistants are controlled by:
```bash
CP4BA_RUN_AUTHORING_AGENT="${CP4BA_INST_GENAI_ENABLED}"
CP4BA_RUN_WORKPLACE_AGENT="${CP4BA_INST_GENAI_ENABLED}"
```

These map to `workflow_assistant_configuration.run_authoring_agent` and `workflow_assistant_configuration.run_workplace_agent` in the CR.

## Database Configuration

Uses WfPS Authoring SQL template: `db-statements-ref-wfps-authoring.sql`

Minimal database set (no ECM databases):

| Database | Variable | Purpose |
|----------|----------|---------|
| `*_baw_1` | `CP4BA_INST_BAS_1_DB_BAW_NAME` | WfPS Authoring / BAS |
| `*_appdb` | `CP4BA_INST_APP_DB_NAME` | Playback Server |
| BTS DB | `CP4BA_INST_DB_BTS_*` | Business Teams Service |
| IM DB | `CP4BA_INST_DB_IM_*` | Identity Management |
| Zen DB | `CP4BA_INST_DB_ZEN_*` | Zen platform |
| Model Gateway DB | `CP4BA_INST_DB_MODELGATEWAY_*` | AI Model Gateway |

Note: `CP4BA_INST_DB_BAW_USER` is not used; `CP4BA_INST_DB_WFPS_EXT_1="true"` indicates WfPS-specific DB schema.

BAI event emitter:
```bash
CP4BA_INST_BAI_EVENT_EMITTER_UNIQUE_ID="EEID1"
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
| BAW/WfPS File Store | 20 Gi (dynamic) |

## Installation Command

```bash
_VV=26.0.0
_KK=26.0.0
_PTC=/home/$USER/cp4ba-projects/cp4ba-installations/configs26
CONFIG_FILE=${_PTC}/env1-authoring-wfps-pfs-bai.properties
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

- [WfPS Runtime Production Deployment](https://www.ibm.com/docs/en/cloud-paks/cp-biz-automation/26.0.0?topic=icpd-installing-cp4ba-workflow-process-service-runtime-production-deployment)
- [BAI Event Processing Parameters](https://www.ibm.com/docs/en/cloud-paks/cp-biz-automation/26.0.0?topic=baip-event-processing-parameters)
- [PFS Production Deployment](https://www.ibm.com/docs/en/cloud-paks/cp-biz-automation/26.0.0?topic=deployments-installing-cp4ba-process-federation-server-production-deployment)
- [CP4BA 26.0.0 Documentation](https://www.ibm.com/docs/en/cloud-paks/cp-biz-automation/26.0.0)
