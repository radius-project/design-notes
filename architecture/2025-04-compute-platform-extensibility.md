# Title

* **Author**: Brooke Hamilton

## Overview

The Radius and ACI integration aims to enable deployments of applications and containers using the Radius control plane in Kubernetes to target ACI environments. The primary goals include deploying to ACI, setting up Radius environments targeting ACI, and managing ACI-specific configurations through the CLI and dashboard. However, the initial scope excludes establishing connections between ACI containers and resources, and the ACI Service Discovery feature, which is expected by September 2025. Moreover, managing external clusters and ensuring configurations remain portable between Kubernetes and ACI are highlighted as key areas of risk. The non-goals emphasize that multi-compute support per environment and full Dapr parity will not be available initially.

## Terms and definitions

* Radius: A control plane that allows users to deploy, manage, and monitor applications and containers in various environments.
* ACI: Azure Container Instances, a service that allows users to run Docker containers on Azure without managing servers.
* Kubernetes: An open-source container orchestration system for automating software deployment, scaling, and management.
* Environment: A defined setup comprising the necessary compute, storage, and networking resources to run applications and containers.
* Application Definition File: A file that describes the configuration and deployment settings for an application (e.g., app.bicep).
* Environment Definition File: A file that describes the configuration and deployment settings for an environment (e.g., env.bicep).
* Container: A lightweight, standalone, and executable software package that includes everything needed to run a piece of software, including the code, runtime, libraries, and settings.
* Control Plane: The part of a system that manages the infrastructure and configuration of an environment.
* P0 Feature: A high-priority feature that is essential for the initial release and demonstration of the product at the Build conference.

## Objectives

* To integrate Azure Container Instances (ACI) with Radius, enabling users to deploy and manage their applications and containers in ACI environments.
* To provide a seamless user experience for deploying Radius applications to ACI, including environment setup, application definition, and deployment.
* To ensure that the P0 features are fully functional and ready for demonstration at the Build conference (May 19-22).
* To gather user feedback on the ACI functionality demonstrated at Ignite 2024 and incorporate it into the development process.

> **Issue Reference:** <!-- (If appropriate) Reference an existing issue that describes the feature or bug. -->

### Goals

* Implement the capability for the Radius control plane hosted in Kubernetes to deploy to ACI.
* Enable users to set up a Radius Environment targeted at ACI using environment definition files (e.g., env.bicep).
* Allow users to deploy Radius applications to ACI using application definition files (e.g., app.bicep).
* Support the deployment of application containers to ACI, ensuring that container definitions are platform-agnostic.
* Provide CLI and Dashboard interfaces for users to view and manage ACI containers deployed using Radius.
* Allow users to apply ACI-specific configurations to containers using extension properties.

### Non goals

* Establishing connections between ACI containers and resources.
* Defining and creating Gateways for use in ACI containers.
* Defining and creating Secret Stores for use in ACI containers.
* Adding and configuring Dapr sidecars to ACI containers.
* Defining Extenders/UDT for use in ACI containers.
* Providing a managed Radius offering in ACI.
* Hosting the Radius control plane in ACI.
* Enabling the Radius control plane hosted in ACI to deploy to Kubernetes clusters.

### User scenarios (optional)

#### User story 1

As a user, I can use the instance of the Radius control plane hosted in my Kubernetes cluster to set up an ACI Environment and target that ACI environment to deploy my Applications and Containers.

#### User story 2

As a user, I can define a new Radius Environment that specifies ACI as the underlying compute platform by creating a new Environment definition file (e.g., env.bicep) and specifying the necessary settings before deploying the environment using rad deploy env.bicep. Applications and container resources that I deploy into this Environment will be targeted at my ACI cluster.

#### User story 3

As a user, I can define a new Radius Application by creating a new Application definition file (e.g., app.bicep) and use this same application definition to deploy to an Environment with either Kubernetes or ACI as its underlying compute platform. Via the extension property in the Application definition, the user can optionally configure custom settings that are specific to ACI and get applied only when deploying to ACI. For any extension properties that are not applicable to ACI, they will be ignored.

#### User story 4

