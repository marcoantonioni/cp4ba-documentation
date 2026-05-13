# CP4BA IDP-LDAP Pipeline Documentation

## Overview

This document provides comprehensive documentation for the CP4BA IDP-LDAP scripts located in `cp4ba-idp-ldap/scripts/`. These scripts manage the installation, configuration, and removal of LDAP servers, Identity Providers (IDP), PHP LDAP Admin interfaces, and user onboarding for IBM Cloud Pak for Business Automation.

## Scripts Summary

| Script | Purpose | Key Operations |
|--------|---------|----------------|
| `add-ldap.sh` | Install LDAP server | Creates namespace, secrets, deployment, and service |
| `add-idp.sh` | Configure IDP | Registers LDAP as identity provider in CP4BA |
| `add-phpadmin.sh` | Deploy PHP LDAP Admin | Installs web UI for LDAP management |
| `onboard-users.sh` | Manage users | Adds/removes users from CP4BA platform |
| `remove-ldap.sh` | Remove LDAP server | Deletes LDAP deployment and resources |
| `remove-idp.sh` | Remove IDP | Unregisters identity provider |
| `remove-phpadmin.sh` | Remove PHP Admin | Deletes PHP LDAP Admin interface |

---

## 1. add-ldap.sh - LDAP Server Installation

### Purpose
Deploys an OpenLDAP server in an OpenShift namespace with custom user data from LDIF files.

### Parameters
- `-c`: Full path to environment configuration file (required)
- `-n`: Target namespace (optional, can be set in config)
- `-p`: Full path to LDAP configuration file (required)

### Main Execution Flow

```mermaid
sequenceDiagram
    participant User
    participant Script as add-ldap.sh
    participant Logger as cp4ba-logger
    participant OC as OpenShift CLI
    participant K8s as Kubernetes Resources

    User->>Script: Execute with -c, -n, -p parameters
    Script->>Logger: Initialize logging system
    Script->>Script: checkParams()
    alt Configuration file missing
        Script->>User: ERROR: Configuration file not found
    end
    
    Script->>Script: Load configuration files
    Script->>Script: Validate LDIF file exists
    
    Script->>Script: createNamespace()
    Script->>OC: Check if namespace exists
    alt Namespace doesn't exist
        Script->>OC: oc new-project ${TNS}
    end
    
    Script->>Script: createEntitlementSecrets()
    Script->>OC: Check if pull-secret exists
    alt Secret doesn't exist
        Script->>OC: Create docker-registry secret
        Script->>OC: Link secret to default SA
    end
    Script->>OC: Check if ibm-entitlement-key exists
    alt Secret doesn't exist
        Script->>OC: Create ibm-entitlement-key secret
    end
    
    Script->>Script: createSecrets()
    Script->>OC: Create ${LDAP_DOMAIN}-secret
    Note over OC: Contains LDAP admin passwords
    Script->>OC: Create ${LDAP_DOMAIN}-customldif
    Note over OC: Contains LDIF user data
    
    Script->>Script: createServiceAccountAndCfgMap()
    Script->>OC: Create ServiceAccount ibm-cp4ba-anyuid
    Script->>OC: Add anyuid SCC to SA
    Script->>OC: Create ConfigMap ${LDAP_DOMAIN}-env
    Note over OC: LDAP configuration parameters
    
    Script->>Script: createDeployment()
    Script->>OC: Create LDAP Deployment
    Note over K8s: Init containers prepare LDIF and folders
    Note over K8s: Main container runs OpenLDAP
    Script->>OC: Expose deployment as service
    
    Script->>Script: waitForDeploymentReady()
    loop Until ready
        Script->>OC: Check deployment status
        OC-->>Script: REPLICAS vs READY_REPLICAS
    end
    
    Script->>OC: Get service name
    Script->>User: Display LDAP service URLs
    Note over User: ldap://${SERVICE}.${NS}.svc.cluster.local:389
    Note over User: ldaps://${SERVICE}.${NS}.svc.cluster.local:636
```

### Key Functions

#### checkParams()
- Validates configuration file exists
- Validates properties file exists
- Validates LDIF file exists (checks multiple paths)
- Validates namespace is set
- Validates entitlement key is set

#### createNamespace()
- Checks if namespace exists using `namespaceExist()`
- Creates new project if needed

#### createEntitlementSecrets()
- Creates `pull-secret` for image pulling
- Creates `ibm-entitlement-key` for IBM registry access
- Links secrets to service accounts

#### createSecrets()
- Creates `${LDAP_DOMAIN}-secret` with admin passwords
- Creates `${LDAP_DOMAIN}-customldif` from LDIF file

