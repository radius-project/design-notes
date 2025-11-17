# GitHub Workflow Contracts

This document defines the interface for the LRT workflow in GitHub Actions, ensuring fork-compatibility and local reproducibility.

## Workflow: Long-Running Tests

**File**: `.github/workflows/long-running-tests.yml`

### Trigger Events

```yaml
on:
  workflow_dispatch:  # Manual trigger (works on forks)
    inputs:
      test_suites:
        description: 'Test suites to run (comma-separated, or "all")'
        required: false
        default: 'all'
      skip_cleanup:
        description: 'Skip resource cleanup after tests'
        required: false
        default: 'false'
  
  schedule:
    # Nightly runs - ONLY on main repository, NOT on forks
    - cron: '0 2 * * *'  # 2 AM UTC daily

  # Schedule filtering in workflow ensures it doesn't run on forks
```

### Required Secrets (Configurable in Fork)

```yaml
# Azure authentication
AZURE_SUBSCRIPTION_ID     # Azure subscription for test infrastructure
AZURE_CLIENT_ID           # Service principal client ID
AZURE_TENANT_ID           # Azure AD tenant ID
AZURE_CLIENT_SECRET       # Service principal secret

# Container registry (optional - defaults to ghcr.io)
CONTAINER_REGISTRY        # Container registry URL (e.g., myregistry.azurecr.io)
CONTAINER_REGISTRY_USERNAME  # Registry username (optional)
CONTAINER_REGISTRY_PASSWORD  # Registry password (optional)
```

### Required Variables (Configurable in Fork)

```yaml
TEST_AKS_RG               # Resource group name for test cluster
TEST_AKS_AZURE_LOCATION   # Azure region for deployment
GRAFANA_ENABLED           # Enable Grafana monitoring (true/false)
```

### Workflow Steps (High-Level)

```yaml
jobs:
  long-running-tests:
    runs-on: ubuntu-latest
    steps:
      # 1. Checkout code
      - uses: actions/checkout@v4
      
      # 2. Azure authentication
      - name: Azure Login
        uses: azure/login@v1
        with:
          creds: ${{ secrets.AZURE_CREDENTIALS }}
      
      # 3. Install prerequisites (all via Make targets)
      - name: Install Azure CLI
        run: make install-azure-cli
      
      - name: Install Radius CLI
        run: make install-rad-cli
      
      # 4. Deploy infrastructure (via Make target - no logic in workflow)
      - name: Deploy Test Infrastructure
        run: make lrt-deploy-infra
        env:
          AZURE_SUBSCRIPTION_ID: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
          TEST_AKS_RG: ${{ vars.TEST_AKS_RG }}
          TEST_AKS_AZURE_LOCATION: ${{ vars.TEST_AKS_AZURE_LOCATION }}
          GRAFANA_ENABLED: ${{ vars.GRAFANA_ENABLED }}
      
      # 5. Health check
      - name: Validate Environment
        run: make lrt-health-check
      
      # 6. Version check and upgrade if needed
      - name: Check Radius Version
        run: make lrt-version-check
      
      - name: Upgrade Radius if Needed
        if: steps.version-check.outputs.upgrade_needed == 'true'
        run: make lrt-upgrade-cluster
      
      # 7. Setup test resources (via Make target)
      - name: Setup Test Resources
        run: make lrt-setup-test-resources
        env:
          AZURE_SUBSCRIPTION_ID: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
          TEST_AKS_RG: ${{ vars.TEST_AKS_RG }}
      
      # 8. Run tests (via Make target - no logic in workflow)
      - name: Run Tests
        run: make lrt-test
        env:
          TEST_TIMEOUT: 2h
          SKIP_CLEANUP: ${{ inputs.skip_cleanup || 'false' }}
      
      # 9. Cleanup (via Make target - runs even on failure)
      - name: Cleanup Infrastructure
        if: always() && inputs.skip_cleanup != 'true'
        run: make lrt-destroy-infra
        env:
          TEST_AKS_RG: ${{ vars.TEST_AKS_RG }}
          FORCE_DESTROY: 'true'
      
      # 10. Upload test results
      - name: Upload Test Results
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: test-results
          path: |
            dist/test-results/*.xml
            dist/container_logs/**/*
```

