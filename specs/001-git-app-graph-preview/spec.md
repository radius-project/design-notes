# Feature Specification: Git App Graph Preview

**Feature Branch**: `001-git-app-graph-preview`   
**Created**: January 30, 2026   
**Status**: Draft   
**Input**: User description: "Radius currently stores the state of application deployments as an app graph within its data store. Today, the app graph does not get generated until the application is deployed. Help me build an app graph representation for applications that are defined (e.g. in an app.bicep file) but not yet deployed. Additionally, enrich the app graph representation with git changelog info (i.e. git commit data) so that I may use this data to visualize how the app graph changes over time (i.e. across commits). The ultimate goal is to be able to visualize the app graph and do diffs of the app graph in GitHub on PRs, commit comparisons, etc."   

## Clarifications

### Session 2026-02-04

- Q: GitHub App vs Action for PR integration? → A: GitHub Action (fork-friendly, no installation approval required, aligns with existing Radius workflow patterns)
- Q: Diff visualization format in PR comments? → A: Table + Mermaid diagrams (change table for details, before/after diagrams for visual topology)
- Q: How to handle Bicep parameters without params file? → A: Require params file (fail with error if Bicep has required parameters but no `--parameters` provided)
- Q: GitHub Action trigger events? → A: `pull_request` + `push` (PR for review comments, push to main for baseline tracking)
- Q: Monorepo support with multiple app graphs? → A: Auto-detect all Bicep entry points (e.g., `**/app.bicep`); each diffed independently

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

1. **Given** a valid `app.bicep` file with container, gateway, and database resources, **When** I run `rad app graph app.bicep`, **Then** I receive a JSON graph representation showing all resources, their connections, and key properties for each resource (e.g., container image, ports, database name).

2. **Given** a Bicep file with syntax errors, **When** I run `rad app graph app.bicep`, **Then** I receive a clear error message indicating the parsing failure with line/column information.

3. **Given** a Bicep file referencing external modules, **When** I run `rad app graph app.bicep`, **Then** the graph includes resources from all referenced modules with proper dependency tracking.

4. **Given** a Bicep file with parameterized values, **When** I run `rad app graph app.bicep --parameters params.json`, **Then** the graph reflects the resolved parameter values.

5. **Given** a Bicep file with required parameters (no defaults) and no `--parameters` flag, **When** I run `rad app graph app.bicep`, **Then** I receive a clear error listing the missing required parameters.

6. **Given** a Bicep file using the Radius Bicep extension types, **When** I run `rad app graph app.bicep`, **Then** the graph correctly identifies Radius-specific resource types and their relationships.

---

### User Story 2 - Export Graph as Diff-Friendly Format (Priority: P1)

As a developer, I want the app graph exported in a deterministic, diff-friendly format, so I can commit it to version control and see meaningful diffs when it changes.

**Why this priority**: Critical for enabling GitHub integration. Without a stable, diffable format, PR visualization is impossible.

**Independent Test**: Generate graph twice from identical Bicep, verify outputs are byte-identical. Modify Bicep, regenerate, verify diff shows only the changed elements.

**Output Model**: JSON is the canonical data format, always generated. Markdown is a rendered preview of the JSON data, generated additively when requested.

**Acceptance Scenarios**:

1. **Given** an app graph, **When** I export it, **Then** the JSON output is deterministically sorted (alphabetical by resource ID) producing identical output for identical inputs.

2. **Given** an app graph, **When** I run `rad app graph app.bicep`, **Then** JSON is written to `.radius/app-graph.json` (default location) serving as the single source of truth for all automation and diff operations.

3. **Given** an app graph, **When** I run `rad app graph app.bicep --stdout`, **Then** JSON is written to stdout instead of a file.

4. **Given** an app graph, **When** I run `rad app graph app.bicep --format markdown`, **Then** I receive **both** `.radius/app-graph.json` and `.radius/app-graph.md` containing:
   - A resource table with name, type, source file, and git metadata
   - An embedded Mermaid diagram showing the topology that GitHub renders automatically

5. **Given** a graph exported to markdown, **When** viewed in GitHub, **Then** the Mermaid diagram renders as a visual flowchart following the Visual Design Guidelines (rounded rectangle nodes, GitHub-consistent color palette for diff states, distinct shapes for databases) and node labels include key properties (e.g., container image tag, port numbers).

6. **Given** two app graphs from different commits, **When** I diff them, **Then** the diff is computed from JSON (not Markdown), and added/removed/modified resources are clearly identified.

