# Container Resource Type Definition

* **Author**: Zach Casper (@zachcasper)

## Overview

The compute extensibility project is implementing Recipe-backed Resource Types for the core Radius Resource Types including Containers, Volumes, Secrets, and Gateways. As part of this effort, Recipes are being developed replacing the imperative code in Applications RP. Because of this, we are taking the opportunity to re-examine the schema and make adjustments as needed. 

## Objectives

The objective of this document is to define version two of the Containers Resource Type. 

### Goals

* Ensure that the Containers Resource Type feels familiar with both developers and platform engineers with experience with Kubernetes. As such, the Containers Resource Type definition is Kubernetes-first.
* Enable developers to use the Containers Resource Type for the vast majority of their use cases without platform engineers needing to modify the Resource Type definition. Concretely, all developer-oriented properties of the Kubernetes Pod and Deployment resources are available in Containers.
* Ensure Containers is modeled such that other container platforms can be implemented via Recipes in the future. This includes AWS ECS, Azure Container Apps, Azure Container Instances, and Google Cloud Run. Containers property names should be similar to other platforms as much as possible.
* Any modifications to the Containers Resource Type should not drive changes within Radius itself. All changes should only affect the implementation of the Containers Recipe.

### Non goals

This document is focused solely on the Containers Resource Type. Volumes, Secrets, and Gateways are discussed elsewhere.

## Limitations of `Applications.Core/containers`

The current version of Containers has several limitations:

### Supports only one container

The Containers Resource Type has a single `container` property. This is in stark contrast to all other container platforms which accept an array of containers. Multiple containers within a larger construct (such as a a Kubernetes Pod or ECS Task) is quite common. Observability tools such as Datadog offer sidecar containers. Container security tools such as Aqua Security are also used as sidecars.  These are platform engineering examples, but the existence of multiple containers in Kubernetes and other platforms allows developers to use this pattern in their applications as well.

### Does not support init containers

Since Containers does not support multiple containers, obviously it does not support any type of sequencing or startup dependencies. Kubernetes, ACA, ACI, and Cloud Run all support init containers. ECS has a more flexible model and offers the ability to specify arbitrary dependencies on containers within a Task.

### Does not support resource requests and limits

Containers has no properties for specifying CPU and memory requirements. All other container platforms support at least requests and most support setting limits (ECS is the exception). Furthermore, most other platforms have enhanced the resource types to include GPUs for AI workloads.

### Does not support autoscaling

Containers only offers the ability to specify a fixed number of replicas. There is no support for autoscaling.

## Proposed New Capabilities

### Addition 1: The `containers` property is now a map

The Containers Resource Type will have a `containers` map property instead of a `container`. Multiple containers is supported by all container platforms:

* Kubernetes: `pod.spec.containers[]`
* ECS Task: `TaskDefinition.containerDefinitions[]`
* ACI Container Group: `containerGroups.containers[]`
* ACA Container App: `containerApps.template.container[]`
* Google Cloud Run: `Service.spec.template.spec.containers[]`

The implications of this are not as impactful as one may think:

* The application graph does not change. Resources are still connected to the parent Containers resource. Environment variables are still created in each of the containers just as they do today in the single container (unless `disableDefaultEnvVars` is true).
* A Kubernetes Service is created for each container that exposes a container port just as today. The only difference is that now multiple Services may be created, one for each container port.
* This forces volumes to be refactored which will enable storage sharing between containers.

> [!CAUTION]
>
> The inclusion of multiple containers raises the question of naming of Containers. As you can see in this document, great pain has been taken to refer to the Containers Resource Type and the `containers` property distinctly. This is why Kubernetes has the Pod term, ECS has the Task term, ACA has the Application term, ACI has the Container Group term, and Cloud Run has the Service and Job term. No change is being proposed today, but Radius is an exception amongst its peers which may be a caution sign.

### Addition 2: Addition of init containers

The Containers Resource Type will have an `initContainers` property. This is only a special instance of multiple containers.