#### createServiceAccountAndCfgMap()
- Creates `ibm-cp4ba-anyuid` service account
- Adds `anyuid` security context constraint
- Creates ConfigMap with LDAP environment variables

#### createDeployment()
- Deploys OpenLDAP with init containers:
  - `openldap-init-ldif`: Copies custom LDIF files
  - `folder-prepare-container`: Prepares filesystem folders
- Main container runs OpenLDAP server
- Exposes ports 389 (LDAP) and 636 (LDAPS)

### Execution Branches

```mermaid
flowchart TD
    A[Start] --> B{Config file exists?}
    B -->|No| C[Exit with error]
    B -->|Yes| D{Properties file exists?}
    D -->|No| C
    D -->|Yes| E{LDIF file exists?}
    E -->|No| F{Check alternate path}
    F -->|Not found| C
    F -->|Found| G[Load configurations]
    E -->|Yes| G
    G --> H{Namespace exists?}
    H -->|No| I[Create namespace]
    H -->|Yes| J[Create secrets]
    I --> J
    J --> K{Secrets exist?}
    K -->|No| L[Create new secrets]
    K -->|Yes| M[Skip creation]
    L --> N[Create SA and ConfigMap]
    M --> N
    N --> O[Create Deployment]
    O --> P[Wait for ready]
    P --> Q[Display service URLs]
    Q --> R[End]
```

---

## 2. add-idp.sh - Identity Provider Configuration

### Purpose
Registers an LDAP server as an Identity Provider in the CP4BA platform, enabling authentication and user management.

### Parameters
- `-c`: Full path to environment configuration file (required)
- `-p`: Full path to IDP configuration file (required)
- `-f`: Force installation (optional, removes existing IDP)

### Main Execution Flow

```mermaid
sequenceDiagram
    participant User
    participant Script as add-idp.sh
    participant Logger as cp4ba-logger
    participant OC as OpenShift CLI
    participant IAM as IAM Service
    participant IDP as IDP Service

    User->>Script: Execute with -c, -p, -f parameters
    Script->>Logger: Initialize logging system
    Script->>Script: checkParams()
    Script->>Script: Load configuration files
    
    Script->>Script: getCommonValues()
    Script->>OC: Determine console route name
    Note over OC: cp-console or platform-id-provider
    Script->>Script: waitForResourceCreated()
    Script->>OC: Wait for platform-auth-idp-credentials
    Script->>OC: Get admin username/password
    Script->>OC: Get console host URL
    Script->>IAM: POST /idprovider/v1/auth/identitytoken
    IAM-->>Script: Return IAM access token
    
    Script->>Script: getIDPInfos()
    Script->>IDP: GET /idprovider/v3/auth/idsource
    IDP-->>Script: Return list of configured IDPs
    
    Script->>Script: verifyIDPAlreadyPresent()
    alt IDP already exists
        alt Force flag set
            Script->>Script: deleteIDP()
            Script->>Script: getUID()
            Script->>IDP: DELETE /idprovider/v3/auth/idsource/${UID}
            IDP-->>Script: Success response
        else Force flag not set
            Script->>User: ERROR: IDP already configured
        end
    end
    
    Script->>Script: createIdp()
    Script->>Script: createIDPConfiguration()
    Note over Script: Generate JSON with LDAP parameters
    Script->>IDP: POST /idprovider/v3/auth/idsource
    Note over IDP: Register new IDP
    IDP-->>Script: Success or error response
    
    alt Success
        Script->>Script: configSCIM()
        Script->>IDP: POST /idmgmt/identity/api/v1/scim/attributemappings
        Note over IDP: Configure SCIM attribute mappings
        IDP-->>Script: Success response
        Script->>User: IDP configured successfully
    else Error
        alt Already exists
            Script->>User: ERROR: Use -f to force
        else Other error
            Script->>User: ERROR with details
        end
    end
    
    Script->>Script: getIDPInfos()
    Script->>Script: showIDPList()
    Script->>User: Display updated IDP list
    Script->>User: WARNING: Restart BAW server for Groups
```

### Key Functions

#### getCommonValues()
- Determines console route name (cp-console or platform-id-provider)
- Waits for `platform-auth-idp-credentials` secret
- Retrieves admin credentials
- Obtains IAM access token for API calls

#### createIDPConfiguration()
- Generates JSON configuration with LDAP parameters:
  - Connection details (URL, host, port, protocol)
  - Bind credentials
  - User and group filters
  - ID mappings
  - Search settings

