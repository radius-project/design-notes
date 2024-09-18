# Radius API Guidelines

<!-- markdownlint-disable MD033 MD049 MD055 -->
<!--
Note to contributors: All guidelines have an anchor tag to allow cross-referencing from associated tooling.
The anchor tags within a section using a common prefix to ensure uniqueness with anchor tags in other sections.
Please ensure that you add an anchor tag to any new guidelines that you add and maintain the naming convention.
-->

## History

<details>
  <summary>Expand change history</summary>

| Date        | Notes                                                          |
| ----------- | -------------------------------------------------------------- |
| 2024-Sep-17 | Added guidance on Secrets                                      |

</details>

## Introduction

These are prescriptive guidelines that Radius contributors MUST follow while designing APIs to ensure that customers have a great experience. These guidelines help make Radius APIs:
- Developer friendly via consistent patterns
- Efficient & cost-effective

Technology and software is constantly changing and evolving, and as such, this is intended to be a living document. [Open an issue](https://github.com/radius-project/design-notes/issues) to suggest a change or propose a new idea. 

### Prescriptive Guidance
This document offers prescriptive guidance labeled as follows:

:white_check_mark: **DO** adopt this pattern. If you feel you need an exception, present your reason at a design review discussion **prior** to implementation.

:ballot_box_with_check: **YOU SHOULD** adopt this pattern. If not following this advice, you MUST disclose your reason during a design review discussion.

:heavy_check_mark: **YOU MAY** consider this pattern if appropriate to your situation.

:warning: **YOU SHOULD NOT** adopt this pattern. If not following this advice, you MUST disclose your reason during a design review discussion.

:no_entry: **DO NOT** adopt this pattern. If you feel you need an exception, present your reason at a design review discussion **prior** to implementation.


## API Foundation:

Radius provides an HTTP-based API that deploys and manages cloud-native applications as well as on-premise or cloud resources.
The overall design of the Radius API is based on the Azure Resource Manager API (ARM). Radius generalizes the design of ARM, removes proprietary Azure concepts, and extends the ARM contract to support the conventions of other resource managers like Kubernetes or AWS Cloud-Control. 

Radius utilizes TypeSpec for describing API model definitions and operations, providing a consistent approach to defining the structure of API payloads and interactions. The TypeSpec specification serves as the source to generate corresponding OpenAPI 2.0 (swagger) API documentation.

Reference: 
- [Radius API](https://docs.radapp.io/concepts/technical/api/)
- [Radius Architecture](https://docs.radapp.io/concepts/technical/architecture/)

<a href="#secrets" name="secrets"></a>
### SECRETS

Radius enables users to securely store sensitive data, such as passwords, OAuth tokens, and SSH keys, in [Kubernetes secrets](https://kubernetes.io/docs/concepts/configuration/secret/).
The secrets are stored in secret store resource [Applications.Core/secretStores](https://docs.radapp.io/reference/resource-schema/core-schema/secretstore/)

SecretReference and SecretConfig are entities prescribed to ensure sensitive information is handled securely and consistently across different components and resources in Radius. The following are their definitions along with usage examples:

### SecretReference Model

<a href="#secret-model" name="secret-model">:white_check_mark:</a> **DO** follow this structure when adding support for secrets to resources or components in Radius.

```diff

@doc("This specifies a reference to a secret. Secrets are encrypted, often have fine-grained access control, auditing and are recommended to be used to hold sensitive data.")
model SecretReference {
  @doc("The ID of an Applications.Core/SecretStore resource containing sensitive data.")
  source: string;

  @doc("The key for the secret in the secret store.")
  key: string;
}

```        

#### Usage of SecretReference in EnvironmentVariables:

<a href="#secret-envvar" name="secret-envvar">:white_check_mark:</a> **DO** The above model for Secrets should be utilized for handling sensitive information through environment variables. This solution is in alignment with Kubernetes design patterns. The structure also allows for environment variables to refer to other resources such as ConfigMaps, Pod Fields etc. in the future. 
Examples include cases where environment variables containing sensitive information are injected into a container or used in a Terraform execution process.


```diff

@doc("environment")
env?: Record<EnvironmentVariable>;

@doc("Environment variables type")
model EnvironmentVariable {

  @doc("The value of the environment variable")
  value?: string;

  @doc("The reference to the variable")
  valueFrom?: EnvironmentVariableReference;
}

@doc("The reference to the variable")
model EnvironmentVariableReference {
  @doc("The secret reference")
  secretRef: SecretReference;
}

```

#### Example in Bicep

```diff

env: {
  DB_USER: { value: 'DB_USER' }
  DB_PASSWORD: {
    valueFrom: {
      secretRef: {
        source: secret.id
        key: 'DB_PASSWORD'
      }
    } 
  }
} 

```


### SecretConfig Model

<a href="#secretconfig-model" name="secretconfig-model">:white_check_mark:</a> **DO** follow this structure when a component or resource in Radius requires authentication to external systems, such as private container registries, TLS certificates.

```diff

@doc("Secret Configuration used to authenticate to external systems.")
model SecretConfig {
  @doc("The ID of an Applications.Core/secretStores resource containing credential information.")
  secret?: string;
}

```        

#### Using SecretConfig for Git repository access

<a href="#secretconfig-ext" name="secretconfig-ext">:white_check_mark:</a> **DO** In the following example SecretConfig is used to manage authentication for accessing private Terraform modules from Git repository sources.

```diff

@doc("Authentication information used to access private Terraform modules from Git repository sources.")
model GitAuthConfig {
  @doc("Personal Access Token (PAT) configuration used to authenticate to Git platforms.")
  pat?: Record<SecretConfig>;
}

```

#### Example in Bicep

```diff

recipeConfig: {
  ...
  git: {
    pat: {
      'github.com':{
         secret: secret.id
      }
    }
  }
  ...
}

```


