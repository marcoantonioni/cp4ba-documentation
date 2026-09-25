# Recipe: BAW + BAI (BAW Runtime with Business Automation Insights)

**Created:** 2025-07-14T00:00:00Z  
**Template:** `cp4ba-installations/templates26/cp4ba-cr-ref-baw-bai.yaml`  
**Config file:** `cp4ba-installations/configs26/env1-runtime-baw-bai.properties`

---

## Description

The **BAW + BAI** recipe deploys IBM Business Automation Workflow (BAW) in **runtime mode** with Business Automation Insights. This is the production execution environment for BAW processes and cases deployed from a BAW Authoring environment.

This recipe provides:
- **BAW Runtime** – process and case execution engine
- **IBM FileNet CPE** – content repository for case documents
- **IBM Content Navigator** – task UI and content browser
- **BAI** – real-time process and case analytics
- **Kafka** and **OpenSearch** – event streaming and indexing
- **Workplace AI Assistant** – when GenAI is enabled

> **Important:** This is a RUNTIME recipe, not an authoring recipe. Processes and cases must be deployed from a BAW Authoring environment. No `baw_authoring` or `bas` optional component is included.

---

## CP4BA Capabilities

| Capability | Enabled | Notes |
|---|---|---|
| Foundation / CPFS | ✅ | Always included |
| BAW Runtime (BPM + Case) | ✅ | `workflow` pattern (without `baw_authoring`) |
| IBM FileNet CPE | ✅ | Required by BAW runtime |
| IBM Content Navigator | ✅ | Task UI for BAW |
| GraphQL API | ✅ | Enabled |
| Business Automation Insights (BAI) | ✅ | `bai` optional component |
| Kafka | ✅ | `kafka` optional component |
| OpenSearch | ✅ | `opensearch` optional component |
| Workflow AI Assistant | ✅ | `workflow_assistant` optional component |
| Workplace AI Assistant | ✅ | `workplace_assistant` optional component |
| BAW Authoring / BAS | ❌ | Not included (runtime only) |
| ADS | ❌ | Not included |
| ODM | ❌ | Not included |
| Application Engine | ❌ | Not included |

---

## Configuration Details

### Deployment Patterns & Optional Components

```
sc_deployment_patterns: foundation,workflow
sc_optional_components: bai,kafka,opensearch,workflow_assistant,workplace_assistant
Namespace: cp4ba-baw-bai-prod
```

### Key Difference from BAW Authoring Recipe

| Feature | BAW Authoring | BAW Runtime |
|---|---|---|
| `baw_authoring` component | ✅ | ❌ |
| `bas` component | ✅ | ❌ |
| `bastudio_configuration` section | Present | Absent |
| Process/case authoring UI | ✅ | ❌ |
| Process/case execution | ✅ | ✅ |
| BAI analytics | Optional | ✅ |
| PFS federation | Optional | Optional |

### Key CR Sections

- `shared_configuration` – licenses (FNCM + BAW), storage, IAM, image repo
- `ldap_configuration` – OpenLDAP with SCIM
- `datasource_configuration` – PostgreSQL connections for GCD, ICN, DOCS, DOS, TOS, CONTENT
- `initialize_configuration` – FileNet domain, object stores, ICN
- `bai_configuration` – BAI event emitters, Flink, BPC
- `ecm_configuration` – CPE + GraphQL
- `navigator_configuration` – IBM Content Navigator

### Databases Required (PostgreSQL)

| Database | Purpose |
|---|---|
| `baw_bai_prod_gcd` | Global Configuration Database |
| `baw_bai_prod_icn` | IBM Content Navigator |
| `baw_bai_prod_bawdocs` | BAW Documents object store |
| `baw_bai_prod_bawdos` | BAW Design object store |
| `baw_bai_prod_bawtos` | BAW Target object store |
| `baw_bai_prod_content` | Content object store |

---

## Installation Commands

```bash
_VV=26.0.2
_KK=26.0.0-IF002
_PTC=/home/$USER/cp4ba-projects/cp4ba-installations/configs26
CONFIG_FILE=${_PTC}/env1-runtime-baw-bai.properties
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

- This is the production runtime recipe for BAW. Processes and cases are deployed from `recipe-name-authoring-baw*.md` environments.
- The `workflow_assistant` and `workplace_assistant` are AI assistants for the Workplace UI (not authoring).
- BAI with Kafka and OpenSearch provides real-time process monitoring and BPC dashboards.
- For a runtime environment with Application Engine, use `recipe-name-baw-bai-ae.md`.
