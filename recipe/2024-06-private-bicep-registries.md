# Adding support for Private Bicep Registries

* **Status**: Pending
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
- Enable support to use bicep recipes stored in private bicep registries.
- Support all OCI compliant private registries.

### Non goals

- Federated authentication(Azure workload identity/ AWS IRSA) for ACR and ECR.(will be part of initial feature release, its in investigation phase) 
- Support to manage(configure/view) OCI registry credentials in Radius environment via CLI. (this is a future priority scenario and would not be in scope for the initial release)

### User scenarios (optional)
As an operator I am responsible for maintaining Bicep recipes for use with Radius. Bicep recipes used contains proprietary configurations, or data that should not be shared publicly and its intended for internal use within our organization and have granular control over who can access, contribute to, or modify bicep recipes. So I store the recipes in a private bicep registry. And I would like to use these private registries as template-path while registering a bicep recipe.


## User Experience (if applicable)
Credential information is stored in `Application.Core/secretstore` resource, users are expected to provide `username` and `password` keys with the appropriate values in the secret store that will be used to authenticate private registry.

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
`bicep` recipe config is added in the environment to stores bicep recipe configurations, update it to point to the `secretStore` with credential information.
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


## Design
![Alt text](./2024-06-private-bicep-registries/overview.png)

At a high level, data flow between radius components:
- Engine calls bicep driver to get secret id of the resource that has credential(username/password) information.
- Engine gets the secret values from the secret loader and calls driver to execute recipe deployment.
- Bicep driver uses ORAS auth client to authenticate the private container registry and fetches the bicep file contents and calls UCP to deploy recipe.



### Design details
Currently, OCI-compliant registries are used to store Bicep recipes, with the ORAS client facilitating operations like publish and pull these recipes from the registries. ORAS provides package auth, enabling secure client authentication to remote registries. Private Registry credentials information i.e username and password must be stored in an `Application.Core/secretStores` resource and the secret resource ID is added to the recipe configuration.

During the recipe deployment bicep driver checks for secrets information for that container registry the recipe deploying is stored on,  and creates an oras auth client to authenticate the private bicep registries.

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
While there are various container registry providers, each offering multiple authentication methods, by leveraging ORAS, we abstract away the need for provider-specific code to handle authentication. 

ORAS auth client example to authenticate private registry using username and password,
```
	repo.Client = &auth.Client{
		Client: retry.DefaultClient,
		Credential: auth.StaticCredential("orasregistry.azurecr.io", auth.Credential{
			Username: "<username>",
			Password: "<password>",
		}),
	}
	repo.Client = client
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
+
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

- Authorization failure error:

  This error can occur if users provide incorrect credential information or omit the Bicep recipe configuration for their private registry, we can add a new error code for this scenario `BicepRegistryAuthFailed`
  
  Incorrect credential information:
  ```
  NewRecipeError("BicepRegistryAuthFailed", fmt.Sprintf("could not authenticate to the bicep registry %s, the credentials provided for username '%s' are incorrect: %s", <private-oras-registry-name>, <username> <error returned by the oras client>))
  ```
  Missing Bicep Recipe Config:
  ```
  NewRecipeError("BicepRegistryAuthFailed", fmt.Sprintf("could not authenticate to the bicep registry %s, missing credentials. : %s", <private-oras-registry-name>, <error returned by the oras client>))
  ```

## Test plan

#### Unit Tests
-   Update environment conversion unit tests to validate recipeConfig property.
-   Update environment controller unit tests to add recipe config.
-   Adding new unit tests in bicep driver validating recipe config changes and retrieving secrets.

#### Functional tests
- Add e2e test to verify recipe deployment using a bicep stored in a private bicep registry.
    - publish a recipe to private ghcr.io/radius-project/private-recipes/<recipe> 
    - Deploy the recipe as part of the functional test using github app token to authenticate ghcr. 

## Security
With this design we enable username-password based authentication for OCI compliant registries, we let the users manage secrets. For secret rotation users need to re deploy the `Applications.Core/secretStores` resource with updated credentials.

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
- Task 4:
    - Adding developer documentation for private bicep registry support feature.

## Design Review Notes
- Maintain the current design for the `rad bicep publish` command as it is, where users takes care of logging into private registry and radius uses the dockerfile based authentication to publish recipes to private registries.