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

## Inventory of software installations

Radius makes the assumption that the installation environment has full access to the internet and liberally installs software components on behalf of the user. Below is an inventory:

Binaries distributed via GitHub:

* `rad`
* `rad-bicep`

These binaries will be downloaded and installed manually.

The Radius Helm chart 

* `oci://ghcr.io/radius-project/helm-chart`

This chart will be downloaded using `helm pull` and stored either in a Helm repository mirror or on the file system.

Container images distributed via GHCR:

* `ucpd`
* `controller`
* `applications-rp`
* `dynamic-rp`
* `bicep`
* `dashboard`

These images will be imported into the user's private OCI registry.

Container images distributed via Docker Hub

* `contour`
* `envoy`

Radius will have an option to not install Contour.

Bicep extensions distributed via ACR:

* `biceptypes.azurecr.io/radius`
* `biceptypes.azurecr.io/aws`

These images will be imported into the user's private OCI registry.

NPM packages distributed via npmjs.com:

* https://www.npmjs.com/package/@radius-project/manifest-to-bicep-extension

This package will be eliminated and implemented in the Go program.

Third-party binaries

* Terraform via https://releases.hashicorp.com/

Radius will offer an installation option to install the Terraform binary from the user's location.

## Scenario 1 – Installing Radius

### User Story 1 – Downloading releases

***As a platform engineer, I need to download the Radius release to an environment not connected to the internet***

The platform engineer consults the release notes for the location of binaries. They will need to download:

* Radius CLI binaries
* Helm chart
* Container images
* Bicep extensions

**Radius CLI binaries**

The platform engineer or security engineer first imports the Radius CLI into their software mirror (for example, Sonatype Nexus Repository mirror). The user consults the Radius release notes for the list of assets. They see a list of CLI binaries clearly named CLI to distinguish from the container images:

The CLI will be renamed from  `rad_<kernel>_<arch>` to  `rad_cli_<kernel>_<arch>`:

| Binary                  | URL                                                          | Digest      |
| ----------------------- | ------------------------------------------------------------ | ----------- |
| `rad_cli_darwin_amd64`  | https://github.com/radius-project/radius/releases/download/<release>/rad_cli_darwin_amd64 | `sha256...` |
| `rad_cli_darwin_arm64`  | https://github.com/radius-project/radius/releases/download/<release>/rad_cli_darwin_arm64 | `sha256...` |
| `rad_cli_linux_amd64`   | https://github.com/radius-project/radius/releases/download/<release>/rad_cli_linux_amd64 | `sha256...` |
| `rad_cli_linux_arm64`   | https://github.com/radius-project/radius/releases/download/<release>/rad_cli_linux_amd64 | `sha256...` |
| `rad_cli_linux_arm`     | https://github.com/radius-project/radius/releases/download/<release>/rad_cli_linux_amd64 | `sha256...` |
| `rad_cli_windows_arm64` | https://github.com/radius-project/radius/releases/download/<release>/rad_cli_windows_amd64 | `sha256...` |

The `rad-bicep` binary will be listed on the release notes:

| Binary                    | URL                                                          | Digest      |
| ------------------------- | ------------------------------------------------------------ | ----------- |
| `rad-bicep_darwin_amd64`  | https://github.com/radius-project/radius/releases/download/<release>/rad-bicep_darwin_amd64 | `sha256...` |
| `rad-bicep_darwin_arm64`  | https://github.com/radius-project/radius/releases/download/<release>/rad-bicepdarwin_amd64 | `sha256...` |
| `rad-bicep_linux_amd64`   | https://github.com/radius-project/radius/releases/download/<release>/rad-biceplinux_amd64 | `sha256...` |
| `rad-bicep_linux_arm64`   | https://github.com/radius-project/radius/releases/download/<release>/rad-bicep_linux_amd64 | `sha256...` |
| `rad-bicep_linux_arm`     | https://github.com/radius-project/radius/releases/download/<release>/rad-biceplinux_amd64 | `sha256...` |
| `rad-bicep_windows_arm64` | https://github.com/radius-project/radius/releases/download/<release>/rad-bicep_cli_windows_amd64 | `sha256...` |

**Helm chart**

If the organization has a Helm chart repository mirror such as Sonatype Nexus, the platform engineer or security engineer can load the Radius chart into the mirror using something similar to:

```bash
helm pull https://github.com/radius-project/radius/tree/main/deploy/Chart
helm repo add nexus http://<nexus_host>/repository/<nexus_repository_name>/ 
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
  --name <acr_registry_name> 
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
  --name <acr_registry_name> 
  --source biceptypes.azurecr.io/radius:0.48
  --image radius:0.48
# Import AWS Bicep extension (optionally)
az acr import \
  --name <acr_registry_name>
  --source biceptypes.azurecr.io/aws:0.48
  --image aws:0.48
```

### User Story 2 – Install CLI

***As a platform engineer, I need to install the Radius CLI***

> [!CAUTION]
>
> **TODO**: Should installation be done via package managers like Homebrew and Winget? Nexus does not have mirrors for these types. How do enterprises mirror Mac and Windows packages?

The platform engineer downloads `rad` and `rad-bicep` and stores in `/usr/local/bin/rad` and confirms that directory is in their $PATH.

The platform engineer creates a `bicepconfig.json` file with these contents:

