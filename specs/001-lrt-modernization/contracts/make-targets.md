# Make Target Contracts

This document defines the public interface (contract) for all Make targets in the LRT system. These targets form the stable API for developers and CI systems to interact with the testing infrastructure.

## Infrastructure Management Targets

### `make lrt-deploy-infra`

**Purpose**: Deploy complete AKS test infrastructure to Azure.

**Prerequisites**:
- Azure CLI installed and authenticated
- Valid Azure subscription
- Required environment variables configured

**Required Environment Variables**:
```bash
AZURE_SUBSCRIPTION_ID    # Azure subscription ID
TEST_AKS_RG              # Resource group name (default: $(whoami)-radius-lrt)
TEST_AKS_AZURE_LOCATION  # Azure region (default: westus3)
```

**Optional Environment Variables**:
```bash
GRAFANA_ENABLED          # Enable Grafana (default: false)
GRAFANA_ADMIN_OBJECT_ID  # Grafana admin Azure AD object ID (required if GRAFANA_ENABLED=true)
ENABLE_DELETION_LOCK     # Prevent accidental deletion (default: true)
```

**Exit Codes**:
- `0`: Deployment successful
- `1`: Configuration validation failed
- `2`: Azure CLI authentication failed
- `3`: Insufficient Azure permissions
- `4`: Bicep deployment failed
- `5`: Health check failed after deployment

**Output**:
- Prints deployment progress to stdout
- Prints cluster connection instructions on success
- Prints detailed error messages on failure

**Side Effects**:
- Creates Azure resource group
- Deploys AKS cluster with all dependencies
- Updates kubeconfig with cluster credentials
- Creates Kubernetes secrets for test resources (if setup-credentials.sh runs)

**Example Usage**:
```bash
export AZURE_SUBSCRIPTION_ID="12345678-1234-1234-1234-123456789abc"
export TEST_AKS_RG="myname-radius-lrt"
export TEST_AKS_AZURE_LOCATION="eastus"
make lrt-deploy-infra
```

**Idempotency**: Re-running against existing resource group resumes partial deployment or reports current state.

---

### `make lrt-destroy-infra`

**Purpose**: Delete all Azure resources created by `make lrt-deploy-infra`.

**Prerequisites**:
- Azure CLI installed and authenticated
- Resource group exists

**Required Environment Variables**:
```bash
TEST_AKS_RG    # Resource group name to delete
```

**Optional Environment Variables**:
```bash
FORCE_DESTROY  # Skip confirmation prompt (default: false)
```

**Exit Codes**:
- `0`: Deletion successful
- `1`: Resource group not found
- `2`: Deletion failed (Azure error)
- `3`: Resource retention lock present (user must remove manually)

**Output**:
- Prints confirmation prompt (unless FORCE_DESTROY=true)
- Prints deletion progress
- Prints completion message with cleanup time

**Side Effects**:
- Deletes entire Azure resource group and all contained resources
- Removes cluster from kubeconfig (if present)

**Example Usage**:
```bash
export TEST_AKS_RG="myname-radius-lrt"
make lrt-destroy-infra

# Or force without confirmation
FORCE_DESTROY=true make lrt-destroy-infra
```

**Safety Mechanisms**:
- Checks for retention tags before deletion
- Prompts for confirmation (unless FORCE_DESTROY=true)
- Validates resource group contains only test resources

---

### `make lrt-health-check`

**Purpose**: Validate test environment is healthy and ready for test execution.

**Prerequisites**:
- kubectl configured for target cluster
- Radius CLI installed

**Required Environment Variables**:
```bash
# None - uses current kubeconfig context
```

**Exit Codes**:
- `0`: All health checks passed
- `1`: Cluster connectivity failed
- `2`: Radius installation not found or wrong version
- `3`: Required Kubernetes resources missing
- `4`: Test resource credentials unavailable
- `5`: Cluster in degraded state

**Output**:
- Prints health check results for each component:
  - Cluster connectivity
  - Radius installation and version
  - Required Kubernetes dependencies (Dapr, cert-manager)
  - Test resource availability (CosmosDB, SQL secrets)
  - Node resource availability
- Prints actionable error messages for failures

**Side Effects**: None (read-only validation)

