# Tasks: Plan 2 - Migrate to pnpm + Remove bicep-types Submodule

**Input**: Design documents from `/specs/001-remove-bicep-types-submodule/`
**Prerequisites**: plan-2-pnpm-submodule.md (required), spec.md (required for user stories), research-2-pnpm.md
**Branch**: `001-remove-bicep-types-submodule-pnpm`
**Depends On**: Plan 1 (Go Modules) must be merged first

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Can run in parallel (different files, no dependencies)
- **[Story]**: Which user story this task belongs to (US1-US5)
- Include exact file paths in descriptions

## Scope

This plan addresses:

- FR-001: bicep-types git submodule completely removed
- FR-002: .gitmodules configuration removed
- FR-003: No `git submodule` commands required for build/test
- FR-007 through FR-023: pnpm migration, CI/CD updates, documentation

---

## Phase 1: Setup (Environment Preparation)

**Purpose**: Prepare pnpm tooling and verify prerequisites

- [x] T001 Verify Plan 1 (Go modules migration) is merged to main
- [x] T002 Verify pnpm is installed locally via `pnpm --version`
- [x] T003 Identify current bicep-types submodule commit SHA for pnpm git references via `git submodule status` in radius/

---

## Phase 2: Foundational (Create Working Branch)

**Purpose**: Prepare the codebase for migration

**⚠️ CRITICAL**: Complete before any user story implementation

- [x] T004 Create feature branch `001-remove-bicep-types-submodule-pnpm` from main
- [x] T005 Pull latest main to ensure Plan 1 changes are included
- [x] T006 Verify current build works via `make build` in radius/

**Checkpoint**: Foundation ready - pnpm migration can proceed

---

## Phase 3: User Story 1 - New Contributor Onboarding (Priority: P1) 🎯 MVP

**Goal**: New contributors can clone and build without submodule initialization

**Independent Test**: Clone repository fresh (no --recurse-submodules), run pnpm install and build commands successfully

### Implementation for User Story 1

#### pnpm Migration - typespec/

- [x] T007 [US1] Delete radius/typespec/package-lock.json
- [x] T008 [US1] Generate pnpm lockfile via `pnpm install` in radius/typespec/
- [x] T009 [US1] Verify typespec builds via `pnpm run build` in radius/typespec/

#### pnpm Migration - hack/bicep-types-radius/src/generator/

- [x] T010 [US1] Update radius/hack/bicep-types-radius/src/generator/package.json: change bicep-types from `file:` to `github:Azure/bicep-types#<sha>&path:/src/bicep-types`
- [x] T011 [US1] Delete radius/hack/bicep-types-radius/src/generator/package-lock.json
- [x] T012 [US1] Generate pnpm lockfile via `pnpm install` in radius/hack/bicep-types-radius/src/generator/
- [x] T013 [US1] Verify generator builds via `pnpm run build` in radius/hack/bicep-types-radius/src/generator/

#### pnpm Migration - hack/bicep-types-radius/src/autorest.bicep/

- [x] T014 [US1] Update radius/hack/bicep-types-radius/src/autorest.bicep/package.json: change bicep-types from `file:` to `github:Azure/bicep-types#<sha>&path:/src/bicep-types`
- [x] T015 [US1] Delete radius/hack/bicep-types-radius/src/autorest.bicep/package-lock.json
- [x] T016 [US1] Generate pnpm lockfile via `pnpm install` in radius/hack/bicep-types-radius/src/autorest.bicep/
- [x] T017 [US1] Verify autorest.bicep builds via `pnpm run build` in radius/hack/bicep-types-radius/src/autorest.bicep/

#### Submodule Removal

- [x] T018 [US1] Remove submodule from git index via `git rm bicep-types` in radius/
- [x] T019 [US1] Delete radius/.gitmodules file
- [x] T020 [US1] Clean up .git/modules/bicep-types directory via `rm -rf .git/modules/bicep-types`

#### Verify Build Works

- [x] T021 [US1] Verify full build via `make build` in radius/
- [x] T022 [US1] Verify tests via `make test` in radius/

**Checkpoint**: Repository builds and tests pass without submodule

---

## Phase 4: User Story 2 - CI/CD Build Reliability (Priority: P1)

