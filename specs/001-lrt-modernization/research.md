# Research: Long-Running Test System Modernization

**Feature**: LRT System Modernization  
**Date**: November 16, 2025  
**Status**: Complete

## Overview

This document captures research findings and technical decisions for modernizing the Radius Long-Running Test (LRT) system. Since this feature builds on existing infrastructure and patterns, most decisions involve selecting established technologies and approaches rather than evaluating new options.

## Key Technical Decisions

### Decision 1: Infrastructure Deployment Technology

**Decision**: Use existing Bicep templates in `test/infra/azure/main.bicep` with Azure CLI for deployment orchestration.

**Rationale**: 
- Bicep templates already exist and are proven for AKS deployment with all required dependencies (Log Analytics, Grafana, Dapr, OIDC)
- Azure CLI provides idempotent deployment via `az deployment group create` with retry/resume capabilities
- Aligns with Constitution Principle VI (Infrastructure as Code Integration) and VIII (Simplicity Over Cleverness)
- Team already familiar with this stack; no learning curve

**Alternatives Considered**:
- **Terraform**: Rejected because existing infrastructure is in Bicep and team expertise is stronger in Bicep for Azure resources
- **ARM templates**: Rejected because Bicep is the modern successor with better readability and maintainability
- **Azure Developer CLI (azd)**: Rejected because it adds unnecessary abstraction layer over existing proven Bicep deployment patterns

**Best Practices**:
- Use parameterized Bicep modules for reusability (already implemented in `test/infra/azure/modules/`)
- Validate deployments with `--what-if` flag for dry-run capability (FR-039)
- Use resource tags for lifecycle management and retention policies (FR-028)
- Implement deployment scripts with trap handlers for cleanup on failure

---

### Decision 2: Test Orchestration Framework

**Decision**: Use GNU Make for test orchestration with shell scripts for deployment logic.

**Rationale**:
- Existing Radius build system uses Make extensively (`build/test.mk`, `build/install.mk`)
- Make targets are simple, discoverable (`make help`), and work locally and in CI
- Shell scripts provide fine-grained control for deployment steps with standard tooling (Azure CLI, kubectl, helm)
- Aligns with Constitution Principle VIII (Simplicity Over Cleverness) - no custom orchestration framework needed

**Alternatives Considered**:
- **GitHub Actions composite actions**: Rejected because they're not executable locally, violating fork-testability and developer self-service requirements
- **Task/Just**: Rejected because Make is already established in the project and team is proficient
- **Python scripts**: Rejected because shell scripts are more direct for CLI orchestration and align with existing patterns

**Best Practices**:
- Follow Make best practices per `.github/instructions/make.instructions.md`
- Use `.PHONY` targets for all non-file-generating targets
- Provide help text for all targets via `make help` integration
- Keep Make targets thin - delegate to shell scripts for complex logic
- Use environment variables for configuration with sensible defaults

---

### Decision 3: Credential Storage and Access

**Decision**: Store credentials in Kubernetes Secrets, accessed by test pods via service account RBAC.

**Rationale**:
- Kubernetes-native approach eliminates external secret store dependencies
- RBAC ensures principle of least privilege - test pods only access required secrets
- Portable across any Kubernetes cluster (local k3d, AKS, future GKE/EKS)
- Aligns with Constitution Principle III (Multi-Cloud Neutrality) and FR-018/FR-041

**Alternatives Considered**:
- **Azure Key Vault with CSI driver**: Rejected because it couples tests to Azure-specific services and adds complexity for local development
- **Environment variables in workflow/developer env**: Rejected because it's insecure, doesn't work for pod-based tests, and violates least-privilege principles
- **GitHub organization secrets**: Rejected because it prevents fork-testability (FR-020)

**Best Practices**:
- Create dedicated service accounts per test suite with minimal required permissions
- Use Kubernetes RBAC to grant read-only access to specific secrets
- Rotate credentials regularly (documented in setup process)
- Never commit secrets to git or include in workflow YAML
- Provide script to provision test resources (CosmosDB, SQL) and create secrets (`setup-credentials.sh`)

