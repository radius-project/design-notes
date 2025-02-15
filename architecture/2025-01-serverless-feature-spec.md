# Topic: Serverless Container Platforms

* **Author**: Will Tsai (@willtsai)

## Topic Summary
<!-- A paragraph or two to summarize the topic area. Just define it in summary form so we all know what it is. -->
Given the importance of serverless infrastructure in the modern application landscape, it is a priority for Radius to expand beyond Kubernetes and support additional container platforms with lower operational overhead. The initial expansion will focus on support for Azure Container Instances, then AWS Elastic Container Service including AWS Fargate. This will be followed by more feature rich platforms including Azure Container Apps and eventually Google CloudRun.

This document describes the high-level overview for deploying Radius-managed applications to serverless container platforms. The goal is to provide a consistent development experience for developers regardless of which container platform the platform engineer has chosen. In other words, with serverless support in Radius, applications will now be portable between cloud providers and container platforms too.

## Terms and definitions

- [AWS Elastic Container Service (ECS)](https://aws.amazon.com/ecs/): A fully managed container orchestration service that allows you to run containerized applications. Containers are grouped into tasks (analogous to Kubernetes pods) which are then grouped into a service which managed ingress and autoscaling (analogous to a Kubernetes service of type loadBalancer). ECS can run containerized applications on EC2 virtual machines or on AWS Fargate serverless infrastructure.
- [Azure Container Instances (ACI)](https://learn.microsoft.com/en-us/azure/container-instances/): A serverless container platform service that enables you to run containers on-demand without having to manage the underlying infrastructure.
- [Azure Container Apps (ACA)](https://learn.microsoft.com/en-us/azure/container-apps/): A fully managed serverless container platform that abstracts away the need to manage containers altogether.
- ACI [Container Groups](https://learn.microsoft.com/en-us/azure/container-instances/container-instances-container-groups): A container group is a collection of containers that get scheduled on the same host machine. The containers in a container group share a lifecycle, resources, local network, and storage volumes. It's similar in concept to a pod in Kubernetes.
- ACI [nGroups](https://learn.microsoft.com/en-us/azure/container-instances/container-instance-ngroups/container-instances-about-ngroups): ACI feature that provides you with advanced capabilities for managing multiple related container groups, e.g. maintaining multiple instances, rolling upgrades, high availability, managed identity support, confidential container support, load balancing, zone rebalancing.
- ACI [Confidential Containers](https://learn.microsoft.com/en-us/azure/container-instances/container-instances-confidential-overview): ACI feature that provides a secure enclave for your containerized applications to run in a confidential computing environment.
- ACI [Spot Instances](https://learn.microsoft.com/en-us/azure/container-instances/container-instances-spot-containers-overview): ACI feature that allows you to run interruptible workloads at a reduced cost compared to the standard price by taking advantage of unused capacity in Azure datacenters.
- [User Defined Types](https://github.com/radius-project/design-notes/blob/main/architecture/2024-06-resource-extensibility-feature-spec.md) (UDT): A Radius feature that allows you to define custom resource types that can be used in your application definitions. As of this writing, the feature is a work in progress with its design document available [here](https://github.com/radius-project/design-notes/blob/main/architecture/2024-07-user-defined-types.md).

### Top level goals
<!-- At the most basic level, what are we trying to accomplish? -->
1. Genericize the Radius platform to support deployment of applications on serverless (and other) compute platforms beyond Kubernetes.
1. Enable developers to deploy and manage serverless containers in a way that is consistent with the existing Radius environment and application model.
1. Make all Radius features (Recipes, Connections, App Graph, UDT, etc.) available to engineers building applications on Radius-enabled supported serverless platforms.
1. Radius support for unopinionated (e.g. AWS Elastic Container Service, Azure Container Instances) or process-driven (Azure Container Apps, AWS Fargate, Google CloudRun) serverless container platforms.

### Non-goals (out of scope)
<!-- What are we explicitly not trying to accomplish? -->
1. Hosting the Radius control plane on serverless (and other) compute platforms outside of Kubernetes. This is a separate project tracked in the roadmap: https://github.com/radius-project/roadmap/issues/39.
1. The ability to run the Radius control plane separately from the compute cluster to which it is deploying applications. This is a separate project tracked in the roadmap: https://github.com/radius-project/roadmap/issues/42. However, given that the initial requirement is for a Kubernetes-hosted Radius control plane to deploy applications to a serverless compute platform, we might need to partially implement or at least take into consideration the separation of the Radius control plane cluster from the target deployment cluster as a part of serverless implementation.
1. Radius support for more opinionated or event-driven serverless platforms (e.g. Azure Functions, AWS Lambda, Knative, etc.).

## User profile and challenges
<!-- Define the primary user and the key problem / pain point we intend to address for that user. If there are multiple users or primary and secondary users, call them out.   -->
Primarily, our users are application developers, operators, and platform engineers who wish to leverage Radius in building and managing applications on serverless compute platforms. Secondarily, our platform engineering users may have their own internal stakeholders (e.g. developers and IT teams) for whom they are building a developer platform.

### User persona(s)
<!-- Who is the target user? Include size/org-structure/decision makers where applicable. -->
**IT Operator**: An IT operator is responsible for helping developers deploy and manage applications on serverless compute platforms. They are familiar with containerized applications and may have experience with Kubernetes and other cloud-native technologies. They are responsible for ensuring that applications are deployed and running correctly, and that the platform is secure, scalable, reliable, and follows best practices. They use Radius to preconfigure Environments and Recipes for use by the application developers that they support.

**Application Developer**: A developer building applications that need to run and be managed on serverless compute platforms. They are familiar with containerized applications (e.g. Docker), but may or may not have experience with Kubernetes and other cloud-native technologies. With Radius, they will primarily deal with application definition files (e.g. `app.bicep`) and will leverage Radius features like Resources and Connections. In many cases, their environments will be provided to them and preconfigured with the necessary settings and Recipes.

**Platform Engineer**: A platform engineer responsible for building and maintaining the developer platform that developers and operators use to build and deploy applications. They are likely familiar with Kubernetes and other cloud-native technologies, but are definitely familiar with containerized applications (e.g. Docker). They are responsible for ensuring that the platform is secure, scalable, reliable, easy to use, and enforces best practices.

**Site Reliability Engineer**: Applies updates or patches to compute infrastructure and workloads as needed to ensure the application scales appropriately to continue running without issue.

### Challenge(s) faced by the user
<!-- What challenges do the user face? Why are they experiencing pain and why do current offerings not meet their need? -->
With the additional complexity and specialized knowledge required to manage and operate Kubernetes clusters, many users are looking to serverless compute platforms as a way to simplify their infrastructure and reduce operational overhead. Services like Azure Container Apps and ECS Fargate provide a way to run containerized workloads without the need to manage the underlying infrastructure, while still providing the benefits of containers like portability and scalability. 

For application developers, particularly those deploying to Kubernetes, using Radius helps them address pain points in: making sense of complex architectures, suboptimal troubleshooting experiences, pain in dealing with a plethora of cross-platform tools, difficulties in enforcing best practices, and hindered team collaboration due to unclear separation of concerns. However, without support for additional compute platforms like serverless, developers will continue to be forced to choose a container platform prior to actually building the application. It is very difficult to move between platforms once an application has been built — even if it is containerized.

### Positive user outcome
<!-- What is the positive outcome for the user if we deliver this, i.e. what is the value proposition for the user? Remember, this is user-centric. -->
The value to the user is that they can now use Radius to build and manage their applications on serverless compute platforms and thus taking advantage of the same benefits that Radius provides for Kubernetes applications.

## Key scenarios
<!-- List ~3-7 high level scenarios to clarify the value and point to how we will decompose this big area into component capabilities. We may ultimately have more than one level of scenario. -->

### Scenario 1: Model Radius Environment, Application, and Container resources for serverless
<!-- One or two sentence summary -->
Enable the ability to define and run serverless compute with necessary extensions in a Radius [environment](https://docs.radapp.io/reference/resource-schema/core-schema/environment-schema/). 

Users can specify Radius [application](https://docs.radapp.io/reference/resource-schema/core-schema/application-schema/) definitions for applications running on serverless compute platforms. 
> Must be sure to include serverless platform specific customizations via a `runtimes` property, similar to how Kubernetes patching was implemented for [containers](https://docs.radapp.io/reference/resource-schema/core-schema/container-schema/#runtimes).

Allow for Radius abstraction of a [container](https://docs.radapp.io/reference/resource-schema/core-schema/container-schema/) resource that can be deployed to serverless compute platforms. Must be sure to include serverless platform specific configurations via [connections](https://docs.radapp.io/reference/resource-schema/core-schema/container-schema/#connections), [extensions](https://docs.radapp.io/reference/resource-schema/core-schema/container-schema/#extensions), and routing to other resources via [gateways](https://docs.radapp.io/reference/resource-schema/core-schema/gateway/).
> One future idea to explore is whether we can we build extensibility via UDT and Recipes - i.e. allow Recipes to deploy predefined Container types, which themselves can be serverless containers. In other words, developers can deploy a UDT container resource using a Recipe such that all the user has to do when declaring a container is specifying the bare minimum (e.g. name, application) and let the predefined Recipe handle the rest. This will require the implementation of UDTs (e.g. defining specific types of containers as individual resource types).

### Scenario 2: "Punch-through" to platform-specific features and incremental adoption of Radius into existing serverless applications
<!-- One or two sentence summary -->
Allow for platform-specific features to be used in Radius applications via abstraction "punch-through" mechanisms, similar to how Kubernetes-specific features are supported in Radius via [base YAML or PodSpec patching](https://docs.radapp.io/guides/author-apps/containers/overview/#kubernetes) functionalities that is achieved through the [`runtimes`](https://docs.radapp.io/reference/resource-schema/core-schema/container-schema/#runtimes) property in the container definition. For example, ACI has a confidential containers offering that may not be common across serverless platforms, but Radius should allow for this feature to be used in ACI containers.
> Stretch goal: Ability to add Radius to existing serverless applications without requiring a full rewrite of the application, similar to how Radius can be added to existing Kubernetes applications via [Kubernetes manifest](https://docs.radapp.io/tutorials/add-radius/) or [Helm chart](https://docs.radapp.io/tutorials/helm/) annotations. Note: The Kubernetes/Helm support works because Kubernetes itself is extensible. Other systems like ACI, ECS, ACA may not be extensible in the same way but we should explore if there are options to make this work.

### Scenario 3: User interfaces for serverless--Radius API, CLI, Dashboard
<!-- One or two sentence summary -->
Enable deployment and management of serverless compute resources via the existing Radius API and CLI commands. Serverless resources that are modeled in Radius should be available in the App Graph and Dashboard for visualization and management.

### Scenario 4: Support for Dapr resources
<!-- One or two sentence summary -->
Enable the ability to define and run Dapr resources in a Radius application definition file (e.g. `app.bicep`) if there is native support for Dapr built into the serverless compute platform. A user would declare a Dapr sidecar in the serverless container definition and connect it to a Dapr resource, much like how it can be done today in [Kubernetes containers](https://docs.radapp.io/guides/author-apps/dapr/how-to-dapr-building-block/).

## Key dependencies and risks
<!-- What dependencies must we take in order to enable this scenario? -->
<!-- What other risks are you aware of that need to be mitigated. If you have a mitigation in mind, summarize here. -->
<!-- **Dependency Name** – summary of dependency.  Issues/concerns/risks with this dependency -->
<!-- **Risk Name** – summary of risk.  Mitigation plan if known. If it is not yet known, no problem. -->
**Dependency: orchestrator and API for the underlying serverless platform** - For each additional serverless platform that Radius supports, we have a dependency on the underlying orchestrator for that platform. For example, for Azure Container Instances, we depend on the Azure API. For AWS Elastic Container Service, we depend on the AWS API. There is a risk that the underlying orchestrator may not provide the necessary APIs to support the Radius model, the APIs may not be stable or reliable, or the APIs may not be publicly available. This will need to be investigated as a part of the implementation.

**Dependency: ingress/gateway for Radius** - Today, Radius installs and has a direct dependency on [Contour](https://projectcontour.io/) to manage and route incoming traffic to the appropriate services within a Kubernetes cluster. The equivalent default ingress/gateway for serverless platforms will need to be evaluated and implemented. Furthermore, it should be implemented in a way that allows for flexibility in the future to support other ingress/gateway options as there is a Radius backlog item for [removing dependency on Contour](https://github.com/radius-project/radius/issues/6521).

**Risk: the serverless compute platform might not provide native Dapr support** - Given that Dapr resources are implemented as top-level resources in Radius, these can only be implemented in compute platforms that support Dapr natively.

**Risk: platform specific features that cannot be implemented in Radius** - There might be platform-specific features that cannot (or should not) be implemented in Radius. For example, Kubernetes has features for taints and tolerations that are not common across compute platforms and thus should not be implemented in Radius. This risk can be mitigated by providing mechanisms to punch-through the Radius abstraction and use platform-specific features directly, like the [Kubernetes customization options](https://docs.radapp.io/guides/author-apps/containers/overview/#kubernetes) currently available in Radius.

**Risk: Differing platforms between Radius control plane and application runtime** - There is a need to deploy and maintain a Kubernetes cluster to host the Radius control plane is needed even if the application is completely serverless and this might not work for some customers (e.g. those who absolutely do not want to use Kubernetes). For now we will accept this risk and consider alternative hosting platforms for the Radius control plane to be out of scope.

**Risk: Deploying to multiple clusters** - Given that the initial implementation for Serverless support will assume that the Radius control plane is hosted on Kubernetes and can deploy applications to serverless platforms, we might have to partially implement (or at least take into consideration) the [Inter-cluster app deployment and management](https://github.com/radius-project/roadmap/issues/42) feature that is currently in our roadmap. In other words, Radius should be able to manage applications across environments that are not on the same Kubernetes cluster as the control plane. This will be something to be explored as part of the tech design and implementation.

**Risk: Significant refactoring work anticipated** - There will be a significant amount of work to refactor the Radius code as a part of implementation, especially since the current codebase is heavily Kubernetes-centric. While this is an unavoidable risk, we will ensure that the refactoring work is done with future extensibility in mind so that it becomes easier to add support for additional compute platforms in the future.

## Key assumptions to test and questions to answer
<!-- If you are making assumptions that, if incorrect, would cause us to significantly alter our approach to this scenario, make them explicit here.  Also call out how / when you plan to validate key assumptions. -->
<!-- What big questions must we answer in order to clarify our plan for this scenario.  When and how do you plan to answer those questions (prototype feature x, user research, competitive research, etc.) -->
**Assumption**: The Radius model can be extended to support serverless compute platforms without significant changes to the existing model. Serverless platforms have orchestrators that expose APIs that can be used to model the resources in Radius. We have validated these assumptions by building a prototype for Azure Container Instances.

**Assumption**: As a result of the implementation, the refactoring done to the core Radius codebase to support Serverless container runtimes should also allow for easier extensibility of Radius to support other new platforms going forward. In other words, the work done for Serverless should be genericized within the Radius code and not resemble custom implementations for Kubernetes and select Serverless platforms.

**Assumption**: If there was an app that the platform team wanted to run on serverless (e.g. ECS, ACI) and another on Kubernetes (e.g. EKS, AKS) they would configure two different environments. Semantically, an environment is a single place, so we should not have multiple places (i.e. different container platforms) within a single environment. We think that this assumption is reasonable given multiple compute targets would require a cross container compute service discovery which users are unlikely to have the desire in setting up. We'll validate this assumption with feedback from potential users.

**Question**: How do we handle the case where the underlying serverless platform (especially opinionated ones like Azure Container Apps or AWS Fargate) does not provide the necessary APIs to support the Radius model? We will answer this question by building prototypes for additional serverless platforms and evaluating the feasibility of supporting them in Radius.

**Question**: Which are Environment scoped properties and which are Container scoped properties for each compute platform? e.g. Are confidential containers or spot instances scoped to the Environment or the individual Container? We will answer this question by consulting the documentation for each serverless platform and validating our assumptions with the implementation.

## Current state
<!-- If we already have some ongoing investment in this area, summarize the current state and point to any relevant documents. -->
There has been a proof-of-concept for ACI support in Radius built on top of the ACI nGroups features. Here is a demo of the prototype that has been created so far: https://youtu.be/NMNZE22nSQI?si=Dq7Q5WVKgHularsO&t=1201

## Detailed user experience
 <!-- <List of steps the user goes through from the start to the end of the scenario to provide more detailed view of exactly what the user is able to do given the new capabilities>  -->
<!-- Step 1
Step 2
… -->

### Step 1: Define and deploy a Radius Environment for serverless

The user defines a new Radius Environment for serverless compute by creating a new Environment definition file (e.g. `env.bicep`) and specifying the necessary settings for the serverless platform before deploying the environment using `rad deploy env.bicep`.

```diff
resource environment 'Applications.Core/environments@2023-10-01-preview' = {
  name: 'myenv'
  properties: {
    compute: {
+     // Required. The kind of container runtime to use, 
+     //  e.g. 'aci' for Azure Container Instances and 'ecs' for AWS Elastic Container Service
+     kind: 'aci'
+     // This is currently required, but may be made optional if 
+     //  it doesn't apply to all compute platforms
+     namespace: 'default'
+     // This was introduced as a part of the ACI prototype, but may be changed 
+     //  depending on the implementation, need to figure out if namespace and 
+     //  resourceGroup should be combined into a single more generic property
+     resourceGroup: '/subscriptions/.../resourceGroups/myrg'
    }
+   // one provider per environment    
+   providers: {
+     // if the compute platform for this environment is ACI, then the provider must be Azure
+     azure: {
+       scope: '/subscriptions/.../resourceGroups/myrg'
+     }
+     // if the compute platform for this environment is ECS, then the provider must be AWS
+     aws: {
+       scope: '/planes/aws/aws/accounts/${account}/regions/${region}'
+     }
+   }
+   extensions: [ // these are all currently Kubernetes-centric, will need to update for serverless
      {
        kind: 'kubernetesMetadata'
        labels: {
          'team.contact.name': 'frontend'
        }
+       // Add other serverless platform-specific extensions here
      }
    ]
  }
}
```

> Note: The `providers` property in the environment is necessary to specify the scope of the environment and is different from the provider credentials that were registered using `rad init`. The provider credentials are used to authenticate with the underlying compute provider, while the `providers` environment property is used to specify the scope of the environment. Thus, we might consider renaming the Environment `providers` property to avoid confusion.

> Note: We need to evaluate if we should add an additional `kubernetes` provider to the environment definition, though this might only be applicable for the [Inter-cluster app deployment and management](https://github.com/radius-project/roadmap/issues/42) feature.

> Note: Currently it's one entry per `extension` for Kubernetes, but as a part of implementation we should determine if this needs to be further nested into one `extension` per platform and within each platform's `extension` can have multiple entries.

### Step 2: Define a Radius Application for serverless

The user defines a new Radius Application for serverless compute by creating a new Application definition file (e.g. `app.bicep`) and specifying the necessary settings for the container runtime platform. Note that the application definitions are container runtime platform-agnostic, thus this same application definition can be deployed to both Kubernetes and serverless compute platforms.

```diff
@description('The environment ID of your Radius Application. Set automatically by the rad CLI.')
param environment string

resource app 'Applications.Core/applications@2023-10-01-preview' = {
  name: 'myapp'
  properties: {
    environment: environment
+   extensions: [ // these are all currently Kubernetes-centric, will need to update for serverless
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
+     // add other serverless extensions as applicable
    ]
  }
}
```

### Step 3: Define a Radius Container for serverless

The user defines a Radius Container within the application definition (e.g. `app.bicep`) and specifies relevant container properties, such as `extensions` or `runtimes` to set platform specific configurations. Note that the container definitions are container runtime platform-agnostic, thus this same container definition can be deployed to both Kubernetes and serverless compute platforms if common functionalities across compute platforms are used.

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
+       // Add ACI-specific properties here to punch-through the Radius abstraction, e.g. sku, osType, etc.
+       sku: 'Confidential' // 'Standard', 'Dedicated', etc.
+     }
+     ecs: {
+       // Add AWS ECS-specific properties here to punch-through the Radius abstraction
+     }
+      }
  }
}
```

> Note: We assume that the `sku: 'Confidential'` property for ACI (and other comparable properties across platforms like ECS) is scoped to the individual container and not the entire application or environment. We will need to validate this assumption as a part of the implementation.

> Note: There are a certain set of core properties that would be common across all container runtimes (e.g. `image`, `ports`, `env`) and these should be included as top-level properties within the container definition and should not be ignored if provided, assuming that these core properties are limited to ones that are universal across container platforms. The properties that may not be universal across platforms (e.g. `sku: 'Confidential'`, `osType`) should not be top-level properties but should rather be encapsulated in the `runtimes` property.

> Note: To make the app definition portable, we must allow several `runtimes` to be declared per container and these should all be optional for cases where the user wants to punch through the Radius abstraction. If they declare a `runtimes` property that doesn't match the targeted deployment environment's compute, we should simply ignore that property for that deployment. If they haven't declared any `runtimes` property that match the compute of the targeted deployment environment, then we should deploy their container assuming that no `runtimes` property was provided and thus no "punch-through" behavior will be applied.

> Schema references:
> - AWS ECS task definition template reference: https://docs.aws.amazon.com/AmazonECS/latest/developerguide/task-definition-template.html 
> - AWS ECS API reference: https://docs.aws.amazon.com/pdfs/AmazonECS/latest/APIReference/ecs-api.pdf#Welcome
> - ACI schema reference: https://learn.microsoft.com/en-us/azure/container-instances/container-instances-reference-yaml
> - ACI Container Group Profiles: https://learn.microsoft.com/en-us/azure/container-instances/container-instance-ngroups/container-instances-about-ngroups#container-group-profile-cg-profile

### Step 4: Define and connect other resources to the serverless container

The user defines other resources (e.g. databases, message queues, etc.) that the container can connect to. These resources can be defined in the same application definition file (e.g. `app.bicep`) and connected to the container using the `connections` property for serverless containers just as they can be today for Kubernetes containers.

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
    }
    runtimes: {
+     aci: {
+       // Add ACI-specific properties here to punch-through the Radius abstraction, e.g. sku, osType, etc.
+       sku: 'Confidential' // 'Standard', 'Dedicated', etc.
+     }
+     ecs: {
+       // Add AWS ECS-specific properties here to punch-through the Radius abstraction
+     }
+      }
+   // connections to other resources remains the same for serverless containers as they are for Kubernetes containers
+   connections: {
+     redis: {
+       source: db.id
+     }
+   }
  }
}

+ // Definitions for other resources that the container can connect to remain unchanged
resource db 'Applications.Datastores/redisCaches@2023-10-01-preview' = {
  name: 'db'
  properties: {
    application: application
    environment: environment
  }
}
```

### Step 5: Define a Dapr sidecar for the serverless container and connect it to a Dapr resource

> Note: This step is only applicable if the serverless compute platform has native support for Dapr.

The user defines a Dapr sidecar in the container definition and connects it to a Dapr resource.

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
    }
    runtimes: {
+     aci: {
+       // Add ACI-specific properties here to punch-through the Radius abstraction, e.g. sku, osType, etc.
+       sku: 'Confidential' // 'Standard', 'Dedicated', etc.
+     }
+     ecs: {
+       // Add AWS ECS-specific properties here to punch-through the Radius abstraction
+     }
+   }
+   extensions: [
+     {
+       kind: 'daprSidecar'
+       appId: 'demo'
+       appPort: 3000
+     }
+   ]
+   // connections to other resources remains the same for serverless containers as they are for Kubernetes containers
+   connections: {
+     redis: {
+       source: daprStateStore.id
+     }
+   }
  }
}

+ // Definitions for Dapr resources that the container can connect to remain unchanged
resource daprStateStore 'Applications.Dapr/stateStores@2023-10-01-preview' = {
  name: 'statestore'
  properties: {
    environment: environment
    application: application
  }
}
```

### Step 6: Deploy the application to the serverless platform

The user deploys the application to the serverless platform by running the `rad run` or `rad deploy` command with the application definition file targeting the serverless environment (e.g. `app.bicep`).

### Step 7: View the serverless containers in the CLI and Radius Dashboard

After successful deployment, the user can view the serverless containers in the CLI using the `rad app graph` command or via the Application Graph in the Radius Dashboard.

## Key investments
<!-- List the features required to enable this scenario(s). -->

### Feature 1: Model Radius Environment resources for serverless
<!-- One or two sentence summary -->
Add support for defining and deploying serverless compute resources in a Radius Environment definition file (e.g. `env.bicep`).

### Feature 2: Model Radius Application resources for serverless
<!-- One or two sentence summary -->
Add support for defining serverless compute resources in a Radius Application definition file (e.g. `app.bicep`).

### Feature 3: Model Radius Container resources for serverless
<!-- One or two sentence summary -->
Add support for defining serverless container resources for container functionalities that are common across all platforms (e.g. `image`, `env`, `volumes`, etc.) within a Radius application definition file. 

### Feature 4: Punch-through to platform-specific features
<!-- One or two sentence summary -->
Add support for platform-specific features for containers via abstraction "punch-through" mechanisms. This will allow users to use platform-specific features, such as confidential containers or spot instances, in Radius applications.

> This is similar to how Kubernetes-specific features are supported in Radius via base YAML or [PodSpec patching](https://docs.radapp.io/guides/author-apps/kubernetes/patch-podspec/) functionalities.

> For example, in AWS ECS, we would support punch-through via the [task definition template](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/task-definition-template.html)

> For example, in ACI, we would support punch-through via the [Container Group Profiles](https://learn.microsoft.com/en-us/azure/container-instances/container-instance-ngroups/container-instances-about-ngroups#container-group-profile-cg-profile)

### Feature 5: Support for Dapr resources and sidecars
<!-- One or two sentence summary -->
Add support for adding a Dapr sidecar to a Radius container definition and connecting it to a Dapr resource. This will only be applicable if the serverless compute platform has native support for Dapr.

### Feature 6: User interfaces for serverless--Radius API, CLI, Dashboard
<!-- One or two sentence summary -->
Add support for deploying and managing serverless compute resources via the existing Radius API and CLI commands. Serverless resources that are modeled in Radius should be available in the App Graph and Dashboard for visualization and management.