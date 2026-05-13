# CP4BA Deploy Environment Script Documentation

## Overview

The `cp4ba-deploy-env.sh` script is a comprehensive bash automation tool for deploying IBM Cloud Pak for Business Automation (CP4BA) environments. It orchestrates the complete deployment lifecycle including prerequisites validation, LDAP setup, database configuration, secrets management, and Custom Resource (CR) deployment.

## Script Metadata

- **Script Name**: `cp4ba-deploy-env.sh`
- **Location**: `cp4ba-installations/scripts/`
- **Purpose**: Automated CP4BA environment deployment and configuration
- **Dependencies**: `jq`, `openssl`, `oc` (OpenShift CLI), `yq`, `envsubst`

## Command Line Parameters

| Parameter | Required | Description | Example |
|-----------|----------|-------------|---------|
| `-c` | Yes | Full path to configuration file | `../configs/env1.properties` |
| `-l` | No | LDAP configuration file | `../configs/_cfg-production-ldap-domain.properties` |
| `-w` | No | Wait only mode - skip deployment, create access info | N/A |
| `-g` | No | Generate YAML only - skip deployment | N/A |
| `-f` | No | Federate only - skip deployment | N/A |

## Execution Modes

### 1. Full Deployment Mode (Default)
Complete deployment including all prerequisites, CR deployment, and post-installation steps.

### 2. Generate Only Mode (`-g`)
Generates the CR YAML and database scripts without deploying.

### 3. Wait Only Mode (`-w`)
Skips deployment and only waits for existing deployment to complete, then generates access info.

### 4. Federate Only Mode (`-f`)
Only performs BAW federation with PFS, skipping deployment steps.

## Main Execution Flow

```mermaid
flowchart TD
    Start([Start Script]) --> ParseArgs[Parse Command Line Arguments]
    ParseArgs --> ValidateConfig{Configuration<br/>File Exists?}
    ValidateConfig -->|No| ErrorExit1[Exit with Error]
    ValidateConfig -->|Yes| LoadConfig[Load Configuration File]
    
    LoadConfig --> CheckLDAP{LDAP Config<br/>Provided?}
    CheckLDAP -->|Yes| ValidateLDAP{LDAP File<br/>Exists?}
    ValidateLDAP -->|No| ErrorExit2[Exit with Error]
    ValidateLDAP -->|Yes| LoadLDAP[Load LDAP Config]
    CheckLDAP -->|No| CheckTools
    LoadLDAP --> CheckTools
    
    CheckTools[Check Prerequisite Tools] --> CheckVars[Check Prerequisite Variables]
    CheckVars --> ValidateVars{All Variables<br/>Valid?}
    ValidateVars -->|No| ErrorExit3[Exit with Error]
    ValidateVars -->|Yes| CheckMode{Execution<br/>Mode?}
    
    CheckMode -->|Generate Only| GenerateMode[Generate CR Mode]
    CheckMode -->|Full/Wait/Federate| CheckOCP{Logged into<br/>OCP?}
    
    GenerateMode --> GenCR[Generate CR YAML]
    GenCR --> GenDB[Generate DB Scripts]
    GenDB --> EndGenerate([End - Generate Mode])
    
    CheckOCP -->|No| ErrorExit4[Exit with Error]
    CheckOCP -->|Yes| CheckNS{Namespace<br/>Exists?}
    
    CheckNS -->|No| ErrorExit5[Exit with Error]
    CheckNS -->|Yes| CheckWaitMode{Wait Only<br/>or Federate<br/>Only?}
    
    CheckWaitMode -->|Yes| WaitDeploy[Wait for Deployment Readiness]
    CheckWaitMode -->|No| CreateOutput[Create Output Folder]
    
    CreateOutput --> DeployPre[Deploy Pre-Environment]
    DeployPre --> DeployEnv[Deploy Environment]
    DeployEnv --> DeployPost[Deploy Post-Environment]
    DeployPost --> DeployPFS[Deploy PFS if Enabled]
    DeployPFS --> WaitDeploy
    
    WaitDeploy --> PostInstall[Post Installation Steps]
    PostInstall --> Success([End - Success])
    
    style Start fill:#90EE90
    style Success fill:#90EE90
    style EndGenerate fill:#90EE90
    style ErrorExit1 fill:#FFB6C1
    style ErrorExit2 fill:#FFB6C1
    style ErrorExit3 fill:#FFB6C1
    style ErrorExit4 fill:#FFB6C1
    style ErrorExit5 fill:#FFB6C1
```

