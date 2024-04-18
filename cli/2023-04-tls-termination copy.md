# TLS termination

* **Status**: Draft
* **Author**: Nithya Subramanian (@nithyatsu)

## Overview

When a Radius user wants to deploy an application, a few concepts come into picture

*	Resource group – Radius resource group mimic ARM resource group and are tied to RBAC. 
*	Environment – Radius environment is what the application uses. The env can have a set of recipes and is “in” a resource group itself, since it requires RBAC to make sure not everyone can edit it and for example, point a Production environment to use Sandbox recipes.
*	 Workspace – Radius workspace has a per user information about the kubernetes context/ cluster where the Radius services are hosted as well as the “default” environment and resource group  of the user.

There are a few points that can be improved in the way these concepts work together today:

*	Today, when we rad init,  we make a “default” resource group which creates a single resource group perception for radius users. However, we would want multiple resource groups for multiple copies of same application, based on the scenario. For instance, a cool-app deployment to production and the same cool-app deployment to staging would not need the same RBAC and hence have to be two copies in two resource groups. Would it be a better idea, to not create default group during rad init, and create/use a resource group based on what the users chooses as the application name and its environment [production, staging,[enter your own]]?


*	Today, if we want two copies of same app, we want to create two resource group, have an environment in each of those resource group, have the user explicitly use rad group switch and rad env switch or specify the group and env using flags, to make a desirable deployment. Would it be a better idea to, instead of expecting users specify instructions explicitly, print a default behavior which is intuitive and deploy the app if user confirms?


*	Since only ResourceGroup  is part of resource ID, if we deploy multiple application to same resource group (ex: we do this in our functional tests) we have to ensure the resource names across applications are all unique. This limitation has no reason to be present in real world and is a hinderance since the developer has to make sure to choose names that are not already taken across applications. Again, we could create a resource group based on user scenario per deployment using an agreed upon convention to solve this problem. 



## Terms and definitions

| Term | Definition |
|---|---|
| cert-manager | Cert-manager is a powerful and extensible X.509 certificate controller for Kubernetes|


## Objectives

### Goals

* Redesign User Experience with rad deploy to make the way it works with various radius concepts more intuitive

### Non-Goals

NA

### User scenarios

1. As a developer, I would like to not have to enter or set the group my      
application would be deployed to, every time explicitly. It would be useful if Radius picks the group name so that all related resources are grouped together and do not run into conflicts with resources from other  applications. I should be able to override this behavior as needed.
   
2. As a developer, I do not want to be responsibile for choosing names of     components within my application so that they dont conflict with any other applications that are deployed.

## Design

We want to redesign rad cli commands using a  convention over configuration approach for choosing which radius resource group to operate on. 
With this, we will not create a default resource group or environment in rad init. The workspace created as part of rad init will look in config.yaml as below: 

**Current config.yaml**

```
workspaces:
    default: default
    items:
        default:
            connection:
                context: radius-agent-aks
                kind: kubernetes
            environment: /planes/radius/local/resourceGroups/default/providers/Applications.Core/environments/default
            scope: /planes/radius/local/resourceGroups/default
```

**Proposed config.yaml**

```
workspaces:
    default: default
    items:
        default:
            connection:
                context: radius-agent-aks
                kind: kubernetes
            environment: /planes/radius/local/resourceGroups/default/providers/Applications.Core/environments/default
```

Based on this, all CLI commands will be impacted. 


### rad deploy 

To enable TLS termination in the Gateway, it is necessary to modify the current 
gateway resource, allowing users to define hostname(FQDN), minimum TLS protocol 
version, and TLS certificate related to the target routes. As the TLS 
certificate data is sensitive information, it must be handled securely as a 
secret by the Radius gateway application model.

One straightforward approach can directly model certificate data within 
Applications.Core/gateways like below. While this provides a more intuitive 
experience, it lacks extensibility for implementing the desired scenarios 
mentioned above.

