# Recipe: Authoring BAW + BAI (BAW Authoring with Business Automation Insights)

**Created:** 2025-07-14T00:00:00Z  
**Template:** `cp4ba-installations/templates26/cp4ba-cr-ref-authoring-baw-bai.yaml`  
**Config file:** `cp4ba-installations/configs26/env1-authoring-baw-bai.properties`

---

## Description

The **Authoring BAW + BAI** recipe extends the basic BAW authoring recipe by adding:

- **Business Automation Insights (BAI)** – real-time process analytics and dashboards
- **Apache Kafka** – event streaming infrastructure
- **OpenSearch** – analytics data store and search
- **Process Federation Server (PFS)** – federated task list

This recipe is the recommended choice when you need both BAW process/case authoring capability and operational analytics for process performance monitoring.

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
| Business Automation Insights (BAI) | ✅ | `bai` optional component |
| Process Federation Server (PFS) | ✅ | `pfs` optional component |
| Kafka | ✅ | `kafka` optional component |
| OpenSearch | ✅ | `opensearch` optional component |
| Workflow AI Assistant | ✅ | When `CP4BA_INST_GENAI_ENABLED=true` |
| Workplace AI Assistant | ✅ | When `CP4BA_INST_GENAI_ENABLED=true` |
| ADS | ❌ | Not included |
| ODM | ❌ | Not included |
| Application Engine | ❌ | Not included |

---

## Configuration Details

### Deployment Patterns & Optional Components

```
sc_deployment_patterns: foundation,workflow
sc_optional_components: baw_authoring,bas,bai,pfs,kafka,opensearch,workflow_assistant,workplace_assistant
Namespace: cp4ba-baw-bai-auth
```

### Key CR Sections

The template includes the following major CR sections:
- `shared_configuration` – licenses, storage, image repo, IAM admin, encryption keys
- `ldap_configuration` – OpenLDAP (Custom type) with SCIM for IAM sync
- `datasource_configuration` – PostgreSQL connections for GCD, ICN, DOCS, DOS, TOS, CONTENT, OS1, AEOS, CHOS, APP, AWS, AWSDOCS
- `initialize_configuration` – LDAP realm, FileNet domain, object stores, ICN desktop
- `bai_configuration` – BAI event emitters, Flink, BPC dashboards, workforce insights
- `baml_configuration` – Workforce insights ML models, intelligent task prioritization
- `bastudio_configuration` – BAS admin user, DB connections, playback server, TLS, resource limits
- `workflow_authoring_configuration` – Case, content integration, federation, storage, business events, custom XML
- `workflow_assistant_configuration` – GenAI authoring and workplace agents
- `ecm_configuration` – CPE + GraphQL
- `navigator_configuration` – IBM Content Navigator

### BAI Configuration (key settings)

```yaml
bai_configuration:
  business_performance_center:
    all_users_access: true       # all users can access BPC
    workforce_insights: true     # workforce insights dashboard
  bpmn:
    install: true
    time_series: true            # BPMN time-series analytics
  content:
    install: true                # content events
  icm:
    install: true                # ICM/Case events
  navigator:
    install: true                # Navigator events
  bawadv:
    install: false
  ads:
    install: false
  odm:
    install: false
  event_forwarder:
    kafka: true                  # egress to Kafka
  flink:
    create_route: true
    additional_task_managers: 1
```

### Databases Required (PostgreSQL)

| Database | Purpose |
|---|---|
| `baw_bai_auth_gcd` | Global Configuration Database |
| `baw_bai_auth_icn` | IBM Content Navigator |
| `baw_bai_auth_bawdocs` | BAW Documents object store |
| `baw_bai_auth_bawdos` | BAW Design object store |
| `baw_bai_auth_bawtos` | BAW Target object store |
| `baw_bai_auth_content` | Content object store |
| `baw_bai_auth_os1` | Custom object store |
| `baw_bai_auth_baw_1` | BAW main process database |
| `baw_bai_auth_chos` | Case History object store |
| `baw_bai_auth_appdb` | Application DB |
| `baw_bai_auth_awsdb` | Advanced Work Services DB |
| `baw_bai_auth_awsdocs` | Advanced Work Services docs |
| `baw_bai_auth_aaedb` | Application Engine DB |
| `baw_bai_auth_aeos` | App Engine Object Store |

### Storage

```
CP4BA_INST_SC_FILE: ocs-external-storagecluster-cephfs
CP4BA_INST_SC_BLOCK: ocs-external-storagecluster-ceph-rbd
CP4BA_INST_BAW_STORAGE_SIZE: 20Gi
```

---

## Installation Commands

### Standard

```bash
_VV=26.0.2
_KK=26.0.0-IF002
_PTC=/home/$USER/cp4ba-projects/cp4ba-installations/configs26
CONFIG_FILE=${_PTC}/env1-authoring-baw-bai.properties
./cp4ba-one-shot-installation.sh -c ${CONFIG_FILE} -m -v ${_VV} -k ${_KK}
```

*Last tested: 20260806*

### With PFS GenAI

```bash
export CP4BA_INST_GENAI_ENABLED="true"
export CP4BA_INST_GENAI_WX_APIKEY="<your-watsonx-api-key>"
export CP4BA_INST_GENAI_WX_PRJ_ID="<your-watsonx-project-id>"

_VV=26.0.2
_KK=26.0.0-IF002
_PTC=/home/$USER/cp4ba-projects/cp4ba-installations/configs26
CONFIG_FILE=${_PTC}/env1-authoring-baw-pfs-genai.properties
./cp4ba-one-shot-installation.sh -c ${CONFIG_FILE} -m -v ${_VV} -k ${_KK}
```

*Last tested: 20260720*

### Parameters Reference

| Parameter | Description |
|---|---|
| `-c ${CONFIG_FILE}` | Path to the properties configuration file |
| `-m` | Install a fresh CP4BA Case Package Manager |
| `-v ${_VV}` | CP4BA version (e.g., `26.0.2`) |
| `-k ${_KK}` | cert-kubernetes version (e.g., `26.0.0-IF002`) |

---

## Notes

- BAI is enabled (`CP4BA_INST_BAI_ENABLE=true`) and workforce insights are active.
- The BAI event emitter ID is `EEID1` with start date `20240301T000000Z`.
- Object store content events are enabled (`CP4BA_INST_BAI_OBJECTSTORE_CONTENT_EVENT_ENABLED=true`).
- Flink task manager: 1 additional task manager, route exposed for admin access.
- This recipe is the most common choice for BAW authoring environments with full observability.