## Detailed Component Flow

### 1. Prerequisites Validation Flow

```mermaid
sequenceDiagram
    participant Script
    participant checkPrereqTools
    participant checkPrereqVars
    participant OCP as OpenShift Cluster
    
    Script->>checkPrereqTools: Validate Tools
    checkPrereqTools->>checkPrereqTools: Check jq installed
    alt jq not found
        checkPrereqTools-->>Script: Exit with error
    end
    checkPrereqTools->>checkPrereqTools: Check openssl installed
    alt openssl not found
        checkPrereqTools-->>Script: Warning (continue)
    end
    
    Script->>checkPrereqVars: Validate Variables
    checkPrereqVars->>checkPrereqVars: Check CP4BA_AUTO_CLUSTER_USER
    checkPrereqVars->>checkPrereqVars: Check CP4BA_AUTO_ENTITLEMENT_KEY
    checkPrereqVars->>checkPrereqVars: Check CP4BA_INST_TYPE
    checkPrereqVars->>checkPrereqVars: Validate INST_TYPE (starter/production)
    checkPrereqVars->>checkPrereqVars: Check CP4BA_INST_PLATFORM
    checkPrereqVars->>checkPrereqVars: Validate PLATFORM (OCP/ROKS)
    checkPrereqVars->>OCP: Check Storage Class FILE exists
    checkPrereqVars->>OCP: Check Storage Class BLOCK exists
    checkPrereqVars->>checkPrereqVars: Check LDAP settings if enabled
    
    alt Any validation fails
        checkPrereqVars-->>Script: Exit with error
    else All validations pass
        checkPrereqVars->>checkPrereqVars: Export environment variables
        checkPrereqVars-->>Script: Continue
    end
```

### 2. Pre-Environment Deployment Flow

```mermaid
sequenceDiagram
    participant Script
    participant deployPreEnv
    participant LDAP as LDAP Script
    participant Secrets as Secrets Script
    
    Script->>deployPreEnv: Execute Pre-Deployment
    
    deployPreEnv->>deployPreEnv: Check CP4BA_INST_LDAP flag
    alt LDAP enabled
        deployPreEnv->>deployPreEnv: Check LDAP config provided
        alt LDAP config exists
            deployPreEnv->>LDAP: Call add-ldap.sh
            LDAP-->>deployPreEnv: LDAP deployed
        else No LDAP config
            deployPreEnv-->>Script: Exit with error
        end
    end
    
    deployPreEnv->>deployPreEnv: Check installation type
    alt Not starter (production)
        deployPreEnv->>Secrets: Call cp4ba-create-secrets.sh
        Secrets-->>deployPreEnv: Secrets created
    end
    
    deployPreEnv-->>Script: Pre-deployment complete
```

### 3. Environment Deployment Flow

```mermaid
sequenceDiagram
    participant Script
    participant deployEnvironment
    participant generateCR
    participant OCP as OpenShift Cluster
    
    Script->>deployEnvironment: Deploy Environment
    
    deployEnvironment->>OCP: Get cluster info
    OCP-->>deployEnvironment: Cluster FQDN suffix
    
    deployEnvironment->>generateCR: Generate CR
    generateCR->>generateCR: Run envsubst on template
    generateCR->>generateCR: Check for missed transformations
    alt Variables not substituted
        generateCR-->>Script: Exit with error
    end
    generateCR->>generateCR: Validate YAML with yq
    alt Invalid YAML
        generateCR-->>Script: Exit with error
    end
    generateCR-->>deployEnvironment: CR generated
    
    deployEnvironment->>OCP: Apply CR to namespace
    alt Apply fails
        deployEnvironment-->>Script: Exit with error
    else Apply succeeds
        deployEnvironment-->>Script: Deployment initiated
    end
```

### 4. Post-Environment Deployment Flow

