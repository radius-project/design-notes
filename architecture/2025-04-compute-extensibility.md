# Compute Platform Extensibility

* **Author**: Brooke Hamilton (@Brooke-Hamilton)

## Overview

Radius will provide extensible support for multiple compute platforms through recipes rather than hard-coded support for each platform.

Core resource types (`containers`, `gateways`, and `secretStores`) will be implemented as user-defined types with default recipes for Kubernetes and Azure Container Instances (ACI). Only `environments` and `applications` will remain as built-in types.

This design enables:

* Architectural separation of Radius core logic from platform provisioning code
* Community-provided extensions to support new compute platforms without Radius code changes
* Consistent platform engineering experience across all resource types

We have two options for implementation:

1. (Recommended) Create UDT versions of the core types paired with recipe-based provisioning for ACI and Kubernetes. Later, remove the core types and existing Kubernetes provisioning code.
2. (More conservative) Phased plan:
    * Add recipe support to existing core types and create recipe-based provisioning for ACI. (Kubernetes provisioning is unchanged.)
    * Create UDT versions of the core types, add recipe-based provisioning for Kubernetes, and update the ACI recipes to use the UDT core types.
    * Remove the core types and existing Kubernetes provisioning code.

Option 1 is a more direct path to the end state, builds on the emerging capabilities of UDTs, and does not require modifying existing resource types. The disadvantage is a higher initial risk and cost before realizing value because we would be simultaneously creating new recipes for ACI and Kubernetes, and we would be building upon UDT, which is still under construction.

Option 2 is less risky and has faster initial time to value because the first usable release would be a recipe-based provisioning for ACI that works side-by-side with the existing Kubernetes provisioning logic. It has the advantages of not depending on the emerging UDT feature, and requiring less overall work at first. However, it requires some throwaway work to add recipe support to existing core types.

An alternate path for Option 2 is to stop work after adding recipe support to the built-in core types and creating provisioning recipes for ACI and Kubernetes. The core types would not become UDTs. This path would achieve the architectural separation of recipe provisioning while preserving the existing application model as currently defined in the hard-coded core types.

## Terms and definitions

| Term | Definition |
|------|------------|
| **Compute platform** | An environment where applications can be deployed and run, such as Kubernetes, Azure Container Instances (ACI), etc. |
| **Core types** | Built-in resource types provided by Radius, including `containers`, `gateways`, `secretStores`, `environments`, and `applications`. |
| **User-defined type (UDT)** | A custom resource type defined separately from Radius core types and loaded into Radius without requiring a new Radius release. |
| **Recipe** | A set of instructions that provisions a Radius resource to a Radius environment. Recipes are implemented in Bicep or Terraform. |
| **Resource Provider (RP)** | A component responsible for handling create, read, update, delete, and list (CRUDL) operations for a specific resource type. |

### Goals

* Provide platform engineers with the ability to deploy to specific platforms other than Radius, like ACI.
* Provide a recipe-based platform engineering experience that is consistent for user-defined types and core types.

### Non-goals

* Running the Radius control plane on a non-Kubernetes platform

## Principles

Radius extensibility should enable creating extensions that are:

| Principle | Description |
|-----------|-------------|
| **Independently upgradeable in a runtime environment** | Recipes are independently upgradeable. |
| **Isolated and over a network protocol** | Recipes are currently implemented through Bicep and Terraform. Both run locally and communicate to the target platform over secure network protocols. |
| **Strongly typed and support versioning** | Recipes support versioning through the use of OCI-compliant registries. They are not strongly typed in the sense of having compile-time validation, but they can be validated upon registration in an environment as having the correct input parameters and output properties. |

### User scenarios

#### Configure a non-Kubernetes Radius environment

As a platform engineer I can initialize a new Radius environment that is connected to a non-Kubernetes compute platform so that Radius can authenticate and deploy to non-Kubernetes platforms.

A new version of `environments` will be added that does not have a hard coded list of compute kinds.

