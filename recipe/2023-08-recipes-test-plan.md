# Recipes Test Plan

- **Author**: `Ryan Nowak (@rynowak)`
- **Approver**: `Karishma Chawla (@kachawla)`

## Overview

This document defines an overall test plan for Terraform recipes. We're creating a test plan for this now because we're executing on Terraform recipes in a highly parallel well. Getting a shared understanding of the test strategy will help individuals build the right tests by doing some _big picture thinking_ upfront.

**As such, this document may not 100% follow our typical design document format, but this is still the correct home for it.**

We've also had a lot of churn in our design practices for tests and recently done some thinking about concepts like:

- Open-box vs Closed-box analysis
- The testing pyramid
- Defining equivalence classes

This will be an opportunity to put these ideas into practice.

## Terms and definitions

- No new terminology defined here. This document will use our existing terminology definitions for Recipes, Terraform, and general testing. Clarifications will be added as needed.

## Objectives

- Define a test plan that can be leveraged to guide functional test additions as part of the definition of done for each feature.

### Goals

In Scope: We're writing this test plan because we're working on Terraform, but you may notice the title says "Recipes" not "Terraform Recipes". Bicep or other kinds of Recipes are in scope, we'll create issues for any gaps we discover and prioritize them alongside all other work.

### Non goals

Out of scope: Features of recipes that we have not yet defined or scheduled are out of scope. eg: stickiness, versioning, shared resources, etc.

Out of Scope: We're not going to spend a lot of time talking about unit tests here. We have established practices for unit-tests and generally we follow them. Our overall code-coverage of Radius is very high. A test plan can usually include unit tests, but it doesn't seem the best use of our time.

Out of Scope: Testing exhaustively all recipes or all types of cloud resources. By-design recipes do not have any logic specific to any type of cloud resource. We will be defining a strategy for testing our curated recipes as part of the recipes repo, separate from this plan.

Out of Scope: Non-functional testing - reliability, resource consumption, security, performance, etc. We're already doing long-haul testing that covers resource consumption and reliability _based on our functional tests_. We're also adding metrics to Recipes that will cover some of the performance basics. This means that we're getting a lot of non-functional testing for "free" every time we add a functional test. For now we're not going to design _separate_ non-functional tests, we'll rely on the functional tests we're writing.

### User scenarios (optional)

#### Defining a recipe

As a operator I define and curate the set of recipes that developers in my organization rely on.

This means that the following operator experiences are in scope:

- Defining recipes
- Publishing recipes
- Registering recipes as part of the environment
- Configuring cloud providers as part of the environment

#### Consuming a recipe

As a developer I can leverage recipes to create the resources my application needs.

This means that the following developer experiences are in scope:

- Deploying applications that use recipes
  - With parameters and recipe names
  - With default configuration

## Design

**We're not actually build features here, we're defining tests. So this is going to contain some of our analysis from the discussion of testing.**

### Strategy

We define the following equivalence classes for testing of Recipe `PUT/DELETE` operations:

- Tasks that are part of the engine, and apply to all resources and drivers.
- Tasks that are part of the driver, but are consistent for all resources.
- Tasks that are required for each cloud provider (or Kubernetes) in use, usually driver specific, but are consistent for all resources.

Therefore we can think of the following buckets of tests:

- Tests for the recipe engine as a whole (driver and resource being tested does not matter).
- Tests for each recipe driver (resource being tested does not matter).
  - A set of tests for common error cases and behaviors that are duplicated per-driver.
  - A set of tests unique to the driver.
- Tests for each cloud provider X each recipe driver (resource being tested does not matter)
  - As set of tests for the common error cases and behaviors that are duplicated per-cloud-provider per-driver.
  - (If needed) A set of tests unique to the cloud-provider X driver combination.

Note that in all of these cases we're agnostic to the cloud resources being created by the recipe. By-design recipes do not have any logic specific to any type of cloud resource. We will be defining a strategy for testing the recipes themselves as part of the recipes repo.

We will additionally define tests for non-deployment aspects of recipes using the same framework:

- `rad recipe show`

