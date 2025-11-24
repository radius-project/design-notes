# Feature Specification: Long-Running Test System Modernization

**Feature Branch**: `001-lrt-modernization`
**Created**: November 11, 2025
**Status**: Draft
**Input**: User description: "Modernize Radius Long-Running Tests system to align with contemporary workflow design principles and improve developer experience"

## Clarifications

### Session 2025-11-11

- Q: How should test resource credentials (CosmosDB, SQL Server) be stored and accessed by tests? → A: Store credentials in Kubernetes secrets with RBAC, accessed by test pods via service accounts
- Q: Should developers provision dedicated AKS clusters or share clusters with isolation? → A: Each developer provisions their own dedicated AKS cluster (no sharing, full isolation)
- Q: How should release detection polling interval be configured? → A: At workflow start, check CLI version vs cluster version; if cluster version is lower than CLI version, upgrade cluster using radius upgrade command

### Session 2025-11-15

- Q: How is test parallelization controlled and what is the default concurrency limit? → A: Parallel by default with concurrency=2 (balanced approach)
- Q: What is the retention policy and storage location for diagnostic artifacts from failed tests? → A: No persistent storage; only display in terminal output during test execution
- Q: What happens when the tagged workload identity doesn't exist during first-time setup? → A: Fail deployment with instructions to manually create identity with required tag first
- Q: What test suite names/components are supported for individual execution? → A: make lrt-test will be a .PHONY target that includes the test targets currently running in the LRT test workflow
- Q: Is cleanup after tests opt-in or automatic, and can it be skipped? → A: Cleanup runs by default; opt-out via flag (e.g., `SKIP_CLEANUP=true`) to preserve resources

### Session 2025-11-16

- Q: What authentication method should be used for Azure access in GitHub workflows and local development? → A: GitHub workflows use Azure OIDC authentication (azure/login@v2 with client-id/tenant-id/subscription-id); developers use their personal Azure accounts (az login) for local testing. Service principals are NOT used.
- Q: How is workload identity configured for AKS clusters? → A: Developers specify the workload identity when deploying infrastructure; the identity must be pre-created with appropriate tags.

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Developer Infrastructure Setup (Priority: P1)

A Radius developer needs to provision their own isolated test environment to validate changes before submitting a PR. They should be able to deploy a complete AKS-based test infrastructure with monitoring, run the full LRT suite against their changes, and tear down the infrastructure when done—all from their local machine using simple commands. Each developer provisions a dedicated AKS cluster with no sharing to ensure complete isolation and prevent test interference.

**Why this priority**: This is the foundation that enables all other scenarios. Without the ability for developers to provision infrastructure and run tests locally, the entire modernization effort provides no value. This delivers immediate value by allowing developers to validate changes before pushing to forks.

**Independent Test**: Can be fully tested by a developer with only an Azure subscription running a sequence of commands (`make lrt-deploy-infra`, `make test-functional-all`, `make lrt-destroy-infra`) and verifying that infrastructure is created, tests execute, and resources are cleaned up successfully. Delivers the value of independent test environments.

**Acceptance Scenarios**:

1. **Given** a developer has Azure CLI configured with valid credentials and required environment variables set, **When** they run `make lrt-deploy-infra`, **Then** an AKS cluster with all dependencies (Log Analytics, Grafana, Dapr, OIDC) is provisioned successfully (P50 <25 min, P95 <45 min) and cluster readiness is validated
2. **Given** an AKS cluster has been provisioned successfully, **When** the developer runs `make test-functional-all`, **Then** all functional tests execute against their cluster and results are displayed locally
3. **Given** tests have completed (successfully or with failures), **When** the developer runs `make lrt-destroy-infra`, **Then** all Azure resources in the test resource group are deleted within 15 minutes
4. **Given** a developer has only basic Azure permissions, **When** they attempt infrastructure deployment, **Then** the system provides clear error messages about missing permissions and documents the minimum required roles
5. **Given** infrastructure deployment fails mid-way, **When** the developer runs `make lrt-deploy-infra` again, **Then** the deployment resumes or rolls back safely without leaving orphaned resources

---

### User Story 2 - Fork-Based Workflow Testing (Priority: P1)

A Radius contributor working on a fork needs to test workflow changes before submitting a PR to the main repository. They should be able to manually deploy LRT infrastructure using Make commands, push workflow changes to their fork, manually trigger the LRT workflow via GitHub UI (which authenticates to Azure and configures kubectl context), and see test results—all without needing access to `radius-project` organization resources or secrets.

