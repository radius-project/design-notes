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

````````
Applications RP is a Radius service that acts as resource provider for application and its resources. When Radius CLI sends a deploy request, UCP receives the request and works with DE for deployment plan. Then, from this plan, it forwards all requests dealing with application management to Applications RP. Applications RP 
* requests Controller to manage kubernetes resources like containers and gateways. 
* requests UCP to provision AWS/Azure resources.
* persists the tracked resources for all application resources into the datastore. For this, it communicates with APIServer. (Should we talk abt etcd in dev setup)
In order to manage recipes, Applications RP
* fetches Recipe from OCI compliant registry if a resource is being provisioned using recipe
* requests UCP to deploy recipes after fetching them from registry
* requests Controller to validate custom recipes

````````````

### Architecture

Applications RP has four types of resource providers for managing various types of resources in an application. 

`Applications.Core` resource provider manages application, environment, container and gateways. 

`Applications.Dapr` resource provider manages all dapr resources that are deployed as part of application. 

`Applications.Datastore` resource provider manages datastore such as SQL database or Mongo DB that an application might be using. 

`Applications.Messaging` resources provider manages queues such as Rabbit MQ. 

RecipeEngine is a key sub-component of Applications RP, which uses the above resource providers for provisioning infrastructure resources. It takes as input a recipe, written in bicep or terraform. It fetches Bicep recipe from any OCI complaint registry. It also fetches Terraform recipes available as public modules. It then requests UCP to help deploy the recipe. Also, if user creates a custom recipe, it requests Controller to first validate the recipe using recipe-webhook and then create it. 

Users could provision application resources without using recipes too, in which case, Applications RP again works with Controller managing for Kubernetes resources and UCP for deploying AWS and Azure resources.

The Applications RP provisions some resources synchronously, whereas for other resources whose creation can be time consuming, it has workers that enable async operation. To faciliate Aync operation, the Applications RP adds the incoming request to an in-memory queue. Workers dequeue requests from the queue and process them. 

Applications RP persists the radius resources metadata in etcd through API Server. For this, it interacts with Kubernetes Controller. 

### Implementation Details

Applications RP deploying a AWS/Azure Resource

Applications RP deploying a Kubernetes Resource

Applications RP deploying a Recipe 


#### Use of Cryptography

#### Storage of secrets

Applications RP creates and stores a kubernetes secret for Gateways. 

#### Data Serialization / Formats



### Clients

In this section, we will discuss the different clients of the Applications RP component. Clients are systems that interact with the Applications RP component to trigger actions. Here are the clients of the Applications RP component:

1. **UCP**: 


### 
## Trust Boundaries

We have a few different trust boundaries for the Controller component:

- **Kubernetes Cluster**: The overall environment where the Controller component operates and receives requests from the clients.
- **Namespaces within the Cluster**: Logical partitions within the cluster to separate and isolate resources and workloads.

The Controller component lives inside the `radius-system` namespace in the Kubernetes cluster where it is installed. UCPD also resides within the same namespace.

The Kubernetes API Server, which is the main interactor of the Controller component, runs in the `kube-system` namespace within the cluster.

### Key Points of Namespaces

1. **Isolation of Resources and Workloads**: Different namespaces separate and isolate resources and workloads within the Kubernetes cluster.
2. **Access Controls and Permissions**: Access controls and other permissions are implemented to manage interactions between namespaces.
3. **Separation of Concerns**: Namespaces support the separation of concerns by allowing different teams or applications to manage their resources independently, reducing the risk of configuration errors and unauthorized changes.

## Assumptions

This threat model assumes that:

1. The Radius installation is not tampered with.
2. The Kubernetes cluster that Radius is installed on is not compromised.
3. It is the responsibility of the Kubernetes cluster to authenticate users. Administrators and users with sufficient privileges can perform their required tasks. Radius cannot prevent actions taken by an administrator.

## Data Flow

### Diagram

![Applications RP via Microsoft Threat Modeling Tool](2024-10-applications-rp-threat-model/apprp.png)

1. **User creates/updates/deletes a Recipe or a Deployment resource**: When a user requests to create, update, or delete a Recipe or a Deployment resource, the request is handled by the Kubernetes API Server. One way, and probably the most common way, a user can do this request is by running a **kubectl** command. Kubernetes takes care of the authentication and the authorization of the user and its request(s) so we (Radius) don't need to worry about anything here.

