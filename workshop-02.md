# Workshop Kubernetes

For this part of the workshop we are going to use `flux` for keeping our Git state in sync with our Kubernetes cluster.

## Download Flux CI

See for installation / download location: https://fluxcd.io/flux/installation/#install-the-flux-cli

Once installed verify the installation using

```bash
$ flux -v
flux version 2.1.2
```

## Start using Flux

Lets first check if our Kubernetes cluster is up and running using the `kubectl` command

```bash
$ kubectl get pods
No resources found in default namespace.
```

## Bootstrap flux in you cluster

See the documentation for bootstrapping flux: https://fluxcd.io/flux/installation/bootstrap/github/#github-personal-account

```bash
$ export GITHUB_TOKEN=<your personal github token>
$ flux bootstrap github \
  --token-auth \
  --owner=my-github-username \
  --repository=my-repository-name \
  --branch=main \
  --path=clusters/local \
  --personal

► connecting to github.com
► cloning branch "main" from Git repository "https://github.com/edwin-bruurs/fluxcd.git"
✔ cloned repository
► generating component manifests
✔ generated component manifests
✔ committed sync manifests to "main" ("cdfcc3b0d69058512f65bdcb7fece53961d32d83")
► pushing component manifests to "https://github.com/edwin-bruurs/fluxcd.git"
► installing components in "flux-system" namespace
✔ installed components
✔ reconciled components
► determining if source secret "flux-system/flux-system" exists
► generating source secret
► applying source secret "flux-system/flux-system"
✔ reconciled source secret
► generating sync manifests
✔ generated sync manifests
✔ committed sync manifests to "main" ("ba5c804e3d4dc6ffe43afb68458ae9620385d9bf")
► pushing sync manifests to "https://github.com/edwin-bruurs/fluxcd.git"
► applying sync manifests
✔ reconciled sync configuration
◎ waiting for Kustomization "flux-system/flux-system" to be reconciled
✔ Kustomization reconciled successfully
► confirming components are healthy
✔ helm-controller: deployment ready
✔ kustomize-controller: deployment ready
✔ notification-controller: deployment ready
✔ source-controller: deployment ready
✔ all components are healthy
```

Flux is now installed in your cluster. Verify this by taking a look at the created Github repository and inspect the pods running in the flux namespace `kubectl get pods -n flux-system`

```bash
$ kubectl get pods -n flux-system
NAME                                       READY   STATUS    RESTARTS   AGE
source-controller-78d959d874-4jrfs         1/1     Running   0          6m19s
notification-controller-7d7747dd84-pzbh9   1/1     Running   0          6m19s
kustomize-controller-5b79d7f755-mhblb      1/1     Running   0          6m19s
helm-controller-7f8469f4bc-f77xc           1/1     Running   0          6m19s
```

To see what the current status is of flux run

```bash
$ flux stats
RECONCILERS          	RUNNING	FAILING	SUSPENDED	STORAGE
GitRepository        	1      	0      	0        	35.3 KiB
OCIRepository        	0      	0      	0        	-
HelmRepository       	0      	0      	0        	-
HelmChart            	0      	0      	0        	-
Bucket               	0      	0      	0        	-
Kustomization        	1      	0      	0        	-
HelmRelease          	0      	0      	0        	-
Alert                	0      	0      	0        	-
Provider             	0      	0      	0        	-
Receiver             	0      	0      	0        	-
ImageUpdateAutomation	0      	0      	0        	-
ImagePolicy          	0      	0      	0        	-
ImageRepository      	0      	0      	0        	-
```

For the following steps it is useful to have the created git repository checkout out locally.

Next, we are going to add some additional resources to our flux-system. Create the following `app.yaml` file in `clusters/local`. This file will instruct Flux to add an additional pipeline for fetching and applying additional sources.

```yaml
---
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: apps
  namespace: flux-system
spec:
  interval: 10m0s
  sourceRef:
    kind: GitRepository
    name: flux-system
  path: ./apps
  prune: false
  wait: true
  timeout: 5m0s
  ```

And create a folder `apps` in the root of the repository.

Create a `Kustomization` spec as `apps/kustomization.yaml` containing the following content

```yaml
---
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
```

Add all files to git and push them to the main branch. 

See `flux stats` again to see that the Running Kustomization states `2`.  
Note this can take some minutes, to speed things up you can run `flux reconcile kustomization flux-system` and `flux reconcile kustomization apps` afterwards.

## Create a deployment

We can now recreate our podinfo deployment again, add the `deployment.yaml` and `service.yaml` files to the `apps` folder and update the `kustomization.yaml` file as follows

```yaml
---
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - ./deployment.yaml
  - ./service.yaml
```

Commit and push your changes again to the repository.

To speed things up a bit, execute a `flux reconcile kustomization apps` command once you pushed your changes to the repository.

To verify everything is starting to update run `kubectl get pods` and once you see pods running, you can test it out using `kubectl exec client -- curl -s podinfo.default.svc.cluster.local:9898`.

Congratulations you now deployed your first application using pull-based GitOps.

## Optional a GitOps Dashboard

Want to have a look at how your deployments are going? You can install a GitOps Dashboard.

Create a `weave-gitops-dashboard.yaml` file in the `clusters/local` folder

```yaml
---
apiVersion: source.toolkit.fluxcd.io/v1beta2
kind: HelmRepository
metadata:
  annotations:
    metadata.weave.works/description: This is the source location for the Weave GitOps
      Dashboard's helm chart.
  labels:
    app.kubernetes.io/component: ui
    app.kubernetes.io/created-by: weave-gitops-cli
    app.kubernetes.io/name: weave-gitops-dashboard
    app.kubernetes.io/part-of: weave-gitops
  name: ww-gitops
  namespace: flux-system
spec:
  interval: 1h0m0s
  type: oci
  url: oci://ghcr.io/weaveworks/charts
---
apiVersion: helm.toolkit.fluxcd.io/v2beta1
kind: HelmRelease
metadata:
  annotations:
    metadata.weave.works/description: This is the Weave GitOps Dashboard.  It provides
      a simple way to get insights into your GitOps workloads.
  name: ww-gitops
  namespace: flux-system
spec:
  chart:
    spec:
      chart: weave-gitops
      sourceRef:
        kind: HelmRepository
        name: ww-gitops
  interval: 1h0m0s
  values:
    adminUser:
      create: true
      passwordHash: $2a$10$/wzmFZv1FCoR1uvS1/Yn2uKYCeY/thKJ0LBQDPMWwlhFbQ2QyMhiy
      username: admin
```

Commit and push the changes again to your repository. Again to speed this up you can use `flux reconcile kustomization flux-system`.

You can explore the dashboard using `kubectl port-forward svc/ww-gitops-weave-gitops -n flux-system 9001:9001` and open your browser at https://localhost:9001.

Credentials: admin / q1w2e3r4t5