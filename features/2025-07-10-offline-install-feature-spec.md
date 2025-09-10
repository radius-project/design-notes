# Offline Installation Feature Spec

[@zachcasper](https://github.com/zachcasper)

## Summary

Many large organizations have mature security practices where all software entering the organization's environment are controlled. Typically all software packages are scanned for security vulnerabilities prior to being stored internally. Access to non-authorized software packages are restricted. 

Radius must support these organizations' requirements by making it easy to install, configure, and upgrade Radius when not connected to the internet, or in *air gapped* environments.

### Goals

* Enable platform engineers to install and upgrade Radius while not connected to the internet
* Enable platform engineers to control what versions of all required software components are installed
* Modify Terraform from silently being installed in the background to explicitly installed by the platform engineer
* Increase the transparency of what software is required to run Radius

### Non-Goals (out of scope)

* Enhancing the Radius installation experience in other ways

## Inventory of software components

Radius makes the assumption that the installation environment has full access to the internet and installs many software components on behalf of the user. Below is an inventory:

| #    | Component                                                    | As-Is Distribution                                           | To-Be Online Distribution                                    | To-Be Offline Distribution                             |
| ---- | ------------------------------------------------------------ | ------------------------------------------------------------ | ------------------------------------------------------------ | ------------------------------------------------------ |
| 1a   | `rad` CLI binary                                             | [install.sh](https://raw.githubusercontent.com/radius-project/radius/main/deploy/install.sh) script | No change                                                    | Enhanced `install.sh`                                  |
| 1b   | `rad-bicep` binary                                           | `rad bicep download`                                         | Installed by `install.sh`                                    | Installed by `install.sh`                              |
| 1c   | `bicepconfig.json`                                           | `rad init`                                                   | Created by to `install.sh`                                   | Created by to `install.sh`                             |
| 2    | [Radius Helm chart](https://github.com/project-radius/radius/tree/main/deploy/Chart) | `rad init`, `rad install` or `helm repo add radius`          | No change                                                    | Chart manually pulled                                  |
| 3    | Radius container images (  `ucpd`, `controller`, `applications-rp`, `dynamic-rp`, `bicep`, `dashboard`, `pre-upgrade`) | Downloaded via the Radius Helm chart from GHCR               | No change                                                    | Manually imported into the user's private OCI registry |
| 4    | Contour container images (`contour`, `envoy`)                | Downloaded via the Radius Helm chart from Docker Hub         | Short-term: No change; Long-term: Removal of Contour from Radius install | Radius will have an option to not install Contour      |
| 5    | `radius` and `aws` Bicep extensions                          | Downloaded bi `rad-bicep` dynamically from ACR               | No change                                                    | Manually imported into the user's private OCI registry |
| 6    | `manifest-to-bicep-extension` NPM package                    | Remote code execution from [npmjs.com](https://www.npmjs.com/package/@radius-project/manifest-to-bicep-extension) | This package will be eliminated and implemented in the Go program | Same                                                   |
| 7    | Terraform binaries                                           | Dynamically installed by Applications RP from [hashicorp.com](https://releases.hashicorp.com/) | Explicitly installed into the Radius control plane by the platform engineer (not in scope for this document, see separate Terraform Settings feature spec) | Same                                                   |

## Scenario 1 – Installing Radius

### User Story 1 – Downloading releases

***As a platform engineer, I need to download the Radius release to an environment not connected to the internet***

The platform engineer consults the installation documentation which instructs them to download:

* Radius CLI (`install.sh`, `rad`, and `rad-bicep`)
* Radius Helm chart
* Radius container images
* Radius and AWS Bicep extensions

> [!NOTE]
>
> The Contour container images can optionally be included in the Radius container images. For this feature specification, it is assumed that Contour will not be installed. 

**Radius CLI**

The platform engineer or security engineer imports `install.sh`, `rad`, and `rad-bicep` into their software mirror (for example, Sonatype Nexus Repository mirror) or other file storage location.

> [!NOTE]
>
> The `rad-bicep` is a legacy of the Bicep fork and is no longer required. However, the `bicep` binary must still be installed. Work is needed to move to `bicep` and deprecate the `rad-bicep` binary. However, this is out of scope for this document.

**Helm chart**

If the organization has a Helm chart repository mirror such as Sonatype Nexus, the platform engineer or security engineer can load the Radius chart into the mirror using something similar to:

```bash
$ helm pull https://github.com/radius-project/radius/tree/main/deploy/Chart
$ helm repo add nexus http://<nexus_hostname>/repository/<nexus_repository_name>/ 
$ helm cm-push radius nexus
```

If the organization does not have a Helm chart repository mirror, the platform engineer or security engineer can download the Radius chart.

```bash
$ helm pull https://github.com/radius-project/radius/tree/main/deploy/Chart
```

**Container images**

The platform engineer or security engineer imports the Radius container images (`ucpd`, `controller`, `applications-rp`, `dynamic-rp`, `bicep`, `dashboard`, `pre-upgrade`) into their OCI registry. 

**Radius and AWS Bicep extensions**

The platform engineer or security engineer imports the Radius Bicep extensions into their OCI registry. 

### User Story 2 – Install CLI

***As a platform engineer or developer, I need to install the Radius CLI using pre-downloaded binaries***

The platform engineer or developer runs the installation script with additional parameters.

```bash
$ install.sh \
  --version <radius_version>
  --binary-install-source <path_to_rad_and_rad_bicep_cli> \
  --bicep-extensions-oci-registry <oci_registry_url>/<repository>
```

This script performs the following:

1. Gets the kernel type and system architecture (as-is)
2. Downloads the `rad` binary from Github (as-is) or the binary install source (new)
3. Gets the latest release information if the version is not specified (as-is)
4. Installs the `rad` binary in `/usr/local/bin` or `%LOCALAPPDATA%\radius` (as-is)
5. Installs the `rad-bicep` binary in `/usr/local/bin` or `%LOCALAPPDATA%\radius` (change from as-is)
6. Creates the `bicepconfig.json` in the user's home directory (new) using the `bicep-extensions-oci-registry`, if specified (new)

The key changes from today's `install.sh` include:

* Today, `rad-bicep` is installed via the `rad bicep download` command. This approach is problematic in that is is not obvious that Radius is downloading a new binary. The command also installs the `rad-bicep` binary in a different directory (`~/.rad/bin`) which is not ideal. The `install.sh` script is enhanced to install the `rad-bicep` binary side-by-side `rad` and the `rad bicep download` command is removed.
* Today, the `bicepconfig.json` file is created by `rad init`. This is problematic since offline installs will not use `rad init`. Also, the contents of `bicepconfig.json` file created by `rad init` is hard-coded. With this change, the `install.sh` script will create the `bicepconfig.json` file based on user input (`--bicep-extensions-oci-registry`).

> [!NOTE]
>
> Distributing the Bicep extension only supports ACR and a ZIP file on the filesystem today. There is future work planned to further optimize this user experience.

### User Story 3 – Install Radius via CLI

***As a platform engineer, I need to install Radius offline using the Radius CLI***

The platform engineer installs Radius using the CLI specifying the private OCI registry and to skip installing Contour.

```bash
rad install kubernetes \
  --set global.imageRegistry=<oci_registry_url>
  --set global.imageTag=<image_tag>
  --set global.imagePullSecrets=<kubernetes_secret_name>
  --skip-contour-install
```

> [!NOTE]
>
>  `rad init` is out of scope for offline installations. We expect `rad init` to continue to be optimized for the quick start scenario. 

The `global.imageRegistry` is the private OCI registry where the Radius container images were stored in the previous step.

The `global.imageTag` is the tag of the container images to optionally be used.

The `global.imagePullSecrets` is an array of Kubernetes secrets of type docker-registry to be used to pull the container images (the secret is created beforehand).

The `--skip-contour-install` does not install Contour. Therefore the Gateway resource types will not function until extensibility is implemented.

> [!CAUTION]
>
> `--skip-contour-install` is expected to be removed from Radius in the near future as part of the compute extensibility work. The design for the Gateway recipe no longer uses Contour and Contour is no longer installed by default.

If a Helm mirror has not been configured via `helm repo add`, the `--chart` argument can be used.

```bash
rad install kubernetes \
  --chart <path_to_radius_helm_chart_from_helm_pull> \
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
  --version <radius_version> \
  --wait --timeout 15m0s \
  --set global.imageRegistry=<oci_registry_url>
  --set global.imageTag=<image_tag>
  --set global.imagePullSecrets=<kubernetes_secret_name>
```

### User Story 5 – Specify trusted certificate authority

***As a platform engineer, I need provide my organization's certificate authority certificate when installing Radius***

The platform engineer installs Radius using the CLI and provides the organization's CA certificate.

```bash
rad install kubernetes \
  --set-file global.caCert=<ca>.pem \
  ...
```

When specified, the CA certificate is copied into the filesystem of the Radius containers such that when Radius connects to an internal URL over TLS, the CA is a trusted CA.

> [!IMPORTANT]
>
> Note that the prototype used the property  `global.appendRootCA.cert` while the property's property name is `global.caCert`.

## Scenario 2 – Upgrading Radius

### User Story 6 – Upgrade Radius via CLI

***As a platform engineer, I need to upgrade Radius offline using the Radius CLI***

The platform engineer uses the `rad upgrade` command with similar arguments when installing.

```bash
rad upgrade kubernetes \
  --chart <path_to_radius_helm_chart_from_helm_pull> \
  --set global.imageRegistry=<oci_registry_url>
  --set global.imageTag=<image_tag>
  --set global.imagePullSecrets=<kubernetes_secret_name>
  --skip-contour-install
```

### User Story 7 – Upgrade Radius via Helm

***As a platform engineer, I need to upgrade Radius using Helm***

The platform engineer upgrades Radius using the same `helm upgrade` command as user story 4.

## Summary of changes

| Priority | Size | Description                                                  |
| -------- | ---- | ------------------------------------------------------------ |
| p0       | S    | Specifying the `global.imageRegistry`, `global.imageTag`, and `global.imagePullSecrets` in the Radius Helm chart |
| p0       | S    | Specifying `--skip-contour-install`                          |
| p1       | S    | Documentation updates for installing the Radius CLI manually |
| p1       | M    | `install.sh` supports offline installation                   |
| p1       | M    | `install.sh` creates `bicepconfig.json`                      |
| p1       | M    | `install.sh` installs `rad-bicep`                            |
| p2       | M    | `rad bicep download` is removed                              |

