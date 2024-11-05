# Radius Applications RP Component Threat Model

- **Author**: @nithyatsu

## Overview

This document provides a threat model for the Radius Applications RP component. It identifies potential security threats to this critical part of Radius and suggests possible mitigations. The document includes an analysis of the system, its assets, identified threats, and recommended security measures to protect the system.

The Applications RP component is responsible for managing applications and their resources. It communicates with UCP, Deployment Engine and Controller components to achieve this. 

## Terms and Definitions

| Term                  | Definition                                                                                                                                                                                  |
| --------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
|RP                 | Resource Provider                                                 |
| UCP                 | Universal Control Plane for Radius                                                                                                                                                   |
## System Description

Applications RP is a Radius service that acts as resource provider for application and its resources. It communicates over HTTP. The RP has a Datastore for storing Radius data, Message Queue for processing asynchronous request and a Secret Store for storing sensitive information such as certficates. All these are configurable components and support multiple implementations. Users and Clients cannot directly communicate with Applications RP. They instead communicate with UCP. UCP forwards relevant requests to Applications RP. Applications may have Kubernetes resources and cloud resources. Applications RP manages these Kubernetes resources on the user's behalf. This may launch user application's code on the same cluster as Radius, or a different cluster. it also has access to user's cloud credentials and manages user's cloud resources. Applications RP can invoke *recipes* which are bicep or terraform code. These recipes are used to deploy application infrastructure components like databases.  

### Architecture

Application RP recieves HTTP requests from UCP. It does not interface directly with user/ cli. These requests have untrusted json payloads. RP validates these payloads before consuming them. 

The RP consists of four types of resource providers for managing various types of resources in an application. `Applications.Core` resource provider manages core application resources such as application, environment, container and gateways. `Applications.Dapr` resource provider manages all dapr resources that are deployed as part of application. `Applications.Datastore` resource provider supports provisioning SQL database, Mongo DB and Redis Cache.
`Applications.Messaging` resources provider manages queues such as Rabbit MQ.

Applications RP has a key sub component `Recipe Engine` to execute `recipes`. 
`Recipes` are Bicep or Terraform code supplied by user that is used to deploy infrastructure components on Azure and AWS. The Bicep recipes are fetched from OCI compliant registries. Terraform recipes are public modules and fetched from internet too. 

In order to execute Terraform recipes, Applications RP installs latest Terraform. It also mounts an empty directory `/terraform` into Applications RP pod. It uses this directory for executing terraform recipes using the installed executable. The output resources generated from terraform module are converted to Radius output resources and stored in our datastore. 

In order to deploy bicp recipes, Applications RP sends a request to UCP, which in turn forwards it to Deployment Engine. 

Applications RP also allows users to create their own recipes and use them to provision their infrastructure. 

The RP can create kubernetes resources and manage them on behalf of the user. It can, for example, create a container based on the image provided by the user, which can in turn execute arbitrary code, and create other resources in the cluster as well as in AWS and Azure. The RP also can create managed identities for azure which will decide who can deploy and run user code.  

Applications RP has user's AWS / Azure credentials so that it can deploy and manage the cloud resources. The credentials are available as a kubernetes secret. While the credentials are registered and stored as secrets using an API, they are not available for retrieval or update through API calls. 

The RP uses a queue to process requests asyncronously. Information about resources that are deployed / being deployed is stored in a datastore. 

Below is a high level overview of various key subcomponents in Applications RP
![Applications RP](2024-10-applications-rp-threat-model/apprp.png) 

### Implementation Details

#### Use of Cryptography


#### Storage of secrets


##### Managing secrets for datastores


#### Access to cloud credentials


#### Access to cluster


#### Exposing User Application to Internet


#### Bicep Recipe execution


#### Terraform Recipe execution


#### Data Serialization / Formats


## Clients




## Trust Boundaries


## Assumptions 


## Data Flow


## Threats