#### configSCIM()
- Configures SCIM (System for Cross-domain Identity Management) attributes
- Maps LDAP attributes to platform user/group attributes
- Defines object classes and member relationships

#### createIdp()
- Creates IDP configuration JSON
- Posts configuration to IDP service
- Handles error responses
- Calls configSCIM on success

#### verifyIDPAlreadyPresent()
- Checks if IDP name already exists
- If force flag set, deletes existing IDP
- Otherwise exits with error

### Execution Branches

```mermaid
flowchart TD
    A[Start] --> B{Config files valid?}
    B -->|No| C[Exit with error]
    B -->|Yes| D[Get common values]
    D --> E[Get IAM token]
    E --> F[Get IDP list]
    F --> G{IDP exists?}
    G -->|No| H[Create IDP config]
    G -->|Yes| I{Force flag set?}
    I -->|No| C
    I -->|Yes| J[Delete existing IDP]
    J --> H
    H --> K[POST to IDP service]
    K --> L{Success?}
    L -->|No| M{Already exists?}
    M -->|Yes| N[Show force message]
    M -->|No| O[Show error details]
    N --> C
    O --> C
    L -->|Yes| P[Configure SCIM]
    P --> Q{SCIM success?}
    Q -->|No| C
    Q -->|Yes| R[Show IDP list]
    R --> S[Show restart warning]
    S --> T[End]
```

---

## 3. add-phpadmin.sh - PHP LDAP Admin Installation

### Purpose
Deploys phpLDAPadmin web interface for managing LDAP entries through a browser.

### Parameters
- `-p`: Full path to LDAP configuration file (required)
- `-n`: Target namespace (required)
- `-s`: Secret name for TLS certificates (required)
- `-w`: Web UI secret name for TLS certificates (required)

### Main Execution Flow

```mermaid
sequenceDiagram
    participant User
    participant Script as add-phpadmin.sh
    participant Logger as cp4ba-logger
    participant OC as OpenShift CLI
    participant K8s as Kubernetes Resources

    User->>Script: Execute with -p, -n, -s, -w parameters
    Script->>Logger: Initialize logging system
    Script->>Script: Load properties file
    
    Script->>Script: extractCreateSecretsTls()
    Script->>OC: Get TLS cert from ${SECRET_NAME}
    Script->>OC: Get TLS key from ${SECRET_NAME}
    Script->>OC: Get web UI cert from ${SECRET_NAME_WEB_UI}
    Script->>OC: Get web UI key from ${SECRET_NAME_WEB_UI}
    Note over Script: Save to temporary files
    
    Script->>OC: Check if phpadminldap-${LDAP_DOMAIN}-root-ca exists
    alt Secret doesn't exist
        Script->>OC: Create TLS secret from cert/key
    end
    
    Script->>OC: Check if phpadminldap-${LDAP_DOMAIN}-prereq-ext exists
    alt Secret doesn't exist
        Script->>OC: Create TLS secret from web UI cert/key
    end
    
    Script->>Script: Clean up temporary files
    
    Script->>Script: deployPHPAdmin()
    
    Script->>OC: Check if ConfigMap exists
    alt ConfigMap doesn't exist
        Script->>OC: Create php-admin-${LDAP_DOMAIN}-cm
        Note over K8s: HTTPS settings and LDAP host
    end
    
    Script->>OC: Check if Deployment exists
    alt Deployment doesn't exist
        Script->>OC: Create phpldapadmin-${LDAP_DOMAIN}
        Note over K8s: Init container copies certificates
        Note over K8s: Main container runs phpLDAPadmin
    end
    
    Script->>OC: Check if Service exists
    alt Service doesn't exist
        Script->>OC: Create php-admin-${LDAP_DOMAIN}
        Note over K8s: Exposes port 443
    end
    
    Script->>OC: Check if Route exists
    alt Route doesn't exist
        Script->>OC: Create temporary route
        Script->>OC: Get route hostname
        Script->>Script: Build custom FQDN
        Script->>OC: Delete temporary route
        Script->>OC: Create final route with custom FQDN
        Note over K8s: Passthrough TLS termination
    end
    
    Script->>OC: Get admin password from secret
    Script->>User: Display phpLDAPadmin URL and credentials
    Note over User: https://${PHP_FQDN}
    Note over User: User: cn=admin,${LDAP_FULL_DOMAIN}
```

### Key Functions

#### extractCreateSecretsTls()
- Extracts TLS certificates from existing secrets
- Creates temporary cert/key files
- Creates two new secrets:
  - `phpadminldap-${LDAP_DOMAIN}-root-ca`: Root CA certificate
  - `phpadminldap-${LDAP_DOMAIN}-prereq-ext`: Web UI certificate
