# Feature Specification: Git App Graph Preview

**Feature Branch**: `001-git-app-graph-preview`   
**Created**: January 30, 2026   
**Status**: Draft   
**Input**: User description: "Radius currently stores the state of application deployments as an app graph within its data store. Today, the app graph does not get generated until the application is deployed. Help me build an app graph representation for applications that are defined (e.g. in an app.bicep file) but not yet deployed. Additionally, enrich the app graph representation with git changelog info (i.e. git commit data) so that I may use this data to visualize how the app graph changes over time (i.e. across commits). The ultimate goal is to be able to visualize the app graph and do diffs of the app graph in GitHub on PRs, commit comparisons, etc."   

## Problem Statement

Radius currently generates application graphs only after deployment, which means:
1. Developers cannot preview the app graph structure before deploying
2. There's no way to track how the application architecture evolves over time
3. PR reviewers cannot see the impact of Bicep changes on the overall application topology
4. No mechanism exists to compare app graphs across commits or branches

This feature introduces **static app graph generation** from Bicep files and **git-aware graph versioning** to enable visualization and diffing in GitHub workflows.

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Generate App Graph from Bicep Files (Priority: P1)

As a **developer**, I want to generate an app graph from my `app.bicep` file without deploying, so I can preview the application topology and validate my changes locally.

As a **platform engineer**, I want to review app graph changes in PRs, so I can ensure architectural changes align with organizational standards before deployment.

**Why this priority**: This is the foundational capability. Without static graph generation, no other features can function. It delivers immediate value by enabling local validation.

**Independent Test**: Can be fully tested by running a CLI command against a Bicep file and verifying the graph output matches expected structure.

**Acceptance Scenarios**:

1. **Given** a valid `app.bicep` file with container, gateway, and database resources, **When** I run `rad app graph app.bicep`, **Then** I receive a JSON graph representation showing all resources and their connections.

2. **Given** a Bicep file with syntax errors, **When** I run `rad app graph app.bicep`, **Then** I receive a clear error message indicating the parsing failure with line/column information.

3. **Given** a Bicep file referencing external modules, **When** I run `rad app graph app.bicep`, **Then** the graph includes resources from all referenced modules with proper dependency tracking.

4. **Given** a Bicep file with parameterized values, **When** I run `rad app graph app.bicep --parameters params.json`, **Then** the graph reflects the resolved parameter values.

5. **Given** a Bicep file using the Radius Bicep extension types, **When** I run `rad app graph app.bicep`, **Then** the graph correctly identifies Radius-specific resource types and their relationships.

---

### User Story 2 - Export Graph as Diff-Friendly Format (Priority: P1)

As a developer, I want the app graph exported in a deterministic, diff-friendly format, so I can commit it to version control and see meaningful diffs when it changes.

**Why this priority**: Critical for enabling GitHub integration. Without a stable, diffable format, PR visualization is impossible.

**Independent Test**: Generate graph twice from identical Bicep, verify outputs are byte-identical. Modify Bicep, regenerate, verify diff shows only the changed elements.

**Acceptance Scenarios**:

1. **Given** an app graph, **When** I export it, **Then** the output is deterministically sorted (alphabetical by resource ID) producing identical output for identical inputs.

2. **Given** an app graph, **When** I export to markdown format, **Then** I receive a human-readable document with a resource table and ASCII connection diagram suitable for GitHub rendering.

3. **Given** an app graph, **When** I export to JSON format, **Then** I receive a structured document with stable key ordering that produces minimal diffs when resources change.

4. **Given** two app graphs from different commits, **When** I diff them, **Then** added resources show as additions, removed resources show as deletions, and modified connections are clearly visible.

---

### User Story 3 - Enrich Graph with Git Metadata (Priority: P2)

As a developer, I want the app graph to include git commit information, so I can track when and why each resource was added or modified.

**Why this priority**: Builds on P1 capabilities to enable historical tracking. Valuable but not blocking core functionality.

