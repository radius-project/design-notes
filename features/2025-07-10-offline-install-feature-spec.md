# Offline Installation Feature Spec

[@zachcasper](https://github.com/zachcasper)

## Summary

Many large organizations have mature security practices where all software entering the organization's environment are controlled. Typically all software packages are scanned for security vulnerabilities prior to being stored internally. Access to non-authorized software packages are restricted. 

Radius must support these organizations' requirements by making it easy to install, configure, and upgrade Radius while offline, or in an *air gapped* environment.

## Goals

* Enable platform engineers to install and upgrade Radius while not connected to the internet
* Enable platform engineers to control what versions of all require software components are installed
* Increase the transparency of what software is required to run Radius

## Non-Goals (out of scope)

* Enhancing the installation experience in any other way

## Inventory of software components

Radius makes the assumption that the installation environment has full access to the internet and installs many software components on behalf of the user. Below is an inventory:

1. **Binaries distributed via GitHub.** These binaries will be downloaded and installed manually.

   * `rad`


   * `rad-bicep`


2. **The Radius Helm chart.** This chart will be downloaded using `helm pull` and stored either in a Helm repository mirror or on the file system.
   * https://github.com/project-radius/radius/tree/main/deploy/Chart


3. **Container images distributed via GHCR**. These images will be imported into the user's private OCI registry.

   * `ucpd`

   * `controller`

   * `applications-rp`

   * `dynamic-rp`

   * `bicep`

   * `dashboard`


4. **Container images distributed via Docker Hub**. Radius will have an option to not install Contour.

   * `contour`

   * `envoy`


5. **Bicep extensions distributed via ACR**. These images will be imported into the user's private OCI registry.

   * `biceptypes.azurecr.io/radius`

   * `biceptypes.azurecr.io/aws`


6. **NPM packages distributed via npmjs.com**. This package will be eliminated and implemented in the Go program.
   * https://www.npmjs.com/package/@radius-project/manifest-to-bicep-extension


7. **Third-party binaries**. Radius will offer an installation option to install the Terraform binary from the user's location.
   * Terraform via https://releases.hashicorp.com/


## Scenario 1 – Installing Radius

### User Story 1 – Downloading releases

***As a platform engineer, I need to download the Radius release to an environment not connected to the internet***

The platform engineer consults the release notes for the location of binaries. They will need to download:

* Radius CLI binaries (`rad` and `rad-bicep`)
* Radius Helm chart
* Radius Container images
* Radius and AWS Bicep extensions

**Radius CLI binaries**

The platform engineer consults the Radius release notes for the list of assets. They see a list of CLI binaries clearly named CLI to distinguish from the container images. This includes:

* The `rad` binary which will be renamed from  `rad_<KERNEL>_<ARCH>` to  `rad_cli_<KERNEL>_<ARCH>`
* The `rad-bicep` binary will now be listed on the release notes

The platform engineer or security engineer imports the Radius CLI into their software mirror (for example, Sonatype Nexus Repository mirror). 

**Helm chart**

If the organization has a Helm chart repository mirror such as Sonatype Nexus, the platform engineer or security engineer can load the Radius chart into the mirror using something similar to:

```bash
helm pull https://github.com/radius-project/radius/tree/main/deploy/Chart
helm repo add nexus http://<NEXUS_HOSTNAME>/repository/<NEXUS_REPOSITORY_NAME>/ 
helm cm-push radius nexus
```

If the organization does not have a Helm chart repository mirror, the platform engineer or security engineer can download the Radius chart.

```bash
helm pull https://github.com/radius-project/radius/tree/main/deploy/Chart
```

**Container images**

The platform engineer or security engineer imports the Radius container images into their OCI registry. For example, an enterprise who uses Azure Container Registry may import the images using a command similar to:

```bash
# Import UCP container image
az acr import \
  --name <ACR_REGISTRY_NAME> 
  --source ghcr.io/radius-project/ucpd:0.48
  --image ucpd:0.48
# Import Controller container image
az acr import ...
# Import Applications RP container image
az acr import ...
# Import Dynamic RP container image
az acr import ...
# Import Bicep container image
az acr import ...
# Import Dashboard container image
az acr import ...
```

**Bicep extensions**

The platform engineer or security engineer imports the Radius Bicep extensions into their OCI registry. 

```bash
# Import Radius Bicep extension
az acr import \
  --name <ACR_REGISTRY_NAME> 
  --source biceptypes.azurecr.io/radius:0.48
  --image radius:0.48
# Import AWS Bicep extension (optionally)
az acr import \
  --name <ACR_REGISTRY_NAME>
  --source biceptypes.azurecr.io/aws:0.48
  --image aws:0.48
```

### User Story 2 – Install CLI

***As a platform engineer, I need to install the Radius CLI***

Rather than running the `install.sh` script, the platform engineer installs the CLI manually. The platform engineer downloads `rad` and `rad-bicep` and stores in `/usr/local/bin/rad` and confirms that directory is in their $PATH.

The platform engineer creates a `bicepconfig.json` file with these contents:

```json
{
	"experimentalFeaturesEnabled": {
		"extensibility": true
	},
	"extensions": {
		"radius": "br:<OCI_REGISTRY_URL>/<REPOSITORY>/radius:0.48",
		"aws": "br:<OCI_REGISTRY_URL>/<REPOSITORY>/aws:0.48",
    "radiusResources": "radiusResources.tgz"
	}
}
```

> [!NOTE]
>
> The installation instructions for installing the Radius CLI are incomplete and duplicated multiple times:
>
> * https://docs.radapp.io/guides/tooling/rad-cli/howto-rad-cli/ 
>   * Binaries should be renamed Offline or Manual
>   * Needs actual instructions for installing `rad`
>   * Need instructions for installing `rad-bicep`
>   * Need instructions for creating the `bicepconfig.json` 
> * https://docs.radapp.io/getting-started/#2-install-radius-cli (duplicate)
> * https://docs.radapp.io/installation/#step-1-install-the-rad-cli (duplicate)
> * https://docs.radapp.io/guides/tooling/rad-cli/overview/
>   * "If you would like to install the rad CLI to a different path, you can specify the path with the `RADIUS_INSTALL_DIR` environment variable" should be in the installation section

### User Story 3 – Install Radius via CLI

***As a platform engineer, I need to install Radius offline using the Radius CLI***

The platform engineer installs Radius using the CLI with override for the various settings.

```bash
rad install kubernetes \
  --set image.repository=<OCI_REGISTRY_URL>/<REPOSITORY>
  --skip-contour-install
```

> [!NOTE]
>
>  `rad init` will not support offline installations.

The `image.repository` is the location of the Radius container images mirror.

The `--skip-contour-install` does not install Contour. Therefore the Gateway resource types will not function until extensibility is implemented.

> [!CAUTION]
>
> `--skip-contour-install` is expected to be removed from Radius in the near future as part of the compute extensibility work. The design for the Gateway recipe no longer uses Contour and Contour is no longer installed by default.

If a Helm mirror has not been configured via `helm repo add`, the `--chart` argument can be used.

```bash
rad install kubernetes \
  --chart <PATH_TO_RADIUS_HELM_CHART_FROM_HELM_PULL> \
  ...
```

### User Story 4 – Install Radius via Helm

***As a platform engineer, I need to install Radius using Helm***

The platform engineer installs Radius using Helm with similar settings.

```bash
helm upgrade radius radius/radius \
  --install \
  --create-namespace 
  --namespace radius-system \
  --version 0.48.0 \
  --wait --timeout 15m0s \
  --set image.repository=<OCI_REGISTRY_URL>/<REPOSITORY> \
```

## Scenario 2 – Managing Terraform

### User Story 5 – Install Terraform

***As a platform engineer, I need to install and configure Terraform for Radius to use***

The platform engineer deploys a Terraform resource. The Terraform resource replaced the `recipeConfig` properties in the Environment today and adds all other [terraformrc configuration options](https://developer.hashicorp.com/terraform/cli/config/config-file).

**`terraform.bicep`**

```yaml
param token string

resource tf 'System.Resources/terrform@2025-08-01-preview' = {
  name: 'tf'
  terraformCLI: {
    url: 'https://<TF_MIRROR_URL>/terraform_1.5.7_linux_amd64.zip'
    checksum: 'sha256:37f2d497ea512324d59b50ebf1b58a6fcc2a2828d638a4f6fdb1f41af00140f3'
  }
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
    credentials: {
      <TF_MIRROR_URL>: {
        secret: providerSecret.id
      }
    }
    // Environment variables injected into the Terraform process
    // Equivilent to recipeConfig.env on Environments today
    env: {
      TF_REGISTRY_CLIENT_TIMEOUT: 15
    }
  }
}

resource providerSecret 'Applications.Core/secretStores@2023-10-01-preview' = {
  name: 'providerSecret'
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

The Terraform resource intentionally does not include these terraformrc properties:

* `credentials_helper`
* `provider_installation.filesystem_mirror`
* `disable_checkpoint`
* `disable_checkpoint_signature`
* `plugin_cache_dir`

The platform engineer deploys the Terraform resource.

````bash
$ rad deploy terraform.bicep
````

When the Terraform resource is deployed, Radius:

* Downloads the specified Terraform binary and validates it against the provided checksum
* Installs the Terraform binary in the appropriate location within the Radius control plane
* Performs a `terraform init` and reports any output to the platform engineer

> [!NOTE]
>
> This feature specification diffs from the initial implementation which took properties on the `rad install` command. 

The Environment resource no longer has a recipeConfig. This includes:

| As-Is Resource | As-Is Property                          | To-Be Resource | To-Be Property            |
| -------------- | --------------------------------------- | -------------- | ------------------------- |
| Environment    | `recipeConfig.terraform.authentication` | Terraform      | `terraformrc.credentials` |
| Environment    | `recipeConfig.terraform.providers`      | Terraform      | ??                        |

#### Alternatives Considered

We considered extending the recipeConfig property of the Environment resource to include the provider mirror. The challenge with this approach is that the `recipeConfig` is on the Environment. As with Recipe Packs, the Environment resource needs to be as slim as possible because there will be hundreds of environments but only one configuration for Terraform.

We also considered installing Terraform during Radius installation. The benefit of the proposed approach here are:

* Enables platform engineers to install Terraform after Radius has been installed
* Enables platform engineers to upgrade Radius without knowing the Terraform installation instructions
* Bundles all Terraform installation and configuration into one user action
* Enables a clear path to future Terraform functionality. In the future, the Terraform resource can be enhanced with the [Terraform backend configuration](https://developer.hashicorp.com/terraform/language/backend). 

### User Story 6 – Upgrading Terraform

***As a platform engineer, I need to upgrade the version of Terraform used by Radius***

The platform engineer modifies the already deployed Terraform resource and redeploys the resource. 

First, the platform engineer gets the existing resource using the new `bicep` output option.

```bash
$ rad resource list System.Resources/terrform
RESOURCE  TYPE                       GROUP          STATE
tf        System.Resources/terrform  radius-system  Succeeded

$ rad resource show System.Resources/terrform -o bicep > terraform.bicep
```

The platform engineer then edits the version and checksum.

```diff
  terraformCLI: {
-    url: 'https://<TF_MIRROR_URL>/terraform_1.5.7_linux_amd64.zip'
+    url: 'https://<TF_MIRROR_URL>/terraform_1.9.1_linux_amd64.zip'
-    checksum: 'sha256:37f2d497ea512324d59b50ebf1b58a6fcc2a2828d638a4f6fdb1f41af00140f3'
+    checksum: 'sha256:f1426fccbf2500202b37993ef6b92e1fc60d114dd32c79bfadbc843929b2c7e2'
```

Then redeploys the Terraform resource.

```bash
$ rad deploy terraform.bicep
```

As before, when the Terraform resource is deployed, Radius:

* Downloads the specified Terraform binary and validates it against the provided checksum
* Installs the Terraform binary in the appropriate location within the Radius control plane
* Performs a `terraform init` and reports any output to the platform engineer

### User Story 7 – Uninstall Terraform

***As a platform engineer, I need to remove all Terraform installations from my environment, including Radius***

Terraform is a third-party solution which the platform engineer installs into Radius. Given this, Radius must enable platform engineers to remove Terraform as well.

The platform engineer deletes the Terraform resource.

```bash
$ rad resource delete System.Resources/terraform tf
```

Radius deletes the Terraform binary from the Radius control plane.

## Scenario 3 – Additional offline configurations 

### User Story 8 – Specify trusted certificate authority

***As a platform engineer, I need provide my organization's certificate authority certificate when installing Radius***

The platform engineer installs Radius using the CLI with override for the various settings.

```bash
rad install kubernetes \
  --set-file global.rootCA.cert=<ROOT_CA.crt> \
  --set global.rootCA.mountPath=/etc/ssl/certs \
  ...
```

When specified, the certificate is stored in `/etc/ssl/certs` on the container file system.

> [!IMPORTANT]
>
> This is existing functionality. It just needs to be tested to ensure the CA certificate is used by Terraform to validate TLS when downloading providers, modules, and configurations.

## Scenario 3 – Upgrading Radius

### User Story 9 – Upgrade Radius via CLI

***As a platform engineer, I need to upgrade Radius offline using the Radius CLI***

The platform engineer uses the `rad upgrade` command with similar arguments when installing.

```bash
rad upgrade kubernetes \
  --chart <PATH_TO_RADIUS_HELM_CHART_FROM_HELM_PULL> \
  --set image.repository=<OCI_REGISTRY_URL>/<REPOSITORY>
  --skip-contour-install
```

### User Story 10 – Upgrade Radius via Helm

***As a platform engineer, I need to upgrade Radius using Helm***

The platform engineer upgrades Radius using the same `helm upgrade` command as user story 4.

## Summary of changes

| Priority | Size | Description                                                  |
| -------- | ---- | ------------------------------------------------------------ |
| p0       | S    | Specifying the `image.repository` in the Radius Helm chart   |
| p0       | S    | Specifying `--skip-contour-install`                          |
| p0       | L    | Downloading the Terraform binary, validating the checksum, and installing Terraform via a new Terraform resource type |
| p0       | M    | Setting the Terraform CLI configuration based on the new Terraform resource |
| p1       | S    | Release notes now include a list of container images URLs with their tags and the URL for the CLI needed for the release |
| p1       | S    | Release notes now includes the `rad-bicep` binary under assets |
| p1       | S    | Documentation updates for installing the Radius CLI manually |
| p1       | M    | Upgrade the Terraform binary when the Terraform resource is modified |
| p1       | S    | New `rad resource show --output bicep` output option         |
| p2       | M    | Uninstall Terraform when the Terraform resource is deleted   |
| p2       | S    | Radius CLI releases are named `rad_cli_<KERNEL>_<ARCH>` rather than just `rad_<KERNEL>_<ARCH>` |

