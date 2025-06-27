# Radius Upgrades with GitOps

## Problem
Radius control plane upgrades require manual CLI commands (`rad upgrade kubernetes`, `rad rollback kubernetes`). These commands wrap `helm upgrade` and `helm rollback` with additional logic, such as preflight checks (and in the future, locks). This additional logic should ideally run however the user chooses to upgrade Radius, including `rad` CLI, `helm` CLI, or GitOps-based workflows such as `Flux` and `ArgoCD`. 

## Current Upgrade System
- `rad upgrade kubernetes` - CLI upgrade command with preflight checks
- `rad rollback kubernetes` - CLI rollback to previous/specific revision
- `pkg/upgrade/preflight/` - Validation framework (connectivity, versions, resources, etc.)

## Options

### Option 1: Helm Pre-Upgrade Job

**How**: Helm hook job runs preflight checks before upgrade starts.

**Context**: [Helm hooks](https://helm.sh/docs/topics/charts_hooks/) run at specific lifecycle points. Pre-upgrade hooks execute before chart resources are updated, and job failure blocks the entire upgrade.

 **Real-world example**: [Kong's Helm chart](https://github.com/Kong/charts/blob/main/charts/kong/templates/migrations-pre-upgrade.yaml) uses this pattern for database migrations before upgrades.

**Implementation**: 
- Create Job template with `helm.sh/hook: pre-install,pre-upgrade`
- Package existing `pkg/upgrade/preflight` code from `pkg/upgrade/preflight/registry.go`
- Job failure blocks entire upgrade

**Example Job Template**:
```yaml
apiVersion: batch/v1
kind: Job
metadata:
  annotations:
    "helm.sh/hook": pre-install,pre-upgrade
    "helm.sh/hook-weight": "-5"
    "helm.sh/hook-delete-policy": before-hook-creation,hook-succeeded
spec:
  template:
    spec:
      containers:
      - name: preflight
        image: ghcr.io/radius-project/preflight:{{ .Chart.AppVersion }}
        command: ["/preflight"]
        args: ["--target-version={{ .Chart.AppVersion }}"]
```

**Pros**:
- Reuses existing preflight code
- Native Helm integration - failure automatically blocks upgrade
- Works with all GitOps tools (Flux, ArgoCD, etc.) as well as Helm directly
- Runs once per upgrade (not per pod replica)

**Cons**:
- Helm-specific, possibility of Helm breaking changes in the future
- Have to build and integrate new container, `preflight`, into the Radius Helm chart
- Code duplication between `preflight` container and `rad` CLI

### Option 2: Init Container per Pod

**How**: Add init container to each control plane pod running preflight checks.

**Context**: [Init containers](https://kubernetes.io/docs/concepts/workloads/pods/init-containers/) run before app containers start. Each pod replica runs its own init container independently.

**Real-world example**: [MySQL Helm chart](https://github.com/helm/charts/blob/master/stable/mysql/templates/deployment.yaml) uses init containers for data directory cleanup before MySQL starts.

**Implementation**: Add to existing deployments in `/deploy/Chart/templates/*/deployment.yaml`:
```yaml
spec:
  template:
    spec:
      initContainers:
      - name: preflight
        image: ghcr.io/radius-project/preflight:{{ .Chart.AppVersion }}
        command: ["/preflight"]
```

**Pros**:
- Reuses existing preflight code
- Kubernetes-native, decouples from Helm

**Cons**:
- Runs redundantly on every pod replica
- Can't prevent partial upgrades

### Option 3: Upgrade Controller

**How**: Custom controller watches GitOps resources, runs preflight checks, and coordinates upgrades.

**Context**: Extends existing Radius controller (`pkg/controller/`) using [controller-runtime](https://pkg.go.dev/sigs.k8s.io/controller-runtime). Watches for GitOps resource changes and coordinates upgrade flow.

**Implementation**: 
```go
// Flux support - watch HelmRelease resources
func (r *FluxUpgradeReconciler) Reconcile(ctx context.Context, req reconcile.Request) {
    // Watch HelmRelease.spec.chart.version changes
    // Run preflight.Registry.RunAll() 
    // Update HelmRelease.status.conditions to block/allow upgrade
}

// ArgoCD support - watch Application resources  
func (r *ArgoUpgradeReconciler) Reconcile(ctx context.Context, req reconcile.Request) {
    // Watch Application.spec.source.targetRevision changes
    // Run preflight checks
    // Update Application.status.conditions
}
```

Add controller registrations in `cmd/controller/main.go`:
```go
// Requires separate reconcilers for each GitOps tool
ctrl.NewControllerManagedBy(mgr).
    For(&helmv2beta1.HelmRelease{}).   // Flux
    Complete(&FluxUpgradeReconciler{})

ctrl.NewControllerManagedBy(mgr).
    For(&argov1alpha1.Application{}). // ArgoCD  
    Complete(&ArgoUpgradeReconciler{})
```

**Pros**:
- Simplest to implement and maintain, taps into Flux and ArgoCD resources and doesn't require large Radius Helm chart changes

**Cons**:
- Requires separate reconcilers for Flux vs ArgoCD
- Possible race conditions - Flux HelmRelease controller and ArgoCD Application controller might start the upgrade before the Radius upgrade controller can stop it

## Open Questions

Q: Should we consider using the `rad` CLI directly in the `preflight` container?
