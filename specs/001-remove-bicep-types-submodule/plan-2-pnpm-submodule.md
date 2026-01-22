# Implementation Plan 2: Migrate to pnpm + Remove bicep-types Submodule

**Branch**: `001-remove-bicep-types-submodule-pnpm` | **Date**: 2026-01-22 | **Spec**: [spec.md](./spec.md)
**Input**: Feature specification from `/specs/001-remove-bicep-types-submodule/spec.md`
**Depends On**: [Plan 1: Go Modules Migration](./plan-1-go-modules.md)

## Summary

Migrate all JavaScript/TypeScript tooling from npm to pnpm, update bicep-types npm dependencies to use pnpm git references, remove the bicep-types git submodule, update all CI/CD workflows, and update documentation. This is the second and final phase of the submodule removal migration.

## Technical Context

**Language/Version**: Node.js (per .node-version), TypeScript
**Primary Dependencies**:

- `bicep-types` npm package (currently via `file:` reference to submodule)
- Various npm packages in `typespec/`, `hack/bicep-types-radius/`
**Package Manager**: npm → pnpm migration
**Storage**: N/A
**Testing**: npm/pnpm scripts, `make test`
**Target Platform**: Linux (CI), macOS/Windows (developer machines)
**Project Type**: Monorepo with TypeScript tooling, Go services
**Performance Goals**: Build time should not regress; pnpm typically improves it
**Constraints**: Must maintain reproducible builds via lockfiles with commit SHA pinning
**Scale/Scope**:
- 3 npm package directories requiring pnpm migration
- 8 CI workflow files with 15 `submodules:` occurrences to remove
- Multiple Makefile targets using npm commands

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

| Principle | Status | Notes |
| --------- | ------ | ----- |
| **I. API-First Design** | ✅ PASS | No API changes - tooling/build system only |
| **II. Idiomatic Code Standards** | ✅ PASS | pnpm is modern, widely adopted package manager |
| **III. Multi-Cloud Neutrality** | ✅ PASS | No cloud-specific changes |
| **IV. Testing Pyramid Discipline** | ✅ PASS | Existing tests validate tooling; CI validates migration |
| **V. Collaboration-Centric Design** | ✅ PASS | Significantly improves contributor experience |
| **VI. Open Source and Community-First** | ✅ PASS | pnpm is open source, standard tooling |
| **VII. Simplicity Over Cleverness** | ✅ PASS | Replacing submodules with standard dependency management |
| **VIII. Separation of Concerns** | ✅ PASS | Clean dependency boundaries via pnpm |
| **IX. Incremental Adoption** | ✅ PASS | Migration includes contributor guide for existing clones |
| **XVI. Repository-Specific Standards** | ⚠️ CHECK | Dev container needs pnpm; devcontainer.json update required |
| **XVII. Polyglot Project Coherence** | ✅ PASS | Consistent with Node.js ecosystem patterns |

**Gate Result**: ✅ All gates pass (XVI addressed in implementation)

## Requirements Addressed

| Requirement | Coverage |
| ----------- | -------- |
| FR-001 | bicep-types git submodule completely removed |
| FR-002 | .gitmodules configuration removed |
| FR-003 | No `git submodule` commands required for build/test |
| FR-007 | All JS/TS tooling migrated to pnpm |
| FR-008 | hack/bicep-types-radius/ uses pnpm git references |
| FR-009 | Lockfiles updated for pnpm git subdirectory references |
| FR-010 | pnpm dependencies pinned to specific commit SHA |
| FR-011 | Makefiles function without submodule commands |
| FR-012 | Build scripts use pnpm |
| FR-013 | Workflow files have no submodule steps |
| FR-014 | All CI/CD workflows pass without submodule operations |
| FR-015 | Codegen workflows complete with new dependency sources |
| FR-016 | All regression tests pass |
| FR-017 | Dependabot configured for pnpm |
| FR-018 | Security scanning covers bicep-types dependencies |
| FR-019 | Dev container includes pnpm |
| FR-020 | CONTRIBUTING.md updated |
| FR-021 | One-time migration guide for existing clones |
| FR-022 | Go and pnpm setup steps documented |
| FR-023 | All READMEs updated |

## Project Structure

### Documentation (this feature)

```text
specs/001-remove-bicep-types-submodule/
├── spec.md                      # Feature specification
├── plan-1-go-modules.md         # Plan 1 (Go modules)
├── plan-2-pnpm-submodule.md     # This file (Plan 2)
├── research-2-pnpm.md           # Phase 0 research for Plan 2
├── data-model.md                # N/A for this feature
├── contracts/                   # N/A for this feature
├── quickstart.md                # Combined quickstart
└── tasks-2-pnpm-submodule.md    # Phase 2 tasks for Plan 2
```

### Source Code Changes (radius repository)

