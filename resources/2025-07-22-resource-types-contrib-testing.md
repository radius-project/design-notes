# Test Framework for Radius Resource Types and Recipes

**Author**: Shruthi Kumar (@sk593)

## Overview

The `resource-types-contrib` repository is the central location for both core (built-in) and extended (optional) Radius resource types and their associated recipes. To ensure the reliability and correctness of these contributions, we need a test framework that validates two key aspects:

1. Resource types can be registered with Radius successfully.
2. Recipes associated with those resource types are deployed correctly and output resources are validated.

This framework will support both core and extended resource types, as well as community contributions. Core and extended types will trigger additional tests using builds from the main Radius repository, while community contributions types will be validated for registration and deployment, assuming contributors have already tested them externally.

## Terms and definitions

- **Core Resource Types**: Built-in resource types provided by Radius, including `containers`, `gateways`, `secretStores`, `volumes`, `environments`, and `applications`.
- **Optional Resource Types**: Community-contributed or extended resource types that are not part of the core Radius distribution.
- **Recipe**:A set of instructions that provisions a Radius resource to a Radius environment. Recipes are implemented in Bicep or Terraform.
- **Registration**: The process of making a resource type known to Radius so it can be used in recipes and deployments.

## Objectives

> **Issue Reference:** _TODO_

### Goals

- Validate that all resource types in the repository can be registered with Radius.
- Ensure that recipes associated with each resource type can be deployed successfully.
- For core and extended resource types, trigger functional tests in the main Radius repository after successful deployment.

### Non goals

- We will not maintain functional testing on our own for community contributed resource types within this repository. See the below options for discussion on requiring functional tests for community contributions. 

### User scenarios

#### User story 1

As a Radius maintainer, I want to define a core or extended resource type and recipe in the `resource-types-contrib` repo and have it automatically tested for registration and deployment, with functional tests triggered in the main Radius repo.

#### User story 2

As a community contributor, I want to submit a new resource type and recipe and have the CI validate that it registers and deploys correctly.

## Design

### High Level Design

The test framework will be integrated into the CI pipeline of the `resource-types-contrib` repository. It will:

- Detect new or modified resource types and recipes.
- Attempt to register each resource type with a local Radius instance.
- Deploy the associated recipe using Radius CLI.
- For core and extended types, trigger a functional test run in the main Radius repository.

### Detailed Design

#### Registration Validation

- Each resource type will be registered using the Radius CLI command `rad resource-type create`.
- The framework will validate successful registration by verifying the output of `rad resource-type list`.

#### Recipe Deployment Validation
- Recipes will be deployed using the Radius CLI on a test environment.
- Deployment success will be validated by checking resource status and CLI output. There are two options for this. See the open questions section for discussion on whether we should split cloud/noncloud recipe verification. 

#### Core and Extended (optional) Resource Type Handling
- If a resource type is core or extended, the test framework will trigger a functional test run in the Radius repository. These are the steps that need to happen when a functional test run is kicked off: 
1. We need to register the new version of the resource type using `rad resource-type register` 
2. We need to publish the test version of the resource types to an extension so the functional tests can run using the test version of the types 
3. Once 1/2 are done, we run the functional tests in the main Radius repo against the resource types from the `resource-types-contrib` PR or main branch.  

#### Community Contribution Resource Type Handling

There are 2 options for how we handle community contributions 

Option 1: No additional testing after Recipe deployment. 

Option 2: Require functional tests in the Radius repository and still trigger a functional test run when changes are made in the PR. 

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
- The functional tests today have validation for output resources. The `resource-types-contrib` repository will validate that the deployment succeeds but do we want to keep verification of specific output resources? 

## Alternatives Considered

## Design Review Notes
_To be updated after design review._