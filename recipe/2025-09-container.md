# Container Resource Recipe Migration

* **Author**: Brooke Hamilton (@brooke-hamilton)

## Overview

This design describes replacing the imperative Go renderer chain for Applications.Core/containers with a Bicep recipe for the new Radius.Compute/containers resource type. The current implementation uses a chain of wrapper renderers (`kubernetesmetadata.Renderer` → `manualscale.Renderer` → `daprextension.Renderer` → `container.Renderer`) that generates Kubernetes Deployments, Services, Secrets, RBAC resources, and Azure identity resources. This will be replaced with a single Bicep recipe that works with the new resource schema.

## Terms and definitions

- **Container Renderer Chain**: The composed renderers in `pkg/corerp/model/application_model.go` that process containers
- **Bicep Recipe**: Declarative template that provisions infrastructure based on input parameters
- **Extension Renderers**: Wrapper renderers that add functionality (Dapr sidecars, manual scaling, metadata)
- **Base Manifest**: User-provided Kubernetes YAML merged with generated resources via `kubeutil.ParseManifest`
- **LocalID**: Internal identifier for tracking output resource relationships (e.g., `rpv1.LocalIDDeployment`)

## Objectives

### Goals

- Replace the container renderer chain with a Bicep recipe for the new Radius.Compute/containers resource type
- Implement all current functionality for the new resource schema: multi-container deployments, volumes, identity, RBAC, connections
- Enable platform engineers to customize container deployment through recipe modifications
- Align with the new resource type schema defined in resource-types-contrib

### Non-goals

- Supporting the old Applications.Core/containers resource type
- Modifying the recipe engine or deployment processor  
- Supporting non-Kubernetes platforms in this phase
- Maintaining compatibility with existing deployed Applications.Core/containers resources

## User Experience

Users will define Radius.Compute/containers resources using the new schema:

```bicep
resource myContainer 'Radius.Compute/containers@2025-08-01-preview' = {
  name: 'myContainer'
  properties: {
    environment: environment
    application: application.id
    containers: {
      web: {
        image: 'nginx:latest'
        ports: {
          http: {
            containerPort: 80
          }
        }
      }
    }
  }
}
```

The recipe produces the same Kubernetes resources as the current renderer chain but works with the new resource type schema.

## Bicep Kubernetes Extension Configuration

**Extension Features:**
- **Preview Status**: The Kubernetes extension is currently in preview 
- **VS Code Integration**: "Import Kubernetes Manifest" command automatically converts YAML to Bicep
- **Resource Type Format**: Uses `{group}/{kind}@{version}` syntax (e.g., `apps/Deployment@v1`)
- **Namespace Management**: Handles namespace creation and validation automatically
- **kubeConfig Parameter**: Requires base64-encoded Kubernetes configuration from Radius environment
- **Type Safety**: Provides full IntelliSense and validation for Kubernetes resource properties

## Design

### Current Architecture

The current container deployment uses a chain of renderers registered in `pkg/corerp/model/application_model.go`:

```go
&kubernetesmetadata.Renderer{
  Inner: &manualscale.Renderer{
    Inner: &daprextension.Renderer{
      Inner: &container.Renderer{
        RoleAssignmentMap: roleAssignmentMap,
      },
    },
  },
}
```

Each renderer wraps the next, adding specific functionality:
- `container.Renderer`: Core Kubernetes resources (Deployment, Service, Secret, RBAC, Identity)
- `daprextension.Renderer`: Adds Dapr sidecar annotations
- `manualscale.Renderer`: Sets replica count
- `kubernetesmetadata.Renderer`: Adds custom labels/annotations

### Target Architecture

Replace the entire renderer chain with a single Bicep recipe that leverages the **Bicep Kubernetes extension (preview)** to directly create Kubernetes resources. This approach provides:

- **Direct Kubernetes Resource Creation**: Use the Bicep Kubernetes extension to create Kubernetes resources with native API schemas
- **Type Safety**: Full Bicep IntelliSense and validation for Kubernetes resource properties  
- **Declarative Approach**: Replace imperative Go code with declarative Bicep templates
- **Development Experience**: VS Code support with automatic YAML-to-Bicep conversion using "Import Kubernetes Manifest"

#### Bicep Kubernetes Extension Integration

The recipe will utilize the **Bicep Kubernetes extension (preview)** configuration:

```bicep
@description('Kubernetes configuration from Radius environment')
param kubeConfig string

extension kubernetes with {
  kubeconfig: kubeConfig
  namespace: 'radius-system'
} as k8s
```

**Key Extension Capabilities:**
- **Resource Type Format**: `{group}/{kind}@{version}` (e.g., `apps/Deployment@v1`, `core/Service@v1`)
- **Namespace Management**: Automatic namespace handling with client-side and server-side dry run support
- **Parameter Validation**: JSON schema validation for kubeconfig and resource properties
- **Resource Discovery**: Dynamic discovery of available Kubernetes API resources and versions

### Detailed Design

#### Current Container Renderer Analysis

The `container.Renderer.Render()` method in `pkg/corerp/renderers/container/render.go` creates resources for Applications.Core/containers. The new recipe must create the same resources but for the Radius.Compute/containers schema:

1. **Kubernetes Deployment** (`rpv1.LocalIDDeployment`)
   - Multi-container support via `properties.containers` map (matches new schema)
   - Health probes (readiness/liveness) via `makeHealthProbe()`
   - Environment variables from connections and direct values
   - Volume mounts for ephemeral and persistent volumes
   - Base manifest merging via `kubeutil.ParseManifest()`
   - Pod spec patching from `properties.runtimes.kubernetes.pod`

2. **Kubernetes Service** (`rpv1.LocalIDService`) 
   - Created only when containers expose ports
   - ClusterIP service with port mapping

3. **Kubernetes Secret** (`rpv1.LocalIDSecret`)
   - Connection environment variables
   - Secret data from `getEnvVarsAndSecretData()`

4. **Identity Resources** (when `identityRequired = true`)
   - Azure Managed Identity (`rpv1.LocalIDUserAssignedManagedIdentity`)
   - Federated Identity Credential (`rpv1.LocalIDFederatedIdentity`) 
   - Service Account (`rpv1.LocalIDServiceAccount`)

5. **RBAC Resources**
   - Kubernetes Role (`rpv1.LocalIDKubernetesRole`) - secrets get/list permissions
   - RoleBinding (`rpv1.LocalIDKubernetesRoleBinding`)

6. **Volume Resources**
   - SecretProviderClass for Azure Key Vault volumes
   - Role assignments for Key Vault access

#### Recipe Parameter Structure

Based on the new Radius.Compute/containers schema:

```bicep
param context object              // Resource metadata, environment settings  
param containers object           // Multi-container specification (required)
param connections object = {}     // Resource connections (optional)
param volumes object = {}         // Volume configurations (optional)
param restartPolicy string = 'Always'  // Container restart policy
param replicas int = 1            // Number of replicas (optional)
param autoScaling object = {}     // Auto-scaling configuration (optional)
param extensions object = {}      // Extensions like daprSidecar (optional)
param platformOptions object = {} // Platform-specific properties (optional)
```

#### Critical Implementation Challenges

1. **Multi-Container Complexity**: The current renderer handles multiple containers in a single Deployment via the `containers` object map, not a single container as the design originally assumed.

2. **Extension Renderer Chain**: The current architecture uses wrapper renderers for extensions. The recipe must replicate:
   - Dapr sidecar annotations (`dapr.io/enabled`, `dapr.io/app-id`, etc.)
   - Manual scaling replica settings
   - Kubernetes metadata (custom labels/annotations)

