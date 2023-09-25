# Radius should do a better job classifying container deployment failures

* **Status**: Approved
* **Author**: Vinaya Damle (@vinayada1)

## Overview

Currently, if some pods in a deployment fail to start, the Radius deployment does not fail immediately. Instead, it keeps waiting for the deployment to complete and eventually it times out. As a result,

The user has to wait till the timeout interval for the failure
The user still has no clue why the deployment failed and the only error seen is "Deployment timed out" which is not useful in debugging the failure.
Radius should do a better job at classifying these failures.

## Terms and definitions

* Readiness Probe - tells you that the application is ready to accept traffic. If this probe fails, the pod status remains running

* Liveness Probe - tells you that the application is up and running. If this probe fails, the pod transitions to CrashLoopBackoff state.

* Terminal Condition - A terminal condition is a definitive state where we can either declare success or failure whereas a non-terminal state is a state which is transient and we cannot conclude anything in that state. For example, if we cannot find any pods for a deployment, it is a valid transient state and we cannot conclude that the deployment has failed just because right now, there are no pods found.

  In general, there are 3 states while waiting for the deployment to complete:-

    * Succeeded - we can report success as of now
    * Failed - we can report failure as of now
    * Waiting - we do not yet know and need to wait longer for the deployment to enter into a terminal state.

## Objectives

Radius should do a good job at classifying deployments and should report success or failure accurately. In case of a failure, Radius should try to best provide information that will help the user to root cause the failure.

> **Issue Reference:** https://github.com/project-radius/radius/issues/5686

### Goals

* Report accurate deployment status to the user - success/failed
* Provide a good error message which will help the user in troubleshooting the failures
* In case of a failure, fail the deployment as early as possible

### Non goals

* This document is not focussed on conveying the deployment progress to the user. I have created a separate feature request: https://github.com/project-radius/radius/issues/5934 for this.

### User scenarios

During deployment, the pods may fail to start for several reasons and enter a terminal state. Some of them could be:-

Crashing docker image
Non-existent docker image
Health check fails
Container requesting too many resources
Image uses python but python not installed
failed service dependencies
Bad volume mount
Network issues
Non-existent secret

#### When can a deployment be declared as failed?

Why a pod failed to start can be deduced only from the pod status and not the deployment status. Therefore, the general idea is that instead of only waiting for the deployment to complete, Radius can check the pod status and the container statuses within a pod. Therefore, if any pod enters a terminally failed state, we can immediately fail the deployment early and return the appropriate error to the user.

#### Failure States

Let's take a look at the different states that we could check to detect pod issues and report deployment failures:-

Container is in "Terminated" state. This can happen when the app crashes.
Container is in "Waiting" state and Reason is ErrImagePull" || "CrashLoopBackOff" || "ImagePullBackOff". This can happen due to a failing liveness check or a bad container image or network issues.
If the pod cannot be scheduled. This can happen when resource requirements (e.g. cpu, memory) specified in the pod spec cannot be met

#### Failures we cannot detect

* If the readinessProbe fails, the pod is not ready but is in running state. However, pod not ready is not a terminal state and therefore we cannot report failure. On the contrary, a failed liveness probe causes the pod to enter a CrashLoopBackoff state which can be reported as a failure
* A pod can briefly enter running state and then crash. If we happen to check the pod status when it is in the "Running" state, we could incorrectly report the deployment as Succeeded. Instead of doing any special handling for this case, we can rely on the user specifying a readiness and liveness probe to make sure that a success is reported only once the pods are fully functional.


## Design

To do a better job at reporting deployment failures, we will add a pod informer and replicaset informer in addition to the existing deployment informer. When an add/update event occurs on any of these, we will go through the following process:-

* Find the deployment
* Find the current replicaset for the deployment
* Find all pods matching the current replicaset
* For every pod in the deployment, check the pod status and container statuses for all containers within the pod
* If a pod or any of the containers within a pod is in a terminally failed state, fail the deployment and return the reason to the user
* Once all the pods are up and running and the readiness probe has passed, mark the deployment as complete

### Design details

Here, note that if a pod fails to go into a Ready state, it is not a terminal failure. Therefore, we cannot report the deployment as failed.


## Development plan

We will add the informers to the code and the logic to check the status of pods and containers within a pod.

## Open issues

* Should we also communicate the deployment progress? - This will be treated as a separate feature https://github.com/project-radius/radius/issues/5934
* If readiness probe for a pod fails, the pod fails to become ready but this failure is not reflected in the pod/container conditions or status. To be able to better detect a failure in this condition, we could check if there is a way to access the pod events which report a failing health probe.
* The current deployment timeout is 10 minutes. However, we hit the async worker timeout of 120 seconds before. With this:-
  * Is this period sufficient to timeout a deployment?
  * Should this timeout be related to the readiness/liveness probe parameters specified by the user?
  * Can a fat application take longer than 2 minutes to start?