```bicep
resource gateway 'Applications.Core/gateways@2022-03-15-privatepreview' = {
  name: 'tlsGateway'
  location: location
  properties: {
    application: app.id
    routes: [
      {
        path: '/'
        destination: tlsServiceContainerRoute.id
      }
    ]
    tls: {
      hostname: 'doc.radapp.dev'
      minimumProtocolVersion: '1.2' // Optional. Default: 1.2
      certificate: {
        kind: 'PEM'
        data: 'Base64 encoded private and public certificate. Only for Create or Update operation'
        // resource: 'Kubernetes secret resource name'
      }
    }
  }
}
```

This proposal aims to design an application model capable of supporting three 
discussed scenarios. This leads to a new application model to represent the 
certificate and allow it to be consumed by the Applications.Core/gateways 
resource.

In Kubernetes, a TLS certificate is modeled as a secret and is either 
referenced by the Ingress resource or mounted as a volume in a Pod. From Radius 
perspective, we have two options: either introduce a new Kubernetes-like secret 
resource type or model it as an Applications.Core/volumes resource type.

1. Applications.Core/secretStores
   * Pros
     - Represents TLS certificates, mapping to Kubernetes secret resource and cloud provider-specific secret stores.
     - Allows the use of secrets as environment variables in containers.
     - Compatible with Azure Key Vault, AWS Secrets Manager, and GCP Secret Manager.
   * Cons
     - Overlaps with the existing Azure Keyvault volume resource model.
2. Applications.Core/Volumes
   * Pros
     - Reuse of the existing resource type.
     - Reuse of the existing implementation for Azure KeyVault Certificate support.
   * Cons
     - Designed for Applications.Core/Containers resource, making it less suitable for Applications.Core/gateways resource reference.

We are inclined towards defining Applications.Core/secretStores to ensure 
future extensibility. Adopting this approach, Radius can support new 
requirements and scenarios more effectively, offering flexibility for managing 
sensitive secret data besides TLS certificate.

To reference Applications.Core/secretStores resource, we will introduce 
properties.tls.certificateFrom in gateways resource. In following example, when 
gateway is being deployed, gateways API ensures that referenced secret is 
available and then completes gateways deployment.

```bicep
resource gateway 'Applications.Core/gateways@2022-03-15-privatepreview' = {
  name: 'tlsGateway'
  location: location
  properties: {
    application: app.id
    routes: [
      {
        path: '/'
        destination: tlsServiceContainerRoute.id
      }
    ]
    tls: {
      hostname: 'doc.radapp.dev'
      minimumProtocolVersion: '1.2' // Optional. Default: 1.2
      certificateFrom: tlsSecret.id
    }
  }
}

resource tlsSecret 'Applications.Core/secretStores@2022-03-15-privatepreview' = {
  name: 'tls-secret'
  location: location
  properties: {
    application: app.id
    // ...
  }
}
```

### Secret Store

Applications.Core/secretStores is a new resource type designed to store and 
retrieve secret data such as secret text, keys, and certificates. This resource 
can be referenced or mounted by various radius resource models – for more 
information, see this issue. This proposal primarily addresses the modeling of 
secrets for TLS certificates; the other use cases will be considered 
separately. We propose the following vendor-agnostic model for secret store 
resource.

```bicep
resource tlsSecret 'Applications.Core/secretStores@2022-03-15-privatepreview' = {
  name: 'tlsSecret'
  location: location
  properties: {
    application: app.id
    resource: UCP_RESOURCE_ID  // Required only if valueFrom specifies referenced secret name.
    type: 'generic'            // Required. Secret type e.g. generic, certificate, etc
    data: {                    // Required to set secret key value.
      <key1>: {
        encoding: 'base64'     // Optional. the encoding of value. "raw, base64", default: raw
        value: "value1"        // Required. Value of secret key.
        // valueFrom: {
        //  name: <referenced_secret_key1_name>
        //  version: 1        // Optional. 
        // }
      }
      <key2>: {
        encoding: 'base64'
        value: "value2"
        // valueFrom: {
        //  name: <referenced_secret_key2_name>
        // }
      }
    }
  }
}
```

#### Properties

**properties.type**
This property defines the type of properties.data secrets:

| Type | Description | Kubernetes secret type |
|---|---|---|
| certificate | A certificate secret such as TLS certificate or self-signed certificate|kubernetes.io/tls |
| generic |	A generic secret that can store unstructured sensitive data.| Opaque |

