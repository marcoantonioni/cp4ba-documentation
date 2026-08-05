# Recipe: BAW Authoring

## Overview

| Attribute | Value |
|-----------|-------|
| **Template file** | [`cp4ba-cr-ref-authoring-baw.yaml`](../cp4ba-installations/templates26/cp4ba-cr-ref-authoring-baw.yaml) |
| **Primary config** | [`env1-authoring-baw.properties`](../cp4ba-installations/configs26/env1-authoring-baw.properties) |
| **Alternate config (CICD)** | [`env1-authoring-baw-cicd.properties`](../cp4ba-installations/configs26/env1-authoring-baw-cicd.properties) |
| **Alternate config (GenAI)** | [`env1-authoring-baw-genai.properties`](../cp4ba-installations/configs26/env1-authoring-baw-genai.properties) |
| **CP4BA Version** | 26.0.0 |
| **Deployment type** | Production (Authoring mode) |
| **Platform** | OCP |
| **Profile size** | small |

## Purpose

This recipe deploys a **full BAW Authoring environment** — including Business Automation Workflow Authoring (BPM + Case), Business Automation Studio, FileNet Content Manager (CPE + ICN), and GraphQL API — **without** Business Automation Insights (BAI).

It is the baseline authoring recipe. Three config variants exist:

- **`env1-authoring-baw`**: Standard authoring, no BAI, no CICD, GenAI disabled by default but configurable via env vars.
- **`env1-authoring-baw-cicd`**: Identical to standard but with GitHub integration pre-configured in BAS (`CP4BA_INST_GIT_ENABLED="true"`, GitHub TLS secret created).
- **`env1-authoring-baw-genai`**: Same as standard, GenAI enabled at startup via environment variable override (requires watsonx.ai API key and project ID at deploy time).

## CP4BA Capabilities

### Patterns

```
foundation,workflow
```

### Optional Components

```
baw_authoring,bas,workflow_assistant,workplace_assistant
```

> `workflow_assistant` and `workplace_assistant` are enabled only when `CP4BA_INST_GENAI_ENABLED=true`.

## Capability Configuration Details

### Foundation (CPFS + Zen + IAM)

