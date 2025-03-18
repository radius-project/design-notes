# Radius Control Plane Upgrades

- **Author**: Yetkin Timocin (@ytimocin)

## Overview

Radius is an open-source, cloud-native application platform that enables developers and operators to define, deploy, and collaborate on applications across cloud environments. As Radius evolves with new features and improvements, users need a reliable way to upgrade their installations without disruption or data loss.

This feature introduces in-place upgrades for the Radius Control Plane, allowing users to seamlessly update their Radius installations to newer versions without the current cumbersome process of uninstalling and reinstalling. By running a single command (`rad upgrade kubernetes`), users can update all Radius components while preserving their application deployments and configurations. **It's important to note that user applications deployed through Radius continue running without interruption during this process, as Radius only maintains deployment metadata and does not directly control application runtime execution.**

## Terms and definitions

| Term                     | Definition                                                                                                                                                                                                                                                                                                                                                                   |
| ------------------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Radius Control Plane** | The set of management components in Radius that coordinate application lifecycle operations. These include the Universal Control Plane, Deployment Engine, Applications Resource Provider, etc.                                                                                                                                                                              |
| **In-place Upgrade**     | An upgrade approach that updates an existing Radius installation in its current location without requiring complete reinstallation. The process preserves all user data, configurations, and deployed application metadata while minimizing downtime of the control plane. This contrasts with alternative approaches like parallel deployments or complete reinstallations. |
| **Rolling Upgrade**      | An upgrade strategy that updates components incrementally (one by one) rather than all at once, reducing downtime.                                                                                                                                                                                                                                                           |
| **User Data Backup**     | A point-in-time backup of user data taken before an upgrade to enable recovery if problems occur.                                                                                                                                                                                                                                                                            |
| **User Data Restore**    | The process of reverting user data to a previously backed up state after an upgrade attempt encounters issues.                                                                                                                                                                                                                                                               |
| **Pre-flight Checks**    | Validation steps performed before an upgrade to ensure prerequisites are met and the system is in a valid state for upgrading.                                                                                                                                                                                                                                               |
| **Version Skipping**     | The ability to upgrade directly from one version to a newer non-adjacent version (e.g., v0.40 → v0.44) without installing intermediate versions.                                                                                                                                                                                                                             |

## Assumptions

1. **User permissions**: Users running the upgrade command have enough permissions on both the Kubernetes cluster and the Radius installation.

2. **Resource requirements**: The Kubernetes cluster has sufficient compute resources (CPU, memory) to run both the existing and new version components during the rolling upgrade process.

3. **CLI version compatibility**: User has a Radius CLI version that includes the `rad upgrade kubernetes` feature. While older CLIs can't perform upgrades, newer CLIs maintain backward compatibility with older control planes.

   Example: If you have CLI v0.42 and Control Plane v0.42:

   - You CAN upgrade to Control Plane v0.44 using the v0.42 CLI (if v0.42 CLI includes the upgrade feature)
   - After upgrading, your CLI (v0.42) will still work with basic operations against your v0.44 Control Plane
   - However, you won't be able to use any new v0.44 features from your v0.42 CLI
   - For full functionality, you should upgrade your CLI to match the Control Plane version after the upgrade completes

4. **Network connectivity**: The upgrade process requires internet connectivity to pull container images from registries and validate versions. The process is not intended to run in air-gapped environments.

   - An air-gapped environment is one where systems are physically isolated from unsecured networks like the public internet.
   - These environments are common in high-security scenarios (military, financial, healthcare, government) where external network connectivity is restricted.
   - Future implementations will need to address version validation without GitHub API access and provide mechanisms for offline image management.

5. **Stable starting state**: The Radius installation being upgraded is in a healthy, stable state. Attempting to upgrade an already failing installation may lead to unpredictable results.

6. **Storage availability**: The Kubernetes cluster has sufficient persistent storage capacity for backup operations during the upgrade process.

## Objectives

> **Issue Reference:** <https://github.com/radius-project/radius/issues/8095>

### Goals

- **Simplify upgrade process**: Provide a single CLI command (`rad upgrade kubernetes`) to upgrade Radius without manual reinstallation.
- **Ensure data safety**: Implement automatic user data backups before upgrades and restore capability if failures occur.
- **Minimize downtime**: Use rolling upgrades where possible to keep Radius control plane available during the upgrade process.
- **Preserve application continuity**: Ensure that user applications continue running without interruption throughout the Radius upgrade process.