**properties.resource**
This property references to the backing secret store resource. Radius can recognize what kind of secret store is referenced by Applications.Core/secretStores by parsing `providers` type from resource id because UCP resource id describes the kind of resource itself.

| Resource | Description |
|---|---|
| Unspecified(default) | Use the platform-dependent secret stores. E.g. Kubernetes secret store |
| Azure Keyvault | /subscriptions/{}/resourceGroups/{} /providers/Microsoft.KeyVault/vaults/{} |
| AWS secrets manager | /planes/aws/{}/accounts/{}/regions/{}/providers/AWS.SecretsManager/Secret/{} |

**properties.data**
This property is an object to represent key-value type secrets. Each secret value includes encoding, value, valueFrom properties to represent the secret value.

encoding defines the encoding type of value data. This supports the following types:
| Encoding type | Description |
|---|---|
| raw (default) | properties.data.key.value must be utf-8 string data type. Radius saves raw value data as is. |
| base64 | properties.data.key.value must be base64 encoded data. Radius checks whether value is base64-encoded or not. |

*properties.data.key.valueFrom* is an object to define the reference of the 
secret store *properties.resource* references. *valueFrom* includes only name 
property. In the future, we can add more metadata properties in *valueFrom*. 
The following bicep example defines <key1> secret referencing 
<secret_key1_name> key of *properties.resource* resource.

```bicep
...
      <key1>: {
        encoding: 'raw'
        valueFrom: {
          name: <secret_key1_name>   // Required. The secret name or key of properties.resource
          version: <secret version>  // Optional. The version of secret.
        }
      }
...
```


#### Kubernetes secret examples

##### Create new secret using Kubernetes secret store
The bicep template upserts new private key(tls.key) and signed certificate(tls.crt) 
for TLS certificate.

```bicep
resource tlsSecret 'Applications.Core/secretStores@2022-03-15-privatepreview' = {
  name: 'tls-secret'
  location: location
  properties: {
    application: app.id
    type: 'certificate'           // Required. type of secret e.g. generic, certificate, etc
    data: {
      tls.crt: {                  // Secret key name.
        encoding: 'base64'        // Optional. the encoding of value. "raw, base64", default: string
        value: tlsCrtData         // Required.
      }
      tls.key: {
        value: tlsKeyData2
      }
    }
  }
}
```

This resource model will be translated to the following Kubernetes secret resource.

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: tls-secret
type: kubernetes.io/tls
data:
  tls.crt: |
        MIIC2DCCAcCgAwIBAgIBATANBgkqh ...
  tls.key: |
        MIIEpgIBAAKCAQEA7yn3bRHQ5FHMQ ...
```

##### Reference the existing Kubernetes secret

To reference an existing Kubernetes secret, resource property should indicate 
the target Kubernetes secret resource. If the secret is stored in a different 
namespace, Applications.Core/secretStores can reference the secret across any 
namespace.

The following example references the existing ‘secret-name' secret in radius-system namespace.

```bicep
resource tlsSecretDirect 'Applications.Core/secretStores@2022-03-15-privatepreview' = {
  name: 'tls-secret-direct'
  location: location
  properties: {
    application: app.id
    type: 'certificate'           // Required. type of secret e.g. generic, certificate, etc
    resource: 'k8s-secret-name'
    data: {
      tls.crt: {                  // Secret key name.
        encoding: 'base64'        // Optional. the encoding of value. "raw, base64", default: raw
      }
      tls.key: {
        encoding: 'base64'        // Optional. the encoding of value. "raw, base64", default: raw
      }
    }
  }
}
```

If the bicep template includes core/Secret@1 resource type, Applications.Core/secretStores can reference this Kubernetes bicep type.

```bicep
resource tlsSecret 'Applications.Core/secretStores@2022-03-15-privatepreview' = {
  name: 'tls-secret'
  location: location
  properties: {
    application: app.id
    type: 'certificate'           // Required. type of secret e.g. generic, certificate, etc
    resource: tlsValue.metadata.name
    data: {
      tls.crt: {                  // Secret key name.
        encoding: 'base64'        // Optional. the encoding of value. "raw, base64", default: raw
      }
      tls.key: {                  // Secret key name.
        encoding: 'base64'        // Optional. the encoding of value. "raw, base64", default: raw
      }
    }
  }
}

