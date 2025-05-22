# Application Autoscaling

## Overview

Autoscaling is a critical capability for modern cloud applications, enabling them to dynamically adjust resources based on demand. Platform engineers and developers need the ability to optimize the resource utilization when deploying applications using Radius across different runtime environments. This document outlines the design and the user experience for configuring autoscaling policies in Radius applications.

## Autoscaling in the cloud-native ecosystem

The following are the most common autoscaling mechanisms available in the cloud-native ecosystem. 

**Kubernetes**
1. **Horizontal Pod Autoscaler (HPA)** - Kubernetes native autoscaling mechanism that scales the number of pods in a deployment based on observed CPU utilization, Memory and other custom metrics. This is the most common autoscaling mechanism used in the Kubernetes ecosystem.
2. **Vertical Pod Autoscaler (VPA)** - Kubernetes native autoscaling mechanism that automatically adjusts the CPU and memory requests of the pods. This is the least common autoscaling mechanism used in the Kubernetes ecosystem as it requires restarting the pods to apply the new resource requests.
3. **KEDA** - Kubernetes Event-driven Autoscaling (KEDA) is an open-source component that enables autoscaling of Kubernetes workloads based on external metrics. KEDA operates on top of the HPA and triggers scaling based on metrics from various sources, such as message queues, databases, or observability platforms. 

**Serverless Container platforms**
1. **Azure Container Instances and Apps** - Azure container instances doesn't provide an inbuilt solution to automatically scale. Container apps provide scaling based on HTTP traffic and other event-driven triggers (KEDA). For web apps, the preferred scaling mechanism is based on HTTP traffic. For event-driven workloads, the preferred scaling mechanism is based on KEDA.
2. **AWS Fargate and App Runner** - AWS Fargate provides autoscaling based on CPU, memory and cloud watch metrics. App runner provides autoscaling based on HTTP traffic 
3. **Google Cloud Run** - Google Cloud Run provides autoscaling based on HTTP traffic.

**Serverless Functions**
Azure Functions, AWS Lambda and Google Cloud Functions provide autoscaling based on the number of incoming requests and other event-driven triggers. They also have options to have some number of warm instances to reduce cold starts.

## Opportunity for Radius 

The main opportunity for Radius is to provide a simple abstraction to configure autoscaling policies enabling `separation of concerns` for the platform engineering and application teams while leveraging the autoscaling mechanisms available in the underlying runtimes. The platform operator should have an ability to configure the autoscaling policies in the environment configuration and the developer should be able to inherit or override the autoscaling policies in the application resource definition.

## Goals

1. **Runtime Agnostic Scaling** - Enable a runtime agnostic way to configure autoscaling policies for applications deployed using Radius.

2. **Unified Scaling Model** - Enable a unified and a consistent scaling policy model that works with different autoscaling mechanisms 

## Out of Scope

This might be out of scope for the initial release but can be considered for future releases.

**KEDA Integration** - Integrate with KEDA to enable event-driven autoscaling for applications deployed using Radius.

## Personas

1. Platform engineer - The platform engineer is responsible for setting up the environment where applications are deployed and ensuring that the applications are running optimally. 

2. Application developer - The application developer is responsible for building and deploying applications using Radius. The developer should be able to leverage the autoscaling policies configured by the platform engineer and override them based on the workloads.

## User experience

### Configuring Autoscaling in container resource type

A developer specifies a simple HPA autoscaling configuration in the container resource definition. Below are the various options to configure autoscaling in the container resource type.

#### Option 1: Adding autoscaling in the compute configuration

This enables to configure compute specific autoscaling policies. For e.g. : If the platform is Kubernetes, the default HPA config can be added to the runtime configuration and for serverless platform, the default HTTP based autoscaling config can be added to the runtime configuration.

Eg : Kubernetes HPA config

```bicep
resource container 'Applications.Containers@2023-10-01-preview' = {
  name: 'myapp'
  properties: {
    image: 'myregistry.azurecr.io/myapp:latest'
    ports: {
    }
    runtimes: {
      kubernetes: {
        autoscaling: {
          minReplicas: 1
          maxReplicas: 10
          cpuUtilization: 50
          memoryLimit: 50      
        }
      }
    }
  }
}
```

Eg : ACA HTTP based autoscaling config

```bicep
resource container 'Applications.Containers@2023-10-01-preview' = {
  name: 'myapp'
  properties: {
    image: 'myregistry.azurecr.io/myapp:latest'
    ports: {
    }
    runtimes: {
      serverless-platforms: {
        autoscaling: {
          minReplicas: 1
          maxReplicas: 10
          http-concurrency: 50  
        }
      }
    }
  }
}
```

