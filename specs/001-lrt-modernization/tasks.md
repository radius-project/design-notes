# Tasks: Long-Running Test System Modernization

**Input**: Design documents from `/specs/001-lrt-modernization/`
**Prerequisites**: plan.md (required), spec.md (required for user stories), research.md, data-model.md, contracts/

**Tests**: Not explicitly requested in feature specification - focusing on implementation tasks

**Organization**: Tasks are grouped by user story to enable independent implementation and testing of each story.

## Progress Summary

- **Total Tasks**: 95
- **Completed**: 95 (100%)
- **Remaining**: 0 (0%)
- **MVP Tasks (P1)**: 50 (Phases 1-4)
- **MVP Completed**: 50 (100% of MVP) ✅

### Phase Breakdown

- Phase 1 (Setup): 4/4 complete ✅
- Phase 2 (Foundational): 10/10 complete ✅
- Phase 3 (User Story 1): 22/22 complete ✅
- Phase 4 (User Story 2): 14/14 complete ✅
- Phase 5 (User Story 3): 12/12 complete ✅
- Phase 6 (User Story 4): 10/10 complete ✅
- Phase 7 (User Story 5): 8/8 complete ✅
- Phase 8 (Polish): 15/15 complete ✅

---

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Can run in parallel (different files, no dependencies)
- **[Story]**: Which user story this task belongs to (e.g., US1, US2, US3)
- Include exact file paths in descriptions

## Path Conventions

All paths relative to `radius/` repository root:
- Build system: `build/`
- Infrastructure: `test/infra/azure/`
- Workflows: `.github/workflows/`
- Documentation: `docs/contributing/`

---

## Phase 1: Setup (Shared Infrastructure)

**Purpose**: Project initialization and directory structure for LRT system

- [X] T001 Create `test/infra/azure/scripts/` directory for deployment scripts
- [X] T002 Create `build/lrt.mk` for LRT-specific Make targets
- [X] T003 [P] Create `.github/instructions/lrt.instructions.md` for contributor guidance
- [X] T004 [P] Create `docs/contributing/lrt-testing.md` documentation directory

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: Core deployment scripts and Make infrastructure that ALL user stories depend on

**⚠️ CRITICAL**: No user story work can begin until this phase is complete

- [X] T005 Create `test/infra/azure/scripts/deploy-infra.sh` with argument parsing, validation, and Azure CLI deployment orchestration
- [X] T006 Create `test/infra/azure/scripts/destroy-infra.sh` with safety checks, retention tag validation, and cleanup logic
- [X] T007 Create `test/infra/azure/scripts/health-check.sh` with cluster connectivity, Radius version, dependency checks
- [X] T008 Create `test/infra/azure/scripts/setup-credentials.sh` with CosmosDB/SQL provisioning and Kubernetes secret creation
- [X] T009 Add `lrt-deploy-infra` target to `build/lrt.mk` calling deploy-infra.sh with environment variables
- [X] T010 Add `lrt-destroy-infra` target to `build/lrt.mk` calling destroy-infra.sh with force flag support
- [X] T011 Add `lrt-health-check` target to `build/lrt.mk` calling health-check.sh
- [X] T012 Add `lrt-setup-test-resources` target to `build/lrt.mk` calling setup-credentials.sh
- [X] T013 Include `build/lrt.mk` in main `Makefile` after existing test.mk inclusion
- [X] T014 Update `build/test.sh` to enhance idempotency with resource existence checks and resume logic

**Checkpoint**: Foundation ready - user story implementation can now begin in parallel

---

## Phase 3: User Story 1 - Developer Infrastructure Setup (Priority: P1) 🎯 MVP

**Goal**: Enable developers to provision AKS test infrastructure, run tests, and clean up using Make commands

**Independent Test**: Run `make lrt-deploy-infra`, verify AKS cluster created with all dependencies, run `make test-functional-all`, verify tests execute, run `make lrt-destroy-infra`, verify all resources deleted

