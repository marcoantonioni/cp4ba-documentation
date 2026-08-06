# CP4BA Projects Configuration Analysis — Knowledge Base

> **Version**: IBM Cloud Pak for Business Automation 26.0.0  
> **Target platform**: OpenShift Container Platform (OCP) — containerized deployments only  
> **Source repositories**:
> - `cp4ba-installations` — installation tooling, templates and configs  
> - `references/cp4ba-yamls` — IBM official FC (Feature Complete) reference CRs  

---

## Table of Contents

1. [CP4BA Overview](#1-cp4ba-overview)
2. [Component Glossary](#2-component-glossary)
3. [Template–Config Relationship](#3-templateconfig-relationship)
4. [Shared Configuration Concepts](#4-shared-configuration-concepts)
5. [LDAP / IAM Configuration](#5-ldap--iam-configuration)
6. [Database Configuration](#6-database-configuration)
7. [CP4BA Capability Patterns and Optional Components](#7-cp4ba-capability-patterns-and-optional-components)
8. [Reference FC Templates (`./references/cp4ba-yamls`)](#8-reference-fc-templates-referencescp4ba-yamls)
9. [Recipes — Templates26 Overview](#9-recipes--templates26-overview)
10. [Installation Command — `cp4ba-one-shot-installation.sh`](#10-installation-command--cp4ba-one-shot-installationsh)
11. [Supporting Infrastructure](#11-supporting-infrastructure)
12. [Storage Configuration](#12-storage-configuration)
13. [Network Policy Support](#13-network-policy-support)
14. [GenAI / watsonx Integration](#14-genai--watsonx-integration)

---

## 1. CP4BA Overview

IBM Cloud Pak for Business Automation (CP4BA) is a modular suite of business automation capabilities deployed on OpenShift Container Platform via a single Kubernetes Custom Resource (CR) of kind `ICP4ACluster`, API version `icp4a.ibm.com/v1`.

The deployment is driven by the `ibm-cp4a-operator` operator, installed via an OLM (Operator Lifecycle Manager) catalog. A single CR named `icp4adeploy` (by convention) drives the entire lifecycle of all enabled capabilities.

Key principles:
- **Patterns** declare the set of top-level capability groups to enable (e.g. `foundation`, `workflow`, `application`).
- **Optional components** declare fine-grained features within those patterns (e.g. `baw_authoring`, `bai`, `pfs`, `opensearch`, `kafka`).
- **Profile sizes** (`small`, `medium`, `large`) control resource allocation for all components at once.
- Operator, catalogs and CP4BA runtime all co-reside in the same namespace in the `private catalog` (single namespace) deployment model used by these recipes.

---

## 2. Component Glossary

| Acronym | Full Name | Description |
|---------|-----------|-------------|
| BAW | Business Automation Workflow | Full workflow runtime with BPM + Case (IBM ECM) |
| BAS | Business Automation Studio | Authoring environment used to design BAW applications |
| BAI | Business Automation Insights | Event processing and analytics (Flink, OpenSearch, dashboards) |
| BAML | Business Automation Machine Learning | AI/ML workforce insights, intelligent task prioritization |
| BPC | Business Performance Center | Part of BAI; workforce dashboards |
| CPE | Content Platform Engine | FileNet Content Manager document storage engine |
| ICN / BAN | IBM Content Navigator | Enterprise document management UI |
| GraphQL | FileNet GraphQL API | REST/GraphQL gateway to CPE |
| CMIS | Content Management Interoperability Services | CMIS protocol gateway to CPE |
| AE | Application Engine | Runs deployed BAW applications (Workplace, Task views) |
| PFS | Process Federation Server | Federates multiple BAW servers into a single process portal |
| WfPS | Workflow Process Service | Lightweight workflow runtime (no Case, no ECM) |
| WFPS Authoring | WfPS Authoring | Authoring mode of WfPS |
| CPFS | Cloud Pak Foundational Services | Common services: IAM, Zen UI, BTS, IM |
| IAM | Identity and Access Management | CPFS identity provider / SSO component |
| Zen | IBM Cloud Pak for Data platform UI | Web UI dashboard for the platform |
| BTS | Business Teams Service | Team management service (CPFS) |
| OS | Object Store | A FileNet Content Manager object store |
| GCD | Global Configuration Database | FileNet CPE global configuration repository |
| TOS | Target Object Store | BAW-specific FileNet object store for case/process documents |
| DOS | Design Object Store | BAW-specific FileNet object store for design artifacts |
| DOCS | Document Object Store | BAW-specific FileNet object store for BAW documents |
| CHOS | Case History Object Store | Object store for case history data |
| GenAI | Generative AI | watsonx.ai integration feature |
| WxO | watsonx Orchestrate | AI-powered automation assistant |
| OpenSearch | OpenSearch | Search/analytics engine replacing Elasticsearch in BAI |
| Kafka | Apache Kafka | Event streaming bus for BAI event pipeline |

---

## 3. Template–Config Relationship

Each recipe is composed of two files:

1. **Template YAML** (`templates26/cp4ba-cr-ref-*.yaml`):  
   A parameterised `ICP4ACluster` CR with shell-style variable placeholders `${VAR_NAME}`.

2. **Config Properties** (`configs26/*.properties`):  
   A Bash-source-able file that exports all variables referenced by the template. The variable `CP4BA_INST_CR_TEMPLATE` points to the template file to use.

At installation time the tooling reads the properties file, substitutes all variables into the template, and applies the resulting CR to OCP.

**Mapping table** (v26):

| Config file | Template file | Env ID |
|-------------|--------------|--------|
| `env1-authoring-baw.properties` | `cp4ba-cr-ref-authoring-baw.yaml` | `baw-auth` |
| `env1-authoring-baw-cicd.properties` | `cp4ba-cr-ref-authoring-baw.yaml` | `baw-auth-cicd` |
| `env1-authoring-baw-genai.properties` | `cp4ba-cr-ref-authoring-baw.yaml` | `baw-ai-auth` |
| `env1-authoring-baw-pfs-genai.properties` | `cp4ba-cr-ref-authoring-baw-bai.yaml` | `baw-ai-pfs-auth` |
| `env1-authoring-baw-bai.properties` | `cp4ba-cr-ref-authoring-baw-bai.yaml` | `baw-bai-auth` |
| `env1-authoring-baw-bai-ae.properties` | `cp4ba-cr-ref-authoring-baw-bai-ae.yaml` | `baw-bai-ae-auth` |
| `env1-authoring-wfps-pfs-bai.properties` | `cp4ba-cr-ref-authoring-wfps-bai.yaml` | `wfps-pfs-bai-auth` |
| `env1-runtime-baw-bai.properties` | `cp4ba-cr-ref-baw-bai.yaml` | `baw-bai-prod` |
| `env1-runtime-os-bai-pfs.properties` | `cp4ba-cr-ref-foundation.yaml` | `os-bai-pfs-prod` |
| `env1-runtime-opensearch-foundation.properties` | `cp4ba-cr-ref-foundation.yaml` | `opensearch-prod` |

---

## 4. Shared Configuration Concepts

Every template contains a `shared_configuration` block that governs platform-wide settings:

| Field | Purpose |
|-------|---------|
| `sc_deployment_license` | License type: `production` or `non-production` |
| `sc_deployment_baw_license` | BAW-specific license: `production`, `non-production`, `user` |
| `sc_deployment_fncm_license` | Content Cortex (FNCM) license: `production`, `non-production`, `concurrent-user`, `authorized-user`, `user` |
| `sc_deployment_patterns` | Comma-separated capability pattern list |
| `sc_optional_components` | Comma-separated optional component list |
| `sc_deployment_type` | Always `Production` in these recipes |
| `sc_deployment_platform` | `OCP` or `ROKS` |
| `sc_deployment_profile_size` | `small`, `medium`, or `large` |
| `sc_image_repository` | Image registry: `cp.icr.io` (IBM Entitled Registry) |
| `sc_iam.default_admin_username` | IAM/Zen platform admin user |
| `sc_content_initialization` | Whether to run content initialization jobs |
| `sc_egress_configuration.sc_restricted_internet_access` | Whether to restrict egress |
| `encryption_key_secret` | Secret name for shared encryption key (`ibm-iaws-shared-key-secret`) |
| `sc_is_multiple_az` | Multi availability zone deployment (false in these recipes) |
| `sc_generate_sample_network_policies` | Auto-generate sample network policies |

---

## 5. LDAP / IAM Configuration

All templates include `ldap_configuration` for connecting to an OpenLDAP server (or compatible):

```yaml
ldap_configuration:
  lc_selected_ldap_type: "Custom"
  lc_ldap_server: "${CP4BA_INST_LDAP_HOST}"
  lc_ldap_port: "${CP4BA_INST_LDAP_PORT}"
  lc_bind_secret: "${CP4BA_INST_LDAP_SECRET}"
  lc_ldap_base_dn: "${CP4BA_INST_LDAP_BASE_DOMAIN}"
  lc_ldap_ssl_enabled: false
  lc_ldap_user_name_attribute: "*:cn"
  lc_ldap_group_membership_search_filter: "(&(cn=%v)(objectclass=groupOfNames))"
  custom:
    lc_user_filter: "(&(cn=%v)(objectclass=person))"
    lc_group_filter: "(&(cn=%v)(objectclass=groupOfNames))"
  scim_configuration_iam:
    user_unique_id_attribute: "uid"
    user_name_attribute: "cn"
    ...
```

The tooling can optionally install an OpenLDAP pod in-namespace (controlled by `CP4BA_INST_LDAP=true`) and onboard users to IAM (controlled by `CP4BA_INST_IAM=true`).

LDAP configuration is driven by `_cfg-production-ldap-domain.properties`.  
IAM/IDP configuration is driven by `_cfg-production-idp.properties`.

---

## 6. Database Configuration

All recipes use **PostgreSQL** (version 16/17/18). The tooling supports:
- Deploying a local PostgreSQL pod in-namespace (`CP4BA_INST_DB=true`)
- Using an external/pre-existing PostgreSQL server (`CP4BA_INST_DB=false`)
- SSL-only connections (`CP4BA_INST_DB_ONLY_SSL=true`)

Key variables:
- `CP4BA_INST_DB_1_SERVER_NAME` — FQDN of the DB service
- `CP4BA_INST_DB_1_TEMPLATE` — SQL script template to create databases
- `CP4BA_INST_DB_SERVER_TYPE` — Always `postgresql` in v26

**Database per component (BAW full)**:

| DB | Purpose |
|----|---------|
| `*_gcd` | FileNet Global Configuration Database |
| `*_icn` | IBM Content Navigator |
| `*_bawdocs` | BAW Document object store |
| `*_bawdos` | BAW Design object store |
| `*_bawtos` | BAW Target (Case) object store |
| `*_content` | Content object store |
| `*_chos` | Case History object store |
| `*_appdb` | BAW Application (Playback Server) |
| `*_aaedb` | Application Engine database |
| `*_aeos` | Application Engine object store |
| `*_awsdb` | Automation Workstream Services |
| `*_awsdocs` | Automation Workstream Services documents |
| `*_os1` | Additional object store |
| `*_baw_1` | Main BAW runtime database |

**Database per component (WfPS)**:  
Uses `db-statements-ref-wfps-authoring.sql` or `db-statements-ref-wfps.sql` which only requires a subset of the above.

---

## 7. CP4BA Capability Patterns and Optional Components

### Patterns

| Pattern | Description |
|---------|-------------|
| `foundation` | Always required. CPFS, Zen, IAM, resource registry |
| `workflow` | BAW runtime or authoring; requires `workflow-process-service` or `workflow` |
| `workflow-process-service` | WfPS lightweight workflow (no ECM/Case) |
| `application` | Application Engine / Workplace |
| `content` | FileNet Content Manager (Content Cortex / CPE + ICN) |
| `decisions` | ODM |
| `decisions_ads` | Automation Decision Services |
| `document_processing` | ADP |

### Optional Components

| Component | Description |
|-----------|-------------|
| `baw_authoring` | Enables BAW Authoring mode (BAS + workflow_authoring) |
| `bas` | Business Automation Studio (authoring UI) |
| `bai` | Business Automation Insights (Flink + OpenSearch analytics) |
| `pfs` | Process Federation Server |
| `kafka` | Apache Kafka for BAI event streaming |
| `opensearch` | OpenSearch for BAI analytics storage |
| `wfps_authoring` | WfPS Authoring mode |
| `app_designer` | Application Designer (part of Application Engine) |
| `ae_data_persistence` | Application Engine data persistence mode |
| `workflow_assistant` | AI Workflow assistant (requires GenAI) |
| `workplace_assistant` | AI Workplace assistant (requires GenAI) |
| `cmis` | CMIS gateway to CPE |
| `css` | Content Search Services |
| `tm` | Task Manager |
| `ier` | IBM Enterprise Records |

---

## 8. Reference FC Templates (`./references/cp4ba-yamls`)

These are the official IBM Feature Complete reference CRs for CP4BA 26.0.0. They serve as the canonical upstream definitions from which local templates are derived.

| File | Topology | Key Patterns |
|------|----------|-------------|
| `ibm_cp4a_cr_production_FC_foundation.yaml` | Foundation only | `foundation` |
| `ibm_cp4a_cr_production_FC_workflow.yaml` | BAW Runtime (production) | `foundation`, `workflow` |
| `ibm_cp4a_cr_production_FC_workflow_authoring.yaml` | BAW Authoring | `foundation`, `workflow` + `baw_authoring` |
| `ibm_cp4a_cr_production_FC_workflow_process_service_authoring.yaml` | WfPS Authoring | `foundation`, `workflow-process-service` + `wfps_authoring` |
| `ibm_cp4a_cr_production_FC_application.yaml` | Application Engine | `foundation`, `application` |
| `ibm_cp4a_cr_production_FC_content.yaml` | Content Cortex (ECM) | `foundation`, `content` |
| `ibm_cp4a_cr_production_FC_process_federation_server.yaml` | PFS only | `foundation` + `pfs` |

These CRs use placeholder values (`<Required>`) and are meant to be customized before use. The local recipes in `templates26/` replace all placeholders with `${VAR_NAME}` variable syntax for automated substitution.

---

## 9. Recipes — Templates26 Overview

Seven templates exist in `cp4ba-installations/templates26/`. Each is described in its own recipe file in `./recipes/`.

### Recipe Summary Table

| Template File | Recipe File | Primary Use Case | Key Capabilities |
|---------------|-------------|-----------------|-----------------|
| `cp4ba-cr-ref-foundation.yaml` | `recipe-name-foundation.md` | Foundation + OpenSearch (optionally + BAI + PFS) | CPFS, Zen, IAM, OpenSearch, optionally BAI/Kafka/PFS |
| `cp4ba-cr-ref-authoring-baw.yaml` | `recipe-name-authoring-baw.md` | BAW Authoring (BPM + Case), no BAI | BAW Authoring, BAS, ECM (CPE+ICN), GenAI-ready |
| `cp4ba-cr-ref-authoring-baw-bai.yaml` | `recipe-name-authoring-baw-bai.md` | BAW Authoring + BAI + PFS | BAW Authoring, BAS, ECM, BAI, Kafka, OpenSearch, PFS |
| `cp4ba-cr-ref-authoring-baw-bai-ae.yaml` | `recipe-name-authoring-baw-bai-ae.md` | BAW Authoring + BAI + Application Engine | BAW Authoring, BAS, ECM, BAI, AE, Kafka, OpenSearch |
| `cp4ba-cr-ref-authoring-wfps-bai.yaml` | `recipe-name-authoring-wfps-bai.md` | WfPS Authoring + BAI + PFS | WfPS Authoring, BAS, BAI, Kafka, OpenSearch, PFS |
| `cp4ba-cr-ref-baw-bai.yaml` | `recipe-name-baw-bai.md` | BAW Runtime + BAI | BAW Runtime, ECM (CPE+ICN), BAI, Kafka, OpenSearch |
| `cp4ba-cr-ref-baw-bai-ae.yaml` | `recipe-name-baw-bai-ae.md` | BAW Runtime + BAI + Application Engine + CMIS | BAW Runtime, ECM, BAI, AE, CMIS, Kafka, OpenSearch |

---

## 10. Installation Command — `cp4ba-one-shot-installation.sh`

Located at `cp4ba-installations/scripts/cp4ba-one-shot-installation.sh`.

This is the **only** installation script to be used for end-to-end deployment. It orchestrates the complete installation lifecycle:

### Prerequisites
- OCP cluster access (`oc` CLI logged in)
- Tools: `curl`, `kubectl`, `podman`, `jq`, `yq`, `openssl`
- Companion repositories cloned alongside `cp4ba-installations`:
  - `cp4ba-logger` (logging framework)
  - `cp4ba-config-tune` (custom XML secrets)
  - `cp4ba-casemanager-setup` or a pre-installed CP4BA Case Package Manager

### Syntax

```bash
./cp4ba-one-shot-installation.sh \
  -c <full-path-to-config-file>        # required: .properties config file
  [-t]                                  # test configuration and exit
  [-p <cert-kubernetes-scripts-folder>] # use pre-installed CP4BA Case Manager
  [-m]                                  # install a fresh CP4BA Case Manager
  [-v <case-package-manager-version>]   # optional: specific CP4BA Case Manager version (e.g. 26.0.0)
  [-k <cert-kubernetes-version>]        # optional: cert-kubernetes version (e.g. 26.0.0)
  [-d <path-to-case-manager-folder>]    # required if -m is set
  [-o]                                  # skip operator installation (reuse existing)
  [-x]                                  # enable trace
```

### Execution flow

1. Sources the config `.properties` file
2. Runs pre-installation hooks (`cp4ba-customPreInstallation.sh`)
3. Validates prerequisites and storage classes
4. Downloads/verifies the CP4BA Case Package Manager (if `-m`)
5. Installs CP4BA operators (`cp4ba-install-operators.sh`)
6. Deploys the environment (`cp4ba-deploy-env.sh`):
   - Creates namespace
   - Installs LDAP (if `CP4BA_INST_LDAP=true`)
   - Creates PostgreSQL databases
   - Creates OCP Secrets
   - Creates PVCs
   - Generates and applies the ICP4ACluster CR
7. Onboards LDAP users to IAM (if `CP4BA_INST_IAM=true`)
8. Configures ZenService TLS certificate
9. Runs post-installation hooks (`cp4ba-customPostInstallation.sh`)

### Typical invocation examples

```bash
# Standard one-shot: install Case Manager automatically
_VV=26.0.0
_KK=26.0.0
_PTC=/home/$USER/cp4ba-projects/cp4ba-installations/configs26
CONFIG_FILE=${_PTC}/env1-authoring-baw.properties
./cp4ba-one-shot-installation.sh -c ${CONFIG_FILE} -m -v ${_VV} -k ${_KK}

# With a pre-installed Case Manager
./cp4ba-one-shot-installation.sh \
  -c ${CONFIG_FILE} \
  -p /your-folder/ibm-cp-automation-26.0.0/ibm-cp-automation/inventory/cp4aOperatorSdk/files/deploy/crs/cert-kubernetes/scripts

# Skip operator installation (operators already present)
./cp4ba-one-shot-installation.sh -c ${CONFIG_FILE} -m -v ${_VV} -k ${_KK} -o

# Test configuration only
./cp4ba-one-shot-installation.sh -c ${CONFIG_FILE} -t

# With GenAI enabled
export CP4BA_INST_GENAI_ENABLED="true"
export CP4BA_INST_GENAI_WX_APIKEY="<your-api-key>"
export CP4BA_INST_GENAI_WX_PRJ_ID="<your-project-id>"
./cp4ba-one-shot-installation.sh -c ${CONFIG_FILE} -m -v ${_VV} -k ${_KK}

# With Git token (CICD recipe)
export CP4BA_INST_GIT_TOKEN="<your-github-token>"
./cp4ba-one-shot-installation.sh -c ${CONFIG_FILE} -m -v ${_VV} -k ${_KK}
```

---

## 11. Supporting Infrastructure

### LDAP (OpenLDAP)

- Provisioned via `cp4ba-idp-ldap` companion repository
- Configured by `_cfg-production-ldap-domain.properties`
- LDIF populated from `_cfg-production-ldap-domain.ldif`
- User onboarding to IAM via `_cfg-production-idp.properties`
- phpLDAPadmin optionally enabled (`CP4BA_INST_LDAP_USE_PHPADMIN`)

### PostgreSQL

- Provisioned as a Kubernetes StatefulSet in the same namespace
- SSL-only endpoint available via separate service (`-ssl-rw`)
- Configuration templates:
  - `postgresql_nossl.conf` / `postgresql_ssl.conf` — tuning
  - `pg_hba.conf` — host-based authentication
  - `pg-nossl.yaml` / `pg-ssl.yaml` — StatefulSet CRs
  - `pg-services.yaml` — Service definitions

### Process Federation Server (PFS)

- Deployed via `cp4ba-process-federation-server` companion repository
- Configured with 2 replicas (`CP4BA_INST_PFS_REPLICAS=2`) in most recipes
- Used to federate BAW or WfPS instances behind a unified portal

---

## 12. Storage Configuration

All recipes require two storage classes:

| Variable | Storage Type | Typical Class |
|----------|-------------|---------------|
| `CP4BA_INST_SC_FILE` | File (RWX) | `ocs-external-storagecluster-cephfs` |
| `CP4BA_INST_SC_BLOCK` | Block (RWO) | `ocs-external-storagecluster-ceph-rbd` |

The file storage class maps to four roles: `sc_dynamic_storage_classname`, `sc_slow_file_storage_classname`, `sc_medium_file_storage_classname`, `sc_fast_file_storage_classname`.

---

## 13. Network Policy Support

Network policies can be optionally deployed alongside the CR:

```bash
# Scan a folder and apply all found network policy YAML files
CP4BA_INST_NP_DEPLOY="true"
CP4BA_INST_NP_SCAN_FOLDER="true"
CP4BA_INST_NP_FOLDER="../templates-networkpolicies"

# Or specify specific policy files
CP4BA_INST_NP_SCAN_FOLDER="false"
CP4BA_INST_NP_TEMPLATE_1="../templates-networkpolicies/mutually-exclusive/my-network-policy-sample-allow-all.yaml"
```

Pre-built templates available in `templates-networkpolicies/`:
- `my-network-policy-sample-allow-all.yaml` — allow all traffic
- `my-network-policy-sample-deny-all.yaml` — deny all traffic
- Samples 1–4 — various selective allow rules

---

## 14. GenAI / watsonx Integration

BAW Authoring and BAW Runtime templates include support for connecting to **watsonx.ai** for generative AI features:

```bash
# Required env vars (set before running installation)
export CP4BA_INST_GENAI_ENABLED="true"
export CP4BA_INST_GENAI_WX_APIKEY="<your-ibm-cloud-api-key>"
export CP4BA_INST_GENAI_WX_PRJ_ID="<your-watsonx-project-id>"
```

Optional watsonx Orchestrate integration:
```bash
export CP4BA_INST_WXO_ENABLED="true"
export CP4BA_INST_WXO_TOKEN_URL="https://iam.platform.saas.ibm.com/siusermgr/api/1.0/apikeys/token"
export CP4BA_INST_WXO_SERVICE_INSTANCE_URL="https://api.hostname/instances/<tenant_id>"
```

In the CR, these control:
- `workflow_assistant_configuration.run_authoring_agent` — AI authoring assistant
- `workflow_assistant_configuration.run_workplace_agent` — AI workplace assistant
- `workflow_authoring_configuration.environment_config.content_security_policy_additional_*` — CSP headers for watsonx domains

---

*End of knowledge base*
