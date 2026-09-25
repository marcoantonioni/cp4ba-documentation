# Recipe: Decision ODM + BAI (ODM Runtime with Business Automation Insights)

**Created:** 2025-07-14T00:00:00Z  
**Template:** `cp4ba-installations/templates26/cp4ba-cr-ref-decision-odm-bai.yaml`  
**Config file:** `cp4ba-installations/configs26/env1-runtime-odm-bai.properties`

---

## Description

The **Decision ODM + BAI** recipe deploys IBM Operational Decision Manager (ODM) in runtime mode with **Business Automation Insights (BAI)** for rule decision analytics. It provides:

- **ODM Decision Center** – business rule management
- **ODM Rule Execution Server** – rule execution runtime
- **ODM Decision Runner** – testing and simulation
- **BAI** – rule execution event analytics and BPC dashboards
- **Kafka** and **OpenSearch** – event streaming and indexing

This is the production ODM runtime with full analytics observability.

---

## CP4BA Capabilities

| Capability | Enabled | Notes |
|---|---|---|
| Foundation / CPFS | ✅ | Always included |
| ODM Decision Center | ✅ | Part of `decisions` pattern |
| ODM Rule Execution Server | ✅ | Part of `decisions` pattern |
| ODM Decision Runner | ✅ | Part of `decisions` pattern |
| Business Automation Insights (BAI) | ✅ | `bai` optional component |
| Kafka | ✅ | Required by BAI |
| OpenSearch | ✅ | Required by BAI |
| ADS | ❌ | Not included |
| BAW | ❌ | Not included |

---

## Configuration Details

### Deployment Patterns & Optional Components

```
sc_deployment_patterns: foundation,decisions
sc_optional_components: bai
Namespace: cp4ba-odm-bai
```

### Key Variables

| Variable | Value |
|---|---|
| `CP4BA_INST_ENV` | `odm-bai` |
| `CP4BA_INST_NAMESPACE` | `cp4ba-odm-bai` |
| `CP4BA_INST_DEPL_PATTERNS` | `foundation,decisions` |
| `CP4BA_INST_OPT_COMPONENTS` | `bai` |

### BAI for ODM

```yaml
bai_configuration:
  odm:
    install: true    # ODM rule execution events
  bpmn:
    install: false
  ads:
    install: false
```

### Databases Required (PostgreSQL)

| Database | Purpose |
|---|---|
| `odm_bai_odmdb` | ODM Decision Center + RES |
| `odm_bai_icn` | IBM Content Navigator |
| `odm_bai_gcd` | Global Configuration DB |

---

## Installation Commands

```bash
_VV=26.0.1
_KK=26.0.0-IF001
_PTC=/home/$USER/cp4ba-projects/cp4ba-installations/configs26
CONFIG_FILE=${_PTC}/env1-runtime-odm-bai.properties
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

- Key difference from `recipe-name-decision-odm.md`: adds `bai` to optional components.
- BAI collects ODM rule execution events for BPC dashboards.
- Kafka and OpenSearch are automatically included when `bai` is in optional components.
- For authoring mode with BAI, use `recipe-name-authoring-decision-odm-bai.md`.
