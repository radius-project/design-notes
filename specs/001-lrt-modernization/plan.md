# Implementation Plan: Long-Running Test System Modernization

**Branch**: `001-lrt-modernization` | **Date**: November 16, 2025 | **Spec**: [spec.md](./spec.md)
**Input**: Feature specification from `/specs/001-lrt-modernization/spec.md`

**Note**: This template is filled in by the `/speckit.plan` command. See `.specify/templates/commands/plan.md` for the execution workflow.

## Summary

Modernize the Radius Long-Running Test (LRT) system to enable developer self-service infrastructure provisioning, fork-based workflow testing, and automated release validation. The system will use existing Bicep templates for AKS cluster deployment, Make targets for test orchestration, shell scripts for deployment logic, and Go-based functional tests (magpiego framework) for test execution. Infrastructure provisioning will be idempotent, credentials will be stored in Kubernetes secrets with RBAC, and all logic will be extracted from workflow YAML to enable local execution.

## Technical Context

**Language/Version**: Go 1.23+ (test framework), Bash 5.x (deployment scripts), Bicep latest (infrastructure as code)
**Primary Dependencies**: magpiego (test framework), Azure CLI 2.x, kubectl 1.30.x, Helm 3.x, gotestsum (test runner)
**Storage**: Kubernetes Secrets (for credentials), Azure resource groups (for infrastructure lifecycle management)
**Testing**: magpiego functional test framework in `test/functional-portable/*`, existing test suite structure
**Target Platform**: AKS 1.30+ clusters on Azure (primary), local Kubernetes (k3d/kind) for development
**Project Type**: Infrastructure testing system (deployment automation + functional test orchestration)
**Performance Goals**: Infrastructure deployment <30 min, full test suite <120 min, health checks <2 min
**Constraints**: Fork-testable workflows (no organization secrets), idempotent deployments, cleanup by default with opt-out, OIDC authentication for workflows (no service principals)
**Scale/Scope**: ~20 test suites, ~5 developers running simultaneously, infrastructure per-developer (no sharing)
**Authentication**: GitHub workflows use Azure OIDC (client-id/tenant-id/subscription-id), developers use personal Azure accounts locally

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

| Principle | Status | Notes |
|-----------|--------|-------|
| **I. API-First Design** | ✅ PASS | This feature does not introduce new Radius APIs - it modernizes test infrastructure and workflows. No TypeSpec/OpenAPI changes required. |
| **II. Idiomatic Code Standards** | ✅ PASS | Will follow Go conventions for test code (existing magpiego), Bash best practices for deployment scripts (per `.github/instructions/shell.instructions.md`), Bicep best practices for infrastructure templates (existing patterns in `test/infra/azure/`). |
| **III. Multi-Cloud Neutrality** | ✅ PASS | Initial implementation targets AKS (Azure) as existing pattern, but architecture supports future extension to AWS/GCP through provider-specific Bicep/Terraform modules. Core testing logic remains cloud-neutral. |
| **IV. Testing Pyramid Discipline** | ✅ PASS | This IS the testing infrastructure. Will include: (1) Unit tests for new shell script functions via bats/shunit2, (2) Integration tests validating deployment scripts create expected resources, (3) Functional validation through existing magpiego test execution against provisioned infrastructure. |
| **V. Collaboration-Centric Design** | ✅ PASS | Enables both developers (self-service testing) and platform engineers (validating infrastructure patterns). Developers get fast feedback; maintainers get automated release validation. |
| **VI. Infrastructure as Code Integration** | ✅ PASS | Uses existing Bicep templates in `test/infra/azure/main.bicep` and modules. All infrastructure defined as code, version-controlled, and deployed via Azure CLI + Bicep. |
| **VII. Open Source and Community-First** | ✅ PASS | Design specification in public design-notes repo before implementation. Fork-testability enables community contributors to validate workflow changes independently. |
| **VIII. Simplicity Over Cleverness** | ✅ PASS | Leverages existing infrastructure (Bicep templates already exist), existing test framework (magpiego), and simple Make + Bash orchestration. No new abstractions introduced. |
| **IX. Incremental Adoption & Backward Compatibility** | ✅ PASS | New Make targets are additive (e.g., `make lrt-deploy-infra`). Existing test targets (`make test-functional-all`) continue working. Developers opt-in to new workflow; existing CI/CD unaffected until explicit migration. |
| **X-XVII. Repository-Specific Standards** | ✅ PASS | Work confined to radius repo. Follows existing Go testing patterns, shell script conventions, and Makefile structure. No dashboard, resource-types-contrib, or docs changes required. |

**Overall Assessment**: ✅ **ALL GATES PASS** - No violations requiring justification. Feature aligns with all applicable constitution principles.

## Project Structure

### Documentation (this feature)

```text
specs/001-lrt-modernization/
├── plan.md              # This file (/speckit.plan command output)
├── research.md          # Phase 0 output (/speckit.plan command)
├── data-model.md        # Phase 1 output (/speckit.plan command)
├── quickstart.md        # Phase 1 output (/speckit.plan command)
├── contracts/           # Phase 1 output (/speckit.plan command)
└── tasks.md             # Phase 2 output (/speckit.tasks command - NOT created by /speckit.plan)
```

### Source Code (radius repository)

```text
# Infrastructure deployment (existing + enhancements)
test/infra/
├── README.md            # Update with new Make targets and workflow
├── azure/
│   ├── main.bicep       # Existing - main AKS deployment template
│   ├── modules/         # Existing - Log Analytics, AKS, monitoring modules
│   └── scripts/         # NEW - deployment helper scripts
│       ├── deploy-infra.sh      # Orchestrates Bicep deployment with validation
│       ├── destroy-infra.sh     # Cleanup script with safety checks
│       ├── health-check.sh      # Pre-flight validation script
│       └── setup-credentials.sh # Provision test resources and create K8s secrets

# Build system integration
build/
├── test.mk              # Update with new LRT targets
├── test.sh              # Existing deployment script - enhance for idempotency
└── lrt.mk               # NEW - LRT-specific Make targets extracted from workflow logic

# GitHub workflows (fork-compatible - uses pre-deployed infrastructure)
.github/
├── workflows/
│   └── long-running-tests.yml  # NEW or enhanced - fork-testable LRT workflow (OIDC auth + runs tests, does NOT deploy infra)
└── instructions/
    └── lrt.instructions.md     # NEW - LRT testing guidance for contributors

# Functional tests (existing structure - no changes to test code itself)
test/functional-portable/
├── corerp/              # Existing test suites
├── daprrp/
├── ucp/
└── ...                  # All existing test directories remain unchanged

# Documentation
docs/
└── contributing/
    └── lrt-testing.md   # NEW - Developer guide for LRT system usage
```

**Structure Decision**: Single project structure within the existing radius repository. This is infrastructure testing tooling, not a new service or application. All new code is added to existing directories following established patterns:
- Infrastructure templates extend existing `test/infra/azure/` Bicep modules
- Deployment scripts added under `test/infra/azure/scripts/` for clear separation
- Make targets added to `build/` directory following existing `test.mk`, `install.mk` pattern
- Workflows added/modified in `.github/workflows/` per existing CI/CD structure
- Functional test code in `test/functional-portable/` remains unchanged (only orchestration changes)

## Complexity Tracking

> **Fill ONLY if Constitution Check has violations that must be justified**

*No violations recorded. All constitution gates passed without requiring complexity justification.*
