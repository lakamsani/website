---
title: "Frequently asked questions"
linkTitle: "FAQ"
description: "Flux and the GitOps Toolkit frequently asked questions."
weight: 100
---

## Kustomize questions

### Are there two Kustomization types?

Yes, the `kustomization.kustomize.toolkit.fluxcd.io` is a Kubernetes
[custom resource](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/)
while `kustomization.kustomize.config.k8s.io` is the type used to configure a
[Kustomize overlay](https://kubectl.docs.kubernetes.io/references/kustomize/).

The `kustomization.kustomize.toolkit.fluxcd.io` object refers to a `kustomization.yaml`
file path inside a Git repository or Bucket source.

### How do I use them together?

Assuming an app repository with `./deploy/prod/kustomization.yaml`:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namespace: default
resources:
  - deployment.yaml
  - service.yaml
  - ingress.yaml
```

Define a source of type `gitrepository.source.toolkit.fluxcd.io`
that pulls changes from the app repository every 5 minutes inside the cluster:

```yaml
apiVersion: source.toolkit.fluxcd.io/v1beta2
kind: GitRepository
metadata:
  name: my-app
  namespace: default
spec:
  interval: 5m
  url: https://github.com/my-org/my-app
  ref:
    branch: main
```

Then define a `kustomization.kustomize.toolkit.fluxcd.io` that uses the `kustomization.yaml`
from `./deploy/prod` to determine which resources to create, update or delete:

```yaml
apiVersion: kustomize.toolkit.fluxcd.io/v1beta2
kind: Kustomization
metadata:
  name: my-app
  namespace: default
spec:
  interval: 15m
  path: "./deploy/prod"
  prune: true
  sourceRef:
    kind: GitRepository
    name: my-app
```

### What is a Kustomization reconciliation?

In the above example, we pull changes from Git every 5 minutes,
and a new commit will trigger a reconciliation of
all the `Kustomization` objects using that source.

Depending on your configuration, a reconciliation can mean:

* generating a kustomization.yaml file in the specified path
* building the kustomize overlay
* decrypting secrets
* validating the manifests with client or server-side dry-run
* applying changes on the cluster
* health checking of deployed workloads
* garbage collection of resources removed from Git
* issuing events about the reconciliation result
* recoding metrics about the reconciliation process

The 15 minutes reconciliation interval, is the interval at which you want to undo manual changes
.e.g. `kubectl set image deployment/my-app` by reapplying the latest commit on the cluster.

Note that a reconciliation will override all fields of a Kubernetes object, that diverge from Git.
For example, you'll have to omit the `spec.replicas` field from your `Deployments` YAMLs if you
are using a `HorizontalPodAutoscaler` that changes the replicas in-cluster.

### Can I use repositories with plain YAMLs?

Yes, you can specify the path where the Kubernetes manifests are,
and kustomize-controller will generate a `kustomization.yaml` if one doesn't exist.

Assuming an app repository with the following structure:

```
├── deploy
│   └── prod
│       ├── .yamllint.yaml
│       ├── deployment.yaml
│       ├── service.yaml
│       └── ingress.yaml
└── src
```

Create a `GitRepository` definition and exclude all the files that are not Kubernetes manifests:

```yaml
apiVersion: source.toolkit.fluxcd.io/v1beta2
kind: GitRepository
metadata:
  name: my-app
  namespace: default
spec:
  interval: 5m
  url: https://github.com/my-org/my-app
  ref:
    branch: main
  ignore: |
    # exclude all
    /*
    # include deploy dir
    !/deploy
    # exclude non-Kubernetes YAMLs
    /deploy/**/.yamllint.yaml
```

Then create a `Kustomization` definition to reconcile the `./deploy/prod` dir:

```yaml
apiVersion: kustomize.toolkit.fluxcd.io/v1beta2
kind: Kustomization
metadata:
  name: my-app
  namespace: default
spec:
  interval: 15m
  path: "./deploy/prod"
  prune: true
  sourceRef:
    kind: GitRepository
    name: my-app
```

With the above configuration, source-controller will pull the Kubernetes manifests
from the app repository and kustomize-controller will generate a
`kustomization.yaml` including all the resources found with `./deploy/prod/**/*.yaml`.

The kustomize-controller creates `kustomization.yaml` files similar to:

```sh
cd ./deploy/prod && kustomize create --autodetect --recursive
```

### How can I safely move resources from one dir to another?

To move manifests from a directory synced by a Flux Kustomization to another dir synced by a
different Kustomization, first you need to disable garbage collection then move the files.

Assuming you have two Flux Kustomization named `app1` and `app2`, and you want to move a
deployment manifests named `deploy.yaml` from `app1` to `app2`:

1. Disable garbage collection by setting `prune: false` in the `app1` Flux Kustomization. Commit, push and reconcile the changes e.g. `flux reconcile ks flux-system --with-source`.
2. Verify that pruning is disabled in-cluster with `flux export ks app1`.
3. Move the `deploy.yaml` manifest to the `app2` dir, then commit, push and reconcile e.g. `flux reconcile ks app2 --with-source`.
4. Verify that the deployment is now managed by the `app2` Kustomization with `flux tree ks apps2`.
5. Reconcile the `app1` Kustomization and verify that the deployment is no longer managed by it `flux reconcile ks app1 && flux tree ks app1`.
6. Finally, enable garbage collection by setting `prune: true` in `app1` Kustomization, then commit and push the changes upstream.


### Why are kubectl edits rolled back by Flux?

If you use kubectl to edit an object managed by Flux, all changes will be undone 
when kustomize-controller reconciles a Flux Kustomization containing that object.

In order for Flux to preserve fields added with kubectl, for example a label or annotation,
you have to specify a field manager named `flux-client-side-apply` e.g.:

```yaml
kubectl annotate --field-manager=flux-client-side-apply 
```

Note that fields specified in Git will always be overridden, the above procedure works only for
adding new fields that don't overlap with the desired state.

### What is the behavior of Kustomize used by Flux?

We referred to the **Kustomize v4** CLI flags here,
so that you can replicate the same behavior using `kustomize build`:

- `---enable-alpha-plugins` is disabled by default, so it uses only the built-in plugins.
- `--load-restrictor` is set to `LoadRestrictionsNone`, so it allows loading files outside the dir containing `kustomization.yaml`.
- `--reorder` is set to `legacy`, so the output will have namespaces and cluster roles/role bindings first, CRDs before CRs, and webhooks last.

To replicate the build and apply dry run locally:

```sh
kustomize build --load-restrictor=LoadRestrictionsNone --reorder=legacy . \
| kubectl apply --server-side --dry-run=server -f-
```

{{% alert color="info" title="kustomization.yaml validation" %}}
To validate changes before committing and/or merging, [a validation
utility script is available](https://github.com/fluxcd/flux2-kustomize-helm-example/blob/main/scripts/validate.sh),
it runs `kustomize` locally or in CI with the same set of flags as
the controller and validates the output using `kubeconform`.
{{% /alert %}}

### How do I resolve `webhook does not support dry run` errors?

If you've installed Kubernetes dynamic admission controls you may see Flux
failing to reconcile with an error similar to
`dry-run failed, error: admission webhook "validation-service.default.svc" does not support dry run`.

To fix this issue, you'll have to find the `ValidatingWebhookConfiguration` or the `MutatingWebhookConfiguration`,
and set the `sideEffects` to `None` or `NoneOnDryRun`:

```yaml
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingWebhookConfiguration
webhooks:
- name: my-webhook.example.com
  sideEffects: None
```

### How do I resolve `some/resource configured` events spam?

If the controller emits change events for a specific resource (even if the resource hasn't changed),
you'll need to edit your YAML manifests and remove any optional field that is set to `null`,
empty string or empty object.

Example of empty fields that will trigger drift events:

```yaml
apiVersion: v1
kind: Example
metadata:
  name: example
spec:
  field1: null
  field2: {}
  field3: []
  field4:
```

The kustomize-controller detects drift between the manifests and the in-cluster resources
by running a server-side apply dry-run, removing the empty fields from your manifests
will help the controller detect drift correctly.

### How to patch CoreDNS and other pre-installed addons?

To patch a pre-installed addon like CoreDNS with customized content,
add a shell manifest with only the changed values and `kustomize.toolkit.fluxcd.io/ssa: merge`
annotation into your Git repository.

Example CoreDNS with custom replicas, the `spec.containers[]` empty list is needed
for the patch to work and will not override the existing containers:

```yaml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    k8s-app: kube-dns
  annotations:
    kustomize.toolkit.fluxcd.io/prune: disabled
    kustomize.toolkit.fluxcd.io/ssa: merge
  name: coredns
  namespace: kube-system
spec:
  replicas: 5
  selector:
    matchLabels:
      eks.amazonaws.com/component: coredns
      k8s-app: kube-dns
  template:
    metadata:
      labels:
        eks.amazonaws.com/component: coredns
        k8s-app: kube-dns
    spec:
      containers: []
```

Note that only non-managed fields should be modified else there will be a conflict with
the `manager` of the fields (e.g. `eks`). For example, while you will be able to modify
affinity/antiaffinity fields, the `manager` (e.g. `eks`) will revert those changes and
that might not be immediately visible to you
(with EKS that would be an interval of once every 30 minutes).
The deployment will go into a rolling upgrade and Flux will revert it back to the patched version.

## Helm questions

### How to debug "not ready" errors?

Misconfiguring the `HelmRelease.spec.chart`, like a typo in the chart name, version or chart source URL
would result in a "HelmChart is not ready" error displayed by:

```console
$ flux get helmreleases --all-namespaces
NAMESPACE	NAME   	READY	MESSAGE
default  	podinfo	False 	HelmChart 'default/default-podinfo' is not ready
```

In order to get to the root cause, first make sure the source e.g. the `HelmRepository`
is configured properly and has access to the remote `index.yaml`:

```console
$ flux get sources helm --all-namespaces
NAMESPACE  	NAME   	READY	MESSAGE
default   	podinfo	False	failed to fetch https://stefanprodan.github.io/podinfo2/index.yaml : 404 Not Found
```

If the source is `Ready`, then the error must be caused by the chart,
for example due to an invalid chart name or non-existing version:

```console
$ flux get sources chart --all-namespaces
NAMESPACE  	NAME           	READY	MESSAGE
default  	default-podinfo	False	no chart version found for podinfo-9.0.0
```

### How to debug "install retries exhausted" errors?

By default, Flux performs a health check of all the Kubernetes resources created at install time.
If any resource fails to become ready after five minutes, the `HelmRelease` will be marked as not ready:

```console
$ flux get helmreleases --all-namespaces
NAMESPACE	NAME   	READY	MESSAGE
default  	podinfo	False 	install retries exhausted
```

To find the resource that caused the installation failure, you can print the events of the Helm release with:

````console
$ kubectl describe helmrelease podinfo -n default
Events:
  Type    Reason  Age    From             Message
  ----    ------  ----   ----             -------
  Normal  info    2m23s  helm-controller  Helm install has started
  Normal  error   82s    helm-controller  Helm install failed: timed out waiting for the condition

Last Helm logs:

creating 4 resource(s)
beginning wait for 4 resources with timeout of 5m0s
Deployment is not ready: default/podinfo. 0 out of 1 expected pods are ready
````

To inspect the failing resources, you can disable the health checks with:

```yaml
apiVersion: helm.toolkit.fluxcd.io/v2beta1
kind: HelmRelease
metadata:
 name: podinfo
 namespace: default
spec:
 install:
   disableWait: true
 upgrade:
   disableWait: true
```

With `disableWait: true`, Flux will no longer wait for the resources to become ready, so you can 
inspect the deployment and find the underlying issue e.g. `kubectl describe deployment podinfo`.

### Can I use Flux HelmReleases without GitOps?

Yes, you can install the Flux components directly on a cluster
and manage Helm releases with `kubectl`.

Install the controllers needed for Helm operations with `flux`:

```sh
flux install \
--namespace=flux-system \
--network-policy=false \
--components=source-controller,helm-controller
```

Create a Helm release with `kubectl`:

```sh
cat << EOF | kubectl apply -f -
---
apiVersion: source.toolkit.fluxcd.io/v1beta2
kind: HelmRepository
metadata:
  name: bitnami
  namespace: flux-system
spec:
  interval: 30m
  url: https://charts.bitnami.com/bitnami
---
apiVersion: helm.toolkit.fluxcd.io/v2beta1
kind: HelmRelease
metadata:
  name: metrics-server
  namespace: kube-system
spec:
  interval: 60m
  releaseName: metrics-server
  chart:
    spec:
      chart: metrics-server
      version: "^5.x"
      sourceRef:
        kind: HelmRepository
        name: bitnami
        namespace: flux-system
  values:
    apiService:
      create: true
EOF
```

Based on the above definition, Flux will upgrade the release automatically
when Bitnami publishes a new version of the metrics-server chart.

### How do I resolve a `Request entity too large: limit is 3145728` error during Helm install or upgrade?

This error is returned by Helm when the release that is attempted to be made does not fit in a
`Secret`. Most of the time this is due to exceptionally large (umbrella) charts, as explained
in [`helm/helm#8281`](https://github.com/helm/helm/issues/8281).

If you are running into this, confirm first that your chart has all the required excludes in
their respective [`.helmignore`](https://helm.sh/docs/chart_template_guide/helm_ignore_file/) and
[`.sourceignore`](/docs/components/source/api/#source.toolkit.fluxcd.io/v1beta2.GitRepository) files.

## Raspberry Pi questions

### How do I resolve a `unable to open the Badger database` that puts `image-reflector-controller` in `CrashLoopBackOff`?

This error is commonly found in low-memory environments where an Out Of Memory condition is likely
to be experienced, like on Raspberry Pi 2 and 3 boards which have only 1GB of RAM each.

If you are on Raspbian, the default size of 100MB swap file is not large enough to avoid the fault
condition, and can be updated through configuration in `/etc/dphys-swapfile`: [add at least 1GB swap](https://raspberrypi.stackexchange.com/a/1605/82185).

## Flux v1 vs v2 questions

### What are the differences between v1 and v2?

Flux v1 is a monolithic do-it-all operator;
Flux v2 separates the functionalities into specialized controllers, collectively called the GitOps Toolkit.

You can find a detailed comparison of Flux v1 and v2 features in the [migration FAQ](../migration/faq-migration/).

### How can I migrate from v1 to v2?

The Flux community has created guides and example repositories
to help you migrate to Flux v2:

- [Migrate from Flux v1](/docs/migration/flux-v1-migration/)
- [Migrate from `.flux.yaml` and kustomize](/docs/migration/flux-v1-migration/#flux-with-kustomize)
- [Migrate from Flux v1 automated container image updates](/docs/migration/flux-v1-automation-migration/)
- [How to manage multi-tenant clusters with Flux v2](https://github.com/fluxcd/flux2-multi-tenancy)
- [Migrate from Helm Operator to Flux v2](/docs/migration/helm-operator-migration/)
- [How to structure your HelmReleases](https://github.com/fluxcd/flux2-kustomize-helm-example)

## Release questions

### Where can I find information on how long versions are supported and how often to expect releases?

Releases are as needed for now and a release page with the schedule will be created when Flux reaches general availability status ([Flux roadmap can be found here](/roadmap/)). We understand the importance of having a release schedule and are aware that such a schedule will support the users' ability to plan for an upgrade in advance. For now you can expect the current release cadence of at least once per month to continue.
