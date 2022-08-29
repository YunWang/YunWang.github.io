---
layout: post
title: Namespace lifecycle admission
tags: kubernetes
---

# Namespace lifecycle admission

# Summary

Namespace lifecycle admission is a built-in mutating admission

# Responsibility

Namespace lifecycle admission is responsible for ensuring the namespace is ok for namespaced resources operation. In detail,

1. Prevent deletion of immortal namespaces, including “default”, “kube-system”, “kube-public”.
2. Always allow non-namespaced resources, like clusterrole, clusterrolebinding
3. Always allow deletion of other resources.
4. Always allow access review checks.
5. Refuse to operate on non-existent namespaces, that is, refuse to create/update pod, secret and so on in non-existent namespaces
6. Ensure not to create objects in terminating namespaces

# Source code

## 1. Prevent deletion of immortal namespaces

Source code is in file ./pkg/admission/plugin/namespace/lifecycle/admission.go.

Immortal namespaces is unconfigurable, it only contains three namespaces, “default”, “kube-system”, “kube-public”, and it will be initialized when registers this plugin.

```go
func Register(plugins *admission.Plugins) {
	plugins.Register(PluginName, func(config io.Reader) (admission.Interface, error) {
		return NewLifecycle(sets.NewString(metav1.NamespaceDefault, metav1.NamespaceSystem, metav1.NamespacePublic))
	})
}

func NewLifecycle(immortalNamespaces sets.String) (*Lifecycle, error) {
	return newLifecycleWithClock(immortalNamespaces, clock.RealClock{})
}

func newLifecycleWithClock(immortalNamespaces sets.String, clock utilcache.Clock) (*Lifecycle, error) {
	forceLiveLookupCache := utilcache.NewLRUExpireCacheWithClock(100, clock)
	return &Lifecycle{
		Handler:              admission.NewHandler(admission.Create, admission.Update, admission.Delete),
		immortalNamespaces:   immortalNamespaces,
		forceLiveLookupCache: forceLiveLookupCache,
	}, nil
}

func (l *Lifecycle) Admit(ctx context.Context, a admission.Attributes, o admission.ObjectInterfaces) error {
	// prevent deletion of immortal namespaces
	if a.GetOperation() == admission.Delete && a.GetKind().GroupKind() == v1.SchemeGroupVersion.WithKind("Namespace").GroupKind() && l.immortalNamespaces.Has(a.GetName()) {
		return errors.NewForbidden(a.GetResource().GroupResource(), a.GetName(), fmt.Errorf("this namespace may not be deleted"))
	}
  ...
}
```

## 2. Allowed case

1. Always allow non-namespaced resources, like clusterrole, clusterrolebinding
2. Always allow deletion of other resources.
3. Always allow access review checks.

Source code is in file ./pkg/admission/plugin/namespace/lifecycle/admission.go.

```go
func (l *Lifecycle) Admit(ctx context.Context, a admission.Attributes, o admission.ObjectInterfaces) error {
	...

	// always allow non-namespaced resources
	if len(a.GetNamespace()) == 0 && a.GetKind().GroupKind() != v1.SchemeGroupVersion.WithKind("Namespace").GroupKind() {
		return nil
	}

	// always allow deletion of other resources
	if a.GetOperation() == admission.Delete {
		return nil
	}

	// always allow access review checks.  Returning status about the namespace would be leaking information
	if isAccessReview(a) {
		return nil
	}
  ...
}
```

## 3. Refuse to operate on non-existent namespaces

Source code is in file ./pkg/admission/plugin/namespace/lifecycle/admission.go.

How to confirm whether a namespace exist? We usually get resources from cache, but it’s not convincing if it only checks local cache, because it’s possible that local cache is out of date. Besides local cache, we should try to get the namespace from apiserver directly. If the namespace isn’t found in apiserver, we can say that the namespace isn’t exist.

In detail, lifecycle namespace admission will try to get the namespace from cache twice, and it will sleep “*`missingNamespaceWait`*” seconds if it cannot get the namespace at the first time. The “missingNamespaceWait” is fixed, 50 * time.Millisecond. Then it tries to get namespace from apiserver directly. If it cannot get the namespace, it means the namespace doesn’t exist.

