# Radius Flux Controller Design

* **Author**: Will Smith (@willdavsmith)

## Overview

<!--
Provide a succinct high-level description of the component or feature and
where/how it fits in the big picture. The overview should be one to three
paragraphs long and should be understandable by someone outside the Radius
team. Do not provide the design details in this, section - there is a
dedicated section for that later in the document.
-->

The Radius Flux Controller is a key component of the Radius GitOps feature. It is responsible for reading the Git repository, compiling Bicep files, and creating the DeploymentTemplate resources on the cluster. This controller ensures that the infrastructure defined in the Git repository is accurately reflected in the Kubernetes cluster, enabling a seamless GitOps workflow.

## Terms and definitions

<!--
Include any terms, definitions, or acronyms that are used in
this design document to assist the reader. They may or may not
be part of the user-facing experience once implemented, and can
be specific to this design context.
-->

- **GitOps**: A set of practices to manage infrastructure and application configurations using Git.
- **Bicep**: A domain-specific language (DSL) for deploying Azure resources declaratively.
- **DeploymentTemplate**: A custom resource in Kubernetes that represents the desired state of a deployment.

## Objectives

<!--
Describe goals/non-goals and user-scenario of this feature to understand
the end-user goals.
* If the feature shares the same objectives of the existing design, link
  to the existing doc and section rather than repeat the same context.
* If the feature has a scenario, UX, or other product feature design doc,
  link it here and summarize the important parts.
-->

> **Issue Reference:** #6689

### Goals

<!--
Describe goals to define why we are doing this work, how we will make
priority decisions, and how we will determine success.
-->

- Enable seamless integration of GitOps workflows with Radius.
- Automate the process of reading, compiling, and deploying infrastructure as code (IaC) from a Git repository.
- Ensure that the infrastructure defined in the Git repository is accurately reflected in the Kubernetes cluster.

### Non goals

<!--
Describe non-goals to identify something that we won’t be focusing on
immediately. We won’t be expending any effort on these matters. If there
will be follow-ups after this work, list them here. If there are things
we plan to do in the future, but are out of scope of this design, list
them here. Provide a brief explanation on why this is a non-goal.
-->

- Implementing advanced error handling and recovery mechanisms.
- Supporting non-Bicep IaC formats.
- Providing a user interface for managing GitOps workflows.

### User scenarios (optional)

<!--
Describe the user scenarios for this design. Ensure that you define the
roles and personas in these user scenarios when it requires API design.
If you have an existing issue that describes the user scenarios, please
link to that issue instead.
-->

#### User story 1

As a DevOps engineer, I want to automate the deployment of my infrastructure using GitOps so that I can ensure consistency and reliability in my deployments.

#### User story 2

As a developer, I want to define my infrastructure as code in a Git repository and have it automatically deployed to my Kubernetes cluster.

## User Experience (if applicable)
<!--
If the change impacts the user experience, provide expected interaction
flow we aim to achieve through this proposal.

When users interact with Radius through the CLI, include sample
input commands and their corresponding output. Include a bicep/helm code
sample, if this proposal involves updates to that experience.
-->

**Sample Input:**

git push origin main

**Sample Output:**

DeploymentTemplate resources created successfully.

**Sample Recipe Contract:**

N/A

## Design

### High Level Design
<!--
High level overview of the data flow and key components.

Provide a high-level description, using diagrams as appropriate, and top-level
explanations to convey the architectural/design overview. Don’t go into a lot
of details yet but provide enough information about the relationship between
these components and other components. Call out or highlight new components
that are not part of this feature (dependencies). This diagram generally
treats the components as black boxes. Provide a pointer to a more detailed
design document, if one exists.
-->

The Radius Flux Controller reads the Git repository, compiles Bicep files, and creates DeploymentTemplate resources on the cluster. It interacts with the DeploymentTemplate controller to ensure that the desired state defined in the Git repository is accurately reflected in the Kubernetes cluster.

### Architecture Diagram
<!--
Provide a diagram of the system architecture, illustrating how different
components interact with each other in the context of this proposal.

Include separate high level architecture diagram and component specific diagrams, wherever appropriate.
-->

!Architecture Diagram

### Detailed Design

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

#### Advantages (of each option considered)
<!--
Describe what's good about this plan relative to other options.
Provides better user experience? Does it feel easy to implement?
Provides flexibility for future work?
-->

- Seamless integration with existing GitOps workflows.
- Automated deployment of infrastructure as code.
- Consistency and reliability in deployments.

#### Disadvantages (of each option considered)
<!--
Describe what's not ideal about this plan. Does it lock us into a
particular design for future changes or is it flexible if we were to
pivot in the future. This is a good place to cover risks.
-->

- Limited to Bicep IaC format.
- Requires integration with the DeploymentTemplate controller.

#### Proposed Option
<!--
Describe the recommended option and provide reasoning behind it.
-->