```mermaid
sequenceDiagram
    participant Script
    participant deployPostEnv
    participant DB_Install as DB Install Script
    participant DB_Create as DB Create Script
    
    Script->>deployPostEnv: Execute Post-Deployment
    
    deployPostEnv->>deployPostEnv: Check CP4BA_INST_DB flag
    alt Database enabled
        deployPostEnv->>DB_Install: Call cp4ba-install-db.sh
        DB_Install-->>deployPostEnv: Database installed
        
        deployPostEnv->>DB_Create: Call cp4ba-create-databases.sh
        DB_Create-->>deployPostEnv: Databases created
    end
    
    deployPostEnv-->>Script: Post-deployment complete
```

### 5. PFS Deployment Flow

```mermaid
sequenceDiagram
    participant Script
    participant deployPFS
    participant PFS_Script as pfs-deploy.sh
    participant TempFile as Temp Config File
    
    Script->>deployPFS: Deploy PFS
    
    deployPFS->>deployPFS: Check CP4BA_INST_PFS flag
    alt PFS enabled
        deployPFS->>deployPFS: Check PFS script exists
        alt Script not found
            deployPFS-->>Script: Exit with error
        end
        
        deployPFS->>TempFile: Create temp params file
        deployPFS->>TempFile: Write PFS_NAME
        deployPFS->>TempFile: Write PFS_NAMESPACE
        deployPFS->>TempFile: Write PFS_STORAGE_CLASS
        deployPFS->>TempFile: Write PFS_APP_VER
        deployPFS->>TempFile: Write PFS_ADMINUSER
        
        deployPFS->>PFS_Script: Execute in embedded mode
        PFS_Script-->>deployPFS: PFS deployed
        
        deployPFS->>TempFile: Remove temp file
    end
    
    deployPFS-->>Script: PFS deployment complete
```

### 6. Deployment Readiness Wait Flow

```mermaid
sequenceDiagram
    participant Script
    participant waitDeploymentReadiness
    participant OCP as OpenShift Cluster
    participant federateBawsInDeployment
    
    Script->>waitDeploymentReadiness: Wait for Readiness
    
    loop Until Ready
        waitDeploymentReadiness->>OCP: Check PFS ready (if enabled)
        waitDeploymentReadiness->>OCP: Check ICP4ACluster CR status
        waitDeploymentReadiness->>OCP: Check Content CR status
        
        alt All CRs Ready and PFS Ready
            waitDeploymentReadiness->>waitDeploymentReadiness: Break loop
        else Not ready
            waitDeploymentReadiness->>OCP: Check pending PVCs
            waitDeploymentReadiness->>OCP: Check operator failures
            waitDeploymentReadiness->>waitDeploymentReadiness: Display progress
            waitDeploymentReadiness->>waitDeploymentReadiness: Sleep 1 second
        end
    end
    
    alt Federate enabled and workflow pattern
        waitDeploymentReadiness->>federateBawsInDeployment: Federate BAW instances
        federateBawsInDeployment-->>waitDeploymentReadiness: Federation complete
    end
    
    loop Until access-info ConfigMap exists
        waitDeploymentReadiness->>OCP: Check for access-info ConfigMap
        alt ConfigMap found
            waitDeploymentReadiness->>waitDeploymentReadiness: Break loop
        else Not found
            waitDeploymentReadiness->>waitDeploymentReadiness: Sleep 1 second
        end
    end
    
    waitDeploymentReadiness->>OCP: Get access-info ConfigMap
    waitDeploymentReadiness->>OCP: Get admin credentials
    waitDeploymentReadiness->>waitDeploymentReadiness: Write access info to file
    
    alt ZenService configuration enabled
        waitDeploymentReadiness->>waitDeploymentReadiness: Update ZenService certificate
        waitDeploymentReadiness->>waitDeploymentReadiness: Check if cert in trusted list
        alt Cert in trusted list
            waitDeploymentReadiness->>waitDeploymentReadiness: Restart StatefulSets
        end
    end
    
    waitDeploymentReadiness-->>Script: Deployment ready
```

### 7. BAW Federation Flow