### Non goals

- **Downgrade support**: The upgrade process is designed to move forward to newer versions only. Downgrading to previous versions is not supported.
- **Multi-cluster upgrades**: Managing upgrades across multiple clusters simultaneously is out of scope, as Radius doesn't support multiple clusters per installation as of March 2025.
- **Dependency major version upgrades**: Upgrading major versions of dependencies like Postgres, Dapr, or Contour is not covered. These upgrades should be handled separately following their respective guidelines.
- **Full Helm upgrade support**: While we use Helm internally, making `helm upgrade` work completely for Radius is not in scope. Running `helm upgrade` on a Radius Helm installation doesn't configure all necessary components for the control plane to work properly.
- **Zero-downtime control plane**: While we aim to minimize disruption to the control plane itself, guaranteeing absolutely no downtime for the Radius control plane components is not a goal for this initial release.
- **Automatic CLI upgrades**: The upgrade command updates only the Radius control plane components running in Kubernetes. It does not automatically update your local Radius CLI version. You must manually download and install the matching CLI version separately.
- **Application management during upgrades**: Since Radius only maintains deployment metadata and doesn't control runtime execution of user applications, managing or modifying user workloads during the upgrade is explicitly not in scope.

### User scenarios (optional)

**Important Note:** Radius upgrades only affect the control plane components and deployment metadata. User applications deployed through Radius continue running without interruption, as Radius does not manage their runtime execution.

The primary users of this feature are system administrators, application developers, and DevOps engineers responsible for maintaining and upgrading the Radius platform.

- **System Administrator**:
  Responsible for managing the infrastructure and ensuring the smooth operation of the Radius platform. They have advanced knowledge of Kubernetes administration and are concerned with stability, security, and minimizing downtime during upgrades. They need clear visibility into the upgrade process and rollback options if issues occur.
- **DevOps Engineer**:
  Focused on automating deployment processes and maintaining the CI/CD pipeline. They typically integrate Radius into larger workflows and want predictable, scriptable upgrade paths that can be incorporated into their automation systems.
- **Application Developer**:
  Uses Radius to deploy and manage their applications but isn't deeply involved in platform administration. They primarily care that their applications continue running during Radius upgrades and that any API or interface changes are clearly documented.

#### Scenario 1: System Administrator performs a standard version upgrade

**User Story:** As a system administrator, I need to upgrade Radius to a newer version while maintaining continuous operation of user applications, as Radius only stores deployment metadata and doesn't control application runtime.

**User Experience:**

```bash
# Check current Radius version
> rad version
RELEASE   VERSION   BICEP     COMMIT
0.44.0    v0.44.0   0.33.93   1dd17270ec9bc9e764f314fa62c248406034edda

# Perform a basic upgrade to a specific version
> rad upgrade kubernetes --version v0.45.0

Initiating Radius upgrade from v0.44.0 to v0.45.0...
Pre-flight checks:
  ✓ Valid version target
  ✓ Compatible upgrade path
Creating backup of current user data...
  ✓ Backup created successfully
Upgrading control plane components:
  ✓ Universal Control Plane
  ✓ Deployment Engine
  ✓ Applications Resource Provider
  ✓ Controller
  ✓ ...
Performing post-upgrade verification...
  ✓ All components healthy

Upgrade complete! Radius has been successfully upgraded to v0.45.0.
Note: Your local Radius CLI is still v0.44.0. To upgrade your CLI, download the v0.45.0 version.
```

**Result:**

1. Pre-flight checks validate the upgrade is possible
1. System automatically creates a user data backup for recovery
1. All control plane components are upgraded in sequence
1. Post-upgrade verification confirms system health
1. User is notified about the CLI version mismatch

**Exceptions:**

1. If version check fails because the target is lower than current version
2. If components fail health checks after upgrade

#### Scenario 2: DevOps engineer upgrades with custom configuration

**User Story:** As a DevOps engineer, I need to upgrade Radius with custom configuration parameters to match organization's infrastructure requirements.

**User Experience:**