The configuration of cloud providers, and registration of recipes is tested by the deployment (`PUT/DELETE`) testing.

---

We can assert the following big decisions in addition to the matrix above:

- We will have E2E tests for each resource type that supports recipes (previously agreed and already implemented). This covers the success path of each resource type, and ensures that the user's desired outcomes are met. Therefore we can focus on functional behaviors and negative cases with this suite of tests.
- We should test most features using `extender` as it is the most generic resource type and unlikely to change.
- We should only use cloud resources when they are necessary for the test being written (a small number of tests).
- We should avoid unnecessary resource deployments in negative tests.

### Analysis

**These are my notes from the brainstorm we had in a previous meeting to serve as a record of what we covered.**

- I'm leaning on the people who know Recipes for the analysis
- I'm leaning on the people who don't know Recipes to ask good questions

Think about how recipes work. Make a mental list of the tasks that our code performs when executing a Recipe.

- Publishing recipes (Bicep only)
- Registering recipes as part of the environment
- Retrieving the set of declared parameters on the template (`rad recipe show`)

PUT operation:

- Retrieving recipe metadata and configuration from the environment (in the engine)
  - Metadata: which template, which driver, and parameters
  - Configuration: cloud provider configuration
- Invoking the driver: (engine)
  - Installing Terraform (in the driver, specific to Terraform, handled by `hc-install`)
  - Downloading the recipe (in the driver)
    - This supports multiple sources
      - For Bicep we support any OCI registry, but we only test ACR (we use the ORAS library)
      - For Terraform we support any Terraform source, but we only test HTTP archive format (tf get, handled by tf-exec)
  - Downloading providers (in the driver, specific to Terraform, handled by tf-exec)
  - Generating inputs to the template (in the driver)
    - Merging parameters
    - Build the context parameter
    - Configure cloud providers with scope and credentials
    - Configure state storage (in the driver, specific to Terraform)
  - Executing the template (in the driver)
    - Bicep: PUT on a deployment resource
    - Terraform: Calling tf apply (handled by tf-exec)
  - Processing template outputs (in the driver)
    - Values
    - Secrets
    - Resource IDs (implicit resources vs explicit resource)
    - Connections to resources (implementation in container RP, tested in our E2E tests)
    - Bicep: reading results of deployment
    - Terraform: using tf show to output the state (handled by tf-exec)

Delete operation:

- Delete controller calls the delete function on the engine
  - Retrieving recipe metadata and configuration from the environment (in the engine, same as PUT)
  - Delete function of driver (in the driver)
    - Bicep: delete resources using resources (loop over output resources)
      - Complex and underverified
    - Terraform: using tf destroy (handled by tf-exec)

The Recipe implementation uses as "ports and adapters" design. There are multiple kinds of recipes (drivers). eg: Bicep, Terraform, Helm, etc. Which tasks are general and which are specific to each driver? We're using external dependencies like Bicep and Terraform, what are we responsible for and not responsible for?

**Hint: we're always responsible for error handling**

Cloud providers also use a "ports and adapters" design. What tasks do we have to perform for each cloud provider? What are we responsible for and not responsible for?

- Configuring the scope and credentials as part of the deployment (either Bicep or tf apply)
  - Some error handling for misconfiguration?
  - Missing cloud provider configuration?
  - What if UCP is behaving badly?
- Deletion
  - For Bicep deletion we all the cloud APIs directly, which means we need to handle errors and potentially retries

A few good equivalence classes emerge:

- Tasks that are part of the engine, and apply to all resources and drivers
- Tasks that are part of the driver, but are consistent for all resources
- Tasks that are required for each cloud provider (or Kubernetes) in use, usually driver specific, but are consistent for all resources

**Reminder: it's not our goal to test all combinations of these classes. It's our goal to be efficient and get a lot of value from a small number of tests.**

### API design (if applicable)

n/a

## Alternatives considered

We always have the alternative of **not** writing a test plan.

## Test plan

### Deployment/Deletion tests for each resource type that supports recipes (already implemented)