resource tlsValue 'core/Secret@v1' existing = {
  metadata: {
    name: 'new-tls-cert'
  }
}
```

### API design
The following APIs needs to be defined to model Applications.Core/secretStores 
resource type:

* GET /{rootScope}/providers/Applications.Core/secretStores?api-version=2022-03-15-privatepreview
  - Description: list the resources of secretStore resource type
  - Type: ARM Synchronous

* GET /{rootScope}/providers/Applications.Core/secretStores/{name}?api-version=2022-03-15-privatepreview
  - Description: get the secret resource
  - Type: ARM Synchronous

* PUT/PATCH /{rootScope}/providers/Applications.Core/secretStores/{name}?api-version=2022-03-15-privatepreview
  - Description: create or update {name} secret resource.
  - Type: ARM Asynchronous
  - Request Body – new secret

```json
{
    "location": "global",
    "properties": {
        "application": "/planes/radius/local/resourceGroups/rg/providers/Applications.Core/applications/example-app",
        "type": "certificate",
        "data": {
             "tls.crt": {
                  "encoding": "base64",
                  "value: "secret"
             },
             "tls.key": {
                  "value": "secret"
             }
        }
    }
}
```

  - Response – new secret
    - 200 – value of each secret must be removed.

```json
{
    "location": "global",
    "properties": {
        "application": "/planes/radius/local/resourceGroups/rg/providers/Applications.Core/applications/example-app",
        "type": "certificate",
        "data": {
             "tls.crt": {
                  "encoding": "base64"
             },
             "tls.key": {
                  "encoding": "base64"
             }
        }
    }
}
```

    - 400 – invalid data key name. invalid encoding.

  -	Request Body – referenced resource

```json
{
    "id": "/planes/radius/local/resourceGroups/rg/providers/Applications.Core/secretStores/tls-secret",
    "name": "tls-secret",
    "type": "Applications.Core/secretStores",
    "location": "global",
    "properties": {
        "application": "/planes/radius/local/resourceGroups/rg/providers/Applications.Core/applications/example-app",
        "type": "resource",
        "resource": {
            "namespace": "radius-system", // Optional
            "name": "secret-name"
        }
    }
}
```

   - Response
     - 200 – OK
     - 400 – referenced resource does not exist.

* DELETE /{rootScope}/providers/Applications.Core/secretStores/{name}?api-version=2022-03-15-privatepreview
  - Description: Delete {name} secret resource
  - Type: ARM Asynchronous

* POST /{rootScope}/providers/Applications.Core/secretStores/{name}/listSecrets?api-version=2022-03-15-privatepreview
  - Description: Retrieve the secrets from {name} secret.
  - Type: ARM Synchronous
  - Response Body – 200 OK

```json
{
  "tls.crt": {
    "encoding": "base64",
    "value": "MIIC2DCCAcCgAwIBAgIBATANBgkqh..." 
  },
  "tls.key": {
    "encoding": "base64",
    "value": "MIIEpgIBAAKCAQEA7yn3bRHQ5FHMQ..."
  }
}
```

### Resource Type: Applications.Core/gateways

With the Secrets resource type now in place for representing certificates, the 
next step is to wire up Applications.Core/secretStores with Applications.Core/gateways 
to enable TLS termination in the following use cases:

* Bring your own TLS certificate using Kubernetes secret
* Use cert-manager for Let's Encrypt certificate authority
* Support Azure Key Vault (or any cloud vendor's managed secret store)

In practical scenarios, TLS certificates are consumed by Gateway/LB and 
Containers through referencing and volume mounting. Each application model 
describes the use cases using references and volumes to enable TLS termination.

#### Bring your own TLS certificate using Kubernetes secret

This is the use-case when operators bring their own TLS certificate a 
Kubernetes secret.

```bicep
@description('base64 encoded private and public certificate only for create or update')
@secure()
param pemCertificate string

