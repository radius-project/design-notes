# Title

* **Author**: Your name (@YourGitHubUserName)

## Overview

Teams consume Bicep recipes from OCI registries to provision shared infrastructure via Radius recipe packs. Today, registration stores a registry URL with a tag (e.g., v1.4.2 or latest) and the recipe type, and at deployment we pull and execute whatever artifact the registry currently serves for that tag. Because tags are mutable, an attacker or any accidental republish could retarget a tag to different contents without Radius detecting the change. This proposal introduces digest pinning and tag→digest verification so deployments are anchored to the exact artifact that was reviewed and registered, preventing silent drift and supply‑chain tampering.

## Terms and definitions

Recipe pack: Radius resource that groups recipes and metadata for provisioning.
Recipe digest: OCI content hash (e.g., sha256:…) uniquely identifying an image manifest.
Mutable tag: Registry tag that can be moved to point at a different digest (latest, semantic versions).
Dependency confusion: Pulling a similarly named artifact from an unintended origin. (Context in template.)

## Objectives
### Goals

- Allow recipe authors/platform teams to pin Bicep recipes to a specific digest inside recipe packs.
- Validate the digest at registration and deployment (fail closed on mismatch).
- Preserve ergonomics: tag remains visible for discoverability; digest is the source of truth.
- Maintain backward compatibility with existing packs, with a path to stricter policy by environment.

### Non goals
- No automatic patching of running apps if digests change; updates are explicit via recipe pack changes.
- No signing (Cosign/Notary) or registry allowlist in v1—those are future enhancements that can layer on top.


### User scenarios (optional)

#### User story 1
App references ghcr.io/org/recipes/secrets:latest. Registration resolves/stores digest sha256:X. Later, the tag moves to sha256:Y. Deployment fails with an actionable digest mismatch. 
#### User story 2
Security‑conscious team uses v1.4.2 with pinned digest. When upgrading to v1.4.3, they update tag and digest in a PR; Radius validates the new digest and proceeds.
## User Experience (if applicable)

- publish a bicep recipe
```diff
    rad bicep publish --file redis-recipe.bicep --target br:vishwaradius.azurecr.io/redis:1.0

    Building /Users/vishwa/Documents/RadiusProject/radius/test/testrecipes/test-bicep-recipes/corerp-redis-recipe.bicep...
    Pushed to test.azurecr.io:redis@sha256:c6a1…eabb
    Successfully published Bicep file "/Users/vishwa/Documents/RadiusProject/radius/test/testrecipes/test-bicep-recipes/corerp-redis-recipe.bicep" to "vishwaradius.azurecr.io/redis:1.0"
+   Copy the digest (sha256:c6a1…eabb) into your recipe pack to pin the artifact immutably.

```
- Registering a recipe pack
```diff
resource redisPack 'Radius.Core/recipePacks@2025-05-01-preview' = {
  name: 'redisPack'
  properties: {
    recipes: {
      'Radius.Cache/redis': {
        recipeKind: 'bicep'
        recipeLocation: 'vishwaradius.azurecr.io/redis:1.0' 
+        digest: 'sha256:c6a1…eabb'
      }
    }
  }
}
```


**Sample Input:**
<!--
Provide a sample CLI command input and/or bicep/helm code.
-->

**Sample Output:**
<!--
Provide a sample output for the inputs provided above.
-->

**Sample Recipe Contract:**
<!--
Provide a sample of updated recipe contract, if this proposal 
updates recipe contract (input parameters, outputs schema, etc)
-->

## Design

### High Level Design
High Level Design

Registration: When a recipe pack is applied, the controller resolves recipeLocation tag → digest via OCI manifest. If the spec already includes digest, the controller compares and rejects on mismatch; otherwise it persists the resolved digest in status.
Deployment: The Bicep engine reads the pack entry and pulls by stored digest. Before pull, it re‑resolves the tag to check for retargeting; if tag now points elsewhere, it fails early.
Updates: Any change of digest requires a recipe pack update/PR; controller re‑validates and persists the new digest.

### Architecture Diagram
<!--
Provide a diagram of the system architecture, illustrating how different
components interact with each other in the context of this proposal.