```bash
# Upgrade with custom configuration values
> rad upgrade kubernetes --version v0.44.0 --set global.monitoring.enabled=true --set global.resources.limits.memory=2Gi

Initiating Radius upgrade from v0.43.0 to v0.44.0 with custom configuration...
Custom configuration detected:
  - global.monitoring.enabled: true
  - global.resources.limits.memory: 2Gi
Pre-flight checks:
  ✓ Valid version target
  ✓ Compatible upgrade path
  ✓ Custom configuration validated
Creating backup of current user data...
  ✓ Backup created successfully
Upgrading control plane components with custom configuration:
  ✓ Universal Control Plane
  ✓ Deployment Engine
  ✓ Applications Resource Provider
  ✓ Controller
  ✓ ...
Applying custom configuration settings...
Performing post-upgrade verification...
  ✓ All components healthy
  ✓ Custom configuration applied successfully

Upgrade complete! Radius has been successfully upgraded to v0.44.0 with your custom configuration.
```

**Result:**

1. System acknowledges custom configuration parameters
1. Pre-flight checks validate both upgrade path and configuration
1. Control plane components are upgraded with custom settings
1. Verification confirms both health and configuration application

**Exceptions:**

1. If custom configuration parameters are invalid (or should it be ignored?)

#### Scenario 3: Handling upgrade failure and recovery

**User Story:** As a system administrator, I need confidence that if an upgrade fails, the system can recover without data loss or extended downtime.

**User Experience:**

```bash
# Attempt upgrade that encounters an issue
> rad upgrade kubernetes --version v0.44.0

Initiating Radius upgrade from v0.43.0 to v0.44.0...
Pre-flight checks:
  ✓ Valid version target
  ✓ Compatible upgrade path
Creating backup of current user data...
  ✓ Backup created successfully
Upgrading control plane components:
  ✓ Universal Control Plane
  ✓ Deployment Engine
  ✗ Applications Resource Provider (ERROR: Container image pull failed)

ERROR: Upgrade failed during Applications Resource Provider update.
Initiating automatic rollback to v0.43.0...
  ✓ Restoring from backup (Not sure if this is needed)
  ✓ Universal Control Plane reverted
  ✓ Deployment Engine reverted
  ✓ System verification complete
  ✓ ...

Rollback complete. System has been restored to v0.43.0.
Review Kubernetes events and logs for more details on the failure.
```

**Result:**

1. System detects failure during the upgrade process
1. Automatic restore is initiated using the pre-upgrade backup
1. All components are restored to their previous state
1. User is informed of the failure and suggested next steps

**Exceptions:**

1. If the user data backup restoration fails (rare but possible)

#### Scenario 4: Upgrading across multiple versions

**User Story:** As a DevOps engineer, I need to upgrade Radius from an older version to the latest version in a single operation.

**User Experience:**

```bash
# Check current version (significantly behind latest)
> rad version
RELEASE   VERSION   BICEP     COMMIT
0.40.0    v0.40.0   0.31.93   1dd17270ec9bc9e725f314fa62c249406034edda

# Upgrade directly to latest version
> rad upgrade kubernetes --version latest

Initiating Radius upgrade from v0.40.0 to v0.44.0 (latest stable)...
Pre-flight checks:
  ✓ Valid version target
  ✓ Multiple version jump detected (v0.40.0 → v0.44.0)
  ✓ Compatible upgrade path confirmed
  ✓ Database schema changes detected
Creating backup of current user data...
  ✓ Backup created successfully
Upgrading control plane components:
  ✓ Universal Control Plane
  ✓ Deployment Engine
  ✓ Applications Resource Provider
  ✓ Controller
Performing post-upgrade verification...
  ✓ All components healthy
  ✓ Database schema updated successfully

Upgrade complete! Radius has been successfully upgraded to v0.44.0.
Note: Your local Radius CLI is still v0.40.0. To upgrade your CLI, download the latest version.
```

**Result:**

1. System detects a multi-version upgrade scenario
1. Pre-flight checks validate that direct upgrade is supported
1. All components are upgraded to the target version
1. Verification confirms system health with the new version

**Exceptions:**

1. If direct upgrade path isn't supported between versions
1. If database migrations encounter issues (this may be the case when we introduce Postgres as the data store)
1. If intermediate upgrades are required first

## Design

### High-Level Design Diagram

