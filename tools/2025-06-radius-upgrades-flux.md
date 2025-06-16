# Radius GitOps Helm Upgrades with Flux

* **Author**: Will Smith (@willdavsmith)
* **Status**: In Progress

## Overview

This design enables automated Radius upgrades in GitOps environments using Flux HelmRelease resources. The Radius Kubernetes controller is extended with a new reconciler that monitors Flux HelmRelease objects and validates upgrade compatibility before allowing Flux to proceed with Helm upgrades. It uses the existing preflight check system to ensure that upgrades are safe and compatible with the current cluster state.

## Terms and Definitions

| Term | Definition |
|------|------------|
| **GitOps** | A deployment methodology where infrastructure and applications are managed declaratively through Git repositories |
| **Flux** | A CNCF graduated GitOps tool that automatically syncs Kubernetes clusters with Git repositories |
| **HelmRelease** | A Flux custom resource that defines how Helm charts should be deployed |
| **Preflight Checks** | Validation tests run before upgrades to ensure compatibility and prevent problematic deployments |
| **Reconciler** | A Kubernetes controller component that continuously monitors and acts on resource changes |

## Objectives

### Goals

- **Enable Safe GitOps Upgrades**: Allow Radius upgrades to be managed through GitOps workflows while maintaining safety guarantees
- **Leverage Existing Preflight System**: Reuse the established preflight check infrastructure from the CLI for consistent validation
- **Provide Opt-in Control**: Users must explicitly enable the feature to avoid unexpected behavior
- **Maintain GitOps Principles**: Work within GitOps patterns without requiring manual intervention
- **Ensure Observability**: Provide clear feedback through Kubernetes events and annotations

### Non-Goals

- Supporting GitOps tools other than Flux (future consideration)
- Automatic rollback of failed upgrades (handled by Flux/Helm)
- Custom preflight checks specific to GitOps (use existing registry)
- Managing non-Radius Helm charts

## User Experience

### Target Users

**Platform Engineers** who:
- Manage Radius deployments in production environments
- Use GitOps workflows for infrastructure and application deployment
- Need automated but safe upgrade processes
- Require audit trails and observability for compliance

### User Scenarios

#### Scenario 1: Successful Automated Upgrade
```yaml
# Platform engineer updates Radius version in Git
apiVersion: helm.toolkit.fluxcd.io/v2
kind: HelmRelease
metadata:
  name: radius
  namespace: flux-system
  annotations:
    radapp.io/upgrade-enabled: "true"  # Opt-in to safety checks
spec:
  chart:
    spec:
      version: "0.47.0"  # Changed from 0.46.0
```

**Expected Flow:**
1. Flux detects the version change
2. Radius controller intercepts and runs preflight checks
3. Checks pass → HelmRelease proceeds normally
4. Kubernetes event: "PreflightSuccess: Preflight checks passed for version 0.47.0"
5. Flux deploys the new version

#### Scenario 2: Blocked Incompatible Upgrade
```yaml
# Platform engineer attempts to skip versions
spec:
  chart:
    spec:
      version: "0.48.0"  # Jumping from 0.46.0 (skipping 0.47.0)
```

**Expected Flow:**
1. Radius controller detects incompatible upgrade
2. HelmRelease is suspended (`spec.suspend: true`)
3. Kubernetes event: "PreflightFailed: Cannot upgrade from 0.46.0 to 0.48.0: version skipping not supported"
4. Platform engineer receives alert and fixes the version
5. Controller resumes HelmRelease when version is corrected

Once the version is corrected:
```yaml
spec:
   chart:
     spec:
       version: "0.47.0"  # Corrected to a compatible version
```

**Expected Flow:**
1. Radius controller resumes the HelmRelease
2. Preflight checks run again
3. Checks pass → HelmRelease proceeds normally
4. Kubernetes event: "PreflightSuccess: Preflight checks passed for version 0.47.0"
5. Flux deploys the new version

#### Scenario 3: Troubleshooting Failed Checks
```bash
# Platform engineer investigates blocked upgrade
kubectl describe helmrelease radius -n flux-system

# Annotations show:
# radapp.io/upgrade-hold: "Cannot upgrade from 0.46.0 to 0.48.0: version skipping not supported"
# radapp.io/upgrade-checked-version: "0.46.0"

# Events show:
# Warning  PreflightFailed  Preflight checks failed: Cannot upgrade from 0.46.0 to 0.48.0
```

## Design

### Architecture Overview