**Example Usage**:
```bash
# Check currently configured cluster
make lrt-health-check

# Check specific cluster context
kubectl config use-context my-test-cluster
make lrt-health-check
```

**Expected Duration**: < 2 minutes

---

### `make lrt-setup-test-resources`

**Purpose**: Provision test resources (CosmosDB, SQL Server) and create Kubernetes secrets.

**Prerequisites**:
- Azure CLI authenticated
- kubectl configured for target cluster
- AKS cluster already deployed

**Required Environment Variables**:
```bash
AZURE_SUBSCRIPTION_ID  # Azure subscription for resource provisioning
TEST_AKS_RG            # Resource group for test resources
```

**Optional Environment Variables**:
```bash
COSMOSDB_ACCOUNT_NAME  # Custom CosmosDB account name (default: auto-generated)
SQL_SERVER_NAME        # Custom SQL server name (default: auto-generated)
```

**Exit Codes**:
- `0`: Resources provisioned and secrets created
- `1`: Azure provisioning failed
- `2`: Kubernetes secret creation failed
- `3`: RBAC configuration failed

**Output**:
- Prints resource provisioning progress
- Prints secret creation status
- Prints service account configuration status

**Side Effects**:
- Creates Azure CosmosDB account (MongoDB API)
- Creates Azure SQL Server and database
- Creates Kubernetes secrets in `radius-system` namespace
- Creates service accounts with RBAC for secret access

**Example Usage**:
```bash
export AZURE_SUBSCRIPTION_ID="12345678-1234-1234-1234-123456789abc"
export TEST_AKS_RG="myname-radius-lrt"
make lrt-setup-test-resources
```

---

### `make lrt-cleanup-orphaned`

**Purpose**: Find and remove abandoned test resources older than threshold.

**Prerequisites**:
- Azure CLI authenticated

**Optional Environment Variables**:
```bash
ORPHAN_AGE_HOURS  # Minimum age for orphan detection (default: 48)
DRY_RUN           # Print orphans without deleting (default: false)
```

**Exit Codes**:
- `0`: Cleanup successful or no orphans found
- `1`: Orphan detection failed
- `2`: Deletion failed for some resources

**Output**:
- Prints list of orphaned resources with creation timestamps
- Prints confirmation prompt before deletion (unless DRY_RUN=true)
- Prints cleanup results

**Side Effects**:
- Deletes resource groups matching pattern and older than threshold
- Does not delete resources with retention tags

**Example Usage**:
```bash
# Find orphans older than 48 hours (default)
make lrt-cleanup-orphaned

# Dry run to see what would be deleted
DRY_RUN=true make lrt-cleanup-orphaned

# Cleanup resources older than 24 hours
ORPHAN_AGE_HOURS=24 make lrt-cleanup-orphaned
```

---

## Test Execution Targets

### `make lrt-test`

**Purpose**: Execute all test targets currently running in the LRT test workflow.

**Prerequisites**:
- AKS cluster deployed and healthy
- Radius installed on cluster
- kubectl configured

**Required Environment Variables**:
```bash
# None - inherits from current environment and kubeconfig
```

**Optional Environment Variables**:
```bash
TEST_TIMEOUT        # Test timeout (default: 1h)
GOTEST_OPTS         # Additional gotestsum options
SKIP_CLEANUP        # Preserve resources after tests (default: false)
```

**Exit Codes**:
- `0`: All tests passed
- `1`: One or more tests failed
- `2`: Test execution interrupted

**Output**:
- Real-time test progress and results
- Summary of passes/failures/skips
- Detailed failure information

**Side Effects**:
- Creates Kubernetes resources during test execution
- Cleans up test resources by default (unless SKIP_CLEANUP=true)

**Example Usage**:
```bash
# Run all LRT tests
make lrt-test

# Run with extended timeout
TEST_TIMEOUT=2h make lrt-test

# Run and preserve resources for debugging
SKIP_CLEANUP=true make lrt-test
```

**Includes**: This is a `.PHONY` target that depends on all test targets from the LRT workflow (e.g., `test-functional-corerp-cloud`, `test-functional-daprrp-cloud`, etc.)

---

### `make test-functional-all`

**Purpose**: Execute complete functional test suite (existing target, enhanced for LRT).