resource gateway 'Applications.Core/gateways@2022-03-15-privatepreview' = {
  name: 'tlsGateway'
  location: location
  properties: {
    application: app.id
    routes: [
      {
        path: '/'
        destination: tlsServiceContainerRoute.id
      }
    ]
    tls: {
      hostname: 'example.com'
      minimumProtocolVersion: '1.2'
      certificateFrom: tlsSecret.id
    }
  }
}

resource tlsServiceContainerRoute 'Applications.Core/httpRoutes@2022-03-15-privatepreview' = {
  name: 'tls-service-route-pem'
  location: location
  properties: {
    application: app.id
    port: 80
  }
}

resource tlsSecret 'Applications.Core/secretStores@2022-03-15-privatepreview' = {
  name: 'tls-secret'
  location: location
  properties: {
    application: app.id
    type: 'certificate'           // Required. type of secret e.g. opaque, certificate, etc
    resource: 'secret-name'
    data: {
      tls.crt: {                  // Secret key name.
        encoding: 'base64'        // Optional. the encoding of value. "raw, base64", default: raw
      }
      tls.key: {                  // Secret key name.
        encoding: 'base64'        // Optional. the encoding of value. "raw, base64", default: raw
      }
    }
  }
}
```

#### Use cert-manager for Let's encrypt certificate authority

To enable Let's Encrypt within a Kubernetes environment, we will make use of 
cert-manager. The operator is required to create Issuer/ClusterIssuer and 
Certificate resources to integrate cert-manager with Contour. This setup 
enables the generation of multiple TLS certificates from the CA defined in the 
ClusterIssuer resource. Since Radius utilizes HTTPProxy instead of Ingress, it 
is necessary to manually create the Certificate resource. This step leads to 
the creation of an Ingress resource for handling HTTP-01 challenge during the 
short-term. The process results in creating doc-radapp-dev-tls TLS secret. 
Finally, HTTPProxy loads the TLS certificate, enabling TLS termination and 
ensuring secure communication.

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-staging
  namespace: cert-manager
spec:
  acme:
    email: admin@radapp.dev
    privateKeySecretRef:
      name: letsencrypt-staging
    server: https://acme-staging-v02.api.letsencrypt.org/directory
    solvers:
    - http01:
        ingress:
          class: contour
---
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: doc-radapp-dev
spec:
  secretName: doc-radapp-dev-tls
  commonName: doc.radapp.dev
  dnsNames:
  - doc.radapp.dev
  issuerRef:
    name: letsencrypt-staging
    kind: ClusterIssuer
---
apiVersion: projectcontour.io/v1
kind: HTTPProxy
metadata:
  name: doc-radapp-dev
spec:
  virtualhost:
    fqdn: doc.radapp.dev
    tls:
      secretName: doc-radapp-dev-tls
  routes:
  - services:
    - name: doc
      port: 8080
```

To utilize Let's Encrypt CA with cert-manager in Kubernetes, we will need to 
consider the following scenarios:

1. Radius creates or manages a new Issuer and Certificate resources dedicated 
   to Let's Encrypt.
2. Radius reuses existing Issuer/ClusterIssuer/Certificate resources, enabling 
   operators to utilize the existing TLS secret created by Certificate resource.

In the first scenario, a single Issuer can be referenced by multiple 
Certificate resources from Let's Encrypt. To accurately represent this 
relationship, Radius requires a new resource type specifically designed for the 
Issuer of cert-manager, defining the process for requesting TLS certificates. 
Issuer/Certificate resource includes a wide range of options and variations for 
a certification authority. This complexity presents a challenge in defining a 
platform-agnostic application model for Radius.

