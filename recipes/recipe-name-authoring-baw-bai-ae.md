# Recipe: Authoring BAW + BAI + AE (BAW Authoring with Application Engine)

**Created:** 2025-07-14T00:00:00Z  
**Template:** `cp4ba-installations/templates26/cp4ba-cr-ref-authoring-baw-bai-ae.yaml`  
**Config file:** `cp4ba-installations/configs26/env1-authoring-baw-bai-ae.properties`

---

## Description

The **Authoring BAW + BAI + AE** recipe extends the BAW+BAI authoring recipe by adding the **Application Engine (AE)** and **App Designer** capability. This provides:

- Full BAW Authoring (BPM + Case) with BAI analytics
- **Application Engine** – runtime for custom process applications
- **App Designer** – low-code application builder integrated in BAS
- IBM FileNet CPE, ICN, BAS, PFS, Kafka, OpenSearch

This recipe is the recommended choice when you need to build and deploy custom process applications (App Designer apps) alongside traditional BAW processes.

---

## CP4BA Capabilities

| Capability | Enabled | Notes |
|---|---|---|
| Foundation / CPFS | ✅ | Always included |
| BAW Authoring (BPM + Case) | ✅ | `baw_authoring` optional component |
| Business Automation Studio (BAS) | ✅ | `bas` optional component |
| App Designer | ✅ | `app_designer` optional component |
| Application Engine | ✅ | via `application` pattern |
| IBM FileNet CPE | ✅ | Required by BAW |
| IBM Content Navigator | ✅ | Required by BAW |
| GraphQL API | ✅ | Enabled |
| Business Automation Insights (BAI) | ✅ | `bai` optional component |
| Process Federation Server (PFS) | ✅ | `pfs` optional component |
| Kafka | ✅ | `kafka` optional component |
| OpenSearch | ✅ | `opensearch` optional component |
| Workflow AI Assistant | ✅ | When `CP4BA_INST_GENAI_ENABLED=true` |
| Workplace AI Assistant | ✅ | When `CP4BA_INST_GENAI_ENABLED=true` |
| ADS | ❌ | Not included |
| ODM | ❌ | Not included |

---

## Configuration Details

### Deployment Patterns & Optional Components

```
sc_deployment_patterns: foundation,workflow,application
sc_optional_components: baw_authoring,bas,app_designer,bai,pfs,kafka,opensearch,workflow_assistant,workplace_assistant
Namespace: cp4ba-baw-bai-ae-auth
```

### Key CR Sections

All sections from `cp4ba-cr-ref-authoring-baw-bai.yaml` plus:
- `application_engine_configuration` (AE instance: `workspace`, type: development)
- Additional CPE object stores: `AE`, `AEOS`, `APP`, `CHOS`, `AWS`, `AWSDOCS`

### Application Engine Configuration

```yaml
# Key variables
CP4BA_INST_AE_PERSISTENCE_ENABLE: false
CP4BA_INST_AE_OS_NAME: "AEOS"           # AE object store
CP4BA_INST_AE_SERVER_ENV_TYPE: development
CP4BA_INST_AE_NAME: workspace
CP4BA_INST_AE_DB_TYPE: postgresql
```

### Databases Required (PostgreSQL)

Same as BAW+BAI recipe, plus:

| Database | Purpose |
|---|---|
| `baw_bai_ae_auth_aaedb` | Application Engine database |
| `baw_bai_ae_auth_aeos` | App Engine Object Store |
| `baw_bai_ae_auth_appdb` | Application database |
| `baw_bai_ae_auth_awsdb` | Advanced Work Services DB |
| `baw_bai_ae_auth_awsdocs` | Advanced Work Services docs |

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
CONFIG_FILE=${_PTC}/env1-authoring-baw-bai-ae.properties
./cp4ba-one-shot-installation.sh -c ${CONFIG_FILE} -m -v ${_VV} -k ${_KK}
```

*Last tested: 20260806*

### Parameters Reference

| Parameter | Description |
|---|---|
| `-c ${CONFIG_FILE}` | Path to the properties configuration file |
| `-m` | Install a fresh CP4BA Case Package Manager |
| `-v ${_VV}` | CP4BA version (e.g., `26.0.2`) |
| `-k ${_KK}` | cert-kubernetes version (e.g., `26.0.0-IF002`) |

---

## Notes

- This recipe adds `application` to the deployment patterns compared to `authoring-baw-bai`.
- The App Designer component (`app_designer`) is the BAS-integrated low-code application authoring tool.
- The Application Engine (`workspace` instance) uses `development` environment type for authoring use cases.
- AE data persistence is disabled by default (`CP4BA_INST_AE_PERSISTENCE_ENABLE=false`).
- Additional CPE object stores (`AE`, `AEOS`, `APP`, `AWS`, `AWSDOCS`) are required compared to the base BAW recipe.
- `CP4BA_INST_LIBERTY_CUSTOM_XML_TEMPLATE_NAME=liberty-custom-xml-template-sample-custom-db` (note: different from basic BAW recipe).