As a user, I can declare a Radius Container within the application definition (e.g., app.bicep) including all the platform-agnostic container properties, e.g., image, env, etc., and deploy the application container to an ACI cluster. Radius will create the necessary resources in ACI for the application, including container instances, NGroups, container group profiles, load balancers, DNS, subnets, etc. Note that the container definitions should be container runtime platform-agnostic, thus this same container definition can be deployed to both Kubernetes and ACI.

#### User story 5

As a user, upon the successful deployment of my application to my ACI environment, I can view the ACI containers in the CLI using the rad app graph command or via the Application Graph in the Radius Dashboard.

#### User story 6

As a user, I may set ACI-specific configurations for my containers via the extensions or runtimes properties. This allows me to make use of ACI-specific features like confidential containers or spot instances that may not be available on other container compute platforms. Note that any properties that are not applicable to the compute platform in the targeted deployment environment will be ignored.

## User Experience (if applicable)

The integration of Azure Container Instances (ACI) with Radius will bring several changes to the user experience, particularly in how users define and deploy their applications. The key changes are as follows:

1. Environment Setup: Users will now be able to set up a Radius Environment targeted at ACI using environment definition files (e.g., env.bicep). This will involve specifying ACI-specific configurations and parameters in the Bicep templates.
1. Application Definition: Users will define their applications using application definition files (e.g., app.bicep). These files can include ACI-specific properties and configurations, also know as “punch-through”, allowing users to tailor their deployments to the ACI environment.
1. Deployment Process: The deployment process will have validation to ensure that when deploying to ACI users specify only configurations and properties that are valid for ACI.
1. Container Definitions: The container definitions in the Bicep templates will be platform-agnostic, ensuring that users can deploy their containers to both Kubernetes and ACI without significant changes. However, users will have the option to apply ACI-specific configurations using extension properties in the Bicep templates.

### Sample Input:Radius Environment*

```bicep
param aciscope string = '/subscriptions/<subscription id/resourceGroups/<resource group name>'
resource env 'Applications.Core/environments@2023-10-01-preview' = {
  name: 'aci-env'
  properties: {
    compute: {
      kind: 'aci'
      resourceGroup: aciscope
    }
    providers: {
      azure: {
        scope: aciscope
      }
    }
  }
}
```

### Sample Input: Containers Type with Unsupported Connections Section

Input (this input should return an error):

```bicep
resource frontend 'Applications.Core/containers@2023-10-01-preview' = {
  name: 'frontend'
  properties: {
    application: app.id
    container: {
      image: 'ghcr.io/radius-project/samples/demo:latest'
      ports: {
        web: {
          containerPort: 3000
        }
      }
    }
    connections: {
      redis: {
        source: redis.id
      }
    }
  }
}
```

### Sample Output

```text
UnsupportedElement: The environment specified for this deployment is of type 'aci', which does not support the 'connections' setting in containers.
```

### Sample Input: Portable Resources