As a result, we aim to support the second scenario so an operator must 
configure ClusterIssuer and Certificate resource during cert-manager 
installation process. For instance, the operator should apply the following 
yaml manifest to configure AcMEv2 DNS-01 challenge validating the domain 
ownership using Azure DNS prior to deploying Applications.Core/gateways 
resource.

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-staging
spec:
  acme:
    server: https://acme-staging-v02.api.letsencrypt.org/directory
    email: admin@radapp.dev
    privateKeySecretRef:
      name: letsencrypt-staging
    solvers:
    - dns01:
        azureDNS:
          hostedZoneName: $AZURE_ZONE_NAME
          resourceGroupName: $AZURE_RESOURCE_GROUP
          subscriptionID: $AZURE_SUBSCRIPTION_ID
          environment: AzurePublicCloud
          managedIdentity:
            clientID: $IDENTITY_CLIENT_ID
---
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: doc-radapp-dev
spec:
  secretName: doc-radapp-dev-tls
  commonName: doc.radapp.dev
  dnsNames:
  - doc.radapp.dev
  issuerRef:
    name: letsencrypt-staging
    kind: ClusterIssuer
```

Next, once the operator deploys gateway resource along with docRadAppDevTLS 
resource, Radius waits until referenced secret is available and then creates or 
updates gateway resources to enable TLS termination.

```bicep
resource gateway 'Applications.Core/gateways@2022-03-15-privatepreview' = {
  name: 'tlsGateway'
  location: location
  properties: {
    application: app.id
    routes: [
      {
        path: '/'
        destination: tlsServiceContainerRoute.id
      }
    ]
    tls: {
      // sslPassthrough: false  // Optional. Default: false
      hostname: 'doc.radapp.dev'
      minimumProtocolVersion: '1.2'
      certificateFrom: docRadAppDevTLS.id
    }
  }
}

resource docRadAppDevTLS 'Applications.Core/secretStores@2022-03-15-privatepreview' = {
  name: 'doc-radapp-dev'
  location: location
  properties: {
    application: app.id
    type: 'certificate'           // Required. type of secret e.g. opaque, certificate, etc
    resource: 'doc-radapp-dev-tls'
    data: {
      tls.crt: {                  // Secret key name.
        encoding: 'base64'        // Optional. the encoding of value. "raw, base64", default: raw
      }
      tls.key: {                  // Secret key name.
        encoding: 'base64'        // Optional. the encoding of value. "raw, base64", default: raw
      }
    }
  }
}
```

#### Support Azure Keyvault

The existing Azure KeyVault support in Radius utilizes the secret store CSI 
driver to mount secret files to a container as a volume. That is, it is only 
available when a POD begins using the secret store CSI driver as a volume. This 
constraint prevents us from modeling the current Azure KeyVault volume as 
Applications.Core/secretStores, as secretStore resources should be referenced 
by non-container resources like Applications.Core/gateways. 

To address this limitation, we can either develop our own KeyVault 
implementation or utilize an open-source, external secret operator. Though 
still in beta, the external secret operator is actively maintained under the 
Linux Foundation and already supports multiple IAMs and various secret 
management systems, such as AWS Secrets Manager, HashiCorp Vault, GCP Secret 
Manager, and Azure KeyVault. By using the external secret operator, we can 
avoid reinventing the wheel and ensure future compatibility with multiple 
secret stores for Radius. Another benefit of using the external secret operator 
is its support for push secret mode, which enables us to implement value update 
features that are not supported by Secret Store CSI driver.

Here is the example to use workload identity and azure keyvault secret using 
external secret operator.

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: workload-identity-sa
  annotations:
    azure.workload.identity/client-id: 7d8cdf74-xxxx-xxxx-xxxx-274d963d358b
    azure.workload.identity/tenant-id: 5a02a20e-xxxx-xxxx-xxxx-0ad5b634c5d8
---
apiVersion: external-secrets.io/v1beta1
kind: SecretStore
metadata:
  name: azure-store
spec:
  provider:
    azurekv:
      authType: WorkloadIdentity
      vaultUrl: "https://xx-xxxx-xx.vault.azure.net"
      serviceAccountRef:
        name: workload-identity-sa
---
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: database-credentials
spec:
  refreshInterval: 1h
  secretStoreRef:
    kind: SecretStore
    name: azure-store

  target:
    name: database-credentials
    creationPolicy: Owner

  data:
  # name of the SECRET in the Azure KV (no prefix is by default a SECRET)
  - secretKey: database-username
    remoteRef:
      key: database-username

  # explicit type and name of secret in the Azure KV
  - secretKey: database-password
    remoteRef:
      key: secret/database-password

  # metadataPolicy to fetch all the tags in JSON format
  - secretKey: database-credentials-metadata
    remoteRef:
      key: database-credentials
      metadataPolicy: Fetch

  # metadataPolicy to fetch a specific tag which name must be in property
  - secretKey: database-credentials
    remoteRef:
      key: database-credentials
      metadataPolicy: Fetch
      property: environment

  # type/name of certificate in the Azure KV
  # raw value will be returned, use templating features for data processing
  - secretKey: db-client-cert
    remoteRef:
      key: cert/db-client-cert

  # type/name of the public key in the Azure KV
  # the key is returned PEM encoded
  - secretKey: encryption-pubkey
    remoteRef:
      key: key/encryption-pubkey
```

