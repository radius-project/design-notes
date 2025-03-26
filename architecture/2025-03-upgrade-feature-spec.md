# Topic: Radius Control Plane Upgrades

- **Author**: Yetkin Timocin (@ytimocin)

## Topic Summary

The Radius in-place upgrade feature aims to streamline the process of upgrading the control plane components of Radius, ensuring minimal downtime and disruption. **Importantly, user applications deployed through Radius continue running without interruption during the upgrade process, as Radius only maintains metadata about these applications and does not manage their runtime execution.**

**Radius Components (as of Mar 2025):**

- Universal Control Plane
- Radius Deployment Engine
- Applications Resource Provider
- Dynamic Resource Provider
- Dashboard
- Controller

**Dependencies:**

- Dapr
- Contour
- Postgres

**Key Features:**

- **Compatibility:** Ensures compatibility with new versions.
- **Data Safety:** Takes snapshots before the upgrade to safeguard user data, with automatic rollback in case of failure.
- **Seamless Experience:** Provides a smooth and reliable upgrade process with minimal downtime.
- **Application Continuity:** User applications remain operational throughout the upgrade process, as Radius only manages deployment metadata, not runtime execution.

This feature will significantly improve the user experience by automating the upgrade process, reducing the risk of errors, and maintaining system stability.

### Top level goals

- Deliver a seamless and intuitive upgrade experience for users.
- Ensure a smooth and reliable upgrade process for the components and dependencies of Radius.
- Minimize downtime and disruption during upgrades, ensuring that the Radius Control Plane is available as soon as possible.
- Ensure user data safety during the upgrade process by taking snapshots and providing rollback capabilities.
- Provide clear documentation and guidance for users performing upgrades.
- Maintain system performance during the upgrade process to avoid significant impact on running applications.

### Non-goals (out of scope)

- Downgrades are not supported. The upgrade process is designed to move forward to newer versions only.
- The upgrade process does not include features for managing upgrades across multiple clusters simultaneously as Radius doesn't support multiple clusters per installation as of March 2025.
- Upgrading major versions of dependencies like Postgres, Dapr, or Contour is not covered by this process. These upgrades should be handled separately following their respective guidelines.
- Upgrading the Radius control plane using Helm. We can run `helm upgrade` on the Radius Helm installation but that is not going to put all the necessary pieces together for the control plane to work. Making this work is not in the scope of this work.

## User profile and challenges

The primary users of this feature are system administrators and DevOps engineers responsible for maintaining and upgrading the Radius platform.

### User persona(s)

- **System Administrator**: Responsible for managing the infrastructure and ensuring the smooth operation of the Radius platform.
- **DevOps Engineer**: Focused on automating deployment processes and maintaining the CI/CD pipeline.

### Challenge(s) faced by the user

- As of March 2025, Radius does not support in-place upgrades or upgrades via the Radius CLI. Users can achieve this using Helm, but this method does not update the local Radius CLI version to the desired version. The current steps are:
  - `helm install radius oci://ghcr.io/radius-project/helm-chart/radius --version 0.43.0`
  - `helm upgrade radius oci://ghcr.io/radius-project/helm-chart/radius --version 0.44.0`
- This approach is cumbersome and does not provide a good user experience. Additionally, it does not automatically install dependencies like Contour and Dapr, requiring users to handle these installations manually.
- Another way to upgrade Radius to the desired version involves the following steps:
  - Uninstall the existing Radius installation by running `rad uninstall kubernetes`. This command doesn't delete any user data or the `radius-system` namespace.
  - Download and install the desired version locally. Ensure it is the correct version by running `rad version`.
  - Run `rad install kubernetes` or `rad init` to finalize the upgrade.
- This process is also not as seamless as a `rad upgrade kubernetes --version desired.version` command could be.

### Positive user outcome

- By running the `rad upgrade kubernetes --version desired.version` command, users will be able to initiate the upgrade process seamlessly.
  - The desired version must be higher than the current version.
- By setting a global flag indicating that an upgrade is in progress, we ensure that users do not start data-changing commands like `rad deploy app.bicep` or `rad delete app` in a different terminal.
- A snapshot of Radius and user data is taken right before the upgrade starts. In case of failure, the snapshot can be restored to ensure data integrity.
- At the end of this process, users will have Radius upgraded to the desired version, ensuring compatibility and stability.

## Key scenarios

### Scenario 1: Upgrading the control plane using the Radius CLI

After the implementation of this feature, the Radius CLI will provide a streamlined approach to upgrade the control plane components with minimal disruption. Users can execute various upgrade scenarios through the `rad upgrade kubernetes` command with appropriate flags.