### Implementation for User Story 1

- [X] T015 [P] [US1] Implement Azure CLI authentication check in `test/infra/azure/scripts/deploy-infra.sh`
- [X] T016 [P] [US1] Implement environment variable validation in `test/infra/azure/scripts/deploy-infra.sh` (AZURE_SUBSCRIPTION_ID, TEST_AKS_RG, TEST_AKS_AZURE_LOCATION)
- [X] T017 [P] [US1] Implement Azure permissions validation in `test/infra/azure/scripts/deploy-infra.sh` checking Contributor role
- [X] T018 [US1] Implement workload identity tag lookup in `test/infra/azure/scripts/deploy-infra.sh` with clear error if not found
- [X] T019 [US1] Implement Bicep deployment orchestration in `test/infra/azure/scripts/deploy-infra.sh` calling `az deployment group create`
- [X] T020 [US1] Implement kubeconfig update in `test/infra/azure/scripts/deploy-infra.sh` using `az aks get-credentials`
- [X] T021 [US1] Implement health check validation call at end of `test/infra/azure/scripts/deploy-infra.sh`
- [X] T022 [US1] Implement colored output helpers (print_info, print_success, print_error, print_warning) in `test/infra/azure/scripts/deploy-infra.sh`
- [X] T023 [US1] Implement trap handlers for cleanup on error in `test/infra/azure/scripts/deploy-infra.sh`
- [X] T024 [P] [US1] Implement resource group existence check in `test/infra/azure/scripts/destroy-infra.sh`
- [X] T025 [P] [US1] Implement retention tag validation in `test/infra/azure/scripts/destroy-infra.sh` preventing deletion of tagged resources
- [X] T026 [P] [US1] Implement confirmation prompt in `test/infra/azure/scripts/destroy-infra.sh` (skip if FORCE_DESTROY=true)
- [X] T027 [US1] Implement resource group deletion in `test/infra/azure/scripts/destroy-infra.sh` using `az group delete`
- [X] T028 [US1] Implement kubeconfig cleanup in `test/infra/azure/scripts/destroy-infra.sh` removing cluster context
- [X] T029 [P] [US1] Implement cluster connectivity check in `test/infra/azure/scripts/health-check.sh` using `kubectl version`
- [X] T030 [P] [US1] Implement Radius installation check in `test/infra/azure/scripts/health-check.sh` using `rad version --server`
- [X] T031 [P] [US1] Implement dependency checks in `test/infra/azure/scripts/health-check.sh` (Dapr, cert-manager pods)
- [X] T032 [US1] Implement test resource secret checks in `test/infra/azure/scripts/health-check.sh` (CosmosDB, SQL secrets)
- [X] T033 [US1] Implement node resource availability check in `test/infra/azure/scripts/health-check.sh`
- [X] T034 [US1] Add `--dry-run` flag support to `test/infra/azure/scripts/deploy-infra.sh` for configuration validation
- [X] T035 [US1] Update `test/infra/README.md` with Make target documentation and usage examples
- [X] T036 [US1] Create `docs/contributing/lrt-testing.md` with setup guide, prerequisites, and troubleshooting

**Checkpoint**: User Story 1 complete - developers can provision infrastructure, run tests, and clean up locally

---

## Phase 4: User Story 2 - Fork-Based Workflow Testing (Priority: P1)

**Goal**: Enable contributors to test LRT workflows on their forks using pre-deployed infrastructure in their own Azure subscriptions

**Independent Test**: Fork repository, manually deploy infrastructure with `make lrt-deploy-infra`, configure Azure OIDC credentials (client-id, tenant-id, subscription-id) and cluster name in fork secrets, push workflow change, trigger via workflow_dispatch, verify workflow authenticates to Azure using OIDC, sets kubectl context, and executes tests using pre-deployed infrastructure

### Implementation for User Story 2

