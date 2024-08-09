# Radius should do a better job classifying container deployment failures

* **Author**: Vinaya Damle (@vinayada1)

## Overview

Currently, when a user starts a deployment, Radius only checks whether the deployment has completed and does not check individual pod and container statuses. In case some pods in the deployment fail to start, Radius has no idea that the deployment has actually failed and therefore the deployment does not fail immediately at this point. Instead, Radius keeps waiting for the deployment to complete till it eventually times out. As a result,
* The user has to wait till the timeout interval for the failure
* The user still has no clue why the deployment failed and the only error seen is "Deployment timed out" which is not useful in debugging the failure.

Radius should do a better job at classifying these failures and reporting them to the user with errors that can assist the user in root causing the deployment failure.

## Terms and definitions

* Readiness Probe - tells you that the application is ready to accept traffic. If this probe fails, the pod status remains running

* Liveness Probe - tells you that the application is up and running. If this probe fails, the pod transitions to CrashLoopBackOff state.

* Informer - is a core concept in Kubernetes that allows client applications to watch and react to changes in the state of resources (e.g., pods, deployments, services) within a Kubernetes cluster.

* Terminal Condition - A terminal condition is a definitive state where we can either declare success or failure whereas a non-terminal state is a state which is transient and we cannot conclude anything in that state. For example, if we cannot find any pods for a deployment, it is a valid transient state and we cannot conclude that the deployment has failed just because right now, there are no pods found.

  In general, there are 3 states while waiting for the deployment to complete:-

    * Succeeded - we can report success as of now
    * Failed - we can report failure as of now
    * Waiting - we do not yet know and need to wait longer for the deployment to enter into a terminal state.

## Objectives

Radius should do a good job at classifying deployment failures and should report success or failure accurately. In case of a failure, Radius should try to best provide information that will help the user to root cause the failure.

> **Issue Reference:** https://github.com/project-radius/radius/issues/5686

### Goals

* Report accurate deployment status to the user - success/failed
* Provide a good error message which will help the user in troubleshooting the failures
* In case of a failure, fail the deployment as early as possible

### Non goals

* This document is not focused on conveying the deployment progress to the user. I have created a separate feature request: https://github.com/project-radius/radius/issues/5934 for this.

### User scenarios

During deployment, the pods may fail to start for several reasons and enter a terminal state. Some of them could be:-

* Crashing docker image
* Non-existent docker image
* Health check fails
* Container requesting too many resources
* Image pre-requisites are not installed e.g. app uses python but python not installed
* Failed service dependencies
* Bad volume mount
* Network issues
* Non-existent secret


## Design

#### When can a deployment be declared as failed?

Why a pod failed to start can be deduced only from the pod status and not the deployment status. Therefore, the general idea is that instead of only waiting for the deployment to complete, Radius can check the pod status and the container statuses within a pod. Therefore, if any pod enters a terminally failed state, we can immediately fail the deployment and return the appropriate error to the user.

The steps to determine success/failure would be:-
* Find the deployment
* Find the current replicaset for the deployment
* Find all pods matching the current replicaset
* For every pod in the deployment, check the pod status and container statuses for all containers within the pod
* If a pod or any of the containers within a pod is in a terminally failed state, fail the deployment and return the reason to the user
* Once all the pods are up and running and the readiness probe has passed, mark the deployment as complete

Advantages:
* Improved error messages to the user
* Ability to fail the deployment sooner

Disadvantages:
* Need to add multiple informers which is more expensive in terms of resources.

### Design details

#### How to watch changes to pod/container/deployment status?

We need to continuously watch all pods inside the deployment to determine if one has entered a terminally failed state. Kubernetes provides an efficient mechanism to watch changes to Kubernetes resources (in our case, pods, deployments) and react to the changes via an Informer.

In addition to the existing deployment informer, we will add a pod informer and a replicaset informer. These informers will help us reactively monitor updates to the pod, deployment and changes to replicaset versions.

#### How to determine the current replicaset for the deployment

The deployment will indicate the current "revision" of the replica set with the deployment.kubernetes.io/revision annotation in the Kubernetes spec.

``````
metadata:
  annotations:
    deployment.kubernetes.io/revision: "1"
``````

This will exist on both the deployment and the replicaset. The replicaset with revision N-1 will be the "old" one. We will match this annotation on the deployment and replicaset to determine the current version.


#### How to determine the pods in the current deployment

To filter out pods that exist in the namespace but do not belong to the current deployment, we will use the IsControlledBy(obj Object, owner Object) function which will check the replicaset pointed to by the OwnerReferences in the pod spec.

#### Failure States

Let's take a look at the different terminal states that we could check to detect pod issues and report deployment failures:-

* Container is in "Terminated" state. This can happen when the app crashes.
* Container is in "Waiting" state and Reason is ErrImagePull" || "CrashLoopBackOff" || "ImagePullBackOff". This can happen due to a failing liveness check or a bad container image or network issues.
* If the pod cannot be scheduled. This can happen when resource requirements (e.g. cpu, memory) specified in the pod spec cannot be met

#### Failures we cannot detect

* A failed liveness probe causes the pod to enter a CrashLoopBackOff state which can be reported as a failure.On the contrary, if the readinessProbe fails, the pod is not ready but is still in Running state. However, pod not ready is not a Terminal state and therefore we cannot report failure. In the future, we could improve this behavior by looking for ways to detect failures when the deployment is stuck in this "Waiting" state where the readiness check fails.

* A pod can briefly enter running state and then crash. If we happen to check the pod status when it is in the "Running" state, we could incorrectly report the deployment as Succeeded. Instead of doing any special handling for this case, we can rely on the user specifying a readiness and liveness probe to make sure that a success is reported only once the pods are fully functional.


### API design (if applicable)
N/A

## Alternatives considered

Instead of informers, we could query the API Server to list the pods, replicasets, etc. However, this can cause throttling issues.

## Test plan

* We will write modular code and write unit tests to test individual functionality. e.g. - getting current replicaset, filtering out pods that do not belong to the deployment, making sure we fail on the different terminal pod status/conditions that we have identified earlier, logic to check if all pods are ready, etc.
* We will add e2e tests which will test the failure conditions such as a failing liveness/readiness probe and a non-existent container image. We will rely on detailed unit tests and will not require to write exhaustive negative functional tests to test the different scenarios.

## Security

N/A

## Monitoring

We will add logs in the Applications RP which can be used to track status transitions for pods, containers, deployment, etc. and diagnose failures.

## Development plan

* Add the informers to the code and the logic to check the status of pods and containers within a pod.
* Add unit tests and functional tests
* Look for potential ways of being able to declare a failure when stuck in the readiness probe failed state

## Open issues

* Should we also communicate the deployment progress? - This will be treated as a separate feature https://github.com/project-radius/radius/issues/5934
* If readiness probe for a pod fails, the pod fails to become ready but this failure is not reflected in the pod/container conditions or status. To be able to better detect a failure in this condition, we could check if there is a way to access the pod events which report a failing health probe. I have created https://github.com/project-radius/radius/issues/5936 to track this effort.
* Currently a deployment times out in 120 seconds due to the async worker timeout. With this:-
  * Is this period sufficient to timeout a deployment?
  * Should this timeout be related to the readiness/liveness probe parameters specified by the user?
  * Can a fat application take longer than 2 minutes to start?