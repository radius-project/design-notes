# `manifest-to-bicep-extension` CLI Technical Design

* **Author**: Will Smith (@willdavsmith)

## Overview

This design document describes the changes to replace the `npx`-based approach for generating Bicep extensions with a native binary approach. Currently, the `rad bicep publish-extension` command uses `npx` to run the `@radius-project/manifest-to-bicep-extension` package. This change will instead download and use a native `manifest-to-bicep-extension` binary, similar to how we handle the Bicep CLI itself.

This change improves the user experience by removing the Node.js/`npm` dependency for users who want to generate and publish Bicep extensions, making the tooling more consistent with our existing approach for managing external binaries.

## Terms and definitions

- **Bicep Extension**: A package that extends Bicep's type system to support custom resource types
- **manifest-to-bicep-extension**: A CLI tool that converts resource provider manifests to Bicep extension format, introduced in this design document
- **npx**: A Node.js package runner that executes packages without installing them globally
- **Resource Provider Manifest**: A YAML file that defines custom resource types and their schemas

## Objectives

> **Issue Reference:**

https://github.com/radius-project/radius/issues/9230
https://github.com/radius-project/radius/issues/9450

### Goals

- Remove the Node.js/NPM dependency from the `rad bicep publish-extension` command
- Provide a consistent binary management experience similar to the Bicep CLI
- Maintain backward compatibility with existing workflows
- Ensure the binary is automatically downloaded when needed

### Non goals

- Changing the functionality of the manifest-to-bicep-extension tool itself
- Modifying the Bicep extension format or publishing process
- Supporting multiple versions of the manifest-to-bicep-extension tool

### User scenarios

#### User story 1
As a platform engineer, I want to generate and publish Bicep extensions for my custom resource types without installing Node.js, so I can use Radius in environments where Node.js is not available. I can use `rad bicep download` to get the necessary binaries and then run `rad bicep publish-extension` to generate the Bicep extension from my resource provider manifest. I could also publish the `manifest-to-bicep-extension` binary to a central repository for my team to use, ensuring everyone has a consistent toolchain.

## User Experience

The user experience should be exactly the same as it is today, with the following changes:
- The `rad bicep publish-extension` command will no longer require `npx` and will instead execute a native binary.
- The user will need to run `rad bicep download` to download the new `manifest-to-bicep-extension` binary, similar to how they currently download the Bicep CLI.
- The command will prompt the user to run `rad bicep download` if the binary is not found.

## Design

### High Level Design

The solution replaces the `npx`-based approach with a native binary approach:

1. The `rad bicep download` command is enhanced to download both the Bicep CLI and the `manifest-to-bicep-extension` binary
2. The binaries are stored in the same directory (`~/.rad/bin/`)
3. The `rad bicep publish-extension` command uses the downloaded binary instead of `npx`
4. The binary download follows the same pattern as the Bicep CLI download

### Architecture Diagram

```
┌─────────────────────┐
│   rad bicep         │
│   download          │
└──────────┬──────────┘
           │
           ├─── Downloads Bicep CLI
           │
           └─── Downloads manifest-to-bicep-extension
                           │
                           v
                   ┌───────────────┐
                   │ ~/.rad/bin/   │
                   │ ├── rad-bicep │
                   │ └── manifest- │
                   │     to-bicep- │
                   │     extension │
                   └───────────────┘
```

### Detailed Design

#### Binary Management

The manifest-to-bicep-extension binary is managed similarly to the Bicep CLI:

1. **Download Location**: The binary is downloaded from GitHub releases at `https://github.com/radius-project/bicep-tools/releases/latest/download/`
2. **Local Storage**: The binary is stored in `~/.rad/bin/manifest-to-bicep-extension` (with `.exe` extension on Windows)
3. **Platform Support**: Binaries are available for the same platforms as Bicep (Windows, Linux, macOS on AMD64 and ARM64)

#### Changes to rad bicep download

The `rad bicep download` command now downloads both binaries:
- The existing Bicep CLI download logic remains unchanged
- After downloading Bicep, it also downloads the manifest-to-bicep-extension binary
- Both downloads use the same retry mechanism and error handling

#### Changes to rad bicep publish-extension

The command now:
1. Checks if the manifest-to-bicep-extension binary exists
2. If not found, prompts the user to run `rad bicep download`
3. Executes the binary directly instead of using `npx`

### Implementation Details

#### CLI Updates

The key changes are in the following files:

1. **pkg/cli/bicep/bicep.go**: Added `GetBicepManifestToBicepExtensionCLIPath()` function and updated `DownloadBicep()` to download both binaries

2. **pkg/cli/bicep/tools/download_tools.go**: Added `DownloadManifestToBicepExtensionCLI()` function
   - Downloads the manifest-to-bicep-extension binary from GitHub releases
   - Handles platform-specific naming and paths

3. **pkg/cli/cmd/bicep/publishextension/publish.go**: Updated to use the binary instead of `npx`
   - Checks for the binary's existence
   - Executes the binary with the provided arguments

#### Build System Updates

1. **GitHub Actions**: Updated workflows to install and use the binary instead of `npx`
2. Updated `radius-project/bicep-tools` repository to include the manifest-to-bicep-extension binary in releases

### Error Handling

- If the manifest-to-bicep-extension binary is not found, the user is prompted to run `rad bicep download`
- Download failures are retried using the existing retry mechanism
- Platform-specific errors (unsupported OS/architecture) provide clear error messages

## Test plan

1. **Unit Tests**:
   - Test binary path resolution
   - Test platform detection logic
   - Test error handling when binary is missing

2. **Functional Tests**:
   - Remove `npx` CLI install from test workflows
   - Use existing test cases to verify that `rad bicep publish-extension` works with the new binary

## Security

- The binary is downloaded over HTTPS from GitHub releases
- The binary is signed and verified (same as Bicep CLI)
- No new security considerations beyond existing binary management

## Compatibility

- This is a breaking change for users who rely on the NPX-based approach
- Users must run `rad bicep download` to get the new binary
- The command-line interface remains unchanged

## Monitoring and Logging

- Download progress and errors are logged to the console
- Binary execution errors are propagated to the user
- No new telemetry is added

## Development plan

1. Implement binary download functionality in `download_tools.go`
2. Update `rad bicep download` to download both binaries
3. Update `rad bicep publish-extension` to use the binary
4. Update build scripts and CI/CD pipelines
5. Add tests for new functionality
6. Update documentation

## Open Questions

1. **Q**: Where should the manifest-to-bicep-extension binaries be hosted?

   **A**: Currently planning to use GitHub releases in the radius-project/bicep-tools repository

2. **Q**: Should we support multiple versions of the tool?

   **A**: For now, we'll only support the latest version, similar to our Bicep CLI approach

3. **Q**: Should we auto-download the binary if it's missing?

   **A**: No, we'll prompt the user to run `rad bicep download` to maintain consistency with Bicep CLI behavior

## Alternatives considered

1. **Continue using `npx`**: Rejected because it requires Node.js, which adds an unnecessary dependency
2. **Bundle the tool with Radius**: Rejected because it would significantly increase the Radius binary size

## Design Review Notes

TBD - To be updated after design review