**Goal**: All CI workflows pass without submodule initialization steps

**Independent Test**: CI pipeline completes successfully on PR

### Implementation for User Story 2

#### Makefile Updates

- [x] T023 [US2] Update radius/build/generate.mk: replace `npm` commands with `pnpm`
- [x] T024 [US2] Update radius/build/generate.mk: remove `git submodule update --init --recursive` commands
- [x] T025 [US2] Verify `make generate-bicep-types` works in radius/

#### Workflow Updates - build.yaml (4 occurrences)

- [x] T026 [US2] Update radius/.github/workflows/build.yaml line ~110: remove `submodules: recursive` from checkout step
- [x] T027 [US2] Update radius/.github/workflows/build.yaml line ~212: remove `submodules: recursive` from checkout step
- [x] T028 [US2] Update radius/.github/workflows/build.yaml line ~369: remove `submodules: recursive` from checkout step
- [x] T029 [US2] Update radius/.github/workflows/build.yaml line ~436: remove `submodules: recursive` from checkout step
- [x] T030 [US2] Add pnpm setup step to radius/.github/workflows/build.yaml (pnpm/action-setup@v4)

#### Workflow Updates - Other Files

- [x] T031 [P] [US2] Update radius/.github/workflows/codeql.yml line ~95: remove `submodules: recursive`, add pnpm setup
- [x] T032 [P] [US2] Update radius/.github/workflows/lint.yaml line ~58: remove `submodules: recursive`, add pnpm setup
- [x] T033 [P] [US2] Update radius/.github/workflows/validate-bicep.yaml line ~64: remove `submodules: true`, add pnpm setup
- [x] T034 [P] [US2] Update radius/.github/workflows/publish-docs.yaml line ~52: remove `submodules: recursive`, add pnpm setup
- [x] T035 [P] [US2] Update radius/.github/workflows/long-running-azure.yaml line ~136: remove `submodules: recursive`, add pnpm setup

#### Workflow Updates - Functional Tests (4 occurrences in cloud.yaml)

- [x] T036 [US2] Update radius/.github/workflows/functional-test-noncloud.yaml line ~208: remove `submodules: recursive`, add pnpm setup
- [x] T037 [US2] Update radius/.github/workflows/functional-test-cloud.yaml line ~172: remove `submodules: recursive`, add pnpm setup
- [x] T038 [US2] Update radius/.github/workflows/functional-test-cloud.yaml line ~328: remove `submodules: recursive`, add pnpm setup
- [x] T039 [US2] Update radius/.github/workflows/functional-test-cloud.yaml line ~336: remove `submodules: recursive`, add pnpm setup
- [x] T040 [US2] Update radius/.github/workflows/functional-test-cloud.yaml line ~626: remove `submodules: recursive`, add pnpm setup

**Checkpoint**: All 15 submodule references removed from 8 workflow files

---

## Phase 5: User Story 3 - Dependency Update Management (Priority: P2)

**Goal**: Dependabot monitors pnpm dependencies correctly

**Independent Test**: Dependabot configuration is valid and covers all pnpm directories

### Implementation for User Story 3

- [x] T041 [US3] Update radius/.github/dependabot.yml: remove `gitsubmodule` package-ecosystem section
- [x] T042 [P] [US3] Update radius/.github/dependabot.yml: add pnpm config for `/hack/bicep-types-radius/src/generator`
- [x] T043 [P] [US3] Update radius/.github/dependabot.yml: add pnpm config for `/hack/bicep-types-radius/src/autorest.bicep`
- [x] T044 [US3] Verify Dependabot config syntax via `actionlint` or manual review

**Checkpoint**: Dependabot configured for all pnpm directories

---

## Phase 6: User Story 4 - Git Worktree Support (Priority: P2)

**Goal**: Git worktrees work without submodule conflicts

**Independent Test**: Create worktree, build successfully

### Implementation for User Story 4

- [ ] T045 [US4] Test git worktree creation via `git worktree add ../radius-test feature-branch`
- [ ] T046 [US4] Verify build in worktree via `make build` in worktree directory
- [ ] T047 [US4] Clean up test worktree via `git worktree remove ../radius-test`

**Checkpoint**: Worktrees work without submodule issues

---

