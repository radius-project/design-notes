# Topic: Multi-cluster support via kubeconfig contexts in Radius CLI

* **Author**: Nick Beenham (@superbeeny)

## Topic Summary
<!-- A paragraph or two to summarize the topic area. Just define it in summary form so we all know what it is. -->
Radius should let users target multiple Kubernetes clusters in a single operation using kubeconfig contexts. Instead of repeating install/upgrade/uninstall or validation commands per cluster, the CLI will fan-out the operation across multiple kubeconfig contexts, aggregate results, and present a concise summary. The approach is additive and backward compatible, reusing existing client creation (`kubeutil`) and Helm workflows.

### Top level goals
<!-- At the most basic level, what are we trying to accomplish? -->
- Enable multi-cluster operations from a single CLI invocation (install, upgrade, uninstall, status/validation).
- Use kubeconfig contexts as the selection mechanism; no new control plane dependencies.
- Keep changes minimally invasive and backward compatible.
- Provide clear per-cluster logging, bounded concurrency, and robust error aggregation.

### Non-goals (out of scope)
<!-- What are we explicitly not trying to accomplish? -->
- Cross-cluster scheduling, data-plane routing, or control-plane federation.
- Server-side multi-cluster reconciliation beyond optional client provider scaffolding.
- Automatic discovery/management of clusters outside kubeconfig.

## User profile and challenges
<!-- Define the primary user and the key problem / pain point we intend to address for that user. If there are multiple users or primary and secondary users, call them out.   -->

### User persona(s)
<!-- Who is the target user? Include size/org-structure/decision makers where applicable. -->
- Platform engineers and SREs managing many clusters (dev/stage/prod, multi-region).
- Application operators who need consistent Radius posture across several environments.

### Challenge(s) faced by the user
<!-- What challenges do the user face? Why are they experiencing pain and why do current offerings not meet their need? -->
- Repetitive one-cluster-at-a-time tasks lead to toil and mistakes.
- Drift across clusters due to version/config mismatches.
- Lack of a concise, aggregated view of success/failures across many contexts.

### Positive user outcome
<!-- What is the positive outcome for the user if we deliver this, i.e. what is the value proposition for the user? Remember, this is user-centric. -->
- One command to perform an operation across N clusters, with per-cluster results.
- Workspace can persist multiple contexts for repeatable execution.
- Faster rollouts with bounded concurrency and consistent logging.
- Clear per-cluster logging and error attribution.
- The ability to provide Radius as a managed and distributed service.

## Key scenarios
<!-- List ~3-7 high level scenarios to clarify the value and point to how we will decompose this big area into component capabilities. We may ultimately have more than one level of scenario. -->

### Scenario 1: Install Radius across multiple clusters
<!-- One or two sentence summary -->
Run `rad install kubernetes --kubecontext dev-west,dev-east` to install (or reinstall) the control plane into two clusters, in parallel, and view a summarized outcome per cluster.

### Scenario 2: Upgrade Radius across multiple clusters
<!-- One or two sentence summary -->
Run `rad upgrade kubernetes --kubecontext dev-west,dev-east --version latest` to fan-out preflight checks and upgrade operations across selected contexts.

### Scenario 3: Uninstall Radius across multiple clusters
<!-- One or two sentence summary -->
Run `rad uninstall kubernetes --kubecontext dev-west,dev-east [--purge]` to remove Radius and optionally purge the namespace across multiple clusters.

### Scenario 4: Use workspace-defined multi-contexts
Persist a workspace with multiple contexts and a default, then run `rad install kubernetes --kubecontext @workspace` to apply the operation to all workspace contexts.

### Scenario 5: Quick status
`rad cluster list` to show kubeconfig contexts and whether Radius is installed, with versions per cluster.

### Scenario 6: Multiple cluster application deployment
`rad deploy --kubecontext dev-west,dev-east` to deploy an application to multiple clusters.

## Key dependencies and risks
<!-- What dependencies must we take in order to enable this scenario? -->
<!-- What other risks are you aware of that need to be mitigated. If you have a mitigation in mind, summarize here. -->
<!-- **Dependency Name** – summary of dependency.  Issues/concerns/risks with this dependency -->
<!-- **Risk Name** – summary of risk.  Mitigation plan if known. If it is not yet known, no problem. -->
- **Helm per-context execution** – We rely on existing Helm workflows that already take a `kubeContext` (`pkg/cli/helm/cluster.go`). Risk: concurrency and registry limits; mitigate with bounded concurrency and retries where appropriate.
- **Kubeconfig loading/auth plugins** – We reuse `kubeutil.NewClientConfigFromLocal(...)`. Risk: missing contexts, auth plugin incompatibility; mitigate with early validation and clear per-cluster errors.
- **Namespace/permissions variance** – Preflight checks will fail fast and report insufficient permissions (`pkg/upgrade/preflight/kubernetes_check.go`).
- **API throttling** – Respect QPS/Burst defaults per cluster; allow tuning if needed via existing `kubeutil` options.

## Key assumptions to test and questions to answer
<!-- If you are making assumptions that, if incorrect, would cause us to significantly alter our approach to this scenario, make them explicit here.  Also call out how / when you plan to validate key assumptions. -->
<!-- What big questions must we answer in order to clarify our plan for this scenario.  When and how do you plan to answer those questions (prototype feature x, user research, competitive research, etc.) -->
- Assumption: most users rely on kubeconfig with multiple contexts, not bespoke discovery mechanisms.
- Assumption: per-cluster Helm operations are acceptable; we don’t need a new API surface.
- Question: do we need `--kubeconfig` flag surfaced broadly to point to a non-default file? Likely yes (optional in v1).
- Question: is aggregated non-zero exit desired if any cluster fails, or only when all fail? Default to non-zero if any fails.