---

### User Story 3 - Git Metadata Enrichment (Priority: P2)

As a developer, I want the app graph to automatically include git commit information, so I can track when and why each resource was added or modified.

**Why this priority**: Builds on P1 capabilities to enable historical tracking. Valuable but not blocking core functionality.

**Independent Test**: Generate graph from a Bicep file in a git repository, verify each resource includes commit SHA, author, and timestamp of last modification by default.

**Acceptance Scenarios**:

1. **Given** a Bicep file in a git repository, **When** I run `rad app graph app.bicep`, **Then** each resource automatically includes the commit SHA, author, date, and message of its last modification.

2. **Given** a resource defined across multiple Bicep files, **When** I generate the graph, **Then** the resource shows the most recent commit that affected any of its defining files.

3. **Given** a newly added resource not yet committed, **When** I generate the graph, **Then** the resource is marked as "uncommitted" with the current working directory state.

4. **Given** a graph with git metadata, **When** I export to markdown, **Then** each resource row includes a linked commit SHA (e.g., `[abc123](../../commit/abc123)`).

5. **Given** a Bicep file in a git repository, **When** I run `rad app graph app.bicep --no-git`, **Then** the graph is generated without git metadata for faster execution.

6. **Given** a Bicep file outside a git repository, **When** I run `rad app graph app.bicep`, **Then** the graph is generated successfully with git fields marked as "not available".

---

### User Story 4 - GitHub Action for PR Graph Diff (Priority: P2)

As a PR reviewer, I want to see a visual diff of the app graph in PR comments, so I can understand the architectural impact of code changes without deploying.

**Why this priority**: High-value GitHub integration, but depends on P1 capabilities being stable.

**Operational Model**: The GitHub Action **generates the app graph automatically** as part of the workflow — developers do NOT need to run `rad app graph app.bicep` manually or commit graph artifacts. The Action checks out the PR head and base branches, runs `rad app graph app.bicep` on each, computes the diff, and posts the result as a PR comment. This keeps the developer experience friction-free at the cost of requiring Bicep/Radius tooling in CI.

**Trigger Events**: The Action supports two trigger modes:
- **`pull_request`**: Posts diff comments on PRs when `.radius/app-graph.json` changes
- **`push` to main/default branch**: Updates baseline tracking for historical comparison

**Monorepo Support**: The Action auto-detects all Bicep entry points (e.g., `**/app.bicep`) in the repository. Each application graph is generated and diffed independently, with separate PR comment sections per application.

**Independent Test**: Create a PR with Bicep changes and updated graph JSON, verify the action posts a comment showing before/after graph comparison.

**Acceptance Scenarios**:

1. **Given** a PR that includes changes to Bicep files, **When** the GitHub Action runs, **Then** it generates the app graph from both the base and head commits, computes the diff, and posts a comment showing added (green), removed (red), and modified (yellow) resources using the Visual Design Guidelines color conventions.

2. **Given** a PR with no changes to Bicep files (or where Bicep changes produce an identical graph), **When** the GitHub Action runs, **Then** it posts a comment indicating "No app graph changes detected."

3. **Given** a PR that adds a new connection between resources, **When** the GitHub Action runs, **Then** the diff clearly shows the new connection with source and target resources.

4. **Given** a PR comment already exists from a previous run, **When** the PR is updated and the action runs again, **Then** the existing comment is updated rather than creating a duplicate.

5. **Given** a developer who has not run `rad app graph` locally, **When** they push Bicep changes and open a PR, **Then** the GitHub Action generates the graph and posts the diff automatically — no manual graph generation or commit is required.

6. **Given** a monorepo with multiple Radius applications (e.g., `apps/frontend/app.bicep` and `apps/backend/app.bicep`), **When** the GitHub Action runs on a PR, **Then** it detects all Bicep entry points and posts a unified comment with separate diff sections per application.

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

### User Story 6 - Environment-Resolved Graph (Priority: P3)

As a platform engineer, I want to see how abstract Radius types resolve to concrete infrastructure in a specific environment, so I can understand the actual resources that will be deployed.

**Why this priority**: Advanced feature for environment-specific analysis. The static graph (showing portable types) serves most PR review needs; resolved views are valuable for deployment planning and troubleshooting.

**Background**: Radius portable types like `Radius.Data/store` resolve differently depending on the environment's recipe configuration:
- Environment → RecipePack → Recipe → Concrete Resource
- The same `Radius.Data/store` might become PostgreSQL in `dev` and CosmosDB in `prod`