- Cleans up temporary files

#### deployPHPAdmin()
- Creates ConfigMap with phpLDAPadmin settings
- Deploys phpLDAPadmin with:
  - Init container to prepare certificates
  - Main container running phpLDAPadmin
  - Volume mounts for certificates
- Creates Service on port 443
- Creates Route with custom FQDN and passthrough TLS

### Execution Branches

```mermaid
flowchart TD
    A[Start] --> B{Properties file exists?}
    B -->|No| C[Exit with error]
    B -->|Yes| D[Extract TLS secrets]
    D --> E{Root CA secret exists?}
    E -->|No| F[Create root CA secret]
    E -->|Yes| G{Web UI secret exists?}
    F --> G
    G -->|No| H[Create web UI secret]
    G -->|Yes| I[Clean temp files]
    H --> I
    I --> J{ConfigMap exists?}
    J -->|No| K[Create ConfigMap]
    J -->|Yes| L{Deployment exists?}
    K --> L
    L -->|No| M[Create Deployment]
    L -->|Yes| N{Service exists?}
    M --> N
    N -->|No| O[Create Service]
    N -->|Yes| P{Route exists?}
    O --> P
    P -->|No| Q[Create temp route]
    Q --> R[Get hostname]
    R --> S[Build custom FQDN]
    S --> T[Delete temp route]
    T --> U[Create final route]
    U --> V[Get admin password]
    P -->|Yes| V
    V --> W[Display credentials]
    W --> X[End]
```

---

## 4. onboard-users.sh - User Management

### Purpose
Manages user onboarding in the CP4BA platform by adding, removing, or updating users from LDAP.

### Parameters
- `-p`: Full path to properties file (required)
- `-l`: Full path to LDAP configuration file (optional)
- `-n`: LDAP namespace (optional)
- `-e`: Environment namespace (required)
- `-u`: Users file path (required if not using -s)
- `-o`: Operation mode: add, remove, remove-and-add (required)
- `-s`: Load users from secret instead of file (optional)
- `-r`: List available roles and groups (optional)

### Main Execution Flow

```mermaid
sequenceDiagram
    participant User
    participant Script as onboard-users.sh
    participant Logger as cp4ba-logger
    participant OC as OpenShift CLI
    participant IAM as IAM Service
    participant ZEN as Zen Service
    participant USERMGMT as User Management API

    User->>Script: Execute with parameters
    Script->>Logger: Initialize logging system
    Script->>Script: Load configuration files
    Script->>Script: setTemporaryFolder()
    
    alt List roles mode (-r flag)
        Script->>Script: getCommonValues()
        Script->>OC: Get admin credentials
        Script->>IAM: Get IAM token
        Script->>ZEN: Get Zen token
        Script->>USERMGMT: GET /usermgmt/v1/roles
        USERMGMT-->>Script: Return roles list
        Script->>User: Display roles
        Script->>USERMGMT: GET /usermgmt/v2/groups
        USERMGMT-->>Script: Return groups list
        Script->>User: Display groups
    else User operations mode
        Script->>Script: getCommonValues()
        Script->>OC: Wait for platform-auth-idp-credentials
        Script->>OC: Wait for cpd route
        Script->>OC: Get admin credentials
        Script->>IAM: POST /idprovider/v1/auth/identitytoken
        IAM-->>Script: Return IAM token
        Script->>ZEN: POST /v1/preauth/validateAuth
        ZEN-->>Script: Return Zen access token
        
        alt Load from secret (-s flag)
            Script->>Script: loadUsersFromSecret()
            Script->>OC: Get ${LDAP_DOMAIN}-customldif secret
            Script->>Script: Extract user UIDs from LDIF
            Script->>Script: Transform to list format
        else Load from file
            Script->>Script: loadUsersFromFile()
            Script->>Script: Read users from file
            Script->>Script: Transform to list format
        end
        
        alt Operation: add
            Script->>Script: onboardUsersAdd()
            Script->>Script: Parse user list
            Script->>Script: Check admin users
            loop For each user
                alt User is admin
                    Script->>Script: Create record with admin roles
                else User is regular
                    Script->>Script: Create record with user role
                end
            end
            Script->>USERMGMT: POST /usermgmt/v1/user/bulk
            Note over USERMGMT: Bulk add users
            USERMGMT-->>Script: Success/error response
            Script->>User: Display operation result
        else Operation: remove
            Script->>Script: onboardUsersRemove()
            loop For each user
                Script->>USERMGMT: DELETE /usermgmt/v1/user/${USER}
                USERMGMT-->>Script: Success/error response
            end
            Script->>User: Display removal count
        else Operation: remove-and-add
            Script->>Script: onboardUsersRemove()
            Script->>Script: onboardUsersAdd()
        end
    end
```