```bash
# Basic upgrade to a specific version
rad upgrade kubernetes --version v0.44.0

# Upgrade to the latest stable version
rad upgrade kubernetes --version latest

# Upgrade with custom configuration values
rad upgrade kubernetes --version v0.44.0 --set global.monitoring.enabled=true

# Perform a dry-run to simulate the upgrade without making changes
rad upgrade kubernetes --version v0.44.0 --dry-run

# Upgrade with extended timeout for large deployments
rad upgrade kubernetes --version v0.44.0 --timeout 600
```

The upgrade process will:

1. **Pre-flight checks**: Detect existing installation, check versions, ensure the user is not attempting a downgrade, etc.
2. **Fetch available chart versions**: Provide a list of known chart versions so the desired version that the users select is a valid one.
3. **Dry-run** (when requested): Simulate the upgrade, logging steps without making changes. Also making sure that the upgrade will work. Helm has this feature available in the `helm upgrade` command: <https://helm.sh/docs/helm/helm_upgrade/>.
4. **Snapshot**: Automatically back up current data (e.g., etcd, resources in the API server, or Postgres) before making changes.
5. **Upgrade**: Apply necessary Helm changes (including timeouts, set args, etc.), optionally perform database migrations if needed.
6. **Rollback** (on failure): If something goes wrong, use the snapshot to restore the prior state.
7. **Post-upgrade checks**: Validate that new control plane components are healthy and confirm the upgrade was successful.

### Scenario 2: Upgrading the control plane using Helm (Non-Goal)

As mentioned above, users can install Radius using Helm, but dependencies like Contour and Dapr are not installed this way. Users can also run `helm upgrade` on a Helm installation to upgrade the version of Radius in the cluster. However, this is not an ideal solution and is considered a workaround because users will need to perform additional steps to achieve a healthy installation, similar to what they would have after running `rad install kubernetes` and `rad init`. The steps are as follows:

- `helm install radius oci://ghcr.io/radius-project/helm-chart/radius --version 0.43.0`
- `helm upgrade radius oci://ghcr.io/radius-project/helm-chart/radius --version 0.44.0`

Users will still need to install Dapr and Contour and run `rad init` to achieve a similar behavior to what `rad upgrade kubernetes` would provide. I also think that the Radius CLI version needs to be updated to the desired version before running `rad init`.

## Key dependencies and risks

- **Dependency Name**: Helm – The upgrade process relies on Helm for managing the upgrade process. Issues/concerns/risks: Compatibility with new versions of Helm.
- **Dependency Name**: Contour – The upgrade process relies on Contour for routing requests. Issues/concerns/risks: Compatibility with new versions of Contour.
- **Dependency Name**: Dapr – The upgrade process relies on Dapr. Issues/concerns/risks: Compatibility with new versions of Dapr.

## Key assumptions to test and questions to answer

### Core Assumptions