```json
{
	"experimentalFeaturesEnabled": {
		"extensibility": true
	},
	"extensions": {
		"radius": "br:<my_oci_registry>/<repository>/radius:0.48",
		"aws": "br:<my_oci_registry>/<repository>/aws:latest",
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
  --set image.repository=<my_oci_registry>/<repository>
  --set terraform.url="https://<mirror>/terraform_1.5.7_linux_amd64.zip"
  --set terraform.checksum="sha256:66bd0ab2d88394e30c618cb60e9ebd5812689d70a79ed0d45d"
  --skip-contour-install
```

> [!NOTE]
>
>  `rad init` will not support offline installations.

The `image.repository` is the location of the Radius container images mirror.

The `terraform.url` is the URL to download the Terraform binary and the `terraform.checksum` is the hash of the binary provided by Hashicorp. When specified, Radius will download the binary, validate the hash value, and install the binary in the Applications RP container.

The `--skip-contour-install` does not install Contour. Therefore the Gateway resource types will not function until extensibility is implemented.

> [!NOTE]
>
> This differs from the [current PR](https://github.com/radius-project/radius/pull/9958/files) in several ways:
>
> * `image.repository` is added as a one time configuration versus the per-image options today (this is a common pattern and naming for Helm charts)
> * `global.terraform.enabled=true` is not necessary since it should always be true
> * `global.terraform.downloadUrl` is renamed `terraform.url` for increased clarity (`global` is not relevant for the platform engineer in this context)
> * `terraform.checksum` is added for validating the binary

If a Helm mirror has not been configured via `helm repo add`, the `--chart` argument can be used.

```bash
rad install kubernetes \
  --chart <path_to_Radius_chart_from_helm_pull> \
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
  --set image.repository=<my_oci_registry>/<repository> \
  --set terraform.url=https://<mirror>/terraform_1.5.7_linux_amd64.zip \
  --set terraform.checksum=sha256:66bd0ab2d88394e30c618cb60e9ebd5812689d70a79ed0d45d
```

## Scenario 2 – Configure Radius for offline use

### User Story 5 – Specify trusted certificate authority

***As a platform engineer, I need provide my organization's certificate authority certificate when installing Radius***

The platform engineer installs Radius using the CLI with override for the various settings.

```bash
rad install kubernetes \
  --set-file global.rootCA.cert=<root-ca.crt> \
  --set global.rootCA.mountPath=/etc/ssl/certs \
  ...
```

When specified, the certificate is stored in `/etc/ssl/certs` on the container file system.

> [!IMPORTANT]
>
> This is existing functionality. It just needs to be tested to ensure the CA certificate is used by Terraform to validate TLS when downloading providers, modules, and configurations.

### User Story 6 – Configure Terraform provider mirror

***As a platform engineer, I need to configure Terraform to use my organization's Terraform provider mirror***

The platform engineer specifies the Terraform `provider_installation` in the environment's `recipeConfig.terraform` property.

```yaml
param token string

resource env 'Applications.Core/environments@2023-10-01-preview' = {
  name: 'dev'
  properties: {
    compute: {
      kind: 'kubernetes'
      namespace: 'dev'
    }
    recipeConfig: {
      terraform: {
        provider_installation: {
          network_mirror: {
            path: 'https://<terraform_provider_mirror>/'
            include: ["*"]
          }
        }
        credentials: {
          '<terraform_provider_mirror>': {
            secret: providerSecret.id
          }
        }
        registry: {
          mirror: <TERRAFORM_PROVIDER_MIRROR>
        }
      }
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

The structure of the `provder_installation` mirrors the [Terraform CLI configuration](https://developer.hashicorp.com/terraform/cli/config/config-file#provider-installation). Similarly, the `credentials` also mirrors the [same configuration](https://developer.hashicorp.com/terraform/cli/config/config-file#credentials-1).

> [!CAUTION]
>
> This schema is different from the air-gapped branch implementation. The main implementation should follow this schema not the current implementation.

## Scenario 3 – Upgrading Radius

### User Story 7 – Upgrade Radius via CLI

***As a platform engineer, I need to upgrade Radius offline using the Radius CLI***

The platform engineer uses the `rad upgrade` command with similar arguments when installing.

```bash
rad upgrade kubernetes \
  --chart <path_to_Radius_chart_from_helm_pull> \
  --set image.repository=<my_oci_registry>/<repository>
  --set terraform.url="https://<mirror>/terraform_1.5.7_linux_amd64.zip"
  --set terraform.checksum="sha256:66bd0ab2d88394e30c618cb60e9ebd5812689d70a79ed0d45d"
  --skip-contour-install
```

### User Story 8 – Upgrade Radius via Helm

***As a platform engineer, I need to upgrade Radius using Helm***

The platform engineer upgrades Radius using the same `helm upgrade` command as user story 4.

## Summary of changes

| Priority | Size | Description                                                  |
| -------- | ---- | ------------------------------------------------------------ |
| p0       | S    | Specifying the `image.repository` in the Radius Helm chart   |
| p0       | M    | Specifying the Terraform binary via `terraform.url` and `terraform.checksum` in the Radius Helm chart tells Radius to download the binary, validate the checksum, and install into Applications RP container |
| p0       | M    | Specifying the `provider_installation` on the `recipeConfig.terraform` property of the environment resource |
| p0       | S    | Specifying `--skip-contour-install`                          |
| p1       | S    | Release notes now include a list of container images URLs with their tags and the URL for the CLI needed for the release |
| p1       | S    | Release notes now includes the `rad-bicep` binary under assets |
| p1       | S    | Documentation updates for installing the Radius CLI manually |
| p2       | S    | Radius CLI releases are named `rad_cli_<kernel>_<arch>` rather than just `rad_<kernel>_<arch>` |
| p2       | M    | Radius CLI is installable via common package managers        |

