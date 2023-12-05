# Title

* **Status**: Pending
* **Author**: `Ryan Nowak` (`@rynowak`)

## Overview

*This design-note describes a prototype we plan to build for application diagnostics. As this document is describing a prototype, it will be less complete than other design proposals.*

Our goal is to prototype an end-to-end scenario for **application** diagnostics that emcompasses:

- A platform engineer configures a metrics dashboard (Prometheus+Grafana) and a logging dashboard (probably Azure to start) and wiring these up to a Radius environment.
- A developer deploys an application into the environment, and has metrics and log collection automatically configured for the application.
- An SRE can configure alerts for the metrics. When they receive an alert, they can use the metadata from the metrics to:
  - Navigate to the application graph and understand the application architecture.
  - Navigate to the logs and craft a query to understand the problem.
  - *stretch* understand when and how the problem was introduced based on logs.

## Terms and definitions

| Term                 | Definition                                                                                                                                                                                                                                                              |
| -------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Diagnostics          | In this context, we're referring to metrics, tracing and logs. There are other technologies that could be considered diagnostic data such as profiling and crash-dumps, but they are out of scope for this discussion.                                                  |
| OpenTelemetry        | The standard cloud-native [project](https://opentelemetry.io/) for all kinds of application diagnostics (metrics, tracing, logs). OpenTelemetry is widely supported by SDKs and cloud services as the de-facto standard for diagnostics.                                |
| Collector            | A [component](https://opentelemetry.io/docs/collector/) that can collect and forward diagnotics data on behalf of an application component. This is usually hosted as part of the application. The OpenTelemetry project provides collectors as a standalone component. |
| Diagnostics store(s) | Infrastructure services (cloud or self-hosted) that process and store diagnostics information. This is a generic term that includes a variety of technologies like Prometheus, Grafana, AWS CloudWatch, etc.                                                            |

## Objectives

### Goals

If we're doing a good job with this, we're going to make life better in some of the following ways:

- *Platform engineers*: Easily enforce and rollout diagnostics configuration. Make using the supported diagnostics systems easy for the teams they support. Centralize configuration of common tasks like enriching metrics or filtering for privacy.
- *Developers*: Simplify configuration for applications. The configuration needed to collect and store diagnostics is provided automatically.
- *SRE/Ops*: Provide necessary context alongside diagnostics to diagnose and follow-up on failures. Leverage the application graph to navigate from diagnostics data, to an understanding of the architecture, to actionable next steps.
- *All roles*: By making the implicit explict, we make it easy for everyone in the organization to understand and locate the relevant diagnostics systems. As an example, it will be easy to navigate from an environment to where metrics and logs are stored.

As a good learning/prototyping exercise, we want to valdiate some of the following assumptions:

- Radius can help centralize and manage **configuration** for diagnostics systems. This is an example of Radius helping multiple roles work together. We want to validate that there's a reasonable division of labor for devs and ops.
- Radius can automatically enrich diagnostics information with meaningful per-component metadata that helps users navigate, understand, and search. We want to validate that app-graph information is useful for this purpose.
- We also want to learn about the support/recommended configurations of OpenTelemetry so we can better understand how to design these features, and what level of extensibility is required in Radius. 

### Non goals

There's a number of things we can't solve inside Radius if we remain committed to our principles. For example Radius doesn't change the way developers write code. We also don't try to change the way operators do common tasks like provisioning non-developer-facing infrastructure.

- Out-of-scope: We don't simplify the creation of diagnostics stores like Prometheus and Grafana. We assume operators create these out-of-band from Radius and provide Radius with the relevant configuration.
- Out-of-scope: We don't auto-instrument the developer's code. Developers will still need to use libraries like the OpenTelemetry SDK to instrument their code for tracing and metrics. 
- Out-of-scope: Because it's a prototype, there will be gaps in this end-to-end, probably due to things we haven't built yet like the dashboard. That's expected for now. 

### User scenarios (optional)

*These three scenarios will work together to form our prototype. These are more specific than usual, because we're going to build a specific demo.*

#### Configuring OpenTelemetry as part of an environment

As an operator, I can configure diagnostics stores for use in environments. Then I can configure a Radius environment to use those those diagnostics stores. When I do this I am accomplishing two things: 1) making the supported diagnostics stores and their configuration explicit as part of the environment, 2) making the configuration of collectors for applications automatic based on the configuration that I provided.