```mermaid
graph TD
  CLI["Radius CLI (rad upgrade kubernetes)"] -->|Initiates Upgrade| KubernetesAPI["Kubernetes API"]
  KubernetesAPI -->|Manages Helm Charts| Helm["Helm"]
  Helm -->|Upgrades Components| ControlPlane["Radius Control Plane"]
  subgraph ControlPlane
      UniversalControlPlane["UCP"]
      DeploymentEngine["Deployment Engine"]
      ARP["Applications RP"]
      Controller["Controller"]
      DynamicRP["Dynamic RP"]
  end
  ControlPlane -->|Stores Metadata| Database["etcd"]
  ControlPlane -->|Interacts with| Dependencies["Dependencies (Dapr, Contour)"]
  CLI -->|Logs Progress| User["User"]
  CLI -->|Performs Pre-flight Checks| PreFlight["Pre-flight Checks"]
  PreFlight -->|Validates| KubernetesAPI
  CLI -->|Creates User Data Backup| Backup["User Data Backup"]
  Backup -->|Restores on Failure| Restore["Restore Mechanism"]
```

- **Important Note:** As of April 2025, Postgres is not fully implemented yet as the data store of Radius. We use etcd in production.

### Architecture Diagram

```mermaid
graph TD
    User([User]) --> CLI["Radius CLI"]
    CLI -->|"Initiates Upgrade"| K8sAPI["Kubernetes API"]

    K8sAPI --> PreflightChecks["Preflight Checks"]
    PreflightChecks --> UserDataBackup["Backup Creation"]
    UserDataBackup --> ComponentUpgrade["Component Upgrade"]
    ComponentUpgrade --> PostUpgradeVerify["Post-Upgrade Verification"]
    ComponentUpgrade -- "Failure" --> UserDataRestore["User Data Restore"]

    subgraph "Radius Control Plane"
        UCP["Universal Control Plane"]
        DE["Deployment Engine"]
        ARP["Applications RP"]
        DynamicRP["Dynamic RP"]
        Controller["Controller"]
        Dashboard["Dashboard"]
    end

    ComponentUpgrade --> UCP
    ComponentUpgrade --> DE
    ComponentUpgrade --> ARP
    ComponentUpgrade --> DynamicRP
    ComponentUpgrade --> Controller
    ComponentUpgrade --> Dashboard

    UCP -->|"Reads/Writes"| Database["etcd / PostgreSQL"]
    UCP -.->|"Only holds metadata"| UserApps["User Applications"]
```

### Detailed Design

```mermaid
graph TD
    Start[User runs rad upgrade kubernetes] --> ParseArgs[Parse command arguments]
    ParseArgs --> ValidateVersion[Validate version compatibility]
    ValidateVersion --> AcquireLock[Acquire upgrade lock]
    AcquireLock --> RunPreflights[Run pre-flight checks]
    RunPreflights --> UserDataBackup[Create user data backup]
    UserDataBackup --> PlanUpgrade[Calculate upgrade plan]
    PlanUpgrade --> ExecuteHelmUpgrade[Execute Helm chart upgrade]
    ExecuteHelmUpgrade --> MonitorProgress[Monitor upgrade progress]
    MonitorProgress --> VerifyComponents[Verify component health]
    VerifyComponents --> Success{Successful?}
    Success -- Yes --> ReleaseLock[Release upgrade lock]
    Success -- No --> UserDataRestore[Restore user data]
    UserDataRestore --> RollbackHelm[Rollback Helm changes]
    RollbackHelm --> ReleaseLock
    ReleaseLock --> End[Display results to user]
```

**Version Detection and Validation:**

The upgrade process begins with detecting the currently installed Radius version (the versions of the components in the cluster) and validating it against the requested target version. We can build an interface (or use/improve an existing one) to achieve this:

```go
// VersionValidator interface
type VersionValidator interface {
    ValidateTargetVersion(currentVersion, targetVersion string) error
    GetLatestVersion() (string, error)
    IsValidVersion(version string) bool
}
```

This interface will be implemented (or existing will be improved) to handle version comparisons, prevent downgrades, and resolve the "latest" version tag to a specific version number. The implementation will (probably) connect to the GitHub API to fetch available release versions when needed.

**Upgrade Lock Mechanism:**