2. **Validating Webhook**: The only type of admission controller we have in Radius is the validating webhook for the Recipe resource. The validating webhook ensures that the Recipe object to be created or updated is one of the Radius portable resources. Whenever Kubernetes API Server receives a request to create or update a Recipe object, it communicates the proposed changes with the validating webhook. If the validating webhook validates the changes, then it is persisted to the **etcd** by the Kubernetes API Server.

3. **Recipe and Deployment Reconcilers**: When there is a request to create, update, or delete a Recipe or a Deployment resource, after being validated if the resource is a Recipe resource, the next step is the reconcilation of the resource by the appropriate reconciler. In the Controller component, there are two reconcilers: Recipe and Deployment. These reconcilers are loops that watch the changes in the Recipe and Deployment resources. Whenever there is a change, the reconcilers take the necessary actions to move the actual state to the desired state. These necessary actions include communication with the UCPD to create, update, and/or delete necessary resources.

   1. Communication:
      1. Controller and UCPD:
         1. Poll long-running operations (create, update, or delete) for a Recipe or a Deployment resource.
         2. Create a Radius Resource Group if needed.
         3. Create a Radius Application if needed.
         4. Get a Radius resource like the Environment that is associated with the resource.
         5. Create/Update/Delete a Recipe or a Deployment resource.
         6. Create/Update/Delete a Secret for a Recipe or a Deployment resource.
      2. Controller and the Kubernetes API Server:
         1. Fetch a Recipe or a Deployment object.
         2. Send events related to the operations running.
         3. Update a Recipe or a Deployment object.
         4. Create/Update/Delete a Secret associated with a Recipe or a Deployment object.
         5. List Deployments by filtering them by a specific Recipe object.

### Threats

#### Spoofing UCP API Server Could Cause Information Disclosure and Denial of Service

**Description:** If a malicious actor can spoof the UCP API Server by tampering with the configuration in the Controller, the Controller will start sending requests to the malicious server. The malicious server can capture the traffic, leading to information disclosure. This would effectively disable the Controller, causing a Denial of Service.

**Impact:** All data sent to UCP by the Controller will be available to the malicious actor, including payloads of resources in the applications. The functionality of the Controller for managing resources will be disabled. Users will not be able to deploy updates to their applications.

**Mitigations:**

1. Tampering with the controller code, configuration, or certificates would require access to modify the `radius-system` namespace. Our threat model assumes that the operator has limited access to the `radius-system` namespace using Kubernetes' existing RBAC mechanism.
2. The resource payloads sent to UCP by the Controller do not contain sensitive operational information (e.g., passwords).

**Status:** All mitigations listed are currently active. Operators are expected to secure their cluster and limit access to the `radius-system` namespace.

#### Spoofing the Kubernetes API Server Leading to Escalation of Privilege

**Description:** If a malicious actor could hijack communication between the controller and the Kubernetes API Server, the actor could send requests to the controller. At that point, the controller would be processing illegitimate data.

**Impact:** A malicious actor could use the controllers (Recipe and/or Deployment) to escalate privileges and perform arbitrary operations against Radius/UCP.

**Mitigations:**

1. The controllers authenticate requests to the Kubernetes API Server using credentials managed and rotated by Kubernetes. Our threat model assumes that the API Server and mechanisms like Kubernetes-managed authentication are not compromised.
2. The webhook follows a known Kubernetes implementation pattern and uses widely supported libraries to communicate (client-go, Kubebuilder).
3. Tampering with the controller code, configuration, or authentication tokens would require access to modify the `radius-system` namespace. Our threat model assumes that the operator has limited access to the `radius-system` namespace using Kubernetes' existing RBAC mechanism.

**Status:** All mitigations listed are currently active. Operators are expected to secure their cluster and limit access to the `radius-system` namespace.

#### Spoofing Requests to the Validating Webhook

**Description:** If a malicious actor could circumvent webhook authentication, they could send unauthorized requests to the webhook.

**Impact:** The webhook performs validation only and does not mutate any state. The security impact of spoofing is unclear, but it could potentially lead to unauthorized actions being validated.

**Mitigations:**