**Independent Test**: Generate graph with git enrichment enabled, verify each resource includes commit SHA, author, and timestamp of last modification.

**Acceptance Scenarios**:

1. **Given** a Bicep file in a git repository, **When** I generate the graph with `--git-enrich`, **Then** each resource includes the commit SHA, author, date, and message of its last modification.

2. **Given** a resource defined across multiple Bicep files, **When** I generate the enriched graph, **Then** the resource shows the most recent commit that affected any of its defining files.

3. **Given** a newly added resource not yet committed, **When** I generate the enriched graph, **Then** the resource is marked as "uncommitted" with the current working directory state.

4. **Given** a graph with git metadata, **When** I export to markdown, **Then** each resource row includes a linked commit SHA (e.g., `[abc123](../../commit/abc123)`).

---

### User Story 4 - GitHub Action for PR Graph Diff (Priority: P2)

As a PR reviewer, I want to see a visual diff of the app graph in PR comments, so I can understand the architectural impact of code changes without deploying.

**Why this priority**: High-value GitHub integration, but depends on P1 capabilities being stable.

**Independent Test**: Create a PR with Bicep changes, verify the action posts a comment showing before/after graph comparison.

**Acceptance Scenarios**:

1. **Given** a PR that modifies Bicep files, **When** the GitHub Action runs, **Then** it posts a comment showing the graph diff with added/removed/modified resources highlighted.

2. **Given** a PR with no Bicep changes, **When** the GitHub Action runs, **Then** it posts a comment indicating "No app graph changes detected."

3. **Given** a PR that adds a new connection between resources, **When** the GitHub Action runs, **Then** the diff clearly shows the new connection with source and target resources.

4. **Given** a PR comment already exists from a previous run, **When** the PR is updated and the action runs again, **Then** the existing comment is updated rather than creating a duplicate.

---

### User Story 5 - Historical Graph Timeline (Priority: P3)

As a developer, I want to view how my app graph evolved across commits, so I can understand architectural decisions and identify when changes were introduced.

**Why this priority**: Advanced feature for historical analysis. Valuable for debugging and auditing but not essential for core workflow.

**Independent Test**: Generate timeline for last 10 commits, verify each entry shows the graph state and changes from previous commit.

**Acceptance Scenarios**:

1. **Given** a git repository with multiple commits affecting Bicep files, **When** I run `rad app graph history app.bicep --commits 10`, **Then** I receive a timeline showing graph snapshots at each commit with change summaries.

2. **Given** a specific commit SHA, **When** I run `rad app graph app.bicep --at abc123`, **Then** I receive the app graph as it existed at that commit.

3. **Given** two commit SHAs, **When** I run `rad app graph diff app.bicep --from abc123 --to def456`, **Then** I receive a detailed diff showing all graph changes between those commits.

---

### User Story 6 - Mermaid Diagram Generation (Priority: P3)

As a developer, I want to export the app graph as a Mermaid diagram, so I can embed it in documentation and have GitHub render it automatically.

**Why this priority**: Nice-to-have visualization enhancement. GitHub natively renders Mermaid in markdown.

**Independent Test**: Export graph to Mermaid format, paste into GitHub markdown file, verify it renders as expected.

**Acceptance Scenarios**:

1. **Given** an app graph, **When** I export with `--format mermaid`, **Then** I receive valid Mermaid flowchart syntax that renders correctly in GitHub.

2. **Given** a graph with multiple resource types, **When** I export to Mermaid, **Then** different resource types use distinct shapes (e.g., containers as rectangles, gateways as diamonds, databases as cylinders).

3. **Given** a graph with connection directions, **When** I export to Mermaid, **Then** arrows indicate the direction of dependencies (outbound connections).

---

### Edge Cases

- What happens when Bicep file references resources outside the current file/module that cannot be resolved?
  - Generate partial graph with unresolved references marked as "external" placeholders
- How does the system handle circular dependencies in Bicep?
  - Detect and report cycles with clear error messaging; still generate graph with cycle annotation