#### T037: Create `.github/workflows/long-running-tests.yml` with workflow_dispatch trigger for manual execution
- [X] T037: Create `.github/workflows/long-running-tests.yml` with workflow_dispatch trigger for manual execution
- [X] T038 [P] [US2] Add schedule trigger to `.github/workflows/long-running-tests.yml` with repository filter (only radius-project/radius)
- [X] T039 [P] [US2] Add workflow inputs to `.github/workflows/long-running-tests.yml` (test_suites, resource_group, cluster_name)
- [X] T040 [US2] Add Azure login step to `.github/workflows/long-running-tests.yml` using OIDC authentication (azure/login@v2 with client-id, tenant-id, subscription-id from fork secrets)
- [X] T041 [US2] Add kubectl context setup step to `.github/workflows/long-running-tests.yml` calling `az aks get-credentials --resource-group ${{ inputs.resource_group }} --name ${{ inputs.cluster_name }}` (infrastructure must be pre-deployed manually)
- [X] T042 [US2] Add health check step to `.github/workflows/long-running-tests.yml` calling `make lrt-health-check`
- [X] T043 [US2] Add test execution step to `.github/workflows/long-running-tests.yml` calling `make lrt-test`
- [X] T044 [US2] Remove cleanup step from workflow (infrastructure is persistent and managed manually by developers, not destroyed by workflow)
- [X] T045 [US2] Add test results upload to `.github/workflows/long-running-tests.yml` using actions/upload-artifact
- [X] T046 [US2] Add repository filter job condition to `.github/workflows/long-running-tests.yml` preventing scheduled runs on forks
- [X] T047: Add run tests step with suite selection logic (parse test_suites input, invoke make targets)
- [X] T048: Add test results upload step (upload-artifact with dist/test_results/ and logs)
- [X] T049: Update workflow to document that infrastructure is pre-deployed and persistent (not destroyed by workflow)
- [X] T050: Create `.github/workflows/README.md` documenting fork testing setup (manual infrastructure deployment using personal Azure account, required OIDC secrets for workflow authentication: client-id/tenant-id/subscription-id, workflow execution)

**Checkpoint**: User Story 2 complete - contributors can test workflows on forks using pre-deployed infrastructure independently

---

## Phase 5: User Story 3 - Automated Release Testing (Priority: P2)

**Goal**: Automatically detect version mismatch and upgrade Radius cluster before running tests

**Independent Test**: Install older Radius version on cluster, install newer CLI, run workflow, verify system detects mismatch, upgrades cluster, executes tests

### Implementation for User Story 3

- [X] T051 [P] [US3] Create `test/infra/azure/scripts/version-check.sh` comparing CLI and cluster Radius versions
- [X] T052 [P] [US3] Create `test/infra/azure/scripts/upgrade-cluster.sh` executing `rad upgrade` command
- [X] T053 [US3] Implement version detection in `test/infra/azure/scripts/version-check.sh` using `rad version` and `rad version --server`
- [X] T054 [US3] Implement semantic version comparison in `test/infra/azure/scripts/version-check.sh`
- [X] T055 [US3] Implement upgrade recommendation output in `test/infra/azure/scripts/version-check.sh`
- [X] T056 [US3] Implement upgrade execution in `test/infra/azure/scripts/upgrade-cluster.sh` using `rad upgrade`
- [X] T057 [US3] Implement post-upgrade validation in `test/infra/azure/scripts/upgrade-cluster.sh` checking pod readiness and API version
- [X] T058 [US3] Add `lrt-version-check` target to `build/lrt.mk` calling version-check.sh
- [X] T059 [US3] Add `lrt-upgrade-cluster` target to `build/lrt.mk` calling upgrade-cluster.sh
- [X] T060 [US3] Add version check step to `.github/workflows/long-running-tests.yml` before test execution
- [X] T061 [US3] Add conditional upgrade step to `.github/workflows/long-running-tests.yml` if version mismatch detected
- [X] T062 [US3] Update `docs/contributing/lrt-testing.md` with version management documentation

