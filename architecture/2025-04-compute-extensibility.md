# Compute Platform Extensibility

* **Author**: Brooke Hamilton (@Brooke-Hamilton)

## Overview

Radius will provide extensible support for multiple compute platforms through recipes rather than hard-coded support for each platform.

Core resource types (`containers`, `gateways`, and `secretStores`) will be implemented as user-defined types with default recipes for Kubernetes and Azure Container Instances (ACI). Only `environments` and `applications` will remain as built-in types.

This design enables:

* Platform owners to deploy to various compute platforms without Radius code changes
* Custom recipes for any compute platform
* Consistent platform engineering experience across all resource types
* Architectural separation of Radius core logic from platform provisioning code
* Community-provided extensions to support new compute platforms

A phased approach will minimize risk and enable fast delivery of value. See the development plan below for details.

0. New versions of core types that support recipes. Provide recipes to provision ACI.
1. Convert core types to user-defined types (UDTs).
2. Move Kubernetes deployments to recipes.
3. Remove existing hard-coded core types.

Two alternate paths exist:

* Alternate 1: Skip phase 0 and start with phase 1.
* Alternate 2: Implement phase 0 and stop. Do not convert core types to UDTs.

Phase 0 is the fastest to market because it builds upon existing patterns in the code. However, the first alternate path has the strength of avoiding throwaway code created in phase 0. The second alternate is good if we are committed to the concept of a core application model where the types are built into Radius. More details are below.

## Terms and definitions

| Term | Definition |
|------|------------|
| **Compute platform** | An environment where applications can be deployed and run, such as Kubernetes, Azure Container Instances (ACI), etc. |
| **Core types** | Built-in resource types provided by Radius, including `containers`, `gateways`, `secretStores`, `environments`, and `applications`. |
| **User-defined type (UDT)** | A custom resource type defined separately from Radius core types and loaded into Radius without requiring a new Radius release. |
| **Recipe** | A component that describes how to deploy and manage a resource to a specific compute platform. Recipes are implemented in Bicep or Terraform. Recipes are registered to provision a core type or UDT in an environment. |
| **Resource Provider (RP)** | A component responsible for handling create, read, update, delete, and list (CRUDL) operations for a specific resource type. |

### Goals

* Provide platform owners with the ability to deploy to specific platforms other than Radius, like ACI.
* Enable platform owners to extend Radius to deploy to any platform through custom recipes.
* Provide a recipe-based platform engineering experience that is consistent for user-defined types and core types.

### Non-goals

* Running the Radius control plane on a non-Kubernetes platform

## Principles

These are the design principles that apply to the customer experience of extending Radius to support multiple compute platforms.

Radius extensibility to support new platforms should enable creating extensions that are:

| Principle | Description |
|-----------|-------------|
| **Independently upgradeable in a runtime environment** | Recipes are independently upgradeable. |
| **Isolated and over a network protocol** | Recipes are currently implemented through Bicep and Terraform. Both run locally and communicate to the target platform over secure network protocols. |
| **Strongly typed and support versioning** | Recipes support versioning through the use of OCI-compliant registries. They are not strongly typed in the sense of having compile-time validation, but they can be validated at the time they are registered as having the correct input parameters and output properties. |

### User scenarios

The design details in this document cover the first phase in which we create new versions of the core built-in resource types which add recipe support. This is a non-breaking change, adding the ability to use recipes for `containers`, `gateways`, and `secretStores`. As part of this phase we will implement ACI deployments using recipes.

The outcome of the first phase will be that customers or product teams can create their own extensions to support their platforms without modifying Radius code. ACI will be the first recipe-based deployment

#### Configure a non-Kubernetes Radius environment

As a platform engineer I can initialize a new Radius environment that is connected to a non-Kubernetes compute platform so that Radius can authenticate and deploy to non-Kubernetes platforms.

Changes include:

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

> NOTE: The resource types shown here are new versions, e.g., `environments@2025-05-01-preview`, not changes to the existing versions of the resource types. This allows us to add recipe support to the core types without breaking existing deployments.

