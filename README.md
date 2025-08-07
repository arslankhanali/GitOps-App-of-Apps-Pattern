# Mastering Kubernetes Deployments with tGitOps

Managing multiple Kubernetes clusters and applications can get complex fast. **GitOps** helps tame this complexity—and the **App of Apps pattern** takes it to the next level with declarative, scalable, and automated infrastructure management.

---

## Why Use the App of Apps Pattern?

To live like this
![relax](https://images.unsplash.com/photo-1496046744122-2328018d60b6?q=80&w=2374&auto=format&fit=crop&ixlib=rb-4.1.0&ixid=M3wxMjA3fDB8MHxwaG90by1wYWdlfHx8fGVufDB8fHx8fA%3D%3D)

Also, The App of Apps is a GitOps design where a **single root ArgoCD application** manages multiple child applications, each representing a platform component or workload. It offers:

- **Declarative control** :  Everything is defined in Git.
- **Zero-touch provisioning** :  GitOps installs and configures your entire stack.
- **Environment-specific overlays** :  Adapt configurations for K3s, OpenShift, Dev, Prod etc.
- **Disaster recovery** :  Rebuild any where
- **Auditable changes** :  Every change is a Git commit.
- **No drift** :  GitOps continuously reconciles desired vs. actual state.
- **Self Healing** :  Accidently deleted something ? Let GitOps fix it for you.

---

## Explain whay is Gitops
## Explain what is app of app patterm and how it works

---
# Let's Deploy everything in seconds now
## Prerequisites to Deploy

- A Kubernetes cluster: This demo is tested on `K3s` but should work on any cluster
- CLI tools :  `kubectl`
- Fork git repo from :  [`arslankhanali/pattern-app-of-apps`](https://github.com/arslankhanali/pattern-app-of-apps)
- 
## 1. Install argocd on your Kubernetes cluster

```bash
export KUBECONFIG=~/k3s-config  # or OpenShift config

helm repo add argo https://argoproj.github.io/argo-helm
helm repo update
kubectl create namespace argocd
helm install argocd argo/argo-cd -n argocd -f values.yaml
```

Apply environment-specific ingress for argocd : 

```bash
# K3s
kubectl apply -f ingress.yaml

# OpenShift
kubectl apply -f apps/argocd/overlays/openshift/route.yaml
```

---
## 2. Set DNS locally
Make sure your `/etc/hosts` file has following entries.    
```
# sudo vim /etc/hosts
<K3s-cluster-IP> k3s.node1 argocd.node1 test.node1 hello.node1
```
## 3. Login to Argo dashboard
You will see apps getting deployed here.
- Argocd [argocd.node1](https://argocd.node1)
```bash
# Get Login password for admin user
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d
```

## 4. Unleash everything
This points to k3s right now
```bash
kubectl apply -f root-app.yaml
```
![oprah](/argocd-app-of-apps/oprah.png)
#### Access apps
- Kubernetes Dashboard [k3s.node1](https://k3s.node1)
```
# Get Bearer Token
kubectl -n kubernetes-dashboard create token admin-user
```
- Guestbook [test.node1](http://test.node1)
- Podinfo [hello.node1](http://hello.node1)
 
# ArgoCD will : 
1. Sync the `env/{k3s}/` directory.
2. Create child applications in {platform & workloads} folders.
3. Deploy all components declaratively.

This pattern allows full cluster rebuilds and updates via Git commits alone.
![alt text](/argocd-app-of-apps/argocd.png)

---

## Delete All
```sh
# delete all argocd apps
for app in $(kubectl get applications -n argocd -o jsonpath='{.items[*].metadata.name}'); do
  kubectl patch application "$app" -n argocd -p '{"metadata":{"finalizers":[]}}' --type=merge
  kubectl delete application "$app" -n argocd --force --grace-period=0
done

kubectl delete ns argocd
kubectl delete ns kubernetes-dashboard
kubectl delete ns podinfo
```
---

## Summary

The ArgoCD App of Apps pattern offers a scalable, Git-driven blueprint for managing Kubernetes clusters : 

- Manage everything declaratively in Git
- Scale across environments like K3s and OpenShift
- Rebuild or recover your clusters on demand

> The App of Apps pattern isn't just a tool—it's a mindset shift for cloud-native GitOps. Adopt it to bring structure, repeatability, and security to your infrastructure.

---

# Appendix
## Repository Structure Overview

```bash
├── apps             # Apps & workload YAMLS, Helm charts or Kustomize can go here
│   ├── guestbook    # Sample App from https://github.com/argoproj/argocd-example-apps/tree/master/kustomize-guestbook
│   │   ├── base
│   │   └── overlays
│   ├── kubernetes-dashboard    # Upstream K8s dashboard https://kubernetes.io/docs/tasks/access-application-cluster/web-ui-dashboard/
│   │   ├── base
│   │   └── overlays
│   └── podinfo       # Sample App from https://github.com/stefanprodan/podinfo/tree/master/kustomize
│       ├── base
│       └── overlays
├── env               # ArgoCD Applications - Folders can be Cluster-specific (k3s,openshift) or Env Specific (dev,
│   ├── k3s
│   │   ├── kustomization.yaml
│   │   ├── platform
│   │   └── workloads
│   └── openshift
│       ├── kustomization.yaml
│       ├── platform
│       └── workloads
├── ingress.yaml      # Ingress to access ArgoCD dashboard
├── README.md
├── root-app.yaml     # Root ARGOCD application
└── values.yaml       # Deploy Argo with insecure access (needed for Ingress) & enable Helm for kustomize
```
![as](/argocd-app-of-apps/flow1.png)
### 1. `apps/` – Add your Apps in a folder here

I have 3 apps here as an example : 
- guestbook :  Kustomize based app [argocd-kustomize-guestbook](https://github.com/argoproj/argocd-example-apps/tree/master/kustomize-guestbook) 
- kubernetes-dashboard/ :  Kustomize calls Helm to install K8s dashboard for K3s.
- podinfo :  Kustomize based app [stefanprodan-podinfo](https://github.com/stefanprodan/podinfo/tree/master/kustomize)

> You can use YAML manifests, kustomize or Helm charts to add more applications in this folder. 

Each app follows : 

```sh
apps/
  └── <app1>/
      ├── base/
      └── overlays/
          ├── <env1-name>/    # <--- Can Change Name - e.g. DEV
          └── <env2-name>/    # <--- Can Change Name - e.g. PROD
```

### 2. `env/` – Create your ARGOCD APPLICATIONS here for your env
"ArgoCD Application" definitions for different environments. They basically call different overlays in apps.

- env/k3s/ :  Deploys K8s Dashboard and uses `Ingress` for apps
- env/openshift/  :  No K8s Dashboard and uses `Route` for apps

Each env follows : 
```sh
── env
│   ├── <env1-name>
│   │   ├── kustomization.yaml
│   │   ├── platform  # <--- Can Change Name - Just used to categorise `argocd-application`
│   │   │   └── 'argocd-application-for-app1'.yaml
│   │   └── workloads # <--- Can Change Name - Just used to categorise `argocd-application`
│   │       ├── 'argocd-application-for-app2'.yaml
│   │       └── 'argocd-application-for-app3'.yaml
```
### 3. `root-app.yaml` – The Orchestrator

> Main reason this pattern is called `APP OF APPS`.   

This `top-level ArgoCD Application` points to env/{k3s} and deploys all children `ArgoCD Application` in it.
