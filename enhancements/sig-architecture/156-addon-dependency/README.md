# Addon Dependency Management

## Release Signoff Checklist

- [ ] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] Graduation criteria for dev preview, tech preview, GA
- [ ] User-facing documentation is created in [website](https://github.com/open-cluster-management/website/)

## Summary

This proposal introduces a dependency management mechanism for managed cluster addons, allowing addon authors to declare dependencies between addons. The addon-manager reports an addon whose dependencies are not installed or not available through its status conditions. It does not block or undo installation.

## Motivation

Currently, there is no way to declare or enforce dependencies between addons in Open Cluster Management. This creates several challenges:

1. **Manual coordination required**: Administrators must manually ensure that prerequisite addons are installed before installing dependent addons.

2. **Silent failures**: When a dependent addon is missing, addons may fail at runtime with unclear error messages, making troubleshooting difficult.

3. **Configuration complexity**: Some addons rely on Custom Resource Definitions (CRDs) or resources provided by other addons (e.g., Managed Service Account addon provides ManagedServiceAccount API that other addons use). Without dependency tracking, these relationships are implicit and undocumented.

4. **Installation ordering**: There's no automated way to ensure correct installation ordering when multiple interdependent addons are deployed simultaneously.

### Goals

- Provide a declarative way for addon authors to specify dependencies on other addons
- Automatically validate that all dependencies are satisfied before considering an addon healthy
- Provide clear status feedback when dependencies are not met
- Maintain backward compatibility with existing addons that have no dependencies

### Non-Goals

- Automatic installation of dependencies (users still need to explicitly install required addons)
- Dependency resolution and ordering during installation
- Support for circular dependencies
- Dependency management across different managed clusters (dependencies are validated per cluster)
- Version constraints for dependencies (addon API does not currently track version information)

## Proposal

We propose to extend the `ClusterManagementAddOn` API to include a `dependencies` field that allows addon authors to declare dependencies on other addons. The addon-manager component will validate these dependencies on each managed cluster and set appropriate status conditions on the `ManagedClusterAddOn` resource.

### User Stories

#### Story 1: Addon depending on Managed Service Account

As an addon author, I want to declare a dependency on the Managed Service Account addon because my addon uses the `ManagedServiceAccount` API to provision service accounts on managed clusters. When the Managed Service Account addon is not installed, I want users to see a clear error message in the addon status.

#### Story 2: Template addon depending on a CRD provided by another addon

As a user, I enable a virtualization addon that is shipped as an `AddOnTemplate` and whose
manifests include an `OperatorPolicy`. That CRD is installed by the Policy Framework
addons (`governance-policy-framework` and `config-policy-controller`).

If I enable the virtualization addon on a cluster that does not have the Policy Framework
addons, today the `ManagedClusterAddOn` reports `Available=True` and the only sign of the
real problem - `the server doesn't have a resource type OperatorPolicy` - is inside the
`ManifestWork` status. I want the dependency failure reported on the addon itself, naming
the prerequisites that are missing.

This story is what motivates
[Ownership of the Available condition](#ownership-of-the-available-condition): the addon in
question uses `HealthProberTypeWorkloadAvailability`, so its `Available` condition is set
on the hub rather than by the klusterlet-agent.

## Design Details

### API Changes

#### ClusterManagementAddOn

Add a `dependencies` field to `ClusterManagementAddOnSpec` in `addon/v1beta1`:

```go
// ClusterManagementAddOnSpec provides information for the add-on.
type ClusterManagementAddOnSpec struct {
	// ... existing fields ...

	// dependencies lists the add-ons that this add-on depends on. Dependencies affect
	// the status conditions of the add-on only; an unsatisfied dependency does not
	// block or undo its installation.
	// +optional
	// +listType=map
	// +listMapKey=name
	Dependencies []AddOnDependency `json:"dependencies,omitempty"`
}

// AddOnDependency is a dependency of an add-on on another add-on.
type AddOnDependency struct {
	// name is the name of the ClusterManagementAddOn this add-on depends on.
	// +required
	// +kubebuilder:validation:MinLength=1
	// +kubebuilder:validation:MaxLength=253
	Name string `json:"name"`

	// type is how the add-on is affected when this dependency is not satisfied.
	// Required, the default, marks the add-on unavailable; Optional marks it
	// degraded only.
	// +optional
	// +kubebuilder:default=Required
	Type AddOnDependencyType `json:"type,omitempty"`

	// message describes the impact of this dependency not being satisfied. It is
	// added to the message of the Degraded condition, for example
	// "Token-based access to managed clusters is unavailable".
	// +optional
	// +kubebuilder:validation:MaxLength=1024
	Message string `json:"message,omitempty"`
}

// AddOnDependencyType is how an add-on is affected when one of its dependencies is
// not satisfied.
// +kubebuilder:validation:Enum=Required;Optional
type AddOnDependencyType string

const (
	// AddOnDependencyTypeRequired means the add-on cannot function without the
	// dependency. An unsatisfied required dependency sets Degraded=True and
	// Available=False, both with reason RequiredDependencyNotSatisfied.
	AddOnDependencyTypeRequired AddOnDependencyType = "Required"

	// AddOnDependencyTypeOptional means the add-on works with reduced functionality
	// without the dependency. An unsatisfied optional dependency sets Degraded=True
	// with reason DependencyNotSatisfied and leaves Available to the health check.
	AddOnDependencyTypeOptional AddOnDependencyType = "Optional"
)
```

#### ManagedClusterAddOn Status

Add reasons for the existing `Degraded` and `Available` conditions, next to the existing
`AddonAvailableReason*` constants:

```go
// The reasons of condition ManagedClusterAddOnConditionDegraded.
const (
	// AddonDegradedReasonDependencyNotSatisfied is the reason of condition Degraded
	// when one or more optional dependencies are not satisfied and every required
	// dependency is. The add-on may still be available with reduced functionality.
	AddonDegradedReasonDependencyNotSatisfied = "DependencyNotSatisfied"

	// AddonDegradedReasonRequiredDependencyNotSatisfied is the reason of condition
	// Degraded when one or more required dependencies are not satisfied.
	AddonDegradedReasonRequiredDependencyNotSatisfied = "RequiredDependencyNotSatisfied"
)

// AddonAvailableReasonRequiredDependencyNotSatisfied is the reason of condition
// Available when it is False because one or more required dependencies are not
// satisfied.
const AddonAvailableReasonRequiredDependencyNotSatisfied = "RequiredDependencyNotSatisfied"
```

#### Storage Version and Conversion

The addon API is still stored as `v1alpha1`; `v1beta1` is served through the conversion
webhook in `pkg/addon/webhook`. A field added only to `v1beta1` would be dropped when the
object is stored, so `dependencies` would silently disappear.

The conversion webhook will round-trip it the same way it already handles
`installNamespace`, which exists only in `v1alpha1` and is kept in the
`addon.open-cluster-management.io/v1alpha1-install-namespace` annotation
([`InstallNamespaceAnnotation`](https://github.com/open-cluster-management-io/ocm/blob/0375f8745966ad0030d83dbd3e24226e621ad1f4/pkg/addon/webhook/v1beta1/managedclusteraddon_conversion.go#L19-L21)):

- `v1beta1` to `v1alpha1`: serialize `spec.dependencies` into an annotation, for example
  `addon.open-cluster-management.io/v1beta1-dependencies`.
- `v1alpha1` to `v1beta1`: restore `spec.dependencies` from that annotation and remove the
  annotation from the returned object.

This is no longer needed once the storage version moves to `v1beta1`.

#### Dependency Types

There are two types of dependencies:

- **Required dependencies (default, type=Required)**: The addon cannot function at all without the dependency. When a required dependency is not satisfied, the addon-manager will set `Degraded=True` with reason `RequiredDependencyNotSatisfied`, and the component that owns the `Available` condition will set `Available=False`. See [Ownership of the Available condition](#ownership-of-the-available-condition).

- **Optional dependencies (type=Optional)**: The addon can still function with reduced functionality when the dependency is missing. When an optional dependency is not satisfied, the addon-manager will set `Degraded=True` with reason `DependencyNotSatisfied`, but no component will modify the `Available` condition, allowing the addon to remain available if its health checks pass.

#### When a Dependency Is Satisfied

A dependency is **satisfied** only when all of the following hold on the dependent addon's
managed cluster namespace:

1. A `ManagedClusterAddOn` with the dependency's name exists.
2. Its `metadata.deletionTimestamp` is not set.
3. Its `Available` condition is `True`.
4. It does not have `Degraded=True` with reason `RequiredDependencyNotSatisfied`.

Condition 2 is required because an addon that is being uninstalled keeps `Available=True`
while its pre-delete hook runs. Without it, a dependent addon would keep reporting healthy
against a prerequisite that is on its way out. A prerequisite with a deletion timestamp is
treated as not satisfied even when its `Available` condition is still `True`.

Condition 4 carries an unsatisfied required dependency down a chain. If A requires B and B
requires C, a missing C must also mark A as not satisfied. `Available` alone is not enough
for this: with an older klusterlet-agent, a `HealthCheck.Mode=Lease` addon keeps
`Available=True` even when its required dependency is missing (see
[Version Skew Strategy](#version-skew-strategy)). The addon-manager sets this `Degraded`
reason itself on the hub, so the check works the same for every health check mode and
agent version. An optional dependency of the prerequisite (`DependencyNotSatisfied`) does
not affect its dependents.

A dependency returns to satisfied when, and only when, all four conditions hold again. At
that point the dependency-related `Degraded` condition is cleared and the dependent addon's
`Available` condition goes back to being driven by its own health check.

#### Status Matrix

| Prerequisite state | `type: Required` | `type: Optional` |
| --- | --- | --- |
| No `ManagedClusterAddOn` | `Available=False` / `RequiredDependencyNotSatisfied` (not for `HealthProberTypeNone`, see [Ownership of the Available condition](#ownership-of-the-available-condition)); `Degraded=True` / `RequiredDependencyNotSatisfied` | `Available` not modified; `Degraded=True` / `DependencyNotSatisfied` |
| Exists, `Available=False` or `Unknown` | same as above | same as above |
| Exists, `Available=True`, `deletionTimestamp` set | same as above | same as above |
| Exists, `Available=True`, `Degraded=True` / `RequiredDependencyNotSatisfied` | same as above | same as above |
| Exists, `Available=True`, not terminating, no unsatisfied required dependency | Dependency gate passes; `Available` is set by the addon's own health check | `Degraded` cleared for this dependency |

### Implementation Details

#### Dependency Validation

**Addon-Manager (on Hub):**

The addon-manager component will implement dependency validation with the following logic:

1. **Read dependencies**: When reconciling a `ManagedClusterAddOn`, read the corresponding `ClusterManagementAddOn` to get the list of dependencies.

2. **Check each dependency**: For each dependency in the list, apply the rule in
   [When a Dependency Is Satisfied](#when-a-dependency-is-satisfied):
   - Check if a `ManagedClusterAddOn` with the same name exists in the same namespace (managed cluster namespace)
   - Check that its `metadata.deletionTimestamp` is not set
   - Check if the prerequisite addon's `Available` condition is `True`
   - Check that the prerequisite addon does not have `Degraded=True` with reason `RequiredDependencyNotSatisfied`

3. **Set Degraded condition based on dependency type**:
   - **Precedence**: `Degraded` carries one reason. If any required dependency is not
     satisfied, the reason is `RequiredDependencyNotSatisfied`, even when optional
     dependencies are also not satisfied; the message lists all of them.
     `DependencyNotSatisfied` applies only when every unsatisfied dependency is optional, so
     when the last required dependency recovers while an optional one is still missing, the
     reason changes to `DependencyNotSatisfied`.

   - **If all dependencies are satisfied**:
     - Ensure the `Degraded` condition is not set with reason `DependencyNotSatisfied` or `RequiredDependencyNotSatisfied`
     - What `Degraded` holds after the dependency reason is cleared is an open question - see
       [Open Questions](#open-questions)

   - **If any optional dependency is not satisfied** (type=Optional):
     - Set the `Degraded` condition to `True` with:
       - Reason: `DependencyNotSatisfied`
       - Message: Clear description of which dependencies are missing, including the custom message if provided (e.g., "Optional addon 'managed-serviceaccount' is not installed or not available. Token-based access to managed clusters is unavailable.")
     - Do NOT modify the `Available` condition

   - **If any required dependency is not satisfied** (type=Required):
     - Set the `Degraded` condition to `True` with:
       - Reason: `RequiredDependencyNotSatisfied` (different reason!)
       - Message: Clear description of which dependencies are missing, including the custom message if provided (e.g., "Required addon 'managed-serviceaccount' is not installed or not available. This addon cannot function without ManagedServiceAccount API.")
     - Do NOT modify the `Available` condition here. The component that owns `Available` for the addon's health check mode acts on this reason - see [Ownership of the Available condition](#ownership-of-the-available-condition)

4. **Re-evaluate dependents when a prerequisite changes**: validation runs when the
   dependent addon is reconciled, so a change to the prerequisite has to enqueue it.
   Otherwise a dependent keeps a stale `RequiredDependencyNotSatisfied` after the
   prerequisite recovers, since the hub health check is skipped while that reason is set.
   - A create, update or delete of a `ManagedClusterAddOn` enqueues every `ManagedClusterAddOn`
     in the same namespace whose `ClusterManagementAddOn` lists it as a dependency.
   - An update to `ClusterManagementAddOn.spec.dependencies` enqueues every
     `ManagedClusterAddOn` of that addon.

#### Ownership of the Available Condition

The klusterlet-agent is not always the component that writes `Available`. Which component
owns it depends on the addon's health check mode, and both paths have to honour the
dependency state or the condition will flap between them.

| `HealthCheck.Mode` | `HealthProberType` | Component that writes `Available` |
| --- | --- | --- |
| `Lease` | `Lease` | klusterlet-agent, `pkg/registration/spoke/addon/lease_controller.go` |
| `Customized` | `Work`, `DeploymentAvailability`, `WorkloadAvailability` | addon-manager on the hub, via addon-framework `pkg/addonmanager/controllers/agentdeploy/healthcheck_sync.go` |
| `Customized` | `None` | the addon itself; the hub health syncer does not probe it |

This matters in practice because every `AddOnTemplate` addon is hard-wired to
`HealthProberTypeWorkloadAvailability`
(`pkg/addon/templateagent/template_agent.go`), so template addons fall entirely in the
second row. If only the klusterlet-agent were changed, a template addon with an
unsatisfied required dependency would keep reporting `Available=True`.

The hub path also overwrites the condition. `probeAddonStatusByWorks` ends by setting
`Available=True` whenever its probes pass, so an `Available=False` written by the agent
would be reverted on the next sync.

`HealthProberTypeNone` is excluded from the `Available` gate. The addon writes `Available`
itself, so no OCM component sets it to `False` for an unsatisfied required dependency. The
addon-manager still sets `Degraded=True` with reason `RequiredDependencyNotSatisfied`, and
the addon is expected to honour that reason when it writes `Available`. Dependents are not
affected: condition 4 of [When a Dependency Is Satisfied](#when-a-dependency-is-satisfied)
reads that `Degraded` reason, so a dependency chain through such an addon still reports
correctly.

**Klusterlet-Agent (on Managed Cluster), `HealthCheck.Mode=Lease`:**

The klusterlet-agent component will check the `Degraded` condition when determining addon availability:

1. **When reconciling addon health**: After checking lease health status
2. **Check for required dependency failures**:
   - If `Degraded=True` with reason `RequiredDependencyNotSatisfied`, set `Available=False` with reason `RequiredDependencyNotSatisfied`
   - If `Degraded=True` with reason `DependencyNotSatisfied` (optional dependency), do not modify `Available` - let the addon's own health checks determine availability

**Addon-Manager (on Hub), `HealthCheck.Mode=Customized`:**

The health check syncer gates the probe on the dependency state. This does not apply to
`HealthProberTypeNone`, which the syncer does not probe and where the addon writes
`Available` itself:

1. **Before probing**: If `Degraded=True` with reason `RequiredDependencyNotSatisfied`, set `Available=False` with reason `RequiredDependencyNotSatisfied` and return without running the probe, so the probe result cannot overwrite it.
2. **Optional dependencies**: If `Degraded=True` with reason `DependencyNotSatisfied`, run the probe as normal and let it decide `Available`.
3. **Ordering**: Dependency validation must run before the health check within the same reconcile. Otherwise the two writers alternate and the condition flaps.

This design keeps a single source of truth and avoids conflicts:
- **addon-manager** owns dependency validation and sets `Degraded` with the appropriate reason
- Whichever component owns `Available` for that health check mode reads that `Degraded` reason and does not re-derive the dependency state itself

**Key Design Benefits:**
1. No component needs access to the `ClusterManagementAddOn` API to interpret the result - all dependency type information is encoded in the `Degraded` condition reason
2. Condition ownership is explicit per health check mode, which prevents components from fighting over `Available`
3. Different dependency types (Optional vs Required) result in different status behaviors automatically

### Example Usage

#### Example 1: Optional Dependency

An addon that can work with reduced functionality without the dependency:

```yaml
apiVersion: addon.open-cluster-management.io/v1beta1
kind: ClusterManagementAddOn
metadata:
  name: my-addon
spec:
  addOnMeta:
    displayName: "My Addon"
    description: "An addon that optionally uses ManagedServiceAccount API"
  dependencies:
  - name: managed-serviceaccount
    type: Optional  # Must explicitly set to Optional (Required is the default)
    message: "Token-based access to managed clusters is unavailable"
  installStrategy:
    type: Manual
```

When the Managed Service Account addon is not installed, the `ManagedClusterAddOn` status would show:

```yaml
apiVersion: addon.open-cluster-management.io/v1beta1
kind: ManagedClusterAddOn
metadata:
  name: my-addon
  namespace: cluster1
status:
  conditions:
  - type: Available
    status: "True"  # Addon is still available with reduced functionality
    reason: AddonAvailable
    message: "Addon is available"
    lastTransitionTime: "2025-10-22T10:00:00Z"
  - type: Degraded
    status: "True"
    reason: DependencyNotSatisfied
    message: "Optional addon 'managed-serviceaccount' is not installed or not available. Token-based access to managed clusters is unavailable"
    lastTransitionTime: "2025-10-22T10:00:00Z"
```

#### Example 2: Required Dependency (Default)

An addon that cannot function without the dependency:

```yaml
apiVersion: addon.open-cluster-management.io/v1beta1
kind: ClusterManagementAddOn
metadata:
  name: my-critical-addon
spec:
  addOnMeta:
    displayName: "My Critical Addon"
    description: "An addon that requires ManagedServiceAccount API"
  dependencies:
  - name: managed-serviceaccount
    # type: Required is the default, can be omitted
    message: "This addon cannot function without ManagedServiceAccount API"
  installStrategy:
    type: Manual
```

When the Managed Service Account addon is not installed, the `ManagedClusterAddOn` status would show:

```yaml
apiVersion: addon.open-cluster-management.io/v1beta1
kind: ManagedClusterAddOn
metadata:
  name: my-critical-addon
  namespace: cluster1
status:
  conditions:
  - type: Available
    status: "False"  # Set by whichever component owns Available for this health check mode
    reason: RequiredDependencyNotSatisfied
    message: "Required addon 'managed-serviceaccount' is not installed or not available. This addon cannot function without ManagedServiceAccount API"
    lastTransitionTime: "2025-10-22T10:00:00Z"
  - type: Degraded
    status: "True"  # Set by addon-manager
    reason: RequiredDependencyNotSatisfied
    message: "Required addon 'managed-serviceaccount' is not installed or not available. This addon cannot function without ManagedServiceAccount API"
    lastTransitionTime: "2025-10-22T10:00:00Z"
```

### Risks and Mitigations

#### Risk: Circular Dependencies

**Risk**: Users might accidentally create circular dependencies (A depends on B, B depends on A).

**Mitigation**:
- Document that circular dependencies are not supported and may go undetected. If both
  addons are already available when the cycle is configured, both pass the satisfaction
  checks and neither is marked degraded.
- Consider adding validation webhook to detect and reject circular dependencies
- Future enhancement: Add a validation controller that detects circular dependencies

#### Risk: Dependency Chain Complexity

**Risk**: Long dependency chains might make troubleshooting difficult.

**Mitigation**:
- Provide clear error messages that list all unsatisfied dependencies
- An unsatisfied required dependency propagates down the chain through the
  `RequiredDependencyNotSatisfied` reason, so every addon above the break reports it
- Document best practices for keeping dependency chains shallow
- Consider adding status field to show the full dependency tree

### Test Plan

#### Unit Tests

- Test dependency validation logic with various scenarios:
  - No dependencies
  - Single dependency (satisfied/unsatisfied)
  - Multiple dependencies (all satisfied, some unsatisfied, none satisfied)

- Test the satisfied/unsatisfied rule itself:
  - Dependency `ManagedClusterAddOn` missing
  - Dependency present with `Available=False` and with `Available=Unknown`
  - Dependency present, `Available=True`, `deletionTimestamp` set - must be unsatisfied
  - Dependency present, `Available=True`, `Degraded=True` with
    `RequiredDependencyNotSatisfied` - must be unsatisfied
  - Dependency present, `Available=True`, `Degraded=True` with `DependencyNotSatisfied` -
    satisfied
  - Dependency present, `Available=True`, not terminating - satisfied

- Test reason precedence: a required and an optional dependency both unsatisfied gives
  `RequiredDependencyNotSatisfied`; recovering only the required one gives
  `DependencyNotSatisfied`.

- Test that a change to a prerequisite `ManagedClusterAddOn` enqueues its dependents.

#### Integration Tests

- Test complete workflow:
  1. Install addon A with dependency on addon B (B not installed) - verify Degraded condition
  2. Install addon B - verify addon A becomes Available
  3. Delete addon B - verify addon A becomes Degraded

- Run the workflow above once per health check mode, since `Available` is written by a
  different component in each:
  - `HealthCheck.Mode=Lease` (klusterlet-agent)
  - `HealthCheck.Mode=Customized`, including an `AddOnTemplate` addon
    (`WorkloadAvailability`, hub) - verify the probe does not overwrite `Available=False`

- Verify the gate holds for both `installStrategy: Manual` and `installStrategy: Placements`,
  since with `Placements` the addon-manager creates the `ManagedClusterAddOn` itself.

- Verify recovery: the dependency `Degraded` condition stays set until the prerequisite is
  present, non-terminating, `Available=True` and not `Degraded=True` with reason
  `RequiredDependencyNotSatisfied`; once all four hold, it is cleared and `Available`
  returns to the value its own health check produces.

#### E2E Tests

- Deploy real addons with dependencies on a test cluster
- Verify status conditions are correctly set
- Verify addon behavior when dependencies are not met

### Graduation Criteria

#### Beta (v1beta1)

- API changes implemented in `open-cluster-management.io/api` addon v1beta1
- Conversion webhook round-trips `dependencies` while `v1alpha1` is the storage version, with a test that a `v1beta1` create and read returns the field
- Dependency validation fully supported by addon-manager component
- Both `Available` owners updated to handle the `RequiredDependencyNotSatisfied` reason: the klusterlet-agent for `HealthCheck.Mode=Lease`, and the hub health check syncer for `HealthCheck.Mode=Customized`
- Unit and integration tests passing
- E2E tests with real addons
- At least 2 real addons using dependency declarations
- Metrics for dependency validation failures
- Documentation of the feature and API

#### GA (v1)

- Proven stability over at least 2 releases in v1beta1
- Comprehensive documentation including troubleshooting guides
- No critical bugs reported
- Widely adopted by addon authors (at least 5 addons using dependencies in production)
- Performance validated at scale (tested with 1000+ clusters)

### Upgrade / Downgrade Strategy

#### Upgrade

- **From version without dependency support to version with dependency support**:
  - Existing addons without dependencies continue to work unchanged
  - New or updated addons can add dependencies field
  - No migration required

#### Downgrade

- **From version with dependency support to version without dependency support**:
  - Dependencies field will be ignored by older controllers
  - Addons will function as if they have no dependencies
  - Status conditions related to dependencies will not be updated
  - A newer klusterlet-agent still sets `Available=False` for a `Lease` addon while a stale
    `RequiredDependencyNotSatisfied` reason is present. Before downgrading the hub, remove
    `spec.dependencies` from every `ClusterManagementAddOn` and wait until no
    `ManagedClusterAddOn` has a dependency `Degraded` reason. The current addon-manager
    clears only those reasons and leaves other status unchanged.

### Version Skew Strategy

- Hub and managed cluster components need to be aware of version compatibility
- Dependency validation happens on the hub side in the addon-manager component
- For `HealthCheck.Mode=Customized` both the dependency check and the `Available` gate run
  in the addon-manager on the hub, so agent version does not matter. `HealthProberTypeNone`
  has no `Available` gate (see
  [Ownership of the Available condition](#ownership-of-the-available-condition)).
- For `HealthCheck.Mode=Lease` the gate runs in the klusterlet-agent. A newer hub paired
  with an older agent, during an upgrade or after a managed cluster downgrade, still sets
  `Degraded=True` with `RequiredDependencyNotSatisfied`, but the agent keeps `Available`
  driven by the lease alone. That is today's behaviour plus a correct `Degraded`
  condition, so it is not a regression and needs no rollout gate. `Available` follows the
  dependency state once the agent is upgraded to the release that ships this gate.
  Dependents of that addon are not affected by the skew: they check its `Degraded` reason,
  which the hub sets, so they still see the dependency as not satisfied.

## Resolved Review Decisions

Settled on [#157](https://github.com/open-cluster-management-io/enhancements/pull/157),
recorded here so they are not relitigated.

- **An unsatisfied dependency does not block or undo installation.** The question was
  raised both ways: whether the addon should not be deployed until its dependencies are
  met, and whether an already-installed addon should be uninstalled when a dependency
  becomes unavailable. The answer to both is no - report a status condition and leave the
  addon in place. Disabling installation was considered and dropped as unnecessary.
- **`type` is an enum, not a boolean.** An earlier draft used a boolean `required` field.
  Booleans are avoided in OCM APIs because they cannot be extended.
- **The default is `Required`, not `Optional`.** Reviewers did not read an unmarked
  dependency as optional.
- **No implicit ordering between multiple dependencies.** Listing two dependencies does not
  order them against each other. Ordering is expressed by chaining dependent addons.
- **No version constraints.** `ClusterManagementAddOn` carries no addon version, so a
  dependency cannot be expressed against one.
- **Per-cluster-type dependencies are deferred.** An addon may depend on different addons
  depending on the kind of managed cluster - for example on OpenShift but not on EKS. There
  is no requirement for it yet, and the API can be extended later with a field such as
  `when` holding a CEL expression, without breaking the shape proposed here. See
  [addon-framework#341](https://github.com/open-cluster-management-io/addon-framework/issues/341)
  for the equivalent case in health checking.

## Open Questions

These are raised but not settled, and are not implied by the text above.

1. **Is `Degraded` the right condition for an unsatisfied required dependency?** It reads
   oddly - at that point the addon has failed rather than degraded. `ManagedClusterAddOn`
   has no `Failed` condition type today (`Available`, `Degraded`, `Configured`,
   `Progressing`), so the choice is between `Available=False` plus `Degraded=True` as
   written above, or `Available=False` alone with the detail carried in the reason and
   message. Keeping `Degraded=True` preserves a machine-readable reason for consumers that
   do not parse messages.

2. **Should a missing optional dependency set `Degraded` at all?** If a dependency is
   genuinely optional, the addon may simply be missing a nice-to-have rather than running
   degraded. One option is to leave it to the addon author, for example only setting
   `Degraded` when `message` is specified.

3. **Interaction with install ordering.** This proposal deliberately stops at declaration
   and status; prerequisites are not installed automatically and no install ordering is
   implied. [Issue #184](https://github.com/open-cluster-management-io/enhancements/issues/184)
   asks for ordering and auto-install. Keeping them separate lets declaration and status
   land first, but the two designs need to stay compatible.

4. **What does `Degraded` hold after a dependency recovers?** `Degraded` carries one reason,
   so a dependency reason can replace one set by another writer, such as the addon agent.
   When the dependency recovers, either the addon-manager keeps the displaced reason and
   message and restores it, or it clears only its own reason and the other writer re-asserts
   its reason on its next sync. This needs input from the addon agent and addon-manager
   owners.

## Implementation History

- 2025-10-22: Initial KEP draft, proposed in
  [#157](https://github.com/open-cluster-management-io/enhancements/pull/157) by @elgnay
- 2026-11-13: Defined when a dependency is satisfied (including terminating prerequisites),
  documented `Available` ownership per health check mode, added the status matrix, the
  template addon user story, the kro alternative, and the resolved and open questions.
  Reopened as a fresh PR after the original author left the project.
- TBD: Beta implementation
- TBD: GA promotion

## Drawbacks

- Adds complexity to the addon API
- Requires addon authors to maintain accurate dependency information
- Does not automatically install dependencies (users must still manually install them)

## Alternatives

### Alternative 1: Only Set Degraded Condition

Instead of adding a `type` field, the addon-manager could only set the `Degraded` condition when dependencies are not satisfied, and never modify the `Available` condition.

**Pros**:
- Simpler implementation - no need for the `type` field
- Addon controllers have full control over their own availability status
- Cleaner separation of concerns

**Cons**:
- Less clear semantics - users cannot declare hard dependencies in the API
- Addon controllers must implement their own logic to set `Available=False` when dependencies are missing
- More work for addon authors who want hard dependency behavior

**Decision**: Not chosen as the primary approach because the `type` field provides clearer semantics and reduces boilerplate for addon authors. However, this remains a valid alternative if the `type` field proves too complex in practice.

### Alternative 2: Use kro Instead of Adding Dependency Support to OCM

[kro](https://kro.run/) was suggested on this proposal and in
[#194](https://github.com/open-cluster-management-io/enhancements/issues/194) as a way to get
dependency handling without changing the addon API. Both placements of it were prototyped or
assessed and neither met the requirement. Recorded here so the evaluation is not repeated.

Evidence: https://gist.github.com/payjain/e12a5d8ea0188cc466dd6edbb2f001e5 (upstream OCM
v1.3.1 on kind, 1 hub and 2 managed clusters, kro v0.9.4).

**kro on the hub.** A `ResourceGraphDefinition` gates creation of the dependent
`ManagedClusterAddOn` until its prerequisites are `Available`.

- Worked: install ordering and gating with no OCM code changes; kro named the missing
  prerequisites; the dependent addon installed once they became available, and reinstalled
  after a prerequisite was restored.
- Only works with `installStrategy: Manual`. With `installStrategy: Placements` the
  addon-manager creates and deletes the `ManagedClusterAddOn` itself and bypasses the gate.
- The failure is reported on the kro instance, not on the addon. The `ManagedClusterAddOn`
  still reported `Available=True`, which is the opposite of what this proposal is for.
- By default kro deletes the dependent `ManagedClusterAddOn` when a prerequisite
  disappears, uninstalling the addon. Avoiding that needs a workaround.

**kro on every managed cluster.** Each addon gets a graph and graphs wait on each other,
which would put the gate below Placement and also give ordered apply and readiness waits.

- Requires shipping and supporting kro - controller, CRDs, and an applier ServiceAccount
  per graph - on every managed cluster, which is significant on single-node and edge
  fleets.
- An RGD resolves each referenced kind's OpenAPI schema when it is created, so a graph
  waiting on a not-yet-installed CRD never reaches `Active` - exactly the state it would
  need to wait in. Only the alpha `Graph` API avoids this.
- The addon would go `Available=False` with a generic message rather than one naming the
  missing prerequisite, because `AddOnTemplate` addons use
  `HealthProberTypeWorkloadAvailability`.
- A managed cluster cannot enable an addon on the hub, so auto-installing prerequisites
  would still need hub-side work.
- Version skew: RGDs written by addon authors must match the kro version on each managed
  cluster.

**Decision**: Not chosen. Dependency declaration and status belong in the addon API, where
the failure can be reported on the addon itself and where both install strategies behave
the same. This does not rule out kro for other purposes, such as
[#194](https://github.com/open-cluster-management-io/enhancements/issues/194).

## Infrastructure Needed

- No new infrastructure required
- Existing CI/CD pipelines can be used for testing
- Documentation updates needed in open-cluster-management website
