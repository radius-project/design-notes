# Radius UCP Component Threat Model

- **Author**: ytimocin

## Overview

This document provides a threat model for the Radius UCP (Universal Control Plane) component. It identifies potential security threats to this critical part of Radius and suggests possible mitigations. The document includes an analysis of the system, its assets, identified threats, and recommended security measures to protect the system.

One of the most important goals of Radius is to be cloud agnostic and to enable portability of applications across platforms and environments. Therefore, the Radius UCP (Universal Control Plane) component helps the user apply ARM capabilities to non-Azure hosting models like AWS, GCP, and Kubernetes.

In brief, the UCP is a lightweight proxy that sits in front of resource providers such as Applications RP, AWS, and Azure. It routes traffic from various clients of Radius to the appropriate resource provider. It holds the necessary credentials (Azure and/or AWS) to manage resources on behalf of Radius users. **As of November 15, 2024, there is no Role-Based Access Control (RBAC) implemented, and the traffic between the UCP and the resource providers is unauthenticated.**

In the near future, we plan to provide Radius users with the capability to add their custom resource providers, increasing the number of resource providers that the UCP will route user requests to.

## Terms and Definitions

| Term | Definition |
| ---- | ---------- |
|      |            |

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

- **Storage Provider**: Radius needs a data store to store all the information related to the resources of the installation. UCP keeps the information of resources by converting them to tracked resources. Available implementations of the Radius Storage Provider:

  1. Cosmos Database
  2. etcd (in-memory or persistent)
  3. In-Memory storage
  4. PostgreSQL
  5. apiserver

- **Secret Provider**: The UCP occasionally needs to create and store secrets. Available implementations of the UCP Secret Provider are:

  1. etcd
  2. Kubernetes Secrets
  3. In-memory

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