**Why this priority**: This addresses a critical pain point preventing contributors from validating workflow changes. Currently, workflows are tightly coupled to organization-specific resources, making it impossible to test on forks. This is equally critical as P1 because it enables workflow improvements and community contributions.

**Independent Test**: Can be fully tested by a contributor forking the repository, manually deploying infrastructure with `make lrt-deploy-infra`, configuring fork secrets with Azure credentials and cluster name, pushing a workflow change, triggering the workflow via GitHub Actions UI with workflow_dispatch, and verifying that the workflow authenticates to Azure, sets kubectl context, executes tests successfully, and uses only resources in their own Azure subscription. Delivers independent validation of workflow correctness.

**Acceptance Scenarios**:

1. **Given** a contributor has manually deployed LRT infrastructure to their Azure subscription using their personal account, **When** they configure their fork with Azure OIDC credentials (client-id, tenant-id, subscription-id) and cluster name, **Then** they can manually trigger the LRT workflow via workflow_dispatch on any branch
2. **Given** a workflow is triggered on a fork with pre-deployed infrastructure, **When** the workflow executes, **Then** it authenticates to Azure using OIDC (azure/login@v2), sets kubectl context using `az aks get-credentials`, and runs tests against the pre-deployed cluster successfully
3. **Given** a contributor has no access to `ghcr.io/radius-project/*` registries, **When** they run the workflow on their fork, **Then** the workflow uses an alternative registry specified in their fork's configuration without code changes
4. **Given** a workflow is scheduled to run, **When** it triggers on a fork, **Then** the schedule is ignored and the workflow does not execute automatically (only manual dispatch works on forks)
5. **Given** workflow execution completes on a fork, **When** the contributor views results, **Then** they see detailed logs, test outcomes, and resource cleanup status in GitHub Actions UI (infrastructure remains deployed unless manually destroyed)

---

### User Story 3 - Automated Release Testing (Priority: P2)

A Radius maintainer needs confidence that each new release works correctly in realistic long-running scenarios. When the LRT workflow executes, the system should check if the cluster needs upgrading (by comparing CLI vs cluster versions), automatically upgrade if needed, run comprehensive functional tests against the current version, and report results to the team—without manual intervention.

**Why this priority**: While automated release testing is valuable, it's less critical than enabling developer self-service (P1). The team can manually trigger tests against releases initially. However, automation reduces maintenance burden and ensures consistent release validation.

**Independent Test**: Can be fully tested by installing an older Radius version on a cluster, installing a newer CLI version, running the LRT workflow, and verifying the system detects the version mismatch, upgrades the cluster, executes tests, and reports results to configured notification channels. Delivers automated version consistency without manual upgrade steps.

**Acceptance Scenarios**:

1. **Given** a test workflow starts execution, **When** the system checks versions, **Then** it compares the Radius CLI version against the cluster-installed version and triggers upgrade if cluster version is lower
2. **Given** cluster version is lower than CLI version, **When** the upgrade process begins, **Then** the cluster is upgraded to match CLI version using `rad upgrade` command
3. **Given** the cluster upgrade completes successfully, **When** functional tests are executed, **Then** all test suites run against the new version and results are collected
---

### User Story 4 - Local Test Execution (Priority: P2)

A Radius developer working on a specific component wants to run just the LRT suite locally against their already-provisioned AKS cluster to validate changes iteratively during development. They should be able to run individual test suites or the complete suite using Make commands, see real-time output, and get fast feedback without waiting for CI/CD pipelines.

**Why this priority**: This enables rapid iteration during development. While developers can run tests, it's less critical than the ability to provision infrastructure (P1) which is a prerequisite. This story focuses on the testing phase after infrastructure exists.

**Independent Test**: Can be fully tested by a developer with a running AKS cluster executing `make lrt-test` for a specific suite or `make test-functional-all` for all suites, observing real-time test output in their terminal, and seeing a summary of passes/failures. Delivers fast feedback for iterative development.

**Acceptance Scenarios**:

1. **Given** a developer has a configured AKS cluster and environment variables set, **When** they run `make test-functional-all`, **Then** all functional test suites execute with progress displayed in real-time
2. **Given** a developer wants to test a specific component, **When** they run `make lrt-test` (or other specific test targets from the LRT workflow), **Then** only that test suite executes, saving time during iterative development
3. **Given** tests are executing, **When** a test fails, **Then** the failure is logged with context (error message, relevant pod logs, resource state) and execution continues to remaining tests unless configured to fail-fast
4. **Given** test execution completes, **When** the developer reviews results, **Then** they see a summary with total tests, passes, failures, execution time, and paths to detailed logs
5. **Given** tests require pre-provisioned resources (CosmosDB, SQL), **When** the developer runs tests, **Then** the system retrieves connection details from Kubernetes secrets via service account RBAC and connects successfully

