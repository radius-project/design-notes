# Title

* **Author**: Vishwanath Hiremath (@vishwahiremat)

## Overview

Radius recipes enable consistent, reusable provisioning of infrastructure by allowing applications to reference
externally stored infrastructure definitions, such as Bicep templates or Terraform modules, through recipe
packs. These recipes are sourced from external systems including registries and repositories and are later
executed as part of application deployment.

In the current model, recipe references typically rely on version identifiers that may be mutable, such as tags
or unpinned versions. If the underlying artifact changes without the reference changing, Radius has no built‑in
way to detect the modification. This can result in unintended infrastructure changes and exposes Radius
deployments to supply‑chain risks.

This proposal introduces a consistent approach to **recipe source integrity**, ensuring that recipes executed
during deployment are identical to those originally intended. By validating artifact identity at both
registration and deployment time, Radius can detect unexpected changes and prevent execution of modified
content.


## Terms and definitions

Recipe pack: Radius resource that groups recipes and metadata for provisioning.
Recipe digest: OCI content hash (e.g., sha256:…) uniquely identifying an image manifest.
Mutable tag: Registry tag that can be moved to point at a different digest (latest, semantic versions).
Dependency confusion: Pulling a similarly named artifact from an unintended origin. (Context in template.)

## Objectives
### Goals

- Allow recipe authors/platform teams to pin Bicep recipes to a specific digest inside recipe packs.
- Ensure that recipes executed during deployment are identical to the artifacts originally published and intended
- Prevent unexpected recipe drift by validating artifact identity at deployment time.
- Maintain backward compatibility for existing recipes that do not specify a digest.

### Non goals
- No automatic patching of running apps if digests change; updates are explicit via recipe pack changes.
- Terraform recipe integrity for http/s3 based module sources.

### User scenarios (optional)
#### User story 1
As a platform engineer, I want to pin a recipe to an immutable digest when I register it in a recipe pack, so that any deployment using that recipe executes exactly the artifact I reviewed and approved.

#### User story 2
As a user of Radius recipes, I want Radius to execute the same recipe artifact that I originally published and intended during deployment, so that hidden registry changes cannot cause unexpected infrastructure behavior.

## User Experience (if applicable)
This proposal introduces minor changes to the recipe publishing and registration experience to make recipe immutability explicit and easy to use. The overall workflow remains the same, with an additional step to surface and persist the recipe digest for bicep recipes.

#### Publishing a Bicep recipe

When publishing a Bicep recipe to an OCI registry, the CLI surfaces the resolved digest in the output. This makes the immutable identifier of the artifact immediately available for use during recipe pack registration.

```diff
    $ rad bicep publish --file redis-recipe.bicep --target br:vishwaradius.azurecr.io/redis:1.0

    Building redis-recipe.bicep...
    Pushed to test.azurecr.io:redis@sha256:c6a1…eabb
    Successfully published Bicep file "redis-recipe.bicep" to "test.acr.io/redis:1.0"
+   Copy the digest (sha256:c6a1…eabb) into your recipe pack to pin the artifact immutably.

```
#### Registering a recipe pack
When registering a recipe pack, users can optionally specify the digest of the recipe artifact. When provided, the digest becomes the authoritative identifier used for deployment.

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
**Sample Recipe Contract:**
recipepacks.tsp 
```diff
@doc("Recipe definition for a specific resource type")
model RecipeDefinition {
  @doc("The type of recipe (e.g., Terraform, Bicep)")
  recipeKind: RecipeKind;

  @doc("Connect to the location using HTTP (not HTTPS). This should be used when the location is known not to support HTTPS, for example in a locally hosted registry for Bicep recipes. Defaults to false (use HTTPS/TLS)")
  plainHttp?: boolean;

  @doc("URL path to the recipe")
  recipeLocation: string;

  @doc("Parameters to pass to the recipe")
  parameters?: Record<unknown>;

+  @doc("Immutable content digest (sha256:...) for Bicep recipes)
+  digest?: string;
}
```

