# Topic: Compute Platform Extensibility for Radius

* **Author**: Will Tsai (@willtsai)

## Terms and definitions

| Term | Definition |
|------|------------|
| **Compute platform** | An environment where applications can be deployed and run, such as Kubernetes, Azure Container Instances (ACI), etc. |
| **Core types** | Built-in resource types provided by Radius, including `containers`, `gateways`, `secretStores`, `volumes`, `environments`, and `applications`. |
| **Radius Resource Type (RRT)** | A custom resource type defined separately from Radius core types and loaded into Radius without requiring a new Radius release. Formerly referred to as User Defined Types (UDTs) |
| **Recipe** | A set of instructions that provisions a Radius resource to a Radius environment. Recipes are implemented in Bicep or Terraform. |
| **Resource Provider (RP)** | A component responsible for handling create, read, update, delete, and list (CRUDL) operations for a specific resource type. |

## Topic Summary
<!-- A paragraph or two to summarize the topic area. Just define it in summary form so we all know what it is. -->
Radius will be enhanced to support multiple compute platforms, secret stores, gateway, and volume resources through a recipe-based extensibility model. This approach decouples Radius's core logic from platform-specific provisioning code. Core resource types (`containers`, `gateways`, `secretStores`, and `volumes`) will be implemented as Radius Resource Types (RRT) and will allow platform engineers to register Bicep or Terraform recipes for them. Radius will provide default recipes for Kubernetes and Azure Container Instances (ACI), but platform engineers can use, modify, or replace these to customize how Radius provisions resources to different environments, or to add support for entirely new platforms without requiring changes to Radius core.

### Top level goals
<!-- At the most basic level, what are we trying to accomplish? -->
- Provide platform engineers with the ability to deploy Radius applications to specific platforms other than the default Kubernetes, such as Azure Container Instances (ACI), and other compute, secret store, and gateway platforms.
- Provide abstraction punch-through mechanisms to allow platform engineers and developers to leverage platform-specific features and configurations not directly supported by Radius core (e.g. confidential containers).
- Provide a recipe-based platform engineering experience that is consistent for user-defined resource types and core types.
- Expand the ability to provision a single application definition to multiple clouds by adding the capability to provision to multiple compute platforms, secret stores, volumes, and gateway types.
- Architecturally separate Radius core logic from platform-specific provisioning code.
- Enable community-provided extensions to support new platforms without Radius code changes.
- Provide Recipe Packs to bundle related recipes for easier management and sharing (e.g., a pack for ACI that includes all necessary recipes).

