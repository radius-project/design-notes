# Recipe Packs

* **Author**: `Nithya (@nithyatsu)`

## Overview

Recipes are external infrastructure-as-code (IaC) templates that operators register on a Radius Environment so developers can use them later for provisioning. They provide a mechanism for separation of concerns between developers and operators. 

Today, Radius supports registering recipes individually, either via the `Applications.Core\environments` resource properties or the CLI (`rad recipe register`). For each Radius Environment, platform engineers have to piece together individual Recipes from scratch. Some customers could have 1000s of environments and putting everything together manually for each environment is error prone. Recipes also do not have a lifecycle of their own and can be managed only by managing the environments pointing to them.

This document details the introduction of Recipe Packs as a feature that makes recipe management easier. Recipe Packs are a collection of related recipes that a platform engineer can manage as a single entity. 

## Terms and definitions

| Term     | Definition                                                                                                                                                                                                 |
| -------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Recipe | IaC templates that operators register on a Radius Environment |
| Recipe Pack| A collection of recipes that can be managed as an entity |
| RRT | [Radius Resource Type](https://docs.radapp.io/guides/author-apps/custom/overview/)|

## Objectives

> **Issue Reference:** 

### Goals

Provide Recipe Packs as a Radius feature to bundle multiple recipes as a single managable entity into a Radius Environment (e.g., a pack for ACI that includes all necessary recipes).

### Non goals

Recipe Packs would bundle together Recipes, as we understand them today. We do not cover recipe versioning / other recipe specific enhancements in this design. 


### User scenarios (optional)

#### Registering several recipes to an environment

As an operator I am responsible for creating Radius Environments using which developers can deploy their applications. As part of creating the environment, I manually link recipes one by one using `rad recipe register` or by updating the environment definition. This can be error prone when there are many recipes and environments.  Radius should provide a way to bulk register( and manage) recipes.

#### Registering recipes to multiple environment

As an operator I am responsible for creating Radius Environments using which developers can deploy their applications. As part of creating the environment, I manually link recipes one by one using `rad recipe register` or by updating the environment definition. I having 100s of environment which mostly use the same recipes. Piecing the same recipes together for each environment feels like rework.

## Design

### Design Overview

We choose modelling Recipe Pack as a first class Radius Resource Type. [Alternatives considered](#alternatives-considered) section details some other possibilities.

Pros:

- Solves the requirement for bulk registering recipes using single command with a one time effort of creating the recipe pack resource
- As first class resource, recipe packs would be displayed in app graphs. They can also have their own lifecycle and rbac independant of environments. 
- Helps reduce the size of environment resource, which could reach serization limits with tons of recipes. 
- Helps reduce overall size of Radius datastore, since common recipe information could now be stored as a single resource instead of being duplicated across several environments.

[Question]: would environment and recipe pack ever have different rbac?

Cons:

- This approach is a deviation from the current tooling approach for recipes. 
- While this brings in many advantages, RRTs can have their schema modified using rad resource-type commands. We should find ways to prevent this from happening. In general, Radius.Core namespace would have resources whose schema should be non-editable so that Radius can work as expected. Some other resources that would fall in this category are environments and applications.

At a high level, this design approach needs the below steps 

* Design schema for Radius.Core/recipePacks type
* Register the resultant schema manifest as part of Radius start up sequence
* Support `rad cli` commands that enable CRUDL operations on resources of type Radius.Core/recipePacks 
* Design Radius.Core/environment schema 
* Register the schema as part of Radius start up sequence
* Support `rad cli` commands that enable managing Radius.Core/environment resources through CRUDL operations on this type of resource
* Support `rad cli` commands that enable registering recipe-packs to a `Radius.Core\environments` environment resource
* Enhance Dynamic RP to look through the set of registered recipe-packs in the active environment, fetch the recipe-pack resources and then fetch the link of recipe for resource provisioning

### 

A sample recipe pack resource would look like below:

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

The schema for the type would look like:

```yaml
namespace: Radius.Core
  types:
    recipePacks:
      description: Recipe Pack for grouping and managing related recipes
      apiVersions:
        '2026-01-01-preview':
          schema:
            type: object
            properties:
              name:
                type: string
                description: The name of the recipe pack
              description:
                type: string
                description: Description of what this recipe pack provides
              recipes:
                type: object
                description: Map of resource types to their recipe configurations
                additionalProperties:
                  type: object
                  properties:
                    recipeKind:
                      type: string
                      description: The type of recipe (e.g., terraform, bicep)
                      enum:
                        - terraform
                        - bicep
                    recipeLocation:
                      type: string
                      description: URL or path to the recipe source
                    parameters:
                      type: object
                      description: Parameters to pass to the recipe
                      additionalProperties:
                        type: any
                  required:
                    - recipeKind
                    - recipeLocation
            required:
              - name
              - recipes
```

Today the RRT schema does not allow `any` as a type for security reasons. We would have to remove that constraint in bicep tooling and radius so that we can have the recipe parameters as defined above.

// Question: The recipe collection should be curly braces, is that OK (feature spec has []) ? OR we keep it array like below. Map could make it easier to look up using resource type.

"recipes": [
    {
      "resourceType": "Radius.Compute/containers",
      "recipeKind": "terraform",
      "recipeLocation": "https://example.com/recipes/containers.zip"
    },
    {
      "resourceType": "Radius.Storage/volumes",
      "recipeKind": "terraform",
      "recipeLocation": "https://example.com/recipes/volumes.zip"
    }
  ]

### API design (if applicable)

We should support CRUDL operations on recipe-pack resource. This should be fairly automatic since recipe packs would be registered as an RRT. Once we register the schema for `Radius.Core/recipePacks` resource type using `rad resource-type create`,  Dynamic RP should be able to dynamically look up the schema and perfom these operations. 


### Server Side changes



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

2. 
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

4. rad recipe-pack delete 

We could delete recipe-packs that are not referenced by any environment in any resource-group. This requires  further thought and is similar to deletion of resources such as resource-type resource we have today.

5. rad env show should be updated

Based on whether the environment namepsace is Applications.Core or Radius.Core, the outputs differ. 

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

```
$ rad recipe list -environment my-env
RECIPE PACK             RESOURCE TYPE                    RECIPE KIND     RECIPE VERSION      RECIPE LOCATION
computeRecipePack       Radius.Compute/containers        terraform                           https://github.com/project-radius/resource-types-contrib.git//recipes/compute/containers/kubernetes?ref=v0.48
computeRecipePack       Radius.Security/secrets          terraform                           https://github.com/project-radius/resource-types-contrib.git//recipes/security/secrets?ref=v0.48
computeRecipePack       Radius.Storage/volumes           terraform                           https://github.com/project-radius/resource-types-contrib.git//recipes/storage/volumes?ref=v0.48
dataRecipePack          Radius.Data/redisCaches          terraform                           https://github.com/project-radius/resource-types-contrib.git//recipes/data/redisCaches?ref=v0.48
networkingRecipePack    Radius.Compute/gateways          terraform                           https://github.com/project-radius/resource-types-contrib.git//recipes/compute/gateways?ref=v0.48
```


### Breaking changes

* Once we support recipe packs, the Radius.Core environments will allow only regitration of recipe-packs and not a single recipe. Applications.Core environments will continue to work as it does today and support recipe registeration but will be deprecated over time allowing transition time for customers move from recipes to recipe packs. 
  
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

## Test plan



## Security



## Compatibility (optional)


## Monitoring


## Development plan




## Open issues


## notes/questions for myself (TBD)

- recipe-pack : RRT or NOT? same question as RRT in the Radius.Config namespace (app, env). These types are not meant to be edited by users, and should be as defined by Radius so that Radius can work. 
- rad init today says instaling a "recipe-pack" -  might need changes here to enable choosing a pack.
- rad init / install must be updated to create the recipe pack resource type ( as part of registering manifests logic we have today) 
- if we want the ability to "init" with a selected pack say for az, how would we do it? it might help to allow url to recipe-pack manifest, and as part of rad init , create the rrt as well as the rrt resource (using the url for yaml manifest) and init the env with it. 
- handling recipe-packs with dup recipes. at the time of creation, of recipe pack, it could have dups with another erescipe-pack. But we have to dtect dups at the time of registering to env. 
-  we are moving away from named recipes. If a customer wishes to use same env for two applications, these application teams have their own recipes, then would we advice them to create 2 enviroments ? We could also guide them to a naming like contoso-recipe-pack and cool-prod-recipe-pack. But this would either require us to allow for duplicate recipes between packs, or require multiple teams to coordinate and ensure their types are different?
-  now that the "unit" of importing recipes is recipe pack, would we "contrib" recipe pack manifests? 
-  would we think about enforcing a size limit on recipe pack? how many recipes it can have ?
-  - should recipe-pack resource have a list of env ids to which it belongs? typically, RRTs have an app id or env id.
-  - cli -support versus deploying through bicep