---

### User Story 5 - Test Environment Health Validation (Priority: P3)

Before running expensive long-running tests, developers and CI systems need to validate that the test environment is healthy and properly configured. They should be able to run a health check command that verifies all prerequisites (cluster connectivity, required resources, Radius installation, environment variables, access to secrets) and get a clear pass/fail result with actionable error messages.

**Why this priority**: While health checks are valuable for preventing wasted time on broken environments, they're less critical than core testing functionality. Developers can manually verify prerequisites initially, and health checks can be added incrementally.

**Independent Test**: Can be fully tested by running `make lrt-health-check` against both healthy and intentionally misconfigured environments, verifying that the command correctly identifies issues (missing resources, incorrect versions, connectivity problems, environment variables, access to secrets) and provides actionable guidance. Delivers fast failure detection before expensive test runs.

**Acceptance Scenarios**:

1. **Given** an AKS cluster has been provisioned, **When** a developer runs `make lrt-health-check`, **Then** the system verifies cluster connectivity, Radius installation, required Kubernetes resources, environment variables, access to secrets, and pre-provisioned test resources
2. **Given** a health check detects missing prerequisites, **When** the check completes, **Then** it exits with non-zero code and displays clear error messages with remediation steps (e.g., "CosmosDB connection not found in AKS secrets - run `make lrt-setup-test-resources`")
3. **Given** all prerequisites are met, **When** health check runs, **Then** it completes in under 2 minutes and exits with zero code and confirmation message
4. **Given** Radius is installed but the wrong version, **When** health check runs, **Then** it reports version mismatch and suggests upgrade commands
5. **Given** cluster has insufficient resources, **When** health check runs, **Then** it warns about potential test failures due to resource constraints

---

### Edge Cases

- **How does the system handle partial test failures?** By default, all test suites run to completion even if individual suites fail, providing complete results. A `fail-fast` configuration option allows stopping on first failure for faster feedback during development.

- **What if AKS cluster is in a failed or degraded state?** Health check detects degraded state and prevents test execution. System provides commands to re-provision infrastructure or recover cluster state.

- **How does version comparison work if CLI or cluster version cannot be determined?** System logs warning and proceeds with tests against current cluster state. Health check (FR-006) validates Radius installation is functional before test execution.

- **What happens if a developer's Bicep deployment fails mid-way?** Deployment script is idempotent - re-running continues from failure point or safely rolls back. Failed deployments don't leave orphaned resources due to Azure resource group lifecycle management.

- **How are test resources cleaned up when tests are interrupted (Ctrl+C, runner crash)?** Cleanup is implemented in trap handlers for shell scripts and deferred functions in Go code, running by default even on interruption. A separate `make lrt-cleanup` command finds and removes abandoned resources. Developers can set `SKIP_CLEANUP=true` to preserve infrastructure for debugging.

- **What if the tagged Azure workload identity doesn't exist during first-time setup?** Deployment fails with clear error message and instructions to manually create the identity with the required tag. Documentation provides step-by-step guide for creating the identity with correct permissions and tags.

- **How are AKS cluster quota limits handled in a subscription?** Pre-deployment validation checks subscription quotas and provides clear error if insufficient quota. Documentation guides developers on requesting quota increases or using different regions/SKUs.

- **What happens when test recipes or UDT types fail to publish?** Recipe publication is separate from test execution - failures are logged but don't block tests. Re-publication is idempotent and can be retried with `make lrt-publish-recipes`.

- **How does the system handle authentication token expiration during long test runs?** Azure CLI tokens are refreshed automatically before expiration. For runs lasting beyond refresh window, workload identity (OIDC) is used where possible to avoid token-based auth.

## Requirements *(mandatory)*

### Functional Requirements

#### Infrastructure Provisioning

- **FR-001**: System MUST provide Make targets (`make lrt-deploy-infra`, `make lrt-destroy-infra`) that deploy and destroy complete dedicated AKS test environments using existing Bicep templates. Each developer environment MUST use a uniquely-named resource group to prevent conflicts.
- **FR-002**: Infrastructure deployment MUST be idempotent - re-running deployment against same resource group either completes partial deployment or reports current state
- **FR-003**: System MUST support environment variable configuration for all deployment parameters (resource group name, location, cluster size, Grafana enablement)
- **FR-004**: System MUST validate Azure permissions before deployment and fail with actionable error messages if insufficient permissions detected
- **FR-005**: System MUST provide a Make target (`make lrt-health-check`) that validates cluster readiness including connectivity, Radius installation, required resources, and test prerequisites

