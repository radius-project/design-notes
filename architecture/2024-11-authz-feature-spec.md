# Radius Authorization Feature Specification

* **Author**: Zach Casper (@zachcasper)

## Summary

Radius is inherently a multi-tenant system with several different personas. Separation of concerns between developers and platform engineers is a core concept. Developers model their applications using resource types abstracted away from the infrastructure while platform engineers configure how those resource types are deployed. Radius is also designed to support multiple different applications, developers, and groups of developers. 

The building blocks of this multi-tenancy is implemented today with resource groups. Every resource, whether it is an application, an environment, or a recipe, resides in a resource group. However, today, access to Radius is binary—you ether have access to the API or you do not. It is not possible to control permissions to these resources and resource groups. This feature specification describes the requirements and user experience for authorizing users to perform a granular set of actions on a granular set of resources and resource groups.

### Goals

This feature specification describes:

* The default configuration for first-time users or local developers
* Out of the box roles which can be assigned or customized

* Creating roles granting permission to perform actions on Radius resources including applications, user-defined resource types, environments, and recipe registrations
* Creating roles granting permission to manage resource groups and permissions

### Non-goals (out of scope)

This feature specification only addresses authorization. A separate feature specification is under development for authenticating users.

## Definition of Terms

Throughout this document, several terms will be used which have specific meaning within the context of Radius authorization.

**Authentication** – Validation that the user is who they claim to be; typically via a certificate, username/password, or passkey; sometimes abbreviated as AuthN

**Authorization** – Validation that the user has permission to perform the requested function within the system; typically is implemented as a set of permitted actions on a specified set of resources (e.g., user X has permission to perform the restart action on the set of virtual machine within this group); typically abbreviated as AuthZ

**Identity Provider** – A system which authenticates users to client applications; user identities can be stored within the identity provider system or federated out to other identity providers; identity providers alleviate the need for client applications to store user credentials or implement authentication mechanisms; typically abbreviated as IdP; examples include Entra ID, Ping Identity, or Okta

**Identity and Access Management** – An overloaded term which can refer to the overall topic of authentication and authorization, modules within a system which implement both authentication and authorization, or product names such as AWS IAM

**Role** – A collection of permitted actions; typically named for the user persona

**Action** – An act performed upon a resource such as create, read (or show), update, delete, list, or a resource-specific action such as restart; typically is mapped one to one with an API call or a CLI command

**Scope** – The set of resources which actions are allowed to be performed; within Radius, the scope is typically a resource group

**Role Definition** – A commonly used pattern in cloud systems which includes the role name and a collection of allowed actions; specific implementations include Kubernetes Roles, AWS IAM roles, Azure role definitions, or Google Cloud custom roles

**Role Assignment** – A commonly used pattern in cloud systems which maps a role definition to a user or set of users; specific implementations include Kubernetes RoleBinding, Azure role assignments, or Google Cloud role binding

## User profiles and challenges

**Platform engineers** – Platform engineers configure and maintain Radius. These users are primarily concerned with configuring Radius to support the various development and infrastructure teams within their organization. They need a flexible authorization scheme built into Radius which is easy to setup in the beginning, but flexible enough to grow as Radius is used by more teams in more ways. This flexibility will enable platform engineers to grant and restrict actions at a high level on a wide range of resources, or on a fine-grained level.

**Developers** – Developers use the resource types available to them to model their applications and deploy to environments which they have access to. Some developers will use Radius locally to run their applications, but many will not use Radius directly; instead relying on their organization's CI/CD system to run their application in cloud environments. These developers will not have deep knowledge of Radius beyond using the resource types made available to them. Developers who are using Radius locally should not have to concern themselves with Radius authorization features.

## Scenarios

### Scenario 1 – Getting started and local development

When a first-time user or a developer installs Radius, the authorization configuration should be wide open to the user. The user should never have to configure anything or be presented with any options. This is the behavior of Radius today. There are no enhancements needed to support this scenario.

### Scenario 2 – Simple internal developer platform

A simple internal developer platform is a configuration similar to a first-time setup by a platform engineer for their organization's developers. In this scenario, Radius will be used to manage a handful of applications each with their own development team. A single platform engineering team manages all environments, resource types, and recipes. 

For this scenario, the platform engineer will use the authorization features of Radius to control permissions. However, since Radius is new to the organization, the platform engineer expects to use prescriptive guidance from Radius using pre-defined roles and best practices.