```
┌─────────────────┐    ┌───────────────────┐
│   Git Repo      │    │   Flux System     │
│                 │    │                   │
│ HelmRelease     │───▶│ source-controller │
│ version: 0.47.0 │    │ helm-controller   │
│ annotation:     │    │                   │
│   upgrade-      │    │                   │
│   enabled: true │    │                   │
└─────────────────┘    └───────────────────|
                                │        
                                │            
                                ▼              
                       ┌─────────────────┐
                       │ Flux            │
                       │ HelmRelease     │
                       │ Resource        │
                       └─────────────────┘
                                │
                                ▼
                       ┌─────────────────┐
                       │ Radius          │
                       │ FluxHelmRelease │
                       │ Reconciler      │
                       │                 │
                       │ • Version Check │
                       │ • Preflight Run │
                       │ • Suspend/Resume│
                       └─────────────────┘
```

### Component Design

#### FluxHelmReleaseReconciler

**Responsibilities:**
- Watch Flux HelmRelease resources with opt-in annotation
- Detect version changes requiring preflight validation
- Execute preflight checks using existing registry
- Control HelmRelease progression via suspend/resume
- Maintain state through annotations
- Generate observability events

**Key Methods:**
- `Reconcile()`: Main reconciliation loop
- `isRadiusChart()`: Validates opt-in annotation
- `runPreflightChecks()`: Executes validation using existing registry
- `holdHelmRelease()`: Suspends HelmRelease on preflight failures
- `clearHoldAndMarkComplete()`: Resumes HelmRelease on success

### State Management

The reconciler uses three key annotations to track state:

1. **`radapp.io/upgrade-enabled: "true"`** (User-controlled)
   - Opt-in mechanism for safety checks
   - Must be explicitly added to HelmRelease

2. **`radapp.io/upgrade-checked-version: "0.46.0"`** (Controller-managed)
   - Tracks last version that passed preflight checks
   - Prevents redundant check execution

3. **`radapp.io/upgrade-hold: "reason"`** (Controller-managed)
   - Indicates HelmRelease is suspended due to preflight failure
   - Contains human-readable failure reason

### Integration Points

#### Preflight Check System
- Reuses existing `pkg/upgrade/preflight` infrastructure
- Supports version compatibility checks
- Extensible for additional validation rules
- Consistent behavior with CLI upgrade command

#### Flux Integration
- Uses `spec.suspend` to control HelmRelease progression
- Leverages Flux's built-in reconciliation patterns
- Maintains compatibility with Flux GitOps workflows

#### Kubernetes Events
- Generates standard Kubernetes events for observability
- Integrates with existing monitoring and alerting systems
- Provides audit trail for compliance requirements

## API Design

### Annotations

```yaml
metadata:
  annotations:
    # User-controlled: Opt-in to upgrade safety checks
    radapp.io/upgrade-enabled: "true"
    
    # Controller-managed: Last checked version (avoids redundant checks)
    radapp.io/upgrade-checked-version: "0.46.0"
    
    # Controller-managed: Hold reason when upgrades are blocked
    radapp.io/upgrade-hold: "Cannot upgrade from 0.46.0 to 0.48.0: version skipping not supported"
```

### Events

```yaml
# Success Event
type: Normal
reason: PreflightSuccess
message: "Preflight checks passed for version 0.47.0"

# Failure Event
type: Warning
reason: PreflightFailed
message: "Preflight checks failed: Cannot upgrade from 0.46.0 to 0.48.0"
```

### RBAC Requirements

```yaml
apiGroups:
- helm.toolkit.fluxcd.io
resources:
- helmreleases
verbs:
- get
- list
- watch
- patch
- update
```

## Implementation Details

### Controller Setup

The reconciler is registered in the controller service with access to:
- Kubernetes client for HelmRelease operations
- Event recorder for observability
- Preflight registry for validation checks

```go
err = (&reconciler.FluxHelmReleaseReconciler{
    Client:            mgr.GetClient(),
    Scheme:            mgr.GetScheme(),
    EventRecorder:     mgr.GetEventRecorderFor("flux-helmrelease-controller"),
    PreflightRegistry: preflightRegistry,
}).SetupWithManager(mgr)
```

### Version Detection Logic

1. **Target Version**: Extracted from `spec.chart.spec.version`
2. **Current Version**: Retrieved from `status.history[0].chartVersion` (fallback to `status.lastAttemptedRevision`)
3. **Comparison**: Only run checks when versions differ and haven't been checked

### Preflight Check Execution

