# Radius UCP Component Threat Model

- **Author**: ytimocin

## Overview

This document provides a threat model for the Radius UCP (Universal Control Plane) component. It identifies potential security threats to this critical part of Radius and suggests possible mitigations. The document includes an analysis of the system, its assets, identified threats, and recommended security measures to protect the system.

One of the most important goals of Radius is to be cloud agnostic and to enable portability of applications across platforms and environments. Therefore, the Radius UCP (Universal Control Plane) component helps the user apply ARM capabilities to non-Azure hosting models like AWS, GCP, and Kubernetes.

<!-- TODO: Add more here -->

## Terms and Definitions

| Term                    | Definition                                                                                                                                  |
| ----------------------- | ------------------------------------------------------------------------------------------------------------------------------------------- |
| Radius Tracked Resource | Ex: `/planes/radius/local/resourceGroups/test-group/providers/System.Resources/resources/test-app-303153687ee5adbcf353bc6c2caa4373f31e04c6` |

## System Description

### Architecture

The UCP (Universal Control Plane) consists of several important services that run as goroutines in the same application process:

- **Storage Provider**: As the name suggests, this service is the service that is used to handle the Radius-data-related operations. Available implementations:
  - Cosmos Database
  - etcd
  - In-Memory
  - PostgreSQL
- **Secret Provider**
- **Queue Provider**
- **Profiler Provider**
- **Metrics Provider**
- **Tracer Provider**

### Implementation Details

#### Use of Cryptography