3. **Environment Variable Processing**: Complex logic in `getEnvVarsAndSecretData()`:
   - Connection-based environment variables with naming patterns
   - URL parsing for connection sources  
   - Type conversion and JSON marshaling for complex values
   - Secret reference handling

4. **Volume Dependencies**: Persistent volumes require:
   - Dependency resolution from `options.Dependencies`
   - Volume resource property extraction
   - SecretProviderClass creation for Azure Key Vault
   - Role assignment creation for volume access

5. **Identity and RBAC Logic**:
   - Conditional identity creation based on connections/volumes
   - Azure resource naming conventions (`azrenderer.MakeResourceName()`)
   - Role assignment scope determination
   - Service account workload identity configuration

6. **Base Manifest Merging**: Current implementation uses:
   - `kubeutil.ParseManifest()` to deserialize YAML
   - Object merging for Deployment, Service, ServiceAccount bases
   - Strategic merge patch for PodSpec (`patchPodSpec()`)

#### Recipe Resource Generation Using Kubernetes Extension

The recipe leverages the Bicep Kubernetes extension to generate resources with native Kubernetes API schemas:

**Extension Configuration:**
```bicep
@description('Container configuration from Radius resource')
param containerSpec object
@description('Kubernetes configuration')
param kubeConfig string
@description('Target namespace')
param targetNamespace string = 'default'

// Configure Kubernetes extension
extension kubernetes with {
  kubeconfig: kubeConfig
  namespace: targetNamespace
} as k8s

// Variables for extension functionality replication
var commonLabels = {
  'app.kubernetes.io/name': containerSpec.name
  'app.kubernetes.io/managed-by': 'radius'
  'radapp.io/application': containerSpec.application ?? containerSpec.name
}

// Dapr extension support
var daprEnabled = containerSpec.extensions?.dapr?.enabled ?? false
var daprLabels = daprEnabled ? {
  'dapr.io/enabled': 'true'
  'dapr.io/app-id': containerSpec.extensions.dapr.appId ?? containerSpec.name
} : {}

var daprAnnotations = daprEnabled ? {
  'dapr.io/enabled': 'true'
  'dapr.io/app-id': containerSpec.extensions.dapr.appId ?? containerSpec.name
  'dapr.io/app-port': string(containerSpec.extensions.dapr.appPort ?? 80)
  'dapr.io/config': containerSpec.extensions.dapr.config ?? 'default'
} : {}

// Final labels combining base, extension metadata, and Dapr
var finalLabels = union(
  commonLabels,
  containerSpec.extensions?.kubernetesMetadata?.labels ?? {},
  daprLabels
)

var finalAnnotations = union(
  containerSpec.extensions?.kubernetesMetadata?.annotations ?? {},
  daprAnnotations
)
```

**ServiceAccount with Azure Workload Identity:**
```bicep
resource serviceAccount 'core/ServiceAccount@v1' = {
  metadata: {
    name: containerSpec.name
    namespace: targetNamespace
    labels: finalLabels
    annotations: union(finalAnnotations, containerSpec.identity?.azure?.enabled ?? false ? {
      'azure.workload.identity/client-id': containerSpec.identity.azure.clientId
    } : {})
  }
}
```

**RBAC Resources:**
```bicep
resource role 'rbac.authorization.k8s.io/Role@v1' = {
  metadata: {
    name: containerSpec.name
    namespace: targetNamespace
    labels: finalLabels
  }
  rules: [
    {
      apiGroups: ['']
      resources: ['pods', 'services', 'configmaps', 'secrets']
      verbs: ['get', 'list', 'watch']
    }
  ]
}

resource roleBinding 'rbac.authorization.k8s.io/RoleBinding@v1' = {
  metadata: {
    name: containerSpec.name
    namespace: targetNamespace
    labels: finalLabels
  }
  subjects: [
    {
      kind: 'ServiceAccount'
      name: serviceAccount.metadata.name
      namespace: targetNamespace
    }
  ]
  roleRef: {
    kind: 'Role'
    name: role.metadata.name
    apiGroup: 'rbac.authorization.k8s.io'
  }
}
```

