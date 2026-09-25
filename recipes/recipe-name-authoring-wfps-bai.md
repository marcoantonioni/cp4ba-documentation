# Recipe: Authoring WFPS + BAI (Workflow Process Service Authoring with Analytics)

**Created:** 2025-07-14T00:00:00Z  
**Template:** `cp4ba-installations/templates26/cp4ba-cr-ref-authoring-wfps-bai.yaml`  
**Config file:** `cp4ba-installations/configs26/env1-authoring-wfps-pfs-bai.properties`

---

## Description

The **Authoring WFPS + BAI** recipe deploys IBM Workflow Process Service (WFPS) in authoring mode with Business Automation Insights and Kafka/OpenSearch for analytics.

WFPS is a cloud-native, lightweight workflow runtime for containerised process applications. Unlike BAW, it does not include Case Management or FileNet Content Manager. It is designed for microservices-style process applications.

This recipe includes:
- **WFPS Authoring** – cloud-native process authoring environment
- **Business Automation Insights (BAI)** – streaming analytics for WFPS events
- **Kafka** – event streaming infrastructure
- **OpenSearch** – analytics data store
- **Business Automation Studio (BAS)** – integrated authoring environment
- **IBM FileNet CPE + Navigator** – required by the foundation BAW pattern

> **Note:** Despite using the `foundation,workflow-process-service` pattern, the template still deploys the standard shared configuration. BAW-specific components such as CPE and ICN are included in the foundation layer.

---

## CP4BA Capabilities

| Capability | Enabled | Notes |
|---|---|---|
| Foundation / CPFS | ✅ | Always included |
| WFPS Authoring | ✅ | `wfps_authoring` optional component |
| Business Automation Studio (BAS) | ✅ | Included via `bastudio_configuration` |
| IBM FileNet CPE | ✅ | Part of shared foundation |
| IBM Content Navigator | ✅ | Part of shared foundation |
| Business Automation Insights (BAI) | ✅ | `bai` optional component |
| Kafka | ✅ | `kafka` optional component |
| OpenSearch | ✅ | `opensearch` optional component |
| Process Federation Server (PFS) | ✅ | Included via config (`opensearch,kafka,bai,pfs`) |
| BAW Authoring | ❌ | Not included (WFPS, not BAW) |
| ADS | ❌ | Not included |
| ODM | ❌ | Not included |
| Application Engine | ❌ | Not included |
| Case Management | ❌ | Not included |

---

## Configuration Details

### Deployment Patterns & Optional Components

```
sc_deployment_patterns: foundation,workflow-process-service
sc_optional_components: wfps_authoring,bai,kafka,opensearch
Namespace: cp4ba-wfps-pfs-bai-auth
```

### Key CR Sections

- `shared_configuration` – FNCM + BAW licenses, storage, image repo, IAM
- `ldap_configuration` – OpenLDAP with SCIM
- `datasource_configuration` – PostgreSQL connections
- `bastudio_configuration` – BAS with WFPS DB
- `workflow_authoring_configuration` – WFPS authoring settings
- `bai_configuration` – BAI emitters for WFPS events
- `ecm_configuration` – CPE + GraphQL
- `navigator_configuration` – IBM Content Navigator

### Key Variables

| Variable | Value |
|---|---|
| `CP4BA_INST_ENV` | `wfps-pfs-bai-auth` |
| `CP4BA_INST_NAMESPACE` | `cp4ba-wfps-pfs-bai-auth` |
| `CP4BA_INST_DEPL_PATTERNS` | `foundation,workflow-process-service` |
| `CP4BA_INST_OPT_COMPONENTS` | `wfps_authoring,bai,kafka,opensearch` |
| `CP4BA_INST_DEPL_PROFILE_SIZE` | `small` |
| `CP4BA_INST_DB` | `true` |
| `CP4BA_INST_LDAP` | `true` |
| `CP4BA_INST_IAM` | `true` |

### Custom XML for WFPS

```bash
CP4BA_INST_CUSTOM_XML_WFPS_CR_NAME_1="wfps-demo-1"
CP4BA_INST_LIBERTY_CUSTOM_XML_SECRET_NAME_WFPS_1="my-liberty-custom-xml-secret-1"
CP4BA_INST_LOMBARDI_CUSTOM_XML_SECRET_NAME_WFPS_1="my-lombardi-custom-xml-secret-1"
```

---

## Installation Commands

### Standard

```bash
_VV=26.0.2
_KK=26.0.0-IF002
_PTC=/home/$USER/cp4ba-projects/cp4ba-installations/configs26
CONFIG_FILE=${_PTC}/env1-authoring-wfps-pfs-bai.properties
./cp4ba-one-shot-installation.sh -c ${CONFIG_FILE} -m -v ${_VV} -k ${_KK}
```

*Last tested: 20260806*

### Parameters Reference

| Parameter | Description |
|---|---|
| `-c ${CONFIG_FILE}` | Path to the properties configuration file |
| `-m` | Install a fresh CP4BA Case Package Manager |
| `-v ${_VV}` | CP4BA version (e.g., `26.0.2`) |
| `-k ${_KK}` | cert-kubernetes version (e.g., `26.0.0-IF002`) |

---

## Notes

- WFPS is suited for cloud-native, stateless process applications that do not require Case Management.
- WFPS Authoring runs processes through BAS tooling but uses a simplified process engine.
- This recipe uses the `workflow-process-service` pattern, not the `workflow` pattern used by BAW.
- PFS (Process Federation Server) can be used alongside WFPS for federated task lists across multiple WFPS instances.
- For a runtime-only Foundation + PFS setup (without WFPS authoring), use the Foundation recipe with `env1-runtime-os-bai-pfs.properties`.
