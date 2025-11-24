# Shell Script Contracts

This document defines the public interface for all shell scripts in the LRT system. These scripts are called by Make targets and must maintain stable interfaces for backward compatibility.

## Deployment Scripts

### `test/infra/azure/scripts/deploy-infra.sh`

**Purpose**: Orchestrate Bicep deployment with validation and health checks.

**Synopsis**:
```bash
deploy-infra.sh [OPTIONS]
```

**Options**:
- `--resource-group <name>`: Resource group name (required)
- `--location <region>`: Azure region (required)
- `--subscription-id <id>`: Azure subscription ID (required)
- `--grafana-enabled`: Enable Grafana monitoring (optional)
- `--grafana-admin-id <object-id>`: Grafana admin object ID (conditional)
- `--enable-deletion-lock`: Prevent accidental deletion (default: enabled)
- `--dry-run`: Validate configuration without deploying (optional)
- `--help`: Display usage information

**Environment Variables**: None (all configuration via CLI arguments)

**Exit Codes**:
- `0`: Deployment successful
- `1`: Invalid arguments
- `2`: Azure CLI not authenticated
- `3`: Insufficient permissions
- `4`: Bicep deployment failed
- `5`: Health check failed

**Output**:
- Stdout: Progress messages, cluster connection instructions
- Stderr: Error messages and debugging information

**Side Effects**:
- Creates Azure resource group if not exists
- Deploys Bicep template via `az deployment group create`
- Updates kubeconfig with cluster credentials
- Creates kubeconfig backup if file already exists

**Behavior**:
- Validates all arguments before starting deployment
- Checks Azure CLI authentication status
- Validates Azure permissions (list subscriptions, read quota)
- Checks workload identity exists via tag lookup
- Executes Bicep deployment (idempotent - resumes partial deployments)
- Runs health check after deployment
- Prints connection instructions on success

**Example**:
```bash
./test/infra/azure/scripts/deploy-infra.sh \
  --resource-group "myname-radius-lrt" \
  --location "eastus" \
  --subscription-id "12345678-1234-1234-1234-123456789abc"
```

---

### `test/infra/azure/scripts/destroy-infra.sh`

**Purpose**: Delete Azure resources with safety checks.

**Synopsis**:
```bash
destroy-infra.sh [OPTIONS]
```

**Options**:
- `--resource-group <name>`: Resource group name to delete (required)
- `--force`: Skip confirmation prompt (optional)
- `--help`: Display usage information

**Exit Codes**:
- `0`: Deletion successful
- `1`: Invalid arguments
- `2`: Resource group not found
- `3`: Deletion lock present (manual intervention required)
- `4`: Azure deletion failed

**Output**:
- Stdout: Confirmation prompt, deletion progress
- Stderr: Error messages

**Side Effects**:
- Deletes Azure resource group
- Removes cluster from kubeconfig

**Behavior**:
- Validates resource group exists
- Checks for retention tags (fails if retention tag present)
- Prompts for confirmation (unless --force)
- Deletes resource group via `az group delete`
- Removes cluster from kubeconfig if present
- Reports deletion time

**Example**:
```bash
./test/infra/azure/scripts/destroy-infra.sh \
  --resource-group "myname-radius-lrt"

# Force deletion without prompt
./test/infra/azure/scripts/destroy-infra.sh \
  --resource-group "myname-radius-lrt" \
  --force
```

---

### `test/infra/azure/scripts/health-check.sh`

**Purpose**: Validate test environment readiness.

**Synopsis**:
```bash
health-check.sh [OPTIONS]
```

**Options**:
- `--kubeconfig <path>`: Path to kubeconfig (optional, default: $HOME/.kube/config)
- `--context <name>`: Kubernetes context to check (optional, default: current context)
- `--help`: Display usage information

**Exit Codes**:
- `0`: All health checks passed
- `1`: Cluster connectivity failed
- `2`: Radius not installed or wrong version
- `3`: Required Kubernetes resources missing
- `4`: Test resource credentials unavailable
- `5`: Cluster in degraded state

**Output**:
- Stdout: Health check results for each component with ✓/✗ indicators
- Stderr: Detailed error messages with remediation steps

**Checks Performed**:
1. Cluster connectivity (kubectl version)
2. Radius installation (rad version --server)
3. Kubernetes dependencies (Dapr, cert-manager pods)
4. Test resource secrets (CosmosDB, SQL secrets existence)
5. Node resource availability (CPU, memory)
6. Workload identity configuration

**Side Effects**: None (read-only validation)

**Example**:
```bash
# Check current context
./test/infra/azure/scripts/health-check.sh

# Check specific context
./test/infra/azure/scripts/health-check.sh --context my-test-cluster
```

**Expected Duration**: < 2 minutes

---

### `test/infra/azure/scripts/setup-credentials.sh`

**Purpose**: Provision test resources and create Kubernetes secrets.

**Synopsis**:
```bash
setup-credentials.sh [OPTIONS]
```