1. **Creating or Updating a Cloud Provider Credential Resource**: [Azure Implementation](https://github.com/radius-project/radius/blob/95409fe179d7adca884a3fc1d82f326bc81c8da0/pkg/ucp/frontend/controller/credentials/azure/createorupdateazurecredential.go#L89), [AWS Implementation](https://github.com/radius-project/radius/blob/95409fe179d7adca884a3fc1d82f326bc81c8da0/pkg/ucp/frontend/controller/credentials/aws/createorupdateawscredential.go#L81). When a **Cloud Provider Credential** resource is deployed, Radius creates a new Kubernetes Secret or updates an existing one. The Kubernetes Secret is deleted when the **Cloud Provider Credential** is deleted. We should also note that these credentials cannot be retrieved or updated through API calls.

#### Data Serialization / Formats

We use custom parsers exclusively for parsing Radius-related resource IDs. For all other parsing needs, we rely on the Go Standard Library's built-in parsers, trusting its security measures to handle data serialization and formats securely. Our custom parser for Radius resource IDs includes security mechanisms that ensure only valid Radius resource IDs are accepted.

### Clients

In this section, we will discuss the different clients of the Radius UCP (Universal Control Plane) component. Clients are systems that interact with the UCP component to trigger actions. Here are the clients of the UCP component:

1. **All Components of Radius**: Every component other than UCP is a client of UCP. This list includes the Radius CLI, Deployment Engine, Controller, Deployment Engine, Dashboard, and Applications RP. An example of how Deployment Engine communicates with UCP can be found below:

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

2. **Health Check Probes**: Kubernetes itself can act as a client by performing health and readiness checks on the Universal Control Plane.

3. **Metrics Scrapers**: If metrics are enabled, Prometheus or other monitoring tools can scrape metrics from the Universal Control Plane.

## Trust Boundaries

### Trust Model of UCP Clients

The clients of the UCP include other Radius components like Applications RP, CLI, Controller, Dashboard, and Deployment Engine. Most of these clients reside within the same Kubernetes cluster and namespace as the UCP, except for the Radius CLI, which operates outside of the Kubernetes cluster and, therefore, outside of the namespace.

For clients that are in the same cluster and namespace as the UCP component, we rely on Kubernetes to implement the necessary security measures and establish the required trust boundary. For external clients--meaning clients that are outside of this trust boundary--we rely on the host machine's trust boundary and assume that it is not compromised. In such cases, proper authentication and authorization are essential to maintain security.

### Trust Model of Internal Resource providers

Currently, our only resource provider is the Applications RP. As mentioned above, the Applications RP lives in the same Kubernetes cluster and namespace as the UCP. Therefore, the trust boundaries are defined by the Kubernetes cluster and the namespace within that cluster.

### Trust Model of External Resource Manages (Azure and AWS)

The UCP routes user requests to Azure and AWS whenever a resource from those providers is requested. In these cases, we rely on Azure and AWS to establish the trust boundary and operate under the assumption that they are secure and trustworthy.

## Assumptions

This threat model assumes that:

1. The Radius installation is not tampered with.
2. The Kubernetes cluster that Radius is installed on is not compromised.
3. It is the responsibility of the Kubernetes cluster to authenticate users. Administrators and users with sufficient privileges can perform their required tasks. Radius cannot prevent actions taken by an administrator.
4. The Data Store (can be one of these: etcd, Cosmos DB, PostgreSQL, in-memory, and API Server) that the UCP uses to keep important data is not compromised.
5. The Secret Store (can be one of these: etcd, Kubernetes Secrets) that the UCP uses to keep important data is not compromised.
6. The Queue that the UCP uses to write messages for the async operations is not compromised.
7. External Resource Managers (Azure and AWS) are not compromised and are working as expected.

## Data Flow

### Diagram

![UCP Component via Microsoft Threat Modeling Tool](./2024-11-ucp-component-threat-model/ucp-component-flow.png)

1. **User runs `rad deploy app.bicep` using Radius CLI**: When a user runs `rad deploy app.bicep` using the Radius CLI, the Bicep file gets converted to a JSON Template file and is sent to the UCP.
2. **UCP forwards the template to the Deployment Engine**
3. **Deployment Engine sends the list of resources back to the UCP as requests**
4. **UCP sends each request to the corresponding Resource Provider**
5. **UCP updates its data store with the information of the resources**

### Threats

#### Threat: Spoofing Deployment Engine Could Cause Information Disclosure and Denial of Service

**Description:** If a malicious actor can spoof the Deployment Engine by tampering with the configuration in the UCP, the UCP will start sending requests to the malicious server. The malicious server can capture the traffic, leading to information disclosure. This would effectively disable the UCP, causing a Denial of Service.

**Impact:** All data sent to the Deployment Engine by the UCP will be available to the malicious actor, including payloads of resources in the applications. The functionality of the UCP for managing resources will be disabled. Users will not be able to deploy updates to their applications.

**Mitigations:**

1. Tampering with the UCP code, configuration, or certificates would require access to modify the `radius-system` namespace. Our threat model assumes that the operator has limited access to the `radius-system` namespace using Kubernetes' existing RBAC mechanism.
2. The resource payloads sent to the Deployment Engine by the UCP **MAY** contain sensitive operational information (e.g., passwords).

**Status:** All mitigations listed are currently active. Operators are expected to secure their cluster and limit access to the `radius-system` namespace.

#### Threat: Information Disclosure by Unauthorized Access to Secrets

**Description:** A malicious actor could circumvent Kubernetes RBAC controls and gain unauthorized access to Kubernetes secrets managed by Radius. These secrets may contain sensitive information, such as credentials intended for use by applications.

**Impact:** A malicious actor could gain access to sensitive information.

**Mitigations:**

1. Secret data managed by the UCP is stored at rest in Kubernetes secrets or etcd. Our threat model assumes that the API server and mechanisms like Kubernetes authentication/RBAC are not compromised.
2. Secrets managed by Radius are always placed in the same namespace as the object that "owns" them. This is a requirement of the Kubernetes RBAC model.
3. Secrets managed by Radius are subject to the Kubernetes RBAC model for controlling access. Operators are expected to limit access for users using existing tools.

**Status:** All mitigations listed are currently active. Operators are expected to secure their cluster and limit access for users.

#### Threat: Escalation of Privilege by Using Radius to Circumvent Kubernetes RBAC Controls

**Description:** A malicious actor could circumvent Kubernetes RBAC controls and create arbitrary resources in Kubernetes by using Universal Control Plane. UCP operates with a wide scope of permissions in Kubernetes and the cloud.

At the time of writing, Radius does not provide granular authorization controls. Any authenticated client can create any Radius resource and execute any action Radius is capable of taking. This is the case in UCP as well as other components of Radius like the Kubernetes controllers.

**Impact:** An authorized user of the Kubernetes cluster with permission to create a resource can execute any application definition in any environment registered with Radius.

**Mitigations:**

1. Operators should limit direct access to the Radius API using Kubernetes RBAC.
2. We should revisit the threat model and provide a more robust set of authorization controls when granular authorization policies are added to Radius.

**Status:** These mitigations are partial and require configuration by the operator. We will revisit and improve this area in the future.

#### Threat: Lack of Role-Based Access Control (RBAC) and Unauthorized Traffic

**Description:** As mentioned above, as of November 15, 2024, the UCP does not implement RBAC, and communication between the UCP and resource providers is unauthenticated.

**Impact:** Increased risk of unauthorized access and actions, making it easier for attackers to interact with resource providers or manipulate user resources without proper authorization.

**Mitigations:**

1. **Implement RBAC within the UCP:** An authentication and authorization mechanism that verifies the identity of clients and enforces access policies must be developed.
2. **Secure Communication Between UCP and Other Components:** mTLS should be enabled.
3. **Network Policies and Firewall Rules:** Application of Kubernetes Network Policies to control traffic flow to and from the UCP.

**Status:** Other than the second mitigation (secure communication) listed above, none of the mitigations is currently active.

#### Threat: Use of Weak Cryptographic Hashing Algorithm (SHA-1)

**Description:** As mentioned above, the UCP uses SHA-1 for hashing resource IDs and generating ETags.

**Impact:** Although used for non-security purposes, SHA-1 has known vulnerabilities. If misused in security contexts, it could lead to hash collisions and potential security weaknesses. Also, if we want to have a stronger cryptographic hashing algorithm for hashing resource IDs and generating ETags, we should use another one.

**Mitigations:**

1. **Replace SHA-1 with a Stronger Algorithm.**

**Status:** The mitigation is not active as of now. An action item is created to update the crypto algorithm used in hashing resource IDs and generating ETags as well as in other components of Radius.

## Open Questions

## Action Items

1. Use a hashing algorithm other than SHA-1 while computing the hash of resource IDs and generating ETags.
2. Check if RBAC with Least Privilege is configured for every component to ensure that each component has only the permissions it needs to function. Make changes to the necessary components if required.
3. Define and implement necessary Network Policies to ensure that communication is accepted only from expected and authorized components. Regularly review and update these policies to maintain security.
4. Containers should run as a non-root user wherever possible to minimize the risks. Check if we can run any of the Radius containers as non-root. Do the necessary updates.

## Review Notes

1. Initial review on the 5th of November, 2024.
2. Comments addressed on the 15th of November, 2024.

## References

1. <https://kubernetes.io/blog/2018/07/18/11-ways-not-to-get-hacked>
2. <https://www.rfc-editor.org/rfc/rfc3174.html>
3. <https://pkg.go.dev/crypto/sha1@go1.23.1>