**Multi-Container Deployment with Extensions:**
```bicep
resource deployment 'apps/Deployment@v1' = {
  metadata: {
    name: containerSpec.name
    namespace: targetNamespace
    labels: finalLabels
    annotations: finalAnnotations
  }
  spec: {
    // Manual scaling extension support
    replicas: containerSpec.extensions?.manualScale?.replicas ?? 1
    selector: {
      matchLabels: commonLabels
    }
    template: {
      metadata: {
        labels: finalLabels
        annotations: finalAnnotations
      }
      spec: {
        serviceAccountName: serviceAccount.metadata.name
        containers: [for (container, i) in containerSpec.containers: {
          name: container.name ?? '${containerSpec.name}-${i}'
          image: container.image
          ports: [for port in (container.ports ?? []): {
            containerPort: port.containerPort
            name: port.name ?? 'port-${port.containerPort}'
            protocol: port.protocol ?? 'TCP'
          }]
          env: buildEnvironmentVariables(container, containerSpec.connections)
          resources: {
            limits: container.resources?.limits ?? {
              cpu: '500m'
              memory: '512Mi'
            }
            requests: container.resources?.requests ?? {
              cpu: '100m'
              memory: '128Mi'
            }
          }
          readinessProbe: container.readinessProbe
          livenessProbe: container.livenessProbe
          volumeMounts: [for mount in (container.volumeMounts ?? []): mount]
        }]
        volumes: buildVolumes(containerSpec.volumes)
      }
    }
  }
}
```

**Service for Network Access:**
```bicep
resource service 'core/Service@v1' = if (length(containerSpec.ports ?? []) > 0) {
  metadata: {
    name: containerSpec.name
    namespace: targetNamespace
    labels: finalLabels
  }
  spec: {
    selector: commonLabels
    ports: [for port in containerSpec.ports: {
      name: port.name ?? 'port-${port.port}'
      port: port.port
      targetPort: port.targetPort ?? port.port
      protocol: port.protocol ?? 'TCP'
    }]
    type: containerSpec.networking?.expose ?? false ? 'LoadBalancer' : 'ClusterIP'
  }
}
```

#### Resource Creation Logic

The recipe implements the same logic as the current Go renderer:

**Deployment Creation:**
```bicep
// Generate deployment with identical metadata and specification
resource deployment 'apps/Deployment@v1' = {
  metadata: {
    name: context.resource.name
    namespace: environment.namespace
    labels: {
      'app.kubernetes.io/name': context.resource.name
      'app.kubernetes.io/part-of': application.name
      'app.kubernetes.io/managed-by': 'radius'
    }
  }
  spec: {
    selector: {
      matchLabels: {
        app: application.name
        resource: context.resource.name
      }
    }
    template: {
      metadata: {
        labels: {
          app: application.name
          resource: context.resource.name
          'azure.workload.identity/use': identityRequired ? 'true' : null
        }
      }
      spec: {
        serviceAccountName: identityRequired ? context.resource.name : null
        enableServiceLinks: false
        containers: [
          {
            name: context.resource.name
            image: container.image
            ports: [for port in items(container.ports): {
              containerPort: port.value.containerPort
              protocol: 'TCP'
            }]
            env: concat(
              // Direct environment variables
              [for env in items(container.env): {
                name: env.key
                value: env.value.value
              } if env.value.value != null],
              // Secret-referenced environment variables  
              [for env in items(container.env): {
                name: env.key
                valueFrom: {
                  secretKeyRef: {
                    name: context.resource.name
                    key: env.key
                  }
                }
              } if env.value.valueFrom != null]
            )
            volumeMounts: [for volume in items(volumes): {
              name: volume.key
              mountPath: volume.value.mountPath
            }]
            readinessProbe: container.readinessProbe
            livenessProbe: container.livenessProbe
          }
        ]
        volumes: concat(
          // Ephemeral volumes
          [for volume in items(volumes): {
            name: volume.key
            emptyDir: {}
          } if volume.value.kind == 'ephemeral'],
          // Persistent volumes (Key Vault)
          [for volume in items(volumes): {
            name: volume.key
            csi: {
              driver: 'secrets-store.csi.k8s.io'
              volumeAttributes: {
                secretProviderClass: '${context.resource.name}-${volume.key}'
              }
            }
          } if volume.value.kind == 'persistent']
        )
      }
    }
  }
}
```