The `compute` platform is removed so that we do not have hard-coded support for specific platforms. The functionality enabled by the `compute.identity` property would be implemented via recipes so that we do not need hard-coded support for specific platforms. We could consider adding a recipe to the `environment` type if there are features enabled by `compute` that we could not achieve using recipes on the other core types.

#### Initialize a workspace

As a platform engineer I can use `rad init` (or the equivalent rad `workspace`, `group` and `environment` commands), plus `rad recipe register`, to set up a non-Kubernetes compute platform.

`rad init` would be unchanged. Platform engineers would have to add recipes for each core type and UDT they plan to use.

We could consider adding a flag to `rad init` that would identify a specific set of recipes. For example, `rad init aci` would install a set of default recipes for the ACI platform. However, this work is not in scope for this design.

#### Extend Radius to a new platform by creating new recipes

The `rad recipe register` CLI command would be unchanged, as it already has the ability to associate a recipe to an environment and a resource type, and it supports setting default values for specified recipe parameters for that environment.

```shell
rad recipe register <recipe name> \
  --environment <environment name> \
  --resource-type <core or UDT resource type name> \
  --parameters throughput=400
```

#### Set recipes on core types

As a platform engineer I can set recipes on the core resource types of container, gateway, and secret store so that I can configure my deployments.

Extensions are removed from the `applications` core type because the type of data being set in extensions would be set as recipe parameters.

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

Extensions and runtimes are removed from `containers` because the data in those sections would be set as parameters to a recipe.

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

The primary risk mitigation is the phased approach. Phase 1 will prove whether recipes can deploy the core types and will uncover additional architectural or design changes that are needed, without breaking existing functionality. Phase 1 includes the development of recipe-based deployments for ACI, which will prove the recipe capability with a real-world implementation.

| Risk | Description | Mitigation |
|------|-------------|------------|
| Radius graph | Updates to the Radius graph may prove difficult and time consuming | Phase 1 will provide early detection of this risk if it becomes an issue. |
| `containers` type complexity | The `containers` type has a large surface area, e.g. port forwarding  | Maintain versioned support for older types during transition, provide clear migration paths |
| `containers.connections` | Recipes will have to create connections, which may uncover complexity. |  |

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

## Development Plan: Core Type Support for Recipes

There will be a phased implementation:

| Phase | Name | Size | Activities | Customer Capabilities |
| ----- | ---- | ---- | ---------- | --------------------- |
| 0 | Support Recipes | M | - Enable recipes on new versions of core types (still hard-coded core types)<br>- Move ACI integration to recipes<br>- Support backward compatibility for existing types | - ACI is deployed via default recipes<br> - Recipes can be modified/replaced by customers |
| 1 | Convert Types | S | - Implement core types as user-defined types<br>- Load these types by default during Radius deployment<br>- Support side-by-side with existing core types | \<Same as Phase 1\> |
| 2 | Migrate Kubernetes | XL | - Convert Kubernetes deployments to recipes<br>- Provide default recipes for Kubernetes platform<br>- Complete platform-agnostic implementation<br> - Deprecate core types | Customers can customize Kubernetes deployments with recipes |
| 3 | Remove Core Types | L | - Remove core types that are hard coded into Radius<br> - Remove Kubernetes and ACI provisioning from Radius | Original core types are no longer available in Radius. |

### Advantages of Core Type Support for Recipes Plan

* **Earlier delivery**: Recipe capabilities and ACI support using recipes can be delivered more quickly since they don't depend on a complete UDT implementation.
* **We could stop at phase 1**: If phases 2 and 3 become problematic for technical reasons, we could stop at phase 1.
* **Lower initial adoption barrier**: Adding recipe support to existing core types allows users to gradually adopt the new approach without immediate migration.
* **Incremental implementation**: The phased approach allows for testing and validation at each step, reducing overall risk.
* **Backward compatibility**: Existing applications continue to work throughout most of the transition process.
* **Customer familiarity**: Users can continue using the same resource types they're familiar with while gaining recipe capabilities.
* **Reduced initial development effort**: Phase 1 has a smaller scope compared to creating full UDTs from scratch.

