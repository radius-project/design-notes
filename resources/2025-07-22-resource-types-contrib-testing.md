# Test Framework for Radius Resource Types and Recipes

**Author**: Shruthi Kumar (@sk593)

## Overview

The `resource-types-contrib` repository is the central location for Radius resource types and their associated recipes. To ensure the reliability and correctness of these contributions, we need a test framework that validates two key aspects:

1. Resource types can be registered with Radius successfully.
2. Recipes associated with those resource types are deployed correctly and output resources are validated.

This framework will support both core and extended resource types, as well as community contributions. Core and extended types will trigger additional tests using builds from the main Radius repository, while community contributions types will be validated for registration and deployment, assuming contributors have already tested them externally.

## Terms and definitions

- **Radius Resource Types**: Built-in resource types provided by Radius, including `containers`, `gateways`, `secretStores`, `redisCaches`, `mongoDB`, etc.
- **Community Contributed Resource Types**: Community-contributed resource types that are not part of the core Radius distribution.
- **Recipe**:A set of instructions that provisions a Radius resource to a Radius environment. Recipes are implemented in Bicep or Terraform.
- **Resource Type Creation**: The process of creating a resource type in Radius so it can be used in recipes and deployments.
- **Output resources**: Resources that are defined within a Recipe. These are resources that Radius keeps track of but does not maintain the lifecycle. 

## Objectives

> **Issue Reference:** _TODO_

### Goals

- Validate that all resource types in the repository can be registered with Radius.
- Ensure that recipes associated with each resource type can be deployed successfully.
- For all resource types, trigger functional tests in the main Radius repository after successful deployment.
- The testing framework should be comprehensive enough such that it allows for integration with core Radius in each release. 

### Non goals

- Contribution guidelines. These are covered in this [folder](https://github.com/radius-project/resource-types-contrib/blob/main/CONTRIBUTING.MD)
- We will not maintain functional testing on our own for community contributed resource types within this repository. We expect the author to write tests and maintain the resource type.

### User scenarios

#### User story 1

As a Radius maintainer, I want to define a Radius resource type and recipe in the `resource-types-contrib` repo and have it automatically tested for registration and deployment, with functional tests triggered in the main Radius repo.

#### User story 2

As a community contributor, I want to submit a new resource type and recipe and have the CI validate that it registers and deploys correctly. The exact steps needed to do this are outlined in the [contribution guidelines](https://github.com/radius-project/resource-types-contrib/blob/main/CONTRIBUTING.MD). 

## Design

### High Level Design

The test framework will be integrated into the CI pipeline of the `resource-types-contrib` repository. It will:

- Detect new or modified resource types and recipes.
- Attempt to register each resource type with a local Radius instance.
- Deploy the associated recipe using Radius CLI.
- For all resource types, trigger a functional test run in the main Radius repository. Community contributed resource types will be expected to write their own functional tests in the Radius repository. 

### Detailed Design

#### Creation Validation

- Each resource type will be created using the Radius CLI command `rad resource-type create`.
- The framework will validate successful creation by verifying the output of `rad resource-type list`.
- The framework will validate the properties in Radius match the input resource type schema via `rad resource-type show`.

#### Recipe Deployment Validation
- Recipes will be deployed using the Radius CLI on a test environment.
- Deployment success will be validated by checking resource status and CLI output. There are two options for this. See the open questions section for discussion on whether we should split cloud/noncloud recipe verification. 

#### Resource Type Functional Tests 
- For any PRs opened in the repo or on merges to main, the test framework will trigger a functional test run in the Radius repository. These are the steps that need to happen when a functional test run is kicked off: 
1. We need to create the new version of the resource type using `rad resource-type create` 
2. We need to publish the test version of the resource types to an extension so the functional tests can run using the test version of the types 
3. Once 1/2 are done, we run the functional tests in the main Radius repo against the resource types from the `resource-types-contrib` PR or main branch.  

#### Release Process
On each Radius release we cut and publish a matching version of the Radius resource types from the `resource-types-contrib` repo. `rad init` will use the release version from this repository when creating Radius resource types.

#### Advantages

- Ensures consistent validation across all contributions.
- Automates testing for core types while keeping optional contributions lightweight.
- Scales well with community growth.

#### Disadvantages
- Relies on contributors to test recipes thoroughly.
- Requires coordination between `resource-types-contrib` and main Radius repo for core type testing.

#### Proposed Option

Implement a test framework that performs registration and deployment validation for all resource types, with conditional triggering of functional tests for core and extended types.

### API Design (if applicable)

**N/A**

### CLI Design (if applicable)

- Use existing Radius CLI commands:
  - `rad resource-type create <resource type>`
  - `rad deploy <recipe>`
- No new CLI commands are introduced.

### Implementation Details
#### Core RP 
With core types moving to Radius resource types, we eliminate the need for core resource functional tests. The business logic is moved to recipes for these core types, so testing will validate whether the recipe deploys the correct output resources. 

#### Portable Resources / Recipes RP (if applicable)

- All recipes will be deployed and validated using the Recipes RP.

### Error Handling

- If registration fails, the framework will log the error and mark the test as failed.
- If deployment fails, logs and CLI output will be captured for debugging.
- For core types, failure to trigger functional tests will be logged and surfaced in CI.

## Test Plan

- Validate registration of each resource type.
- Deploy each recipe and confirm success.
- For core types:
  - Confirm functional tests are triggered based off the main branch in the Radius repo.
- CI will run these validations on every PR and commit.

## Security

- No new security challenges introduced.
- Framework will use existing Radius authentication and CLI mechanisms.

## Compatibility (optional)

- No breaking changes expected. Any testing will be additional to what's already covered in the functional tests. 
- Recipes and resource types must be compatible with the current Radius CLI and runtime.

## Monitoring and Logging

- CI logs will capture registration and deployment results.
- Errors will be surfaced in GitHub Actions.

## Development Plan
- **Phase 1**: Implement registration and deployment validation.
- **Phase 2**: Add core type detection and GitHub event emission.
- **Phase 3**: Integrate with Radius repo to trigger functional tests.

## Open Questions

- In theory, we'll have recipes that target both cloud and noncloud resources. Do we want to deploy recipes for all scenarios or should be cover a minimum subset of test cases? 
Note: Yes, Radius supports three target platforms: Kubernetes, Azure, and AWS using two IaC systems: Terraform and Bicep. We need recipes for each combination.

- The functional tests today have validation for output resources. The `resource-types-contrib` repository will validate that the deployment succeeds but do we want to keep verification of specific output resources? 
Note: Yes, we want to maintain the same level of testing as all types and recipes destined for this repo are meant to be "Radius certified."

## Alternatives Considered

We considered doing no additional testing after Recipe deployment for community contributed resource types. However, we want the resource types in this repo to be held to the same bar as Radius-maintained resource types. So we'll require functional tests to be added to the Radius repository to maintain that bar. 

## Design Review Notes
Notes added in line for open questions