Always enabled — provides CPFS, Zen UI, IAM, and resource registry. See [knowledge base §4](../knowledge-bases/knowledge.md#4-shared-configuration-concepts).

### BAW Authoring (`baw_authoring`)

Enables the BAW Authoring server (formerly called BAW Authoring or BAW Studio). Activated via `workflow_authoring_configuration`.

Key settings:
```yaml
workflow_authoring_configuration:
  storage:
    use_dynamic_provisioning: true
    size_for_filestore: "20Gi"
  environment_config:
    show_task_prioritization_service_toggle: true
    always_run_task_prioritization_service: false
    # GenAI CSP headers
    content_security_policy_additional_connect_src:
      - '*.watson.appdomain.cloud'
  custom_xml_secret_name: my-liberty-custom-xml-secret
  lombardi_custom_xml_secret_name: my-lombardi-custom-xml-secret
```

Content integration (CPE, case) is configured within `workflow_authoring_configuration.content_integration` and `workflow_authoring_configuration.case`.

### BAS (Business Automation Studio)

Enabled via `bas` optional component. Activated via `bastudio_configuration`.

BAS provides the low-code authoring web UI for designing BAW processes, cases, and applications. It connects to both the BAW Authoring server and the Playback Server.

Key settings:
```yaml
bastudio_configuration:
  admin_user: "cp4admin"
  database:
    host: <DB_SERVER>
    name: <ENV>_baw_1
    port: "5432"
    type: postgresql
    cm_min_pool_size: 10
    cm_max_pool_size: 100
  playback_server:
    admin_user: "cp4admin"
    database:
      host: <DB_SERVER>
      name: <ENV>_appdb
  tls:
    tlsTrustList: []   # Git TLS cert added here when CICD variant
  resources:
    bastudio:
      limits:
        cpu: '5000m'
        memory: 3096Mi
```

### ECM / Content Platform Engine (CPE)

Enabled implicitly when the `workflow` pattern is used with BAW Authoring. Configured via `ecm_configuration`.

```yaml
ecm_configuration:
  fncm_secret_name: ibm-fncm-secret
  cpe:
    # replicas, resources, object store connections...
  graphql:
    # GraphQL API settings
```

CPE manages the following object stores for BAW:
- BAWINS1DOS (Design Object Store)
- BAWINS1DOCS (Document Object Store)
- BAWINS1TOS (Target/Case Object Store)
- BAWINS1CONTENT (Content Object Store)
- MYOS1 (Custom Object Store)

### IBM Content Navigator (ICN / Navigator)

Enabled as part of the `workflow` pattern. Configured via `navigator_configuration`.

Provides the document management UI and is used as the Workplace entry point.

### AI Assistants (GenAI variant only)

```yaml
workflow_assistant_configuration:
  run_authoring_agent: true    # AI assistant for BAW authoring
  run_workplace_agent: true    # AI assistant in Workplace
```

Requires `workflow_assistant` and `workplace_assistant` in optional components.

### BAML (Business Automation Machine Learning)

Present in the template for potential intelligent task prioritization:

```yaml
baml_configuration:
  workforce_insights:
    replicas: 1
    resources: ...
  intelligent_task_prioritization:
    retrain_model_schedule: "*/30 * * * *"
```

## Database Configuration

Uses BAW Authoring SQL template: `db-statements-ref-baw-authoring.sql`

Key databases provisioned:

| Database | Variable | Purpose |
|----------|----------|---------|
| `*_baw_1` | `CP4BA_INST_BAS_1_DB_BAW_NAME` | BAW authoring / BAS |
| `*_appdb` | `CP4BA_INST_APP_DB_NAME` | Playback Server |
| `*_aaedb` | `CP4BA_INST_AE_DB_NAME` | Application Engine |
| `*_gcd` | `CP4BA_INST_GCD_DB_NAME` | FileNet GCD |
| `*_icn` | `CP4BA_INST_ICN_DB_NAME` | Content Navigator |
| `*_bawdocs` | `CP4BA_INST_DOCS_DB_NAME` | BAW DOCS object store |
| `*_bawdos` | `CP4BA_INST_DOS_DB_NAME` | BAW DOS object store |
| `*_bawtos` | `CP4BA_INST_TOS_DB_NAME` | BAW TOS / Case object store |
| `*_content` | `CP4BA_INST_CONTENT_DB_NAME` | Content object store |
| `*_chos` | `CP4BA_INST_CHOS_DB_NAME` | Case history object store |
| `*_os1` | `CP4BA_INST_OS1_DB_NAME` | Additional object store |

## Infrastructure

| Component | Configuration |
|-----------|--------------|
| PostgreSQL | 1 instance, SSL-enabled (`CP4BA_INST_DB_ONLY_SSL=true`), in-namespace |
| OpenLDAP | 1 instance, in-namespace |
| Storage (File) | `ocs-external-storagecluster-cephfs` |
| Storage (Block) | `ocs-external-storagecluster-ceph-rbd` |
| DB Storage | 10 Gi |
| BAW File Store | 20 Gi (dynamic) |

## CICD Variant Specifics

When using `env1-authoring-baw-cicd.properties`:

```bash
CP4BA_INST_GIT_ENABLED="true"
CP4BA_INST_GIT_TLS_SECRET_NAME="my-git-tls"
CP4BA_INST_GIT_HOST_NAME="github.com"
CP4BA_INST_GIT_HOST_PORT="443"
CP4BA_INST_GIT_USER_ID="marcoantonioni"
CP4BA_INST_GIT_REPO="test-cp4ba-cicd-sogei"
```

The GitHub TLS certificate is added to `bastudio_configuration.tls.tlsTrustList` and a dedicated custom XML template (`liberty-custom-xml-template-cicd`) is applied.

**Requires the GitHub token to be set before installation:**
```bash
export CP4BA_INST_GIT_TOKEN="<your-github-personal-access-token>"
```

## Installation Command

### Standard BAW Authoring

```bash
_PTC=/home/$USER/cp4ba-projects/cp4ba-installations/configs26
_VV=26.0.0
_KK=26.0.0
CONFIG_FILE=${_PTC}/env1-authoring-baw.properties
./cp4ba-one-shot-installation.sh -c ${CONFIG_FILE} -m -v ${_VV} -k ${_KK}
```

### CICD Variant (requires GitHub token)

```bash
export CP4BA_INST_GIT_TOKEN="<your-github-personal-access-token>"

_PTC=/home/$USER/cp4ba-projects/cp4ba-installations/configs26
_VV=26.0.0
_KK=26.0.0
CONFIG_FILE=${_PTC}/env1-authoring-baw-cicd.properties
./cp4ba-one-shot-installation.sh -c ${CONFIG_FILE} -m -v ${_VV} -k ${_KK}
```

### GenAI Variant (requires watsonx.ai credentials)

```bash
export CP4BA_INST_GENAI_ENABLED="true"
export CP4BA_INST_GENAI_WX_APIKEY="<your-ibm-cloud-api-key>"
export CP4BA_INST_GENAI_WX_PRJ_ID="<your-watsonx-project-id>"

_PTC=/home/$USER/cp4ba-projects/cp4ba-installations/configs26
_VV=26.0.0
_KK=26.0.0
CONFIG_FILE=${_PTC}/env1-authoring-baw-genai.properties
./cp4ba-one-shot-installation.sh -c ${CONFIG_FILE} -m -v ${_VV} -k ${_KK}
```

## References

- [BAW Authoring Parameters](https://www.ibm.com/docs/en/cloud-paks/cp-biz-automation/26.0.0?topic=parameters-business-automation-workflow-authoring)
- [BAS Parameters](https://www.ibm.com/docs/en/cloud-paks/cp-biz-automation/26.0.0?topic=parameters-business-automation-studio)
- [CP4BA 26.0.0 Documentation](https://www.ibm.com/docs/en/cloud-paks/cp-biz-automation/26.0.0)
- [IBM BAW Documentation](https://www.ibm.com/docs/en/baw/26.0.x)
