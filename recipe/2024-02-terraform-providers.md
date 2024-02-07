# Title

* **Status**: Pending
* **Author**: @lakshmimsft

## Overview

<!--
Provide a succinct high-level description of the component or feature and 
where/how it fits in the big picture. The overview should be one to three 
paragraphs long and should be understandable by someone outside the Radius
team. Do not provide the design details in this, section - there is a
dedicated section for that later in the document.
-->
As part of Terraform support in Radius, we currently support azurerm, kubernetes and aws providers when creating recipes. We pull credentials stored in UCP and setup and save provider configuration which is accessible to recipes.\
This document  describes a proposal to support multiple providers, setup their provider configuration, which would be accessible to all recipes within an  environment.

## Terms and definitions

<!--
Include any terms, definitions, or acronyms that are used in
this design document to assist the reader. They may or may not
be part of the user-facing experience once implemented, and can
be specific to this design context.
-->
| Term     | Definition                                                                                                                                                                                                 |
| -------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Terraform Provider | A Terraform provider is a plugin that allows Terraform to interact with and manage resources of a specific infrastructure platform or service, such as AWS, Azure, or Google Cloud. |


## Objectives

<!--
Describe goals/non-goals and user-scenario of this feature to understand
the end-user goals.
* If the feature shares the same objectives of the existing design, link
  to the existing doc and section rather than repeat the same context.
* If the feature has a scenario, UX, or other product feature design doc,
  link it here and summarize the important parts.
