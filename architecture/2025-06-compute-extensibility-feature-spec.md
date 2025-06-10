# Topic: Compute Platform Extensibility for Radius

* **Author**: Will Tsai (@willtsai)

## Terms and definitions

| Term | Definition |
|------|------------|
| **Compute platform** | An environment where applications can be deployed and run, such as Kubernetes, Azure Container Instances (ACI), etc. |
| **Core types** | Built-in resource types provided by Radius, including `containers`, `gateways`, `secretStores`, `environments`, and `applications`. |
| **User-defined type (UDT)** | A custom resource type defined separately from Radius core types and loaded into Radius without requiring a new Radius release. |
| **Recipe** | A set of instructions that provisions a Radius resource to a Radius environment. Recipes are implemented in Bicep or Terraform. |
| **Resource Provider (RP)** | A component responsible for handling create, read, update, delete, and list (CRUDL) operations for a specific resource type. |

## Topic Summary
<!-- A paragraph or two to summarize the topic area. Just define it in summary form so we all know what it is. -->
Radius will be enhanced to support multiple compute platforms, secret stores, and gateway resources through a recipe-based extensibility model. This approach decouples Radius's core logic from platform-specific provisioning code. Core resource types (`containers`, `gateways`, and `secretStores`) will be implemented as User-Defined Types (UDTs) and will allow platform engineers to register Bicep or Terraform recipes for them. Radius will provide default recipes for Kubernetes and Azure Container Instances (ACI), but platform engineers can use, modify, or replace these to customize how Radius provisions resources to different environments, or to add support for entirely new platforms without requiring changes to Radius core.

### Top level goals
<!-- At the most basic level, what are we trying to accomplish? -->
- Provide platform engineers with the ability to deploy Radius applications to specific platforms other than the default Kubernetes, such as Azure Container Instances (ACI), and other compute, secret store, and gateway platforms.
- Provide abstraction punch-through mechanisms to allow platform engineers and developers to leverage platform-specific features and configurations not directly supported by Radius core (e.g. confidential containers).
- Provide a recipe-based platform engineering experience that is consistent for user-defined types and core types.
- Expand the ability to provision a single application definition to multiple clouds by adding the capability to provision to multiple compute platforms, secret stores, and gateway types.
- Architecturally separate Radius core logic from platform-specific provisioning code.
- Enable community-provided extensions to support new platforms without Radius code changes.

