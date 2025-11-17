# Quickstart: Long-Running Test System

This guide walks through deploying test infrastructure, running tests, and cleaning up—the complete LRT workflow in under 30 minutes (plus test execution time).

## Prerequisites

Before starting, ensure you have:

- **Azure subscription** with Contributor permissions
- **Azure CLI** installed and authenticated (`az login`)
- **Git** for cloning the Radius repository
- **Make** for running build targets (typically pre-installed on Linux/macOS)
- **kubectl** for Kubernetes cluster management
- **~45 minutes** for infrastructure deployment and validation

## Quick Start (5 Steps)

### Step 1: Clone the Repository

```bash
# Clone Radius repository
git clone https://github.com/radius-project/radius.git
cd radius

# Switch to feature branch (during development)
git checkout 001-lrt-modernization
```

### Step 2: Configure Environment

```bash
# Set required environment variables
export AZURE_SUBSCRIPTION_ID="your-subscription-id-here"
export TEST_AKS_RG="$(whoami)-radius-lrt"  # Your personal test resource group
export TEST_AKS_AZURE_LOCATION="eastus"    # Or your preferred region

# Optional: Enable Grafana monitoring
export GRAFANA_ENABLED="false"  # Set to "true" if you want Grafana
```

**Tip**: Add these to your `~/.bashrc` or `~/.zshrc` to persist across sessions.

### Step 3: Deploy Infrastructure

```bash
# Deploy AKS cluster with all dependencies
# This takes ~30 minutes on first run
make lrt-deploy-infra
```

**What This Does**:
- Creates Azure resource group
- Deploys AKS cluster (Kubernetes 1.31.8)
- Installs Log Analytics for monitoring
- Configures OIDC issuer for workload identity
- Installs Dapr, cert-manager, and other dependencies
- Updates your kubeconfig with cluster credentials

**Expected Output**:
```
ℹ Creating resource group 'yourname-radius-lrt' in location 'eastus'...
✓ Resource group created successfully
ℹ Deploying Bicep template...
✓ Deployment completed successfully!
ℹ AKS Cluster: yourname-radius-lrt-aks
ℹ To connect to the cluster, run:
  az aks get-credentials --resource-group yourname-radius-lrt --name yourname-radius-lrt-aks
```

### Step 4: Run Tests

```bash
# Validate cluster health before testing
make lrt-health-check

# Run the full LRT test suite
# This takes ~2 hours
make lrt-test

# Or run specific test suite
make test-functional-corerp-cloud
```

**What This Does**:
- Validates cluster connectivity and readiness
- Checks Radius installation and version
- Executes functional tests against your cluster
- Displays real-time progress and results
- Cleans up test resources by default

**Expected Output**:
```
✓ Cluster connectivity: OK
✓ Radius installation: v0.35.0
✓ Kubernetes dependencies: OK
✓ Test resource credentials: OK
✓ Cluster health: healthy

Running tests...
=== RUN   TestAzureStorageAccount
=== PASS  TestAzureStorageAccount (12.34s)
...

Test Summary:
  Total: 45 tests
  Passed: 43
  Failed: 2
  Duration: 1h 23m 45s
```

### Step 5: Clean Up

```bash
# Delete all Azure resources
make lrt-destroy-infra
```

**What This Does**:
- Prompts for confirmation (unless FORCE_DESTROY=true)
- Deletes entire resource group and all contained resources
- Removes cluster from kubeconfig
- Completes in ~15 minutes

**Expected Output**:
```
⚠ This will delete resource group 'yourname-radius-lrt' and all resources
  Continue? [y/N]: y
ℹ Deleting resource group...
✓ Resource group deleted successfully
```

## Common Workflows

### Local Development Testing

```bash
# Deploy infrastructure once
make lrt-deploy-infra

# Run tests iteratively during development
make test-functional-corerp-cloud

# Preserve resources for debugging
SKIP_CLEANUP=true make test-functional-corerp-cloud

# Clean up when done
make lrt-destroy-infra
```

### Testing Workflow Changes on Fork

```bash
# 1. Fork the radius repository on GitHub

# 2. Clone your fork
git clone https://github.com/YOUR-USERNAME/radius.git
cd radius

# 3. Configure secrets in your fork's GitHub settings:
#    Settings → Secrets and variables → Actions → New repository secret
#    Add: AZURE_SUBSCRIPTION_ID, AZURE_CLIENT_ID, AZURE_TENANT_ID, AZURE_CLIENT_SECRET

# 4. Make workflow changes and push to your fork
git checkout -b test-workflow-change
# ... make changes ...
git commit -am "Test workflow changes"
git push origin test-workflow-change

# 5. Trigger workflow manually via GitHub UI:
#    Actions → Long-Running Tests → Run workflow → Select your branch
```

### Running Only Specific Test Suites

```bash
# Run only corerp cloud tests
make test-functional-corerp-cloud

# Run only daprrp tests
make test-functional-daprrp-cloud

# Run only UCP tests
make test-functional-ucp-cloud

# Run all cloud tests
make test-functional-all-cloud
```

### Debugging Failed Tests

