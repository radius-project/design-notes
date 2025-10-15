# Terraform and Bicep Settings Feature Specification

[@zachcasper](https://github.com/zachcasper)

## Summary

Terraform is the most commonly used infrastructure as code (IaC) solution. Many large organizations have significant investments in Terraform. Not only do large organizations have large libraries of Terraform configurations, but they have also invested in establishing governance procedures, testing frameworks, and leverage Terraform for security, compliance, and cost controls. 

Radius integrates with Terraform today. It is highly opinionated in that Radius exposes only a subset of Terraform setting. This makes using Radius a challenge for users with mature Terraform set ups. These challenges include:

* Inability to manage the Terraform binary (Radius automatically installs the latest version from hashicorp.com)
* Inability to specify provider mirrors
* Inability to specify credentials for provider mirrors and module sources
* Inability to use an existing Terraform backend state store
* Friction-full process for injecting cloud provider credentials
* Lack of observability of Terraform executions
* Lack of `terraform plan` functionality (although this is out of scope for this document)

This feature specification refactors the existing `Environments.recipeConfig` functionality following these principles:

1. Platform engineers are able to use their existing Terraform modules with Radius without modifications or the need to create an intermediary main.tf configuration.
2. Radius is unopinionated about how Terraform is configured. Users can provide their existing Terraform settings to Radius and should expect the exact same behavior as if they ran Terraform from their workstation. 
3. Radius does not block the ability to use any Terraform features including `terraform plan`.
4. Radius does not manage Terraform, it only executes the Terraform CLI.
5. Ensures Bicep and Terraform have similar user experiences

> [!NOTE]
>
> This feature spec does not add any functionality for Bicep but does refactor the existing `Environments.recipeConfig.bicep` to be consistent with Terraform.

### Goals

* Enable platform engineers to install and upgrade their preferred version of the Terraform binary independently of Radius
* Enable platform engineers the ability to specify Recipe settings independently of Environments to ease managing Environments at scale (similar objectives of Recipe Packs)
* Enable platform engineers to provide their existing Terraform settings to Radius without modification, including using and authenticating to provider mirrors and internal module sources
* Provide trace-level logs for Terraform when executed by Radius

### Non-Goals (out of scope)

* `terraform plan` functionality (this will be addressed in a future feature specification)

## Scenario 1 – Installing and Updating Terraform

### User Story 1 – Install Terraform

***As a platform engineer, I need to install Terraform into the Radius control plane in my connected environment.***

##### **Summary**

Today, the Application RP container dynamically downloads the Terraform binary from hashicorp.com when a Terraform recipe is executed. This is problematic because Terraform is Radius dependency, not a Radius component. So platform engineers need to be in control of what Terraform is being used. They may use a specific version of Terraform within their organization. They may have specific licensing concerns about Terraform. Or they may simply not want Terraform to be running in their environment.

Therefore, Radius will move to a model where the platform engineer explicitly installs Terraform. This is true for offline installations as well as connected installations.

##### **User Experience 1 – Quick Install**

The platform engineer installs Terraform using the new `rad terraform install` command.

```bash
$ rad terraform install
No Terraform release specified, installing the latest from releases.hashicorp.com
terraform_1.12.2_linux_amd64 installed in the Radius control plane
```

When this command is executed, Radius:

* Downloads the latest Terraform release from releases.hashicorp.com and validates the checksum
* Installs the Terraform binary in the appropriate location within the Radius control plane
* Performs a `terraform init` and reports any output to the platform engineer

Note that `terraform init` will need to be re-executed on recipe execution to initialize to a user-provided backend which happens later.

Changes from today's Radius:

* Terraform is no longer dynamically installed in Application RP

**User Experience 2 – Advanced Install**

The platform engineer uses the same command but with specific URL of the binary and its checksum.

```bash
$ rad terraform install \
    --url="https://<tf_mirror_url>/terraform_1.5.7_linux_amd64.zip" \
    --checksum="sha256:37f2d497ea512324d59b50ebf1b58a6fcc2a2828d638a4f6fdb1f41af00140f3"
Downloading Terraform from <tf_mirror_url>
Verifying checksum
The specified Terraform CLI has been installed in the Radius control plane
```

When this command is executed, Radius:

* Downloads the Terraform binary specified by the user
* If the checksum argument is provided, the checksum is validated; if no checksum is provided, this step is skipped
* Installs the Terraform binary in the appropriate location within the Radius control plane
* Performs a `terraform init` and reports any output to the platform engineer

### User Story 2 – Upgrading Terraform

***As a platform engineer, I need to upgrade the version of Terraform used by Radius***

The platform engineer uses the same `rad terraform install` command as before. The install command overrides the existing installation with either the latest version or the version specified in the URL argument.

### User Story 3 – Uninstall Terraform

***As a platform engineer, I need to remove all Terraform installations from my environment, including Radius***

Terraform is a third-party solution which the platform engineer installs into Radius. Given this, Radius must enable platform engineers to remove Terraform as well.

The platform engineer deletes the Terraform resource.

```bash
$ rad terraform uninstall
```

Radius deletes the Terraform binary from the Radius control plane.

## Scenario 2 – Configuring Terraform

### User Story 4 – Configure Terraform CLI

***As a platform engineer, I need to configure the Terraform CLI running in the Radius control plane.***

**Summary**

Similar to Recipe Packs, Terraform settings are modeled as resources. A TerraformSettings resource can be created declaratively by deploying a `Radius.Core/terraformSettings` resource into a resource group. Unlike Recipe Packs, there is no way to configure Terraform via imperative commands. Imperative commands exist for simpler use cases. In the case of Terraform, there is no need to configure Terraform unless the platform engineer needs to configure advanced settings. In other words, Terraform works with the default out-of-the-box settings so imperative commands are not necessary.

The TerraformSettings resource is a like for like copy of the Terraform CLI configuration file (`terraformrc`) and settings available in the terraform block of main.tf files. The goal is for users to be able to bring their existing Terraform settings, provide them to Radius, and Terraform behaves exactly as if Terraform was ran from the user's workstation.

**Radius Behavior**

For `terraformrc` settings, Radius will pass this configuration directly to the Terraform CLI. The only processing of the file is the injection of secret data stored in a Radius Secret resource.

**User Experience**

The platform engineer creates a TerraformSettings resource and references that resource in a new property on the Environment.

**`terraformEnvironment.bicep`**

```yaml
param string token

resource myEnvironment 'Radius.Core/environments@2025-08-01-preview' = {
  name: 'myEnvironment'
  properties: {
    recipePacks: [myRecipePack.id]
    terraformSettings: myTerraformSettings.id
  }
}

resource myRecipePack 'Radius.Core/recipePacks@2025-08-01-preview'= { ... }

resource myTerraformSettings 'Radius.Core/terraformSettings@2025-08-01-preview' = {
  name: 'myTerraformSettings'
  terraformrc: {
    provider_installation: {
      network_mirror: {
        path: 'https://<tf_mirror_url>/'
        include: ["*"]
      }
      direct: {
        exclude: ["azurerm"]
      }
    }
    // Equivilent to recipeConfig.authentication on Environments today
    credentials: [
      <url>: {
        secret: providerSecret.id
      }
    ]
  }
  // Environment variables injected into the Terraform process
  // Equivilent to recipeConfig.env on Environments today
  env: {
    TF_REGISTRY_CLIENT_TIMEOUT: 15
    TF_LOG: 'TRACE'
  }
}

resource terraformProviderSecret 'Radius.Security/secrets@2025-08-01-preview' = {
  name: 'terraformProviderSecret'
  properties: {
    type: 'generic'
    data: {
      token: {
        value: token
      }
    }
  }
}
```

The platform engineer then deploys the Environment and TerraformSettings.

```bash
$ rad deploy terraformEnvironment.bicep --group myGroup
```

#### Available Terraform Settings

##### Terraform CLI Configuration File (`terraformrc`)

The `terraformrc` file is the configuration file for the Terraform binary.

| Setting                                             | Description                                                  | Supported by Radius |
| --------------------------------------------------- | ------------------------------------------------------------ | ------------------- |
| credentials                                         | Token for private registry and modules                       | ✅                   |
| credential_helper                                   | External program for authenticating                          | ❌                   |
| disable_checkpoint and disable_checkpoint_signature | Disables checking with Hashicorp about upgrade and security updates | ❌                   |
| plugin_cache_dir                                    | Location of plugin cache                                     | ❌                   |
| provider_installation                               | Specifies the location of providers                          | ✅                   |

The entire set of properties for credentials and provider_installation are supported. See the [Terraform documentation](https://developer.hashicorp.com/terraform/cli/config) for the full set of properties.

Unsupported settings are not part of the TerraformSettings resource schema, so the TerraformSettings resource will not deploy if unsupported settings are specified. 

### User Story 5 – Configure Terraform Backend

***As a platform engineer, I need to configure the Terraform backend state store.***

**Summary**

The Terraform backend is the state store for Terraform. Today, Radius uses an opinionated Kubernetes backend. Platform engineers need the ability to specify an existing backend. Terraform backends are specified in the `terraform` block similar to how providers are specified. 

Radius will support all backend types supported by Terraform except the local backend and remote types. Local does not make sense when running Terraform via Radius and remote has been replaced with Terraform Cloud.

There will be two tiers of support:

* **Tier 1** – Properties available in the TerraformSettings resource, integrated authentication, tested by the Radius project
* **Tier 2** – Only properties available in the TerraformSettings resource. It is left to future development to implement authentication for these backend types.

The backend types include:

| Tier 1                                 | Tier 2                                             |
| -------------------------------------- | -------------------------------------------------- |
| Kubernetes (the default)               | Alibaba Cloud Object Storage Service (`oss`)       |
| AWS S3 (`s3`)                          | Consul (`consul`)                                  |
| Azure Blob Storage Account (`azurerm`) | Google Cloud Storage (`gcs`)                       |
|                                        | HTTP (`http`)                                      |
|                                        | Oracle Cloud Infrastructure Object Storage (`oci`) |
|                                        | PostgreSQL database (`pg`)                         |
|                                        | Tencent Cloud Object Storage (`cos`)               |

As with the Terraform CLI settings, Terraform backend is configured via the TerraformSettings resource.

**`terraformS3Backend.bicep`**

```yaml
resource myTerraformSettings 'Radius.Core/terraformSettings@2025-08-14-preview' = {
  name: 'myTerraformSettings'
  terraformrc: { ... }
  backend: {
    type: 's3'
    bucket: 'my-company-dev-tfstate-bucket'
    key: 'radius/terraform.tfstate'
    region: 'us-east-1'
    encrypt: 'true'
    dynamodb_table: 'terraform-global-locks'
  }
}
```

**`terraformAzureBackend.bicep`**

```yaml
resource myTerraformSettings 'Radius.Core/terraformSettings@2025-08-14-preview' = {
  name: 'myTerraformSettings'
  terraformrc: { ... }
  backend: {
    type: 'azurerm'
    subscription_id: '<subscription_id>'
    resource_group_name: 'terraform-state-rg'
    storage_account_name: 'terraformstate'
    container_name: 'tfstate'
    key: 'terraform.tfstate'
  }
}
```

##### Authentication

Terraform backends support authentication very similar to Terraform providers. Specifically:

* **azurerm** – Supports service principals with client secrets set via environment variables (`ARM_CLIENT_ID`, `ARM_CLIENT_SECRET`, `ARM_TENANT_ID`, `ARM_SUBSCRIPTION_ID`) and managed identity 
* **s3** – Supports access keys via environment variables (`AWS_ACCESS_KEY_ID` and `AWS_SECRET_ACCESS_KEY`)

No additional authentication configuration is support, therefore, users do not need to configure any additional credentials beyond what is already in place with Radius.

### User Story 6 –Injecting Radius Secrets into Terraform Configurations

***As a platform engineer, I need inject credentials into a Terraform provider other than azurerm and aws.***

**Summary**

Terraform configurations often have multiple Terraform providers in addition to the core Kubernetes, Azure, or AWS providers. Observability platforms such as Datadog is a prime example. Each Terraform provider may need credentials in order to deploy resources. Platform engineers need the ability to inject a Radius Secret value into an arbitrary Terraform provider.

Radius supports this capability today via the [Customer Terraform Providers functionality](https://docs.radapp.io/guides/recipes/terraform/howto-custom-provider/). This works via the `Environments.recipeConfig.terraform.providers` property. It  allows an arbitrary map of data to be injected into specified Terraform providers. While the input is an arbitrary map, injecting a Radius Secret is possible.

**User Experience**

The new user experience leverages Recipe parameters. The platform engineer first defined a secret with an apiKey, for example:

```yaml
resource datadogCredentials 'Radius.Security/secrets@2025-08-01-preview' = {
  name: 'datadogCredentials'
  properties: {
    environment: environment
    application: myApplication.id
    data: {
      apikey: {
        value: <datadog-api-key-value>
      }
      appKey: {
        value <datadog-app-key-value>
      }
    }
  }
```

They then add a parameter on the Recipe with the new apiKey secret:

```yaml
resource computeRecipePack 'Radius.Core/recipePacks@2025-05-01-preview' = { ...
    recipes: [
      Radius.Compute/container: {
        recipeKind: 'terraform'
        recipeLocation: '<recipe-git-url>'
        parameters: {
          apiKey: datadogCredentials.properties.data.apikey.value
          appKey: datadogCredentials.properties.data.appkey.value
        } ...
```

In the Recipe itself, a var for the Recipe parameter and the the var is used in the provider block:

```yaml
terraform {
  required_providers {
    datadog = {
      source = "DataDog/datadog"
    }
  }
}

var datadog_api_key
var datadog_app_key

provider "datadog" {
  api_key = var.datadog_api_key
  app_key = var.datadog_app_key
}
```

**Engineering Considerations**

The Secret values must be handled securely within Radius. When a Recipe is called with a Recipe parameter referencing a Secret, Radius must retrieve the secret value and pass that to Terraform without storing the secret value within the Radius control plane or any logs.

## Scenario 3 – Observing Terraform

### User Story 7 – Collecting Trace Logs

***As a platform engineer, I need to troubleshoot Terraform and Radius, therefore I need to see trace-level logs.***

**Summary**

The platform engineer must be able to:

* Set the Terraform logging level (`TRACE` , `DEBUG` , `INFO` , `WARN` , `ERROR` and `FATAL`)
* Inspect Terraform logs separately from other Radius logs via a `rad` or `kubectl` command
* Collect Terraform logs via fluentbit or equivalent log collector

The user experience is deferred to the technical design.

## Scenario 4 – Configuring Bicep

### User Story 8 – Private Bicep Recipes

**Summary**

To meet the goal of *enabling platform engineers the ability to specify Recipe settings independently of Environments to ease managing Environments at scale*, the recipeConfig property needs to be refactored completely out of the Environments resource. Environments has a Bicep property named `Environments.recipeConfig.bicep.authentication` which is used to [specify a Secret for accessing OCI registries](https://docs.radapp.io/guides/recipes/howto-private-bicep-registry/) where Bicep templates are stored. In order to completely remove the recipeConfig property, an alternative property must be provided.

**User Experience**

Just like Terraform, Bicep will have a BicepSettings resource. The platform engineer creates a BicepSettings resource and references that resource in a new property on the Environment.

**`bicepEnvironment.bicep`**

```yaml
param string token

resource myEnvironment 'Radius.Core/environments@2025-08-01-preview' = {
  name: 'myEnvironment'
  properties: {
    recipePacks: [myRecipePack.id]
    bicepSettings: myBicepSettings.id
  }
}

resource myRecipePack 'Radius.Core/recipePacks@2025-08-01-preview'= { ... }

resource myBicepSettings 'Radius.Core/bicepSettings@2025-08-01-preview' = {
  name: 'myBicepSettings'
  properties: {
    registryAuthentication: {
      // Supported authentication methods: BasicAuth (username, password), AzureWI, and AwsIrsa
      authenticationMethod: 'BasicAuth'
      // If using BasicAuth, Secret must have a `username` and `password` key
      basicAuthSecretId: bicepRegistrySecret.id
      azureWiClientId: '12345678-abcd-efgh-ijkl-9876543210ab'
      azureWiTenantId: '12345678-abcd-efgh-ijkl-9876543210ab'
      awsIamRoleArn: 'arn:aws:iam::012345678901:role/test-role'
    }
  }
}

resource bicepRegistrySecret 'Radius.Security/secrets@2025-08-01-preview' = {
  name: 'bicepRegistrySecret'
  properties: {
    data: {
      username: {
        value: '<username>'
      }
      password: {
        value: '<password>'
      }
    }
  }
}
```

The platform engineer then deploys the Environment and TerraformSettings.

```bash
$ rad deploy bicepEnvironment.bicep --group myGroup
```

**Other Engineering Changes**

Azure workload identity client ID and tenant ID are not considered secrets. Nor is the AWS IAM ARN. Therefore this information is not required to be stored in a secret. With this change, the Secret kind property can be removed. 

## Summary of Property Changes

| As Is                                                        | To Be                                                        |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| `Environment.recipeConfig.terraform.authentication.git.pat.secret` | `TerraformSettings.credentials`                              |
| `Environment.recipeConfig.terraform.providers.<provider>.secrets` | Replaced by Recipe parameters                                |
| `Environment.recipeConfig.bicep.authentication.<oci-registry>.secret` | `BicepSettings.registryAuthentication`                       |
| `SecretStore.type: awsIRSA`                                  | `BicepSettings.registryAuthentication.awsIamRoleArn`         |
| `SecretStore.type: azureWorkloadIdeneity`                    | `BicepSettings.registryAuthentication.azureWiClientId` and `azureWiTenantId` |
| `Environment.recipeConfig.env`                               | `TerraformSettings.env`                                      |
| `Environment.recipeConfig.envSecrets`                        | Not implemented, replaced with Recipe parameters             |

## Summary of changes

| Priority | Size | Description                                                  |
| -------- | ---- | ------------------------------------------------------------ |
| p0       | L    | New `rad terraform install` command with URL and checksum; removal of dynamic Terraform installation |
| p1       | M    | Setting the Terraform CLI configuration based on the new TerraformSettings resource |
| p1       | M    | `azurerm` backend type in TerraformSettings; service principal and managed identity authentication from Radius credentials |
| p1       | S    | Support for `TF_LOG` and `TF_LOG_PATH`                       |
| p2       | M    | `s3` backend type in TerraformSettings; access key authentication from Radius credentials |
| p3       | M    | `kubernetes` backend type in TerraformSettings (same functionality as today but customizable) |
| p3       | M    | New `rad terraform uninstall` command                        |

## Appendix: TerraformSettings Schema

```yaml
namespace: Radius.Core
types:
  terraformSettings:
    apiVersions:
      '2025-08-01-preview':
        schema: 
          type: object
          properties:
            terraformrc:
              type: object
              properties:
                provider_installation:
                  type: object
                  properties:
                    network_mirror:
                      type: object
                      properties:
                        path:
                          type: string
                        include:
                          type: array
                          items:
                            type: string
                        exclude:
                          type: array
                          items:
                            type: string
                    direct:
                      type: object
                      properties:
                        include:
                          type: array
                          items:
                            type: string
                        exclude:
                          type: array
                          items:
                            type: string
                credentials:
                  type: object:
                  additionalProperties:
                    type: object
                    properties:
                      secret:
                        type: string
                env:
                  type: object:
                  additionalProperties:
                    type: string
```

## Appendix: BicepSettings Schema

```yaml
namespace: Radius.Core
types:
  bicepSettings:
    apiVersions:
      '2025-08-01-preview':
        schema: 
          type: object
          properties:
            registryAuthentication:
              type: string
              enum: [BasicAuth, AzureWI, AwsIrsa]

  properties: {
    registryAuthentication: {
      // Supported authentication methods: BasicAuth (username, password), AzureWI, and AwsIrsa
      authenticationMethod: 'BasicAuth'
      // If using BasicAuth, Secret must have a `username` and `password` key
      basicAuthSecretId: bicepRegistrySecret.id
      azureWiClientId: '12345678-abcd-efgh-ijkl-9876543210ab'
      azureWiTenantId: '12345678-abcd-efgh-ijkl-9876543210ab'
      awsIamRoleArn: 'arn:aws:iam::012345678901:role/test-role'
    }
  }
}

resource bicepRegistrySecret 'Radius.Security/secrets@2025-08-01-preview' = {
  name: 'bicepRegistrySecret'
  properties: {
    data: {
      username: {
        value: '<username>'
      }
      password: {
        value: '<password>'
      }
    }
  }
}
```

