# reconciler-runtime <!-- omit in toc -->

![CI](https://github.com/vmware-labs/reconciler-runtime/workflows/CI/badge.svg?branch=main)
[![GoDoc](https://godoc.org/github.com/vmware-labs/reconciler-runtime?status.svg)](https://godoc.org/github.com/vmware-labs/reconciler-runtime)
[![Go Report Card](https://goreportcard.com/badge/github.com/vmware-labs/reconciler-runtime)](https://goreportcard.com/report/github.com/vmware-labs/reconciler-runtime)
[![codecov](https://codecov.io/gh/vmware-labs/reconciler-runtime/branch/main/graph/badge.svg)](https://codecov.io/gh/vmware-labs/reconciler-runtime)

`reconciler-runtime` is an opinionated framework for authoring and testing Kubernetes reconcilers using [`controller-runtime`](https://github.com/kubernetes-sigs/controller-runtime) project. `controller-runtime` provides infrastructure for creating and operating controllers, but provides little support for the business logic of implementing a reconciler within the controller. The [`Reconciler` interface](https://pkg.go.dev/sigs.k8s.io/controller-runtime/pkg/reconcile#Reconciler) provided by `controller-runtime` is the handoff point with `reconciler-runtime`.

<!-- ToC managed by https://marketplace.visualstudio.com/items?itemName=yzhang.markdown-all-in-one -->
- [Reconcilers](#reconcilers)
	- [ParentReconciler](#parentreconciler)
	- [SubReconciler](#subreconciler)
		- [SyncReconciler](#syncreconciler)
		- [ChildReconciler](#childreconciler)
	- [Higher-order Reconcilers](#higher-order-reconcilers)
		- [CastParent](#castparent)
		- [Sequence](#sequence)
		- [WithConfig](#withconfig)
		- [WithFinalizer](#withfinalizer)
- [Testing](#testing)
	- [ReconcilerTestSuite](#reconcilertestsuite)
	- [SubReconcilerTestSuite](#subreconcilertestsuite)
- [Utilities](#utilities)
	- [Config](#config)
	- [Stash](#stash)
	- [Tracker](#tracker)
	- [Status](#status)
	- [Finalizers](#finalizers)
- [Contributing](#contributing)
- [Acknowledgements](#acknowledgements)
- [License](#license)

## Reconcilers

### ParentReconciler

A [`ParentReconciler`](https://pkg.go.dev/github.com/vmware-labs/reconciler-runtime/reconcilers#ParentReconciler) is responsible for orchestrating the reconciliation of a single resource. The reconciler delegates the manipulation of other resources to SubReconcilers.

The parent is responsible for:
- fetching the resource being reconciled
- creating a stash to pass state between sub reconcilers
- passing the resource to each sub reconciler in turn
- initialize conditions on the status by calling status.InitializeConditions() if defined
- normalizing the .status.conditions[].lastTransitionTime for status conditions that are metav1.Condition (the previous timestamp is preserved if the condition is otherwise unchanged)
- reflects the observed generation on the status
- updates the resource status if it was modified
- logging the reconcilers activities
- records events for mutations and errors

The implementor is responsible for:
- defining the set of sub reconcilers

**Example:**

Parent reconcilers tend to be quite simple, as they delegate their work to sub reconcilers. We'll use an example from projectriff of the Function resource, which uses Kpack to build images from a git repo. In this case the FunctionTargetImageReconciler resolves the target image for the function, and FunctionChildImageReconciler creates a child Kpack Image resource based on the resolve value. 

```go
func FunctionReconciler(c reconcilers.Config) *reconcilers.ParentReconciler {
	return &reconcilers.ParentReconciler{
		Name: "Function",
		Type: &buildv1alpha1.Function{},
		Reconciler: reconcilers.Sequence{
			FunctionTargetImageReconciler(c),
			FunctionChildImageReconciler(c),
		},

		Config: c,
	}
}
```
[full source](https://github.com/projectriff/system/blob/4c3b75327bf99cc37b57ba14df4c65d21dc79d28/pkg/controllers/build/function_reconciler.go#L39-L51)

**Recommended RBAC:**

Replace `<group>` and `<resource>` with values for the parent type.

```go
// +kubebuilder:rbac:groups=<group>,resources=<resource>,verbs=get;list;watch;create;update;patch;delete
// +kubebuilder:rbac:groups=<group>,resources=<resource>/status,verbs=get;update;patch
// +kubebuilder:rbac:groups=core,resources=events,verbs=get;list;watch;create;update;patch;delete
```

or

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: # any name that is bound to the ServiceAccount used by the client
rules:
- apiGroups: ["<group>"]
  resources: ["<resource>"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
- apiGroups: ["<group>"]
  resources: ["<resource>/status"]
  verbs: ["get", "update", "patch"]
- apiGroups: ["core"]
  resources: ["events"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
```

### SubReconciler

The [`SubReconciler`](https://pkg.go.dev/github.com/vmware-labs/reconciler-runtime/reconcilers#SubReconciler) interface defines the contract between the parent and sub reconcilers.

#### SyncReconciler

The [`SyncReconciler`](https://pkg.go.dev/github.com/vmware-labs/reconciler-runtime/reconcilers#SyncReconciler) is the minimal type-aware sub reconciler. It is used to manage a portion of the parent's reconciliation that is custom, or whose behavior is not covered by another sub reconciler type. Common uses include looking up reference data for the reconciliation, or controlling resources that are not kubernetes resources.

When a resource is deleted that has pending finalizers, the Finalize method is called instead of the Sync method. If the SyncDuringFinalization field is true, the Sync method will also by called. If creating state that must be manually cleaned up, it is the users responsibility to define and clear finalizers. Using the [parent finalizer helper methods](#finalizers) is strongly encouraged with working under a [ParentReconciler](#parentreconciler).

**Example:**

While sync reconcilers have the ability to do anything a reconciler can do, it's best to keep them focused on a single goal, letting the parent reconciler structure multiple sub reconcilers together. In this case, we use the parent resource and the client to resolve the target image and stash the value on the parent's status. The status is a good place to stash simple values that can be made public. More [advanced forms of stashing](#stash) are also available. Learn more about [status and its contract](#status).

```go
func FunctionTargetImageReconciler(c reconcilers.Config) reconcilers.SubReconciler {
	return &reconcilers.SyncReconciler{
		Name: "TargetImage",
		Sync: func(ctx context.Context, parent *buildv1alpha1.Function) error {
			log := logr.FromContextOrDiscard(ctx)

			targetImage, err := resolveTargetImage(ctx, c.Client, parent)
			if err != nil {
				return err
			}
			parent.Status.MarkImageResolved()
			parent.Status.TargetImage = targetImage
			return nil
		},
	}
}
```
[full source](https://github.com/projectriff/system/blob/4c3b75327bf99cc37b57ba14df4c65d21dc79d28/pkg/controllers/build/function_reconciler.go#L53-L74)

#### ChildReconciler

The [`ChildReconciler`](https://pkg.go.dev/github.com/vmware-labs/reconciler-runtime/reconcilers#ChildReconciler) is a sub reconciler that is responsible for managing a single controlled resource. A developer defines their desired state for the child resource (if any), and the reconciler creates/updates/deletes the resource to match the desired state. The child resource is also used to update the parent's status. Mutations and errors are recorded for the parent.

The ChildReconciler is responsible for:
- looking up an existing child
- creating/updating/deleting the child resource based on the desired state
- setting the owner reference on the child resource (when not using a finalizer)
- logging the reconcilers activities
- recording child mutations and errors for the parent resource
- adapting to child resource changes applied by mutating webhooks
- adding and clearing of a finalizer, if specified

The implementor is responsible for:
- defining the desired resource
- indicating if two resources are semantically equal
- merging the actual resource with the desired state (often as simple as copying the spec and labels)
- updating the parent's status from the child
- defining the status subresource [according to the contract](#status) 

When a finalizer is defined, the parent resource is patched to add the finalizer before creating the child; it is removed after the child is deleted. If the parent resource is pending deletion, the desired child method is not called, and existing children are deleted.

Using a finalizer means that the child resource will not use an owner reference. The OurChild method must be implemented in a way that can uniquely and unambiguously identify the child that this parent resource is responsible for from any other resources of the same kind. The child resource is tracked explicitly.

> Warning: It is crucial that each ChildReconciler using a finalizer have a unique and stable finalizer name. Two reconcilers that use the same finalizer, or a reconciler that changed the name of its finalizer, may leak the child resource when the parent is deleted, or the parent resource may never terminate.

**Example:**

Now it's time to create the child Image resource that will do the work of building our Function. This reconciler looks more more complex than what we have seen so far, each function on the reconciler provides a focused hook into the lifecycle being orchestrated by the ChildReconciler.

```go
func FunctionChildImageReconciler(c reconcilers.Config) reconcilers.SubReconciler {
	return &reconcilers.ChildReconciler{
		Name:          "ChildImage",
		ChildType:     &kpackbuildv1alpha1.Image{},
		ChildListType: &kpackbuildv1alpha1.ImageList{},

		DesiredChild: func(ctx context.Context, parent *buildv1alpha1.Function) (*kpackbuildv1alpha1.Image, error) {
			if parent.Spec.Source == nil {
				// don't create an Image, and delete any existing Image
				return nil, nil
			}

			child := &kpackbuildv1alpha1.Image{
				ObjectMeta: metav1.ObjectMeta{
					Labels: reconcilers.MergeMaps(parent.Labels, map[string]string{
						buildv1alpha1.FunctionLabelKey: parent.Name,
					}),
					Annotations:  make(map[string]string),
					// Name or GenerateName are supported
					GenerateName: fmt.Sprintf("%s-function-", parent.Name),
					Namespace:    parent.Namespace,
				},
				Spec: kpackbuildv1alpha1.ImageSpec{
					Tag: parent.Status.TargetImage, // value set by sync reconciler
					// ... abbreviated
				},
			}

			return child, nil
		},
		SemanticEquals: func(r1, r2 *kpackbuildv1alpha1.Image) bool {
			// if the two resources are semantically equal, then we don't need
			// to update the server
			return equality.Semantic.DeepEqual(r1.Spec, r2.Spec) &&
				equality.Semantic.DeepEqual(r1.Labels, r2.Labels)
		},
		MergeBeforeUpdate: func(actual, desired *kpackbuildv1alpha1.Image) {
			// mutate actual resource with desired state
			actual.Labels = desired.Labels
			actual.Spec = desired.Spec
		},
		ReflectChildStatusOnParent: func(parent *buildv1alpha1.Function, child *kpackbuildv1alpha1.Image, err error) {
			// child is the value of the freshly created/updated/deleted child
			// resource as returned from the api server

			// If a fixed desired resource name is used instead of a generated
			// name, check if the err is because the resource already exists.
			// The ChildReconciler will not claim ownership of another resource.
			//
			// See https://github.com/projectriff/system/blob/1fcdb7a090565d6750f9284a176eb00a3fe14663/pkg/controllers/core/deployer_reconciler.go#L277-L283

			if child == nil {
				// image was deleted
				parent.Status.LatestImage = parent.Status.TargetImage
				parent.Status.MarkBuildNotUsed()
			} else {
				// image was created/updated/unchanged
				parent.Status.KpackImageRef = refs.NewTypedLocalObjectReferenceForObject(child, c.Scheme)
				parent.Status.LatestImage = child.Status.LatestImage
				parent.Status.PropagateKpackImageStatus(&child.Status)
			}
		},
		Sanitize: func(child *kpackbuildv1alpha1.Image) interface{} {
			// log only the resources spec. If the resource contained sensitive
			// values (like a Secret) we'd remove them here so they don't end
			// up in our logs
			return child.Spec
		},
	}
}
```
[full source](https://github.com/projectriff/system/blob/4c3b75327bf99cc37b57ba14df4c65d21dc79d28/pkg/controllers/build/function_reconciler.go#L76-L151)

**Recommended RBAC:**

Replace `<group>` and `<resource>` with values for the child type.

```go
// +kubebuilder:rbac:groups=<group>,resources=<resource>,verbs=get;list;watch;create;update;patch;delete
```

or

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: # any name that is bound to the ServiceAccount used by the client
rules:
- apiGroups: ["<group>"]
  resources: ["<resource>"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
```


### Higher-order Reconcilers

Higher order reconcilers are SubReconcilers that do not perform work directly, but instead compose other SubReconcilers in new patterns.

#### CastParent

A [`CastParent`](https://pkg.go.dev/github.com/vmware-labs/reconciler-runtime/reconcilers#CastParent) casts the ParentReconciler's type by projecting the resource data onto a new struct. Casting the parent resource is useful to create cross cutting reconcilers that can operate on common portion of multiple parent resources, commonly referred to as a duck type.

JSON encoding is used as the intermediate representation. Operations on a cast parent are read-only. Attempts to mutate the parent will result in the reconciler erring, although read/write support may be added in the future.

**Example:**

```go
func FunctionReconciler(c reconcilers.Config) *reconcilers.ParentReconciler {
	return &reconcilers.ParentReconciler{
		Name: "Function",
		Type: &buildv1alpha1.Function{},
		Reconciler: reconcilers.Sequence{
			&reconcilers.CastParent{
				Type: &duckv1alpha1.ImageRef{},
				Reconciler: &reconcilers.SyncReconciler{
					Sync: func(ctx context.Context, parent *duckv1alpha1.ImageRef) error {
						// do something with the duckv1alpha1.ImageRef instead of a buildv1alpha1.Function
						return nil
					},
				},
			},
			FunctionChildImageReconciler(c),
		},

		Config: c,
	}
}
```

#### Sequence

A [`Sequence`](https://pkg.go.dev/github.com/vmware-labs/reconciler-runtime/reconcilers#Sequence) composes multiple SubReconcilers as a single SubReconciler. Each sub reconciler is called in turn, aggregating the result of each sub reconciler. A reconciler returning an error will interrupt the sequence.

**Example:**

A Sequence is commonly used in a ParentReconciler, but may be used anywhere a SubReconciler is accepted. 

```go
func FunctionReconciler(c reconcilers.Config) *reconcilers.ParentReconciler {
	return &reconcilers.ParentReconciler{
		Name: "Function",
		Type: &buildv1alpha1.Function{},
		Reconciler: reconcilers.Sequence{
			FunctionTargetImageReconciler(c),
			FunctionChildImageReconciler(c),
		},

		Config: c,
	}
}
```
[full source](https://github.com/projectriff/system/blob/4c3b75327bf99cc37b57ba14df4c65d21dc79d28/pkg/controllers/build/function_reconciler.go#L39-L51)

#### WithConfig

[`WithConfig`](https://pkg.go.dev/github.com/vmware-labs/reconciler-runtime/reconcilers#WithConfig) overrides the config that nested reconcilers consume. The config can be retrieved from the context via [`RetrieveConfig`](https://pkg.go.dev/github.com/vmware-labs/reconciler-runtime/reconcilers#RetrieveConfig). The config used to load the parent resource should be used for interactions with the parent resource, which can be retrieved from the context via [`RetrieveParentConfig`](https://pkg.go.dev/github.com/vmware-labs/reconciler-runtime/reconcilers#RetrieveParentConfig).

**Example:**

`WithConfig` can be used to change the REST Config backing the clients. This could be to make requests to the same cluster with a user defined service account, or target an entirely different Kubernetes cluster.

```go
func SwapRESTConfig(rc *rest.Config) *reconcilers.SubReconciler {
	return &reconcilers.WithConfig{
		Reconciler: reconcilers.Sequence{
			LookupReferenceDataReconciler(),
			DoSomethingChildReconciler(),
		},

		Config: func(ctx context.Context, c reconciler.Config) (reconciler.Config, error ) {
			// the rest config could also be stashed from a lookup in a SyncReconciler based on a dynamic value
			cl, err := clusters.New(rc)
			if err != nil {
				return reconciler.Config{}, err
			}
			return c.WithCluster(cl), nil
		}
	}
}
```

#### WithFinalizer

[`WithFinalizer`](https://pkg.go.dev/github.com/vmware-labs/reconciler-runtime/reconcilers#WithFinalizer) allows external state to be allocated and then cleaned up once the parent resource is deleted. When the parent resource is not terminating, the finalizer is set on the parent resource before the nested reconciler is called. When the parent resource is terminating, the finalizer is cleared only after the nested reconciler returns without an error.

The [Finalizers](#finalizers) utilities are used to manage the finalizer on the parent resource.

> Warning: It is crucial that each WithFinalizer have a unique and stable finalizer name. Two reconcilers that use the same finalizer, or a reconciler that changed the name of its finalizer, may leak the external state when the parent is deleted, or the parent resource may never terminate.

**Example:**

`WithFinalizer` can be used to wrap any other [SubReconciler](#subreconciler), which can then safely allocate external state while the parent resource is not terminating, and then cleanup that state once the parent resource is terminating.

```go
func SyncExternalState() *reconcilers.SubReconciler {
	return &reconcilers.WithFinalizer{
		Finalizer: "unique.finalizer.name"
		Reconciler: &reconcilers.SyncReconciler{
			Sync: func(ctx context.Context, parent *resources.TestResource) error {
				// allocate external state
				return nil
			},
			Finalize: func(ctx context.Context, parent *resources.TestResource) error {
				// cleanup the external state
				return nil
			},
		},
	}
}
```

## Testing

While `controller-runtime` focuses its testing efforts on integration testing by spinning up a new API Server and etcd, `reconciler-runtime` focuses on unit testing reconcilers. The state for each test case is pure, preventing side effects from one test case impacting the next.

The table test pattern is used to declare each test case in a test suite with the resource being reconciled, other given resources in the cluster, and all expected resource mutations (create, update, delete).

The tests make extensive use of given and mutated resources. It is recommended to use a library like [dies](https://dies.dev) to reduce boilerplate code and to highlight the delta unique to each test.

There are two test suites, one for reconcilers and an optimized harness for testing sub reconcilers.

### ReconcilerTestSuite

[`ReconcilerTestCase`](https://pkg.go.dev/github.com/vmware-labs/reconciler-runtime/testing#ReconcilerTestCase) run the full reconciler via the controller runtime Reconciler's Reconcile method.

```go
testKey := ... // NamesapcedName of the resource to reconcile
inMemoryGatewayImagesConfigMap := ... // ConfigMap with images
inMemoryGateway := ... // resource to reconcile
gatewayCreate := ... // expected to be created
scheme := ... // scheme registered with all resource types the reconcile interacts with

rts := rtesting.ReconcilerTestSuite{{
	...
}, {
	Name: "creates gateway",
	Key:  testKey,
	GivenObjects: []client.Object{
		inMemoryGateway,
		inMemoryGatewayImagesConfigMap,
	},
	ExpectTracks: []client.Object{
		rtesting.NewTrackRequest(inMemoryGatewayImagesConfigMap, inMemoryGateway, scheme),
	},
	ExpectEvents: []rtesting.Event{
		rtesting.NewEvent(inMemoryGateway, scheme, corev1.EventTypeNormal, "Created",
			`Created Gateway "%s"`, testName),
		rtesting.NewEvent(inMemoryGateway, scheme, corev1.EventTypeNormal, "StatusUpdated",
			`Updated status`),
	},
	ExpectCreates: []client.Object{
		gatewayCreate,
	},
	ExpectStatusUpdates: []client.Object{
		// example using an https://dies.dev style die to mutate the resource
		inMemoryGateway.
			StatusDie(func(d *diestreamingv1alpha1.InMemoryGatewayStatusDie) {
				d.ObservedGeneration(1)
				d.ConditionsDie(
					// the condition will be unknown since the child resource
					// was just created and hasn't been reconciled by its
					// controller yet
					inMemoryGatewayConditionGatewayReady.Unknown(),
					inMemoryGatewayConditionReady.Unknown(),
				)
			}),
	},
}, {
	...
}}

rts.Test(t, scheme, func(t *testing.T, rtc *rtesting.ReconcilerTestCase, c reconcilers.Config) reconcile.Reconciler {
	return streaming.InMemoryGatewayReconciler(c, testSystemNamespace)
})
```
[full source](https://github.com/projectriff/system/blob/4c3b75327bf99cc37b57ba14df4c65d21dc79d28/pkg/controllers/streaming/inmemorygateway_reconciler_test.go#L142-L169)

### SubReconcilerTestSuite

For more complex reconcilers, the number of moving parts can make it difficult to fully cover all aspects of the reonciler and handle corner cases and sources of error. The [`SubReconcilerTestCase`](https://pkg.go.dev/github.com/vmware-labs/reconciler-runtime/testing#SubReconcilerTestCase) enables testing a single sub reconciler in isolation from the parent. While very similar to ReconcilerTestCase, these are the differences:

- `Key` is replaced with `Parent` since the parent resource is not lookedup, but handed to the reconciler. `ExpectParent` is the mutated value of the parent resource after the reconciler runs.
- `GivenStashedValues` is a map of stashed value to seed, `ExpectStashedValues` are individually compared with the actual stashed value after the reconciler runs.
- `ExpectStatusUpdates` is not available

**Example:**

Like with the tracking example, the processor reconciler in projectriff also looks up images from a ConfigMap. The sub reconciler under test is responsible for tracking the ConfigMap, loading and stashing its contents. Sub reconciler tests make it trivial to test this behavior in isolation, including error conditions.

```go
processor := ...
processorImagesConfigMap := ...

rts := rtesting.SubReconcilerTestSuite{
	{
		Name:   "missing images configmap",
		Parent: processor,
		ExpectTracks: []rtesting.TrackRequest{
			rtesting.NewTrackRequest(processorImagesConfigMap, processor, scheme),
		},
		ShouldErr: true,
	},
	{
		Name:   "stash processor image",
		Parent: processor,
		GivenObjects: []client.Object{
			processorImagesConfigMap,
		},
		ExpectTracks: []rtesting.TrackRequest{
			rtesting.NewTrackRequest(processorImagesConfigMap, processor, scheme),
		},
		ExpectStashedValues: map[reconcilers.StashKey]interface{}{
			streaming.ProcessorImagesStashKey: processorImagesConfigMap.Create().Data,
		},
	},
}

rts.Test(t, scheme, func(t *testing.T, rtc *rtesting.SubReconcilerTestCase, c reconcilers.Config) reconcilers.SubReconciler {
	return streaming.ProcessorSyncProcessorImages(c, testSystemNamespace)
})
```
[full source](https://github.com/projectriff/system/blob/4c3b75327bf99cc37b57ba14df4c65d21dc79d28/pkg/controllers/streaming/processor_reconciler_test.go#L279-L305)

## Utilities

### Config

The [`Config`](https://pkg.go.dev/github.com/vmware-labs/reconciler-runtime/reconcilers#Config) is a single object that contains the key APIs needed by a reconciler. The config object is provided to the reconciler when initialized and is preconfigured for the reconciler.

### Stash

The stash allows passing arbitrary state between sub reconcilers within the scope of a single reconciler request. Values are stored on the context by [`StashValue`](https://pkg.go.dev/github.com/vmware-labs/reconciler-runtime/reconcilers#StashValue) and accessed via [`RetrieveValue`](https://pkg.go.dev/github.com/vmware-labs/reconciler-runtime/reconcilers#RetrieveValue).

**Example:**

```go
const exampleStashKey reconcilers.StashKey = "example"

func StashExampleSubReconciler(c reconcilers.Config) reconcilers.SubReconciler {
	return &reconcilers.SyncReconciler{
		Name: "StashExample",
		Sync: func(ctx context.Context, resource *examplev1.MyExample) error {
			value := Example{} // something we want to expose to a sub reconciler later in this chain
			reconcilers.StashValue(ctx, exampleStashKey, *value)
			return nil
		},
	}
}


func StashExampleSubReconciler(c reconcilers.Config) reconcilers.SubReconciler {
	return &reconcilers.SyncReconciler{
		Name: "StashExample",
		Sync: func(ctx context.Context, resource *examplev1.MyExample) error {
			value, ok := reconcilers.RetrieveValue(ctx, exampleStashKey).(Example)
			if !ok {
				return nil, fmt.Errorf("expected stashed value for key %q", exampleStashKey)
			}
			... // do something with the value
		},
	}
}
```

### Tracker

The [`Tracker`](https://pkg.go.dev/github.com/vmware-labs/reconciler-runtime/tracker#Tracker) provides a means for one resource to watch another resource for mutations, triggering the reconciliation of the resource defining the reference.

It's common to work with a resource that is also tracked. The [Config.TrackAndGet](https://pkg.go.dev/github.com/vmware-labs/reconciler-runtime/reconcilers#Config.TrackAndGet) method uses the same signature as client.Get, but additionally tracks the resource.

In the [Setup](https://pkg.go.dev/github.com/vmware-labs/reconciler-runtime@v0.4.0/reconcilers#SyncReconciler) method, a watch is created that will notify the handler every time a resource of that kind is mutated. The [EnqueueTracked](https://pkg.go.dev/github.com/vmware-labs/reconciler-runtime/reconcilers#EnqueueTracked) helper returns a list of resources that are tracking the given resource, those resources are enqueued for the reconciler.

The tracker will automatically expire a track request if not periodically renewed. By default, the TTL is 2x the resync internal. This ensures all tracked resources will naturally have the tracking relationship refreshed as part of the normal reconciliation resource. There is no need to manually untrack a resource.

**Example:**

The stream gateways in projectriff fetch the image references they use to run from a ConfigMap, when the values change, we want to detect and rollout the updated images.

```go
func InMemoryGatewaySyncConfigReconciler(c reconcilers.Config, namespace string) reconcilers.SubReconciler {
	return &reconcilers.SyncReconciler{
		Name: "SyncConfig",
		Sync: func(ctx context.Context, parent *streamingv1alpha1.InMemoryGateway) error {
			log := logr.FromContextOrDiscard(ctx)
			c := reconciler.RetrieveConfig(ctx)

			var config corev1.ConfigMap
			key := types.NamespacedName{Namespace: namespace, Name: inmemoryGatewayImages}
			// track config for new images, get the configmap
			if err := c.TrackAndGet(ctx, key, &config); err != nil {
				return err
			}
			// consume the configmap
			parent.Status.GatewayImage = config.Data[gatewayImageKey]
			parent.Status.ProvisionerImage = config.Data[provisionerImageKey]
			return nil
		},

		Setup: func(ctx context.Context, mgr reconcilers.Manager, bldr *reconcilers.Builder) error {
			// enqueue the tracking resource for reconciliation from changes to
			// tracked ConfigMaps. Internally `EnqueueTracked` sets up an 
			// Informer to watch to changes of the target resource. When the
			// informer emits an event, the tracking resources are looked up
			// from the tracker and enqueded for reconciliation.
			bldr.Watches(&source.Kind{Type: &corev1.ConfigMap{}}, reconcilers.EnqueueTracked(ctx, &corev1.ConfigMap{}))
			return nil
		},
	}
}
```
[full source](https://github.com/projectriff/system/blob/4c3b75327bf99cc37b57ba14df4c65d21dc79d28/pkg/controllers/streaming/inmemorygateway_reconciler.go#L58-L84)

### Status

The `apis` package provides means for conveniently managing a custom resource's `.status`.

A resource's status subresource is expected to meet the following contract:

```go
type MyStatus struct {
	ObservedGeneration int64 `json:"observedGeneration,omitempty"`
	Conditions []metav1.Condition `json:"conditions,omitempty" patchStrategy:"merge" patchMergeKey:"type"`
}
```

**Example:**

Embed `api.Status` into your resource's status and add more fields:

```go
type MyResourceStatus struct {
  apis.Status `json:",inline"`
  UsefulMessage string `json:"usefulMessage,omitempty"`
}

type MyResource struct {
  metav1.TypeMeta   `json:",inline"`
  metav1.ObjectMeta `json:"metadata,omitempty"`
  
  Spec MyResourceSpec `json:"spec"`
  // +optional
  Status MyResourceStatus `json:"status"`
}
```

### Finalizers

[Finalizers](https://kubernetes.io/docs/concepts/overview/working-with-objects/finalizers/) allow a reconciler to clean up state for a resource that has been deleted by a client, and not yet fully removed. Terminating resources have `.metadata.deletionTimestamp` set. Resources with finalizers will stay in this terminating state until all finalizers are cleared from the resource. While using the [Kubernetes garbage collector](https://kubernetes.io/docs/concepts/architecture/garbage-collection/) is recommended when possible, finalizer are useful for cases when state exists outside of the same cluster, scope, and namespace of the parent resource that needs to be cleaned up when no longer used.

Deleting a resource that uses finalizers requires the controller to be running.

> Note: [WithFinalizer](#withfinalizer) can be used in lieu of, or in conjunction with, [ChildReconciler](#childreconciler)#Finalizer. The distinction is the scope within the reconciler tree where a finalizer is applied. While a reconciler can define as many finalizer on the resource as it desires, in practice, it's best to minimize the number of finalizers as setting and clearing each finalizer makes a request to the API Server. 
>
> A single WithFinalizer will always add a finalizer to the parent resource. It can then compose multiple ChildReconcilers, as well as other reconcilers that do not natively support managing finalizers (e.g. SyncReconciler). On the other hand, the ChildReconciler will only set the finalizer when it is required potentially reducing the number of finalizers, but only covers that exact sub-reconciler. It's important the external state that needs to be cleaned up be covered by a finalizer, it does not matter which finalizer is used.

The [AddParentFinalizer](https://pkg.go.dev/github.com/vmware-labs/reconciler-runtime/reconcilers#AddParentFinalizer) and [ClearParentFinalizer](https://pkg.go.dev/github.com/vmware-labs/reconciler-runtime/reconcilers#ClearParentFinalizer) functions patch the parent resource to update its finalizers. These methods work with [CastParents](#castparent) resources and use the same client the [ParentReconciler](#parentreconciler) used to originally load the parent resource. They can be called inside [SubReconcilers](#subreconciler) that may use a different client.

When an update is required, only the `.metadata.finalizers` field is patched. The parent's `.metadata.resourceVersion` is used as an optimistic concurrency lock, and is updated with the value returned from the server. Any error from the server will cause the resource reconciliation to err. When testing with the [SubReconcilerTestSuite](#subreconcilertestsuite), the resource version of the parent defaults to `"999"`, the patch bytes include the resource version and the response increments the parent's resource version. For a parent with the default resource version that patches a finalizer, the expected parent will have a resource version of `"1000"`.

A minimal test case for a sub reconciler that adds a finalizer may look like:

```go
	...
	{
		Name: "add 'test.finalizer' finalizer",
		Parent: resourceDie,
		ExpectEvents: []rtesting.Event{
			rtesting.NewEvent(resourceDie, scheme, corev1.EventTypeNormal, "FinalizerPatched",
				`Patched finalizer %q`, "test.finalizer"),
		},
		ExpectParent: resourceDie.
			MetadataDie(func(d *diemetav1.ObjectMetaDie) {
				d.Finalizers("test.finalizer")
				d.ResourceVersion("1000")
			}),
		ExpectPatches: []rtesting.PatchRef{
			{
				Group:     "testing.reconciler.runtime",
				Kind:      "TestResource",
				Namespace: resourceDie.GetNamespace(),
				Name:      resourceDie.GetName(),
				PatchType: types.MergePatchType,
				Patch:     []byte(`{"metadata":{"finalizers":["test.finalizer"],"resourceVersion":"999"}}`),
			},
		},
	},
	...
```

## Contributing

The reconciler-runtime project team welcomes contributions from the community. If you wish to contribute code and you have not signed our contributor license agreement (CLA), our bot will update the issue when you open a Pull Request. For any questions about the CLA process, please refer to our [FAQ](https://cla.vmware.com/faq). For more detailed information, refer to [CONTRIBUTING.md](CONTRIBUTING.md).

## Acknowledgements

`reconciler-runtime` was conceived in [`projectriff/system`](https://github.com/projectriff/system/) and implemented initially by [Scott Andrews](https://github.com/scothis), [Glyn Normington](https://github.com/glyn) and the [riff community](https://github.com/orgs/projectriff/people) at large, drawing inspiration from [Kubebuilder](https://www.kubebuilder.io) and [Knative](https://knative.dev) reconcilers.

## License

Apache License v2.0: see [LICENSE](./LICENSE) for details.
