# Adding support for Private Bicep Registries

* **Status**: Pending/Approved
* **Author**: Vishwanath Hiremath (@vishwahiremat)

## Overview

Today, infrastructure recipes are restricted to being stored on publicly accessible OCI-compliant registries. This limitation means that Radius does not support the use of privately-hosted Bicep recipes, nor does it provide a mechanism to configure authentication for accessing these private registries. This gap is a significant hindrance for organizations that develop their own infrastructure modules and prefer to store them in secure, privately-accessible repositories.

To address this issue and enhance the usability of Radius for serious enterprise use cases, we propose adding support for private Bicep registries. This feature will enable Radius to authenticate and interact with private OCI-compliant registries, ensuring secure access to proprietary bicep recipes. 

## Terms and definitions

| Term     | Definition                                                                                                                                                                                                 |
| -------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| OCI compliant registry | An OCI-compliant registry refers to a container registry that adheres to the specifications set by the Open Container Initiative  |
| ORAS | ORAS is the de facto tool for working with OCI Artifacts.(https://oras.land/) |
| Private Bicep Registry | A private Bicep registry is a dedicated repository for storing and managing Bicep modules which is not accessible publicly need to authenticate into it for registry operations.| 

## Objectives
> **Issue Reference:** https://github.com/radius-project/radius/issues/6917

### Goals
- Add support to publish bicep recipes to a private bicep registry.
- Enable support to use bicep recipes stored in private bicep registries.
- Support all OCI compliant private registries.

### Non goals

- Workload identity based authentication(Will be covered as part of different design) 
- Configure private OCI registry credentials in Radius via CLI and API.

### User scenarios (optional)
As an operator I am responsible for maintaining Bicep recipes for use with Radius. Bicep recipes used contains sensitive information, proprietary configurations, or data that should not be shared publicly and its intended for internal use within our organization and have granular control over who can access, contribute to, or modify bicep recipes. So I store the recipes in a private bicep registry. And I would like to use these private registries as template-path while registering a bicep recipe.


## User Experience (if applicable)
**Sample Input:**
Command to publish recipe to private bicep registry.
```diff
-rad bicep publish --file ./redis-test.bicep --target br:ghcr.io/myregistry/redis-test:v1

+rad bicep publish --file ./redis-test.bicep --target br:ghcr.io/myregistry/redis-test:v1 --username <user> --password <password>
```

**Sample Output:**

No change in the output


## Design
### Design details
Currently, OCI-compliant registries are used to store Bicep recipes, with the ORAS client facilitating operations like publish and pull these recipes from the registries. ORAS provides package auth, enabling secure client authentication to remote registries. Private Registry credentials information i.e username and password must be stored in a Application.Core/secretStores resource and the secret resource ID is added to the recipe configuration.

During the recipe deployment bicep driver checks for secrets information for that container registry the recipe deploying is stored on,  and creates an oras auth client to authenticate the private bicep registries.


e.g: creating a secret store with keys "username" and "password".

```
resource secretStore 'Applications.Core/secretStores@2023-10-01-preview' = {
  name: 'acrsecrets'
  properties:{
    type: 'generic'
    resource: 'registry-secrets/acr'
    data: {
      'username': {
        value: '<username>'
      }
      'password': {
        value: '<registry-password>'
      }
    }
  }
}
```
Update the recipe config in the environment to point to this secret resource id.
```
"recipeConfig": {
  "terraform": {
    ...
  },
  "bicep": {
    "authentication":{
      "test.azurecr.io":{
        "secret": "/planes/radius/local/resourcegroups/default/providers/Applications.Core/secretStores/acrsecrets"
      },
    }
  }
  "env": {
    ...
  }
},
```

#### Different container registry providers provide different authenticate methods to login.

***Azure***:

Azure provides multiple ways to authenticate ACR using ORAS, we can use different types of credentials depending on the setup and preferences. Here are the main types of credentials to use:

#### Admin User Credentials:
Users can enable the admin user and obtain these credentials from the Azure portal:
```
Username : The ACR admin username.
Password : The ACR admin password.
```

#### Service Principal:
```
Username: The service principal's client ID.
Password: The service principal's client secret.
```

#### Azure CLI Token:
```
Username: Use '00000000-0000-0000-0000-000000000000' (a placeholder indicating token-based authentication).
Password: Use the token obtained from az acr login.
```

You can obtain the token using the Azure CLI:
```
az acr login --name <registry-name> --expose-token
```
***Github***:
Github uses personal access token as password to auth into ghcr.
```
Username: Github username.
Password: Personal access token.
```

***AWS***:
```
Username: AWS
Password: output of `aws ecr get-login-password --region region`
```

### API design (if applicable)
***Model changes***

environment.tsp
```diff
@doc("Configuration for Recipes. Defines how each type of Recipe should be configured and run.")
model RecipeConfigProperties {
  @doc("Configuration for Terraform Recipes. Controls how Terraform plans and applies templates as part of Recipe deployment.")
  terraform?: TerraformConfigProperties;

+  @doc(Configuration for Bicep Recipes. Controls how Bicep plans and applies templates as part of Recipe deployment.)
+  bicep?: BicepConfigProperties;

  @doc("Environment variables injected during Terraform Recipe execution for the recipes in the environment.")
  env?: EnvironmentVariables;
}

+@doc(Configuration for Bicep Recipes. Controls how Bicep plans and applies templates as part of Recipe deployment.)
+model BicepConfigProperties {
+  @doc("Authentication information used to access private bicep registries, which is a map of registry hostname to secret config that contains credential information.")
+  authentication?: Record<SecretConfig>;
}

@doc("Personal Access Token (PAT) configuration used to authenticate to Git platforms.")
model SecretConfig {
  @doc("Secret represents the resource ID of the secret store that has credential information.")
  secret?: string;
}

```
***Bicep Example***

environment.bicep
```
"recipeConfig": {
  "terraform": {
    ...
  },
  "bicep": {
    "authentication":{
      "test.azurecr.io":{
        "secret": "/planes/radius/local/resourcegroups/default/providers/Applications.Core/secretStores/acr-secret"
      },
      "123456789012.dkr.ecr.us-west-2.amazonaws.com":{
        "secret": "/planes/radius/local/resourcegroups/default/providers/Applications.Core/secretStores/aws-secret"
      },
      "ghcr.io":{
        "secret": "/planes/radius/local/resourcegroups/default/providers/Applications.Core/secretStores/ghcr-secret"
      },
    }
  }
  "env": {
    ...
  }
},
```

### CLI Design

`rad bicep publish` need to be updated to ask the user to provide credential information(i.e username and password) for the private bicep registry user is trying to publish the recipe to.

```
rad bicep publish --file ./redis-test.bicep --target br:ghcr.io/myregistry/redis-test:v1 --username <user> --password <password>
```


### Implementation Details
#### Portable Resources / Recipes RP (if applicable)
***Bicep Driver***

- Implement FindSecretIDs() function in bicep driver, to retrieve the secret store resource IDs associated with private bicep registries.
    ```
    // FindSecretIDs is used to retrieve a map of secretStoreIDs and corresponding secret keys.
    // associated with the given environment configuration and environment definition.
    func (d *bicepDriver) FindSecretIDs(ctx context.Context, envConfig recipes.Configuration, definition recipes.EnvironmentDefinition) (secretStoreIDResourceKeys map[string][]string, err error) {
        ...
    }
    ```
- Update Bicep driver apis i.e execute, delete and getMetadata to use oras auth package for private registry authentication.

### Error Handling
<!--
Describe the error scenarios that may occur and the corresponding recovery/error handling and user experience.
-->

## Test plan

#### Unit Tests
-   Update environment conversion unit tests to validate recipeConfig property.
-   Update environment controller unit tests to add recipe config.
-   Adding new unit tests in bicep driver validating recipe config changes and retrieving secrets.

#### Functional tests
- Add e2e test to verify recipe deployment using a bicep stored in a private bicep registry.
    - publish a recipe to private ghcr.io/radius-project/private-recipes/<recipe> 
    - Deploy the recipe as part of the functional test using github app token to authenticate ghcr. 


## Development plan

- Task 1:  
    - Adding a new property bicepConfig in recipeConfig to environment and making necessary changes to typespec, datamodel and conversions.
    - Updating unit tests.
- Task 2:
    - Adding changes to bicep driver to retrieve secrets.
    - Adding changes to bicep driver to use oras auth package to authenticate private bicep registries
    - Update/Add unit tests
- Task 3:
    - Adding cli changes for `rad bicep publish` to support private bicep registries.
    - Update the cli unit tests.
- Task 3:
    - Manual Validation and adding e2e tests to verify using private bicep registries


## Design Review Notes

<!--
Update this section with the decisions made during the design review meeting. This should be updated before the design is merged.
-->