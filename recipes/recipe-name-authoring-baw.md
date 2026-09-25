# Recipe: Authoring BAW (BAW Authoring – no BAI)

**Created:** 2025-07-14T00:00:00Z  
**Template:** `cp4ba-installations/templates26/cp4ba-cr-ref-authoring-baw.yaml`  
**Config file:** `cp4ba-installations/configs26/env1-authoring-baw.properties`

---

## Description

The **Authoring BAW** recipe deploys a full IBM Business Automation Workflow (BAW) authoring environment without Business Automation Insights (BAI). It provides:

- Full BAW Authoring (BPM process authoring + Case Management design)
- IBM FileNet Content Platform Engine (CPE) with multiple object stores
- IBM Content Navigator (ICN)
- Business Automation Studio (BAS) – the low-code authoring environment
- AI Workflow and Workplace Assistants (when GenAI is enabled)

This is the baseline BAW authoring recipe, suited for process and case development without the overhead of streaming analytics infrastructure.

---

## CP4BA Capabilities

| Capability | Enabled | Notes |
|---|---|---|
| Foundation / CPFS | ✅ | Always included |
| BAW Authoring (BPM + Case) | ✅ | `baw_authoring` optional component |
| Business Automation Studio (BAS) | ✅ | `bas` optional component |
| IBM FileNet CPE | ✅ | Required by BAW |
| IBM Content Navigator | ✅ | Required by BAW |
| GraphQL API | ✅ | `CP4BA_INST_GRAPHQL=true` |
| Workflow AI Assistant | ✅ | When `CP4BA_INST_GENAI_ENABLED=true` |
| Workplace AI Assistant | ✅ | When `CP4BA_INST_GENAI_ENABLED=true` |
| Business Automation Insights (BAI) | ❌ | Not included in this recipe |
| Process Federation Server (PFS) | ❌ | Not included |
| Kafka / OpenSearch | ❌ | Not included |
| ADS | ❌ | Not included |
| ODM | ❌ | Not included |
| Application Engine | ❌ | Not included |

---

## Configuration Details

### Deployment Patterns & Optional Components

```
sc_deployment_patterns: foundation,workflow
sc_optional_components: baw_authoring,bas,workflow_assistant,workplace_assistant
Namespace: cp4ba-baw-auth
```

### Key CR Sections

The template includes the following major CR sections:
- `shared_configuration` – licenses, storage, image repo, IAM admin, encryption keys
- `ldap_configuration` – OpenLDAP (Custom type) with SCIM for IAM sync
- `datasource_configuration` – PostgreSQL connections for GCD, ICN, DOCS, DOS, TOS, CONTENT, OS1, BTS, IM, Zen
- `initialize_configuration` – LDAP realm, FileNet domain, object stores, ICN desktop
- `baml_configuration` – Workforce insights ML models, intelligent task prioritization
- `bastudio_configuration` – BAS admin user, DB connections, playback server, TLS, resource limits
- `workflow_authoring_configuration` – Case integration, content integration, federation, storage, custom XML
- `workflow_assistant_configuration` – GenAI authoring and workplace agents
- `ecm_configuration` – CPE (Content Platform Engine) + GraphQL
- `navigator_configuration` – IBM Content Navigator

### Databases Required (PostgreSQL)

| Database | Purpose |
|---|---|
| `baw_auth_gcd` | Global Configuration Database (CPE/FNCM) |
| `baw_auth_icn` | IBM Content Navigator |
| `baw_auth_bawdocs` | BAW Documents object store |
| `baw_auth_bawdos` | BAW Design object store |
| `baw_auth_bawtos` | BAW Target object store |
| `baw_auth_content` | Content object store |
| `baw_auth_os1` | Custom object store |
| `baw_auth_baw_1` | BAW main process database |
| `baw_auth_chos` | Case History object store |
| `baw_auth_appdb` | Application DB (playback server) |
| `baw_auth_awsdb` | Advanced Work Services DB |

### Storage

```
CP4BA_INST_SC_FILE: ocs-external-storagecluster-cephfs
CP4BA_INST_SC_BLOCK: ocs-external-storagecluster-ceph-rbd
CP4BA_INST_BAW_STORAGE_SIZE: 20Gi
```

### BAS Resource Limits

```
CPU limit: 5000m
Memory limit: 3096Mi
Replicas: 1
```

### GenAI / WatsonX Configuration (optional)

When enabling GenAI:
```bash
export CP4BA_INST_GENAI_ENABLED="true"
export CP4BA_INST_GENAI_WX_APIKEY="<your-watsonx-api-key>"
export CP4BA_INST_GENAI_WX_PRJ_ID="<your-watsonx-project-id>"
```

---

## Installation Commands

### Standard (no GenAI)

```bash
_VV=26.0.2
_KK=26.0.0-IF002
_PTC=/home/$USER/cp4ba-projects/cp4ba-installations/configs26
CONFIG_FILE=${_PTC}/env1-authoring-baw.properties
./cp4ba-one-shot-installation.sh -c ${CONFIG_FILE} -m -v ${_VV} -k ${_KK}
```

*Last tested: 20260806*

### With GenAI enabled

```bash
export CP4BA_INST_GENAI_ENABLED="true"
export CP4BA_INST_GENAI_WX_APIKEY="<your-watsonx-api-key>"
export CP4BA_INST_GENAI_WX_PRJ_ID="<your-watsonx-project-id>"

_VV=26.0.2
_KK=26.0.0-IF002
_PTC=/home/$USER/cp4ba-projects/cp4ba-installations/configs26
CONFIG_FILE=${_PTC}/env1-authoring-baw-genai.properties
./cp4ba-one-shot-installation.sh -c ${CONFIG_FILE} -m -v ${_VV} -k ${_KK}
```

*Last tested: 20260717*

### Parameters Reference

| Parameter | Description |
|---|---|
| `-c ${CONFIG_FILE}` | Path to the properties configuration file |
| `-m` | Install a fresh CP4BA Case Package Manager |
| `-v ${_VV}` | CP4BA version (e.g., `26.0.2`) |
| `-k ${_KK}` | cert-kubernetes version (e.g., `26.0.0-IF002`) |

---

## Notes

- This recipe does **not** include BAI. Use `recipe-name-authoring-baw-bai.md` if streaming analytics are needed.
- BAI is disabled in the config (`CP4BA_INST_BAI_ENABLE=false`) even though `bai_configuration` section may be present in the template with disabled settings.
- BAML (Business Automation Machine Learning) is configured with 1 replica for workforce insights and task prioritization models.
- The `bastudio_configuration` includes a Git integration section (`CP4BA_INST_GIT_ENABLED=false` by default). For CICD use case, refer to `env1-authoring-baw-cicd.properties`.
- Custom XML secrets (Liberty + Lombardi) are configured via `CP4BA_INST_LIBERTY_CUSTOM_XML_SECRET_NAME` and `CP4BA_INST_LOMBARDI_CUSTOM_XML_SECRET_NAME`.