## Phase 7: User Story 5 - Documentation Clarity (Priority: P3)

**Goal**: All documentation reflects pnpm workflow without submodule references

**Independent Test**: Documentation accurately describes setup process

### Implementation for User Story 5

#### Dev Container

- [x] T048 [US5] Update radius/.devcontainer/devcontainer.json: add corepack pnpm activation to postCreateCommand

#### Contributing Documentation

- [x] T049 [P] [US5] Update radius/CONTRIBUTING.md: replace npm with pnpm, remove all submodule references
- [x] T050 [P] [US5] Create radius/docs/contributing/migration-guide.md with cleanup commands for existing clones

#### README Updates

- [x] T051 [P] [US5] Review and update radius/README.md if it mentions submodules
- [x] T052 [P] [US5] Review and update radius/hack/bicep-types-radius/README.md if it exists

**Checkpoint**: All documentation updated

---

## Phase 8: Polish & Validation

**Purpose**: Final verification before PR

- [x] T053 Run full `make build` in radius/
- [ ] T054 Run full `make test` in radius/
- [ ] T055 [P] Run `make generate-bicep-types` to verify codegen workflow
- [x] T056 [P] Verify pnpm lockfiles are consistent via `pnpm install --frozen-lockfile` in each directory
- [ ] T057 Fresh clone test: clone to new directory without `--recurse-submodules`, verify build
- [ ] T058 Create PR from `001-remove-bicep-types-submodule-pnpm` to main
- [ ] T059 Verify all CI workflows pass on PR

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: Depends on Plan 1 being merged
- **Foundational (Phase 2)**: Depends on Setup completion
- **User Story 1 (Phase 3)**: Depends on Foundational - core migration + submodule removal
- **User Story 2 (Phase 4)**: Depends on User Story 1 - workflows need submodule removed first
- **User Story 3 (Phase 5)**: Can start after User Story 2 (Dependabot config)
- **User Story 4 (Phase 6)**: Can run in parallel with Phase 5 (worktree testing)
- **User Story 5 (Phase 7)**: Can run in parallel with Phases 5-6
- **Polish (Phase 8)**: Depends on all user stories complete

### Within User Story 1 (Phase 3)

```text
pnpm migrations can run in parallel:
  ├── T007-T009: typespec/
  ├── T010-T013: generator/
  └── T014-T017: autorest.bicep/
Then sequentially:
  └── T018-T022: Submodule removal + verification
```

### Within User Story 2 (Phase 4)

```text
T023-T025: Makefile updates (sequential)
T026-T030: build.yaml updates (sequential - same file)
T031-T035: Other workflow files (parallel - different files)
T036-T040: Functional test workflows (sequential for cloud.yaml)
```

### Parallel Opportunities After Phase 4

```text
After Phase 4 (US2) completes:
  ├── T041-T044 [US3] Dependabot config
  ├── T045-T047 [US4] Worktree testing
  └── T048-T052 [US5] Documentation updates
```

---

## Implementation Strategy

### MVP (User Stories 1-2)

1. Complete Phases 1-4
2. **STOP and VALIDATE**: Repository builds, CI passes, no submodule
3. This is sufficient for core functionality

### Full Plan 2 Completion

1. Complete all phases
2. Merge PR
3. Feature complete: submodule removed, pnpm migrated

---

## Notes

- Plan 1 MUST be merged before starting this plan
- pnpm git references use format: `github:Azure/bicep-types#<sha>&path:/src/bicep-types`
- Use same commit SHA as was in submodule for initial migration
- 15 total `submodules:` occurrences across 8 workflow files to update
- Dev container requires corepack for pnpm activation
- Existing clones need migration guide for cleanup

---

## Summary

| Metric | Value |
| ------- | ----- |
| Total Tasks | 59 |
| Setup/Foundational | 6 |
| US1 (P1) | 16 |
| US2 (P1) | 18 |
| US3 (P2) | 4 |
| US4 (P2) | 3 |
| US5 (P3) | 5 |
| Polish | 7 |
| Parallel Opportunities | 15+ tasks can run in parallel (pnpm dirs, workflows, docs) |
| MVP Scope | Tasks T001-T040 (Phases 1-4) |
| Workflow Files Updated | 8 |
| Submodule References Removed | 15 |
