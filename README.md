# Part-1---ArgoCD-Technical-Implementation
This repository is used to store all configuration files, manifests, and documentation needed to run and review the solution I created.

## Architecture Overview

## Prerequisites

- [Docker](https://www.docker.com/get-started/) (v29.6.2)
- [Kubectl](https://kubernetes.io/docs/tasks/tools/#kubectl) (v1.36.1)
- [Kind](https://kind.sigs.k8s.io/docs/user/quick-start/#installing-from-release-binaries) (v0.32.0)
- [Helm](https://helm.sh/docs/intro/install/) (v4.2.3)

## How to Deploy


### Create a Kubernetes cluster

First, we must create a Kubernetes cluster in which Argo CD and our application will be deployed. In this assessment, we will use [kind](https://kind.sigs.k8s.io/docs/user/quick-start/#creating-a-cluster) to quickly deploy a cluster locally to our machine: `kind create cluster`.

This should take a minute or so to run and provision a simple single-node Kubernetes cluster that we can use.


### Deploy Argo CD & Reach the UI

To deploy Argo CD in a way where Argo CD manages itself, we first must take a few prerequisite steps.
1. Create a directory in our Git repo to host Argo CD's manifest files

```
mkdir -p argocd/install
```

2. Create the kustomization.yaml file with our targeted version (v3.4.5)

```
vim argocd/install/kustomization.yaml

---
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namespace: argocd
resources:
  - github.com/argoproj/argo-cd//manifests/cluster-install?ref=v3.4.5
```

3. Now we deploy Argo CD using the kustomization file we created:

```
#Creates the argocd namespace:
kubectl create namespace argocd

#Applies argocd based on the kustomization file created in step 2:
kubectl apply -n argocd --server-side --force-conflicts -k argocd/install/
```

4. Confirm pods in the Argo CD namespace reach 'Running' status

```
kubectl get pods -n argocd

---
NAME                                                READY   STATUS    RESTARTS   AGE
argocd-application-controller-0                     1/1     Running   0          2m7s
argocd-applicationset-controller-7d4d7c7b89-ftqmc   1/1     Running   0          2m8s
argocd-dex-server-cbddb9676-spbnw                   1/1     Running   0          2m8s
argocd-notifications-controller-7b55c64b69-nbmzf    1/1     Running   0          2m8s
argocd-redis-68bc658cfb-grh2x                       1/1     Running   0          2m8s
argocd-repo-server-56c67cf674-mr86f                 1/1     Running   0          2m8s
argocd-server-66b7d96445-dgghw                      1/1     Running   0          2m7s
```

5. Access the Argo CD UI:

```
#Get UI password:
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d

#use kubectl port-forward to quickly access the Argo CD UI
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

6. If you're hosting on a local machine, the UI should now be accessible via `https://localhost:8080`. From here, you can accept the self-signed certificate warning and sign in using the username `admin` and the password we retrieved from the last step.

If you've done everything correctly. You should be logged in into the Argo CD UI and see the applications page.


### Manage Argo CD with Argo CD

We want Argo CD to be "self-managed" so we need to make our Argo CD instance an Application. Using the template manifest below, create the application in a directory `apps/` in your Git repository.

```
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: argocd
  namespace: argocd
spec:
  project: default
  source:
    repoURL: <your-repoURL>
    path: argocd/install          # <-- points at the dir where kustomization.yaml lives
    targetRevision: main          # <-- our repo's branch, not the upstream stable tag"
  destination:
    server: https://kubernetes.default.svc
    namespace: argocd
  syncPolicy:
    automated:
      prune: false            
      selfHeal: true
    syncOptions:
      - ServerSideApply=true

#Template from https://argo-cd.readthedocs.io/en/stable/operator-manual/declarative-setup/#server-side-apply-requirement
```
Save this to your git repository and then apply the manifest inside your cluster with something like `kubectl apply -f <file-name-here>`. We need to use `kubectl apply` at first because ArgoCD can't manage an Application it doesn't know exists yet. Once the Argo CD appears as an application in the UI and you verify a simple test-sync works, feel free to go into your repo and change `prune:` to `true` for self-management.

>If you'd like to do a quick test to make sure that ArgoCD reacts to changes in your Git repository, you can add a simple label section to your kustomize.yaml file. Commit the change and push it to your git repository. Re-sync the application and see if the change is propagated:
```
labels:
  - pairs:
      test-label: loop-check
    includeSelectors: false
```
>Once tested, you can remove this label block and continue with the next steps.

### Bootstrap the root app-of-apps application

Now that we've created an application that allows Argo CD to 'self-manage' itself, we can deploy a root app-of-apps application that will use the `apps` directory we created earlier which will host all the application manifests we plan on creating.


Create a bootstrapping directory in your Git repo and add the following `.yaml` file:

```
# bootstrap/root-app.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: root
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io      #Cascading delete: removing root also removes its child apps
spec:
  project: default
  source:
    repoURL: <your-repoURL>
    path: apps                 # <-- the folder full of Application manifests
    targetRevision: main
    directory:
      recurse: true            # picks up nested apps too, future-proofing
  destination:
    server: https://kubernetes.default.svc
    namespace: argocd          # where Application CRs live
  syncPolicy:
    automated:
      prune: false             # Like before, flip to 'true' once tested        
      selfHeal: true
    syncOptions:
      - ServerSideApply=true
```

Once this is created, we need to paradoxically manually deploy the root application using `kubectl apply -f bootstrap/<file-name>.yaml` to initialize the root app-of-apps application. With this deployed, the root application will now manage the `apps/` directory and deploy new applications when new manifests become available.


### Deploy Prometheus, metrics, dashboards, and alerts

1. Deploy Prometheus, Grafana, and Alertmanager via Helm

We're going to use Helm to deploy Prometheus into our clusters. It's simple and will deploy Alertmanager, Grafana, Prometheus, and the necessary operators. Let's first confirm the latest versions of prometheus that are available using Helm

```
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
helm search repo kube-prometheus-stack --versions | head
```

On the day of performing this test, these were the chart versions available:

```
NAME                                      	CHART VERSION	APP VERSION	DESCRIPTION                                       
prometheus-community/kube-prometheus-stack	87.18.0      	v0.92.1    	kube-prometheus-stack collects Kubernetes manif...
prometheus-community/kube-prometheus-stack	87.17.0      	v0.92.1    	kube-prometheus-stack collects Kubernetes manif...
prometheus-community/kube-prometheus-stack	87.16.1      	v0.92.1    	kube-prometheus-stack collects Kubernetes manif...
prometheus-community/kube-prometheus-stack	87.16.0      	v0.92.1    	kube-prometheus-stack collects Kubernetes manif...
prometheus-community/kube-prometheus-stack	87.15.2      	v0.92.1    	kube-prometheus-stack collects Kubernetes manif...
prometheus-community/kube-prometheus-stack	87.15.1      	v0.92.1    	kube-prometheus-stack collects Kubernetes manif...
prometheus-community/kube-prometheus-stack	87.15.0      	v0.92.1    	kube-prometheus-stack collects Kubernetes manif...
prometheus-community/kube-prometheus-stack	87.14.0      	v0.92.1    	kube-prometheus-stack collects Kubernetes manif...
prometheus-community/kube-prometheus-stack	87.13.0      	v0.92.1    	kube-prometheus-stack collects Kubernetes manif...
```

Using this information, we will deploy `v87.18.0` of the chart, which we will reference in the application manifest for Prometheus:

```
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: monitoring
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: default
  source:
    repoURL: https://prometheus-community.github.io/helm-charts
    chart: kube-prometheus-stack
    targetRevision: 87.18.0          # <-- pin the exact version from helm search
    helm:
      values: |
        # kind single-node: disable control-plane scrape targets that aren't reachable
        kubeControllerManager:
          enabled: false
        kubeScheduler:
          enabled: false
        kubeEtcd:
          enabled: false
        kubeProxy:
          enabled: false
        # Let Prometheus discover ServiceMonitors in ALL namespaces, not just its own
        prometheus:
          prometheusSpec:
            serviceMonitorSelectorNilUsesHelmValues: false. # Used so Prometheus can scrape any ServiceMonitor
            ruleSelectorNilUsesHelmValues: false
  destination:
    server: https://kubernetes.default.svc
    namespace: monitoring
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - ServerSideApply=true        # REQUIRED — CRD annotation size limit
      - CreateNamespace=true        # makes the monitoring namespace for you if not already created
```

Create this file in the `apps/` directory of our Git repo and head over to the Argo CD UI to confirm that Prometheus is running under root.

2. Deploy ServiceMonitors for the argocd namespace resources

Next, we will want to deploy ServiceMonitors for the resources in the ArgoCD namespace so metrics can be collected for them. We will create the manifest for these ServiceMonitors under the `argocd/install` directory:

```
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: argocd-metrics
  namespace: argocd
spec:
  selector:
    matchLabels:
      app.kubernetes.io/name: argocd-metrics
  endpoints:
    - port: metrics
---
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: argocd-server-metrics
  namespace: argocd
spec:
  selector:
    matchLabels:
      app.kubernetes.io/name: argocd-server-metrics
  endpoints:
    - port: metrics
---
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: argocd-repo-server-metrics
  namespace: argocd
spec:
  selector:
    matchLabels:
      app.kubernetes.io/name: argocd-repo-server
  endpoints:
    - port: metrics
---
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: argocd-applicationset-controller-metrics
  namespace: argocd
spec:
  selector:
    matchLabels:
      app.kubernetes.io/name: argocd-applicationset-controller
  endpoints:
    - port: metrics
---
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: argocd-dex-server
  namespace: argocd
spec:
  selector:
    matchLabels:
      app.kubernetes.io/name: argocd-dex-server
  endpoints:
    - port: metrics
---
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: argocd-notifications-controller
  namespace: argocd
spec:
  selector:
    matchLabels:
      app.kubernetes.io/name: argocd-notifications-controller-metrics
  endpoints:
    - port: metrics
```

We will then reference this new file in the kustomization.yaml manifest:

```
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namespace: argocd
resources:
  - github.com/argoproj/argo-cd//manifests/cluster-install?ref=v3.4.5
  - <servicemonitors-file-name-here>.yaml      
```

Commit these changes to your Git repository and and synce your applications in the Argo CD UI.


## Design Decisions & Trade-offs
- Using kind to create the Kubernetes cluster
    - kind gives a reproducible local cluster ideal for evaluation and simple testing; a production deployment would likely target a managed cluster (EKS/GKE/AKS) or OpenShift for HA, real networking, and multi-node scale.
- Pinned Argo CD to a specific version (v3.4.5) rather than tracking `stable`
    - `stable` is a floating tag that moves as new releases land, which makes the desired state non-deterministic. Argo CD could reconcile itself toward a version we never chose or tested. Pinning a specific version  in Git as the source of truth makes the install reproducible and upgrades an auditable, reviewable commit. The trade-off is that we don't automatically get new releases, which mirrors real organizational practice.
- Using kubectl port-forward, we're able to easily access the Argo CD UI
    - This is a quick way to gain access but is only an option as long as the port-forward tunnel is open. In production environments, exposing Argo CD via something like an Ingress with proper DNS and TLS configured would be best practice.
- Using an app-of-app pattern and creating each manifest in the `/apps` directory in our repository.
    - app-of-apps is the foundational pattern (explicit, one-file-per-app); ApplicationSet is the scale-up (templated, generator-driven) that the [docs](https://argo-cd.readthedocs.io/en/stable/operator-manual/cluster-bootstrapping/#application-sets-and-cluster-labels-recommended) now recommend for bootstrapping.

## Assumptions

- One heavy assumption that I had was the ability to use something like Claude to verify, troubleshoot, and parse information as needed.