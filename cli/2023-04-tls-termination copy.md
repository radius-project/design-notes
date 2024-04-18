# TLS termination

* **Status**: Draft
* **Author**: Nithya Subramanian (@nithyatsu)

## Overview

When a Radius user wants to deploy an application, a few concepts come into picture

*	Resource group – Radius resource group mimic ARM resource group and are tied to RBAC. 
*	Environment – Radius environment is what the application uses. The env can have a set of recipes and is “in” a resource group itself, since it requires RBAC to make sure not everyone can edit it and for example, point a Production environment to use Sandbox recipes.
*	 Workspace – Radius workspace has a per user information about the kubernetes context/ cluster where the Radius services are hosted as well as the “default” environment and resource group  of the user.

There are a few points that can be improved in the way these concepts work together today:

*	Today, when we rad init,  we make a “default” resource group which creates a single resource group perception for radius users. However, we would want multiple resource groups for multiple copies of same application, based on the scenario. For instance, a cool-app deployment to production and the same cool-app deployment to staging would not need the same RBAC and hence have to be two copies in two resource groups. Would it be a better idea, to not create default group during rad init, and create/use a resource group based on what the users chooses as the application name and its environment [production, staging,[enter your own]]?


*	Today, if we want two copies of same app, we want to create two resource group, have an environment in each of those resource group, have the user explicitly use rad group switch and rad env switch or specify the group and env using flags, to make a desirable deployment. Would it be a better idea to, instead of expecting users specify instructions explicitly, print a default behavior which is intuitive and deploy the app if user confirms?


*	Since only ResourceGroup  is part of resource ID, if we deploy multiple application to same resource group (ex: we do this in our functional tests) we have to ensure the resource names across applications are all unique. This limitation has no reason to be present in real world and is a hinderance since the developer has to make sure to choose names that are not already taken across applications. Again, we could create a resource group based on user scenario per deployment using an agreed upon convention to solve this problem. 



## Terms and definitions

| Term | Definition |
|---|---|
| cert-manager | Cert-manager is a powerful and extensible X.509 certificate controller for Kubernetes|


## Objectives

### Goals

Redesign User Experience with rad deploy to make the way it works with various radius concepts more intuitive

### Non-Goals

NA

### User scenarios

1. As a developer, I would like to not have to enter or set the group my      
application would be deployed to, every time explicitly. It would be useful if Radius picks the group name so that all related resources are grouped together and do not run into conflicts with resources from other  applications. I should be able to override this behavior as needed.
   
2. As a developer, I do not want to be responsibile for choosing names of     components within my application so that they dont conflict with any other applications that are deployed.

## Design

We want to redesign rad cli commands using a  convention over configuration approach for choosing which radius resource group to operate on. 
With this, we will not create a default resource group or environment in rad init. The workspace created as part of rad init will look in config.yaml as below: 

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

Based on this, all CLI commands will be impacted. 

### rad deploy 

We could consider a convention over configuation approach while deploying applications. At present, we use the resource group stored in configuration file as where the application would be deployed.

We could instead consider deploying applicationss into a resource-group that’s created on fly  (unless it already exists) based on a convention, say  <app-name>-<env-name>. the environment name would be input by the user (developer) and would refer an existing environment set up by operations.

```rad deploy``` would then have below high level flow:
1.	Get the application name (from bicep)
2.	Prompt user for the ID of the environment we are deploying into
3.	PUT group <env-name>-<application-name> 
4.	PUT application as /planes/radius/local/resourceGroups/<env-name>-<application-name>/providers/Applications.Core/applications/<application-name>

If the application was already deployed to this resource group, we would inform the user and confirm patching the application.

The deploy command should print an informational message, ask for proceeding, and print informational message about ```rad deploy app.bicep -g cool-group -e cool-env``` usage to enforce deployment into specific resource-group/environments.


### rad init

Today, rad init flow is as below

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

Some points of interest

1. It creates environment "default" if that does not already exist
2. It creates resource group "default" if that does not exist
3. Always scaffolds application, with the name = current directory
4. Updates config.yaml and rad.yaml

This will change to 

1. Requesting default environment's name from user and creating the environment with that name
2. Not creating any resource group
3. Always scaffolds application, with the name = current directory
4. Updates config.yaml and rad.yaml. No scope will be stored in any config file. 

### rad group 

The ```rad group``` family of commands have no changes. ```rad group switch``` alone will become obsolete since it deals with setting the current scope as a configuation in config.yaml.

### rad env 

The ```rad env``` family of commands currently operate in the scope that is stored in config.yaml, unless a scope is explicitly specified using -g flag.   

Since the idea of "default" scope is going away, all rad env commands will need a mandatory -g flag to specify the group.

### rad workspace 

workspace will not contain scope informtion and all workspace commands will be modified to reflect this. 

### rad app

The ```rad app``` family of commands currently operate in the scope that is stored in config.yaml, unless a scope is explicitly specified using -g flag. 

#### rad app graph app-name
#### rad app show app-name  
#### rad app delete app-name  

We construct the resource group  = <env-name-from-config>-<app-name>.
Then we could GET or DELETE the desired information, such as application graph or application info.

The commands could retain -g flag, to specify a resource group explicitly.

#### rad app list

This list can be at the below levels:
1. list all applications across all resource groups - should be supported.
2. list all applications in a given resource group - requires -g flag to specify the resource group.