The proposed option is to implement the Radius Flux Controller as described, leveraging the existing DeploymentTemplate controller for managing the desired state of the infrastructure.

### API design (if applicable)

<!--
Include if applicable – any design that changes our public REST API, CLI
arguments/commands, or Go APIs for shared components should provide this
section. Write N/A here if not applicable.
- Describe the REST APIs in detail for new resource types or updates to
  existing resource types. E.g. API Path and Sample request and response.
- Describe new commands in the CLI or changes to existing CLI commands.
- Describe the new or modified Go APIs for any shared components.
-->

N/A

### CLI Design (if applicable)
<!--
Include if applicable – any design that changes Radius CLI
arguments/commands. Write N/A here if not applicable.
- Describe new commands in the CLI or changes to existing CLI commands.
-->

N/A

### Implementation Details
<!--
High level description of updates to each component. Provide information on
the specific sub-components that will be updated, for example, controller, processor, renderer,
recipe engine, driver, to name a few.
-->

- **Controller**: Implement the Radius Flux Controller to read the Git repository, compile Bicep files, and create DeploymentTemplate resources.
- **Processor**: Ensure that the compiled Bicep files are processed correctly.
- **Renderer**: Render the DeploymentTemplate resources based on the compiled Bicep files.
- **Recipe Engine**: Integrate with the existing recipe engine to manage the deployment process.

#### UCP (if applicable)
N/A

#### Bicep (if applicable)
N/A

#### Deployment Engine (if applicable)
N/A

#### Core RP (if applicable)
N/A

#### Portable Resources / Recipes RP (if applicable)
N/A

### Error Handling
<!--
Describe the error scenarios that may occur and the corresponding recovery/error handling and user experience.
-->

- **Git Repository Not Found**: Log an error and retry after a specified interval.
- **Bicep Compilation Error**: Log the error and notify the user.
- **DeploymentTemplate Creation Error**: Log the error and retry after a specified interval.

## Test plan

<!--
Include the test plan to validate the features including the areas that
need functional tests.

Describe any functionality that will create new testing challenges:
- New dependencies
- External assets that tests need to access
- Features that do I/O or change OS state and are thus hard to unit test
-->

- **Unit Tests**: Test the individual components of the Radius Flux Controller.
- **Integration Tests**: Test the integration with the Git repository and the DeploymentTemplate controller.
- **End-to-End Tests**: Test the entire GitOps workflow from pushing changes to the Git repository to deploying the infrastructure on the Kubernetes cluster.

## Security

<!--
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

The Radius Flux Controller will use existing authentication mechanisms to access the Git repository and Kubernetes cluster. Secrets and credentials will be stored securely using Kubernetes Secrets. No new cryptographic methods are introduced.

## Compatibility (optional)

<!--
Describe potential compatibility issues with other components, such as
incompatibility with older CLIs, and include any breaking changes to
behaviors or APIs.
-->

The Radius Flux Controller is compatible with existing Radius components and does not introduce any breaking changes.

## Monitoring and Logging

<!--
Include the list of instrumentation such as metric, log, and trace to
diagnose this new feature. It also describes how to troubleshoot this feature
with the instrumentation.
-->

- **Metrics**: Track the number of successful and failed deployments.
- **Logs**: Log all actions performed by the Radius Flux Controller, including errors and retries.
- **Tracing**: Implement tracing to track the flow of operations from reading the Git repository to deploying the infrastructure.

## Development plan

<!--
Describe how you will deliver your features. This includes aligning work items
to features, scenarios, or requirements, defining what deliverable will be
checked in at each point in the product and estimating the cost of each work
item. Don’t forget to include the Unit Test and functional test in your
estimates.
-->

- **Phase 1**: Implement the basic functionality of the Radius Flux Controller.
- **Phase 2**: Integrate with the DeploymentTemplate controller.
- **Phase 3**: Implement error handling and retry mechanisms.
- **Phase 4**: Write unit, integration, and end-to-end tests.
- **Phase 5**: Monitor and log the Radius Flux Controller's actions.

## Open Questions

<!--
Describe (Q&A format) the important unknowns or things you're not sure about.
Use the discussion to answer these with experts after people digest the
overall design.
-->

- How should we handle large Git repositories with many Bicep files?
- What is the best way to notify users of errors during the deployment process?

## Alternatives considered

<!--
Describe the alternative designs that were considered or should be considered.
Give a justification for why alternative approaches should be rejected if
possible.
-->

- **Alternative 1**: Use a different IaC format instead of Bicep. Rejected because Bicep is the preferred format for Azure resources.
- **Alternative 2**: Implement a custom deployment controller instead of using the DeploymentTemplate controller. Rejected because the DeploymentTemplate controller provides the necessary functionality.

## Design Review Notes

<!--
Update this section with the decisions made during the design review meeting. This should be updated before the design is merged.
-->

To be updated after the design review meeting.