**Checkpoint**: User Story 3 complete - automated version detection and upgrade working

---

## Phase 6: User Story 4 - Local Test Execution (Priority: P2)

**Goal**: Enable developers to run LRT test suites locally with real-time feedback

**Independent Test**: With provisioned cluster, run `make lrt-test`, verify all test suites execute with real-time output and summary

### Implementation for User Story 4

- [X] T063 [P] [US4] Add `lrt-test` target to `build/lrt.mk` as .PHONY meta-target invoking all LRT workflow test targets
- [X] T064 [US4] Add test suite dependencies to `lrt-test` target in `build/lrt.mk` (test-functional-corerp-cloud, test-functional-daprrp-cloud, etc.)
- [X] T065 [P] [US4] Implement CosmosDB provisioning in `test/infra/azure/scripts/setup-credentials.sh` using `az cosmosdb create`
- [X] T066 [P] [US4] Implement SQL Server provisioning in `test/infra/azure/scripts/setup-credentials.sh` using `az sql server create`
- [X] T067 [US4] Implement connection string retrieval in `test/infra/azure/scripts/setup-credentials.sh`
- [X] T068 [US4] Implement Kubernetes secret creation in `test/infra/azure/scripts/setup-credentials.sh` for CosmosDB and SQL
- [X] T069 [US4] Implement service account creation in `test/infra/azure/scripts/setup-credentials.sh` with RBAC for secret access
- [X] T070 [US4] Add SKIP_CLEANUP environment variable support to test execution in `build/lrt.mk`
- [X] T071 [US4] Add TEST_TIMEOUT environment variable support to test execution in `build/lrt.mk`
- [X] T072 [US4] Update `docs/contributing/lrt-testing.md` with test execution examples and troubleshooting

**Checkpoint**: User Story 4 complete - local test execution with credential management working

---

## Phase 7: User Story 5 - Test Environment Health Validation (Priority: P3)

**Goal**: Provide comprehensive health checks to prevent wasted time on broken environments

**Independent Test**: Run `make lrt-health-check` against healthy and intentionally misconfigured clusters, verify accurate detection and actionable errors

### Implementation for User Story 5

- [X] T073 [P] [US5] Add workload identity validation to `test/infra/azure/scripts/health-check.sh` checking OIDC configuration
- [X] T074 [P] [US5] Add detailed error messages with remediation steps to `test/infra/azure/scripts/health-check.sh`
- [X] T075 [P] [US5] Add version mismatch detection to `test/infra/azure/scripts/health-check.sh` comparing expected vs actual Radius version
- [X] T076 [US5] Add node resource capacity warnings to `test/infra/azure/scripts/health-check.sh` for insufficient CPU/memory
- [X] T077 [US5] Add environment variable validation to `test/infra/azure/scripts/health-check.sh` checking required vars are set
- [X] T078 [US5] Add timeout to health checks in `test/infra/azure/scripts/health-check.sh` (max 2 minutes)
- [X] T079 [US5] Add structured output format to `test/infra/azure/scripts/health-check.sh` with ✓/✗ indicators per component
- [X] T080 [US5] Update `docs/contributing/lrt-testing.md` with health check interpretation guide

**Checkpoint**: User Story 5 complete - comprehensive health validation implemented

---

## Phase 8: Polish & Cross-Cutting Concerns

**Purpose**: Improvements that affect multiple user stories and final cleanup