```mermaid
sequenceDiagram
    participant federateBawsInDeployment
    participant federateBaw
    participant OCP as OpenShift Cluster
    participant CR as ICP4ACluster CR
    
    federateBawsInDeployment->>federateBawsInDeployment: Loop through BAW instances (1-10)
    
    loop For each BAW instance
        federateBawsInDeployment->>federateBawsInDeployment: Check if instance enabled
        federateBawsInDeployment->>federateBawsInDeployment: Check if federation enabled
        
        alt Instance and federation enabled
            federateBawsInDeployment->>OCP: Wait for StatefulSet ready
            federateBawsInDeployment->>federateBaw: Federate BAW instance
            
            federateBaw->>OCP: Get PFS CR name
            federateBaw->>OCP: Wait for PFS resource created
            federateBaw->>OCP: Wait for PFS ready
            
            federateBaw->>OCP: Get PFS endpoints
            federateBaw->>federateBaw: Extract PFS host and context
            
            federateBaw->>CR: Get current CR JSON
            federateBaw->>federateBaw: Extract BAW section
            federateBaw->>federateBaw: Update federation settings
            federateBaw->>federateBaw: Remove old BAW section
            federateBaw->>federateBaw: Add updated BAW section
            
            loop Retry up to 10 times
                federateBaw->>OCP: Apply updated CR
                alt Apply succeeds
                    federateBaw->>federateBaw: Break retry loop
                else Apply fails
                    federateBaw->>federateBaw: Sleep and retry
                end
            end
            
            federateBaw->>federateBaw: Cleanup temp files
            federateBaw-->>federateBawsInDeployment: Federation complete
        end
    end
    
    federateBawsInDeployment-->>Script: All federations complete
```

### 8. Post Installation Steps Flow

```mermaid
sequenceDiagram
    participant Script
    participant postInstallationSteps
    participant GenAI as GenAI Config Script
    participant BAI as BAI Workforce Script
    
    Script->>postInstallationSteps: Execute Post-Installation
    
    postInstallationSteps->>postInstallationSteps: Check CP4BA_INST_GENAI_ENABLED
    alt GenAI enabled
        postInstallationSteps->>GenAI: Call cp4ba-configure-genai.sh
        GenAI-->>postInstallationSteps: GenAI configured
    end
    
    postInstallationSteps->>postInstallationSteps: Check CP4BA_INST_BAI_BPC_WORKFORCE
    alt BAI Workforce enabled
        postInstallationSteps->>BAI: Call cp4ba-configure-bai-workforce.sh
        BAI-->>postInstallationSteps: BAI Workforce configured
    end
    
    postInstallationSteps-->>Script: Post-installation complete
```

## Key Functions Reference

### Utility Functions

| Function | Purpose | Parameters | Returns |
|----------|---------|------------|---------|
| `checkPrereqTools()` | Validates required tools (jq, openssl) | None | Exit on error |
| `checkPrereqVars()` | Validates required environment variables | None | Exit on error |
| `namespaceExist()` | Check if namespace exists | `$1`: namespace name | 0=not exist, 1=exists |
| `storageClassExist()` | Check if storage class exists | `$1`: storage class name | 0=not exist, 1=exists |
| `resourceExist()` | Check if resource exists | `$1`: namespace, `$2`: resource type, `$3`: resource name | 0=not exist, 1=exists |
| `waitForResourceCreated()` | Wait until resource is created | `$1`: namespace, `$2`: resource type, `$3`: resource name, `$4`: sleep interval | None |

### Deployment Functions

| Function | Purpose | Key Actions |
|----------|---------|-------------|
| `deployPreEnv()` | Pre-deployment setup | - Deploy LDAP if enabled<br>- Create secrets for production |
| `deployEnvironment()` | Main deployment | - Get cluster info<br>- Generate CR<br>- Apply CR to cluster |
| `deployPostEnv()` | Post-deployment setup | - Install database<br>- Create databases |
| `deployPFS()` | Deploy Process Federation Server | - Create temp config<br>- Execute PFS deployment script |
| `generateCR()` | Generate Custom Resource YAML | - Substitute environment variables<br>- Validate YAML<br>- Check for missed transformations |

### Monitoring Functions

| Function | Purpose | Key Actions |
|----------|---------|-------------|
| `waitDeploymentReadiness()` | Wait for deployment completion | - Monitor CR status<br>- Check PFS readiness<br>- Federate BAW instances<br>- Generate access info |
| `loopWaitForPfsReady()` | Wait for PFS to be ready | - Check PFS components status<br>- Loop until all ready |
| `checkForPfsReady()` | Check PFS readiness status | - Query PFS deployment, service, zen integration |
| `waitForBawStatefulSetReady()` | Wait for BAW StatefulSet | - Wait for StatefulSet creation<br>- Wait for ready replicas |

