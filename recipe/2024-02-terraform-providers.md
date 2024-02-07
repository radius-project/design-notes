# Title

* **Status**: Pending
* **Author**: @lakshmimsft

## Overview

As part of Terraform support in Radius, we currently support azurerm, kubernetes and aws providers when creating recipes. We pull credentials stored in UCP and setup and save provider configuration which is accessible to recipes.\
This document  describes a proposal to support multiple providers, setup their provider configuration, which would be accessible to all recipes within an  environment.

## Terms and definitions

| Term     | Definition                                                                                                                                                                                                 |
| -------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Terraform Provider | A Terraform provider is a plugin that allows Terraform to interact with and manage resources of a specific infrastructure platform or service, such as AWS, Azure, or Google Cloud. |


## Objectives

**Reference for new type recipeConfig and handling of secrets:** [Design document for Private Terraform Repository](https://github.com/radius-project/design-notes/blob/3644b754152edc97e641f537b20cf3d87a386c43/recipe/2024-01-support-private-terraform-repository.md)

> **Issue Reference:** <!-- (If appropriate) Reference an existing issue that describes the feature or bug. -->
https://github.com/radius-project/radius/issues/6539

### Goals

1. Enable users to use terraform recipes with multiple providers (including and outside of Azure, Kubernetes, AWS).
2. Enable users to use terraform recipes with multiple configurations for the same provider using *alias*. 
3. Enable provider configurations to be available for all recipies used in an environment.


### Non goals
1. Updates to Bicep Provider configuration.
2. Authentication for providers hosted in private registries.
3. Support for .terraformrc files.

### User scenarios (optional)

#### User story 1
As an operator, I maintain a set of Terraform recipes for use with Radius. I have a set of provider configurations which are applicable across multiple recipes and I would like to configure them in a single centralized location for ease of maintenance.


#### User story 2
As an operator, I would like to manage cloud resources in different geographical regions. To enable this, I need to set up multiple configurations for the same provider for different regions and use an alias to refer to individual provider configurations. 

## Design
### Design details

Reviewing popular provider configurations (GCP, Oracle, Heroku, DigitalOcean, Docker etc.) a large number of provider configurations can be setup by handling a combination of key value pairs, key value pairs in nested objects, secrets and environment variables.

### Key-value pairs
Key-value pairs will be parsed and saved to a Terraform configuration file.

### Secrets
Secrets will be handled similarly to the approach described in document [Design document for Private Terraform Repository](https://github.com/radius-project/design-notes/blob/3644b754152edc97e641f537b20cf3d87a386c43/recipe/2024-01-support-private-terraform-repository.md) wherein Applications.Core/secretStores can point to an existing K8s secret.

The system will call the ListSecrets() api in Applications.Core namespace, retrieve contents of the secret and build the Terraform provider configuration.

### Environment variables
In a significant number of providers, as per documentation, environment variables are used one of the methods of saving sensitive credential data along with insensitive data. We allow the users to set environment variables for provider configuration. For sensitive information, we recommend the users save these values as secrets and point to them inside the env block.

```
...
 recipeConfig: {
    terraform: {
      ...
     providers: [...]
     env: {
       key: value
       secrets: {
        source: secretStoreOCICred.id
      }
     }
```

Environment variables apply to all providers configured in the environment. The system cannot set two separate values for the same environment variable for multiple provider configurations. In such cases, per provider documentation, there may be alternatives that users can avail (eg. For GCP, users can set  *credentials* field inside each instance of provider config as opposed to using env variable GOOGLE_APPLICATION_CREDENTIALS).


The system will allow for ability to set up multiple configurations for the same provider using the keyword *alias*. Validation for configuration 'correctness' will be handled by Terraform with calls to terraform init and terraform apply.

```
...
 recipeConfig: {
    terraform: {
      ...
      providers: ['azurerm': {  
          subscriptionid: 1234
          secrets: {
            source: secretStoreAz.id
          }
        },
        'azurerm': {
        subscriptionid: 1234
        alias: 'az-paymentservice'
        secrets: {
          source: secretStoreAzPayment.id
        }         
      }]
...       
```
Configuration for providers as described in this document will take precedence over provider credentials stored in UCP (currently these include azurerm, aws, kubernetes providers). So, for eg. In the scenario where credentials for Azure are saved with UCP during Radius install and a Terraform recipe created by an operator declares 'azurerm' under the the *required_providers* block; If there exists a provider configuration under the *providers* block under *recipeConfig*, these would take precedence and used to build the Terraform configuration file instead of Azure credentials stored in UCP. 


Example :
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
+      providers: [azurerm: {        
+          subscriptionid: 1234
+          secrets: {
+            source: secretStoreAz.id
+          }
+        },
+        'azurerm': {
+         subscriptionid: 1234
+         alias: 'az-paymentservice'
+         secrets: {
+           source: secretStoreAzPayment.id
+         }         
+       },
+       'gcp': {
+         project: 1234
+         regions: ['us-east1', 'us-west1']
+       },
+       'oraclepass': {
+         database_endpoint = "..."
+         java_endpoint     = "..."
+         mysql_endpoint    = "..."
+       }]
+     env: {
+       key: value
+       secrets: {
+        source: secretStoreOCICred.id
+      }
+     }
    }
  }
    recipes: {      
      ...
    }
  }
}
```


Limitations: Customers may store sensitive data in other formats which may not be supported. eg. sensitive data is stored on files, which customers will not currently be able to load on disk in applications-rp where tf init/apply commands are run.
Containerization work may alleviate this limitations. Further design work will be needed for towards this which is planned for the near future. 

### API design (if applicable) (TBD)

TBD

## Alternatives considered

Mentioned under Limitations above, work to containerize running terraform jobs was considered as a precursor to this work. The time sensitivity for unblocking customers on ability to configure and use providers they use today was given a priority and current design held. Containerization will be taken up as a parallel effort in the near future.

## Test plan (TBD)

#### Unit Tests (TBD)
-   Update environment conversion unit tests to validate providers, env type under recipeConfig.
-   Update environment controller unit tests for providers, env.

#### Functional tests
- Add e2e test to verify recipe using multiple provider configuration is deployed.
- (TBD) Discuss/List providers we want to test with.

## Security

Largely following security considerations and secret handling described in design for private terraform modules: [Design document for Private Terraform Repository](https://github.com/radius-project/design-notes/blob/3644b754152edc97e641f537b20cf3d87a386c43/recipe/2024-01-support-private-terraform-repository.md)
The work done here will be to read, decipher values and secrets and environment variables set by user in the *providers* block, to build internal terraform configuration file used by tf setenv, init and apply commands. 

## Monitoring

## Development plan (TBD)

## Open issues/questions
Do we consider first class support for GCP and other popular providers or continue with a generic approach??
