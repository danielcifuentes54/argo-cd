<img src="icons/argo.svg" />

# ArgoCD
Information about ArgoCD

## GitOps

GitOps is a set of best practices where the entire code delivery process is controlled via Git, including infrastructure and application definition as code and automation to complete updates and rollbacks.

### The Key GitOps Principles:

* The entire system (infrastructure and applications) is described declaratively.
* The canonical desired system state is versioned in Git.
* Changes approved are automated and applied to the system.
* Software agents ensure correctness and alert on divergence.

### GitOps Use Cases

* Continuous deployment of applications
* Continuous deployment of cluster resources
* Detecting/Avoiding configuration drift
* Multi-cluster deployments

### GitOps Pros and Challenges

Pros
* Faster deployments
* Safer deployments
* Easier rollbacks
* Straightforward auditing
* Better traceability
* Eliminating configuration drift

Challenges

* Good testing and CI already in place
* A strategy for dealing with promotions between environments
* Secrets strategy

## ArgoCD

* You install Argo CD as a controller in the Kubernetes cluster. Usually you install Argo CD on the same cluster that it manages. It is also possible for Argo CD to manage external clusters.
* You store your manifests in Git. Argo CD is agnostic on the type of manifests you can use. It supports plain Kubernetes manifests, Helm charts, Kustomize definitions, and other templating mechanisms.
* You create an Argo CD application by defining which Git repository to monitor and to which cluster/namespace this application should be installed.
* From now on, Argo CD monitors the Git repository, and when there is a change, it automatically brings the cluster to the same state.
* Optionally Argo CD can deploy applications to other clusters (and not just the one on which it is installed).
* Argo CD contains an integrated UI that shows you the structure of your application as well as the synchronization status (whether the cluster matches Git at any point in time).
* Argo CD is just one member of the Argo family of projects. These are:
  * Argo CD (GitOps controller)
  * Argo Rollouts (Progressive Delivery controller)
  * Argo Workflows (Workflow engine for Kubernetes)
  * Argo Events (Event handling for Kubernetes)
