# Adding ProvisioningStatus field to the OutputResource

* **Status**: In Progress
* **Author**: Nithya Subramanian (@nithyatsu)

## Overview

Output resources are the actual platform specific resources that get deployed in order to deploy a Radius resource. For example to deploy a container as part of radius application deployment on kubernetes, we deploy a kubernetes ServiceAccount, Role, RoleBinding and Deployment Output resources.

There are a couple of problems related to how we manage output resources today -

1. Output resources do not have a ProvisioningStatus property. Instead, if any of the output resource deployment fails, we set the Provisioning state of the owning radius resource accordingly and do not store the output resources. With this approach, we lose the information on which output resources failed deployment since we aggregate failures into the owning Radius resource's provisioningStatus property. 

2. We do not record any of the output resources as part of the radius resource whose deployment failed, although we deployed them with an errored status, leading to orphaned platform specific resources which should be manually cleaned up.

This design proposes addition of ProvisioningStatus to outputResources so that, 
when we track these resources in case of both successful and failed deployment to avoid orphaning resources, there will be clearer indication of what lead to failed deployment to the user. 


## Terms and definitions

| Term | Definition |
|---|---|
| OutputResource | one or more platform specific resources that make up a Radius resource |
| Resource | Radius resource |
| ProvisioningStatus | Indicates whether the resource was provisioned successfully |



## Objectives

> **Issue Reference:**

<https://github.com/radius-project/radius/issues/7052>

### Goals

The goal is to record the output resources whether or not deployment succeeds, so that radius can still manage the cleanup of resources it failed to deploy successfully. 

With this change, for both successful and failure scenarios, a radius resource will look as below. When the resource.properties.provisisoningState is failure but populated with output resources that look correct, the user can be confused. Therefore we should add a provisioningState to the OutputResource to make the situation clearer. 

We could begin with a simple introduction of property as Succeeded versus Failed based on deployment returning an error. 



***Example with current scenario***

```
rad resource list containers -a allinoneapp -o json                
[
  {
    "id": "/planes/radius/local/resourcegroups/randomgroup/providers/Applications.Core/containers/allinonecontainer",
    "location": "global",
    "name": "allinonecontainer",
    "properties": {
      "application": "/planes/radius/local/resourcegroups/randomgroup/providers/Applications.Core/applications/allinoneapp",
      :
      "provisioningState": "Succeeded",  
      "status": {
        "outputResources": [
          {
            "id": "/planes/kubernetes/local/namespaces/allinoneenv-allinoneapp/providers/core/ServiceAccount/allinonecontainer",
            "localId": "ServiceAccount"
          },
          {
            "id": "/planes/kubernetes/local/namespaces/allinoneenv-allinoneapp/providers/rbac.authorization.k8s.io/Role/allinonecontainer",
            "localId": "KubernetesRole"
          },
          {
            "id": "/planes/kubernetes/local/namespaces/allinoneenv-allinoneapp/providers/rbac.authorization.k8s.io/RoleBinding/allinonecontainer",
            "localId": "KubernetesRoleBinding"
          },
          {
            "id": "/planes/kubernetes/local/namespaces/allinoneenv-allinoneapp/providers/apps/Deployment/allinonecontainer",
            "localId": "Deployment"
          },
          {
            "id": "/planes/kubernetes/local/namespaces/allinoneenv-allinoneapp/providers/core/Service/allinonecontainer",
            "localId": "Service"
          }
        ]
      }
      :
]
```

### Additional enhancement

We could also encapsulate the error message upon deployment of an output resource as one of its property. We could also map the provisioning status to the exact value to make it more informative. These two fields could then be used to to provide clearer communication to the user on what has gone wrong. 

### Non goals

* No changes to any of the existing APIs. The APIs will automatically pick up the changes.

### User scenarios (optional)

#### User story 1

As a Radius User/Operator, I would like to get the information of a resource. If this resource provisioning has failed, I would like to know which of the out resources associated with the resource could not be deployed. 

#### User story 2

As a Radius User/Operator, I would like to be able to issue a `rad app delete` -a myapp to delete all resources provisioned and deployed by radius, even if the deployment of some of these resources had issues. 

## Design

Changes involve adding an attribute in resources.tsp, regenerating swagger files, handling the new attribute in convertors. 


### API design

***Model changes***

Addition of provisioningStatus to OutputResource type in resources.tsp

`typespec``

@doc("Properties of an output resource.")
model OutputResource {
  @doc("The logical identifier scoped to the owning Radius resource. This is only needed or used when a resource has a dependency relationship. LocalIDs do not have any particular format or meaning beyond being compared to determine dependency relationships.")
  localId?: string;

  @doc("The UCP resource ID of the underlying resource.")
  id?: string;

  @doc("Determines whether Radius manages the lifecycle of the underlying resource.")
  radiusManaged?: boolean;

  @doc("Determines whether the resource was provisioned successfully")
  provisioningStatus: ProvisioningStatus
}

@doc("Provisioning Status of Output Resource")
enum ProvisioningStatus {
  @doc("The resource is successfully provisioned")
  Succeeded,

  @doc("The resource provisioning failed")
  Failed,
}
`typespec``

***Example with proposed change***
rad resource list containers -a allinoneapp -o json                
[
  {
    "id": "/planes/radius/local/resourcegroups/randomgroup/providers/Applications.Core/containers/allinonecontainer",
    "location": "global",
    "name": "allinonecontainer",
    "properties": {
      "application": "/planes/radius/local/resourcegroups/randomgroup/providers/Applications.Core/applications/allinoneapp",
      :
      "provisioningState": "Succeeded",  ### or "Failed/ Non-Success status" 
      "status": {
        "outputResources": [
          {
            "id": "/planes/kubernetes/local/namespaces/allinoneenv-allinoneapp/providers/core/ServiceAccount/allinonecontainer",
            "localId": "ServiceAccount"
            "provisioningState": "Succeeded"
          },
          {
            "id": "/planes/kubernetes/local/namespaces/allinoneenv-allinoneapp/providers/rbac.authorization.k8s.io/Role/allinonecontainer",
            "localId": "KubernetesRole"
            "provisioningState": "Succeeded"
          },
          {
            "id": "/planes/kubernetes/local/namespaces/allinoneenv-allinoneapp/providers/rbac.authorization.k8s.io/RoleBinding/allinonecontainer",
            "localId": "KubernetesRoleBinding"
            "provisioningState": "Succeeded"
          },
          {
            "id": "/planes/kubernetes/local/namespaces/allinoneenv-allinoneapp/providers/apps/Deployment/allinonecontainer",
            "localId": "Deployment"
            "provisioningState": "Failed"
          },
          {
            "id": "/planes/kubernetes/local/namespaces/allinoneenv-allinoneapp/providers/core/Service/allinonecontainer",
            "localId": "Service"
            "provisioningState": "Succeeded"
          }
        ]
      }
      :
]
```


## Alternatives considered

NA

## Test plan

1. Updating conversion unit tests to make sure the new attribute is handled correctly
2. Updating some API unit tests to see if the information is being populated on the resource level.

## Monitoring

N/A

## Development plan

There is an in-progress PR which brings in the proposed changes as part of fix for rad app delete. 
https://github.com/radius-project/radius/issues/7052

## Open issues

NA


