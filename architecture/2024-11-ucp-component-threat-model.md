# Radius UCP Component Threat Model

- **Author**: ytimocin

## Overview

This document provides a threat model for the Radius UCP (Universal Control Plane) component. It identifies potential security threats to this critical part of Radius and suggests possible mitigations. The document includes an analysis of the system, its assets, identified threats, and recommended security measures to protect the system.

One of the most important goals of Radius is to be cloud agnostic and to enable portability of applications across platforms and environments. Therefore, the Radius UCP (Universal Control Plane) component helps the user apply ARM capabilities to non-Azure hosting models like AWS, GCP, and Kubernetes.

In brief, UCP is a lightweight proxy that stands in front of the resource providers like Applications RP, AWS, and Azure. Based on the requests UCP gets, it forwards those requests to the appropriate service.

## Terms and Definitions

| Term                    | Definition                                                                                                                                  |
| ----------------------- | ------------------------------------------------------------------------------------------------------------------------------------------- |
| Radius Tracked Resource | Ex: `/planes/radius/local/resourceGroups/test-group/providers/System.Resources/resources/test-app-303153687ee5adbcf353bc6c2caa4373f31e04c6` |

## System Description

An example flow:

1. Radius user runs `rad deploy app.bicep`. `app.bicep` includes a Radius environment, a Radius application, a Redis Cache on AWS.
2. ARM JSON Template is generated from the `app.bicep` file.
3. CLI makes a PUT call to UCP: `/planes/radius/local/resourcegroups/{rg}/providers/Microsoft.Resources/deployments/{name}`
4. UCP forwards that request to the Deployment Engine: `/planes/radius/local/resourcegroups/{rg}/providers/Microsoft.Resources/deployments/{name}`
5. Deployment Engine makes 3 calls back to the UCP:
   1. PUT for the Radius environment.
   2. PUT for the Radius application.
   3. PUT for the Redis Cache on AWS.
6. UCP proxies these requests to the necessary resource providers and then starts polling the statuses of these operations.
7. Deployment Engine also polls the operation statuses and results for these 3 resources.

### Architecture

The UCP (Universal Control Plane) consists of several important pieces:

- **Storage Provider**: Radius needs a data store to store all the information related to the resources of the installation. UCP keeps the information of resources by converting them to tracked resources. On the other hand, available implementations of the Radius Storage Provider:

  1. Cosmos Database
  2. etcd (in-memory or persistent)
  3. In-Memory storage
  4. PostgreSQL

- **Secret Provider**: The UCP occasionally needs to create and store secrets. Available implementations of the UCP Secret Provider are:

  1. etcd
  2. Kubernetes Secrets

- **Queue Provider**: This component handles asynchronous operations. Whenever an operation that is handled asynchronously is requested, it is added as a message to the queue, which is then processed by the UCP worker.

- **Worker**: As mentioned above, the worker is for handling the asynchronous operations. It gets the operation messages from the queue and starts handling them.

### Implementation Details

#### Use of Cryptography