* Installation: 
  * [argo autopilot(recommended)](https://github.com/argoproj-labs/argocd-autopilot) 
  * ```
    kubectl create namespace argocd
    kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml 
    ```
###  The reconciliation loop

By default, the sync period is 3 minutes. This means that every 3 minutes Argo CD will:

1. Gather all applications that are marked to auto-sync.
2. Fetch the latest Git state from the respective repositories.
3. Compare the Git state with the cluster state.
4. Take an action:
   * If both states are the same, do nothing and mark the application as synced.
   * If states are different mark the application as "out-of-sync".

>You can change the default period by editing the argocd-cm configmap found on the same namespace as Argo CD.

>[you can use Git webhooks](https://argo-cd.readthedocs.io/en/stable/operator-manual/webhook/)

###  Application Health

Apart from the synced/out-of-sync status, Argo CD also keeps track of the service health for each deployed application. The health status is shown in the UI and can also be retrieved by the CLI.

* Healthy
* Progressing
* Suspended
* Missing
* Degraded
* Unknown

> For custom Kubernetes resources, health is defined in Lua scripts


> Argo CD health checks are completely independent from [Kubernetes health probes](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/).

>Having a custom health check just provides a better experience with Argo CD. You can still deploy your custom application even without a health check, and it will never show as Healthy in the Argo CD dashboards.

###  Sync Strategies

There are 3 parameters that you can change when defining the sync strategy:
1. Manual or automatic sync.
2. Auto-pruning of resources (this is only applicable for automatic sync): defines what Argo CD does when you remove/delete files from Git. If it is enabled, Argo CD will also remove the respective resources in the cluster as well. If disabled, Argo CD will never delete anything from the cluster.
3. Self-Heal of cluster (this is only applicable for automatic sync): efines what Argo CD does when you make changes directly to the cluster (via kubectl or any other way).

> Adopting GitOps in its purest form may require organizational changes or a reexamination of policies and your organization may not be ready at this point.

###  Managing Secrets

* there is no single accepted practice for how secrets are managed with GitOps. If you already have a solid solution in place such as HashiCorp vault, then it would make sense to use that even though technically it is against GitOps practices.
* [Argo CD can be used with any secret solution that you already have deployed](https://argo-cd.readthedocs.io/en/stable/operator-manual/secret-management/)
* All solutions that handle secrets using Git are storing them in an encrypted form. This means that you can get the best of both worlds. Secrets can be managed with GitOps, and they can also be placed in a secure manner in any Git repository (even public repositories).
* 
> ```
> Example:
> 1. Install a tool like bitnami sealed secret
> 2. Encrypt the secrets:
> kubeseal < unsealed_secrets/db-creds.yml > sealed_secrets/db-creds-encrypted.yaml -o yaml
> kubeseal < unsealed_secrets/paypal-cert.yml > sealed_secrets/paypal-cert-encrypted.yaml -o yaml
> 3. Upload encrypted files to git
>```

###  Declarative Setup

* Argo CD comes with its own custom resources that can be stored in Git, and applied in a cluster using kubectl or even better Argo CD itself.
* Example - [more details](https://argo-cd.readthedocs.io/en/stable/operator-manual/declarative-setup/):
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: canary-demo
spec:
  destination:
    server: 'https://kubernetes.default.svc' 
    namespace: demo
  project: default
  source:
    repoURL: 'https://github.com/danielcifuentes54/argo-cd'
    path: ./01-simple-app
    targetRevision: HEAD
```
* you can have one repository for all the argo apps, and create just one application manually, and this application will control the rest of the applications

### Deploying with Helm

* ArgoCD provides native support for Helm, meaning you can directly connect a packaged Helm chart and Argo CD will monitor it for new versions. When this takes place, the Helm chart will no longer function as a Helm chart and instead, is rendered with the Helm template when Argo is installed, using the Argo CD application manifest.

* ArgoCD doesn't include the Helm payload information. When deploying a Helm application, Argo CD runs "helm template" and deploys the resulting manifests.

###  Deploying with Kustomize

* Both Kustomize and Argo CD are declarative tools for Kubernetes that follow the GitOps pattern and work well together. Argo CD supports Kustomize and has the ability to read a kustomization.yaml file. To deploy Kustomize with Argo CD, ensure the Kubernetes cluster is set up and you are logged into Argo CD so that these resources are provided and can be deployed.

### Progresive Delivery

* Progressive Delivery is the practice of deploying an application in a gradual manner allowing for minimum downtime and easy rollbacks. There are several forms of progressive delivery such as blue/green, canary, a/b and feature flags.

#### Argo Rollouts

Argo Rollouts is a progressive delivery controller created for Kubernetes. It allows you to deploy your application with minimal/zero downtime by adopting a gradual way of deploying instead of taking an “all at once” approach.

Argo Rollouts supercharges your Kubernetes cluster and in addition to the rolling updates you can now do:

* Blue/green deployments
* Canary deployments (It is necessary to have a service mesh)
* A/B tests
* Automatic rollbacks
* Integrated Metric analysis

* Automated Rollbacks with Metrics: While you can use canaries with simple pauses between the different stages, Argo Rollouts offers the powerful capability to look at application metrics and decide automatically if the deployment should continue or not. The idea behind this approach is to completely automate canary deployments. Instead of having a human running manual smoke tests, or looking at graphs, you can set different thresholds that define if a deployment is “successful” or not

## ArgoCD CLI

### Create an argo app
```bash
 argocd app create demo \
 --project default \
 --repo https://github.com/danielcifuentes54/argo-cd \
 --path "./05-helm-app/" \
 --sync-policy auto \
 --dest-namespace default \
 --dest-server https://kubernetes.default.svc
```
### Synchronizing an Argo CD application
```bash
argocd app sync {APP NAME}
```