### Non-goals (out of scope)
<!-- What are we explicitly not trying to accomplish? -->
- Running the Radius control plane on a non-Kubernetes platform.
- Changes to portable types beyond what's necessary to support recipe-based provisioning for core functionalities.
- Implementing a new custom resource provider (RP) framework beyond the existing UDT and recipe mechanism for this specific feature.
- Versioning support for Recipes is out of scope, this is tracked as a separate feature in [#3 Versioning for Recipes, resource types, apis, etc.](https://github.com/radius-project/roadmap/issues/3)

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
As a platform engineer, I can confidently adopt Radius across my organization, knowing I can extend its capabilities to support any compute platform, secret store, or gateway my teams require, without waiting for built-in support or modifying Radius core. I can register, customize, and share recipes that define how Radius provisions these resources, ensuring consistency with our application definitions while maintaining flexibility in our infrastructure choices and adhering to organizational standards. This empowers my development teams to leverage Radius benefits regardless of the underlying infrastructure.

## Key scenarios
<!-- List ~3-7 high level scenarios to clarify the value and point to how we will decompose this big area into component capabilities. We may ultimately have more than one level of scenario. -->

### Scenario 1: Configure a non-Kubernetes Radius environment
A platform engineer initializes a new Radius environment and registers recipes for `Applications.Core/containers@2025-05-01-preview` to deploy applications to Azure Container Instances (ACI) instead of Kubernetes, allowing teams to use Radius in environments without Kubernetes clusters. They would similarly register recipes for ACI-compatible gateway and secret store solutions.

### Scenario 2: Customize platform deployment behavior for core resources
A platform engineer customizes the default Kubernetes recipes for `Applications.Core/containers@2025-05-01-preview` (and similarly for `gateways` and `secretStores`) to align with company-specific infrastructure policies, such as adding mandatory security sidecars, custom labels for cost tracking, or integrating with internal monitoring systems, by modifying the recipe and re-registering it.

### Scenario 3: Extend Radius to a new, unsupported platform
A platform engineer creates new Bicep or Terraform recipes to enable Radius to deploy `Applications.Core/containers@2025-05-01-preview` to AWS Fargate, `Applications.Core/gateways@2025-05-01-preview` to an AWS Application Load Balancer, and `Applications.Core/secretStores@2025-05-01-preview` to AWS Secrets Manager. They then register these recipes in their Radius environment.

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
- **Migration Path for Existing Users**: Users with existing Radius deployments will need a clear and manageable path to migrate to the new UDT-based core types and recipe model.
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
- **Assumption**: Recipes can reliably output the necessary information for Radius to construct and maintain the application graph, including resource connections.
    - Validation: Test recipe deployments and inspect the resulting Radius graph.
- **Question**: What is the most effective way for recipes to declare their input parameters and output properties to ensure consistency and enable validation?
    - Exploration: Define a clear contract for recipes, potentially using a schema or metadata definition that can be validated.
- **Question**: How can platform engineers easily test and debug their custom recipes before registering them in a live environment?
    - Exploration: Investigate CLI tooling or local sandbox environments for recipe testing.

## Current state
<!-- If we already have some ongoing investment in this area, summarize the current state and point to any relevant documents. -->
Radius currently has built-in support for Kubernetes as its primary compute platform, as well as initial support for ACI. Core resource types like `Applications.Core/containers`, `Applications.Core/gateways`, and `Applications.Core/secretStores` have hard-coded provisioning logic that targets Kubernetes or Azure for ACI. While Radius supports User-Defined Types (UDTs) and recipes for custom resources, this mechanism is not used for the core functionalities mentioned. Extending Radius to other platforms or significantly customizing the provisioning of these core resources typically requires modifying the Radius codebase. The `environments` resource type has a hard-coded `compute` property primarily designed for Kubernetes and ACI.

## Details of user problem
<!-- <Write this in first person. You basically want to summarize what “I” as a user am trying to accomplish, why the current experience is a problem and the impact it has on me, my team, my work and or biz, etc…. i.e. “When I try to do x aspect of cloud native app development, I have the following challenges / issues….<details>. Those issues result in <negative impact those challenges / issues have on my work and or business.> -->
When I, as a platform engineer, try to use Radius to deploy and manage applications across my organization, I face significant hurdles if our infrastructure strategy involves more than just Kubernetes. For example, if a development team wants to deploy a service to AWS Elastic Container Service (ECS) for cost or simplicity, or if we need to integrate with a managed cloud gateway service instead of the default Radius gateway on Kubernetes, I can't easily make Radius do this. I'm forced to tell my teams they can only use Radius for Kubernetes, or I have to build and maintain complex workarounds outside of Radius. This means either Radius doesn't fit our diverse needs, or I'm stuck maintaining a custom version of Radius or else needing to contribute core code changes upstream in to Radius, which is a huge operational burden and makes upgrades very risky. These limitations prevent us from fully leveraging Radius as a unified application platform across our varied infrastructure.

## Desired user experience outcome
<!-- <Write this as an “I statement” that expresses the new capability from user perspective … i.e. After this scenario is implemented “I can do, x, y, z, steps in cloud native app developer and seamlessly etc... As a result <summarize positive impact on my work / business>  -->
After this feature is implemented, I can confidently use Radius as our central application platform, regardless of the specific compute, secret store, or gateway technologies we use. I can define how `Applications.Core/containers@2025-05-01-preview` (and similarly versioned `gateways` and `secretStores`) are provisioned on any target platform—be it Kubernetes with our custom configurations, Azure Container Instances, AWS Fargate, or a future platform—by simply writing and registering a Bicep or Terraform recipe. I can ensure all deployments adhere to our organizational standards by embedding those standards into the recipes. I can manage these recipes like any other piece of infrastructure-as-code, versioning them, testing them, and sharing them across my teams. As a result, my development teams get a consistent experience for defining their applications, while my platform team retains control and flexibility over the underlying infrastructure, all without needing to modify Radius core or manage complex forks. This significantly reduces our operational overhead and allows us to adapt quickly to new technologies and business needs.

### Detailed user experience
 <!-- <List of steps the user goes through from the start to the end of the scenario to provide more detailed view of exactly what the user is able to do given the new capabilities>  -->
<!-- Step 1
Step 2
… -->
#### Setting up and deploying an application to a Radius environment using existing Recipes for core types:

1.  **Discover existing Recipes**:
    *   Find community-provided recipes for desired platforms (e.g., ACI, AWS Fargate) from an OCI registry or Terraform module repo.
1.  **Initialize Workspace & Environment (as Platform Engineer)**:
    *   Use `rad workspace`, `group`, and `environment` commands to create a Radius Environment and/or Workspace. 
    *   Define an environment in a Bicep file and then deploy it using `rad deploy env.bicep`. The new environment version (e.g., `Applications.Core/environments@2025-05-01-preview`) will not have a hard-coded `compute` kind. For example, an ACI environment might look like:
        ```diff
        extension radius

        resource env 'Applications.Core/environments@2025-05-01-preview' = {
            name: 'my-aci-env'
            properties: {
        -       compute: {
        -           // compute kind is no longer hard-coded here
        -       }
                recipes: {
                    'Applications.Core/containers': {
                        default: {
                        templateKind: 'bicep'
                        plainHttp: true
                        templatePath: 'ghcr.io/radius-project/recipes/azure/aci-container:latest'
                        parameters: {
                            defaultCpu: 1
                            defaultMemoryInGB: 2
                        }
                        }
                    }
                    'Applications.Core/gateways': {
                        default: {
                            templateKind: 'bicep'
                            plainHttp: true
                            templatePath: 'ghcr.io/radius-project/recipes/azure/aci-gateway:latest'
                            parameters: {
                                defaultCpu: 1
                                defaultMemoryInGB: 2
                            }
                        }
                    }
                    'Applications.Core/secretStores': {
                        default: {
                            templateKind: 'bicep'
                            plainHttp: true
                            templatePath: 'ghcr.io/radius-project/recipes/azure/aci-keyvault:latest'
                            parameters: {
                                defaultSku: 'standard'
                            }
                        }
                    }
                    'Applications.Datastores/redisCaches': {
                        default: {
                            templateKind: 'bicep'
                            plainHttp: true
                            templatePath: 'ghcr.io/radius-project/recipes/azure/rediscaches:latest'
                        }
                    }
                }
        -       providers: {
        -           // provider configurations no longer hard-coded here
        -       }
            }
        }
        ```
        > Note: the `compute` and `providers` properties are removed in favor of Recipe configurations for all the core resource types registered to the Environment.
    *   By default, `rad init` might still register UDTs and recipes for Kubernetes provisioning for core types like `Applications.Core/containers@2025-05-01-preview` to provide a local kubernetes experience out of the box.
1.  **Developers Define Applications**:
    *   Application developers define their applications using the new UDT versions of core types (e.g., `resource myapp 'Applications.Core/containers@2025-05-01-preview' = { ... }`). They do not need to be aware of the underlying recipe details if default parameters are suitable.
1.  **Deploy Applications**:
    *   Developers (or CI/CD) run `rad deploy <bicep-file>`. Radius uses the registered recipes for the UDTs in the target environment to provision the resources.

#### Deploying an application with mixed standard and confidential containers

This scenario demonstrates how a single application definition, containing both standard and confidential compute requirements, can be deployed to different environments, with Radius leveraging environment-specific recipes to fulfill those requirements.

1.  **Platform Engineer: Configure Environments with Appropriate Recipes**
    *   **Standard Environment (`std-env`):**
        *   The platform engineer configures `std-env` with recipes for `Applications.Core/containers@2025-05-01-preview` that deploy to standard compute (e.g., regular ACI or Kubernetes pods). These recipes might ignore or log a warning for confidential container requests if they don't support them.
        *   Example recipe registration (conceptual):
            ```bash
            rad recipe register std-container-recipe --environment std-env \
              --resource-type Applications.Core/containers@2025-05-01-preview \
              --template-path oci://ghcr.io/radius-project/recipes/core/aci-standard-container:1.0.0
            rad recipe register std-redis-recipe --environment std-env \
              --resource-type Applications.Datastores/redisCaches \
              --template-path oci://ghcr.io/radius-project/recipes/azure/redis:1.0.0
            ```
    *   **Confidential Environment (`confi-env`):**
        *   The platform engineer configures `confi-env` with specialized recipes for `Applications.Core/containers@2025-05-01-preview` that support deploying confidential containers (e.g., ACI Confidential Containers). These recipes will interpret a specific property on the container resource to provision confidential compute.
        *   Example recipe registration (conceptual):
            ```bash
            rad recipe register confi-container-recipe --environment confi-env \
              --resource-type Applications.Core/containers@2025-05-01-preview \
              --template-path oci://ghcr.io/radius-project/recipes/core/aci-confidential-container:1.0.0
            rad recipe register confi-redis-recipe --environment confi-env \
              --resource-type Applications.Datastores/redisCaches \
              --template-path oci://ghcr.io/radius-project/recipes/azure/redis:1.0.0 
            ```
            (Note: The Redis recipe might be the same if its deployment doesn't change based on the compute's confidentiality.)

2.  **Developer: Define Application with Mixed Container Types and Connections**
    *   The developer defines an application in a Bicep file (`app.bicep`). This definition includes:
        *   A standard container (e.g., a frontend web server).
        *   A confidential container (e.g., a backend service processing sensitive data), marked with a property like `confidential: true` (the exact property name and structure to be defined by the UDT schema).
        *   A data store, like a Redis cache.
        *   A connection from the confidential backend container to the Redis cache.
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
               // This should go into the runtimes.kubernetes property
        -       {
        -           kind: 'kubernetesMetadata'
        -           labels: {
        -           'team.contact.name': 'frontend'
        -           }
        -       }
            ]
            // This container requests confidential compute
        +   runtimes: {
        +       aci: {
        +           // Add ACI-specific properties here to punch-through the Radius abstraction, e.g. sku, osType, etc.
        +           sku: 'Confidential' // 'Standard', 'Dedicated', etc.
        +       }
        +   }
          }
        }

        resource cache 'Applications.Datastores/redisCaches@2023-10-01-preview' = { // Assuming existing Redis type
          name: 'mycache'
          properties: {
            application: application
            // Redis specific properties
          }
        }
        ```

3.  **Developer/CI/CD: Deploy the Same Application Definition to Both Environments**
    *   **Deploy to Standard Environment:**
        *   `rad deploy ./app.bicep --environment std-env`
        *   Radius uses the `std-container-recipe` registered in `std-env`.
        *   The `frontend` container is deployed as a standard container.
        *   The `backend` container, despite its `extensions.confidentialCompute` property, is deployed as a standard container because the `std-container-recipe` does not support confidential compute (or is configured to treat it as standard).
        *   The `cache` is deployed, and the connection between `backend` and `cache` is established.
    *   **Deploy to Confidential Environment:**
        *   `rad deploy ./app.bicep --environment confi-env`
        *   Radius uses the `confi-container-recipe` registered in `confi-env`.
        *   The `frontend` container is deployed as a standard container (as its definition doesn't request confidential compute).
        *   The `backend` container's `extensions.confidentialCompute` property is interpreted by the `confi-container-recipe`, and it is deployed as a confidential container (e.g., an ACI confidential container).
        *   The `cache` is deployed, and the connection between `backend` (now confidential) and `cache` is established.

This scenario highlights that the application definition remains consistent. The underlying infrastructure and specific compute capabilities (standard vs. confidential) are determined by the recipes configured in the target Radius environment, allowing for flexible deployment to diverse compute platforms without altering the core application logic or Bicep code.

#### Creating and registering custom Recipes for core types:
1.  **Create and Register Recipes for Core Types**:
    *   Create custom Bicep/Terraform recipes for `Applications.Core/containers@2025-05-01-preview`, `Applications.Core/gateways@2025-05-01-preview`, and `Applications.Core/secretStores@2025-05-01-preview` to target a specific platform or customize existing behavior.
    * Publish these recipes to an OCI registry for Bicep or a Terraform module repository.
    *   Use `rad recipe register <recipe-name> --environment <env-name> --resource-type Applications.Core/containers@2025-05-01-preview --template-path <path-to-recipe-or-oci-uri> [--parameters <key=value> ...]` to associate a recipe with the new UDT version of the container type in a specific environment.
    *   Repeat for `gateways` and `secretStores` UDTs.
    *   Example: `rad recipe register aci-container-recipe --environment my-aci-env --resource-type Applications.Core/containers@2025-05-01-preview --template-path oci://ghcr.io/radius-project/recipes/core/aci-container:1.0.0 --parameters defaultCpu=1 defaultMemoryInGB=2`
