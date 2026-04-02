# Title

* **Author**: Nithya Subramanian (@nithyatsu)

## Overview

Radius provides an **application resource** that lets teams define and deploy their entire application — including compute, relationships, and infrastructure — as a single unit. Developers express the resources that make up an application (containers, databases, message queues, etc.) along with the relationships between them. Together, these form the **Radius application graph**: a directed graph of resources and their connections.

The application graph serves two key purposes:

1. **Deployment and configuration** — Radius uses the graph to understand resource dependencies, enabling it to orchestrate deployment and inject configuration automatically.
2. **Visualization** — The graph gives users an intuitive, topology-based view of their application rather than a flat list of resources.

### What exists today

Radius currently supports a single type of application graph — the **run-time deployment graph** — via the `rad app graph` CLI command. This command calls a Radius API that queries the control plane for all deployed resources, constructs edges based on the `connections` property of each resource, and returns the serialized graph. See [Radius App Graph](2023-10-app-graph.md) for details on how the API builds this graph. Because it reflects live infrastructure, this graph is only available after an application has been deployed.

### Proposed graph types

This design proposes extending Radius to support three kinds of application graph:

#### 1. Static application graph

A graph constructed from application definitions authored in Bicep files (or their compiled JSON output), **without** deploying the application. This is useful for:

- Visualizing application architecture from source code checked into a repository.
- Highlighting infrastructure changes introduced by a Pull Request.

**Limitation:** Because the concrete infrastructure resources depend on the recipe bound to each resource type — which in turn depends on the target Radius environment — the static graph cannot include infrastructure-level details.

#### 2. Run-time application graph (deployment graph)

The graph of a **live, deployed** application, as described above. This is the only graph type supported today.

#### 3. Simulated deployment graph

A graph that shows what the concrete infrastructure resources and their dependencies **would be** if an application definition were deployed against a specific environment, without actually deploying it. This could be surfaced via a command such as:

```sh
rad app graph -e <env-id> --dry-run
```

Radius should provide a way to access all three kinds of graph.


## Terms and definitions

| Term | Definition |
|---|---|
| Application Graph | A directed graph representing an application as its constituent resources and the relationships between them. |
| Static Application Graph | An application graph inferred from a Bicep template or its compiled JSON output, without deploying the application. |
| Deployment Graph | An application graph constructed by querying the Radius control plane for the live resources of a deployed application. |
| Simulated Deployment Graph | An application graph that represents what would be deployed if an application definition were applied to a specific environment. |
| rootScope | The current UCP scope (e.g., `/planes/radius/local/resourceGroups/default`). |


## Objectives

> **Issue Reference:** 


### Goals

* Finalize graph datamodel flexible/ extensible for both static and run-time application graphs  
* Finalize persistence details
* Radius should provide a cli command that outputs app graph from  application definition files
* Radius should provide an API that retrieves the run time application graph
* Radius should provide a cli command that outputs graph of a deployed application

### Non goals

* Authorization/ RBAC for viewing graph


### User scenarios (optional)

#### User story 1

## Design

### Schema

Thes graph schema should be applicable to both kind of graph, selectively populating relevant fields. The graph should be a DAG. Today, ARM (specifically DE) constructs a Dependency graph for deploying one or more resources using a arm json template ( or bicep template) and specifies dependency using a "dependsOn" contruct. This graph is in-memory. With Radius, these multiple resources would be tied to an application root resource. The scale (number of nodes) is comparable, making an in-memory graph construction a good choice for radius graph too. 

While the dependency graph is mainly used to determine the order of construction of resources, Radius users might be interested in querying dependencies of specific resources. For example, 

Query adjecent nodes of a given node  (the frontend depends on both auth server and backend).  

Querying specific details of one/ more nodes . For example, the user might be interested in whether a resource the application is using is owned by the application or shared (environment resource). 

### Cli support (static graph)

### Server side support
Since we are querying an Application's details, UCP should proxy this API to Applications.Core resource provider. We should be able to reuse much of the ApplicationGraph structure we currently have in  `cli` for supporting the `rad app connections` command. The ApplicationGraph is a list of Resources, with each Resource including information about its Dependencies(Connections). 

The Applications.Core RP should be able to 
1. query all resources in the `rootScope`
2. filter resources relevant to the given Application based on the app.id field
3. construct the graph object based on `connections`.

We should be able to handle connections that take a resourceID for destination as well as those which take a URL. 

As requirement evolves, we would be able to add properties such as a repository link to a container or a health url and retrieve these as part of application graph. This graph object could then be consumed by react components to provide the desired UX experience. 


### API design

The API to retrieve an Application Graph looks like

`POST /{rootScope}/providers/Applications.Core/applications/{applicationName}/getGraph`

  - Description: retrieve {applicationName}'s  Application Graph.
  - Type: ARM Synchronous

