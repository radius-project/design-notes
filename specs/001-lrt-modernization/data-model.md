# Data Model: Long-Running Test System Modernization

**Feature**: LRT System Modernization  
**Date**: November 16, 2025  
**Status**: Draft

## Overview

This document defines the data entities, structures, and relationships for the Long-Running Test system. Since this is infrastructure testing tooling rather than a runtime service, the "data model" consists primarily of configuration structures, state representations, and result artifacts used by deployment scripts and test execution.

## Core Entities

### 1. Deployment Configuration

Represents the parameters and settings for deploying test infrastructure.

**Fields**:
- `resourceGroupName` (string, required): Azure resource group name, typically `{username}-radius-lrt`
- `location` (string, required): Azure region for deployment (default: `westus3`)
- `clusterName` (string, required): AKS cluster name (default: `{prefix}-aks`)
- `logAnalyticsWorkspaceName` (string, optional): Log Analytics workspace name (default: `{prefix}-workspace`)
- `grafanaEnabled` (boolean, optional): Enable Grafana monitoring (default: false)
- `grafanaAdminObjectId` (string, conditional): Required if grafanaEnabled=true
- `installKubernetesDependencies` (boolean, optional): Install Dapr, cert-manager, etc. (default: true)
- `enableDeletionLock` (boolean, optional): Prevent accidental deletion (default: true)
- `tags` (map[string]string, optional): Resource tags for lifecycle management (default: `{radius: "infra"}`)

**Validation Rules**:
- `resourceGroupName` must be unique within subscription
- `location` must be valid Azure region with AKS availability
- `grafanaAdminObjectId` must be valid Azure AD object ID if `grafanaEnabled=true`
- `tags` must include retention policy if `enableDeletionLock=true`

**State Transitions**:
```
Not Configured -> Validated -> Deploying -> Deployed -> Destroying -> Destroyed
                     ↓              ↓
                   Failed        Failed
```

**Persistence**: Environment variables in shell scripts, passed as Bicep parameters via Azure CLI

**Relationships**:
- One Deployment Configuration → One AKS Cluster State
- One Deployment Configuration → One Test Configuration (for test execution)

---

### 2. AKS Cluster State

Represents the current state and metadata of a provisioned test cluster.

**Fields**:
- `clusterName` (string, required): AKS cluster name
- `resourceGroup` (string, required): Containing resource group
- `subscriptionId` (string, required): Azure subscription ID
- `kubernetesVersion` (string, required): K8s version (e.g., "1.31.8")
- `radiusVersion` (string, optional): Installed Radius version (e.g., "v0.35.0")
- `provisioningState` (string, required): Azure provisioning state (Succeeded, Failed, Updating)
- `healthStatus` (enum, required): `healthy` | `degraded` | `failed` | `unknown`
- `createdAt` (timestamp, required): Cluster creation time
- `lastValidated` (timestamp, optional): Last health check timestamp
- `oidcIssuerUrl` (string, optional): OIDC issuer URL for workload identity
- `workloadIdentityEnabled` (boolean, required): Whether workload identity is configured
- `dependencies` (object, required): Status of installed dependencies
  - `daprInstalled` (boolean)
  - `certManagerInstalled` (boolean)
  - `logAnalyticsConnected` (boolean)
  - `grafanaConfigured` (boolean)
- `testResources` (object, optional): Pre-provisioned test resource connections
  - `cosmosDbConfigured` (boolean)
  - `sqlServerConfigured` (boolean)
  - `awsConfigured` (boolean)

**Validation Rules**:
- `provisioningState` must be "Succeeded" before test execution
- `radiusVersion` must be semantic version string (e.g., "v0.35.0")
- `healthStatus=healthy` requires all critical dependencies installed
- `testResources` validation depends on test suite requirements

**State Transitions**:
```
Creating -> Provisioning -> Running -> Upgrading -> Running
             ↓                ↓           ↓
           Failed         Degraded    Failed
                             ↓
                          Running (after remediation)
```

**Persistence**: Retrieved dynamically via `az aks show`, `rad version --server`, `kubectl get pods`

**Relationships**:
- One AKS Cluster State → Many Test Results (historical execution records)
- One AKS Cluster State → One Workload Identity (via tag-based lookup)

---

### 3. Test Configuration

Represents the configuration for executing a test suite.

**Fields**:
- `targetCluster` (object, required): Reference to target AKS cluster
  - `name` (string)
  - `resourceGroup` (string)
  - `subscriptionId` (string)
- `radiusVersion` (string, required): Expected Radius version for tests
- `testSuites` (array[string], required): Test suites to execute (e.g., `["corerp", "daprrp"]`)
- `executionMode` (enum, required): `serial` | `parallel`
- `concurrency` (int, conditional): Required if executionMode=parallel (default: 2)
- `timeout` (duration, required): Overall test timeout (default: "1h")
- `failFast` (boolean, optional): Stop on first failure (default: false)
- `containerRegistry` (string, required): Registry for pulling test images
- `awsCredentials` (object, optional): AWS access for cloud tests
  - `accountId` (string)
  - `region` (string)
  - `accessKeySecretRef` (string): Kubernetes secret reference