```diff
+resource environment 'Applications.Core/environments@2025-05-01-preview' = {
  name: 'myenv'
  properties: {
-    compute: {
-      kind: 'kubernetes'
-      namespace: 'default'
-      }
-    }
    recipeConfig: {
      env {
        foo: 'bar'
      }
    }
    recipes: {
      'Applications.Datastores/redisCaches':{
        default: {
          templateKind: 'bicep'
          plainHttp: true
          templatePath: 'ghcr.io/radius-project/recipes/azure/rediscaches:latest'
        }
      }
    }
-   extensions: [
-      {
-        kind: 'kubernetesMetadata'
-        labels: {
-          'team.contact.name': 'frontend'
-        }
-      }
-    ]
  }
}
```

> NOTE: The resource types shown here are new versions, e.g., `environments@2025-05-01-preview`, not changes to the existing versions of the resource types. This allows us to avoid breaking existing deployments.

The `compute` platform is removed so that we do not have hard-coded support for specific platforms. The functionality enabled by the `compute.identity` property would be implemented via recipes so that we do not need hard-coded support for specific platforms. We could consider adding a recipe to the `environment` type if there are features enabled by `compute` that we could not achieve using recipes on the other core types.

#### Initialize a workspace

As a platform engineer I can use `rad init` (or the equivalent rad `workspace`, `group` and `environment` commands), plus `rad recipe register`, to set up a non-Kubernetes compute platform.

`rad init` would be unchanged (in terms of the user experience). Platform engineers would have to add recipes for each core type and UDT they plan to use. By default, the command will register resource types and recipes for Kubernetes provisioning.

We could consider adding a flag to `rad init` that would identify a specific set of recipes. For example, `rad init aci` would install a set of default recipes for the ACI platform. However, this work is not in scope for this design.

#### Extend Radius to a new platform by creating new recipes

The `rad recipe register` CLI command would be unchanged, as it already has the ability to associate a recipe to an environment and a resource type, and it supports setting default values for specified recipe parameters for that environment.

```shell
rad recipe register <recipe name> \
  --environment <environment name> \
  --resource-type <core or UDT resource type name> \
  --parameters throughput=400
```

#### Register recipes for types

As a platform engineer I can set recipes on the resource types of container, gateway, and secret store so that I can configure my deployments.

Extensions are removed from the `applications` core type because the type of data being set in extensions would be set as recipe parameters. This is a new version of `applications`.

```diff
+resource app 'Applications.Core/applications@2025-05-01-preview' = {
  name: 'myapp'
  properties: {
    environment: environment
-   extensions: [
-     {
-       kind: 'kubernetesNamespace'
-       namespace: 'myapp'
-     }
-     {
-       kind: 'kubernetesMetadata'
-       labels: {
-         'team.contact.name': 'frontend'
-       }
-     }
-   ]
  }
}
```

#### Container resource type

Extensions and runtimes are removed from `containers` because the data in those sections would be set as parameters to a recipe.

This is a new version of `containers`. `Applications.Core` remains as the namespace because the purpose of the `containers` type remains the same whether the type is hard-coded or created as a UDT.

```diff
+resource frontend 'Applications.Core/containers@2025-05-01-preview' = {
  name: 'frontend'
  properties: {
    application: app.id
    container: {
      image: 'registry/container:tag'
      env:{
        DEPLOYMENT_ENV: {
          value: 'prod'
        }
        DB_CONNECTION: {
          value: db.listSecrets().connectionString
        }
      }
      ports: {
        http: {
          containerPort: 80
          protocol: 'TCP'
        }
      }
    }
-   extensions: [
-     {
-       kind: 'daprSidecar'
-       appId: 'frontend'
-     }
-     {
-       kind:  'manualScaling'
-       replicas: 5
-     }
-     {
-       kind: 'kubernetesMetadata'
-       labels: {
-         'team.contact.name': 'frontend'
-       }
-     }
-   ]
-   runtimes: {
-     kubernetes: {
-       base: loadTextContent('base-container.yaml')
-       pod: {
-         containers: [
-           {
-             name: 'log-collector'
-             image: 'ghcr.io/radius-project/fluent-bit:2.1.8'
-           }
-         ]
-         hostNetwork: true
-       }
-     }
-   }
+   // Recipe would only be added if the selected dev plan includes the step of enabling recipes on core types. The recipe property is not needed if the core types are created as UDTs.
+   recipe: {
+      // Name a specific Recipe to use
+      name: 'azure-aci-container'
+      // Set parameters that will be passed to the recipe.
+      parameters: {
+        subscriptionId: 'xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx'
+        resourceGroup: 'myrg'
+      }
+    }
  }
}
```