Given an init container is just a container, this highlights one of the limitations of today's Resource Type definition YAML implementation. To model both a container and an initContainer, the schema for each of these properties must be duplicated. This makes the YAML file very unwieldy. See [Feature request: Support referring to existing object schemas in Resource Type definition YAML files #10276](https://github.com/radius-project/radius/issues/10276).

### Addition 3: New resource request and limits properties

The Containers Resource Type does not have the ability to set CPU and memory request and limits. These are added on the `containers` property.

The impact of this on the Recipe is minimal. These property are one-to-one match and no special testing is needed.

```yaml
resources:
  type: object
  description: (Optional) Compute resource requirements for the container.
  properties:
    requests:
      type: object
      description: (Optional) Requests define the minimum amount of CPU or memory that is required by the container.
      properties:
        cpu:
          type: float
          description: (Optional) The minimum number of vCPUs required by the container. `0.1` results in one tenth of a vCPU being reserved.
        memoryInMib:
          type: integer
          description: (Optional) The minimum amount of memory required by the container in MiB. `1024` results in 1 GiB of memory being reserved.
      limits:
        cpu:
          type: float
          description: (Optional) The maximum number of vCPUs which can be used by the container.
        memoryInMiB:
          type: integer
          description: (Optional) The maximum amount of memory which can be used by the container in MiB.
```

### Addition 4: New autoscaling rules

The Containers Resource Type only has a replicas property (`extensions.manualScaling.replicas`) and no way to specify autoscaling. Containers will have a new `autoscaling` property.

This new property will require the recipe to configure autoscaling. On Kubernetes, this entails creating a horizontal pod autoscaler. The Recipe design will address whether this new functionality is implemented now or in the future.

```yaml
autoScaling:
  type: object
  properties:
    maxReplicas:
      type: integer
      description: (Optional) The maximum number of replicas for the autoscaler.
    metric:
      type: object
      description: (Required) The metric to measure and target used to autoscale. 
      additionalProperties:
        kind:
          type: string
          enum: [cpu, memory, custom]
          description: (Required) The metric to measure. 
        customMetric: 
          type: string
          description: (Optional) The custom metric exposed by the application. Implementation specific. See platform engineer for further guidance.
        target:
          type: object
          description: (Required) When the metric exceeds the target value specified, autoscaling is triggered. Only one target value can be specified dependent upon the type.
          properties:
            averageUtilization:
              type: integer
              description: (Optional) The average CPU or memory utilization across all containers expressed as a percentage. Kind must be CPU or memory.
            averageValue:
              type: integer
              description: (Optional) The average value of the metric as a quantity.
            value:
              type: integer
              description: (Optional) The absolute value of the metric as a quantity.
      required: [kind, target]
    required: [metric]
```

## Proposed Changes

### Change 1: Refactored `volumes`

Volumes on the new Container Resource Type is refactored with these goals:

* Support sharing storage between multiple containers
* Aligning property names (using `emptyDir` for ephemeral storage for example)
* Mounting Radius Secrets not just Azure Key Vaults
* Alignment with other container platforms

The `volumes` property on the container property resource will be replaced with a `volumeMount` property on the container property and a `volumes` at the Container Resource Type level.

```yaml
containers:
    type: object
    additionalProperties:
      ...
      volumeMounts:
        type: object
        additionalProperties:
          type: object
            properties:
              mountPath:
                type: string
              volumeName:
                type: string
                description: (Required) The name of the volume defined in Containers.properties.volumes.
            required: [mountPath, volumeName]
volumes:
  type: object
    additionalProperties:
      type: object
      properties:
        persistentVolumeId: 
          type: string
          description: (Optional) The Radius PersistentVolume resource ID.
        secretId:
          type: string
          description: (Optional) The Radius Secret resource ID.
        emptyDir:
          type: null
          description: (Optional) An empty ephemeral directory.
```

The persistent volume is a separate resource type.

The `volumeMounts` with a separate `volumes` property is the same pattern for:

| Container Platform | Volume Mounts                                                | Volumes                                      |
| ------------------ | ------------------------------------------------------------ | -------------------------------------------- |
| ACA                | `containerApps.properties.configurations.containers.volumeMounts` | `containerApps.properties.templates.volumes` |
| ACI                | `containerGroups.properties.containers.volumeMounts`         | `containerGroups.properties.volumes`         |
| Cloud Run:         | `Service.spec.template.spec.containers.volumeMounts`         | `Service.spec.template.spec.volumes`         |
| ECS                | `TaskDefinition.ContainerDefinitions.mountPoints`            | `TaskDefinition.volumes`                     |
| Kubernetes         | `PodSpec.containers.volumeMounts`                            | `PodSpec.volumes`                            |

In the future, we can enhance these types with:

- An NFS mount point
- A connected resource properties volume type at provides the connected resource properties via the file system, similar to mounting a secret

### Change 2: The `command` and `args` properties are string arrays

This is a small change to make Radius consistent with other container platforms. The `command` and `args` properties should be an array of strings. This 

### Change 3: Other small changes

* The `env.valueFrom.secretRef` property is renamed `secretKeyRef` to be consistent with Kubernetes. 
* The `env.valueFrom.secretRef.source` property is renamed `secretId` to be the more descriptive `secretId`.
* There are several small changes to readinessProbe and livenessProbe to be consistent with Kubernetes. 

## Removed Functionality

### Removal 1: Removal of the `imagePullPolicy`

The Container Resource Type has an `imagePullPolicy` today. However, no other container platform other than Kubernetes supports this option. This property is being removed. Platform engineers can set this in a Recipe if needed. A common use case would be to set `imagePullPolicy: Always` in a test environment. 

### Removal 2: Removal of `iam` property on connections

It is unclear that this is needed. If the use case is identified it can be added back in.

### Removal 3: Removal of the Kubernetes metadata extension

The Kubernetes metadata extension allows developers to set Kubernetes labels and annotations on deployed resources. Setting labels and annotations is primarily a platform engineering function so this capability moves to the Recipe. 

## Open Questions



## Appendix 1: Container platform API references