## Design

### High Level Design
This design ensures the integrity of Bicep recipes by binding recipe execution to an immutable OCI digest instead of a mutable version tag.

The core idea is :
- A recipe is registered with both a tag and a digest
- The digest represents the exact recipe artifact that is allowed to execute
- During deployment, recipes are pulled by digest, not by tag

This guarantees that the infrastructure deployed by Radius is always derived from the same recipe artifact that was originally published and registered.and persists the new digest.

### Detailed Design

#### Recipe Registration: Digest Handling
To bind recipe execution to an immutable artifact, the recipe pack must record a digest that uniquely identifies the intended recipe contents. This section evaluates two possible approaches for how the digest is introduced during recipe registration.

##### Option 1: User‑Provided Digest at Registration Time
In this approach, the digest is explicitly supplied by the user when creating or updating a recipe pack. The digest is typically obtained from the output of the recipe publish command and added as part of the recipe definition.
Workflow
- The user publishes a Bicep recipe to an OCI registry.
- The publish command surfaces the resolved digest.
- The user includes this digest in the recipe pack definition during registration.
- Receipe is created with digest details.

Advantages:
- **Strong integrity guarantee**: Protects against scenarios where the recipe is modified in the registry between publish and registration.
- **Explicit user intent**: The user declares exactly which artifact is trusted and allowed to execute.
- **Simple implementation**: Leverages existing registry metadata resolution with minimal additional logic.
Clear trust boundary: Digest is treated as an explicit, trusted input rather than derived implicitly.

Disadvantages:
- **Manual step required**: Users must copy the digest from publish output into the recipe pack definition.

##### Option 2: Controller‑Computed Digest During Registration
In this approach, the recipe pack controller resolves and records the digest automatically during recipe registration. The user supplies only the recipe location and version, and the system derives the digest from the registry.

Workflow
- The user publishes a Bicep recipe to an OCI registry.
- The user registers a recipe pack using a tag‑based recipe location.
- During registration, the controller queries the registry for the current digest associated with the tag.
- The resolved digest is stored as part of the recipe definition.

Advantages
- **No user experience change**: Users are not required to handle digests explicitly.
- Simplifies recipe pack authoring.

Disadvantages

- **Weaker integrity guarantees**:If a recipe is modified—maliciously or accidentally—after publish but before recipe registration, Radius will silently register and trust the modified artifact.
- **Implicit trust in registry state**: The system assumes the registry contents at registration time are correct, which reintroduces supply‑chain risk.
- The absence of an explicit digest in the recipe definition obscures which artifact was intended at authoring time.

#### Proposed Option
Option 1 is recommended. While Option 2 offers a slightly smoother authoring experience, it can result in the recipe pack being bound to a different artifact than the one originally published. Option 1 requires the digest to be explicitly provided by the user, making the intended recipe artifact clear, stable, and auditable. The additional manual step is acceptable for infrastructure recipes and aligns with established digest‑pinning practices used for OCI artifacts.

#### Recipe Execution
During recipe execution, the recipe driver uses the digest information recorded in the recipe definition to ensure that the correct recipe artifact is retrieved and executed.
When a recipe includes a digest:
- The recipe driver retrieves the digest field from the recipe definition stored in the recipe pack.
- Instead of pulling the recipe using a version tag, the driver constructs a digest‑qualified OCI reference.
- The recipe artifact is fetched using ORAS with the digest reference, for example:
```
oras pull test.acr.io/redis@sha256:c6a1...eabb
```

Because OCI digests are immutable, this pull operation deterministically retrieves the exact recipe artifact that was originally published and registered.

### Integrity Enforcement for Terraform Recipes
This proposal focuses more on digest‑based integrity enforcement for Bicep recipes, where Radius directly pulls and executes recipe artifacts. Terraform recipes differ in execution model and include several native integrity mechanisms. This section outlines how similar integrity guarantees can be achieved for Terraform recipes and clarifies how the principles introduced in this design apply in that context.