#### Gateways resource type

```diff
+resource gateway 'Applications.Core/gateways@2025-05-01-preview' = {
  name: 'gateway'
  properties: {
    application: app.id
    hostname: {
      // Omitting hostname properties results in gatewayname.appname.PUBLIC_HOSTNAME_OR_IP.nip.io

      // Results in prefix.appname.PUBLIC_HOSTNAME_OR_IP.nip.io
      prefix: 'prefix'
      // Alternately you can specify your own hostname that you've configured externally
      fullyQualifiedHostname: 'hostname.radapp.io'
    }
    routes: [
      {
        path: '/frontend'
        destination: 'http://${frontend.name}:3000'
      }
      {
        path: '/backend'
        destination: 'http://${backend.name}:8080'

        // Enable websocket support for the route (default: false)
        enableWebsockets: true
      }
    ]
    tls: {
      // Specify SSL Passthrough for your app (default: false)
      sslPassthrough: false

      // The Radius Secret Store holding TLS certificate data
      certificateFrom: secretstore.id
      // The minimum TLS protocol version to support. Defaults to 1.2
      minimumProtocolVersion: '1.2'
    }
+   // Recipe would only be added if the selected dev plan includes the step of enabling recipes on core types. The recipe property is not needed if the core types are created as UDTs.
+   recipe: {
+      // Name a specific Recipe to use
+      name: 'azure-aci-gateway'
+      // Set parameters that will be passed to the recipe.
+      parameters: {
+        subscriptionId: 'xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx'
+        resourceGroup: 'myrg'
+      }
+    }
  }
}
```

#### secretStores resource type

```diff
+resource appCert 'Applications.Core/secretStores@2025-05-01-preview' = {
  name: 'appcert'
  properties:{
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
+   // Recipe would only be added if the selected dev plan includes the step of enabling recipes on core types. The recipe property is not needed if the core types are created as UDTs.
+   recipe: {
+      // Name a specific Recipe to use
+      name: 'azure-aci-gateway'
+      // Set parameters that will be passed to the recipe.
+      parameters: {
+        subscriptionId: 'xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx'
+        resourceGroup: 'myrg'
+      }
+    }
  }
}
```

<!--
## Design

### High Level Design
High level overview of the data flow and key components.

Provide a high-level description, using diagrams as appropriate, and top-level
explanations to convey the architectural/design overview. Don’t go into a lot
of details yet but provide enough information about the relationship between
these components and other components. Call out or highlight new components
that are not part of this feature (dependencies). This diagram generally
treats the components as black boxes. Provide a pointer to a more detailed
design document, if one exists. 
-->

<!--
### Architecture Diagram
Provide a diagram of the system architecture, illustrating how different
components interact with each other in the context of this proposal.

Include separate high level architecture diagram and component specific diagrams, wherever appropriate.
-->

<!--
### Detailed Design

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

#### Risks and Mitigations

The primary risk mitigation is to begin with provisioning ACI using UDTs and recipes, as that activity will not break existing logic and will expose any unknown issues like additional architectural or design changes that are needed.

| Risk | Description | Mitigation |
|------|-------------|------------|
| Radius graph | Updates to the Radius graph may prove difficult and time consuming | Phase 1 will provide early detection of this risk if it becomes an issue. |
| `containers.connections` | Recipes will have to create connections, which may uncover complexity. | This risk is related to the graph risk, and we will use Phase 1 to provide early detection. |
| `containers` type complexity | The `containers` type has a large surface area.  | Maintain versioned support for older types during transition, provide clear migration paths. |

add: risk that bicep cannot do things we do in imperative code - kubernetes bicep extension, can we add a bicep provider?
Connections = DNS, so if DNS succeeds the connection exists

