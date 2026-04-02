# Automated Default Registration of Resource Types from resource-types-contrib

* **Author**: Karishma Chawla (@karishmachawla)

## Overview

Today, resource type manifests for default registration in Radius are manually duplicated from the `resource-types-contrib` repository into the `radius` repository under `deploy/manifest/built-in-providers/`. This creates a maintenance burden: when a resource type schema is updated in `resource-types-contrib`, the corresponding file in `radius` must be manually updated, leading to schema drift, stale definitions, and duplicated effort.

This design introduces a mechanism to automatically embed resource type manifests from `resource-types-contrib` as a Go module dependency of `radius`. A central configuration file (`defaults.yaml`) in `resource-types-contrib` declares which resource types should be default-registered. At build time, only those manifests are embedded into the Radius binary via `go:embed`. At startup, the UCP initializer reads the embedded manifests and registers them alongside any existing directory-based manifests.

This eliminates the need to copy files between repositories, ensures schemas stay in sync via standard Go dependency management, and provides a clear, reviewable way to control which resource types ship as defaults.

## Terms and definitions

| Term | Definition |
|---|---|
| **Resource type manifest** | A YAML file defining the namespace, types, API versions, and schemas for a resource provider (e.g., `Radius.Compute/containers`). |
| **Default registration** | The process of registering resource types into UCP at Radius startup so they are available out of the box without user action. |
| **`resource-types-contrib`** | The community repository containing resource type definitions and recipes (`github.com/radius-project/resource-types-contrib`). |
| **UCP** | Universal Control Plane, the Radius component responsible for routing and managing resource providers. |
| **dynamic-rp** | The dynamic resource provider in Radius that handles UDT-based resource types. |
| **`DefaultDownstreamEndpoint`** | A UCP routing config that provides a fallback endpoint when a resource provider location has no explicit address. Points to dynamic-rp. |

## Objectives

> **Issue Reference:** <!-- Add issue link when available -->

### Goals

1. **Eliminate schema drift**: Resource type schemas defined in `resource-types-contrib` should be the single source of truth. Radius should consume them directly rather than maintaining copies.
2. **Controlled default registration**: Provide a clear, centralized mechanism to declare which resource types from `resource-types-contrib` are registered by default in Radius.
3. **Minimal binary bloat**: Only embed the manifests needed for default registration, not the entire `resource-types-contrib` repository (which includes recipes, tests, and documentation).
4. **Simple contribution workflow**: Adding a new default resource type should require editing a single configuration file, with no Go code changes.
5. **Version-pinned updates**: Schema updates are applied to Radius via standard Go dependency management (`go get -u`), providing version pinning, audit trails, and compatibility with Dependabot.

### Non goals

- **Automatic sync without review**: We intentionally require a `go.mod` version bump in `radius` to pick up changes. Fully automated, unreviewed sync is out of scope.
- **Runtime fetching of manifests**: Manifests are embedded at build time, not downloaded at runtime. This avoids network dependencies during startup.
- **Migrating non-dynamic-rp providers**: Resource types served by `applications-rp` or the deployment engine (e.g., `Applications.Core`, `Microsoft.Resources`) require explicit `location` addresses and remain as directory-based manifests in `radius`. Migrating them is out of scope.
- **Recipe registration**: This design covers resource type schema registration only, not recipe registration or recipe pack management.

### User scenarios (optional)

#### Platform engineer adds a new default resource type

A platform engineer creates a new resource type `Radius.Networking/loadBalancers` in `resource-types-contrib`. To make it a default in Radius:

1. They add the YAML manifest at `Networking/loadBalancers/loadBalancers.yaml`.
2. They add the path to `defaults.yaml`:
   ```yaml
   defaultRegistration:
     - Networking/loadBalancers/loadBalancers.yaml
   ```
3. They run `go generate` and commit `defaults.yaml`, `manifests_gen.go`.
4. After the `resource-types-contrib` release, a Radius maintainer runs `go get -u github.com/radius-project/resource-types-contrib` to pick up the new version.

#### Platform engineer updates a resource type schema

A platform engineer updates the schema for `Radius.Compute/containers` in `resource-types-contrib`. The change flows to Radius when a maintainer bumps the dependency version in `go.mod`. No file copying or sync scripts are needed.

## User Experience (if applicable)

N/A. This change is transparent to end users. Resource types continue to be available at startup as they are today. The change is to the internal mechanism by which they are loaded.

