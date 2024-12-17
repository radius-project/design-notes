# Extend the use-cases of `Applications.Core/secretStores`

* **Author**: Will Tsai (@willtsai)

## Target users
**Developers and operators** who are building and managing microservice applications, are using Radius to deploy applications and wish to effortlessly and securely manage the secrets for use in their environments and applications.

## Existing user problem

Currently, Radius provides an `Applications.Core/secretStores` resource type that allows developers to store and retrieve secrets in a secure and reliable way. The `secretStores` resource type can be referenced and used by the following Radius resources today: 
- `Applications.Core/gateways` to manage [TLS certificates for HTTPS connections](https://docs.radapp.io/guides/author-apps/networking/tls/)
- `Applications.Core/environments` for authentication into [private Recipe registries](https://docs.radapp.io/guides/recipes/terraform/howto-private-registry/), [custom Terraform Providers](https://docs.radapp.io/guides/recipes/terraform/howto-custom-provider/), and in [Recipe configurations](https://github.com/radius-project/radius/blob/594faf60683351e4be2dee7309ebc369dfac26ad/test/functional-portable/corerp/noncloud/resources/testdata/corerp-resources-terraform-postgres.bicep#L32).

As an operator, I can create a `secretStores` resource to securely manage secrets for use in the Radius Environments I provide for my developers, which is handy to allow for authentication into private Recipe registries and custom Terraform Providers. However, the `secretStores` resource type is not yet supported by the `Applications.Core/containers`, `Applications.Core/extenders`, `Applications.Core/volumes`, `Applications.Datastores/*`, and `Applications.Messaging/*` resource types. Thus, I have to do extra work to inject secrets as environment variables by configuring each custom Recipe in order to propagate secrets to my application developers.

As an operator, I cannot reference a `secretStores` resource in the Recipes I create in order to securely manage secrets to be used in the Recipe configurations. Instead, I'm having to store my secrets in plain text within the `secrets` field of my Recipe's `output` object. This is a critical gap in the secret management capabilities of Radius that is hindering my ability to securely manage secrets for use in my Recipes.

As an application developer, I cannot reference a `secretStores` resource in the `Applications.Core/containers`, `Applications.Core/extenders`, `Applications.Core/volumes`, `Applications.Datastores/*`, and `Applications.Messaging/*` resource types to securely manage secrets for use in my application. Instead, I'm having to store my secrets in plain text within the `properties.secrets` field for each resource. This is a critical gap in the secret management capabilities of Radius that is hindering my ability to securely manage secrets for use in my containers.

## Desired user experience outcome

As an operator, I can define an `Applications.Core/secretStores` resource and deploy it along with an Environment so that the developers I support can securely leverage secrets I manage on their behalf for use in their application resources.

As an operator, I can reference an `Applications.Core/secretStores` in my Recipes so that I no longer have to store secrets as plain text in the `secrets` field of my Recipe's `output` object for authentication, etc.

As a developer, I can reference an `Applications.Core/secretStores` in my `Applications.Core/containers` resource definition so that Radius will inject secrets as environment variables into my container at deploy time so that credentials can be provided to the container for authentication, etc.

As a developer, I can reference an `Applications.Core/secretStores` in my `Applications.Extenders` or `Applications.Volumes` resource definition so that I no longer have to store secrets as plain text in the `properties.secrets` field of my resource for authentication, etc.

As a developer, I can reference an `Applications.Core/secretStores` in my `Applications.Datastores/*` or `Applications.Messaging/*` resource definition so that I no longer have to store secrets as plain text in the `properties.secrets` field of my resource for authentication, etc.

### Detailed User Experience

Step 1: Operator creates or references an existing `Applications.Core/secretStores` resource containing the secret data required for authenticating into resources that their developers may use.

Creating a new `secretStore` resource: 

```bicep
resource authcreds 'Applications.Core/secretStores@2023-10-01-preview' = {
  name: 'authcreds'
  properties:{
    application: application
    // today the type is an enum, see https://github.com/radius-project/radius/pull/7816
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

Referencing an existing `secretStore` resource:

```bicep
resource azurekeyvaultsecrets 'Applications.Core/secretStores@2023-10-01-preview' = {
  name: 'existing-azurekeyvault-secrets'
  properties:{
    application: application
    resource: 'secret-app-existing-secret'
    // today the type is an enum, see https://github.com/radius-project/radius/pull/7816
    type: 'generic'
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

Step 2: Operator references the `Applications.Core/secretStores` resource in their Recipe's `output` object to securely pass secrets as outputs to Radius that can be used in provisioning and deploying the resource using the Recipe.

Bicep example:
```diff
param context object

@description('The ID of the subnet where the Azure Cache for Redis will be deployed')
param subnetID string = ''

@description('The location of the Azure Cache for Redis')
param location string = resourceGroup().location

resource redis 'Microsoft.Cache/redis@2022-05-01' = {
  name: 'redis-${uniqueString(context.resource.id)}'
  location: location
  properties: {
    sku: {
      capacity: 1
      family: 'P'
      name: 'Premium'
    }
    enableNonSslPort: true
    minimumTlsVersion: '1.2'
    subnetId: subnetID == '' ? null : subnetID
  }
}

output result object = {
  values: {
    host: redis.properties.hostName
    port: redis.properties.port
    username: ''
  }
  secrets: {
+    password: {
+      valueFrom: {
+        secretRef: {
+          source: authcreds.id
+          key: 'password'
+        }
+      }
+    }
  }
}
```

Terraform example:
```diff
output "result" {
  description = "Output of the recipe. These values will be used to configure connections."

  value = {
    values = {
      host = memory_db.cluster_endpoint_address
      port = memory_db.cluster_endpoint_port
    }
    secrets = {
+      password = {
+        valueFrom = {
+          secretRef = {
+            source = authcreds.id
+            key = "password"
+          }
+        }
+      }
    }
  }
}
```

Step 3: Developer references the `Applications.Core/secretStores` resource in their `Applications.Core/containers` resource definition to inject secrets as environment variables into their container at deploy time.

> The example below follows the implementation that has been completed in https://github.com/radius-project/radius/pull/7744

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

Step 4: Developer references the `Applications.Core/secretStores` resource in their `Applications.Extenders` or `Applications.Volumes` resource definition to securely manage secrets for use in their application.

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

Step 5: Developer references the `Applications.Core/secretStores` resource in their `Applications.Datastores/*` or `Applications.Messaging/*` resource definition to securely manage secrets for use in their application.

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

Step 6: The developer specifies the `Applications.Core/secretStores` resource as a parameter to the Bicep resource that requires the secret.

```bicep
param secretStore object = authcreds

resource demo 'Applications.Core/containers@2023-10-01-preview' = {
  name: 'demo'
  properties: {
    application: application
    container: {
      image: 'ghcr.io/radius-project/samples/demo:latest'
      env:{
        DB_CONNECTION: {
          valueFrom: {
            secretRef: {
              source: secretStore.id
              key: 'username'
            }
          }
        }
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

Step 7: Developer deploys the resources to Radius and the secrets required are either injected into the container as environment variables or used as credentials for authentication into the extender, volume, datastore, or messaging resource at deploy time. If the resource is deployed using a Recipe, the secrets are securely passed as outputs to Radius by the Recipe for use in provisioning and deploying the resource.

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

> We should revisit the overall plan for provisioning types we plan to support on portable resources before implementing this change. There are some changes with UDT that haven't been solidified yet.

### Feature 4: Add functionality to reference `Applications.Core/secretStores` objects as a parameter to a Bicep resource
<!-- One or two sentence summary -->
Add the ability for developers to specify the `Applications.Core/secretStores` resource as a parameter of the `object` type to the Bicep resource that requires the secret so that secrets can be passed as parameters into a Radius Bicep resource.

### Feature 5: Add functionality to reference `Applications.Core/secretStores` in Recipes
<!-- One or two sentence summary -->
Add the ability for operators to reference values from their `Applications.Core/secretStores` resources in their Recipe's `output` object so that secrets can be securely passed as outputs to Radius that can be used in provisioning and deploying the resource using the Recipe.

## Key dependencies and risks
<!-- What dependencies must we take in order to enable this scenario? -->
<!-- What other risks are you aware of that need to be mitigated. If you have a mitigation in mind, summarize here. -->
<!-- Dependency Name – summary of dependency.  Issues/concerns/risks with this dependency -->
<!-- Risk Name – summary of risk.  Mitigation plan if known. If it is not yet known, no problem. -->

**Dependency: ability to reference values from `Applications.Core/secretStores`.** This feature is dependent on the ability to reference values from `Applications.Core/secretStores`, which should be doable given the existing functionality of referencing secret values for TLS Termination in `Applications.Core/gateways`.

> The TLS certificate data secret in `Applications.Core/gateways` is referenced today by `tls: { certificateFrom: secretstore.id }`. As a part of implementation we should evaluate if the `valueFrom: { secretRef: { ... } }` pattern proposed here is an acceptable deviation from the previous pattern implemented for `gateways`.

**Risk: use of secrets in core and portable resources.** This pattern of referencing and leveraging secrets in core and portable resources might not be a common or desirable pattern for users. We will validate this by opening up this feature spec for discussion with the community.

## Key assumptions to test and questions to answer
<!-- If you are making assumptions that, if incorrect, would cause us to significantly alter our approach to this scenario, make them explicit here.  Also call out how / when you plan to validate key assumptions. -->
<!-- What big questions must we answer in order to clarify our plan for this scenario.  When and how do you plan to answer those questions (prototype feature x, user research, competitive research, etc) -->

**Assumption: operators can create and deploy `Applications.Core/secretStores` resources.** We assume that operators create and manage secrets on behalf of developers, which are encapsulated in `Applications.Core/secretStores` resources. The developers can subsequently reference these resources in their application resources to securely manage secrets for use in their applications. We will validate this assumption by opening up this feature spec for discussion with the community.

**Assumption: the `Applications.secretStores` resource type can be referenced as a parameter to a Bicep resource.** We assume that the `Applications.secretStores` resource type can be referenced as a parameter to a Bicep resource so that secrets can be passed as object parameters into a Radius Bicep resource. We will validate this assumption by investigating the feasibility of this approach during tech design.

## Current state
<!-- If we already have some ongoing investment in this area, summarize the current state and point to any relevant documents. -->
- Feature request tracking this scenario: https://github.com/radius-project/radius/issues/5520
- Pull request (merged) for adding secretstore references into container env variables: https://github.com/radius-project/radius/pull/7744

## Design Review Notes

- Considerations for `Applications.Core/secretstores` resource `type` values -- *this has been addressed by [PR #7816](https://github.com/radius-project/radius/pull/7816) and referenced in this feature spec doc*
- Discussion around whether `Applications.Core/secretStores` is right, specifically whether it should be part of `Applications.Core` or whether it should be part of UCP (something like `System.Resources/secretStores`) -- *this is a larger discussion beyond the scope of this particular feature spec*
- Scenario that allows for secrets to be passed as parameters into a Radius Bicep resource is missing -- *this scenario was added*
- Need scenarios for referencing `Applications.Core/secretStores` in Radius Recipes -- *this scenario was added*
- Add more information on customers bringing in their own secrets, secrets stores resource currently supporting only k8s secrets -- *addressed in the spec*