<!--
Describe what's not ideal about this plan. Does it lock us into a 
particular design for future changes or is it flexible if we were to 
pivot in the future. This is a good place to cover risks.
-->

<!--
#### Proposed Option
Describe the recommended option and provide reasoning behind it.
-->

<!--
### API design (if applicable)

Include if applicable – any design that changes our public REST API, CLI
arguments/commands, or Go APIs for shared components should provide this
section. Write N/A here if not applicable.
- Describe the REST APIs in detail for new resource types or updates to
  existing resource types. E.g. API Path and Sample request and response.
- Describe new commands in the CLI or changes to existing CLI commands.
- Describe the new or modified Go APIs for any shared components.
-->

<!--
### CLI Design (if applicable)
Include if applicable – any design that changes Radius CLI
arguments/commands. Write N/A here if not applicable.
- Describe new commands in the CLI or changes to existing CLI commands.
-->

<!--
### Implementation Details
High level description of updates to each component. Provide information on 
the specific sub-components that will be updated, for example, controller, processor, renderer,
recipe engine, driver, to name a few.

#### UCP (if applicable)
#### Bicep (if applicable)
#### Deployment Engine (if applicable)
#### Core RP (if applicable)
#### Portable Resources / Recipes RP (if applicable)
-->

<!--
### Error Handling
Describe the error scenarios that may occur and the corresponding recovery/error handling and user experience.
-->

<!--
## Test plan

Include the test plan to validate the features including the areas that
need functional tests.

Describe any functionality that will create new testing challenges:
- New dependencies
- External assets that tests need to access
- Features that do I/O or change OS state and are thus hard to unit test
-->

<!--
## Security

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

<!--
## Compatibility (optional)

Describe potential compatibility issues with other components, such as
incompatibility with older CLIs, and include any breaking changes to
behaviors or APIs.
-->

<!--
## Monitoring and Logging

Include the list of instrumentation such as metric, log, and trace to 
diagnose this new feature. It also describes how to troubleshoot this feature
with the instrumentation. 
-->

## Recommended Development Plan: Move core types to UDTs

The recommended option is to implement core types as UDTs, and later remove the hard-coded core types.

| Phase | Name | Size | Activities | Customer Capabilities |
| ----- | ---- | ---- | ---------- | --------------------- |
| 1 | Create UDTs for core types, provision ACI with recipes | M | - Implement core types as UDTs for `containers`, `gateways`, and `secretStores`.<br>- Implement ACI provisioning from recipes<br>| - ACI is provisioned from default recipes<br> - ACI recipes can be modified/replaced by customers |
| 2 | Provision Kubernetes with recipes| L | Convert Kubernetes deployments to recipes for the UDT core types<br> | Kubernetes recipes can be modified/replaced by customers |
| 3 | Remove Existing Core Types | S | - Remove core types that are hard coded into Radius<br> - Remove Kubernetes and ACI provisioning from Radius | Original core types are no longer available in Radius. |

### Advantages vs Alternate Plans

* **Single migration path**: Users would only need to migrate their resources once, reducing disruption.
* **Simplest path to the target architecture**: This plan immediately implements a clear architectural separation between core functionality (environments and applications) and platform-specific provisioning (UDTs and recipes).
* **Simplified transitional**: Transitional phases and code is avoided.
* **More focused development effort**: Engineering resources can focus on the target architecture instead of transitional states.
* **Simplified testing strategy**: Testing can focus on the final implementation rather than multiple transitional states.

### Disadvantages vs Alternate Plans

* **Higher initial complexity**: Implementing core types as UDTs requires solving more problems simultaneously, including 
* **Delayed delivery**: Initial capabilities will take longer to deliver.
* **Increased initial risk**: More substantial architectural changes increase the risk of unforeseen issues.

## Alternate Development Plan Option 1: Core Type Support for Recipes

This is an alternate to the recommended development plan in which we add recipe support to new versions of the core resource types, which has the advantage of being a more phased approach, but the disadvantage of creating some throwaway work in phase 0.

Phase 0 shown below is added to the development plan above.