```go
func (r *FluxHelmReleaseReconciler) runPreflightChecks(ctx context.Context, currentVersion, targetVersion string) error {
    tempRegistry := preflight.NewRegistry(r.PreflightRegistry.GetOutput())
    
    if currentVersion != "" && targetVersion != "" && currentVersion != targetVersion {
        tempRegistry.AddCheck(preflight.NewVersionCompatibilityCheck(currentVersion, targetVersion))
    }
    
    _, err := tempRegistry.RunChecks(ctx)
    return err
}
```

### HelmRelease Control Mechanism

**Suspend on Failure:**
```go
spec["suspend"] = true
annotations[RadiusUpgradeHoldAnnotation] = reason
```

**Resume on Success:**
```go
delete(spec, "suspend")
delete(annotations, RadiusUpgradeHoldAnnotation)
annotations[RadiusUpgradeCheckedAnnotation] = version
```

## Test Plan

### Unit Tests

**Coverage Areas:**
- Chart detection and opt-in validation (`isRadiusChart`)
- Version extraction from HelmRelease specs (`getChartVersion`)
- Current version detection from status (`getCurrentDeployedVersion`)
- Preflight check execution (`runPreflightChecks`)
- HelmRelease suspension/resumption logic (`holdHelmRelease`, `clearHoldAndMarkComplete`)

**Test Scenarios:**
- Valid vs invalid annotation configurations
- Version comparison edge cases (missing versions, same versions)
- Preflight check success and failure paths
- Annotation and spec modification correctness

### Functional Tests

**Integration Scenarios:**
- End-to-end reconciliation with real HelmRelease objects
- Multiple preflight events for version updates
- Preflight failure handling and recovery
- Event generation and annotation management

**Test Data:**
- Sample HelmRelease YAML with Radius chart configuration
- Test cluster with Flux CRDs installed
- Mock preflight check implementations

### Manual Testing

**GitOps Workflow Validation:**
1. Deploy Radius via Flux HelmRelease with opt-in annotation
2. Update version in Git repository
3. Verify preflight checks execute before Flux deployment
4. Test failure scenarios with incompatible versions
5. Confirm proper event generation and annotation updates

## Compatibility

### Radius Versions

**Upgrade Path Support:**
- Leverages existing preflight check system
- Compatible with established version compatibility rules
- Supports incremental upgrade validation (no version skipping)

### Backwards Compatibility

**Opt-in Design:**
- Feature is disabled by default (requires annotation)
- Existing HelmRelease resources unaffected
- No breaking changes to current GitOps workflows

**Migration Strategy:**
- Users can gradually adopt by adding annotations to specific HelmReleases
- Rollback possible by removing opt-in annotation
- Coexistence with manual upgrade processes

## Monitoring and Logging

### Metrics

**Controller Metrics:**
- Reconciliation frequency and duration
- Preflight check success/failure rates
- HelmRelease suspension events

### Logging

**Log Levels:**
- INFO: Version changes detected, preflight results
- WARN: Preflight failures, suspension events
- ERROR: Controller errors, invalid configurations

**Structured Logging:**
```go
logger.Info("Running preflight checks for Radius upgrade",
    "currentVersion", currentVersion, 
    "targetVersion", chartVersion,
    "helmRelease", req.NamespacedName)
```

## Development Plan

### Phase 1: Core Implementation
- [ ] FluxHelmReleaseReconciler implementation
- [ ] Preflight check integration
- [ ] Basic annotation management
- [ ] Unit test coverage

### Phase 2: Integration & Testing
- [ ] Controller service registration
- [ ] RBAC configuration
- [ ] Functional test implementation
- [ ] End-to-end validation

### Phase 3: Documentation & Release
- [ ] User documentation
- [ ] Example configurations

## Open Questions

### Resolved

1. **Q: Should we support other GitOps tools besides Flux?**
   A: Start with Flux for focused implementation, evaluate others based on community demand.

2. **Q: How do we handle HelmRelease resources not managed by our controller?**
   A: Use opt-in annotation to avoid interference with existing workflows.

3. **Q: What happens if preflight checks take too long?**
   A: Use standard controller-runtime timeout mechanisms and requeue with backoff.

### Pending

1. **Q: Should we support custom preflight checks defined by users?**
   A: Future consideration - could be implemented via CRDs or configuration files.

2. **Q: How do we handle scenarios where Flux is temporarily unavailable?**
   A: Controller follows standard Kubernetes patterns - will reconcile when Flux returns.

3. **Q: Should we provide webhook validation for HelmRelease annotations?**
   A: Evaluate based on user feedback - current event-based feedback may be sufficient.

## Alternatives Considered

N/A

## Design Review Notes

*This section will be populated during the design review process.*