**Independent Test**: Generate resolved graph for an environment with known recipe bindings, verify concrete resource types appear instead of abstract Radius types.

**Acceptance Scenarios**:

1. **Given** a Bicep file with `Radius.Data/store` and a connected Radius environment with PostgreSQL recipes, **When** I run `rad app graph app.bicep --environment prod`, **Then** the graph shows the resolved `Azure.DBforPostgreSQL/flexibleServers` (or equivalent) instead of the abstract `Radius.Data/store`.

2. **Given** a Bicep file with portable types, **When** I run `rad app graph app.bicep --environment dev` and `rad app graph app.bicep --environment prod`, **Then** I can compare how the same application resolves to different infrastructure across environments.

3. **Given** an environment where a recipe is not configured for a portable type, **When** I run `rad app graph app.bicep --environment prod`, **Then** the graph shows the abstract type with an annotation indicating "no recipe bound".

4. **Given** a Bicep file, **When** I run `rad app graph app.bicep` (no `--environment` flag), **Then** the graph shows the abstract portable types (default behavior unchanged).

---

### Edge Cases

- What happens when Bicep file references resources outside the current file/module that cannot be resolved?
  - Generate partial graph with unresolved references marked as "external" placeholders
- How does the system handle circular dependencies in Bicep?
  - Detect and report cycles with clear error messaging; still generate graph with cycle annotation
- What happens when git history is shallow (e.g., `--depth 1` clone)?
  - Gracefully degrade: use available history, mark resources as "history unavailable" when git blame fails
- How does the system handle large graphs (100+ resources)?
  - Paginate CLI output, provide `--filter` options, optimize JSON/Markdown output for size
- What happens when Bicep uses runtime expressions that can't be statically resolved?
  - Mark affected values as "dynamic" in the graph, use placeholder notation
- What happens when Bicep files use cloud-specific resources (Azure, AWS)?
  - Graph generation MUST work regardless of cloud provider; cloud-specific resources are represented with their provider prefix (e.g., `Microsoft.Storage/storageAccounts`, `AWS::S3::Bucket`)
- What happens if a developer commits `.radius/app-graph.json` alongside Bicep changes?
  - The GitHub Action ignores committed graph artifacts and always generates fresh graphs from Bicep source to ensure accuracy
  - Committed graph files are treated as optional local convenience, not as CI inputs
- What does the graph show for portable Radius types like `Radius.Data/store` that resolve differently per environment?
  - **Static graph shows abstract types**: The declared `Radius.Data/store` is shown, not the resolved infrastructure (PostgreSQL, CosmosDB, etc.)
  - This is intentional—the static graph represents the **portable application architecture** independent of environment-specific recipe resolution
  - For environment-resolved views, see User Story 6 (P3)

---

## CLI Design

This feature extends the existing `rad app graph` command with file-based input for static graph generation. The command intelligently distinguishes between deployed apps and Bicep files based on the argument:

| Command | Input Type | Output |
|---------|------------|--------|
| `rad app graph myapp` | App name | Deployed app graph (existing behavior) |
| `rad app graph myapp -e prod` | App name + environment | Deployed graph in specific environment |
| `rad app graph app.bicep` | Bicep file (`.bicep` extension) | JSON to `.radius/app-graph.json` (default) |
| `rad app graph app.bicep --stdout` | Bicep file + stdout flag | JSON to stdout (no file written) |
| `rad app graph app.bicep -o custom.json` | Bicep file + custom output | JSON to specified file |
| `rad app graph app.bicep --format markdown` | Bicep file + markdown | JSON + Markdown to `.radius/` |
| `rad app graph app.bicep --no-git` | Bicep file + no-git | JSON without git metadata (faster) |
| `rad app graph app.bicep --at abc123` | Bicep file + commit | JSON at specific commit |
| `rad app graph diff app.bicep --from abc123 --to def456` | Bicep file + commits | Diff computed from JSON, output as JSON or Markdown |
| `rad app graph history app.bicep --commits 10` | Bicep file + count | Historical timeline |
| `rad app graph app.bicep --environment prod` | Bicep file + environment | JSON with resolved recipe types |

**Output Model**:
- **JSON is canonical**: Always generated, serves as the single source of truth for all automation and diff operations
- **Markdown is additive**: When `--format markdown` is specified, Markdown is generated *in addition to* JSON as a human-readable preview
- **GitHub Action uses JSON**: Diff computation is always JSON-to-JSON; Markdown is purely a rendering/presentation layer for PR comments

