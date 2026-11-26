# Cluster-Specific Manifest Overrides for ManifestWorkReplicaSet

## Release Signoff Checklist

- [ ] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] Graduation criteria for dev preview, tech preview, GA
- [ ] User-facing documentation is created in [website](https://github.com/open-cluster-management-io/open-cluster-management-io.github.io/)

## Summary

`ManifestWorkReplicaSet` (MWRS) distributes one `manifestWorkTemplate` to every cluster
selected by its `placementRefs`. The payload is copied verbatim: only the `ManifestWork`'s
namespace, labels and annotations differ per cluster. Users who need a small per-cluster
difference — a `nodeSelector` value that matches the local node pool, a pod label that lets
a service mesh distinguish copies — have to abandon MWRS and maintain one `ManifestWork`
per cluster.

This proposal introduces a new namespaced resource, `ManifestOverridePolicy`, which selects
manifests by resource identity and clusters by `ManagedCluster` labels, and applies ordered
patches (JSON Patch, JSON Merge Patch, or strategic merge patch) to the matched manifests as
they are rendered for each cluster. An MWRS lists the policies that apply to it in a new
`spec.overridePolicies` field, so the MWRS spec states exactly what shapes its payload, and
only the listed policies are applied. A platform team can define an override once and have
many workloads reference it.

A companion cluster-scoped resource, `ResourceOverrideSchema`, supplies the list-merge metadata
(`x-kubernetes-list-type` and `x-kubernetes-list-map-keys`) that strategic merge patching needs
for custom resources. A CRD schema already carries this metadata, but the hub often does not
have the CRD, because the type is installed only on the managed clusters. The registered schema
fills that gap, and also overrides parts of a hub CRD schema when needed.

## Motivation

The request originates in [ocm#1175][ocm-1175] and was carried into
[enhancements#198][issue-198]. Two distinct needs appear in those threads, and separating
them explains the shape of this proposal.

**Values that are facts about a cluster.** The application node pool is tagged
`nodepool: apps-a` on `cluster-a` and `nodepool: apps-b` on `cluster-b`. A Deployment's
`nodeSelector` must match whichever one it lands on. This is true of every workload on that
cluster, not just this one. Region-scoped settings — a higher `priorityClassName` in
`region-a` — have the same shape. These are naturally expressed as "for clusters matching
this label, patch this field", which is what this proposal provides.

**Values derived from cluster identity.** Each cluster's copy of a Deployment needs a
distinguishing pod label so Envoy or Istio can route to the copies individually. Nothing
about `cluster-a` says the label should be any particular string, only that it must differ
from `cluster-b`'s. Expressing this with static patches means one override entry per cluster,
which does not survive fleet growth. This need is **out of scope here** and is deferred to a
separate templating enhancement (see [Non-Goals](#non-goals)); the API below is shaped so
that templated patch values can be added without a breaking change.

### Current Limitations

Verified against `open-cluster-management-io/api` and `open-cluster-management-io/ocm` at
`main`:

- `ManifestWorkReplicaSetSpec` (`work/v1alpha1/types_manifestworkreplicaset.go`) has three
  fields — `manifestWorkTemplate`, `placementRefs`, `cascadeDeletionPolicy`. There is no
  customization surface.
- Delivery is a literal struct copy: `Spec: mwrSet.Spec.ManifestWorkTemplate` in
  `manifestworkreplicaset_deploy_reconcile.go`.
- The MWRS controller watches `ManifestWorkReplicaSet`, `ManifestWork`, `Placement` and
  `PlacementDecision` (`manifestworkreplicaset_controller.go`). None of them carries cluster
  labels: `ClusterDecision` has just `clusterName`, `reason` and `score`. No `ManagedCluster`
  attribute is reachable from the controller today.

### Goals

- Let users keep one shared `manifestWorkTemplate` in MWRS and declare per-cluster
  differences separately.
- Select target clusters by `ManagedCluster` label, and target manifests by resource identity.
- Support JSON Patch (RFC 6902), JSON Merge Patch (RFC 7386) and Kubernetes strategic merge
  patch, including for custom resources.
- Make the applied result deterministic: the same inputs always produce the same rendered
  `ManifestWork`.
- Make override failures visible per cluster, and surface them to the MWRS rollout machinery
  rather than silently stalling.
- Make it possible to tell which policy changed a given `ManifestWork`.

### Non-Goals

- **Templating or CEL expressions in patch values.** Deferred to a separate enhancement
  covering both MWRS and `ManifestOverridePolicy`. The consensus on [#198][issue-198] is that
  it should use CEL with the full `ManagedCluster` object as context rather than fixed
  placeholders such as `{{CLUSTER_NAME}}`.
- **`forEach`-style iteration** over a list to generate repeated patches. Keeping `overrides`
  a list leaves syntactic room, but iteration source, variable scope and execution order are
  genuinely undecided; see [Open Questions](#open-questions).
- **Overriding anything outside the manifest payload.** `manifestConfigs`, `deleteOption` and
  the other `ManifestWorkSpec` fields are not patchable by this proposal.
- **Changing a manifest's identity.** A patch may not change `apiVersion`, `kind`, `name` or
  `namespace`; see [Identity invariant](#identity-invariant).
- **Selecting clusters by `Placement` or `ManagedClusterSet`.** Only a label selector in the
  first iteration; see [Alternatives](#alternatives).

## Proposal

### User Stories

#### Story 1: node pool differs per cluster

As a platform engineer, I distribute one Deployment to twenty clusters. Each cluster tags its
application node pool with a `nodepool` label on its `ManagedCluster`. I write one
`ManifestOverridePolicy` with one override per distinct node pool value. Each MWRS that lists
it gets the right `nodeSelector` on every cluster, without one copy of the Deployment per node
pool.

#### Story 2: more replicas in production

As an application owner, I want `replicas: 5` on clusters labelled `env: prod` and the
template's `replicas: 2` everywhere else, without maintaining two copies of the Deployment.

#### Story 3: a shared override reused by many workloads

As a platform engineer, every workload my team distributes must carry a compliance label and
run with the `eu-critical` `priorityClassName` on clusters in `region: eu`. I define that once
as a policy, and each MWRS lists it by name rather than having the patch pasted into it.

An earlier draft of this story also added a toleration. That part was removed because no
patch type can append to `tolerations` on every workload; see the known limitation under
[Patch application](#patch-application).

#### Story 4: different overrides for different workloads

As a batch platform owner, I run jobs whose priority depends on both the job and the cluster
(from [@everpeace on #202][pr-202-everpeace]):

| Job | `cluster-x` | `cluster-y` |
| --- | --- | --- |
| A | low (template) | low (template) |
| B | high | middle |
| C | middle | low (template) |

I write one policy per (priority, cluster) pair I need — `x-high`, `x-middle`, `y-middle` —
and each job's MWRS lists the ones it wants: B lists `x-high` and `y-middle`, C lists
`x-middle`, A lists none. Adding a new policy for job D changes nothing for A, B or C.

#### Story 5: patching a custom resource

As a platform engineer, I distribute a Knative `Service` and need to add one environment
variable to its container on clusters in `region: east`, leaving the image and the sidecar
container intact. I register a `ResourceOverrideSchema` for `serving.knative.dev/v1 Service`
once, and strategic merge patches on that type then merge the container list by name instead
of replacing it.

### API Design

#### ManifestOverridePolicy

Namespaced. Applies only to a `ManifestWorkReplicaSet` in the same namespace that lists it in
`spec.overridePolicies`.

```go
// ManifestOverridePolicy defines patches applied to manifests distributed by the
// ManifestWorkReplicaSets that reference it, for the clusters it selects.
//
// +genclient
// +kubebuilder:object:root=true
type ManifestOverridePolicy struct {
    metav1.TypeMeta   `json:",inline"`
    metav1.ObjectMeta `json:"metadata,omitempty"`

    Spec ManifestOverridePolicySpec `json:"spec"`
}

// ManifestOverridePolicySpec selects manifests and clusters, and the patches to apply
// where both match.
type ManifestOverridePolicySpec struct {
    // ResourceSelectors selects the manifests this policy patches. A manifest is
    // selected if it matches any selector. Name and namespace accept "*" to match any
    // value, as elsewhere in ManifestWork.
    // +kubebuilder:validation:MinItems=1
    // +kubebuilder:validation:MaxItems=32
    ResourceSelectors []workv1.ResourceIdentifier `json:"resourceSelectors"`

    // Overrides are evaluated in list order. Every override whose ClusterSelector
    // matches the cluster contributes its patches, in order.
    // +kubebuilder:validation:MinItems=1
    // +kubebuilder:validation:MaxItems=32
    Overrides []Override `json:"overrides"`
}

// Override applies a list of patches on the clusters matched by ClusterSelector.
type Override struct {
    // ClusterSelector selects clusters by ManagedCluster labels. An omitted selector and
    // an empty one ({}) both match every cluster selected by the MWRS placements.
    // +optional
    ClusterSelector *metav1.LabelSelector `json:"clusterSelector,omitempty"`

    // Patches are applied in list order to each selected manifest.
    // +kubebuilder:validation:MinItems=1
    // +kubebuilder:validation:MaxItems=32
    Patches []Patch `json:"patches"`
}

// Patch is a single patch applied to a selected manifest.
type Patch struct {
    // Type is the patch format.
    Type PatchType `json:"type"`

    // Patch is the patch body, written as an object or array rather than a string so
    // it stays readable in the resource. The controller validates it against Type.
    //
    // It is apiextensionsv1.JSON rather than runtime.RawExtension because
    // controller-gen renders RawExtension as "type: object", and an RFC 6902 patch body
    // is a JSON array, so every JSONPatch would be rejected at admission.
    Patch apiextensionsv1.JSON `json:"patch"`

    // AtomicValues sets what happens when a StrategicMergePatch would discard a value
    // it cannot merge into: a list, map or struct that the schema marks atomic, or a
    // list whose merge semantics the available schema does not describe. It applies
    // only where the manifest already holds a non-empty value at that path; setting a
    // value that is absent or empty discards nothing and is always allowed.
    // Ignored for JSONPatch and JSONMergePatch.
    // +kubebuilder:default=Reject
    // +optional
    AtomicValues AtomicValuePolicy `json:"atomicValues,omitempty"`
}

// PatchType is the format of a Patch body.
// +kubebuilder:validation:Enum=JSONPatch;JSONMergePatch;StrategicMergePatch
type PatchType string

const (
    // PatchTypeJSONPatch is an RFC 6902 JSON Patch.
    PatchTypeJSONPatch PatchType = "JSONPatch"
    // PatchTypeJSONMergePatch is an RFC 7386 JSON Merge Patch.
    PatchTypeJSONMergePatch PatchType = "JSONMergePatch"
    // PatchTypeStrategicMergePatch is a Kubernetes strategic merge patch.
    PatchTypeStrategicMergePatch PatchType = "StrategicMergePatch"
)

// AtomicValuePolicy sets how a strategic merge patch treats a value it cannot merge
// into.
// +kubebuilder:validation:Enum=Reject;Replace
type AtomicValuePolicy string

const (
    // AtomicValueReject fails the patch rather than discarding the existing value.
    AtomicValueReject AtomicValuePolicy = "Reject"
    // AtomicValueReplace replaces the whole value, for users who intend it.
    AtomicValueReplace AtomicValuePolicy = "Replace"
)

// ManifestOverridePolicyList is a list of ManifestOverridePolicy.
// +kubebuilder:object:root=true
type ManifestOverridePolicyList struct {
    metav1.TypeMeta `json:",inline"`
    metav1.ListMeta `json:"metadata,omitempty"`

    Items []ManifestOverridePolicy `json:"items"`
}
```

`ManifestOverridePolicy` has no status in alpha. A policy does not render on its own: it is
rendered once per MWRS and cluster, inside the reconcile of each MWRS that lists it, and each
of those reconciles sees only its own result. A per-policy condition written from there would
have many writers and would flip with reconcile order. The outcome is reported where it is
produced, on the MWRS (see [Observability](#observability)). Whether the policy should carry
an aggregate is [open question 10](#open-questions).

`clusterSelector` treats an omitted selector and `{}` alike. The implementation has to
special-case the omitted one, because `metav1.LabelSelectorAsSelector(nil)` returns a selector
that matches nothing.

#### ResourceOverrideSchema

Cluster-scoped and administrator-managed. Schema selection has to be unambiguous, so
uniqueness is defined explicitly rather than left to "one per type".

**The uniqueness key is `(spec.target.group, spec.target.resource, spec.version)`.** Two
objects sharing that triple conflict, whatever they are named.

`spec.kind` is deliberately *not* part of the key. Within one group/version a resource maps to
exactly one kind, so admitting `kind` into the key would permit two objects that disagree
about the kind for the same resource — precisely the ambiguity the rule exists to prevent. The
reverse mapping must be unique for the same reason: two objects mapping the same
`(group, version, kind)` to different resources also conflict.

Different versions of one resource are *not* a conflict. A type served at both `v1` and
`v1beta1` may legitimately carry different list metadata per version, and the manifest's
`apiVersion` selects which schema applies.

`spec.target` has only `group` and `resource` — it names a type, not an instance. It is a
type of its own rather than `ResourceIdentifier`, because `ResourceIdentifier` requires a
`name` (`manifestConfigs` needs one) and a schema has none to give.

Conflicts are rejected deterministically rather than resolved by picking a winner:

- Both objects get a `Conflicting` condition naming the other, so the problem is visible on
  the objects themselves and not only in controller logs.
- Any render needing that schema fails for the affected cluster, with an error naming both
  objects and the duplicated key. It does **not** fall back to treating the type as
  schemaless: that would silently downgrade a merge to an atomic replacement, which is the
  exact failure `atomicValues: Reject` exists to prevent.
- Renders that do not touch the contested type are unaffected.

Last-writer-wins and lowest-name-wins were both considered and rejected: either makes the
rendered payload depend on creation order or on object naming, which contradicts the
determinism guarantee in [Ordering](#ordering). A validating webhook rejecting the second
object at admission is worthwhile hardening on top, but the controller-side rejection is
required regardless, since a webhook cannot catch objects created while it is unavailable.

```go
// ResourceOverrideSchema supplies list-merge metadata for a resource type, so that
// strategic merge patches can merge lists instead of replacing them. It affects
// override rendering only: it does not modify any CRD or API server validation.
//
// +genclient
// +genclient:nonNamespaced
// +kubebuilder:object:root=true
// +kubebuilder:resource:scope=Cluster
// +kubebuilder:subresource:status
type ResourceOverrideSchema struct {
    metav1.TypeMeta   `json:",inline"`
    metav1.ObjectMeta `json:"metadata,omitempty"`

    Spec   ResourceOverrideSchemaSpec   `json:"spec"`
    Status ResourceOverrideSchemaStatus `json:"status,omitempty"`
}

// ResourceOverrideSchemaSpec names a resource type and describes how its lists merge.
type ResourceOverrideSchemaSpec struct {
    // Target is the resource type this schema describes, using the same group/resource
    // identity as ManifestOverridePolicy resourceSelectors.
    Target ResourceTypeIdentifier `json:"target"`

    // Kind is the manifest kind served for Target. With Version, it gives the hub the
    // kind-to-resource mapping for a type that is installed only on managed clusters,
    // where the hub has no RESTMapper entry.
    // +kubebuilder:validation:MinLength=1
    Kind string `json:"kind"`

    // Version is the served version this schema describes.
    // +kubebuilder:validation:MinLength=1
    Version string `json:"version"`

    // OpenAPIV3Schema describes the merge semantics of the paths a patch may touch.
    // If the hub has the CRD, it is overlaid on the CRD schema; otherwise it is the
    // whole schema. See Effective schema.
    OpenAPIV3Schema apiextensionsv1.JSON `json:"openAPIV3Schema"`
}

// ResourceTypeIdentifier names a resource type by group and resource.
type ResourceTypeIdentifier struct {
    // Group is the API group of the resource. Empty means the core group.
    // +optional
    Group string `json:"group"`

    // Resource is the plural resource name, for example "services".
    // +kubebuilder:validation:MinLength=1
    Resource string `json:"resource"`
}

// ResourceOverrideSchemaStatus is the observed state of a ResourceOverrideSchema.
type ResourceOverrideSchemaStatus struct {
    // ObservedGeneration is the generation of the schema that Conditions describe.
    // +optional
    ObservedGeneration int64 `json:"observedGeneration,omitempty"`

    // Conditions describe the state of the schema. The known type is Conflicting.
    // +listType=map
    // +listMapKey=type
    // +optional
    Conditions []metav1.Condition `json:"conditions,omitempty"`
}

const (
    // ResourceOverrideSchemaConditionConflicting is True when another
    // ResourceOverrideSchema has the same uniqueness key. The message names the other
    // object. While it is True, renders that need this type fail.
    ResourceOverrideSchemaConditionConflicting string = "Conflicting"
)

// ResourceOverrideSchemaList is a list of ResourceOverrideSchema.
// +kubebuilder:object:root=true
type ResourceOverrideSchemaList struct {
    metav1.TypeMeta `json:",inline"`
    metav1.ListMeta `json:"metadata,omitempty"`

    Items []ResourceOverrideSchema `json:"items"`
}
```

Making `ResourceOverrideSchema` carry the mapping resolves two things at once. It gives the
API a single resource-identification scheme, as @kahirokunn asked for on [#198][issue-198]
and as review feedback on [#155][pr-155] preferred. And it closes a hole that group/resource
selection would otherwise have: the hub cannot discover a type it does not have installed, so
without this the Knative example below could not be selected at all.

Example, declaring that a Knative `Service`'s containers and their environment variables
merge by `name`:

```yaml
apiVersion: work.open-cluster-management.io/v1alpha1
kind: ResourceOverrideSchema
metadata:
  name: knative-service
spec:
  target:
    group: serving.knative.dev
    resource: services
  kind: Service
  version: v1
  openAPIV3Schema:
    type: object
    properties:
      spec:
        type: object
        properties:
          template:
            type: object
            properties:
              spec:
                type: object
                properties:
                  containers:
                    type: array
                    x-kubernetes-list-type: map
                    x-kubernetes-list-map-keys: ["name"]
                    items:
                      type: object
                      properties:
                        name:
                          type: string
                        env:
                          type: array
                          x-kubernetes-list-type: map
                          x-kubernetes-list-map-keys: ["name"]
                          items:
                            type: object
                            properties:
                              name:
                                type: string
```

Knative is installed only on the managed clusters, so the hub has no `Service` schema to
inherit from and this document is the whole of the type information. It is written out in
full for that reason: every ancestor carries `type: object`, every map list declares its
`items` and the scalar property named by `x-kubernetes-list-map-keys`. A key that is named
but not declared cannot be honoured.

The markers are the structural ones, `x-kubernetes-list-type` and
`x-kubernetes-list-map-keys`, because those are what `structured-merge-diff` reads.

This is a deliberate difference from the [custom OpenAPI schema example][kustomize-openapi]
Kustomize documents, which uses the older `x-kubernetes-patch-strategy` /
`x-kubernetes-patch-merge-key` pair that `strategicpatch` reads. The difference matters
rather than being cosmetic: users will arrive carrying the Kustomize form, and an overlay
supplying only those markers leaves the list with no effective merge metadata under the
engine this proposal uses. The list would be treated as atomic, so the merge fails under the
default `atomicValues: Reject`, or discards existing entries under `Replace`. Neither is
acceptable as a silent outcome.

`ResourceOverrideSchema` therefore rejects the legacy pair at validation, with an error
giving the structural equivalent:

| Legacy marker | Structural equivalent |
| --- | --- |
| `x-kubernetes-patch-strategy: merge` + `x-kubernetes-patch-merge-key: name` | `x-kubernetes-list-type: map` + `x-kubernetes-list-map-keys: ["name"]` |
| array with no `x-kubernetes-patch-strategy` | `x-kubernetes-list-type: atomic` |

Accepting both forms and converting internally was considered and rejected: it puts two ways
of expressing one thing on a brand-new API, and commits us to keeping the conversion correct
forever, to save the author a one-line edit that the validation error spells out.

#### Effective schema

Strategic merge for a custom resource needs a schema to convert against, and in alpha the
hub may have none, so the resolution order has to be stated rather than assumed:

1. If the hub has the CRD, its structural schema for the matching version is the base, and
   the registered `ResourceOverrideSchema` is overlaid on it. Overlaid metadata wins.
2. If the hub does not have the CRD, the registered schema is the base. There is nothing to
   inherit.
3. In both cases, any path the resulting schema does not describe falls back to the deduced
   semantics `structured-merge-diff` uses for untyped data.

Step 3 is the part that matters, because the fallback is not neutral. In the vendored
`structured-merge-diff` v6, `DeducedParseableType` defines `__untyped_deduced_` with maps
`separable` and lists `atomic`. So an undescribed map still merges field by field, while an
**undescribed list is atomic**. Combined with the `atomicValues: Reject` default, a list the
author forgot to describe fails the patch with an error naming the path, rather than
silently replacing the list. That is the safe direction, and it is why the fallback is
acceptable for alpha rather than a hole.

The practical rule for authors is therefore narrow: describe every list you expect to merge,
down to the scalar properties named as its map keys. Everything else may be omitted.

This is also the sharpest argument for the agent-sourced schema cache in
[open question 1](#open-questions). Once schemas are collected from the clusters that
actually serve the type, step 2 stops being a hand-written document and the fallback stops
being reachable in normal use.

#### ManifestWorkReplicaSet references

An MWRS names the policies that apply to it. This is the only change to an existing API in
this proposal.

```go
// ManifestWorkReplicaSetSpec gains one optional field. Existing fields are unchanged.
type ManifestWorkReplicaSetSpec struct {
    // ... ManifestWorkTemplate, PlacementRefs and CascadeDeletionPolicy ...

    // OverridePolicies lists the ManifestOverridePolicy objects, in the same namespace,
    // applied to this ManifestWorkReplicaSet. Order is significant: policies are
    // applied in list order, so a later policy wins a conflict. See Ordering.
    // +listType=map
    // +listMapKey=name
    // +kubebuilder:validation:MaxItems=16
    // +optional
    OverridePolicies []LocalOverridePolicyReference `json:"overridePolicies,omitempty"`
}

// LocalOverridePolicyReference names a ManifestOverridePolicy in the same namespace.
// It is a struct rather than a plain string so that fields can be added later without
// changing the type of spec.overridePolicies.
type LocalOverridePolicyReference struct {
    // Name is the name of the ManifestOverridePolicy.
    // +kubebuilder:validation:MinLength=1
    Name string `json:"name"`
}
```

The policies applied to an MWRS are exactly the ones listed in `spec.overridePolicies`. Nothing
is added to the list, and nothing outside it is applied. The MWRS spec then shows every policy
that shapes its payload, following the Kubernetes object model. This follows @everpeace's
proposal on [#202][pr-202-everpeace], and @kahirokunn's decision as approver on
[#202][pr-202-kahirokunn-declarative] to keep the alpha API declarative.

Names in `overridePolicies` are unique. The list is `+listType=map` keyed by `name`, so the
API server rejects a duplicate and the validating webhook does not check for one. A name that
does not exist yet is **not** rejected, so an MWRS and its policies can be applied in any
order, for example by a GitOps tool syncing a directory. A listed policy that does not exist
is reported on the MWRS (see [Failure handling](#failure-handling)), not ignored.

Adding policies to an MWRS automatically (default policies), and making some policies
mandatory, are out of scope for alpha; see [open question 8](#open-questions).

### Example

```yaml
apiVersion: work.open-cluster-management.io/v1alpha1
kind: ManifestOverridePolicy
metadata:
  name: nodepool-per-cluster
  namespace: team-a
spec:
  resourceSelectors:
    - group: apps
      resource: deployments
      namespace: "*"
      name: "*"
  overrides:
    - clusterSelector:
        matchLabels:
          nodepool: apps-a
      patches:
        - type: JSONMergePatch
          patch:
            spec:
              template:
                spec:
                  nodeSelector:
                    nodepool: apps-a
    - clusterSelector:
        matchLabels:
          nodepool: apps-b
      patches:
        - type: JSONMergePatch
          patch:
            spec:
              template:
                spec:
                  nodeSelector:
                    nodepool: apps-b
```

The `ManifestWorkReplicaSet` lists the policy it uses:

```yaml
apiVersion: work.open-cluster-management.io/v1alpha1
kind: ManifestWorkReplicaSet
metadata:
  name: web-app
  namespace: team-a
spec:
  manifestWorkTemplate:
    workload:
      manifests:
        - apiVersion: apps/v1
          kind: Deployment
          metadata:
            name: web-app
            namespace: default
          spec:
            replicas: 2
            # ...
  placementRefs:
    - name: web-app-placement
  overridePolicies:
    - name: nodepool-per-cluster
```

On a cluster labelled `nodepool: apps-a` the Deployment gets `nodeSelector: {nodepool: apps-a}`,
on `nodepool: apps-b` it gets `apps-b`, and on any other cluster it is the plain template.

### Why explicit references

Three models were discussed on [#198][issue-198] and [#202][pr-202]:

1. **Implicit application with an opt-out.** Every policy in the namespace applies to every
   MWRS, unless the MWRS sets `overrideMode: Disabled`. This was an earlier draft of this
   proposal. @kahirokunn's point behind it was that "policies will inevitably be missed in
   enumeration".
2. **Explicit references with defaults.** The MWRS lists its policies, a mutating webhook adds
   policies marked `default: true` when the MWRS is created, and an MWRS can skip one with
   `disabled: true`. This was the draft after (1).
3. **Explicit references only.** The MWRS lists its policies and nothing else applies.

This proposal takes (3).

- (1) cannot express Story 4. An all-or-nothing switch cannot give a job some policies but
  not others. A deny list could, but then every new policy reaches every existing MWRS until
  each one is updated to deny it.
- (2) was dropped on review. @everpeace asked whether a default that MWRS can skip is really a
  default, and pointed out a race between changing `default` and creating an MWRS. It also
  meant the stored MWRS differs from what the user wrote, which shows up as drift in GitOps
  tools, and it needed a new mutating webhook that fails MWRS creation when unavailable.
  @kahirokunn agreed to keep alpha declarative.

What (3) gives up: an author can forget to list a platform policy, which was the concern
behind (1). Users who need policies added or enforced automatically can use existing
Kubernetes admission mechanisms or GitOps tooling for now, as @kahirokunn noted. OCM support
for it is open question 8.

### Implementation Details

#### Feature gate

The feature is behind a new hub work feature gate, `ManifestOverridePolicy`, Alpha and off by
default. It is named after the resource it adds, like the existing `ManifestWorkReplicaSet`
gate, and it only has an effect when that gate is also enabled.

With the gate off:

- The MWRS controller starts none of the new informers, so a hub that does not use the
  feature does not pay for the `ManagedCluster` cache. It ignores `spec.overridePolicies` and
  renders the plain template.
- The MWRS validating webhook rejects a create with a non-empty `spec.overridePolicies`, and an
  update that adds an entry, with an error naming the feature gate. Put as one rule: the new
  list must be a subset of the old one. An update that keeps or removes entries is allowed, so
  an MWRS created while the gate was on can still be edited, and cleaned up, after it is turned
  off. This is new logic. The existing webhook has no per-field gating: it is not installed at
  all when the `ManifestWorkReplicaSet` gate is off (`pkg/work/webhook/start.go`), and its only
  gate check rejects every request.
- The controller removes the `OverridesRendered` condition and the `override-policies`
  annotation wherever they are still set, since it no longer renders overrides.
- The validating webhooks for `ManifestOverridePolicy` and `ResourceOverrideSchema` are not
  registered, and the objects have no effect.

#### A single renderer, used by every call site

This is a correctness constraint, not a preference. It covers every place that copies the
template. Today the MWRS controller has three, and two of them are drift detection:

| Site | Purpose |
|---|---|
| `manifestworkreplicaset_deploy_reconcile.go` (candidate build) | builds a candidate and calls `workapplier.ManifestWorkEqual` |
| `manifestworkreplicaset_status_reconcile.go` | the same comparison, for status accounting |
| `manifestworkreplicaset_deploy_reconcile.go` (create) | creates the `ManifestWork` |

If overrides were applied only at creation, both comparison sites would compare an
un-rendered template against a rendered live object. They would never agree. The cluster
would stay `ToApply`, `Progressive` rollout would keep selecting the same clusters,
`PlacementRolledOut` would never reach `Complete`, and the status counts would be wrong.

Every such site must therefore route through one function:

```go
// Render returns the ManifestWorkSpec for one cluster: template with the patches of
// every matching policy applied. It does not modify its arguments, and the same inputs
// always produce the same output.
func Render(template *workv1.ManifestWorkSpec, cluster *clusterv1.ManagedCluster,
    policies []*workv1alpha1.ManifestOverridePolicy, types TypeResolver) (*Result, error)

// TypeResolver maps a manifest's kind to its resource and its merge schema. It is
// backed by hub discovery and by ResourceOverrideSchema objects.
type TypeResolver interface {
    ResourceFor(gvk schema.GroupVersionKind) (schema.GroupVersionResource, error)
    // MergeSchema returns nil for a built-in type, which uses strategicpatch.
    MergeSchema(gvk schema.GroupVersionKind) (*typed.ParseableType, error)
}

// Result is a rendered ManifestWorkSpec and the policies that changed it.
type Result struct {
    Spec *workv1.ManifestWorkSpec
    // AppliedPolicies are the names of the policies that patched at least one
    // manifest, in the order they were applied.
    AppliedPolicies []string
}
```

`Render` takes the whole `ManagedCluster` rather than its labels so that the templating
enhancement, which will evaluate CEL against the cluster object, does not need a new
signature. It returns an error of type `*RenderError`, which carries the policy, manifest,
override and patch index of the failure.

The two comparison sites also need a candidate whose metadata comes from the render, not from
the live object. Today both copy the live `ManifestWork`'s `ObjectMeta` into the candidate
before calling `ManifestWorkEqual`, so labels and annotations always compare equal and only
the spec is checked. That stops being enough once the controller writes the `override-policies`
annotation; the comparison rule is given under [Observability](#observability).

A prototype with unit tests for the scenarios in [#198][issue-198] is in
[ocm#1792][ocm-1792]. It still uses the earlier signature and will be updated to match.

#### Ordering

1. The policies listed in `spec.overridePolicies` are applied in list order, so a later
   policy wins a conflict. Each MWRS sets its own order, so two MWRS can apply the same
   policies in different orders without copying a policy. The list is `+listType=map`
   keyed by `name`, and order still matters in such a list, as it does for
   `initContainers`.
2. Within a policy, `overrides` are evaluated in list order; an override contributes only if
   its `clusterSelector` matches the cluster.
3. Within an override, `patches` are applied in list order.

Later patches see the result of earlier ones. Because every list is ordered, two renders of
the same inputs always produce the same bytes.

Policies have no `priority` field. An earlier draft had one, because policies then applied
implicitly and there was no list to take an order from. With explicit references the list is
that order, per @everpeace and @kahirokunn on [#202][pr-202-priority]. Adding or removing a
priority later would change the rendered result of existing MWRS, so this is decided before
alpha rather than after.

#### Resource selection

`resourceSelectors` reuses `ManifestWork`'s existing `ResourceIdentifier`
(group / resource / namespace / name), the same type `manifestConfigs` uses, with the same
`"*"` wildcard support for name and namespace. This keeps one resource-addressing concept in
the work API and lets the implementation reuse the matching that `manifestConfigs` already
has. That is `resourceMatch` in `pkg/work/helper`, which is unexported today and has to be
exported or moved.

This requires mapping each manifest's `apiVersion`/`kind` to its resource, which the work hub
does not do today — it needs a `RESTMapper`. A `RESTMapper` built from hub discovery covers
built-in types and any CRD installed on the hub. For a type installed only on managed
clusters, the mapping comes from the `kind`/`version`/`target` triple on its
`ResourceOverrideSchema`, as described above.

A manifest whose type neither route can resolve has no resource name to match against, but
its group is still known from its `apiVersion`. The rule uses that:

- If no selector names the manifest's group, the manifest cannot be a target and is left
  unpatched.
- If a selector does name its group, the render fails for that cluster. The controller cannot
  tell whether the selector was meant to match the manifest, and skipping it would drop an
  override silently. The error names the manifest's `apiVersion` and `kind`, and says that a
  `ResourceOverrideSchema` for that version, or the CRD on the hub, is missing.

Only the selectors of policies that have an override matching the cluster are considered,
since the others contribute no patch there.

The check starts from the manifests in the template, not from the selectors. A selector that
names a type the template does not contain is never an error: a shared policy legitimately
selects types that a given MWRS does not ship. So deleting the Knative
`ResourceOverrideSchema` fails the MWRS whose template holds a `serving.knative.dev` manifest,
and leaves alone the ones that list the same policy but ship only Deployments.

#### Patch application

- **JSONPatch** — RFC 6902, via `github.com/evanphx/json-patch/v5`. v5 is used deliberately:
  v4 lets `replace` on a missing field succeed silently, v5 returns an error. `sdk-go`
  already uses v5. Patches are applied with the library's default options, in which
  `EnsurePathExistsOnAdd` and `AllowMissingPathOnRemove` are both off: an `add` never creates
  a missing parent, and a `remove` of a missing path is an error.
- **JSONMergePatch** — RFC 7386.
- **StrategicMergePatch** — for built-in types the merge keys come from the Go types in
  `client-go`'s scheme. For custom resources they come from the type's CRD schema, overlaid
  with any `ResourceOverrideSchema` registered for that type.

For custom resources the implementation uses typed conversion plus merge from
`sigs.k8s.io/structured-merge-diff`, which is already vendored in OCM, rather than
`strategicpatch`. This is the approach Kubernetes' own `MutatingAdmissionPolicy`
`ApplyConfiguration` takes. It is necessary because a raw CRD schema declares list semantics
with `x-kubernetes-list-type` and `x-kubernetes-list-map-keys`, which `strategicpatch` does
not read; handing it a CRD schema is not sufficient on its own. A composite
`x-kubernetes-list-map-keys` must be honoured in full and must never be silently reduced to
its first field.

**What the result is defined as.** The two engines do not agree everywhere, so this proposal
states what each one promises instead of claiming they are the same.

- For a built-in type, a patch that is applied gives the result the Kubernetes API server's
  own strategic merge patch gives for the same patch on the same object. It is computed by
  the same library from the same Go types.
- For a custom resource, which the API server does not accept strategic merge patches for at
  all, the result is defined by the structural markers in the
  [effective schema](#effective-schema).

The same patch can therefore merge differently when a custom resource embeds a structure that
a built-in type also has. The known cases:

| Field | Built-in (`strategicpatch`) | Custom resource (typed merge) |
| --- | --- | --- |
| `containers[].ports` | merged by `containerPort` | merged by `containerPort` and `protocol`, when the schema declares both |
| `Service` `spec.ports` | merged by `port` | merged by `port` and `protocol`, when the schema declares both |
| `topologySpreadConstraints` | merged by `topologyKey` | merged by `topologyKey` and `whenUnsatisfiable`, when the schema declares both |
| `nodeSelector` and other maps marked atomic | merged key by key | one value, replaced as a whole |

In the first three rows, a patch entry that shares the first key with an existing entry and
differs in the second overwrites that entry on a built-in, and is added next to it on a custom
resource. Neither is an error. The last row is covered by `atomicValues`, below. An author who
needs one exact outcome on every type should use a JSON Patch. Whether to run built-ins
through the typed merge as well, which would remove the table, is
[open question 9](#open-questions).

**Values that cannot be merged into.** When a strategic merge patch would discard a value it
cannot merge into, the patch fails rather than silently replacing the value, unless the patch
sets `atomicValues: Replace`. This covers:

- a list the schema marks atomic, or a map list whose keys cannot be honoured;
- on the custom resource path, a map or struct the schema marks atomic
  (`x-kubernetes-map-type: atomic`). `structured-merge-diff` treats these as one value, just
  as it does an atomic list, and `MutatingAdmissionPolicy` refuses to mutate all three for the
  same reason. On the built-in path there is nothing to guard, because `strategicpatch` merges
  every map key by key.

The check applies only where the manifest already holds a non-empty value at that path.
Setting a value that is absent or empty discards nothing, so it is allowed under `Reject`.
The outcome can then differ between two workloads that list the same policy, when one has the
field and the other does not. The one that would lose data fails with an error naming the
path, which is what the default is for.

**Known limitation: appending to an atomic list.** `tolerations` is an atomic list with no
merge key, on built-ins and custom resources alike. A strategic merge patch or a JSON Merge
Patch can only set it as a whole, which drops the workload's own tolerations if it has any. A
JSON Patch `add` to `/spec/template/spec/tolerations/-` appends, but fails when the template
has no `tolerations` field, because `add` does not create the parent. So "append this entry,
creating the list if needed" cannot be written as one patch that works on every workload,
which is what a shared policy needs. Alpha does not solve this. A template that declares
`tolerations: []` makes the `add` form work, at the cost of touching every template. Turning
on `EnsurePathExistsOnAdd` would also work, but it makes `add` succeed where RFC 6902 says it
fails, and strictness is the reason v5 was chosen. It is recorded here so that it is a known
gap, not a surprise.

**Deletion semantics must not diverge between built-ins and custom resources.** This is a
trap in the approach above. A strategic merge patch deletes a field by setting it to `null`,
and removes a list entry with the `$patch: delete` directive; `strategicpatch` implements
both. `structured-merge-diff`'s `TypedValue.Merge` does not — it merges, and expresses
removal through field-set subtraction in the apply path, not through nulls in the merge
input. Using `strategicpatch` for built-ins and typed merge for custom resources would
therefore give the same patch two different meanings depending on the target type, and the
difference would be silent.

The implementation must close this explicitly. Before the typed merge, the patch is scanned
for `null` values and `$patch` directives; those are extracted and applied as removals
against the merged result, so that `spec: {field: null}` and `$patch: delete` mean the same
thing for a Knative `Service` as they do for a `Deployment`. Any directive that cannot be
honoured for a custom resource is an error naming the directive and the path, never a
silent no-op. The test plan covers deletion on both a built-in and a custom resource with
the same patch body, asserting identical results.

**A `null` can be lost before the renderer sees it.** The patch body is stored in a custom
resource, and how the policy is written to the API decides whether a `null` in it is stored.
A create keeps it. An update of the policy that is sent as a merge patch does not: in a merge
patch `null` means "remove this key", so client-side `kubectl apply`, `kubectl edit` and
`kubectl patch --type=merge` drop a newly added `null` instead of storing it. The policy then
renders without the deletion, and nothing reports an error, because the stored policy is
valid. Deletions should therefore be written in a form that does not depend on `null`: a JSON
Patch `remove`, or `$patch: delete` in a strategic merge patch. The renderer still honours a
`null` that is present in the stored policy, and the test plan covers which write paths keep
one.

#### Validation

The `ManifestOverridePolicy` validating webhook checks what the CRD schema cannot:

- the patch body matches its `type`: an array of operations for `JSONPatch`, each with a
  valid `op` and `path`, and an object for `JSONMergePatch` and `StrategicMergePatch`;
- every `clusterSelector` converts to a `labels.Selector`;
- each patch body is at most 64 KiB once serialized. `apiextensionsv1.JSON` has no length
  marker, so this cannot be a schema bound.

It does not try to apply a patch, since that needs the manifests and the cluster, and it does
not resolve `resourceSelectors`, since a type may be installed after the policy is created.
Both are render-time checks and fail as described under [Failure handling](#failure-handling).

The lists carry `MaxItems` bounds: 16 entries in `overridePolicies`, and 32 each in
`resourceSelectors`, `overrides` and `patches`. They are starting values, generous for the
stories above, and exist so that one object cannot make every render of an MWRS slow. Raising
a bound later is compatible; lowering one is not, so they start loose rather than tight.

#### RBAC

The work hub controller's `ClusterRole` gains:

- `get`, `list`, `watch` on `manifestoverridepolicies` and `resourceoverrideschemas`
  (`work.open-cluster-management.io`);
- `update`, `patch` on `resourceoverrideschemas/status`, for the `Conflicting` condition;
- `get`, `list`, `watch` on `managedclusters` (`cluster.open-cluster-management.io`) and on
  `customresourcedefinitions` (`apiextensions.k8s.io`). The controller has neither today.

These are added only to the role the controller already has, and are unused while the gate
is off. For users, the proposal ships aggregated roles so that namespace `admin` and `edit`
can manage `ManifestOverridePolicy` in their namespace and `view` can read it.
`ResourceOverrideSchema` is cluster-scoped and changes how every policy merges a type, so it
is left to cluster administrators and is not aggregated.

#### Identity invariant

After all patches are applied, the rendered manifest's `apiVersion`, `kind`, `name` and
`namespace` must equal the original's. A patch that changes any of them is an error. Without
this, a patch could retarget a manifest at a different object, orphan the previously applied
resource, and make `ManifestWork` status accounting meaningless.

#### Controller wiring

The MWRS controller gains four informers:

- `ManifestOverridePolicy`, to re-render when a policy changes. An index on the MWRS informer,
  keyed by the policies each MWRS lists in `spec.overridePolicies`, lets a policy change
  enqueue only those MWRS instead of every MWRS in the namespace. It has the same shape as the
  existing `manifestWorkReplicaSetByPlacement` index.
- `ManagedCluster`, because label selectors need cluster labels, which `PlacementDecision`
  does not carry.
- `ResourceOverrideSchema`, on create, update **and delete**. Merge metadata is an input to
  the rendered output, so editing a schema changes what the manifest should be. Without this
  watch, existing `ManifestWork`s keep the payload rendered under the old metadata until some
  unrelated event happens to reconcile their MWRS, which is both wrong and hard to diagnose.
  Deletion matters as much as update: removing a schema can turn a previously valid merge
  into one that must now fail under `atomicValues: Reject`.
- `CustomResourceDefinition` on the hub, because a hub CRD is an input in two ways:
  - **Resource mapping, for every patch type.** The hub `RESTMapper` resolves
    `resourceSelectors` through hub CRDs (see [Resource selection](#resource-selection)), so
    creating or deleting a CRD, or changing the versions it serves, changes which manifests a
    selector matches. Without this, a render that failed on an unresolvable type is never
    retried once the CRD is installed, and a deleted CRD leaves the old render in place.
  - **Effective schema, for strategic merge.** The hub CRD is the base of the
    [effective schema](#effective-schema), so a schema change such as a list moving from `map`
    to `atomic` changes the merge result.

  Unrelated CRD churn must not enqueue anything, so a CRD event is mapped to MWRS through the
  policies. Which MWRS care about a group and resource depends on what their policies select,
  so this cannot be one informer index. It is two lookups: an index on the policy informer
  from group and resource to the policies whose selectors name it, then the MWRS index above
  from those policies to the MWRS that list them. `ResourceOverrideSchema` events use the same
  two lookups.

  A newly created CRD is not in discovery until its `Established` condition turns `True`, and
  that arrives as a status-only update. So the trigger is the condition, not the create event.
  When `Established` or `NamesAccepted` changes, when the CRD is deleted, or when its served
  versions change, the `RESTMapper` is reset and every MWRS found by the lookup is enqueued.
  An update that changes only the schema enqueues just the MWRS whose policies carry a
  strategic merge patch for that type. Other updates are dropped.

  Discovery is the one input that is not an informer, and it can lag the condition briefly. A
  render that failed because a type could not be resolved is therefore requeued with backoff.
  No other render failure is retried on a timer: every other input is watched, and rendering
  unchanged inputs again gives the same error.

The `ManagedCluster` informer is the significant cost. It caches every `ManagedCluster` on the
hub and must re-render affected MWRS when labels change. Mitigations to measure before
committing: an informer `transform` that drops `managedFields` and `status`, and event
filtering so that only an add, a delete or a label change enqueues work. Add and delete have
to stay: a selected cluster whose `ManagedCluster` is not in the cache is a render failure
(see [Failure handling](#failure-handling)), and it has to be rendered again when the object
appears. Dropping `status` is safe only while selection reads labels alone. `Render` is given
the whole object so that templating can read more of it later, and the transform has to be
revisited when that lands. See [Risks and Mitigation](#risks-and-mitigation).

#### Failure handling

A render failure belongs to one cluster. For that cluster the implementation:

- does not create or update the `ManifestWork`. An existing one is left as it is;
- reports the cluster to the rollout machinery as `TimeOut`, so that it counts toward
  `maxFailures` at once (the reasoning is below);
- does not count the cluster as up to date, so `PlacementRolledOut` cannot become `Complete`
  while a selected cluster fails to render;
- sets the MWRS `OverridesRendered` condition to `False` with reason `RenderFailed` (see
  [Observability](#observability)), so a failure anywhere is visible without inspecting each
  cluster;
- includes the offending policy, manifest index, override index and patch index in the
  message. The prototype's `RenderError` already carries exactly these.

**Why `TimeOut`.** The other two statuses each fail in a different way. A cluster left in
`ToApply` never counts toward `maxFailures`, so the rollout stalls without saying why.
`Failed` counts, but the rollout handler then measures `progressDeadline` from the status's
`LastTransitionTime`, and the controller takes that time from `ManifestWork` conditions. A
render failure writes no work, so there is no such time: the deadline would start again on
every reconcile and the cluster would hold a `maxConcurrency` slot for good. There is also
nothing to wait for: a render gives the same result until one of its inputs changes, and
every input change triggers a new render. `TimeOut` is the status the handler gives a cluster
once its deadline has passed, and the handler accepts it as input. It counts toward
`maxFailures`, holds no slot, and needs no per-cluster timestamp to be stored.

**What is isolated, and what is not.** Rendering is isolated per cluster: one cluster's
failure never changes the payload rendered for another. Whether the rollout carries on is the
rollout strategy's decision, as it is for a cluster whose workload fails:

- `All` carries on to every other cluster.
- `Progressive` and `ProgressivePerGroup` count the failed cluster toward `maxFailures`, which
  defaults to 0. With the default, one render failure stops the rollout from reaching further
  clusters until it is fixed. Clusters that already have the new payload keep it.

This is what surfacing failures to the rollout machinery means in practice: a policy edit
that breaks one cluster halts a progressive rollout the same way a bad image would.

**What keeps running.** Work that does not depend on a render is not held back by a failure:
deleting the `ManifestWork`s of clusters the placement no longer selects, of placements
removed from the spec, and of duplicates; the finalizer when the MWRS is deleted; and status
accounting over the existing `ManifestWork`s. While `OverridesRendered` is `False`,
`ManifestworkApplied` is `NotAsExpected` with the same message, so the clusters that did
render cannot report overall success. The controller already does this when a `Placement` is
missing.

Three cases use the same path, and differ only in how many clusters they affect.

**A rendered work over the size limit** fails the render for that cluster. The work webhook
rejects a `ManifestWork` whose manifests total more than the manifest limit, 500 KiB by
default ([validator.go][ocm-manifest-limit]), and a patch can push the template over it on one
cluster only. The renderer checks the rendered size against the same configured limit, so
the cluster takes the `TimeOut` path above with a message naming the size, instead of
surfacing as an apply error from the webhook.

**A listed policy that does not exist** fails the render for every selected cluster, since
none of them can be rendered without it. `OverridesRendered` is `False` with reason
`OverridePolicyNotFound` naming the policy. Everything above then follows: existing
`ManifestWork`s are left as they are rather than rewritten to the template without that
policy, a newly selected cluster gets no `ManifestWork`, the works of clusters that are no
longer selected are still deleted, and `PlacementRolledOut` stays `Progressing` with a message
naming the policy. Rendering resumes when the policy is created or the reference is removed.
Leaving the works alone is the safer direction: if a policy is deleted by mistake, the
workload keeps running as it was. It is also what the controller does today for a missing
`Placement`, whose works it retains while the conditions say why.

**A selected cluster with no `ManagedCluster` in the cache** fails the render for that
cluster. It is not rendered as a cluster with no labels, which would rewrite its
`ManifestWork` to a payload without the overrides its labels would have selected. This
happens when a `ManagedCluster` is deleted while a `PlacementDecision` still lists it, and it
ends when the placement drops the cluster and the work is deleted in the normal way.

#### Observability

Two mechanisms, in the spirit of `ManagedClusterAddOn.status.configReferences`: the object
that was changed says what changed it, and the object the user owns says whether it worked.

**On each `ManifestWork`: an annotation.**

```go
const (
    // ManifestWorkOverridePoliciesAnnotationKey is set by the MWRS controller on each
    // ManifestWork it renders. The value is a comma-separated list of the names of the
    // ManifestOverridePolicy objects that patched at least one manifest, in the order
    // they were applied. It is not set when no policy patched the ManifestWork.
    ManifestWorkOverridePoliciesAnnotationKey = "work.open-cluster-management.io/override-policies"
)
```

For example, `work.open-cluster-management.io/override-policies: base,prod,prod-hotfix`. Names
are enough because policies are in the MWRS namespace.

An annotation rather than a status field, because `ManifestWork` status is written by the work
agent, while the hub controller owns the object's spec and metadata. It already sets the
`work.open-cluster-management.io/manifestworkreplicaset-owner` annotation the same way.

The annotation has to change whenever the applied policies or their order change, including
when the rendered spec does not. That happens: if `prod` is removed from an MWRS that also lists
`prod-hotfix`, and `prod-hotfix` already overrode every field `prod` set, the spec is
identical and only the provenance differs. The two drift-detection sites do not detect this
today. `ManifestWorkEqual` compares the annotation maps exactly, by length and by value, but
both sites copy the live object's `ObjectMeta` into the candidate first, so the annotations
it is given always match.

The implementation therefore compares this one key at both drift-detection sites, against
the rendered value: it must be absent when no policy applied, and equal to the rendered value
otherwise. The rest of the live metadata is still copied, because comparing the whole map
would report an annotation set by any other controller as drift. The apply path needs no
change: its candidate is built from scratch, so `ManifestWorkEqual` already sees the
difference there. A provenance-only change is applied like any other update. It does not
change the `ManifestWork`'s generation, so the cluster returns to its previous rollout status
on the next reconcile.

**On the MWRS: one condition.**

```go
const (
    // ManifestWorkReplicaSetConditionOverridesRendered reports whether the listed
    // override policies rendered for every selected cluster. It is set only when
    // spec.overridePolicies is not empty.
    ManifestWorkReplicaSetConditionOverridesRendered string = "OverridesRendered"

    // ReasonRenderFailed means the render failed for one or more clusters.
    ReasonRenderFailed = "RenderFailed"

    // ReasonOverridePolicyNotFound means a listed policy does not exist.
    ReasonOverridePolicyNotFound = "OverridePolicyNotFound"
)
```

| Status | Reason | When |
| --- | --- | --- |
| `True` | `AsExpected` (existing reason) | Every selected cluster rendered. |
| `False` | `OverridePolicyNotFound` | A listed policy does not exist. Takes precedence, since it affects every cluster. |
| `False` | `RenderFailed` | One or more clusters failed to render. |

For `RenderFailed`, the message gives the failed count and the total, then the first five
failed clusters sorted by name, each with its `RenderError`:

```yaml
- type: OverridesRendered
  status: "False"
  reason: RenderFailed
  message: '1 of 3 clusters failed: cluster-c (policy gpu-pool, manifest 0, override 0, patch 0: add operation does not apply: doc is missing path "/spec/template/spec/nodeSelector/pool")'
```

This answers "is anything failing?" from the MWRS alone, as asked on [#198][issue-198]. The
condition exists only while the controller renders overrides for the MWRS. It is not set on
an MWRS that lists no policies, so the status of existing MWRS does not change on upgrade. It
is removed when `spec.overridePolicies` becomes empty or the feature gate is turned off, so a
stale `False` cannot outlive what it describes.

#### Interaction with rollback and rollout plugins

Two implementable proposals change the same reconcile: [MWRS rollback][enh-229] and
[MWRS rollout plugins][enh-231]. Neither is implemented at the time of writing, so this
states how the pieces have to fit, not what the code does.

- **A revision is a template revision.** Rollback identifies a revision by a hash of
  `.spec.manifestWorkTemplate`. That stays as it is. Editing a policy, or changing
  `spec.overridePolicies`, creates no revision. The policies are applied, as they are at
  render time, on top of whichever template revision is being delivered.
- **Abort goes through `Render`.** Rollback's abort path applies the template stored in a
  `ControllerRevision` to the affected clusters at once. That is one more place that copies
  the template, so the single-renderer rule above binds it. Applying the stored template
  directly would strip every override from those clusters in one step.
- **Abort does not see a render failure yet.** Both proposals start an abort from clusters
  that passed `progressDeadline`: 229 when more than `maxFailures` of the current group has,
  231 when timeout clusters exist. A cluster passed in as `TimeOut` counts toward
  `maxFailures`, but the rollout handler returns it before it reaches
  `RolloutResult.ClustersTimeOut` ([rollout.go][sdk-rollout-timeout]), so a check on that
  list does not see a render failure. A progressive rollout stops, but does not abort. For an
  abort to cover render failures, the check has to read `RolloutResult.MaxFailureBreach`, or
  count render failures itself. This is open question 11.
- **An abort cannot undo a policy edit.** If a render failure does trigger an abort, the
  abort renders the previous template with the current policies. If the failure came from the new template, that fixes
  it. If it came from a policy, the same render fails again and those clusters keep the
  `ManifestWork` they have, as for any render failure. Reverting a policy is done by editing
  the policy.
- **Plugins see the rendered work.** The plugin hook `MutateManifestWork()` runs before a
  `ManifestWork` is applied. Overrides are rendered first, and the plugin receives the
  rendered object.

### Risks and Mitigation

- **`ManagedCluster` informer cost.** Memory grows with fleet size and label churn triggers
  re-render. Needs measurement at realistic fleet scale before this graduates. Mitigated by an
  informer `transform` and event filtering.
- **One policy edit changes many workloads.** A shared policy is listed by many MWRS, so
  editing it re-renders all of them. This is the point of sharing, but the blast radius is
  real. Mitigated by policies being namespaced, by provenance on each `ManifestWork`, and by
  MWRS rollout strategies applying to the change like any other.
- **A platform policy is not listed.** Policies apply only where they are listed, so an MWRS
  author can forget one. This is the cost of a declarative spec. Teams that need policies
  added or enforced automatically can use Kubernetes admission (for example a
  `MutatingAdmissionPolicy`) or their GitOps tooling until OCM support is decided in open
  question 8.
- **Silent data loss through replacement.** The reason `atomicValues` defaults to `Reject`.
  Replacing a container list and losing a sidecar, or a `nodeSelector` and losing its other
  keys, is the kind of failure that is discovered in production.
- **One strategic merge patch, two results.** A custom resource that embeds a structure a
  built-in also has can merge it differently, because the two types go through different
  engines. Mitigated by defining what each engine promises, listing the known differences
  under [Patch application](#patch-application), and pinning each with a test. Removing the
  difference is open question 9.
- **Two overlapping customisation mechanisms.** If templating lands later, users will have
  patches and templates to choose between, with a permanent documentation cost. Keeping
  templating out of this proposal limits the combined surface for now, but the interaction
  must be designed when templating is proposed.
- **A new CRD is a permanent commitment.** Install, RBAC, bundle packaging downstream,
  upgrade handling, QE matrix and support burden. This is the main argument for the
  MWRS-embedded alternative below.
- **Divergence from the `ManagedCluster` as source of truth.** A fleet-wide value such as a
  node pool name is a fact about the cluster and is already recorded on the `ManagedCluster`.
  Restating the cluster-to-value mapping inside policies duplicates it, and the duplicate can
  go stale. Templating would remove this duplication; until then, label selectors group
  clusters so that the mapping is written per distinct value, not per cluster.

## Design Details

### Open Questions

1. ~~**Where do CRD schemas come from?**~~ **Resolved.** Reading a CRD schema requires the
   CRD on the hub, but a type such as Knative may only be installed on managed clusters.
   Options discussed on [#198][issue-198]:
   a. require a `ResourceOverrideSchema` and error otherwise — no new machinery, but the
      administrator must supply the schema;
   b. add discovery on the hub — only works for types installed on the hub;
   c. collect schemas from managed clusters and keep a content-hash-deduplicated cache on the
      hub, with small per-cluster references, as proposed by @kahirokunn. This handles clusters
      whose schemas genuinely differ for the same type, but requires an agent-to-hub schema
      reporting channel that does not exist in `work` today, plus a refresh and
      garbage-collection story. That is arguably its own enhancement.

   Settled on [#198][issue-198] (2026-11-06): **alpha ships (a)** — built-in types plus
   custom resources with an explicitly registered `ResourceOverrideSchema` — and **(c) gates
   beta graduation rather than existence**.

   The disagreement was about sequencing, not destination. @kahirokunn's objection was that
   manual registration is a poor first experience and does not handle clusters whose schemas
   genuinely differ for the same type, which is correct, and (c) remains the intended normal
   path. The counter-position was that (c) is a new hub/agent contract carrying its own skew,
   refresh and garbage-collection design, so making it a prerequisite couples this proposal's
   delivery to a second and larger one. Gating graduation keeps both true: no user meets
   manual registration as the permanent answer, and neither design blocks the other. See
   [Graduation Criteria](#graduation-criteria).

2. ~~**One identifier scheme or two?**~~ **Resolved.** Group/resource identity is used
   throughout — `resourceSelectors` as `ResourceIdentifier`, and
   `ResourceOverrideSchema.spec.target` as the same group and resource without a name — per
   @kahirokunn on [#198][issue-198] and review feedback on [#155][pr-155]. The objection
   that group/resource needs hub discovery for types the hub does not have installed is
   answered by `ResourceOverrideSchema` carrying `kind` and `version`, so registering a
   schema also registers the mapping. See [Resource selection](#resource-selection).

3. **How is `forEach` added later, if at all?** Keeping `overrides` a list leaves room, but
   the loop boundary determines variable scope and execution order. Given
   `forEach: [{region: [east, west]}]` with two patches appending `--start=$region` and
   `--end=$region`, an override-level loop yields
   `["--start=east", "--end=east", "--start=west", "--end=west"]` while a patch-level loop
   yields `["--start=east", "--start=west", "--end=east", "--end=west"]`. Also undecided:
   what may drive iteration (literal lists, `ManagedCluster` data, fields in the matched
   manifest), and whether a loop variable may be used in `clusterSelector` — which in turn
   decides whether expansion happens before or after cluster matching. Recorded here so the
   templating enhancement inherits the questions rather than rediscovering them.

4. ~~**Scope of the first implementation.**~~ **Resolved.** Built-in types and
   `ResourceOverrideSchema`-backed custom resources ship together in alpha, per the same
   [#198][issue-198] agreement. This follows from question 1: with (a) as the alpha answer
   there is no reason to hold custom resources back, and splitting them would have put one
   behaviour through two rounds of review. Delivery is still expected to be several PRs.

5. ~~**Implicit application versus explicit `overridePolicies` references.**~~
   **Resolved.** Explicit references only, per @everpeace and @kahirokunn on
   [#202][pr-202-kahirokunn-declarative]. Default policies and a per-entry `disabled` field
   were in an earlier draft and were removed. See
   [Why explicit references](#why-explicit-references).

6. **Should policies be namespaced or cluster-scoped?** Proposed: **namespaced**, as
   suggested on [#198][issue-198] with no objection raised. Waiting for approver
   confirmation before this is marked resolved.
   - Authorisation follows the namespace that already governs the MWRS: anyone who can
     create an MWRS there can see and reference the policies that may change it, and nobody
     outside the namespace can change its payload.
   - It matches `placementRefs`, which also name objects in the MWRS namespace.
   - The cost: a platform team wanting a fleet-wide override creates one policy per
     namespace. A cluster-scoped kind can be added later without changing this one, if field
     experience shows that cost matters.

7. ~~**How does `clusterSelector` combine with other targeting added later?**~~
   **Resolved.** Alpha selects clusters only by `clusterSelector`. If ClusterSet or Placement
   targeting is added to `Override` later, the conditions are combined with AND: a cluster
   must match every condition set on the same override, per @kahirokunn on
   [#202][pr-202-kahirokunn].

8. **Should OCM add policies to an MWRS automatically, or enforce them?** Out of scope for
   alpha, per @kahirokunn and @everpeace on [#202][pr-202-kahirokunn-declarative]. Two
   separate needs:
   - *Default policies*: added to new MWRS so authors don't have to remember them. An
     earlier draft did this with `default: true` and a mutating webhook, and it was dropped:
     a default that MWRS can skip is not really a default, changing `default` races with
     MWRS creation, and webhook-added entries show up as GitOps drift.
   - *Required policies*: applied whether or not the MWRS lists them, for compliance. This
     is enforcement rather than convenience, and needs its own design.

   Until then, Kubernetes admission or GitOps tooling can cover both. Since policies apply
   in list order, either design also has to say where an added policy goes in the list.

9. **One merge engine for built-ins and custom resources?** Alpha uses `strategicpatch` for
   built-in types and typed merge for custom resources, and
   [Patch application](#patch-application) lists where the two differ. Running built-ins
   through the typed merge too, with their schema read from the hub's OpenAPI, would make one
   patch mean one thing on every type. The costs: the hub's OpenAPI becomes an input that has
   to be fetched and refreshed, every strategic merge directive has to be reimplemented on
   the typed path, and a built-in would no longer merge the way `kubectl patch` merges it. To
   be decided before beta, with the divergence tests as evidence of how often it matters.

10. **Should a policy carry status?** An earlier draft gave `ManifestOverridePolicy` a
    `Rendered` condition summarising every render that used it. It was removed because nothing
    could write it correctly: renders happen inside per-MWRS reconciles, and an aggregate
    needs a single writer and per-cluster results stored in a structured form, which the MWRS
    condition message is not. If field experience shows that policy authors need "is my policy
    failing anywhere?" answered on the policy itself, that takes a per-policy reconcile and a
    stored result, designed together.

11. **Should a render failure trigger an automatic abort?** As written, the abort checks in
    [229][enh-229] and [231][enh-231] read clusters past `progressDeadline`, and a render
    failure never appears there (see
    [Interaction with rollback and rollout plugins](#interaction-with-rollback-and-rollout-plugins)).
    Proposed: yes, by keying the abort on `MaxFailureBreach`, so a render failure aborts the
    same way a failing workload does. To be settled with the owners of 229 and 231, since
    it changes their check rather than this proposal's.

### Test Plan

**Unit tests** on the renderer, which is pure and the natural place for most coverage:
each patch type; ordering across the `overridePolicies` list, overrides and patches,
including that reordering the list changes which policy wins a conflict; wildcard and non-matching
resource selectors; an unresolvable manifest whose group a selector names, asserting a failure
that names the manifest, one whose group no selector names, asserting it is left unpatched,
and a selector naming a type the template does not contain, asserting no failure; cluster
selectors that match, do not match, are empty, and are omitted, asserting that the last two
both match every cluster; the identity invariant; JSON Patch `add` under a missing parent and
`remove` of a missing path, asserting errors; rejection of a discarded atomic list, map and
struct, the explicit `Replace` opt-in, and an atomic value set where the manifest has none or
an empty one, asserting it is allowed under `Reject`; composite
`x-kubernetes-list-map-keys`; `ResourceOverrideSchema` overlay precedence over the CRD schema;
two schemas sharing a uniqueness key, asserting that the render fails naming both rather than
picking one, and that a second schema differing only in `spec.version` is accepted; an overlay
carrying the legacy `x-kubernetes-patch-strategy` pair, asserting a validation error that
names the structural equivalent rather than a silently atomic list; a managed-cluster-only
type whose registered schema omits a list the patch touches, asserting that the deduced
fallback makes it atomic and that `atomicValues: Reject` therefore fails with the path named,
rather than replacing the list; determinism, by rendering
twice and comparing bytes. [ocm#1792][ocm-1792] already carries this suite for the agreed
subset.

Four cases deserve naming because they guard against silent divergence rather than an
outright bug. **Deletion parity**: the same strategic merge patch using `null` and
`$patch: delete` is applied to a built-in and to a custom resource, and the two results must
agree — this is the gap between `strategicpatch` and typed merge described above. **JSON
Patch body shape**: a policy whose `patch` is a JSON array round-trips through the CRD
schema, which is what `apiextensionsv1.JSON` buys over `runtime.RawExtension`. **Documented
divergence**: each row of the table under [Patch application](#patch-application) is applied
to a built-in and to a custom resource, and both results are asserted, so a change in either
engine shows up as a test failure and not as a changed payload. **`null` round trip**: a
policy whose patch contains a `null` is created, then updated by client-side apply, by
server-side apply and by replace, recording which write paths keep the `null`.

**Integration tests** on the controller: that a policy change re-renders existing
`ManifestWork`s; that the two drift-detection sites agree with the rendered object, so no
churn occurs — a regression test for the failure mode described above; that a render failure
leaves that cluster's `ManifestWork` unchanged and does not change the payload rendered for
any other cluster; that under `All` the other clusters still receive the new payload; that
under `Progressive` a render-failed cluster counts toward `maxFailures` at once, does not hold
a `maxConcurrency` slot, and with the default `maxFailures` stops the rollout from reaching
further clusters; that a render whose result exceeds the manifest limit fails for that
cluster with the size in the message, and no `ManifestWork` update is sent; that a selected cluster with no `ManagedCluster` fails to render instead of
rendering without labels; that
`ManagedCluster` label changes re-render; that creating, editing and deleting a
`ResourceOverrideSchema` re-renders the `ManifestWork`s whose merge result it changes; that
a hub CRD whose list changes from `map` to `atomic` re-renders the affected MWRS and fails
under `atomicValues: Reject`, while a change to an unrelated CRD enqueues nothing; that
a hub CRD becoming `Established` retries a JSON Patch render that failed on an unresolvable
type, and deleting it re-renders the MWRS; that
an MWRS is patched only by the policies it lists, while a sibling MWRS in the same
namespace that does not list the policy renders the plain template; that removing a reference
re-renders without that policy; that the `override-policies` annotation lists the applied
policies in order and is removed when none apply; that removing a policy whose fields a later
policy already overrode, so that the rendered spec is unchanged, still updates the annotation
— a regression test for the comparison rule under [Observability](#observability); that
`OverridesRendered` is `False` with `RenderFailed` when one cluster fails and `True` again
once it renders, that `ManifestworkApplied` is `NotAsExpected` meanwhile, and that the
condition is removed when the list is emptied; that a missing referenced policy sets
`OverridePolicyNotFound`, leaves the existing `ManifestWork`s unchanged, still deletes the
work of a cluster the placement dropped, creates none for a newly selected cluster, and
reports `PlacementRolledOut` as `Progressing`.

**API and webhook tests**: the API server rejects duplicate names in `overridePolicies`, from
the list-map marker and with no webhook involved, and a name that does not exist yet is
accepted. With the `ManifestOverridePolicy` gate off, the validating webhook
rejects a create with entries and an update that adds one, and allows an update that keeps or
removes entries. A `ResourceOverrideSchema` whose `target` has only `group` and `resource`,
as in the example, is accepted. The policy webhook rejects a `JSONPatch` body that is not an
array of operations, a merge or strategic merge body that is not an object, a
`clusterSelector` that does not parse, and a patch body over 64 KiB; the API server rejects a
list over its `MaxItems` bound.

**E2E tests**: a `ManifestWorkReplicaSet` plus a `ManifestOverridePolicy` producing different
payloads on two managed clusters, including a custom resource patched with a registered
`ResourceOverrideSchema`.

**Scale**: measure hub memory and reconcile latency with the `ManagedCluster` informer at
realistic fleet size, with and without the informer `transform`, before graduation.

### Graduation Criteria

#### Alpha

- `ManifestOverridePolicy` served at `v1alpha1` behind the `ManifestOverridePolicy` feature
  gate, off by default (see [Feature gate](#feature-gate)).
- Renderer used by every MWRS site that copies the template; unit and integration tests above
  passing.
- Per-cluster failure reporting and provenance implemented.
- `ResourceOverrideSchema` and custom-resource strategic merge implemented, backed by
  explicitly registered schemas (open question 1, option a).

#### Beta

- Schema collection from managed clusters, with the content-hash-deduplicated hub cache of
  open question 1 option c, implemented and serving as the normal path. Explicit
  `ResourceOverrideSchema` registration remains available as an override.
- E2E coverage on a real multi-cluster setup.
- Scale measurements published for the `ManagedCluster` informer.
- Decisions on open questions 9 (one merge engine) and 10 (policy status).
- Feature gate on by default.

#### GA

- No API changes for one release.
- User-facing documentation on the OCM website.
- A decision on open question 8, based on field experience with explicit references.

### Upgrade / Downgrade Strategy

Both new resources are optional, and the change to an existing API — one optional field on
`ManifestWorkReplicaSetSpec` — keeps today's behaviour when it is unset: an MWRS with no
`overridePolicies` renders the plain template. Nothing changes on upgrade until a user lists
a policy on an MWRS.

On downgrade, or if the feature gate is turned off, rendering reverts to the plain template;
the next reconcile rewrites each `ManifestWork` back to the un-overridden payload, which is a
real workload change and must be documented. With the gate off, the same reconcile removes
the `override-policies` annotation and the `OverridesRendered` condition. A downgrade to a
release without the feature leaves the annotation on existing `ManifestWork`s, because the
older controller does not know the key; it is stale but harmless. Removing the references
before downgrading produces the same result through the normal reconcile path, with rollout
strategies applied, and is the recommended procedure.

Downgrade also drops `overridePolicies` from stored objects, so
re-enabling the feature after a downgrade does not bring the references back. Users must
re-apply them, which is one more reason to remove them deliberately first.

### Version Skew Strategy

Rendering happens entirely on the hub, and the agent receives an ordinary `ManifestWork`. An
`n-2` klusterlet that knows nothing about this feature is unaffected. Option (c) in open
question 1, which gates beta, will introduce a genuine hub/agent contract and will need its
own skew analysis.

## Implementation History

- 2026-10-07: [enhancements#198][issue-198] opened by @kahirokunn, from
  [ocm#1175][ocm-1175].
- 2026-10-13 to 2026-10-15: discussion converges on a separate override resource rather than
  fields on MWRS; @mikeshng rejects attaching overrides to `Placement` on the grounds that
  `Placement` must stay workload-agnostic.
- 2026-10-22 to 2026-10-23: implicit application, priority ordering, per-cluster failure
  isolation and provenance agreed; templating and `forEach` moved out of scope.
- 2026-10-30: renderer prototype opened as [ocm#1792][ocm-1792].
- 2026-11-02: atomic-list rejection agreed as the default; `ResourceOverrideSchema` and the
  cross-cluster schema cache proposed; this proposal opened.
- 2026-11-03: identifier question resolved on `ResourceIdentifier` throughout, with
  `ResourceOverrideSchema` carrying the kind/version mapping so the hub can resolve types it
  does not have installed. Patch body changed from `runtime.RawExtension` to
  `apiextensionsv1.JSON` so JSON Patch arrays pass CRD validation. Deletion-semantics
  divergence between `strategicpatch` and typed merge identified and specified away. A
  `ResourceOverrideSchema` watch added to the controller wiring.
- 2026-11-11: `ResourceOverrideSchema` uniqueness defined; structural list markers required;
  effective-schema resolution defined for types the hub does not have; `overrideMode` opt-out
  added.
- 2026-11-13 to 2026-11-16: @everpeace shows a per-workload case that an all-or-nothing
  opt-out cannot express; with @kahirokunn, the proposal moves to an explicit
  `spec.overridePolicies` list with `default: true` policies added at creation, replacing
  implicit application and `overrideMode`. @kahirokunn assigned as approver.
- 2026-11-20: hub `CustomResourceDefinition` watch added to the controller wiring, since hub
  CRDs drive both resource mapping and the effective schema. Open question 7 added.
- 2026-11-20: @kahirokunn confirms the separate policy resource and strategic merge in alpha,
  resolves open question 7 with AND, and suggests a per-entry `disabled` field in place of
  `defaultOverridePolicies: Add | Skip`, so an author can skip one default and keep the rest.
- 2026-11-23: status types defined for both new resources; the `OverridesRendered` MWRS
  condition and the `override-policies` ManifestWork annotation specified; the
  `ManifestOverridePolicy` feature gate named, with webhook and controller behaviour when it
  is off; a proposed answer recorded for open question 6.
- 2026-11-25: @everpeace asks whether a default that MWRS can skip is really a default;
  @kahirokunn agrees and decides to keep alpha declarative. `default`, `disabled` and the
  mutating webhook are removed; MWRS apply exactly the policies they list. Default and
  required policies become open question 8.
- 2026-11-26: review against `api`, `ocm` and `sdk-go` at `main`.
  - Statements about current code corrected: the controller's informers, the gate-off
    webhook, `resourceMatch`, and what `ManifestWorkEqual` compares. The `override-policies`
    annotation is compared exactly.
  - A render failure is reported to the rollout as `TimeOut`, and its effect under each
    rollout strategy is stated. A missing policy and a missing `ManagedCluster` use the same
    path.
  - `ResourceOverrideSchema.spec.target` gets a group/resource-only type. The `atomicLists`
    field becomes `atomicValues` and covers atomic maps and structs. The unresolvable-type
    rule is restated from the manifest's group.
  - Known differences between the two merge engines, the `tolerations` limitation and the
    loss of `null` on merge-patch updates are documented. Story 3 no longer adds a toleration.
  - The policy `Rendered` condition is removed (open question 10). The hub CRD trigger moves
    to the `Established` condition. Interaction with rollback and rollout plugins added.
  - A rendered work over the manifest size limit is a render failure. The policy webhook
    checks, `MaxItems` bounds and RBAC are defined. Whether a render failure triggers an
    abort becomes open question 11.
- 2026-11-27: `priority` removed, per @everpeace and @kahirokunn. Policies apply in the order
  `spec.overridePolicies` lists them.

## Drawbacks

The strongest argument against this proposal is cost against benefit. It adds two CRDs, a
second cluster-selection mechanism alongside `Placement`, and a `ManagedCluster` informer in
the work hub — and it still does not solve the per-copy-uniqueness need that started
[ocm#1175][ocm-1175], which needs the templating enhancement. A reviewer could reasonably
hold that the narrower MWRS-embedded alternative below delivers most of the value for a
fraction of the permanent surface, and that users who need more should use a GitOps tool.

The counter-argument is that overrides are genuinely shared across workloads — the point
@everpeace raised — and that a field inside MWRS cannot express sharing without being
restated in every MWRS. A separate resource that each MWRS references by name gives sharing
while keeping each MWRS's spec complete.

## Alternatives

### Overrides embedded in MWRS, keyed by clusterSet or placement

This is the direction of [#155][pr-155] and of review feedback on it, which suggested an
extensible `overrides` field with an `overrideStrategy` selecting between `clustersets` and
`placements`:

```yaml
overrides:
  overrideStrategy: placements
  placements:
    - name: aws-placement
      patch: ...
    - name: gke-placement
      patch: ...
```

*For:* near-zero cost — the reconciler already iterates `placementRefs` and holds the
`Placement` per iteration. No new CRD, no second selection mechanism, no new informer if
keyed by placement. Fields on an API that is still `v1alpha1`.

*Against:* overrides cannot be shared across `ManifestWorkReplicaSet`s; the same patch has
to be copied into each one.

This alternative was not chosen because of the sharing argument, even though it is materially
cheaper. @kahirokunn, as approver, confirmed the separate resource on
[#202][pr-202-kahirokunn].

### Implicit application with an `overrideMode` opt-out

The earlier draft of this proposal: every policy in the namespace applies to every MWRS,
and `overrideMode: Disabled` on an MWRS turns all of them off. Rejected because
all-or-nothing cannot express Story 4, and a per-policy deny list to fix that would make every
new policy reach every existing MWRS until each one denies it. It also hides from the MWRS
spec which policies apply to it.

### Overrides referenced from Placement

@everpeace's original sketch added an `overridePolicies` list to `Placement`, with MWRS
continuing to reference only the `Placement`. Rejected because `Placement` is used for things
other than workload distribution — selecting clusters to audit or observe, for example — and
coupling it to workload concerns narrows it.

### Templating instead of patches

Make patch values reference cluster data: `{{CLUSTER_NAME}}`, or CEL over the `ManagedCluster`
object. This addresses both needs in the motivation and removes the need to enumerate
clusters or distinct values anywhere, which is the property that fails at fleet scale. OCM has
precedent: `pkg/addon/templateagent` substitutes `{{CLUSTER_NAME}}` with `fasttemplate`, and
the policy framework offers Go templating with `lookup` and `fromClusterClaim`.

Not an alternative so much as a complement, and the thread agreed to sequence it separately.
It is listed here because it weakens the case for sharing: much of what teams would want to
share is precisely a cluster-to-value mapping, and the architecturally correct home for that
is the `ManagedCluster` object rather than a list that restates it.

### Do nothing; direct users to GitOps

`ApplicationSet` over `PlacementDecision`s already solves per-cluster customisation for users
on Argo CD. Zero cost, and it genuinely serves a large population. It does not help anyone
using MWRS directly, which is the population MWRS exists for — more relevant now that the MWRS
feature gate is enabled by default.

## References

- [enhancements#198 — Feature request: Support cluster-specific resource overrides][issue-198]
- [enhancements#155 — Add proposal for cluster set level override in MWRS][pr-155]
- [ocm#1175 — the originating request][ocm-1175]
- [ocm#1792 — renderer prototype][ocm-1792]
- [Support Rollback for ManifestWorkReplicaSet][enh-229]
- [ManifestWorkReplicaSet Rollout Plugin][enh-231]
- [Karmada OverridePolicy](https://karmada.io/docs/userguide/scheduling/override-policy/)
- [Kustomize custom OpenAPI schema][kustomize-openapi]

[issue-198]: https://github.com/open-cluster-management-io/enhancements/issues/198
[pr-202]: https://github.com/open-cluster-management-io/enhancements/pull/202
[pr-202-everpeace]: https://github.com/open-cluster-management-io/enhancements/pull/202#issuecomment-6863619311
[pr-202-kahirokunn]: https://github.com/open-cluster-management-io/enhancements/pull/202#issuecomment-7042250849
[pr-202-kahirokunn-declarative]: https://github.com/open-cluster-management-io/enhancements/pull/202#issuecomment-7165932619
[pr-202-priority]: https://github.com/open-cluster-management-io/enhancements/pull/202#discussion_r4697379043
[pr-155]: https://github.com/open-cluster-management-io/enhancements/pull/155
[ocm-1175]: https://github.com/open-cluster-management-io/ocm/issues/1175
[ocm-1792]: https://github.com/open-cluster-management-io/ocm/pull/1792
[enh-229]: ../229-manifestworkreplicaset-rollback/README.md
[ocm-manifest-limit]: https://github.com/open-cluster-management-io/ocm/blob/3ee15b8960998361a429c399337d33327bad33f1/pkg/work/webhook/common/validator.go#L19
[sdk-rollout-timeout]: https://github.com/open-cluster-management-io/sdk-go/blob/1cf5ae6e53dc5cef2d7ccdb6d01a5c3b4eae46d3/pkg/apis/cluster/v1alpha1/rollout.go#L484
[enh-231]: ../231-manifestworkreplicaset-rollout-plugin/README.md
[kustomize-openapi]: https://github.com/kubernetes-sigs/kustomize/blob/master/examples/customOpenAPIschema.md
