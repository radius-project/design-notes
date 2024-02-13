# Application Model promises

* **Status**: Pending
* **Author**: Ryan Nowak (`@rynowak`)

## Overview

Radius provides a runtime-neutral application model that allows the same application definition to be deployed across multiple clouds, on premises, or different choices of infrastructure, and dependencies. We provide *flexible* concepts like recipes, applications, and environments as well as *fixed* concepts like containers.

The hard part for us is to get the design *right*. If concepts like environments and applications are not flexible enough then they aren't reusable and we can't fulfil our promise. If concepts like applications and environments are too flexible then we miss the change to provide guidance and structure to users. For fixed concepts like containers, we need to define an abstraction and give that abstraction a *meaning* that's useful and consistent across platforms.


## Terms and definitions

| Term                   | Definition                                                                                                                                                                                                                                                                                                                                             |
| ---------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| Application model      | An abstact description of a runtime system and its capabilities. This is usually coupled to a runtime system like Kubernetes. An example would be the Kubernetes application model with core concepts like `Namespace`, `Pod`, `Service`, `Secret`. In the case of Radius our application model is *virtual* and doesn't target any specific platform. |
| Application definition | A written and deployable description of an application for a specific application model. An example would be a Helm Chart (Kubernetes application model) or a Bicep file with Radius definitions.                                                                                                                                                      |

## Objectives

