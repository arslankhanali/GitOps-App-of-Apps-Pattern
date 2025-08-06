+++
date = '2025-08-06T18:00:00+10:00'
title = 'Argocd App Of Apps Pattern'
series = ['homelab']
tags = ['argocd','gitops','kubernetes']
topics = ['deployment']
weight = 1
indexable = true
featured = false
draft = true
+++

# ArgoCD - App of Apps Pattern

## Why

The **App of Apps** pattern is a scalable GitOps strategy where a single ArgoCD application (the "root app") manages everything on kubernetes declaratively.  
It enables:

- Manage everything via Git — including ArgoCD itself
- Disaster Recovery
- Zero Drift
- Zero touch provisioning

## Prerequisites

- A running Kubernetes or K3s cluster
- Installed CLIs: `kubectl`, `helm`, `kustomize`, `argocd`
- You can use my baseline git repo `https://github.com/arslankhanali/pattern-app-of-apps`

## Git Repo Structure
folder | use 
Applications | put your app templates, helm charts, kustomize here
Argocd | is for ...
```
tree pattern-app-of-apps
pattern-app-of-apps
├── apps
│   ├── argocd
│   │   ├── base
│   │   │   ├── kustomization.yaml
│   │   │   └── values.yaml
│   │   └── overlays
│   │       ├── k3s
│   │       │   ├── ingress.yaml
│   │       │   └── kustomization.yaml
│   │       └── openshift
│   │           └── kustomization.yaml
│   ├── hello-world-kustomize
│   │   ├── base
│   │   │   ├── deployment.yaml
│   │   │   ├── kustomization.yaml
│   │   │   └── service.yaml
│   │   ├── overlays
│   │   │   ├── k3s
│   │   │   │   ├── ingress.yaml
│   │   │   │   └── kustomization.yaml
│   │   │   └── openshift
│   │   │       └── kustomization.yaml
│   │   └── README.md
│   └── kubernetes-dashboard
│       ├── base
│       │   ├── kustomization.yaml
│       │   ├── serviceaccount.yaml
│       │   └── values.yaml
│       └── overlays
│           ├── k3s
│           │   ├── ingress.yaml
│           │   └── kustomization.yaml
│           └── openshift
│               └── kustomization.yaml
├── env
│   ├── k3s
│   │   ├── platform
│   │   │   ├── argocd.yaml
│   │   │   └── kubernetes-dashboard.yaml
│   │   └── workloads
│   │       └── hello-world-kustomize.yaml
│   └── openshift
│       ├── platform
│       │   ├── argocd.yaml
│       │   └── kubernetes-dashboard.yaml
│       └── workloads
│           └── hello-world-kustomize.yaml
├── README.md
└── root-app.yaml
```

Each child app points to a specific path in the Git repo. The root `all-apps.yaml` declares them as ArgoCD Applications.

## Bootstrapping

```sh
# Set kubeconfig to your cluster
export KUBECONFIG=~/k3s-config

# Add ArgoCD Helm repo
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update

# Create ArgoCD namespace
kubectl create namespace argocd

# Install ArgoCD via Helm using custom values
helm install argocd argo/argo-cd -n argocd -f apps/argocd/base/values.yaml

# Apply ingress if defined
kubectl apply -f apps/argocd/overlays/k3s/ingress.yaml
```

## Deploy App of Apps

Once ArgoCD is up and running:

```sh
# Apply the root application (declares all child apps)
kubectl apply -f root-app.yaml
```

ArgoCD will now sync the defined child apps automatically, including itself (self-healing).  

## Summary

- The root `Application` points to a directory with `argocd.yaml`, `kubernetes-dashboard.yaml`, etc.
- Each child `Application` deploys one app (via Helm or Kustomize).
- This pattern supports scaling, environment segregation, and GitOps best practices.

## Next

- Use ArgoCD Projects to isolate apps by teams/env
- Integrate SSO, RBAC, and ArgoCD Notifications
- Secure the GitOps flow (GPG signing, image scanning)

```tip
The App of Apps is not just a pattern—it's your GitOps blueprint for managing complex clusters cleanly and declaratively.
```