**Design Rationale**: Unifying under `rad app graph` provides:
- Conceptual consistency: both are "app graphs" (prospective vs. deployed)
- Discoverability: all graph functionality in one place
- Intuitive disambiguation: `.bicep` extension clearly indicates file input
- Alignment with existing `rad app graph <appname>` pattern

---

## Visual Design Guidelines

All graph visualizations — Mermaid diagrams, PR comment diff tables, and Markdown output — MUST follow a visual style consistent with the **native GitHub UI** to feel integrated rather than bolted-on.

### Color Palette (Diff States)

| State | Color | Usage | GitHub Reference |
|-------|-------|-------|------------------|
| **Added** | Green (`#2da44e` / `--color-success-fg`) | New resources, new connections | Same as GitHub diff additions |
| **Removed** | Red (`#cf222e` / `--color-danger-fg`) | Deleted resources, deleted connections | Same as GitHub diff deletions |
| **Modified** | Yellow (`#bf8700` / `--color-attention-fg`) | Changed properties or connections | Same as GitHub warning/attention |
| **Unchanged** | Gray (`#656d76` / `--color-fg-muted`) | Context resources shown for topology reference | Same as GitHub muted text |

### Mermaid Diagram Styling

- **Node shapes**: Rounded rectangles (`([...])`) for all resource nodes to match GitHub's card/box aesthetic
  - Containers: rounded rectangle with container icon annotation
  - Gateways: rounded rectangle with gateway/diamond icon annotation
  - Databases: cylinder shape (Mermaid built-in `[(...)]`)
- **Node labels**: Resource name as primary label, key properties (e.g., image tag, port) as secondary text
- **Edges**: Solid lines for direct connections, dashed lines for implicit/inferred dependencies
- **Diff coloring**: Use Mermaid `classDef` styles to apply the color palette above:
  ```mermaid
  classDef added fill:#dafbe1,stroke:#2da44e,color:#1a7f37
  classDef removed fill:#ffebe9,stroke:#cf222e,color:#82071e
  classDef modified fill:#fff8c5,stroke:#bf8700,color:#6a5600
  classDef unchanged fill:#f6f8fa,stroke:#d0d7de,color:#656d76
  ```
- **Layout**: Top-to-bottom (`TB`) flow direction for application topology

### PR Comment Styling

- **Change summary table**: Use GitHub Markdown table with status emoji prefix for scanability:
  - `🟢 Added` — green circle for new resources
  - `🔴 Removed` — red circle for deleted resources
  - `🟡 Modified` — yellow circle for changed resources
- **Before/After diagrams**: Labeled with `### Before` and `### After` headings inside a collapsible `<details>` block to avoid overwhelming long PRs
- **Comment header**: Include a brief summary line (e.g., "App Graph: +2 resources, -1 resource, ~1 modified") so reviewers can assess impact at a glance without expanding details
- **Borders and containers**: Use GitHub's native `> [!NOTE]`, `> [!WARNING]`, and `> [!IMPORTANT]` callout syntax where appropriate for status messages

### General Principles

- Visual output MUST render correctly in both GitHub light and dark themes
- Avoid hardcoded colors that become illegible in dark mode — use GitHub's CSS variable equivalents as reference and test both themes
- Prefer semantic color usage (green = added, red = removed) over decorative color
- Keep visualizations compact — prioritize information density over whitespace
- All visual elements should feel like a native part of the GitHub PR review experience, not an external tool's output

---

The app graph JSON can be generated locally as a **developer preview** but is NOT required to be committed. The GitHub Action generates graphs automatically from Bicep source during CI.

### Default Output Location

Locally, `rad app graph app.bicep` writes to `.radius/app-graph.json` relative to the Bicep file's directory:

```
myapp/
├── app.bicep
├── modules/
│   └── database.bicep
└── .radius/
    ├── app-graph.json      # Canonical graph data (committed)
    └── app-graph.md        # Optional preview (if --format markdown)
```

### Developer Workflow

```bash
# 1. Make changes to Bicep files
vim app.bicep

# 2. (Optional) Preview the graph locally
rad app graph app.bicep            # writes to .radius/app-graph.json
rad app graph app.bicep --stdout    # or view in terminal

# 3. Commit Bicep changes (no need to commit graph artifacts)
git add app.bicep
git commit -m "Add redis cache to application"

# 4. Push and create PR — GitHub Action generates graphs and posts diff automatically
git push
```