| Phase | Name | Size | Activities | Customer Capabilities |
| ----- | ---- | ---- | ---------- | --------------------- |
| 0 | Support Recipes on Core Types | M | - Enable recipes on new versions of core types (still hard-coded core types)<br>- Move ACI integration to recipes<br>- Support backward compatibility for existing types | - ACI is deployed via default recipes<br> - Recipes can be modified/replaced by customers |

### Advantages vs Recommended Plan

* **Earlier delivery**: Recipe capabilities and ACI support using recipes can be delivered more quickly since they don't depend on a complete UDT implementation.
* **We could stop at phase 1**: If phases 2 and 3 become problematic for technical reasons, we could stop at phase 1.
* **Lower initial adoption barrier**: Adding recipe support to existing core types allows users to gradually adopt the new approach without immediate migration.
* **Incremental implementation**: The phased approach allows for testing and validation at each step, reducing overall risk.
* **Backward compatibility**: Existing applications continue to work throughout most of the transition process.
* **Customer familiarity**: Users can continue using the same resource types they're familiar with while gaining recipe capabilities.
* **Reduced initial development effort**: Phase 1 has a smaller scope compared to creating full UDTs from scratch.

### Disadvantages vs Recommended Plan

* **Throwaway code**: The recipe support added in Phase 1 will be removed when core types are removed.
* **Extended transition period**: The complete transition takes longer across four phases compared to the alternative approach.
* **Potential confusion**: Users may be confused about which version of core types to use during the transition period.
* **Delayed architecture benefits**: The clean separation of core functionality from implementation details is achieved later in the process.
* **Inconsistent developer experience**: Until all phases are complete, developers will experience a mix of hard-coded and recipe-based resource types.
* **Multiple migrations**: Users may need to migrate their resources multiple times as the architecture evolves.

## Alternate Development Plan Option 2: Phase 0 Only

Another alternative is to implement Phase 0 of the alternate development plan and stop there. This approach would keep the core types (`containers`, `gateways`, and `secretStores`) as built-in types in the Radius application model, but add recipe support to them. 

The advantage would be keeping the Radius built-in types as the recommended application model, while still allowing customers to create their own application models using UDTs if they choose to do so. We would choose this plan if we want the Radius application model to be expressed through built-in resource types instead of UDTs.

### Advantages vs Other Development Plans

* **Stable application model**: Maintains the familiar Radius application model that customers are already using.
* **Incremental change**: Adding recipe support to core types is an additive change, and adding ACI provisioning with recipes is not disruptive to the existing provisioning logic.
* **Earlier delivery**: Same early delivery advantage as the alternate multi-phase plan.

### Disadvantages vs Other Development Plans

* **Less architectural flexibility**: Core types remain hard-coded in Radius, limiting some extensibility options, e.g., the ability of a customer to copy and modify a core type.
* **Inconsistent resource type model**: Core types and UDTs would have different implementation approaches. However, customers can choose to ignore the core types and implement their own.

## Open Questions

* What is the impact on the Radius graph, and how would we continue to support it? We will could implement the graph as the relationships defined in Bicep, or implement connections in a Bicep/Terraform reusable module.
* Is the Radius group concept affected by this design?
* Can everything currently deployed by Go code be deployed using recipes? We think so, but need to prove it through prototyping.

## Architecture Alternatives Considered for Extensibility

An alternative architecture for extensibility that we considered, but did not select, was to enable the registration of custom resource providers. This architecture could be added to Radius later; it does not conflict with or replace with recipe-based provisioning.

* Customers can register their own RPs for UDTs.
* Customer RPs implement an OpenAPI specification generated from UDT type definitions that defines a set of REST endpoints.
* Radius routes CRUDL operations to the custom RPs.
* RPs can be written in any language, hosted on any platform, and must be managed and deployed by the customer.
* Operations in addition to CRUDL could be supported.

We did not select this option because:

* The chosen option is simpler for users because extending Radius through recipes makes the overall Radius user experience more consistent across all resource types.
* Recipes are simpler and lower effort for users to implement.
* Having recipes on the core types will enable most customization scenarios.
* Implementing recipes does not exclude adding custom RPs later as an additional extensibility point.

<!-- 
## Design Review Notes
-->