## Current state
<!-- If we already have some ongoing investment in this area, summarize the current state and point to any relevant documents. -->
- Client creation via kubeconfig context:
  - `radius/pkg/cli/kubernetes/kubernetes.go` – `NewCLIClientConfig(context string)` uses `kubeutil.NewClientConfigFromLocal`.
  - `radius/pkg/components/kubernetesclient/kubernetesclientprovider/types.go` – wraps a single `*rest.Config` and exposes `ClientGoClient`, `DynamicClient`, controller-runtime `Client`.
- CLI commands accept a single `--kubecontext` today and act on one cluster:
  - Install: `radius/pkg/cli/cmd/install/kubernetes/kubernetes.go`
  - Upgrade: `radius/pkg/cli/cmd/upgrade/kubernetes/kubernetes.go`
  - Uninstall: `radius/pkg/cli/cmd/uninstall/kubernetes/kubernetes.go`
  - Helm operations are per-context in `radius/pkg/cli/helm/cluster.go`.
- Workspaces encode single Kubernetes connection:
  - `radius/pkg/cli/workspaces/connection.go` `KubernetesConnectionConfig{ Context string }`.
- Preflight checks for a single context:
  - `radius/pkg/upgrade/preflight/kubernetes_check.go`.

## Details of user problem
<!-- <Write this in first person...> -->
When I manage multiple clusters, I currently repeat the same command per kube context. It’s slow, error-prone, and easy to forget one environment. There’s no concise success/failure summary across clusters, so I need to collate logs manually. I want a simple way to select multiple contexts and apply the same Radius operation across them with consistent logging and a clear roll-up of results.

## Desired user experience outcome
<!-- <Write this as an “I statement”...>  -->
As an operator, I can select multiple kubeconfig contexts and run one `rad` command (install, upgrade, uninstall, status). I see per-cluster progress prefixed with the context, and a final summary table. If one or more clusters fail, the command exits non-zero and shows exactly which clusters failed and why. I can persist my cluster set in a workspace and reuse it with `@workspace`.

### Detailed user experience
 <!-- <List of steps the user goes through...>  -->
- Step 1: Configure kubeconfig with contexts `dev-west`, `dev-east`.
- Step 2: Optionally, define a workspace with multiple contexts.
- Step 3: Run `rad install kubernetes --kubecontext dev-west,dev-east`.
- Step 4: CLI performs preflight checks per cluster, installs Helm charts, and prints `[ctx=dev-west]`, `[ctx=dev-east]` logs.
- Step 5: CLI prints a summary (context, action, version, status, error if any) and returns non-zero if any cluster failed.
- Step 6: Use `rad upgrade kubernetes --kubecontext @workspace --version latest` for ongoing maintenance.

## Key investments
<!-- List the features required to enable this scenario(s). -->

### Feature 1: Multi-context CLI fan-out (install/upgrade/uninstall)
<!-- One or two sentence summary -->
- Parse comma-separated `--kubecontext` and iterate with bounded concurrency (e.g., 4). Aggregate errors and present a per-cluster summary.
- Reuse existing functions in `pkg/cli/helm/cluster.go` and runners in `pkg/cli/cmd/*/kubernetes/*.go`.
- Logging prefixes with context: `output.LogInfo("[ctx=%s] ...", kubeContext)`.
- Backward compatible when a single context or no context is provided.

Acceptance criteria:
- Single context behavior unchanged.
- Multiple contexts run in parallel; failures don’t block other clusters; overall exit non-zero if any fail.

### Feature 2: Workspace multi-context support (backward compatible)
<!-- One or two sentence summary -->
- Extend `KubernetesConnectionConfig` to support optional `contexts` (list) and `defaultContext` while keeping the existing `context` string.
- Add `Targets()` helper that returns 1 target if only `context` set, or N if `contexts` set. Support `--kubecontext @workspace`.

Acceptance criteria:
- Existing workspaces continue to work.
- New workspace schema can declare multiple contexts; CLI can resolve them.

### Feature 3: kubeutil multi-config helpers (optional v1)
<!-- One or two sentence summary -->
- Add `NewClientConfigsFromTargets(...)` to build `*rest.Config` per context with existing QPS/Burst defaults.

Acceptance criteria:
- Unit tests validate multiple configs created from multiple contexts and proper error attribution.

### Feature 4: MultiKubernetesClientProvider (internal scaffold)
<!-- One or two sentence summary -->
- Introduce `MultiKubernetesClientProvider` that wraps existing single-cluster provider for future server-side multi-cluster tasks.

Acceptance criteria:
- No immediate behavior change; improves internal ergonomics for potential future use.

### Feature 5: Preflight aggregator
<!-- One or two sentence summary -->
- Run existing preflight checks per context and aggregate results with severity levels, reusing `pkg/upgrade/preflight/kubernetes_check.go`.

Acceptance criteria:
- Clear per-cluster pass/fail output and a combined summary affecting exit code.

### Feature 6: `rad cluster list` UX (nice-to-have)
<!-- One or two sentence summary -->
- Enumerate kubeconfig contexts and show whether Radius is installed by calling `CheckRadiusInstall(kubeContext)`, plus chart version.

Acceptance criteria:
- Fast discovery with readable output; no changes required to install/upgrade paths.

### Feature 7: Observability and output polish
<!-- One or two sentence summary -->
- Prefix all logs with `[ctx=<context>]`. Optionally, set tracing attributes (cluster.context) in CLI spans.

Acceptance criteria:
- Users can confidently attribute logs to the right cluster; summary is compact and actionable.