**Options**:
- `--resource-group <name>`: Resource group for test resources (required)
- `--subscription-id <id>`: Azure subscription ID (required)
- `--kubeconfig <path>`: Path to kubeconfig (optional)
- `--namespace <name>`: Kubernetes namespace for secrets (default: radius-system)
- `--help`: Display usage information

**Exit Codes**:
- `0`: Resources provisioned and secrets created
- `1`: Invalid arguments
- `2`: Azure resource provisioning failed
- `3`: Kubernetes secret creation failed
- `4`: RBAC configuration failed

**Output**:
- Stdout: Provisioning progress, secret creation status
- Stderr: Error messages

**Side Effects**:
- Creates Azure CosmosDB account (MongoDB API)
- Creates Azure SQL Server and database
- Creates Kubernetes secrets with connection strings
- Creates service accounts with RBAC policies

**Behavior**:
- Validates arguments and cluster connectivity
- Provisions CosmosDB account (idempotent - skips if exists)
- Provisions SQL Server (idempotent - skips if exists)
- Retrieves connection strings from Azure
- Creates Kubernetes secrets in specified namespace
- Creates service accounts for test pods
- Configures RBAC for secret access

**Example**:
```bash
./test/infra/azure/scripts/setup-credentials.sh \
  --resource-group "myname-radius-lrt" \
  --subscription-id "12345678-1234-1234-1234-123456789abc"
```

---

## Utility Scripts

### `build/test.sh` (Enhanced)

**Purpose**: Deploy AKS cluster for long-running tests (existing script, enhanced for idempotency).

**Synopsis**:
```bash
test.sh [OPTIONS]
```

**Options**: None (uses environment variables)

**Environment Variables**:
- `TEST_AKS_AZURE_LOCATION`: Azure region (default: westus3)
- `TEST_AKS_RG`: Resource group name (default: $(whoami)-radius-lrt)

**Exit Codes**:
- `0`: Deployment successful
- `1`: Deployment failed

**Note**: This is an existing script being enhanced for idempotency and better error handling. See current implementation in `build/test.sh`.

---

## Script Behavior Standards

All scripts must adhere to these standards per `.github/instructions/shell.instructions.md`:

### Error Handling
```bash
set -euo pipefail  # Exit on error, undefined vars, pipe failures

trap cleanup EXIT INT TERM  # Cleanup on exit/interrupt
```

### Argument Parsing
```bash
while [[ $# -gt 0 ]]; do
  case $1 in
    --resource-group)
      RESOURCE_GROUP="$2"
      shift 2
      ;;
    --help)
      print_usage
      exit 0
      ;;
    *)
      echo "Unknown option: $1" >&2
      exit 1
      ;;
  esac
done
```

### Output Formatting
```bash
# Colors
readonly RED='\033[0;31m'
readonly GREEN='\033[0;32m'
readonly YELLOW='\033[1;33m'
readonly BLUE='\033[0;34m'
readonly NC='\033[0m'  # No Color

# Helper functions
print_info() { echo -e "${BLUE}ℹ${NC} $1"; }
print_success() { echo -e "${GREEN}✓${NC} $1"; }
print_warning() { echo -e "${YELLOW}⚠${NC} $1"; }
print_error() { echo -e "${RED}✗${NC} $1" >&2; }
```

### Validation
```bash
# Validate required arguments
if [[ -z "${RESOURCE_GROUP:-}" ]]; then
  print_error "Missing required argument: --resource-group"
  print_usage
  exit 1
fi

# Validate Azure CLI authentication
if ! az account show &>/dev/null; then
  print_error "Azure CLI is not authenticated"
  exit 2
fi
```

### Idempotency
```bash
# Check if resource exists before creating
if az group show --name "$RESOURCE_GROUP" &>/dev/null; then
  print_info "Resource group already exists, skipping creation"
else
  print_info "Creating resource group..."
  az group create --name "$RESOURCE_GROUP" --location "$LOCATION"
fi
```

### Cleanup
```bash
cleanup() {
  local exit_code=$?
  
  if [[ $exit_code -ne 0 ]]; then
    print_error "Script failed, performing cleanup..."
    # Cleanup logic here
  fi
  
  return $exit_code
}

trap cleanup EXIT INT TERM
```

## Integration with Make Targets

Scripts are invoked by Make targets as follows:

```makefile
.PHONY: lrt-deploy-infra
lrt-deploy-infra:
	@bash ./test/infra/azure/scripts/deploy-infra.sh \
		--resource-group "$(TEST_AKS_RG)" \
		--location "$(TEST_AKS_AZURE_LOCATION)" \
		--subscription-id "$(AZURE_SUBSCRIPTION_ID)" \
		$(if $(filter true,$(GRAFANA_ENABLED)),--grafana-enabled) \
		$(if $(GRAFANA_ADMIN_OBJECT_ID),--grafana-admin-id "$(GRAFANA_ADMIN_OBJECT_ID)")
```

## Backward Compatibility

New scripts added under `test/infra/azure/scripts/` do not conflict with existing scripts. Enhanced `build/test.sh` maintains backward compatibility with existing environment variable interface.