#### Extension Handling Complexity

Each extension requires specific processing that must be replicated in the recipe:

**Dapr Sidecar Extension** (`daprextension.Renderer`):
```bicep
// Annotations added to pod template
var daprAnnotations = contains(extensions, 'daprSidecar') ? {
  'dapr.io/enabled': 'true'
  'dapr.io/app-id': extensions.daprSidecar.?appId ?? context.resource.name
  'dapr.io/app-port': string(extensions.daprSidecar.?appPort ?? '')
  'dapr.io/config': extensions.daprSidecar.?config ?? ''
  'dapr.io/protocol': extensions.daprSidecar.?protocol ?? 'http'
} : {}
```

**Manual Scaling Extension** (`manualscale.Renderer`):
```bicep
// Replica count setting
var replicaCount = contains(extensions, 'manualScaling') 
  ? extensions.manualScaling.replicas 
  : (replicas ?? 1)
```

**Kubernetes Metadata Extension** (`kubernetesmetadata.Renderer`):
```bicep
// Custom labels and annotations
var customLabels = contains(extensions, 'kubernetesMetadata') 
  ? extensions.kubernetesMetadata.?labels ?? {} 
  : {}
var customAnnotations = contains(extensions, 'kubernetesMetadata')
  ? extensions.kubernetesMetadata.?annotations ?? {}
  : {}
```

#### Resource Provisioning

The new Radius.Compute/containers schema uses recipes by default, eliminating the need for manual vs internal provisioning modes. All resources are created through the recipe.

#### Base Manifest Integration

Complex merging logic from `manifest.go` must be replicated:

```bicep
// Parse base manifest (no direct Bicep equivalent to kubeutil.ParseManifest)
var baseManifest = runtimes.?kubernetes.?base ?? ''
var deploymentBase = parseJson(baseManifest) // Limited Bicep YAML parsing

// Merge with generated deployment
var finalDeployment = union(deploymentBase, generatedDeployment)

// Pod spec patching (no Bicep equivalent to strategicpatch.StrategicMergePatch)
var podPatch = runtimes.?kubernetes.?pod ?? ''
// This requires custom JSON merging logic in Bicep
```

#### Implementation Complexity Analysis

**Bicep Kubernetes Extension Advantages:**

1. **Native Kubernetes Resources**: Direct creation using official Kubernetes API schemas with full type safety
2. **VS Code Integration**: "Import Kubernetes Manifest" command enables easy YAML-to-Bicep conversion 
3. **Type Safety & IntelliSense**: Full Bicep validation and autocompletion for Kubernetes resource properties
4. **Namespace Management**: Automatic namespace handling with client-side and server-side dry run support
5. **Resource Discovery**: Dynamic discovery of available Kubernetes API resources and versions

**Manageable Complexity Areas:**

1. **Extension Functionality Replication**: 
   - Dapr: Use conditional annotations and labels in Bicep
   - Manual Scaling: Simple replica count parameter
   - Kubernetes Metadata: Direct label/annotation merging

2. **Multi-Container Support**: Native support through Bicep array iteration over containers

3. **Resource Dependencies**: Handled through Bicep resource references and dependency ordering

4. **Identity Integration**: Azure workload identity annotations on ServiceAccount resources

