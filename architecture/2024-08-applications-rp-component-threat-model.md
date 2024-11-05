# Radius Applications RP Component Threat Model

- **Author**: @nithyatsu

## Overview

This document provides a threat model for the Radius Applications RP component. It identifies potential security threats to this critical part of Radius and suggests possible mitigations. The document includes an analysis of the system, its assets, identified threats, and recommended security measures to protect the system.

The Applications RP component is responsible for managing applications and their resources. It communicates with UCP, Deployment Engine and Controller components to achieve this. 

## Terms and Definitions

| Term                  | Definition                                                                                                                                                                                  |
| --------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
|RP                 | Resource Provider                                                 |
| UCP                 | Universal Control Plane for Radius                                                                                                                                                   |
## System Description

Applications RP is a Radius service that acts as resource provider for application and its resources. It communicates over HTTP. The RP has a Datastore for storing Radius data, Message Queue for processing asynchronous request and a Secret Store for storing sensitive information such as certficates. All these are configurable components and support multiple implementations. Users and Clients cannot directly communicate with Applications RP. They instead communicate with UCP. UCP forwards relevant requests to Applications RP. Applications may have Kubernetes resources and cloud resources. Applications RP manages these Kubernetes resources on the user's behalf. This may launch user application's code on the same cluster as Radius, or a different cluster. it also has access to user's cloud credentials and manages user's cloud resources. Applications RP can invoke *recipes* which are bicep or terraform code. These recipes are used to deploy application infrastructure components like databases.  

### Architecture

Application RP recieves HTTP requests from UCP. It does not interface directly with user/ cli. These requests have untrusted json payloads. RP validates these payloads before consuming them. 

The RP consists of four types of resource providers for managing various types of resources in an application. `Applications.Core` resource provider manages core application resources such as application, environment, container and gateways. `Applications.Dapr` resource provider manages all dapr resources that are deployed as part of application. `Applications.Datastore` resource provider supports provisioning SQL database, Mongo DB and Redis Cache.
`Applications.Messaging` resources provider manages queues such as Rabbit MQ.

Applications RP has a key sub component `Recipe Engine` to execute `recipes`. 
`Recipes` are Bicep or Terraform code supplied by user that is used to deploy infrastructure components on Azure and AWS. The Bicep recipes are fetched from OCI compliant registries. Terraform recipes are public modules and fetched from internet too. 

In order to execute Terraform recipes, Applications RP installs latest Terraform. It also mounts an empty directory `/terraform` into Applications RP pod. It uses this directory for executing terraform recipes using the installed executable. The output resources generated from terraform module are converted to Radius output resources and stored in our datastore. 

In order to deploy bicp recipes, Applications RP sends a request to UCP, which in turn forwards it to Deployment Engine. 

Applications RP also allows users to create their own recipes and use them to provision their infrastructure. 

The RP can create kubernetes resources and manage them on behalf of the user. It can, for example, create a container based on the image provided by the user, which can in turn execute arbitrary code, and create other resources in the cluster as well as in AWS and Azure. 

Applications RP has user's AWS / Azure credentials so that it can deploy and manage the cloud resources. The credentials are available as a kubernetes secret. While the credentials are registered and stored as secrets using an API, they are not available for retrieval or update through API calls. 

The RP uses a queue to process requests asyncronously. Information about resources that are deployed / being deployed is stored in a datastore. 

Below is a high level overview of various sub components in Applications RP
![Applications RP](2024-10-applications-rp-threat-model/apprp.png) 

### Implementation Details

`Applications.Core` resource provider manages core application resources such as application, environment, container and gateways. Applications RP managed containers can use Azure Key Vault for storing secrets, TLS keys, and certificates. To deploy gateways, the RP uses contour as ingress controller. These gateways support TLS termination. In order to do this, Applications RP stores sensitive TLS information as secret using Secret Store.
Applications.Core RP implements Secret stores using kubernetes as secret provider. 

`Applications.Dapr` resource provider manages all dapr resources that are deployed as part of application. These include dapr state store, dapr secret store, dapr pubsub and dapr configuartion store.

`Applications.Datastore` resource provider supports provisioning SQL database, Mongo DB and Redis Cache. The secrets associated with these resources are stored in plain text in the corresponding tracked resource kubernetes object.

`Applications.Messaging` resources provider manages queues such as Rabbit MQ.


*Applications RP deploying a cloud Resource*

For deploying an AWS / Azure resource, Applications RP sends a request to UCP. It stores neccsary provider configs in environment objects and provides them to UCP. The provider config for AWS is account id and region. The provider config for Azure is subscription ID and resource group. The credentials required for deployment are managed by UCP. 

*Applications RP deploying a Kubernetes Resource* 

For deploying a kubernetes resource, Applications RP sends a request to Controller. 

*Applications RP deploying a Recipe*

Recipe Engine sub component in Applications RP is responsible for managing and deploying recipes. It sends a request to controller to validate custom recipes. 
It sends a request to UCP to deploy bicep recipes. It fetches the recipes to be deployed from the public module link for terraform recipe or the OCI registry for bicep recipe. It works with AWS/Azure to deploy terraform recipes.

