# Recipe: Decision ADS + BAI (ADS Runtime with Business Automation Insights)

**Created:** 2025-07-14T00:00:00Z  
**Template:** `cp4ba-installations/templates26/cp4ba-cr-ref-decision-ads-bai.yaml`  
**Config file:** `cp4ba-installations/configs26/env1-runtime-ads-bai.properties`

---

## Description

The **Decision ADS + BAI** recipe deploys IBM ADS (Decision Intelligence) in **runtime-only mode** with **Business Automation Insights (BAI)** for analytics. It provides:

- **ADS Decision Runtime** – execution engine for deployed decision services
- **BAI** – decision event analytics and BPC dashboards
- **Kafka** and **OpenSearch** – event streaming and indexing

This recipe is the production ADS runtime with full observability.

---

## CP4BA Capabilities

| Capability | Enabled | Notes |
|---|---|---|
| Foundation / CPFS | ✅ | Always included |
| ADS Decision Runtime | ✅ | `ads_runtime` optional component |
| Business Automation Insights (BAI) | ✅ | `bai` optional component |
| Kafka | ✅ | Required by BAI |
| OpenSearch | ✅ | Required by BAI |
| ADS Decision Designer | ❌ | Not included (runtime only) |
| Business Automation Studio (BAS) | ❌ | Not included |
| BAW | ❌ | Not included |
| ODM | ❌ | Not included |

---

## Configuration Details

### Deployment Patterns & Optional Components

```
sc_deployment_patterns: foundation,decisions_ads
sc_optional_components: ads_runtime,bai
Namespace: cp4ba-ads-bai-prod
```

### Key Variables

| Variable | Value |
|---|---|
| `CP4BA_INST_ENV` | `ads-bai-prod` |
| `CP4BA_INST_NAMESPACE` | `cp4ba-ads-bai-prod` |
| `CP4BA_INST_DEPL_PATTERNS` | `foundation,decisions_ads` |
| `CP4BA_INST_OPT_COMPONENTS` | `ads_runtime,bai` |

### BAI for ADS

```yaml
bai_configuration:
  ads:
    install: true
  bpmn:
    install: false
  odm:
    install: false
```

### Databases Required (PostgreSQL)

| Database | Purpose |
|---|---|
| `ads_bai_prod_adsruntimedb` | ADS Decision Runtime state |
| `ads_bai_prod_icn` | IBM Content Navigator |
| `ads_bai_prod_gcd` | Global Configuration DB |

---

## Installation Commands

### Without ADS GenAI

```bash
_VV=26.0.1
_KK=26.0.0-IF001
_PTC=/home/$USER/cp4ba-projects/cp4ba-installations/configs26
CONFIG_FILE=${_PTC}/env1-runtime-ads-bai.properties
./cp4ba-one-shot-installation.sh -c ${CONFIG_FILE} -m -v ${_VV} -k ${_KK}
```

### With ADS GenAI enabled

```bash
export CP4BA_INST_ADS_GENAI_APIKEY="<your-ads-genai-api-key>"
export CP4BA_INST_ADS_GENAI_PRJ_ID="<your-ads-genai-project-id>"

_VV=26.0.1
_KK=26.0.0-IF001
_PTC=/home/$USER/cp4ba-projects/cp4ba-installations/configs26
CONFIG_FILE=${_PTC}/env1-runtime-ads-bai.properties
./cp4ba-one-shot-installation.sh -c ${CONFIG_FILE} -m -v ${_VV} -k ${_KK}
```

### Parameters Reference

| Parameter | Description |
|---|---|
| `-c ${CONFIG_FILE}` | Path to the properties configuration file |
| `-m` | Install a fresh CP4BA Case Package Manager |
| `-v ${_VV}` | CP4BA version (e.g., `26.0.1`) |
| `-k ${_KK}` | cert-kubernetes version (e.g., `26.0.0-IF001`) |

---

## Notes

- Key difference from `recipe-name-decision-ads.md`: adds `bai` to optional components.
- BAI collects ADS decision execution events and makes them available in BPC dashboards.
- Kafka and OpenSearch are automatically included when `bai` is in optional components.
