# User-Defined Types

* **Author**: `Nithya Subramanian` (`@nithyatsu`)

## Overview

A Radius application typically defines resources of different types and connects them using [connections](https://docs.radapp.io/guides/author-apps/containers/howto-connect-dependencies/). These connections mainly serve two purposes:

1.	Build application graph 
The connections provide information on dependecies between resources. Thus, this metadata helps in building teh application graph.
2.	Allow the use of connected resource’s properties
Connections automatically inject environment variables into the container when a container defines a connection to a portable resource.

The ability to use connections should be extended to user defined types. 
A container should be able to connect to a UDT. For example, an application backend container needs a connection to a postgreSQL database. Also, an UDT should be able to connect to another UDT. For example, consider a WebService UDT comprising of multiple containers connecting to a JIRA external service UDT.

This document clarifies the requirements of connections defined with an UDT resource as source or destination.


## Terms and definitions

## Objectives

Define expectations on injecting environment variables and enriching recipe context when the application definition includes connections from or to a UDT resource.

### Goals

- Support definiting connections from container to UDT
- Support definiting connections from container to UDT
- When connections are appropriately defined, UDTsshould be displayed in Application Graph.
- When a container is connected to a UDT, appropriate environment variables should be autoinjected into the container. 
- When an UDT is connected to UDT, The desitination UDT's properties should be available for use in the source UDT's recipe.

### Non goals

Supporting connections from UDT to Application.Core/containers is not in scope of this document.
  
### User scenarios (optional)

The scenario document for user-defined types is in progress [here](https://github.com/radius-project/design-notes/pull/81). Please refer to the scenarios defined in that doc, specifically user story 8 and 9.

## User Experience (if applicable)

### Resource Type Manifest for a UDT to allow connections

``` yaml
name: MyCompany.Cache
types:
  cache:
    apiVersions:
      '2023-10-01-preview':
        schema: 
          type: object
          properties:
            environment:
              type: string
            application:
              type: string
            size:
              type: enum
              read-only: true
            # schema should support connections through below construct  
            connections:
              type: object
              additionalProperties:
                source: 
                  type: string
                  description: The resourceID of the source of the connection.
    capabilities: ["SupportsRecipes"]
```

### Using connections in application definition

### UDT to UDT

``` bicep
resource cache 'MyCompany.Cache/coolCache@2023-10-01-preview' = {
  name: 'cache'
  location: 'global'
  properties: {
    application: coolapp.id
    environment: coolenv.id
    connections: {
      databaseresource: {
        source: pg.id
      }
    }
  } 
}

resource postgres 'MyCompany.Datastores/postgres@2023-10-01-preview' = {
  name: 'postgres'
  location: 'global'
  properties: {
    application: coolapp.id
    environment: coolenv.id
  }  
}
```

### Applications.Core/Container to UDT

``` bicep
resource container 'Applications.Core/containerss@2023-10-01-preview' = {
  name: 'ctnr-ctnr'
  location: location
  properties: {
    application: app.id
    container: {
      image: magpieimage
      ports: {
        web: {
          containerPort: port
        }
      }
    }
    connections: {
      databaseresource: {
        disableDefaultEnvVars: true //if we dont want autoenv variable injection
        source: pg.id
      }
    }
  }
}

resource postgres 'MyCompany.Datastores/postgres@2023-10-01-preview' = {
  name: 'postgres'
  location: 'global'
  properties: {
    application: coolapp.id
    environment: coolenv.id
  }  
}
```

## Design

### Applications.Core/containers to UDT

When there is a `connections` construct in a `Applications.Core/containers` resource definition with source set to a UDT resource's id, 
1. The UDT should be visible as a dependency in the application graph.
2. All of the properties of the UDT that are of numeric or string type should be set as environment variable in the container. The environment variable key would be `CONNECTION_CONN_NAME_VARIABLE_NAME` and value would be the value of varoable. 
3. Since some of these could be a secret/ sensitive data, by default we should handle them the same way we handle computed values today.
4. We could use  disableDefaultEnvVars property in a connection to disable auto injection.

### UDT to UDT

When there is a `connections` construct in an UDT(UDT-source) resource definition with source set to another UDT resource's id (UDT-dest), 
1. UDT-source, UDT-dest and their dependency should be visible in application graph.
2. UDT-dest's properties including the ones set by its recipe should be accesible to UDT-source and its recipe through the recipe context.
Thus, within recipe, we should be able to access these properties usinga  construct similar to `context.resource.connections.[connection-name].[connected-resource-property]`

### Architecture Diagram

This document focus on connections, which is part of UDT architecture which is in progress [here](https://github.com/radius-project/design-notes/pull/81)

### Detailed Design 

### Testing

## Security


## Compatibility (optional)
NA

## Monitoring and Logging
NA 

## Development plan

1. bicep tooling should support additionalProperties.
2. connections should enable graph for container --> UDT
3. connections should enable graph for UDT --> UDT
4. connections should inject env variables for container --> UDT 
5. connections should enrich recipe context for UDT --> UDT 


## Open Questions


## Alternatives considered


## Design Review Notes