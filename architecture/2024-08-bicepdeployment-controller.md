# BicepDeployment Controller

* **Author**: Will Smith (@willdavsmith)

## Overview

<!--
Provide a succinct high-level description of the component or feature and 
where/how it fits in the big picture. The overview should be one to three 
paragraphs long and should be understandable by someone outside the Radius
team. Do not provide the design details in this, section - there is a
dedicated section for that later in the document.
-->

Today, users of Radius (and importantly, future users of Radius) use many Kubernetes-specific tools in their production workflows, such as Flux and ArgoCD. The usage of these tools presents an issue when trying to integrate Radius - they can't deploy Radius resources directly since Radius resources are not Kubernetes objects. This design document proposes updates to Radius will allow users to deploy Radius resources by authoring Kubernetes CRDs, which will enable usage of Radius with GitOps tools as well as other Kubernetes-specific tools.

## Terms and definitions

<!--
Include any terms, definitions, or acronyms that are used in
this design document to assist the reader. They may or may not
be part of the user-facing experience once implemented, and can
be specific to this design context.
-->

**CRD (Custom Resource Definition)**: A Kubernetes resource that allows users to define their own resource types.

**GitOps** A way to do Kubernetes cluster management and application delivery. It works by using Git as a single source of truth for declarative infrastructure and applications.

**Flux**: A GitOps tool that automates the deployment of applications to Kubernetes clusters.

**ArgoCD**: Another GitOps tool that automates the deployment of applications to Kubernetes clusters.

## Objectives

<!--
Describe goals/non-goals and user-scenario of this feature to understand
the end-user goals.
* If the feature shares the same objectives of the existing design, link
  to the existing doc and section rather than repeat the same context.
* If the feature has a scenario, UX, or other product feature design doc,
  link it here and summarize the important parts.
-->

### Goals

<!--
Describe goals to define why we are doing this work, how we will make
priority decisions, and how we will determine success.
-->

**Goal: Users can use Kubernetes GitOps tools (Flux, ArgoCD) to deploy and manage resources defined in Bicep manifests**
- With this work, users will be able to deploy Radius resources using GitOps tools by authoring Kubernetes CRDs that contain Bicep manifests. We will essentially be providing a "translation layer" between Kubernetes resources (that Flux and ArgoCD can understand) and Radius resources (that Radius can understand).

**Goal: Users can deploy Radius resources using GitOps tools**
- We will be supporting the full suite of Radius resources in the BicepDeployment resource, so users can deploy any Radius resource using GitOps tools.

**Goal: Users can enable this feature without writing code of their own**
- We will provide a CLI command that generates the BicepDeployment resource from a Bicep manifest and parameters file, so users can enable this feature without writing code of their own.

### Non-goals

<!--
Describe non-goals to identify something that we won’t be focusing on 
immediately. We won’t be expending any effort on these matters. If there
will be follow-ups after this work, list them here. If there are things
we plan to do in the future, but are out of scope of this design, list
them here. Provide a brief explanation on why this is a non-goal.
-->

**Non-goal: Full support for GitOps**
- We will not yet be implementing automatic generation of BicepDeployment resources from Bicep manifests or querying Git repositories. This design will be covered in a separate design document.

### User scenarios

<!--
Describe the user scenarios for this design. Ensure that you define the
roles and personas in these user scenarios when it requires API design.
If you have an existing issue that describes the user scenarios, please
link to that issue instead.
-->

#### Jon can use GitOps to deploy a standard Radius environment for developer use

Jon is an infrastructure operator who uses GitOps to automate the creation of developer Kubernetes clusters. Today, he publishes Kubernetes manifests to a GitHub repository, and a GitOps tool watches the repository and applies the manifests to the Kubernetes cluster. He wants to have a Radius environment and a set of recipes available on the clusters for his developers to use, since they are building applications using Radius.

He first sets up the Radius Helm chart as a dependency in the cluster, and then runs the `rad bicep generate bicepdeployment env.bicep --parameters env.bicepparam` command on his machine to create an BicepDeployment resource that contains the template and parameters specified. He then commits the BicepDeployment resource to the Git repository.