- `skipCleanup` (boolean, optional): Preserve resources after tests (default: false)

**Validation Rules**:
- `testSuites` array must not be empty
- `concurrency` must be 1-10 if executionMode=parallel
- `timeout` must be positive duration parseable by Go time.Duration
- `radiusVersion` must match cluster version or trigger upgrade
- `containerRegistry` must be accessible (validated at start)

**State Transitions**:
```
Configured -> Validating -> Ready -> Executing -> Complete
                 ↓                       ↓           ↓
              Invalid                 Failed      Failed
```

**Persistence**: Environment variables, Make parameters, config files (future: YAML config)

**Relationships**:
- One Test Configuration → One Test Results
- One Test Configuration → One AKS Cluster State (target cluster)

---

### 4. Test Results

Represents the outcomes of test execution.

**Fields**:
- `sessionId` (string, required): Unique test session identifier (UUID or timestamp-based)
- `startTime` (timestamp, required): Test execution start time
- `endTime` (timestamp, optional): Test execution completion time
- `duration` (duration, optional): Total execution time
- `overallStatus` (enum, required): `passed` | `failed` | `partial` | `skipped`
- `radiusVersion` (string, required): Tested Radius version
- `clusterInfo` (object, required): Target cluster identification
  - `name` (string)
  - `resourceGroup` (string)
  - `kubernetesVersion` (string)
- `suiteResults` (array[SuiteResult], required): Per-suite test results
- `failedTests` (array[TestFailure], optional): Details of failed tests
- `diagnosticArtifacts` (array[string], optional): Paths to diagnostic outputs (terminal display only)

**Nested Entity: SuiteResult**:
- `suiteName` (string, required): Test suite identifier (e.g., "corerp-cloud")
- `status` (enum, required): `passed` | `failed` | `skipped`
- `passCount` (int, required): Number of passed tests
- `failCount` (int, required): Number of failed tests
- `skipCount` (int, required): Number of skipped tests
- `duration` (duration, required): Suite execution time

**Nested Entity: TestFailure**:
- `testName` (string, required): Fully qualified test name
- `suiteName` (string, required): Containing suite
- `errorMessage` (string, required): Failure reason
- `stackTrace` (string, optional): Error stack trace
- `podLogs` (string, optional): Relevant pod logs (terminal output)
- `resourceState` (string, optional): Kubernetes resource state dump

**Validation Rules**:
- `overallStatus=passed` requires all suiteResults.status=passed
- `duration` must equal (endTime - startTime)
- `failedTests` array must not be empty if overallStatus=failed

**State Transitions**:
```
Initialized -> InProgress -> Collecting Results -> Complete
                    ↓                                ↓
                 Interrupted                    Partial/Failed
```

**Persistence**: 
- Structured results in JUnit XML format (FR-015)
- Terminal output with real-time progress
- No long-term persistent storage (FR-019)

**Relationships**:
- One Test Results → One Test Configuration
- One Test Results → Many Test Failures (if any)

---

### 5. Test Resource Credentials

Represents connection information for pre-provisioned test resources.

**Fields**:
- `cosmosDb` (object, optional): CosmosDB MongoDB connection
  - `connectionString` (secret, required): MongoDB connection string
  - `databaseName` (string, required): Database name for tests
- `sqlServer` (object, optional): Azure SQL Server connection
  - `resourceId` (string, required): SQL Server resource ID
  - `serverName` (string, required): SQL Server hostname
  - `username` (secret, required): SQL authentication username
  - `password` (secret, required): SQL authentication password
  - `databaseName` (string, required): Database name for tests
- `aws` (object, optional): AWS resource access
  - `accountId` (string, required): AWS account ID
  - `region` (string, required): AWS region
  - `accessKeyId` (secret, required): AWS access key ID
  - `secretAccessKey` (secret, required): AWS secret access key

**Storage Location**:
- All secrets stored in Kubernetes Secret resources (not environment variables)
- Secret names: `radius-test-cosmosdb`, `radius-test-sql`, `radius-test-aws`
- Namespace: `radius-system` (or test-specific namespace)
- Access: Service accounts with RBAC read permissions

**Validation Rules**:
- All secrets must be base64-encoded when stored in Kubernetes
- Connection strings must be valid format for respective services
- Service accounts must have read-only access (no write/delete permissions)

**Relationships**:
- One Test Resource Credentials → Many Test Pods (via service account volume mount)
- Credentials created by `setup-credentials.sh` script during infrastructure provisioning

---

### 6. Workload Identity

Represents Azure workload identity for OIDC authentication.

**Fields**:
- `identityName` (string, required): Azure managed identity name
- `resourceGroup` (string, required): Identity resource group
- `clientId` (string, required): Application (client) ID
- `tenantId` (string, required): Azure AD tenant ID
- `tags` (map[string]string, required): Tags for discovery (e.g., `{purpose: "radius-lrt"}`)
- `federatedCredentials` (array[object], required): OIDC federation config
  - `name` (string)
  - `issuer` (string): AKS OIDC issuer URL
  - `subject` (string): Kubernetes service account subject