### Fork-Compatibility Design

**Principle**: Workflow YAML only orchestrates tool invocation; all logic lives in Make targets and shell scripts.

**How It Works**:
1. Fork owner configures their own secrets/variables in fork settings
2. Workflow uses fork-configured credentials (not organization secrets)
3. All deployment logic in Make targets → executable locally
4. Scheduled triggers filtered via repository check:

```yaml
jobs:
  long-running-tests:
    # Only run scheduled tests on main repository
    if: github.event_name != 'schedule' || github.repository == 'radius-project/radius'
```

**Testing on Fork**:
```bash
# Fork owner sets up secrets in GitHub UI:
# - AZURE_SUBSCRIPTION_ID
# - AZURE_CLIENT_ID, AZURE_TENANT_ID, AZURE_CLIENT_SECRET
# - (optional) CONTAINER_REGISTRY credentials

# Then triggers workflow manually via GitHub Actions UI:
# - Go to Actions tab
# - Select "Long-Running Tests" workflow
# - Click "Run workflow"
# - Select branch to test
# - (optional) Specify test suites
# - Click "Run workflow" button
```

### Workflow Outputs

```yaml
outputs:
  cluster_name:
    description: 'AKS cluster name'
    value: ${{ steps.deploy.outputs.cluster_name }}
  
  test_status:
    description: 'Overall test status (passed/failed/partial)'
    value: ${{ steps.test.outputs.status }}
  
  test_duration:
    description: 'Total test execution time'
    value: ${{ steps.test.outputs.duration }}
```

### Artifacts

```yaml
artifacts:
  test-results:
    description: 'JUnit XML test results'
    path: 'dist/test-results/*.xml'
    retention-days: 30
  
  container-logs:
    description: 'Container logs from test execution'
    path: 'dist/container_logs/**/*'
    retention-days: 7
```

## Workflow Behavior

### Schedule Filtering

Scheduled runs MUST NOT execute on forks to prevent unexpected resource consumption:

```yaml
if: github.event_name != 'schedule' || github.repository == 'radius-project/radius'
```

### Manual Dispatch

`workflow_dispatch` event MUST be enabled for all branches to allow fork testing:

```yaml
on:
  workflow_dispatch:  # No branch restrictions
```

### Container Registry Configuration

Workflow MUST support alternative registries for forks:

```yaml
env:
  CONTAINER_REGISTRY: ${{ secrets.CONTAINER_REGISTRY || 'ghcr.io/radius-project/dev' }}
```

If fork doesn't have access to `ghcr.io/radius-project/*`, they configure their own registry in secrets.

### Cleanup Behavior

Cleanup MUST run by default even on failure, unless explicitly skipped:

```yaml
- name: Cleanup Infrastructure
  if: always() && inputs.skip_cleanup != 'true'
  run: make lrt-destroy-infra
```

## Local Reproduction

**Critical Requirement**: Every workflow step MUST be reproducible locally.

```bash
# Exactly matches workflow steps:

# 1. Authenticate to Azure
az login

# 2. Deploy infrastructure
make lrt-deploy-infra

# 3. Health check
make lrt-health-check

# 4. Version check
make lrt-version-check

# 5. Setup test resources
make lrt-setup-test-resources

# 6. Run tests
make lrt-test

# 7. Cleanup
make lrt-destroy-infra
```

## Error Handling

All Make targets MUST return appropriate exit codes. Workflow MUST NOT contain retry logic (retry belongs in scripts).

```yaml
- name: Deploy Infrastructure
  run: make lrt-deploy-infra  # Script handles retries internally
```

## Success Criteria

A workflow is fork-compatible if:

1. ✅ Works when triggered via workflow_dispatch on any branch
2. ✅ Uses only secrets/variables configurable in fork settings
3. ✅ Does NOT execute scheduled runs on forks
4. ✅ All steps reproducible locally via Make targets
5. ✅ No organization-specific resources referenced (except as defaults)
6. ✅ Container registry is configurable
7. ✅ No embedded Bash/PowerShell scripts in workflow YAML
8. ✅ Exit codes properly propagated from Make targets

## Backward Compatibility

This is a new workflow file. No breaking changes to existing workflows.
