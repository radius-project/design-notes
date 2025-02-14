# User-Defined Types Schema Validation

* **Author**: @nithyatsu

## Overview

The [user-defined-types](https://github.com/nithyatsu/design-notes/blob/main/architecture/2024-07-user-defined-types.md) feature enables end-users to define their own resource types as part of their tenant of Radius. User-defined types have the same set of capabilities and enable the same features (connections, app graph, recipes, etc) as system-defined types in Radius. 

User-defined types are created with rad resource-type create command, which takes a resource type manifest file as input. Here is a very simple resource manifest -

 ```yaml
        name: 'Mycompany.Messaging'
        types:
            plaidResource:
              apiVersions:
                "2023-10-01-preview":
                  schema: 
                    openAPIV3Schema:
                      type: object
                      properties:
                        type: object
                              properties:
                                host:
                            type: string
                          description: hostname 
                          port:
                            type: string
                          description: port
                      required:
                      - host
                      - port                  
              capabilities: []
```

The `schema` serves as a contract to support users, defining what properties developers are allowed to set and what data is provided to applications. Essentially, the schema of a resource type is its API.

The ability to define schema allows for differentiation and customization. For instance, in case of a databse UDT, an organization might use t-shirt sizes to describe the storage capacity of a database (e.g., S, M, L) or define their own vocabulary for fault domains (e.g., zonal, regional).

Technically, defining schemas is complex. It should be described with OpenAPI,  which represents the state of the art but is a challenging tool to wield. It includes many constructs that are not conducive to good API design. Consequently, every project leveraging OpenAPI tends to define its own supported subset of its functionality.

This document summarizes the key decisions that Radius makes on what constitutes a valid schema in an UDT.

## Terms and definitions

*Please read the [Radius API](https://docs.radapp.io/concepts/technical/api/) conceptual documentation. This document will heavily use the terminology and concepts defined there.*

|Term| Definition|                                                              
| ----------------------------- | ------------------------------------------|
| User-defined type | A resource-type that can be defined or modified by end-users.|
| OpenAPI/Schema | A format for documenting HTTP-based APIs. In this document we're primarily concerned with the parts of OpenAPI pertaining to request/response bodies.   |                                                                     

## Objectives

To arrive at the initial "valid" schema for UDTs which will be the basis of implementation of schema validation.


### Guiding principles

We begin with an initial subset of OpenAPI features that Radius would support. 
Deepending on user input, we would add more fatures in subsequent iterations.


### Goals

Summarize the initial subset of OpenAPI features that Radius would support for user-defined types. 

### Non goals

Validation of schema of a resource of a user-defined resource type
  
### User scenarios 

TBD: ref to comcase UDT scenario and write up

## User Experience (if applicable)

Users author a manifest like the following to defind a user-defined resource type.

**Sample Input:**

```yaml
name: MyCompany.Resources
types:
  postgresDatabases:
    apiVersions:
      '2025-01-01-preview':
        schema: 
          type: object
          properties:
            size:
              type: string
              description: The size of database to provision
              enum:
              - S
              - M
              - L
              - XL
            status:
              type: object
              readOnly: true
              properties: 
                binding:
                  type: 'object'
                  properties:
                    hostname: 
                      type: string
                    username:
                      type: string
                    secret:
                      type: string
                      schema:
                        type: 'object'
                        properties:
                          password:
                            type: 'string'
                recipe: 
                  $ref: 'https://radapp.io/schemas/v1#RecipeStatus'
                  
          required:
          - size

    capabilities: ["SupportsRecipes"]
```
**Sample Output:**

Users can use this schema to generate a Bicep extension providing strongly-typed editor support. Here's an example of Bicep code that matches this API definition.

```bicep
extension radius
extension mycompany // generated from the schema

resource webapp 'Applications.Core/containers@2024-01-01' = {
  name: 'sample-webapp'
  properties: {
    image: '...'
    env: {
      // Bicep editor has completion for these 
      DB_HOSTNAME: { fromValue: { value: db.properties.binding.hostname } }
      DB_USERNAME: { fromValue: { value: db.properties.binding.username } }
      DB_HOSTNAME: {
        fromSecret: {
          secret: db.properties.binding.secret 
          key: 'password'
        }
      }
    }
  }
}

resource db 'MyCompany.Resources/postgresDatabases@2025-01-01-preview' = {
  name: 'sample-db'
  properties: {
    size: 'L' // Bicep editor can validate this field
  }
}
```

## Design

When user executes a `rad resource-type create` command, the cli should validate the schema object provided in the manifest and report any errors back to the users before making the API call. 

The server side, upon receiving a resource payload for a resource-type resource, must again validate the schema before it saves the resource into database.

We should implement validations on both client and server since rad cli need not be the only client. It is important to validate payloads since these can be any arbitrary data.

### High Level Design

We expect users will provide the structural schema for new resource types and describe it use using [Open API v3](https://github.com/OAI/OpenAPI-Specification/blob/main/versions/3.0.2.md#schema-object). Structural schema ensures explicit type definition for properties and enforces strict validation rules.

When a create or update request arrives for a UDT resource, Radius should retrieve the schema of corresponding UDT from database and validate the resource' schema against it before creation.

### Architecture Diagram

NA 

### Detailed Design

#### Structure

Users provide us with the schema of UDT by defining `properties` in schema. 
There is no support for defining custom fields outside of `properties`.

This schema defines the structure and validation rules for the object.

`properties` is an `Object`. Each of the property is a user defined property of the UDT. It has a type and optional description and format. The type can be 

* a scalar. 
  
```
schema: 
  openAPIV3Schema:
    type: object
    properties:
      size:
        type: string
        description: The size of database to provision
        enum:
        - S
        - M
        - L
        - XL
      host:
        type: string
        description: hostname
        maxLength: 20
```

* an array
  
```
schema:
  openAPIV3Schema: 
    type: object
    properties:
      ports:
        type: array
        items:
          type: object
```

* a free-form set of properties, allowing undefined properties of a specific type
  
example #1

```
schema:
  openAPIV3Schema:
    type: object
    additionalProperties:
      type: integer
```

example #2

```
schema:
  openAPIV3Schema:
    type: object
    additionalProperties:
      type: object
      properties:
        value:
          type: string
        timestamp:
          type: string
          format: date-time
```

# Scalars

Schemas will support every construct OpenAPI provides for a scalar field. That includes all of the modifiers like `enum` and `format` and also all of the validation attributes like `minLength`.


# Maps

Schemas support maps using `additional properties`. The additionalProperties keyword allows for dynamic keys that are not predefined in the schema
The Schema is allowed to have either user-defined properties or additional properties which specifies a type. The type can be `Object` or another scalar.

# Arrays

Arrays must specify a type for `item`.

# limitations

#### Objectives

Objects may not use the following constructs that support polymorphism:

- `allOf`
- `anyOf`
- `oneOf`
- `not`

This is a point in time decision to limit complexity, and we'll likely get feedback that polymorphism is important. 

Objects may not set both `additionalProperties` AND define their own properties.


### Implementation Details

N/A for this document. This is a spec. 

### Error Handling

N/A for this document. This is a spec. 

## Test plan

N/A for this document. This is a spec. 

## Security

N/A

## Compatibility (optional)


## Monitoring and Logging


## Development plan


## References
https://github.com/readmeio/oas-examples/tree/main/3.0/yaml