Radius will ship with several roles pre-defined. These roles grant permission to perform actions on certain resource types. These roles will not be assigned to any user out of the box except the Radius administrator role. The documentation will describe how to assign a role to a user or group of users and describe how to define the scope of those permissions.

#### Out-of-the-box roles

**Radius Administrator** – The Radius Administrator has superuser access to the Radius system. 

**Resource Group Administrator** – The Resource Group Administrator role grants permission to manage resource groups, role definitions, and role assignments.

**Developer** – The Developer role grants permission to create applications and application resources. It also allows them to deploy these resources to environments. Out of the box, developers create, update, delete, list, get, and deploy resources from `Application.Core` (except Environment and Extenders), `Applications.Datastores`, `Applications.Messaging`, and `Applications.Dapr`. Environments is reserved for the platform engineering team. Extenders are not included by default to encourage the use of pre-defined resource types. If the platform engineer creates additional resource types, they can add the resource type namespace (aka resource provider name) to this role.

**Deployer** – The Deployer role is for the same persona as the Developer role. The role exists to control which environments the developer can deploy applications to. The Developer role and Deployer role are typically assigned to the same user at the same time, just with different scopes.

**Environment Administrator** – The Environment Administrator role grants permission to manage environments and connections to cloud platforms. 

**Resource Type Administrator** – The Resource Type Administrator role grants permission to manage the resource types which are available in the Radius tenant. The Radius Administrator or Resource Group Administrator can further control who can use those resource types within their applications.

**Recipe Administrator** – The Recipe Administrator role grants permission to register and unregistered recipes in an environment. 