```text
radius/
├── .gitmodules                     # DELETE: Remove entirely
├── bicep-types/                    # DELETE: Remove submodule directory from git index
├── pnpm-workspace.yaml             # CREATE: Define pnpm workspace (if needed)
├── .npmrc                          # CREATE/MODIFY: pnpm configuration
│
├── typespec/
│   ├── package.json                # MODIFY: No changes to dependencies
│   ├── package-lock.json           # DELETE: Replace with pnpm-lock.yaml
│   └── pnpm-lock.yaml              # CREATE: pnpm lockfile
│
├── hack/bicep-types-radius/
│   └── src/
│       ├── autorest.bicep/
│       │   ├── package.json        # MODIFY: bicep-types from file: → git:
│       │   ├── package-lock.json   # DELETE
│       │   └── pnpm-lock.yaml      # CREATE
│       └── generator/
│           ├── package.json        # MODIFY: bicep-types from file: → git:
│           ├── package-lock.json   # DELETE
│           └── pnpm-lock.yaml      # CREATE
│
├── build/
│   └── generate.mk                 # MODIFY: npm → pnpm, remove submodule commands
│
├── .github/
│   ├── dependabot.yml              # MODIFY: npm → pnpm, remove gitsubmodule
│   └── workflows/
│       ├── build.yaml              # MODIFY: Remove submodules: recursive
│       ├── codeql.yml              # MODIFY: Remove submodules: recursive
│       ├── lint.yaml               # MODIFY: Remove submodules: recursive
│       ├── validate-bicep.yaml     # MODIFY: Remove submodules: true
│       ├── publish-docs.yaml       # MODIFY: Remove submodules: recursive
│       ├── long-running-azure.yaml # MODIFY: Remove submodules: recursive
│       ├── functional-test-noncloud.yaml  # MODIFY: Remove submodules: recursive
│       └── functional-test-cloud.yaml     # MODIFY: Remove submodules: recursive
│
├── .devcontainer/
│   └── devcontainer.json           # MODIFY: Add corepack pnpm activation to postCreateCommand
│
├── CONTRIBUTING.md                 # MODIFY: Update setup instructions
└── docs/contributing/
    └── migration-guide.md          # CREATE: One-time migration guide for existing clones
```

## Technical Approach

### Current State

**Package References** (in `hack/bicep-types-radius/src/*/package.json`):

```json
{
  "devDependencies": {
    "bicep-types": "file:../../../../bicep-types/src/bicep-types"
  }
}
```

**Makefile** (`build/generate.mk`):

```makefile
generate-bicep-types:
 git submodule update --init --recursive; \
 npm --prefix bicep-types/src/bicep-types install; \
 npm --prefix bicep-types/src/bicep-types ci && npm --prefix bicep-types/src/bicep-types run build; \
 npm --prefix hack/bicep-types-radius/src/autorest.bicep ci && ...
```

**Workflows** (multiple files):

```yaml
- uses: actions/checkout@<sha>
  with:
    submodules: recursive
```

**Dependabot**:

```yaml
- package-ecosystem: gitsubmodule
  directory: /
  schedule:
    interval: weekly
```

### Target State

**Package References** (pnpm git subdirectory reference):

```json
{
  "devDependencies": {
    "bicep-types": "github:Azure/bicep-types#<commit-sha>&path:/src/bicep-types"
  }
}
```

**Makefile**:

```makefile
generate-bicep-types:
 pnpm --prefix hack/bicep-types-radius/src/autorest.bicep install && \
 pnpm --prefix hack/bicep-types-radius/src/autorest.bicep run build; \
 pnpm --prefix hack/bicep-types-radius/src/generator install && \
 pnpm --prefix hack/bicep-types-radius/src/generator run generate -- ...
```

**Workflows**:

```yaml
- uses: actions/checkout@<sha>
  # No submodules property needed
```

**Dependabot** (add missing directories, remove gitsubmodule):

```yaml
# KEEP: Already exists in current config
- package-ecosystem: npm  # pnpm uses npm ecosystem in dependabot
  directory: /typespec
  ...

# ADD: New directories for pnpm migration
- package-ecosystem: npm
  directory: /hack/bicep-types-radius/src/generator
  ...

- package-ecosystem: npm
  directory: /hack/bicep-types-radius/src/autorest.bicep
  ...

# REMOVE: No longer needed
# - package-ecosystem: gitsubmodule
```

### Migration Steps

