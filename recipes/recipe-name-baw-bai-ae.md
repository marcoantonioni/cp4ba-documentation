# Recipe: BAW + BAI + AE (BAW Runtime with Application Engine)

**Created:** 2025-07-14T00:00:00Z  
**Template:** `cp4ba-installations/templates26/cp4ba-cr-ref-baw-bai-ae.yaml`  
**Config file:** `cp4ba-installations/configs26/env1-runtime-baw-bai.properties`

> **Note:** The config file `env1-runtime-baw-bai.properties` is the base for this recipe; however, the template `cp4ba-cr-ref-baw-bai-ae.yaml` adds the `application` pattern and `app_designer` component. --THIS SECTION MUST BE UPDATED-- if a dedicated properties file exists for this template.

---

## Description

The **BAW + BAI + AE** recipe extends the BAW Runtime recipe by adding the **Application Engine (AE)** with **App Designer** capability. This provides a production runtime environment for:

- **BAW Runtime** – process and case execution
- **Application Engine** – hosting environment for custom process applications
- **App Designer** – low-code application runtime
- **BAI** – real-time process and application analytics
- **Kafka** and **OpenSearch** – event streaming and indexing

---

## CP4BA Capabilities

| Capability | Enabled | Notes |
|---|---|---|
| Foundation / CPFS | ✅ | Always included |
| BAW Runtime (BPM + Case) | ✅ | `workflow` pattern |
| Application Engine | ✅ | `application` pattern |
| App Designer | ✅ | `app_designer` optional component |
| IBM FileNet CPE | ✅ | Required by BAW runtime |
| IBM Content Navigator | ✅ | Task UI for BAW |
| GraphQL API | ✅ | Enabled |
| Business Automation Insights (BAI) | ✅ | `bai` optional component |
| Kafka | ✅ | `kafka` optional component |
| OpenSearch | ✅ | `opensearch` optional component |
| Workflow AI Assistant | ✅ | `workflow_assistant` optional component |
| Workplace AI Assistant | ✅ | `workplace_assistant` optional component |
| BAW Authoring / BAS | ❌ | Runtime only |
| ADS | ❌ | Not included |
| ODM | ❌ | Not included |

---

## Configuration Details

### Deployment Patterns & Optional Components

```
sc_deployment_patterns: foundation,workflow,application
sc_optional_components: bai,kafka,opensearch,app_designer,workflow_assistant,workplace_assistant
Namespace: --THIS SECTION MUST BE UPDATED--
```

### Key CR Sections

All sections from `cp4ba-cr-ref-baw-bai.yaml` plus:
- `application_engine_configuration` – AE instance for running App Designer applications

### Databases Required (PostgreSQL)

Same as BAW+BAI recipe, plus:

| Database | Purpose |
|---|---|
| `*_aaedb` | Application Engine database |
| `*_aeos` | App Engine Object Store |
| `*_appdb` | Application database |

---

## Installation Commands

--THIS SECTION MUST BE UPDATED--

> See `recipe-name-baw-bai.md` for the base BAW runtime installation command. Use the template `cp4ba-cr-ref-baw-bai-ae.yaml` with an appropriate configuration file that sets `application` in patterns and `app_designer` in optional components.

```bash
_VV=26.0.2
_KK=26.0.0-IF002
_PTC=/home/$USER/cp4ba-projects/cp4ba-installations/configs26
# --THIS SECTION MUST BE UPDATED-- : provide the correct config file name
CONFIG_FILE=${_PTC}/env1-runtime-baw-bai.properties
./cp4ba-one-shot-installation.sh -c ${CONFIG_FILE} -m -v ${_VV} -k ${_KK}
```

### Parameters Reference

| Parameter | Description |
|---|---|
| `-c ${CONFIG_FILE}` | Path to the properties configuration file |
| `-m` | Install a fresh CP4BA Case Package Manager |
| `-v ${_VV}` | CP4BA version (e.g., `26.0.2`) |
| `-k ${_KK}` | cert-kubernetes version (e.g., `26.0.0-IF002`) |

---

## Notes

- This recipe adds `application` pattern and `app_designer` to the BAW+BAI runtime recipe.
- The `cp4ba-cr-ref-baw-bai-ae.yaml` template references both BAW runtime and Application Engine parameters.
- For the authoring counterpart with Application Engine, see `recipe-name-authoring-baw-bai-ae.md`.
