# Radius Applications RP Component Threat Model

- **Author**: @nithyatsu

## Overview

This document provides a threat model for the Radius Applications RP component. It identifies potential security threats to this critical part of Radius and suggests possible mitigations. The document includes an analysis of the system, its assets, identified threats, and recommended security measures to protect the system.

The Applications RP component is responsible for managing applications and their resources. It communicates with UCP, Deployment Engine and Controller components to achieve this. 

## Terms and Definitions

| Term                  | Definition                                                                                                                                                                                  |
| --------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| mTLS                  | Mutual Transport Layer Security (mTLS) allows two parties to authenticate each other during the initial connection of an SSL/TLS handshake.                                                 |
| UCP                 | Universal Control Plane for Radius                                                                                                                                                   |
## System Description

Applications RP is a Radius service that acts as resource provider for application and its resources. The resources can be core resources like application or environment or container. They can also be a dapr resource, message queue or datastore. Applications RP lives in `radius-system` namespace in a kubernetes cluster. It is a client of Controller and UCP. It also receives requests from UCP for managing the above mentioned resources. 

### Architecture

Applications RP has four types of resource providers for managing various types of resources in an application. 

`Applications.Core` resource provider manages application, environment, container and gateways. 

`Applications.Dapr` resource provider manages all dapr resources that are deployed as part of application. 

`Applications.Datastore` resource provider manages datastore such as SQL database or Mongo DB that an application might be using. 

`Applications.Messaging` resources provider manages queues such as Rabbit MQ. 

RecipeEngine is a key sub-component of Applications RP, which uses the above resource providers for provisioning infrastructure resources. It takes as input a recipe, written in bicep or terraform. It fetches Bicep recipe from any OCI complaint registry. It also fetches Terraform recipes available as public modules. It then requests UCP to help deploy the recipe. Also, if user creates a custom recipe, it requests Controller to first validate the recipe using recipe-webhook and then create it. 

Users could provision application resources without using recipes too, in which case, Applications RP again works with Controller managing for Kubernetes resources and UCP for managing AWS and Azure resources. For deploying Terraform Recipe, it directly communicates with AWS and Azure.

The Applications RP provisions some resources synchronously, whereas for other resources whose creation can be time consuming, it has workers that enable async operation. To faciliate Async operation, the Applications RP adds the incoming request to an in-memory queue. Workers dequeue requests from the queue and process them. 

Applications RP persists the radius resources metadata in etcd through API Server. For this, it interacts with Kubernetes Controller. 

Below is a high level overview of various sub components in Applications RP
![Applications RP](2024-10-applications-rp-threat-model/apprp.png) 

### Implementation Details

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