**Reduced Complexity with Extension:**

1. **Direct Resource Creation**: No need for complex YAML parsing - use native Kubernetes resource definitions
2. **Type Validation**: Bicep + Kubernetes extension provides compile-time validation 
3. **Declarative Approach**: Replace complex imperative Go logic with declarative Bicep templates
4. **Resource Relationships**: Natural Bicep dependency management vs manual renderer coordination

**Remaining Challenges:**

1. **Environment Variable Processing**: Complex type conversion logic still needs Bicep implementation
2. **Base Manifest Integration**: May need alternative approach for user-provided YAML merging
3. **Extension Preview Status**: Kubernetes extension is in preview and may have limitations

#### Risk Assessment

**Medium Risk**: The Bicep Kubernetes extension significantly reduces implementation complexity:
- **Reduced**: Direct Kubernetes resource creation vs complex Go renderer chains
- **Simplified**: Type-safe declarative templates vs imperative resource generation
- **Improved**: VS Code tooling support with IntelliSense and validation
- **Concern**: Extension preview status may introduce stability risks
- Limited Bicep capabilities for complex logic

### Resource Registration Changes

Current registration in `pkg/corerp/model/application_model.go` for Applications.Core/containers:

```go
{
  ResourceType: container.ResourceType, // "Applications.Core/containers"
  Renderer: &mux.Renderer{...}
}
```

New registration will target Radius.Compute/containers with recipe configuration:

```go
{
  ResourceType: "Radius.Compute/containers",
  Recipe: RecipeConfiguration{
    TemplateName: "containers",
    Parameters: containerParameterMapping,
  },
}
```

### Implementation Plan

#### Phase 1: Recipe Development

**Core Recipe Creation:**
1. Create `containers.bicep` recipe with multi-container support
2. Implement resource generation logic:
   - Kubernetes Deployment with container array
   - Conditional Service creation
   - Secret generation for connections
   - RBAC Role/RoleBinding creation

**Extension Integration:**
3. Replicate extension functionality in recipe:
   - Dapr sidecar annotations
   - Manual scaling replica count
   - Kubernetes metadata labels/annotations

**Identity and Volume Support:**
4. Implement conditional identity creation
5. Add volume mounting logic
6. Handle persistent volume dependencies

#### Phase 2: Advanced Features

**Base Manifest Support:**
1. Develop Bicep-based manifest merging (limited capabilities)
2. Implement pod spec patching approximation
3. Handle deployment/service/serviceaccount base objects

**Environment Variable Processing:**
4. Replicate connection-based environment variables
5. Implement type conversion logic for complex values
6. Add secret reference handling

#### Phase 3: Migration

**Registration Updates:**
1. Add recipe registration for Radius.Compute/containers  
2. Create parameter mapping from new schema to recipe parameters
3. Remove Applications.Core/containers renderer registration

**Cleanup:**
4. Applications.Core/containers renderer chain can remain for existing deployments
5. Update tests for new resource type
6. Update documentation for Radius.Compute/containers

### Technical Challenges and Gaps

#### Critical Missing Capabilities

1. **Dependency Access**: Recipe cannot access `options.Dependencies` map containing volume resource properties and computed values
2. **Schema Translation**: Complex mapping from Applications.Core/containers internal model to Radius.Compute/containers schema
3. **Extension Structure**: New schema uses different extension structure requiring significant recipe logic changes
4. **Complex Type Processing**: Limited ability to replicate Go's flexible type conversion in environment variable processing
5. **Volume Resolution**: No direct access to volume resource computed values needed for persistent volume setup

#### Workarounds Required

1. **Recipe Engine Enhancement**: Pre-process dependencies and schema mapping before recipe execution
2. **Parameter Expansion**: Flatten complex objects into recipe-compatible parameters
3. **Extension Mapping**: Convert between extension array model (current) and object model (new schema)
4. **Simplified Type Support**: Support only basic types in environment variables initially
5. **Monolithic Recipe**: Single recipe handling all functionality vs modular renderer composition

