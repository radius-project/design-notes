# Redacting Sensitive Data for Radius.Security/secrets

* **Author**: Lakshmi Javadekar (@lakshmimsft)

## Overview

The `Radius.Security/secrets` resource type, defined in the [resource-types-contrib](https://github.com/radius-project/resource-types-contrib) repository, enables users to deploy sensitive data such as tokens, passwords, keys, and certificates to various secret backends (Azure Key Vault, HashiCorp Vault, Kubernetes secrets, etc.). This resource is handled by dynamic-rp, a type-agnostic processor providing CRUD operations for user-defined types.

Currently, dynamic-rp stores all resource properties, including sensitive `data` fields, in the Radius database. This design proposes a mechanism to redact (nullify) sensitive data after the recipe successfully deploys the secrets to their designated backend, ensuring that plaintext secrets do not persist in the Radius database long-term.

## Terms and definitions

| Term | Definition |
|------|------------|
| **Dynamic-RP** | Type-agnostic resource provider that handles CRUD operations for user-defined types without prior knowledge of their schemas |
| **Recipe** | Deployment template (Bicep/Terraform) that provisions infrastructure resources |
| **Recipe-based deployment** | Asynchronous deployment flow where recipes execute to create actual infrastructure |
| **Exposure window** | Time period during which sensitive data exists in plaintext in the database |
| **Applications.Core/secretStores** | Existing Radius resource type for managing Kubernetes secrets (imperative, K8s-only) |
| **Radius.Security/secrets** | New resource type for managing secrets across multiple backends (recipe-based, multi-backend) |

## Objectives

> **Issue Reference:** Implementation of sensitive data handling for Radius.Security/secrets resource type to prevent long-term storage of plaintext secrets in Radius database. ref: https://github.com/radius-project/radius/issues/10421

### Goals

1. **Enable recipe-based secret deployment**: Allow recipes to access sensitive data during execution to deploy secrets to various backends (Azure Key Vault, HashiCorp Vault, K8s, etc.)
2. **Prevent persistent storage**: Ensure sensitive data is nullified from Radius database after successful recipe deployment
3. **Minimal impact to dynamic-rp core**: Implement redaction as an extensible processor hook without requiring dynamic-rp to understand resource schemas
4. **Support multi-backend flexibility**: Enable the same implementation to work with any recipe backend

### Non goals

1. **Prevent all temporary storage**: The design accepts that sensitive data will temporarily exist in the database during recipe execution (seconds to minutes)
2. **Guaranteed cleanup in all failure scenarios**: Cannot guarantee cleanup if process crashes, database is unavailable, or operation context is cancelled before defer completes. Cleanup is best-effort.

### User scenarios (optional)

#### User story 1: Developer deploying secrets to Azure Key Vault

Alice, a platform engineer, wants to deploy application secrets to Azure Key Vault using Radius. She defines a `Radius.Security/secrets` resource with sensitive credentials and associates it with an Azure Key Vault recipe. The recipe deploys the secrets to Azure Key Vault, and after successful deployment, the sensitive data is removed from Radius database. Alice can still reference the secrets via output resources, but plaintext values are no longer stored in Radius.

#### User story 2: Multi-environment secret management

Bob manages multiple environments (dev, staging, prod) and needs different secret backends for each. He uses the same `Radius.Security/secrets` resource definition but associates different recipes: Kubernetes secrets for dev, HashiCorp Vault for staging, and Azure Key Vault for prod. The redaction mechanism works consistently across all backends without requiring backend-specific code.

## User Experience (if applicable)

**Sample Input:**

```bicep
resource appSecrets 'Radius.Security/secrets@2025-08-01-preview' = {
  name: 'app-secrets'
  properties: {
    environment: environment.id
    kind: 'generic'
    data: {
      databasePassword: {
        value: 'super-secret-password'
        encoding: 'string'
      }
      apiKey: {
        value: 'YXBpLWtleS1iYXNlNjQ='
        encoding: 'base64'
      }
    }
    recipe: {
      name: 'azure-keyvault'
      parameters: {
        vaultName: 'my-keyvault'
      }
    }
  }
}
```

**Sample Output:**

After deployment, the resource stored in Radius database:

```json
{
  "id": "/planes/radius/local/resourceGroups/my-rg/providers/Radius.Security/secrets/app-secrets",
  "type": "Radius.Security/secrets",
  "properties": {
    "environment": "/planes/radius/local/resourceGroups/my-rg/providers/Applications.Core/environments/myenv",
    "kind": "generic",
    "data": null,  // ← Sensitive data removed
    "recipe": {
      "name": "azure-keyvault",
      "parameters": {
        "vaultName": "my-keyvault"
      }
    },
    "status": {
      "outputResources": [
        {
          "id": "/subscriptions/sub-id/resourceGroups/rg/providers/Microsoft.KeyVault/vaults/my-keyvault"
        }
      ]
    }
  }
}
```

## Design

### High Level Design

The design introduces a redaction mechanism that operates in the backend async controller after recipe execution. The flow ensures that:

1. **Frontend (sync)**: Stores full resource including sensitive data in database, queues async operation
2. **Backend (async)**: Reads resource from database (with sensitive data), executes recipe using the data
3. **Redaction**: After successful recipe execution, nullifies sensitive data fields
4. **Final save**: Stores redacted resource back to database

Key components:
- **Recipe Controller** (`pkg/portableresources/backend/controller/createorupdateresource.go`): Orchestrates recipe execution and resource updates
- **Redaction Logic**: New functionality to detect `Radius.Security/secrets` type and nullify `properties.data`
- **Cleanup Handlers**: Defer blocks ensuring redaction even on failures

### Architecture Diagram

```mermaid
sequenceDiagram
    participant User
    participant Frontend as Frontend Controller<br/>(Sync)
    participant DB as Radius Database
    participant Queue as Async Queue
    participant Backend as Backend Controller<br/>(Async)
    participant Recipe as Recipe Engine
    participant SecretStore as Secret Backend<br/>(Azure KV/K8s/etc)

    Note over User,SecretStore: Phase 1: Frontend Processing (Synchronous)
    User->>Frontend: PUT /Radius.Security/secrets<br/>{data: {password: "secret123"}}
    Frontend->>Frontend: Validate request
    Frontend->>DB: Save resource<br/>(with secrets data)
    activate DB
    Note over DB: ⚠️ Exposure window starts<br/>Secrets in plaintext
    Frontend->>Queue: Queue async operation
    Frontend-->>User: 202 Accepted

    Note over User,SecretStore: Phase 2: Backend Processing (Asynchronous)
    Queue->>Backend: Process operation
    Backend->>DB: Read resource<br/>(with plaintext secrets)
    DB-->>Backend: Return full resource
    Backend->>Backend: Extract recipe config
    Backend->>Recipe: Execute recipe<br/>(with sensitive data)
    Recipe->>SecretStore: Deploy secrets<br/>(create KeyVault/K8s secret)
    SecretStore-->>Recipe: Success + Output Resources
    Recipe-->>Backend: Recipe output

    Note over Backend,DB: Phase 3: Redaction
    Backend->>Backend: Redact sensitive data<br/>properties.data = null
    Backend->>DB: Save resource<br/>(data: null, outputResources: [...])
    deactivate DB
    Note over DB: ✅ Exposure window ends<br/>Secrets removed
    Backend-->>User: Operation complete

    Note over User,SecretStore: Consumer accesses secrets via output resources
    User->>SecretStore: Access secrets via<br/>output resource reference
```

**Exposure Window:** T0 → T4

### Detailed Design

#### Option 1: Frontend Nullification (Rejected)

**Description**: Nullify sensitive data in the frontend controller before initial database save, similar to Applications.Core/secretStores.

**Reasons for rejection:
- **Cannot support recipes**: Recipe execution happens in backend async operation and needs access to sensitive data
- Breaks the recipe-based deployment model

#### Option 2: Backend Post-Recipe Redaction (Recommended)

**Description**: Store full resource in database temporarily, execute recipe using the data, redact after successful deployment.

**Advantages**:
- ✅ **Recipe compatible**: Recipe has full access to sensitive data during execution
- ✅ **Type-agnostic**: Dynamic-rp doesn't need to understand specific schemas
- ✅ **Multi-backend support**: Works with any recipe backend (Azure KV, HashiCorp, K8s)
- ✅ **Follows recipe patterns**: Aligns with how other recipe outputs are handled
- ✅ **Pragmatic**: Achieves security goal with minimal complexity
- ✅ **Testable**: Clear boundaries for testing redaction logic

**Disadvantages**:
- Temporary exposure window (seconds to minutes) during recipe execution
- Requires infrastructure-level encryption at rest
- Needs robust failure handling to ensure cleanup

#### Proposed Option

**Option 3: Backend Post-Recipe Redaction** is the recommended approach.

**Rationale**:
1. Preserves recipe-based deployment model which is core to Radius architecture
2. Enables multi-backend flexibility without type-specific coupling
3. Temporary exposure window is acceptable with proper safeguards:
   - Automatic cleanup on all exit paths (success/failure)
   - Infrastructure-level encryption at rest (etcd encryption, DB encryption)


### API design (if applicable)

N/A - No changes to public REST API, CLI, or Go APIs. This is an internal implementation detail of how dynamic-rp handles specific resource types.

### CLI Design (if applicable)

N/A - No CLI changes required.

### Implementation Details

#### Core RP (if applicable)

No changes required to Core RP. The implementation is isolated to portable resources and dynamic-rp.

#### Portable Resources / Recipes RP (if applicable)

**Primary Changes in Recipe Controller** (`pkg/portableresources/backend/controller/createorupdateresource.go`):

-Add redaction hook after recipe execution** :
-Add cleanup handler to ensure redaction on errors**:
-Type Detection Helper

### Error Handling

**Scenario 1: Recipe execution fails**
- Error: Recipe fails to deploy secrets to backend
- Handling: Defer block ensures `data` field is nullified even on failure
- User Experience: Recipe error is returned to user, but sensitive data is still cleaned up from database

**Scenario 2: Redaction fails during cleanup**
- Error: Nullification logic encounters unexpected error
- Handling: Log error but do not fail the overall operation since recipe already succeeded
- User Experience: Recipe succeeds, warning logged about redaction issue

**Scenario 3: Database save fails after redaction**
- Error: Final save operation fails
- Handling: Return error to user, operation will be retried
- User Experience: Operation shows as failed, will be retried by async controller

**Scenario 4: Resource type detection fails**
- Error: Unable to determine if resource requires redaction
- Handling: Skip redaction, log warning
- User Experience: No impact, operation proceeds normally

## Test plan

### Unit Tests

1. **Processor redaction logic**:
   - Test function that correctly nullifies `data` field
   - Test redaction is no-op for non-secret resource types
   - Test redaction handles missing `data` field gracefully
   - Test type detection logic correctly identifies `Radius.Security/secrets`

2. **Controller integration**:
   - Test redaction is called after successful recipe execution
   - Test defer block redacts on recipe failure
   - Test defer block redacts on unexpected errors
   - Test duration tracking and logging

### Integration/Functional Tests

1. **Recipe-based deployment**:
   - Deploy `Radius.Security/secrets` with K8s secret recipe
   - Verify secrets are created
   - Verify `data` field is null in Radius database after deployment
   - Verify output resources are correctly populated

2. **Failure scenarios**:
   - Trigger recipe execution failure
   - Verify `data` field is still nullified in database
   - Verify recipe error is properly surfaced to user

3. **Update scenarios**:
   - Update existing `Radius.Security/secrets` resource
   - Verify updated secrets are deployed and redactd

4. **End-to-end secret lifecycle**:
   - Create resource with sensitive data
   - Verify recipe deploys to backend
   - Verify data is redactd after deployment
   - Read resource back and verify `data` is null
   - Reference secrets via output resources in consuming application


## Security

### Security Model

The design introduces a **temporary exposure window** during which sensitive data exists in plaintext in the Radius database (from frontend save to backend redaction, typically seconds to minutes). This is a deliberate trade-off to support recipe-based deployment while maintaining type-agnostic dynamic-rp design.

### Security Threats and Mitigations

**Threat 1: Plaintext secrets in database backups**
- Risk: Database backup taken during exposure window contains plaintext secrets
- Mitigation:
  - Short exposure window (seconds) reduces probability
  - Document requirement for point-in-time backup strategy
  - Future enhancement: Exclude sensitive data from backups

**Threat 2: Unauthorized database access**
- Risk: Attacker with database read access can retrieve plaintext secrets
- Mitigation:
  - Existing RBAC controls on database access
  - Principle of least privilege for service accounts
  - Audit logging of database access

**Threat 3: Secrets in audit/query logs**
- Risk: Database query logs may capture sensitive data
- Mitigation:
  - Review audit log configuration to exclude data payloads
  - Encrypt audit logs at rest
  - Retention policies for log cleanup

**Threat 4: Recipe failure leaves secrets in database**
- Risk: Failed recipe execution leaves plaintext secrets indefinitely
- Mitigation:
  - Defer block ensures cleanup on all exit paths
  - Best-effort save in error handler
  - Future enhancement: Background cleanup job for orphaned secrets

**Threat 5: Etcd storage (Kubernetes)**
- Risk: Radius database backed by etcd may not have encryption at rest
- Mitigation:
  - **Required**: Enable etcd encryption at rest for Kubernetes deployments
  - Document encryption setup in installation guide
  - Add pre-flight checks to warn if encryption is disabled

### Infrastructure Requirements

Organizations deploying Radius with `Radius.Security/secrets` should ensure:

1. **Encryption at rest** (Required):
   - Enable etcd encryption for Kubernetes-based deployments
   - Enable native encryption at rest for future database backends

2. **Network encryption** (Required):
   - TLS for all database connections
   - Encrypted communication between Radius components

3. **Access controls** (Required):
   - RBAC on database restricting access to minimum required services
   - Service accounts with least privilege
   - Regular access audits

4. **Monitoring** (Recommended):
   - Alert on abnormal secret exposure durations
   - Dashboard tracking secret lifecycle metrics
   - Audit logging of secret operations

### Comparison to Existing Patterns

| Security Aspect | Applications.Core/secretStores | Radius.Security/secrets |
|-----------------|-------------------------------|------------------------|
| Plaintext DB storage | None | Temporary (seconds) |
| Encryption requirement | K8s etcd encryption | Radius DB encryption |
| Cleanup guarantee | Immediate | Deferred with fallback |
| Backend flexibility | Single (K8s only) | Multiple (recipe-based) |
| Type coupling | Imperative, K8s-specific | Generic, recipe-agnostic |

## Compatibility (optional)

**No breaking changes**. This is a new feature for a new resource type (`Radius.Security/secrets`).

**Forward compatibility**: The design allows future enhancements (field encryption, cache-based storage) without breaking changes to the API or user experience.

## Monitoring and Logging

### Metrics

1. **Secret exposure duration**:
   - Metric: `radius.security.secrets.exposure_duration_seconds`
   - Labels: `resource_id`, `recipe_name`, `result` (success/failure)
   - Purpose: Track how long sensitive data exists in database

2. **Redaction success rate**:
   - Metric: `radius.security.secrets.redaction_total`
   - Labels: `resource_id`, `status` (success/failure)
   - Purpose: Monitor redaction reliability

3. **Recipe execution with secrets**:
   - Metric: `radius.recipes.execution_total`
   - Labels: `resource_type`, `recipe_name`, `has_sensitive_data`
   - Purpose: Track recipe usage with sensitive data

### Logging

1. **Redaction events** (Info level):
   ```
   "Sanitizing sensitive data field"
   Fields: resourceID, resourceType, duration
   ```

2. **Redaction failures** (Error level):
   ```
   "Failed to redact sensitive data"
   Fields: resourceID, error
   ```

3. **Secret lifecycle completion** (Info level):
   ```
   "Secret data lifecycle completed"
   Fields: resourceID, duration, recipe_name
   ```

### Troubleshooting

**Issue**: Secrets not removed from database after deployment
- Check: Logs for redaction errors
- Check: Recipe execution succeeded
- Check: Async operation completed successfully
- Action: Review controller logs, check database state, verify processor implementation

**Issue**: Recipe execution fails with missing data
- Check: Ensure recipe is receiving full data before redaction
- Check: Verify redaction happens after recipe execution, not before
- Action: Review recipe parameters, check recipe logs

**Issue**: High secret exposure duration
- Check: Recipe execution duration metrics
- Check: Async operation queue depth
- Action: Investigate recipe performance, scale async workers

## Development plan

### Phase 1: Core Implementation (Sprint 1-2)
- **Week 1**: Implement redaction logic in processor
  - Create `SensitiveDataProcessor` interface
  - Implement `RedactSensitiveData()` in `DynamicProcessor`
  - Add type detection helper functions
  - Unit tests for redaction logic
  - **Estimate**: 3 days

- **Week 2**: Integrate with recipe controller
  - Add redaction hook in `CreateOrUpdateResource`
  - Implement defer block for cleanup on errors
  - Add duration tracking and logging
  - Integration tests with mock recipes
  - **Estimate**: 5 days

### Phase 2: Testing and Validation (Sprint 3)
- **Week 3-4**: Comprehensive testing
  - Integration tests with real recipes (Azure KV, K8s, HashiCorp)
  - Failure scenario testing
  - Performance testing for exposure duration
  - Security review
  - **Estimate**: 7 days

### Phase 3: Documentation and Hardening (Sprint 4)
- **Week 5**: Documentation and monitoring
  - Update user documentation
  - Security best practices guide
  - Add metrics and alerting
  - Dashboard for secret lifecycle tracking
  - **Estimate**: 3 days

### Future Enhancements (Post-MVP)
- **Phase 4**: Field-level encryption (Optional)
- **Phase 5**: Cache-based storage (Optional)
- **Phase 6**: Background cleanup job for orphaned secrets

## Open Questions

1. **Q: Should we enforce encryption at rest or just document it as required?**
   - A: Document as required for production deployments, add pre-flight checks to warn if disabled

2. **Q: What should happen if redaction fails after successful recipe execution?**
   - A: Log error but don't fail the operation; recipe already succeeded. Add monitoring alert for failed redaction.

3. **Q: Should we add a configuration option to disable redaction for debugging?**
   - A: Yes, add environment variable `RADIUS_DISABLE_SECRET_SANITIZATION=true` for development/debugging only

4. **Q: How do we handle secrets in database backups?**
   - A: Document backup strategy recommendations: point-in-time backups, exclude temporary data, encrypt backups at rest

5. **Q: Should we add support for multiple redaction strategies (nullify vs encrypt)?**
   - A: Start with nullify-only. Add interface for pluggable strategies in future if needed.

## Alternatives considered

### Alternative 1: Frontend Nullification (Rejected)
- Store secrets in cache/queue, nullify before database save
- Rejected: Breaks recipe compatibility, adds infrastructure complexity
- See Alternative 4 for detailed analysis of queue-based approaches

### Alternative 2: Resource-Specific Controller (Rejected)
- Create dedicated controller for `Radius.Security/secrets` instead of using dynamic-rp
- Rejected: Breaks type-agnostic model, doesn't scale to other sensitive types

### Alternative 3: Database Encryption at Rest (Recommended Enhancement)

**Description**: Enable encryption at the infrastructure/database layer to protect sensitive data during the temporary exposure window (T0-T4).

**Current Database Architecture:**
- **Provider abstraction**: `pkg/components/database/databaseprovider` supports pluggable backends
- **Current backend**: Kubernetes API Server (backed by etcd)
- **Future support**: Database will be configurable to support alternative backends
- **Current state**: No built-in encryption at application level
- **Database client interface**: Generic `Save/Get/Delete` operations, no encryption hooks

**Implementation Options:**

**Option A: Infrastructure-Level Encryption (Recommended)**
- **For current etcd backend**:
  - Kubernetes supports native etcd encryption via `EncryptionConfiguration`
  - Transparent to Radius application code
  - **Recommendation**: Document as required for production deployments
  - **Setup**: Kubernetes admin configures encryption provider (AES-CBC, AES-GCM, or KMS)
  - **Reference**: https://kubernetes.io/docs/tasks/administer-cluster/encrypt-data/
- **For future database backends**:
  - Rely on native encryption at rest features of each database
  - Document encryption requirements per backend type

**Advantages**:
- ✅ **No code changes**: Completely transparent to Radius application
- ✅ **Customer-controlled**: Infrastructure teams configure and manage encryption
- ✅ **Proven solutions**: Battle-tested, industry-standard implementations
- ✅ **Defense in depth**: Protects secrets during exposure window (T0-T4)
- ✅ **Protects backups**: Database backups are also encrypted
- ✅ **Separation of concerns**: Security handled at appropriate infrastructure layer
- ✅ **Works for all data**: Entire database encrypted, not just secrets
- ✅ **No performance impact to Radius**: Encryption handled transparently by database
- ✅ **Simpler than alternatives**: No special queue handling or application-level key management

**Disadvantages**:
- ❌ **Requires customer setup**: Not automatic, customers must configure
- ❌ **Infrastructure dependency**: Requires platform/infrastructure support
- ❌ **Validation difficulty**: Hard for Radius to verify encryption is enabled programmatically

**Recommendation**:
- ✅ **Strongly recommended** as the preferred enhancement for defense-in-depth
- Aligns with industry best practices and security recommendations (see Threat 5, line 548)
- Simpler than queue bypass (Alternative 6) with no architectural changes
- Maintains clean architecture - no special-case handling in Radius code
- Scales naturally as new database backends are added

**Implementation Plan**:
1. **Phase 1** (Immediate): Document encryption requirement in installation/security guides
2. **Phase 2** (Near-term): Add validation/warnings if encryption appears disabled
3. **Phase 3** (Future): Provide tooling to help customers enable encryption

---

### Alternative 4: Bypass Frontend Database Save (Not Recommended)

**Description**: Pass sensitive data directly from frontend to async queue without saving to database, backend reads from queue instead of database.

**Advantages**:
- ✅ Eliminates database exposure window entirely
- ✅ No plaintext secrets in database at any point

**Disadvantages**:
- ❌ **Queue becomes sensitive storage**: Queue now stores plaintext secrets temporarily
  - Queue backend (internal in-memory or distributed) needs same security as database
  - Queue messages need encryption at rest and in transit
  - Problem just moves from database to queue
- ❌ **Message size limits**: Queue messages have size limits
  - Large resources with many secrets might exceed limits
  - Current internal queue implementation may not handle large payloads
- ❌ **Queue durability concerns**:
  - In-memory queues: Lost on restart (need durable queue)
  - Distributed queues: Still persisted, same encryption concerns as database
- ❌ **Operational complexity**:
  - Harder to debug (data not in standard location)
  - Increases testing surface area
- ❌ **Race conditions**:
  - Frontend saves redacted resource to DB
  - If queue message lost, backend can't reconstruct full resource
  - Need fallback/retry mechanism
- ❌ **Breaking change**: Requires changes to core async operation infrastructure used by ALL resource types

**Current Queue Implementation:**
- **Location**: `pkg/components/queue`
- **Type**: Internal queue (configurable backend)
- **Message structure**: Designed for metadata only
- **Persistence**: Depends on configuration

**Recommendation**:
- **Not recommended**: Complexity and operational risk outweigh benefits
- **Key insight**: Queue is just as sensitive as database - moving secrets from DB to queue doesn't eliminate the security challenge, just moves it
- **Better alternatives**:
  - Infrastructure-level encryption (etcd encryption)
  - Application-level encryption with proper key management
- **If pursued in future**:
  - Must be part of broader architectural redesign
  - Requires queue encryption, message size handling, durability guarantees
  - Not isolated to secrets - affects all async operations

## Design Review Notes

_(To be updated after design review meeting)_

---

## References

- [Radius.Security/secrets type definition](https://github.com/radius-project/resource-types-contrib/blob/main/Security/secrets/secrets.yaml)
- [Applications.Core/secretStores implementation](https://github.com/radius-project/radius/blob/main/pkg/corerp/frontend/controller/secretstores/kubernetes.go)
- [Dynamic-RP architecture](https://github.com/radius-project/radius/tree/main/pkg/dynamicrp)
- [Recipe controller](https://github.com/radius-project/radius/blob/main/pkg/portableresources/backend/controller/createorupdateresource.go)
