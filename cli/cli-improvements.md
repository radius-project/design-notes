# CLI improvements

* **Status**: Draft
* **Author**: Nithya Subramanian (@nithyatsu)

## Overview

When a Radius user wants to deploy an application, a few concepts come into picture

*	Resource group – Radius resource group mimic ARM resource group and are tied to RBAC. 
*	Environment – Radius environment is what the application uses. The environment can have a set of recipes and is “in” a resource group itself, since it requires RBAC to make sure not everyone can edit it and for example, point a Production environment to use Sandbox recipes.
*	Workspace – Radius workspace has a "per user" information about the kubernetes context/ cluster where the Radius services are hosted as well as the “default” environment and resource group  of the user.

There are a few points that can be improved in the way these concepts work together:

*	Today, when we rad init,  we make a “default” resource group which creates a single resource group perception for radius users. However, we would want multiple resource groups for multiple copies of same application, based on the scenario. For instance, a cool-app deployment to production and the same cool-app deployment to staging would not need the same RBAC and hence have to be two copies in two resource groups. Also, they have to be in different resource groups so that there are no resource-id conflicts. Would it be a better idea, to not create default group during rad init, and create/use a resource group based on what the users chooses as the application name and its environment [production, staging,[enter your own]]?


*	Today, if we want two copies of same app, we want to create two resource group, have an environment in each of those resource group, have the user explicitly use rad group switch and rad env switch or specify the group and env using flags, to make a desirable deployment. Would it be a better idea to, instead of expecting users specify instructions explicitly, print a default behavior which is intuitive and deploy the app if user confirms?


*	Since only ResourceGroup  is part of resource ID, if we deploy multiple application to same resource group (ex: we do this in our functional tests) we have to ensure the resource names across applications are all unique. This limitation has no reason to be present in real world and is a hinderance since the developer has to make sure to choose names that are not already taken across applications. Again, we could create a resource group based on user scenario per instance of application using an agreed upon convention to solve this problem. 

## Terms and definitions

| Term | Definition |
|---|---|
| Workspace |Workspaces allow managing multiple Radius environments using a local, client-side, configuration file. |
| Environment | Environments are server-side resources that exist within the Radius control-plane. Applications deployed to an environment will inherit the container runtime, configuration, Recipes, and other settings from the environment.|
| Scope | Radius resource groups (scopes) are used to organize Radius resources, such as applications, environments and portable resources.|


## Objectives

### Goals

* Improve the deploy experience (rad deploy, rad run) with better default behaviors for common patterns (deploy app to multiple environments, deploy multiple apps to save environment).
* Improve the ability of various CLI commands and the Dashboard (rad resource list) to work with resource groups, and to work across all resource groups.
* Make resource groups more visible in our documentation, and provide recommendations towards common patterns.

### Non-Goals

* (out of scope) Server-side changes (other than Dashboard).

### User scenarios

* As someone new to Radius, I am using the tutorials to get familiar with its concepts. I do not want to get into details, yet. 

* I would like to deploy multiple applications to the same environment.

* I would like to deploy a single application to multiple environments.
  
## Design

Today, we establish a default resource group during `rad init` process in `config.yaml`.  All of the `rad` cli commands operate on this resource group, unless a different resource group is specified using option `-g`.

With respect to our user scenarios, this means, 

* A new user has to understand and manage Radius resource groups to be able to deploy a simple tutorial application.

* The user has to pick unique resource names across all applications since they would all be deployed to the same resource group (default behavior).

* The user has to create multiple resource groups and manage deploying the same application to different resource groups in order to have multiple copies of the application.

In order to improve the default experience of `rad` commands, we want to adopt a convention-over-configuration approach in determining the Radius resource group to be operated on. This means,

* we will not establish a default resource group during the 'rad init' process in config.yaml.
* we will establish the convention that by default, an application ```<app-name>``` to be deployed using environment ```<env-name>``` will be a under the resource-group ```<app-name>-<env-name>```. This resource group will be created if needed during the execution of the command.

The workspace entry created as part of rad init will be as below: 

**Current config.yaml**

```
workspaces:
    default: default
    items:
        default:
            connection:
                context: radius-agent-aks
                kind: kubernetes
            environment: /planes/radius/local/resourceGroups/default/providers/Applications.Core/environments/default
            scope: /planes/radius/local/resourceGroups/default
```

**Proposed config.yaml**

```
workspaces:
    default: default
    items:
        default:
            connection:
                context: radius-agent-aks
                kind: kubernetes
            environment: /planes/radius/local/resourceGroups/default/providers/Applications.Core/environments/default
```

All CLI commands will be impacted due to this change. Most commands have to  support taking a mandatory resource group argument.


### Examples

Consider a user new to Radius. As initial experiment, the user runs rad init and deploys a simple application simple-app using ```rad deploy simple-app.bicep```. Authors a second simple application my-simple-app, retaining names of some resource in the first bicep. ```rad deploy my-simple-app.bicep``` fails with message similar to