-->
**Reference for new type recipeConfig and handling of secrets:** [Design document for Private Terraform Repository](https://github.com/radius-project/design-notes/blob/3644b754152edc97e641f537b20cf3d87a386c43/recipe/2024-01-support-private-terraform-repository.md)

> **Issue Reference:** <!-- (If appropriate) Reference an existing issue that describes the feature or bug. -->
https://github.com/radius-project/radius/issues/6539

### Goals

<!--
Describe goals to define why we are doing this work, how we will make
priority decisions, and how we will determine success.
-->
1. Enable users to use terraform recipes with multiple providers (including and outside of Azure, Kubernetes, AWS).
2. Enable users to use terraform recipes with multiple configurations for the same provider. 
3. Enable provider configurations to be available for all recipies used in an environment.


### Non goals
1. Updates to Bicep Provider configuration.
2. Authentication for providers hosted in private registries.
3. Support for .terraformrc files.

<!--
Describe non-goals to identify something that we won’t be focusing on 
immediately. We won’t be expending any effort on these matters. If there
will be follow-ups after this work, list them here. If there are things
we plan to do in the future, but are out of scope of this design, list
them here.
-->

### User scenarios (optional)

<!--
Describe the user scenarios for this design. Ensure that you define the
roles and personas in these user scenarios when it requires API design.
If you have an existing issue that describes the user scenarios, please
link to that issue instead.
-->

#### User story 1
As an operator, I maintain a set of Terraform recipes for use with Radius. I have a set of provider configurations which are applicable across multiple recipes and I would like to configure them in a single centralized location for ease of maintenance.


#### User story 2
As an operator, I would like to manage cloud resources in different geographical regions. To enable this, I need to set up multiple configurations for the same provider for different regions and use an alias to refer to individual provider configurations. 

## Design

<!--
Provide a high-level description, using diagrams as appropriate, and top-level
explanations to convey the architectural/design overview. Don’t go into a lot
of details yet but provide enough information about the relationship between
these components and other components. Call out or highlight new components
that are not part of this feature (dependencies). This diagram generally
treats the components as black boxes. Provide a pointer to a more detailed
design document, if one exists. 
-->
### Design details

<!--
This section should be detailed and thorough enough that another developer
could implement your design and provide enough detail to get a high confidence
estimate of the cost to implement the feature but isn’t as detailed as the 
code. Be sure to also consider testability in your design.

For each change, give each "change" in the proposal its own section and
describe it in enough detail that someone else could implement it. Cover
ALL of the important decisions like names. Your goal is to get an agreement
to proceed with coding and PRs.

If there are alternatives you are considering please include that in the open
questions section. If the product has a layered architecture, it's good to
align these sections with the product's layers. This will help readers use
their current understanding to understand your ideas.

* Advantages of this design - Describe what's good about this plan relative to
  other options. Does it feel easy to implement? Provide flexibility for
  future work?
* Disadvantages - Describe what's not ideal about this plan. If you don't
  point these things out other people will do it for you. This is a good place
  to cover risks.
-->
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
     providers: {
      ...
     }    
     env: {
       key: value
       secrets: {
        source: secretStoreOCICred.id
      }
     }
```

Environment variables apply to all providers configured in the environment. The system cannot set two separate values for the same environment variable for multple provider configurations. In such cases, per provider documentation, there may be alternatives that users can avail (eg. For GCP, users can set  *credentials* field inside each instance of provider config as opposed to using env variable GOOGLE_APPLICATION_CREDENTIALS).


The system will allow for ability to set up multiple configurations for the same provider using the keyword *alias*. Validation for configuration 'correctness' will be handled by Terraform with calls to terraform init and terraform apply.

```
...
 recipeConfig: {
    terraform: {
      ...
      providers: {
        [azurerm: {  
          subscriptionid: 1234
          secrets: {
            source: secretStoreAz.id
          }
        },
        azurerm: {
        subscriptionid: 1234
        alias: 'az-paymentservice'
        secrets: {
          source: secretStoreAzPayment.id
        }         
      }]
    } 
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
        git:{  
          pat:{
            'github.com':{
              secret: secretStoreGithub.id
            }
            'dev.azure.com': {
              secret: secretStoreAzureDevOps.id
            }
          }
        }          
      }
+      providers: {
+        [azurerm: {  
+          subscriptionid: 1234
+          secrets: {
+            source: secretStoreAz.id
+          }
+        },
+        azurerm: {
+         subscriptionid: 1234
+         alias: 'az-paymentservice'
+         secrets: {
+           source: secretStoreAzPayment.id
+         }         
+       },
+       gcp: {
+         project: 1234
+         regions: ['us-east1', 'us-west1']
+       },
+        oraclepass: {
+          database_endpoint = "..."
+          java_endpoint     = "..."
+          mysql_endpoint    = "..."
+        }]
+     }
+     env: {
+       key: value
+       secrets: {
+        source: secretStoreOCICred.id
+      }
+     }
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


Limitations: Customers may store sensitive data in other formats which may not be supported. eg. sensitive data is stored on files, which customers will not currently be able to load on disk in applications-rp where tf init/apply commands are run.
Containerization work may alleviate this limitations. Further design work will be needed for towards this which is planned for the near future. 

### API design (if applicable) (TBD)

<!--
Include if applicable – any design that changes our public REST API, CLI
arguments/commands, or Go APIs for shared components should provide this
section. Write N/A here if not applicable.
- Describe the REST APIs in detail for new resource types or updates to
  existing resource types. E.g. API Path and Sample request and response.
- Describe new commands in the CLI or changes to existing CLI commands.
- Describe the new or modified Go APIs for any shared components.
-->
TBD

## Alternatives considered

<!--
Describe the alternative designs that were considered or should be considered.
Give a justification for why alternative approaches should be rejected if
possible. 
-->
Mentioned under Limitations above, work to containerize running terraform jobs was considered as a precursor to this work. The time sensitivity for unblocking customers on ability to configure and use providers they use today was given a priority and current design held. Containerization will be taken up as a parallel effort in the near future.

## Test plan

<!--
Include the test plan to validate the features including the areas that
need functional tests.

Describe any functionality that will create new testing challenges:
- New dependencies
- External assets that tests need to access
- Features that do I/O or change OS state and are thus hard to unit test
-->
#### Unit Tests (TBD)
-   Update environment conversion unit tests to validate providers, env type under recipeConfig.
-   Update environment controller unit tests for providers, env.

#### Functional tests
- Add e2e test to verify recipe using multiple provider configuration is deployed.

## Security

<!--
Describe any changes to the existing security model of Radius or security 
challenges of the features. For each challenge describe the security threat 
and its mitigation with this design. 

Examples include:
- Authentication 
- Storing secrets and credentials
- Using cryptography

If this feature has no new challenges or changes to the security model
then describe how the feature will use existing security features of Radius.
-->
Largely following security considerations and secret handling described in design for private terraform modules: [Design document for Private Terraform Repository](https://github.com/radius-project/design-notes/blob/3644b754152edc97e641f537b20cf3d87a386c43/recipe/2024-01-support-private-terraform-repository.md)
The work done here will be to read, decipher values and secrets and environment variables set by user in the *providers* block, to build internal terraform configuration file used by tf setenv, init and apply commands. 

## Compatibility (optional)

<!--
Describe potential compatibility issues with other components, such as
incompatibility with older CLIs, and include any breaking changes to
behaviors or APIs.
-->

## Monitoring

<!--
Include the list of instrumentation such as metric, log, and trace to 
diagnose this new feature. It also describes how to troubleshoot this feature
with the instrumentation. 
-->

## Development plan (TBD)

<!--
Describe how you will deliver your features. This includes aligning work items
to features, scenarios, or requirements, defining what deliverable will be
checked in at each point in the product and estimating the cost of each work
item. Don’t forget to include the Unit Test and functional test in your
estimates.
-->

## Open issues

<!--
Describe (Q&A format) the important unknowns or things you're not sure about. 
Use the discussion to answer these with experts after people digest the 
overall design.
-->