Where

`/{rootScope}/providers/Applications.Core/applications/{applicationName}` is the resource ID of the Application for which we want the graph.

`getGraph` is the custom action on this resource. Ref. [ARM Custom Actions](https://learn.microsoft.com/en-us/azure/azure-resource-manager/custom-providers/custom-providers-action-endpoint-how-to)



Possible Responses

* `HTTP 200 OK` with Serialized `ApplicationGraph` as response data.
* `HTTP 404 Not Found` for Application Not Found

***Model changes***

Addition of ApplicationGraphResponse type and getGraph method to applications.tsp

```
@doc("Describes the application architecture and its dependencies.")
model ApplicationGraphResponse {
  @doc("The resources in the application graph.")
  @extension("x-ms-identifiers", ["id"])
  resources: Array<ApplicationGraphResource>;
}

@doc("Describes the connection between two resources.")
model ApplicationGraphConnection {
  @doc("The resource ID ")
  id: string;

  @doc("The direction of the connection. 'Outbound' indicates this connection specifies the ID of the destination and 'Inbound' indicates indicates this connection specifies the ID of the source.")
  direction: Direction;
}

@doc("The direction of a connection.")
enum Direction {
  @doc("The resource defining this connection makes an outbound connection resource specified by this id.")
  Outbound,

  @doc("The resource defining this connection accepts inbound connections from the resource specified by this id.")
  Inbound,
}

@doc("Describes a resource in the application graph.")
model ApplicationGraphResource {
  @doc("The resource ID.")
  id: string;

  @doc("The resource type.")
  type: string;

  @doc("The resource name.")
  name: string;

  @doc("The resources that comprise this resource.")
  @extension("x-ms-identifiers", ["id"])
  resources: Array<ApplicationGraphOutputResource>;

  @doc("The connections between resources in the application graph.")
  @extension("x-ms-identifiers",[])
  connections: Array<ApplicationGraphConnection>;

  @doc("provisioningState of this resource") 
  provisioningState?: string
}

@doc("Describes an output resource that comprises an application graph resource.")
model ApplicationGraphOutputResource {
  @doc("The resource ID.")
  id: string;

  @doc("The resource type.")
  type: string;

  @doc("The resource name.")
  name: string;
}
```

```
 @doc("Gets the application graph and resources.")
  @action("getGraph")
  getGraph is ArmResourceActionSync<
    ApplicationResource,
    {},
    ApplicationGraphResponse,
    UCPBaseParameters<ApplicationResource>
  >;
```

***Example***

`rad deploy app.bicep`

Contents of `app.bicep`

```
import radius as radius

@description('Specifies the location for resources.')
param location string = 'local'

@description('Specifies the environment for resources.')
param environment string

@description('Specifies the port for the container resource.')
param port int = 3000

@description('Specifies the image for the container resource.')
param magpieimage string

resource app 'Applications.Core/applications@2023-10-01-preview' = {
  name: 'corerp-resources-gateway'
  location: location
  properties: {
    environment: environment
  }
}

resource gateway 'Applications.Core/gateways@2023-10-01-preview' = {
  name: 'http-gtwy-gtwy'
  location: location
  properties: {
    application: app.id
    routes: [
      {
        path: '/'
        destination: frontendRoute.id
      }
      {
        path: '/backend1'
        destination: backendRoute.id
      }
      {
        // Route /backend2 requests to the backend, and
        // transform the request to /
        path: '/backend2'
        destination: backendRoute.id
        replacePrefix: '/'
      }
    ]
  }
}

resource frontendRoute 'Applications.Core/httpRoutes@2023-10-01-preview' = {
  name: 'http-gtwy-front-rte'
  location: location
  properties: {
    application: app.id
    port: 81
  }
}

resource frontendContainer 'Applications.Core/containers@2023-10-01-preview' = {
  name: 'http-gtwy-front-ctnr'
  location: location
  properties: {
    application: app.id
    container: {
      image: magpieimage
      ports: {
        web: {
          containerPort: port
          provides: frontendRoute.id
        }
      }
      readinessProbe: {
        kind: 'httpGet'
        containerPort: port
        path: '/healthz'
      }
    }
    connections: {
      backend: {
        source: backendRoute.id
      }
    }
  }
}

resource backendRoute 'Applications.Core/httpRoutes@2023-10-01-preview' = {
  name: 'http-gtwy-back-rte'
  location: location
  properties: {
    application: app.id
  }
}

resource backendContainer 'Applications.Core/containers@2023-10-01-preview' = {
  name: 'http-gtwy-back-ctnr'
  location: location
  properties: {
    application: app.id
    container: {
      image: magpieimage
      env: {
        gatewayUrl: gateway.properties.url
      }
      ports: {
        web: {
          containerPort: port
          provides: backendRoute.id
        }
      }
      readinessProbe: {
        kind: 'httpGet'
        containerPort: port
        path: '/healthz'
      }
    }
  }
}

```

Assuming we set up Radius with the default  `rad init` command, Rest API For querying the above Application's graph would look like

`POST /ucphostname:ucpport/apis/api.ucp.dev/v1alpha3/planes/radius/local/resourceGroups/default/providers/Applications.Core/applications/corerp-resources-gateway/getGraph?api-version=2023-10-01-preview`

Response indicating Success would be 

`HTTP 200 OK` With response body as below. 

```
{
    "resources": [
        {
            "connections": [
                {
                    "direction": "Inbound",
                    "id": "/planes/radius/local/resourcegroups/default/providers/Applications.Core/containers/http-gtwy-front-ctnr"
                }
            ],
            "id": "/planes/radius/local/resourcegroups/default/providers/Applications.Core/httpRoutes/http-gtwy-front-rte",
            "name": "http-gtwy-front-rte",
            "resources": [
                {
                    "id": "/planes/kubernetes/local/namespaces/default-corerp-resources-gateway/providers/core/Service/http-gtwy-front-rte",
                    "name": "http-gtwy-front-rte",
                    "type": "core/Service"
                }
            ],
            "type": "Applications.Core/httpRoutes",
            "provisioningState": "Succeeded"
        },
        {
            "connections": [
                {
                    "direction": "Outbound",
                    "id": "/planes/radius/local/resourcegroups/default/providers/Applications.Core/httpRoutes/http-gtwy-back-rte"
                }
            ],
            "id": "/planes/radius/local/resourcegroups/default/providers/Applications.Core/containers/http-gtwy-back-ctnr",
            "name": "http-gtwy-back-ctnr",
            "resources": [
                {
                    "id": "/planes/kubernetes/local/namespaces/default-corerp-resources-gateway/providers/apps/Deployment/http-gtwy-back-ctnr",
                    "name": "http-gtwy-back-ctnr",
                    "type": "apps/Deployment"
                },
                {
                    "id": "/planes/kubernetes/local/namespaces/default-corerp-resources-gateway/providers/core/ServiceAccount/http-gtwy-back-ctnr",
                    "name": "http-gtwy-back-ctnr",
                    "type": "core/ServiceAccount"
                },
                {
                    "id": "/planes/kubernetes/local/namespaces/default-corerp-resources-gateway/providers/rbac.authorization.k8s.io/Role/http-gtwy-back-ctnr",
                    "name": "http-gtwy-back-ctnr",
                    "type": "rbac.authorization.k8s.io/Role"
                },
                {
                    "id": "/planes/kubernetes/local/namespaces/default-corerp-resources-gateway/providers/rbac.authorization.k8s.io/RoleBinding/http-gtwy-back-ctnr",
                    "name": "http-gtwy-back-ctnr",
                    "type": "rbac.authorization.k8s.io/RoleBinding"
                }
            ],
            "type": "Applications.Core/containers",
            "provisioningState": "Succeeded"
        },
        {
            "connections": [
                {
                    "direction": "Inbound",
                    "id": "/planes/radius/local/resourcegroups/default/providers/Applications.Core/httpRoutes/http-gtwy-back-rte"
                },
                {
                    "direction": "Inbound",
                    "id": "/planes/radius/local/resourcegroups/default/providers/Applications.Core/httpRoutes/http-gtwy-back-rte"
                },
                {
                    "direction": "Outbound",
                    "id": "/planes/radius/local/resourcegroups/default/providers/Applications.Core/httpRoutes/http-gtwy-front-rte"
                }
            ],
            "id": "/planes/radius/local/resourcegroups/default/providers/Applications.Core/containers/http-gtwy-front-ctnr",
            "name": "http-gtwy-front-ctnr",
            "resources": [
                {
                    "id": "/planes/kubernetes/local/namespaces/default-corerp-resources-gateway/providers/apps/Deployment/http-gtwy-front-ctnr",
                    "name": "http-gtwy-front-ctnr",
                    "type": "apps/Deployment"
                },
                {
                    "id": "/planes/kubernetes/local/namespaces/default-corerp-resources-gateway/providers/core/Secret/http-gtwy-front-ctnr",
                    "name": "http-gtwy-front-ctnr",
                    "type": "core/Secret"
                },
                {
                    "id": "/planes/kubernetes/local/namespaces/default-corerp-resources-gateway/providers/core/ServiceAccount/http-gtwy-front-ctnr",
                    "name": "http-gtwy-front-ctnr",
                    "type": "core/ServiceAccount"
                },
                {
                    "id": "/planes/kubernetes/local/namespaces/default-corerp-resources-gateway/providers/rbac.authorization.k8s.io/Role/http-gtwy-front-ctnr",
                    "name": "http-gtwy-front-ctnr",
                    "type": "rbac.authorization.k8s.io/Role"
                },
                {
                    "id": "/planes/kubernetes/local/namespaces/default-corerp-resources-gateway/providers/rbac.authorization.k8s.io/RoleBinding/http-gtwy-front-ctnr",
                    "name": "http-gtwy-front-ctnr",
                    "type": "rbac.authorization.k8s.io/RoleBinding"
                }
            ],
            "type": "Applications.Core/containers",
            "provisioningState": "Succeeded"
        },
        {
            "id": "/planes/radius/local/resourcegroups/default/providers/Applications.Core/gateways/http-gtwy-gtwy",
            "name": "http-gtwy-gtwy",
            "resources": [
                {
                    "id": "/planes/kubernetes/local/namespaces/default-corerp-resources-gateway/providers/projectcontour.io/HTTPProxy/http-gtwy-back-rte",
                    "name": "http-gtwy-back-rte",
                    "type": "projectcontour.io/HTTPProxy"
                },
                {
                    "id": "/planes/kubernetes/local/namespaces/default-corerp-resources-gateway/providers/projectcontour.io/HTTPProxy/http-gtwy-front-rte",
                    "name": "http-gtwy-front-rte",
                    "type": "projectcontour.io/HTTPProxy"
                },
                {
                    "id": "/planes/kubernetes/local/namespaces/default-corerp-resources-gateway/providers/projectcontour.io/HTTPProxy/http-gtwy-gtwy",
                    "name": "http-gtwy-gtwy",
                    "type": "projectcontour.io/HTTPProxy"
                }
            ],
            "type": "Applications.Core/gateways",
            "provisioningState": "Succeeded"
        },
        {
            "connections": [
                {
                    "direction": "Inbound",
                    "id": "/planes/radius/local/resourcegroups/default/providers/Applications.Core/containers/http-gtwy-back-ctnr"
                }
            ],
            "id": "/planes/radius/local/resourcegroups/default/providers/Applications.Core/httpRoutes/http-gtwy-back-rte",
            "name": "http-gtwy-back-rte",
            "resources": [
                {
                    "id": "/planes/kubernetes/local/namespaces/default-corerp-resources-gateway/providers/core/Service/http-gtwy-back-rte",
                    "name": "http-gtwy-back-rte",
                    "type": "core/Service"
                }
            ],
            "type": "Applications.Core/httpRoutes",
            "provisioningState": "Succeeded"
        }
    ]
}
```


## Alternatives considered

There are multiple ways of representing a graph. We considered below model as an option since it is optimized for bandwidth. However, the model does not work well for pagination (to be added in future), which would be a requirement to support large applications.  


```
@doc("Describes the application architecture and its dependencies.")
model ApplicationGraphResponse {
  @doc("The connections between resources in the application graph.")
  @extension("x-ms-identifiers",[])
  connections: Array<ApplicationGraphConnection>;

  @doc("The resources in the application graph.")
  @extension("x-ms-identifiers", ["id"])
  resources: Array<ApplicationGraphResource>;
}

@doc("Describes the connection between two resources.")
model ApplicationGraphConnection {
  @doc("The source of the connection.")
  source: string;

  @doc("The destination of the connection.")
  destination: string;
}

@doc("Describes a resource in the application graph.")
model ApplicationGraphResource {
  @doc("The resource ID.")
  id: string;

  @doc("The resource type.")
  type: string;

  @doc("The resource name.")
  name: string;

  @doc("The resources that comprise this resource.")
  @extension("x-ms-identifiers", ["id"])
  resources: Array<ApplicationGraphResource>;
}

@doc("Describes an output resource that comprises an application graph resource.")
model ApplicationGraphOutputResource {
  @doc("The resource ID.")
  id: string;

  @doc("The resource type.")
  type: string;

  @doc("The resource name.")
  name: string;
}
```


## Test plan

We should add a E2E that deploys an application and tests if the ApplicationGraph object that can be retrieved using the new API as expected. 

We should also add UTs as needed for all the functions introduced/changed.


## Monitoring

Trace and Metrics will be generated automatically for the new API. 
We should return appropriate errors so that logs are generated for these conditions.

## Development plan

1. Support new Route in Applications.Core 
2. Implement controller and UT for the new Route 
3. Implement E2E that tests the new API
4. Add documentation
5. Rewrite rad app connections using the new API, update relevant tests.


## Open issues

1. The serialized ApplicationGraph in HTTP response could be quite heavy for huge Applications, consuming more network bandwidth. We might have to make this better based on the requirement. 

2. We currently do not have a efficient query to retrieve only the resources in a given application. This might need to be revisited.