- [Kubernetes Pod](https://kubernetes.io/docs/reference/kubernetes-api/workload-resources/pod-v1/)
- [ECS TaskDefinition](https://docs.aws.amazon.com/AmazonECS/latest/APIReference/API_TaskDefinition.html)
- [ACI Container Group](https://learn.microsoft.com/en-us/azure/templates/microsoft.containerinstance/containergroups)
- [ACA Container App](https://learn.microsoft.com/en-us/azure/templates/microsoft.app/containerapps?pivots=deployment-language-bicep)
- [Google Cloud Run](https://cloud.google.com/run/docs/reference/yaml/v1)

## Appendix 2: Kubernetes feature not exposed by the Container resource type

These Deployment and Pod properties are intentionally not included in the Containers Resource Type. Broadly, these properties are platform-level properties and not application-level.

### DeploymentSpec.minReadySeconds (int32)
Minimum number of seconds for which a newly created pod should be ready without any of its container crashing, for it to be considered available. Defaults to 0 (pod will be considered available as soon as it is ready)

### DeploymentSpec.strategy (DeploymentStrategy)

The deployment strategy to use to replace existing pods with new ones.

### DeploymentSpec.revisionHistoryLimit (int32)

The number of old ReplicaSets to retain to allow rollback. This is a pointer to distinguish between explicit zero and not specified. Defaults to 10.

### DeploymentSpec.progressDeadlineSeconds (int32)

The maximum time in seconds for a deployment to make progress before it is considered to be failed. The deployment controller will continue to process failed deployments and a condition with a ProgressDeadlineExceeded reason will be surfaced in the deployment status. Note that progress will not be estimated during the time a deployment is paused. Defaults to 600s.

### DeploymentSpec.paused (boolean)

Indicates that the deployment is paused.

### PodSpec.initContainers ([]Container)

List of initialization containers belonging to the pod. Init containers are executed in order prior to containers being started. If any init container fails, the pod is considered to have failed and is handled according to its restartPolicy. The name for an init container or normal container must be unique among all containers. Init containers may not have Lifecycle actions, Readiness probes, Liveness probes, or Startup probes. The resourceRequirements of an init container are taken into account during scheduling by finding the highest request/limit for each resource type, and then using the max of that value or the sum of the normal containers. Limits are applied to init containers in a similar fashion. Init containers cannot currently be added or removed. Cannot be updated. More info: https://kubernetes.io/docs/concepts/workloads/pods/init-containers/

### PodSpec.ephemeralContainers ([]EphemeralContainer)

List of ephemeral containers run in this pod. Ephemeral containers may be run in an existing pod to perform user-initiated actions such as debugging. This list cannot be specified when creating a pod, and it cannot be modified by updating the pod spec. In order to add an ephemeral container to an existing pod, use the pod's ephemeralcontainers subresource.

### PodSpec.imagePullSecrets ([]LocalObjectReference)

ImagePullSecrets is an optional list of references to secrets in the same namespace to use for pulling any of the images used by this PodSpec. If specified, these secrets will be passed to individual puller implementations for them to use. More info: https://kubernetes.io/docs/concepts/containers/images#specifying-imagepullsecrets-on-a-pod

### PodSpec.enableServiceLinks (boolean)

EnableServiceLinks indicates whether information about services should be injected into pod's environment variables, matching the syntax of Docker links. Optional: Defaults to true.

### PodSpec.os (PodOS)

Specifies the OS of the containers in the pod. Some pod and container fields are restricted if this is set.

### PodSpec.volumes ([]Volume)

List of volumes that can be mounted by containers belonging to the pod. More info: https://kubernetes.io/docs/concepts/storage/volumes

### All  scheduling properties in the PodSpec

- nodeSelector
- nodeName
- affinity
- tolerations
- schedulerName
- runtimeClassName
- priorityClassName
- priority
- preemptionPolicy
- topologySpreadConstraints
- overhead

### All lifecycle policies in the PodSpec

- restartPolicy
- terminationGracePeriodSeconds
- activeDeadlineSeconds
- readinessGates

### Aall host and name resolution properties in the PodSpec

- hostname
- setHostnameAsFQDN
- subdomain
- hostAliases
- dnsConfig
- dnsPolicy

### All hosts namespaces properties in the PodSpec

- hostNetwork
- hostPID
- hostIPC
- shareProcessNamespace

### All service account properties in the PodSpec

- serviceAccountName
- automountServiceAccountToken

### All security context properties in the PodSpec

- securityContext

### PodSpec.containers.imagePullPolicy

The imagePullPolicy is only implemented by Kubernetes. It is not available in ECS, ACI, ACA, or Google Cloud Run. The ECS agent on the instance can be customized with ECS_IMAGE_PULL_BEHAVIOR but this is not available to the developer. 

### PodSpec.containers.ports.hostIP 

What host IP to bind the external port to.

### PodSpec.containers.ports.hostPort (int32)

Number of port to expose on the host. If specified, this must be a valid port number, 0 < x < 65536. If HostNetwork is specified, this must match ContainerPort. Most containers do not need this.

### PodSpec.containers.env.valueFrom.configMapKeyRef 

Selects a key of a ConfigMap.

### PodSpec.containers.env.valueFrom.fieldRef

Selects a field of the pod: supports metadata.name, metadata.namespace, metadata.labels['\<KEY>'], metadata.annotations['\<KEY>'], spec.nodeName, spec.serviceAccountName, status.hostIP, status.podIP, status.podIPs.

### PodSpec.containers.env.valueFrom.resourceFieldRef 

Selects a resource of the container: only resources limits and requests (limits.cpu, limits.memory, limits.ephemeral-storage, requests.cpu, requests.memory and requests.ephemeral-storage) are currently supported.

### PodSpec.containers.envFrom

List of sources to populate environment variables in the container. The keys defined within a source must be a C_IDENTIFIER. All invalid keys will be reported as an event when the container is starting. When a key exists in multiple sources, the value associated with the last source will take precedence. Values defined by an Env with a duplicate key will take precedence. Cannot be updated.

### PodSpec.containers.volumeMounts.mountPropagation

mountPropagation determines how mounts are propagated from the host to container and the other way around. When not set, MountPropagationNone is used. This field is beta in 1.10. When RecursiveReadOnly is set to IfPossible or to Enabled, MountPropagation must be None or unspecified (which defaults to None).

### PodSpec.containers.volumeMounts.recursiveReadOnly (string)

RecursiveReadOnly specifies whether read-only mounts should be handled recursively.

### PodSpec.containers.volumeMounts.subPath (string)

Path within the volume from which the container's volume should be mounted. Defaults to "" (volume's root).

### PodSpec.containers.volumeMounts.subPathExpr (string)

Expanded path within the volume from which the container's volume should be mounted. Behaves similarly to SubPath but environment variable references $(VAR_NAME) are expanded using the container's environment. Defaults to "" (volume's root). SubPathExpr and SubPath are mutually exclusive.

### PodSpec.containers.volumeDevices

volumeDevices is the list of block devices to be used by the container.

### PodSpec.containers.resources.claims

Claims lists the names of resources, defined in spec.resourceClaims, that are used by this container.

### PodSpec.containers.resizePolicy

Resources resize policy for the container.

### PodSpec.containers.lifecycle.postStart 

PostStart is called immediately after a container is created. If the handler fails, the container is terminated and restarted according to its restart policy. Other management of the container blocks until the hook completes. More info: https://kubernetes.io/docs/concepts/containers/container-lifecycle-hooks/#container-hooks

### PodSpec.containers.lifecycle.preStop (LifecycleHandler)

PreStop is called immediately before a container is terminated due to an API request or management event such as liveness/startup probe failure, preemption, resource contention, etc. The handler is not called if the container crashes or exits. The Pod's termination grace period countdown begins before the PreStop hook is executed. Regardless of the outcome of the handler, the container will eventually terminate within the Pod's termination grace period (unless delayed by finalizers). Other management of the container blocks until the hook completes or until the termination grace period is reached. More info: https://kubernetes.io/docs/concepts/containers/container-lifecycle-hooks/#container-hooks

### PodSpec.containers.lifecycle.stopSignal (string)

StopSignal defines which signal will be sent to a container when it is being stopped. If not specified, the default is defined by the container runtime in use. StopSignal can only be set for Pods with a non-empty .spec.os.name

### PodSpec.containers.terminationMessagePath (string)

Optional: Path at which the file to which the container's termination message will be written is mounted into the container's filesystem. Message written is intended to be brief final status, such as an assertion failure message. Will be truncated by the node if greater than 4096 bytes. The total message length across all containers will be limited to 12kb. Defaults to /dev/termination-log. Cannot be updated.

### PodSpec.containers.terminationMessagePolicy (string)

Indicate how the termination message should be populated. File will use the contents of terminationMessagePath to populate the container status message on both success and failure. FallbackToLogsOnError will use the last chunk of container log output if the termination message file is empty and the container exited with an error. The log output is limited to 2048 bytes or 80 lines, whichever is smaller. Defaults to File. Cannot be updated.

### PodSpec.containers.startupProbe (Probe)

StartupProbe indicates that the Pod has successfully initialized. If specified, no other probes are executed until this completes successfully. If this probe fails, the Pod will be restarted, just as if the livenessProbe failed. This can be used to provide different probe parameters at the beginning of a Pod's lifecycle, when it might take a long time to load data or warm a cache, than during steady-state operation. This cannot be updated. More info: https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle#container-probes

### PodSpec.containers.restartPolicy (string)

RestartPolicy defines the restart behavior of individual containers in a pod. This field may only be set for init containers, and the only allowed value is "Always". For non-init containers or when this field is not specified, the restart behavior is defined by the Pod's restart policy and the container type. Setting the RestartPolicy as "Always" for the init container will have the following effect: this init container will be continually restarted on exit until all regular containers have terminated. Once all regular containers have completed, all init containers with restartPolicy "Always" will be shut down. This lifecycle differs from normal init containers and is often referred to as a "sidecar" container. Although this init container still starts in the init container sequence, it does not wait for the container to complete before proceeding to the next init container. Instead, the next init container starts immediately after this init container is started, or after any startupProbe has successfully completed.

### All PodSpec.containers security context properties

- securityContext

### All PodSpec.containers debudding properties

- stdin
- stdinOnce
- tty