Include separate high level architecture diagram and component specific diagrams, wherever appropriate.
-->

### Detailed Design

<!--
This section should be detailed and thorough enough that another developer
could implement your design and provide enough detail to get a high confidence
estimate of the cost to implement the feature but isn’t as detailed as the 
code. Be sure to also consider testability in your design.

For each change, give each "change" in the proposal its own section and
describe it in enough detail that someone else could implement it. Cover
ALL of the important decisions like names. Your goal is to get an agreement
to proceed with coding and PRs.

If there are alternatives you are considering please include that in the open
questions section. If the product has a layered architecture, it's good to
align these sections with the product's layers. This will help readers use
their current understanding to understand your ideas.

Discuss the rationale behind architectural choices and alternative options 
considered during the design process.
-->

#### Advantages (of each option considered)
<!--
Describe what's good about this plan relative to other options. 
Provides better user experience? Does it feel easy to implement? 
Provides flexibility for future work?
-->

#### Disadvantages (of each option considered)
<!--
Describe what's not ideal about this plan. Does it lock us into a 
particular design for future changes or is it flexible if we were to 
pivot in the future. This is a good place to cover risks.
-->

#### Proposed Option
<!--
Describe the recommended option and provide reasoning behind it.
-->

### API design (if applicable)

<!--
Include if applicable – any design that changes our public REST API, CLI
arguments/commands, or Go APIs for shared components should provide this
section. Write N/A here if not applicable.
- Describe the REST APIs in detail for new resource types or updates to
  existing resource types. E.g. API Path and Sample request and response.
- Describe new commands in the CLI or changes to existing CLI commands.
- Describe the new or modified Go APIs for any shared components.
-->

### CLI Design (if applicable)
<!--
Include if applicable – any design that changes Radius CLI
arguments/commands. Write N/A here if not applicable.
- Describe new commands in the CLI or changes to existing CLI commands.
-->

### Implementation Details
<!--
High level description of updates to each component. Provide information on 
the specific sub-components that will be updated, for example, controller, processor, renderer,
recipe engine, driver, to name a few.
-->

#### UCP (if applicable)
#### Bicep (if applicable)
#### Deployment Engine (if applicable)
#### Core RP (if applicable)
#### Portable Resources / Recipes RP (if applicable)

### Error Handling
<!--
Describe the error scenarios that may occur and the corresponding recovery/error handling and user experience.
-->

## Test plan

<!--
Include the test plan to validate the features including the areas that
need functional tests.

Describe any functionality that will create new testing challenges:
- New dependencies
- External assets that tests need to access
- Features that do I/O or change OS state and are thus hard to unit test
-->

## Security

<!--
Describe any changes to the existing security model of Radius or security 
challenges of the features. For each challenge describe the security threat 
and its mitigation with this design. 

Examples include:
- Authentication 
- Storing secrets and credentials
- Using cryptography

If this feature has no new challenges or changes to the security model
then describe how the feature will use existing security features of Radius.
-->

## Compatibility (optional)

<!--
Describe potential compatibility issues with other components, such as
incompatibility with older CLIs, and include any breaking changes to
behaviors or APIs.
-->

## Monitoring and Logging

<!--
Include the list of instrumentation such as metric, log, and trace to 
diagnose this new feature. It also describes how to troubleshoot this feature
with the instrumentation. 
-->

## Development plan

<!--
Describe how you will deliver your features. This includes aligning work items
to features, scenarios, or requirements, defining what deliverable will be
checked in at each point in the product and estimating the cost of each work
item. Don’t forget to include the Unit Test and functional test in your
estimates.
-->

## Open Questions

<!--
Describe (Q&A format) the important unknowns or things you're not sure about. 
Use the discussion to answer these with experts after people digest the 
overall design.
-->

## Alternatives considered

<!--
Describe the alternative designs that were considered or should be considered.
Give a justification for why alternative approaches should be rejected if
possible. 
-->

## Design Review Notes

<!--
Update this section with the decisions made during the design review meeting. This should be updated before the design is merged.
-->