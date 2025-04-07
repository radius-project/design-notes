# Compute Platform Extensibility

* **Author**: Brooke Hamilton (@Brooke-Hamilton)

## Overview

Radius will enable users to deploy and manage their applications to multiple compute platforms, in addition to Kubernetes, by adding support for recipes to the core resource types. Recipes will enable the Radius team to provide default implementations for multiple platforms, and allow customers to modify or provide their own deployment recipes.

Modifications to Radius include:

* The [core resource types](https://docs.radapp.io/reference/resource-schema/core-schema/) of application, container, gateway, and secret store will support recipes.
* Radius will provide default recipes for Azure Container Instances (ACI).
* Customers can replace or modify default recipes, or create recipes for new platforms.
* Existing properties on the environment core type will be used to provide Bicep or Terraform environment configuration.

> NOTE: This design builds upon and extends the [Serverless Container Runtimes](https://github.com/radius-project/design-notes/pull/77) feature spec.

## Terms and definitions

* Compute platform: An environment where applications can be deployed and run, such as Kubernetes, Azure Container Instances (ACI), etc.

### Goals

* Provide platform owners with the ability to deploy to specific platforms other than Radius, like ACI.
* Enable platform owners to extend Radius to deploy to any platform through custom recipes.
* Provide a recipe-based platform engineering experience that is consistent for user-defined types and core types.

### Non-goals

* Running the Radius control plane on a non-Kubernetes platform

## Principles

These are the design principles that apply to the customer experience of extending Radius to support multiple compute platforms.

Extensions should be:

* **Independently upgradeable in a runtime environment**: recipes are independently upgradeable.
* **Isolated and over a network protocol**: recipes are currently implemented through Bicep and Terraform. Both run locally and communicate to the target platform over network protocols. We could separate recipe execution to its own container in Radius (but that is not currently planned).
* **Strongly typed and support versioning**: recipes support versioning through the use of OCI-compliant registries. They are not strongly types in the sense of having compile-time validation, but they can be validated at the time they are registered as having the correct input parameters and output properties.

### User scenarios

#### Configure a non-Kubernetes Radius environment

As a platform engineer I can initialize a new Radius environment that is connected to a non-Kubernetes compute platform so that Radius can authenticate and deploy to non-Kubernetes platforms.

The ability to set a recipe on the environment will set up the integration with the compute platform.

Changes include:

* `kind` can be any string that identifies a platform. If set to `kubernetes` the existing logic will stay in place.
* `namespace` will only be required for "kubernetes" compute kind. It will be deprecated or removed if we convert the Kubernetes deployments to a recipe.
* `recipe` will be added, which allows configuring default values on all recipes that run in this environment. Platform-specific required parameters like subscription ID and resource group can be added here and provided to all recipes that run for this environment. `recipe` is not allowed if the `kind` is `kubernetes`.

```diff
resource environment 'Applications.Core/environments@2023-10-01-preview' = {
  name: 'myenv'
  properties: {
    compute: {
+      kind: 'aci'   // Allow any name. This is a descriptive label only unless set to 'kubernetes'.
      namespace: 'default' // Only required for Kubernetes compute kind.
      identity: {          // Optional. External identity providers to use for connections
        kind: 'azure.com.workload'
        oidcIssuer: oidcIssuer
      }
    }
+   recipe: {
+      // Name a specific Recipe to use
+      name: 'azure-aci-environment'
+      // Set parameters that will be passed to every recipe in this environment unless overridden in the resource definitions. Platform-specific configuration goes here so taht it can be passed to each recipe.
+      parameters: {
+        subscriptionId: 'xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx'
+        resourceGroup: 'myrg'
+        size: 'large'
+      }
+    }
   extensions: [ // Can be removed if we convert the current Kubernetes deployment to use recipes
      {
        kind: 'kubernetesMetadata'
        labels: {
          'team.contact.name': 'frontend'
        }
      }
    ]
  }
}
```

> NOTE: Specifying a recipe in the Bicep above is optional. If there are multiple environments, e.g., development, staging, and production, a team may want to configure recipes with the `rad recipe register` command that is specific to each environment. That would allow setting parameters for each recipe in each environment, or have separate recipes for each environment.

#### Initialize a workspace

As a platform engineer I can use `rad init` (or the equivalent rad `workspace`, `group` and `environment` commands), plus `rad recipe register`, to set up a non-Kubernetes compute platform.

`rad init` would be unchanged. Platform engineers would have to add recipes for each core type and UDT they plan to use.

We could consider adding a flag to `rad init` that would identify specific compute platforms and provide the registration of default recipes for that platform, based on a pre-defined set of compute type names. For example, `rad init aci` would install a set of default recipes for the ACI platform. However, this work is not in scope for this design.

#### Extend Radius to a new platform by creating new recipes

The `rad recipe register` CLI command would be unchanged, as it already has the ability to associate a recipe to an environment and a resource type, and it supports setting default values for specified recipe parameters for that environment.

```shell
rad recipe register <recipe name> \
  --environment <environment name> \
  --resource-type <core or UDT resource type name> \
  --parameters throughput=400
```

#### Set recipes on core types

As a platform engineer I can set recipes on the core resource types of application, container, gateway, and secret store so that I can configure my deployments for an environment.

```diff
resource app 'Applications.Core/applications@2023-10-01-preview' = {
  name: 'myapp'
  properties: {
    environment: environment
+   recipe: {
+      // Name a specific Recipe to use
+      name: 'azure-aci-application'
+      // Set parameters that will be passed to every recipe in this environment unless overridden in the resource definitions. Platform-specific configuration goes here so taht it can be passed to each recipe.
+      parameters: {
+        subscriptionId: 'xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx'
+        resourceGroup: 'myrg'
+      }
+    }
    extensions: [ // Extensions are only useful for Kubernetes deployments. Can be removed if we convert the current Kubernetes deployment to recipes.
      {
        kind: 'kubernetesNamespace'
        namespace: 'myapp'
      }
      {
        kind: 'kubernetesMetadata'
        labels: {
          'team.contact.name': 'frontend'
        }
      }
    ]
  }
}
```

<!--
## Design

### High Level Design
High level overview of the data flow and key components.

Provide a high-level description, using diagrams as appropriate, and top-level
explanations to convey the architectural/design overview. Don’t go into a lot
of details yet but provide enough information about the relationship between
these components and other components. Call out or highlight new components
that are not part of this feature (dependencies). This diagram generally
treats the components as black boxes. Provide a pointer to a more detailed
design document, if one exists. 
-->

<!--
### Architecture Diagram
Provide a diagram of the system architecture, illustrating how different
components interact with each other in the context of this proposal.

Include separate high level architecture diagram and component specific diagrams, wherever appropriate.
-->

<!--
### Detailed Design

This section should be detailed and thorough enough that another developer
could implement your design and provide enough detail to get a high confidence
estimate of the cost to implement the feature but isn’t as detailed as the 
code. Be sure to also consider testability in your design.

For each change, give each "change" in the proposal its own section and
describe it in enough detail that someone else could implement it. Cover
ALL of the important decisions like names. Your goal is to get an agreement
to proceed with coding and PRs.

If there are alternatives you are considering please include that in the open
questions section. If the product has a layered architecture, it's good to
align these sections with the product's layers. This will help readers use
their current understanding to understand your ideas.

Discuss the rationale behind architectural choices and alternative options 
considered during the design process.
-->

<!--
#### Advantages (of each option considered)
Describe what's good about this plan relative to other options. 
Provides better user experience? Does it feel easy to implement? 
Provides flexibility for future work?
-->

<!--
#### Disadvantages (of each option considered)
Describe what's not ideal about this plan. Does it lock us into a 
particular design for future changes or is it flexible if we were to 
pivot in the future. This is a good place to cover risks.
-->

<!--
#### Proposed Option
Describe the recommended option and provide reasoning behind it.
-->

<!--
### API design (if applicable)

Include if applicable – any design that changes our public REST API, CLI
arguments/commands, or Go APIs for shared components should provide this
section. Write N/A here if not applicable.
- Describe the REST APIs in detail for new resource types or updates to
  existing resource types. E.g. API Path and Sample request and response.
- Describe new commands in the CLI or changes to existing CLI commands.
- Describe the new or modified Go APIs for any shared components.
-->

<!--
### CLI Design (if applicable)
Include if applicable – any design that changes Radius CLI
arguments/commands. Write N/A here if not applicable.
- Describe new commands in the CLI or changes to existing CLI commands.
-->

<!--
### Implementation Details
High level description of updates to each component. Provide information on 
the specific sub-components that will be updated, for example, controller, processor, renderer,
recipe engine, driver, to name a few.

#### UCP (if applicable)
#### Bicep (if applicable)
#### Deployment Engine (if applicable)
#### Core RP (if applicable)
#### Portable Resources / Recipes RP (if applicable)
-->

<!--
### Error Handling
Describe the error scenarios that may occur and the corresponding recovery/error handling and user experience.
-->

<!--
## Test plan

Include the test plan to validate the features including the areas that
need functional tests.

Describe any functionality that will create new testing challenges:
- New dependencies
- External assets that tests need to access
- Features that do I/O or change OS state and are thus hard to unit test
-->

<!--
## Security

Describe any changes to the existing security model of Radius or security 
challenges of the features. For each challenge describe the security threat 
and its mitigation with this design. 

Examples include:
- Authentication 
- Storing secrets and credentials
- Using cryptography

If this feature has no new challenges or changes to the security model
then describe how the feature will use existing security features of Radius.
-->

<!--
## Compatibility (optional)

Describe potential compatibility issues with other components, such as
incompatibility with older CLIs, and include any breaking changes to
behaviors or APIs.
-->

<!--
## Monitoring and Logging

Include the list of instrumentation such as metric, log, and trace to 
diagnose this new feature. It also describes how to troubleshoot this feature
with the instrumentation. 
-->

<!--
## Development plan

Describe how you will deliver your features. This includes aligning work items
to features, scenarios, or requirements, defining what deliverable will be
checked in at each point in the product and estimating the cost of each work
item. Don’t forget to include the Unit Test and functional test in your
estimates.
-->

## Open Questions

* Should we convert Kubernetes deployments to recipes with this work, after this work, or not at all?
* Can the environment core type be configured to set environment variables or certain parameters to be passed to all recipes? For example, set subscription ID on the environment, and have it automatically flow to each recipe through a parameter.
* Should we use the Radius group concept to manage resource groups in cloud providers?

## Alternatives considered

An alternative that we considered, but did not select, was to allow the registration of custom resource providers.

* Customers can register their own RPs for the core types (application, container, gateway, secret store). RPs may also be registered for UDTs.
* The registered RPs belong to a customer-defined environment "kind" that is declared on the Environment core type.
* Customer RPs implement a versioned OpenAPI specification generated from Radius that defines a set of REST endpoints. RPs can be written in any language, hosted on any platform, and must be managed and deployed by the customer.
* Radius routes CRUDL operations to the custom RPs.

We did not select this option because:

* The chosen option is simpler for users because having recipes on the core types makes the overall Radius user experience more consistent across all resource types.
* Recipes are simpler and lower effort for users to implement.
* Having recipes on the core types will enable most customization scenarios.

We could add the option to create custom resource types later if we see demand for more complex extension scenarios.

## Design Review Notes

<!--
Update this section with the decisions made during the design review meeting. This should be updated before the design is merged.
-->