### Key Functions

#### setTemporaryFolder()
- Validates temporary folder exists and is writable
- Uses `CP4BA_INST_TMP_FOLDER` environment variable or defaults to `/tmp`

#### getCommonValues()
- Determines console route name
- Waits for required secrets and routes
- Retrieves admin credentials
- Obtains IAM and Zen access tokens

#### loadUsersFromSecret()
- Extracts LDIF data from Kubernetes secret
- Parses user UIDs from LDIF format
- Transforms to internal list format

#### loadUsersFromFile()
- Reads users from specified file
- Transforms to internal list format

#### onboardUsersAdd()
- Parses user list
- Checks against admin list from configuration
- Creates user records with appropriate roles:
  - Admins: Multiple roles including zen_administrator_role
  - Regular users: zen_user_role only
- Bulk posts to user management API

#### onboardUsersRemove()
- Iterates through user list
- Deletes each user individually via API
- Handles errors for non-existent users

### Execution Branches

```mermaid
flowchart TD
    A[Start] --> B{Config files valid?}
    B -->|No| C[Exit with error]
    B -->|Yes| D{List roles flag?}
    D -->|Yes| E[Get tokens]
    E --> F[Fetch roles]
    F --> G[Fetch groups]
    G --> H[Display and exit]
    D -->|No| I[Get common values]
    I --> J{Load from secret?}
    J -->|Yes| K[loadUsersFromSecret]
    J -->|No| L[loadUsersFromFile]
    K --> M{Operation mode?}
    L --> M
    M -->|add| N[Parse users]
    N --> O{Is admin?}
    O -->|Yes| P[Add admin roles]
    O -->|No| Q[Add user role]
    P --> R[Bulk POST users]
    Q --> R
    R --> S{Success?}
    S -->|Yes| T[Display success]
    S -->|No| U[Display error]
    M -->|remove| V[Loop users]
    V --> W[DELETE each user]
    W --> X[Display count]
    M -->|remove-and-add| Y[Remove users]
    Y --> N
    T --> Z[End]
    U --> Z
    X --> Z
```

---

## 5. remove-ldap.sh - LDAP Server Removal

### Purpose
Removes LDAP server deployment and associated resources from OpenShift namespace.

### Parameters
- `-p`: Full path to LDAP properties file (required)
- `-n`: Target namespace (optional, can be set in config)

### Main Execution Flow

```mermaid
sequenceDiagram
    participant User
    participant Script as remove-ldap.sh
    participant Logger as cp4ba-logger
    participant OC as OpenShift CLI

    User->>Script: Execute with -p, -n parameters
    Script->>Logger: Initialize logging system
    Script->>Script: Load properties file
    Script->>Script: Validate namespace
    
    Script->>Script: deleteSecrets()
    Script->>OC: Check if ${LDAP_DOMAIN}-secret exists
    alt Secret exists
        Script->>OC: Delete ${LDAP_DOMAIN}-secret
    end
    Script->>OC: Check if ${LDAP_DOMAIN}-customldif exists
    alt Secret exists
        Script->>OC: Delete ${LDAP_DOMAIN}-customldif
    end
    
    Script->>Script: deleteCfgMap()
    Script->>OC: Check if ${LDAP_DOMAIN}-env exists
    alt ConfigMap exists
        Script->>OC: Delete ${LDAP_DOMAIN}-env
    end
    
    Script->>Script: deleteDeployment()
    Script->>OC: Check if ${LDAP_DOMAIN}-ldap deployment exists
    alt Deployment exists
        Script->>OC: Delete deployment ${LDAP_DOMAIN}-ldap
    end
    Script->>OC: Check if ${LDAP_DOMAIN}-ldap service exists
    alt Service exists
        Script->>OC: Delete service ${LDAP_DOMAIN}-ldap
    end
    
    Script->>User: Deletion complete
```

### Key Functions

#### deleteSecrets()
- Checks and deletes `${LDAP_DOMAIN}-secret`
- Checks and deletes `${LDAP_DOMAIN}-customldif`

#### deleteCfgMap()
- Checks and deletes `${LDAP_DOMAIN}-env` ConfigMap

#### deleteDeployment()
- Checks and deletes LDAP deployment
- Checks and deletes LDAP service

### Execution Branches