When a developer creates a new development cluster using the configuration specified in Jon's GitHub repository with a GitOps tool, the BicepDeployment resource is applied on the cluster, and the Radius environment defined in the `env.bicep` Bicep manifest is deployed. The developer can then use the environment to deploy and test their applications using Radius.

#### Bran can use GitOps to deploy and manage a Radius application in production

Bran is a developer who uses GitOps to automate the deployment of Radius applications to production Kubernetes clusters. His infrastructure team has set up a Radius installation and environment on the production cluster, and Bran wants to deploy his application using Radius. He has authored a Bicep manifest that defines the resources for his application and a Bicep parameters file that specifies the parameters for the manifest. He runs the `rad bicep generate bicepdeployment app.bicep --parameters app.bicepparam` command on his machine to create an BicepDeployment resource that contains the template and parameters specified. He then commits the BicepDeployment resource to the Git repository.

When the GitOps tool watches the repository and applies the BicepDeployment resource to the production cluster, the Radius resources defined in the `app.bicep` Bicep manifest are deployed.

#### Sam can use GitOps to patch a Radius application in production

Sam is an SRE who is responsible for maintaining a Radius application in production. He notices that the application is experiencing performance issues and the number of application container replicas needs to be increased. He updates the Bicep parameters file for the application to specify the new number of replicas and runs the `rad bicep generate bicepdeployment app.bicep --parameters app.bicepparam` command on his machine to create an BicepDeployment resource that contains the updated parameters. He then commits the BicepDeployment resource to the Git repository.

When the GitOps tool watches the repository and applies the BicepDeployment resource to the production cluster, the Radius resources defined in the `app.bicep` Bicep manifest are updated with the new parameters. The application container replicas are increased, and the performance issues are resolved.


## User Experience
<!--
If the change impacts the user experience, provide expected interaction 
flow we aim to achieve through this proposal.

When users interact with Radius through the CLI, include sample 
input commands and their corresponding output. Include a bicep/helm code 
sample, if this proposal involves updates to that experience.
-->

**Sample Input:**
<!--
Provide a sample CLI command input and/or bicep/helm code.
-->

#### `env.bicep`
```bicep
extension radius

param k8sNamespace string

resource env 'Applications.Core/environments@2023-10-01-preview' = {
  name: 'env'
  properties: {
    compute: {
      kind: 'kubernetes'
      namespace: k8sNamespace
    }
  }
}
```

#### `env.bicepparam`
```bicep
using './env.bicep'

param k8sNamespace string = 'demo'
```

**Sample Output:**
<!--
Provide a sample output for the inputs provided above.
-->
```
> rad bicep generate bicepdeployment env.bicep --parameters env.bicepparam --out env.yaml

Generating BicepDeployment resource...
BicepDeployment resource generated at env.yaml

To apply the BicepDeployment resource onto your cluster, run:
kubectl apply -f env.yaml
```