1.  **Manage and Update Recipes**:
    *   Platform engineers can update recipe definitions (e.g., to a new version from OCI or a modified local file) and re-register them using `rad recipe register` (which would effectively update the association).
    *   They can list registered recipes and their associations.
1.  **Configure Environment-Specific Recipe Parameters (Optional)**:
    *   If recipes have parameters that need to be set globally for an environment, configure them using `rad recipe update` or during registration.

#### Packaging and Registering a "Recipe Pack" for an Environment:

1.  **Define a Recipe Pack**:
    *   A platform engineer creates a manifest file (e.g., `recipe-pack.json` or `recipe-pack.yaml`) that defines a collection of recipes. This manifest would list each core resource type (e.g., `Applications.Core/containers@2025-05-01-preview`, `Applications.Core/gateways@2025-05-01-preview`, `Applications.Core/secretStores@2025-05-01-preview`) and associate it with a specific recipe (OCI URI or local path) and its default parameters.
    *   Example `recipe-pack.yaml`:
        ```yaml
        name: aci-production-pack
        version: 1.0.0
        description: "Recipe Pack for deploying to ACI in production."
        recipes:
            - resourceType: "Applications.Core/containers@2025-05-01-preview"
            name: "aci-prod-container" # Optional: a friendly name for this recipe registration
            templateKind: "bicep"
            templatePath: "oci://ghcr.io/my-org/recipes/core/aci-container:1.2.0"
            parameters:
                cpu: "1.0"
                memoryInGB: "2.0"
                environmentVariables:
                LOG_LEVEL: "Information"
            - resourceType: "Applications.Core/gateways@2025-05-01-preview"
            name: "aci-prod-gateway"
            templateKind: "bicep"
            templatePath: "oci://ghcr.io/my-org/recipes/core/aci-gateway:1.1.0"
            parameters:
                sku: "Standard_v2"
            - resourceType: "Applications.Core/secretStores@2025-05-01-preview"
            name: "aci-prod-keyvault"
            templateKind: "bicep"
            templatePath: "oci://ghcr.io/my-org/recipes/azure/keyvault-secretstore:1.0.0"
            parameters:
                skuName: "premium"
        ```
