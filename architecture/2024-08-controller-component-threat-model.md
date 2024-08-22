# Title

- **Author**: ytimocin

## Overview

This document provides a threat model for the Radius Controller component. It identifies potential security threats to this critical part of Radius and suggests possible mitigations. The document includes an analysis of the system, its assets, identified threats, and recommended security measures to protect the system.

The Radius Controller component monitors changes (create, update, delete) in the definitions of Recipe and Deployment resources. Based on these changes, the appropriate controller takes the necessary actions. Below, you will find detailed information about the key parts of the Radius controllers and the objects they manage.

## Terms and Definitions

| Term                  | Definition                                                                                                                                                                                  |
| --------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Admission Controllers | An admission controller is a piece of code that intercepts requests to the Kubernetes API server prior to persistence of the object, but after the request is authenticated and authorized. |
| mTLS                  | Mutual Transport Layer Security (mTLS) allows two parties to authenticate each other during the initial connection of an SSL/TLS handshake.                                                 |

## System Description

The Controller component is a critical part of the Radius system, responsible for managing Kubernetes resources through custom controllers and webhooks. It ensures that the desired state of the system is maintained by continuously monitoring and reconciling resources.

The Controller component consists of two Kubernetes controllers (Recipe and Deployment controllers), a validating webhook for changes in the Recipe object, and several other important parts. We will dive into more details on the controller below.