#### `env.yaml`
```yaml
kind: BicepDeployment
apiVersion: radapp.io/v1alpha3
metadata:
  name: env
  namespace: radius-system
spec:
  template: |
    extension radius

    param k8sNamespace string

    resource env 'Applications.Core/environments@2023-10-01-preview' = {
      name: 'env'
      properties: {
        compute: {
          kind: 'kubernetes'
          namespace: k8sNamespace
        }
      }
    }
  parameters: |
    using './env.bicep'

    param k8sNamespace string = 'demo'
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

This design proposes the creation of a new Kubernetes controller (BicepDeployment controller) packaged within the `radius-controller` service.

The BicepDeployment controller will be implemented as a Kubernetes controller that watches and reconciles BicepDeployment resources on the cluster. The BicepDeployment controller will adhere to the following control loop for each of the BicepDeployment resources on the cluster:

1. When a BicepDeployment resource is created or updated on the cluster, the controller will compile the Bicep template and parameters into ARM JSON. If there are any errors during the compilation, the controller will update the `status.phrase` and the `status.description` of the BicepDeployment resource to indicate that the deployment has failed.

2. Check if we have an "operation" in progress. If so, check it's status.
    1. If the operation is still in progress, then queue another reconcile (polling).
	  2. If the operation completed successfully then update the status and continue processing (happy-path).
	  3. If the operation failed then update the status and continue processing (retry).
3. If the ApplicationDeployment is being deleted then process deletion.
	  1. This may require us to start a DELETE operation. After that we can continue polling.
    2. If the ApplicationDeployment is not being deleted then process this as a creation or update.
	  3. This may require us to start a PUT operation. After that we can continue polling.
  
1. The controller will perform some simple lookups to Radius to determine if the provided template can be deployed. It will make two checks:
    - Check if the template contains an empty `param environment`. If it does, the controller will query the Radius API to get an environment ID (must have exactly one environment if left unspecified like this) and deploy the template using that environment.
    - Check if the template contains a `param application`. If it does, the controller will query the Radius API to get an application ID (must have exactly one application if left unspecified like this) and deploy the template using that application.
2. The controller will send the ARM JSON to the Radius API to deploy the resources. If there are any errors during the deployment, the controller will update the `status.phrase` and the `status.description` of the BicepDeployment resource to indicate that the deployment has failed and include the error message.

> TODO: We need to figure out deletion of specific resources within the template. Let's discuss in the design meeting.

### Architecture Diagram
<!--
Provide a diagram of the system architecture, illustrating how different
components interact with each other in the context of this proposal.

Include separate high level architecture diagram and component specific diagrams, wherever appropriate.
-->

![Architecture Diagram](2024-08-bicepdeployment-controller/architecture.png)

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

#### BicepDeployment Custom Resource Definition

The `BicepDeployment` resource will be a new Kubernetes CRD that will be used to contain the Bicep template and parameters for deploying Radius resources. The CRD will have the following fields:

```go
// BicepDeploymentSpec defines the desired state of an BicepDeployment
type BicepDeploymentSpec struct {
  // Template is the Bicep template that defines the resources to deploy.
  // +kubebuilder:validation:Required
  Template string `json:"template"`

  // Parameters is the Bicep parameters file that specifies the parameters for the template.
  // +kubebuilder:validation:Optional
  Parameters string `json:"parameters"`
}

// BicepDeploymentPhrase is a string representation of the current status of an Bicep Deployment.
type BicepDeploymentPhrase string

const (
	// BicepDeploymentPhraseUpdating indicates that the Bicep Deployment is being updated.
	BicepDeploymentPhraseUpdating BicepDeploymentPhrase = "Updating"

	// BicepDeploymentPhraseReady indicates that the Bicep Deployment is ready.
	BicepDeploymentPhraseReady BicepDeploymentPhrase = "Ready"

	// BicepDeploymentPhraseFailed indicates that the Bicep Deployment has failed.
	BicepDeploymentPhraseFailed BicepDeploymentPhrase = "Failed"

	// BicepDeploymentPhraseDeleting indicates that the Bicep Deployment is being deleted.
	BicepDeploymentPhraseDeleting BicepDeploymentPhrase = "Deleting"

	// BicepDeploymentPhraseDeleted indicates that the Bicep Deployment has been deleted.
	BicepDeploymentPhraseDeleted BicepDeploymentPhrase = "Deleted"
)