#### Release Detection & Upgrade

- **FR-007**: System MUST compare Radius CLI version against cluster-installed version at workflow start. If cluster version is lower than CLI version, system MUST upgrade cluster using `rad upgrade` command before executing tests.
- **FR-008**: System MUST use the current release version to install using the standard installer script provided by Radius (not hardcoded versions).
- **FR-010**: System MUST upgrade Radius installation on AKS cluster using `rad upgrade` command if the cli version is greater than the radius server version.
- **FR-011**: Upgrade process MUST verify successful upgrade by checking pod readiness and Radius API availability before proceeding to tests

#### Test Execution

- **FR-013**: System MUST provide Make targets for executing test suites. The `make lrt-test` target MUST be a .PHONY meta-target that invokes all test targets currently running in the LRT test workflow (enabling LRT-specific suite execution), while `make test-functional-all` executes the complete functional test suite across all components.
- **FR-014**: Test execution MUST display real-time progress and output to terminal when run locally
- **FR-015**: System MUST collect test results in structured format (JUnit XML or similar) for integration with CI systems
- **FR-016**: Test execution MUST complete within 120 minutes for full suite on standard AKS cluster configuration (3-node cluster, Standard_D4s_v3 VMs, 16GB RAM and 4 vCPUs per node)
- **FR-017**: System MUST support parallel test execution (configurable, serial by default for resource compatibility)
- **FR-018**: System MUST retrieve test resource credentials (CosmosDB, SQL Server) from Kubernetes secrets using service account RBAC (not from environment variables or GitHub organization secrets). Test pods MUST authenticate via service accounts with minimal required permissions.
- **FR-019**: Failed tests MUST generate diagnostic output including relevant pod logs, resource state, and error context displayed in terminal output (no persistent artifact storage)

#### Fork Compatibility

- **FR-020**: GitHub workflows MUST execute successfully on repository forks using pre-deployed infrastructure accessible to fork owner (not radius-project organization resources). Workflows authenticate to Azure and configure kubectl context but do NOT deploy infrastructure.
- **FR-021**: Workflows MUST support manual triggering via workflow_dispatch event on any branch
- **FR-022**: Scheduled workflow triggers MUST NOT execute on forks (schedule ignored unless repository matches configured organization/repository pattern)
- **FR-023**: Workflows MUST accept configuration through environment variables or repository secrets (container registries, cloud credentials, cluster names, resource group names)
- **FR-024**: All core test logic MUST be implemented in Make targets and shell scripts, not embedded in workflow YAML (enabling local execution)
- **FR-029**: Workflows MUST authenticate to Azure using fork-configured credentials and set kubectl context to pre-deployed cluster using `az aks get-credentials --resource-group <rg> --name <cluster>` (infrastructure deployment is manual and occurs before workflow execution)

#### Test Resource Management

- **FR-025**: System MUST provide commands to setup pre-provisioned test resources (CosmosDB, SQL Server) and store credentials in Kubernetes secrets with appropriate RBAC policies. Service accounts for test execution MUST be created with read-only access to credential secrets.
- **FR-026**: System MUST clean up test-created cloud resources (Azure resource groups, AWS resources) after test execution completes by default, with opt-out via environment variable (e.g., `SKIP_CLEANUP=true`) to preserve resources for debugging
- **FR-027**: Cleanup MUST be implemented in trap handlers and deferred functions to execute by default even when tests are interrupted, unless opt-out flag is set
- **FR-028**: System MUST support retention tags on resource groups to prevent accidental deletion of long-lived resources
- **FR-029**: System MUST provide a command (`make lrt-cleanup-orphaned`) to identify and remove abandoned test resources older than configured threshold

#### Authentication & Identity

- **FR-040**: Workload identity MUST be configured on the test cluster and MUST reuse the same Azure identity based on a tag that is set on the identity. The system MUST locate and bind the existing Azure identity to the cluster using the tag selector. If the tagged identity does not exist, deployment MUST fail with clear instructions to manually create the identity with the required tag before proceeding.
- **FR-041**: All secrets (database credentials, API keys, connection strings) MUST be stored in Kubernetes as Secret resources. All non-sensitive environment settings MUST be stored in Kubernetes ConfigMap resources. No credentials MUST be stored in environment variables or external secret stores. Test code and test scripts MAY reference secrets and configuration settings via environment variables, but these environment variables MUST be automatically populated from values retrieved from Kubernetes Secrets and ConfigMaps (not manually set by developers).
- **FR-042**: Local connection to Kubernetes clusters (both AKS and other Kubernetes distributions) MUST be done via the kubectl configuration file (~/.kube/config or KUBECONFIG environment variable). The system MUST NOT require additional authentication mechanisms beyond standard kubeconfig contexts.