Note: If you would like to learn more about Kubernetes controllers, you can visit [this link](https://kubernetes.io/docs/concepts/architecture/controller/).

Note: Kubernetes has [admission controllers](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/) that intercept requests to the Kubernetes API Server before the persistence of the object. Admission controllers may be **validating**, **mutating**, or both. Here is a simple diagram of the flow from the command entered by the user to the persistence of the object to etcd.

![Admission Controllers' Flow](./2024-08-controller-component-threat-model/admission-controllers-flow.png)

### Architecture

The Controller component consists of several key parts:

- **Recipe and Deployment Controllers**: Kubernetes controllers registered with the manager. Each controller is responsible for watching specific Kubernetes resources (Recipe and Deployment in our case) and reconciling their state.
  - **Recipe Controller**: This controller specifically watches for additions of and changes in Recipe objects in the cluster. The controller calls UCPD to perform the following operations:
    - Create, update, or delete a Recipe object based on the input received from the Kubernetes API Server.
    - Create, update, or delete a Secret if it is requested by the Recipe object.
  - **Deployment Controller**: This controller specifically watches for additions of and changes in Deployment objects in the cluster. The controller calls UCPD to perform the following operations:
    - Create, update, or delete a Deployment object based on the input received from the Kubernetes API Server.
    - Create, update, or delete a Secret if it is requested by the Deployment object.
- **Recipe Validating Webhook**: This webhook is triggered by the Kubernetes API Server in case of a create, update, or delete in a Recipe object. The webhook tries to validate the action and responds to the Kubernetes API with an approval or a rejection. For more information about webhooks, refer to the [official documentation](https://kubernetes.io/docs/reference/access-authn-authz/extensible-admission-controllers/).
- **Health Checks**: Health checks are implemented to monitor the status and performance of the controllers. They ensure that the controllers are functioning correctly and can take corrective actions if any issues are detected.

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

### Key Points

1. **Namespace Isolation**: Different namespaces separate and isolate resources and workloads within the Kubernetes cluster.
2. **Access Controls**: Access controls and other permissions are implemented to manage interactions between namespaces.

## Data Flow

### Diagram

![Controller Component via Microsoft Threat Modeling Tool](./2024-08-controller-component-threat-model/controller-component.png)

1. **Resource Creation/Update/Delete**: When a user requests to create, update, or delete a Recipe or Deployment object, the request is sent to the Kubernetes API Server. For example, this request can be made by a human interactor using the `kubectl` command. The Kubernetes API Server authenticates and authorizes the request before the admission controllers intercept it.

2. **Validating Webhook**: The only type of admission controller we have in Radius is the validating webhook for the Recipe resource. The validating webhook ensures that the Recipe object is one of the Radius portable resources. If the operation and the object is validated, then it is persisted to the **etcd**. You can see the details of the Recipe Webhook implementation via [this link](https://github.com/radius-project/radius/blob/main/pkg/controller/reconciler/recipe_webhook.go#L46).

3. **Controller Reconciliation**: When the action (create/update/delete) is validated by the Validating Webhook (applicable only to the Recipe resource and not to the Deployment resource) as discussed in the previous step, the Kubernetes controllers we have (Recipe and Deployment Reconcilers) watch for changes via the Kubernetes API Server. The controllers then reconcile the resource's state to match the desired configuration specified in the resource definition. This involves ensuring that the actual state of the cluster matches the desired state, making any necessary adjustments to achieve this.

4. **Communication with UCPD**: Controller also calls UCPD when doing reconcilations. Here is the list of instances where Controller calls UCPD:

   1. Recipe Reconciler:

      - Poll operations like create, update, or delete a Recipe resource.
      - Resolve dependencies for a Recipe resource:
        - Get the Radius environment the Recipe is associated with.
        - Create the Radius Resource Group if needed.
        - Create the Radius Application if needed.
      - Fetch a Radius resource.
      - List Secrets of a Recipe resource.
      - Delete a Recipe resource.

   2. Deployment Reconciler:

      - Poll operations like create, update, or delete a Deployment resource.
      - Resolve dependencies for a Deployment resource:
        - Get the Radius environment the Deployment is associated with.
        - Create the Radius Resource Group if needed.
        - Create the Radius Application if needed.
      - Fetch a Recipe resource.
      - List Secrets of a Recipe resource.

5. **Communication with Kubernetes API Server**: Controller also calls the Kubernetes API Server when doing reconcilations. Here is the list of instances where Controller calls the Kubernetes API Server:

   1. Recipe Reconciler:

      - Fetch a Recipe object.
      - Send events related to the operations.
      - Update a Recipe object:
        - Status update.
        - Addition and removal of a Finalizer.
      - Delete a Secret object that is associated with the Recipe.
      - Create a Secret object to associate with the Recipe if needed.

   2. Deployment Reconciler:

      - Fetch a Deployment object.
      - Send events related to the operations.
      - Update a Deployment object:
        - Status update.
        - Addition and removal of a Finalizer.
      - Delete a Secret object associated with a Deployment that is no longer needed.
      - Fetch a Secret object.
      - Create or Update a Secret object and associate it with a Deployment object.
      - List Deployments filtered with specific Recipe objects.

### Threats

#### Impersonation of Radius Controllers to trigger resource access

Risk: An attacker can impersonate the Radius Controllers to talk to UCPD.

Impact: The Controller Component and UCPD communicate in order to create/update/delete necessary resources and these resources can also be cloud resources. This can lead to loss of resources or creation/update of existing resources.

Mitigation:

1. **mTLS**: mTLS should be (and is) enabled for communication between the Controller and the UCPD.
2. **Network Policies**: We can add necessary policies that UCPD can only accept communication from the Controller and other important components that it is should be talking to.

#### Compromised TLS Certificates and private keys of the Validating Webhook

Risk: If the TLS certificates and the private keys used by the validating webhook are compromised, an attacker could impersonate the webhook server, intercepting the requests from the Kubernetes API Server or altering the responses.

Impact: The validating webhook functions should return an approval or a validation error to the caller. The impersonater can return incorrect responses to the Kubernetes API Server which could affect the lifecycle of resources.

Risk: If the TLS certificates and the private keys used by the validating webhook are compromised, an attacker could enable unauthorized clients to communicate with the webhook.

Impact: Unauthorized clients wouldn't have any affect in the lifecycle of resources because, even if the webhook returns an approval or a rejection to the unauthorized client, the unauthorized client should also have access to the Kubernetes API Server to further trigger changes in the resource.

Mitigation:

1. **mTLS (Mutual TLS)**: Implement mutual TLS to ensure that both the client (Kubernetes API Server) and the server (Validating Webhook) authenticate each other. This helps in verifying the authenticity of both parties and prevents unauthorized access. Note that mTLS is enabled for the communication between the Validating Webhook and the Kubernetes API Server in Radius as of now.
2. **Network Policies**: Implement necessary Network Policies to ensure that communication between components is restricted to authorized components only. In this case, the Validating Webhook and the Kubernetes API Server should only accept requests from each other.
3. **Strong Certificate Management**: Implement strong certificate management practices, including regular rotation of certificates, using short-lived certificates, and monitoring for certificate anomalies.
4. **Secure Storage**: Store TLS certificates securely using secrets management solutions to prevent unauthorized access. The certificates and the private keys are being kept in a Secret object on Kubernetes.

#### Compromised TLS Certificates and private keys of the Kubernetes API Server

Risk: If the TLS certificates and private keys used by the Kubernetes API Server are compromised, an attacker could impersonate the Kubernetes API Server. This would allow the attacker to intercept, alter, or forge communication between the API Server and other components, including the Validating Webhook and Reconcilers. This could lead to unauthorized access, data breaches, and manipulation of cluster operations.

Impact:

1. The attacker could gain unauthorized access to sensitive data including the data it can get from the etcd if proper policies are not in place.
2. The attacker can manipulate the operations of the cluster, causing unauthorized changes.

Mitigation:

1. **Network Policies**: Implement necessary Network Policies to ensure that communication between components is restricted to authorized components only. For example, the Validating Webhook should only accept requests from the original Kubernetes API Server.
2. **Strong Certificate Management**: Implement strong certificate management practices, including regular rotation of certificates, using short-lived certificates, and monitoring for certificate anomalies.
3. **Secure Storage**: Store TLS certificates securely using secrets management solutions to prevent unauthorized access. The certificates and the private keys are being kept in a Secret object on Kubernetes.

#### Interception and manipulation of the communication between the webhook and the Kubernetes API Server

**Risk:**

1. An attacker could intercept and manipulate the communication between the webhook and the Kubernetes API Server. This could lead to unauthorized access, data breaches, or manipulation of admission decisions, potentially compromising the security and integrity of the Kubernetes cluster.

**Mitigation:**

1. **mTLS (Mutual TLS)**: Implement mutual TLS to ensure that both the client (Kubernetes API Server) and the server (Validating Webhook) authenticate each other. This helps in verifying the authenticity of both parties and prevents unauthorized access. Note that mTLS is enabled for the communication between the Validating Webhook and the Kubernetes API Server in Radius as of now.
2. **Network Policies**: Use Kubernetes Network Policies to restrict the network traffic between the Kubernetes API Server and the webhook. This limits the exposure of the webhook to only trusted sources. Reference: <https://kubernetes.io/docs/concepts/services-networking/network-policies/>.

#### An attacker gaining access to etcd and manually changing resources

Risk: If an attacker gains access to etcd, they can manually change resources stored in the etcd datastore. This could lead to unauthorized access, data breaches, and manipulation of cluster operations.

Mitigation:

1. **Access Controls for etcd**: Implement strict access controls for etcd to ensure that only authorized components can access and modify its data.
2. **Network and Firewall Policies**: Add the necessary Network and Firewall Policies to filter access to the etcd.
3. **Audit Logs**
4. **Encryption**

#### Dangerous data hidden in the Recipe or Deployment resource

Risk: When an external interactor sends a create or update request for a Recipe or Deployment, there is a risk that the interactor could hide dangerous data within the resource. This hidden data could potentially exploit vulnerabilities in the system, leading to security breaches or system malfunctions.

Mitigation:

1. **Use of Kubernetes Parser**: Radius relies on the parser provided by Kubernetes, which is well-tested and maintained. This reduces the risk associated with parsing malicious data.
2. **Avoid Custom Parsers**: By not using custom or third-party parsers, Radius avoids the additional risk and complexity associated with ensuring the security of these parsers.
3. **Fuzz Testing**: If a custom parser were to be used, it would be essential to implement fuzz testing. Fuzz testing involves providing invalid, unexpected, or random data inputs to the parser to identify potential vulnerabilities. For more information on fuzz testing, refer to [OWASP Fuzzing](https://owasp.org/www-community/Fuzzing).

#### Users with access to the webhook server modifying the behavior of the webhook server

Risk: Could an insider with access to the webhook server modify its behavior to approve malicious requests?

Mitigation: Implement strict access controls, audit logs, and monitoring for changes to the webhook server configuration and code.

#### Unauthorized access to the Kubernetes API Server

- **Mitigation**: Implement strong authentication and authorization mechanisms, such as RBAC (Role-Based Access Control) and network policies to restrict access.

#### Security vulnerabilities in the controller code

- **Mitigation**: Conduct regular security audits and code reviews, and follow secure coding practices to minimize vulnerabilities.

## Open Questions

<!--
List any unresolved questions or uncertainties about the threat model. Use this section to gather feedback from experts or team members and to track decisions made during the review process.
-->

## Action Items

1. Check if TLS is enabled for every component to ensure secure communication. Make changes to the necessary components if required.
2. Ensure that all communication uses mTLS (Mutual TLS) to authenticate both the client and server, providing an additional layer of security. Verify that mTLS is correctly configured for all components and endpoints. Make changes to the necessary components if required.
3. Check if RBAC with Least Privilege is configured for every component to ensure that each component has only the permissions it needs to function. Make changes to the necessary components if required.
4. Define and implement necessary Network Policies to ensure that communication is accepted only from expected and authorized components. Regularly review and update these policies to maintain security.
5. Separate and firewall the etcd cluster to ensure the safety of the datastore. Implement network segmentation to isolate the etcd cluster from other components. Configure firewall rules to restrict access to the etcd cluster, allowing only authorized components and administrators to communicate with it. Regularly review and update firewall rules and network policies to maintain security.
6. Containers should run as a non-root user wherever possible to minimize the risks. Check if we can run any of the Radius containers as non-root. Do the necessary updates.

## Review Notes

<!--
Update this section with the decisions and feedback from the threat model review meeting. Document any changes made to the model based on the review.
-->

## References

1. <https://kubernetes.io/blog/2018/07/18/11-ways-not-to-get-hacked>
