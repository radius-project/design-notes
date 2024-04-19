# CLI improvements

* **Status**: Draft
* **Author**: Nithya Subramanian (@nithyatsu)

## Overview

When a Radius user wants to deploy an application, a few concepts come into picture

*	Resource group – Radius resource group mimic ARM resource group and are tied to RBAC. 
*	Environment – Radius environment is what the application uses. The env can have a set of recipes and is “in” a resource group itself, since it requires RBAC to make sure not everyone can edit it and for example, point a Production environment to use Sandbox recipes.
*	Workspace – Radius workspace has a "per user" information about the kubernetes context/ cluster where the Radius services are hosted as well as the “default” environment and resource group  of the user.

There are a few points that can be improved in the way these concepts work together today:

*	Today, when we rad init,  we make a “default” resource group which creates a single resource group perception for radius users. However, we would want multiple resource groups for multiple copies of same application, based on the scenario. For instance, a cool-app deployment to production and the same cool-app deployment to staging would not need the same RBAC and hence have to be two copies in two resource groups. Would it be a better idea, to not create default group during rad init, and create/use a resource group based on what the users chooses as the application name and its environment [production, staging,[enter your own]]?


*	Today, if we want two copies of same app, we want to create two resource group, have an environment in each of those resource group, have the user explicitly use rad group switch and rad env switch or specify the group and env using flags, to make a desirable deployment. Would it be a better idea to, instead of expecting users specify instructions explicitly, print a default behavior which is intuitive and deploy the app if user confirms?


*	Since only ResourceGroup  is part of resource ID, if we deploy multiple application to same resource group (ex: we do this in our functional tests) we have to ensure the resource names across applications are all unique. This limitation has no reason to be present in real world and is a hinderance since the developer has to make sure to choose names that are not already taken across applications. Again, we could create a resource group based on user scenario per deployment using an agreed upon convention to solve this problem. 

## Terms and definitions

| Term | Definition |
|---|---|
| workspace | |
| environment | |
| scope | |


## Objectives

### Goals

Enhance the user experience with 'rad deploy' by making its interaction with various Radius concepts more intuitive.

### Non-Goals

NA

### User scenarios
As a developer, I would prefer not to manually enter or set the group to which my application will be deployed each time. It would be beneficial if Radius could automatically select the group name, ensuring all related resources are grouped together and avoiding conflicts with resources from other applications. I should have the option to override this behavior when necessary.

As a developer, I would like to avoid the responsibility of selecting unique names for components within my application to prevent conflicts with other deployed applications.

## Design

We aim to redesign the rad CLI commands, adopting a convention-over-configuration approach to determine the Radius resource group to be operated on. Consequently, we will not establish a default resource group or environment during the 'rad init' process in config.yaml.

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

All CLI commands will be impacted due to this change.

### rad deploy 

We might consider adopting a convention-over-configuration approach for deploying applications. Currently, we use the resource group specified in the configuration file as the deployment target for the application.

Alternatively, we could deploy applications into a resource group that's dynamically created (unless it already exists) based on a convention, such as <app-name>-<env-name>. The environment name would be provided by the user (developer) and would reference an existing environment set up by operations.

The rad deploy command would then follow this high-level flow:

1. Retrieve the application name (from bicep)
2. Prompt the user for the ID of the environment to deploy into
3. PUT group ```/planes/radius/local/resourceGroups/<app-name>-<env-name>```
4. PUT application as ```/planes/radius/local/resourceGroups/<app-name>-<env-name>/providers/Applications.Core/applications/<app-name>```

If the application has already been deployed to this resource group, we would notify the user and confirm the application's patching.

The deploy command should display an informational message capturing where the application is created such as namespace and resource group, request confirmation to proceed, and provide an informational message about the ```rad deploy app.bicep -g cool-group -e cool-env``` usage to enforce deployment into specific resource groups/environments.


### rad init

rad init's flow today: 

```
nithya@Nithyas-MacBook-Pro bug7052 % rad init
                                              
Setup application in the current directory?   
  >  1. Yes                                   
    2. No

nithya@Nithyas-MacBook-Pro bug7052 % rad init
                                                         
Initializing Radius. This may take a minute or two...    
                                                         
✅ Use existing Radius 0.30.0 install on radius-agent-aks
✅ Use existing environment default                      
✅ Scaffold application bug7052                          
✅ Update local configuration                            
 ```

Key points to note:

1. If it doesn't already exist, create a "default" environment.
2. If it doesn't already exist, create a "default" resource group.
3. Scaffold a default application, using the current directory's name.
4. Update config.yaml and rad.yaml.

This will be modified to:

1. Request the name of the environment from the user and create an environment with that name.
2. Request user for application name.
3. PUT group ```/planes/radius/local/resourceGroups/<app-name>-<env-name>```
4. PUT application as ```/planes/radius/local/resourceGroups/<app-name>-<env-name>/providers/Applications.Core/applications/<app-name>```
5. Continue to scaffold the application, using the current directory's name.
6. Update both config.yaml and rad.yaml. No scope will be stored in any configuration file.

### rad group 

The ```rad group``` family of commands have no changes. ```rad group switch``` alone will become obsolete since it deals with setting the current scope as a configuation in config.yaml.

### rad env 

The ```rad env``` family of commands currently operate in the scope that is stored in config.yaml, unless a scope is explicitly specified using -g flag.   

Since the idea of "default" scope is going away, all rad env commands will need a mandatory -g flag to specify the group.

#### rad env list 

This could operate at two levels
1. list all environments in a specified resource group
```rad env list -g cool-group```
2. list all environments across resource groups
```rad env list```

#### rad env show
#### rad env delete
#### rad env create

The above commands require we specify the resource group using -g flag.

### rad workspace 

workspace will not contain scope information and all workspace commands will be modified to reflect this. 

### rad app

The ```rad app``` family of commands currently operate in the scope that is stored in config.yaml, unless a scope is explicitly specified using -g flag. 

#### rad app graph app-name
#### rad app show app-name  
#### rad app delete app-name  

We construct the resource group  = <env-name-from-config>-<app-name>.
Then this resource group can be used while retaining rest of the above commands implementation.

The commands could retain -g flag, to specify a resource group explicitly.

#### rad app list

The list can be generated at the following levels:

1. List all applications across all resource groups - this should be supported.
2. List all applications within a specific resource group - this requires the -g flag to specify the resource group.

### rad resource 

### rad resource list

The list can be generated at the following levels:

1. List all resources in a given application and group. 
2. List all resources in a given environment and group.
3. List all resources across all applications and environment.



