- [X] T081 [P] Create `test/infra/azure/scripts/cleanup-orphaned.sh` to find and remove abandoned resources older than threshold (default: 7 days)
- [X] T082 [P] Add `lrt-cleanup-orphaned` target to `build/lrt.mk` calling cleanup-orphaned.sh
- [X] T083 [P] Add help text to all Make targets in `build/lrt.mk` for `make help` integration
- [X] T084 [P] Update `build/help.mk` to include LRT targets if not automatically discovered
- [X] T085 [P] Add integration tests for deployment scripts in `test/infra/azure/scripts/test/`
- [X] T086 Update `test/infra/README.md` with complete Make target reference and architecture overview
- [X] T087 Add troubleshooting section to `docs/contributing/lrt-testing.md` with common errors and solutions
- [X] T088 Add quickstart validation to ensure `quickstart.md` examples work end-to-end
- [X] T089 Add security review for credential handling and secret management patterns
- [X] T090 Add performance benchmarks for infrastructure deployment and test execution times
- [X] T091 Add code duplication validation: Review scripts and Make targets to ensure zero duplicate logic (SC-007)
- [X] T092 Code cleanup: Ensure all scripts follow shell.instructions.md conventions (set -euo pipefail, trap handlers)
- [X] T093 Code cleanup: Ensure all Make targets follow make.instructions.md conventions (.PHONY, help text)
- [X] T094 Code cleanup: Ensure workflow follows github.workflows.instructions.md conventions (fork-testability, workflow_dispatch)
- [X] T095 Final documentation review: Verify all paths, commands, and examples are accurate and tested

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: No dependencies - can start immediately
- **Foundational (Phase 2)**: Depends on Setup completion - BLOCKS all user stories
- **User Stories (Phase 3-7)**: All depend on Foundational phase completion
  - US1 & US2 (P1): Can proceed in parallel after Foundational
  - US3 & US4 (P2): Can proceed in parallel after Foundational (US3 builds on US1, US4 builds on US1)
  - US5 (P3): Can proceed after Foundational (enhances US1's health-check)
- **Polish (Phase 8)**: Depends on all desired user stories being complete

### User Story Dependencies

- **User Story 1 (P1) - Infrastructure Setup**: Can start after Foundational (Phase 2) - No dependencies on other stories
- **User Story 2 (P1) - Fork Testing**: Can start after Foundational (Phase 2) - No dependencies, but uses Make targets from US1
- **User Story 3 (P2) - Automated Release Testing**: Can start after Foundational (Phase 2) - Adds version check to US1 workflow
- **User Story 4 (P2) - Local Test Execution**: Can start after Foundational (Phase 2) - Adds credential setup to US1 infrastructure
- **User Story 5 (P3) - Health Validation**: Can start after Foundational (Phase 2) - Enhances health-check.sh from Foundational

### Within Each User Story

- Setup tasks marked [P] can run in parallel
- Foundational tasks marked [P] can run in parallel within Phase 2
- Within user stories: tasks marked [P] can run in parallel (different files)
- Scripts before Make targets that call them
- Make targets before workflow steps that use them
- Implementation before documentation

### Parallel Opportunities

**Phase 1 - Setup**: All 4 tasks can run in parallel (different directories/files)

**Phase 2 - Foundational**: No parallelization - scripts must be created in sequence as they have logical dependencies

**Phase 3 - User Story 1**: 
- T015, T016, T017 can run in parallel (different validation sections in deploy-infra.sh)
- T024, T025, T026 can run in parallel (different validation sections in destroy-infra.sh)
- T029, T030, T031 can run in parallel (different check sections in health-check.sh)

**Phase 4 - User Story 2**: 
- T037, T038, T039 can run in parallel (different workflow sections)

**Phase 5 - User Story 3**: 
- T051, T052 can run in parallel (different scripts)
- T053, T054, T055 must be sequential (same script)

**Phase 6 - User Story 4**: 
- T063, T065, T066 can run in parallel (different files)

**Phase 7 - User Story 5**: 
- T073, T074, T075 can run in parallel (different validation sections)

**Phase 8 - Polish**: 
- T081, T082, T083, T084, T085, T086, T087, T088 can all run in parallel (different files)

---

## Parallel Example: User Story 1

```bash
# Launch validation checks in parallel:
Task T015: "Implement Azure CLI authentication check"
Task T016: "Implement environment variable validation"  
Task T017: "Implement Azure permissions validation"

# Launch destroy script validations in parallel:
Task T024: "Implement resource group existence check"
Task T025: "Implement retention tag validation"
Task T026: "Implement confirmation prompt"

# Launch health check components in parallel:
Task T029: "Implement cluster connectivity check"
Task T030: "Implement Radius installation check"
Task T031: "Implement dependency checks"
```

---

## Implementation Strategy

### MVP First (User Stories 1 & 2 Only - Both P1)

1. Complete Phase 1: Setup (4 tasks)
2. Complete Phase 2: Foundational (10 tasks) - **CRITICAL - blocks all stories**
3. Complete Phase 3: User Story 1 (22 tasks) - Developer infrastructure setup
4. Complete Phase 4: User Story 2 (14 tasks) - Fork-based workflow testing
5. **STOP and VALIDATE**: Test both US1 and US2 independently
6. Deploy/demo if ready - this delivers core value

**Estimated MVP Task Count**: 4 + 10 + 22 + 14 = **50 tasks**

### Incremental Delivery

1. **Foundation** (Phases 1-2): Setup + Foundational → 14 tasks
2. **MVP** (Phases 3-4): US1 + US2 → 36 tasks → **Deploy/Demo** ✓
3. **Enhancement 1** (Phase 5): US3 (Automated release testing) → 12 tasks → **Deploy/Demo** ✓
4. **Enhancement 2** (Phase 6): US4 (Local test execution) → 10 tasks → **Deploy/Demo** ✓
5. **Enhancement 3** (Phase 7): US5 (Health validation) → 8 tasks → **Deploy/Demo** ✓
6. **Polish** (Phase 8): Cross-cutting improvements → 15 tasks → **Final Release** ✓

Each increment adds value without breaking previous functionality.

### Parallel Team Strategy

With multiple developers:

**Week 1-2**: Team completes Foundation together (Phases 1-2)

**Week 3-4**: Once Foundational is done:
- **Developer A**: User Story 1 (Infrastructure setup)
- **Developer B**: User Story 2 (Fork workflows) - coordinates with Dev A on Make targets
- **Developer C**: Start on User Story 3 (Version checking) - can begin script development

**Week 5**: 
- **Developer A**: User Story 4 (Test execution) - builds on US1 infrastructure
- **Developer B**: User Story 5 (Health validation) - enhances health-check.sh
- **Developer C**: Finish User Story 3

**Week 6**: Team collaborates on Phase 8 (Polish) and final validation

---

## Task Summary

**Total Tasks**: 95

**Tasks by Phase**:
- Phase 1 (Setup): 4 tasks
- Phase 2 (Foundational): 10 tasks
- Phase 3 (US1 - P1): 22 tasks
- Phase 4 (US2 - P1): 14 tasks
- Phase 5 (US3 - P2): 12 tasks
- Phase 6 (US4 - P2): 10 tasks
- Phase 7 (US5 - P3): 8 tasks
- Phase 8 (Polish): 15 tasks

**Parallel Opportunities**: 28 tasks marked [P] can be executed in parallel

**Independent Test Criteria**:
- US1: Deploy infra → Run tests → Destroy infra sequence works end-to-end
- US2: Fork workflow executes using fork resources without organization access
- US3: Version mismatch detected and cluster upgraded automatically
- US4: Test suites run locally with credential access and real-time output
- US5: Health check accurately detects healthy and broken configurations

**Suggested MVP Scope**: Phases 1-4 (US1 & US2 only) = 50 tasks delivering core developer self-service and fork-based testing

---

## Notes

- [P] tasks = different files, no dependencies within that section
- [Story] label maps task to specific user story for traceability
- Each user story should be independently completable and testable
- All scripts must follow `.github/instructions/shell.instructions.md` conventions
- All Make targets must follow `.github/instructions/make.instructions.md` conventions  
- Workflow must follow `.github/instructions/github.workflows.instructions.md` conventions
- Commit after each logical group of tasks
- Stop at any checkpoint to validate story independently
- Implementation code goes in `radius/` repository, not `design-notes/`
