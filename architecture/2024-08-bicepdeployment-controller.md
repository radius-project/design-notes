# Radius BicepDeployment Controller

* **Author**: Will Smith (@willdavsmith)

## Overview

Today, users of Radius and future adopters of Radius use many Kubernetes-specific tools in their production workflows. Most of these tools operate on Kubernetes resources exclusively - which presents a problem when trying to deploy resources defined in Bicep manifests. This design proposes the creation of a new Kubernetes reconciler in Radius called the BicepDeployment Reconciler that will allow users to deploy resources defined in Bicep manifests using Kubernetes tooling.

## Today's Status

Today, Radius users are unable to deploy resources defined in Bicep manifests using Kubernetes tooling. This is because Kubernetes tools do not understand Bicep manifests. Users must use the Radius CLI to deploy resources defined in Bicep manifests.

## Terms and definitions

**CRD (Custom Resource Definition)**: A definition for a Kubernetes resource that allows users to define their own resource types.

**CR (Custom Resource)**: An instance of a CRD that represents a custom resource in Kubernetes.

**Bicep**: An infrastructure-as-code language that when used with Radius, can deploy Radius resources, Azure resources, and AWS resources.
 
## Objectives

### Goals

**Goal: Users can use Kubernetes tooling to deploy and manage resources defined in Bicep manifests**
- With this work, users will be able to deploy resources defined in Bicep using only Kubernetes. We will essentially be providing a "translation layer" between Kubernetes resources (that Kubernetes tools can understand) and Radius resources (that Radius can understand).

**Goal: Users can quickly generate a Kubernetes Custom Resource from Bicep using the Radius CLI**
- We will provide a CLI command that generates the BicepDeployment resource from a Bicep manifest to make this feature easy to adopt.

**Goal: The behavior of BicepDeployment is consistent with user expectations for a Kubernetes-enabled tool**
- We will follow user expectations for Kubernetes controllers and resources when designing the BicepDeployment controller and resource. This includes building in support for retries, status updates, and error handling.

### Non-goals

**Non-goal: Full support for GitOps**

- We will not yet be implementing automatic generation of BicepDeployment resources from Bicep manifests or querying Git repositories. This design will enable this work, and it will be covered in a future design document.

### User scenarios

#### Jon can deploy cloud (Azure or AWS) resources defined in Bicep manifests using Kubernetes tools

Jon is an infrastructure operator for an enterprise company. His team manages a production application that is deployed on Kubernetes and uses cloud resources. Jon wants to declare all dependencies for his application in Kubernetes manifests, including cloud resources. He installs Radius on his cluster and uses the rad CLI to generate a custom resource from his Bicep manifest. Jon applies the custom resource to his cluster, and Radius deploys the cloud resources defined in the Bicep manifest. If he wants to update or delete the cloud resources, he can do so by re-applying the updated custom resource or deleting the custom resource.

#### Jon can deploy Radius resources defined in Bicep manifests using Kubernetes tools

Now that he can see that Radius can deploy cloud resources defined in Bicep manifests, Jon wants to take advantage of Radius tooling, such as the App Graph, and fully "Rad-ify" his application. He writes a Bicep manifest that defines a Radius container that connects to the cloud resources, and uses the rad CLI to generate a custom resource from the Bicep manifest. Jon applies the custom resource to his cluster, and Radius deploys the Radius resources defined in the Bicep manifest. Now, Jon can take advantage of Radius tooling, such as the Radius Dashboard and App Graph, to manage his application.

## User Experience

### Radius Container

**Sample Input:**

#### `app.bicep`
```bicep
extension radius

param application string
param tag string
param port int

resource container 'Applications.Core/containers@2023-10-01-preview' = {
  name: 'container'
  properties: {
    application: application
    container: {
      image: 'ghcr.io/radius-project/samples/demo:${tag}'
      ports: {
        web: {
          containerPort: port
        }
      }
    }
  }
}
```

#### `app.bicepparam`
```bicep
using 'app.bicep'

param application = ''
param tag = ''
param port = 3000
```

**Sample Output:**
```shell
rad bicep generate-kubernetes app.bicep --parameters @app.bicepparam --parameters tag=latest --parameters appnetworking.bicepparam --outfile app.yaml

Generating BicepDeployment resource...
BicepDeployment resource generated at app.yaml

To apply the BicepDeployment resource onto your cluster, run:
kubectl apply -f app.yaml
```