## Design

### High Level Design

The design introduces `resource-types-contrib` as a Go module dependency of `radius`. Resource type manifests are embedded into the Radius binary using Go's `embed.FS` mechanism. A central `defaults.yaml` file in `resource-types-contrib` lists which manifests should be embedded and registered by default.

At startup, the UCP initializer service:
1. Reads `defaults.yaml` from the embedded filesystem to discover which manifests to load.
2. Parses each listed manifest, validates its schema, and merges manifests sharing a namespace into a single resource provider.
3. Registers the merged resource providers with UCP.
4. Proceeds to register any additional directory-based manifests as before.

The `location` field is intentionally omitted from `resource-types-contrib` manifests. When a manifest has no `location`, UCP's existing fallback mechanism routes requests to `DefaultDownstreamEndpoint` (dynamic-rp), which is the correct handler for all UDT-based resource types.

### Architecture Diagram

```
┌─────────────────────────────────────────────────────────────┐
│  resource-types-contrib (Go module)                         │
│                                                             │
│  defaults.yaml ─── lists paths ──► go generate              │
│                                        │                    │
│  Compute/containers/containers.yaml    ▼                    │
│  Compute/routes/routes.yaml       manifests_gen.go          │
│  Security/secrets/secrets.yaml    (//go:embed directives)   │
│  ...                                   │                    │
│                                        ▼                    │
│                              embed.FS DefaultManifests      │
└─────────────────────┬───────────────────────────────────────┘
                      │  Go module dependency
                      ▼
┌─────────────────────────────────────────────────────────────┐
│  radius                                                     │
│                                                             │
│  go.mod ──► github.com/radius-project/resource-types-contrib│
│                                                             │
│  pkg/ucp/server/server.go                                   │
│    └─► initializer.NewService(options, DefaultManifests)     │
│                                                             │
│  pkg/ucp/initializer/service.go                             │
│    └─► Run():                                               │
│         1. manifest.RegisterFS(embeddedManifests)            │
│         2. manifest.RegisterDirectory(manifestDir)           │
│                                                             │
│  pkg/cli/manifest/registermanifest.go                       │
│    └─► RegisterFS():                                        │
│         - Read defaults.yaml for manifest paths             │
│         - Parse & validate each manifest                    │
│         - Merge by namespace                                │
│         - Register with UCP                                 │
│                                                             │
│  deploy/manifest/built-in-providers/                        │
│    └─► radius_compute.yaml (REMOVED, now embedded)          │
│    └─► radius_security.yaml (REMOVED, now embedded)         │
└─────────────────────────────────────────────────────────────┘
```

### Detailed Design

#### Option 1: Per-file annotation (`defaultRegistration: true` in YAML)

Add a `defaultRegistration` boolean field to each manifest YAML and to the `ResourceProvider` Go struct. Embed all manifest YAMLs (`*/*/*.yaml`) and filter at runtime.

##### Advantages

- Flag is co-located with the type it describes; self-documenting.
- No central file to maintain.

##### Disadvantages

- **Struct pollution**: Adds a deployment concern (`defaultRegistration`) to the `ResourceProvider` data model, which is used by the CLI, API, and tests. It leaks a distribution detail into the schema.
- **Parser coupling**: The Radius YAML parser uses `yaml.Strict()` mode, so the `ResourceProvider` struct must include the field or parsing fails. This couples the contrib repo's metadata to the Radius parser.
- **Binary bloat**: Embedding all YAMLs (`*/*/*.yaml`) includes every resource type in the binary, even though only a handful are defaults. As `resource-types-contrib` grows to dozens or hundreds of types, this wastes binary space.
- **Discoverability**: Requires grepping across many files to determine which types are defaults.
- **Accidental removal**: The flag could be silently dropped during a type refactor.

#### Option 2: Central `defaults.yaml` + `go generate` (Proposed)

A `defaults.yaml` file at the repo root lists which manifest paths should be default-registered. A `go generate` script reads this file and produces `manifests_gen.go` with `//go:embed` directives for exactly those files (plus `defaults.yaml` itself). At runtime, `RegisterFS` reads `defaults.yaml` from the embedded FS to know which paths to load.

##### Advantages