In our demo setup we're going to use Azure Monitor + Grafana for both metrics, and logs. We're not addressing tracing in this demo. These technologies are good choices because Grafana is self-hostable and widely used, and Azure Monitor as a backend is a good subtitute for any OTEL-compatible source. It would be easy for us to move this demo to another cloud by swapping out Azure Monitor.

#### Collecting and storing diagnostics data automatically for applications

As a developer, I can instrument my code using the OpenTelemetry SDK to provide metrics. I can write logs to the console using the logging packages I prefer for my application. When I deploy my application, using Radius, the configuration for OpenTelemetry will be automatically injected. I don't know have to know about the configuration of, or have access to where the diagnostics data goes.

In our demo setup we're going to start by adding metrics to our tutorial/demo application that's written using Node.js/TypeScript. We'll introduce a change that causes some request to randomly fail, resulting in an HTTP 502 status code. When we place load on the application we'll see a high number of HTTP 502 failures, which will trigger an alert in the next scenario. 

For the evolution of this feature it should not matter what language we use to build the application.

#### Responding to an alert about application availability

As an SRE, I need to support applications that I did not write, and I may not understand their architecture. I have to use the diagnostics from applications to understand each incident and recommend actions. For applications that are managed by Radius I get comprehensive metadata that helps with navigation and comprehension of the telemetry signals. I can easily identify the environment, application name, and the specific component that has issues. Then I can use the application graph to understand how the application is architected and which cloud resources are involved. This helps me identify the source of a problem as well as whom to contact.

In our demo setup, we're going to create an alert for the error rate coming out of an HTTP application (error golden signal). Our SRE will use the metadata to query the application graph and understand the architecture. Then they will craft a log query that helps them identify the failure pattern. 

*We probably know the least about this scenario for now. Some of the navigation steps that will occur here will have gaps at the start.*

## Design


### Design details

There are three major components of this prototype:

- The application (responsible for emitting diagnostics).
- The Radius environment and the configuration it provides.
- The diagnostics store infrastructure configured out-of-band from Radius.

This section will be brief in discussing the application (category 1) and the diagnostics store (category 3) because there are many possible solutions here. We want to support a wide varity of applications as well as diagnostics stores.

Fortunately, this is the value provided by OpenTelemetry. It acts as a standard compatible interface that works for any application and any diagnostics store.

