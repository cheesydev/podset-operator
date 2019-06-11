# podset-operator

Example of a Kubernetes Operator.

It defines a `PodSet`, a new Custom Resource Definition (CRD).

```
https://medium.com/faun/writing-your-first-kubernetes-operator-8f3df4453234
```


```
# deploy/crds/app_v1alpha1_podset_crd.yaml

apiVersion: apiextensions.k8s.io/v1beta1
kind: CustomResourceDefinition
metadata:
  name: podsets.app.rafaelportela.com.br
spec:
  group: app.rafaelportela.com.br
  names:
    kind: PodSet
    ...

```

Example of a PodSet manifest:
```
# deploy/crds/app_v1alpha1_podset_cr.yaml

apiVersion: app.rafaelportela.com.br/v1alpha1
kind: PodSet
metadata:
  name: example-podset
spec:
  # Add fields here
  size: 3
```

## Deploy the CRD in your cluster

```
kubectl create -f deploy/service_account.yaml
kubectl create -f deploy/role.yaml
kubectl create -f deploy/role_binding.yaml
```

Deploy the CRD so your cluster will be aware of a `PodSet` definition and APIs.
```
kubectl create -f deploy/crds/app_v1alpha1_podset_crd.yaml
```

Then, build and push the operator image, and deploy the operator controller.
```
operator-sdk build rafaelportela/podset-operator
docker push rafaelportela/podset-operator

# the yaml references the image created above
kubectl create -f deploy/operator.yaml

kubectl get pods      # there's one podset-operator pod running, the operator itself
kubectl get podsets   # no podset instances yet
```

Let's finally run a PodSet.
```
kubectl create -f deploy/crds/app_v1alpha1_podset_cr.yaml
```


### API definition

The API fields is defined in `pkg/apis/app/v1alpha1/podset_types.go`. A new
`Replicas` field was added to the PodSecSpec and `PodNames` array was added to
PodSetStatus.

After updating the API, some code should be re-generated with `operator-sdk
generate k8s`.

### Controller business logic

The code for the controller is in `pkg/controller/podset/podset_controller.go`.
The generated code for the operator watches PodSets resources it creates
a single pod in the namespace.

```go
...

// Reconcile reads that state of the cluster for a PodSet object and makes changes based on the state read
// and what is in the PodSet.Spec
func (r *ReconcilePodSet) Reconcile(request reconcile.Request) (reconcile.Result, error) {

	instance := &appv1alpha1.PodSet{}
	r.client.Get(context.TODO(), request.NamespacedName, instance)

	// Define a new Pod object
	pod := newPodForCR(instance)

	// Check if this Pod already exists
	found := &corev1.Pod{}
	err = r.client.Get(context.TODO(), types.NamespacedName{Name: pod.Name, Namespace: pod.Namespace}, found)
	if err != nil && errors.IsNotFound(err) {

		reqLogger.Info("Creating a new Pod", "Pod.Namespace", pod.Namespace, "Pod.Name", pod.Name)
		err = r.client.Create(context.TODO(), pod)

		// Pod created successfully - don't requeue
		return reconcile.Result{}, nil
	}

  ...
}

...
```

