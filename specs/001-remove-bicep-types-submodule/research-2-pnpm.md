# Research: pnpm Migration and Submodule Removal

**Plan**: [plan-2-pnpm-submodule.md](./plan-2-pnpm-submodule.md)
**Date**: 2026-01-22
**Status**: Complete

## Research Questions

From Plan 2, the following items required research:

1. pnpm Git Reference Syntax - Exact syntax for subdirectory git references
2. pnpm + Dependabot - Integration and configuration
3. npm to pnpm Migration - Best practices and commands
4. pnpm in GitHub Actions - Installation and caching strategies
5. Dev Container pnpm - Installation method for dev containers
6. bicep-types npm Package - Verify package structure and validity

---

## Findings

### 1. pnpm Git Subdirectory References

| Aspect | Details |
|--------|---------|
| **Decision** | Use `github:Azure/bicep-types#path:/src/bicep-types` syntax |
| **Evidence** | [pnpm.io/package-sources](https://pnpm.io/package-sources) documents the `#path:/` parameter |
| **Rationale** | This is the official pnpm syntax for referencing packages in subdirectories |
| **Alternatives Considered** | npm tarball from GitHub releases - not available; fork to separate repo - unnecessary complexity |

**Syntax Options:**

```json
// Basic subdirectory reference
"bicep-types": "github:Azure/bicep-types#path:/src/bicep-types"

// With specific commit SHA (recommended for reproducibility)
"bicep-types": "github:Azure/bicep-types#c1a289be58be&path:/src/bicep-types"

// With semver (if tags exist)
"bicep-types": "github:Azure/bicep-types#semver:^1.0.0&path:/src/bicep-types"
```

**Recommended Approach:**

Use commit SHA pinning for reproducibility:

```json
{
  "devDependencies": {
    "bicep-types": "github:Azure/bicep-types#c1a289be58bea8e23cecbce871a11a3fad8c3467&path:/src/bicep-types"
  }
}
```

**Note:** pnpm classifies git subdirectory references as "exotic sources". For transitive dependency security, consider `blockExoticSubdeps: true` in `.npmrc`.

---

### 2. pnpm + Dependabot Integration

| Aspect | Details |
|--------|---------|
| **Decision** | Use `package-ecosystem: "npm"` for pnpm projects; git dependencies require manual updates |
| **Evidence** | [GitHub Dependabot docs](https://docs.github.com/code-security/dependabot) - pnpm v7-v10 lockfiles supported under npm ecosystem |
| **Rationale** | Dependabot treats pnpm as npm-compatible; git-based deps have limited auto-update support |
| **Alternatives Considered** | Renovate bot - more pnpm-native but adds complexity; manual updates only - reduces automation |

**Configuration:**

```yaml
# .github/dependabot.yml
version: 2
updates:
  # For typespec directory
  - package-ecosystem: "npm"  # Works for pnpm
    directory: "/typespec"
    schedule:
      interval: "weekly"
    groups:
      typespec:
        patterns:
          - "*"

  # For generator directory
  - package-ecosystem: "npm"
    directory: "/hack/bicep-types-radius/src/generator"
    schedule:
      interval: "weekly"

  # For autorest.bicep directory
  - package-ecosystem: "npm"
    directory: "/hack/bicep-types-radius/src/autorest.bicep"
    schedule:
      interval: "weekly"
```

**Limitations:**

| Feature | Support |
|---------|---------|
| pnpm lockfile updates | ✅ Supported (v7-v10) |
| Registry package updates | ✅ Full support |
| Git-based dependency updates | ⚠️ Limited - manual process needed |
| Security alerts | ✅ Supported |

**Workaround for Git Dependencies:**

Create a scheduled GitHub Action to check for bicep-types updates:

```yaml
name: Check bicep-types updates
on:
  schedule:
    - cron: '0 0 * * 0'  # Weekly
jobs:
  check:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: |
          CURRENT=$(git ls-remote https://github.com/Azure/bicep-types HEAD | cut -f1)
          echo "Latest bicep-types commit: $CURRENT"
          # Compare with pinned version and create issue if different
```

---

### 3. npm to pnpm Migration

| Aspect | Details |
|--------|---------|
| **Decision** | Use `pnpm import` for lockfile conversion, then `pnpm install` |
| **Evidence** | [pnpm.io/cli/import](https://pnpm.io/cli/import) documents the import command |
| **Rationale** | Preserves exact dependency resolutions from package-lock.json |
| **Alternatives Considered** | Fresh `pnpm install` - may resolve different versions; delete node_modules first - not necessary |

**Migration Steps:**

```bash
# Per-directory migration
cd typespec/

# 1. Import existing lockfile (converts package-lock.json → pnpm-lock.yaml)
pnpm import

# 2. Install dependencies with pnpm (validates the import)
pnpm install

# 3. Run tests to verify
pnpm test

# 4. Remove old lockfile
rm package-lock.json

# Repeat for each directory:
# - hack/bicep-types-radius/src/generator/
# - hack/bicep-types-radius/src/autorest.bicep/
```

**Supported Import Sources:**

- `package-lock.json` (npm v5+) ✅
- `npm-shrinkwrap.json` ✅
- `yarn.lock` ✅

**Key Differences from npm:**

| Aspect | npm | pnpm |
|--------|-----|------|
| node_modules structure | Flat | Symlinked from store |
| Disk usage | Duplicated | Content-addressable (shared) |
| Install speed | Slower | Faster |
| Phantom dependencies | Allowed | Blocked by default |
| Lock file | package-lock.json | pnpm-lock.yaml |

---

### 4. pnpm in GitHub Actions

| Aspect | Details |
|--------|---------|
| **Decision** | Use `pnpm/action-setup@v4` with `actions/setup-node@v4` caching |
| **Evidence** | [github.com/pnpm/action-setup](https://github.com/pnpm/action-setup) |
| **Rationale** | Official pnpm action with built-in store caching |
| **Alternatives Considered** | Manual npm install of pnpm - slower and no caching benefits |

**Recommended Configuration:**

```yaml
- name: Install pnpm
  uses: pnpm/action-setup@v4
  with:
    version: 10  # or specific: 10.8.1

- name: Setup Node.js
  uses: actions/setup-node@v4
  with:
    node-version-file: '.node-version'
    cache: 'pnpm'  # Built-in pnpm cache support

- name: Install dependencies
  run: pnpm install --frozen-lockfile
```

**Action Features:**

| Option | Description | Recommended |
|--------|-------------|-------------|
| `version` | pnpm version (10, 10.x, 10.8.1) | `10` |
| `run_install` | Auto-run install | `false` (explicit is better) |
| Built-in cache | Automatic store caching | Uses setup-node cache |

**Full Workflow Example:**

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Install pnpm
        uses: pnpm/action-setup@v4
        with:
          version: 10
      
      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version-file: '.node-version'
          cache: 'pnpm'
          cache-dependency-path: |
            typespec/pnpm-lock.yaml
            hack/bicep-types-radius/src/*/pnpm-lock.yaml
      
      - name: Install TypeSpec dependencies
        run: pnpm --prefix typespec install --frozen-lockfile
      
      - name: Install generator dependencies
        run: pnpm --prefix hack/bicep-types-radius/src/generator install --frozen-lockfile
```

---

### 5. Dev Container pnpm Installation

| Aspect | Details |
|--------|---------|
| **Decision** | Use official Node.js feature with Corepack activation |
| **Evidence** | [containers.dev/features](https://containers.dev/features) - Node feature includes pnpm via Corepack |
| **Rationale** | Corepack is the Node.js-native package manager manager |
| **Alternatives Considered** | Separate pnpm feature from devcontainers-extra - adds unnecessary dependency |

**Recommended Configuration:**

```json
// .devcontainer/devcontainer.json
{
  "features": {
    "ghcr.io/devcontainers/features/node:1": {
      "version": "20"
    }
  },
  "postCreateCommand": "corepack enable && corepack prepare pnpm@latest-10 --activate"
}
```

**Alternative (dedicated pnpm feature):**

```json
{
  "features": {
    "ghcr.io/devcontainers-extra/features/pnpm:2": {
      "version": "10"
    }
  }
}
```

**Current Dev Container Status:**

The Radius dev container already includes Node.js. The update needed:

1. Add `corepack enable` to postCreateCommand
2. Add `corepack prepare pnpm@latest-10 --activate`

---

### 6. bicep-types npm Package Structure

| Aspect | Details |
|--------|---------|
| **Decision** | The `src/bicep-types/` directory is a valid, self-contained TypeScript npm package |
| **Evidence** | Repository exploration of Azure/bicep-types |
| **Rationale** | Package has complete package.json, TypeScript config, and test suite |
| **Alternatives Considered** | Wait for official npm publish - uncertain timeline; not necessary |

**Package Structure:**

```
src/bicep-types/
├── package.json          # Package configuration
├── tsconfig.json         # TypeScript compilation config
├── jest.config.ts        # Test configuration
├── .eslintrc.js          # Linting rules
├── README.md             # Package documentation
├── src/
│   ├── index.ts          # Main exports
│   ├── types.ts          # Core type definitions
│   ├── indexer.ts        # Type indexing
│   ├── utils.ts          # Utilities
│   └── writers/
│       ├── json.ts       # JSON serialization
│       └── markdown.ts   # Markdown generation
└── test/
    └── integration/      # Integration tests
```

**Main Exports:**

```typescript
// From src/index.ts
export * from "./writers/json";      // writeTypesJson, readTypesJson, writeIndexJson
export * from "./writers/markdown";  // writeMarkdown, writeIndexMarkdown
export * from "./indexer";           // buildIndex
export * from "./types";             // TypeFactory, TypeIndex, BicepType, etc.
```

**Key Types Used by Radius:**

| Type | Purpose |
|------|---------|
| `TypeFactory` | Creates and manages Bicep types |
| `TypeIndex` | Index structure for resources and functions |
| `BicepType` | Union of all Bicep type variants |
| `ResourceType` | Resource type definition |
| `ObjectType` | Object type definition |
| `FunctionType` | Function type definition |

**Compatibility:** The package is standard TypeScript/npm and fully compatible with pnpm.

---

## Implementation Decisions

### Package.json Updates

**Before (file: reference to submodule):**

```json
{
  "devDependencies": {
    "bicep-types": "file:../../../../bicep-types/src/bicep-types"
  }
}
```

**After (pnpm git reference):**

```json
{
  "devDependencies": {
    "bicep-types": "github:Azure/bicep-types#c1a289be58bea8e23cecbce871a11a3fad8c3467&path:/src/bicep-types"
  }
}
```

### Makefile Updates

**Before:**

```makefile
generate-bicep-types:
 git submodule update --init --recursive; \
 npm --prefix bicep-types/src/bicep-types install; \
 npm --prefix bicep-types/src/bicep-types ci && npm --prefix bicep-types/src/bicep-types run build; \
 npm --prefix hack/bicep-types-radius/src/autorest.bicep ci && ...
```

**After:**

```makefile
generate-bicep-types:
 pnpm --prefix hack/bicep-types-radius/src/autorest.bicep install && \
 pnpm --prefix hack/bicep-types-radius/src/autorest.bicep run build; \
 pnpm --prefix hack/bicep-types-radius/src/generator install && \
 pnpm --prefix hack/bicep-types-radius/src/generator run generate -- ...
```

### Workflow Updates

**Before:**

```yaml
- uses: actions/checkout@v4
  with:
    submodules: recursive
```

**After:**

```yaml
- uses: actions/checkout@v4
  # No submodules property

- uses: pnpm/action-setup@v4
  with:
    version: 10

- uses: actions/setup-node@v4
  with:
    node-version-file: '.node-version'
    cache: 'pnpm'
```

### Dependabot Updates

**Remove:**

```yaml
- package-ecosystem: gitsubmodule
  directory: /
```

**Update npm entries to cover pnpm directories** (already using `npm` ecosystem which supports pnpm).

---

## Risks and Mitigations

| Risk | Likelihood | Impact | Mitigation |
|------|------------|--------|------------|
| pnpm git subdirectory refs unstable | Low | High | Well-documented feature; test thoroughly |
| Dependabot can't update git deps | Medium | Low | Document manual process; consider future Renovate |
| Dev container pnpm issues | Low | Medium | Use Corepack which is Node.js-native |
| CI cache invalidation | Low | Low | Configure cache-dependency-path properly |
| Phantom dependency issues | Medium | Medium | pnpm's strictness catches issues early; fix during migration |

---

## Summary

| Research Question | Answer | Confidence |
|-------------------|--------|------------|
| Git subdirectory syntax | `github:Azure/bicep-types#<sha>&path:/src/bicep-types` | ✅ High |
| Dependabot integration | `npm` ecosystem; manual git dep updates | ✅ High |
| npm to pnpm migration | `pnpm import` + `pnpm install` | ✅ High |
| GitHub Actions setup | `pnpm/action-setup@v4` + `cache: 'pnpm'` | ✅ High |
| Dev container pnpm | Corepack activation in postCreateCommand | ✅ High |
| bicep-types package valid | Yes, standard TypeScript npm package | ✅ High |

**All research questions resolved. Plan 2 is ready for task breakdown.**
