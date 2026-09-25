# Recipe: Authoring Decision ADS (ADS Authoring – no BAI)

**Created:** 2025-07-14T00:00:00Z  
**Template:** `cp4ba-installations/templates26/cp4ba-cr-ref-authoring-decision-ads.yaml`  
**Config file:** `cp4ba-installations/configs26/env1-authoring-ads.properties`

---

## Description

The **Authoring Decision ADS** recipe deploys IBM Decision Intelligence Client Managed Software (ADS, also known as DICM) in full authoring mode:

- **Decision Designer** – visual DMN-based decision model authoring
- **Decision Runtime** – execution engine for deployed decision services
- **Business Automation Studio (BAS)** – integrated authoring UI

This recipe is for teams focused on decision automation using AI-augmented decision modeling without BAW workflow capabilities.

---

## CP4BA Capabilities

| Capability | Enabled | Notes |
|---|---|---|
| Foundation / CPFS | ✅ | Always included |
| ADS Decision Designer | ✅ | `ads_designer` optional component |
| ADS Decision Runtime | ✅ | `ads_runtime` optional component |
| Business Automation Studio (BAS) | ✅ | `bas` optional component |
| Business Automation Insights (BAI) | ❌ | Not included (use authoring-decision-ads-bai) |
| BAW Authoring | ❌ | Not included |
| ODM | ❌ | Not included |
| Application Engine | ❌ | Not included |

---

## Configuration Details

### Deployment Patterns & Optional Components

```
sc_deployment_patterns: foundation,decisions_ads
sc_optional_components: ads_designer,ads_runtime,bas
Namespace: cp4ba-ads-auth
```

### Key CR Sections

- `shared_configuration` – deployment license, storage, IAM, image repo
- `ldap_configuration` – OpenLDAP with SCIM
- `datasource_configuration` – PostgreSQL connections for ADS designer and runtime
- `bastudio_configuration` – BAS configuration
- `ads_configuration` – ADS Decision Designer + Runtime
- `ecm_configuration` – CPE (required by foundation/BAS)
- `navigator_configuration` – IBM Content Navigator

### ADS Configuration

```yaml
ads_configuration:
  decision_designer:
    enabled: true
    database:
      # DB: ads_auth_adsdesignerdb
  decision_runtime:
    enabled: true
    database:
      # DB: ads_auth_adsruntimedb
```

### GenAI / WatsonX for ADS

```bash
export CP4BA_INST_ADS_GENAI_APIKEY="<your-ads-genai-api-key>"
export CP4BA_INST_ADS_GENAI_PRJ_ID="<your-ads-genai-project-id>"
```

### Databases Required (PostgreSQL)

| Database | Purpose |
|---|---|
| `ads_auth_adsdesignerdb` | ADS Decision Designer models |
| `ads_auth_adsruntimedb` | ADS Decision Runtime state |
| `ads_auth_icn` | IBM Content Navigator (foundation) |
| `ads_auth_gcd` | Global Configuration DB (foundation) |

### Storage

```
CP4BA_INST_SC_FILE: ocs-external-storagecluster-cephfs
CP4BA_INST_SC_BLOCK: ocs-external-storagecluster-ceph-rbd
```

---

## Installation Commands

### Without GenAI

```bash
_VV=26.0.1
_KK=26.0.0-IF001
_PTC=/home/$USER/cp4ba-projects/cp4ba-installations/configs26
CONFIG_FILE=${_PTC}/env1-authoring-ads.properties
./cp4ba-one-shot-installation.sh -c ${CONFIG_FILE} -m -v ${_VV} -k ${_KK}
```

### With ADS GenAI enabled

```bash
export CP4BA_INST_ADS_GENAI_APIKEY="<your-ads-genai-api-key>"
export CP4BA_INST_ADS_GENAI_PRJ_ID="<your-ads-genai-project-id>"

_VV=26.0.1
_KK=26.0.0-IF001
_PTC=/home/$USER/cp4ba-projects/cp4ba-installations/configs26
CONFIG_FILE=${_PTC}/env1-authoring-ads.properties
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

- This recipe uses `decisions_ads` pattern (not `decisions` which is used for ODM).
- ADS Decision Designer provides a browser-based authoring environment for decision models (DMN, decision tables, predictive models).
- The ADS REST API is exposed as an OpenAPI endpoint for invoking decisions programmatically.
- For ADS with analytics, use `recipe-name-authoring-decision-ads-bai.md`.
- For ADS runtime only (no authoring), use `recipe-name-decision-ads.md`.