To prevent concurrent modifications during upgrades, we'll implement a lock system that's acquired early in the process:

```go
type UpgradeLock interface {
    // Acquires the upgrade lock, preventing other commands from modifying data
    AcquireLock(ctx context.Context) error

    // Releases the upgrade lock, allowing other commands to proceed
    ReleaseLock(ctx context.Context) error

    // Checks if an upgrade is currently in progress
    IsUpgradeInProgress(ctx context.Context) (bool, error)
}
```

We can utilize Kubernetes Lease objects (coordination.k8s.io/v1) for implementing the distributed locking mechanism (open to discussion and suggestions). Leases are purpose-built for this use case, providing built-in lease duration and automatic expiration capabilities. For more information, see: <https://kubernetes.io/docs/concepts/architecture/leases/>.

Other CLI commands (`rad deploy app.bicep`, `rad delete app my-app` or other data-changing commands) that modify data will check for this lock before proceeding:

```go
inProgress, _ := upgradeLock.IsUpgradeInProgress(ctx)
if inProgress {
    return errors.New("An upgrade is currently in progress. Please try again after the upgrade completes.")
}
```

**Pre-flight Check System:**

Pre-flight checks run before any changes are made to ensure the upgrade can proceed safely.

```go
type PreflightCheck interface {
    Run(ctx context.Context) (bool, string, error)
    Name() string
    // Error, Warning, Info
    Severity() CheckSeverity
}
```

Checks will include:

1. Version compatibility verification
2. Existing installation detection
3. Database connectivity
4. Custom configuration validation

**User Data Backup and Restore System:**

Rather than taking complete snapshots of the underlying databases (etcd/PostgreSQL), we'll implement a more targeted approach that backs up only the user application metadata and configuration that Radius manages:

```go
type UserDataBackup interface {
    // Creates a backup of all user application metadata and configurations
    BackupUserData(ctx context.Context) (BackupID string, err error)
    // Lists available user data backups with metadata
    ListBackups(ctx context.Context) ([]BackupInfo, error)
}

type UserDataRestore interface {
    // Restores user data from a previous backup
    RestoreUserData(ctx context.Context, backupID string) error
}
```

**Helm-based Upgrade Process:**

The core of the upgrade functionality will be implemented through Helm, leveraging its chart upgrade capabilities while adding Radius-specific safety measures:

```go
type HelmUpgrader interface {
    // Upgrades the Radius installation to a specified version
    UpgradeRadius(ctx context.Context, options UpgradeOptions) error

    // Returns the current status of an ongoing upgrade
    GetUpgradeStatus(ctx context.Context) (UpgradeStatus, error)

    // Validates that an upgrade to the target version is possible
    ValidateUpgrade(ctx context.Context, targetVersion string) error
}

type UpgradeOptions struct {
    Version string // Target version to upgrade to
    Values map[string]interface{} // Custom configuration values
    Timeout time.Duration // Maximum time allowed for upgrade

    EnableUserDataBackup  bool // Whether automatic user data backup is enabled
    BackupID              string // ID of user data backup to use for recovery
}
```

**Component Health Verification:**

After upgrades, the system verifies all components are healthy:

```go
type HealthChecker interface {
    CheckComponentHealth(ctx context.Context, component string) (bool, error)
    CheckAllComponents(ctx context.Context) (map[string]ComponentHealth, error)
    WaitForHealthyState(ctx context.Context, timeout time.Duration) error
}

type ComponentHealth struct {
    Status      HealthStatus
    Message     string
    ReadinessDetails map[string]string
    LastChecked time.Time
}

type HealthStatus string

const (
    StatusHealthy     HealthStatus = "Healthy"
    StatusDegraded    HealthStatus = "Degraded"
    StatusUnavailable HealthStatus = "Unavailable"
    StatusUnknown     HealthStatus = "Unknown"
)
```

#### Advantages (of each option considered)

1. **User Experience**: Provides a single command for upgrading all Radius components, significantly simplifying the process compared to manual uninstall/reinstall
1. **Safety**: Built-in user data backup and restore capabilities ensure user data is protected during upgrades
1. **Flexibility**: Support for custom configuration parameters allows adaptation to different environments
1. **Transparency**: Clear, step-by-step output keeps users informed of the upgrade process
1. **Consistency**: Ensures all Radius components are upgraded together to compatible versions