---

### Decision 4: Test Execution Framework

**Decision**: Continue using existing magpiego functional test framework in `test/functional-portable/`.

**Rationale**:
- magpiego is already proven for Radius functional testing across all resource providers
- Tests are written in Go, aligning with Constitution Principle II (Idiomatic Code Standards)
- No changes to test code required - only orchestration changes needed
- Maintains consistency with existing test execution patterns

**Alternatives Considered**:
- **Rewrite tests in different framework**: Rejected because existing tests are comprehensive, stable, and team is proficient
- **Ginkgo/Gomega BDD framework**: Rejected because migration effort not justified by incremental benefits

**Best Practices**:
- Use gotestsum for improved test output formatting and JUnit XML generation (FR-015)
- Configure parallel execution with concurrency limits (FR-017: default concurrency=2)
- Implement fail-fast option for rapid feedback during development
- Collect test results in JUnit XML format for CI integration

---

### Decision 5: Workload Identity Management

**Decision**: Reuse Azure workload identity via tag-based discovery; fail deployment if tagged identity doesn't exist.

**Rationale**:
- Prevents proliferation of identities (one identity per developer would be unsustainable)
- Tag-based discovery allows flexible identity management by platform team
- Explicit failure with clear instructions ensures developers understand prerequisites (FR-040)
- Aligns with Constitution Principle V (Collaboration-Centric Design) - platform engineers control identity, developers use it

**Alternatives Considered**:
- **Auto-create identity per deployment**: Rejected because identity lifecycle management becomes complex and cleanup is unreliable
- **Hardcoded identity name**: Rejected because it prevents flexibility for different environments/subscriptions
- **User-assigned managed identity per cluster**: Rejected because it increases cost and management overhead

**Best Practices**:
- Document identity creation process with required tags and permissions
- Validate identity exists early in deployment (health-check phase)
- Use Azure RBAC to grant identity minimal required permissions (AKS contributor, resource reader)
- Provide troubleshooting guide for identity-related issues

---

### Decision 6: Cleanup Strategy

**Decision**: Cleanup runs by default with opt-out via `SKIP_CLEANUP=true` environment variable.

**Rationale**:
- Default cleanup prevents resource accumulation and runaway costs
- Opt-out preserves flexibility for debugging failed tests (FR-026)
- Trap handlers ensure cleanup runs even on script interruption (FR-027)
- Aligns with resource lifecycle best practices and cost management

**Alternatives Considered**:
- **Opt-in cleanup**: Rejected because developers often forget to clean up, leading to resource sprawl
- **Time-based retention**: Rejected because it requires additional tracking infrastructure and doesn't help with immediate resource cleanup
- **Manual cleanup only**: Rejected because it's error-prone and doesn't address interrupted test runs

**Best Practices**:
- Implement cleanup in shell trap handlers (`trap cleanup EXIT INT TERM`)
- Use Go defer functions for cleanup in test code
- Tag resources with creation timestamp and owner for orphan detection
- Provide `make lrt-cleanup-orphaned` command to find and remove abandoned resources (FR-029)
- Document cleanup behavior clearly in error messages and documentation

---

### Decision 7: Version Detection and Upgrade

**Decision**: Compare CLI version vs cluster version at workflow start; upgrade cluster using `rad upgrade` if cluster version is lower.

**Rationale**:
- Ensures tests run against expected Radius version (FR-007, FR-010)
- Uses official Radius upgrade command (maintained by team, tested path)
- Simple version comparison logic (semantic version string comparison)
- Aligns with automated release testing requirements (FR-011)

**Alternatives Considered**:
- **Always upgrade regardless of version**: Rejected because unnecessary upgrades waste time and introduce instability risk
- **Polling for new releases**: Rejected because test execution timing should be explicit, not reactive to external release events
- **Manual version management**: Rejected because it violates automation goals (US3)