**Key Point**: Developers do NOT need to run `rad app graph` or commit `.radius/app-graph.json` for the PR workflow to function. The GitHub Action handles graph generation and diffing entirely. Local graph generation is available as an optional preview tool.

### Why This Model?

| Benefit | Explanation |
|---------|-------------|
| **Zero developer friction** | Developers only commit Bicep files — no extra `rad app graph` step or graph artifacts to manage |
| **Always up-to-date** | Graph is generated fresh from Bicep source on every PR, eliminating stale graph drift |
| **Reproducible** | Graph generated from committed Bicep source using pinned tooling in CI |
| **Fork-friendly** | Works in forks — the GitHub Action installs its own tooling |
| **Optional local preview** | Developers can still run `rad app graph app.bicep` locally to preview changes before pushing |

### Graph Metadata

Generated graphs include metadata for traceability and debugging:

```json
{
  "metadata": {
    "generatedAt": "2026-01-30T10:15:00Z",
    "sourceFiles": ["app.bicep", "modules/database.bicep"],
    "sourceHash": "sha256:abc123...",
    "radiusCliVersion": "0.35.0",
    "generatedBy": "github-action"  
  },
  "resources": [...],
  "connections": [...]
}
```

Since the GitHub Action generates graphs on-the-fly from Bicep source, there is no staleness concern — the graph always reflects the current state of the code at each commit.

---

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: System MUST parse Bicep files and extract resource definitions without requiring deployment
- **FR-002**: System MUST resolve module references and build a complete graph across multiple Bicep files
- **FR-003**: System MUST extract connection relationships from resource properties (connections, routes, ports)
- **FR-004**: System MUST produce deterministic output (same input = byte-identical output)
- **FR-005**: System MUST always generate JSON output as the canonical data format with stable key ordering for deterministic diffs
- **FR-006**: System MUST support Markdown output as an **additive** preview format (generated alongside JSON when `--format markdown` is specified), containing a resource table and embedded Mermaid diagram
- **FR-007**: System MUST perform all diff computations using JSON data, with Markdown used only as a rendering layer for human consumption
- **FR-008**: System MUST enrich graph nodes with git metadata (commit SHA, author, date, message) by default when in a git repository, with `--no-git` flag to disable
- **FR-009**: System MUST track which Bicep file(s) define each resource for git blame integration
- **FR-010**: System MUST write graph output to `.radius/app-graph.json` by default (relative to Bicep file location), with `--stdout` flag for stdout output and `-o` flag for custom path
- **FR-011**: System MUST include `sourceHash` metadata in JSON output for traceability
- **FR-012**: System MUST provide a GitHub Action that generates app graphs from Bicep source for both base and head commits, computes the diff, and posts the result as a PR comment (developers are NOT required to generate or commit graph artifacts manually)
- **FR-013**: System MUST update existing PR comments rather than creating duplicates
- **FR-014**: System MUST support generating graphs at specific git commits/refs
- **FR-015**: System MUST handle Bicep parameter files to resolve parameterized values
- **FR-016**: System MUST report clear errors for invalid Bicep syntax with file/line/column information
- **FR-017**: System MUST handle unresolvable references gracefully with placeholder annotations
- **FR-018**: System MUST work with Bicep files targeting any cloud provider (multi-cloud neutrality per Constitution Principle III)
- **FR-019**: System MUST be compatible with the Radius Bicep extension type definitions
- **FR-020**: System MUST extract and include resource-specific properties (e.g., container image, ports, hostnames, database names) in each graph node so that visualizations can display them as node labels or detail annotations. Secret values MUST NOT be captured.

### Non-Functional Requirements

- **NFR-001**: All Go code MUST follow Effective Go patterns and pass `golangci-lint` (Constitution Principle II)
- **NFR-002**: All exported Go packages, types, and functions MUST have godoc comments (Constitution Code Quality Standards)
- **NFR-003**: Feature MUST NOT require changes to existing deployment workflows (Constitution Principle IX - Incremental Adoption)
- **NFR-004**: CLI commands MUST follow existing `rad` CLI patterns and conventions
- **NFR-005**: Error messages MUST be actionable with clear guidance for resolution (Constitution Principle VI)
- **NFR-006**: All visual output (Mermaid diagrams, PR comment tables, Markdown previews) MUST follow a GitHub-native visual style — rounded corners, GitHub's diff color palette (green/red/yellow), and correct rendering in both light and dark themes

### Key Entities