To utilize the external secret operator, we can adopt the same secret store 
resource model as shown below. Radius will examine the properties.resource to 
determine the secret store resource type. If it is an Azure KeyVault resource, 
Radius will manage the Azure workload identity, core/ServiceAccount, and 
external-secrets.io/SecretStore resources, just as it does for container and 
volume resources. Additionally, it will create an external-secrets.io/ExternalSecret 
resource from the metadata of Applications.Core/secretStores resource.

```bicep
resource gateway 'Applications.Core/gateways@2022-03-15-privatepreview' = {
  name: 'tlsGateway'
  location: location
  properties: {
    application: app.id
    routes: [
      {
        path: '/'
        destination: tlsServiceContainerRoute.id
      }
    ]
    tls: {
      // sslPassthrough: false  // Optional. Default: false
      hostname: 'doc.radapp.dev'
      minimumProtocolVersion: '1.2'
      certificateFrom: tlsCert.id
    }
  }
}

resource tlsCert 'Applications.Core/secretStores@2022-03-15-privatepreview' = {
  name: 'tls-cert'
  location: location
  properties: {
    application: app.id
    type: 'certificate'           // Required. type of secret e.g. generic, certificate, etc
    resource: 'azure-keyvault-resource-id'
    data: {
      tls.crt: {                  // Secret key name.
        encoding: 'base64'        // Optional. the encoding of value. "raw, base64", default: raw
        valueFrom: {
          name: 'secret/tls_publickey'
        }
      }
      tls.key: {                  // Secret key name.
        encoding: 'base64'        // Optional. the encoding of value. "raw, base64", default: raw
        valueFrom: {
          name: 'secret/tls_privatekey'
        }
      }
    }
  }
}
```

#### API Design

To implement the proposed application model, the existing versioned model and 
APIs should be updated:

* PUT /{rootScope}/providers/Applications.Core/gateways/{name}?api-version=2022-03-15-privatepreview
  - Description: Create or update {name} secret resource.
  - Type: ARM Asynchronous
  - Request Body

```json
{
    "location": "global",
    "properties": {
        "application": "/planes/radius/local/resourceGroups/rg/providers/Applications.Core/applications/example-app",
        "routes": [
            {
                "path": "/",
                "destination": "/planes/radius/local/resourcegroups/rg/providers/Applications.Core/httproutes/example-route"
            }
        ],
        "tls": {
            // "sslPassthrough": true,
            "hostname": "example.com",
            "minimumProtocolVersion": "1.2",
            "certificateFrom":  "/planes/radius/local/resourcegroups/rg/providers/Applications.Core/secretStores/tls-secret"
        }
    }
}
```

  - Response: API response must follow asynchronous API response contract.

```json
{
    "id": "/planes/radius/local/resourceGroups/rg/providers/Applications.Core/gateways/example-gw",
    "name": "example-gw",
    "type": "Applications.Core/gateways",
    "location": "global",
    "properties": {
        "provisioningState": "Succeeded",
        "application": "/planes/radius/local/resourceGroups/rg/providers/Applications.Core/applications/example-app",
        "routes": [
            {
                "path": "/",
                "destination": "/planes/radius/local/resourcegroups/rg/providers/Applications.Core/httproutes/example-route"
            }
        ],
        "tls": {
            // "sslPassthrough": true,
            "hostname": "example.com",
            "minimumProtocolVersion": "1.2",
            "certificateFrom":  "/planes/radius/local/resourcegroups/rg/providers/Applications.Core/secretStores/tls-secret"
        }
    }
}
```

