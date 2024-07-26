# Extend the use-cases of `Applications.Core/secretStores`

* **Status**: Pending
* **Author**: Will Tsai (@willtsai)

## Target customers
**Developers and operators** who are building and managing microservice applications, are using Radius to deploy applications and wish to effortlessly and securely manage the secrets for use in their environments and applications.

## Existing customer problem

Currently, Radius provides an `Applications.Core/secretStores` resource type that allows developers to store and retrieve secrets in a secure and reliable way. The `secretStores` resource type can be referenced and used by the following Radius resources today: 
- `Applications.Core/gateways` to manage [TLS certificates for HTTPS connections](https://docs.radapp.io/guides/author-apps/networking/tls/)
- `Applications.Core/environments` for authentication into [private Recipe registries](https://docs.radapp.io/guides/recipes/terraform/howto-private-registry/), [custom Terraform Providers](https://docs.radapp.io/guides/recipes/terraform/howto-custom-provider/), and in [Recipe configurations](https://github.com/radius-project/radius/blob/594faf60683351e4be2dee7309ebc369dfac26ad/test/functional-portable/corerp/noncloud/resources/testdata/corerp-resources-terraform-postgres.bicep#L32).

As an operator, I can create a `secretStores` resource to securely manage secrets for use in the Radius Environments I provide for my developers, which is handy to allow for authentication into private Recipe registries and custom Terraform Providers. However, the `secretStores` resource type is not yet supported by the `Applications.Core/containers`, `Applications.Core/extenders`, `Applications.Core/volumes`, `Applications.Datastores/*`, and `Applications.Messaging/*` resource types. Thus, I have to do extra work to inject secrets as environment variables by configuring each custom Recipe in order to propagate secrets to my application developers.

As an application developer, I cannot reference a `secretStores` resource in the `Applications.Core/containers`, `Applications.Core/extenders`, `Applications.Core/volumes`, `Applications.Datastores/*`, and `Applications.Messaging/*` resource types to securely manage secrets for use in my application. Instead, I'm having to store my secrets in plain text within the `properties.secrets` field for each resource. This is a critical gap in the secret management capabilities of Radius that is hindering my ability to securely manage secrets for use in my containers.

## Desired customer experience outcome
<!-- <Write this as an “I statement” that expresses the new capability from customer perspective … i.e. After this scenario is implemented “I can do, x, y, z, steps in cloud native app developer and seamlessly blah blah blah …. As a result <summarize positive impact on your work / business>  -->

As an operator, I can define an `Applications.Core/secretStores` resource and deploy it along with an Environment so that the developers I support can securely leverage secrets I manage on their behalf for use in their application resources.

As a developer, I can reference an `Applications.Core/secretStores` in my `Applications.Core/containers` resource definition so that Radius will inject secrets as environment variables into my container at deploy time so that credentials can be provided to the container for authentication, etc.

As a developer, I can reference an `Applications.Core/secretStores` in my `Applications.Extenders` or `Applications.Volumes` resource definition so that I no longer have to store secrets as plain text in the `properties.secrets` field of my resource for authentication, etc.

As a developer, I can reference an `Applications.Core/secretStores` in my `Applications.Datastores/*` or `Applications.Messaging/*` resource definition so that I no longer have to store secrets as plain text in the `properties.secrets` field of my resource for authentication, etc.

### Detailed Customer Experience
 <!-- <List of steps the customer goes through from the start to the end of the scenario to provide more detailed view of exactly what the user is able to do given the new capabilities>  -->
<!-- Step 1
Step 2
… -->

Step 1: Operator creates and deploys `Applications.Core/secretStores` resources containing the secret data required for authenticating into resources that their developers may use.

```bicep
resource authcreds 'Applications.Core/secretStores@2023-10-01-preview' = {
  name: 'authcreds'
  properties:{
    application: application
    type: 'credentials' // this can change based on detailed tech design
    data: {
      'username': {
        value: username
      }
      'password': {
        value: password
      }
      'uri': {
        value: uri
      }
      'connectionString': {
        value: connectionString
      }
    }
  }
}
```

```bicep
resource azurekeyvaultsecrets 'Applications.Core/secretStores@2023-10-01-preview' = {
  name: 'azurekeyvaultsecrets'
  properties:{
    application: application
    type: 'generic' // this can change based on detailed tech design
    data: {
      'name': {
        value: secret1
      }
      'version': {
        value: 1
      }
      'encoding': {
        value: 'base64'
      }
      'alias': {
        value: secretalias
      }
    }
  }
}
```

Step 2: Developer references the `Applications.Core/secretStores` resource in their `Applications.Core/containers` resource definition to inject secrets as environment variables into their container at deploy time.

```bicep
resource demo 'Applications.Core/containers@2023-10-01-preview' = {
  name: 'demo'
  properties: {
    application: application
    container: {
      image: 'ghcr.io/radius-project/samples/demo:latest'
      env:{
        // validate accuracy of referencing the values from the secret store
        DB_CONNECTION: authcreds.properties.data.connectionString
      }
      ports: {
        web: {
          containerPort: 3000
        }
      }
    }
  }
}
```

Step 3: Developer references the `Applications.Core/secretStores` resource in their `Applications.Extenders` or `Applications.Volumes` resource definition to securely manage secrets for use in their application.

```bicep
resource twilio 'Applications.Core/extenders@2023-10-01-preview' = {
  name: 'twilio'
  properties: {
    application: application
    environment: environment
    recipe: {
      name: 'twilio'
    }
    secrets: {
        // validate accuracy of referencing the values from the secret store
        password: authcreds.properties.data.password
    }
  }
}
```

```bicep
resource volume 'Applications.Core/volumes@2023-10-01-preview' = {
  name: 'myvolume'
  properties: {
    application: app.id
    kind: 'azure.com.keyvault'
    resource: keyvault.id
    secrets: {
      mysecret: {
        // validate accuracy of referencing the values from the secret store
        name: azurekeyvaultsecrets.properties.data.name
        version: azurekeyvaultsecrets.properties.data.version
        alias: azurekeyvaultsecrets.properties.data.alias
        encoding: azurekeyvaultsecrets.properties.data.encoding
      }
    }
  }
}
```

Step 4: Developer references the `Applications.Core/secretStores` resource in their `Applications.Datastores/*` or `Applications.Messaging/*` resource definition to securely manage secrets for use in their application.

```bicep
resource db 'Applications.Datastores/mongoDatabases@2023-10-01-preview' = {
  name: 'db'
  properties: {
    environment: environment
    application: app.id
    resourceProvisioning: 'manual'
    host: substring(cosmosAccount.properties.documentEndpoint, 0, lastIndexOf(cosmosAccount.properties.documentEndpoint, ':'))
    port: int(split(substring(cosmosAccount.properties.documentEndpoint,lastIndexOf(cosmosAccount.properties.documentEndpoint, ':') + 1), '/')[0])
    database: cosmosAccount::database.name
    username: ''
    resources: [
      { id: cosmosAccount.id }
    ]
    secrets: {
        // validate accuracy of referencing the values from the secret store
        connectionString: authcreds.properties.data.connectionString
        password: authcreds.properties.data.password
    }
  }
}
```

```bicep
resource rabbitmq 'Applications.Messaging/rabbitmqQueues@2023-10-01-preview' = {
  name: 'rabbitmq'
  properties: {
    environment: environment
    application: app.id
    resourceProvisioning: 'manual'
    queue: 'radius-queue'
    host: rmqHost
    port: rmqPort
    vHost: vHost
    username: rmqUsername
    secrets: {
      // validate accuracy of referencing the values from the secret store
      password: authcreds.properties.data.password
    }
  }
}
```

Step 5: Developer deploys the resources to Radius and the secrets required for authentication are injected into the container, extender, volume, datastore, or messaging resource at deploy time.

## Key investments
<!-- List the features required to enable this scenario. -->

### Feature 1: Add functionality to reference `Applications.Core/secretStores` in `Applications.Core/containers` resources
<!-- One or two sentence summary -->
Add the ability for developers to reference values from their `Applications.Core/secretStores` resources in their `Applications.Core/containers` resource definitions under the `properties.container.env` field so that secrets can be injected as environment variables into their container at deploy time.

### Feature 2: Add functionality to reference `Applications.Core/secretStores` in `Applications.Core/*` resources
<!-- One or two sentence summary -->
Add the ability for developers to reference values from their `Applications.Core/secretStores` resources in their core resources (namely `Applications.Core/extenders` and `Applications.Core/volumes`) so that secrets can be securely managed for use in their application to authenticate into those resources.

### Feature 3: Add functionality to reference `Applications.Core/secretStores` in portable resources
<!-- One or two sentence summary -->
Add the ability for developers to reference values from their `Applications.Core/secretStores` resources in their portable resources (namely `Applications.Datastores/*` and `Applications.Messaging/*`) so that secrets can be securely managed for use in their application to authenticate into those resources.

## Key dependencies and risks
<!-- What dependencies must we take in order to enable this scenario? -->
<!-- What other risks are you aware of that need to be mitigated. If you have a mitigation in mind, summarize here. -->
<!-- Dependency Name – summary of dependency.  Issues/concerns/risks with this dependency -->
<!-- Risk Name – summary of risk.  Mitigation plan if known. If it is not yet known, no problem. -->

**Dependency: ability to reference values from `Applications.Core/secretStores`.** This feature is dependent on the ability to reference values from `Applications.Core/secretStores`, which should be doable given the existing functionality of referencing secret values for TLS Termination in `Applications.Core/gateways`. 

**Risk: use of secrets in core and portable resources.** This pattern of referencing and leveraging secrets in core and portable resources might not be a common or desirable pattern for users. We will validate this by opening up this feature spec for discussion with the community.

## Key assumptions to test and questions to answer
<!-- If you are making assumptions that, if incorrect, would cause us to significantly alter our approach to this scenario, make them explicit here.  Also call out how / when you plan to validate key assumptions. -->
<!-- What big questions must we answer in order to clarify our plan for this scenario.  When and how do you plan to answer those questions (prototype feature x, customer research, competitive research, etc) -->

**Assumption: operators can create and deploy `Applications.Core/secretStores` resources.** We assume that operators create and manage secrets on behalf of developers, which are encapsulated in `Applications.Core/secretStores` resources. The developers can subsequently reference these resources in their application resources to securely manage secrets for use in their applications. We will validate this assumption by opening up this feature spec for discussion with the community.

## Current state
<!-- If we already have some ongoing investment in this area, summarize the current state and point to any relevant documents. -->
- Feature request tracking this scenario: https://github.com/radius-project/radius/issues/5520
- Pull request (work in progress) for adding secretstore references into container env variables: https://github.com/radius-project/radius/pull/7744