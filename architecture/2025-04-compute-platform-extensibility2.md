# Radius Compute Platform Extensibility

* **Author**: Brooke Hamilton (@brooke-hamilton)

## Overview

This design describes how Radius will support deploying applications to new compute platforms in addition to Kubernetes. Radius will enable users to deploy and manage their applications on various compute environments, providing greater flexibility and choice.

A pluggable service architecture will be implemented for extending Radius to new compute platorms. New platforms can be deployed, registered, upgraded, and removed independently from the Radius installation. Radius does not need compile-time knowlege of such platforms.

To achieve this, we will use an Open API specification written in TypeSpec to define a set of interfaces that must be implemented by compute platforms. Compute platforms will implement the Open API interfaces and be deployed in their own container(s). A registration process will allow adding and removing extensions, and discovery will be handled by Dapr.

We will re-implement the Kubernetes platform using these interfaces to ensure consistency and compatibility.

## Terms and definitions

* Compute Platform: An environment where applications can be deployed and run, such as Kubernetes, Azure Container Instances (ACI), etc.
* TypeSpec: A specification language for defining APIs and data models.
* Open API: A standard for defining RESTful APIs. TypeSpec is already used by Radius to generate Open API specifications.
* Radius compute platform extension: a plugin that allows Radius to deploy core resource types to a specific platform.

## Principles

* API Versions should be strongly typed and support versioning.
* Implementations should be independently upgradable in a runtime environment.
* The logic should be isolated and over a network protocol.

### Goals

The scope of this design is constrained to the following:

* Provide platform owners the ability to create their own implementation of a specified set of services that will allow Radius to deploy to their platform.
* Each platform will be able to deploy the [core](https://docs.radapp.io/reference/resource-schema/core-schema/) Radius resource types: application, container, gateway, extender, environment, and secret store.
* Connections between core resource types and portable resource types will be supported by extensions.

### Non goals

* The design of the implementation for any other platform (This design covers the interfaces that such a platform would have to implement.)
* Running the Radius control plane on a non-Kubernetes platform
* Support for user-defined types is deferred until that feature is fully implemented.

### User scenarios

#### User story 1

As the owner of a compute platform, I want to add support for Radius deployments to my platform so that my platform will have an application model.

#### User story 2

As the owner of a compute platform, I have access to an example implementation written in Go that I can use to help me understand how to implement my platform integration with Radius.

## Design

### High Level Design

Interfaces will be defined using TypeSpec (compiled to Open API) that compute platforms must implement using REST. These interfaces will include methods for deploying and managing the core Radius resource types. The Radius control plane will interact with these interfaces over https.

Each platform extension will run in its own container(s) within the same Kubernetes cluster as Radius. The specific number of containers and their deployment configuration is determined by the extension author.

Service invocation to and from extensions will be handled through Dapr.

Registration of new extensions will be handled by a new registration service hosted by Radius.

A CLI command will be added that allows extensions to be listed, registered, and un-registered. Extensions can also register themselves by calling a service on Radius.

### Architecture Diagrams
<!--
Provide a diagram of the system architecture, illustrating how different
components interact with each other in the context of this proposal.

Include separate high level architecture diagram and component specific diagrams, wherever appropriate.

#### Current State


#### Future State


### Detailed Design
-->

<!--
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

### Error Handling

* Deployment errors will be reported to Radius and be available in the output of the Radius CLI and the logs generated by the Radius control plane.

## Test plan

* All existing functional tests pass for Radius deployments.
* New tests are added for each extension platform that cover all the supported resource types for that platform.

## Security

Workload identity will be used to authenticate service registration in Radius and REST calls from Radius to extension endpoints.

## Compatibility

All extension platforms will be able to deploy the [core schema resource](https://docs.radapp.io/reference/resource-schema/core-schema/) types.

Note: Azure Key Vault is in the core resource types of the Radius documentation, but for the purposes of this design it is not included in the supported types for extensions because it is specific to Azure and only runs on Azure. No changes will be made to KeyVault deployments.

## Monitoring and Logging

The customer experience for monitoring and logging will not change: logs will continue to be emitted by Radius. However, each platform extension will emit its own logs.
<!--
Include the list of instrumentation such as metric, log, and trace to 
diagnose this new feature. It also describes how to troubleshoot this feature
with the instrumentation. 
-->

## Development plan

Major milestones would include:

* Implement the TypeSpec definition and code generators for the Open API specification.
* Refactor the Kubernetes platform to implement the specification.
* Implement one other platform.

The second two milestones could be done in parallel.

## Open Questions

* Should we use Open API version 2 or 3?
* Will platform extension containers run in the Radius namespace or in their own namespace?
* Do we require extensions to run in the Radius Kubernetes cluster, or could they run anywhere?
* Will workload identity be required and implemented in Radius, or is that a deployment configuration in Kubernetes handled by the platform engineers. What if the extension is not running in Kubernetes?

## Alternatives considered

We considered using Go interfaces to define the extension point. In that scenario each extension would implemented as a module and would be compiled into the main Radius binaries. The Radius team would invite public contributors to add additional compute platforms, but they would control contributions through the established open-source contributing process.

We decided against this option because it would not align with the design principles listed in this document; each change to a compute platform would require a full Radius release, implementations would not be independently upgradable in a runtime environment, and implementations would not be isolated over a network protocol.

## Design Review Notes

<!--
Update this section with the decisions made during the design review meeting. This should be updated before the design is merged.
-->