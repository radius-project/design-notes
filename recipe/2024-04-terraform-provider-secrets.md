# Title

* **Status**: Pending
* **Author**: @lakshmimsft

## Overview

We've described the design for support of multiple Terraform Providers in Radius in the following PR: [Design Document to support multiple Terraform Providers](https://github.com/radius-project/design-notes/pull/39/files).\
This document  describes in more detail the handling of secrets data between the engine and driver for teraform providers.

## Terms and definitions

| Term     | Definition                                                                                                                                                                                                 |
| -------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Terraform Provider | A Terraform provider is a plugin that allows Terraform to interact with and manage resources of a specific infrastructure platform or service, such as AWS, Azure, or Google Cloud. |



## Objectives

**Reference for new type recipeConfig and handling of secrets:** [Design document for Private Terraform Repository](https://github.com/radius-project/design-notes/blob/3644b754152edc97e641f537b20cf3d87a386c43/recipe/2024-01-support-private-terraform-repository.md)

**Reference for support of multiple Terraform Providers in Radius:** [Design Document to support multiple Terraform Providers](https://github.com/radius-project/design-notes/pull/39/files)

> **Issue Reference:** <!-- (If appropriate) Reference an existing issue that describes the feature or bug. -->
https://github.com/radius-project/radius/issues/6539

### Goals

1. Describe data flow when using secrets to configure Terraform Providers building on design in earlier design document.


### Non goals


### User scenarios (optional)

#### User story 1
As an operator, I maintain a set of Terraform recipes for use with Radius. I have a set of provider configurations which are applicable across multiple recipes and I would like to include passing in secrets in these providers configurations.

## Design
Secrets handling for Terraform providers will expand on current implementation  the approach described in PR [Private Terraform module-Secrets](https://github.com/radius-project/radius/pull/7306) and document [Design document for Private Terraform Repository](https://github.com/radius-project/design-notes/blob/3644b754152edc97e641f537b20cf3d87a386c43/recipe/2024-01-support-private-terraform-repository.md) where Applications.Core/secretStores can point to an existing K8s secret.

### Design details

The Terraform driver implements the optional interface DriverWithSecrets which has method FindSecretIDs.


The system will call the ListSecrets() api in Applications.Core namespace, retrieve contents of the secret and build the Terraform provider configuration.


Secrets {
moduleSecrets <ListSecretsResponse>
providerSecrets map[string]map[string]string
envSecrets map[string]string
}

The system will allow for ability to set up multiple configurations for the same provider using the keyword *alias*. Validation for configuration 'correctness' will be handled by Terraform with calls to terraform init and terraform apply.

```
...
 recipeConfig: {
    terraform: {
      ...
      providers: [
      {
        name: 'azurerm',
        properties: {
          subscriptionid: 1234,
          secrets: {                  // Individual Secrets from SecretStore
            'my_secret_1': {
              source: secretStoreAz.id
              key: 'secret.one'
            }
            'my_secret_2': {
               source: secretStoreAzPayment.id
              key: 'secret.two'
            }
          }
        }
      },
      {
        name: 'azurerm',
        properties: {
          subscriptionid: 1234,
          alias: 'az-paymentservice'
       }
     }]
...       
```
Configuration for providers as described in this document will take precedence over provider credentials stored in UCP (currently these include azurerm, aws, kubernetes providers). So, for eg. In the scenario where credentials for Azure are saved with UCP during Radius install and a Terraform recipe created by an operator declares 'azurerm' under the the *required_providers* block; If there exists a provider configuration under the *providers* block under *recipeConfig*, these would take precedence and used to build the Terraform configuration file instead of Azure credentials stored in UCP. 


### Example Bicep Input :
**Option 1**: 

``` diff
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
          ...        
        }
+       providers: [
+         {
+            name: 'azurerm',
+            properties: {          
+             subscriptionid: 1234,
+             secrets: {                  // Individual Secrets from SecretStore
+               'my_secret_1': {
+                  source: secretStoreAz.id
+                  key: 'secret.one'
+                }
+               'my_secret_2': {
+                  source: secretStoreAzPayment.id
+                  key: 'secret.two'
+               }
+             }
+            }
+          },
+          {
+            name: 'azurerm',
+            properties: {
+              subscriptionid: 1234,
+              tenant_id: '745fg88bf-86f1-41af-'
+              alias: 'az-paymentservice',
+            }
+          },
+          {
+            name: 'gcp',
+            properties: {
+              project: 1234,
+              regions: ['us-east1', 'us-west1']
+            }
+          },
+          {
+            name: 'oraclepass',
+            properties: {
+              database_endpoint: "...",
+              java_endpoint: "...",
+              mysql_endpoint: "..."
+            }
+          }
+        ]
+      }
+      env: {
+        'MY_ENV_VAR_1': 'my_value'
+        secrets: {                       // Individual Secrets from SecretStore
+        'MY_ENV_VAR_2': {
+            source: secretStoreConfig.id
+            key: 'envsecret.one'
+          }
+        'MY_ENV_VAR_3': {
+            source: secretStoreConfig.id
+            key: 'envsecret.two'
+          }
+        }
+      }      
+    }
  }
  recipes: {      
    ...
  }
}
```
**Option 2**:
This option provides grouping and efficiency to retrieve all details for a single provider. 
``` diff
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
          ...        
        }
+       providers: {
+         'azurerm': [
+           {
+             subscriptionid: 1234,
+             secrets: {                  // Individual Secrets from SecretStore
+               'my_secret_1': {
+                  source: secretStoreAz.id
+                  key: 'secret.one'
+                }
+               'my_secret_2': {
+                  source: secretStoreAzPayment.id
+                  key: 'secret.two'
+               }
+             }
+          },
+          {
+             subscriptionid: 1234,
+             tenant_id: '745fg88bf-86f1-41af-'
+             alias: 'az-paymentservice', 
+          }]
+          'gcp': [
+            {
+              project: 1234,
+              regions: ['us-east1', 'us-west1']
+            }
+          ]
+          'oraclepass': [
+            {
+              database_endpoint: "...",
+              java_endpoint: "...",
+              mysql_endpoint: "..."
+            }
+          ]
+        }
+     }
+     env: {
+        'MY_ENV_VAR_1': 'my_value'
+        secrets: {                       // Individual Secrets from SecretStore
+        'MY_ENV_VAR_2': {
+            source: secretStoreConfig.id
+            key: 'envsecret.one'
+          }
+        'MY_ENV_VAR_3': {
+            source: secretStoreConfig.id
+            key: 'envsecret.two'
+          }
+        }
+      }   
+    }
  }
  recipes: {      
    ...
  }
}


```

**Option 3** - Environment Variables : With this option a question was raised if 'env' could be updated to 'envVariables'. We decided against it for consistency (with environment variables used in Applications.Core/Containers resource) and decided to keep the name as env as described in Option 2 above.

``` diff
resource env 'Applications.Core/environments@2023-10-01-preview' = {
  name: 'dsrp-resources-env-recipes-context-env'
  location: 'global'
  properties: {
  ...
  recipeConfig: {
      ... 
      terraform: {
         ...
         providers: [...]         // Same as Option 2
+     envVariables: {                     // ** Change from Option 2 **
+       'MY_ENV_VAR_1': 'my_value'
+        secrets: {                 // Individual Secrets from SecretStore
+           'MY_ENV_VAR_2': {
+              source: secretStoreConfig.id
+              key: 'envsecret.one'
+            }
+           'MY_ENV_VAR_3': {
+              source: secretStoreConfig.id
+              key: 'envsecret.two'
+           }
+        }
+      }   
+    }
  }
  recipes: {      
    ...
  }
}
```
**Option 4** - Environment Variables: With the structure described in Option 2 for environment variables, we are unable to use a extends Record\<string\> as  described in the API design section:

```
model EnvironmentVariables extends Record<string>{
  secrets?: Record<ProviderSecret>
}
```
and in order to continue to maintain type strictness for EnvironmentVariables, we discussed the following design amongst others and are going ahead with:

``` diff
resource env 'Applications.Core/environments@2023-10-01-preview' = {
  name: 'dsrp-resources-env-recipes-context-env'
  location: 'global'
  properties: {
  ...
    recipeConfig: {
      ... 
      terraform: {
         ...
         providers: [...]         // Same as Option 2
+     env: {                
+         'MY_ENV_VAR_1': 'my_value'
+     } 
+     envSecrets: {      // Individual Secrets from SecretStore
+         'MY_ENV_VAR_2': {
+            source: secretStoreConfig.id
+            key: 'envsecret.one'
+         }
+        'MY_ENV_VAR_3': {
+            source: secretStoreConfig.id
+            key: 'envsecret.two'
+         }
+     }
+  }   
+}
  recipes: {      
    ...
  }
}

```
Limitations: Customers may store sensitive data in other formats which may not be supported. eg. sensitive data is stored on files, which customers will not currently be able to load on disk in applications-rp where tf init/apply commands are run.
Containerization work may alleviate this limitations. Further design work will be needed for towards this which is planned for the near future. 

### API design

***Model changes providers***

### Option 1:
```
Addition of new property to TerraformConfigProperties in `recipeConfig` under environment properties.

model TerraformConfigProperties{
  @doc(Specifies authentication information needed to use private terraform module repositories.)  
  authentication?: AuthConfig
+ providers?: Array<ProviderConfig>
}

@doc("ProviderConfig specifies provider configurations needed for recipes")
model ProviderConfig {
 name: string
 properties: ProviderConfigProperties
}

@doc("ProviderConfigProperties specifies provider configuration details needed for recipes")
model ProviderConfigProperties extends Record<unknown> {
  @doc("The secrets for referenced resource")
  secrets?: Record<ProviderSecret>;
}
```
### Option 2:
```
Addition of new property to TerraformConfigProperties in `recipeConfig` under environment properties.

model TerraformConfigProperties{
  @doc(Specifies authentication information needed to use private terraform module repositories.)  
  authentication?: AuthConfig
  providers?: Record<Array<ProviderConfigProperties>>
}

@doc("ProviderConfigProperties specifies provider configuration details needed for recipes")
model ProviderConfigProperties extends Record<unknown> {
  @doc("The secrets for referenced resource")
  secrets?: Record<ProviderSecret>;
}
```
***Model changes env***
```
Addition of new property to RecipeConfigProperties under environment properties.

@doc("Specifies recipe configurations needed for the recipes.")
model RecipeConfigProperties {
  @doc("Specifies the terraform config properties")
  terraform?: TerraformConfigProperties;
+ env?: EnvironmentVariables
}

@doc("EnvironmentVariables describes structure enabling environment variables to be set")
model EnvironmentVariables extends Record<string>{
  secrets?: Record<ProviderSecret>
}

@doc("Specifies the secret details")
model ProviderSecret {
  @doc("The resource id for the secret store containing credentials")
  source: string;
  key: string;
}

```

### Option 3:
Providers section is same as Option 2

***Model changes env***
```
Addition of new property to RecipeConfigProperties under environment properties.

@doc("Specifies recipe configurations needed for the recipes.")
model RecipeConfigProperties {
  @doc("Specifies the terraform config properties")
  terraform?: TerraformConfigProperties;
+ envVariables?: EnvironmentVariables
}

@doc("EnvironmentVariables describes structure enabling environment variables to be set")
model EnvironmentVariables extends Record<unknown>{
  secrets?: Record<RecipeSecret>
}

@doc("Specifies the secret details")
model RecipeSecret {
  @doc("The resource id for the secret store containing credentials")
  source: string;
  key: string;
}

```

### Option 4: Final Design

``` 
Addition of new property to TerraformConfigProperties in `recipeConfig` under environment properties.

model TerraformConfigProperties{
  @doc(Specifies authentication information needed to use private terraform module repositories.)  
  authentication?: AuthConfig
  providers?: Record<Array<ProviderConfigProperties>>
}

@doc("ProviderConfigProperties specifies provider configuration details needed for recipes")
model ProviderConfigProperties extends Record<unknown> {
  @doc("The secrets for referenced resource")
  secrets?: Record<SecretReference>;
}
```

***Model changes env***
``` 
Addition of new property to RecipeConfigProperties under environment properties.

@doc("Specifies recipe configurations needed for the recipes.")
model RecipeConfigProperties {
  @doc("Specifies the terraform config properties")
  terraform?: TerraformConfigProperties;
+ env?: EnvironmentVariables
+ envSecrets?: Record<SecretReference>
}

@doc("EnvironmentVariables describes structure enabling environment variables to be set")
model EnvironmentVariables extends Record<string>{}

@doc("Specifies the secret details")
model SecretReference {
  @doc("The resource id for the secret store containing credentials")
  source: string;
  key: string;
}

```
## Decision on Options above:
We initially decided to go ahead with Option 2 and discussed adding validation to check number of provider configurations to be a minimum of 1. Option 2 helps users keep track of all provider configurations for a provider in one place and lowers probability of, say, duplication of provider configurations if it is laid out in one list as in Option 1. Also, we can enforce some constraints on, say, minimum number of configurations for a provider. Option 2 is optimized for multiple provider configurations per provider and that may not apply for every provider configuration that users set up.

Notes: After initial work on implementation we encountered some questions and issues with EnvironmentVariables (listed in Options 3 and 4) above which we brought up in our design meetings and have updated the EnvironmentVariables API design and renamed ProviderSecret to SecretReference.
The latest API design is described in Options 4. 



## Alternatives considered

Mentioned under Limitations above, work to containerize running terraform jobs was considered as a precursor to this work. The time sensitivity for unblocking customers on ability to configure and use providers they use today was given a priority and current design held. Containerization will be taken up as a parallel effort in the near future.

## Test plan (In Progress)

#### Unit Tests
- Update  conversion unit tests to validate providers, env type and later secrets under recipeConfig.
- Update  controller unit tests for providers, env, secrets.
- Unit tests for functions related to building provider config with new provider data when module is loaded.
- Unit tests for secret retrieval and handling/building provider config.



#### Functional tests
- Add e2e test to verify multiple provider configuration is created as expected.
- (TBD) Discuss approach to validation of provider configuration: Is it possible to be static data and not a actual provider? Do we use provider like random?

## Security

Largely following security considerations and secret handling described in design for private terraform modules: [Design document for Private Terraform Repository](https://github.com/radius-project/design-notes/blob/3644b754152edc97e641f537b20cf3d87a386c43/recipe/2024-01-support-private-terraform-repository.md)
The work done here will be to read, decipher values and secrets and environment variables set by user in the *providers* block, to build internal terraform configuration file used by tf setenv, init and apply commands.

## Monitoring

## Development plan
We're breaking down the effort into smaller units in order to enable parallel work and faster delivery of this feature within a sprint.
The user stories created are as follows:
The numbers indicate sequential order of work that can be done. Having said that, work for numbers 1 and 2 are going ahead in parallel and we will resolve conflicts as PRs get merged.
1. Update Provider DataModel, TypeSpec, Convertor, json examples
1. Update Environment Variables DataModel, TypeSpec, Convertor, json examples
1. Documentation
2. Build Provider Config (minus secrets)
2. Process, update environment variables - minus secrets
3. Functional Tests
4. Update Secret DataModel, TypeSpec, Convertor, json examples
4. Secret processing - Providers + Environment Variables

The objective is to deliver within a sprint. We will create unit tests for each task mentioned above as it is completed and test functionality.
It will then be possible to deliver the feature in the following order:
1. Building provider configuration based on provider configuration block
2. Updating environment variables
3. Updating provider configuration and environment variables with processing secrets.

We've currently decided on Secret processing to be taken up following completion of building providing provider based on providers block and env variable block. The goal is to deliver as much functionality as possible within this sprint.

## High Level Design details:
Disclaimer: The following are early assessments and may change as we progress:
No changes are anticipated to payload in Driver and Engine.
The new types containing provider and environment variable data are contained within environment configuration which will be now passed into constructor of TerraformConfig as part of Private Terraform Module work. [link](https://github.com/radius-project/design-notes/blob/3644b754152edc97e641f537b20cf3d87a386c43/recipe/2024-01-support-private-terraform-repository.md) 
Large amount of work will be within pkg/terraform/config where we process new providers and env blocks and update current processing for building provider configuration.

## Open issues/questions
Do we consider first class support for GCP and other popular providers or continue with a generic approach??
Answer-We should start with a generic approach to unlock everything, and then add first-class support where users request it later on. That we support everything, and then we can add convenience later on.