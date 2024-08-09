# Title Simulated Environment

* **Author**: Vinaya Damle (@vinayada1)

## Overview

We want to introduce the concept of a "simulated environment" that doesn't deploy any backing resources. The purpose of this document is to review the schema changes required to support this.

## Terms and definitions

NA

## Objectives

We would like the users to have the ability to see what resources would be deployed and visualize the application graph without actually deploying any resources. For this, we will introduce the concept of a simulated environment.

> **Issue Reference:** https://github.com/radius-project/radius/issues/6336\

### Goals

- Add a boolean flag on the environment resource to mark it as a simulated environment

### Non goals

NA

## Design


### Design details

We will add a boolean property called "simulated" on the environment resource. If this property on the environment is set, the application deployment will skip the actual deployment of resources.

### API design (if applicable)

Schema Changes:-
```
doc("Environment properties")
model EnvironmentProperties {
  .....

  @doc("Simulated environment.")
  simulated?: boolean;
```

## Alternatives considered

NA

## Test plan

NA

## Security

NA

## Compatibility (optional)

NA

## Monitoring

NA

## Development plan

- Schema Changes
- Code changes to skip deployments based on the newly added property in the schema
- Tests

## Open issues

- Do we need simulated environments to be supported with recipes? Yes