#### Development Effort Estimate

- **High Complexity**: 4-6 months for experienced developer
- **Risk Factors**: Schema mapping complexity, Bicep limitations, recipe engine enhancements
- **Testing Overhead**: Comprehensive testing with new resource type schema

## Test Plan

### Functional Implementation Tests

1. **Multi-Container Deployment**: Test containers map with multiple containers, shared volumes, different configurations using new schema
2. **Extension Functionality**: Verify Dapr sidecars, scaling, auto-scaling work with new extension structure
3. **Volume Integration**: Test ephemeral volumes, persistent volumes, volume mounts using new volume schema
4. **Identity and RBAC**: Verify Azure workload identity, service accounts, role bindings work with recipe
5. **Connection Processing**: Test environment variable generation from connections using new schema
6. **Platform Options**: Test platform-specific properties for Kubernetes

### New Schema Validation

1. **Schema Compliance**: Verify recipe works with all Radius.Compute/containers schema fields
2. **Auto-scaling**: Test new auto-scaling configuration (maxReplicas, metrics, targets)
3. **Resource Constraints**: Test CPU/memory requests and limits
4. **Extension Structure**: Test new extension object structure vs old array structure
5. **Error Scenarios**: Invalid configurations, missing required fields

### Recipe-Specific Testing

1. **Parameter Mapping**: Verify correct translation from resource properties to recipe parameters
2. **Resource Output**: Validate recipe produces expected Kubernetes resources
3. **Performance Impact**: Measure recipe execution time and resource usage

## Alternatives Considered

### Alternative 1: Traditional Bicep Templates (Rejected)

Use standard Bicep ARM templates to create Azure Container Instances or other Azure container services.

**Pros**: Well-established patterns, stable APIs
**Cons**: Doesn't support Kubernetes deployment model, breaks compatibility with current architecture

### Alternative 2: Helm Chart Integration (Rejected)

Generate Helm charts from Bicep recipes and deploy via Helm.

**Pros**: Leverages existing Helm ecosystem, maintains Kubernetes deployment
**Cons**: Adds complexity with dual template systems, loses Bicep type safety

### Alternative 3: Bicep Kubernetes Extension (Selected)

**Leverage the Bicep Kubernetes extension (preview) for direct Kubernetes resource creation.**

**Pros**: 
- Native Kubernetes resource creation with full type safety
- VS Code integration with YAML-to-Bicep conversion
- Declarative approach with IntelliSense and validation
- Maintains Kubernetes deployment model
- Future-aligned with Bicep extensibility roadmap

**Cons**: 
- Preview status introduces potential stability risks
- Requires experimental feature enablement
- Limited to Kubernetes-compatible clusters

### Alternative 4: Hybrid Approach (Rejected)

Keep core container rendering in Go, move only extensions to recipes.

**Pros**: Lower migration risk, preserves complex logic in Go
**Cons**: Doesn't achieve full recipe-based architecture, increases complexity

### Alternative 5: Gradual Migration (Considered)

Migrate one feature at a time (base containers → extensions → volumes → identity).

**Pros**: Reduced risk per iteration, easier testing
**Cons**: Complex interim states, longer overall timeline, multiple resource type versions

### Alternative 6: Enhanced Recipe Engine (Rejected)

Extend recipe engine with container-specific capabilities (dependency access, strategic merge).

**Pros**: Addresses traditional Bicep limitations, enables full functionality
**Cons**: Significant recipe engine changes, affects other resource types, engineering overhead

**Pros**: No migration risk, can validate functionality independently
**Cons**: Maintains two implementations, increases maintenance burden

## Recommendation

**Proceed with Bicep Kubernetes Extension - Reduced complexity with strong technical foundation.**

