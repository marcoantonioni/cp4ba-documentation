# Recipe: Authoring Decision ADS + BAI (ADS Authoring with Business Automation Insights)

**Created:** 2025-07-14T00:00:00Z  
**Template:** `cp4ba-installations/templates26/cp4ba-cr-ref-authoring-decision-ads-bai.yaml`  
**Config file:** `cp4ba-installations/configs26/env1-authoring-ads-bai.properties`

---

## Description

The **Authoring Decision ADS + BAI** recipe extends the ADS authoring recipe by adding Business Automation Insights (BAI) for decision analytics. It provides:

- **ADS Decision Designer** – visual decision model authoring
- **ADS Decision Runtime** – execution engine
- **Business Automation Studio (BAS)** – integrated authoring UI
- **BAI** – decision event analytics and dashboards
- **Kafka** and **OpenSearch** – event streaming and indexing

This recipe is for teams who need ADS authoring **and** analytics for monitoring decision service performance and outcomes.

---

## CP4BA Capabilities

| Capability | Enabled | Notes |
|---|---|---|
| Foundation / CPFS | ✅ | Always included |
| ADS Decision Designer | ✅ | `ads_designer` optional component |
| ADS Decision Runtime | ✅ | `ads_runtime` optional component |
| Business Automation Studio (BAS) | ✅ | `bas` optional component |
| Business Automation Insights (BAI) | ✅ | `bai` optional component |
| BAW Authoring | ❌ | Not included |
| ODM | ❌ | Not included |
| Application Engine | ❌ | Not included |
| Process Federation Server (PFS) | ❌ | Not included |

---

## Configuration Details

### Deployment Patterns & Optional Components

```
sc_deployment_patterns: foundation,decisions_ads
sc_optional_components: ads_designer,ads_runtime,bas,bai
Namespace: cp4ba-ads-bai-auth
```

### Key Variables

| Variable | Value |
|---|---|
| `CP4BA_INST_ENV` | `ads-bai-auth` |
| `CP4BA_INST_NAMESPACE` | `cp4ba-ads-bai-auth` |
| `CP4BA_INST_DEPL_PATTERNS` | `foundation,decisions_ads` |
| `CP4BA_INST_OPT_COMPONENTS` | `ads_designer,ads_runtime,bas,bai` |
| `CP4BA_INST_DEPL_PROFILE_SIZE` | `small` |

### BAI for ADS

```yaml
bai_configuration:
  ads:
    install: true    # ADS decision events collected by BAI
  bpmn:
    install: false
  icm:
    install: false
  odm:
    install: false
```

### Databases Required (PostgreSQL)

| Database | Purpose |
|---|---|
| `ads_bai_auth_adsdesignerdb` | ADS Decision Designer models |
| `ads_bai_auth_adsruntimedb` | ADS Decision Runtime state |
| `ads_bai_auth_icn` | IBM Content Navigator |
| `ads_bai_auth_gcd` | Global Configuration DB |

---

## Installation Commands

### Without ADS GenAI

```bash
_VV=26.0.1
_KK=26.0.0-IF001
_PTC=/home/$USER/cp4ba-projects/cp4ba-installations/configs26
CONFIG_FILE=${_PTC}/env1-authoring-ads-bai.properties
./cp4ba-one-shot-installation.sh -c ${CONFIG_FILE} -m -v ${_VV} -k ${_KK}
```

### With ADS GenAI enabled

```bash
export CP4BA_INST_ADS_GENAI_APIKEY="<your-ads-genai-api-key>"
export CP4BA_INST_ADS_GENAI_PRJ_ID="<your-ads-genai-project-id>"

_VV=26.0.1
_KK=26.0.0-IF001
_PTC=/home/$USER/cp4ba-projects/cp4ba-installations/configs26
CONFIG_FILE=${_PTC}/env1-authoring-ads-bai.properties
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

- The key difference from `recipe-name-authoring-decision-ads.md` is the addition of `bai` to optional components.
- BAI ADS emitter collects decision events (invocations, outcomes) for BPC dashboards.
- Kafka and OpenSearch are automatically added when `bai` is in optional components.
- For ADS runtime-only with BAI, use `recipe-name-decision-ads-bai.md`.
