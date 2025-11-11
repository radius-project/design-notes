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

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Developer Infrastructure Setup (Priority: P1)

A Radius developer needs to provision their own isolated test environment to validate changes before submitting a PR. They should be able to deploy a complete AKS-based test infrastructure with monitoring, run the full LRT suite against their changes, and tear down the infrastructure when done—all from their local machine using simple commands. Each developer provisions a dedicated AKS cluster with no sharing to ensure complete isolation and prevent test interference.

**Why this priority**: This is the foundation that enables all other scenarios. Without the ability for developers to provision infrastructure and run tests locally, the entire modernization effort provides no value. This delivers immediate value by allowing developers to validate changes before pushing to forks.

**Independent Test**: Can be fully tested by a developer with only an Azure subscription running a sequence of commands (`make lrt-deploy-infra`, `make lrt-run-all`, `make lrt-destroy-infra`) and verifying that infrastructure is created, tests execute, and resources are cleaned up successfully. Delivers the value of independent test environments.

**Acceptance Scenarios**:

1. **Given** a developer has Azure CLI configured with valid credentials and required environment variables set, **When** they run `make lrt-deploy-infra`, **Then** an AKS cluster with all dependencies (Log Analytics, Grafana, Dapr, OIDC) is provisioned within 30 minutes and cluster readiness is validated
2. **Given** an AKS cluster has been provisioned successfully, **When** the developer runs `make lrt-run-all`, **Then** all functional tests execute against their cluster and results are displayed locally
3. **Given** tests have completed (successfully or with failures), **When** the developer runs `make lrt-destroy-infra`, **Then** all Azure resources in the test resource group are deleted within 15 minutes
4. **Given** a developer has only basic Azure permissions, **When** they attempt infrastructure deployment, **Then** the system provides clear error messages about missing permissions and documents the minimum required roles
5. **Given** infrastructure deployment fails mid-way, **When** the developer runs `make lrt-deploy-infra` again, **Then** the deployment resumes or rolls back safely without leaving orphaned resources

---

### User Story 2 - Fork-Based Workflow Testing (Priority: P1)

A Radius contributor working on a fork needs to test workflow changes before submitting a PR to the main repository. They should be able to push changes to their fork, manually trigger the LRT workflow via GitHub UI, and see test results—all without needing access to `radius-project` organization resources or secrets.

**Why this priority**: This addresses a critical pain point preventing contributors from validating workflow changes. Currently, workflows are tightly coupled to organization-specific resources, making it impossible to test on forks. This is equally critical as P1 because it enables workflow improvements and community contributions.

**Independent Test**: Can be fully tested by a contributor forking the repository, making a change to a workflow file, pushing to their fork, triggering the workflow via GitHub Actions UI with workflow_dispatch, and verifying that the workflow executes successfully using only resources in their own Azure subscription. Delivers independent validation of workflow correctness.

**Acceptance Scenarios**:

1. **Given** a contributor has forked the radius repository, **When** they configure their fork with Azure credentials and push a workflow change, **Then** they can manually trigger the LRT workflow via workflow_dispatch on any branch
2. **Given** a workflow is triggered on a fork, **When** the workflow executes, **Then** it uses environment variables/secrets configured in the fork (not radius-project organization secrets) and completes successfully
3. **Given** a contributor has no access to `ghcr.io/radius-project/*` registries, **When** they run the workflow on their fork, **Then** the workflow uses an alternative registry specified in their fork's configuration without code changes
4. **Given** a workflow is scheduled to run, **When** it triggers on a fork, **Then** the schedule is ignored and the workflow does not execute automatically (only manual dispatch works on forks)
5. **Given** workflow execution completes on a fork, **When** the contributor views results, **Then** they see detailed logs, test outcomes, and resource cleanup status in GitHub Actions UI

---

### User Story 3 - Automated Release Testing (Priority: P2)