[Portable Resources](https://docs.radapp.io/guides/author-apps/portable-resources/overview/) are expected to work on ACI with no changes from the way they would be configured for Kubernetes:

```bicep
extension radius
@description('Specifies the environment for resources.')
param environment string

@description('Specifies the application for resources.')
param application string

resource redis 'Applications.Datastores/redisCaches@2023-10-01-preview'= {
  name: 'myredis'
  properties: {
    environment: environment
    application: application
  }
}
```

#### Output

The above example will deploy on either Kubernetes or ACI.

## Design

### High Level Design

The integration with ACI will be achieved by extending the existing Radius control plane, which will continue to be hosted in Kubernetes, to support ACI as a target deployment environment. The following summarizes the key changes and enhancements that will be made to the Radius codebase to support ACI deployments:

1. Environment Definition and Setup: The Radius control plane will be updated to recognize and process environment definition files (env.bicep) that specify ACI as the target compute platform. This will involve adding support for ACI-specific configurations and parameters in the environment setup process.
1. Application Definition and Deployment: The application definition files (app.bicep) will be extended to include ACI-specific properties and configurations. The Radius deployment engine will be enhanced to interpret these properties and deploy applications to ACI environments. This includes creating and managing ACI container groups, configuring networking, and handling ACI-specific resource settings.
1. Container Definitions: The container definitions within the application definition files will be made platform-agnostic, allowing users to deploy their containers to both Kubernetes and ACI without significant changes. The Radius deployment engine will be updated to handle the creation of ACI container instances, including setting up container groups, load balancers, DNS, and other necessary resources.
1. CLI and Dashboard Enhancements: The Radius CLI and Dashboard interfaces will be updated to support ACI deployments. This includes adding new CLI commands and Dashboard workflows for setting up ACI environments, deploying applications to ACI, and managing ACI-specific configurations. Users will be able to view and manage their ACI containers using the rad app graph command and the Application Graph in the Radius Dashboard.
1. Extension Properties: The Radius codebase will be enhanced to support extension properties in the application definition files. These properties will allow users to apply ACI-specific configurations to their containers, such as custom resource settings, networking configurations, and other ACI-specific parameters.
1. Recipe Contract Updates: The recipe contract will be updated to include ACI-specific parameters and outputs. This will ensure that the Radius deployment engine can correctly interpret and apply these parameters when deploying applications to ACI environments.
1. Error Handling and Logging: The error handling and logging mechanisms within the Radius codebase will be updated to account for ACI-specific scenarios. This includes handling errors related to ACI resource creation, configuration, and deployment, as well as providing detailed logs for troubleshooting and diagnostics.
1. Secret Management: Radius will provision an Azure KeyVault resource on behalf of the user to store and manage secrets. These secrets can then be referenced in the application definition files to inject secret values as environment variables in ACI containers or to authenticate into services.
1. Gateways: Radius will provision the necessary Azure resources to establish gateways for routing internet traffic to services hosted in ACI. This includes defining and creating gateways and secret stores for use in ACI containers.

### Architecture Diagram
<!--
Provide a diagram of the system architecture, illustrating how different
components interact with each other in the context of this proposal.

Include separate high level architecture diagram and component specific diagrams, wherever appropriate.
-->

### Detailed Design

#### Key Components and Relationships

1. Core Resource Provider
    * The central framework for handling resources in Radius
    * `datamodel` defines the core resources like Environment, Container, Gateway
    * `model` configures the application model with renderers and handlers
    * `frontend` handles API requests and communicates with the backend
    * `backend/deployment` processes deployments across different compute environments
1. Azure Container Instances (ACI) Integration
    * `renderers/aci` implements ACI-specific resource rendering logic
    * `renderers/aci/gateway` provides gateway functionality for ACI
    * `renderers/aci/manualscale` handles scaling for ACI containers
1. Kubernetes Integration
    * Standard Kubernetes resource renderers for containers, gateways, volumes
    * Extensions like Dapr support, manual scaling, and Kubernetes metadata
1. Renderer Multiplexer
    * `renderers/mux` routes requests to appropriate renderers based on compute environment
    * Selects between Kubernetes and ACI renderers based on environment configuration
1. Recipes Framework
    * `recipes/configloader` loads configurations from Environment resources
    * Supports both Kubernetes and ACI compute environments
1. Resource Handlers
    * Implement the deployment of rendered resources to target platforms
    * Handle Azure and Kubernetes resources

This architecture supports multi-cloud deployments with a unified programming model, where applications can be defined once and deployed to either Kubernetes or Azure Container Instances environments based on the environment configuration.

#### Understanding the Radius/ACI Architecture Relationships

##### Core Resource Model Flow

1. API and Data Flow
    * Client Requests → Frontend Controller → DataModel → Backend → Infrastructure
    * The frontend controllers receive API requests, convert them to internal data models, and pass them to the backend for processing
    * The backend renders these models into infrastructure-specific resources and deploys them
1. Configuration Loading
    * ConfigLoader → Environment/Application Resources → Runtime Configuration
    * The `configloader` package extracts environment and application configurations from ARM resources
    * It identifies compute platform (Kubernetes or ACI) and sets up appropriate runtime configurations

#### Renderer Architecture

The rendering system is a key part of the architecture with several important relationships:

1. Multiplexed Rendering
    * The `mux.Renderer` acts as a router that directs rendering requests to the appropriate platform-specific renderer
    * It examines the `EnvironmentComputeKind` (Kubernetes or ACI) from the environment configuration and selects the correct renderer chain
1. Layered Rendering
    * Renderers are organized in a layered pattern using composition:
   KubernetesCompute: kubernetesmetadata → manualscale → daprextension → container
   ACICompute: aci_manualscale → aci
    * Each layer handles a specific aspect of rendering, with inner renderers focusing on core functionality and outer renderers adding specialized features
1. Extension Pattern
    * Extensions like `daprextension`, `manualscale`, and `kubernetesmetadata` wrap inner renderers to add functionality
    * This follows a decorator pattern where each extension enhances the resources produced by the inner renderer

#### Resource Type Handling

1. Resource Type Registration
    * The application model (`model.ApplicationModel`) registers handlers and renderers for specific resource types
    * It maps resource types like containers and gateways to their appropriate renderers
    * Each resource type can have multiple renderers based on compute environment
1. Deployment Process
    * `deploymentprocessor` coordinates the entire rendering and deployment process
    * It manages dependencies between resources, orchestrates rendering, and handles resource deployment
    * After rendering, it uses appropriate handlers to deploy resources to target infrastructure

#### Cross-Platform Abstractions

1. Environment Abstraction
    * The environment model provides a uniform interface for different compute platforms
    * `EnvironmentCompute` can be specialized as `KubernetesCompute` or `AzureContainerInstanceCompute`
    * This allows the system to handle both platforms with consistent APIs
1. Resources and Connections
    * Resources can define connections to other resources regardless of platform
    * The renderer system translates these connections into platform-specific implementations
    * For example, container-to-database connections are implemented differently in Kubernetes vs. ACI

#### ACI-Specific Implementation

1. ACI Resource Creation
    * The ACI renderer translates Radius resources into Azure-specific resources:
    * Creates subnets, NSGs, and network profiles for container groups
    * Configures load balancers and gateways for networking
1. ACI Gateway Integration
    * `aci_gateway.Renderer` handles creating Application Gateway resources in Azure
    * Manages DNS prefixes, public IPs, and routing rules specific to ACI deployments

#### Extension Mechanism

1. Resource Extensions
    * The `datamodel.Extension` system allows resources to be extended with platform-specific capabilities
    * Extensions like `ManualScaling`, `DaprSidecar`, and `AzureContainerInstance` provide platform-specific configurations
    * The rendering system interprets these extensions to produce appropriate infrastructure resources

This architecture separates the resource model from platform-specific implementations. The system can be extended to support additional compute platforms by implementing new renderers and integrating them through the multiplexer pattern.

#### Key Type Relationships in the Multiplexer Pattern

1. Core Interface Hierarchy
    * `renderers.Renderer` is the central interface that defines the rendering contract
    * All concrete renderers implement this interface, providing a consistent API
1. Multiplexer Implementation
    * `mux.Renderer` implements the `renderers.Renderer` interface but delegates to inner renderers
    * It maintains a map of inner renderers keyed by `EnvironmentComputeKind` (Kubernetes or ACI)
    * During rendering, it examines the environment compute kind and delegates to the appropriate renderer.
1. Concrete Renderer Implementations
    * Platform-specific renderers like `aci.Renderer` and `container.Renderer` (Kubernetes) implement the same interface
    * Resource-specific renderers like `gateway.Renderer` and `aci_gateway.Renderer` handle specific resource types
1. Resource Type Hierarchy
    * All resources implement the `v1.DataModelInterface`
    * Specific resource types like `ContainerResource` and `Gateway` extend this interface
    * Renderers process these resources based on their type and produce appropriate outputs
1. Environment Configuration
    * `EnvironmentCompute` defines the compute environment with its `Kind` property
    * The multiplexer uses this to determine which inner renderer to invoke
1. Input and Output Types
    * `renderers.RenderOptions` provides the context for rendering, including environment configuration
    * `renderers.RendererOutput` represents the standardized output of the rendering process

This pattern enables the system to:

1. Handle multiple compute environments through a single interface
1. Dynamically select the appropriate renderer at runtime
1. Share common interfaces across different platform implementations
1. Easily extend the system with new compute environments by adding renderers to the map

The design follows the Strategy Pattern, where the rendering strategy is selected based on the environment configuration, while maintaining a consistent interface for the rest of the system.

### Options Considered

#### Proposed Option: Two Milestones

This option has two concurrent milestones. The first is focused on near-term goals. The second is the more traditional design --> build process.

* Milestone 1: Build upon the demo code
* Milestone 2: Concurrently design an extensibility solution

##### Advantages

* We will have a customer-ready demo faster.
* Higher-quality design informed by the demo work

##### Disadvantages

* Longer calendar schedule

#### Option 2: Design then build

This option is a traditional development cycle of design --> build.

* Design an extensibility solution.
* Implement the solution.

### API design (if applicable)

No changes from current design.

### CLI Design (if applicable)

No changes from current design.

### Implementation Details

The implementation involves conditional logic based on the compute kind. (A longer-term solution provide extensibility instead of conditional logic hard coded for each platform.)

```go
pkg/rp/v1/types.go: EnvironmentComputeKind

const (
    UnknownComputeKind EnvironmentComputeKind = "unknown"
    KubernetesComputeKind EnvironmentComputeKind = "kubernetes"
    ACIComputeKind EnvironmentComputeKind = "aci"
)
```

The “mux” renderer was introduced to hold a map of renderers that are keyed on compute kind, i.e. ACI or Kubernetes.

```go
pkg/corerp/renderers/mux/renderer.go: Renderer (29, 47)

type Renderer struct {
    Inners map[rpv1.EnvironmentComputeKind]renderers.Renderer
}

…

    if c != nil {
        switch c.Kind {
        case rpv1.KubernetesComputeKind, rpv1.ACIComputeKind:
            inner = r.Inners[c.Kind]
        default:
            err = errors.New("unsupported compute kind")
            return
        }
    }
```

This code sets up the renderers, with specific setup for ACI vs. Kubernetes.

```go
pkg/corerp/model/application_model.go (91)

    radiusResourceModel := []RadiusResourceModel{
        {
            ResourceType: container.ResourceType,
            Renderer: &mux.Renderer{
                Inners: map[rpv1.EnvironmentComputeKind]renderers.Renderer{
                    rpv1.KubernetesComputeKind: &kubernetesmetadata.Renderer{
                        Inner: &manualscale.Renderer{
                            Inner: &daprextension.Renderer{
                                Inner: &container.Renderer{
                                    RoleAssignmentMap: roleAssignmentMap,
                                },
                            },
                        },
                    },
                    rpv1.ACIComputeKind: &aci_manualscale.Renderer{
                        Inner: &aci.Renderer{},
                    },
                },
            },
        },
        {
            ResourceType: gateway.ResourceType,
            Renderer: &mux.Renderer{
                Inners: map[rpv1.EnvironmentComputeKind]renderers.Renderer{
                    rpv1.KubernetesComputeKind: &gateway.Renderer{},
                    rpv1.ACIComputeKind:        &aci_gateway.Renderer{},
                },
            },
        },
        {
            ResourceType: volume.ResourceType,
            Renderer:     volume.NewRenderer(arm),
        },
    }
```

```go
pkg/corerp/api/v20231001preview/environment_conversion.go (362)

func toEnvironmentComputeKindDataModel(kind string) (rpv1.EnvironmentComputeKind, error) {
    switch kind {
    case EnvironmentComputeKindKubernetes:
        return rpv1.KubernetesComputeKind, nil
    case EnvironmentComputeKindACI:
        return rpv1.ACIComputeKind, nil
    default:
        return rpv1.UnknownComputeKind, &v1.ErrModelConversion{PropertyName: "$.properties.compute.kind", ValidValue: "[kubernetes]"}
    }
}

func fromEnvironmentComputeKind(kind rpv1.EnvironmentComputeKind) *string {
    var k string
    switch kind {
    case rpv1.KubernetesComputeKind:
        k = EnvironmentComputeKindKubernetes
    case rpv1.ACIComputeKind:
        k = EnvironmentComputeKindACI
    default:
        k = EnvironmentComputeKindKubernetes // 2023-10-01-preview supports only kubernetes.
    }

    return &k
}
```

```go
pkg/recipes/configloader/environment.go (79)

    switch environment.Properties.Compute.(type) {
    case *v20231001preview.KubernetesCompute:
        config.Runtime.Kubernetes = &recipes.KubernetesRuntime{}
        var err error

        // Environment-scoped namespace must be given all the time.
        config.Runtime.Kubernetes.EnvironmentNamespace, err = kube.FetchNamespaceFromEnvironmentResource(environment)
        if err != nil {
            return nil, err
        }

        if application != nil {
            config.Runtime.Kubernetes.Namespace, err = kube.FetchNamespaceFromApplicationResource(application)
            if err != nil {
                return nil, err
            }
        } else {
            // Use environment-scoped namespace if application is not set.
            config.Runtime.Kubernetes.Namespace = config.Runtime.Kubernetes.EnvironmentNamespace
        }
    case *v20231001preview.AzureContainerInstanceCompute:
        config.Runtime.AzureContainerInstances = &recipes.AzureContainerInstancesRuntime{}
    default:
        return nil, ErrUnsupportedComputeKind
    }
```

```go
pkg/corerp/backend/deployment/deploymentprocessor.go: deploymentProcessor (392)

    switch env.Properties.Compute.Kind {
    case rpv1.KubernetesComputeKind:
        kubeProp := &env.Properties.Compute.KubernetesCompute

        if kubeProp.Namespace == "" {
            return renderers.EnvironmentOptions{}, errors.New("kubernetes' namespace is not specified")
        }
        envOpts.Namespace = kubeProp.Namespace

    case rpv1.ACIComputeKind:
        c := &env.Properties.Compute.ACICompute
        if c.ResourceGroup == "" {
            c.ResourceGroup = env.Properties.Providers.Azure.Scope
        }
        if c.ResourceGroup == "" {
            return renderers.EnvironmentOptions{}, errors.New("resource group is not specified")
        }
        envOpts.Compute = &env.Properties.Compute

    default:
        return renderers.EnvironmentOptions{}, fmt.Errorf("%s is unsupported", env.Properties.Compute.Kind)
    }
```

```go
pkg/corerp/api/v20231001preview/application_conversion.go: (98)
func fromAppExtensionClassificationDataModel(e datamodel.Extension) ExtensionClassification {
    switch e.Kind {
    case datamodel.KubernetesMetadata:
        var ann, lbl = fromExtensionClassificationFields(e)
        return &KubernetesMetadataExtension{
            Kind:        to.Ptr(string(e.Kind)),
            Annotations: *to.StringMapPtr(ann),
            Labels:      *to.StringMapPtr(lbl),
        }
    case datamodel.KubernetesNamespaceExtension:
        return &KubernetesNamespaceExtension{
            Kind:      to.Ptr(string(e.Kind)),
            Namespace: to.Ptr(e.KubernetesNamespace.Namespace),
        }
    case datamodel.ACIExtension:
        return &AzureContainerInstanceExtension{
            Kind:          to.Ptr(string(e.Kind)),
            ResourceGroup: to.Ptr(e.AzureContainerInstance.ResourceGroup),
        }
    }

    return nil
}
```

```go
pkg/corerp/frontend/controller/environments/createorupdateenvironment.go (98)

    if newResource.Properties.Compute.Kind == rpv1.ACIComputeKind {
        if err := e.createOrUpdateACIEnvironment(ctx, newResource); err != nil {
            return nil, err
        }
    }
```

<!--
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

The approach to error handling will stay the same, with two additional features:

## Test plan

A basic test scenario will be added to Radius that will be included in the cloud functional tests.

## Security

The Radius [Secret Store](https://docs.radapp.io/reference/resource-schema/core-schema/secretstore/) portable type will create an Azure KeyVault when deploying to ACI.

## Compatibility

* When a resource type is not supported by the compute kind, a descriptive error will be emitted.
* When a "punch through" specification is added to a bicep file to provide platform specific configuration, configurations that do not apply to the target environment will be ignored and not emit an error.

## Monitoring and Logging

No changes to monitoring and logging.

## Development plan

### Milestone 1: Add the ACI feature to Radius

Exit Criteria

* The ACI integration is merged with the main branch in the Radius repo.
* A recorded presentation exists.
* Customers can reproduce the demo on their own.

### Milestone 2: Serverless Extensibility

Exit Criteria

* An extensibility interface that allows compute platforms to be added.
* Documentation on how to create a compute environment.

## Open Questions

<!--
Describe (Q&A format) the important unknowns or things you're not sure about. 
Use the discussion to answer these with experts after people digest the 
overall design.
-->

## Design Review Notes

<!--
Update this section with the decisions made during the design review meeting. This should be updated before the design is merged.
-->