1. The webhook authenticates requests (mTLS) from the Kubernetes API Server using a certificate managed and rotated by Kubernetes. Our threat model assumes that the API Server and mechanisms like Kubernetes-managed certificates are not compromised.
2. The webhook follows a known Kubernetes implementation pattern and uses widely supported libraries to implement mTLS (Kubebuilder).
3. Tampering with the webhook code, configuration, or certificates would require access to modify the `radius-system` namespace. Our threat model assumes that the operator has limited access to the `radius-system` namespace using Kubernetes' existing RBAC mechanism.

**Status:** All mitigations listed are currently active. Operators are expected to secure their cluster and limit access to the `radius-system` namespace.

#### Denial of Service Caused by Invalid Request Data

**Description:** If a malicious actor sends a malformed request that triggers unbounded execution on the server.

**Impact:** A malicious actor could cause a denial of service or waste compute resources.

**Mitigations:**

1. The controllers and webhooks use widely supported libraries for all parsing of untrusted data in standard formats.
   1. The Go standard libraries are used for HTTP.
   2. The Kubernetes YAML libraries are used for YAML parsing.
2. Radius/UCP implements a custom parser for resource IDs, a custom string format. This requires fuzz-testing.

**Status:** All mitigations listed are currently active. Operators are expected to secure their cluster and limit access to the `radius-system` namespace.

#### Information Disclosure by Unauthorized Access to Secrets

**Description:** A malicious actor could circumvent Kubernetes RBAC controls and gain unauthorized access to Kubernetes secrets managed by Radius. These secrets may contain sensitive information, such as credentials intended for use by applications.

**Impact:** A malicious actor could gain access to sensitive information.

**Mitigations:**

1. Secret data managed by the controllers is stored at rest in Kubernetes secrets. Our threat model assumes that the API server and mechanisms like Kubernetes authentication/RBAC are not compromised.
2. Secrets managed by Radius are always placed in the same namespace as the object that "owns" them. This is a requirement of the Kubernetes RBAC model.
3. Secrets managed by Radius are subject to the Kubernetes RBAC model for controlling access. Operators are expected to limit access for users using existing tools.

**Status:** All mitigations listed are currently active. Operators are expected to secure their cluster and limit access for users.

#### Escalation of Privilege by Using Radius to Circumvent Kubernetes RBAC Controls

**Description:** A malicious actor could circumvent Kubernetes RBAC controls and create arbitrary resources in Kubernetes by using the `Recipe` custom resource.

The `Recipe` controller has limited permissions, so it cannot be used directly to escalate privileges in Kubernetes. However, it calls into UCP/Radius, which operates with a wide scope of permissions in Kubernetes and the cloud.

Authorized users with access to create a `Recipe` resource in Kubernetes can execute any Recipe in any Environment registered with Radius.

At the time of writing, Radius does not provide granular authorization controls. Any authenticated client can create any Radius resource and execute any action Radius is capable of taking. This is not limited to the Kubernetes controllers.

**Impact:** An authorized user of the Kubernetes cluster with permission to create a `Recipe` resource can execute any Recipe in any Environment registered with Radius.

**Mitigations:**

1. Operators should limit access to the `Recipe` resource using Kubernetes RBAC.
2. Operators should limit direct access to the Radius API using Kubernetes RBAC.
3. We should revisit the threat model and provide a more robust set of authorization controls when granular authorization policies are added to Radius.

**Status:** These mitigations are partial and require configuration by the operator. We will revisit and improve this area in the future.

## Open Questions

## Action Items

1. Use a hashing algorithm other than SHA-1 while computing the hash of the configuration of a Deployment object. This is a breaking change because deployments that are already hashed with SHA1 should be redeployed so that reconciler can work as expected.
2. Check if RBAC with Least Privilege is configured for every component to ensure that each component has only the permissions it needs to function. Make changes to the necessary components if required.
3. Define and implement necessary Network Policies to ensure that communication is accepted only from expected and authorized components. Regularly review and update these policies to maintain security.
4. Containers should run as a non-root user wherever possible to minimize the risks. Check if we can run any of the Radius containers as non-root. Do the necessary updates.

## Review Notes

## References

1. <https://kubernetes.io/blog/2018/07/18/11-ways-not-to-get-hacked>
2. <https://www.rfc-editor.org/rfc/rfc3174.html>
3. <https://pkg.go.dev/crypto/sha1@go1.23.1>