- E2E test for 'recipe' scenario
- E2E test for 'manual' scenario
- Negative API validation of each resource type is sufficiently covered by unit tests:
  - 'recipe' scenario
  - 'manual' scenario

### Deployment/Deletion tests for "all drivers" functionality (recipe engine)

Analysis: There's very little logic that's truly shared for deployment and deletion.

- Negative: Attempt to use a recipe that's not registered.
- Negative: Attempt to use an environment that does not exist.
- Positive: verify that the non-cloud provider configuration is generated correctly. Cloud provider configuration will have it's own tests.

### Deployment/Deletion tests for "each driver" functionality (eg: Bicep, Terraform)

- Deployment/Execution Tests
  - Terraform Driver
    - Positive: Test for successful Terraform initialization and application of the existing Terraform recipe
    - Positive: Test for the Kubernetes secret creation if applicable
    - Negative: Test for deploying a non-existing/removed Terraform module
  - Bicep Driver
    - Positive: Test for successful deployment of a Bicep recipe
    - Positive: Test for successful garbage collection of resources that are no longer in use
    - Negative: Test for deploying a non-existing/removed Bicep recipe
- Deletion Tests
  - Terraform Driver
    - Positive: Test for successful Terraform initialization and destruction of the existing resources provided by the Terraform recipe
    - Positive: Test for the Kubernetes secret deletion if applicable
    - Negative: Test for deleting a non-existing/removed Terraform module
  - Bicep Driver
    - Positive: Test for successful deletion of resources associated with a Bicep recipe
    - Positive: Testing successful deletion of orphaned resources
    - Negative: Test for deletion of a non-existing/removed Bicep recipe

When filling this out, focus on:

- Positive: P1 features not covered on main success path
- Positive: Anything 'tricky'
- Negative: External failure cases (deployment fails)
- Negative: Authoring mistakes (where can the user get it wrong, and how do they know)
- Negative: Misconfiguration (how do we deal with invalid data)

### Deployment/Deletion tests for "each provider" x "each driver" functionality (eg: Bicep with Azure, Terraform with AWS, etc)

Recipe Deployment Tests

- Recipe Template Kind : Bicep
  - Azure
    - Positive: Test for deploying azure resource.
    - Negative: Test for deploying azure resource without configuring Azure provider.
  - AWS
    - Positive: Test for deploying AWS resource.
  - Kubernetes
    - Positive: Test for deploying kubernetes resource.
- Recipe Template Kind : Terraform
  - Azure
    - Positive: Test for deploying azure resource and verifying terraform created kubernetes secret.
  - AWS
    - Positive: Test for deploying AWS resource.
    - Negative: Test with no provider information in the terraform module.
  - Kubernetes
    - Positive: Test for deploying azure recipe.

Recipe Deletion Tests

- Recipe Template Kind : Bicep
  - Azure
    - Positive: Test for deleting azure resource.
  - AWS
    - Positive: Test for deleting AWS resource.
  - Kubernetes
    - Positive: Test for deleting kubernetes resource.
- Recipe Template Kind : Terraform
  - Azure
    - Positive: Test for deleting azure resource.
  - AWS
    - Positive: Test for deleting AWS resource verifying deletion of kubernetes secret.
    - Negative: Test for deleting AWS resource without configuring AWS provider.
  - Kubernetes
    - Positive: Test for deleting kubernetes recipe.

### rad recipe CLI commands

- For list, register, unregister:
  - E2E test for positive scenario
- For show:
  - E2E test for 'bicep' positive scenario
  - E2E test for 'terraform' positive scenario
- Negative testing (negative cases are primarily handled with unit testing and we can establish equivalency to the positive cases)
  - Invalid Recipe property types (template kind, version, and path)
  - Invalid portable resource type
  - Invalid recipe name
  - Invalid registry or module paths

## Security

n/a

## Compatibility (optional)

n/a

## Monitoring

n/a

## Development plan

Since we're writing a test plan, there are two categories of work being described here:

- Gaps in tests for existing features.
- New tests for features currently being built.

Test gaps we identify will

## Open issues

None yet.