1.  **Package the Recipe Pack (Optional but Recommended)**:
    *   The manifest file and any local recipe files (if not using OCI URIs exclusively) could be bundled into an OCI artifact or a simple archive (e.g., .zip, .tar.gz) for easier distribution and versioning.
1.  **Register the Recipe Pack to an Environment**:
    *   The platform engineer uses a new CLI command to register the entire pack to a Radius environment.
    *   Example: `rad recipe-pack register --environment my-aci-env --manifest-path ./recipe-pack.yaml`
    *   Or, if packaged as an OCI artifact: `rad recipe-pack register --environment my-aci-env --oci-ref oci://ghcr.io/my-org/recipe-packs/aci-production-pack:1.0.0`
    *   This command would iterate through the recipes defined in the manifest and register each one to the specified environment, similar to individual `rad recipe register` calls.
1.  **Environment Utilizes Recipes from the Pack**:
    *   Once the Recipe Pack is registered, the environment is configured with all the specified recipes for the core UDTs.
    *   When applications are deployed to this environment, Radius automatically uses the corresponding recipes from the pack to provision `Applications.Core/containers@2025-05-01-preview`, `Applications.Core/gateways@2025-05-01-preview`, and `Applications.Core/secretStores@2025-05-01-preview` resources.
1.  **Manage and Update Recipe Packs**:
    *   Platform engineers can update the Recipe Pack manifest (e.g., point to new recipe versions, change default parameters) and re-register it. The CLI could offer options to overwrite existing registrations or manage versions of the pack within the environment.
    *   Commands like `rad recipe-pack list --environment my-env` or `rad recipe-pack show <pack-name> --environment my-env` would allow inspection of registered packs.