1. **Hashing Secret Data for the Container**: [Link to code](https://github.com/radius-project/radius/blob/main/pkg/corerp/renderers/container/render.go#L618)

   1. **Purpose:** To hash the secret data of the container resource to determine if the deployment is up-to-date or needs an update.
   2. **Library:** Uses the Go standard `crypto/sha1` package: [crypto/sha1](https://pkg.go.dev/crypto/sha1)
   3. **Type:** SHA-1  
      _Note:_ SHA-1 is cryptographically broken and should not be used for secure applications. It is used here solely as an optimization for detecting changes, not as a security protection.

2. **Hashing Combined IDs for Kubernetes Secret in Terraform Backend**: [Link to code](https://github.com/radius-project/radius/blob/main/pkg/recipes/terraform/config/backends/kubernetes.go#L110)

   1. **Purpose:** To hash a combination of the resource ID, environment ID, and application ID to generate a unique key for the Kubernetes secret used in Terraform recipe execution.
   2. **Library:** Uses the Go standard `crypto/sha1` package: [crypto/sha1](https://pkg.go.dev/crypto/sha1)
   3. **Type:** SHA-1  
      _Note:_ As above, SHA-1 is used here for optimization, not for security purposes.

3. **Generating Unique Keys for Queue Messages**: [Link to code](https://github.com/radius-project/radius/blob/main/pkg/ucp/queue/apiserver/client.go#L152)

   1. **Purpose:** To generate unique keys for messages to be added to the queue in the Kubernetes CRD-based implementation of the Radius Queue.
   2. **Library:** Uses the Go standard `crypto/rand` package: [crypto/rand](https://pkg.go.dev/crypto/rand)
   3. **Type:** Random data generated using `crypto/rand`

4. **Hashing Resource IDs for Data Store Resource Names**: [Link to code](https://github.com/radius-project/radius/blob/main/pkg/ucp/store/apiserverstore/apiserverclient.go#L406)

   1. **Purpose:** To hash the resource ID to generate a unique key for the resource name in the data store.
   2. **Library:** Uses the Go standard `crypto/sha1` package: [crypto/sha1](https://pkg.go.dev/crypto/sha1)
   3. **Type:** SHA-1  
      _Note:_ SHA-1 is used here for generating unique identifiers, not for security.

5. **Hashing Resource IDs for Tracking Resource Names**: [Link to code](https://github.com/radius-project/radius/blob/main/pkg/ucp/trackedresource/name.go#L52)

   1. **Purpose:** To hash a given resource ID to compute the tracked resource name.
   2. **Library:** Uses the Go standard `crypto/sha1` package: [crypto/sha1](https://pkg.go.dev/crypto/sha1)
   3. **Type:** SHA-1  
      _Note:_ As previously noted, SHA-1 is used for optimization purposes only.

6. **Hashing Input Data to Generate New ETags**: [Link to code](https://github.com/radius-project/radius/blob/main/pkg/ucp/util/etag/etag.go#L30)

   1. **Purpose:** To hash input data to generate new ETags for resource versioning.
   2. **Library:** Uses the Go standard `crypto/sha1` package: [crypto/sha1](https://pkg.go.dev/crypto/sha1)
   3. **Type:** SHA-1  
      _Note:_ SHA-1 usage here is for generating ETags and not for cryptographic security.

---

##### General Note on SHA-1 Usage

In all instances where SHA-1 is utilized within the codebase, it serves for generating hashes to ensure uniqueness or detect changes efficiently. It is **not** used for cryptographic security purposes due to known vulnerabilities. For security-critical hashing, a more secure algorithm (e.g., SHA-256) should be employed.

---

#### Storage of Secrets

Below you will find where and how Radius stores secrets. We create Kubernetes Secret objects and rely on Kubernetes security measures to protect these secrets.

1. **Creating or Updating a Radius Secret Store Resource**: [Link to code](https://github.com/radius-project/radius/blob/95409fe179d7adca884a3fc1d82f326bc81c8da0/pkg/corerp/frontend/controller/secretstores/kubernetes.go#L201). When a **Radius Secret Store** resource is deployed, Radius creates a new Kubernetes Secret or updates an existing one, depending on whether it was previously deployed. The Kubernetes Secret is deleted when the **Radius Secret Store** is deleted.
2. **Creating or Updating a Radius Container Resource**: [Link to code](https://github.com/radius-project/radius/blob/95409fe179d7adca884a3fc1d82f326bc81c8da0/pkg/corerp/renderers/container/render.go#L250). When a **Radius Container** resource is deployed, Radius creates or updates the necessary resources that the container needs, including Kubernetes Secrets, if required. The Kubernetes Secret is deleted when the **Radius Container** is deleted.
3. **Creating or Updating a Cloud Provider Credential Resource**: [Azure](https://github.com/radius-project/radius/blob/95409fe179d7adca884a3fc1d82f326bc81c8da0/pkg/ucp/frontend/controller/credentials/azure/createorupdateazurecredential.go#L89), [AWS](https://github.com/radius-project/radius/blob/95409fe179d7adca884a3fc1d82f326bc81c8da0/pkg/ucp/frontend/controller/credentials/aws/createorupdateawscredential.go#L81). When a **Cloud Provider Credential** resource is deployed, Radius creates a new Kubernetes Secret or updates an existing one. The Kubernetes Secret is deleted when the **Cloud Provider Credential** is deleted.

#### Data Serialization / Formats

We use custom parsers to parse Radius-related resource IDs and do not use any other custom parsers and instead rely on Kubernetes built-in parsers. Therefore, we trust Kubernetes security measures to handle data serialization and formats securely. The custom parser that parses Radius resource IDs has its own security mechanisms that don't accept anything other than a Radius resource ID.

### Clients

In this section, we will discuss the different clients of the Controller component. Clients are systems that interact with the Controller component to trigger actions. Here are the clients of the Controller component:

1. **Kubernetes API Server**: The primary client that interacts with the controller. It communicates with the validating webhook and the controllers in case of resource changes (e.g., creation, update, or deletion of a Recipe or Deployment) requested by another interactor (for example; a human running `kubectl` commands). The controllers watch for these calls and reconcile the state of the resources accordingly.
1. **Health Check Probes**: Kubernetes itself can act as a client by performing health and readiness checks on the controller manager.
1. **Metrics Scrapers**: If metrics are enabled, Prometheus or other monitoring tools can scrape metrics from the controller manager.

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

![Controller Component via Microsoft Threat Modeling Tool](./2024-08-controller-component-threat-model/controller-component.png)

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