- What happens when git history is shallow (e.g., `--depth 1` clone)?
  - Gracefully degrade: use available history, mark resources as "history unavailable" when git blame fails
- How does the system handle large graphs (100+ resources)?
  - Paginate CLI output, provide `--filter` options, optimize JSON/Mermaid output for size
- What happens when Bicep uses runtime expressions that can't be statically resolved?
  - Mark affected values as "dynamic" in the graph, use placeholder notation
- What happens when Bicep files use cloud-specific resources (Azure, AWS)?
  - Graph generation MUST work regardless of cloud provider; cloud-specific resources are represented with their provider prefix (e.g., `Microsoft.Storage/storageAccounts`, `AWS::S3::Bucket`)

---

## CLI Design

This feature extends the existing `rad app graph` command with file-based input for static graph generation. The command intelligently distinguishes between deployed apps and Bicep files based on the argument:

| Command | Input Type | Behavior |
|---------|------------|----------|
| `rad app graph myapp` | App name | Show deployed app graph (existing) |
| `rad app graph myapp -e prod` | App name + environment | Show deployed graph in specific environment |
| `rad app graph app.bicep` | Bicep file (`.bicep` extension) | Generate preview graph from file |
| `rad app graph app.bicep --format mermaid` | Bicep file + format | Preview with Mermaid output |
| `rad app graph app.bicep --git-enrich` | Bicep file + git | Preview with git metadata |
| `rad app graph app.bicep --at abc123` | Bicep file + commit | Preview at specific commit |
| `rad app graph diff app.bicep --from abc123 --to def456` | Bicep file + commits | Diff between commits |
| `rad app graph history app.bicep --commits 10` | Bicep file + count | Show historical timeline |

**Design Rationale**: Unifying under `rad app graph` provides:
- Conceptual consistency: both are "app graphs" (prospective vs. deployed)
- Discoverability: all graph functionality in one place
- Intuitive disambiguation: `.bicep` extension clearly indicates file input
- Alignment with existing `rad app graph <appname>` pattern

---

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: System MUST parse Bicep files and extract resource definitions without requiring deployment
- **FR-002**: System MUST resolve module references and build a complete graph across multiple Bicep files
- **FR-003**: System MUST extract connection relationships from resource properties (connections, routes, ports)
- **FR-004**: System MUST produce deterministic output (same input = byte-identical output)
- **FR-005**: System MUST support JSON output format with stable key ordering
- **FR-006**: System MUST support Markdown output format with resource tables and ASCII diagrams
- **FR-007**: System MUST support Mermaid output format for GitHub-native rendering
- **FR-008**: System MUST enrich graph nodes with git metadata (commit SHA, author, date, message) when requested
- **FR-009**: System MUST track which Bicep file(s) define each resource for git blame integration
- **FR-010**: System MUST provide a GitHub Action that generates and posts graph diffs on PRs
- **FR-011**: System MUST update existing PR comments rather than creating duplicates
- **FR-012**: System MUST support generating graphs at specific git commits/refs
- **FR-013**: System MUST handle Bicep parameter files to resolve parameterized values
- **FR-014**: System MUST report clear errors for invalid Bicep syntax with file/line/column information
- **FR-015**: System MUST handle unresolvable references gracefully with placeholder annotations
- **FR-016**: System MUST work with Bicep files targeting any cloud provider (multi-cloud neutrality per Constitution Principle III)
- **FR-017**: System MUST be compatible with the Radius Bicep extension type definitions

### Non-Functional Requirements

- **NFR-001**: All Go code MUST follow Effective Go patterns and pass `golangci-lint` (Constitution Principle II)
- **NFR-002**: All exported Go packages, types, and functions MUST have godoc comments (Constitution Code Quality Standards)
- **NFR-003**: Feature MUST NOT require changes to existing deployment workflows (Constitution Principle IX - Incremental Adoption)
- **NFR-004**: CLI commands MUST follow existing `rad` CLI patterns and conventions
- **NFR-005**: Error messages MUST be actionable with clear guidance for resolution (Constitution Principle VI)