#### Registering default recipes for core types in a Radius Environment:

1.  **Identify the Default Recipe Pack**:
    *   The platform engineer consults Radius documentation to find the OCI URI for the default Recipe Pack corresponding to their target platform (e.g., Kubernetes, ACI). Radius will publish these official packs to a well-known OCI registry (e.g., `ghcr.io/radius-project/radius-recipe-packs/kubernetes-default:latest`).
2.  **Initialize or Select Radius Environment**:
    *   Ensure a Radius environment (e.g., `my-k8s-env`) is created and configured. This environment should be based on the new extensible environment model (e.g., `Applications.Core/environments@2025-05-01-preview`).
    *   If using `rad init --full` for a new setup, this command could potentially automatically register a default Recipe Pack (e.g., for Kubernetes) as part of its scaffolding.
3.  **Register the Default Recipe Pack**:
    *   The platform engineer uses the `rad recipe-pack register` command to apply the chosen default Recipe Pack from Radius's OCI registry to their environment.
    *   Example for Kubernetes: `rad recipe-pack register --environment my-k8s-env --oci-ref oci://ghcr.io/radius-project/radius-recipe-packs/kubernetes-default:1.0.0`
    *   Example for ACI: `rad recipe-pack register --environment my-aci-env --oci-ref oci://ghcr.io/radius-project/radius-recipe-packs/aci-default:1.0.0`
    *   This command fetches the Recipe Pack manifest and registers each recipe defined within it (for `Applications.Core/containers@2025-05-01-preview`, `Applications.Core/gateways@2025-05-01-preview`, `Applications.Core/secretStores@2025-05-01-preview`, etc.) to the specified environment with their default parameters.