- **Clean separation of concerns**: The `ResourceProvider` struct is not polluted with deployment metadata.
- **Minimal binary size**: Only the listed manifests are embedded.
- **Discoverability**: A single file shows all defaults at a glance.
- **Reviewability**: PR diffs for `defaults.yaml` clearly show what's being added or removed.
- **No parser coupling**: `resource-types-contrib` metadata stays out of the Radius manifest parser.
- **Extensible**: Works for any directory structure; new top-level directories (e.g., `Networking/`) work without changing Go code.

##### Disadvantages

- Requires running `go generate` after editing `defaults.yaml` (mitigated by CI validation).
- Paths in `defaults.yaml` can go stale if files are renamed (mitigated by `go generate` failing on missing files).
- Two-file commit requirement (`defaults.yaml` + `manifests_gen.go`).

#### Proposed Option

**Option 2: Central `defaults.yaml` + `go generate`**, for the reasons described above.

### API design (if applicable)

N/A. No changes to REST APIs or CLI commands.

### CLI Design (if applicable)

N/A

### Implementation Details

#### resource-types-contrib repository

| File | Purpose |
|---|---|
| `go.mod` | Makes the repository a Go module (`github.com/radius-project/resource-types-contrib`). |
| `defaults.yaml` | Central list of manifest paths for default registration. |
| `gen_embed.go` | `go generate` script that reads `defaults.yaml` and produces `manifests_gen.go`. Build-tagged `//go:build ignore`. |
| `manifests.go` | Contains `//go:generate go run gen_embed.go` directive and package documentation. |
| `manifests_gen.go` | **Generated**. Contains `//go:embed` directives for `defaults.yaml` and each listed manifest. Exports `DefaultManifests embed.FS`. |

**`defaults.yaml` format:**
```yaml
defaultRegistration:
  - Compute/containers/containers.yaml
  - Compute/persistentVolumes/persistentVolumes.yaml
  - Compute/routes/routes.yaml
  - Data/mySqlDatabases/mySqlDatabases.yaml
  - Data/postgreSqlDatabases/postgreSqlDatabases.yaml
  - Security/secrets/secrets.yaml
```

**Manifest YAML files** remain unchanged: no `location` field, no `defaultRegistration` field. They contain only `namespace` and `types`.

#### UCP

**`pkg/cli/manifest/registermanifest.go`**: New `RegisterFS` function:
- Reads `defaults.yaml` from the provided `fs.FS` to get the list of manifest paths.
- For each path, reads and parses the manifest using the existing `ReadBytes` function.
- Validates schemas using the existing `validateManifestSchemas` function.
- Merges manifests sharing a namespace (e.g., three `Radius.Compute` files) into a single `ResourceProvider` with all types under one `Types` map.
- Registers each merged provider using the existing `RegisterResourceProvider` function.

**Refactoring opportunity**: Extract shared parsing and validation logic from `RegisterFile` and `RegisterFS` into a common internal function (e.g., `registerFromBytes(ctx, data, ...) (*ResourceProvider, error)`) so both code paths share a single validation and registration pipeline. This reduces duplication and ensures consistency when either path evolves.