```bash
# Run tests with cleanup disabled
SKIP_CLEANUP=true make lrt-test

# Inspect cluster state
kubectl get pods -A
kubectl logs -n radius-system <pod-name>

# Check test resource credentials
kubectl get secrets -n radius-system

# Clean up manually when done
make lrt-destroy-infra
```

## Troubleshooting

### Problem: "Azure CLI is not authenticated"

```bash
# Solution: Authenticate to Azure
az login
az account set --subscription "your-subscription-id"
```

### Problem: "Insufficient Azure permissions"

```bash
# Solution: Verify you have Contributor role on subscription
az role assignment list --assignee $(az account show --query user.name -o tsv)

# Request Contributor access if missing
```

### Problem: "Workload identity with tag not found"

```bash
# Solution: Create the required Azure managed identity manually
# (This is a one-time setup per subscription)

az identity create \
  --name "radius-lrt-identity" \
  --resource-group "radius-lrt-shared" \
  --tags purpose=radius-lrt

# Grant required permissions
az role assignment create \
  --assignee $(az identity show -n radius-lrt-identity -g radius-lrt-shared --query principalId -o tsv) \
  --role Contributor \
  --scope /subscriptions/$AZURE_SUBSCRIPTION_ID
```

### Problem: "Health check failed - Radius not installed"

```bash
# Solution: Check Radius installation status
rad version --server

# If not installed, verify deployment completed
kubectl get pods -n radius-system

# Re-run deployment if needed
make lrt-deploy-infra
```

### Problem: "Test resources (CosmosDB, SQL) not found"

```bash
# Solution: Setup test resources explicitly
make lrt-setup-test-resources

# Verify secrets were created
kubectl get secrets -n radius-system | grep radius-test
```

### Problem: "Deployment failed mid-way"

```bash
# Solution: Re-run deployment (it's idempotent)
make lrt-deploy-infra

# If issues persist, check Azure deployment status
az deployment group show \
  --resource-group "$TEST_AKS_RG" \
  --name main

# Delete and start fresh if necessary
make lrt-destroy-infra
make lrt-deploy-infra
```

## Advanced Usage

### Custom Cluster Configuration

```bash
# Enable Grafana monitoring
export GRAFANA_ENABLED="true"
export GRAFANA_ADMIN_OBJECT_ID="your-azure-ad-object-id"
make lrt-deploy-infra
```

### Custom Test Timeout

```bash
# Increase timeout for slow environments
TEST_TIMEOUT=3h make lrt-test
```

### Parallel Test Execution

```bash
# Tests run with concurrency=2 by default
# Increase for faster execution (if cluster has capacity)
GOTEST_OPTS="-parallel 5" make lrt-test
```

### Dry-Run Deployment

```bash
# Validate configuration without actually deploying
./test/infra/azure/scripts/deploy-infra.sh \
  --resource-group "$TEST_AKS_RG" \
  --location "$TEST_AKS_AZURE_LOCATION" \
  --subscription-id "$AZURE_SUBSCRIPTION_ID" \
  --dry-run
```

### Cleanup Orphaned Resources

```bash
# Find resources older than 48 hours (default)
make lrt-cleanup-orphaned

# Preview without deleting
DRY_RUN=true make lrt-cleanup-orphaned

# Cleanup resources older than 24 hours
ORPHAN_AGE_HOURS=24 make lrt-cleanup-orphaned
```

## Next Steps

- **Read the full documentation**: See `docs/contributing/lrt-testing.md` for detailed information
- **Review test code**: Explore `test/functional-portable/` to understand test structure
- **Customize infrastructure**: Modify `test/infra/azure/main.bicep` for your needs
- **Add new test suites**: Follow patterns in existing test directories
- **Contribute improvements**: Submit PRs to improve the LRT system

## Environment Variable Reference

| Variable | Required | Default | Description |
|----------|----------|---------|-------------|
| `AZURE_SUBSCRIPTION_ID` | Yes | - | Azure subscription for infrastructure |
| `TEST_AKS_RG` | Yes | `$(whoami)-radius-lrt` | Resource group name |
| `TEST_AKS_AZURE_LOCATION` | No | `westus3` | Azure region |
| `GRAFANA_ENABLED` | No | `false` | Enable Grafana monitoring |
| `GRAFANA_ADMIN_OBJECT_ID` | Conditional | - | Grafana admin (required if GRAFANA_ENABLED=true) |
| `SKIP_CLEANUP` | No | `false` | Preserve resources after tests |
| `TEST_TIMEOUT` | No | `1h` | Test execution timeout |
| `FORCE_DESTROY` | No | `false` | Skip confirmation for cleanup |

## Estimated Timings

| Operation | First Time | Subsequent Runs |
|-----------|-----------|-----------------|
| Infrastructure deployment | ~30 min | ~30 min (delete + redeploy) |
| Health check | ~1 min | ~1 min |
| Full test suite | ~2 hours | ~2 hours |
| Individual test suite | ~10-30 min | ~10-30 min |
| Infrastructure cleanup | ~15 min | ~15 min |

## Getting Help

- **Documentation**: `docs/contributing/lrt-testing.md`
- **GitHub Issues**: https://github.com/radius-project/radius/issues
- **Slack**: #radius-dev channel
- **Weekly Community Call**: Check GOVERNANCE.md for schedule
