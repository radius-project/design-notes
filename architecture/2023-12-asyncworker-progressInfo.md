# Title

* **Status**: Pending
* **Author**: Vinaya Damle (@vinayada1)

## Overview

Currently async worker has no knowledge of the operation progress till the operation is complete. If the operation times out, there is no mechanism in the current design that any information about the operation progress so far can be communicated to the user. As a result, the user only sees the error message that the operation timed out.

The proposal here is to add a mechanism so that the running operation can send out progress information to the async worker. As a result, when the deployment operation eventually times out, along with the timed out message, the user will also be able to see the operation progress so far in the error message which could help in troubleshooting the deployment failure.

## Terms and definitions

AsyncWorker - The current architecture which assigns workers to process queued long-running jobs asynchronously.

## Objectives

> **Issue Reference:** https://github.com/radius-project/radius/issues/6835

### Goals
- Enable reporting of operation progress in async worker

### Non-Goals
- Modify reporting of operation progress in CLI

### User scenarios (optional)

#### User story 1

The readiness probe for the application pod fails. This is a non-terminal failure condition. The current code keeps waiting for the deployment to complete and eventually the deployment times out. The error message to the user is "Deployment timed out after xxx seconds".

If we implement https://github.com/radius-project/radius/issues/6284 and look at pod events, we can detect the readiness probe failure. However, we still need a way to report this information to the user.

#### User story 2

## Design

In the current design, the following is used to track operation status:-

```
// AsyncOperationStatus represents an OperationStatus resource.
type AsyncOperationStatus struct {
	// Id represents the async operation id.
	ID string `json:"id,omitempty"`

	// Name represents the async operation name and is usually set to the async operation id.
	Name string `json:"name,omitempty"`

	// Status represents the provisioning state of the resource.
	Status ProvisioningState `json:"status,omitempty"`

	// StartTime represents the async operation start time.
	StartTime time.Time `json:"startTime,omitempty"`

	// EndTime represents the async operation end time.
	EndTime *time.Time `json:"endTime,omitempty"`

	// Error represents the error occurred during provisioning.
	Error *ErrorDetails `json:"error,omitempty"`
}
```

### Design details

We will add the following field to AsyncOperationStatus to report the operation progress:-

```
// AsyncOperationStatus represents an OperationStatus resource.
type AsyncOperationStatus struct {
  ....

  // ProgressEvents represents the progress of the async operation.
  // The events are represented as a map of key-value pairs where the key is 
  // the event type such as Info, Error and the value is the event message.
  ProgressEvents map[string]string `json:"progressEvents,omitempty"`

  ....
}
```

In the ProgressEvents map, the key represents the event type - Info or Error and the value represents the message string. An info event could be something like "All containers for pod are ready". The readiness probe failure described earlier could be reported as an error event.

When the operation times out, with this change, the user should now be able to see an error message similar to:-
"Operation (APPLICATIONS.CORE/CONTAINERS|PUT) has timed out because it was processing longer than xx s. Progress events: 
Info - Container ctnr-bad-readiness is running but not ready
Error - Container failed readiness probe. Reason: Unhealthy, Message: Readiness probe failed: Get \"http://10.244.0.10:5000/bad\": dial tcp 10.244.0.10:5000: connect: connection refused"

### API design (if applicable)

NA

## Alternatives considered

The following field in the AsyncOperationStatus is not really being used right now:-

```
  // Error represents the error occurred during provisioning.
  Error *ErrorDetails `json:"error,omitempty"`
```
We could repurpose it to report failures on the async operation. However,  adding an explicit field on the status to report progress seems like a better option to report even informational progress and not just errors.

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

NA


## Monitoring

NA

## Development plan

- Implement the design
- Unit tests

## Open issues

NA