# Recipe Packs

* **Author**: `Nithya (@nithyatsu)`

## Overview

Recipes are external infrastructure-as-code (IaC) templates that operators register on a Radius Environment so developers can use them later for provisioning. They provide a mechanism for separation of concerns between developers and operators. 

Today, Radius supports registering recipes individually, either via the `Applications.Core/environments` resource properties or the CLI (`rad recipe register`). For each Radius Environment, platform engineers have to piece together individual Recipes from scratch. Some customers could have 1000s of environments and putting everything together manually for each environment is error prone. Recipes also do not have a lifecycle of their own and can be managed only by managing the environments pointing to them.

This document proposes the design of a Recipe Pack as a first class resource type in Radius. The Recipe Pack enables bundling multiple recipe selections for different resource types into a reusable unit that can be referenced by environments. 

## Terms and definitions

| Term        | Definition                                                                                         |
| ----------- | -------------------------------------------------------------------------------------------------- |
| Recipe      | IaC templates that operators register on a Radius Environment                                     |
| Recipe Pack | A collection of recipes that can be managed as an entity                                          |
| RRT         | [Radius Resource Type](https://docs.radapp.io/guides/author-apps/custom/overview/)               |

## Objectives

> **Issue Reference:** 

### Goals

Provide Recipe Packs as a Radius feature to bundle multiple recipes as a single manageable entity into a Radius Environment (e.g., a pack for ACI that includes all necessary recipes).

  - Radius should provide APIs to manage Radius.Core/recipePacks resource through CRUDL operations
  

### Non goals

Recipe Packs would bundle together "recipes" as we understand them today. We do not cover recipe versioning / other recipe specific enhancements in this design. 

### User scenarios (optional)

#### Registering several recipes to an environment

As an operator I am responsible for creating Radius Environments using which developers can deploy their applications. As part of creating the environment, I manually link recipes one by one using `rad recipe register` or by updating the environment definition. This can be error prone when there are many recipes and environments. Radius should provide a way to bulk register (and manage) recipes.

#### Registering recipes to multiple environments

As an operator I am responsible for creating Radius Environments using which developers can deploy their applications. As part of creating the environment, I manually link recipes one by one using `rad recipe register` or by updating the environment definition. I have 100s of environments which mostly use the same recipes. Piecing the same recipes together for each environment feels like rework.

#### Sharing Recipe Packs Across Environments and Organizations

As an operator, I want to share and reuse Recipe Packs across different environments or organizations. Instead of manually registering individual recipes, I can import a pre-bundled Recipe Pack (e.g., for Kubernetes or ACI) published by a provider or another team. This streamlines environment setup and reduces errors.

## Design

### Design Overview

In general, Radius.Core namespace has resources whose schema should be non-editable so that Radius can work as expected, for example Applications, Environments and recipePacks. These resources must be provisioned imperatively  and their schema must be protected. With this constraint in mind, 2 approaches are possible: 
1. Radius.Core/recipePacks provisioned imperatively by Applications RP
2. Radius.Core/recipePacks provisioned Manually by Dynamic RP

Both approaches have the below benefits:

- Solves the requirement for bulk registering recipes using single command with a one time effort of creating the recipe pack resource
- As first class resource, recipe packs would be displayed in app graphs. They can also have their own lifecycle and RBAC independent of environments. 
- Helps reduce the size of environment resource, which could reach serialization limits with tons of recipes. 
- Helps reduce overall size of Radius datastore, since common recipe information could now be stored as a single resource instead of being duplicated across several environments.
- Helps with change isolation, since recipe pack updates are isolated from env updates.
- Promotes reusablity since multiple environments can point to a recipe pack using the recipe pack ID. 

Below table highlights the trade offs:

| Aspect | Applications RP provisioning| Dynamic RP provisioning |
|---------|-----------| ------------------------------------------|
| **Tooling complexity** | ❌ Higher (TSP, converter, schema, API implemenetations needed) | ✅ Lower (YAML only, dynamic resource controllers reused)|
| **Versioning**  | ❌ requires versioning support in Radius  |✅ Supports schema versioning| 
| **Custom implementation for operations**  | ✅ can customize details of CRUDL|  ❌ falls back on dynamic resource controllers  |

Based on the above differences, *we choose Radius.Core/recipePacks to be provisioned imperatively by Applications RP as a first class Radius resource*. The main reason for this decision is that Radius Core resources such as enviroments and recipe-packs could have complex and custom deletion logic compared to what a dynamic resource deletion does. For instance, we need a cascade of deletion when an environment is deleted, or we might want to restrict deleting a recipe pack that is referenced in one or more environment. It is important to support robust core management operations of these resources where as the versioning of the type can follow once Radius versioning support is available. 

We should make sure the rad resource-type commands cannot alter the schema of these types as part of schema validation (this namespace is reserved for Radius's use). Appropriate error message should be provided to the user. 



### Other alternatives considered 

***Embed all recipe mappings inline in the Environment***

This is similar to what we have today. We could introduce a yaml spec similar to below, and when user executes a `rad recipe-pack register` this spec could be parsed and all recipes added to environment. 

```yaml
name: aci-production-pack
version: 1.0.0
description: "Recipe Pack for deploying to ACI in production."
recipes:
  - resourceType: "Radius.Compute/containers@2025-05-01-preview"   
    recipeKind: "bicep"
    recipeLocation: "oci://ghcr.io/my-org/recipes/core/aci-container:1.2.0"
    parameters:
      cpu: "1.0"
      memoryInGB: "2.0"
      environmentVariables:
        LOG_LEVEL: "Information"
      # Optional: allow platform-specific options like containerGroupProfile for ACI
      allowPlatformOptions: true
  - resourceType: "Radius.Compute/gateways@2025-05-01-preview"
    recipeKind: "bicep"
    recipeLocation: "oci://ghcr.io/my-org/recipes/core/aci-gateway:1.1.0"
    parameters:
      sku: "Standard_v2"
  - resourceType: "Radius.Security/secrets@2025-05-01-preview"
    recipeKind: "bicep"
    recipeLocation: "oci://ghcr.io/my-org/recipes/azure/keyvault-secretstore:1.0.0"
    parameters:
      skuName: "premium"
```

Pros: 

- Most compatible to what we have in Radius today and hence the fastest approach. 
- Solves the requirement for bulk registering recipes using single command with a one time effort of creating the yaml manifest

Cons: 

- Environment still stays a bloated object. Environment resource houses a lot of other properties and we could potentially risk hitting the mechanical limits that apply to serializing objects. 
- Add Radius commands to publish recipe-packs, similar to what we have for recipes today. 
- A list of recipes could potentially be managed as a collection, including having its own rbac and appearance in app graph. The above approach does not allow for that possibility.

***Store a URL to a YAML manifest in the Environment***

We could fetch the yaml when needed, and use the available recipe.

`rad environment update my-env --recipe-packs aci-production-pack='git::https://github.com/my-org/recipe-packs.git//aci-production-pack.yaml?ref=1.0.0'`

Pros:

- Helps manage the size of environment resource
- Solves the requirement for bulk registering recipes using single command with a one time effort of creating the yaml manifest

Cons:

- For each provisioning of resource, we make a call to registry to fetch the list to check if the recipe is available, and then a call to the specified recipe location to fetch it. We could cache the list and construct an in-memory recipe pack ephemeral object. But we still do not get the benefits of recipe pack as a first-class resource type. 

### High level flow


At a very high level, this design approach needs the below steps:

* Add support for Radius.Core/recipePacks resource in Applications RP 
  * Schema + API design and implementation
* Support `rad cli` commands that enable CRUDL operations on resources of type Radius.Core/recipePacks 
* Design and support Radius.Core/environment schema in Applications RP
* Support `rad cli` commands that enable managing Radius.Core/environment resources through CRUDL operations on this type of resource
* Support `rad cli` commands that enable registering recipe-packs to a `Radius.Core/environments` environment resource
* Enhance Dynamic RP, Applications RP and UCP to support the feature.

### Schema and API design

As part of supporting Recipe Pack as a resource type, at a high-level, we define a recipePacks.tsp
   
```tsp
namespace Radius.Core;

@doc("The recipe pack resource")
model RecipePackResource 
  is TrackedResourceRequired<RecipePackProperties, "recipePacks"> {
  @doc("recipe pack name")
  @key("recipePackName")
  @path
  @segment("recipePacks")
  name: ResourceNameString;
}

@doc("Recipe Pack properties")
model RecipePackProperties {
  @doc("The status of the asynchronous operation.")
  @visibility("read")
  provisioningState?: ProvisioningState;

  @doc("Description of what this recipe pack provides")
  description?: string;

  @doc("Map of resource types to their recipe configurations")
  recipes: Record<RecipeDefinition>;
}

@doc("Recipe definition for a specific resource type")
model RecipeDefinition {
  @doc("The type of recipe (e.g., terraform, bicep)")
  recipeKind: RecipeKind;

  @doc("URL or path to the recipe source")
  recipeLocation: string;

  @doc("Parameters to pass to the recipe")
  parameters?: {};
}

@doc("The type of recipe")
enum RecipeKind {
  @doc("Terraform recipe")
  terraform: "terraform",

  @doc("Bicep recipe")
  bicep: "bicep",
}

@armResourceOperations
interface RecipePacks {
  get is ArmResourceRead<
    RecipePackResource,
    UCPBaseParameters<RecipePackResource>
  >;

  createOrUpdate is ArmResourceCreateOrReplaceSync<
    RecipePackResource,
    UCPBaseParameters<RecipePackResource>
  >;

  update is ArmResourcePatchSync<
    RecipePackResource,
    RecipePackProperties,
    UCPBaseParameters<RecipePackResource>
  >;

  delete is ArmResourceDeleteSync<
    RecipePackResource,
    UCPBaseParameters<RecipePackResource>
  >;

  listByScope is ArmResourceListByParent<
    RecipePackResource,
    UCPBaseParameters<RecipePackResource>,
    "Scope",
    "Scope"
  >;
}
```
* As an enhancement that can fast follow, we would add a version attribute so that users can version their recipe packs. We could chose name+version as the internal name which would be used to construct recipe pack's resource id. 

* We choose a map of resource types to their recipe configurations so that the relevant recipe for a type can be easily accessed. 
  
* We will not be supporting named recipes going forward as documented in [RRT feature spec](https://github.com/willtsai/design-notes-radius/blob/f9c98baf515263c27e7637131d7a48ae5a01b2c0/features/2025-02-user-defined-resource-type-feature-spec.md#user-story-7--registering-recipes). Therefore the `RecipeDefinition` model does not include a name.
  
* we are not supporting "scheme" (http|https|...). We can use the information in recipe location to determine that. We might have to introduce it back if we support other kinds of location for recipes in future.
  
* The operations are all Synchronous, since Recipe Pack is a lightweight configuration resource unless we choose to validate the recipe location.
In this case we might want to store the md5 of the recipe too, so that the content could be fetched and the md5 matched.

  
#### Examples

Below is a sample bicep definition of a recipe pack resource:

```bicep
resource computeRecipePack 'Radius.Core/recipePacks@2026-01-01-preview' = {
  name: 'computeRecipePack'
  description: "Recipe Pack for deploying to Kubernetes."
  properties: {
    recipes: { 
      'Radius.Compute/containers': {
        recipeKind: 'terraform'
        recipeLocation: 'https://github.com/project-radius/resource-types-contrib.git//recipes/compute/containers/kubernetes?ref=v0.48'
        parameters: {
          allowPlatformOptions: true
          anIntegerParam: 1
        }
      }
      'Radius.Security/secrets': {
        recipeKind: 'terraform'
        recipeLocation: 'https://github.com/project-radius/resource-types-contrib.git//recipes/security/secrets/kubernetes?ref=v0.48'
      }
      'Radius.Storage/volumes': {
        recipeKind: 'terraform'
        recipeLocation: 'https://github.com/project-radius/resource-types-contrib.git//recipes/storage/volumes/kubernetes?ref=v0.48'
      }
    }
  }
}
```

```
resource env 'Radius.Core/environments@2025-05-01-preview' = { 
    name: 'my-env' 
    properties: { 
+       // The recipePacks property is an array of Recipe Pack IDs 
+       recipePacks: [computeRecipePack.id, dataRecipePack.id]
        // Other properties
    } 
} 
```


Below are sample HTTP requests for managing a recipe pack resource

CREATE request:

```
curl -X PUT \
    "http://localhost:9000/apis/api.ucp.dev/v1alpha3/planes/radius/local/resourceGroups/default/providers/Radius.Core/recipePacks/testrecipepack?api-version=2023-10-01-preview" \
    -H "Content-Type: application/json" \
    -d '{
      "location": "global",
      "properties": {
        "description": "Test recipe pack with sample recipes",
        "recipes": {
          "Applications.Datastores/sqlDatabases": {
            "recipeKind": "terraform",
            "recipeLocation": "https://github.com/example/recipes/sql-database",
            "parameters": {
              "size": "small",
              "backup": false
            }
          },
          "Applications.Datastores/redisCaches": {
            "recipeKind": "bicep",
            "recipeLocation": "https://github.com/example/recipes/redis-cache.bicep",
            "parameters": {
              "tier": "basic"
            }
          }
        }
      }
    }'
 ```   

CREATE response:

```json
{
  "id": "/planes/radius/local/resourcegroups/default/providers/Radius.Core/recipePacks/testrecipepack",
  "location": "global",
  "name": "testrecipepack",
  "properties": {
    "description": "Test recipe pack with sample recipes",
    "provisioningState": "Succeeded",
    "recipes": {
      "Applications.Datastores/redisCaches": {
        "parameters": {
          "tier": "basic"
        },
        "recipeKind": "bicep",
        "recipeLocation": "https://github.com/example/recipes/redis-cache.bicep"
      },
      "Applications.Datastores/sqlDatabases": {
        "parameters": {
          "backup": false,
          "size": "small"
        },
        "recipeKind": "terraform",
        "recipeLocation": "https://github.com/example/recipes/sql-database"
      }
    }
  },
  "systemData": {
    "createdAt": "0001-01-01T00:00:00Z",
    "createdBy": "",
    "createdByType": "",
    "lastModifiedAt": "0001-01-01T00:00:00Z",
    "lastModifiedBy": "",
    "lastModifiedByType": ""
  },
  "tags": {},
  "type": "Applications.Core/recipePacks"
}

```

READ request:

```
nithya@MacBook-Pro ~ %  curl -X GET\
    "http://localhost:9000/apis/api.ucp.dev/v1alpha3/planes/radius/local/resourceGroups/default/providers/Radius.Core/recipePacks/testrecipepack?api-version=2023-10-01-preview"
```

READ response:

```json    
{
  "id": "/planes/radius/local/resourcegroups/default/providers/Radius.Core/recipePacks/testrecipepack",
  "location": "global",
  "name": "testrecipepack",
  "properties": {
    "description": "Test recipe pack with sample recipes",
    "provisioningState": "Succeeded",
    "recipes": {
      "Applications.Datastores/redisCaches": {
        "parameters": {
          "tier": "basic"
        },
        "recipeKind": "bicep",
        "recipeLocation": "https://github.com/example/recipes/redis-cache.bicep"
      },
      "Applications.Datastores/sqlDatabases": {
        "parameters": {
          "backup": false,
          "size": "small"
        },
        "recipeKind": "terraform",
        "recipeLocation": "https://github.com/example/recipes/sql-database"
      }
    }
  },
  "systemData": {
    "createdAt": "0001-01-01T00:00:00Z",
    "createdBy": "",
    "createdByType": "",
    "lastModifiedAt": "0001-01-01T00:00:00Z",
    "lastModifiedBy": "",
    "lastModifiedByType": ""
  },
  "tags": {},
  "type": "Applications.Core/recipePacks"
}                                                                               
```


### Server Side changes

At a high level, below changes are necessary:

#### UCP

Add support to UCP to route `Radius.Core/recipePacks` resource operations to Applications RP in below section. (might need more changes)

/radius/deploy/Chart/templates/ucp/configmaps.yaml

```yaml
initialization:
  planes:
    - id: "/planes/radius/local"
      properties:
        resourceProviders:
          Applications.Core: "http://applications-rp.radius-system:5443"
          Applications.Dapr: "http://applications-rp.radius-system:5443"
          Applications.Datastores: "http://applications-rp.radius-system:5443"
          Applications.Messaging: "http://applications-rp.radius-system:5443"
          Microsoft.Resources: "http://bicep-de.radius-system:6443"
        kind: "UCPNative"
```

#### Applications RP

There are two prerequisites for implementing recipe packs:

1. Support new Radius.Core namespace. We would add Radius.Core namespace so that there is a transition time for users to move from Applications.Core/environments and recipes to Radius.Core/environment and Radius.Core/recipePacks. Eventually Applications.Core will be deprecated and removed.

2. Radius.Core/environments design and implementation.
   
Below changes are needed for supporting recipe packs as new feature:

1. Add schema /swagger changes to support the `Radius.Core/recipePacks` resource type ([typespec changes](#schema-and-api-design))

2. Create datamodel and convertors for handling recipe pack resource in /radius/`pkg/corerp/api/v20231001preview/ and /radius/pkg/corerp/datamodel/`
        
3. Add controller support for creating/updating/listing/deleting the resource in `/radius/pkg/corerp/frontend/controller/`. Constraints for each operation are captured in [Recipe Pack Operations](#schema-and-api-design)
   
4. Update applications_core.yaml manifest to include the new type.

5. When an Applications RP supported resource is being deployed, the resource could be recipe based (portable) or non recipe based today (applications and environment). For recipe based resources, `radius/pkg/rp/util/recipepacks.go` should be created, and have ability to fetch recipepacks, similar to `/radius/pkg/rp/util/environment.go`. Then `/radius/env-sup-rp/pkg/recipes/configloader/environment.go#L72` `getConfiguration` function must be updated to fetch the recipe pack resource one by one, and iterate until a recipe pack containing recipe for resource type of interest is found.   
   
This flow should come in place if the environment used for deploying is of type Radius.Core/environments only. We retain current behavior for Applications.Core/environments.

As part of Radius.Core/environments design/implementation below points should be considered:

1. Finalize the `Radius.Core/environments` details and then

2. Add schema / swagger changes to support the `Radius.Core/environments` resource type

3. Add convertors for handling conversions from and to version agnostic data model. 

4. Add backend/controller support for creating/updating/deleting the resource. 

5. rad env register should support registering a recipe pack to Radius.Core/environments resource, and disallow recipes.

#### Dynamic RP changes

In Dynamic RP, while deploying a dynamic resource:

1. Add support to look up the `Radius.Core/environments` that is in use, fetch environment's recipe-pack ids
2. Go over the recipe packs registered in environment one by one until the first recipe pack holding the recipe for the resource type of interest is found.  
3. Use the recipe information just fetched and construct recipe details that can be passed to the existing recipe engine mechanism. 
  
Since Dynamic RP shares the recipe engine code with Applications RP, Dynamic Resources should be able to avail recipe packs once Applications RP code changes are complete. 


#### Other components

No changes to controller or DE.

### CLI design

We should introduce rad cli commands to help manage recipe-packs. We should add documentation to rad recipe commands indicating their future deprecation plan.

1. Creating a recipe-pack should work once we create and register the recipe pack schema:

```bicep
computeRecipePack.bicep:

resource computeRecipePack 'Radius.Core/recipePacks@2025-05-01-preview' = {
  name: 'computeRecipePack'
  description: "Recipe Pack for deploying to Kubernetes."
  properties: {
    recipes: {
      'Radius.Compute/containers': {
        recipeKind: 'terraform'
        recipeLocation: 'https://github.com/project-radius/resource-types-contrib.git//recipes/compute/containers/kubernetes?ref=v0.48'
        parameters: {
          allowPlatformOptions: true
        }
      }
      'Radius.Security/secrets': {
        recipeKind: 'terraform'
        recipeLocation: 'https://github.com/project-radius/resource-types-contrib.git//recipes/security/secrets/kubernetes?ref=v0.48'
      }
      'Radius.Storage/volumes': {
        recipeKind: 'terraform'
        recipeLocation: 'https://github.com/project-radius/resource-types-contrib.git//recipes/storage/volumes/kubernetes?ref=v0.48'
      }
    }
  }
}

rad deploy computeRecipePack.bicep
```

The deploy operation should fail if the recipepack already exists AND there is atleast one resource deployed using the recipe pack, unless we are updating the pack to incrementally adding another type to the recipe pack. 

If there is an edit to any of the recipe attributes (recipeKind/recipeLocation/parameters), For now, we should return error mentioning a new recipe pack should be created, but once we have the version support, we should provide error message indicating such changes should happen in a newer version of recipe.

Note: rad recipe-pack create command could be added as a fast follow feature. For now, we are using rad deploy to create recipe packs.  


1. Show recipe pack details:

```
$ rad recipe-pack show computeRecipePack............................................

RESOURCE                TYPE                      GROUP     STATE
computeRecipePack       Radius.Core/recipePacks   default   Succeeded

RESOURCE TYPE                    RECIPE KIND          RECIPE LOCATION    RECIPE PARAMETERS
Radius.Compute/containers        terraform                           https://github.com/project-radius/resource-types-contrib.git//recipes/compute/containers/kubernetes?ref=v0.48  allowPlatformOptions(bool)
Radius.Security/secrets          terraform                           https://github.com/project-radius/resource-types-contrib.git//recipes/security/secrets?ref=v0.48
Radius.Storage/volumes           terraform                           https://github.com/project-radius/resource-types-contrib.git//recipes/storage/volumes?ref=v0.48
```

3. List recipe packs:

```
$ rad recipe-pack list [-g mygroup]

Lists all recipe packs in all scopes. 

RESOURCE                TYPE                      GROUP     STATE
computeRecipePack       Radius.Core/recipePacks   default   Succeeded
dataRecipePack          Radius.Core/recipePacks   default   Succeeded
```

if -g is provided, list recipe packs in that scope.

4. Show environment

```
$ rad environment show my-env
RESOURCE            TYPE                            GROUP     STATE
my-env              Radius.Core/environments        default   Succeeded

RECIPE PACK         RESOURCE TYPE                    RECIPE KIND     RECIPE VERSION      RECIPE LOCATION            RECIPE PARAMETERS
computeRecipePack   Radius.Compute/containers        terraform                           https://github.com/project-radius/resource-types-contrib.git//recipes/compute/containers/kubernetes?ref=v0.48.   myparameter(int)
computeRecipePack   Radius.Security/secrets          terraform                           https://github.com/project-radius/resource-types-contrib.git//recipes/security/secrets?ref=v0.48
computeRecipePack   Radius.Storage/volumes           terraform                           https://github.com/project-radius/resource-types-contrib.git//recipes/storage/volumes?ref=v0.48
dataRecipePack      Radius.Data/redisCaches          terraform                           https://github.com/project-radius/resource-types-contrib.git//recipes/data/redisCaches?ref=v0.48
```

1. Delete recipe pack:

```
$ rad recipe-pack delete <recipe-pack-name>
```

We could delete recipe-packs that are not referenced by any environment in any resource-group. This requires further thought and is similar to deletion of resources such as resource-type resource we have today.

1. Create an Environment with a Recipe Pack:

```
$ rad environment create myEnv
If Recipe Packs is not specified when creating the Environment, the `rad environment create` command sets default Recipe Packs (the same behavior as `rad init`).

$ rad environment create myEnv \
  --recipe-packs computeRecipePack
This command creates an environment with the `recipePacks[]` property populated with `computeRecipePack`.

$ rad environment update myEnv \
  --recipe-packs computeRecipePack
The command should use the resource ID to identify the environment is `Radius.Core\environment` resource and only then allow the `recipePack` property to be updated. If the namespace is `Applications.Core`, an error is provided.

$ rad environment update myEnv \
  --recipe-packs otherResourceGroup/computeRecipePack
If the Recipe Pack is in a different Resource Group than the Environment, the Resource Group is passed as a prefix to the Recipe Pack name.
```


1. Show environment details:

Based on whether the environment namespace is Applications.Core or Radius.Core, the outputs differ. Eventually Applications.Core support will be removed.

```
$ rad environment show my-env
RESOURCE            TYPE                      GROUP     STATE
my-env              Radius.Core/environments  default   Succeeded

RECIPE PACK         RESOURCE TYPE                    RECIPE KIND     RECIPE VERSION      RECIPE LOCATION
computeRecipePack   Radius.Compute/containers        terraform                           https://github.com/project-radius/resource-types-contrib.git//recipes/compute/containers/kubernetes?ref=v0.48
computeRecipePack   Radius.Security/secrets          terraform                           https://github.com/project-radius/resource-types-contrib.git//recipes/security/secrets?ref=v0.48
computeRecipePack   Radius.Storage/volumes           terraform                           https://github.com/project-radius/resource-types-contrib.git//recipes/storage/volumes?ref=v0.48
dataRecipePack      Radius.Data/redisCaches          terraform                           https://github.com/project-radius/resource-types-contrib.git//recipes/data/redisCaches?ref=v0.48
```

7. List recipes in environment:

```
$ rad recipe list -environment my-env
RECIPE PACK             RESOURCE TYPE                    RECIPE KIND     RECIPE VERSION      RECIPE LOCATION
computeRecipePack       Radius.Compute/containers        terraform                           https://github.com/project-radius/resource-types-contrib.git//recipes/compute/containers/kubernetes?ref=v0.48
computeRecipePack       Radius.Security/secrets          terraform                           https://github.com/project-radius/resource-types-contrib.git//recipes/security/secrets?ref=v0.48
computeRecipePack       Radius.Storage/volumes           terraform                           https://github.com/project-radius/resource-types-contrib.git//recipes/storage/volumes?ref=v0.48
dataRecipePack          Radius.Data/redisCaches          terraform                           https://github.com/project-radius/resource-types-contrib.git//recipes/data/redisCaches?ref=v0.48
networkingRecipePack    Radius.Compute/gateways          terraform                           https://github.com/project-radius/resource-types-contrib.git//recipes/compute/gateways?ref=v0.48
```

8. Initialize Radius:

Today, rad init works as shown below:

```                                                  
Initializing Radius. This may take a minute or two...
                                                     
🕒 Install Radius 7af38a9                            
   - Kubernetes cluster: kind-kin2                   
   - Kubernetes namespace: radius-system             
⏳ Create new environment default                    
   - Kubernetes namespace: default                   
   - Recipe pack: local-dev                          
⏳ Scaffold application resource-types-contrib       
⏳ Update local configuration  
```

The default environment created is initialized with a "recipe pack" which is a bunch of kubernetes recipes. 

We are choosing to keep the same behavior. But behind the scenes rad init would create a recipe pack resource, with recipe links we use today to construct recipe properties and add this recipe pack's resource id to the environment. 

Providing an option to initialize Radius for az/aws based recipe packs requires considerable work and would be a future follow up to the feature. At a high level, that feature would require the users to be able to specify a location to  yaml definition for recipe-packs at the time of init and then construct the recipe-pack resource based on that.

### Graph support

Recipe packs will be displayed in application graphs as first-class resources with their own lifecycle and relationships to environments.

### Logging/Tracing support

Standard logging and tracing will be implemented for all recipe pack operations through the existing Applications RP logging/tracing infrastructure.


### Community

Recipe Packs would be authored and shared across community through resource-types-contrib repo. 


### Breaking changes

* Once we support recipe packs, the Radius.Core environments will allow only registration of recipe-packs and not a single recipe. Applications.Core environments will continue to work as it does today and support recipe registration but will be deprecated over time allowing transition time for customers to move from recipes to recipe packs. 
  
We could also explore providing some tools to create the new environment resource and recipe packs based on existing environment and recipe information to ease transition. 

* We will drop the support for named recipe - a way to register multiple recipes for the same resource-type in a single environment.



## Test plan

[To be completed]

## Security

[To be completed]

## Compatibility (optional)

[To be completed]

## Monitoring

[To be completed]

## Development plan

1. 

## Open issues

[To be completed]


## notes/questions for myself (to be deleted before merging)

- recipe-pack : RRT or NOT? same question as RRT in the Radius.Config namespace (app, env). These types are not meant to be edited by users, and should be as defined by Radius so that Radius can work. (imperatative)
- rad init today says installing a "recipe-pack" -  might need changes here to enable choosing a pack. (retain behavior)
- rad init / install must be updated to create the recipe pack resource type ( as part of registering manifests logic we have today) 
- if we want the ability to "init" with a selected pack say for az, how would we do it? it might help to allow url to recipe-pack manifest, and as part of rad init , create the rrt as well as the rrt resource (using the url for yaml manifest) and init the env with it. (in future)
- would environment and recipe pack ever have different rbac? should we "allow" recipepack is in different radius resource grpup from that of environment? (yes , with -g, consider 2 diff env owned by 2 diff team)
- prereq: finalize new environment design
- handling recipe-packs with dup recipes. at the time of creation, of recipe pack, it could have dups with another recipe-pack. But we have to dtect dups at the time of registering to env. 
-  we are moving away from named recipes. If a customer wishes to use same env for two applications, these application teams have their own recipes, then would we advice them to create 2 enviroments ? We could also guide them to a naming like contoso-recipe-pack and cool-prod-recipe-pack. But this would either require us to allow for duplicate recipes between packs, or require multiple teams to coordinate and ensure their types are different? (its is upto the teams to decide how they want to organize)
-  now that the "unit" of importing recipes is recipe pack, would we "contrib" recipe pack manifests? (yes)
-  would we think about enforcing a size limit on recipe pack? how many recipes it can have ? (not now)
-  - should recipe-pack resource have a list of env ids to which it belongs? typically, RRTs have an app id or env id. (no, causes a cycle)
-  - cli -support versus deploying through bicep (create cmd can come later for recipe pack)