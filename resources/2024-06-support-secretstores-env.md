# Title

* **Status**: Pending/Approved
* **Author**: Nick Beenham (@superbeeny)

## Overview

Currently the only way to provide kubernetes secrets to a container is to mount them through a secret volume. This is not ideal for many use cases, especially when the secret is only needed in the environment variables of the container. This proposal aims to add support for secret stores to the environment variables of a container.

## Terms and definitions

<!--
Include any terms, definitions, or acronyms that are used in
this design document to assist the reader. They may or may not
be part of the user-facing experience once implemented, and can
be specific to this design context.
-->

## Objectives

> **Issue Reference:** [Issue #5520](https://github.com/radius-project/radius/issues/5520)

### Goals

- Allow users to provide secrets to a container through environment variables

### Non goals

<!--
Describe non-goals to identify something that we won’t be focusing on 
immediately. We won’t be expending any effort on these matters. If there
will be follow-ups after this work, list them here. If there are things
we plan to do in the future, but are out of scope of this design, list
them here.
-->

### User scenarios (optional)

<!--
Describe the user scenarios for this design. Ensure that you define the
roles and personas in these user scenarios when it requires API design.
If you have an existing issue that describes the user scenarios, please
link to that issue instead.
-->

#### User story 1
As a radius user, I want to provide a secret to a container through an environment variable so that I can avoid mounting the secret as a volume. I want to be able to reference the secret within the application bicep file.

```diff
import radius as radius

@description('The Radius Application ID. Injected automatically by the rad CLI.')
param application string

resource demo 'Applications.Core/containers@2023-10-01-preview' = {
  name: 'demo'
  properties: {
    application: application
    container: {
      image: 'ghcr.io/radius-project/samples/demo:latest'
      ports: {
        web: {
          containerPort: 3000
        }
      }
      env: {
        DB_USER: { value: 'DB_USER' }
        DB_PASSWORD: {
          valueFrom: {
            secretRef: {
              source: secret.id
              key: 'DB_PASSWORD'
            }
          }
        }
      }
    }
  }
}

resource secret 'Applications.Core/secretStores@2023-10-01-preview' = {
  name: 'secret'
  properties: {
    application: application
    data: {
      DB_PASSWORD: {
        value: 'password'
      }
    }
  }
}

```

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

### Architecture Diagram
<!--
Provide a diagram of the system architecture, illustrating how different
components interact with each other in the context of this proposal.

Include separate high level architecture diagram and component specific diagrams, wherever appropriate.
-->

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

#### Disadvantages (of each option considered)
<!--
Describe what's not ideal about this plan. Does it lock us into a 
particular design for future changes or is it flexible if we were to 
pivot in the future. This is a good place to cover risks.
-->

#### Proposed Option
<!--
Describe the recommended option and provide reasoning behind it.
-->

### API design 

<!--
Include if applicable – any design that changes our public REST API, CLI
arguments/commands, or Go APIs for shared components should provide this
section. Write N/A here if not applicable.
- Describe the REST APIs in detail for new resource types or updates to
  existing resource types. E.g. API Path and Sample request and response.
- Describe new commands in the CLI or changes to existing CLI commands.
- Describe the new or modified Go APIs for any shared components.
-->
Updates to the container typespec
```diff
@doc("Definition of a container")
model Container {
  @doc("The registry and image to download and run in your container")
  image: string;

  @doc("The pull policy for the container image")
  imagePullPolicy?: ImagePullPolicy;

  @doc("environment")
  env?: Record<EnvironmentVariable>;

  @doc("container ports")
  ports?: Record<ContainerPortProperties>;

  @doc("readiness probe properties")
  readinessProbe?: HealthProbeProperties;

  @doc("liveness probe properties")
  livenessProbe?: HealthProbeProperties;

  @doc("container volumes")
  volumes?: Record<Volume>;

  @doc("Entrypoint array. Overrides the container image's ENTRYPOINT")
  command?: string[];

  @doc("Arguments to the entrypoint. Overrides the container image's CMD")
  args?: string[];

  @doc("Working directory for the container")
  workingDir?: string;
}

@doc("Envinronment variables type")
model EnvironmentVariable {

  @doc("The value of the environment variable")
  value?: string;

  @doc("The reference to the variable")
  valueFrom?: EnvironmentVariableReference;
}

@doc("The reference to the variable")
model EnvironmentVariableReference {
  @doc("The secret reference")
  secretRef: EnvironmentVariableSecretReference;
}

@doc("The secret reference")
model EnvironmentVariableSecretReference {
  @doc("The secret source identifier")
  source: string;

  @doc("The secret key")
  key: string;
}
```

### CLI Design (if applicable)
<!--
Include if applicable – any design that changes Radius CLI
arguments/commands. Write N/A here if not applicable.
- Describe new commands in the CLI or changes to existing CLI commands.
-->

### Implementation Details
<!--
High level description of updates to each component. Provide information on 
the specific sub-components that will be updated, for example, controller, processor, renderer,
recipe engine, driver, to name a few.
-->

#### UCP (if applicable)
#### Bicep (if applicable)
#### Deployment Engine (if applicable)
#### Core RP (if applicable)
#### Portable Resources / Recipes RP (if applicable)

### Error Handling
<!--
Describe the error scenarios that may occur and the corresponding recovery/error handling and user experience.
-->

## Test plan

<!--
Include the test plan to validate the features including the areas that
need functional tests.

Describe any functionality that will create new testing challenges:
- New dependencies
- External assets that tests need to access
- Features that do I/O or change OS state and are thus hard to unit test
-->

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

## Compatibility (optional)

<!--
Describe potential compatibility issues with other components, such as
incompatibility with older CLIs, and include any breaking changes to
behaviors or APIs.
-->

## Monitoring and Logging

<!--
Include the list of instrumentation such as metric, log, and trace to 
diagnose this new feature. It also describes how to troubleshoot this feature
with the instrumentation. 
-->

## Development plan

<!--
Describe how you will deliver your features. This includes aligning work items
to features, scenarios, or requirements, defining what deliverable will be
checked in at each point in the product and estimating the cost of each work
item. Don’t forget to include the Unit Test and functional test in your
estimates.
-->

## Open Questions

<!--
Describe (Q&A format) the important unknowns or things you're not sure about. 
Use the discussion to answer these with experts after people digest the 
overall design.
-->

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