- **AppGraph**: Root container holding all resources, connections, and metadata for a single application
  - Resources: Collection of AppGraphResource nodes
  - Metadata: Git commit info, generation timestamp, source files
  
- **AppGraphResource**: Single resource node in the graph
  - ID: Unique resource identifier (matches Radius resource ID format)
  - Name: Human-readable resource name
  - Type: Resource type (e.g., `Applications.Core/containers`)
  - Properties: Key resource-specific properties useful for visualization (see below)
  - SourceFile: Bicep file path where resource is defined
  - SourceLine: Line number in source file
  - Connections: Outbound connections to other resources
  - GitInfo: Last commit SHA, author, date, message affecting this resource
  
  **Resource Properties for Visualization**: Each resource node MUST include salient properties extracted from the Bicep definition so that visualizations can display them as node labels or detail panels. The properties captured are resource-type-specific:
  
  | Resource Type | Properties Captured |
  |---------------|--------------------|
  | `Applications.Core/containers` | `image` (container image reference), `ports` (port mappings), `env` (environment variable names, not values) |
  | `Applications.Core/gateways` | `hostname`, `routes` (path prefixes) |
  | `Applications.Core/httpRoutes` | `port`, `scheme` |
  | `Applications.Datastores/*` | `resourceProvisioning`, `database` (name), `server` (host) |
  | `Applications.Messaging/*` | `resourceProvisioning`, `queue`/`topic` (name) |
  | Cloud-specific resources | `sku`, `location`, provider-specific key properties |
  | All resources | `application` (parent app ID), `environment` (environment ID) |
  
  Properties with values that cannot be statically resolved (e.g., runtime expressions, secrets) are included with a `"dynamic"` placeholder value. Secret values (e.g., passwords, connection strings) are NEVER captured — only non-sensitive configuration is included.
  
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
- **SC-005**: Markdown output (including embedded Mermaid diagram) renders correctly in GitHub in both light and dark themes, following the Visual Design Guidelines for color and layout
- **SC-006**: Git enrichment adds < 2 seconds overhead for repositories with up to 1000 commits
- **SC-007**: System handles Bicep files up to 5000 lines without performance degradation
- **SC-008**: Error messages include actionable guidance in 100% of failure cases

---

## Testing Requirements *(per Constitution Principle IV)*

This feature MUST include comprehensive testing across the testing pyramid:

### Unit Tests
- Test individual graph parsing functions in isolation
- Test git metadata extraction logic
- Test output formatters (JSON, Markdown with embedded Mermaid) with known inputs
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

2. ~~**Graph Storage**: Should generated graphs be committed to the repo (e.g., `app-graph.json`)? Trade-off: visibility vs. repo noise.~~ **RESOLVED (Initial Implementation)**: Graphs are committed to `.radius/app-graph.json`. This enables lightweight GitHub Action (reads from git history, no tooling required) and provides auditable graph evolution. **Future Evolution**: External storage backends (e.g., SQLite, cloud databases) could be supported for scenarios requiring graph queries across repositories, historical analytics, or enterprise-scale graph management.

3. ~~**GitHub App vs Action**: Should the PR integration be a GitHub Action (user-managed) or a GitHub App (centrally managed)? Trade-off: flexibility vs. ease of setup.~~ **RESOLVED**: GitHub Action. Fork-friendly, no installation approval required, aligns with existing Radius workflow patterns, supports incremental adoption.

4. ~~**Diff Visualization**: What's the preferred format for showing diffs in PR comments—table-based, Mermaid side-by-side, or unified text diff?~~ **RESOLVED**: Table + Mermaid diagrams. Change table shows added/removed/modified resources with details; before/after Mermaid diagrams provide visual topology comparison. Both render natively in GitHub.

5. ~~**Parameter Handling**: How should we handle Bicep parameters without a params file—use defaults, require params, or mark as "unknown"?~~ **RESOLVED**: Require params file. If Bicep has required parameters (no defaults) but no `--parameters` flag is provided, fail with a clear error message listing the missing parameters.

---

## Cross-Repository Impact *(per Constitution Principle XVII)*

This feature may affect multiple Radius repositories:

| Repository | Impact |
|------------|--------|
| `radius` | CLI implementation (`pkg/cli/cmd/app/graph/`), core graph logic |
| `docs` | User documentation for new CLI commands, GitHub Action setup guide |
| `design-notes` | This specification and implementation plan |

Coordinate changes across repositories per Constitution guidance on polyglot project coherence.