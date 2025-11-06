<!--
Sync Impact Report:
- Version change: 1.0.0 → 1.1.0 (MINOR: added new principle, governance clarifications)
- Modified principles: VIII. Simplicity Over Cleverness (clarified ordering rationale wording); Compliance section now references new principle explicitly
- Added sections: Principle IX (Incremental Adoption & Backward Compatibility); Quarterly Review note in Governance
- Removed sections: None
- Templates requiring updates:
  ✅ .specify/templates/plan-template.md (Constitution Check remains generic, no change required)
  ✅ .specify/templates/spec-template.md (Incremental adoption supported via independent user stories pattern)
  ✅ .specify/templates/tasks-template.md (Incremental delivery language already present; aligns with new principle)
  ⚠ .specify/templates/commands/* (Directory absent in repository; reference in prompt instructions only—TODO if future command templates are added)
- Follow-up TODOs:
  TODO(COMMAND_TEMPLATES): Create `.specify/templates/commands/` directory if command workflows are adopted to enable constitution alignment checks.
-->

# Radius Design Notes Constitution

## Core Principles

### I. API-First Design

All features MUST be designed with well-defined APIs before implementation begins. API definitions MUST use TypeSpec for generating OpenAPI specifications, following established patterns in the `typespec/` directory. APIs MUST be versioned according to semantic versioning with backward compatibility maintained within major versions. Resource Provider APIs MUST follow ARM-RPC patterns for consistency across the Radius control plane.

**Rationale**: Radius is fundamentally an API-driven platform enabling multi-cloud deployments and integration with various tools (Bicep, Terraform, Kubernetes). Clear API contracts ensure developers and platform engineers can collaborate effectively with well-understood interfaces.

### II. Go Idiomatic Code

All Go code MUST follow Effective Go patterns and conventions. Code MUST be formatted with `gofmt`. Every exported package, type, variable, constant, and function requires godoc comments explaining its purpose. Minimize exported surface area to simplify design. Code SHOULD leverage Go's simplicity rather than complex abstractions or object-oriented hierarchies. Errors MUST NOT be suppressed without explicit justification and handling of specific error types.

**Rationale**: Go's strength lies in its simplicity and directness. Adhering to idiomatic Go makes the codebase approachable for contributors, reduces cognitive load, and aligns with the broader Go ecosystem that Radius integrates with.

### III. Multi-Cloud Neutrality

All designs MUST account for multi-cloud deployment scenarios (Kubernetes, Azure, AWS, on-premises). Cloud-specific implementations MUST be abstracted behind provider interfaces defined in `pkg/aws/`, `pkg/azure/`, and `pkg/kubernetes/`. Portable resources MUST work across all supported platforms through the Recipe system. Features MUST NOT assume availability of cloud-specific services unless explicitly designed as cloud-specific extensions.

**Rationale**: Radius enables organizations to avoid cloud lock-in and deploy applications consistently across environments. This principle ensures the platform remains true to its core value proposition of cloud neutrality.

### IV. Testing Pyramid Discipline (NON-NEGOTIABLE)

Every feature MUST include comprehensive testing across three layers:

- **Unit tests**: Test individual functions and types in `pkg/` directories, runnable with basic prerequisites only (no external dependencies). Use `make test` to run all unit tests.
- **Integration tests**: Test features with dependencies (databases, external services, cloud providers) in appropriate `test/` subdirectories.
- **Functional tests**: End-to-end scenarios using the `magpiego` test framework in `test/functional/`, exercising realistic user workflows.

Tests MUST be written during feature implementation, not as an afterthought. Code coverage reports are reviewed in PRs to ensure adequate test coverage. New features MUST NOT be merged without corresponding tests at appropriate pyramid levels. Tests MUST fail before implementation (Red-Green-Refactor cycle).

**Rationale**: Quality and reliability are paramount for a platform managing critical infrastructure across multiple clouds. The testing pyramid ensures bugs are caught early and features remain stable as the codebase evolves.

### V. Collaboration-Centric Design

Design specifications MUST explicitly address how the feature enables collaboration between developers and platform engineers. Features MUST consider both perspectives:

- **Developer experience**: How does this simplify application authoring and deployment? Does it reduce cognitive load?
- **Platform engineer experience**: How does this enable governance, compliance, and operational best practices?

Environment and Recipe abstractions MUST be designed to allow platform engineers to define infrastructure patterns while giving developers the flexibility they need. APIs and tooling MUST support both audiences without forcing one perspective onto the other.

**Rationale**: Radius exists to bridge the gap between development and operations teams. Every feature should reinforce this collaboration rather than creating new silos or imposing unnecessary constraints.

### VI. Infrastructure as Code Integration

All infrastructure-related features MUST support Bicep as the primary authoring experience. Bicep type definitions in `bicep-types/` MUST be generated from TypeSpec definitions and kept in sync through `make generate`. Features SHOULD integrate with Terraform through the Recipe system where appropriate. Kubernetes manifests MUST be supported for native Kubernetes resources through existing integration patterns in `pkg/kubernetes/`.

**Rationale**: Organizations have existing IaC investments and expertise. Radius must meet teams where they are rather than forcing wholesale adoption of new tooling, enabling incremental adoption.

### VII. Open Source and Community-First

Design specifications MUST be authored in markdown and stored in the public `design-notes` repository before implementation begins. Significant features MUST follow the issue-first workflow at github.com/radius-project/radius, with community discussion before work begins. Design decisions MUST be documented with clear rationale. Breaking changes MUST be called out explicitly with migration guidance. All commits MUST include a `Signed-off-by` line (Developer Certificate of Origin).

**Rationale**: As a CNCF sandbox project, transparency and community involvement are essential. Public design discussions ensure better outcomes, build trust with the community, and align with open source governance practices.

### VIII. Simplicity Over Cleverness

Start simple and add complexity only when proven necessary through actual requirements. Question every abstraction layer—each one adds cognitive overhead. Optimize for correctness first, testability second, and simplicity third. Reject over-engineering and "future-proofing" in favor of solving immediate, well-understood requirements. Apply YAGNI (You Aren't Gonna Need It) principles rigorously.

**Rationale**: Premature complexity is the enemy of maintainability. Simple, direct solutions are easier to understand, test, debug, and evolve. Complexity should be justified by concrete needs, not hypothetical future scenarios.

### IX. Incremental Adoption & Backward Compatibility

Features, abstractions, and workflow changes MUST support gradual opt-in rather than forcing a disruptive migration. Breaking changes MUST provide a documented migration path and a deprecation period (minimum two release cycles) before removal. New abstractions MUST start behind feature flags or clearly labeled "experimental" status until validated by real usage. Backward compatibility MUST be maintained within a major version; removal or hard behavioral shifts REQUIRE either a guarded rollout or a MAJOR version bump with explicit migration guidance. Documentation MUST call out required user actions for any change that affects existing workflows.

**Rationale**: Radius integrates with diverse existing toolchains (Bicep, Terraform, Kubernetes). Enforcing big-bang changes erodes trust and slows adoption. Iterative, reversible evolution encourages early feedback, reduces risk, and preserves stability for production users.

## Technology Stack & Standards

### Supported Languages and Tools

- **Go**: Primary implementation language for control plane services (`pkg/corerp/`, `pkg/ucp/`, etc.), CLI (`pkg/cli/`), and controllers (`pkg/controllers/`). Version specified in `go.mod`.
- **TypeScript/Node.js**: TypeSpec definitions in `typespec/` directory, Bicep tooling in `bicep-tools/`, build scripts.
- **Python**: Code generation scripts in `hack/` and tooling automation.
- **Bicep**: Primary Infrastructure as Code language for Radius resource definitions.
- **TypeSpec**: API definition language for generating OpenAPI specifications in `swagger/`.

### Development Environment Requirements

All contributors MUST be able to develop using either:

- **VS Code with Dev Containers** (recommended): Pre-configured environment with all tools in `.devcontainer/devcontainer.json`
- **Local installation**: Following prerequisites documented in `docs/contributing/contributing-code/contributing-code-prerequisites/`

The dev container includes: Git, GitHub CLI, Go, Node.js, Python, gotestsum, kubectl, Helm, Docker, jq, k3d, kind, stern, Dapr CLI, and VS Code extensions (Go, Python, Bicep, Kubernetes, TypeSpec, YAML, shellcheck, Makefile Tools).

For local Kubernetes testing, prefer **k3d** as the primary tool. Secondarily consider **kind** for compatibility testing.

### Code Quality Standards

- **Formatting**: All Go code MUST be formatted with `gofmt` (enforced by `make format-check`)
- **Linting**: All code MUST pass `golangci-lint` checks (run via `make lint`)
- **Documentation**: All exported Go packages, types, variables, constants, and functions MUST have godoc comments
- **Security**: CodeQL security analysis findings MUST be addressed or explicitly justified with rationale
- **Generated Code**: All generated code (OpenAPI specs, Go types from specs, mocks via mockgen, Kubernetes API types via controller-gen) MUST be checked into source control and kept up-to-date via `make generate`
- **Dependencies**: Submodules (e.g., `bicep-types`) MUST be updated with `git submodule update --init --recursive` before building

## Development Workflow & Review

### Issue-First Development

Contributors MUST start by selecting an existing issue or creating a new issue on github.com/radius-project/radius before beginning work. For significant changes, maintainers MUST confirm the approach is in scope and aligns with project direction before implementation begins. Trivial changes (typos, minor documentation improvements) may proceed directly to PR without prior issue discussion.

### Commit and Pull Request Requirements

- All commits MUST include `Signed-off-by` line certifying Developer Certificate of Origin (use `git commit -s`)
- PRs MUST reference the issue they address in the description
- PR descriptions MUST explain what changed, why it changed, and any trade-offs considered
- Breaking changes MUST be clearly documented in PR descriptions with migration guidance
- PRs MUST pass all CI checks: `make build`, `make test`, `make lint`, `make format-check`
- Generated code MUST be up-to-date (run `make generate` and commit results if schemas or mocks changed)

### Design Specification Process for Major Features

Features requiring design specifications (new resource types, architectural changes, breaking changes) MUST follow the Spec Kit workflow in the `design-notes` repository:

1. **Constitution** (this document): Establish and validate project principles
2. **Specify** (`.specify/features/*/spec.md`): Define user scenarios, requirements, and success criteria (technology-agnostic)
3. **Plan** (`.specify/features/*/plan.md`): Create technical implementation plan with architecture, file structure, and constitution compliance check
4. **Tasks** (`.specify/features/*/tasks.md`): Break down plan into actionable, testable tasks organized by user story priority
5. **Implement**: Execute tasks according to plan with iterative validation and testing

Each phase MUST be reviewed and approved before proceeding to the next. The design note PR process in `design-notes` repository precedes implementation work in the `radius` repository.

### Code Review Standards

Reviewers MUST verify:

- **Principle Alignment**: Design and implementation align with constitution principles (especially Multi-Cloud Neutrality, Testing Pyramid, API-First)
- **Testing**: Appropriate tests are present across the testing pyramid; tests were written before or during implementation
- **API Contracts**: APIs are properly versioned using TypeSpec; OpenAPI specs are generated and checked in
- **Documentation**: godoc comments are complete for exported symbols; user-facing docs are updated if needed
- **Generated Code**: `make generate` has been run and all generated files are current
- **Commit Hygiene**: Conventional commit messages; Signed-off-by present; no merge commits
- **Error Handling**: Errors are not suppressed without justification; specific error types are handled appropriately
- **Complexity**: Any violations of simplicity principles (e.g., new abstraction layers) are justified with concrete requirements
- **Incremental Adoption**: Changes altering existing workflows include migration guidance, optionality (flags or config), and do not silently break existing deploy paths

## Governance

This constitution supersedes all other development practices for the Radius design notes repository. All design specifications, implementation plans, and pull requests MUST demonstrate compliance with these principles.

### Amendment Process

Amendments to this constitution require:

1. Proposal via GitHub issue in the `design-notes` repository with clear rationale for the change
2. Discussion and consensus among Radius maintainers and community stakeholders
3. Version bump according to semantic versioning:
   - **MAJOR**: Backward incompatible governance changes, principle removals, or fundamental redefinitions
   - **MINOR**: New principles added, sections materially expanded, or significant new guidance
   - **PATCH**: Clarifications, wording improvements, typo fixes, non-semantic refinements
4. Update to this document with Sync Impact Report documenting changes and affected artifacts
5. Approval from at least two maintainers before merging

### Compliance and Enforcement

All design specifications, plans, and implementation PRs MUST demonstrate compliance with this constitution. Maintainers MAY request changes to bring work into compliance with stated principles. Complexity that violates principles (especially Multi-Cloud Neutrality, Simplicity Over Cleverness, or Incremental Adoption & Backward Compatibility) MUST be justified with explicit trade-off analysis documented in the PR or design note.

### Periodic Review

This constitution MUST undergo a scheduled review at least quarterly (January, April, July, October) to assess relevance of principles, identify emerging gaps (e.g., security, observability evolution), and plan any prospective MINOR or MAJOR amendments transparently.

For day-to-day development guidance beyond this constitution, refer to:

- [CONTRIBUTING.md](https://github.com/radius-project/radius/blob/main/CONTRIBUTING.md) for contribution workflow
- [Developer guides](https://github.com/radius-project/radius/tree/main/docs/contributing) for detailed technical instructions
- [Code organization guide](https://github.com/radius-project/radius/blob/main/docs/contributing/contributing-code/contributing-code-organization/README.md) for repository structure

**Version**: 1.1.0 | **Ratified**: 2025-11-06 | **Last Amended**: 2025-11-06