> Radius deployment processor for gateways needs to wait until properties.tls.certificateFrom secret is available. If it is failed or timed out, then Radius will mark properties.provisioningState as Failed. The existing gateway configuration should not be impacted by this failure.

## Alternatives considered

### CSI driver support
The following diagram shows the application model to support Azure Keyvault CSI driver integration.

![CSI driver model](./2023-04-tls-termination/highlevel-csi.png)

## Test plan

* Unit-tests
* Functional tests
  - Create bring your own certificate scenario.
  - Create Azure KeyVault volume scenario.
  - Create Let's encrypt integration (it may be unnecessary to validate let's encrypt integration scenario because the functionality of radius is same as bring your own certificate scenario)

## Security

TLS private certificate is considered as a secret. As a result, Applications.
Core/secretStores should remove the secret data from the incoming request body 
and save it in the designated secret store, like Kubernetes secret, specified 
by properties.kind.


## Monitoring

1. Default incoming request instrumentation for Applications.Core/secretStores


## Development Plan

### Milestone 1 – estimate: 2 sprints

* Implement Applications.Core/secretStores for Kubernetes secret.
* Enable TLS termination in Applications.Core/gateways using Applications.Core/secretStores.
* Implement functional test for bring your own certificate scenario and let's encrypt
* Documentation

### Milestone 2 – estimate: 2 sprint

* Implement Applications.Core/secretStores using External secret operator for Azure Keyvault.
* Implement functional tests (Azure Keyvault)
* Documentation

### Milestone 3 – estimate: 2 sprints

* Support AWS IAM OIDC in Applications.Core/Environments.
* Implement Applications.Core/secretStores using External secret operator for AWS Secrets manager.
* Implement functional tests.
* Documentation

## Open issues

1. Secret Store CSI driver is more popular than External secret operator because it is under the official Kubernetes project. Do we still need to support azure keyvault volume for gateways?
1. Leveraging External secret operator is applicable only for Kubernetes platform. In the future, when we support non-kubernetes platform, such as EC2 or ACI, we need to have our own implementation for secret stores.

## Appendices

### ACME Challenge
Regarding challenge options for let's encrypt, the current cert-manager supports only HTTP-01 and DNS-01 challenge of ACME protocol.

1. HTTP-01
  * Pros:
    - It's easy to automate without extra knowledge about a domain's configuration.
    - It allows hosting providers to issue certificates for domains CNAMEd to them.
    - It works with off-the-shelf web servers.
  * Cons:
    - It doesn't work if your ISP blocks port 80 (this is rare, but some residential ISPs do this).
    - Let's Encrypt doesn't let you use this challenge to issue wildcard certificates.
    - If you have multiple web servers, you have to make sure the file is available on all of them
    - Operator needs to open port 80 with 443 HTTPS port redirection to challenge the domain validity (According to Let's encrypt base practice, opening 80 and 443 is recommended)
2. DNS-01
  * Pros:
    - Operator doesn't need to open port 80.
    - Operator can use this challenge to issue certificates containing wildcard domain names.
    - It works well even if operator has multiple web servers.
  * Cons:
    - Operator needs to give cert-manager permission to change DNS record of DNS server via ACME protocol. - Keeping credentials on your web server is risky.
    - DNS provider might not offer an API.
    - DNS API may not provide information on propagation times.

## References

| Title | Links |
|---|---|
| Let's encrypt challenge | https://letsencrypt.org/docs/challenge-types/, https://letsencrypt.org/docs/allow-port-80/ |
| Cert-manager HTTP-01 challenge support | https://cert-manager.io/docs/configuration/acme/http01/ |
| Kubernetes secret | https://kubernetes.io/docs/concepts/configuration/secret/ |
| ARM Async API specification | https://github.com/Azure/azure-resource-manager-rpc/blob/master/v1.0/async-api-reference.md#creating-or-updating-resources-asynchronously |
