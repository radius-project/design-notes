## Authoring a User-Defined-Type schema

* **Author**: Reshma Abdul Rahim (@reshrahim)

## Summary
<!-- A paragraph or two to summarize the topic area. Just define it in summary form so we all know what it is. -->
User-Defined Types enable Platform engineers to define and deploy their organization's custom services in Radius. More details around the feature, user scenarios and end-end-experience can be found [here](/architecture/2024-06-resource-extensibility-feature-spec.md). This feature specification document details on the requirements and the user experience for authoring and updating the schema for the User-Defined-Type.

### Top level goals
<!-- At the most basic level, what are we trying to accomplish? -->

- Enable a simple and intuitive experience for platform engineers to define the schema for their User-Defined-Type with low concept count and minimal learning curve
- Ensure that the schema definitions are authored in a unified, consistent and standard format across all resource types
- Ensure that users receive transparent and detailed error messages when the schema definition is incorrect or when there are breaking changes in the schema definition.

### Non-goals (out of scope)
<!-- What are we explicitly not trying to accomplish? -->
Below goals are out of scope for the first iteration but might be considered in future iterations based on user feedback.

- Advanced scenarios defining the capabilities of the resource type in the schema definition. This includes defining capabilities like connections to other resource, output resources or infrastructure resources produced by the resource type etc.
- Defining child resources or nested resources in the schema definition
- Adding a new resource methods or operations to the resource type
- Providing a full-fledged TypeSpec tooling experience for authoring the schema definition with autocompletion, error validation at development time.
- Providing solutions to handle breaking changes on API versions of the resource type

Below goals are out of scope 

- Extensibility on CRUDL operations for the resource type

## User profile and challenges
<!-- Define the primary user and the key problem / pain point we intend to address for that user. If there are multiple users or primary and secondary users, call them out.   -->

### User persona(s)
<!-- Who is the target user? Include size/org-structure/decision makers where applicable. -->
- Platform engineers: Platform engineers are responsible for building internal developer platforms (IDP) for streamlining the developer-experience for their organization by defining a set of recommended practices for application teams and provide self-service capabilities for application teams/developers to use. They are responsible for defining and maintaining the schema definitions of the custom resource types that are available in the platform.

### Challenge(s) faced by the user
<!-- What challenges do the user face? Why are they experiencing pain and why do current offerings not meet their need? -->
- Platform engineers are responsible for defining the schema for the custom resource types that are available in the platform. The current experience of defining the schema for the custom resource types is complex requires a deep understanding of how native Radius resource types are defined in TypeSpec. This makes it difficult for platform engineers to learn a new language TypeSpec and define the schema. There is steep learning curve and effort involved to define a custom resource type and making it work end-end in Radius. 

### Positive user outcome
<!-- What is the positive outcome for the user if we deliver this, i.e. what is the value proposition for the user? Remember, this is user-centric. -->

- Platform engineers can define the schema in a simple and intuitive way with minimal effort and learning curve.

- Platform engineers can extend Radius with their custom services and provide self-service capabilities for application teams to use their custom services in Radius.

- Platform engineers are empowered to enforce guardrails and best practices for application teams with the schema for the custom resource types.

## Key scenarios
<!-- List ~3-7 high level scenarios to clarify the value and point to how we will decompose this big area into component capabilities. We may ultimately have more than one level of scenario. -->
The key scenarios for UDTs are outlined [here](/architecture/2024-06-resource-extensibility-feature-spec.md). We will decompose these scenarios specific to the authoring experience for defining the schema for the User-Defined-Type.

Deb, a platform engineer at a Financial Services company who is responsible for building the custom internal developer platform. He has started the journey to enable radification one of their Budgets app. The Budgets app relies on an internal messaging service called Plaid. Deb wants to use Plaid as a resource type in Radius to deploy the application seamlessly and leverage all the features of Radius, such as Recipes, Connections, and the rad CLI, with ease and flexibility.

### Scenario 1: Deb defines the schema for the internal messaging service Plaid
<!-- One or two sentence summary -->
As called out in the [feature spec](/architecture/2024-06-resource-extensibility-feature-spec.md), since `yaml` is the most common and standard format across open-source communities, we want to enable users to define the schema in a custom `yaml` format.

1. Deb creates a sample schema definition for the internal messaging service Plaid
   
   Below are the basic set of inputs needed required to author a schema definition
    
    |Input | Description | Required|
    |------|-------------|---------|
    | Resource namespace | The namespace for the resource provider | Yes |
    | Resource type name | The name of the resource type | Yes |
    | Resource type description | The description of the resource type | No |
    | Properties | The data model of the resource type | Yes |
    | API Version | The version of the resource type | Yes |

    Deb creates a plaid.yaml file with the following schema definition

    ```yaml
    resource namespace: 'Mycompany.Messaging'
        'plaidResource':
            apiVersions: 
                2024-10-01:
                    schema:
                        properties: 
                            queueName:
                                type: 'string'
                                description: 'Name of the queue'
                                required: true
                            host:
                                type: 'string'
                                description: 'Hostname of the messaging service'
                                required: true        
                            port:
                                type: 'string'
                                description: 'Port'
                                required: true
                            connectionString:
                                type: 'string'
                                description: 'Connection string to the messaging service' 
                                required: true             
    ....
    ```

 Now Deb register this resource type via the CLI

    ```bash
    rad resource-provider create -f Mycompany.Messaging.yaml
    ```