**Validation Rules**:
- Identity must exist before deployment (FR-040)
- Tag selector must uniquely identify exactly one identity
- Federated credentials must match cluster OIDC issuer
- Identity must have required Azure RBAC roles (Contributor on resource group)

**State Transitions**:
```
Not Found -> Discovered -> Bound to Cluster -> Active
   ↓
Failed (deployment aborted)
```

**Persistence**: Azure resource, discovered via `az identity list --query` with tag filter

**Relationships**:
- One Workload Identity → Many AKS Clusters (reused across deployments)
- One Workload Identity → Many Kubernetes Service Accounts (federation)

---

## Entity Relationship Diagram

```
┌─────────────────────────┐
│ Deployment Configuration│
└───────────┬─────────────┘
            │ configures
            ▼
┌─────────────────────────┐      ┌──────────────────────┐
│   AKS Cluster State     │◄─────┤ Workload Identity    │
└───────────┬─────────────┘      └──────────────────────┘
            │ uses                   (reused, tag-based)
            │
            ▼
┌─────────────────────────┐
│   Test Configuration    │
└───────────┬─────────────┘
            │ targets
            │
            ▼
┌─────────────────────────┐      ┌──────────────────────────┐
│     Test Results        │──────┤  Test Resource Credentials│
└───────────┬─────────────┘      └──────────────────────────┘
            │                       (Kubernetes Secrets + RBAC)
            │ contains
            ▼
┌─────────────────────────┐
│   Suite Results         │
│   (per test suite)      │
└───────────┬─────────────┘
            │ contains
            ▼
┌─────────────────────────┐
│    Test Failures        │
│  (diagnostic details)   │
└─────────────────────────┘
```

## Configuration File Formats

### Environment Variables (Deployment)

```bash
# Required
export AZURE_SUBSCRIPTION_ID="..."
export TEST_AKS_RG="username-radius-lrt"
export TEST_AKS_AZURE_LOCATION="westus3"

# Optional
export GRAFANA_ENABLED="false"
export GRAFANA_ADMIN_OBJECT_ID=""
export ENABLE_DELETION_LOCK="true"
export SKIP_CLEANUP="false"
```

### Kubernetes Secret (CosmosDB Example)

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: radius-test-cosmosdb
  namespace: radius-system
type: Opaque
data:
  connectionString: <base64-encoded-connection-string>
  databaseName: <base64-encoded-database-name>
```

### JUnit XML Test Results (Output Format)

```xml
<?xml version="1.0" encoding="UTF-8"?>
<testsuites>
  <testsuite name="corerp-cloud" tests="45" failures="2" errors="0" time="1234.56">
    <testcase name="TestAzureStorageAccount" classname="corerp.cloud" time="12.34">
      <failure message="resource deployment timeout">...</failure>
    </testcase>
    <testcase name="TestAzureKeyVault" classname="corerp.cloud" time="8.92"/>
  </testsuite>
</testsuites>
```

## Data Flow

### Deployment Flow
```
Environment Variables (Deployment Config)
  → Bash Script Validation
    → Bicep Parameters
      → Azure Resource Manager
        → AKS Cluster State (created)
          → Kubernetes Secrets (Test Credentials)
```

### Test Execution Flow
```
Make Target (Test Configuration)
  → Environment Variables
    → gotestsum (Test Runner)
      → magpiego Tests (Go)
        → Kubernetes API (reads Cluster State)
          → Kubernetes Secrets via RBAC (reads Test Credentials)
            → Test Results (JUnit XML + terminal output)
```

### Health Check Flow
```
health-check.sh
  → az aks show (reads Cluster State)
  → rad version --server (reads Radius version)
  → kubectl get pods (reads dependency state)
  → kubectl get secrets (validates credential availability)
    → Health Status (healthy/degraded/failed)
```

## Validation Rules Summary

| Entity | Critical Validations |
|--------|---------------------|
| Deployment Configuration | Unique resource group, valid location, Azure quota availability |
| AKS Cluster State | provisioningState=Succeeded, healthStatus=healthy, all dependencies installed |
| Test Configuration | Non-empty test suites, valid timeout, registry accessible, cluster version match |
| Test Results | Duration consistency, failure details if overallStatus=failed |
| Test Resource Credentials | All secrets base64-encoded, service account RBAC configured |
| Workload Identity | Tag uniquely identifies one identity, federated credentials configured |

## Future Enhancements (Not in Initial Implementation)

- **Configuration File**: YAML-based test configuration instead of environment variables
- **Result Storage**: Optional persistent storage for historical test results and trends
- **Multi-Cloud Credentials**: GCP and AWS credential structures for expanded cloud testing
- **Performance Metrics**: Resource utilization tracking during test execution
- **Test Sharding**: Distribute tests across multiple clusters for faster execution