#### `app.yaml`
```yaml
kind: BicepDeployment
apiVersion: radapp.io/v1alpha3
metadata:
  name: env
  namespace: radius-system
spec:
  template: |
    {
      "$schema": "https://schema.management.azure.com/schemas/2019-04-01/deploymentTemplate.json#",
      "languageVersion": "2.1-experimental",
      "contentVersion": "1.0.0.0",
      "metadata": {
        "_EXPERIMENTAL_WARNING": "This template uses ARM features that are experimental. Experimental features should be enabled for testing purposes only, as there are no guarantees about the quality or stability of these features. Do not enable these settings for any production usage, or your production environment may be subject to breaking.",
        "_EXPERIMENTAL_FEATURES_ENABLED": [
          "Extensibility"
        ],
        "_generator": {
          "name": "bicep",
          "version": "0.29.47.4906",
          "templateHash": "14156924711842038952"
        }
      },
      "parameters": {
        "application": {
          "type": "string"
        },
        "tag": {
          "type": "string"
        },
        "port": {
          "type": "int"
        }
      },
      "imports": {
        "Radius": {
          "provider": "Radius",
          "version": "latest"
        }
      },
      "resources": {
        "container": {
          "import": "Radius",
          "type": "Applications.Core/containers@2023-10-01-preview",
          "properties": {
            "name": "container",
            "properties": {
              "application": "[parameters('application')]",
              "container": {
                "image": "[format('ghcr.io/radius-project/samples/demo:{0}', parameters('tag'))]",
                "ports": {
                  "web": {
                    "containerPort": "[parameters('port')]"
                  }
                }
              }
            }
          }
        }
      }
    }
  parameters: |
    {
      "application": {
        "value": ""
      },
      "tag": {
        "value": "latest"
      },
      "port": {
        "value": 3000
      }
    }
```

## Design

### High Level Design

This design proposes the creation of two new Kubernetes custom resources and their corresponding reconcilers: `BicepDeployment` and `BicepResource`. The `BicepDeployment` controller will be responsible for deploying and reconciling resources defined in ARM JSON manifests via Radius, while the `BicepResource` controller will be responsible for reconciling individual resources. The `BicepDeployment` controller will have a control loop that will check the status of an in-progress operation, process deletion, and process creation or update. The `BicepResource` controller will have a similar control loop that will check the status of an in-progress operation or process deletion if necessary.

### Architecture Diagram

![Architecture Diagram](2024-08-bicepdeployment-controller/architecture.png)

### Detailed Design

#### BicepDeployment Custom Resource Definition

The `BicepDeployment` CRD will be a new Kubernetes CRD that will be used to contain the ARM JSON manifest and parameters. The CRD will have the following fields:

```go
// BicepDeploymentSpec defines the desired state of a BicepDeployment
type BicepDeploymentSpec struct {
  // Template is the ARM JSON manifest that defines the resources to deploy.
  Template string `json:"template"`

  // Parameters is the ARM JSON parameters for the template.
  Parameters string `json:"parameters"`
}

// BicepDeploymentStatus defines the observed state of the Bicep Deployment.
type BicepDeploymentStatus struct {
  // ObservedGeneration is the most recent generation observed for this BicepDeployment.
  ObservedGeneration int64 `json:"observedGeneration,omitempty"`

	// Scope is the resource ID of the scope.
	Scope string `json:"scope,omitempty"`

	// Resource is the resource ID of the deployment.
	Resource string `json:"resource,omitempty"`

	// Operation tracks the status of an in-progress provisioning operation.
	Operation *ResourceOperation `json:"operation,omitempty"`

	// Phrase indicates the current status of the Bicep Deployment.
	Phrase BicepDeploymentPhrase `json:"phrase,omitempty"`

  // Message is a human-readable description of the status of the Bicep Deployment.
  Message string `json:"message,omitempty"`
}

// BicepDeploymentPhrase is a string representation of the current status of a Bicep Deployment.
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
```

#### BicepResource Custom Resource Definition

The `BicepResource` CRD is another CRD that will be responsible for tracking the state of individual resources. The CRD will have the following fields:

```go
// BicepResourceSpec defines the desired state of a Bicep Resource.
type BicepResourceSpec struct {
  // ResourceId is the Radius resource Id.
  ResourceId string `json:"resourceId"`
}

type BicepDeploymentResourceStatus struct {
  // ObservedGeneration is the most recent generation observed for this BicepDeploymentResource.
  ObservedGeneration int64 `json:"observedGeneration,omitempty"`

  // Operation tracks the status of an in-progress provisioning operation.
  Operation *ResourceOperation `json:"operation,omitempty"`

  // Phrase indicates the current status of the Bicep Deployment Resource.
  Phrase BicepDeploymentResourcePhrase `json:"phrase,omitempty"`

  // Message is a human-readable description of the status of the Bicep Deployment Resource.
  Message string `json:"description,omitempty"`
}

// BicepResourcePhrase is a string representation of the current status of a Bicep Resource.
type BicepResourcePhrase string

const (
	// BicepResourcePhraseReady indicates that the Bicep Resource is ready.
	BicepResourcePhraseReady BicepResourcePhrase = "Ready"

	// BicepResourcePhraseFailed indicates that the Bicep Resource has failed to delete.
	BicepPhraseFailed BicepResourcePhrase = "Failed"

	// BicepResourcePhraseDeleting indicates that the Bicep Resource is being deleted.
	BicepPhraseDeleting BicepResourcePhrase = "Deleting"

	// BicepResourcePhraseDeleted indicates that the Bicep Resource has been deleted.
	BicepPhraseDeleted BicepResourcePhrase = "Deleted"
)
```

#### `rad bicep generate-kubernetes` Command