4.  **Verify Recipe Registration (Optional)**:
    *   The platform engineer can verify that the recipes from the pack have been correctly registered to the environment.
    *   Example: `rad recipe list --environment my-k8s-env`
    *   This should show the default recipes for core UDTs associated with the environment.
5.  **Environment Ready**:
    *   The Radius environment is now configured with the default, Radius-provided recipes for core functionalities. Developers can deploy applications using the UDT versions of core types (e.g., `Applications.Core/containers@2025-05-01-preview`), and Radius will use these registered recipes for provisioning.

#### Contributing new Recipes and Recipe Packs for core types to the Radius repo:

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
    *   Fork the [radius-project/recipes](https://github.com/radius-project/recipes) repository.
    *   Create a new branch for your contribution.
    *   Organize your recipe files, documentation, and Recipe Pack manifest (if applicable) in a clear and logical directory structure, likely within a designated `recipes` or `contributions` area of the repository.
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

#### Migrating existing Radius applications to use the new UDT-based core types and recipes:

1.  **Understand the New Model**:
    *   Platform engineers and developers review documentation on the UDT-based core types (`Applications.Core/containers@2025-05-01-preview`, `Applications.Core/gateways@2025-05-01-preview`, `Applications.Core/secretStores@2025-05-01-preview`) and the recipe-driven approach.
2.  **Update Radius Environment**:
    *   Platform engineers update their Radius environment definition to use the new `Applications.Core/environments@2025-05-01-preview` (or later) resource type.
    *   They register recipes for the new UDT versions of core types in their respective environments. For users migrating from existing Kubernetes or ACI setups, they would register the default recipes provided by Radius for these platforms.
3.  **Update Application Bicep Files**:
    *   Developers modify their application Bicep files to reference the new versioned UDTs (e.g., change `Applications.Core/containers` to `Applications.Core/containers@2025-05-01-preview`).
    *   They adjust any application properties to align with the schema of the new UDTs and the parameters expected by the registered recipes.
4.  **Test Migrated Applications**:
    *   Deploy the updated application Bicep files to the upgraded Radius environment.
    *   Thoroughly test the application's functionality, connectivity, and configuration to ensure it behaves as expected under the new recipe-based provisioning.
5.  **Iterate and Adjust**:
    *   If issues arise, consult migration logs, recipe documentation, and UDT schemas to make necessary adjustments to application definitions or recipe parameters in the environment.
6.  **Update CI/CD Pipelines**:
    *   Modify CI/CD pipelines to use the updated application Bicep files and any new CLI commands or Radius versions required for the UDT model.

## Key investments
<!-- List the features required to enable this scenario(s). -->

### Feature 1: UDT Implementation of Core Application Model Types
<!-- One or two sentence summary -->
Re-implement existing core types (`Applications.Core/containers`, `Applications.Core/gateways`, `Applications.Core/secretStores`) as User-Defined Types (UDTs) with new, versioned resource type names (e.g., `Applications.Core/containers@2025-05-01-preview`). These UDTs will form the basis of the extensible application model. These new core types must allow for the configuration of platform-specific capabilities (e.g. confidential containers) in the resource types themselves and they must support Connections.

### Feature 2: Extensible Environment Configuration
<!-- One or two sentence summary -->
Introduce a new version of the `Applications.Core/environments` resource type (e.g., `Applications.Core/environments@2025-05-01-preview`) that removes the hard-coded `compute` and `providers` properties and relies on recipe configurations and parameters for platform-specific settings.

### Feature 3: Default Recipes for Kubernetes and ACI
<!-- One or two sentence summary -->
Develop and provide default, production-quality Bicep/Terraform recipes for the new UDT versions of `containers`, `gateways`, and `secretStores` that target Kubernetes (replicating current functionality) and Azure Container Instances (ACI). These will serve as out-of-the-box solutions and examples.

### Feature 4: Recipe Pack Creation and Management
<!-- One or two sentence summary -->
Implement a system for packaging and versioning sets of related recipes (Recipe Packs) to simplify distribution and management. This includes defining a clear structure for Recipe Packs and providing CLI commands for creating, updating, and applying them. Default Recipes for Kubernetes and ACI will be provided by Radius as part of the initial Recipe Packs.

### Feature 5: CLI Enhancements for Recipe Management (as necessary)
<!-- One or two sentence summary -->
Validate that `rad recipe` CLI capabilities for registering, managing, and validating recipes associated with core type UDTs are sufficient for compute extensibility scenarios. This includes robust validation of recipe parameters and outputs upon registration. Implement any necessary enhancements to the CLI to support this.

### Feature 6: Migration Path and Documentation
<!-- One or two sentence summary -->
Develop comprehensive documentation, examples, and potentially tooling to guide existing Radius users in migrating their applications and environments from the old hard-coded core types to the new UDT-based, recipe-driven model.

### Feature 7: Community Contribution Process
<!-- One or two sentence summary -->
Establish a clear process and guidelines for community members to contribute new recipes and Recipe Packs for core types, including documentation standards, testing requirements, and PR submission processes. This will encourage community engagement and extension of Radius capabilities.