### Federation Functions

| Function | Purpose | Key Actions |
|----------|---------|-------------|
| `federateBaw()` | Federate single BAW instance | - Get PFS endpoints<br>- Update CR with federation config<br>- Retry on failure |
| `federateBawsInDeployment()` | Federate all BAW instances | - Loop through BAW 1-10<br>- Call federateBaw for each enabled instance |

### Post-Installation Functions

| Function | Purpose | Key Actions |
|----------|---------|-------------|
| `postInstallationSteps()` | Execute post-installation tasks | - Configure GenAI if enabled<br>- Configure BAI Workforce if enabled |
| `updateZenServiceCertificate()` | Update Zen certificate | - Call TLS update script<br>- Return configuration status |
| `zenCertInTrustedList()` | Check and restart if cert trusted | - Check if cert in trusted list<br>- Restart StatefulSets if needed |
| `restartStatefulSets()` | Restart StatefulSets | - Scale to 0<br>- Scale back to original replicas |

## Execution Branches

### Branch 1: Generate Only Mode

```mermaid
graph LR
    A[Start] --> B[Parse Args -g]
    B --> C[Load Config]
    C --> D[Check Prerequisites]
    D --> E[Generate CR]
    E --> F[Generate DB Scripts]
    F --> G[Display Warnings]
    G --> H[End]
    
    style A fill:#90EE90
    style H fill:#90EE90
```

**Conditions**: `-g` flag provided  
**Actions**:
- Generate CR YAML from template
- Generate database creation scripts
- Display warnings about manual prerequisites
- Exit without deployment

### Branch 2: Wait Only Mode

```mermaid
graph LR
    A[Start] --> B[Parse Args -w]
    B --> C[Load Config]
    C --> D[Check Prerequisites]
    D --> E[Check OCP Login]
    E --> F[Check Namespace]
    F --> G[Wait for Readiness]
    G --> H[Generate Access Info]
    H --> I[End]
    
    style A fill:#90EE90
    style I fill:#90EE90
```

**Conditions**: `-w` flag provided  
**Actions**:
- Skip all deployment steps
- Wait for existing deployment to complete
- Generate access information file
- Exit

### Branch 3: Federate Only Mode

```mermaid
graph LR
    A[Start] --> B[Parse Args -f]
    B --> C[Load Config]
    C --> D[Check Prerequisites]
    D --> E[Check OCP Login]
    E --> F[Check Namespace]
    F --> G[Wait for Readiness]
    G --> H[Federate BAW Instances]
    H --> I[Generate Access Info]
    I --> J[End]
    
    style A fill:#90EE90
    style J fill:#90EE90
```

**Conditions**: `-f` flag provided  
**Actions**:
- Skip deployment steps
- Wait for deployment readiness
- Federate BAW instances with PFS
- Generate access information file
- Exit

### Branch 4: Full Deployment Mode

```mermaid
graph TD
    A[Start] --> B[Parse Args]
    B --> C[Load Config]
    C --> D[Check Prerequisites]
    D --> E[Check OCP Login]
    E --> F[Check Namespace]
    F --> G[Deploy Pre-Environment]
    G --> H[Deploy Environment]
    H --> I[Deploy Post-Environment]
    I --> J[Deploy PFS]
    J --> K[Wait for Readiness]
    K --> L[Federate BAW if enabled]
    L --> M[Post Installation Steps]
    M --> N[Generate Access Info]
    N --> O[End]
    
    style A fill:#90EE90
    style O fill:#90EE90
```

**Conditions**: No special flags (default mode)  
**Actions**:
- Complete deployment lifecycle
- All prerequisites setup
- CR deployment
- Post-deployment configuration
- Federation if enabled
- Post-installation steps
- Access information generation

## Configuration Variables

### Required Variables

