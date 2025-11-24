# design-notes Development Guidelines

Auto-generated from all feature plans. Last updated: 2025-11-16

## Repository Purpose

**This repository contains specifications, plans, and task lists only - NO implementation code.**

Implementation code resides in the following repositories within this workspace:
- **radius** - Main Radius control plane, CLI, and functional tests
- **dashboard** - Backstage-based UI for Radius
- **docs** - Documentation and guides
- **resource-types-contrib** - Community-contributed resource types and recipes

## Active Technologies

- Go 1.23+ (test framework), Bash 5.x (deployment scripts), Bicep latest (infrastructure as code) + magpiego (test framework), Azure CLI 2.x, kubectl 1.30.x, Helm 3.x, gotestsum (test runner) (001-lrt-modernization)

## Project Structure

```text
src/
tests/
```

## Commands

# Add commands for Go 1.23+ (test framework), Bash 5.x (deployment scripts), Bicep latest (infrastructure as code)

## Code Style

Go 1.23+ (test framework), Bash 5.x (deployment scripts), Bicep latest (infrastructure as code): Follow standard conventions

## Recent Changes

- 001-lrt-modernization: Added Go 1.23+ (test framework), Bash 5.x (deployment scripts), Bicep latest (infrastructure as code) + magpiego (test framework), Azure CLI 2.x, kubectl 1.30.x, Helm 3.x, gotestsum (test runner)

## Radius Repository Code Guidelines

When implementing features in the radius-project/radius repository, follow these comprehensive coding standards:

### Go Development
- **Reference**: `radius/.github/instructions/golang.instructions.md`
- Follow idiomatic Go practices and community standards
- Use `gofmt` for formatting, handle errors explicitly
- Provide godoc comments for all exported items
- Minimize exported surface area, leverage Go's simplicity

### Shell Scripting
- **Reference**: `radius/.github/instructions/shell.instructions.md`
- Use Bash conventions with `set -euo pipefail`
- Implement trap handlers for cleanup
- Use colored output helpers (print_info, print_success, print_error)
- Ensure idempotent operations and proper argument validation

### Makefile Development
- **Reference**: `radius/.github/instructions/make.instructions.md`
- Follow GNU Make best practices
- Use `.PHONY` for non-file targets
- Provide help text via `make help` integration
- Keep Make targets thin, delegate complex logic to scripts

### GitHub Workflows
- **Reference**: `radius/.github/instructions/github.workflows.instructions.md`
- Build testable, secure, and efficient CI/CD pipelines
- Emphasize fork-testability and local development patterns
- Extract workflow logic to Make targets and scripts
- Support workflow_dispatch for manual triggering on forks

<!-- MANUAL ADDITIONS START -->
<!-- MANUAL ADDITIONS END -->
