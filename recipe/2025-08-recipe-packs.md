# Recipe Packs

* **Author**: `Nithya (@nithyatsu)`

## Overview

Recipes are external infrastructure-as-code (IaC) templates that operators register on a Radius Environment so developers can use them later for provisioning. They provide a mechanism for separation of concerns between developers and operators. 

Today, Radius supports registering recipes individually, either via the `Applications.Core\environments` resource properties or the CLI (`rad recipe register`). For each Radius Environment, platform engineers have to piece together individual Recipes from scratch. Some customers could have 1000s of environments and putting everything together manually for each environment is error prone. Recipes also do not have a lifecycle of their own and can be managed only by managing the environments pointing to them.

This document proposes the design of a Recipe Pack as an first class resource type in Radius. The Recipe Pack enables bundling multiple recipe selections for different resource types into a reusable unit that can be referenced by environments. 

## Terms and definitions

| Term     | Definition                                                                                                                                                                                                 |
| -------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Recipe | IaC templates that operators register on a Radius Environment |
| Recipe Pack| A collection of recipes that can be managed as an entity |
| RRT | [Radius Resource Type](https://docs.radapp.io/guides/author-apps/custom/overview/)|

## Objectives

> **Issue Reference:** 

### Goals

Provide Recipe Packs as a Radius feature to bundle multiple recipes as a single manageable entity into a Radius Environment (e.g., a pack for ACI that includes all necessary recipes).

- Radius should provide APIs to manage Radius.Core/recipePacks resource through CRUDL operations  

### Non goals

Recipe Packs would bundle together "recipes" as we understand them today. We do not cover recipe versioning / other recipe specific enhancements in this design. 


### User scenarios (optional)

#### Registering several recipes to an environment

As an operator I am responsible for creating Radius Environments using which developers can deploy their applications. As part of creating the environment, I manually link recipes one by one using `rad recipe register` or by updating the environment definition. This can be error prone when there are many recipes and environments.  Radius should provide a way to bulk register( and manage) recipes.

#### Registering recipes to multiple environment

As an operator I am responsible for creating Radius Environments using which developers can deploy their applications. As part of creating the environment, I manually link recipes one by one using `rad recipe register` or by updating the environment definition. I having 100s of environment which mostly use the same recipes. Piecing the same recipes together for each environment feels like rework.

## Design

### Design Overview

We choose modeling recipe packs as a first class Radius resource of type 
Radius.Core/recipePacks provisioned imperatively by Applications RP . [Alternatives considered](#alternatives-considered) section details some other options we considered.

Pros:

- Solves the requirement for bulk registering recipes using single command with a one time effort of creating the recipe pack resource
- As first class resource, recipe packs would be displayed in app graphs. They can also have their own lifecycle and RBAC independent of environments. 
- Helps reduce the size of environment resource, which could reach serialization limits with tons of recipes. 
- Helps reduce overall size of Radius datastore, since common recipe information could now be stored as a single resource instead of being duplicated across several environments.
- Imperative provisioning prevents users from accidentally experimenting with the resource schema


Cons:

- This approach is a deviation from the current tooling approach for recipes. 
- Supporting recipe packs as a Applications RP type requires manual implementation of schema and API in contrast to modeling it as a dynamic resources. 

At a very high level, this design approach needs the below steps 

* Add support for Radius.Core/recipePacks resource in Applications RP 
  * Schema + API design and implementation
* Support `rad cli` commands that enable CRUDL operations on resources of type Radius.Core/recipePacks 
* Design and support Radius.Core/environment schema in Applications RP
* Support `rad cli` commands that enable managing Radius.Core/environment resources through CRUDL operations on this type of resource
* Support `rad cli` commands that enable registering recipe-packs to a `Radius.Core\environments` environment resource
* Enhance Dynamic RP, Applications RP and UCP to support the feature.

### Schema and API design

As part of supporting Recipe Pack as a resource type, at a high-level,
We define a recipePacks.tsp (Note we currently do not have a Radius.Core namespace and we have to make code changes to set this namespace up with at-least environments and recipePacks)
   
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

* We choose map of resource types to their recipe configurations so that the relevant recipe can be easily accessed. 
* We will not be supporting named recipes going forward as documented in [RRT feature spec](https://github.com/willtsai/design-notes-radius/blob/f9c98baf515263c27e7637131d7a48ae5a01b2c0/features/2025-02-user-defined-resource-type-feature-spec.md#user-story-7--registering-recipes).  Therefore the `RecipeDefinition` model does not include a name.
* The operations are all Synchronous, since Recipe Pack a light weight configuration resource.


* createOrUpdate returns a `HTTP 201 Created` when recipe pack creation is successful and `HTTP 200 OK` for  successful updates, with json payload representing the recipe pack resource created. 

[Sample to be added]

* get returns `HTTP 200 OK` + payload of the serialized recipe pack resource that corresponds to the resource ID.

[Sample to be added]

* delete returns `HTTP 200 OK`  upon successful deletion of recipe pack resource that corresponds to the resource ID. Other possible response is `HTTP 404 Not Found`.
  
[Sample to be added]

* listByScope returns `HTTP 200 OK` with payload containing a list of recipe packs in the given scope. 

[Sample to be added]
  
#### Example

Below is a sample bicep definition of a recipe pack resource -

```
resource computeRecipePack 'Radius.Core/recipePacks@2026-01-01-preview' = {
    name: 'computeRecipePack'
    description: "Recipe Pack for deploying to Kubernetes."
    properties: {
        recipes: { 
            Radius.Compute/container: {
                recipeKind: 'terraform'
                recipeLocation: 'https://github.com/project-radius/resource-types-contrib.git//recipes/compute/containers/kubernetes?ref=v0.48'
                parameters: {
                  allowPlatformOptions: true
                  anIntegerParam : 1
                }
            }
            Radius.Security/secrets: {
                recipeKind: 'terraform'
                recipeLocation: 'https://github.com/project-radius/resource-types-contrib.git//recipes/security/secrets/kubernetes?ref=v0.48'
            }
            Radius.Storage/volumes: {
                recipeKind: 'terraform'
                recipeLocation: 'https://github.com/project-radius/resource-types-contrib.git//recipes/storage/volumes/kubernetes?ref=v0.48'
            }
        }
    }
}
```

### Server Side changes

At a high level, below changes are necessary -

#### UCP

 Add support to UCP to route `Radius.Core\recipePacks` resource operations to Applications RP in below section. (might need more changes)

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

1. Support new Radius.Core namespace. We would add Radius.Core namespace so that there is a transition time for users to move from Applications.Core/environments and recipes to Radius.Core/environment and Radius.Core/recipePacks.
Eventually Applications.Core will be deprecated and removed.

2. Radius.Core/environments design and implementation.
   
Below changes are needed for supporting recipe packs as new feature. 

   1. Add schema /swagger changes to support the `Radius.Core\recipePacks` resource type ([typepec changes](#schema-and-api-design))

   2. Create datamodel and convertors for handling recipe pack resource in /radius/pkg/corerp/api/v20231001preview/ and /radius/pkg/corerp/datamodel/
            
   3. Add controller support for creating/updating/listing/deleting the resource in `/radius/pkg/corerp/frontend/controller/`. Constraints for each operation are captured in [Recipe Pack Operations](#schema-and-api-design)
   
   4. When an Applications RP supported resource is being deployed, the resource could be recipe based (portable) or non recipe based today (applications and environment). For recipe based resources, `/radius/pkg/portableresources/backend/controller/createorupdateresource.go` should be updated to 
      1. fetch environment's recipe-pack ids
      2. get recipe-pack resource one at a time, stopping as soon as a match for resource-type is found. 
      3. Then it should populate RecipeMetaData appropriately before calling the engine. 
   This flow should come in place if the environment used for deploying is of type Radius.Core/environments only.
   

As part of Radius.Core/environments design/ implementation below points should be considered:

   1. Finalize the `Radius.Core\environments` details and then

   2. Add schema / swagger changes to support the `Radius.Core\environments` resource type

   3. Add convertors for handling conversions from and to version agnostic data model. 

   4. Add backend/controller support for creating/updating/deleting the resource. 

   5. rad env register should support registering a recipe pack to Radius.Core/environments resource, and disallow recipes.



#### Dynamic RP changes

1. In Dynamic RP, while deploying a dynamic resource
   1. Add support to look up the `Radius.Core\environments` that is in use, fetch environment's recipe-pack ids
   2. Go over the recipe packs registered in environment one by one until the first recipe pack holding the recipe for the resource type of interest is found.  
   3. Use the recipe information just fetched and construct recipe details that can be passed to the existing recipe engine mechanism. 
   
#### Other components

2. No changes to controller or DE.


### CLI design

We should introduce rad cli commands to help manage recipe-packs.
We should add documentation to rad recipe commands indicating their future deprecation plan.

1. Creating a recipe-pack should work once we create and register the recipe pack schema. 

```
computeRecipePack.bicep:

resource computeRecipePack 'Radius.Core/recipePacks@2025-05-01-preview' = {
    name: 'computeRecipePack'
    description: "Recipe Pack for deploying to Kubernetes."
    properties: {
        recipes: [
            Radius.Compute/container: {
                recipeKind: 'terraform'
                recipeLocation: 'https://github.com/project-radius/resource-types-contrib.git//recipes/compute/containers/kubernetes?ref=v0.48'
                parameters: {
                allowPlatformOptions: true
                }
            }
            Radius.Security/secrets: {
                recipeKind: 'terraform'
                recipeLocation: 'https://github.com/project-radius/resource-types-contrib.git//recipes/security/secrets/kubernetes?ref=v0.48'
            }
            Radius.Storage/volumes: {
                recipeKind: 'terraform'
                recipeLocation: 'https://github.com/project-radius/resource-types-contrib.git//recipes/storage/volumes/kubernetes?ref=v0.48'
            }
        ]
    }
}

rad deploy computeRecipePack.bicep
```

Note: rad recipe-pack create command could be added in future. For now, we are using rad deploy to create recipe packs. 

1. 
```
$ rad recipe-pack show computeRecipePack

RESOURCE                TYPE                            GROUP     STATE
computeRecipePack       Radius.Core/recipePacks       default   Succeeded

RESOURCE TYPE                    RECIPE KIND     RECIPE VERSION      RECIPE LOCATION
Radius.Compute/containers        terraform                           https://github.com/project-radius/resource-types-contrib.git//recipes/compute/containers/kubernetes?ref=v0.48
Radius.Security/secrets          terraform                           https://github.com/project-radius/resource-types-contrib.git//recipes/security/secrets?ref=v0.48
Radius.Storage/volumes           terraform                           https://github.com/project-radius/resource-types-contrib.git//recipes/storage/volumes?ref=v0.48

```

3. 
```
$ rad recipe-pack list [optionally -e myenv]

RESOURCE                TYPE                          GROUP     STATE
computeRecipePack       Radius.Core/recipePacks       default   Succeeded
dataRecipePack.         Radius.Core/recipePacks       default   Succeeded

```

If no -e is specified, this command should list all recipe packs in current scope. 

4. rad recipe-pack delete 

We could delete recipe-packs that are not referenced by any environment in any resource-group. This requires  further thought and is similar to deletion of resources such as resource-type resource we have today.

5. rad env register recipe-pack recipe-pack-name [-g group id] 
We specify group resource id if the recipe-pack is in a different scope from the environment.

6. rad env show should be updated

Based on whether the environment namespace is Applications.Core or Radius.Core, the outputs differ. Eventually Applications.Core support will be removed.

```
$ rad environment show my-env
RESOURCE            TYPE                            GROUP     STATE
my-env              Radius.Core/environments        default   Succeeded

RECIPE PACK         RESOURCE TYPE                    RECIPE KIND     RECIPE VERSION      RECIPE LOCATION
computeRecipePack   Radius.Compute/containers        terraform                           https://github.com/project-radius/resource-types-contrib.git//recipes/compute/containers/kubernetes?ref=v0.48
computeRecipePack   Radius.Security/secrets          terraform                           https://github.com/project-radius/resource-types-contrib.git//recipes/security/secrets?ref=v0.48
computeRecipePack   Radius.Storage/volumes           terraform                           https://github.com/project-radius/resource-types-contrib.git//recipes/storage/volumes?ref=v0.48
dataRecipePack      Radius.Data/redisCaches          terraform                           https://github.com/project-radius/resource-types-contrib.git//recipes/data/redisCaches?ref=v0.48
```

7. rad recipe list
```
$ rad recipe list -environment my-env
RECIPE PACK             RESOURCE TYPE                    RECIPE KIND     RECIPE VERSION      RECIPE LOCATION
computeRecipePack       Radius.Compute/containers        terraform                           https://github.com/project-radius/resource-types-contrib.git//recipes/compute/containers/kubernetes?ref=v0.48
computeRecipePack       Radius.Security/secrets          terraform                           https://github.com/project-radius/resource-types-contrib.git//recipes/security/secrets?ref=v0.48
computeRecipePack       Radius.Storage/volumes           terraform                           https://github.com/project-radius/resource-types-contrib.git//recipes/storage/volumes?ref=v0.48
dataRecipePack          Radius.Data/redisCaches          terraform                           https://github.com/project-radius/resource-types-contrib.git//recipes/data/redisCaches?ref=v0.48
networkingRecipePack    Radius.Compute/gateways          terraform                           https://github.com/project-radius/resource-types-contrib.git//recipes/compute/gateways?ref=v0.48
```

8. rad init 

Today, rad init works as show below:

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

We are choosing to keep the same behavior. But behind the scenes rad init would create. recipe pack resource, with recipe links we use today to construct recipe properties and add this recipe pack's resource id to the environment. 

Providing an option for az/ aws based recipe packs requires considerable work and would be a future follow up to the feature. 

### Graph support

### Logging/ Tracing support

### Breaking changes

* Once we support recipe packs, the Radius.Core environments will allow only registration of recipe-packs and not a single recipe. Applications.Core environments will continue to work as it does today and support recipe registration but will be deprecated over time allowing transition time for customers move from recipes to recipe packs. 
  
* We will drop the support for named recipe - a way to register multiple recipes for the same resource-type in a single environment.


## Alternatives considered

Below options were considered as alternatives to modeling recipe pack as a first class RRT -

1. embed all recipe mappings inline in the Environment 

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

- Environment still stays a bloated object. Environment resource houses a lot of other properties and we could potentially risk hiting the mechanical limits that apply to serializing objects. 
  
- Add Radius commands to publish recipe-packs, similar to what we have for recipes today. 

- a list of recipes could potentially be managed as a collection, including having its own rbac and appearance in app graph. The above approach does not allow for that possibility.


1. store a URL to a YAML manifest in the Environment

We could fetch the yaml when needed, and use the available recipe.

`rad environment update my-env --recipe-packs aci-production-pack='git::https://github.com/my-org/recipe-packs.git//aci-production-pack.yaml?ref=1.0.0'`

Pros:

- Helps manage the size of environment resource
- Solves the requirement for bulk registering recipes using single command with a one time effort of creating the yaml manifest

Cons:

- For each provisioning of resource, we make a call to registry to fetch the list to check if the recipe is available, and then a call to the specified recipe location to fetch it. We could cache the list and construct an im-memory recipe pack ephemeral object. But we still do not get the benefits of recipe pack as a first-class resource type. 

1. model recipe packs as Radius Resource type provided by Dynamic RP. 

Pros:

- Solves the requirement for bulk registering recipes using single command with a one time effort of creating the recipe pack resource
- As first class resource, recipe packs would be displayed in app graphs. They can also have their own lifecycle and rbac independent of environments. 
- Helps reduce the size of environment resource, which could reach serialization limits with tons of recipes. 
- Helps reduce overall size of Radius datastore, since common recipe information could now be stored as a single resource instead of being duplicated across several environments.

Cons:

- While this brings in just as many advantages as the chosen design approach, RRTs can have their schema modified using rad resource-type commands. We would have to find ways to prevent this from happening. 

In general, Radius.Core namespace has resources whose schema should be non-editable so that Radius can work as expected, for example Applications, Environments and recipePacks. These resources must be provisioned imperatively by Applications RP.

## Test plan



## Security



## Compatibility (optional)


## Monitoring


## Development plan




## Open issues


## notes/questions for myself (to be deleted)

- recipe-pack : RRT or NOT? same question as RRT in the Radius.Config namespace (app, env). These types are not meant to be edited by users, and should be as defined by Radius so that Radius can work. 
- rad init today says installing a "recipe-pack" -  might need changes here to enable choosing a pack.
- rad init / install must be updated to create the recipe pack resource type ( as part of registering manifests logic we have today) 
- if we want the ability to "init" with a selected pack say for az, how would we do it? it might help to allow url to recipe-pack manifest, and as part of rad init , create the rrt as well as the rrt resource (using the url for yaml manifest) and init the env with it. 
- would environment and recipe pack ever have different rbac? should we "allow" recipepack is in different radius resource grpup from that of environment?
- - prereq: finalize new environment design
- handling recipe-packs with dup recipes. at the time of creation, of recipe pack, it could have dups with another recipe-pack. But we have to dtect dups at the time of registering to env. 
-  we are moving away from named recipes. If a customer wishes to use same env for two applications, these application teams have their own recipes, then would we advice them to create 2 enviroments ? We could also guide them to a naming like contoso-recipe-pack and cool-prod-recipe-pack. But this would either require us to allow for duplicate recipes between packs, or require multiple teams to coordinate and ensure their types are different?
-  now that the "unit" of importing recipes is recipe pack, would we "contrib" recipe pack manifests? 
-  would we think about enforcing a size limit on recipe pack? how many recipes it can have ?
-  - should recipe-pack resource have a list of env ids to which it belongs? typically, RRTs have an app id or env id.
-  - cli -support versus deploying through bicep