| Variable | Description | Example |
|----------|-------------|---------|
| `CP4BA_AUTO_CLUSTER_USER` | OCP cluster username | `admin` |
| `CP4BA_AUTO_ENTITLEMENT_KEY` | IBM entitlement key | `eyJhbGc...` |
| `CP4BA_INST_TYPE` | Installation type | `starter` or `production` |
| `CP4BA_INST_PLATFORM` | Platform type | `OCP` or `ROKS` |
| `CP4BA_INST_SC_FILE` | File storage class | `ocs-storagecluster-cephfs` |
| `CP4BA_INST_SC_BLOCK` | Block storage class | `ocs-storagecluster-ceph-rbd` |
| `CP4BA_INST_NAMESPACE` | Target namespace | `cp4ba-prod` |
| `CP4BA_INST_ENV` | Environment name | `production` |
| `CP4BA_INST_CR_NAME` | CR instance name | `icp4adeploy` |
| `CP4BA_INST_CR_TEMPLATE` | CR template path | `templates/cp4ba-cr-ref-baw.yaml` |
| `CP4BA_INST_OUTPUT_FOLDER` | Output directory | `../output` |
| `CP4BA_INST_APPVER` | Application version | `25.0.1` |

### Optional Variables

| Variable | Description | Default |
|----------|-------------|---------|
| `CP4BA_INST_LDAP` | Enable LDAP deployment | `false` |
| `CP4BA_INST_LDAP_SECRET` | LDAP secret name | - |
| `CP4BA_INST_DB` | Enable database deployment | `false` |
| `CP4BA_INST_PFS` | Enable PFS deployment | `false` |
| `CP4BA_INST_GENAI_ENABLED` | Enable GenAI configuration | `false` |
| `CP4BA_INST_BAI_BPC_WORKFORCE` | Enable BAI Workforce | `false` |
| `CP4BA_INST_ZS_CONFIGURE` | Configure ZenService certificate | `false` |

### BAW Federation Variables (1-10 instances)

| Variable Pattern | Description |
|-----------------|-------------|
| `CP4BA_INST_BAW_N` | Enable BAW instance N |
| `CP4BA_INST_BAW_N_NAME` | BAW instance N name |
| `CP4BA_INST_BAW_N_FEDERATED` | Enable federation for instance N |
| `CP4BA_INST_BAW_N_HOST_FEDERATED_PORTAL` | Host federated portal flag |

## Error Handling

### Exit Codes

| Code | Condition |
|------|-----------|
| `1` | Configuration file not found |
| `1` | LDAP configuration file not found |
| `1` | Required tool (jq) not installed |
| `1` | Required variables not set |
| `1` | Storage class not found |
| `1` | Not logged into OCP cluster |
| `1` | Namespace doesn't exist |
| `1` | CR generation failed |
| `1` | CR deployment failed |
| `1` | LDAP deployment failed |
| `1` | Secrets creation failed |
| `1` | Database installation failed |
| `1` | Database creation failed |
| `1` | PFS deployment failed |
| `1` | GenAI configuration failed |
| `1` | BAI Workforce configuration failed |
| `1` | Federation failed |
| `0` | Success |

### Validation Checks

1. **Tool Validation**
   - `jq` must be installed (critical)
   - `openssl` should be installed (warning only)

2. **Variable Validation**
   - Cluster user and entitlement key must be set
   - Installation type must be 'starter' or 'production'
   - Platform must be 'OCP' or 'ROKS'
   - Storage classes must exist in cluster
   - LDAP secret required if LDAP enabled

3. **File Validation**
   - Configuration file must exist
   - LDAP configuration file must exist (if provided)
   - CR template must exist
   - Generated CR must be valid YAML
   - All environment variables must be substituted

4. **Cluster Validation**
   - Must be logged into OCP cluster
   - Target namespace must exist
   - Storage classes must be available

## Output Files

| File | Location | Content |
|------|----------|---------|
| CR YAML | `${CP4BA_INST_OUTPUT_FOLDER}/cp4ba-${CR_NAME}-${ENV}.yaml` | Generated Custom Resource |
| Access Info | `${CP4BA_INST_OUTPUT_FOLDER}/cp4ba-${CR_NAME}-${ENV}-access-info.txt` | Platform URLs and credentials |
| DB Scripts | Generated by `cp4ba-create-databases.sh` | Database creation scripts |

## Monitoring and Progress

### Progress Indicators