**`pkg/ucp/initializer/service.go`** (updated):
- `NewService` accepts an additional `fs.FS` parameter for embedded manifests.
- `Run` calls `manifest.RegisterFS` for embedded manifests **before** `manifest.RegisterDirectory` for directory-based manifests.
- If both embedded and directory manifests exist, both are registered. Directory-based manifests can override embedded ones (last-write-wins via UCP's `CreateOrUpdate`).

**`pkg/ucp/server/server.go`** (updated):
- Imports `resource-types-contrib` and passes `resourcetypes.DefaultManifests` to `initializer.NewService`.

**`deploy/manifest/built-in-providers/`** (removed files):
- `radius_compute.yaml` (now embedded from `resource-types-contrib`)
- `radius_security.yaml` (now embedded from `resource-types-contrib`)

Remaining files (`applications_core.yaml`, `applications_dapr.yaml`, `applications_datastores.yaml`, `applications_messaging.yaml`, `microsoft_resources.yaml`, `radius_core.yaml`) stay because they require explicit `location` addresses pointing to `applications-rp` or the deployment engine.

#### Location field inference

Manifests from `resource-types-contrib` intentionally omit the `location` field. The existing code in `extractLocationInfo` handles this:

1. When `Location` is nil, it defaults to `locationName = "global"` with `address = ""`.
2. `createLocationResource` skips setting `Address` when it's empty.
3. At request time, UCP's proxy controller falls back to `DefaultDownstreamEndpoint` (dynamic-rp) when a location has no address.

This is the documented, intended behavior. The config comment for `DefaultDownstreamEndpoint` states: *"DefaultDownstreamEndpoint is the default destination when a resource provider does not provide a downstream endpoint."*

This fallback is correct for all UDT-based resource types (`Radius.Compute`, `Radius.Security`, `Radius.Data`, and future namespaces) since they are all served by dynamic-rp.

#### Bicep (if applicable)

N/A

#### Deployment Engine (if applicable)

N/A

#### Core RP (if applicable)

N/A

#### Portable Resources / Recipes RP (if applicable)

N/A

### Error Handling

| Scenario | Behavior |
|---|---|
| `defaults.yaml` missing from embedded FS | `RegisterFS` returns error: `"failed to read defaults.yaml"`. Startup fails. |
| `defaults.yaml` lists a non-existent manifest path | `RegisterFS` returns error: `"failed to read manifest <path> listed in defaults.yaml"`. Startup fails. |
| Manifest YAML has invalid syntax | `ReadBytes` returns parse error. Startup fails with the specific file identified. |
| Manifest schema validation fails | `validateManifestSchemas` returns error. Startup fails with the specific file identified. |
| `defaults.yaml` is empty (no entries) | `RegisterFS` logs a message and returns nil. Startup continues with directory-based manifests only. |
| UCP not reachable at startup | Existing `waitForServer` timeout behavior. No change from current behavior. |
| 409 conflict during registration | Existing retry logic with exponential backoff. No change from current behavior. |

## Test plan

1. **Unit tests for `RegisterFS`**:
   - Test with a valid `fs.FS` containing `defaults.yaml` and matching manifests; verify correct registration calls.
   - Test namespace merging: multiple manifests with the same namespace produce a single provider with all types.
   - Test missing `defaults.yaml`: returns appropriate error.
   - Test invalid manifest YAML: returns parse error with file path.
   - Test empty `defaults.yaml`: returns nil without registering anything.
   - Test manifest path listed in `defaults.yaml` but missing from FS: returns appropriate error.

2. **Integration tests**:
   - Existing `Test_ResourceProvider_RegisterManifests` continues to work (tests directory-based registration).
   - New test that passes an `embed.FS` to `NewService` and verifies the resource provider is registered correctly.

3. **CI validation for `manifests_gen.go`**:
   - In `resource-types-contrib` CI: run `go generate` and verify no diff to ensure the generated file is up to date.
   ```bash
   go generate ./...
   git diff --exit-code manifests_gen.go
   ```

4. **Build verification**:
   - Verify Radius binary size doesn't increase significantly (only a handful of small YAML files are embedded).

## Security

No changes to the security model. The embedded manifests are static YAML files compiled into the binary at build time, so there is no new attack surface for injection or tampering beyond what exists for any compiled-in resource. The `defaults.yaml` file is validated at startup, and invalid entries cause a clear startup failure.

## Compatibility (optional)

- **No breaking changes**: The existing `ManifestDirectory` config continues to work. Directory-based manifests are registered after embedded manifests, so existing deployments that set a custom manifest directory will continue to function.
- **Removed files**: `radius_compute.yaml` and `radius_security.yaml` are removed from `deploy/manifest/built-in-providers/`. Any tooling or scripts that reference these files directly would need to be updated. The `copy-manifests` Makefile target will copy fewer files but continues to work.
- **Redundant registration**: If a deployment provides the same resource type via both embedded manifests and a directory-based manifest, the directory-based one will overwrite the embedded one (UCP uses `CreateOrUpdate`). This is harmless and provides an escape hatch.

## Monitoring and Logging

The initializer service logs at each stage:
- `"Loaded manifest <path> (namespace: <ns>)"` for each embedded manifest loaded.
- `"Registering resource provider <ns> from embedded manifests"` for each merged provider.
- `"Successfully registered default resource type manifests"` on completion of embedded registration.
- `"Successfully registered manifests" directory=<dir>` on completion of directory registration (existing).

No new metrics are added. Existing startup health checks and log monitoring apply.

## Development plan

1. **PR 1 (resource-types-contrib)**: Add `go.mod`, `defaults.yaml`, `gen_embed.go`, `manifests.go`, `manifests_gen.go`. Add CI step to validate `manifests_gen.go` is up to date.
2. **PR 2 (radius)**: Add `resource-types-contrib` to `go.mod`. Add `RegisterFS` to the manifest package. Update `initializer.Service` and `server.NewServer`. Remove `radius_compute.yaml` and `radius_security.yaml` from `built-in-providers/`. Add unit/integration tests.

### Makefile

Add a target in `resource-types-contrib` for convenience:

```make
generate-defaults:
	go generate ./...
```

In `radius`, a target to bump the dependency:

```make
update-resource-types:
	go get -u github.com/radius-project/resource-types-contrib
	go mod tidy
```

## Open Questions

1. **Should `Radius.Data` types (`mySqlDatabases`, `postgreSqlDatabases`) be included as defaults?** They exist in `resource-types-contrib` but were not previously registered as built-in providers in Radius. This design includes them in `defaults.yaml`; confirm this is intentional.

2. **Should `Radius.Core` be migrated?** `Radius.Core` (environments, applications, recipePacks) is currently defined in `radius`'s built-in providers with a `location` pointing to `applications-rp`. It is not in `resource-types-contrib`. Should it be migrated in the future, or does it belong in `radius` permanently?

3. **`go generate` enforcement**: Should `resource-types-contrib` CI block merges if `manifests_gen.go` is out of date, or should CI auto-regenerate and commit?

4. **Defaults key format**: Should `defaults.yaml` entries be file paths (e.g., `Compute/containers/containers.yaml`) or logical resource type names (e.g., `Radius.Compute/containers`)? File paths are directly resolvable but break on renames. Logical names are more stable but require a name-to-path resolution step in the generator.

5. **Multiple provider channels**: Should we support different default sets for different deployment targets (e.g., self-hosted, AWS, Azure)? This could be modeled as multiple lists in `defaults.yaml` or separate files.

6. **CLI for listing defaults**: Should we expose a CLI command (e.g., `rad resource-type list-defaults`) to show which resource types are embedded as defaults?

7. **Schema compatibility validation at build time**: Should the `go generate` step or CI validate that embedded schemas are compatible with the current Radius version (e.g., API version compatibility, required fields)?

## Future Enhancements

- **Multi-location bundles**: Support embedding manifests with location-specific configurations for different deployment targets.
- **Stability levels**: Add alpha/beta/GA annotations to resource types, allowing defaults to be filtered by stability.
- **Automated schema validation CI**: Run schema compatibility checks in CI when `resource-types-contrib` is updated, before the dependency is bumped in Radius.
- **Documentation generation**: Auto-generate documentation of default resource types from the embedded manifests.
- **Dev mode override**: Allow developers to override embedded defaults via an external directory (the current `ManifestDirectory` config already supports this, since directory-based manifests registered after embedded ones will overwrite them via `CreateOrUpdate`).

## Alternatives considered

### 1. GitHub Actions sync workflow (push model)

A workflow in `resource-types-contrib` opens PRs against `radius` when default manifests change, using a merge/transform script to combine files by namespace and inject `location` blocks.

**Rejected because**: Requires cross-repo PATs, couples `resource-types-contrib` CI to Radius manifest structure, and the transform logic (merging namespaces, injecting location) is fragile. The Go module approach makes the transform unnecessary since `location` can be inferred.

### 2. `raven-actions/repo-files-sync` GitHub Action

A third-party action that syncs files between repositories via PRs.

**Rejected because**: Does 1:1 file copying and cannot merge multiple per-type files into a single per-namespace manifest, and cannot inject `location` blocks. The necessary transformations make it unsuitable.

### 3. Git submodule

Add `resource-types-contrib` as a submodule in `radius`.

**Rejected because**: Submodule workflows are cumbersome for contributors (requires `git submodule update`), and version pinning is less ergonomic than Go module versioning.

### 4. Per-file `defaultRegistration: true` annotation

Add a boolean field to each YAML manifest instead of using a central file.

**Rejected because**: Pollutes the `ResourceProvider` struct with a deployment concern, requires embedding all YAML files (binary bloat), and is harder to review/discover than a central list. See detailed comparison in the Design section.

### 5. Runtime fetch from GitHub

Download manifests from `resource-types-contrib` at startup.

**Rejected because**: Introduces a runtime network dependency, breaks reproducibility (non-deterministic builds), and makes startup fragile. Air-gapped deployments would fail entirely.

## Design Review Notes

<!-- Update this section with the decisions made during the design review meeting. This should be updated before the design is merged. -->
