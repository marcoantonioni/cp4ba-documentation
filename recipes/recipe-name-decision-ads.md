# Recipe: Decision ADS (ADS Runtime only)

**Created:** 2025-07-14T00:00:00Z  
**Template:** `cp4ba-installations/templates26/cp4ba-cr-ref-decision-ads.yaml`  
**Config file:** `cp4ba-installations/configs26/env1-runtime-ads.properties`

---

## Description

The **Decision ADS** recipe deploys IBM Decision Intelligence Client Managed Software (ADS) in **runtime-only mode**. It provides only the **ADS Decision Runtime** execution engine without the authoring tools (Decision Designer). 

This recipe is suitable for production environments where decision services have been authored and deployed from a separate ADS Authoring environment.

---

## CP4BA Capabilities

| Capability | Enabled | Notes |
|---|---|---|
| Foundation / CPFS | ✅ | Always included |
| ADS Decision Runtime | ✅ | `ads_runtime` optional component |
| ADS Decision Designer | ❌ | Not included (runtime only) |
| Business Automation Studio (BAS) | ❌ | Not included |
| Business Automation Insights (BAI) | ❌ | Not included (use decision-ads-bai) |
| BAW | ❌ | Not included |
| ODM | ❌ | Not included |

---

## Configuration Details

### Deployment Patterns & Optional Components

```
sc_deployment_patterns: foundation,decisions_ads
sc_optional_components: ads_runtime
Namespace: cp4ba-ads-prod
```

### Key Variables

| Variable | Value |
|---|---|
| `CP4BA_INST_ENV` | `ads-prod` |
| `CP4BA_INST_NAMESPACE` | `cp4ba-ads-prod` |
| `CP4BA_INST_DEPL_PATTERNS` | `foundation,decisions_ads` |
| `CP4BA_INST_OPT_COMPONENTS` | `ads_runtime` |
| `CP4BA_INST_DEPL_PROFILE_SIZE` | `small` |

### ADS Runtime Configuration

```yaml
ads_configuration:
  decision_runtime:
    enabled: true
    # DB: ads_prod_adsruntimedb
  decision_designer:
    enabled: false
```

### Databases Required (PostgreSQL)

| Database | Purpose |
|---|---|
| `ads_prod_adsruntimedb` | ADS Decision Runtime state |
| `ads_prod_icn` | IBM Content Navigator (foundation) |
| `ads_prod_gcd` | Global Configuration DB (foundation) |

---

## Installation Commands

### Without ADS GenAI

```bash
_VV=26.0.1
_KK=26.0.0-IF001
_PTC=/home/$USER/cp4ba-projects/cp4ba-installations/configs26
CONFIG_FILE=${_PTC}/env1-runtime-ads.properties
./cp4ba-one-shot-installation.sh -c ${CONFIG_FILE} -m -v ${_VV} -k ${_KK}
```

### With ADS GenAI enabled

```bash
export CP4BA_INST_ADS_GENAI_APIKEY="<your-ads-genai-api-key>"
export CP4BA_INST_ADS_GENAI_PRJ_ID="<your-ads-genai-project-id>"

_VV=26.0.1
_KK=26.0.0-IF001
_PTC=/home/$USER/cp4ba-projects/cp4ba-installations/configs26
CONFIG_FILE=${_PTC}/env1-runtime-ads.properties
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

- This is the minimal ADS recipe: only the runtime is deployed, no authoring tools.
- Decision services must be authored in a separate ADS Authoring environment and then deployed to this runtime.
- The ADS Decision Runtime exposes a REST API for invoking deployed decision services.
- For runtime with BAI analytics, use `recipe-name-decision-ads-bai.md`.
- For authoring + runtime, use `recipe-name-authoring-decision-ads.md`.