#### Disadvantages (of each option considered)

1. **Additional Complexity**: Implementing backup/restore functionality adds complexity to the codebase
1. **Limited Control**: Users have less granular control compared to manually upgrading components
1. **Resource Requirements**: The upgrade process temporarily requires additional resources during the transition period
1. **Upgrade Path Constraints**: Some version combinations may require intermediate upgrades, limiting direct jump capability
1. **CLI Version Mismatch**: Potential confusion for users when their CLI version doesn't match the server version

#### Proposed Option

I recommend implementing the helm-based upgrade approach with automatic user data backup/restore functionality (we can even discuss if we would like this functionality in version 1 or not) because:

1. It provides the best balance between user experience simplicity and technical safety
2. Leverages existing Helm functionality while adding Radius-specific safety features
3. The automated user data backup/restore mechanism offers critical protection against data loss
4. Clear progress indicators and health checks give users confidence in the process
5. This approach aligns with how users currently install/reinstall Radius, providing consistency

### API design (if applicable)

No specific REST API addition is necessary.

### CLI Design (if applicable)

Main CLI changes:

- Addition of `rad upgrade kubernetes` command (details mentioned above)
- Lock logic will be added to the existing commands that change user data like:
  - `rad deploy app.bicep`
  - `rad app delete my-app`
  - `rad env delete default`

### Implementation Details

The implementation will primarily focus on the following components:

1. **Upgrade Command**: The `rad upgrade kubernetes` command implementation in the CLI codebase
2. **Version Validation**: Logic to verify compatibility between versions
3. **Lock Mechanism**: Kubernetes Lease-based distributed locking system
4. **Backup/Restore**: User data protection system using ConfigMaps/PVs
5. **Helm Integration**: Enhanced wrapper around Helm's upgrade capabilities
6. **Health Verification**: Component readiness and health check mechanisms

All components will follow Radius coding standards and include comprehensive unit tests.

### Error Handling

The upgrade process will implement the following error handling strategies:

1. **Pre-flight Validation**: Catch incompatibility issues before starting the upgrade
2. **Graceful Timeouts**: All operations will respect user-defined or default timeouts
3. **Automatic Rollback**: Failed upgrades trigger automatic restoration of previous state
4. **Detailed Error Reporting**: Clear error messages with troubleshooting guidance
5. **Idempotent Operations**: Commands can be safely retried after addressing issues
6. **Resource Cleanup**: Temporary resources created during the upgrade are properly removed

## Test Plan

### Unit Tests

- Test each interface implementation independently

### Integration Tests

- Verify interactions between components during upgrade process
- Test lock acquisition and release across multiple commands
- Validate backup/restore operations against actual databases

### End-to-End Tests

- Perform upgrades across consecutive versions (e.g., v0.43 → v0.44)
- Test version skipping scenarios (e.g., v0.40 → v0.44)
- Verify custom configuration is properly applied
- Simulate failures and verify automatic rollback

## Security

No changes to the existing security model needed.

## Compatibility (optional)

<!--
Describe potential compatibility issues with other components, such as
incompatibility with older CLIs, and include any breaking changes to
behaviors or APIs.
-->

## Monitoring and Logging

- Upgrade operations will emit detailed logs indicating progress, success, or failure of each step.
- Metrics can be collected to track upgrade success rates, duration, and rollback occurrences.
- Users will have clear visibility into upgrade status through CLI output and Kubernetes events.

## Development Plan

The following outlines the key implementation steps required to deliver the Radius Control Plane upgrade feature. Each step includes necessary unit and functional tests to ensure reliability and correctness, along with dependency information.

