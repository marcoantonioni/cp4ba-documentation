# Recipe: RPA (Robotic Process Automation – Unmanaged)

**Created:** 2025-07-14T00:00:00Z  
**Template:** `cp4ba-installations/templates26/cp4ba-cr-ref-rpa.yaml`  
**Config file:** `cp4ba-installations/configs26/env1-runtime-rpa.properties`

---

## Description

The **RPA** recipe deploys IBM Robotic Process Automation in **unmanaged mode** into a namespace that already contains (or will contain) CP4BA Foundation services.

The `ICP4ACluster` CR for this recipe uses **only the `foundation` pattern** with no optional components. Its purpose is to establish the shared platform context (IAM, Zen UI, LDAP) that the IBM RPA Operator then integrates with for Single Sign-On.

> **Important:** The RPA software itself (IBM MQ Operator + IBM RPA Operator + RpaServer CR + Microsoft SQL Server) is deployed **separately** by the RPA operator. The `ICP4ACluster` CR in this recipe provides only the IAM/Zen/CPFS foundation layer.

Two RPA deployment modes are available:

| Mode | Config file | Script |
|---|---|---|
| **Unmanaged** (this recipe) | `env1-runtime-rpa.properties` | `cp4ba-one-shot-installation.sh` |
| Managed (separate) | `env1-runtime-rpa-managed.properties` | `cp4ba-install-rpa.sh` (not covered here) |

---

## CP4BA Capabilities

| Capability | Enabled | Notes |
|---|---|---|
| Foundation / CPFS | ✅ | IAM, Zen UI, LDAP, Navigator |
| IBM RPA Operator | ✅ | Deployed by RPA operator (separate) |
| IBM MQ | ✅ | Required by RPA (separate operator) |
| BAW Authoring | ❌ | Not included |
| ADS | ❌ | Not included |
| ODM | ❌ | Not included |
| BAI | ❌ | Not included |
| Application Engine | ❌ | Not included |

---

## Configuration Details

### Deployment Patterns & Optional Components

```
sc_deployment_patterns: foundation
sc_optional_components: (none)
Namespace: cp4ba-rpa
```

### Shared Configuration

```yaml
shared_configuration:
  sc_deployment_type: "production"
  sc_deployment_platform: "OCP"
  sc_deployment_profile_size: "small"
  sc_deployment_license: "production"
  sc_image_repository: cp.icr.io
  storage_configuration:
    sc_dynamic_storage_classname: "ocs-external-storagecluster-cephfs"
    sc_block_storage_classname: "ocs-external-storagecluster-ceph-rbd"
  sc_iam:
    default_admin_username: "cpadmin"
  encryption_key_secret: ibm-iaws-shared-key-secret
  root_ca_secret: icp4a-root-ca
```

### Key Variables

| Variable | Value |
|---|---|
| `CP4BA_INST_ENV` | `rpa` |
| `CP4BA_INST_NAMESPACE` | `cp4ba-rpa` |
| `CP4BA_INST_DEPL_PATTERNS` | `foundation` |
| `CP4BA_INST_OPT_COMPONENTS` | _(empty)_ |
| `CP4BA_INST_DEPL_PROFILE_SIZE` | `small` |
| `CP4BA_INST_DB` | `true` |
| `CP4BA_INST_LDAP` | `true` |
| `CP4BA_INST_IAM` | `true` |

---

## Installation Commands

### Unmanaged RPA Deployment

> Must be deployed into a namespace with Foundation services already present (or deployed simultaneously).

```bash
_VV=26.0.2
_KK=26.0.0-IF002
_PTC=/home/$USER/cp4ba-projects/cp4ba-installations/configs26
CONFIG_FILE=${_PTC}/env1-runtime-rpa.properties
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

- The `cp4ba-cr-ref-rpa.yaml` template is structurally identical to `cp4ba-cr-ref-foundation.yaml`.
- For the **managed** RPA deployment, use `env1-runtime-rpa-managed.properties` with the `cp4ba-install-rpa.sh` script.
- The IBM RPA Operator requires **Microsoft SQL Server** with 5 databases: `address`, `automation`, `knowledge`, `wordnet`, `audit`.
- IBM MQ Operator channel: `v3.9`; IBM RPA Operator channel: `v3.3`.