Terraform is different as recipes are executed by invoking `terraform init` and `terraform apply`, and Terraform itself provides built‑in integrity mechanisms:
- Git sources can be pinned to immutable commit SHAs.
- Terraform Registry modules are versioned and protected by checksums recorded in .terraform.lock.hcl.
- Provider binaries are checksum‑verified during init.

However, these guarantees are post‑selection: Terraform validates integrity after a module source has been chosen. Terraform does not prevent a malicious or accidental change to a module before the first initialization if Radius passes a mutable or unpinned source to Terraform. 

Terraform recipes require source‑specific immutability rules that align with Terraform’s supported module sources. However, this can be achieved for most module sources, but plain HTTPS or S3 needs explicit versioning or checksums.

- **Git/Mercurial Based Modules** : Pin module sources using immutable commit SHAs (ref=<commit‑sha>)
- **Terraform Registry Modules** : Registry enforces immutability of published versions.
- **S3/HTTP URL based modules** : S3 or HTTP URLs are mutable and do not provide first‑download integrity guarantees.


### CLI Design (if applicable)
Adding digest details to `rad recipe-pack show` command.

```diff
$ rad recipe-pack show computeRecipePack
RECIPE PACK        GROUP
computeRecipePack  default

RECIPES:
Radius.Compute/containers
   Kind: bicep
   Location: test.acr.io/computepack/recipe:1.0
+  Digest: sha256:c6a1…eabb   
```


### Implementation Details
<!--
High level description of updates to each component. Provide information on 
the specific sub-components that will be updated, for example, controller, processor, renderer,
recipe engine, driver, to name a few.
-->

#### Core RP (if applicable)
#### Portable Resources / Recipes RP (if applicable)
Bicep Driver:
- Adding 


### Error Handling
- Digest not found (registration/deploy): `Recipe digest not found: <registry>/<repo>@sha256:<digest> (404 from registry)`
- Tag retargeted (deploy re-resolve): `Tag digest changed: <repo>:<tag> now points to sha256:<actual>; expected sha256:<expected>. Deployment aborted.`

## Test plan
**Unit**
- Resolve tag → digest (happy path; plainHttp=false/true).
- Mismatch detection: provided digest vs registry digest.
- Not-found digest.
- Deployment pull-by-digest path; fallback pull-by-tag when digest missing.
- TSP/schema: `digest?: string` serialization/deserialization.

**E2E / Functional **
- publish, register with digest, deploy; verify recipe pulled by digest.
- pack without digest
- Retargeted tag: detect and block at deploy.

## Security

This design strengthens the security of Radius recipe execution by introducing immutable artifact enforcement for Bicep recipes. It specifically addresses the risk of recipe tampering and tag‑based drift when recipes are sourced from external registries.
The primary threats addressed by this design are:
- Tag retargeting attacks
An attacker or accidental process modifies an existing OCI tag to point to different recipe contents after the recipe has been published or reviewed.

- Undetected recipe drift
Changes to recipe contents occur without any modification to recipe pack configuration, leading to unexpected infrastructure changes at deployment time.

## Development plan

**Phase 1: Schema & UX**
- Add `digest?: string` to recipe pack schema (Bicep only); update TSP and conversions.
- CLI: ensure `rad bicep publish` output highlights digest; add `rad recipe-pack show` to display stored digest.

**Phase 2: Bicep engine (deployment)**
- Pull recipes by stored digest (`repo@sha256:...`); tag used only for readability/re-resolve checks.
- Fall back to tag pull only when no digest exists (back-compat), and record resolved digest.

**Phase 3: Tests**
- Unit: resolve/compare logic, error paths (mismatch, not found, auth).
- Integration: publish → register with digest; register without digest (auto-resolve); deploy with retargeted tag (fail); deploy without digest (legacy).
- E2E/func: happy path and negative paths against a test ACR.

**Phase 4: Docs**
- Update authoring guide: copy digest from publish output into recipe pack.
- Add docs for terraform recipe integrity. 

## Open Questions
## Design Review Notes