1. **Generating Unique Keys for Queue Messages**: [Link to code](https://github.com/radius-project/radius/blob/main/pkg/ucp/queue/apiserver/client.go#L152)

   1. **Purpose:** To generate unique keys for messages to be added to the queue in the Kubernetes CRD-based implementation of the Radius Queue.
   2. **Library:** Uses the Go standard `crypto/rand` package: [crypto/rand](https://pkg.go.dev/crypto/rand)
   3. **Type:** Random data generated using `crypto/rand`

2. **Hashing Resource IDs for Data Store Resource Names**: [Link to code](https://github.com/radius-project/radius/blob/main/pkg/ucp/store/apiserverstore/apiserverclient.go#L406)

   1. **Purpose:** To hash the resource ID to generate a unique key for the resource name in the data store.
   2. **Library:** Uses the Go standard `crypto/sha1` package: [crypto/sha1](https://pkg.go.dev/crypto/sha1)
   3. **Type:** SHA-1  
      _Note:_ SHA-1 is used here for generating unique identifiers, not for security.

3. **Hashing Resource IDs for Tracking Resource Names**: [Link to code](https://github.com/radius-project/radius/blob/main/pkg/ucp/trackedresource/name.go#L52)

   1. **Purpose:** To hash a given resource ID to compute the tracked resource name.
   2. **Library:** Uses the Go standard `crypto/sha1` package: [crypto/sha1](https://pkg.go.dev/crypto/sha1)
   3. **Type:** SHA-1  
      _Note:_ As previously noted, SHA-1 is used for optimization purposes only.

4. **Hashing Input Data to Generate New ETags**: [Link to code](https://github.com/radius-project/radius/blob/main/pkg/ucp/util/etag/etag.go#L30)

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

1. **Creating or Updating a Cloud Provider Credential Resource**: [Azure](https://github.com/radius-project/radius/blob/95409fe179d7adca884a3fc1d82f326bc81c8da0/pkg/ucp/frontend/controller/credentials/azure/createorupdateazurecredential.go#L89), [AWS](https://github.com/radius-project/radius/blob/95409fe179d7adca884a3fc1d82f326bc81c8da0/pkg/ucp/frontend/controller/credentials/aws/createorupdateawscredential.go#L81). When a **Cloud Provider Credential** resource is deployed, Radius creates a new Kubernetes Secret or updates an existing one. The Kubernetes Secret is deleted when the **Cloud Provider Credential** is deleted.

#### Data Serialization / Formats

We use custom parsers to parse Radius-related resource IDs and do not use any other custom parsers and instead rely on Kubernetes built-in parsers. Therefore, we trust Kubernetes security measures to handle data serialization and formats securely. The custom parser that parses Radius resource IDs has its own security mechanisms that don't accept anything other than a Radius resource ID.

### Clients

In this section, we will discuss the different clients of the Radius UCP (Universal Control Plane) component. Clients are systems that interact with the UCP component to trigger actions. Here are the clients of the UCP component:

1. **Radius CLI**: Radius CLI interacts with UCP under the hood through the generated clients. For example, a `rad deploy` command needs to go to UCP and then through UCP to the Deployment Engine. UCP will do the forwarding of the requests to the Deployment Engine and the other components like Applications RP.

2. **Direct API Querying**: Clients can interact with the UCP component directly by making API queries. For example, a client can query the status of a deployment or retrieve information about a specific resource by making a direct API call to the UCP server.

3. **Deployment Engine**: Deployment Engine makes requests to the UCP to deploy and also ask about the deployment status of certain objects within the ongoing deployment. Example:

   ```text
   {
   "timestamp": "2024-11-03T14:06:54.684Z",
   "severity": "INFO",
   "message": "Start processing HTTP request GET https://ucp.radius-system/apis/api.ucp.dev/v1alpha3/planes/radius/local/resourceGroups/default/providers/Applications.Core/containers/demo?api-version=2023-10-01-preview",
   "name": "RequestPipelineStart",
   "scope": "System.Net.Http.HttpClient.local.LogicalHandler",
   "serviceName": "deployment.engine",
   "version": "0.38\u002B1b2449770894506793f865dc7ecc9ed8e6df0d93",
   "hostName": "bicep-de-6b86cc859f-tt2r8",
   "traceId": "a0d750fc4e5d35bd1c615d87c1261f72",
   "spanId": "5b624ecddd0b19c2",
   "HttpMethod": "GET",
   "Uri": "https://ucp.radius-system/apis/api.ucp.dev/v1alpha3/planes/radius/local/resourceGroups/default/providers/Applications.Core/containers/demo?api-version=2023-10-01-preview"
   },
   {
   "timestamp": "2024-11-03T14:06:54.684Z",
   "severity": "INFO",
   "message": "Sending HTTP request GET https://ucp.radius-system/apis/api.ucp.dev/v1alpha3/planes/radius/local/resourceGroups/default/providers/Applications.Core/containers/demo?api-version=2023-10-01-preview",
   "name": "RequestStart",
   "scope": "System.Net.Http.HttpClient.local.ClientHandler",
   "serviceName": "deployment.engine",
   "version": "0.38\u002B1b2449770894506793f865dc7ecc9ed8e6df0d93",
   "hostName": "bicep-de-6b86cc859f-tt2r8",
   "traceId": "a0d750fc4e5d35bd1c615d87c1261f72",
   "spanId": "5b624ecddd0b19c2",
   "HttpMethod": "GET",
   "Uri": "https://ucp.radius-system/apis/api.ucp.dev/v1alpha3/planes/radius/local/resourceGroups/default/providers/Applications.Core/containers/demo?api-version=2023-10-01-preview"
   }
   ```

4. **Health Check Probes**: Kubernetes itself can act as a client by performing health and readiness checks on the Universal Control Plane.

5. **Metrics Scrapers**: If metrics are enabled, Prometheus or other monitoring tools can scrape metrics from the Universal Control Plane.

## Trust Boundaries

We have a few different trust boundaries for the UCP component:

- **Kubernetes Cluster**: The overall environment where the UCP component operates and receives requests from the clients.
- **Namespaces within the Cluster**: Logical partitions within the cluster to separate and isolate resources and workloads.

The UCP component lives inside the `radius-system` namespace in the Kubernetes cluster where it is installed. Other components of Radius like Controller, Deployment Engine, Dashboard, and Applications RP also live in the same namespace.

### Key Points of Namespaces

1. **Isolation of Resources and Workloads**: Different namespaces separate and isolate resources and workloads within the Kubernetes cluster.
2. **Access Controls and Permissions**: Access controls and other permissions are implemented to manage interactions between namespaces.
3. **Separation of Concerns**: Namespaces support the separation of concerns by allowing different teams or applications to manage their resources independently, reducing the risk of configuration errors and unauthorized changes.

## Assumptions

This threat model assumes that:

1. The Radius installation is not tampered with.
2. The Kubernetes cluster that Radius is installed on is not compromised.
3. It is the responsibility of the Kubernetes cluster to authenticate users. Administrators and users with sufficient privileges can perform their required tasks. Radius cannot prevent actions taken by an administrator.
4. The Data Store (can be one of these: etcd, Cosmos DB, PostgreSQL, in-memory, and API Server) that the UCP uses to keep important data is not compromised.
5. The Secret Store (can be one of these: etcd, Kubernetes Secrets) that the UCP uses to keep important data is not compromised.
6. The Queue that the UCP uses to write messages for the async operations is not compromised.

## Data Flow

### Diagram

![UCP Component via Microsoft Threat Modeling Tool](./2024-11-ucp-component-threat-model/ucp-component-flow.png)

1. **User runs `rad deploy app.bicep` using Radius CLI**: When a user runs `rad deploy app.bicep` using the Radius CLI, the Bicep file gets converted to a JSON Template file and is sent to the UCP.
2. **UCP forwards the template to the Deployment Engine**
3. **Deployment Engine sends the list of resources back to the UCP as requests**
4. **UCP sends each request to the corresponding Resource Provider**
5. **UCP updates its data store with the information of the resources**

### Threats

#### Spoofing Deployment Engine Could Cause Information Disclosure and Denial of Service

**Description:** If a malicious actor can spoof the Deployment Engine by tampering with the configuration in the UCP, the UCP will start sending requests to the malicious server. The malicious server can capture the traffic, leading to information disclosure. This would effectively disable the UCP, causing a Denial of Service.

**Impact:** All data sent to the Deployment Engine by the UCP will be available to the malicious actor, including payloads of resources in the applications. The functionality of the UCP for managing resources will be disabled. Users will not be able to deploy updates to their applications.

**Mitigations:**

1. Tampering with the UCP code, configuration, or certificates would require access to modify the `radius-system` namespace. Our threat model assumes that the operator has limited access to the `radius-system` namespace using Kubernetes' existing RBAC mechanism.
2. The resource payloads sent to the Deployment Engine by the UCP **MAY** contain sensitive operational information (e.g., passwords).

**Status:** All mitigations listed are currently active. Operators are expected to secure their cluster and limit access to the `radius-system` namespace.

#### Information Disclosure by Unauthorized Access to Secrets

**Description:** A malicious actor could circumvent Kubernetes RBAC controls and gain unauthorized access to Kubernetes secrets managed by Radius. These secrets may contain sensitive information, such as credentials intended for use by applications.

**Impact:** A malicious actor could gain access to sensitive information.

**Mitigations:**

1. Secret data managed by the UCP is stored at rest in Kubernetes secrets or etcd. Our threat model assumes that the API server and mechanisms like Kubernetes authentication/RBAC are not compromised.
2. Secrets managed by Radius are always placed in the same namespace as the object that "owns" them. This is a requirement of the Kubernetes RBAC model.
3. Secrets managed by Radius are subject to the Kubernetes RBAC model for controlling access. Operators are expected to limit access for users using existing tools.

**Status:** All mitigations listed are currently active. Operators are expected to secure their cluster and limit access for users.

#### Escalation of Privilege by Using Radius to Circumvent Kubernetes RBAC Controls

**Description:** A malicious actor could circumvent Kubernetes RBAC controls and create arbitrary resources in Kubernetes by using Universal Control Plane. UCP operates with a wide scope of permissions in Kubernetes and the cloud.

At the time of writing, Radius does not provide granular authorization controls. Any authenticated client can create any Radius resource and execute any action Radius is capable of taking. This is the case in UCP as well as other components of Radius like the Kubernetes controllers.

**Impact:** An authorized user of the Kubernetes cluster with permission to create a resource can execute any application definition in any environment registered with Radius.

**Mitigations:**

1. Operators should limit direct access to the Radius API using Kubernetes RBAC.
2. We should revisit the threat model and provide a more robust set of authorization controls when granular authorization policies are added to Radius.

**Status:** These mitigations are partial and require configuration by the operator. We will revisit and improve this area in the future.

## Open Questions

## Action Items

1. Use a hashing algorithm other than SHA-1 while computing the hash of the configuration of a Deployment object. This is a breaking change because deployments that are already hashed with SHA1 should be redeployed so that reconciler can work as expected.
2. Check if RBAC with Least Privilege is configured for every component to ensure that each component has only the permissions it needs to function. Make changes to the necessary components if required.
3. Define and implement necessary Network Policies to ensure that communication is accepted only from expected and authorized components. Regularly review and update these policies to maintain security.
4. Containers should run as a non-root user wherever possible to minimize the risks. Check if we can run any of the Radius containers as non-root. Do the necessary updates.

## Review Notes

1. Initial review on the 5th of November, 2024.

## References

1. <https://kubernetes.io/blog/2018/07/18/11-ways-not-to-get-hacked>
2. <https://www.rfc-editor.org/rfc/rfc3174.html>
3. <https://pkg.go.dev/crypto/sha1@go1.23.1>