The script displays real-time progress with:
- **Rotating character**: Visual indicator of activity (`|/-\`)
- **Warning count**: Number of pending PVCs and operator failures
- **Elapsed time**: Hours, minutes, and seconds since start
- **Status messages**: Color-coded messages for different stages

### Status Checks

The script monitors:
1. **ICP4ACluster CR status**: Ready condition
2. **Content CR status**: Ready condition (if applicable)
3. **PFS status**: Deployment, Service, and Zen Integration readiness
4. **Pending PVCs**: Count of pending persistent volume claims
5. **Operator failures**: Count of FAIL messages in operator logs
6. **Access-info ConfigMap**: Availability of access information

## Dependencies and Integration

### External Scripts Called

1. **LDAP Management**
   - `${CP4BA_INST_LDAP_TOOLS_FOLDER}/add-ldap.sh`

2. **Secrets Management**
   - `./cp4ba-create-secrets.sh`

3. **Database Management**
   - `./cp4ba-install-db.sh`
   - `./cp4ba-create-databases.sh`

4. **PFS Management**
   - `${CP4BA_INST_PFS_TOOLS_FOLDER}/scripts/pfs-deploy.sh`

5. **Post-Installation**
   - `./cp4ba-configure-genai.sh`
   - `./cp4ba-configure-bai-workforce.sh`

6. **Certificate Management**
   - `${CP4BA_INST_UTILS_TOOLS_FOLDER}/cp4ba-tls-update-ep.sh`

### OpenShift CLI Commands Used

- `oc whoami`: Verify login status
- `oc cluster-info`: Get cluster information
- `oc get`: Query resources (namespaces, storage classes, CRs, pods, etc.)
- `oc apply`: Apply Custom Resources
- `oc scale`: Scale StatefulSets
- `oc logs`: Check operator logs

## Best Practices and Recommendations

1. **Configuration Management**
   - Keep configuration files in version control
   - Use separate configs for different environments
   - Document all custom variables

2. **Pre-Deployment**
   - Verify all prerequisites before running
   - Test in non-production environment first
   - Ensure sufficient cluster resources

3. **Monitoring**
   - Watch for warnings during deployment
   - Check operator logs for errors
   - Monitor PVC binding status

4. **Post-Deployment**
   - Save access-info file securely
   - Verify all components are ready
   - Test federation if enabled

5. **Troubleshooting**
   - Use `-g` flag to generate and inspect CR before deployment
   - Check generated YAML for variable substitution issues
   - Review operator logs for detailed error messages

## Troubleshooting Guide

### Common Issues

| Issue | Possible Cause | Solution |
|-------|---------------|----------|
| "jq not installed" | Missing prerequisite | Install jq: `yum install jq` or `apt-get install jq` |
| "Not logged in to OCP" | No active OCP session | Run `oc login` with cluster credentials |
| "Namespace doesn't exist" | Target namespace not created | Create namespace or check configuration |
| "Storage class not found" | Invalid storage class name | Verify with `oc get sc` and update config |
| "CR deployment failed" | Invalid YAML or permissions | Check YAML with `yq` and verify permissions |
| "Pending PVCs" | Storage provisioning issues | Check storage class and cluster capacity |
| "Operator failures" | Configuration errors | Review operator logs for details |
| "Federation failed" | PFS not ready or config error | Verify PFS deployment and endpoints |

### Debug Mode

To enable detailed debugging:
```bash
# Uncomment at top of script
set -x  # Enable command tracing
```

### Log Analysis

Check operator logs:
```bash
oc logs -n ${NAMESPACE} -c operator $(oc get pods -n ${NAMESPACE} | grep "cp4a-operator-" | awk '{print $1}')
```

## Version History and Compatibility

This script is designed for:
- **CP4BA Version**: 23.x, 24.x, 25.x
- **OpenShift**: 4.10+
- **ROKS**: Compatible
- **Bash**: 4.0+

## Security Considerations

1. **Credentials**: Never commit entitlement keys or passwords to version control
2. **Access Info**: Protect access-info files containing credentials
3. **LDAP Secrets**: Ensure LDAP secrets are properly secured
4. **Certificates**: Manage TLS certificates according to security policies
5. **RBAC**: Ensure proper OpenShift permissions for deployment

## Conclusion

The `cp4ba-deploy-env.sh` script provides a comprehensive, automated approach to deploying CP4BA environments. Its modular design, extensive validation, and flexible execution modes make it suitable for various deployment scenarios from development to production environments.