### Disadvantages of Core Type Support for Recipes Plan

* **Throwaway code**: The recipe support added in Phase 1 will be removed when core types are removed.
* **Extended transition period**: The complete transition takes longer across four phases compared to the alternative approach.
* **Potential confusion**: Users may be confused about which version of core types to use during the transition period.
* **Delayed architecture benefits**: The clean separation of core functionality from implementation details is achieved later in the process.
* **Inconsistent developer experience**: Until all phases are complete, developers will experience a mix of hard-coded and recipe-based resource types.
* **Multiple migrations**: Users may need to migrate their resources multiple times as the architecture evolves.

## Alternate Path: Eliminate Phase 0

Instead of adding recipe support to existing core types, we could implement core types as UDTs from the beginning. This approach would involve:

### Advantages of Core Types as UDTs from the Start Plan

* **Clean architecture from day one**: Implementing a clear separation between core functionality and platform-specific implementations from the beginning.
* **Simplified final state**: The end-state architecture is implemented earlier, avoiding transitional code.
* **Single migration path**: Users would only need to migrate their resources once, reducing disruption.
* **More focused development effort**: Engineering resources can focus on the target architecture instead of transitional states.
* **Simplified testing strategy**: Testing can focus on the final implementation rather than multiple transitional states.

### Disadvantages of Core Types as UDTs from the Start Plan

* **Higher initial complexity**: Implementing core types as UDTs requires solving more architectural problems upfront.
* **Delayed delivery**: Initial capabilities would take longer to deliver, delaying value to customers.
* **Increased initial risk**: More substantial architectural changes increase the risk of unforeseen issues.

## Alternate Development Plan: Phase 0 Only

Another alternative is to implement Phase 1 of the first development plan and stop there. This approach would keep the core types (`containers`, `gateways`, and `secretStores`) as built-in types in the Radius application model, but add recipe support to them. This would maintain the Radius built-in types as the recommended application model, while still allowing customers to create their own application models using UDTs if they choose to do so.

### Advantages of Supporting Recipes on Core Types Only

* **Stable application model**: Maintains the familiar Radius application model that customers are already using.
* **Earlier delivery**: Same early delivery advantage as the multi-phase plan.

### Disadvantages of Supporting Recipes on Core Types Only

* **Less architectural flexibility**: Core types remain hard-coded in Radius, limiting some extensibility options.
* **Inconsistent resource type model**: Core types and UDTs would have different implementation approaches. However, customers can choose to ignore the core types and implement their own.

## Open Questions

* What is the impact on the Radius graph, and how would we continue to support it? We will could implement the graph as the relationships defined in Bicep, or implement connections in a Bicep/Terraform reusable module.
* Is the Radius group concept affected by this design?
* Can everything currently deployed by Go code be deployed using recipes? We think so, but need to prove it through prototyping.

## Alternatives considered

An alternative that we considered, but did not select, was to allow the registration of custom resource providers.

* Customers can register their own RPs for the core types (application, container, gateway, secret store). RPs may also be registered for UDTs.
* The registered RPs belong to a customer-defined environment "kind" that is declared on the Environment core type.
* Customer RPs implement a versioned OpenAPI specification generated from Radius that defines a set of REST endpoints. RPs can be written in any language, hosted on any platform, and must be managed and deployed by the customer.
* Radius routes CRUDL operations to the custom RPs.

We did not select this option because:

* The chosen option is simpler for users because having recipes on the core types makes the overall Radius user experience more consistent across all resource types.
* Recipes are simpler and lower effort for users to implement.
* Having recipes on the core types will enable most customization scenarios.
* Implementing recipes does not exclude adding custom RPs later as an additional extensibility point.

## Design Review Notes