The `rad bicep generate-kubernetes` command will be a new command that will generate a Kubernetes custom resource when provided a Bicep template and its parameters. The command will take the following arguments:

- `-p, --parameters`: The parameters for the Bicep template. Can be specified in the same way as [`rad deploy` parameters](https://edge.docs.radapp.io/reference/cli/rad_deploy/).
- `-o, --outfile`: The path to the output file where the `BicepDeployment` resource will be written.

#### `BicepDeployment` Reconciler

The `radius-controller` will be updated to include a new reconciler that reconciles `BicepDeployment` resources. It will have the following control loop:

1. Check if there is an in-progress operation. If so, check its status:
    1. If the operation is still in progress, then queue another reconcile operation and continue processing.
    2. If the operation completed successfully, then update the `status.phrase` field as `Ready`. Create `BicepResource` resources for each resource in the `outputResources` field returned by the Radius API. and set their OwnerReferences fields to the current `BicepDeployment` resource and their `status.phrase` fields to `Ready`.
    3. If the operation failed, then update the `status.phrase` and `status.message` as `Failed` with the reason for the failure.
    4. Continue processing.
2. If the `BicepDeployment` is being deleted, then process deletion:
    1. Set the `status.phrase` for the BicepDeployment to `Deleting`.
    2. Check the cluster for and `BicepResource` resources that have this `BicepDeployment` as their owner. If found, set their `status.phrase` to `Deleting`.
    3. If there are no `BicepResource` resources found, then set the `status.phrase` for the BicepDeployment to `Deleted`.
    4. Continue processing.
3. If the `BicepDeployment` has the `status.phrase` of `Deleted`, then delete the resource from the cluster.
4. If the `BicepDeployment` is not being deleted then process this as a create or update:
    1. Perform some simple lookups to Radius:
        1. Check if the template contains an empty `param environment`. If it does, the controller will query the Radius API to get an environment Id (must have exactly one environment if left unspecified like this) and deploy the template using that environment.
        2. Check if the template contains an empty `param application`. If it does, the controller will query the Radius API to get an application Id. If there is exactly one application present, the controller will deploy the template using that application.
    2. Queue a PUT operation against the Radius API to deploy the ARM JSON.
    3. Continue processing.

#### BicepResource Controller

The `radius-controller` will be updated to include a new reconciler that reconciles `BicepResource` resources. It will have the following control loop:

1. Check if there is an in-progress deletion. If so, check its status:
    1. If the deletion is still in progress, then queue another reconcile operation and continue processing.
	  2. If the deletion completed successfully, then update the `status.phrase` field as `Deleted`.
	  3. If the operation failed, then update the `status.phrase` and `status.message` as `Failed`.
2. If the `BicepDeployment` is being deleted, then process deletion:
    1. Send a DELETE operation to the Radius API to delete the resource.
    2. Continue processing.

### Alternatives Considered

- Package the Bicep compilation into the Radius controller. We decided against this because it would make the controller more complex and harder to maintain. Instead, we will rely on the CLI to generate the BicepDeployment resource.

- We could create an application if one does not exist when trying to deploy the template. We decided against this because the edge cases may be difficult to build around. Instead, we will require users to specify an application in the Bicep template or have one available already if they need it.

### API design (if applicable)

#### API

There will be no changes to the Radius API.

#### CLI

There will be a new CLI command `rad bicep generate-kubernetes` that will generate a BicepDeployment resource from a Bicep template and parameters file.

### Error Handling

The `BicepDeployment` controller will handle errors by updating the `status.phrase` and `status.message` fields of the `BicepDeployment` resource. The controller will also emit events for the `BicepDeployment` resource to track the progress of the deployment.

## Test plan

We will need to write unit tests for the new controllers and integration tests to ensure that the controllers work as expected. We will also need to write tests to ensure that the `rad bicep generate-kubernetes` command works as expected. We will also need to write functional tests to ensure that the controllers work as expected.

## Security

The new controllers will use the existing security model of other Radius controllers, such as fine-grained Kubernetes RBAC permissions. The controllers will also use the Radius API to deploy resources, which will require the same authentication and authorization as other Radius API calls.

## Compatibility

The changes describes are only additive, so there should be no breaking changes or compatibility issues with other components.

## Monitoring and Logging

The new controllers will emit logs and metrics in the same way as other Radius controllers.

We will be leveraging the Kubernetes Events API to emit events for the BicepDeployment and BicepDeploymentResource resources. These events will be used to track the progress of the deployment and to provide feedback to the user.

## Development plan

- Implement `BicepDeployment` and `BicepResource` CRDs.
- Implement the `rad bicep generate-kubernetes` command.
- Write tests for the `rad bicep generate-kubernetes` command.
- Implement the `BicepDeployment` controller.
- Implement the `BicepResource` controller.
- Write unit tests for the new controllers.
- Write integration and functional tests for the new controllers.

## Open Questions

- Q: What use cases are there for integration with Kubernetes tools that we should consider when designing the BicepDeployment controller? Are there any gaps in the design that we should address?
- A: We will see how users adopt this feature set and iterate on the design based on user feedback. For now, we will implement this feature and enable the scenarios that we know about.
