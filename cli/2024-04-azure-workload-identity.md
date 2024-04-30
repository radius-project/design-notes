# Radius Azure Workload Identity Support

* **Status**: In Review
* **Author**: Will Smith (@willdavsmith)

## Overview

A software workload such as a container-based application, service or script needs an identity to authenticate, access, and communicate with services that are distributed across different platforms and/or cloud providers. For workloads running outside of Azure, the workload need a credential (secret or certificate) to access Azure resources. These credentials pose a security threat and must be stored and rotated regularly.  

Radius supports service principal identity to configure the Azure provider to deploy and manage Azure resources. However, this adds an extra task for operators to rotate the credentials for the identity. A workload identity or federated identity helps us avoid this maintenance challenge of manually managing the credentials and eliminates the risk of exposing secrets or having certificates expire. 

The goal of the scenario is to enable infrastructure operators to configure workload identity support for the Azure provider in Radius to deploy and manage Azure resources.

## Terms and definitions

1. **Service principal** is the local representation of an application object in a tenant or directory. It's the identity of the application instance. Service principals define application access and resources the application accesses.

1. **Managed Identities** are a special type of service principal that eliminate the need for users to manage credentials by providing an identity for the Azure resource in Azure AD and using it to obtain Azure Active Directory (Azure AD) tokens. These can be used only with the scope of Azure ecosystem. There are two types of managed identities:
    * **System assigned managed identities**: This managed identity is tied to the lifecycle of the Azure resource and are used to access a particular resource.
    * **User assigned managed identities**: This managed identity is created as a separate resource or entity in Azure and are assigned to multiple Azure resources.

1. **Workload identities** use the same service principal or managed identity to trust tokens from an external identity provider (IdP). Once that trust relationship is created, your external software workload exchanges trusted tokens from the external IdP for access tokens from the Microsoft Identity platform. Your software workload uses that access token to access the protected Azure resources to which the workload has been granted access. This eliminates the burden of managing and rotating the secrets.

1. **Azure AD Workload Identity mutating admission webhook (AzWI mutating admission webhook)** is a pod on the Kubernetes cluster that intercepts the pod creation request and injects the signed service account token into the pod. This signed service account token is used by the Azure provider to authenticate with Azure resources.


## Objectives

> **Issue Reference: https://github.com/radius-project/radius/issues/7308**

### Goals

* Radius users can configure Azure provider to use Workload Identity for authentication.
* Workload Identity can be configured via interactive experience.
* Workload Identity can be configured manually.
* Radius users can deploy and manage Azure resources without using service principal credentials.

### Non-goals

* Azure Managed Identity support
* AWS Workload Identity (IRSA) support

### User scenarios

#### User story 1: As an infrastructure operator, I can configure the Radius Azure Provider to use Workload Identity for authentication via interactive experience.

1. Install the `rad` CLI.

1. run `rad init --full` to begin the interactive experience
    ```
    rad init --full
    ```

1. Enter the environment name, namespace and select "yes" to add a cloud provider and select "Azure" to add an Azure cloud provider.
    ```
    Select your cloud provider:
    > 1. Azure
      2. AWS
      3. [back]
    ```

1. Enter or use the Azure subscription and resource group name. Select  "Workload Identity" for the Azure cloud provider.
    ```
    Select the identity for the Azure cloud provider 
    1. Service principal                               
    > 2. Workload Identity    
    ```

1. Enter the Entra ID Application Client ID.
    ```
    Please follow the guidance at aka.ms/rad-workload-identity to set up workload identity for Radius.

    Enter the Client ID of the Entra ID Application that you have created for Radius.

    > Enter the Client ID:
    ```

Radius is now configured to use Workload Identity for authentication with Azure.

#### User story 2: As an infrastructure operator, I can configure the Radius Azure Provider to use Workload Identity for authentication manually.

1. Install the `rad` CLI.

1. Install Radius using `rad install kubernetes`
    ```
    rad install kubernetes --set global.azureWorkloadIdentity.enabled=true
    ```

1. Create a Radius environment and add an Azure cloud provider.
    ```
    rad group create default
    rad env create default
    rad env update default --azure-subscription-id <subscription-id> --azure-resource-group <resource-group> 
    ```

1. Register the Entra ID Application using `rad credential register azure`.
    ```
    rad credential register azure --client-id <client-id>
    ```

Radius is now configured to use Workload Identity for authentication with Azure.

## User Experience

We will be adding a new option to the `rad init --full` command to allow the user to choose between service principal and Azure Workload Identity.

