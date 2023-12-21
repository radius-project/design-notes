# Irregular Types

* **Status**: Pending
* **Author**: Ryan Nowak (`@rynowak`)

## Overview

> ⚠️⚠️⚠️ 
> 
> *This design document is focused on low-level API design details. These details matter greatly for the consistency and construction of Radius/UCP as an API. It's hard to directly reason about the user-impact of the many small decisions we make, but getting those decisions right is still going to affect the experience users have.*
> 
> ⚠️⚠️⚠️

The majority of Radius/UCP HTTP APIs follow a regular pattern. This is important enough that it is documented as part of our [API concept docs](https://docs.radapp.io/concepts/api-concept/#resource-ids). It's a core concept in UCP called the resource ID.

```txt
{rootScope}/providers/{resourceNamespace}/{resourceType}/{resourceName}
```

The commonly encountered "things" in Radius/UCP (example: `Applications.Core/containers`) follow this pattern. An API consumer that can call the `LIST Applications.Core/containers` API intrinsically knows how to list other resource types such as `Applications.Core/environments` because of the high degree of consistency between the APIs.

This pattern is valuable from a mechanical point-of-view because of the information conveyed unambiguously in the URL. Anyone viewing this URL can "know" the resource type being acted upon without an additional source of information.

However, there are other APIs in Radius/UCP that do not follow this pattern. In particular the APIs provided as part of UCP for meta or mechanical functionality do not follow the consistent pattern. 

As an example the APIs for resource groups look like:

```
/planes/radius/{planeName}/resourceGroups/{resourceGroupName}
```

There are good reasons why. In particular the current design means that each level of the URL heirarchy has a meaning (example later in the doc). 

The purpose of this document will be to explore some of these design decisions and align on a set of principles for these irregular APIs. In particular I'd like to address some of the implementation and operational challenges.

## Terms and definitions

**Please read the [API Concept docs](https://docs.radapp.io/concepts/api-concept) if you are unfamiliar with UCP/ARM/Radius APIs. This is required reading for this document.**

| Term                    | Definition                                                                                                                                                                                                                               |
| ----------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Regular Resource Type   | A resource type that follows the `{rootScope}/providers/{resourceNamespace}/{resourceType}/{resourceName}` pattern. Also includes child resource types and extension resource types, though those will not be discussed in the document. |
| Irregular Resource Type | A resource type or other API that does not follow the regular pattern. Examples include resource groups, resource providers, and other "meta" functionality.                                                                             |

## Objectives

> **Issue Reference:** N/A so far. Came across this topic while working on https://github.com/radius-project/radius/issues/6688
> 
### Goals

- Define principles for irregular APIs in UCP that will:
  - Improve consistency of our APIs.
  - Simplify implementation of irregular APIs with respect to telemetry and RBAC.
  - Enhance code-reuse by enabling the use of our shared controllers in UCP.
- Update implementation to leverage these design principles.

### Non goals

- Non-goal: deviate significantly from the original inspiration (ARM). 
  - It's our goal to be *spiritually* compatible with ARM in most cases, and 100% compatible in some.
  - In particular "meta" functionality like resource groups and resource provider discovery APIs only require spiritual compatibility.

### User scenarios (optional)

Specific scenarios will not be written for this design. All users benefit if we build more consistent and maintainable APIs.

## Design

### Why irregular APIs?

Why have irregular APIs at all? Why not build everything as regular API? This section will provide some of the background. 

----

Here's an example of a **regular** resource type. 

```txt
# Example URI/resource ID.
/planes/radius/local/resourceGroups/examplegroup/providers/Applications.Core/containers/examplecontainer

rootScope = /planes/radius/local/resourceGroups/examplegroup
resourceNamespace = Applications.Core
resourceType = containers
resourceName = examplecontainer
```

---

Here's an example of an **irregular** resource type.

As an example the APIs for resource groups look like:

```
/planes/radius/{planeName}/resourceGroups/{resourceGroupName}
```

There are good reasons why. It makes logical sense that each level of the URL hierarchy should return something sensible. If you start with `/planes/radius/local/resourceGroups/examplegroup/providers/Applications.Core/containers/examplecontainer` you can iteratively delete segments of the URL and each new URL has a meaning:

```
/planes/radius/local/resourceGroups/examplegroup/providers/Applications.Core/containers/examplecontainer => Get details about a container
/planes/radius/local/resourceGroups/examplegroup/providers/Applications.Core/containers/ => List containers in 'examplegroup'
/planes/radius/local/resourceGroups/examplegroup/providers/Applications.Core => Describe the type metadata for types offered by 'Applications.Core'
/planes/radius/local/resourceGroups/examplegroup/providers => Describe the type metadata for all types valid in 'examplegroup'
/planes/radius/local/resourceGroups/examplegroup => Get details about 'examplegroup'
/planes/radius/local/resourceGroups => List resource groups in `local`
/planes/radius/local => Get the details of the 'local' Radius tenant/plane
/planes/radius => List the Radius tenants/planes
/planes => List all planes regardless of type
/ => (not currently defined)
```

This is elegant and **cool** 😎.

There are good reasons why it's this way. It makes logical sense that each level of the URL hierarchy should return something sensible. If you start with `/planes/radius/local/resourceGroups/examplegroup/providers/Applications.Core/containers/examplecontainer` you can iteratively delete segments of the URL and each new URL has a meaning:

### What's hard about this?

So given that we've already implemented a bunch of irregular APIs, what's the problem?

**One problem is consistency with our shared controllers.** The shared controller implementations can't fully be used with irregular APIs because the IDs can't be determined by the URL.

**One problem is consistency of behavior.** You can query a resource group using a URL like:

```txt
/planes/radius/local/resourceGroups/example-group
```

You'll get back a response like:

```json
{
  "id": "/planes/radius/local/resourcegroups/default",
  "location": "global",
  "name": "default",
  "tags": {},
  "type": "System.Resources/resourceGroups"
}
```

What if you tried to query the resource group using the regular API syntax?

```txt
/planes/radius/local/providers/System.Resources/resourceGroups/example-group
```

This results in a 404.

There's a few things that are odd about the this response. The `id` returned is a special form of a resource ID called a scope ID. There's a resource type being returned here, but it's not part of the `id` that's returned or part of the URL. We had to relax the resource ID parsing code in Radius to handle cases like this. 

**Another problem is the implementation.** Our storage, telemetry, and RBAC system are based on resource IDs, and operations. 

There are some big challenges that come up based on the fact that we can't determine the resource type based on the resource ID without hardcoding. In particular we don't know the resource type until we start executing-resource group-specific-code. Any functionality that relies on parsing the incoming URL into a resource ID will be wrong, because it can't determine the resource type. This makes code reuse hard between UCP and our other components.

### What about ARM?

ARM is the inspiration for UCP's design, and we kept this approach.

```bash
az rest --method get --url "/subscriptions/84bed6f7-783b-4cb5-a3c5-d42837c57770/resourceGroups/example-group?api-version=2020-06-01"
```

```json
{
  "id": "/subscriptions/84bed6f7-783b-4cb5-a3c5-d42837c57770/resourceGroups/example-group",
  "name": "example-group",
  "type": "Microsoft.Resources/resourceGroups",
  "location": "eastus",
  "tags": {},
  "properties": {
    "provisioningState": "Succeeded"
  }
}
```

The regular form of this ID also results in a 404 on ARM. 

```bash
az rest --method get --url "/subscriptions/84bed6f7-783b-4cb5-a3c5-d42837c57770/providers/Microsoft.Resources/resourceGroups/example-group?api-version=2020-06-01"
```


### Design details / API design

**This section proposes design principles for these cases.**

#### Avoid irregular APIs

First of all, we should limit the set of cases where we have to use irregular APIs. We can list all of these cases and the are all part of UCP.

- Planes/Tenants: eg. `/planes/radius/local`
- Resource Groups: eg. `/planes/radius/local/resourceGroups/example-group`
- Resource Providers (type discovery): eg. `/planes/radius/local/resourceGroups/example-group/providers`

We should forbid resource providers from adding their own irregular APIs. UCP is the front-proxy for resource providers, and we can validate during proxying that incoming requests match a regular pattern.

#### Standardize on regular APIs

For the cases listed above we should provide these as regular APIs in addition to the irregular APIs.

- Planes/Tenants: eg. `/planes/radius/local` -> `/planes/providers/System.Planes/radius/local`
- Resource Groups: eg. `/planes/radius/local/resourceGroups/example-group` -> `/planes/radius/local/providers/System.Resources/resourceGroups/example-group`
- Resource Providers (type discovery): eg. `/planes/radius/local/resourceGroups/example-group/providers` -> `/planes/radius/local/System.Resources/providers`

For the irregular types, the URL segment they provide (eg. `resourceGroups`) must match the unqualified resource type. 

Importantly, we should standardize on the **regular** format for resource IDs. For the resource types that represent a **scope** we can also include the scope format in a new property for use in client code.

**Looking for input on the right property name. I'm not sure scope is a great one.**

```jsonc
{
  "id": "/planes/radius/local/providers/System.Resource/resourceGroups/default", // Changed
  "scope": "/planes/radius/local/resourcegroups/default", // Added
  "location": "global",
  "name": "default",
  "tags": {},
  "type": "System.Resources/resourceGroups"
}
```

This change will simplify storage greatly. Our storage layer treats "scopes" and "resources" differently. We can remove a special case by being more consistent about how we use ids and types.

#### Irregular APIs are aliases for regular APIs

The irregular APIs (eg: `/planes/radius/local/resourcegroups/default`) will act as *aliases* for their regular counterpart (eg: `/planes/radius/local/providers/System.Resource/resourceGroups/default`). 

This means that these two URLs will map to the same operation for RBAC, telemetry, and storage purposes. 

We could also consider making the irregular APIs read-only. This would reduce the number of special cases we need to handle which keeping the elegant API design.

#### Remove the use of ARMRPC context middlware in UCP

UCP uses the ARM RPC context middleware, but it doesn't work well for irregular APIs. This isn't a great choice, and it was done out of expediency. The wire-protocol between the external client and UCP is subtly different from the wire-protocol between UCP and the resource provider.

We should specialize the ARM RPC context middleware for resource providers and regular resources. UCP can define the context in its own way using its own middlware. This will allow us to special-case the irregular APIs appropriately.

## Alternatives considered

The most promising alternative is of course to do nothing. This came up because I was working on adding resource provider discovery to the API (the last unimplemented irregular case). I realized we were working without a set of principles. 

The only downside I can think of with the proposed approach is inconsistency with ARM. I think that's OK in this case because UCP's "meta" functionality does not need to be compatible with ARM.

If we want to standardize on ARM's prior art, then we should document those principles instead.

## Test plan

TBD for now.

## Security

No additional security concerns. This is a change to URL structure. 

## Compatibility (optional)

The change to the resource ID format returned by scope-like resources is a breaking change. Clients can use the current format of the `id` to build resource IDs.

Any client that does this would need to update an consume the new API property.

## Monitoring

No new monitoring needed. This will improve our ability to monitor existing functionality through greater consistency.

## Development plan

TBD for now.

## Open issues

Basically everything about this proposal is an open issue :+1:. Will update when we discuss. 