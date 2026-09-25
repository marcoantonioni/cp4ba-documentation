# Recipe: Authoring BAW + BAI + AE + ADS (Full BAW Authoring with ADS)

**Created:** 2025-07-14T00:00:00Z  
**Template:** `cp4ba-installations/templates26/cp4ba-cr-ref-authoring-baw-bai-ae-ads.yaml`  
**Config file:** `cp4ba-installations/configs26/env1-authoring-baw-bai-ae-ads.properties`

---

## Description

The **Authoring BAW + BAI + AE + ADS** recipe is the most comprehensive authoring recipe, combining:

- Full BAW Authoring (BPM + Case Management)
- Business Automation Insights (BAI) analytics
- Application Engine (AE) with App Designer
- **IBM Decision Intelligence / ADS** (Decision Designer + Decision Runtime)

This recipe addresses organizations that want to author and run both workflow processes/cases **and** AI-augmented decision automation from a single environment.

---

## CP4BA Capabilities

| Capability | Enabled | Notes |
|---|---|---|
| Foundation / CPFS | ✅ | Always included |
| BAW Authoring (BPM + Case) | ✅ | `baw_authoring` optional component |
| Business Automation Studio (BAS) | ✅ | `bas` optional component |
| App Designer | ✅ | `app_designer` optional component |
| Application Engine | ✅ | via `application` pattern |
| ADS Decision Designer | ✅ | `ads_designer` optional component |
| ADS Decision Runtime | ✅ | `ads_runtime` optional component |
| IBM FileNet CPE | ✅ | Required by BAW |
| IBM Content Navigator | ✅ | Required by BAW |
| GraphQL API | ✅ | Enabled |
| Business Automation Insights (BAI) | ✅ | `bai` optional component |
| Process Federation Server (PFS) | ✅ | `pfs` optional component |
| Kafka | ✅ | `kafka` optional component |
| OpenSearch | ✅ | `opensearch` optional component |
| Workflow AI Assistant | ✅ | When `CP4BA_INST_GENAI_ENABLED=true` |
| Workplace AI Assistant | ✅ | When `CP4BA_INST_GENAI_ENABLED=true` |
| ODM | ❌ | Not included |

---

## Configuration Details

### Deployment Patterns & Optional Components

```
sc_deployment_patterns: foundation,workflow,application,decisions_ads
sc_optional_components: baw_authoring,bas,app_designer,ads_designer,ads_runtime,bai,pfs,kafka,opensearch,workflow_assistant,workplace_assistant
Namespace: cp4ba-baw-bai-ae-ads-auth
```

### Key CR Sections

All sections from `cp4ba-cr-ref-authoring-baw-bai-ae.yaml` plus:
- `ads_configuration` – ADS Decision Designer + Decision Runtime

### ADS Configuration

```yaml
ads_configuration:
  decision_designer:
    enabled: true
    # DB: baw_bai_ae_ads_auth_adsdesignerdb
  decision_runtime:
    enabled: true
    # DB: baw_bai_ae_ads_auth_adsruntimedb
```

### GenAI / WatsonX for ADS

ADS supports GenAI integration for AI-augmented decision authoring:
```bash
export CP4BA_INST_ADS_GENAI_APIKEY="<your-ads-genai-api-key>"
export CP4BA_INST_ADS_GENAI_PRJ_ID="<your-ads-genai-project-id>"
```

### Databases Required (PostgreSQL)

Same as BAW+BAI+AE recipe, plus:

| Database | Purpose |
|---|---|
| `baw_bai_ae_ads_auth_adsruntimedb` | ADS Decision Runtime state |
| `baw_bai_ae_ads_auth_adsdesignerdb` | ADS Decision Designer models |

### Storage

```
CP4BA_INST_SC_FILE: ocs-external-storagecluster-cephfs
CP4BA_INST_SC_BLOCK: ocs-external-storagecluster-ceph-rbd
CP4BA_INST_BAW_STORAGE_SIZE: 20Gi
```

---

## Installation Commands

### Standard (without ADS GenAI)

```bash
_VV=26.0.1
_KK=26.0.0-IF001
_PTC=/home/$USER/cp4ba-projects/cp4ba-installations/configs26
CONFIG_FILE=${_PTC}/env1-authoring-baw-bai-ae-ads.properties
./cp4ba-one-shot-installation.sh -c ${CONFIG_FILE} -m -v ${_VV} -k ${_KK}
```

### With ADS GenAI enabled

```bash
export CP4BA_INST_ADS_GENAI_APIKEY="<your-ads-genai-api-key>"
export CP4BA_INST_ADS_GENAI_PRJ_ID="<your-ads-genai-project-id>"

_VV=26.0.1
_KK=26.0.0-IF001
_PTC=/home/$USER/cp4ba-projects/cp4ba-installations/configs26
CONFIG_FILE=${_PTC}/env1-authoring-baw-bai-ae-ads.properties
./cp4ba-one-shot-installation.sh -c ${CONFIG_FILE} -m -v ${_VV} -k ${_KK}
```

### With RPA integration

```bash
_VV=26.0.2
_KK=26.0.0-IF002
_PTC=/home/$USER/cp4ba-projects/cp4ba-installations/configs26
CONFIG_FILE=${_PTC}/env1-authoring-baw-bai-ae-rpa.properties
./cp4ba-one-shot-installation.sh -c ${CONFIG_FILE} -m -v ${_VV} -k ${_KK}
```

*Last tested: 20260902*

### Parameters Reference

| Parameter | Description |
|---|---|
| `-c ${CONFIG_FILE}` | Path to the properties configuration file |
| `-m` | Install a fresh CP4BA Case Package Manager |
| `-v ${_VV}` | CP4BA version (e.g., `26.0.1`) |
| `-k ${_KK}` | cert-kubernetes version (e.g., `26.0.0-IF001`) |

---

## Notes

- This is the most feature-rich authoring recipe in the repository.
- ADS Decision Designer allows authors to create DMN-based decision models.
- ADS Decision Runtime executes deployed decision services via REST API.
- When `CP4BA_INST_ADS_GENAI_APIKEY` is set, ADS Decision Designer can use AI assistance for rule authoring.
- All BAW+BAI+AE components are included; refer to those recipe files for their specific configuration details.
- An RPA integration variant exists (`env1-authoring-baw-bai-ae-rpa.properties`) that adds RPA capability to this combined recipe.