A Radius maintainer needs confidence that each new release works correctly in realistic long-running scenarios. When the LRT workflow executes, the system should check if the cluster needs upgrading (by comparing CLI vs cluster versions), automatically upgrade if needed, run comprehensive functional tests against the current version, and report results to the team—without manual intervention.

**Why this priority**: While automated release testing is valuable, it's less critical than enabling developer self-service (P1). The team can manually trigger tests against releases initially. However, automation reduces maintenance burden and ensures consistent release validation.

**Independent Test**: Can be fully tested by installing an older Radius version on a cluster, installing a newer CLI version, running the LRT workflow, and verifying the system detects the version mismatch, upgrades the cluster, executes tests, and reports results to configured notification channels. Delivers automated version consistency without manual upgrade steps.

**Acceptance Scenarios**:

1. **Given** a test workflow starts execution, **When** the system checks versions, **Then** it compares the Radius CLI version against the cluster-installed version and triggers upgrade if cluster version is lower
2. **Given** cluster version is lower than CLI version, **When** the upgrade process begins, **Then** the cluster is upgraded to match CLI version using `rad upgrade` command
3. **Given** the cluster upgrade completes successfully, **When** functional tests are executed, **Then** all test suites run against the new version and results are collected
4. **Given** tests complete with any outcome, **When** results are available, **Then** a summary is posted to configured notification channels (Slack, GitHub issue, etc.) with pass/fail status and links to detailed logs
5. **Given** tests fail on a new release, **When** failures are detected, **Then** the system optionally rolls back to the previous Radius version (configurable behavior)
6. **Given** release detection fails due to GitHub API rate limiting, **When** the error is encountered, **Then** the system retries with exponential backoff and logs the error for investigation

---

### User Story 4 - Local Test Execution (Priority: P2)

A Radius developer working on a specific component wants to run just the LRT suite locally against their already-provisioned AKS cluster to validate changes iteratively during development. They should be able to run individual test suites or the complete suite using Make commands, see real-time output, and get fast feedback without waiting for CI/CD pipelines.

**Why this priority**: This enables rapid iteration during development. While developers can run tests, it's less critical than the ability to provision infrastructure (P1) which is a prerequisite. This story focuses on the testing phase after infrastructure exists.

**Independent Test**: Can be fully tested by a developer with a running AKS cluster executing `make lrt-test-corerp` for a specific suite or `make lrt-run-all` for all suites, observing real-time test output in their terminal, and seeing a summary of passes/failures. Delivers fast feedback for iterative development.

**Acceptance Scenarios**:

1. **Given** a developer has a configured AKS cluster and environment variables set, **When** they run `make lrt-run-all`, **Then** all functional test suites execute serially with progress displayed in real-time
2. **Given** a developer wants to test a specific component, **When** they run `make lrt-test-<component>` (e.g., `make lrt-test-corerp`), **Then** only that test suite executes, saving time during iterative development
3. **Given** tests are executing, **When** a test fails, **Then** the failure is logged with context (error message, relevant pod logs, resource state) and execution continues to remaining tests unless configured to fail-fast
4. **Given** test execution completes, **When** the developer reviews results, **Then** they see a summary with total tests, passes, failures, execution time, and paths to detailed logs
5. **Given** tests require pre-provisioned resources (CosmosDB, SQL), **When** the developer runs tests, **Then** the system retrieves connection details from Kubernetes secrets via service account RBAC and connects successfully

---

### User Story 5 - Test Environment Health Validation (Priority: P3)

Before running expensive long-running tests, developers and CI systems need to validate that the test environment is healthy and properly configured. They should be able to run a health check command that verifies all prerequisites (cluster connectivity, required resources, Radius installation) and get a clear pass/fail result with actionable error messages.

**Why this priority**: While health checks are valuable for preventing wasted time on broken environments, they're less critical than core testing functionality. Developers can manually verify prerequisites initially, and health checks can be added incrementally.

