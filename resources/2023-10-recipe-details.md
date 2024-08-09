# Adding Recipe Information to the Resource

* **Author**: Yetkin Timocin (@ytimocin)

## Overview

Recipes is a feature that is offered by Radius to make the deployment of resources easier for its users. To learn more about recipes please visit <https://edge.radapp.dev/guides/recipes/overview/>. If a resource is deployed by a recipe, we would like to know this in the resource and data model level.

## Terms and definitions

| Term | Definition |
|---|---|
| Recipe | Recipes enable a separation of concerns between infrastructure operators and developers by automating infrastructure deployment. |
| Template Kind | The format of the template provided by the recipe. Allowed values: bicep, terraform. |
| Template Path | The path to the template provided by the recipe. |
| Template Version | The version of the template to deploy. Only applies to Terraform recipes. |

## Objectives

> **Issue Reference:**

<https://github.com/radius-project/radius/issues/6440>

### Goals

* The goal is to add the recipe information to the resource and the data model level so that when the resource details is requested, recipe information will also be available.

### Non goals

* Everything other than adding the recipe information to the resource and data model level.
* No changes to any of the existing APIs. The APIs will automatically pick up the changes.

### User scenarios (optional)

#### User story 1

As a Radius User/Operator, I would like to get the recipe information of a resource to use it in the client-side. For example, I would like to get the application graph which is going to give me information about the resources. I would also like to know which resource was deployed by which recipe.

## Design

We are going to add a new property to the resource status properties. This new property will be a map and will include templateKind, templatePath, and templateVersion (if the kind is Terraform). We need to add this to all the resources that could be provisioned by a recipe: containers, extenders, and all the portable resources.

The design will consist of the following steps:

1. Update all the necessary typespec files (an example will be given below).
2. All the related conversion files need to be updated.
3. All the related conversion test files need to be updated.
4. The function to create and/or update should be updated for both corerp and all the Portable Resources.
5. RecipeEngine and the Drivers need to be updated so that they can return the necessary Recipe details. (Not sure about this one!)
6. Swagger definitions need to be updated for the necessary resources.
7. typespec details need to be updated.

### API design

All the LIST and GET APIs will automatically be updated to return the recipe information. The new response will look something like this:

`json``
{
  "id": "/planes/radius/local/resourcegroups/default/providers/Applications.Datastores/redisCaches/redis",
  "location": "global",
  "name": "redis",
  "properties": {
    "application": "/planes/radius/local/resourceGroups/default/providers/applications.core/applications/demo",
    "environment": "/planes/radius/local/resourceGroups/default/providers/applications.core/environments/default",
    "host": "redis-hjo6ha3uqagio.default-demo.svc.cluster.local",
    "port": 6379,
    "provisioningState": "Succeeded",
    "recipe": {
      "name": "default"
    },
    "resourceProvisioning": "recipe",
    "status": {
      "recipe": {
        "templateKind": "bicep",
        "templatePath": "radius.azurecr.io/recipes/local-dev/rediscaches:latest"
      },
      "outputResources": [
        {
          "id": "/planes/kubernetes/local/namespaces/default-demo/providers/core/Service/redis-hjo6ha3uqagio",
          "radiusManaged": true
        },
        {
          "id": "/planes/kubernetes/local/namespaces/default-demo/providers/apps/Deployment/redis-hjo6ha3uqagio",
          "radiusManaged": true
        }
      ]
    },
    "tls": false,
    "username": ""
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
  "type": "Applications.Datastores/redisCaches"
}
`json``

***Model changes***

Addition of RecipeInformation type to resources.tsp

`typespec``
@doc("Recipe details used at deployment time for a resource.")
model RecipeInformation {
  @doc("Template kind is the kind of template.")
  templateKind: string;

  @doc("TemplatePath is the path specified in the recipe.")
  templatePath: string;

  @doc("TemplateVersion is the version for the given template.")
  templateVersion: string;
}
`typespec``

***Example***

`rad deploy app.bicep`

Contents of `app.bicep`

`bicep``
import radius as rad

resource env 'Applications.Core/environments@2023-10-01-preview' = {
  name: 'default'
  properties: {
    compute: {
      kind: 'kubernetes'
      namespace: 'default'
    }
    recipes: {
      'Applications.Datastores/redisCaches': {
        'default': {
          templateKind: 'bicep'
          templatePath: 'radius.azurecr.io/recipes/local-dev/rediscaches'
        }
      }
    }
  }
}

resource app 'Applications.Core/applications@2023-10-01-preview' = {
  name: 'myapp'
}

resource redis 'Applications.Datastores/redisCaches@2023-10-01-preview' = {
  name: 'redis'
  properties: {
    environment: env.id
    application: app.id
  }
}
`bicep``

`rad resource show rediscaches redis -o json`

`json``
{
  "id": "/planes/radius/local/resourcegroups/default/providers/Applications.Datastores/redisCaches/redis",
  "location": "global",
  "name": "redis",
  "properties": {
    "application": "/planes/radius/local/resourceGroups/default/providers/applications.core/applications/demo",
    "environment": "/planes/radius/local/resourceGroups/default/providers/applications.core/environments/default",
    "host": "redis-hjo6ha3uqagio.default-demo.svc.cluster.local",
    "port": 6379,
    "provisioningState": "Succeeded",
    "recipe": {
      "name": "default"
    },
    "resourceProvisioning": "recipe",
    "status": {
      "recipe": {
        "templateKind": "bicep",
        "templatePath": "radius.azurecr.io/recipes/local-dev/rediscaches:latest"
      },
      "outputResources": [
        {
          "id": "/planes/kubernetes/local/namespaces/default-demo/providers/core/Service/redis-hjo6ha3uqagio",
          "radiusManaged": true
        },
        {
          "id": "/planes/kubernetes/local/namespaces/default-demo/providers/apps/Deployment/redis-hjo6ha3uqagio",
          "radiusManaged": true
        }
      ]
    },
    "tls": false,
    "username": ""
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
  "type": "Applications.Datastores/redisCaches"
}
`json``

## Alternatives considered

There is not a lot of alternatives to add the recipe information to the resource level. This is the most straightforward and easiest one.

## Test plan

1. Updating conversion unit tests to see if the recipe information is being populated on the resource level.
2. Updating some API unit tests to see if the recipe information is being populated on the resource level.

## Monitoring

N/A

## Development plan

1. There is an in-progress PR by Vinaya Damle and the link is <https://github.com/radius-project/radius/pull/6450>.

## Open issues

1. The name of the new map that is going to be added to the resource level. Should it be RecipeInformation, RecipeData, or just Recipe? Open to suggestions.
