# Topic: Radius Control Plane Upgrades

- **Author**: Yetkin Timocin (@ytimocin)

## Topic Summary

The Radius control plane upgrades feature aims to streamline the process of upgrading the control plane components of the Radius platform. This includes ensuring compatibility with new versions, minimizing downtime, and providing a seamless upgrade experience for users. The feature will draw on best practices from similar projects such as Dapr, Crossplane, and Istio.

### Top level goals

- Ensure a smooth and reliable upgrade process for the Radius control plane.
- Minimize downtime and disruption during upgrades.
- Provide clear documentation and guidance for users performing upgrades.

### Non-goals (out of scope)

- Upgrading user data and application components.
- Introducing new features unrelated to the upgrade process.

## User profile and challenges

The primary users of this feature are system administrators and DevOps engineers responsible for maintaining and upgrading the Radius platform.

### User persona(s)

- **System Administrator**: Responsible for managing the infrastructure and ensuring the smooth operation of the Radius platform.
- **DevOps Engineer**: Focused on automating deployment processes and maintaining the CI/CD pipeline.

### Challenge(s) faced by the user

- Ensuring compatibility with new versions of the control plane components.
- Minimizing downtime and disruption during the upgrade process.
- Navigating complex upgrade procedures without clear documentation.

### Positive user outcome

Users will be able to upgrade the Radius control plane with minimal downtime and disruption, ensuring compatibility with new versions and maintaining the stability of their systems.

## Key scenarios

### Scenario 1: Upgrading the control plane using the Radius CLI

The Radius CLI provides a streamlined approach to upgrade the control plane components with minimal disruption. Users can execute various upgrade scenarios through the `rad upgrade kubernetes` command with appropriate flags.

```bash
# Basic upgrade to a specific version
rad upgrade kubernetes --version v0.44.0

# Upgrade with custom configuration values
rad upgrade kubernetes --version v0.44.0 --set global.monitoring.enabled=true

# Perform a dry-run to simulate the upgrade without making changes
rad upgrade kubernetes --version v0.44.0 --dry-run

# Force an upgrade to a specific version
rad upgrade kubernetes --version v0.44.0

# Basic downgrade to a specific version
# (Pending an answer to the open question of if we are going to support donwgrades)
rad upgrade kubernetes --version v0.30.0

# Force a downgrade to a previous version (with confirmation prompt)
# (Pending an answer to the open question of if we are going to support donwgrades)
rad upgrade kubernetes --version v0.30.0 --force

# Upgrade with extended timeout for large deployments
rad upgrade kubernetes --version v0.44.0 --timeout 600
```

The upgrade process will:

1. Detect the existing Radius installation and version
1. Verify compatibility between current and target versions
1. Check dependency requirements and compatibility
1. Backup configuration where applicable
1. Perform a rolling upgrade to minimize downtime
1. Apply any necessary CRD updates
1. Migrate data if required (e.g., etcd to PostgreSQL migration)
1. Validate the upgraded installation
1. Report success or failures with actionable information

For complex upgrades involving storage migrations, additional flags can be used to control the migration process and validation steps.

### Scenario 2: Upgrading the control plane using Helm

Users can upgrade the control plane components using Helm, leveraging Helm charts to manage the upgrade process.

## Key dependencies and risks

- **Dependency Name**: Radius CLI – The upgrade process relies on the Radius CLI for executing upgrade commands. Issues/concerns/risks: Compatibility with new versions of the CLI.
- **Dependency Name**: Helm – The upgrade process relies on Helm for managing the upgrade process. Issues/concerns/risks: Compatibility with new versions of Helm.
- **Dependency Name**: Contour – The upgrade process relies on Contour for routing requests. Issues/concerns/risks: Compatibility with new versions of Contour.
- **Dependency Name**: Dapr – The upgrade process relies on Dapr. Issues/concerns/risks: Compatibility with new versions of Dapr.
- **Risk Name**: Downtime during upgrades – Mitigation plan: Implement rolling upgrades and provide clear documentation to minimize downtime. This may not be that important since we are only upgrading the Radius components and not user resources.

## Key assumptions to test and questions to answer

### Core Assumptions