```go
const missingNamespaceWait = 50 * time.Millisecond

func (l *Lifecycle) Admit(ctx context.Context, a admission.Attributes, o admission.ObjectInterfaces) error {

	...

	var (
		exists bool
		err    error
	)

  // get namespace from local cache firstly
	namespace, err := l.namespaceLister.Get(a.GetNamespace())
	if err != nil {
		if !errors.IsNotFound(err) {
			return errors.NewInternalError(err)
		}
	} else {
		exists = true
	}

	if !exists && a.GetOperation() == admission.Create {
		// give the cache time to observe the namespace before rejecting a create.
		// this helps when creating a namespace and immediately creating objects within it.
		time.Sleep(missingNamespaceWait)
    // second time
		namespace, err = l.namespaceLister.Get(a.GetNamespace())
		switch {
		case errors.IsNotFound(err):
			// no-op
		case err != nil:
			return errors.NewInternalError(err)
		default:
			exists = true
		}
		if exists {
			klog.V(4).InfoS("Namespace existed in cache after waiting", "namespace", klog.KRef("", a.GetNamespace()))
		}
	}

  ...

	// refuse to operate on non-existent namespaces
	if !exists || forceLiveLookup {
		// as a last resort, make a call directly to storage
    // get namespace from apiserver directly
		namespace, err = l.client.CoreV1().Namespaces().Get(context.TODO(), a.GetNamespace(), metav1.GetOptions{})
		switch {
		case errors.IsNotFound(err):
			return err
		case err != nil:
			return errors.NewInternalError(err)
		}

		klog.V(4).InfoS("Found namespace via storage lookup", "namespace", klog.KRef("", a.GetNamespace()))
	}

	...
}
```

## 4. Ensure not to create objects in terminating namespaces

Source code is in file ./pkg/admission/plugin/namespace/lifecycle/admission.go.

There is a corner case, that is, if the local cache is out of date, it need to get namespace from storage. How to know whether local cache is out of date? If user is deleting namespace, and the namespace is in active phase, that means the local cache is out of date.

What’s the function of “`forceLiveLookupCache`”? It will record deleting namespace for “*`forceLiveLookupTTL`*”, which is “`30 * time.*Second*`”. So the function of “`forceLiveLookupCache`” is to cache deleting namespace for a while, 30s, and if a namespace is deleting and in the meanwhile, the namespace in local cache is in active phase, that means the local cache is out of date. 

```go
const forceLiveLookupTTL = 30 * time.Second

func (l *Lifecycle) Admit(ctx context.Context, a admission.Attributes, o admission.ObjectInterfaces) error {
	...
  
	if a.GetKind().GroupKind() == v1.SchemeGroupVersion.WithKind("Namespace").GroupKind() {
		if a.GetOperation() == admission.Delete {
      // Record deleting namespace in forceLiveLookupCache
			l.forceLiveLookupCache.Add(a.GetName(), true, forceLiveLookupTTL)
		}
		// allow all operations to namespaces
		return nil
	}

	...
	namespace, err := l.namespaceLister.Get(a.GetNamespace())
	if err != nil {
		if !errors.IsNotFound(err) {
			return errors.NewInternalError(err)
		}
	} else {
		exists = true
	}

	if !exists && a.GetOperation() == admission.Create {
		// give the cache time to observe the namespace before rejecting a create.
		// this helps when creating a namespace and immediately creating objects within it.
		time.Sleep(missingNamespaceWait)
		namespace, err = l.namespaceLister.Get(a.GetNamespace())
		switch {
		case errors.IsNotFound(err):
			// no-op
		case err != nil:
			return errors.NewInternalError(err)
		default:
			exists = true
		}
		if exists {
			klog.V(4).InfoS("Namespace existed in cache after waiting", "namespace", klog.KRef("", a.GetNamespace()))
		}
	}

	// forceLiveLookup if true will skip looking at local cache state and instead always make a live call to server.
	forceLiveLookup := false
	if _, ok := l.forceLiveLookupCache.Get(a.GetNamespace()); ok {
		// we think the namespace was marked for deletion, but our current local cache says otherwise, we will force a live lookup.
		forceLiveLookup = exists && namespace.Status.Phase == v1.NamespaceActive
	}

	// refuse to operate on non-existent namespaces
	if !exists || forceLiveLookup {
		// as a last resort, make a call directly to storage
    // get namespace from storage directly
		namespace, err = l.client.CoreV1().Namespaces().Get(context.TODO(), a.GetNamespace(), metav1.GetOptions{})
		switch {
		case errors.IsNotFound(err):
			return err
		case err != nil:
			return errors.NewInternalError(err)
		}

		klog.V(4).InfoS("Found namespace via storage lookup", "namespace", klog.KRef("", a.GetNamespace()))
	}

	// ensure that we're not trying to create objects in terminating namespaces
	if a.GetOperation() == admission.Create {
		if namespace.Status.Phase != v1.NamespaceTerminating {
			return nil
		}

		err := admission.NewForbidden(a, fmt.Errorf("unable to create new content in namespace %s because it is being terminated", a.GetNamespace()))
		if apierr, ok := err.(*errors.StatusError); ok {
			apierr.ErrStatus.Details.Causes = append(apierr.ErrStatus.Details.Causes, metav1.StatusCause{
				Type:    v1.NamespaceTerminatingCause,
				Message: fmt.Sprintf("namespace %s is being terminated", a.GetNamespace()),
				Field:   "metadata.namespace",
			})
		}
		return err
	}

	return nil
}
```