```mermaid
flowchart TD
    A[Start] --> B{Properties file exists?}
    B -->|No| C[Exit with error]
    B -->|Yes| D{Namespace set?}
    D -->|No| C
    D -->|Yes| E{Secret exists?}
    E -->|Yes| F[Delete secret]
    E -->|No| G{CustomLDIF exists?}
    F --> G
    G -->|Yes| H[Delete customldif]
    G -->|No| I{ConfigMap exists?}
    H --> I
    I -->|Yes| J[Delete ConfigMap]
    I -->|No| K{Deployment exists?}
    J --> K
    K -->|Yes| L[Delete deployment]
    K -->|No| M{Service exists?}
    L --> M
    M -->|Yes| N[Delete service]
    M -->|No| O[End]
    N --> O
```

---

## 6. remove-idp.sh - Identity Provider Removal

### Purpose
Unregisters an Identity Provider from the CP4BA platform.

### Parameters
- `-p`: Full path to IDP properties file (required)
- `-n`: Target namespace (optional, can be set in config)

### Main Execution Flow

```mermaid
sequenceDiagram
    participant User
    participant Script as remove-idp.sh
    participant Logger as cp4ba-logger
    participant OC as OpenShift CLI
    participant IAM as IAM Service
    participant IDP as IDP Service

    User->>Script: Execute with -p, -n parameters
    Script->>Logger: Initialize logging system
    Script->>Script: Load properties file
    Script->>Script: Validate namespace
    
    Script->>Script: getCommonValues()
    Script->>OC: Determine console route
    Script->>OC: Get admin credentials
    Script->>OC: Get console host
    Script->>IAM: POST /idprovider/v1/auth/identitytoken
    IAM-->>Script: Return IAM token
    
    Script->>Script: getIDPInfos()
    Script->>IDP: GET /idprovider/v3/auth/idsource
    IDP-->>Script: Return IDP list
    
    Script->>Script: deleteIDP()
    Script->>Script: getUID()
    Note over Script: Extract UID for IDP name
    
    alt IDP found
        Script->>IDP: DELETE /idprovider/v3/auth/idsource/${UID}
        IDP-->>Script: Success/error response
        alt Success
            Script->>User: IDP deleted successfully
        else Error
            Script->>User: ERROR with details
        end
    else IDP not found
        Script->>User: WARNING: IDP not found
    end
    
    Script->>Script: getIDPInfos()
    Script->>Script: showIDPList()
    Script->>User: Display updated IDP list
```

### Key Functions

#### getCommonValues()
- Determines console route name
- Retrieves admin credentials
- Obtains IAM access token

#### getIDPInfos()
- Fetches list of configured IDPs
- Extracts IDP names

#### getUID()
- Searches IDP list for matching name
- Returns UID for deletion

#### deleteIDP()
- Gets IDP UID
- Sends DELETE request to IDP service
- Handles success/error responses

### Execution Branches

```mermaid
flowchart TD
    A[Start] --> B{Properties file exists?}
    B -->|No| C[Exit with error]
    B -->|Yes| D{Namespace set?}
    D -->|No| C
    D -->|Yes| E[Get common values]
    E --> F[Get IAM token]
    F --> G[Get IDP list]
    G --> H[Find IDP UID]
    H --> I{IDP found?}
    I -->|No| J[Show warning]
    I -->|Yes| K[DELETE IDP]
    K --> L{Success?}
    L -->|Yes| M[Show success]
    L -->|No| N[Show error]
    J --> O[Get updated list]
    M --> O
    N --> C
    O --> P[Display IDP list]
    P --> Q[End]
```

---

## 7. remove-phpadmin.sh - PHP LDAP Admin Removal

### Purpose
Removes phpLDAPadmin web interface and associated resources.

### Parameters
- `-p`: Full path to LDAP properties file (required)
- `-n`: Target namespace (optional, can be set in config)

### Main Execution Flow

