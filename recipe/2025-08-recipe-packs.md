# Recipe Packs

* **Author**: `Nithya (@nithyatsu)`

## Overview

Recipes are external infrastructure-as-code (IaC) templates that operators register on a Radius Environment so developers can use them later for provisioning. They provide a mechanism for separation of concerns between developers and operators. 

Today, Radius supports registering recipes individually, either via the Environment resource properties or the CLI (`rad recipe register`). For each Radius Environment, platform engineers have to piece together individual Recipes from scratch. Putting everything together manually this way for each environment is error prone. Recipes also do not have a lifecycle of their own and can be managed only by managing the environments pointing to them.

This document details the introduction of Recipe Packs as a feature that makes recipe management easier. Recipe Packs are a collection of related recipes that a platform engineer can manage as a single entity. 

## Terms and definitions

| Term     | Definition                                                                                                                                                                                                 |
| -------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Recipe | IaC templates that operators register on a Radius Environment |
|Recipe Pack| A collection of recipes that can be managed as an entity |

## Objectives

> **Issue Reference:** 

### Goals

Provide Recipe Packs as a Radius feature to bundle related recipes for easier management and sharing (e.g., a pack for ACI that includes all necessary recipes).

### Non goals

Recipe Packs would bundle together Recipes, as we understand them today. We do not cover recipe versioning / other recipe specific enhancements in this design. 


### User scenarios (optional)

#### Registering several recipes to an environment

As an operator I am responsible for creating Radius Environments using which developers can deploy their applications. As part of creating the environment, I manually link recipes one by one using `rad recipe register` or by updating the environment definition. This can be error prone when there are many recipes.  Radius should provide a way to bulk register( and manage) recipes.

#### Registering recipes to multiple environment

As an operator I am responsible for creating Radius Environments using which developers can deploy their applications. Some of my developers use ACI computes while others use Kubernetes. But they all use same networking resources. Today, I have to create 2 Radius environment and manually link the recipe for compute, data. and networking for each of these teams, since they need different computes. Or I can create one dev environment and have named recipes, so that each team can see a compute recipe they dont use along with the one they use. I would like to be able to better organize recipes so that I can group them together independent of the environment, and reuse them across environment when suitable. In my above situation, I could register the same data, networking recipe-packs across environments and only link a different compute pack.

## Design

### Design details

We model Recipe Pack as a first class Radius resource.

Pros:

- Solves the requirement for bulk registering recipes using single command with a one time effort of creating the recipe pack resource
- As first class resource, recipe packs would be displayed in app graphs. They can also have their own lifecycle and rbac independant of environments. 
- Helps manage the size of environment resource

[Question]: would environment and recipe pack ever have different rbac?

Cons:

- This approach is a deviation from the current tooling approach for recipes. 



### Schema

A sample recipe pack resource looks like below:

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
                        type: string
                  required:
                    - recipeKind
                    - recipeLocation
            required:
              - name
              - recipes
```

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
  
- a list of recipes could potentially be managed as a collection, including having its own rbac and appearance in app graph. The above approach does not allow for that possibility.

2. store a URL to a YAML manifest in the Environment

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



