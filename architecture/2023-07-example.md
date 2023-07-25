## Overview

Radius service architecture vNext proposal aims to enhance Radius service architecture and define a new programming model that addresses the following problems:

#### Not using rewrap column width limit

Radius has three core services: Universal Control Plane (UCP), Applications RP, and Deployment Engine (DE). UCP and Applications RP utilize Kubernetes CRD for storing resource metadata and implementing queue, while Deployment Engine uses an in-memory data store for resource metadata and queue. The in-memory data store is designed for development/test purposes, which raises questions about service reliability and resiliency. 

#### Using rewrap column width limit

> Note: it is automatically word-wrap in the text file. So it is easier to read
> on any editor and add the comment on PR.

Radius has three core services: Universal Control Plane (UCP), Applications RP,
and Deployment Engine (DE). UCP and Applications RP utilize Kubernetes CRD for
storing resource metadata and implementing queue, while Deployment Engine uses
an in-memory data store for resource metadata and queue. The in-memory data
store is designed for development/test purposes, which raises questions about
service reliability and resiliency. 