- **Version Compatibility:** If you are upgrading from a version of Radius that does not include the `rad upgrade kubernetes` command, you must first update to a release that provides the upgrade functionality. Older versions do not support automated upgrades in the CLI.
- **User Permissions:** Users have the necessary permissions and access to perform upgrades, including cluster-admin roles where required.
- **Data Integrity:** The upgrade process will maintain data integrity and prevent corruption during migration operations (particularly for database migrations).
- **Dependency Management:** All required dependencies (Contour, Dapr) can be successfully upgraded in the same operation or are backward compatible.
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

### Success Metrics & Validation

- **Upgrade Success Rate:** Define metrics to track successful vs. failed upgrades
- **Performance Impact:** Measure and minimize performance degradation during upgrades
- **Downtime Duration:** Establish acceptable limits for component unavailability during the process
- **Validation Testing:** Define comprehensive test cases to verify proper operation post-upgrade

### Research Plan

- Conduct user research and gather feedback to identify common issues with current manual upgrade processes
- Analyze upgrade approaches from similar platforms (Dapr, Crossplane, Istio) to adopt best practices
- Create a compatibility matrix for supported upgrade paths between versions
- Develop and test rollback procedures for various failure scenarios

## Current state

Upgrading the Radius control plane currently involves manual steps and limited documentation. These manual steps include:

1. Downloading and installing a newer version of Radius locally.
1. Running `rad uninstall kubernetes` to remove the existing Radius installation from the Kubernetes cluster.
1. Running `rad install kubernetes` to install the newly downloaded version locally.

This procedure is time-consuming and prone to error, often leading to unwanted downtime and disruptions. Improved automation and clear documentation are critical to streamline this process.

## Details of user problem

As a system administrator, I need to upgrade the Radius control plane components to ensure compatibility with new versions and maintain the stability of my systems. However, the current upgrade process is complex and prone to errors, leading to downtime and disruption.

## Desired user experience outcome

After this scenario is implemented, I can upgrade the Radius control plane components seamlessly, with minimal downtime and disruption. As a result, my systems remain stable and compatible with new versions.

### Detailed user experience

1. User initiates the upgrade process using the Radius CLI or Helm.
2. The system performs pre-upgrade checks to ensure compatibility.
3. The upgrade process is executed, with rolling upgrades to minimize downtime.
4. The system performs post-upgrade checks to verify the success of the upgrade.
5. User receives a notification confirming the successful upgrade.

## Key investments

### Feature 1

Implement pre-upgrade and post-upgrade checks to ensure compatibility and verify the success of the upgrade.

### Feature 2

Provide clear documentation and guidance for users performing upgrades using the Radius CLI and Helm.

### Feature 3

Implement rolling upgrades to minimize downtime and disruption during the upgrade process.

### Database Upgrade Process

The Radius database upgrade process involves migrating the data store from etcd to a PostgreSQL database. This change aims to improve service continuity and decouple state management from etcd. Here are the key steps involved in the upgrade process:

1. **Preparation**: Before starting the upgrade, ensure that PostgreSQL is deployed to the Kubernetes cluster. This can be done using the Helm chart for installing Radius, which now includes deploying PostgreSQL.
2. **Migration**: The data from etcd is migrated to the PostgreSQL database. This involves exporting the existing data from etcd and importing it into PostgreSQL. The migration process ensures that all necessary data is transferred accurately.
3. **Configuration**: Update the Radius configuration to point to the new PostgreSQL database. This includes updating connection strings and any other relevant settings to ensure that Radius can communicate with the PostgreSQL database.
4. **Verification**: After the migration and configuration updates, perform thorough testing to verify that the upgrade was successful. This includes checking that all data is accessible and that the system is functioning as expected.
5. **Monitoring**: Monitor the system closely after the upgrade to ensure that there are no issues. This includes checking logs, performance metrics, and any other relevant indicators to ensure that the system is stable.

### Dry-Run Upgrade Process

Implementing a dry-run option for the Radius control plane upgrades will allow users to simulate the upgrade process without making any actual changes. This helps in identifying potential issues and ensuring a smooth upgrade when executed for real. The dry-run process will include the following steps:

1. **Initiate Dry-Run**: User initiates the dry-run process using the command `rad upgrade --dry-run`.
2. **Simulate Upgrade**: The system simulates the upgrade process, performing all the steps without making any actual changes.
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
