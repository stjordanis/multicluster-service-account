# Multicluster-Service-Account

Multicluster-service-account makes it easy to use Kubernetes service accounts as multicluster identities. It imports and automounts remote service account tokens inside pods, for them to call the Kubernetes APIs of other clusters. Multicluster-service-account works well with [multicluster-controller](https://github.com/admiraltyio/multicluster-controller), but any cross-cluster Kubernetes client can benefit from it.

Why? Check out [Admiralty's blog post introducing multicluster-service-account](https://admiralty.io/blog/introducing-multicluster-service-account).

## How it Works

Multicluster-service-account consists of:

1. a ServiceAccountImport custom resource definition (CRD) and controller to **import remote service accounts** (and their secrets);
1. a dynamic admission webhook to **automount service account import secrets inside annotated pods**, the same way regular service accounts are automounted inside pods;
1. helper methods to **generate client-go configurations from service account imports** (as well as generic methods to fall back to kubeconfig contexts and regular service accounts).

## Getting Started

We assume that you are a cluster admin on two clusters, associated with, e.g., the contexts "cluster1" and "cluster2" in your kubeconfig. We're going to install multicluster-service-account and run a multicluster client example in cluster1, listing pods in cluster2.

```bash
CLUSTER1=cluster1 # change me
CLUSTER2=cluster2 # change me
```

### Step 1: Installation

Install multicluster-service-account in cluster1:

```bash
RELEASE_URL=https://github.com/admiraltyio/multicluster-service-account/releases/download/v0.2.0
MANIFEST_URL=$RELEASE_URL/install.yaml
kubectl apply -f $MANIFEST_URL --context $CLUSTER1
```

Cluster1 is now able to import service accounts, but it hasn't been given permission to import them from cluster2 yet. This is a chicken-and-egg problem: cluster1 needs a token from cluster2, before it can import service accounts from it. To solve this problem, download the kubemcsa binary and run the bootstrap command:

```bash
OS=linux # or darwin (i.e., OS X) or windows
ARCH=amd64 # if you're on a different platform, you must know how to build from source
BINARY_URL="$RELEASE_URL/kubemcsa-$OS-$ARCH"
curl -Lo kubemcsa $BINARY_URL
chmod +x kubemcsa
sudo mv kubemcsa /usr/local/bin

kubemcsa bootstrap $CLUSTER1 $CLUSTER2
```

### Step 2: Example

The `multicluster-client` example includes:

- in cluster2:
  - a service account named `pod-lister` in the default namespace, bound to a role that can only list pods in its namespace;
  - a dummy NGINX deployment (to have pods to list);
- in cluster1:
  - a service account import named `cluster2-default-pod-lister`, importing `pod-lister` from the default namespace of cluster2;
  - a `multicluster-client` job, whose pod is annotated to automount `cluster2-default-pod-lister`'s secret—it will list the pods in the default namespace of cluster2, and stop without restarting (we'll check the logs).

```bash
kubectl config use-context $CLUSTER2
kubectl create serviceaccount pod-lister
kubectl create role pod-lister --verb=list --resource=pods
kubectl create rolebinding pod-lister --role=pod-lister \
  --serviceaccount=default:pod-lister
kubectl run nginx --image nginx

kubectl config use-context $CLUSTER1
cat <<EOF | kubectl create -f -
apiVersion: multicluster.admiralty.io/v1alpha1
kind: ServiceAccountImport
metadata:
  name: $CLUSTER2-default-pod-lister
spec:
  clusterName: $CLUSTER2
  namespace: default
  name: pod-lister
---
apiVersion: batch/v1
kind: Job
metadata:
  name: multicluster-client
spec:
  template:
    metadata:
      annotations:
        multicluster.admiralty.io/service-account-import.name: $CLUSTER2-default-pod-lister
    spec:
      restartPolicy: Never
      containers:
      - name: multicluster-client
        image: quay.io/admiralty/multicluster-service-account-example-multicluster-client:latest
EOF
```

In cluster1, check that:

1. The service account import controller created a secret for the `cluster2-default-pod-lister` service account import, containing the token and namespace of the remote service account, and the URL and root certificate of the remote Kubernetes API:
    ```bash
    kubectl get secret -l multicluster.admiralty.io/service-account-import.name=$CLUSTER2-default-pod-lister -o yaml
    # the data is base64-encoded
    ```
1. The service account import secret was mounted inside the `multicluster-client` pod by the service account import admission controller:
    ```bash
    kubectl get pod -l job-name=multicluster-client -o yaml
    # look at volumes and volume mounts
    ```
1. The `multicluster-client` pod was able to list pods in the default namespace of cluster2:
    ```bash
    kubectl logs job/multicluster-client
    ```

## Service Account Imports

Service account imports tell the service account import controller to maintain a secret in the same namespace, containing the remote service account's namespace and token, as well as the URL and root certificate of the remote Kubernetes API, which are all necessary data to configure a Kubernetes client. If a pod needs to call several clusters, it will use several service account imports, e.g.:

```yaml
apiVersion: multicluster.admiralty.io/v1alpha1
kind: ServiceAccountImport
metadata:
  name: cluster2-default-pod-lister
spec:
  clusterName: cluster2
  namespace: default
  name: pod-lister
---
apiVersion: multicluster.admiralty.io/v1alpha1
kind: ServiceAccountImport
metadata:
  name: cluster3-default-pod-lister
spec:
  clusterName: cluster3
  namespace: default
  name: pod-lister
```

## Annotations

The `multicluster.admiralty.io/service-account-import.name` annotation on a pod (or pod template) tells the service account import admission controller to automount the corresponding secrets inside it. If a pod needs several service account imports, separate their names with commas, e.g.:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: multicluster-client
  annotations:
    multicluster.admiralty.io/service-account-import.name: cluster2-default-pod-lister,cluster3-default-pod-lister
spec:
  # ...
```

Note: just like with local service accounts, there is a race condition if a service account import and a pod requesting it are created at the same time: the service account import admission controller will likely reject the pod because the secret to automount won't be ready. Luckily, if the pod is controlled by another object, such as a deployment, job, etc., pod creation will be retried.

## Client Configuration

Multicluster-service-account includes a Go library (cf. [`pkg/config`](pkg/config)) to facilitate the creation of [client-go `rest.Config`](https://godoc.org/k8s.io/client-go/rest#Config) instances from service account imports. From there, you can create [`kubernetes.Clientset`](https://godoc.org/k8s.io/client-go/kubernetes#NewForConfig) instances as usual. The namespaces of the remote service accounts are also provided:

```go
cfg, ns, err := NamedServiceAccountImportConfigAndNamespace("cluster2-default-pod-lister")
// ...
clientset, err := kubernetes.NewForConfig(cfg)
// ...
pods, err := clientset.CoreV1().Pods(ns).List(metav1.ListOptions{})
```

Usually, however, you don't want to hardcode the name of the mounted service account import. If you only expect one, you can get a Config for it and its remote namespace like this:

```go
cfg, ns, err := ServiceAccountImportConfigAndNamespace()
```

If several service account imports are mounted, you can get Configs and namespaces for all of them by name as a `map[string]ConfigAndNamespaceTuple`:

```go
all, err := AllServiceAccountImportConfigsAndNamespaces()
// ...
for name, cfgAndNs := range all {
  cfg := cfgAndNs.Config
  ns := cfgAndNs.Namespace
  // ...
}
```

### Generic Client Configuration

The true power of multicluster-service-account's `config` package is in its generic functions, that can fall back to kubeconfig contexts or regular service accounts when no service account import is mounted:

```go
cfg, ns, err := ConfigAndNamespace()
```

```go
all, err := AllNamedConfigsAndNamespaces()
```

The service account import controller uses `AllNamedConfigsAndNamespaces()` internally. That's how we were able to bootstrap in the getting started guide: the same binary was used with a kubeconfig out-of-cluster and with service account imports in-cluster. The [generic client example](examples/generic-client/) uses `ConfigAndNamespace()`.

## API Reference

For more details on the `config` package, or to better understand how the service account import controller and admission control work, please refer to the API documentation:

https://godoc.org/admiralty.io/multicluster-service-account/

or

```bash
go get admiralty.io/multicluster-service-account
godoc -http=:6060
```

then http://localhost:6060/pkg/admiralty.io/multicluster-service-account/