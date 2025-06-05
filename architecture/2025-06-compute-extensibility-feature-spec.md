# Topic: Compute Platform Extensibility for Radius

* **Author**: Will Tsai (@willtsai)

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

#### Contributing new Recipes for core types to the Radius repo:

#### Registering multiple Recipes for core types in a single Environment:

#### Migrating existing Radius applications to use the new UDT-based core types and recipes:

## Key investments
<!-- List the features required to enable this scenario(s). -->

### Feature 1: UDT Implementation of Core Application Model Types
<!-- One or two sentence summary -->
Re-implement existing core types (`Applications.Core/containers`, `Applications.Core/gateways`, `Applications.Core/secretStores`) as User-Defined Types (UDTs) with new, versioned resource type names (e.g., `Applications.Core/containers@2025-05-01-preview`). These UDTs will form the basis of the extensible application model.

### Feature 2: Extensible Environment Configuration
<!-- One or two sentence summary -->
Introduce a new version of the `Applications.Core/environments` resource type (e.g., `Applications.Core/environments@2025-05-01-preview`) that removes the hard-coded `compute` property and relies on recipe configurations and parameters for platform-specific settings.

### Feature 3: Default Recipes for Kubernetes and ACI
<!-- One or two sentence summary -->
Develop and provide default, production-quality Bicep/Terraform recipes for the new UDT versions of `containers`, `gateways`, and `secretStores` that target Kubernetes (replicating current functionality) and Azure Container Instances (ACI). These will serve as out-of-the-box solutions and examples.

### Feature 4: Enhanced Recipe Management and Validation
<!-- One or two sentence summary -->
Improve `rad recipe` CLI capabilities for registering, managing, and validating recipes associated with UDTs. This includes robust validation of recipe parameters and outputs upon registration.

### Feature 5: Migration Path and Documentation
<!-- One or two sentence summary -->
Develop comprehensive documentation, examples, and potentially tooling to guide existing Radius users in migrating their applications and environments from the old hard-coded core types to the new UDT-based, recipe-driven model.