### Key Advantages of Kubernetes Extension Approach

1. **Reduced Technical Risk**: Bicep Kubernetes extension provides native Kubernetes resource creation with full type safety
2. **Improved Developer Experience**: VS Code integration with IntelliSense, validation, and automatic YAML-to-Bicep conversion
3. **Future-Aligned Architecture**: Leverages Microsoft's investment in Bicep extensibility for infrastructure-as-code
4. **Maintained Functionality**: All current container deployment capabilities preserved with cleaner implementation

### Technical Benefits

1. **Direct Resource Creation**: Eliminate complex Go renderer chains in favor of declarative Bicep templates
2. **Type Safety**: Compile-time validation and IntelliSense for Kubernetes resource properties  
3. **Simplified Extension Logic**: Replace wrapper renderer pattern with conditional Bicep logic
4. **Resource Dependency Management**: Native Bicep dependency resolution vs manual renderer coordination

### Recommended Implementation Strategy

1. **Enable Extension Support**: Configure bicepconfig.json with experimental extensibility features
2. **Iterative Development**: Start with core deployment, add extension functionality progressively
3. **VS Code Integration**: Leverage "Import Kubernetes Manifest" for existing YAML conversion
4. **Comprehensive Testing**: Validate extension preview stability and performance characteristics
5. **Documentation**: Create migration guides and best practices for Kubernetes extension usage

### Risk Mitigation

1. **Extension Preview Status**: Monitor extension stability and provide fallback options
2. **Performance Validation**: Benchmark recipe execution vs current Go renderer performance  
3. **Feature Parity**: Ensure all current functionality is preserved in Kubernetes extension implementation

## Open Questions

1. **Kubernetes Extension Stability**: How stable is the preview Kubernetes extension for production use?
2. **kubeConfig Management**: How will Kubernetes configuration be securely passed from Radius environment to recipes?
3. **Extension Capability Gaps**: Are there any current renderer capabilities that cannot be replicated with the Kubernetes extension?
4. **Performance Characteristics**: What is the performance impact of Kubernetes extension calls compared to direct Go renderer execution?
5. **Development & Debugging**: What tooling and debugging capabilities are available for Bicep recipes using the Kubernetes extension?
6. **Volume Integration**: How will Azure Key Vault and persistent volume integration work with the Kubernetes extension?
7. **Base Manifest Support**: Can user-provided YAML manifests be integrated with Kubernetes extension resources?

## Appendix: Technical Details

### Current Renderer Chain Analysis

**File**: `pkg/corerp/model/application_model.go:99-109`
```go
&kubernetesmetadata.Renderer{
  Inner: &manualscale.Renderer{
    Inner: &daprextension.Renderer{
      Inner: &container.Renderer{
        RoleAssignmentMap: roleAssignmentMap,
      },
    },
  },
}
```

**Extension Processing Order**: metadata → scaling → dapr → container (inner-to-outer execution)

### Resource Type Schema Differences

**Current**: Applications.Core/containers (internal Go model)
**Target**: Radius.Compute/containers (new public schema)

Key schema differences requiring recipe parameter mapping:
- Multi-container model: `properties.containers` (object map)
- Extension structure: `properties.extensions.daprSidecar` (object) vs old array model
- Volume configuration: `properties.volumes` with emptyDir/persistentVolume/secretId options
- Scaling options: `properties.replicas` and `properties.autoScaling` (direct properties)
- Resource constraints: `properties.containers[name].resources.requests/limits`
- Platform options: `properties.platformOptions` for Kubernetes-specific configuration

### Computed Values System

The current renderer uses computed values with transformers:
```go
computedValues[handlers.IdentityProperties] = rpv1.ComputedValueReference{
  Value: options.Environment.Identity,
  Transformer: func(r v1.DataModelInterface, cv map[string]any) error {
    // Update resource with computed identity info
  },
}
```

Recipes cannot replicate this post-processing capability.