**Independent Test**: Can be fully tested by running `make lrt-health-check` against both healthy and intentionally misconfigured environments, verifying that the command correctly identifies issues (missing resources, incorrect versions, connectivity problems) and provides actionable guidance. Delivers fast failure detection before expensive test runs.

**Acceptance Scenarios**:

1. **Given** an AKS cluster has been provisioned, **When** a developer runs `make lrt-health-check`, **Then** the system verifies cluster connectivity, Radius installation, required Kubernetes resources, and pre-provisioned test resources
2. **Given** a health check detects missing prerequisites, **When** the check completes, **Then** it exits with non-zero code and displays clear error messages with remediation steps (e.g., "CosmosDB connection not found in AKS secrets - run `make lrt-setup-test-resources`")
3. **Given** all prerequisites are met, **When** health check runs, **Then** it completes in under 2 minutes and exits with zero code and confirmation message
4. **Given** Radius is installed but the wrong version, **When** health check runs, **Then** it reports version mismatch and suggests upgrade commands
5. **Given** cluster has insufficient resources, **When** health check runs, **Then** it warns about potential test failures due to resource constraints

---

### Edge Cases

- **What happens when release detection identifies a pre-release or draft release?** System should have configurable filters to only process stable releases (non-pre-release, non-draft) unless explicitly configured otherwise.

- **How does the system handle partial test failures?** By default, all test suites run to completion even if individual suites fail, providing complete results. A `fail-fast` configuration option allows stopping on first failure for faster feedback during development.

- **What if AKS cluster is in a failed or degraded state?** Health check detects degraded state and prevents test execution. System provides commands to re-provision infrastructure or recover cluster state.

- **How does version comparison work if CLI or cluster version cannot be determined?** System logs warning and proceeds with tests against current cluster state. Health check (FR-006) validates Radius installation is functional before test execution.

- **What happens if a developer's Bicep deployment fails mid-way?** Deployment script is idempotent - re-running continues from failure point or safely rolls back. Failed deployments don't leave orphaned resources due to Azure resource group lifecycle management.

- **How are test resources cleaned up when tests are interrupted (Ctrl+C, runner crash)?** Cleanup is implemented in trap handlers for shell scripts and deferred functions in Go code. Resource groups have auto-expiry tags. A separate `make lrt-cleanup-orphaned` command finds and removes abandoned resources.

- **What if multiple developers accidentally deploy to the same resource group?** This scenario is prevented by design - each developer provisions a dedicated AKS cluster in their own uniquely-named resource group (including user principal name hash or explicit prefix). Deployment scripts check for existing resources and fail with clear error if conflict detected, ensuring complete isolation between developer environments.

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
- **FR-005**: Infrastructure deployment MUST complete within 30 minutes for standard configuration (AKS, Log Analytics, no Grafana) and 45 minutes with Grafana enabled
- **FR-006**: System MUST provide a Make target (`make lrt-health-check`) that validates cluster readiness including connectivity, Radius installation, required resources, and test prerequisites

#### Release Detection & Upgrade

- **FR-007**: System MUST compare Radius CLI version against cluster-installed version at workflow start. If cluster version is lower than CLI version, system MUST upgrade cluster using `rad upgrade` command before executing tests.
- **FR-008**: System MUST filter releases based on configurable criteria (stable only vs. including pre-releases, version pattern matching)
- **FR-009**: Release detection MUST capture release metadata including version number, Helm chart URL, container image tags, and release notes URL
- **FR-010**: System MUST upgrade Radius installation on AKS cluster using `rad upgrade` command (preferred) or Helm upgrade commands as fallback
- **FR-011**: Upgrade process MUST verify successful upgrade by checking pod readiness and Radius API availability before proceeding to tests
- **FR-012**: System MUST support rollback to previous Radius version if tests fail after upgrade (configurable behavior, disabled by default)