```
{
    "code": "BadRequest",
    "message": "Attempted to deploy existing resource 'frontendcontainer' which has a different application and/or environment. Options to resolve the conflict are: change the name of the 'frontendcontainer' resource in 'my-simple-app' application to create a new resource, or use '/planes/radius/local/resourcegroups/default/providers/Applications.Core/applications/simple-app' application and '' environment to update the existing resource 'frontendcontainerdns'.",
    "target": "/planes/radius/local/resourceGroups/default/providers/Applications.Core/containers/frontendcontainer"
}
```

----
Consider a user wants to deploy two instances of `cool-app` application, into `production` and `qa` environments. 

##### How can the user do it today

Ops creates environments that can be used to deploy applications.
1. rad group create qa
2. rad group switch qa
3. rad env create qa 
4. rad group create production
5. rad group switch qa
6. rad env create qa

Dev deploys single application to production and qa
1. rad group switch production
2. rad env switch production
3. rad deploy cool-app.bicep # app deployed to qa resource group 
4. rad group switch qa
5. rad env switch qa
6. rad deploy cool-app.bicep # app deployed to production resource group 

##### How it would change

Ops creates environments that can be used to deploy applications.
1. rad group create qa
2. rad group switch qa
3. rad env create qa 
4. rad group create production
5. rad group switch production
6. rad env create production

Dev deploys single application to production and qa environments
1. rad env switch production -g production
2. rad deploy cool-app.bicep 
`cool-app` being deployed using environment production in resource group cool-app-production ...
:  
3. rad env switch qa -g qa
4. rad deploy cool-app.bicep 
`cool-app` being deployed using environment qa in resource group cool-app-qa ...
:

------

Consider user wants to deploy 2 applications to ```production``` environment, each of the application has a container ```front-end```

##### How can the user do it today

Ops creates environments that can be used to deploy applications.
1. rad group create production
2. rad group switch production
3. rad env create production

Dev deploys application1 and application2 to production 
1. rad group switch app1
2. rad env switch production
3. rad deploy cool-app.bicep # app deployed to app1 resource group 
4. rad group switch app2 
5. rad deploy cool-app.bicep # app deployed to production resource group 

Note: step 4 is neccessary to avoid name collison between the applications since they both have a container ```front-end```. If the user had chosen to deploy app1 and app2 to default group, app2's deployment would have failed.

##### How it would change

Ops creates environments that can be used to deploy applications.
1. rad group create production
2. rad group switch production
3. rad env create production

Dev deploys app1 and app2 to production environment
1. rad env switch production -g production
2. rad deploy app1.bicep 
`app1` being deployed using environment production in resource group app1-production ...
:  
3. rad deploy app2.bicep 
`app2` being deployed using environment production in resource group app2-production ...
:


## Impact on CLI commands

### rad deploy 

As outlined in previous sections, the 'rad deploy' command would adhere to this high-level flow:

* Extract the application name (from bicep) and environment name from ```config.yaml```.
* PUT group at ```/planes/radius/local/resourceGroups/<app-name>-<env-name>```.
* PUT application at ```/planes/radius/local/resourceGroups/<app-name>-<env-name>/providers/Applications.Core/applications/<app-name>```.

If the application has already been deployed to this resource group, we would inform the user about the application's patching. The deploy command should present an informational message indicating the resource group in which the application is created. Documentation should include the rad deploy app.bicep -g cool-group -e cool-env usage to enforce deployment into specific resource groups/environments.

### rad init
                                                       
rad init will not set a default resource group in config.yaml.

### rad group 

```rad group switch``` will become obsolete since it deals with setting the current scope as a configuation in config.yaml.

### rad env 

#### rad env show
#### rad env delete
#### rad env create

Since the idea of "default" scope is going away, the aforementioned 'rad env' commands will require a mandatory -g flag to specify the group.

#### rad env list 

User should be able to request list of environments at two levels:
1. list all environments in a specified resource group
```rad env list -g cool-group```
2. list all environments across all resource groups 
```rad env list -A```


### rad workspace 

workspace will not contain deafult scope and all workspace commands should be modified to reflect this. 

### rad app

#### rad app graph app-name
#### rad app show app-name  
#### rad app delete app-name  

For aforementioned commands, we could construct resource group  = <app-name>-<env-name>. This resource group can be used while retaining rest of the implementation.

The commands could maintain the -g flag to accommodate applications deployed using behavior that deviates from the default.

#### rad app list

User should be able to request following list of applications:

1. List all applications across all resource groups - this should be supported.
2. List all applications within a specific resource group - this requires the -g flag to specify the resource group.
3. List all applications deployed to specific environment - this requires the -e flag to specify the environment. This would query all applications across resource groups, and filter applications in an environment using pattern *-<env-name> for resource groups.

### rad resource 

### rad resource list

The list can be generated at the following levels:

1. List all resources of a given type across all groups.
2. List all resources of a given type in a group. Requires a -g group flag.
3. List all resources of a given type in a given application and group. Requires a -a application and -g group flags.

### rad resource show, delete, expose, log 

The aforementioned commands require a -g flag to specify group.

### rad recipes

### rad recipe show, list, register, unregister 

These commands should support -g flag mandatorily if a -e is used to specify the environment name. If -e is not specified, default environmentID in config.yaml can be used.














