| Role                         | Short Name   | Resources                                            | Allowed Action |
| ---------------------------- | ------------ | ---------------------------------------------------- | -------------- |
| Radius Administrator         | radius-admin | *                                                    | *              |
| Resource Group Administrator | group-admin  | /planes/radius/{planeName}/resourcegroups/           | *              |
|                              |              | /planes/radius/MyCompany/rolesdefinitions/           | *              |
|                              |              | /planes/radius/MyCompany/roleassignments/            | *              |
| Developer                    | developer    | Applications.Core/applications/applications          | *              |
|                              |              | Applications.Core/applications/applications          | *              |
|                              |              | Applications.Core/applications/containers            | *              |
|                              |              | Applications.Core/applications/gateways              | *              |
|                              |              | Applications.Core/applications/secretStores          | *              |
|                              |              | Applications.Core/applications/volumes               | *              |
|                              |              | Applications.Datastores/*                            | *              |
|                              |              | Applications.Messaging/*                             | *              |
|                              |              | Applications.Dapr/*                                  | *              |
| Deployer                     | deployer     | Applications.Core/environments/                      | deployTo       |
| Environment Administrator    | env-admin    | Applications.Core/environments                       | *              |
|                              |              | Applications.Core/secretStores                       | *              |
|                              |              | System.AWS/credentials                               | *              |
|                              |              | System.Azure/credentials                             | *              |
|                              |              | System.Kubernetes/credentials (future functionality) |                |
| Resource Type Administrator  | type-admin   | System.Resources/resourceproviders                   | *              |
| Recipe Administrator         | recipe-admin | Only the recipe property of environments             | *              |

#### User story 1 – Assigning the Radius Administrator role

> [!NOTE]
>
> In a new Radius installation, there is no identity provider (IdP) configured. Radius does not have a built-in identity system, therefore the Radius administrator must configure an IdP before Radius will recognize and authorize the user. Roles can be assigned prior to IdP configuration, but without an IdP, the user still cannot be authorized. 

As a platform engineer, I just installed Radius using my Kubernetes cluster credentials. I want to access Radius via my typical user credentials from my IdP.

**User Experience**

```bash
# Assign the Radius Administrator role to a user from the IdP
rad role assignment create \
  --assignee me@my-company.net \
  --role radius-admin \
  --scope /planes/radius/*
```

**Result**

1. `me@my-company.net` is added to the out-of-the-box role `radius-admin` 
2. The user can now authenticate to Radius using the IdP and continue administering Radius with the users standard credentials

**Exceptions**

The operation fails and informs the user interactively if:

* The current user is not a member of the `cluster-admin` RoleBinding in the Kubernetes cluster

#### User story 2 – Assigning the Developer role to the default application resource group

As a platform engineer, I need to assign the Developer role to a developer for the first time. There are no applications created in Radius yet and I am not familiar with resource groups yet.

 **User Experience**

When Radius is installed, two resource groups are created:

1. `rg-default-application` – When a user deploys an application without specifying a resource group, it is deployed to this resource group.
2. `rg-default-environment` – When a user creates an environment without specifying a resource group, it is deployed to this resource group.

> [!NOTE]
>
> The default resource group is split to encourage best practices for actual Radius use. Splitting resource groups introduces the concept of resource groups and Radius' authorization model.

Given this behavior, the developer will be granted the Developer role scoped to the `default-application` resource group and the Deployer role scoped to the `rg-default-environment`

```bash
# Assign the Developer role to a user from the IdP scoped to the rg-default-application resource group
rad role assignment create \
  --assignee developer@my-company.net \
  --role developer \
  --scope /planes/radius/MyCompany/resourceGroup/rg-default-application
rad role assignment create \
  --assignee developer@my-company.net \
  --role deployer \
  --scope /planes/radius/MyCompany/resourceGroup/rg-default-environment
```

**Result**

1. `developer@my-company.net` is added to the out-of-the-box role `developer` for the `rg-default-application` resource group 
2. `developer@my-company.net` is added to the out-of-the-box role `deplorer` for the `rg-default-environment` resource group

**Exceptions**

The operation fails and informs the user interactively if:

* The current user is not a member of the `radius-admin` or `group-admin` roles

### Scenario 3 – Multi-tenant enterprise

In a complex, multi-tenant enterprise environment, Radius will be used by multiple development teams to build and maintain multiple different applications. Infrastructure management duties will be split across multiple infrastructure teams. For example, cloud environments may be maintained by a cloud infrastructure team while on-premises is maintained be a data center team. Resource types and recipes may be maintained by different teams as well. The DBA team may maintain database resource types and recipes while an enterprise architecture team maintains other resource types. 

Large enterprises need a flexible authorization model which enables them to tightly control a granular set of actions permitted on granular set of resources. To describe this scenario in more depth, a series of user stories will be described based on the diagram below.

![](2024-11-authz-feature-spec/authz.png)

#### User story 3 – Create resource group for environments

As a platform engineer at a large enterprise, I need to create a resource group for a cloud environment and grant access to manage environments to my cloud infrastructure team. I have deleted the `rg-default-environment` resource group.

**User Experience**

```bash
# Create resource group for non-production environments
rad group create rg-non-prod-env
# Assign Environment Administrator role to the cloud engineering user group
rad role assignment create \
  --assignee cloud-engineering@my-company.net \
  --role env-admin \
  --scope /planes/radius/MyCompany/resourceGroup/rg-non-prod-env
```

**Result**

1. An empty resource group is created called `rg-non-prod-env`
2. The `env-admin` role is granted to the cloud engineering group to manage environments within the `rg-non-prod-env` resource group

**Exceptions**

The operation fails and informs the user interactively if:

* The current user is not a member of the `radius-admin` or `group-admin` roles
* The resource group already exists
* The role assignment already exists

#### User story 4 – Create resource group for an application

As a platform engineer at a large enterprise, I need to create a resource group for a new application and grant access to a development team. I only want the development team to use my company's resource types. I have deleted the `rg-default-application` resource group.

**User Experience**

The user needs to (1) modify the Developer role so that only MyCompany.App resource types can be used and (2) grant the updated Developer role to the IdP group.

```bash
# Update the Developer role
rad role definition update -f developer-role-definition.yaml
```

The contents of `developer-role-definition.yaml` is similar to:

```yaml
---
name: developer
description: Role to manage applications and application resources within a resource group
actions:
  # Grant permissions to CRUDL applications in the scoped resource group
  - Applications.Core/applications/*
  # Restrict using any resource type except those in the MyCompany.App namespace
  - MyCompany.App/*
```



```bash
# Create application resource group 
rad group create rg-app-1
# Assign the Developer role to the IdP group the new group
rad role assignment create \
  --assignee app-1-dev-team@my-company.net \
  --role developer \
  --scope /planes/radius/MyCompany/resourceGroup/rg-app-1
```

**Result**

1. `Applications.Core`, `Applications.Datastores`, `Applications.Messaging`, and `Applications.Dapr` were resource type namespaces were removed from the Developer role and `MyCompany.App` was added
2. An empty resource group is created called `rg-app-1`
3. The `developer` role is assigned to the app-1-dev-team@my-company.net group

**Exceptions**

The operation fails and informs the user interactively if:

* The current user is not a member of the `radius-admin` or `group-admin` roles
* The resource group already exists
* The role assignment already exists

#### User story 5 – Assigning the Resource Type Admin role

As a platform engineer at a large enterprise, I need to delegate the ability to manage application resource types in Radius to my enterprise architecture team.

**User Experience**

The user needs to (1) modify the Resource Type Administrator role so that only MyCompany.App resource types can managed and (2) grant the updated role to the IdP group.

```bash
# Create resource type administrator role
rad role definition create -f mycompany-app-resource-type-admin-definition.yaml
# Assign the Resource Type Administrator role to users in the enterprise architecture user group
rad role assignment create \
  --assignee ent-arch@my-company.net \
  --role mycompany-app-resource-type-admin \
  --scope /planes/radius/MyCompany/ResourceTypes
```

The contents of `resource-type-admin-definition.yaml` is similar to:

```yaml
---
name: mycompany-app-resource-type-admin
description: Role to manage resource types for MyCompany
actions:
  # Grant permissions to CRUDL resource types in the MyCompany.App resource type namespace only
  - System.Resources/resourceproviders/MyCompany.App/*
```

**Result**

1. `mycompany-app-resource-type-admin` role definition is created
2. The `mycompany-app-resource-type-admin` role is granted to the enterprise architecture IdP group

**Exceptions**

The operation fails and informs the user interactively if:

* The current user is not a member of the `radius-admin` or `group-admin` roles
* The role definition or assignment already exists

#### User story 6 – Assigning the Recipe Admin role

As a platform engineer at a large enterprise, I need to delegate the ability to manage recipes in Radius to my cloud engineering and DBA teams.

**User Experience**

```bash
# Assign the Recipe Administrator role to users in the cloud engineering user group
rad role assignment create \
  --assignee cloud-engineering@my-company.net \
  --role recipe-admin \
  --scope /planes/radius/MyCompany/resourceGroup/*
# Assign the resource type administrator role to users in the DBA user group
rad role assignment create \
  --assignee dba@my-company.net \
  --role recipe-admin \
  --scope /planes/radius/MyCompany/resourceGroup/*
```

**Result**

1. The `recipe-admin` role is granted to the cloud engineering and DBA groups to register and unregistered recipes on environments in resource groups in scope

**Exceptions**

The operation fails and informs the user interactively if:

* The current user is not a member of the `radius-admin` or `group-admin` roles
* The role assignment already exists

#### User story 7 – Deleting a role definition and assignment

As a platform engineer at a large enterprise, I need to delete a role definition and assignment I previously created.

**User Experience**

```bash
# Delete the previously defined role
rad role definition delete my-role-definition
# Error is shown because existing role assignments for this definition exist
ERROR: The following role assignments exist for the my-role-definition role definition
ROLE                ASSIGNEE            SCOPE
my-role-definition  dba@my-company.net  /planes/radius/MyCompany/resourceGroup/*
# Delete the role assignment
rad role assignment delete \
  --assignee dba@my-company.net \
  --role recipe-admin \
  --scope /planes/radius/MyCompany/resourceGroup/*
# Delete the role definition
rad role definition delete my-role-definition
```

**Result**

1. The role assignment for the `my-role-definition` role is removed from the `dba@my-company.net` group
2. The `my-role-definition` role definition is deleted

**Exceptions**

The operation fails and informs the user interactively if:

* The current user is not a member of the `radius-admin` or `group-admin` roles
* Role assignments for the role definition exist as shown in the example

## Feature Summary

| Priority | Size | Feature                                                      |
| -------- | ---- | ------------------------------------------------------------ |
| p0       | S    | Out-of-the-box role definitions                              |
| p0       | L    | UCP validates membership in either a role definition or the Kubernetes cluster-admin RoleBinding on each API call |
| p0       | S    | Create or delete a role assignment for an individual user from the configured IdP system for a resource group (`rad role assignment create --assignee user@mycompany.net`) |
| p1       | S    | Create or delete a role assignment for a group from the configured IdP system for a resource group (`rad role assignment create --assignee my-group@mycompany.net`) |
| p1       | M    | Ability to create, update, and delete a role definition      |
| p2       | M    | Replace the `default` resource group with `rg-default-application` and `rg-default-environment` and modify `rad environment`, `rad deploy`, and `rad run` to use these new groups |

## Future Features

In the future, Radius should consider these features:

* Ability to delegate role definitions and assignments to users and groups from the IdP (this spec assumes all role defining and assigning requires being a part of the `radius-system` RoleBinding)
* Ability to manage role definitions and assignments via the Radius dashboard
* Ability to automatically grant permission to an environment based upon existing permissions assigned to the AWS account or Azure resource group