#### Test Execution

- **FR-013**: System MUST provide Make targets for executing test suites individually (`make lrt-test-<component>`) and collectively (`make lrt-run-all`)
- **FR-014**: Test execution MUST display real-time progress and output to terminal when run locally
- **FR-015**: System MUST collect test results in structured format (JUnit XML or similar) for integration with CI systems
- **FR-016**: Test execution MUST complete within 120 minutes for full suite on standard AKS cluster configuration
- **FR-017**: System MUST support parallel test execution (configurable, serial by default for resource compatibility)
- **FR-018**: System MUST retrieve test resource credentials (CosmosDB, SQL Server) from Kubernetes secrets using service account RBAC (not from environment variables or GitHub organization secrets). Test pods MUST authenticate via service accounts with minimal required permissions.
- **FR-019**: Failed tests MUST generate diagnostic artifacts including relevant pod logs, resource state, and error context

#### Fork Compatibility

- **FR-020**: GitHub workflows MUST execute successfully on repository forks using only resources accessible to fork owner (not radius-project organization resources)
- **FR-021**: Workflows MUST support manual triggering via workflow_dispatch event on any branch
- **FR-022**: Scheduled workflow triggers MUST NOT execute on forks (schedule ignored unless repository matches configured organization/repository pattern)
- **FR-023**: Workflows MUST accept configuration through environment variables or repository secrets (container registries, cloud credentials, cluster names)
- **FR-024**: All core test logic MUST be implemented in Make targets and shell scripts, not embedded in workflow YAML (enabling local execution)

#### Test Resource Management

- **FR-025**: System MUST provide commands to setup pre-provisioned test resources (CosmosDB, SQL Server) and store credentials in Kubernetes secrets with appropriate RBAC policies. Service accounts for test execution MUST be created with read-only access to credential secrets.
- **FR-026**: System MUST clean up test-created cloud resources (Azure resource groups, AWS resources) after test execution completes
- **FR-027**: Cleanup MUST be implemented in trap handlers and deferred functions to execute even when tests are interrupted
- **FR-028**: System MUST support retention tags on resource groups to prevent accidental deletion of long-lived resources
- **FR-029**: System MUST provide a command (`make lrt-cleanup-orphaned`) to identify and remove abandoned test resources older than configured threshold

#### Reporting & Observability

- **FR-030**: Test results MUST be reported with summary including total tests, passes, failures, skipped tests, and execution time
- **FR-031**: System MUST support configurable notification channels for automated test runs (GitHub issue creation, Slack webhook, email)
- **FR-032**: Test execution MUST generate log artifacts including container logs, resource state dumps, and test framework output
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

- **SC-005**: Failed tests provide actionable diagnostics in 90% of failures (developers can identify root cause from logs and error messages without additional investigation)

- **SC-006**: Infrastructure provisioning succeeds on first attempt for 85% of developers following documented setup steps (15% tolerance for environment-specific issues like quota limits)

- **SC-007**: Zero code duplication across workflow files and Make targets (all shared logic extracted to reusable scripts or composite Make targets)

- **SC-008**: Complete documentation exists covering setup (prerequisites, environment variables), usage (common commands), architecture (system components, data flow), and troubleshooting (common errors, resolution steps)

- **SC-009**: Health check completes in under 2 minutes and correctly identifies common issues (missing secrets, incorrect Radius version, cluster degradation) in 95% of misconfigurations

- **SC-010**: Cleanup mechanisms remove all test resources within 30 minutes of test completion for 99% of test runs (1% tolerance for edge cases requiring manual intervention)

- **SC-011**: System handles transient failures gracefully with automatic retries for operations like release detection (GitHub API), cluster operations (transient k8s API errors), and resource provisioning (Azure RP throttling)

- **SC-012**: Local test execution provides real-time feedback with test progress visible within 10 seconds of starting execution and incremental results displayed as tests complete