// BicepDeploymentStatus defines the observed state of the Bicep Deployment.
type BicepDeploymentStatus struct {
	// ObservedGeneration is the most recent generation observed for this Bicep Deployment. It corresponds to the Deployment's generation, which is updated on mutation by the API Server.
	// +kubebuilder:validation:Optional
	ObservedGeneration int64 `json:"observedGeneration,omitempty" protobuf:"varint,1,opt,name=observedGeneration"`

	// Scope is the resource ID of the scope.
	// +kubebuilder:validation:Optional
	Scope string `json:"scope,omitempty"`

	// Resource is the resource ID of the deployment.
	// +kubebuilder:validation:Optional
	Resource string `json:"resource,omitempty"`

	// Operation tracks the status of an in-progress provisioning operation.
	// +kubebuilder:validation:Optional
	Operation *ResourceOperation `json:"operation,omitempty"`

	// Phrase indicates the current status of the Bicep Deployment.
	// +kubebuilder:validation:Optional
	Phrase BicepDeploymentPhrase `json:"phrase,omitempty"`

  // Description is a human-readable description of the status of the Bicep Deployment.
  // +kubebuilder:validation:Optional
  Description string `json:"description,omitempty"`
}
```

#### `rad bicep generate bicepdeployment` Command

The `rad bicep generate bicepdeployment <bicep file>` command will be a new command that will generate an BicepDeployment resource when provided a Bicep template and (optionally) a Bicep parameters file. The command will take the following arguments:

- `-p, --parameters`: The path to the Bicep parameters file
- `-o, --output`: The path to the output file where the BicepDeployment resource will be written

The command will read the Bicep template and parameters files and generate a BicepDeployment resource with the template and parameters set. The command will write the BicepDeployment resource to the output file.

#### BicepDeployment Controller

The `radius-controller` will be updated to include a new controller that reconciles BicepDeployment resources. When an BicepDeployment resource is created or updated, the controller will compile the Bicep template and parameters into ARM JSON and send the ARM JSON to the Radius API to deploy the resources. The controller will update the `status.phrase` and the `status.description` of the BicepDeployment resource based on the status of the deployment (see `BicepDeploymentPhrase` above).

The controller will also perform some simple lookups to Radius to determine if the provided template can be deployed. It will make two checks:
- Check if the template contains an empty `param environment`. If it does, the controller will query the Radius API to get an environment ID (must have exactly one environment if left unspecified like this) and deploy the template using that environment.
- Check if the template contains a `param application`. If it does, the controller will query the Radius API to get an application ID (must have exactly one application if left unspecified like this) and deploy the template using that application.

The controller will also be shipped with a Bicep binary which will allow it to compile Bicep templates and parameters into ARM JSON. The Bicep binary will be included in the `radius-controller` container image.

### Alternatives Considered

- We could create a separate service for Bicep compilation.

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

#### API

There will be no changes to the Radius API.

#### CLI

There will be a new CLI command `rad applicationdeployment generate` that will generate an BicepDeployment resource from a Bicep template and parameters file.

### Error Handling
<!--
Describe the error scenarios that may occur and the corresponding recovery/error handling and user experience.
-->

TBW

## Test plan

<!--
Include the test plan to validate the features including the areas that
need functional tests.

Describe any functionality that will create new testing challenges:
- New dependencies
- External assets that tests need to access
- Features that do I/O or change OS state and are thus hard to unit test
-->

TBW

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

TBW

## Compatibility

<!--
Describe potential compatibility issues with other components, such as
incompatibility with older CLIs, and include any breaking changes to
behaviors or APIs.
-->

The changes describes are only additive, so there should be no breaking changes or compatibility issues with other components.

## Monitoring and Logging

<!--
Include the list of instrumentation such as metric, log, and trace to 
diagnose this new feature. It also describes how to troubleshoot this feature
with the instrumentation. 
-->

TBW

## Development plan

<!--
Describe how you will deliver your features. This includes aligning work items
to features, scenarios, or requirements, defining what deliverable will be
checked in at each point in the product and estimating the cost of each work
item. Don’t forget to include the Unit Test and functional test in your
estimates.
-->

TBW

## Open Questions

<!--
Describe (Q&A format) the important unknowns or things you're not sure about. 
Use the discussion to answer these with experts after people digest the 
overall design.
-->

Do we need to worry about developer vs SRE permission boundaries? e.g. developer can deploy an application but SRE can only edit the parameters?

## Design Review Notes

<!--
Update this section with the decisions made during the design review meeting. This should be updated before the design is merged.
-->