Pros: 
1. Easy to understand and configure.
1. Autoscale config is run-time specific and doesn't have to be standardized across runtimes as different platforms have different autoscaling mechanisms. 
1. Enterprises pick platforms based on their application workloads and the underlying platform provides default autoscaling policies based on the workloads. Radius provides a clear abstraction to leverage default autoscaling mechanisms from the underlying platform.

Cons:
1. More configuration to manage across different runtimes.
2. The developer needs to know the underlying platform and the autoscaling policies required for the platform.

#### Option 2: Adding it as part extensions

`extensions` have been the way to provide punch through capabilities which do not change the behaviors of the resource. The following is an example of adding autoscaling as part of the extensions. 

```bicep
resource container 'Applications.Containers@2023-10-01-preview' = {
  name: 'myapp'
  properties: {
    image: 'myregistry.azurecr.io/myapp:latest'
    extensions: [
      {
        kind: 'autoscaling'
        minReplicas: 1
        maxReplicas: 10
        cpuUtilization: 50
        memoryLimit: 50
        httpConcurrency: 50
        }
      }
    ]
  }
}
```

Pros: consistent with existing manual scaling config

Cons: not straightforward or intuitive as user doesn't know which autoscale config is required for the platform ? needs a platform specific discriminator
May be, this is not a developer problem and should be handled by the platform operator. 

#### Option 3: Adding it as top level common property

```bicep
resource container 'Applications.Containers@2023-10-01-preview' = {
  name: 'myapp'
  properties: {
    image: 'myregistry.azurecr.io/myapp:latest'
    autoscaling: {
      minReplicas: 1
      maxReplicas: 10
      hpa: {
        cpuUtilization: 50
        memoryLimit: 50
      }
      http: {
        httpConcurrency: 50
      }  
    }
  }
}
``` 

Cons:same as above, not straightforward or intuitive as user doesn't know which autoscale config is required for the platform ? May be, this is not a developer problem and should be handled by the platform operator.

### Configuring Autoscaling in Radius Environment

A platform engineer or the operator sets up the Radius environment for the applications to be deployed. Today Radius environment contains the configurations for the container runtime, identity provider and the secrets store. Since scaling is a runtime concern, the platform operator would also need an ability to configure the scaling policies in the environment configuration.

```bicep
resource environment 'Applications.Core/environments@2023-10-01-preview' = {
  name: 'myenv'
  properties: {
    compute: {
      kind: 'kubernetes'
      namespace: 'default' 
      identity: {          
        kind: 'azure.com.workload'
        oidcIssuer: oidcIssuer
      }
    autoscaling: {
        minReplicas: 1
        maxReplicas: 10
        cpuUtilization: 50
        memoryLimit: 50
      }
    }
}
```

Pros:
1. Autoscaling is an infrastructure problem, the platform engineer has the ability to configure the autoscaling policies in the environment configuration thus separating the concerns of the platform engineer and the developer. 
1. Dev, test environments wouldn't require autoscaling policies and the platform engineer has more flexibility to configure the scaling policies based on the environment.
1. The platform engineer can configure the scaling policies once and all the applications deployed in the environment will inherit the scaling policies.
1. Developer can focus on the application modelling and not worry about the scaling policies. If needed the developer can override the scaling policies in the application resource definition. Overriding scenario could be for example, if the developer wants to scale based on some events from the Kafka or RabbitMQ modelled in the application definition 

### Configuring Autoscaling as a core resource type

Since Autoscaling is a core functionality of any underlying runtime, it can also be abstracted as a core resource type. The following is an example of configuring autoscaling as a core resource type. 

```bicep 
resource autoscaling 'Applications.Autoscaling@2023-10-01-preview' = {
  name: 'myapp'
  environment: 'myenv'
  properties: {
    minReplicas: 1
    maxReplicas: 10
    hpa: {
      cpuUtilization: 50
      memoryLimit: 50
    }
    http: {
      httpConcurrency: 50
    }
    keda: {
      queue: rabbitmq.QueueName
      queueLength: 10 
    }
  }
}

resource rabbitmq 'Applications.Messaging/rabbitmqQueues@2023-10-01-preview' = {
  name: 'rabbitmq'
  properties: {
    queueName: 'myqueue'
    ....
  }
}
```

Pros:
1. Provides a declarative way to manage the autoscaling policies across different runtimes. Consistent with how Azure, AWS and GCP provides a declarative way to manage the autoscaling config for their compute services.
1. Decouples the autoscaling configuration from the container resource type. Makes it easier to manage updates to autoscaling policies without having to update the container resource type.
1. Gives flexibility as the resource type can be modelled within an environment or an application catering to different enterprise needs. Platform engineers can also control if their developers need to manage the autoscaling policies.
1. For advanced autoscaling using KEDA, the developer can declaratively connect to the messaging service (RabbitMQ, Kafka) and configure the autoscaling policies in the application resource definition.

Cons:
1. Another core resource type to manage for Radius

## Feature Summary
< tbd>