#### Reporting & Observability

- **FR-030**: Test results MUST be reported with summary including total tests, passes, failures, skipped tests, and execution time
- **FR-032**: Test execution MUST generate log output including container logs, resource state dumps, and test framework output displayed in terminal during execution
- **FR-033**: System MUST integrate with existing monitoring (Grafana, Azure Monitor) if enabled on test cluster
- **FR-034**: Release testing results MUST include metadata linking results to specific Radius version tested

#### Developer Experience

- **FR-035**: System MUST provide comprehensive documentation including setup guide, environment variable reference, troubleshooting guide, and architecture overview
- **FR-036**: Error messages MUST be actionable with specific remediation steps (not generic failures)
- **FR-037**: Make targets MUST have help text describing purpose and usage (visible via `make help`)
- **FR-038**: System MUST validate environment variables at start of execution and fail fast with clear error if required variables are missing or invalid
- **FR-039**: Infrastructure deployment MUST support "dry-run" mode that validates configuration without creating resources

### Key Entities

- **Release Metadata**: Represents a published Radius release including version string (e.g., "v0.35.0"), Helm chart download URL, container image registry and tags for all components (applications-rp, dynamic-rp, controller, ucpd, bicep), release notes URL, publication timestamp, and release type (stable, pre-release, draft)

- **Test Configuration**: Represents the complete configuration for a test run including target AKS cluster (name, resource group, subscription), Radius version being tested, test suites to execute (all or subset), execution mode (serial or parallel), timeout values, cloud provider credentials (Azure subscription, AWS access keys), container registries for pulling test images and recipes, and resource retention settings

- **AKS Cluster State**: Represents the current state of a test cluster including cluster health status (healthy, degraded, failed), installed Radius version, Kubernetes version, available node capacity, installed dependencies (Dapr, cert-manager, workload identity), configured test resources (CosmosDB connection, SQL connection), and provisioning metadata (creation timestamp, owner, purpose)

- **Test Results**: Represents outcomes of test execution including overall status (passed, failed, partial), individual test suite results (suite name, pass count, fail count, skip count, duration), failure details (test name, error message, stack trace, diagnostic artifact locations), total execution time, tested Radius version, cluster identifier, and execution timestamp

- **Test Resource Credentials**: Represents connection information for pre-provisioned test resources including CosmosDB MongoDB connection string and database name, Azure SQL Server resource ID, username, password, and database name, AWS account ID and region for AWS resources, and storage location (Kubernetes secret name, namespace, service account with read permissions)

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: A developer with no prior LRT experience can provision an AKS cluster, run full test suite, and destroy infrastructure in under 4 hours total (30 min setup reading docs, 45 min infra deployment, 120 min test execution, 15 min teardown, balance for interruptions/issues)

- **SC-002**: Contributors can successfully execute LRT workflows on their forks with zero modifications to workflow YAML files (only configuration via fork secrets/variables)

- **SC-003**: Cluster version is validated and upgraded (if needed) within 10 minutes at workflow start on 95% of executions (allowing for occasional upgrade failures requiring manual intervention)

- **SC-004**: Test coverage remains at 100% parity with current LRT system (all existing test suites execute with same coverage)

- **SC-005**: Failed tests provide actionable diagnostics in 90% of failures (developers can identify root cause from terminal output and error messages without additional investigation)

- **SC-006**: Infrastructure provisioning succeeds on first attempt for 85% of developers following documented setup steps (15% tolerance for environment-specific issues like quota limits)

- **SC-007**: Zero code duplication across workflow files and Make targets (all shared logic extracted to reusable scripts or composite Make targets)

- **SC-008**: Complete documentation exists covering setup (prerequisites, environment variables), usage (common commands), architecture (system components, data flow), and troubleshooting (common errors, resolution steps)

- **SC-009**: Health check completes in under 2 minutes and correctly identifies common issues (missing secrets, incorrect Radius version, cluster degradation) in 95% of misconfigurations

- **SC-010**: Cleanup mechanisms remove all test resources within 30 minutes of test completion for 99% of test runs (1% tolerance for edge cases requiring manual intervention)

- **SC-011**: Local test execution provides real-time feedback with test progress visible within 10 seconds of starting execution and incremental results displayed as tests complete