1. **Install pnpm**: Update dev container, document installation for contributors
2. **Migrate typespec/**: Convert npm to pnpm, generate lockfile
3. **Migrate hack/bicep-types-radius/**: Convert npm to pnpm, update bicep-types references to git
4. **Update Makefile**: Replace all npm commands with pnpm
5. **Update CI workflows**: Remove `submodules: recursive/true` from checkout steps, add pnpm/action-setup steps
6. **Remove submodule**: `git rm bicep-types`, delete `.gitmodules`
7. **Update Dependabot**: Remove `gitsubmodule`, add missing pnpm directories
8. **Update dev container**: Add corepack pnpm activation to postCreateCommand
9. **Update documentation**: CONTRIBUTING.md, create migration guide
10. **Verify**: Full CI pipeline passes

### Rollback Strategy

Standard git revert of the PR. Since this is atomic, reverting:

- Restores `.gitmodules` and submodule reference
- Restores npm lockfiles and commands
- Restores workflow submodule settings

Contributors would need to re-initialize submodule after revert.

## Research Required (Phase 0)

> **Status**: ✅ Complete - See [research-2-pnpm.md](./research-2-pnpm.md) for findings

1. **pnpm Git Reference Syntax**: ✅ Use `github:Azure/bicep-types#<sha>&path:/src/bicep-types`
2. **pnpm + Dependabot**: ✅ Use `package-ecosystem: npm`; git deps require manual updates
3. **bicep-types Build**: ✅ Package is self-contained, no local build required
4. **Workflow Caching**: ✅ Use `pnpm/action-setup@v4` with `actions/setup-node@v4` cache
5. **Dev Container pnpm**: ✅ Use corepack in postCreateCommand

## Design Artifacts (Phase 1)

### data-model.md

Not applicable - this is a build/tooling change with no data model changes.

### contracts/

Not applicable - no API changes.

### quickstart.md

Developer quickstart after migration (combined for both plans):

```bash
# Clone repository - no submodules needed!
git clone https://github.com/radius-project/radius
cd radius

# Enable pnpm via corepack (Node.js 16.13+)
corepack enable
corepack prepare pnpm@latest-10 --activate

# Install TypeScript dependencies
pnpm --prefix typespec install --frozen-lockfile
pnpm --prefix hack/bicep-types-radius/src/generator install --frozen-lockfile
pnpm --prefix hack/bicep-types-radius/src/autorest.bicep install --frozen-lockfile

# Verify Go dependencies
go mod download

# Build everything
make build

# Run tests
make test

# Generate Bicep types (if needed)
make generate-bicep-types
```

### Migration Guide for Existing Contributors

```bash
# If you have an existing clone with the submodule
cd radius

# Remove the submodule artifacts
rm -rf bicep-types
rm -rf .git/modules/bicep-types

# Remove stale npm artifacts
rm -rf hack/bicep-types-radius/src/*/node_modules
rm -rf typespec/node_modules

# Fetch latest changes
git fetch origin
git checkout main
git pull

# Install pnpm and dependencies
corepack enable
corepack prepare pnpm@latest-10 --activate
pnpm --prefix typespec install --frozen-lockfile
pnpm --prefix hack/bicep-types-radius/src/generator install --frozen-lockfile
pnpm --prefix hack/bicep-types-radius/src/autorest.bicep install --frozen-lockfile

# Verify build works
make build
```

## Dependencies

- **Upstream**: Plan 1 (Go modules) must be merged first
- **Upstream**: Azure/bicep-types npm package must be consumable via git reference

## Success Criteria

| Criterion | Validation |
| --------- | ---------- |
| SC-001 | New contributors complete setup in <10 minutes |
| SC-002 | Zero submodule-related build failures |
| SC-003 | 100% of workflows have no git submodule commands |
| SC-004 | Dependabot creates PRs for bicep-types pnpm updates |
| SC-005 | All regression tests pass |
| SC-006 | Git worktrees work without conflicts |
| SC-007 | Documentation has no submodule references |

## Risks and Mitigations

| Risk | Impact | Mitigation |
| ---- | ------ | ---------- |
| pnpm git subdirectory references not stable | HIGH | Research in Phase 0; test extensively before merge |
| bicep-types requires local build steps | MEDIUM | Document additional build steps if needed |
| Contributor confusion during transition | MEDIUM | Clear migration guide, announcement in release notes |
| CI cache invalidation causing slow builds | LOW | Configure pnpm caching properly in workflows |
| Dependabot not supporting pnpm git refs | LOW | Document manual update process as fallback |

## Workflow Files Requiring Updates

| File | Current Setting | Target Setting |
| ---- | --------------- | -------------- |
| `.github/workflows/build.yaml` | `submodules: recursive` (4 occurrences, lines 110, 212, 369, 436) | Remove property |
| `.github/workflows/codeql.yml` | `submodules: recursive` (line 95) | Remove property |
| `.github/workflows/lint.yaml` | `submodules: recursive` (line 58) | Remove property |
| `.github/workflows/validate-bicep.yaml` | `submodules: true` (line 64) | Remove property |
| `.github/workflows/publish-docs.yaml` | `submodules: recursive` (line 52) | Remove property |
| `.github/workflows/long-running-azure.yaml` | `submodules: recursive` (line 136) | Remove property |
| `.github/workflows/functional-test-noncloud.yaml` | `submodules: recursive` (line 208) | Remove property |
| `.github/workflows/functional-test-cloud.yaml` | `submodules: recursive` (4 occurrences, lines 172, 328, 336, 626) | Remove property |

**Total**: 15 occurrences across 8 workflow files

## Complexity Tracking

> No complexity violations identified. This plan simplifies the build system.

---

## Phase Summary

| Phase | Output | Status |
| ----- | ------ | ------ |
| Phase 0 | [research-2-pnpm.md](./research-2-pnpm.md) | ✅ COMPLETE |
| Phase 1 | quickstart.md (above), migration-guide.md (above) | ✅ INCLUDED |
| Phase 2 | tasks-2-pnpm-submodule.md | NOT STARTED (via /speckit.tasks) |
