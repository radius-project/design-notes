# Radius Integration with Azure Container Instances (ACI) - Confidential Containers

* **Author**: Shruthi Kumar (@sk593)

## Overview

The integration of Azure Container Instances (ACI) with Radius aims to enable deployments of applications and containers using the Radius control plane in Kubernetes to target ACI environments. One of the key features we want to implement is the use of confidential containers to enhance data security and privacy. Confidential containers on Azure provide a set of features and capabilities to further secure your standard container workloads. They run in a hardware-backed Trusted Execution Environment (TEE) that provides intrinsic capabilities like data integrity, data confidentiality, and code integrity.

Confidential containers require a CCE policy to be set on the container group profile associated with ACI instances. CCE policies can only be generated using an extension on the Azure CLI. As a result, Radius will need access to the Azure CLI and the `confcom` extension to support this feature. More information on confidential containers with ACI can be found [here](https://learn.microsoft.com/en-us/azure/container-instances/container-instances-tutorial-deploy-confidential-containers-cce-arm)

The key components that will be deployed by Radius to support ACI are container group profiles, NGroups, and the container instance itself. 

## Terms and definitions

- ACI: Azure Container Instances
- TEE: Trusted Execution Environment
- Confidential Containers: Containers that run in a hardware-backed TEE to provide enhanced security features
- Container Group Profile: A configuration template in Azure Container Instances that defines the properties and settings for a container group. It includes specifications such as container images, resource allocations (CPU/memory), networking settings, storage mounts, OS type, SKU (Standard/Dedicated/Confidential), and security policies. For confidential containers, the Container Group Profile contains the CCE policy that enables the Trusted Execution Environment.

## Objectives

This document is one component of the larger serverless extensibility experience. See below for relevant design documents. 

### Goals

- Support the deployment of application containers to ACI, ensuring that container definitions are platform-agnostic.
- Allow users to apply ACI-specific configurations to containers using extension and runtime properties.
- Support generating a CCE policy as part of a container group profile creation. 

### Non-goals

- A serverless extensibility design. This document will only focus on components ot Radius relevant to confidential container deployment. Compute extensibility design can be found [here](https://github.com/radius-project/design-notes/pull/91).  
- Multi-compute support per environment will not be available initially.  
- Full Dapr parity will not be available initially.  
- ACI Service Discovery.

### User scenarios (optional)

<!--
Describe the user scenarios for this design. Ensure that you define the
roles and personas in these user scenarios when it requires API design.
If you have an existing issue that describes the user scenarios, please
link to that issue instead.
-->

The scenario document for severless is in progress [here](https://github.com/radius-project/design-notes/blob/9bcaa014c013b254593426280add5ef5c69b265e/architecture/2025-01-serverless-feature-spec.md#scenario-3-punch-through-to-platform-specific-features-and-incremental-adoption-of-radius-into-existing-serverless-applications). Please refer to the scenarios defined in that doc, specifically scenario 3 as it relates to `runtimes` punchthroughs. 

## User Experience (if applicable)
<!--
If the change impacts the user experience, provide expected interaction 
flow we aim to achieve through this proposal.

When users interact with Radius through the CLI, include sample 
input commands and their corresponding output. Include a bicep/helm code 
sample, if this proposal involves updates to that experience.
-->

Ted is a developer trying to run his applications on a serverless compute platform. His application needs to run on confidential containers. His E2E experience defining a Bicep template using ACI compute looks like this: 

**Setup**

1. Run `rad init`.
2. When prompted to create an environment, choose the `aci` compute option.
3. When prompted, choose whether or not to support confidential containers. This will download the Azure CLI and the `confcom` extension. 

Note: The Kubernetes flow will not change, minus the extra step to select Kubernetes compute. 

**Sample Input:**
<!--
Provide a sample CLI command input and/or bicep/helm code.
-->
```
rad init --full 

Enter an environment name              
> default  
                                                      
Select the compute for the environment
  >  1. kubernetes 
  >  2. aci

Setup support for confidential containers? Selecting 'Yes' will install the Azure CLI and the `confcom` extension that are required to support confidential containers on ACI.  
  >  1. Yes
  >  2. No    
```

**Authoring:**
<!--
Provide a sample output for the inputs provided above.
-->

Ted now has a default environment setup using `ACI` as the underlying compute. He can define a Radius container that runs with a `Confidential` SKU. Ted can set additional values on the container group profile like `name`, `osType`, etc to pass through when Radius creates the container group profile. 

Note: The `runtimes` property is not meant to be a replacement for defining a `ContainerGroupProfile` resource in Bicep. It is meant to be used as a punchthrough for specific values that the user wants to set on the default container group profile that Radius creates.  

**Sample Bicep Template:**

```diff
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
+     // all the other container properties should also be implemented (e.g. `env`, `readinessProbe`, `livenessProbe`, etc.)
    }
    extensions: [
+     {
+       kind:  'manualScaling'
+       replicas: 2
+     }
    ]
    runtimes: {
+     aci: {
+       containergroupprofile: {
+          name: 'mycgname'
+       // Add ACI-specific properties here to punch-through the Radius abstraction, e.g. sku, osType, etc. These should be in the same format as the Bicep definition for a container group profile 
+         properties: {
+             sku: 'Confidential' // 'Standard', 'Dedicated', etc.
+         }
+       }
+     }
  }
}
```

Ted now deploys his application using `rad deploy` as usual. 


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

When deploying a container through Radius with ACI as the compute target, Radius creates three underlying Azure resources to support the container:
1. Container Group Profile: Defines the configuration template with settings like SKU, networking, and security policies (including the CCE policy required for confidential containers)
2. NGroup: Provides a way to create and manage `n` container groups with a single set of operations
3. Container Instance: The ACI container that runs the workload

The implementation of confidential containers will primarily focus on enhancing these three components, with special emphasis on the Container Group Profile generation process to include the CCE policy.

The renderers and handlers for ACI should be able to 
1. Create a container group profile. The container group profile may or may not contain punchthrough properties set by the user using the `runtimes` field.
  1. In the context of confidential containers, the CCE policy will be generated and set by Radius before the container group profile gets deployed
2. Create an NGroups object 
3. Create an ACI instance 

In addition, we will need to support an `aci` property and `containergroupprofile` property under `runtimes`.

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

#### Bicep changes  
The Bicep schema will be updated to include a field for the `containergroupprofile` data and properties. These changes will be automatically generated through the TypeSpec schema changes. 

#### Core RP changes

**ACI Renderer**  
The ACI renderer will need to be updated to support punchthrough values on the container group profile. Radius sets up a container group profile by default. If punchthrough values are provided by the user, the default values will be updated to use the user-provided values. This will be done through a merge process between the user inputs and the ContainerGroupProfile object provided by the ACI SDK. 
```
// ContainerGroupProfile - container group profile object
type ContainerGroupProfile struct {
	// The resource location.
	Location *string `json:"location,omitempty"`

	// The container group profile properties
	Properties *ContainerGroupProfileProperties `json:"properties,omitempty"`

	// The resource tags.
	Tags map[string]*string `json:"tags,omitempty"`

	// The zones for the container group.
	Zones []*string `json:"zones,omitempty"`

	// READ-ONLY; The resource id.
	ID *string `json:"id,omitempty" azure:"ro"`

	// READ-ONLY; The resource name.
	Name *string `json:"name,omitempty" azure:"ro"`

	// READ-ONLY; Metadata pertaining to creation and last modification of the resource.
	SystemData *SystemData `json:"systemData,omitempty" azure:"ro"`

	// READ-ONLY; The resource type.
	Type *string `json:"type,omitempty" azure:"ro"`
}
```

**Azure CG Profile handler**  
To enable confidential container support, the Azure Container Group Profile handler requires updates. Following the official guidance for confidential containers [here](https://learn.microsoft.com/en-us/azure/container-instances/container-instances-tutorial-deploy-confidential-containers-cce-arm), Radius will automate the setup process through these implementation steps:

1. Prior to deployment, the Azure CLI and `confcon` extension should have been downloaded. This would have been set up during `rad init` by the user when they elected to support confidential containers on ACI.
2. In the ACI renderer, populate a container group profile object using the default values or the user-provided inputs. See the previous section on the ACI renderer for details. 
3. In the container group profile handler, generate a temporary ARM template that contains the same data in the container group profile object. The container group profile object is passed as in input to the handler. 
4. Run the following command on the ARM template to generate a CCE policy: `az confcom acipolicygen -a .\template.json`. This command modifies the existing template to include the base64 encoded CCE policy.
5. Extract the CCE policy string from the ARM template and add it to the existing container group profile object. 
6. Create the container group profile using the ACI SDK with the CCE policy set.

#### Advantages (of each option considered)
<!--
Describe what's good about this plan relative to other options. 
Provides better user experience? Does it feel easy to implement? 
Provides flexibility for future work?
-->

This approach ensures the user experience has parity with Kubernetes `runtimes`. Radius takes care of setting up the backing resources for a container instance, including generating the necessary security policies.

#### Disadvantages (of each option considered)
<!--
Describe what's not ideal about this plan. Does it lock us into a 
particular design for future changes or is it flexible if we were to 
pivot in the future. This is a good place to cover risks.
-->

The CCE policy can only be generated using the `confcon` CLI extension. There are no available SDKs to generate this value. As a result, Radius must have access to the extension to generate policies on behalf of the user. 

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

`typespec/Applications.Core/containers.tsp`

```typespec
@doc("The properties for runtime configuration")
model RuntimesProperties {
  @doc("The runtime configuration properties for Kubernetes")
  kubernetes?: KubernetesRuntimeProperties;

  @doc("The runtime configuration properties for ACI")
  aci?: ACIRuntimeProperties;
}

#suppress "@azure-tools/typespec-azure-core/bad-record-type"
@doc("A merge will be applied to the ContainerGroupProfile object when this container is being deployed.")
model ContainerGroupProfile is Record<unknown>;

@doc("The runtime configuration properties for ACI")
model ACIRuntimeProperties {
  #suppress "@azure-tools/typespec-azure-core/bad-record-type"
  @doc("A merge will be applied to the ContainerGroupProfile object when this container is being deployed.")
  containergroupprofile?: ContainerGroupProfile;
}
```

### CLI Design (if applicable)
<!--
Include if applicable – any design that changes Radius CLI
arguments/commands. Write N/A here if not applicable.
- Describe new commands in the CLI or changes to existing CLI commands.
-->

`rad init` would add 2 steps: one step to determine which compute platform to use when the user chooses to set up an environment and one step to choose confidential containers. See the user experience section for details. 

### Error Handling
<!--
Describe the error scenarios that may occur and the corresponding recovery/error handling and user experience.
-->

Radius would expect the `runtimes` property to have the correct format. We would return errors if the formatting is incorrect (this would be surfaced during the merge process) or when deploying the container group profile (this would be surfaced by ARM if the ACI SDK throws an error on creation). 

## Test plan

<!--
Include the test plan to validate the features including the areas that
need functional tests.

Describe any functionality that will create new testing challenges:
- New dependencies
- External assets that tests need to access
- Features that do I/O or change OS state and are thus hard to unit test
-->

TBD

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

The CCE policy is a base64 encoded string, however this is not considered a secret value. 

## Compatibility (optional)

<!--
Describe potential compatibility issues with other components, such as
incompatibility with older CLIs, and include any breaking changes to
behaviors or APIs.
-->

This shouldn't cause any breaking changes to other components since it will be an added feature.

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

Is there an easier way to generate the CCE policy? The CLI extension is the only way to generate the CCE policy right now. There are no SDKs or APIs by design. 

## Alternatives considered

Option 1: 

We could require the user to create a container group profile first and pass the resource to Radius. This would eliminate the need for Radius to install the `confcom` extension, generating ARM templates, and extracting the CCE policy. 

This would require additional setup by the user, but allows more flexibility when creating a container group profile. Radius would still be able to support updates to the container group profile through the `runtimes` property, but the creation would be handled by the user. 


## Design Review Notes

<!--
Update this section with the decisions made during the design review meeting. This should be updated before the design is merged.
-->