**Best Practices**:
- Use `rad version` and `rad version --server` for version detection
- Validate upgrade success by checking pod readiness and API availability
- Fail fast if upgrade fails (don't proceed to tests with mismatched versions)
- Log detected versions and upgrade decisions for debugging

---

### Decision 8: Fork-Based Testing Approach

**Decision**: Extract all workflow logic to Make targets and shell scripts; workflows only orchestrate tool invocation using fork-configured secrets.

**Rationale**:
- Enables local execution of entire workflow (FR-024)
- Contributors can test workflow changes on forks using their own Azure resources (FR-020, FR-021)
- Aligns with Constitution Principle VII (Open Source and Community-First)
- Eliminates dependency on radius-project organization secrets

**Alternatives Considered**:
- **Separate fork-specific workflows**: Rejected because it creates maintenance burden and divergence between org and fork workflows
- **Conditional logic in workflows**: Rejected because it's complex, error-prone, and still embeds logic in YAML
- **Require access to organization runners**: Rejected because it prevents external contributor testing

**Best Practices**:
- Use workflow_dispatch for manual triggering on forks (FR-021)
- Disable scheduled triggers on forks via repository context check (FR-022)
- Document required secrets/variables for fork configuration (FR-023)
- Use environment variables for all configurable values (registry, cluster names, regions)
- Test workflow execution on a real fork before finalizing

---

## Technology Stack Summary

| Component | Technology | Version | Justification |
|-----------|-----------|---------|---------------|
| Infrastructure Deployment | Bicep | Latest | Existing templates, team expertise, ARM-native |
| Orchestration | GNU Make | 4.x+ | Existing build system, simple, discoverable |
| Deployment Scripts | Bash | 5.x+ | Standard shell, existing patterns, wide compatibility |
| Cloud Provider | Azure CLI | 2.x | Azure-native management, idempotent deployments |
| Kubernetes Client | kubectl | 1.30.x | Cluster version compatibility, standard tooling |
| Package Manager | Helm | 3.x | Existing Radius deployment method, chart management |
| Test Framework | magpiego (Go) | Existing | Proven functional test framework, no migration needed |
| Test Runner | gotestsum | Latest | JUnit XML generation, improved output formatting |
| Credential Storage | Kubernetes Secrets | Native | RBAC integration, cloud-neutral, simple |
| Identity Management | Azure Workload Identity | OIDC | AKS-native, secure, federated auth |

## Integration Patterns

### Deployment Flow
```
Developer -> make lrt-deploy-infra -> deploy-infra.sh -> Azure CLI + Bicep -> AKS cluster
                                    -> setup-credentials.sh -> Kubernetes Secrets
                                    -> health-check.sh -> Validation
```

### Test Execution Flow
```
Developer -> make test-functional-all -> gotestsum -> magpiego tests -> Kubernetes API -> Test resources
                                                                      -> Secrets (via RBAC)
```

### Cleanup Flow
```
Test completion -> trap handler -> destroy-infra.sh -> Azure CLI -> Resource group deletion
                                -> cleanup pods/secrets (if local K8s)
```

## Risk Mitigation

| Risk | Mitigation |
|------|-----------|
| Accidental deletion of long-lived resources | Resource tags + retention checks in destroy script (FR-028) |
| Deployment failures leaving orphaned resources | Idempotent deployments + orphan cleanup command (FR-029) |
| Credential exposure | Kubernetes Secrets + RBAC + no env vars/git commits |
| Fork contributors can't test workflows | Extract logic to Make/scripts + workflow_dispatch (FR-024) |
| Version mismatch between CLI and cluster | Automated version detection and upgrade (FR-007) |
| Quota limits preventing deployment | Pre-deployment quota validation + clear error messages (Edge case documented) |

## Open Questions / Future Research

None identified. All technical decisions are based on existing, proven technologies and patterns within the Radius project. Implementation can proceed to Phase 1 (Design & Contracts).