The topic of this paper is to define the *promises* Radius makes to users and the *responsibilities* that users must fulfil to use Radius successfully. To describe the promises and responsibilities we adopt the vocabulary defined in [RFC-2119](https://datatracker.ietf.org/doc/html/rfc2119).

This document is a vision/direction statement rather than a design-document for new functionality. As we're working towards serverless application management it's useful for us to write down the promises we're making.

### Goals

- Define the *meaning* of Radius application model concepts in terms of the promises and responsibilities.
- As proof, define the mappings between Radius concepts and runtime platforms.

### Non goals

- Implement new features.
- Add new application model concepts.
- Significantly revise existing features.
- (out of scope) Discuss recipe design topics in significant depth. 

### How we decide

It's useful to write down some principles motivating how we make decisions during application model design. These aren't new ideas, they describe the tradeoffs we make. 

- Operations roles (which have many names) are already capable of managing infrastructure using existing tools like Bicep, Helm, and Terraform. We optimize for operations roles using their existing tools for environment management.
  - Corollary: The primary value of Radius features like environments and recipes is to empower developers with self-service, and by doing-so decrease the burder on operations.
- The Radius application model should define the widely-portable set of useful designs and features. It should be clear to users when a feature is portable or non-portable.
  - Corollary: The application model solves the *lowest-common-denominator* problem by giving users ways to punch through the abstraction when desired.
- Application definitions should be minimally coupled to operational concerns. We leverage *separation-of-concerns* to avoid mixing application and operational configuration.

### User scenarios (optional)

These user scenarios are somewhat abstract but help provide motivation for why this discussion is valuable.

#### Managing Radius Environments

Alfonso manages environment definitions for multiple clouds and runtimes as part of a platform engineering team. He writes Terraform templates that create the supporting infrastructure like AKS, EKS, or ECS clusters. This includes supporting infrastructure like secret stores or diagnostics dashboards. He also includes the relevant recipes for services like AWS MemoryDB or Azure Cache for Redis that the platform team supports. 

These templates are parameterized and used by developers as part of a self-service environment creation workflow. Because of the promises defined by Radius, Alfonso knows these environments will meet the organization's requirements.

#### Deploying Radius Applications

Janine builds business applications using Java, and deploys them with Radius. She's responsible for providing the application definition using Radius concepts like containers, gateways, and recipes. 

These templates are deployed through a CI/CD process to multiple kinds of environments. Because of the promises provided by Radius and the responsibilities fulfiled by Alfonso, the application definition can be deployed to any supported platform.

## Design

### Introducing concepts

*These are the core concepts we'll be working with in this document. Where it's helpful we generalize to simplify the discussion.*

---

**Environment:** A pool of compute, networking, and supporting resources use by potentially many applications. Applications inside the environment share the pool of available compute and exist inside the same network namespace. The *supporting resources* address potentially *many* different needs and scenarios.

For our purposes, a supporting resource of the environment probably has:

- The same lifecycle as the environment.
- Is shared for all applications in the environment (not part of a specific application).
- Is likely not managed by developers.
- Likely has some direct integration with Radius:
  - Storing secrets.
  - Storing diagnostic data.

---

**Application:** A composite grouping of compute and non-compute resources defined by the developer. The application is explictly not tied to access control or to deployment/lifecycle. The application is a boundary for common configuration and metadata. 

The question is often asked, "what does an application *mean*?" Our answer is that the application is whatever the developers say it is.

**Compute Resource:**



### Mapping of concepts

The best way to introduce the topic is probably with a table explaning how Radius concepts map to runtime platforms. This reflects a mixture of the current state and best ideas we have going forward. 


| Platform/Concept                                                                     | Environment                    | Application              | Container                                                                            | Gateway                        |
| ------------------------------------------------------------------------------------ | ------------------------------ | ------------------------ | ------------------------------------------------------------------------------------ | ------------------------------ |
| **Kubernetes (any)** | Compute: `<Cluster>` | Scoping: `Namespace` | Compute: `Deployment`<br/> Identity: `<Workload Identity>`<br/> Network: `Service`<br/> Secret: `Secret`<br/> Storage: `Volume` | Network: `HttpRoute` (Contour) |
| **Kubernetes (AKS)** | Compute: `<AKS Cluster>`<br/> Diagnostics: `<Azure Monitor>`<br/> Identity: `<OIDC config>`<br/> Network: `<VNet>`<br/> Secret: `<KeyVault>`| Scoping: `Namespace` | Compute: `Deployment`<br/> Identity: `<Workload Identity>`<br/> Network: `Service`<br/> Secret: `Secret`<br> Storage: `Volume` | Network: `HttpRoute` (Contour) |
| **Kubernetes (EKS)** | Compute: `<EKS Cluster>`<br/> Diagnostics: `<CloudWatch>`<br/> Identity: `<OIDC config>`<br/> Network: `<VPC/NLB/ELB>`<br /> Secret: `<SecretManager>` | Scoping: `Namespace` | Compute: `Deployment`<br/> Identity: `<IRSA>`<br/> Network: `Service`<br/> Secret: `Secret`<br> Storage: `Volume` | Network: `HttpRoute` (Contour) |
| **Serverless (ECS)** | Compute: `<Cluster>`<br/> Diagnostics: `<CloudWatch>`<br/> Identity: `<OIDC config>`<br/> Network: `<VPC/NLB/ELB>/<CloudMap>`<br /> Secret: `<SecretManager>` | - | Compute: `TaskDefinition/Service`<br/> Identity: `<IAM Role>`<br/> Network: `<ServiceConnectService>`<br/>Secret: `<SecretManager (entries)>`<br/>Storage: `<EBS>` | Network: `<NLB/ALB/Route53 (entry)>`|

## Alternatives considered

<!--
Describe the alternative designs that were considered or should be considered.
Give a justification for why alternative approaches should be rejected if
possible. 
-->

## Test plan

<!--
Include the test plan to validate the features including the areas that
need functional tests.

Describe any functionality that will create new testing challenges:
- New dependencies
- External assets that tests need to access
- Features that do I/O or change OS state and are thus hard to unit test
-->

## Security

<!--
Describe any changes to the existing security model of Radius or security 
challenges of the features. For each challenge describe the security threat 
and its mitigation with this design. 

Examples include:
- Authentication 
- Storing secrets and credentials
- Using cryptography

If this feature has no new challenges or changes to the security model
then describe how the feature will use existing security features of Radius.
-->

## Compatibility (optional)

<!--
Describe potential compatibility issues with other components, such as
incompatibility with older CLIs, and include any breaking changes to
behaviors or APIs.
-->

## Monitoring

<!--
Include the list of instrumentation such as metric, log, and trace to 
diagnose this new feature. It also describes how to troubleshoot this feature
with the instrumentation. 
-->

## Development plan

<!--
Describe how you will deliver your features. This includes aligning work items
to features, scenarios, or requirements, defining what deliverable will be
checked in at each point in the product and estimating the cost of each work
item. Don’t forget to include the Unit Test and functional test in your
estimates.
-->

## Open issues

<!--
Describe (Q&A format) the important unknowns or things you're not sure about. 
Use the discussion to answer these with experts after people digest the 
overall design.
-->