The [diagrams](https://opentelemetry.io/docs/demo/collector-data-flow-dashboard/) on this page describe the relationship between the application, the OpenTelemetry collector, and the diagnostics store(s) (Prometheus + Jaeger).

#### The application

To instrument our existing demo application we'll use the [@opentelemetry/instrumentation-express](https://www.npmjs.com/package/@opentelemetry/instrumentation-express) package. This is provided by the OpenTelemetry project and will automatically provide metrics for HTTP traffic.

Our demo scenario is a high rate of HTTP `502` - so the second step is that we introduce a commit to throw exceptions randomly from application code. This will result in an HTTP `502`.

#### Environment configuration

The Radius environment will:

- Schematize and store the diagnostics configuration.
- Inject collectors and configuration into applications that are deployed in the environment.

For the first step of the demo, the plan is to run the collector as a [daemonset](https://opentelemetry.io/docs/kubernetes/getting-started/#daemonset-collector). This will run a single collector pod on each Kuberentes node. Those collectors will discover running pods and collect their data. We will enrich the data with attributes specific to Radius using the [kubernetes attributes processor](https://opentelemetry.io/docs/kubernetes/collector/components/#kubernetes-attributes-processor).

To start with, the best way to configure the collector will be to install the Helm chart. This is a per-cluster operation, so it's not 1:1 with an environment. Similar to other Kubernetes components like cert-manager, we don't want to replace the existing installation technology if we're not innovating.

The configuration that needs to be stored by Radius should reflect the data passed to each application pod, setting the relevant [environment variables](https://opentelemetry.io/docs/concepts/sdk-configuration/general-sdk-configuration/). These environment variables are read and used consistently by SDKs.

---

The environment should also store the locations of diagnostic stores. For our example, this probably means the URL of the grafana server. The goal is to help users navigate to the diagnostics store.

Additionally, the environment should store the resource ID of Azure Monitor as used for this demo and make it available for recipes as part of the recipe context.

**This is an area where experimentation will help us identify the important details.**

#### Diagnostics store configuration

The diagnostics store configuration is out-of-band of Radius. Since we're building a demo it's helpful to describe what it will do.

This includes:

- Configuring Azure Monitor as a store of diagnostics.
- Configuring Grafana as a dashboard, using Azure Monitor as a data source.
  - Creating alerts and visualizations for our demo.
- Installing and configuring the OpenTelemetry collector Helm Chart.
  - Understanding Radius' labels and enriching data.
  - Sending data to the Azure Monitor store.

### API design (if applicable)

Here's a speculative design of the API for new settings. This will be improved as we iterate.

To briefly cover what settings are defined here (net new):

- Enabling/Disabling the OpenTelemetry collector, and saying how it's managed (eg: `external`, `none`).
- Providing the destination address for the OpenTelemetry collector inside the cluster. 
- Describing for documentation purposes where to find diagnostics data. This is to support tooling experiences.
- Describing a range of diagnostics store technologies for Use in Recipes. This example includes Azure Monitor.

```tsp
@doc("Environment properties")
model EnvironmentProperties {
  @doc("The status of the asynchronous operation.")
  @visibility("read")
  provisioningState?: ProvisioningState;

  @doc("The compute resource used by application environment.")
  compute: EnvironmentCompute;

  @doc("Cloud providers configuration for the environment.")
  providers?: Providers;

  @doc("Simulated environment.")
  simulated?: boolean;

  @doc("Specifies Recipes linked to the Environment.")
  recipes?: Record<Record<RecipeProperties>>;

  @doc("The environment extension.")
  @extension("x-ms-identifiers", [])
  extensions?: Array<Extension>;
 
  // NEW PROPERTY
  @doc("Specifies settings for diagnostics collection")
  diagnostics?: EnvironmentDiagnostics;
}

// NEW TYPE
model EnvironmentDiagnostics {
  @doc("Configures settings related to OpenTelemetry")
  openTelemetry?:  EnvironmentOpenTelemetryDiagnostics;

  @doc("Configures the location where diagnostics for the environment can be viewed")
  primaryDashboard: EnvironmentDashboardDiagnostics;

  @doc("Configures settings related to Azure monitor")
  azureMonitor?: EnvironmentAzureMonitorDiagnostics;
}

// NEW TYPE
model EnvironmentOpenTelemetryDiagnostics {
  // External means the collector is present, but was configured out-of-band with Radius.
  @doc("Specifies the mode of the OpenTelemetry collector")
  collectorMode: 'none' | 'external';

  // Administrator sets this so it gets injected into the pods.
  @doc("Specifies the address of the OpenTelemetry collector")
  collectorAddress: string;
}

// NEW TYPE
model EnvironmentDashboard {
  @doc("URL of the dashboard where diagnostics can be viewed.")
  url: string
}

// NEW TYPE
model EnvironmentAzureMonitorDiagnostics  {
  @doc("The resource ID of the Azure Monitor instance")
  id: string
}
```

## Alternatives considered

<!--
Describe the alternative designs that were considered or should be considered.
Give a justification for why alternative approaches should be rejected if
possible. 
-->

*TBD for when this is a non-prototype proposal*

## Test plan

This is a demo/prototype, manual testing only.

## Security

This demo includes configuration of an OpenTelemetry collector, which may require secrets to connect to the diagnostics store. This is managed by the user and not be Radius.

Diagnostics data can include sensitive data. Sanitizing this data is the responsibility of the developer, and can be augmented by configuration of the collector. This is out-of-scope of Radius.

## Compatibility (optional)

No concerns, this is net-new functionality.

## Monitoring

No concerns, the functionality inside Radius is limited to processing configuration data. 

## Development plan

There are a few major milestones for us to hit:

- Add `@opentelemetry/instrumentation-express` to the demo appliation. This will automatically emit metrics.
- Add some code to the demo application to make one of the actions fail. Potentially we introduce a new page for this. 

- Add new APIs and datamodels to Radius.
- Inject settings (environment variables) into pods based on new APIs. 

Create a repeatable demo environment:

- Automate configuration of Azure resources for Azure Monitor and Grafana.
- Automate installation and configuration of OpenTelemetry Helm Chart.
- Create a Redis recipe that wires up Azure Monitor
- Create Grafana dashboard and alerts

## Open issues

<!--
Describe (Q&A format) the important unknowns or things you're not sure about. 
Use the discussion to answer these with experts after people digest the 
overall design.
-->

*TBD*