```mermaid
sequenceDiagram
    participant User
    participant Script as remove-phpadmin.sh
    participant Logger as cp4ba-logger
    participant OC as OpenShift CLI

    User->>Script: Execute with -p, -n parameters
    Script->>Logger: Initialize logging system
    Script->>Script: Load properties file
    Script->>Script: Validate namespace
    
    Script->>Script: deleteSecretsTls()
    Script->>OC: Check if phpadminldap-${LDAP_DOMAIN}-root-ca exists
    alt Secret exists
        Script->>OC: Delete phpadminldap-${LDAP_DOMAIN}-root-ca
    end
    Script->>OC: Check if phpadminldap-${LDAP_DOMAIN}-prereq-ext exists
    alt Secret exists
        Script->>OC: Delete phpadminldap-${LDAP_DOMAIN}-prereq-ext
    end
    
    Script->>Script: deletePHPAdmin()
    Script->>OC: Check if php-admin-${LDAP_DOMAIN}-cm exists
    alt ConfigMap exists
        Script->>OC: Delete php-admin-${LDAP_DOMAIN}-cm
    end
    Script->>OC: Check if phpldapadmin-${LDAP_DOMAIN} deployment exists
    alt Deployment exists
        Script->>OC: Delete phpldapadmin-${LDAP_DOMAIN}
    end
    Script->>OC: Check if php-admin-${LDAP_DOMAIN} service exists
    alt Service exists
        Script->>OC: Delete php-admin-${LDAP_DOMAIN}
    end
    Script->>OC: Check if php-admin-${LDAP_DOMAIN} route exists
    alt Route exists
        Script->>OC: Delete php-admin-${LDAP_DOMAIN}
    end
    
    Script->>User: Deletion complete
```

### Key Functions

#### deleteSecretsTls()
- Checks and deletes `phpadminldap-${LDAP_DOMAIN}-root-ca`
- Checks and deletes `phpadminldap-${LDAP_DOMAIN}-prereq-ext`

#### deletePHPAdmin()
- Checks and deletes ConfigMap
- Checks and deletes Deployment
- Checks and deletes Service
- Checks and deletes Route

### Execution Branches

```mermaid
flowchart TD
    A[Start] --> B{Properties file exists?}
    B -->|No| C[Exit with error]
    B -->|Yes| D{Namespace set?}
    D -->|No| C
    D -->|Yes| E{Root CA secret exists?}
    E -->|Yes| F[Delete root CA]
    E -->|No| G{Prereq secret exists?}
    F --> G
    G -->|Yes| H[Delete prereq secret]
    G -->|No| I{ConfigMap exists?}
    H --> I
    I -->|Yes| J[Delete ConfigMap]
    I -->|No| K{Deployment exists?}
    J --> K
    K -->|Yes| L[Delete deployment]
    K -->|No| M{Service exists?}
    L --> M
    M -->|Yes| N[Delete service]
    M -->|No| O{Route exists?}
    N --> O
    O -->|Yes| P[Delete route]
    O -->|No| Q[End]
    P --> Q
```

---

## Common Patterns and Dependencies

### Logger Integration

All scripts integrate with the `cp4ba-logger` package:

```bash
source $_SCRIPT_DIR/../../cp4ba-logger/scripts/logger.sh
```

Environment variables for logging:
- `CP4BA_LOGGING_ENABLED`: Enable/disable logging (default: true)
- `CP4BA_LOG_LEVEL`: Log level (default: INFO)
- `CP4BA_LOG_TO_CONSOLE`: Console output (default: true)
- `CP4BA_LOG_TO_FILE`: File output (default: false)
- `CP4BA_LOG_FILE`: Log file path
- `CP4BA_LOG_MAX_SIZE`: Max log file size (default: 10MB)
- `CP4BA_LOG_BACKUP_COUNT`: Number of backup files (default: 5)

### Resource Existence Check

Common function pattern:
```bash
resourceExist () {
  if [ $(oc get $2 -n $1 $3 2> /dev/null | grep $3 | wc -l) -lt 1 ]; then
    return 0  # Resource doesn't exist
  fi
  return 1    # Resource exists
}
```

### Color Codes

All scripts use consistent color coding:
- `_CLR_RED`: Error messages
- `_CLR_GREEN`: Success messages and labels
- `_CLR_YELLOW`: Values and highlights
- `_CLR_BLUE`: Information
- `_CLR_NC`: Reset color

---

## Typical Installation Workflow

```mermaid
flowchart LR
    A[1. add-ldap.sh] --> B[2. add-idp.sh]
    B --> C[3. add-phpadmin.sh]
    C --> D[4. onboard-users.sh]
    
    style A fill:#90EE90
    style B fill:#87CEEB
    style C fill:#FFB6C1
    style D fill:#FFD700
```

### Step-by-Step Process

1. **Install LDAP Server** (`add-ldap.sh`)
   - Creates namespace and secrets
   - Deploys OpenLDAP with custom users
   - Exposes LDAP service

2. **Configure Identity Provider** (`add-idp.sh`)
   - Registers LDAP as IDP in CP4BA
   - Configures SCIM mappings
   - Enables authentication

3. **Deploy PHP Admin** (`add-phpadmin.sh`)
   - Installs web management interface
   - Configures TLS certificates
   - Creates accessible route

4. **Onboard Users** (`onboard-users.sh`)
   - Adds users to CP4BA platform
   - Assigns roles and permissions
   - Enables user access

