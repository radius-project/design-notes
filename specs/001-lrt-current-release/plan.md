# Implementation Plan: Long-Running Tests Use Current Release

**Branch**: `001-lrt-current-release` | **Date**: 2024-12-15 | **Spec**: [spec.md](spec.md)
**Input**: Feature specification from `/specs/001-lrt-current-release/spec.md`

## Summary

Update the long-running Azure test workflow to install the current official Radius release instead of building from the main branch. The workflow will use the official installer script to install the CLI, detect the control plane version on the cluster using `rad version`, and intelligently install/upgrade based on version comparison. All build-related logic (container image builds, caching, skip-build conditions) will be removed.

## Technical Context

**Language/Version**: YAML (GitHub Actions), Bash  
**Primary Dependencies**: Radius CLI (installed via official installer), `rad version`, `rad upgrade kubernetes`, `rad install kubernetes`  
**Storage**: N/A (workflow only)  
**Testing**: Manual workflow execution, CI validation  
**Target Platform**: GitHub Actions runners (ubuntu-24.04), AKS cluster  
**Project Type**: CI/CD workflow modification  
**Performance Goals**: Workflow completion time should be comparable or faster (no build step)  
**Constraints**: Must work with existing AKS test infrastructure, must handle version detection edge cases  
**Scale/Scope**: Single workflow file modification

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

| Principle | Status | Notes |
|-----------|--------|-------|
| I. API-First Design | ✅ Pass | N/A - workflow change, not API |
| II. Idiomatic Code Standards | ✅ Pass | Bash scripts follow shell.instructions.md, YAML follows github-workflows.instructions.md |
| III. Multi-Cloud Neutrality | ✅ Pass | N/A - test infrastructure only |
| IV. Testing Pyramid Discipline | ✅ Pass | Workflow tests itself via functional test execution |
| V. Collaboration-Centric Design | ✅ Pass | Simplifies workflow for all contributors |
| VI. Open Source and Community-First | ✅ Pass | Design documented in design-notes |
| VII. Simplicity Over Cleverness | ✅ Pass | Removes complexity (build logic), uses standard CLI commands |
| VIII. Separation of Concerns | ✅ Pass | Version detection logic in shell script, workflow handles orchestration |
| IX. Incremental Adoption | ✅ Pass | No breaking changes to other workflows |

## Project Structure

### Documentation (this feature)

```text
specs/001-lrt-current-release/
├── spec.md                      # Feature specification
├── plan.md                      # This file
├── technical-plan-context.md    # CLI command reference
├── research.md                  # Phase 0 output
├── checklists/
│   └── requirements.md          # Specification quality checklist
└── tasks.md                     # Phase 2 output (created by /speckit.tasks)
```

### Source Code (radius repository)

```text
radius/
├── .github/
│   ├── workflows/
│   │   └── long-running-azure.yaml    # Primary file to modify
│   └── scripts/
│       └── manage-radius-installation.sh  # New: version detection and install/upgrade logic
└── Makefile                           # No changes needed (tests run via existing make targets)
```

**Structure Decision**: Minimal changes - single workflow file modification plus a helper script for version management logic. The helper script follows the principle of extracting complex logic from workflow YAML into testable shell scripts.

## Complexity Tracking

No constitution violations. This change reduces complexity by removing:

- Build job (~200 lines)
- Skip-build logic (~50 lines)
- Caching logic (~40 lines)
- Build-related environment variables (~10 lines)

## Post-Design Constitution Re-Check

*Re-evaluated after Phase 1 design completion.*

| Principle | Status | Post-Design Notes |
|-----------|--------|-------------------|
| I. API-First Design | ✅ Pass | No API changes |
| II. Idiomatic Code Standards | ✅ Pass | Shell script follows `set -euo pipefail`, proper quoting, clear variable names |
| III. Multi-Cloud Neutrality | ✅ Pass | Test infrastructure only, no cloud-specific dependencies added |
| IV. Testing Pyramid Discipline | ✅ Pass | Functional tests validate the workflow; manual verification steps documented |
| V. Collaboration-Centric Design | ✅ Pass | Simplified workflow benefits all contributors |
| VI. Open Source and Community-First | ✅ Pass | Design fully documented in design-notes repository |
| VII. Simplicity Over Cleverness | ✅ Pass | Net reduction of ~300 lines; uses standard CLI commands |
| VIII. Separation of Concerns | ✅ Pass | Version logic in shell script, workflow handles orchestration |
| IX. Incremental Adoption | ✅ Pass | No breaking changes to other workflows or user-facing behavior |

**Gate Status**: ✅ PASSED - Ready for task breakdown (/speckit.tasks)

## Implementation Phases

### Phase 1: Remove Build Job

- Delete the entire `build` job from workflow
- Remove build-related environment variables
- Update workflow file header comments

### Phase 2: Add CLI Installation

- Add step to install CLI via official installer
- Add step to verify CLI installation
- Update PATH configuration

### Phase 3: Add Version Detection & Installation Logic

- Create `manage-radius-installation.sh` script
- Implement version parsing from `rad version`
- Implement conditional install/upgrade logic
- Add error handling for upgrade failures

### Phase 4: Update Tests Job

- Remove build job dependencies (`needs: build`, output references)
- Move recipe publishing steps (if still needed)
- Update step conditions (remove SKIP_BUILD references)

### Phase 5: Cleanup & Documentation

- Remove unused environment variables
- Update workflow file documentation/comments
- Test workflow execution

## References

- [Feature Specification](spec.md)
- [Technical Context](technical-plan-context.md)
- [Research Findings](research.md)
- [Quick Implementation Reference](quickstart.md)
- [GitHub Workflows Best Practices](../../.github/instructions/github-workflows.instructions.md)
- [Shell Script Guidelines](../../.github/instructions/shell.instructions.md)
