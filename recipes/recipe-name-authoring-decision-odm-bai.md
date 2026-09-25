# Recipe: Authoring Decision ODM + BAI (ODM Authoring with Business Automation Insights)

**Created:** 2025-07-14T00:00:00Z  
**Template:** `cp4ba-installations/templates26/cp4ba-cr-ref-authoring-decision-odm-bai.yaml`  
**Config file:** `cp4ba-installations/configs26/env1-authoring-odm-bai.properties`

---

## Description

The **Authoring Decision ODM + BAI** recipe extends the ODM authoring recipe by adding Business Automation Insights (BAI) for rule decision analytics. It provides:

- **ODM Decision Center** – business rule management and authoring
- **ODM Rule Execution Server (RES)** – rule execution runtime
- **ODM Decision Runner** – testing and simulation
- **BAI** – decision event analytics and BPC dashboards
- **Kafka** and **OpenSearch** – event streaming and indexing

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
| BAW Authoring | ❌ | Not included |
| Application Engine | ❌ | Not included |

---

## Configuration Details

### Deployment Patterns & Optional Components

```
sc_deployment_patterns: foundation,decisions
sc_optional_components: bai
Namespace: cp4ba-odm-bai-auth
```

### BAI for ODM

```yaml
bai_configuration:
  odm:
    install: true    # ODM rule execution events collected by BAI
  bpmn:
    install: false
  icm:
    install: false
  ads:
    install: false
```

### Databases Required (PostgreSQL)

| Database | Purpose |
|---|---|
| `odm_bai_auth_odmdb` | ODM Decision Center + RES |
| `odm_bai_auth_icn` | IBM Content Navigator |
| `odm_bai_auth_gcd` | Global Configuration DB |

---

## Installation Commands

```bash
_VV=26.0.1
_KK=26.0.0-IF001
_PTC=/home/$USER/cp4ba-projects/cp4ba-installations/configs26
CONFIG_FILE=${_PTC}/env1-authoring-odm-bai.properties
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

- The key difference from `recipe-name-authoring-decision-odm.md` is the addition of `bai` to optional components.
- BAI ODM emitter collects rule execution events for BPC dashboards.
- Kafka and OpenSearch are automatically included when `bai` is in optional components.
- For ODM runtime-only with BAI, use `recipe-name-decision-odm-bai.md`.