### Non-goals (out of scope)
<!-- What are we explicitly not trying to accomplish? -->
- Running the Radius control plane on a non-Kubernetes platform, this is tracked as a separate roadmap item in [Host Radius control plane in any container platform #39](https://github.com/radius-project/roadmap/issues/39)
- Changes to portable types beyond what's necessary to support recipe-based provisioning for core functionalities.
- Implementing a new custom resource provider (RP) framework beyond the existing RRT and Recipe mechanism for this specific feature.
- Versioning support for Recipes is out of scope, this is tracked as a separate feature in [#3 Versioning for Recipes, resource types, apis, etc.](https://github.com/radius-project/roadmap/issues/3). However, versioning is in scope for the implementation of Recipe Packs from the get go, even if the underlying Recipes packaged within the Pack don't support versioning yet.

## User profile and challenges
<!-- Define the primary user and the key problem / pain point we intend to address for that user. If there are multiple users or primary and secondary users, call them out.   -->

### User persona(s)
<!-- Who is the target user? Include size/org-structure/decision makers where applicable. -->
**Platform Engineers / Infrastructure Operators**
- Work in medium to large organizations, supporting multiple development teams by providing and managing infrastructure platforms.
- Responsible for setting up, configuring, and maintaining diverse compute, storage, and networking environments, often spanning multiple clouds or hybrid setups.
- Make platform choices based on business requirements, cost optimization, technical capabilities, and existing enterprise infrastructure.
- Need to enforce organizational standards, security policies, and operational best practices across all deployed applications.
- Often tasked with enabling developers to use modern application platforms while integrating with established enterprise systems.

### Challenge(s) faced by the user
<!-- What challenges do the user face? Why are they experiencing pain and why do current offerings not meet their need? -->
- **Limited Platform Support:** Current Radius versions primarily target Kubernetes with recent hardcoded expansion to ACI, making it difficult to use Radius in organizations that have standardized on or require other compute platforms (e.g., AWS Fargate, Google Cloud Run), or different types of secret stores and gateways.
- **Lack of Customization:** The provisioning logic for core Radius resources is hard-coded, offering limited ways to customize deployments to meet specific organizational policies (e.g., specific labels, sidecars, network configurations) without forking Radius.
- **Extensibility Barriers:** Extending Radius to support new, unsupported platforms requires modifying Radius core code, which is a significant barrier for most platform teams and leads to maintenance overhead.
- **Inconsistent Tooling:** Managing applications across different platform types often requires different toolsets and deployment pipelines, increasing complexity.

### Positive user outcome
<!-- What is the positive outcome for the user if we deliver this, i.e. what is the value proposition for the user? Remember, this is user-centric. -->
As a platform engineer, I can confidently adopt Radius across my organization, knowing I can extend its capabilities to support any compute platform, secret store, gateway, or volume my teams require, without waiting for built-in support or modifying Radius core. I can register, customize, and share recipes that define how Radius provisions these resources, ensuring consistency with our application definitions while maintaining flexibility in our infrastructure choices and adhering to organizational standards. This empowers my development teams to leverage Radius benefits regardless of the underlying infrastructure.

> Note that compute extensibility will allow for one compute platform configured per Radius Environment

## Key scenarios
<!-- List ~3-7 high level scenarios to clarify the value and point to how we will decompose this big area into component capabilities. We may ultimately have more than one level of scenario. -->

### Scenario 1: Configure a non-Kubernetes Radius environment
A platform engineer initializes a new Radius environment and registers recipes for `Applications.Core/containers@2025-05-01-preview` to deploy applications to Azure Container Instances (ACI) instead of Kubernetes, allowing teams to use Radius in environments without Kubernetes clusters. They would similarly register recipes for ACI-compatible gateway and secret store solutions.

### Scenario 2: Customize platform deployment behavior for core resources
A platform engineer customizes the default Kubernetes recipes for `Applications.Core/containers@2025-05-01-preview` (and similarly for `gateways` and `secretStores`) to align with company-specific infrastructure policies, such as adding mandatory security sidecars, custom labels for cost tracking, or integrating with internal monitoring systems, by modifying the recipe and re-registering it.

### Scenario 3: Extend Radius to a new, unsupported platform
A platform engineer creates new Bicep or Terraform recipes to enable Radius to deploy `Applications.Core/containers@2025-05-01-preview` to AWS Elastic Container Service (ECS), `Applications.Core/gateways@2025-05-01-preview` to an AWS Application Load Balancer, and `Applications.Core/secretStores@2025-05-01-preview` to AWS Secrets Manager. They then register these recipes in their Radius environment.

## Key dependencies and risks
<!-- What dependencies must we take in order to enable this scenario? -->
<!-- What other risks are you aware of that need to be mitigated. If you have a mitigation in mind, summarize here. -->
<!-- **Dependency Name** – summary of dependency.  Issues/concerns/risks with this dependency -->
<!-- **Risk Name** – summary of risk.  Mitigation plan if known. If it is not yet known, no problem. -->
- **Bicep/Terraform Capabilities and Limitations**: The feasibility of implementing all necessary provisioning logic (currently in Go for Kubernetes) within Bicep or Terraform recipes.
    - Risk: Some complex logic might be difficult or impossible to replicate.
    - Mitigation: Early Proof of Concept (POC) for Kubernetes provisioning in Bicep/Terraform. Implement ACI provisioning first as a less complex target to identify limitations. Consider Terraform if Bicep has significant gaps for certain platforms.
- **Radius Application Graph Integrity**: Ensuring that recipes can correctly define and maintain relationships and connections between resources (e.g., `containers.connections`) as the current system does.
    - Risk: Recipes might not fully capture or might incorrectly represent resource relationships, leading to broken application deployments or incorrect graph data.
    - Mitigation: Design and provide reusable Bicep/Terraform modules or clear patterns for recipes to declare outputs that contribute to the Radius graph.
- **Complexity of Core Types**: The `containers` resource type, in particular, has a large and complex schema.
    - Risk: Re-implementing its provisioning via recipes could be a large effort and error-prone.
    - Mitigation: Phased approach, thorough testing, and clear documentation. Maintain versioned support for older types during transition.
- **Migration Path for Existing Users**: Users with existing Radius deployments will need a clear and manageable path to migrate to the new RRT-based core types and recipe model.
    - Risk: Breaking changes or complex migration steps could deter adoption.
    - Mitigation: Provide versioned resource types (e.g., `@2025-05-01-preview`), detailed migration guides, and potentially migration tooling.
- **Recipe Validation and Debugging**: Lack of strong, compile-time typing for recipes means invalid recipes might only be caught at registration or deployment time.
    - Risk: Poor user experience due to difficult-to-debug deployment failures.
    - Mitigation: Implement robust validation checks upon recipe registration (schema, parameters, outputs). Enhance CLI tooling for local recipe validation and testing.

## Key assumptions to test and questions to answer
<!-- If you are making assumptions that, if incorrect, would cause us to significantly alter our approach to this scenario, make them explicit here.  Also call out how / when you plan to validate key assumptions. -->
<!-- What big questions must we answer in order to clarify our plan for this scenario.  When and how do you plan to answer those questions (prototype feature x, user research, competitive research, etc.) -->
- **Assumption**: All essential provisioning logic currently in Radius's Go code for Kubernetes (for containers, gateways, secret stores) can be effectively replicated using Bicep or Terraform recipes.
    - Validation: POC implementation of Kubernetes recipes for core functionalities.
- **Assumption**: AppCore can reliably construct the application graph and provide recipes with all information needed to execute via their connections to other resources.
    - Validation: Test recipe deployments and inspect the resulting Radius graph.
- **Question**: What is the most effective way for recipes to declare their input parameters and output properties to ensure consistency and enable validation?
    - Exploration: Define a clear contract for recipes, potentially using a schema or metadata definition that can be validated.
- **Question**: How can platform engineers easily test and debug their custom recipes before registering them in a live environment?
    - Exploration: Investigate CLI tooling or local sandbox environments for recipe testing.
- **Question**: Recipe Packs seems redundant as sets of recipes can already be registered to an Environment, which effectively acts as a Recipe Pack. Is there a need for a separate concept of Recipe Packs, or can we simply guide users to bundle their Recipes into Environment definitions?
    - Answer: The convenience of Recipe packs will be needed so that platform engineers don't have to piece together individual Recipes for each environment from scratch as a single Recipe Pack may be re-used for many Environments. Putting everything together manually each time also opens up the door for error (e.g. forgetting to include gateway Recipes). The Environment definition is a good place to group *collections of Recipe packs* - e.g. bundle ACI and OpenAI Recipe packs together in an Environment definition.

## Current state
<!-- If we already have some ongoing investment in this area, summarize the current state and point to any relevant documents. -->
Radius currently has built-in support for Kubernetes as its primary compute platform, as well as initial support for ACI. Core resource types like `Applications.Core/containers`, `Applications.Core/gateways`, and `Applications.Core/secretStores` have hard-coded provisioning logic that targets Kubernetes or Azure for ACI. While Radius supports RRTs and recipes for custom resources, this mechanism is not used for the core functionalities mentioned. Extending Radius to other platforms or significantly customizing the provisioning of these core resources typically requires modifying the Radius codebase. The `environments` resource type has a hard-coded `compute` property primarily designed for Kubernetes and ACI.

## Details of user problem
<!-- <Write this in first person. You basically want to summarize what “I” as a user am trying to accomplish, why the current experience is a problem and the impact it has on me, my team, my work and or biz, etc…. i.e. “When I try to do x aspect of cloud native app development, I have the following challenges / issues….<details>. Those issues result in <negative impact those challenges / issues have on my work and or business.> -->
When I, as a platform engineer, try to use Radius to deploy and manage applications across my organization, I face significant hurdles if our infrastructure strategy involves more than just Kubernetes. For example, if a development team wants to deploy a service to AWS Elastic Container Service (ECS) for cost or simplicity, or if we need to integrate with a managed cloud gateway service instead of the default Radius gateway on Kubernetes, I can't easily make Radius do this. I'm forced to tell my teams they can only use Radius for Kubernetes, or I have to build and maintain complex workarounds outside of Radius. This means either Radius doesn't fit our diverse needs, or I'm stuck maintaining a custom version of Radius or else needing to contribute core code changes upstream in to Radius, which is a huge operational burden and makes upgrades very risky. These limitations prevent us from fully leveraging Radius as a unified application platform across our varied infrastructure.

## Desired user experience outcome
<!-- <Write this as an “I statement” that expresses the new capability from user perspective … i.e. After this scenario is implemented “I can do, x, y, z, steps in cloud native app developer and seamlessly etc... As a result <summarize positive impact on my work / business>  -->
After this feature is implemented, I can confidently use Radius as our central application platform, regardless of the specific compute, secret store, or gateway technologies we use. I can define how `Applications.Core/containers@2025-05-01-preview` (and similarly versioned `gateways`, `secretStores`, and `volumes`) are provisioned on any target platform—be it Kubernetes with our custom configurations, serverless container platforms including Azure Container Instances, Azure Container Apps, AWS ECS, Google CloudRun, or a future platform—by simply writing and registering a Bicep or Terraform recipe. I can ensure all deployments adhere to our organizational standards by embedding those standards into the recipes. I can manage these recipes like any other piece of infrastructure-as-code, versioning them, testing them, and sharing them across my teams. As a result, my development teams get a consistent experience for defining their applications, while my platform team retains control and flexibility over the underlying infrastructure, all without needing to modify Radius core or manage complex forks. This significantly reduces our operational overhead and allows us to adapt quickly to new technologies and business needs.

### Detailed user experience
 <!-- <List of steps the user goes through from the start to the end of the scenario to provide more detailed view of exactly what the user is able to do given the new capabilities>  -->
<!-- Step 1
Step 2
… -->

#### User Story 1: As a Platform Engineer, I want to create a Recipe Pack that bundles multiple recipes for core resource types, so that I can easily register and manage them in a Radius environment:

1.  **Define a Recipe Pack**:
    *   A platform engineer creates a manifest file (e.g. `recipe-pack.yaml`) that defines a collection of recipes. This manifest would list each core resource type (e.g., `Applications.Core/containers@2025-05-01-preview`, `Applications.Core/gateways@2025-05-01-preview`, `Applications.Core/secretStores@2025-05-01-preview`) and associate it with a specific Recipe (OCI URI or local path) and its default parameters.
    *   Example `recipe-pack.yaml`:
        ```yaml
        name: aci-production-pack
        version: 1.0.0
        description: "Recipe Pack for deploying to ACI in production."
        recipes:
            - resourceType: "Applications.Core/containers@2025-05-01-preview"   
            name: "aci-prod-container" # Optional: a friendly name for this recipe registration
            recipeKind: "bicep"
            recipeLocation: "oci://ghcr.io/my-org/recipes/core/aci-container:1.2.0"
            parameters:
                cpu: "1.0"
                memoryInGB: "2.0"
                environmentVariables:
                LOG_LEVEL: "Information"
                # Optional: allow platform-specific options like containerGroupProfile for ACI
                allowPlatformOptions: true
            - resourceType: "Applications.Core/gateways@2025-05-01-preview"
            name: "aci-prod-gateway"
            recipeKind: "bicep"
            recipeLocation: "oci://ghcr.io/my-org/recipes/core/aci-gateway:1.1.0"
            parameters:
                sku: "Standard_v2"
            - resourceType: "Applications.Core/secretStores@2025-05-01-preview"
            name: "aci-prod-keyvault"
            recipeKind: "bicep"
            recipeLocation: "oci://ghcr.io/my-org/recipes/azure/keyvault-secretstore:1.0.0"
            parameters:
                skuName: "premium"
        ```
        > Note: `templateKind` is changed to `recipeKind` and `templatePath` is changed to `recipeLocation`

1.  **Add the Recipe Pack to an Environment**:
    *   The platform engineer uses a new CLI command to add the entire pack to a Radius environment.
    *   Example: `rad environment update my-env --recipe-pack ./aci-production-pack.yaml`
    *   Or, if stored in a repo: `rad environment update my-env     --recipe-packs aci-production-pack='git::https://github.com/my-org/recipe-packs.git//aci-production-pack.yaml?ref=1.0.0`
    *   This command would iterate through the recipes defined in the manifest and register each one to the specified environment, similar to individual `rad recipe register` calls.
    *   Alternatively, the Recipe Pack could be added to the Environment definition file (e.g., `env.bicep`) in a `recipe-packs` property like below and then deployed using `rad deploy env.bicep`.
        ```bicep
        resource env 'Applications.Core/environments@2025-05-01-preview' = {
            name: 'my-env'
            properties: {
                recipePacks: [
                    {
                        name: 'local-k8s-pack'
                        uri: 'git::https://github.com/project-radius/resource-types-contrib.git//recipe-packs/local-k8s-pack.yaml?ref=1.0.0'
                    }
                ]
            }
        }
        ```
    > Note: If the RRT for which a Recipe in the pack is being registered does not exist, the Recipe Pack addition process should fail gracefully, indicating which RRTs are missing in the error message.

1.  **Environment Utilizes Recipes from the Pack**:
    *   Once the Recipe Pack is registered, the environment is configured with all the specified recipes for the core RRTs.
    *   When applications are deployed to this environment, Radius automatically uses the corresponding recipes from the pack to provision `Applications.Core/containers@2025-05-01-preview`, `Applications.Core/gateways@2025-05-01-preview`, and `Applications.Core/secretStores@2025-05-01-preview` resources.

1.  **Manage and Update Recipe Packs**:
    *   Platform engineers can update the Recipe Pack manifest (e.g., point to new recipe versions, change default parameters) and re-add it. The CLI could offer options to overwrite existing registrations or manage versions of the pack within the environment.
    *   Commands like `rad environment show my-env` or `rad recipe show --pack <pack-name>` would allow inspection of recipe packs.

#### User Story 2: As a platform engineer, I want to create a Radius Environment that leverages default recipe packs provided by Radius for core types, so that I can quickly set up a new environment without needing to write custom recipes:

1.  **Initialize Workspace & Environment using an `env.bicep` file**:
    * Use `rad workspace`, `group`, and `environment` commands to create a Radius Environment and/or Workspace. 
    * Define an environment in a Bicep file and then deploy it using `rad deploy env.bicep`. The new environment version (e.g., `Applications.Core/environments@2025-05-01-preview`) will not have a hard-coded `compute` kind.
    * For example, an ACI environment might look like:
        ```diff
        extension radius

        resource env 'Applications.Core/environments@2025-05-01-preview' = {
            name: 'my-aci-env'
            properties: {
        -       compute: {
        -           // compute kind is no longer hard-coded here
        -       }
                recipePacks: [
                    {
                        name: 'azure-aci-pack'
                        uri: 'git::https://github.com/project-radius/resource-types-contrib.git//recipe-packs/azure-aci-pack?ref=1.0.0'
                    }
                ]
                // providers property remains scoped to the environment
                providers: {
                    azure: {
                        scope: '/subscriptions/<>/resourceGroups/<>'
                    }
                }
            }
        }
        ```
    * Similarly, a Kubernetes environment might look like:
        ```diff
        extension radius

        resource env 'Applications.Core/environments@2025-05-01-preview' = {
            name: 'my-k8s-env'
            properties: {
        -       compute: {
        -           // compute kind is no longer hard-coded here
        -       }
        +       recipePacks: [
        +           {
        +               name: 'local-k8s-pack'
        +               uri: 'git::https://github.com/project-radius/resource-types-contrib.git//recipe-packs/local-k8s-pack.yaml?ref=1.0.0'
        +           }
        +       ]
                // providers property remains scoped to the environment
                providers: {
        +           // The Kubernetes namespace is moved here from properties.compute
        +           kubernetes: {
        +               namespace: 'default'
        +           }
               }
            }
        }
        ```
        > Note: the `compute` property is removed in favor of Recipe configurations, with the Kubernetes namespace moved to the providers section.

    * The default Recipe packs will have the `allowPlatformOptions` parameter set to `true` for core types like `Applications.Core/containers@2025-05-01-preview`, which allows platform engineers to punch through the Radius abstraction and use platform-specific options (e.g., `containerGroupProfile` for ACI) in their application definitions.

    * By default, `rad init` will register recipes for Kubernetes provisioning for core types like `Applications.Core/containers@2025-05-01-preview` so that Radius may continue providing a local kubernetes experience out of the box.

1. **Alternatively, create an environment using the `rad` CLI**:
    ```bash
    rad environment create my-aci-env \
    --recipe-packs azure-aci-pack='git::https://github.com/project-radius/resource-types-contrib.git//recipe-packs/azure-aci-pack?ref=1.0.0'
    ```

> There needs to be a set of default Recipes and Recipe Packs for core types built into Radius for Kubernetes, ACI, and AWS ECS, so that Platform Engineers can quickly set up environments without needing to create custom recipes from scratch. The default Recipes need to be in both Bicep and Terraform formats, so that platform engineers can choose the IAC they prefer as they may need to adapt the recipes to their specific use cases.

#### User Story 3: As an application developer, I want to define my application using the new RRT versions of core types, so that I can deploy my `app.bicep` across different environments without needing to understand the underlying provisioning details:

1.  **Developers Define Applications**:
    *   Application developers define their applications using the new RRT versions of core types (e.g., `resource myapp 'Applications.Core/containers@2025-05-01-preview' = { ... }`). They do not need to be aware of the underlying recipe details if default parameters are suitable.
1.  **Deploy Applications**:
    *   Developers (or CI/CD) run `rad deploy <bicep-file>`. Radius uses the registered recipes for the RRTs in the target environment to provision the resources.

> Note: Recipes should use Radius resource types to deploy containers and secrets. For example, environment variable values are stored as Kubernetes secrets today. The new recipe-based container needs to use the Application.Core/secrets resource type instead of storing secrets directly in Kubernetes secrets. 

#### User Story 4: As a platform engineer and/or application developer, I would like to leverage platform-specific capabilities

##### User Story 4a: As a platform engineer, I want to configure three environments with different recipes for the same core resource type, so that I can deploy applications with standard, confidential, and Kubernetes compute requirements:

1. Configure a Radius Environment with Recipes for Standard ACI containers:
    * The platform engineer configures `std-env` with recipes for `Applications.Core/containers@2025-05-01-preview` that deploy to standard ACI compute. These recipes will ignore the platform-specific properties on the container resource since the recipe parameter `allowPlatformOptions=false`, and log a warning.
    * Example Environment definition using Recipe packs (conceptual):
        ```diff
        resource environment 'Applications.Core/environments@2023-10-01-preview' = {
        name: 'std-env'
            properties: {
                recipePacks: [
                    {
                        name: 'azure-aci-pack'
                        uri: 'git::https://github.com/project-radius/resource-types-contrib.git//recipe-packs/azure-aci-pack.yaml?ref=1.0.0'
        +          // Platform engineers can disable the punch-through parameter for developers:
        +               parameters: {
        +                   Applications.Core/containers@2025-05-01-preview: {
        +                       // This will ignore platform specific options (e.g. containerGroupProfile for ACI)
        +                       //   specified in the container definition, such as confidential container configs
        +                       allowPlatformOptions: false
        +                   }
        +               }
                    }
                ]
                providers: {
                azure: {
                    scope: '/subscriptions/mySubscriptionId/resourceGroups/my-resource-group'
                }
                }
            }
        }
        ```

1. Configure a Radius Environment with Recipes for Confidential ACI containers:
    * The platform engineer configures `confi-env` with the `allowPlatformOptions: true` parameter for the `Applications.Core/containers@2025-05-01-preview` Recipe. These recipes will interpret the platform-specific properties on the container resource to provision confidential ACI compute. These recipes will ignore and log a warning for confidential container requests if the platform-specific properties are not applicable to the target environment (e.g. ACI containerGroupProfile configurations would be ignored if the deployment target environment is Kubernetes).
    * Example Environment definition (conceptual):
        ```diff
        resource environment 'Applications.Core/environments@2023-10-01-preview' = {
        name: 'confi-env'
            properties: {
                recipePacks: [
                    {
                        name: 'azure-aci-pack'
                        uri: 'git::https://github.com/project-radius/resource-types-contrib.git//recipe-packs/azure-aci-pack.yaml?ref=1.0.0'
        +          // Platform engineers can enable the punch-through parameter for developers:
        +               parameters: {
        +                   Applications.Core/containers@2025-05-01-preview: {
        +                       // This will apply the platform specific options (e.g. containerGroupProfile for ACI)
        +                       //   specified in the container definition, such as confidential container configs.
        +                       // This parameter is true in the default recipe pack, so technically
        +                       //   platform engineers don't need to specify this
        +                       allowPlatformOptions: true
        +                   }
        +               }
                    }
                ]
                providers: {
                azure: {
                    scope: '/subscriptions/mySubscriptionId/resourceGroups/my-resource-group'
                }
                }
            }
        }
        ```

1. Configure a Radius Environment with Recipes for Kubernetes containers:
    * The platform engineer configures `k8s-env` with recipes for `Applications.Core/containers@2025-05-01-preview` that deploy to Kubernetes. These recipes will interpret the platform-specific properties on the container resource, such as tolerations, and ignore ACI-specific properties.
    * The platform engineer can also configure Kubernetes metadata, such as labels, into the parameters of the Recipe Pack, which will be applied to all containers deployed to this environment.
    * Example Environment definition (conceptual):
        ```diff
        resource environment 'Applications.Core/environments@2023-10-01-preview' = {
        name: 'k8s-env'
            properties: {
                recipePacks: [
                    {
                        name: 'local-k8s-pack'
                        uri: 'git::https://github.com/project-radius/resource-types-contrib.git//recipe-packs/local-k8s-pack.yaml?ref=1.0.0'
                        parameters: {
                            Applications.Core/containers@2025-05-01-preview: {
                                allowPlatformOptions: true
        +                       // The Kubernetes metadata.labels at the environment level can be specified here, 
        +                       //   this will apply to all containers deployed to this environment.
        +                       // Developers can override in container definitions via platformOptions.
        +                       metadata: {
        +                           labels: {
        +                               'team.contact.name': 'frontend'
        +                           }
        +                       }
                            }
                        }
                    }
                ]
                // providers property remains scoped to the environment
                providers: {
                    // The Kubernetes namespace is moved here from properties.compute
                    kubernetes: {
                        namespace: 'default'
                    }
               }
            }
        }
        ```

##### User Story 4b: As a developer, I want to define an application that can be deployed to standard ACI, confidential ACI, and Kubernetes compute environments, without changing the core application logic, so that I can maintain a consistent application definition across different deployment targets:

This scenario demonstrates how a single application definition, containing both standard and confidential compute requirements, can be deployed to different environments, with Radius leveraging environment-specific recipes to fulfill those requirements.

1.  The developer defines an application in a Bicep file (`app.bicep`). This definition includes:
    *   A standard container (e.g., a frontend web server).
    *   A confidential container (e.g., a backend service processing sensitive data), defined through a `platformOptions.aci` property that encapsulates an ACI containerGroupProfile object which will include the confidential container configuration.
    *   A data store, like a Redis cache.
    *   A connection from the confidential backend container to the Redis cache.
    *   The recipes will ignore and log a warning for confidential container requests if the platform-specific properties are not applicable to the target environment (e.g. ACI containerGroupProfile configurations would be ignored if the deployment target environment is Kubernetes).
*   Example `app.bicep`:
    ```diff
    import radius as radius

    @description('The Radius Application ID')
    param application string

    resource frontend 'Applications.Core/containers@2025-05-01-preview' = {
        name: 'frontend'
        properties: {
        application: application
        container: {
            image: 'nginx:latest'
            ports: {
            web: {
                containerPort: 80
            }
            }
        }
        // This container is standard by default
        }
    }

    resource backend 'Applications.Core/containers@2025-05-01-preview' = {
        name: 'backend'
        properties: {
        application: application
        container: {
            image: 'mycorp/sensitive-processor:v1.2'
            ports: {
            api: {
                containerPort: 5000
            }
            }
            env: {
            REDIS_CONNECTION: cache.properties.connectionStrings.default
            }
        }
        connections: {
            cache: {
            source: cache.id
            }
        }
        extensions: [
            // Dapr sidecar definition stays the same for now
            {
                kind: 'daprSidecar'
                appId: 'frontend'
            }
            // This should be moved to a top-level property in the container definition:
    -       {
    -           kind:  'manualScaling'
    -           replicas: 5
    -       }
            // This should go into the platformOptions.kubernetes property
    -       {
    -           kind: 'kubernetesMetadata'
    -           labels: {
    -           'team.contact.name': 'frontend'
    -           }
    -       }
        ]
        // This container requests confidential compute
        +   platformOptions: {
        +       // Platform-specific properties are included here, following the respective platform's API schema
        +       //     e.g. podSpec for k8s, taskDefinition for ECS, containerGroupProfile for ACI
        +       aci: {
        +           sku: 'confidential'
                    // and any other ACI-specific properties available via the containerGroupProfile spec
        +       }
        +       kubernetes: {
        +           apiVersion: 'v1'
        +           kind: Pod
        +           // The Kubernetes metadata.labels at the container level can be specified here:
        +           metadata: {
        +               labels: {
        +                   'team.contact.name': 'frontend'
        +               }
        +           }
        +           spec: {
        +               // Kubernetes-specific PodSpec properties, such as tolerations, specified here:
        +               tolerations: [
        +                   {
        +                       key: 'example-key'
        +                       operator: 'Exists'
        +                       effect: 'NoSchedule'
        +                   }
        +               ]
        +           }
        +       }
        +   }
        }
    }

    resource cache 'Applications.Datastores/redisCaches@2023-10-01-preview' = {
        name: 'mycache'
        properties: {
        application: application
        }
    }
    ```

1. The developer deploys the same application definition to each Environment:
    *   **Deploy to Standard Environment:**
        *   `rad deploy ./app.bicep --environment std-env`
        *   The `frontend` container is deployed as a standard container.
        *   The `backend` container, despite its platform-specific confidential container property, is deployed as a standard container because the `allowPlatformOptions` parameter is set to `false` in the `std-env` environment's recipe pack for `Applications.Core/containers@2025-05-01-preview`. Kubernetes-specific properties like tolerations and labels are ignored in this case, as they are not applicable to ACI.
        *   The `cache` is deployed, and the connection between `backend` and `cache` is established.
    *   **Deploy to Confidential Environment:**
        *   `rad deploy ./app.bicep --environment confi-env`
        *   The `frontend` container is deployed as a standard container (as its definition doesn't request confidential compute).
        *   The `backend` container's platform-specific confidential container property is interpreted by the recipes in `confi-env` because the `allowPlatformOptions` parameter is set to `true`, and it is deployed as a confidential container (e.g., an ACI confidential container). Kubernetes-specific properties like tolerations and labels are ignored in this case, as they are not applicable to ACI.
        *   The `cache` is deployed, and the connection between `backend` (now confidential) and `cache` is established.

1. The developer can also deploy the same application to a Kubernetes environment:
    *   Given: platform engineers have configured a Radius Environment with a Recipe Pack that supports Kubernetes deployments and has `allowPlatformOptions` set to `true`.
    *   **Deploy to Kubernetes Environment:**
        *   `rad deploy ./app.bicep --environment my-k8s-env`
        *   The `frontend` container is deployed as a standard Kubernetes Pod.
        *   The `backend` container is deployed as a Kubernetes Pod with the specified tolerations and labels because the `allowPlatformOptions` parameter was set to `true` in the Kubernetes recipe pack by the platform engineer. ACI-specific properties are ignored in this case, as they are not applicable to Kubernetes.
        *   The `cache` is deployed, and the connection between `backend` and `cache` is established.

> This scenario highlights that the application definition remains consistent. The underlying infrastructure and specific compute capabilities (ACI standard vs. ACI confidential vs. Kubernetes) are determined by the recipes configured in the target Radius environment, allowing for flexible deployment to diverse compute platforms without altering the core application logic or Bicep code.

> Note that Radius will have to merge the configurations from the environment and container definitions with the configurations provided within the `platformOptions` property. For example, if the `platformOptions.kubernetes.metadata.labels` is specified in the container definition, it will override the labels defined at the environment level.


#### User Story 5: As a platform engineer, I want to create or update my Radius Environments using the `rad environment` and `rad recipe` commands so that I can change the resource provisioning behavior without needing to modify and redeploy the environment definition:

Given that I have created and published custom Bicep or Terraform recipes for core resource types, I want to register these recipes in a Radius environment so that they can be used by application developers to provision resources according to our organizational standards and practices.

1. **Register Recipes to an environment for a core resource type**:
    * Use `rad recipe register` to register a specific recipe for a core resource type in a specific environment.
    * Example command to register a recipe for ACI containers with platform-specific options disabled:
    ```bash
    rad recipe register default --environment my-aci-env \
    --resource-type Applications.Core/containers@2025-05-01-preview \
    --recipe-kind bicep \
    --recipe-location ghcr.io/radius-project/resource-types-contrib/core/containers/recipes/aci-container:0.48 \
    --parameters {"allowPlatformOptions": false}
    ```
    * Example command to register a recipe for Kubernetes containers with metadata labels applied and platform-specific options disabled:
    ```bash
    rad recipe register default --environment my-k8s-env \
    --resource-type Applications.Core/containers@2025-05-01-preview \
    --recipe-kind bicep \
    --recipe-location ghcr.io/radius-project/resource-types-contrib/core/containers/recipes/k8s-container:0.48 \
    --parameters metadata.labels={"team.contact.name": "frontend"}, allowPlatformOptions=false
    ```
    > Note: If there are existing recipes registered for the same resource type in the environment, this command will overwrite them with the new recipe.

1. **Register Recipe Packs to an environment**:
    * Use `rad recipe pack register` to register a recipe for a core resource type in a specific environment.
    * Example command to register a recipe pack for ACI compute with platform-specific options disabled:
    ```bash
    rad recipe pack register default --environment my-aci-env \
    --recipe-pack-location git::https://github.com/project-radius/resource-types-contrib.git//recipe-packs/azure-aci-pack.yaml?ref=1.0.0
    --parameters {"enablePlatformOptions": false}
    ```
    > Note: if there are existing recipes registered for the same resource type in the environment, this command will overwrite them with the recipes defined in the pack.

1. **List the Registered Recipes**:
    * Use `rad recipe list --environment <env-name>` to see all registered recipes for core resource types in the specified environment.
    * This command will show the resource type, recipe kind (Bicep/Terraform), and the location of the recipe (existing experience today).

**Update the Kubernetes namespace in the Environment**:
* Use `rad environment update` to change the Kubernetes namespace for an existing environment.
* Example command to update the Kubernetes namespace:
```bash
rad environment update my-k8s-env --kubernetes-namespace new-namespace
```
* This command will update the `providers.kubernetes.namespace` property in the environment definition, which will be used by Radius when deploying applications to this environment.
* The `--kubernetes-namespace` flag is also available in the `rad environment create` command to set the initial namespace when creating a new environment.


#### User Story 6: As a platform engineer, I want to contribute new Recipes and Recipe Packs for core types to the Radius repository, so that I can extend the capabilities of Radius and share my work with the community:

1.  **Understand Contribution Guidelines**:
    *   Familiarize yourself with the Radius project's general contribution guidelines (often found in `CONTRIBUTING.md`).
    *   Check for any specific guidelines related to recipes or extensibility.
2.  **Develop Your Recipe(s)**:
    *   Create high-quality, well-tested Bicep or Terraform recipes for one or more core types (`Applications.Core/containers@2025-05-01-preview`, `Applications.Core/gateways@2025-05-01-preview`, `Applications.Core/secretStores@2025-05-01-preview`).
    *   Ensure recipes are generic enough for broad use or clearly document their specific use case.
    *   Follow established patterns for parameters, outputs, and resource naming within the Radius ecosystem.
3.  **Document Your Recipe(s)**:
    *   Provide clear documentation for each recipe, including:
        *   The platform it targets.
        *   Prerequisites for using the recipe.
        *   Input parameters (with descriptions, types, and defaults).
        *   Output values and their significance.
        *   An example of how to use the recipe.
4.  **(If contributing a Recipe Pack) Define the Recipe Pack Manifest**:
    *   Create a `recipe-pack.yaml` (or similar) manifest file as described in the "Packaging and Registering a 'Recipe Pack'" scenario.
    *   Ensure all referenced recipes (OCI URIs or local paths if bundled) are correct and accessible.
    *   Document the purpose and intended use of the Recipe Pack.
5.  **Test Thoroughly**:
    *   Locally test the recipes and/or Recipe Pack in a Radius environment to ensure they deploy correctly and integrate as expected.
    *   Verify that applications deployed using these recipes function correctly.
6.  **Prepare Your Contribution**:
    *   Fork the `/resource-types-contrib` repository on GitHub.
    *   Clone your fork to your local machine.
    *   Create a new branch for your changes (e.g., `feature/my-recipes`).
    *   Add your recipe files and/or Recipe Pack manifest to the appropriate directory structure in your fork.
7.  **Submit a Pull Request (PR)**:
    *   Push your changes to your fork and open a Pull Request against the main Radius repository.
    *   In the PR description, clearly explain:
        *   What your recipes/Recipe Pack do.
        *   The problem they solve or the functionality they add.
        *   How to test them.
        *   Any relevant issue numbers.
8.  **Engage in the Review Process**:
    *   Respond to feedback and questions from Radius maintainers.
    *   Be prepared to make changes to your code, documentation, or structure based on the review.
9.  **Iteration and Merge**:
    *   Work with the maintainers to address any concerns until the PR is approved.
    *   Once approved, your contribution will be merged into the Radius repository where your contributed Recipes and/or Recipe Packs will be published to the Radius GHCR registry, making your Recipes/Recipe Pack available to the community.

> The proposal is add core RRTs and Recipes to a new repo called `radius-project/resource-types-contrib` where both core (built-in) and extended (optional) types and Recipes will be stored in a structure like: <img src="2025-06-compute-extensibility-feature-spec/types-contrib-repo-structure.png" alt="diagram showing types-contrib repo structure that includes root directories for core and extended types" width="700"/>

#### User Story 7: As a platform engineer or app developer, I want to migrate existing Radius applications to use the new RRT-based core types and recipes, so that I can take advantage of the new extensibility features without breaking existing applications:

1.  **Understand the New Model**:
    *   Platform engineers and developers review documentation on the RRT-based core types (`Applications.Core/containers@2025-05-01-preview`, `Applications.Core/gateways@2025-05-01-preview`, `Applications.Core/secretStores@2025-05-01-preview`, `Applications.Core/volumes@2025-05-01-preview`) and the recipe-driven approach.
2.  **Update Radius Environment**:
    *   Platform engineers update their Radius environment definition to use the new `Applications.Core/environments@2025-05-01-preview` (or later) resource type.
    *   They register recipes for the new RRT versions of core types in their respective environments. For users migrating from existing Kubernetes or ACI setups, they would register the default recipes provided by Radius for these platforms.
3.  **Update Application Bicep Files**:
    *   Developers modify their application Bicep files to reference the new versioned RRTs (e.g., change `Applications.Core/containers` to `Applications.Core/containers@2025-05-01-preview`).
    *   They adjust any application properties to align with the schema of the new RRTs and the parameters expected by the registered recipes.
4.  **Test Migrated Applications**:
    *   Deploy the updated application Bicep files to the upgraded Radius environment.
    *   Thoroughly test the application's functionality, connectivity, and configuration to ensure it behaves as expected under the new recipe-based provisioning.
5.  **Iterate and Adjust**:
    *   If issues arise, consult migration logs, recipe documentation, and RRT schemas to make necessary adjustments to application definitions or recipe parameters in the environment.
6.  **Update CI/CD Pipelines**:
    *   Modify CI/CD pipelines to use the updated application Bicep files and any new CLI commands or Radius versions required for the RRT model.

#### User Story 8: As an application developer, I want to mount a secret store to my container as a volume so that my application container can access secrets securely:

Given: my platform engineer has set up a Radius environment with recipes registered for `Applications.Core/Containers@2025-05-01-preview` and `Applications.Core/secretStores@2025-05-01-preview` resources. The secret store recipe is configured to provision a Key Vault or similar secret management service with the secret values provided by the developer.

1. **Define a Secret Store**:
    * The developer defines a secret store in their application Bicep file, that will leverage the default recipe to provision.
    ```bicep
    resource mySecretStore 'Applications.Core/secretStores@2025-05-01-preview' = {
        name: 'mySecretStore'
        properties: {
            environment: env.id
            application: app.id
            type: 'certificate'
            data: {
                'tls.key': {
                    value: tlskey
                }
                'tls.crt': {
                    value: tlscrt
                }
            }
        }
    }
    ```

1. **Mount the Secret Store to a Container**:
    * The developer updates their container definition to include a volume that mounts the secret store.
    ```diff
    resource myContainer 'Applications.Core/containers@2025-05-01-preview' = {
        name: 'myContainer'
        properties: {
            environment: env.id
            application: app.id
            container: {
                image: 'myContainerImage'
                volumes: {
                    secrets: {
                        kind: 'persistent'
    +                   source: mySecretStore.id
                        mountPath: '/var/secrets'
                    }
                }
            }
    }
    ```

1. **Deploy the Application**:
* The developer deploys the application using `rad deploy app.bicep --environment my-env`.
* Radius uses the registered recipe for `Applications.Core/secretStores@2025-05-01-preview` to provision the secret store (e.g., Azure Key Vault) and the recipe for `Applications.Core/containers@2025-05-01-preview` to deploy the container with the secret store mounted as a volume in the container.

> Note: this replaces the current implementation that requires users to associate a KeyVault to a volume resource before mounting the volume to the container, i.e. https://docs.radapp.io/reference/resource-schema/core-schema/volumes/azure-keyvault/

> This approach will require deprecation of the current [Azure KeyVault Radius Volumes resource type](https://docs.radapp.io/reference/resource-schema/core-schema/volumes/azure-keyvault/) where the Azure KeyVault will just be a Secret Store resource type provisioned using a recipe for Azure KeyVault that can be mounted to a Radius Container resource as a volume.

#### User Story 9: As an application developer, I want to create custom gateway resources and configure them to route traffic to my application containers, so that I can manage ingress traffic using my organization's preferred gateway solution:

Given: my platform engineer has set up a Radius environment with recipes registered for `Applications.Core/Containers@2025-05-01-preview` and `Applications.Core/gateways@2025-05-01-preview` resources. The gateway recipe is configured to provision custom gateways based on the specifications determined by my organization.

1. **Define a container for the service that needs to be exposed to the internet**:
* The developer defines a container in their application Bicep file that will be provisioned by the default recipe.
    ```bicep
    resource frontend 'Applications.Core/containers@2025-05-01-preview' = {
        name: 'frontend'
        properties: {
            environment: env.id
            application: app.id
            container: {
                image: 'myfrontendimage'
                ports: {
                    web: {
                        containerPort: 3000
                    }
                }
            }
        }
    }
    ```

1. **Define a Custom Gateway Resource**:
* The developer defines a custom gateway resource in their application Bicep file that will be provisioned by the default recipe.
    ```bicep
    resource myCustomGateway 'Applications.Core/gateways@2025-05-01-preview' = {
        name: 'myCustomGateway'
        properties: {
            environment: env.id
            application: app.id
            routes: [
                {
                    path: '/'
                    destination: 'http://${frontend.name}:3000'
                }
            ]
        }
    }
    ```

1. **Deploy the Application**:
* The developer deploys the application using `rad deploy app.bicep --environment my-env`.
* Radius uses the registered recipe for `Applications.Core/gateways@2025-05-01-preview` to provision the custom gateway and the recipe for `Applications.Core/containers@2025-05-01-preview` to deploy the container with the custom gateway configured to route traffic to the container.
    ```bash
    Building app.bicep...
    Deploying template './app.bicep' for application 'myapp' and environment 'my-env' from workspace 'default'...

    Deployment In Progress...

    Completed            myCustomGateway         Applications.Core/gateways
    Completed            frontend        Applications.Core/containers

    Deployment Complete

    Resources:
        myCustomGateway         Applications.Core/gateways
        frontend        Applications.Core/containers

        Public endpoint http://1.1.1.1.nip.io/
    ```

> Note: Contour is currently installed as a [hard dependency for Radius](https://github.com/radius-project/radius/blob/main/pkg/kubernetes/object.go#L27) to provide http routing for Kubernetes deployments. This change allows users to disable the Contour installation in their Radius environments if they want to use a different gateway solution, such as NGINX Ingress Controller or Traefik, by registering a custom recipe for `Applications.Core/gateways@2025-05-01-preview` that provisions the desired gateway solution.

#### User Story 10: As an application developer, I want to create custom volume resources and mount them to my containers, so that I can use custom storage solutions or configurations:

Given: my platform engineer has set up a Radius environment with recipes registered for `Applications.Core/Containers@2025-05-01-preview` and `Applications.Core/volumes@2025-05-01-preview` resources. The volume recipe is configured to provision custom volumes based on the specifications determined by my organization.

1. **Define a Custom Volume Resource**:
    * The developer defines a custom volume resource in their application Bicep file that will be provisioned by the default recipe.
    ```bicep
    resource myCustomVolume 'Applications.Core/volumes@2025-05-01-preview' = {
        name: 'myCustomVolume'
        properties: {
            environment: env.id
            application: app.id
        }
    }
    ```
    > Note that the configurations for this volume resource will be implemented in the recipe and not specified by the developer.

1. **Mount the Custom Volume to a Container**:
    * The developer updates their container definition to include a volume that mounts the custom volume.
    ```diff
    resource myContainer 'Applications.Core/containers@2025-05-01-preview' = {
        name: 'myContainer'
        properties: {
            environment: env.id
            application: app.id
            container: {
                image: 'myContainerImage'
                volumes: {
        +           persistentVolume: {
        +               kind: 'persistent'
        +               source: myCustomVolume.id
        +           }
                }
            }
    }
    ```

1. **Deploy the Application**:
* The developer deploys the application using `rad deploy app.bicep --environment my-env`.
* Radius uses the registered recipe for `Applications.Core/volumes@2025-05-01-preview` to provision the custom volume and the recipe for `Applications.Core/containers@2025-05-01-preview` to deploy the container with the custom volume mounted as a volume in the container.

#### User Story 11: As a platform engineer, I want a local development experience for Recipes and RRTs, so that I can iterate on custom Recipes without needing to publish them to an OCI registry:
This scenario outlines how a platform engineer can develop, test, and iterate on Recipes (and associated Radius Resource Types, if custom) using local files before publishing them.

**Local Iterative Development of a Custom Recipe for a built-in or custom RRT**

A platform engineer wants to customize the behavior of `Applications.Core/containers@2025-05-01-preview` for their specific Kubernetes setup by creating a new recipe or modifying an existing one. They want to test these changes locally before publishing the recipe to an OCI registry.

1.  **Setup Local Development Environment**:
    *   Ensure Radius CLI is installed and a local Radius control plane is running (e.g., via `rad install kubernetes` or a local kind cluster).
    *   Create or select a local Radius environment for testing (e.g., `rad env create my-dev-env` or use an existing one).
    *   The RRT for `Applications.Core/containers@2025-05-01-preview` is assumed to be already known by Radius (as it's a built-in resource type).

2.  **Create and iterate on the Recipe Locally**:
    *   Create a new or modify an existing Bicep or Terraform file for the recipe (e.g., `./my-custom-recipes/kubernetes-container/template.bicep`).
    *   Modify the Bicep/Terraform template file (e.g., `template.bicep`) to implement the desired custom logic (e.g., add specific annotations, configure a default sidecar, change resource limits).

4.  **Register the Local Recipe**:
    *   Use `rad recipe register` with the `--recipe-location` pointing to the local recipe template file (or the directory containing the template and schema). This allows testing without publishing to an OCI registry.
    *   `rad recipe register default --environment my-dev-env --resource-type Applications.Core/containers@2025-05-01-preview --recipe-location ./my-custom-recipes/kubernetes-container/template.bicep --parameters customParam=valueForDev`
    *   Radius will validate the recipe against the RRT schema and its own parameter definitions.

5.  **Define/Deploy an Application Using the Local Recipe**:
    *   Create or use an existing application Bicep file (`app.bicep`) that defines an `Applications.Core/containers@2025-05-01-preview` resource.
    *   Deploy the application to the local test environment: `rad deploy ./app.bicep --environment my-dev-env`
    *   Radius will use the locally registered recipe to provision the container.

6.  **Test and Debug**:
    *   Verify the deployment in the target platform (e.g., check Kubernetes resources using `kubectl get pods,svc,deploy -n my-app-my-dev-env`).
    *   Inspect Radius application graph: `rad app graph my-app -e my-dev-env`

7.  **Publish the Recipe (Once Satisfied)**:
    *   Once the recipe is finalized, publish it to an OCI registry (for Bicep) or a Terraform module repository.
    *   Update any environment configurations or Recipe Packs to point to the published OCI URI instead of the local path for broader use.

**Scenario: Using Recipes locally from a Cloned Git Repository**

A platform engineer clones a Git repository (e.g., a company's internal IaC repo or a community repo) that contains a collection of RRT definitions (if custom) and their corresponding Recipes as local files.

1.  **Clone the Repository**:
    *   `git clone https://github.com/my-org/radius-custom-definitions.git`
    *   `cd radius-custom-definitions`

2.  **Inspect Repository Structure**:
    *   Familiarize with how RRTs (e.g., in a `types/` directory) and recipes (e.g., in a `recipes/` directory, possibly organized by type) are laid out.

3.  **Register RRTs from Local Files (if they are custom RRTs)**:
    *   If the repository contains custom RRT definitions (e.g., `types/my-custom-widget.yaml`), register them to the desired Radius environment.
    *   `rad resource-type create myCustomWidget -f ./types/my-custom-widget.yaml`
    *   (If they want to use built-in types like `Applications.Core/containers@2025-05-01-preview`, this step can be skipped.)

4.  **Register Recipes from Local Files**:
    *   For each core type (or custom RRT) for which a recipe is provided in the cloned repository, register it using its local path.
    *   Example for a custom container recipe:
        `rad recipe register default --environment my-dev-env --resource-type Applications.Core/containers@2025-05-01-preview --recipe-location ./recipes/core/kubernetes-custom-container/template.bicep`
    *   Example for a recipe for a custom RRT:
        `rad recipe register default --environment my-dev-env --resource-type Radius.Resources/myWidget --recipe-kind bicep --recipe-location ./recipes/custom/my-widget-recipe/template.bicep`

5.  **Deploy Applications**:
    *   Developers can now define applications using these RRTs, and Radius will use the locally registered recipes from the cloned repository when deploying to `my-dev-env`.

This approach allows teams to manage and version their RRTs and recipes in Git, and easily set up development or testing environments by registering these assets directly from their local checkout without needing an intermediate OCI publishing step for every iteration.

<!-- #### User Story 12: As a platform engineer, I want platform-specific `rad init` experiences to quickly set up Radius environments with default recipes for core types based on the platform I'm targeting:

1.  **Kubernetes**:
* `rad init kubernetes` initializes a Radius environment for Kubernetes with default recipes for core types.
* Platform engineers initialize a new Radius environment for Kubernetes using the default recipes provided by Radius.
* They register any additional or modified RRTs and their associated recipes in the environment.

2.  **ACI**:
* `rad init aci` initializes a Radius environment for Azure Container Instances (ACI) with default recipes for core types.
* Platform engineers initialize a new Radius environment for ACI using the default recipes provided by Radius.
* They register any additional or modified RRTs and their associated recipes in the environment.

3.  **AWS ECS**:
* `rad init ecs` initializes a Radius environment for AWS Elastic Container Service (ECS) with default recipes for core types.
* Platform engineers initialize a new Radius environment for ECS using the default recipes provided by Radius.
* They register any additional or modified RRTs and their associated recipes in the environment.

4.  **etc. etc. for other platforms that can be added to `rad init`** -->

### Summary of key changes:
* The `compute` property in the `Applications.Core/environments` resource type is removed, allowing for extensibility through Recipes.
* The `compute.namespace` property is moved to `providers.kubernetes.namespace` in the `Applications.Core/environments` resource type.
* The following [Recipe properties](https://docs.radapp.io/reference/resource-schema/core-schema/environment-schema/#recipe-properties) are renamed for better clarity going forward:
    * `templateKind` -> `recipeKind`
    * `templatePath` -> `recipeLocation`
* Recipes accept a new `allowPlatformOptions` parameter that determines whether platform-specific configurations via their own schemas (e.g., ACI containerGroupProfile, Kubernetes podSpec) are allowed in the resource type definition.
* Ability to package and register sets of related recipes as "Recipe Packs" to simplify distribution and management. Environment definitions can reference these packs, which will include the necessary recipes for core types.
* Core types (`Applications.Core/containers`, `Applications.Core/gateways`, `Applications.Core/secretStores`, `Applications.Core/volumes`) are re-implemented as Radius Resource Types (RRT) with new, versioned resource type names (e.g., `Applications.Core/containers@2025-05-01-preview`).
* New `radius-project/resource-types-contrib` repository is created to host both core (built-in) and extended (optional) types and Recipes, allowing for community contributions and extensions.
* The `runtimes` property in the `Applications.Core/containers` resource type is replaced with a `platformOptions` property that allows for platform-specific configurations (e.g., ACI confidential containers, Kubernetes tolerations). This property will be a key-value object, with the key corresponding to the platform and the value is an object that matches the platform's API schema, e.g. `'kubernetes':{<podspec>}` or `'aci':{<containerGroupProfile>}`.
* The `{kind:  'manualScaling', replicas: 5}` extension in the `Applications.Core/containers` resource type is moved to a top-level property in the container definition, as this is a container property common to all platforms.
* The `{kind: 'kubernetesMetadata', labels: {key: value}}` extension in the `Applications.Core/containers` resource type is moved to the `platformOptions.kubernetes` property, as it is a Kubernetes-specific configuration.
* Gateway resources customization is enabled through the `Applications.Core/gateways@2025-05-01-preview` resource type, where platform engineers can implement custom gateway solutions using Recipes. Contour installation is no longer a hard dependency for Radius, allowing users to use alternative gateway solutions like NGINX Ingress Controller or Traefik.

## Key investments
<!-- List the features required to enable this scenario(s). -->

### Feature 1: RRT Implementation of Core Application Model Types
<!-- One or two sentence summary -->
Re-implement existing core types (`Applications.Core/containers`, `Applications.Core/gateways`, `Applications.Core/secretStores`, `Applications.Core/volumes`) as Radius Resource Types (RRT) with new, versioned resource type names (e.g., `Applications.Core/containers@2025-05-01-preview`). These RRTs will form the basis of the extensible application model. These new core types must support Connections and may be modified by platform engineers to allow for the configuration of platform-specific capabilities (e.g. confidential containers) in the resource types themselves.

### Feature 2: Extensible Environment Configuration
<!-- One or two sentence summary -->
Introduce a new version of the `Applications.Core/environments` resource type (e.g., `Applications.Core/environments@2025-05-01-preview`) that removes the hard-coded `compute` property and relies on recipe configurations and parameters for platform-specific settings.

### Feature 3: Default Recipes for Kubernetes and ACI
<!-- One or two sentence summary -->
Develop and provide default, production-quality Bicep/Terraform recipes for the new RRT versions of `containers`, `gateways`, `volumes` and `secretStores` that target Kubernetes (replicating current functionality) and Azure Container Instances (ACI). These will serve as out-of-the-box solutions and examples.

### Feature 4: Recipe Pack Creation and Management
<!-- One or two sentence summary -->
Implement a system for packaging and versioning sets of related recipes (Recipe Packs) to simplify distribution and management. This includes defining a clear structure for Recipe Packs and providing CLI commands for creating, updating, and applying them. Default Recipes for Kubernetes and ACI will be provided by Radius as part of the initial Recipe Packs.

### Feature 5: CLI Enhancements for Recipe Management (as necessary)
<!-- One or two sentence summary -->
Validate that `rad recipe` CLI capabilities for registering, managing, and validating recipes associated with core type RRTs are sufficient for compute extensibility scenarios. This includes robust validation of recipe parameters and outputs upon registration. Implement any necessary enhancements to the CLI to support this.

### Feature 6: Migration Path and Documentation
<!-- One or two sentence summary -->
Develop comprehensive documentation, examples, and potentially tooling to guide existing Radius users in migrating their applications and environments from the old hard-coded core types to the new RRT-based, recipe-driven model.

### Feature 7: Community Contribution Process
<!-- One or two sentence summary -->
Establish a clear process and guidelines for community members to contribute new recipes and Recipe Packs for core types, including documentation standards, testing requirements, and PR submission processes. This will encourage community engagement and extension of Radius capabilities.

### Feature 8: Initialize Radius for Specific Platforms with Default Recipes
<!-- One or two sentence summary -->
Implement a `rad init <platform>` command that initializes a Radius environment for specific platforms (e.g., Kubernetes, ACI, ECS) with default recipes for core types. This will streamline the setup process for platform engineers and developers, ensuring they have a consistent starting point for their Radius environments.

### Feature 9: Local Development and Testing of Recipes
<!-- One or two sentence summary -->
Implement a workflow for platform engineers to develop, test, and iterate on recipes locally before publishing them. This includes the ability to register local recipe files directly in a Radius environment without needing to upload them to an OCI registry, allowing for rapid development and testing cycles. This feature should also support the registration of custom RRTs if applicable.

## Notes from design discussions

### Platform specific capabilities

There is a design decision to be made regarding how advanced capabilities specific to the underlying compute platform (e.g. ACI confidential containers) should be exposed to platform engineers and developers. There were three options considered, with Option 3 being the preferred after seeking feedback from the community and users:

> Note that all options advocate for the implementation of the capabilities via Recipes.

#### Option 1: Open by default in the standard container properties
Platform-specific configurations are exposed in the standard container properties provided by Radius out of the box, in a freeform format that is up to the Recipe for the underlying platform to implement (e.g. an optional [PodSpec](https://kubernetes.io/docs/reference/kubernetes-api/workload-resources/pod-v1/#PodSpec), [ECS task definition template](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/task-definition-template.html) or [ACI ContainerGroupProfile](https://learn.microsoft.com/en-us/azure/container-instances/container-instances-reference-yaml#schema) object encapsulated in a `runtimes` property on the standard Container resource that gets passed through for the underlying Recipe to implement as it sees fit). The default Recipe will pass along the platform-specific properties to the underlying platform API, allowing developers to leverage advanced capabilities without requiring platform engineers to modify the container resource or Recipe. If platform engineers do not want to allow certain punch-throughs, they need to modify the RRT and/or Recipes. Alternatively, we can add a configuration to the Environment definition that platform engineers can set to enable or disable punch-throughs for developers per Environment.

```diff
resource backend 'Applications.Core/containers@2025-05-01-preview' = {
  name: 'backend'
  properties: {
    application: application
    container: {
      image: 'mycorp/sensitive-processor:v1.2'
      ports: {
        api: {
          containerPort: 5000
        }
      }
    }
+   runtimes: {
+        // Platform-specific properties are included here, following the respective platform's API schema
+        //     e.g. podSpec for k8s, taskDefinition for ECS, containerGroupProfile for ACI
+   }
  }
}
```

**Pros:**
- Platform specific capabilities can be leveraged using built-in properties of the standard container resource type, without requiring platform engineers to modify the container resource or Recipe.
- Provides an avenue for developers to "punch-through" the Radius abstraction, which may be especially necessary for onboarding existing (brownfield) applications on to Radius.

**Cons:**
- Exposing the entire set of container configurations in order to allow for platform specific capabilities may be too broad of an abstraction punch-through, resulting in container infrastructure configurations being split or duplicated across two places (i.e. in the container resource definition itself and in the Recipe).
- May lead to a proliferation of properties in the standard container resource type by developers, making it harder to manage and control for the platform engineering team. Such a punch-through exposed to developers can lead to misuse and complications as the container contract becomes ambiguous (i.e. platform engineers have to manage a separate contract with their developers on what is and isn't allowed in the punch-through).

#### Option 2: Closed by default, requiring platform engineers to extend the container resource
This option is where the advanced capabilities are not exposed in the standard container properties provided by Radius out of the box, but rather require platform engineers to modify the container resource definitions to enable these capabilities. This could be done by creating a custom Radius Resource Type (RRT) that extends the built-in container RRT to include platform-specific properties or configurations, and then registering a Recipe that implements the extended container resource type.

```diff
resource backend 'Applications.Core/containers@2025-05-01-preview' = {
  name: 'backend'
  properties: {
    application: application
    container: {
      image: 'mycorp/sensitive-processor:v1.2'
      ports: {
        api: {
          containerPort: 5000
        }
      }
+     // Platofrm engineers modify the resource definition to enable the advanced capabilities they need:
+     sku: 'confidential' // platform engineers add this property to enable confidential compute
    }
-   runtimes: {
-        // Remove the platform-specific properties from the core container resource type, 
-        //     instead requiring platform engineers to extend the resource type as needed.
-   }
  }
}
```

> If we go with Option 2: Closed by default, we will need to bring into scope the expansion of standard container properties to include additional common properties, such as `tags`/`labels` and `restartPolicy`, so that the majority of developer use cases are covered by the standard container resource type. Thus, platform engineers only need to extend the container resource type for advanced capabilities, rather than for common properties.

**Pros:**
- Keeps the standard container resource type clean and focused on core properties, avoiding clutter from platform-specific configurations or misuse of advanced container configurations, thus maintaining the separation of concern between platform engineers and developers.
- The Radius maintainers retain the option to add advanced capabilities to the standard container resource type in the future as needed. For example, if confidential containers become a standard offering across all platforms, the maintainers can add a `confidentialCompute` property to the standard container resource type and provide default Recipes that implement it. Properties that are common across multiple platforms (e.g. `tags`, `labels`, `restartPolicy`) can be added to the standard container resource type to cover the majority of developer use cases.

**Cons:**
- Platform engineers must modify the container resource definitions and Recipes to enable advanced capabilities, which may require additional effort and coordination, increasing the maintenance overhead.
- Limits the ability of developers to leverage advanced container features without involving platform engineers, potentially slowing down the development process.

#### Option 3 (PREFERRED): Open by default, but allow platform engineers to easily disable the punch-through via Environment configuration
This option is a hybrid approach where the advanced capabilities are exposed in the standard container properties, but their enablement is controlled by a configuration in the Radius Environment definition. Platform engineers can enable or disable the punch-through for developers on a per-Environment basis. In this implementation, the configuration would be a boolean property (e.g. `allowAdvancedContainerCapabilities`) in the Environment definition that gets passed to the Recipe when it is registered. The Recipe can then use this configuration to determine whether to apply the advanced capabilities or not. The default Environments provided by Radius would have this configuration set to `true`, allowing developers to leverage advanced capabilities out of the box. The rationale is that default Radius Environments will primarily be used for testing and experimenting with Radius features, and thus enabling the punch-through by default is preferable. We expect platform engineers to create custom Environments for production or even development use cases, where they may choose to disable the punch-through for developers.


```diff
resource environment 'Applications.Core/environments@2023-10-01-preview' = {
  name: 'myenv'
  properties: {
+       allowAdvancedContainerCapabilities: true // Platform engineers can enable or disable the punch-through for developers
    recipes: {
+       // Recipes registered here must implement the allowAdvancedContainerCapabilities property
    }
    providers: {
      azure: {
        scope: '/subscriptions/mySubscriptionId/resourceGroups/my-resource-group'
      }
    }
  }
}

resource backend 'Applications.Core/containers@2025-05-01-preview' = {
  name: 'backend'
  properties: {
    application: application
    container: {
      image: 'mycorp/sensitive-processor:v1.2'
      ports: {
        api: {
          containerPort: 5000
        }
      }
    }
+   runtimes: {
+        // Platform-specific properties are included here, following the respective platform's API schema
+        //     e.g. podSpec for k8s, taskDefinition for ECS, containerGroupProfile for ACI
+   }
  }
}
```

**Pros:**
- Platform specific capabilities can be leveraged using built-in properties of the standard container resource type, without requiring platform engineers to modify the container resource or Recipe.
- Platform engineers retain control of the enablement of advanced capabilities on a per-Environment basis, eliminating the need for modifying the core resource type.

**Cons:**
- Adds cognitive load for developers to understand when the punch-through is enabled or not, requiring them to understand how the Environment to which they are deploying their application is configured.
- Platform engineers will have an additional configuration to manage in the Environment definition.

### Recipe Packs
<!-- What role does Recipe packs play in this feature? What are solutions for bundling together multiple packs of Recipes? -->
This specification proposes the introduction of "Recipe Packs" to group related Recipes together for easier management and deployment. The convenience of Recipe packs will be needed so that platform engineers don't have to piece together individual Recipes for each environment from scratch, as a single Recipe Pack may be re-used for many Environments. Putting everything together manually each time also opens up the door for error (e.g. forgetting to include gateway Recipes).

There were several questions about whether Environments, to which platform engineers may register multiple Recipes, essentially serve the same purpose and thus negates the need for Recipe Packs. While this is a valid point, it lacks the flexibility that a dedicated Recipe Pack feature provides where a Recipe Pack can have a lifecycle independent of the Environment and thus may be re-used for many Environments. The Environment definition remains a good place to group collections of Recipe packs - e.g. bundle ACI and OpenAI Recipe packs together in an Environment definition.