---

## Typical Removal Workflow

```mermaid
flowchart LR
    A[1. remove-phpadmin.sh] --> B[2. remove-idp.sh]
    B --> C[3. remove-ldap.sh]
    
    style A fill:#FFB6C1
    style B fill:#87CEEB
    style C fill:#90EE90
```

### Removal Order

1. **Remove PHP Admin** (`remove-phpadmin.sh`)
   - Removes web interface
   - Cleans up routes and services

2. **Remove Identity Provider** (`remove-idp.sh`)
   - Unregisters IDP from CP4BA
   - Removes authentication configuration

3. **Remove LDAP Server** (`remove-ldap.sh`)
   - Deletes LDAP deployment
   - Removes secrets and ConfigMaps

---

## Configuration Files

### Environment Configuration
Typically contains:
- `TNS`: Target namespace
- `ENTITLEMENT_KEY`: IBM entitlement key

### LDAP Configuration
Typically contains:
- `LDAP_DOMAIN`: Domain name
- `LDAP_DOMAIN_EXT`: Domain extension
- `LDAP_HOST`: LDAP server host
- `LDAP_PORT`: LDAP server port
- `LDAP_PROTOCOL`: Protocol (ldap/ldaps)
- `LDAP_BASEDN`: Base DN
- `LDAP_BINDDN`: Bind DN
- `LDAP_BINDPASSWORD`: Bind password
- `LDAP_USERFILTER`: User filter
- `LDAP_USERIDMAP`: User ID mapping
- `LDAP_GROUPFILTER`: Group filter
- `LDAP_GROUPIDMAP`: Group ID mapping
- `LDAP_GROUPMEMBERIDMAP`: Group member mapping
- `LDAP_NESTEDSEARCH`: Nested search setting
- `LDAP_PAGINGSEARCH`: Paging search setting
- `LDAP_LDIF_NAME`: LDIF file name
- `LDAP_ADMINS`: Comma-separated admin users

### IDP Configuration
Typically contains:
- `IDP_NAME`: Identity provider name
- All LDAP configuration parameters

---

## Error Handling

All scripts implement consistent error handling:

1. **Parameter Validation**
   - Check required parameters
   - Validate file existence
   - Verify namespace settings

2. **Resource Checks**
   - Verify resources before operations
   - Handle existing resources gracefully
   - Provide clear error messages

3. **API Response Handling**
   - Check for error responses
   - Parse and display error details
   - Exit with appropriate codes

4. **Logging**
   - Use color-coded messages
   - Provide context for operations
   - Display success/failure clearly

---

## Best Practices

1. **Always use configuration files** instead of hardcoding values
2. **Check resource existence** before creation or deletion
3. **Wait for resources** to be ready before proceeding
4. **Use force flags carefully** when overwriting existing resources
5. **Follow the installation order** for proper setup
6. **Follow the removal order** for clean uninstallation
7. **Verify credentials** after installation
8. **Restart BAW server** after IDP configuration to see groups

---

## Troubleshooting

### Common Issues

1. **LDAP not accessible**
   - Check deployment status
   - Verify service creation
   - Check network policies

2. **IDP registration fails**
   - Verify LDAP is running
   - Check LDAP connection parameters
   - Ensure IAM token is valid

3. **Users not appearing**
   - Verify IDP is configured
   - Check SCIM mappings
   - Restart BAW server

4. **PHP Admin not accessible**
   - Check route creation
   - Verify TLS certificates
   - Check deployment logs

### Debug Commands

```bash
# Check LDAP deployment
oc get deployment -n ${TNS} ${LDAP_DOMAIN}-ldap

# Check LDAP logs
oc logs -n ${TNS} deployment/${LDAP_DOMAIN}-ldap

# Test LDAP connection
ldapsearch -x -H ldap://${LDAP_HOST}:389 -b "${LDAP_BASEDN}"

# Check IDP configuration
curl -sk -H "Authorization: Bearer ${IAM_TOKEN}" \
  ${CONSOLE_HOST}/idprovider/v3/auth/idsource

# Check user list
curl -sk -H "Authorization: Bearer ${ZEN_TOKEN}" \
  ${PAK_HOST}/usermgmt/v1/usermgmt/users
```

---

## Summary

This pipeline provides a complete solution for:
- Deploying LDAP servers in OpenShift
- Integrating LDAP with CP4BA as an Identity Provider
- Managing LDAP through a web interface
- Onboarding and managing users
- Clean removal of all components

All scripts follow consistent patterns, use proper error handling, and integrate with the logging framework for operational visibility.