**Prerequisites**:
- Same as `make lrt-test`

**Exit Codes**:
- `0`: All tests passed
- `1`: One or more tests failed

**Output**:
- Test execution progress and results

**Side Effects**:
- Same as `make lrt-test`

**Example Usage**:
```bash
make test-functional-all
```

**Note**: This is an existing target being reused for LRT. See `build/test.mk` for current implementation.

---

### `make test-functional-corerp-cloud`

**Purpose**: Execute corerp functional tests requiring cloud resources.

**Prerequisites**:
- AKS cluster with cloud credentials configured
- Azure and/or AWS test resources available

**Exit Codes**:
- `0`: Tests passed
- `1`: Tests failed

**Example Usage**:
```bash
make test-functional-corerp-cloud
```

**Note**: Existing target from `build/test.mk`. Part of `make lrt-test` suite.

---

## Helper Targets

### `make lrt-version-check`

**Purpose**: Compare Radius CLI version against cluster-installed version.

**Prerequisites**:
- Radius CLI installed
- kubectl configured for target cluster

**Exit Codes**:
- `0`: Versions match or cluster version is higher
- `1`: Cluster version is lower than CLI version (upgrade recommended)
- `2`: Version detection failed

**Output**:
- Prints CLI version
- Prints cluster version
- Prints recommendation (match, upgrade, or error)

**Side Effects**: None (read-only)

**Example Usage**:
```bash
make lrt-version-check
```

---

### `make lrt-upgrade-cluster`

**Purpose**: Upgrade Radius installation on cluster to match CLI version.

**Prerequisites**:
- Radius CLI installed
- kubectl configured for target cluster
- Cluster version is lower than CLI version

**Exit Codes**:
- `0`: Upgrade successful
- `1`: Upgrade failed
- `2`: Upgrade not needed (versions already match)

**Output**:
- Prints upgrade progress
- Prints version validation results

**Side Effects**:
- Upgrades Radius installation using `rad upgrade`
- Restarts Radius pods

**Example Usage**:
```bash
make lrt-upgrade-cluster
```

---

## Environment Variable Reference

| Variable | Required | Default | Description |
|----------|----------|---------|-------------|
| `AZURE_SUBSCRIPTION_ID` | Yes (deploy) | - | Azure subscription ID |
| `TEST_AKS_RG` | Yes (deploy/destroy) | `$(whoami)-radius-lrt` | Resource group name |
| `TEST_AKS_AZURE_LOCATION` | No | `westus3` | Azure region |
| `GRAFANA_ENABLED` | No | `false` | Enable Grafana monitoring |
| `GRAFANA_ADMIN_OBJECT_ID` | Conditional | - | Grafana admin object ID |
| `ENABLE_DELETION_LOCK` | No | `true` | Prevent accidental deletion |
| `SKIP_CLEANUP` | No | `false` | Preserve resources after tests |
| `FORCE_DESTROY` | No | `false` | Skip confirmation for destroy |
| `TEST_TIMEOUT` | No | `1h` | Test execution timeout |
| `GOTEST_OPTS` | No | - | Additional gotestsum options |
| `ORPHAN_AGE_HOURS` | No | `48` | Minimum age for orphan cleanup |
| `DRY_RUN` | No | `false` | Dry-run mode (no modifications) |

## Backward Compatibility

All new targets use `lrt-` prefix to avoid conflicts with existing targets. Existing test targets (`make test-functional-all`, etc.) remain unchanged and continue to work as before.

## Target Dependencies

```
make lrt-test
  └─ depends on: all test targets from LRT workflow
      ├─ test-functional-corerp-cloud
      ├─ test-functional-daprrp-cloud
      ├─ test-functional-ucp-cloud
      └─ ... (other cloud test suites)

make lrt-deploy-infra
  └─ (no dependencies, can run standalone)

make lrt-destroy-infra
  └─ (no dependencies, can run standalone)

make lrt-health-check
  └─ requires: cluster deployed (lrt-deploy-infra)

make lrt-setup-test-resources
  └─ requires: cluster deployed (lrt-deploy-infra)

make lrt-upgrade-cluster
  └─ requires: cluster deployed (lrt-deploy-infra)
```
