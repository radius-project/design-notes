# Terraform Settings Feature Specification

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

This feature specification refactors the existing Terraform functionality following these principles:

1. Platform engineers are able to use their existing Terraform modules with Radius without modifications or the need to create an intermediary main.tf configuration.
2. Radius is unopinionated about how Terraform is configured. Users can provide their existing Terraform settings to Radius and should expect the exact same behavior as if they ran Terraform from their workstation. 
3. Radius does not block the ability to use any Terraform features including `terraform plan`.
4. Radius does not manage Terraform, it only executes the Terraform CLI.

### Goals

* Enable platform engineers to install and upgrade their preferred version of the Terraform binary
* Enable platform engineers to provide their existing Terraform settings to Radius without modification
* Enable platform engineers to use, and authenticate to, their existing provider mirrors 
* Enable platform engineers to specify credentials for their module sources
* Enable platform engineers to use their existing Terraform modules without modifications or the need to create an intermediary main.tf configuration.
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
    --url="https://<TF_MIRROR_URL>/terraform_1.5.7_linux_amd64.zip" \
    --checksum="sha256:37f2d497ea512324d59b50ebf1b58a6fcc2a2828d638a4f6fdb1f41af00140f3"
Downloading Terraform from <TF_MIRROR_URL>
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

Similar to Recipe Packs, Terraform settings are modeled as resources. A TerraformSetting resource can be created declaratively by deploying a `Radius.Config/terraformSettings` resource into a resource group. Unlike Recipe Packs, there is no way to configure Terraform via imperative commands. Imperative commands exist for simpler use cases. In the case of Terraform, there is no need to configure Terraform unless the platform engineer needs to configure advanced settings. In other words, Terraform works with the default out-of-the-box settings so imperative commands are not necessary.

The TerraformSettings resource is a like for like copy of the Terraform CLI configuration file (`terraformrc`) and settings available in the terraform block of main.tf files. The goal is for users to be able to bring their existing Terraform settings, provide them to Radius, and Terraform behaves exactly as if Terraform was ran from the user's workstation.

**Radius Behavior**

For `terraformrc` settings, Radius will pass this configuration directly to the Terraform CLI. The only processing of the file is the injection of secret data stored in a Radius Secret resource.

**User Experience**

The platform engineer creates a TerraformSettings resource and references that resource in a new property on the Environment.

**`terraformEnvironment.bicep`**

```yaml
param string token

resource myEnvironment 'Radius.Core/environments@2025-05-01-preview' = {
  name: 'myEnvironment'
  properties: {
    terraformSettings: myTerraformSettings.id
  }
}

resource myTerraformSettings 'Radius.Config/terraformSettings@2025-08-14-preview' = {
  name: 'myTerraformSettings'
  terraformrc: {
    provider_installation: {
      network_mirror: {
        path: 'https://<TF_MIRROR_URL>/'
        include: ["*"]
      }
      direct: {
        exclude: ["azurerm"]
      }
    }
    // Equivilent to recipeConfig.authentication on Environments today
    credentials: [
      <URL>: {
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

resource terraformProviderSecret 'Radius.Security/secret@2025-05-01-preview' = {
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

#### Refactoring Current Functionality

The TerraformSettings resource replaces the `recipeConfig` property of the Environment. This approach is similar to the Recipe Pack approach of abstracting out properties historically on the Environment into their own resource which can be shared across multiple Environments (some Radius users are planning for hundreds of Environments).

* `Environment.recipeConfig.terraform.authentication.pat` – Replaced by `TerraformSettings.credentials`
* `Environment.recipeConfig.terraform.env` – Replaced by `TerraformSettings.env`

#### Deprecating Current Functionality

Radius currently supports customizing Terraform providers via the `Environment.recipeConfig.terraform.providers` property. The original feature request was to support injecting credentials into an arbitrary Terraform provider. 

Tentatively, this feature is planned to be deprecated. This feature is a duplicate of the out-of-the-box credential injection. Rather than have duplicate features, we will identify gaps in the existing out-of-the-box credentials and close those gaps. 

One user is known to be using this feature to inject Azure credentials into the AzureDevops Terraform provider. Once feedback is received from this user, this section will be updated.

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
resource myTerraformSettings 'Radius.Config/terraformSettings@2025-08-14-preview' = {
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
resource myTerraformSettings 'Radius.Config/terraformSettings@2025-08-14-preview' = {
  name: 'myTerraformSettings'
  terraformrc: { ... }
  backend: {
    type: 'azurerm'
    subscription_id: '<MY_SUBSCRIPTION_ID>'
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

## Scenario 3 – Observing Terraform

### User Story 5 – Collecting Trace Logs

***As a platform engineer, I need to troubleshoot Terraform and Radius, therefore I need to see trace-level logs.***

**Summary**

Terraform logging level is set via the `TF_LOG` environment variable. Environment variables for Terraform are a property of the TerraformSettings resource. User story 4 includes the `env` block:

```yaml
  env: {
    TF_REGISTRY_CLIENT_TIMEOUT: 15
    TF_LOG: 'TRACE'
    TF_LOG_PATH: 'path/from/radius/documentation
  }
```

Platform engineers will be able to set `TF_LOG` and `TF_LOG_PATH` and:

* Inspect Terraform logs separately from other Radius logs via a `rad` or `kubectl` command
* Collect Terraform logs via fluentbit or equivalent log collector

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

## Future Work

Credentials need additional work in the future. Today, Radius stores only one AWS and Azure credential globally. In the future, credentials will needs to be broken out into its own resource type and include:

* `Radius.Config/kubernetesClientKey` – A client key for an arbitrary Kubernetes cluster not running the Radius control plane
* `Radius.Config/azureServicePrincipal` – A client ID and secret for Azure
* `Radius.Config/azureWorkloadIdentitySettings` – The client ID and tenant ID for workload identity configured on an AKS cluster
* `Radius.Config/awsAccessKeys` – An access key for AWS
* `Radius.Config/awsIamRole` – An AWS IAM role for use with AWS EKS configured with IRSA

Once these credentials resources exist, the Environment can be augmented with an Environment-specific set of credentials.