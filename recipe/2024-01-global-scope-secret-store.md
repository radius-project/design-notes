# Adding support to extend the capabilities of secretStores to a global scope.

* **Status**: Pending
* **Author**: Vishwanath Hiremath (@vishwahiremat)

## Overview

Today we have Application.Core/secretStores resource to securely manage secrets for the Application. However a gap exists if user wants to create a secret before application or environment is created. To address this limitation, we need to enhance the capabilities of the existing secretStores resource. The objective is to extend its support to a global scope, enabling users to store secrets prior to the creation of the application or environment.


## Terms and definitions
| Term     | Definition                                                                                                                                                                                                 |
| -------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Global scope | The ability of the secretStores resource to operate universally, allowing users to manage secrets independently of a particular application or environment. |
| secretStores | Resource used to securely manage secrets for Environment and Application. |
| Private Terraform Repository | A private Terraform repository typically refers to a version control repository that contains Terraform module code, but is not publicly accessible. | 

## Objectives

> **Issue Reference:** https://github.com/radius-project/radius/issues/7030

### Goals

- Enable support to extend the capabilities of secretStores to a global scope.

### Non goals


### User scenarios (optional)

#### User story 1
As a operator I define and curate the set of terraform recipes that developers in my organization rely on. And these terraform modules are stored in a private module sources (e.g private git repository). I would need to securely store the credential information or use the existing secret (e.g kubernetes secret) with credential information. So it can be used while creating the env and registering recipes.

```diff
resource env 'Applications.Core/environments@2023-10-01-preview' = {
  name: 'dsrp-resources-env-recipes-context-env'
  location: 'global'
  properties: {
    compute: {
      ...
    }
    providers: {
      ...
    }
   recipeConfig: {
      terraform: {
        authentication:{
          git:{
            pat: {
              "dev.azure.com": {
+               secret: secretStoreAzure.id
              }
              "github.com": {
+                secret: secretStoreGithub.id
              }
            }
          }
        }
     }
   }
    recipes: {      
      'Applications.Datastores/mongoDatabases':{
        default: {
          templateKind: 'terraform'
          templatePath: 'https://dev.azure.com/test-private-repo'
        }
      }
    }
  }
}
```


## Design
Today we use Application.Core/secretstores resource to store secrets/credentials information for an application but we cannot use it in this scenario as secretStores is application scoped and expects to have an application created before we create a secretStore.

And we cannot have secretStore created for environment scope in this scenario as we are adding the secret to the environment which creates cyclic dependency between environment and secretStore resource.

#### Application.Core/secretStores as a global scoped resource.
We could make secretStores as a global scoped resource by removing application as a required property.

```diff
"SecretStoreProperties": {
      "type": "object",
      "description": "The properties of SecretStore",
      "properties": {
        "environment": {
          ...
        },
        "application": {
          ...
        },
        "provisioningState": {
          ...
        },
        "status": {
          ...
        },
        "type": {
          ...
        },
        "data": {
          "type": "object",
          "description": "An object to represent key-value type secrets",
          "additionalProperties": {
            "$ref": "#/definitions/SecretValueProperties"
          }
        },
        "resource": {
          "type": "string",
          "description": "The resource id of external secret store."
        }
      },
      "required": [
-        "application",
        "data"
      ]
    },
```
With this change we can create a secret store resource before an application or environment is created. But, if the user is creating a secretStore of kind kubernetes secret, it is deployed in application/environment namespace. But for secret resource with global scope use the `resource` property which is used to specify the secret ref of the existing secret to provide namespace and secret name details. User is expected to provide these details in `<namespace>/<secretName>` format for secretStore with global scope.

```diff
resource secretStore 'Applications.Core/secretStores@2023-10-01-preview' = {
  name: 'github'
  properties:{
-  	app: app.id
+   resource: <namespace>/<secretName>
    type: 'generic'
    data: {
      'pat': {
        value: '<personal-access-token>'
      }
      'username': {
        value: '<username>'
      }
    }
  }
}
```

### API design (if applicable)

#### Typespec changes

radius/typespec/radius/v1/resources.tsp
```diff
+@doc("Base properties of a Global-scoped resource")
+model GlobalScopedResource {
+  @doc("Fully qualified resource ID for the environment that the application is linked to")
+  environment?: string;

+  @doc("Fully qualified resource ID for the application")
+  application?: string;

+  @doc("The status of the asynchronous operation.")
+  @visibility("read")
+  provisioningState?: ProvisioningState;

+  @doc("Status of a resource.")
+  @visibility("read")
+  status?: ResourceStatus;
+}
```

typespec/Applications.Core/secretstores.tsp

```diff
@doc("The properties of SecretStore")
model SecretStoreProperties {
-  ...ApplicationScopedResource;
+  ...GlobalScopedResource;

  #suppress "@azure-tools/typespec-azure-resource-manager/arm-resource-duplicate-property"
  @doc("The type of secret store data")
  type?: SecretStoreDataType = SecretStoreDataType.generic;

  @doc("An object to represent key-value type secrets")
  data: Record<SecretValueProperties>;

  @doc("The resource id of external secret store.")
  resource?: string;
}
```

## Alternatives considered

#### Adding default a namespace for global scoped secretstore resource.
Add a default namespace `global-secretStores` to store the global scoped secretStores.

```diff
resource secretStore 'Applications.Core/secretStores@2023-10-01-preview' = {
  name: 'github'
  properties:{
-  	app: app.id
    type: 'generic'
    data: {
      'pat': {
        value: '<personal-access-token>'
      }
      'username': {
        value: '<username>'
      }
    }
  }
}
```


#### Adding a new property `namespace` 

Add a new property `namespace` for kubernetes secret type.
```diff
resource secretStore 'Applications.Core/secretStores@2023-10-01-preview' = {
  name: 'github'
  properties:{
-  	app: app.id
    type: 'generic'
+   namespace: <namespace>
    data: {
      'pat': {
        value: '<personal-access-token>'
      }
      'username': {
        value: '<username>'
      }
    }
  }
}
```
for existing kubernetes secret
```diff
resource secretStore 'Applications.Core/secretStores@2023-10-01-preview' = {
  name: 'github'
  properties:{
-  	app: app.id
    type: 'generic'
+   namespace: <namespace>
+   resource: <secretName>
    data: {
      'pat': {}
      'username': {}
    }
  }
}
```

## Test plan

Unit tests:
- Update and add unit tests for changes in conversions for secret store.

Functional Tests:
- Functional test for private repository support takes care of this scenario.

## Development plan

Tasks:
- Adding typespec changes and conversions to secretstores resource.
- Adding unit tests for conversions.
- Updating secretStores frontend controller to support global scope.