- **Version Compatibility:** If you are upgrading from a version of Radius that does not include the `rad upgrade kubernetes` command, you must first update to a release that provides the upgrade functionality. Older versions do not support automated upgrades in the CLI.
- **User Permissions:** Users have the necessary permissions and access to perform upgrades, including cluster-admin roles where required.
- **Data Integrity:** The upgrade process will maintain data integrity and prevent corruption during migration operations (particularly for database migrations) by using snapshots and the ability to restore in case of failure.
- **Dependency Management:** We use a specific version of Contour and Dapr (<https://github.com/radius-project/radius/blob/main/pkg/cli/helm/cluster.go#L34>) and they can not be updated with the configuration. This ensures consistent dependency management. There is a case where the default chart versions may be updated in a newer version of Radius but that means that those versions of Contour and Dapr are probably tested with that new version of Radius and shouldn't provide problems during the upgrade from the current version and the desired version which is the newer version that we just discussed about.
- **Resource Requirements:** The upgrade process won't exceed available cluster resources during the transition period when both old and new components may be running simultaneously.

### Technical Questions to Resolve

- **Downgrade Support:**
  - Should we support downgrading to previous versions? If yes, what are the limitations?
  - How should we handle cases where users attempt to downgrade to versions that don't support the upgrade feature itself?
- **Version Skipping:**
  - Can users skip multiple versions in a single upgrade (e.g., v0.40 → v0.44), or should we enforce incremental upgrades?
- **Failure Recovery:**
  - What recovery mechanisms should be implemented if an upgrade fails mid-process?
  - How do we ensure that partial upgrades don't leave the system in an inconsistent state?
- **User Experience:**
  - What are the common issues users face during the upgrade process, and how can we address them?
  - How should progress and status updates be communicated during long-running upgrades?
- **Component Versions:**
  - Does `radius upgrade kubernetes` mean that all the components (Dapr, Contour, Postgres, etc.) will be upgraded? Will those versions be provided by Radius release or the user?

### Success Metrics & Validation

- **Upgrade Success Rate:** Define metrics to track successful vs. failed upgrades
- **Performance Impact:** Measure and minimize performance degradation during upgrades
- **Downtime Duration:** Establish acceptable limits for component unavailability during the process
- **Validation Testing:** Define comprehensive test cases to verify proper operation post-upgrade

### Research Plan

- Analyze upgrade approaches from similar platforms (Dapr, Crossplane, Istio, Flux) to adopt best practices
- Develop and test rollback procedures for various failure scenarios

## Current state

Upgrading the Radius control plane currently involves manual steps and limited documentation. These manual steps include:

1. Downloading and installing a newer version of Radius locally.
2. Running `rad uninstall kubernetes` to remove the existing Radius installation from the Kubernetes cluster.
3. Running `rad install kubernetes` to install the newly downloaded version locally.

This procedure is time-consuming and prone to error, often leading to unwanted downtime and disruptions. Improved automation and clear documentation are critical to streamline this process.

## Details of user problem

As a system administrator, I need to upgrade the Radius control plane components to ensure compatibility with new versions and maintain the stability of my systems. However, the current upgrade process is complex and prone to errors, leading to downtime and disruption.

## Desired user experience outcome

After this scenario is implemented, I can upgrade the Radius control plane components seamlessly, with minimal downtime and disruption. As a result, my systems remain stable and compatible with new versions.

### Detailed user experience

1. User initiates the upgrade process using the Radius CLI.
2. The system performs pre-upgrade checks to ensure compatibility.
3. The upgrade process is executed, with rolling upgrades to minimize downtime.
4. The system performs post-upgrade checks to verify the success of the upgrade.
5. User receives a notification confirming the successful upgrade.

## Key investments

### Feature 1

Implement pre-upgrade and post-upgrade checks to ensure compatibility and verify the success of the upgrade. These checks are:

1. **Pre-Upgrade Checks:**
   1. **Version Compatibility:** Ensure the current version is behind the desired version to prevent downgrades.
   2. **Snapshot Creation:** Automatically create a snapshot of the current data (e.g., etcd, resources in the API server, or Postgres) to ensure user data safety in case of rollback.
2. **Post-Upgrade Checks:**
   1. **Component Health:** Verify that all upgraded control plane components are up and running.

### Feature 2

Provide clear documentation and guidance for users performing upgrades using the Radius CLI.

### Dry-Run Upgrade Process

Implementing a dry-run option for the Radius control plane upgrades will allow users to simulate the upgrade process without making any actual changes. This helps in identifying potential issues and ensuring a smooth upgrade when executed for real. The dry-run process will include the following steps:

1. **Initiate Dry-Run**: User initiates the dry-run process using the command `rad upgrade --version 0.44 --dry-run`.
2. **Simulate Upgrade**: The system simulates the upgrade process, performing all the steps without making any actual changes. Helm actually has a flag that we can use for the `helm upgrade` command.
3. **Generate Report**: The system generates a report detailing the steps that would be taken during the actual upgrade, including any potential issues or conflicts.
4. **Review Report**: User reviews the report to identify and address any potential issues before proceeding with the actual upgrade.

### Plan for `rad upgrade kubernetes`

The `rad upgrade kubernetes` command will be designed to facilitate the upgrade process for the Radius control plane components. Here is the plan for implementing the `rad upgrade kubernetes` command:

1. **Command Structure**: The `rad upgrade kubernetes` command will follow a similar structure to the `rad install kubernetes` command, with additional options for performing upgrades.
2. **Pre-Upgrade Checks**: The command will perform pre-upgrade checks to ensure compatibility with the existing configuration and identify any potential issues.
3. **Dry-Run Option**: The command will include a `--dry-run` option to simulate the upgrade process without making any actual changes. This will help users identify potential issues before performing the actual upgrade.
4. **Upgrade Execution**: The command will execute the upgrade process, including updating the control plane components, applying any necessary database migrations, and updating configurations.
5. **Post-Upgrade Checks**: The command will perform post-upgrade checks to verify the success of the upgrade and ensure that the system is functioning as expected.
6. **Rollback Option**: The command will include a rollback option to revert to the previous version in case of any issues during the upgrade process.

By following this plan, the `rad upgrade kubernetes` command will provide a reliable and user-friendly way to upgrade the Radius control plane components, ensuring compatibility with new versions and minimizing downtime and disruption.