`/radius/deploy/Chart/templates/rp/configmaps.yaml` has the settings for all the providers used by Applications RP. 
```
storageProvider:
      provider: "apiserver"
      apiserver:
        context: ""
        namespace: "radius-system"
    queueProvider:
      provider: "apiserver"
      name: "radius"
      apiserver:
        context: ""
        namespace: "radius-system"
    metricsProvider:
      prometheus:
        enabled: true
        path: "/metrics"
        port: 9090
    profilerProvider:
      enabled: true
      port: 6060
    secretProvider:
      provider: kubernetes
    server:
      host: "0.0.0.0"
      port: 5443
    workerServer:
      maxOperationConcurrency: 10
      maxOperationRetryCount: 2
    ucp:
      kind: kubernetes
    logging:
      level: "info"
      json: true
    {{- if and .Values.global.zipkin .Values.global.zipkin.url }}
    tracerProvider:
      serviceName: "applications.core"
      zipkin: 
        url: {{ .Values.global.zipkin.url }}
    {{- end }}
```

#### Use of Cryptography

#### Storage of secrets

Applications RP stores secrets for rendering some resources. For instance TLS termination in gateways needs storing TLS cert and key. For these cases, Applications RP uses kubernetes secrets. 

#### Data Serialization / Formats 

### Clients

**UCP** is the only client of Applications RP. It communicates with Applications RP for deploying various Radius Application resources. 

All communications use mTLS.

## Trust Boundaries

We have a few different trust boundaries for the Applications RP component:

- **Kubernetes Cluster**: The overall environment where the Applications RP  component operates and serves clients.
- **Namespaces within the Cluster**: Logical partitions within the cluster to separate and isolate resources and workloads. 

The Applications RP component lives inside the `radius-system` namespace in the Kubernetes cluster where it is installed. UCP and Controller also reside within the same namespace. Namespaces within Kubernetes can help set Role-Based Access Control (RBAC) policies.

Applications RP component communicates with Kube API server through Kube Controller for saving Radius resources and for queuing async operations. The API Server and Kube Controller run in the `kube-system` namespace within the cluster. 

## Assumptions

This threat model assumes that:

1. The Radius installation is not tampered with.
2. The Kubernetes cluster that Radius is installed on is not compromised.
3. It is the responsibility of the Kubernetes cluster to authenticate users. Administrators and users with sufficient privileges can perform their required tasks. Radius cannot prevent actions taken by an administrator.

## Data Flow      

### Diagram

![Applications RP via Microsoft Threat Modeling Tool](2024-10-applications-rp-threat-model/apprp-dataflow.png)

Below are the key points associated with data flow:
1. Applications RP receives request to dpeloy resources from UCP and sends back appropriate response.
2. Applications RP communicates with Controller for managing kubernetes resources and validating custom recipes.
3. Applications RP requests UCP to deploy bicep recipes.
4.  Application RP (terraform provider) requests AWS/ Azure to deploy resources for terraform recipes.                   
5.  Application RP uses API server through Kubernetes Controller to save Radius resources and Async Operations ( APIServer is used as datastore and Queue).
6.  Application RP fetches recipes from OCI registries and public terraform modules.


### Threats

#### Spoofing Applications RP could cause information disclosure, DDoS and misuse of cloud resources.

**Description:** If a malicious actor can spoof Applications RP, requests from UCP to be sent to the malicious actor. The malicious actor can also send requests to UCP such as fetching credentials. 

**Impact:** All data sent from UCP to Applications RP will be available to the malicious actor, such as payloads of resources. Applications RP would also be able to retreive credentials through UCP and make it available to malicious actor. The credentials can be used to misuse Az/ AWS resources. It might also request controller to update Environment recipes to use an outdated/ vulnerable version of the resource.

**Mitigations:**

Spoofing Applications RP, tampering with the applications rp code and configuration would require access to modify the `radius-system` namespace. Our threat model assumes that the operator has limited access to the `radius-system` namespace using Kubernetes' existing RBAC mechanism.

**Status:** All mitigations listed are currently active. Operators are expected to secure their cluster and limit access to the `radius-system` namespace.

#### Malicious user can make arbitrary requests to Applications RP API. 

**Impact**

This can cause a load of various Radius services and result in DDoS.

**Mitigation**

Radius currently does not support RBAC. Once we have that, we should restrict Radius API access to only Radius users/ admins. 

**Status**

Pending

#### Sniffing the communication between Applications RP  and Kubernetes API Server could cause information disclosure 

**Description:** If a malicious actor could sniff communication between the applications RP and the Kubernetes API Server Or UCP or Controller (Should I say API Server), the actor could replay the packets and cause a DDoS. 

**Impact:** A malicious actor could use the information about the resources and operations in progress. They can also replay the same requests to cause a DDoS.

**Mitigations:**

1. Tampering with Application RP code/ configs would require access to modify the `radius-system` namespace. Our threat model assumes that the operator has limited access to the `radius-system` namespace using Kubernetes' existing RBAC mechanism.

**Status:** All mitigations listed are currently active. Operators are expected to secure their cluster and limit access to the `radius-system` namespace.


## Open Questions

## Action Items

## Review Notes

## References

1. <https://kubernetes.io/blog/2018/07/18/11-ways-not-to-get-hacked>
2. <https://www.rfc-editor.org/rfc/rfc3174.html>
3. <https://pkg.go.dev/crypto/sha1@go1.23.1>

#### Questions
- should we cover dataflow and secrets for contour/ envoy, and workload identity?
- should I say kubernetes controller since we use those libraries and invoke API Server calls? or directly say Appcore RP --> API Server?
- there seems to be no cert for Applications RP when I list secrets
- kubernetes secrets are not encrypted. Should we cover the point and add support of some other secret store as default secret management for Radius going forward?
- how do we store datastore sensitive info such as conn string and password
