# Recipe: Decision ODM (ODM Runtime only)

**Created:** 2025-07-14T00:00:00Z  
**Template:** `cp4ba-installations/templates26/cp4ba-cr-ref-decision-odm.yaml`  
**Config file:** `cp4ba-installations/configs26/env1-runtime-odm.properties`

---

## Description

The **Decision ODM** recipe deploys IBM Operational Decision Manager (ODM) in a full deployment configuration including Decision Center, Rule Execution Server, and Decision Runner — without Business Automation Insights.

This recipe is for production rule execution environments where BAI analytics are not required.

---

## CP4BA Capabilities

| Capability | Enabled | Notes |
|---|---|---|
| Foundation / CPFS | ✅ | Always included |
| ODM Decision Center | ✅ | Part of `decisions` pattern |
| ODM Rule Execution Server | ✅ | Part of `decisions` pattern |
| ODM Decision Runner | ✅ | Part of `decisions` pattern |
| Business Automation Insights (BAI) | ❌ | Not included |
| ADS | ❌ | Not included |
| BAW | ❌ | Not included |

---

## Configuration Details

### Deployment Patterns & Optional Components

```
sc_deployment_patterns: foundation,decisions
sc_optional_components: (none)
Namespace: cp4ba-odm
```

### Key Variables

| Variable | Value |
|---|---|
| `CP4BA_INST_ENV` | `odm` |
| `CP4BA_INST_NAMESPACE` | `cp4ba-odm` |
| `CP4BA_INST_DEPL_PATTERNS` | `foundation,decisions` |
| `CP4BA_INST_OPT_COMPONENTS` | _(empty)_ |

### Databases Required (PostgreSQL)

| Database | Purpose |
|---|---|
| `odm_odmdb` | ODM Decision Center + RES |
| `odm_icn` | IBM Content Navigator (foundation) |
| `odm_gcd` | Global Configuration DB (foundation) |

---

## Installation Commands

```bash
_VV=26.0.1
_KK=26.0.0-IF001
_PTC=/home/$USER/cp4ba-projects/cp4ba-installations/configs26
CONFIG_FILE=${_PTC}/env1-runtime-odm.properties
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

- For ODM with BAI analytics, use `recipe-name-decision-odm-bai.md`.
- For ODM in authoring mode, use `recipe-name-authoring-decision-odm.md`.
- All ODM components are included with the `decisions` pattern; no optional components needed.