The boilerplate code for CRUDL operations for the resource type is generated automatically and the resource type is registered in Radius.

Now on creation, the schema should be validated for correctness and completeness. Deb should be provided with error messages if the schema is incorrect or incomplete.

E.g.:

    ```bash
    rad resource-type create -f plaid.yaml
    Error: The schema definition is incorrect. The required property 'port' is missing a type in the schema definition
    ```

### Scenario 2: Deb updates the schema for the internal messaging service Plaid
<!-- One or two sentence summary -->
Updates to schema definitions can be classified into two categories: non-breaking changes and breaking changes. 

| Change type | Description | Example |
|-------------|-------------|---------|
| Non-breaking changes | Changes that can be made to the schema definition without impacting existing applications that use the resource type. | Eg: Adding an optional property to the schema definition |
| Breaking changes | Changes that are not backward/forward compatible. These changes are disruptive and often requires the users to update their applications to use the new schema definition. | Eg: Adding a required property to the schema definition |

There are two key sub-scenarios to consider when Deb updates the schema for the internal messaging service Plaid

#### Sub-scenario 1: Deb adds a optional property and updates the resource type Plaid.
Deb wants to add an optional property `logAnalytics` to have log analytics enabled for the resource-type. This is considered to be a non-breaking change as it is an optional property with default value. Users should be able to use the new property without any impact across versions on their existing applications.

```yaml
resource namespace: 'Mycompany.Messaging'
    'plaidResource':
        apiVersions: 
            2024-10-01:
                schema:
                    properties: 
                        queueName:
                            type: 'string'
                            description: 'Name of the queue'
                            required: true
                        host:
                            type: 'string'
                            description: 'Hostname of the messaging service'
                            required: true        
                        port:
                            type: 'string'
                            description: 'Port'
                            required: true
                        connectionString:
                            type: 'string'
                            description: 'Connection string to the messaging service' 
                            required: true             
                        logAnalytics:
                            type: 'string'
                            description: 'Usage type of the messaging service'  
                            default: 'off'        
```

Deb creates the updated resource type schema via the CLI

```bash
rad resource-type create -f plaid.yaml
Resource type `plaid` created successfully for version `2024-11-01`
```

Question 

1. We leave it up to the user to decide whether to use the new optional property in their application definitions and Recipes. Should we provide any guidance?
2. If Deb tries to add an optional property within the same version or without a default value, what should be the behavior ? Should we consider that any change warrants a api version upgrade like Azure/ ARM does?

#### Sub-scenario 2: Deb adds a required property and updates the resource type Plaid.
Deb wants to add a required property `messageForwarding`. This is considered to be a breaking change as it would require the Infrastructure operator teams and developers to update their Recipes and application definitions to use the new property in the resource type Plaid. Deb should be provided with an error message when a breaking change is detected in the schema definition.

```yaml
resource namespace: 'Mycompany.Messaging'
    'plaidResource':
        apiVersions: 
            2024-10-01:
                schema:
                    properties: 
                        queueName:
                            type: 'string'
                            description: 'Name of the queue'
                            required: true
                        host:
                            type: 'string'
                            description: 'Hostname of the messaging service'
                            required: true        
                        port:
                            type: 'string'
                            description: 'Port'
                            required: true
                        connectionString:
                            type: 'string'
                            description: 'Connection string to the messaging service' 
                            required: true             
                        messageForwarding:
                            type: 'string'
                            description: 'Usage type of the messaging service'          
```

Deb creates the updated resource type schema via the CLI

```bash
rad resource-type create -f plaid.yaml
Error: Unable to create the resource-type `plaid` for version `2024-10-01`. The schema definitions has a breaking change with the required property `instance_type`. Please delete older versions of the resource type and try again.
```

Questions
1. For MVP, can we just flag breaking changes and provide documentation on how to handle breaking changes ? 
2. What are the breaking changes that we need to consider ? 
    - Adding a required property
    - Removing a required property
    - Changing the type of a property
    - Changing the name of a property
    ?

### Scenario 3: Deb deletes the schema for the internal messaging service Plaid
<!-- One or two sentence summary -->

Deb wants to delete the schema for the internal messaging service Plaid. Deb should be able to delete the schema definition via the CLI.

```bash
rad resource-type delete -n Mycompany.Messaging/plaidResource
Warning : There are resources provisioned with the resource type `plaidResource`. Deleting the resource type will disrupt the resources provisioned with the resource type. Do you want to continue ? (y/n)
```

Questions
1. Should we delete the resource type if there are resources provisioned with the resource type ? or 
2. Should we delete the resources provisioned with the resource type when the resource type is deleted ?


## Key investments
<!-- List the features required to enable this scenario(s). -->

### Feature 1
<!-- One or two sentence summary -->

### Feature 2
<!-- One or two sentence summary -->

### Feature 3
<!-- One or two sentence summary -->

