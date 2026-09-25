# Recipe: Authoring Decision ODM (ODM Authoring – no BAI)

**Created:** 2025-07-14T00:00:00Z  
**Template:** `cp4ba-installations/templates26/cp4ba-cr-ref-authoring-decision-odm.yaml`  
**Config file:** `cp4ba-installations/configs26/env1-authoring-odm.properties`

---

## Description

The **Authoring Decision ODM** recipe deploys IBM Operational Decision Manager (ODM) in a full authoring and runtime configuration:

- **Decision Center** – business rule management and authoring
- **Rule Execution Server (RES)** – rule execution runtime with management console
- **Decision Runner** – testing and simulation environment for rule projects

This recipe addresses teams who use traditional business rule management systems (BRMS) for rule-based decision automation.

---

## CP4BA Capabilities

| Capability | Enabled | Notes |
|---|---|---|
| Foundation / CPFS | ✅ | Always included |
| ODM Decision Center | ✅ | Part of `decisions` pattern |
| ODM Rule Execution Server | ✅ | Part of `decisions` pattern |
| ODM Decision Runner | ✅ | Part of `decisions` pattern |
| Business Automation Insights (BAI) | ❌ | Not included (use authoring-decision-odm-bai) |
| ADS | ❌ | Not included |
| BAW Authoring | ❌ | Not included |
| Application Engine | ❌ | Not included |

---

## Configuration Details

### Deployment Patterns & Optional Components

```
sc_deployment_patterns: foundation,decisions
sc_optional_components: (none)
Namespace: cp4ba-odm-auth
```

> **Note:** For ODM, no optional components are needed — all ODM components (Decision Center, RES, Decision Runner) are enabled by the `decisions` pattern itself.

### Key CR Sections

- `shared_configuration` – deployment license, storage, IAM, image repo
- `ldap_configuration` – OpenLDAP with SCIM
- `datasource_configuration` – PostgreSQL connection for ODM
- `odm_configuration` – Decision Center, RES, Decision Runner settings

### ODM Configuration

```yaml
odm_configuration:
  decisionCenter:
    enabled: true
    # DB: odm_auth_odmdb
  decisionServerRuntime:
    enabled: true
    # DB: odm_auth_odmdb (same DB)
  decisionServerConsole:
    enabled: true
  decisionRunner:
    enabled: true
```

### Databases Required (PostgreSQL)

| Database | Purpose |
|---|---|
| `odm_auth_odmdb` | ODM Decision Center + RES (shared) |
| `odm_auth_icn` | IBM Content Navigator (foundation) |
| `odm_auth_gcd` | Global Configuration DB (foundation) |

### Storage

```
CP4BA_INST_SC_FILE: ocs-external-storagecluster-cephfs
CP4BA_INST_SC_BLOCK: ocs-external-storagecluster-ceph-rbd
```

---

## Installation Commands

```bash
_VV=26.0.1
_KK=26.0.0-IF001
_PTC=/home/$USER/cp4ba-projects/cp4ba-installations/configs26
CONFIG_FILE=${_PTC}/env1-authoring-odm.properties
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

- ODM uses the `decisions` pattern (not `decisions_ads` which is for the newer ADS platform).
- All ODM components (Decision Center, RES, Runner) are deployed together in the `decisions` pattern; no optional components are required.
- ODM is the traditional BRMS (Business Rules Management System) platform, while ADS is the newer AI-augmented decision platform.
- For ODM with BAI analytics, use `recipe-name-authoring-decision-odm-bai.md`.
- For ODM runtime only, use `recipe-name-decision-odm.md`.