We will be adding a new option to the Radius Helm chart to allow the user to enable Azure Workload Identity during installation via `rad install kubernetes --set global.azureWorkloadIdentity.enabled=true`.

**Sample Input:**

See above

**Sample Output:**

See above

**Sample Recipe Contract:**

n/a

## Design

### High Level Design

During installation, if the user enables the Azure Workload Identity feature, the Radius control plane (bicep-de, ucp, applications-rp) pods will be annotated with the `azure.workload.identity/use: "true"` label. The AzWI mutating admission webhook will see this label on the pods and will project a signed service account token to each of the pods. The signed service account token and the credential registered by the user will be used by Radius to authenticate with Azure.

Please see https://azure.github.io/azure-workload-identity/docs/concepts.html for more information on how the AzWI mutating admission webhook works.

### Architecture Diagram

![Architecture Diagram](./2024-04-azure-workload-identity/architecture.png)

### Detailed Design

#### Advantages (of each option considered)

n/a

#### Disadvantages (of each option considered)

n/a

#### Proposed Option

n/a

### API design

We should not need to make any functional changes to the Radius API design for this feature as long as we re-use the `ClientID` field for Azure credentials. We should update the documentation for the API and maybe some of the field names to represent that the `ClientID` field can be used for both service principal and Azure Workload Identity. We should also ensure that the `ClientSecret` field is not marked as required anywhere in the API spec since it is not required for Azure Workload Identity.


### CLI Design

`rad init --full` will need to be updated to ask the user to choose between service principal auth and Azure Workload Identity.

We will need to update the `rad credential register azure` command to not require a client secret when registering the Entra ID Application Client ID.

We will need to update the Radius Helm chart to allow the user to enable Azure Workload Identity during installation with the `global.azureWorkloadIdentity.enabled` value. We will need to update the installation flow (for `rad init` and `rad install kubernetes`) to annotate the Radius control plane pods with the `azure.workload.identity/use: "true"` label if the user sets this value in the Helm chart.

### Implementation Details

#### UCP

UCP should be updated to read the Entra ID Application Client ID from the credential store. It should then authenticate with Azure using this Client ID and the signed service account token.

The credential storage and management code will need to be updated to store a ClientID without a ClientSecret.

#### Bicep

n/a

#### Deployment Engine

Deployment Engine should be updated to read the Entra ID Application Client ID from the credential store. It should then authenticate with Azure using this Client ID and the signed service account token.

#### Core RP

Core RP should be updated to read the Entra ID Application Client ID from the credential store. It should then authenticate with Azure using this Client ID and the signed service account token.

#### Portable Resources / Recipes RP

n/a

### Error Handling

#### AzWI mutating admission webhook is not installed in the cluster.
* At Radius install time if Radius detects that the AzWI mutating admission webhook is not installed on the cluster, we should return an error to the user: "Azure Workload Identity mutating admission webhook is not installed in the cluster. Please follow the guidance at aka.ms/rad-workload-identity to set up workload identity for Radius."

## Test plan

We should add a functional test that installs Radius with Azure Workload Identity enabled and then deploys an Azure resource to ensure that the resource is deployed successfully. We will need to maintain an Entra ID Application in Azure for testing purposes.

## Security

We will not be storing any secrets in Radius. The user will have to register the Entra ID Application Client ID with Radius using the `rad credential register azure` command. This command will not require a client secret. We are using a project and process that is a well-known way to authenticate with Azure via Azure Workload Identity. The user will have the same security concerns as today - their Entra ID Application should have least-privileges.

## Compatibility

There should be no breaking changes as a result of this design.

To use Azure Workload Identity with Radius, the user will have to re-install the Radius control plane.

## Monitoring and Logging

We will have the same monitoring and logging as today. We will not be adding any new instrumentation.

## Development plan

* Create POC for Radius + Azure Workload Identity (1 engineer, 0.5 sprint)
* Create and review technical design (1 engineer, 0.5 sprint)
* Implement CLI and Helm chart changes (1 engineer, 1 sprint)
* Implement changes in UCP, Bicep, and Applications RP (1 engineer, 0.5 sprint)
* End-to-end testing and documentation (1 engineer, 0.5 sprint)

Total: 3 sprints of engineering effort

## Open Questions

Q: Is it a bad idea (confusing) to "overload" the `rad credential register azure` command to register the Entra ID Application Client ID for Azure Workload Identity? We could change the registration to use `--workload-identity-client-id` instead of re-using `--client-id`.

A: TODO

## Alternatives considered 

n/a

## Design Review Notes

<!--
Update this section with the decisions made during the design review meeting. This should be updated before the design is merged.
-->

TODO