1. **Radius Helm Client Updates**

   - Implement the upgrade functionality in the Radius Helm client: [helmclient.go](https://github.com/radius-project/radius/blob/main/pkg/cli/helm/helmclient.go).
   - Add unit tests to validate Helm upgrade logic.
   - This task can be worked on in parallel with items 2-4. It is a blocker for item 6.

2. **Radius Contour Client Updates**

   - Implement the upgrade functionality in the Radius Contour client: [contourclient.go](https://github.com/radius-project/radius/blob/main/pkg/cli/helm/contourclient.go).
   - Add unit tests to verify correct behavior.
   - This task can be worked on in parallel with items 1, 3-4. It is a blocker for item 6.

3. **Cluster Upgrade Interface**

   - Extend the existing cluster management interface ([cluster.go](https://github.com/radius-project/radius/blob/main/pkg/cli/helm/cluster.go#L249)) to include a new method for upgrading Radius.
   - Implement this method in all relevant interface implementations.
   - Integrate with version validation and custom configuration handling.
   - Add comprehensive unit tests for this functionality.
   - This task can be worked on in parallel with items 1-2, 4. It is a blocker for item 6.

4. **User Data Backup and Restore Interfaces**

   - Define two new interfaces in the `components/database` package:
     - `UserDataBackup`: Responsible for creating backups of user data before the upgrade.
     - `UserDataRestore`: Responsible for restoring data from the backup in case of rollback.
   - Design versioned backup formats to handle schema migrations between versions.
   - This task can be worked on in parallel with items 1-3. It's a blocker for item 5.

5. **User Data Backup and Restore Implementation**

   - Implement the backup and restore interfaces in the following data store implementations:
     - **In-memory datastore**: [inmemory/client.go](https://github.com/radius-project/radius/blob/main/pkg/components/database/inmemory/client.go)
     - **Postgres datastore**: [postgresclient.go](https://github.com/radius-project/radius/blob/main/pkg/components/database/postgres/postgresclient.go)
   - Add comprehensive unit tests for each implementation.
   - Implement backup storage mechanism in Kubernetes (ConfigMaps or PVs depending on size).
   - This task depends on item 4 (interfaces) and blocks item 6 (CLI implementation).

6. **Upgrade Lock Mechanism**

   - Implement the upgrade lock interface to prevent concurrent modifications.
   - Update existing CLI commands to check for locks before data modification.
   - Can be implemented in parallel with items 1-5. Required for item 7.

7. **CLI Command Implementation**

   - Implement the `rad upgrade kubernetes` command, integrating all previously defined components and interfaces.
   - Ensure the command performs pre-flight checks, user data backup creation, component upgrades, rollback on failure, and post-upgrade verification.
   - Include detailed CLI output and logging for user visibility.
   - Add necessary unit and functional tests to validate command behavior.
   - This task depends on all previous tasks (1-6) and should be implemented last.

### Out of Scope for Implementation

- **Dry-run functionality**: After team discussion, the dry-run feature was explicitly excluded from this implementation.
- **Dependency upgrades**: Upgrading major versions of dependencies (Contour, Dapr, Postgres) is explicitly out of scope and should be handled separately.
- **Automatic CLI upgrades**: Updating the local Radius CLI automatically is not included; users must manually update their CLI version separately.

### Implementation Risks and Mitigations

- **Backup Reliability**: User data backup and restore mechanisms must be thoroughly tested to ensure reliability. Consider edge cases such as backup corruption or restoration failures.
- **Lock Persistence**: Ensure upgrade locks have proper timeout mechanisms to avoid permanently locked systems if a process terminates unexpectedly.

### Testing Strategy

- **Unit Tests**: Cover all new code paths, especially backup and restore logic, upgrade logic, and error handling.
- **Functional Tests**: Validate end-to-end upgrade scenarios, including successful upgrades, upgrades with custom configurations, failure scenarios, and rollback procedures.
- **Compatibility Tests**: Verify compatibility between different Radius CLI versions and control plane components.

## Open Questions

- **Rollback reliability**: What specific scenarios could cause rollback to fail, and how should we mitigate these risks?
  - Do we need rollbacks in version 1?
  - If the answer to the item above is yes, then do we need to add the implementation of the interfaces for Postgres in version 1?
- **Version skipping limits**: Should we enforce incremental upgrades for certain major version jumps, or always allow direct version skipping?
- **Upgrade notifications**: How should we notify users clearly about the CLI version mismatch after upgrading the control plane?
- **Resource constraints**: How do we handle scenarios where the cluster lacks sufficient resources to perform a rolling upgrade?

## Alternatives considered

<!--
Describe the alternative designs that were considered or should be considered.
Give a justification for why alternative approaches should be rejected if
possible.
-->

## Design Review Notes

<!--
Update this section with the decisions made during the design review meeting. This should be updated before the design is merged.
-->
