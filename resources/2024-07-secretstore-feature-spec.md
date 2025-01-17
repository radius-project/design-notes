# Extend the use-cases of `Applications.Core/secretStores`

* **Author**: Will Tsai (@willtsai)

## Topic Summary
<!-- A paragraph or two to summarize the topic area. Just define it in summary form so we all know what it is. -->

This feature spec outlines the desired user experience for extending the use-cases of `Applications.Core/secretStores` to allow for referencing secret stores in `Applications.Core/containers`, `Applications.Core/extenders`, `Applications.Core/volumes`, `Applications.Datastores/*`, and `Applications.Messaging/*` resources. The goal is to enable developers to securely manage secrets for use in their applications by referencing `Applications.Core/secretStores` in their resources.

### Top level goals
<!-- At the most basic level, what are we trying to accomplish? -->
- Enable developers to reference `Applications.Core/secretStores` in all core and portable Radius resources so that secrets can be securely managed for use in their applications.

### Non-goals (out of scope)
<!-- What are we explicitly not trying to accomplish? -->
- The ability to [reference secret stores in a Recipe](https://github.com/radius-project/roadmap/issues/59) is out of scope for this feature spec, i.e. *As an operator, I can reference an `Applications.Core/secretStores` in my Recipes so that I no longer have to store secrets as plain text in the `secrets` field of my Recipe's `output` object for authentication, etc.*
- Today the `type` property in the `Applications.Core/secretStores` is an enum -- this feature spec does not propose to change the existing [implementation](https://github.com/radius-project/radius/pull/7816) of the `type` property in the `Applications.Core/secretStores` resource type.

## User profile and challenges
<!-- Define the primary user and the key problem / pain point we intend to address for that user. If there are multiple users or primary and secondary users, call them out.   -->
- The primary user for this feature is the **developer** who is building and managing microservice applications.
- The secondary user would be the **operator** who is creating and managing the `Applications.Core/secretStores` resources on behalf of the developers.

### User persona(s)
<!-- Who is the target user? Include size/org-structure/decision makers where applicable. -->
**Developers and operators** who are building and managing microservice applications, are using Radius to deploy applications and wish to effortlessly and securely manage the secrets for use in their environments and applications.

### Challenge(s) faced by the user
<!-- What challenges do the user face? Why are they experiencing pain and why do current offerings not meet their need? -->

Today, developers have no way to reference secrets in their Radius resources other than in gateways and containers, and thus have to store secrets in plain text within the `properties.secrets` field for each resource.

Operators may have to do extra work to inject secrets as environment variables in order to propagate secrets without needing the developers to store secrets in plain text.

### Positive user outcome
<!-- What is the positive outcome for the user if we deliver this, i.e. what is the value proposition for the user? Remember, this is user-centric. -->

As a developer, I can reference an `Applications.Core/secretStores` in my `Applications.Core/containers`, `Applications.Core/extenders`, `Applications.Core/volumes`, `Applications.Datastores/*`, and `Applications.Messaging/*` resource definitions so that Radius may securely manage secrets for use in my application to authenticate into those resources.

As an operator, I want to ensure that the developers I support can securely leverage secrets I manage on their behalf for use in their application resources. Today, these secrets have to be stored in plain text within resources.

## Key scenarios
<!-- List ~3-7 high level scenarios to clarify the value and point to how we will decompose this big area into component capabilities. We may ultimately have more than one level of scenario. -->

### Scenario 1: Secrets are stored and then mounted as environment-variables in a container
<!-- One or two sentence summary -->
Barry is a developer who is building a microservice application that requires a database connection string to authenticate into a Cosmos DB instance. Barry wants to securely manage the database connection string for his application and thus creates a `Applications.Core/secretStores` resource containing the database connection string. Barry then references the `Applications.Core/secretStores` resource in his `Applications.Core/containers` resource definition so that Radius will inject the database connection string as an environment variable into his container at deploy time.

### Scenario 2: Secrets are stored and then mounted as files-on-disk in a container.
<!-- One or two sentence summary -->
Maya is a developer who is building a microservice application that requires a TLS certificate to authenticate into a RabbitMQ instance. Maya wants to securely manage the TLS certificate for her application and thus creates a `Applications.Core/secretStores` resource containing the TLS certificate. Maya then references the `Applications.Core/secretStores` resource in her `Applications.Core/volumes` resource definition so that Radius will write the TLS certificate into a file on disk and mount that volume into her container at deploy time.

### Scenario 3: Secrets are stored and used internally by the Radius infrastructure or some other piece of automation (not application code).
<!-- One or two sentence summary -->
Aditi is a developer who is building a microservice application that requires a password to authenticate into a Twilio extender. She wants to securely manage the password for her application and thus creates a `Applications.Core/secretStores` resource containing the password. Aditi then references the `Applications.Core/secretStores` resource in her `Applications.Core/extenders` resource definition so that Radius will use the password internally to authenticate into the Twilio extender at deploy time.

## Key dependencies and risks
<!-- What dependencies must we take in order to enable this scenario? -->
<!-- What other risks are you aware of that need to be mitigated. If you have a mitigation in mind, summarize here. -->
<!-- Dependency Name – summary of dependency.  Issues/concerns/risks with this dependency -->
<!-- Risk Name – summary of risk.  Mitigation plan if known. If it is not yet known, no problem. -->

**Dependency: ability to reference values from `Applications.Core/secretStores`.** This feature is dependent on the ability to reference values from `Applications.Core/secretStores`, which should be doable given the existing functionality of referencing secret values for TLS Termination in `Applications.Core/gateways`.

> The TLS certificate data secret in `Applications.Core/gateways` is referenced today by `tls: { certificateFrom: secretstore.id }` while the implementation for referencing secret stores in `Applications.Core/containers` is `env: { DB_CONNECTION: { valueFrom: { secretRef: { source: secretstore.id, key: 'username' } } } }`. This is by design, see the design document [here](https://github.com/radius-project/design-notes/blob/main/resources/2024-06-support-secretstores-env.md).

## Key assumptions to test and questions to answer
<!-- If you are making assumptions that, if incorrect, would cause us to significantly alter our approach to this scenario, make them explicit here.  Also call out how / when you plan to validate key assumptions. -->
<!-- What big questions must we answer in order to clarify our plan for this scenario.  When and how do you plan to answer those questions (prototype feature x, user research, competitive research, etc) -->

**Assumption: operators can create and deploy `Applications.Core/secretStores` resources.** We assume that operators create and manage secrets on behalf of developers, which are encapsulated in `Applications.Core/secretStores` resources. The developers can subsequently reference these resources in their application resources to securely manage secrets for use in their applications. We will validate this assumption by opening up this feature spec for discussion with the community.

## Current state
<!-- If we already have some ongoing investment in this area, summarize the current state and point to any relevant documents. -->
- Feature request tracking this scenario: https://github.com/radius-project/radius/issues/5520
- Pull request (merged) for adding secretstore references into container env variables: https://github.com/radius-project/radius/pull/7744

## Details of user problem
<!-- <Write this in first person. You basically want to summarize what “I” as a user am trying to accomplish, why the current experience is a problem and the impact it has on me, my team, my work and or biz, etc…. i.e. “When I try to do x aspect of cloud native app development, I have the following challenges / issues….<details>. Those issues result in <negative impact those challenges / issues have on my work and or business.> -->

Currently, Radius provides an `Applications.Core/secretStores` resource type that allows developers to store and retrieve secrets in a secure and reliable way. The `secretStores` resource type can be referenced and used by the following Radius resources today: 
- `Applications.Core/gateways` to manage [TLS certificates for HTTPS connections](https://docs.radapp.io/guides/author-apps/networking/tls/)
- `Applications.Core/containers` to inject secrets as [environment variables into a container](https://docs.radapp.io/reference/resource-schema/core-schema/container-schema/#container) at deploy time
- `Applications.Core/environments` for authentication into [private Recipe registries](https://docs.radapp.io/guides/recipes/terraform/howto-private-registry/), [custom Terraform Providers](https://docs.radapp.io/guides/recipes/terraform/howto-custom-provider/), and in [Recipe configurations](https://github.com/radius-project/radius/blob/594faf60683351e4be2dee7309ebc369dfac26ad/test/functional-portable/corerp/noncloud/resources/testdata/corerp-resources-terraform-postgres.bicep#L32).

As an operator, I can create a `secretStores` resource to securely manage secrets for use in the Radius Environments I provide for my developers, which is handy to allow for authentication into private Recipe registries and custom Terraform Providers. However, the `secretStores` resource type is not yet supported by the `Applications.Core/extenders`, `Applications.Core/volumes`, `Applications.Datastores/*`, and `Applications.Messaging/*` resource types. Thus, I have to do extra work to inject secrets as environment variables in order to propagate secrets to my application developers.

As an application developer, I cannot reference a `secretStores` resource in the `Applications.Core/extenders`, `Applications.Core/volumes`, `Applications.Datastores/*`, and `Applications.Messaging/*` resource types to securely manage secrets for use in my application. Instead, I'm having to store my secrets in plain text within the `properties.secrets` field for each resource. This is a critical gap in the secret management capabilities of Radius that is hindering my ability to securely manage secrets for use in my containers.

## Desired user experience outcome

[existing feature] As an operator, I can define an `Applications.Core/secretStores` resource and deploy it along with an Environment so that the developers I support can securely leverage secrets I manage on their behalf for use in accessing private Recipes. This is a supported scenario in an existing feature, specifically for private [Bicep](https://docs.radapp.io/guides/recipes/howto-private-bicep-registry/) and [Terraform](https://docs.radapp.io/guides/recipes/terraform/howto-private-registry/) Recipes.

[existing feature] As a developer, I can reference an `Applications.Core/secretStores` in my `Applications.Core/gateways` resource definition so that Radius can set up the gateway with the TLS certificate required for HTTPS connections. This is a supported scenario in an existing feature, see [here](https://docs.radapp.io/guides/author-apps/networking/tls/#step-3-add-a-gateway).

[existing feature] As a developer, I can reference an `Applications.Core/secretStores` in my `Applications.Core/containers` resource definition so that Radius will inject secrets as environment variables into my container at deploy time so that credentials can be provided to the container for authentication, etc. This is a supported scenario in an existing feature, see [here](https://docs.radapp.io/reference/resource-schema/core-schema/container-schema/#container).

[proposed feature] As a developer, I can reference an `Applications.Core/secretStores` in my `Applications.Volumes` resource definition so that Radius will write the secret into a file on disk and mount that volume into the container at deploy time. I no longer have to store secrets as plain text in the `properties.secrets` field of my resource for authentication, etc.

[proposed feature] As a developer, I can reference an `Applications.Core/secretStores` in my `Applications.Datastores/*`, `Applications.Messaging/*`, or `Applications.Extenders` resource definition so that Radius will use the password internally to authenticate into the resource at deploy time. I no longer have to store secrets as plain text in the `properties.secrets` field of my resource for authentication, etc.

[future feature] As a developer, I can specify a `Applications.Core/secretStores` in my custom user-defined resource once the [user-defined types](https://github.com/radius-project/design-notes/blob/main/architecture/2024-07-user-defined-types.md) feature is implemented.

### Detailed User Experience

Step 1: Operator or developer creates or references an existing `Applications.Core/secretStores` resource containing the secret data required for authenticating into resources that their developers may use.

Creating a new `secretStore` resource in the `app.bicep` application definition: 

```bicep
resource authcreds 'Applications.Core/secretStores@2023-10-01-preview' = {
  name: 'authcreds'
  properties:{
    application: application
    type: 'generic'
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

Step 2: Developer references stored secrets to: (a) mount as environment-variables in a container, (b) mount as files-on-disk in a container, or (c) use internally by the Radius infrastructure or some other piece of automation (not application code).

(a) Developer references the `Applications.Core/secretStores` resource in their `Applications.Core/containers` resource definition within the `app.bicep` to inject secrets as environment variables into their container at deploy time.

> The example below follows the implementation that has been completed in https://github.com/radius-project/radius/pull/7744

Add a reference to the secret store in the container resource within the `app.bicep` application definition:

```diff
resource demo 'Applications.Core/containers@2023-10-01-preview' = {
  name: 'demo'
  properties: {
    application: application
    container: {
      image: 'ghcr.io/radius-project/samples/demo:latest'
      env:{
+        DB_CONNECTION: {
+          valueFrom: {
+            secretRef: {
+              source: authcreds.id
+              key: 'username'
+            }
+          }
+        }
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

(b) Developer references the `Applications.Core/secretStores` resource in their `Applications.Core/volumes` resource definition within the `app.bicep` application definition to write the secret into a file on disk and mount that volume into their container at deploy time.

```diff
resource volume 'Applications.Core/volumes@2023-10-01-preview' = {
  name: 'myvolume'
  properties: {
    application: app.id
    kind: 'azure.com.keyvault'
    resource: keyvault.id
    secrets: {
      mysecret: {
+        name: {
+          valueFrom: {
+            secretRef: {
+              source: azurekeyvaultsecrets.id
+              key: 'name'
+            }
+          }
+        }
+        version: {
+          valueFrom: {
+            secretRef: {
+              source: azurekeyvaultsecrets.id
+              key: 'version'
+            }
+          }
+        }
+        alias: {
+          valueFrom: {
+            secretRef: {
+              source: azurekeyvaultsecrets.id
+              key: 'alias'
+            }
+          }
+        }
+        encoding: {
+          valueFrom: {
+            secretRef: {
+              source: azurekeyvaultsecrets.id
+              key: 'encoding'
+            }
+          }
+        }
      }
    }
  }
}
```

(c) Developer references the `Applications.Core/secretStores` resource in their `Applications.Extenders` `Applications.Datastores/*`, or `Applications.Messaging/*` resource definition to securely manage secrets for use in their application. Radius then uses the secret to authenticate into the resource at deploy time. The secrets might be referenced in the resources within the `app.bicep` application definition as follows:

```diff
resource twilio 'Applications.Core/extenders@2023-10-01-preview' = {
  name: 'twilio'
  properties: {
    application: application
    environment: environment
    recipe: {
      name: 'twilio'
    }
    secrets: {
+        password: {
+          valueFrom: {
+            secretRef: {
+              source: authcreds.id
+              key: 'password'
+            }
+          }
+        }
    }
  }
}
```

```diff
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
+        connectionString: {
+          valueFrom: {
+            secretRef: {
+              source: authcreds.id
+              key: 'connectionString'
+            }
+          }
+        }
+        password: {
+          valueFrom: {
+            secretRef: {
+              source: authcreds.id
+              key: 'password'
+            }
+          }
+        }
    }
  }
}
```

```diff
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
+      password: {
+          valueFrom: {
+            secretRef: {
+              source: authcreds.id
+              key: 'password'
+            }
+          }
+      }
    }
  }
}
```

Step 3: Developer deploys the resources to Radius and the secrets required are either injected into the container as environment variables, written to a file on a volume and mounted to the container, or used as credentials for authentication into the extender, datastore, or messaging resource at deploy time.

## Key investments
<!-- List the features required to enable this scenario. -->

### Feature 1: Add functionality to reference `Applications.Core/secretStores` to write into mounted `Applications.Core/volumes` resources
<!-- One or two sentence summary -->
Add the ability for developers to reference values from their `Applications.Core/secretStores` resources in their `Applications.Core/volumes` resources so that secrets can be securely managed for use in their application to be written to the volume and mounted at deploy time by Radius.

### Feature 2: Add functionality to reference `Applications.Core/secretStores` to enable Radius to use internally for authentication into resources
<!-- One or two sentence summary -->
Add the ability for developers to reference values from their `Applications.Core/secretStores` resources in their Radius resources (namely `Applications.Core/extenders`, `Applications.Datastores/*`, and `Applications.Messaging/*`) so that secrets can be securely managed for use by Radius to authenticate into those resources.

> We should revisit the overall plan for provisioning types we plan to support on portable resources before implementing this change. There are some changes with UDT that haven't been solidified yet.

## Design Review Notes

- [x] Considerations for `Applications.Core/secretstores` resource `type` values -- *this has been addressed by [PR #7816](https://github.com/radius-project/radius/pull/7816) and referenced in this feature spec doc*
- [x] Discussion around whether `Applications.Core/secretStores` is right, specifically whether it should be part of `Applications.Core` or whether it should be part of UCP (something like `System.Resources/secretStores`) -- *this is a larger discussion beyond the scope of this particular feature spec*
- [x] Scenario that allows for secrets to be passed as parameters into a Radius Bicep resource is missing -- *this scenario was added*
- [x] Need scenarios for referencing `Applications.Core/secretStores` in Radius Recipes -- *this scenario was added*
- [x] Add more information on customers bringing in their own secrets, secrets stores resource currently supporting only k8s secrets -- *addressed in the spec*
- [x] More clarity needed on the actual features that need to be implemented as the spec is somewhere in between vision doc and user scenarios descriptions. The document needs to clearly identify feature gaps and the priority order for addressing them in order for it to be actionable.
- [x] The recipes section should be separated into its own design document to provide more specific and actionable guidance.
- [x] The ability to specify secret managers per environment is a feature that is out of scope for this feature spec and should be addressed as a separate feature. Other environment-wide concerns may include other things like federated identity, VPC, firewall rules, diagnostics, etc.