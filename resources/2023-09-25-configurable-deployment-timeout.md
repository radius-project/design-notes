# Title

* **Status**: Pending/Approved
* **Author**: Vinaya Damle (vinayada1)

## Overview

This proposal is for:-
1. Making deployment timeout configurable by the user at the container level.
2. Looking at pod events to detect readiness probe failure and reporting it as a possible cause of failure if the deployment times out.

## Terms and definitions

* Deployment Timeout - The time after which Radius stops monitoring deployment progress and declares the deployment as failed due to timeout.

* Readiness Probe - tells you that the application is ready to accept traffic. If this probe fails, the pod status remains running

* Terminal Condition - A terminal condition is a definitive state where we can either declare success or failure whereas a non-terminal state is a state which is transient and we cannot conclude anything in that state. For example, if we cannot find any pods for a deployment, it is a valid transient state and we cannot conclude that the deployment has failed just because right now, there are no pods found.

## Objectives

* Make the deployment timeout configurable by the user at the container level.
* Detect readiness probe failure and report that as a possible cause of failure if the deployment times out.

> **Issue Reference:** 
https://github.com/radius-project/radius/issues/6283,
https://github.com/radius-project/radius/issues/6284

### Goals

- Make the deployment timeout configurable by the user at the container level
- If a deployment times out and the readiness probe has failed, report it as a possible cause of failure while reporting the deployment failure

### Non goals

Adding readiness probe failure detection to report the deployment failed as soon as the probe fails.

## Design

We want to address two issues:-
1. The current Radius deployment timeout is 120s which might be too small to deploy big applications.
2. Detection of Readiness probe failure which is not a terminal failure condition.

### Design details

#### Configurable Deployment Timeout

The current Radius deployment timeout is 120s (enforced by the async worker timeout) which might be too small to deploy big applications. The proposal here is to make this timeout configurable by adding an optional property at the container level as below. This will allow the end user to handle scenarios where the application takes a long time to start running.

```
resource container 'Applications.Core/containers@2023-10-01-preview' = {
  name: 'ctnr-ctnr'
  location: location
  properties: {
    application: app.id
    container: {
      image: magpieimage
      ports: {
        ....
      }
      deploymentTimeout: 10
    }
  }
}
```

The deployment timeout property will be used to determine the timeout interval for the container deployment. If this property is not specified, a default value of 120s will be used.

We should also make the async worker timeout dynamic and dependent on this property so that it is always larger than this value. This will ensure that deployment will not fail before the time interval specified by the user.

#### Readiness Probe Failure Detection

When the readiness check for a pod fails, no further traffic is sent to the pod. However, the pod does not restart. Due to this, it is hard to determine the readiness check failure as a terminal condition.

I ran an experiment to detect readiness probe failures by watching pod events and report the deployment as failed as soon as the readiness probe failed. I noticed that many of the functional tests started failing due to readiness check failure. I tweaked the readinessProbe parameters of the container to increase the probe failure threshold, initial period, etc. and that was sufficient for the readiness probe to pass. The functional test applications are trivial and still the default readiness probe parameters seem insufficient.

There are several options here:-
1. Increase the default readiness probe parameter thresholds. The disadvantages are:-
  - If we increase the default initialDelay, then no traffic will be sent to the pod for a longer interval in all cases.
  - If we increase the default failure threshold, it will take longer for the probe to fail for every container.
  - Inspite of increasing default values, there is still no guarantee that the values will work for all container applications.
2. We could rely on the user to specify the readiness probe parameters. However, the disadvantage is that it will be a burden on the end users to specify these  parameters to ensure a successful deployment.
3. If a readiness probe failure is detected, retry a fixed number of times before failing the deployment. This increases the chance of success but still with no guarantee that the number of retries will work for all cases.
4. Do not fail the deployment due to readiness check failure and instead let the deployment timeout. Report that the readiness check had failed at the time of failing the deployment which could serve as a pointer to the user for troubleshooting the failure. The disadvantage is that the deployment will take longer to fail.

I am proposing option 4 because inspite of its disadvantages, it is still a deterministic option which will work the same way for all container applications.


## Alternatives considered

For readiness probe failure detection:-
1. Increase the default readiness probe parameter thresholds.
2. We could rely on the user to specify the readiness probe parameters.
3. If a readiness probe failure is detected, retry a fixed number of times before failing the deployment.

## Test plan

1. Ensure that the configurable deployment timeout is being enforced.
2. Ensure that the async worker timeout is greater than the user specified timeout
3. Specify a bad readiness probe so that it fails. The deployment will eventually timeout and the error message will mention about a failing readiness check.

## Security

NA

## Compatibility (optional)

NA

## Monitoring

We will add logs to log the user specified timeout value, the dynamically calculated async worker timeout. For the readiness check failure detection, we will report the timed out deployment as:-

"Deployment Timed Out. Possible Causes - Readiness check failed"

## Development plan

1. Implement configurable timeout
2. Add a pod events informer to detect readiness check failure
3. If a deployment times out, make changes to look at pod events to determine possible causes of failure and report them to the user.

## Open issues

NA