### Key Entities

- **AppGraph**: Root container holding all resources, connections, and metadata for a single application
  - Resources: Collection of AppGraphResource nodes
  - Metadata: Git commit info, generation timestamp, source files
  
- **AppGraphResource**: Single resource node in the graph
  - ID: Unique resource identifier (matches Radius resource ID format)
  - Name: Human-readable resource name
  - Type: Resource type (e.g., `Applications.Core/containers`)
  - SourceFile: Bicep file path where resource is defined
  - SourceLine: Line number in source file
  - Connections: Outbound connections to other resources
  - GitInfo: Last commit SHA, author, date, message affecting this resource
  
- **AppGraphConnection**: Edge between two resources
  - SourceID: Origin resource
  - TargetID: Destination resource
  - Direction: Outbound/Inbound
  - Type: Connection type (connection, route, port binding)
  
- **GraphDiff**: Comparison result between two graphs
  - AddedResources: Resources present in new but not old
  - RemovedResources: Resources present in old but not new
  - ModifiedResources: Resources with changed properties/connections
  - AddedConnections: New edges
  - RemovedConnections: Removed edges

---

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: Graph generation completes in < 5 seconds for applications with up to 50 resources
- **SC-002**: Generated graphs are 100% deterministic (identical input produces byte-identical output)
- **SC-003**: Graph diff correctly identifies all added, removed, and modified resources with zero false positives
- **SC-004**: GitHub Action posts PR comments within 60 seconds of workflow trigger
- **SC-005**: Markdown/Mermaid output renders correctly in GitHub without manual formatting
- **SC-006**: Git enrichment adds < 2 seconds overhead for repositories with up to 1000 commits
- **SC-007**: System handles Bicep files up to 5000 lines without performance degradation
- **SC-008**: Error messages include actionable guidance in 100% of failure cases

---

## Testing Requirements *(per Constitution Principle IV)*

This feature MUST include comprehensive testing across the testing pyramid:

### Unit Tests
- Test individual graph parsing functions in isolation
- Test git metadata extraction logic
- Test output formatters (JSON, Markdown, Mermaid) with known inputs
- Test error handling for malformed Bicep files
- All unit tests runnable with `make test` without external dependencies

### Integration Tests
- Test Bicep CLI integration for file parsing
- Test git operations (blame, log) with real git repositories
- Test module resolution across multiple Bicep files

### Functional Tests
- End-to-end test: Bicep file → graph generation → output validation
- Test GitHub Action in a real PR workflow
- Test graph diff accuracy with known before/after states

---

## Open Questions

1. **Bicep Compiler Integration**: Should we use the official Bicep CLI for parsing, or implement a lightweight parser? Trade-off: accuracy vs. dependency management. **Recommendation**: Use official Bicep CLI per Constitution Principle VII (Simplicity Over Cleverness).

2. **Graph Storage**: Should generated graphs be committed to the repo (e.g., `app-graph.json`)? Trade-off: visibility vs. repo noise.

3. **GitHub App vs Action**: Should the PR integration be a GitHub Action (user-managed) or a GitHub App (centrally managed)? Trade-off: flexibility vs. ease of setup.

4. **Diff Visualization**: What's the preferred format for showing diffs in PR comments—table-based, Mermaid side-by-side, or unified text diff?

5. **Parameter Handling**: How should we handle Bicep parameters without a params file—use defaults, require params, or mark as "unknown"?

---

## Cross-Repository Impact *(per Constitution Principle XVII)*

This feature may affect multiple Radius repositories:

| Repository | Impact |
|------------|--------|
| `radius` | CLI implementation (`pkg/cli/cmd/app/graph/`), core graph logic |
| `docs` | User documentation for new CLI commands, GitHub Action setup guide |
| `design-notes` | This specification and implementation plan |

Coordinate changes across repositories per Constitution guidance on polyglot project coherence.