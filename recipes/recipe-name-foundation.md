# Recipe: Foundation (OpenSearch / BAI / PFS)

**Created:** 2025-07-14T00:00:00Z  
**Template:** `cp4ba-installations/templates26/cp4ba-cr-ref-foundation.yaml`  
**Config files:**
- `cp4ba-installations/configs26/env1-runtime-opensearch-foundation.properties`
- `cp4ba-installations/configs26/env1-runtime-os-bai-pfs.properties`

---

## Description

The **Foundation** recipe deploys only the CP4BA foundation pattern, providing the shared platform services (CPFS, IAM, Zen UI, Resource Registry) without any workflow, decision, or content capabilities.

Two configurations exist:
1. **OpenSearch Foundation** – Foundation + OpenSearch only (minimal footprint for shared search infrastructure)
2. **OS+BAI+PFS** – Foundation + OpenSearch + Kafka + BAI + PFS (shared infrastructure for WFPS runtime federation with analytics)

This recipe is the base building block used when:
- Setting up a shared infrastructure namespace for WFPS runtime environments
- Deploying Process Federation Server as a standalone shared service
- Testing or validating the CP4BA platform installation itself

---

## CP4BA Capabilities

| Capability | Enabled | Notes |
|---|---|---|
| Foundation / CPFS | ✅ | Always included |
| IBM Content Navigator (ICN) | ✅ | Included with foundation |
| OpenSearch | ✅ | Both configurations |
| Kafka | ✅ | OS+BAI+PFS config only |
| Business Automation Insights (BAI) | ✅ | OS+BAI+PFS config only |
| Process Federation Server (PFS) | ✅ | OS+BAI+PFS config only |
| BAW Authoring | ❌ | Not included |
| ADS | ❌ | Not included |
| ODM | ❌ | Not included |
| Application Engine | ❌ | Not included |
| RPA | ❌ | Not included |

---

## Configuration Details

### Deployment Patterns & Optional Components

**Config 1 – OpenSearch Foundation:**
```
sc_deployment_patterns: foundation
sc_optional_components: opensearch
Namespace: cp4ba-opensearch-prod
```

**Config 2 – OS+BAI+PFS:**
```
sc_deployment_patterns: foundation
sc_optional_components: opensearch,kafka,bai,pfs
Namespace: cp4ba-os-bai-pfs-prod
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

| Variable | Value (OpenSearch) | Value (OS+BAI+PFS) |
|---|---|---|
| `CP4BA_INST_ENV` | `opensearch-prod` | `os-bai-pfs-prod` |
| `CP4BA_INST_NAMESPACE` | `cp4ba-opensearch-prod` | `cp4ba-os-bai-pfs-prod` |
| `CP4BA_INST_DEPL_PATTERNS` | `foundation` | `foundation` |
| `CP4BA_INST_OPT_COMPONENTS` | `opensearch` | `opensearch,kafka,bai,pfs` |
| `CP4BA_INST_DEPL_PROFILE_SIZE` | `small` | `small` |
| `CP4BA_INST_DB` | `true` | `true` |
| `CP4BA_INST_LDAP` | `true` | `true` |
| `CP4BA_INST_IAM` | `true` | `true` |

---

## Installation Commands

### OpenSearch Foundation

```bash
_VV=26.0.2
_KK=26.0.0-IF002
_PTC=/home/$USER/cp4ba-projects/cp4ba-installations/configs26
CONFIG_FILE=${_PTC}/env1-runtime-opensearch-foundation.properties
./cp4ba-one-shot-installation.sh -c ${CONFIG_FILE} -m -v ${_VV} -k ${_KK}
```

*Last tested: 20260717*

### OS + BAI + PFS (Foundation for WfPS federation)

```bash
_VV=26.0.2
_KK=26.0.0-IF002
_PTC=/home/$USER/cp4ba-projects/cp4ba-installations/configs26
CONFIG_FILE=${_PTC}/env1-runtime-os-bai-pfs.properties
./cp4ba-one-shot-installation.sh -c ${CONFIG_FILE} -m -v ${_VV} -k ${_KK}
```

*Last tested: 20260717*

### Parameters Reference

| Parameter | Description |
|---|---|
| `-c ${CONFIG_FILE}` | Path to the properties configuration file |
| `-m` | Install a fresh CP4BA Case Package Manager |
| `-v ${_VV}` | CP4BA version (e.g., `26.0.2`) |
| `-k ${_KK}` | cert-kubernetes version (e.g., `26.0.0-IF002`) |

---

## Notes

- This recipe does **not** deploy BAW, ADS, ODM, FNCM, or Application Engine.
- The `foundation` pattern always includes CPFS (IBM Cloud Pak Foundational Services), IAM, Zen UI, and IBM Content Navigator.
- The OS+BAI+PFS variant is typically the shared infrastructure namespace used when deploying WFPS runtime instances in separate namespaces, providing federated task list (PFS) and analytics (BAI).
- LDAP and PostgreSQL are deployed in the same namespace as CP4BA (single-namespace isolation model).
