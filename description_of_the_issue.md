# Problem: Infinite reconciliation if two KIP CRs in the same namespace
### Explanation on why this happens:
If we have these two KIP resources in the same namespace, where `example` was created before `example-2`:

```
apiVersion: che.eclipse.org/v1alpha1
kind: KubernetesImagePuller
metadata:
  name: example
  namespace: kubernetes-image-puller-operator
spec:
  configMapName: k8s-image-puller
  deploymentName: kubernetes-image-puller
  imagePullerImage: 'quay.io/eclipse/kubernetes-image-puller:next'
status:
  imagePullerImage: 'quay.io/eclipse/kubernetes-image-puller:next'
```

```
apiVersion: che.eclipse.org/v1alpha1
kind: KubernetesImagePuller
metadata:
  name: example-2
  namespace: kubernetes-image-puller-operator
spec:
  images: 'test=quay.io/dkwon17/test:base'
  configMapName: k8s-image-puller
  deploymentName: kubernetes-image-puller
  imagePullerImage: 'quay.io/eclipse/kubernetes-image-puller:next'
```

In this case, we have the same `configMapName`, but different configmap data (ie, different images to prepull).
When reconciling `example-2`, the operator will update the `k8s-image-puller` configmap, and restart daemonset pods. See code [here](https://github.com/che-incubator/kubernetes-image-puller-operator/blob/eda6cc352391778f928a6d645c136b4c835495c3/controllers/kubernetesimagepuller_controller.go#L156C4-L173C25). 
The update to the configmap reconciles `example` KIP. But since `example` is reconciled, the same `k8s-image-puller` configmap is updated again, which leads to pods restarting again.
This loop happens infinitely.

But If `example-2` has a different `configMapName`, `deploymentName` altogether:

```
apiVersion: che.eclipse.org/v1alpha1
kind: KubernetesImagePuller
metadata:
  name: example-2
  namespace: kubernetes-image-puller-operator
spec:
  images: 'test=quay.io/dkwon17/test:base'
  configMapName: diffconfigmap
  deploymentName: diffdeployment
  daemonsetName: diffdaemonset
  imagePullerImage: 'quay.io/eclipse/kubernetes-image-puller:next'
```

In this case, the `example` KIP's deployment (which is not named `diffdeployment`) is removed when reconciling `example-2`. This is done [here](https://github.com/che-incubator/kubernetes-image-puller-operator/blob/eda6cc352391778f928a6d645c136b4c835495c3/controllers/kubernetesimagepuller_controller.go#L253-L264).
Because `example` KIP's deployment is removed, `example` will reconcile, and will remove `example-2` KIP's deployment, causing an endless loop of reconciling and removing deployments.
