# Adding support for Terraform Modules from Private Git Repository

* **Status**: Pending
* **Author**: Vishwanath Hiremath (@vishwahiremat)

## Overview

Today, radius can work with publicly-hosted Terraform modules across a few different module sources. We don't support working with privately-hosted Terraform modules, and there's no way to configure authentication.

This is important because organizations write their own Terraform modules and store them in privately-accessible sources. In order for us to support serious use, we need to enable private sources.

## Terms and definitions
| Term     | Definition                                                                                                                                                                                                 |
| -------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Terraform module source | The source argument in a module block tells Terraform where to find the source code for the desired child module |
| Module registry | A module registry is the native way of distributing Terraform modules for use across multiple configurations |
| Terraform registry | Terraform Registry is an index of modules shared publicly |
| HTTP URLs | When you use an HTTP or HTTPS URL, Terraform will make a GET request to the given URL, which can return another source address | 
| Private Terraform Repository | A private Terraform repository typically refers to a version control repository that contains Terraform module code, but is not publicly accessible. | 


## Objectives

> **Issue Reference:** https://github.com/radius-project/radius/issues/6911

### Goals
- Enable support to use terraform modules from private Git repositories.
- It should support git from any platforms(like Bitbucket, Gitlab, Azure DevOps Git etc.)

### Non goals
- To Support other terraform module sources like S3, GCP, Mercurial repository.
- Registering Terraform modules stored in repositories from different git accounts.

### User scenarios (optional)

As an operator I am responsible for maintaining Terraform recipes for use with Radius. Terraform modules used contains sensitive information, proprietary configurations, or data that should not be shared publicly and its intended for internal use within our organization and have granular control over who can access, contribute to, or modify Terraform modules. So I store the terraform modules in a private git repository. And I would like to use these github sources from a private repository as template-path while registering a terraform recipe.

## Design
### Design details
Today, we support only Terraform registry and HTTP URLs as allowed module sources for Terraform recipe template paths. Git public repository are also supported by providing the HTTP URl to the module directory. But to support repositories we need a way to authenticate the git account where the modules are stored.

Git provides different ways to authenticate:

#### Personal Access Token:
Users need to a Git personal access token with very limited access (just read access) and also specify token validity. And this is used along with the username to clone the terraform module repository through HTTPS as part of `terraform init`.

#### SSH key:
SSH key can be used to provide access to private git repository. But it requires adding the generated ssh key to the users account.

#### Service Principle(Only for Azure DevOps Git):
We could use the Azure Service Principle details used for azure score to authenticate Azure DevOps Git. But most often users have diff tenant IDs for the production environment and for git.

Personal Access Token with Username way of authentication is supported by git from most of the platforms(e.g Bitbucket,Gitlab,Github etc) and generic git repository URL format as template path can be used to while registering recipe to access the terraform modules from private git repository from any platform.

Generic Git Repository URL format:
```diff
resource env 'Applications.Core/environments@2023-10-01-preview' = {
  name: 'corerp-resources-recipe-env'
  location: location
  properties: {
    ...,
    recipes: {
      'Applications.Datastores/mongoDatabases':{
        recipe1: {
          templateKind: 'terraform'
+          templatePath: "git::https://{username}:{PERSONAL_ACCESS_TOKEN}@example.com.com/test-private-repo.git"
        }
      }
    }
  }
}
```
this takes latest as the default version if not provided. To specify a particular version the URL should be appended with `ref=<version>"`
E.g:
```
templatePath: "git::https://{username}:{PERSONAL_ACCESS_TOKEN}@example.com.com/test-private-repo.git?ref=v1.2.0"
```

Since adding sensitive information like tokens to the terraform configuration files which may be stored in version control can pose security issues. So use different ways to store the github credentials.

Option 1 : Using Git Credential Store

Git needs to know when and where to use the token when checking out code from a private repository. We want git to automatically detect when Terraform modules are being loaded from a private repository and insert the token for the duration of the session. This can be done by running the below command.
```
git config --global url."https://<username>:<personal-access-token>@dev.azure.com.com".insteadOf https://dev.azure.com
```
or by adding this entry on the .gitconfig file on the cluster.
```
[url "https://<username>:<personal-access-token>@dev.azure.com.com""]
	insteadOf = https://dev.azure.com
```
But it wont work well when we have multiple environments. As global .gitconfig file has the git credentials stores for all the environments and it may be from different accounts.

Option 2: Saving it as part of the environment resource.

Add a new property `recipeConfig` write-only to store the information about the git credentials, and have a custom action `getRecipeConfiglike` (like `list-secrets`) to get these details. 

```diff
+@secure()
+param username string
+@secure()
+param token string
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
+   recipeConfig: {
+     terraform: {
+       secrets: {
+         "dev.azure.com": {
+            username : username
+            pat: token
+          }
+       }
+     }
+   }
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

But the majority of users frequently update their git personal access tokens (like once a day), and which means updating the environment resource every time the token is updated.


Option 4 : Using Applications.Core/secretStores

Use secretStore to store the username and personal access token and add the secret to the new property recipeConfig in the environment.

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
+   recipeConfig: {
+     terraform: {
+       secrets: {
+         "dev.azure.com": {
+            secret: secretStore.id
+          }
+       }
+     }
+   }
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
But todays secret store implementation is tied to application scope and in this case, secretStore needs to be created before application and environment creation. So we can only use the secretStore resource if we extend it to have global scope(extending to environment scope would create cyclic dependency between environment and secret store).

Option 4 : Using kubernetes secret

Using existing kubernetes secret i.e asking the users to have the git credentials already stored in the kubernetes secret on the same cluster and use it in the recipeConfig.

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
+   recipeConfig: {
+     terraform: {
+       secrets: {
+         "dev.azure.com": {
+            secret: secretStore.id
+            namespace: <namespace>
+          }
+       }
+     }
+   }
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

### API design (if applicable)

## Test plan

## Development plan

## Open issues

