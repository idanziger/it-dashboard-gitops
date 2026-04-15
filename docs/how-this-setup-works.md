# How This Setup Works

## Purpose

This document explains the `IT-Dashboard` GitOps lab so it is easy to understand later or on a new computer.

It answers:

- what this repo is for
- how it connects to the app repo
- how deployment works
- what files matter
- what was already done conceptually
- what you need to recreate or verify on another machine

---

## The Two-Repo Model

This setup uses two separate GitHub repositories:

### 1. App Repo

Repo: `https://github.com/idanziger/IT-Dashboard`

This is where the application code lives:

- Next.js / TypeScript app
- Dockerfile
- GitHub Actions workflow
- application logic and UI

This repo is owned by the application development workflow.

### 2. GitOps Repo

Repo: `https://github.com/idanziger/it-dashboard-gitops`

This is the repo you are in now.

This repo contains only deployment state:

- Helm chart
- Kubernetes manifests via Helm templates
- ArgoCD Application definition
- deployment configuration like image name and tag

This repo is the desired state for the cluster.

---

## High-Level Deployment Flow

This is the operating model:

```text
You edit code in IT-Dashboard
        |
        v
Push to GitHub
        |
        v
GitHub Actions runs in the app repo
        |
        v
Build Docker image
        |
        v
Push image to ghcr.io
        |
        v
Update image tag in this GitOps repo
        |
        v
ArgoCD detects the Git change
        |
        v
ArgoCD syncs the cluster to match this repo
        |
        v
Kubernetes runs the new version of IT-Dashboard
```

The core idea is:

- app repo builds artifacts
- GitOps repo declares what should run
- ArgoCD enforces that desired state

---

## What ArgoCD Does Here

ArgoCD is the CD engine.

Its job in this lab is:

- watch this GitOps repo
- look at the folder `environments/local/it-dashboard`
- compare Git state to cluster state
- apply changes automatically
- fix drift if someone changes resources manually in the cluster

The ArgoCD application definition is here:

- [applications/it-dashboard.yaml](/Users/Ilan/src/personal/it-dashboard-gitops/applications/it-dashboard.yaml:1)

Important current behavior from that file:

- repo watched: `https://github.com/idanziger/it-dashboard-gitops.git`
- branch watched: `main`
- chart path watched: `environments/local/it-dashboard`
- deploy namespace: `it-dashboard`
- automated sync enabled
- `prune: true`
- `selfHeal: true`
- namespace auto-create enabled

That means:

- if Git changes, ArgoCD deploys it
- if cluster state drifts from Git, ArgoCD tries to correct it
- if a resource is removed from Git, ArgoCD can remove it from the cluster

---

## What Helm Does Here

Helm is being used as a templating layer for Kubernetes manifests.

This repo contains a small Helm chart here:

- [environments/local/it-dashboard/Chart.yaml](/Users/Ilan/src/personal/it-dashboard-gitops/environments/local/it-dashboard/Chart.yaml:1)

The main config file is:

- [environments/local/it-dashboard/values.yaml](/Users/Ilan/src/personal/it-dashboard-gitops/environments/local/it-dashboard/values.yaml:1)

That file controls things like:

- image repository
- image tag
- port
- replica count
- selected runtime env vars

The templates are:

- [environments/local/it-dashboard/templates/deployment.yaml](/Users/Ilan/src/personal/it-dashboard-gitops/environments/local/it-dashboard/templates/deployment.yaml:1)
- [environments/local/it-dashboard/templates/service.yaml](/Users/Ilan/src/personal/it-dashboard-gitops/environments/local/it-dashboard/templates/service.yaml:1)

In practice:

- `values.yaml` is the input
- Helm renders the Kubernetes YAML
- ArgoCD applies it to the cluster

---

## What Kubernetes Runs

This setup currently deploys:

### Deployment

The Deployment runs the `it-dashboard` application container.

Current expectations in the template:

- deployment name: `it-dashboard`
- namespace: `it-dashboard`
- replica count comes from `values.yaml`
- container image comes from:
  - repository: `ghcr.io/idanziger/it-dashboard`
  - tag: stored in `values.yaml`
- container port: `3000`

### Service

The Service gives the app a stable internal address inside Kubernetes.

Current expectations:

- service name: `it-dashboard`
- namespace: `it-dashboard`
- service type: `ClusterIP`
- service port: `3000`
- target port: `3000`

Because this is `ClusterIP`, local access is typically done with port-forwarding.

---

## Current Runtime Configuration Model

There are two types of configuration in this repo:

### 1. Non-secret config in `values.yaml`

Examples:

- image repository
- image tag
- pull policy
- port
- replicas
- `NEXTAUTH_URL`
- `NODE_ENV`
- `AUTH_TRUST_HOST`

### 2. Secret config from Kubernetes Secrets

The Deployment expects a Kubernetes Secret named `it-dashboard-secrets`.

It reads values for:

- `NEXTAUTH_SECRET`
- `AUTH_ALLOWED_EMAIL`
- `GOOGLE_CLIENT_ID`
- `GOOGLE_CLIENT_SECRET`
- `GOOGLE_ADMIN_IMPERSONATE_EMAIL`
- `SENTINELONE_BASE_URL`
- `SENTINELONE_API_TOKEN`
- `JUMPCLOUD_API_KEY`

It also mounts:

- `google-service-account.json`

inside the container at:

- `/secrets/google-service-account.json`

This means the app will not work correctly unless that secret exists in the cluster.

---

## What Was Done Conceptually

The setup that this repo represents was built around a local learning lab:

- local Kubernetes environment chosen: `minikube`
- deployment model chosen: GitOps
- CD engine chosen: ArgoCD
- packaging model chosen: Docker image pushed to `ghcr.io`
- manifest structure chosen: Helm chart in GitOps repo
- app and infrastructure state intentionally split into two repos

Operationally, the intended path was:

1. Build the app image in the app repo.
2. Push the image to GitHub Container Registry.
3. Update the image tag in this repo.
4. Let ArgoCD redeploy automatically.

That means this repo is not where app code changes happen.

This repo is where deployment state changes happen.

---

## File Map

These are the key files in this repo:

### ArgoCD Pointer

- [applications/it-dashboard.yaml](/Users/Ilan/src/personal/it-dashboard-gitops/applications/it-dashboard.yaml:1)

Tells ArgoCD what repo/folder to watch and where to deploy it.

### Helm Chart Metadata

- [environments/local/it-dashboard/Chart.yaml](/Users/Ilan/src/personal/it-dashboard-gitops/environments/local/it-dashboard/Chart.yaml:1)

Defines the chart as an application chart.

### Deployment Values

- [environments/local/it-dashboard/values.yaml](/Users/Ilan/src/personal/it-dashboard-gitops/environments/local/it-dashboard/values.yaml:1)

Main configuration file. This is where the image tag is expected to change over time.

### Kubernetes Deployment Template

- [environments/local/it-dashboard/templates/deployment.yaml](/Users/Ilan/src/personal/it-dashboard-gitops/environments/local/it-dashboard/templates/deployment.yaml:1)

Defines how the app container runs in Kubernetes.

### Kubernetes Service Template

- [environments/local/it-dashboard/templates/service.yaml](/Users/Ilan/src/personal/it-dashboard-gitops/environments/local/it-dashboard/templates/service.yaml:1)

Defines the stable network service for the app inside the cluster.

### Prompt Reference

- [prompts/devops-teacher.md](/Users/Ilan/src/personal/it-dashboard-gitops/prompts/devops-teacher.md:1)

Reusable learning prompt plus repo-specific context.

---

## Safety Rule

This lab was built on a machine that also had access to real company Kubernetes contexts.

Because of that, the main safety rule is:

```bash
kubectl config current-context
```

It must show:

```bash
minikube
```

If it does not, switch before doing anything cluster-changing.

That applies before commands like:

- `kubectl apply`
- `kubectl delete`
- `kubectl create`
- `helm install`
- `helm upgrade`

---

## How To Recreate This On A New Computer

If you move to another machine, think in layers.

### Layer 1: Local Tools

You will need the basic local tooling again:

- `git`
- `docker`
- `kubectl`
- `helm`
- `minikube`
- `argocd` CLI if you want CLI access
- `gh` if you want GitHub CLI workflows

### Layer 2: Repos

Clone both repos:

- `IT-Dashboard`
- `it-dashboard-gitops`

### Layer 3: Local Cluster

Recreate or verify:

- `minikube` is running
- current context is `minikube`
- ArgoCD is installed in the cluster
- the `argocd` namespace exists

### Layer 4: Secrets

Recreate the Kubernetes secret material that the app expects:

- OAuth secrets
- Google service account JSON
- SentinelOne token
- JumpCloud API key
- auth-related secrets

Without these, the deployment can exist but the app will be broken or partially nonfunctional.

### Layer 5: ArgoCD Wiring

Reapply or verify the ArgoCD Application:

- [applications/it-dashboard.yaml](/Users/Ilan/src/personal/it-dashboard-gitops/applications/it-dashboard.yaml:1)

ArgoCD must be able to access:

- this GitOps repo
- the `main` branch
- the `environments/local/it-dashboard` path

### Layer 6: Access Path

You may need to port-forward the service locally, depending on how you want to access the app.

The app config currently assumes:

- `http://localhost:3000`

---

## What To Verify First On A New Machine

These are the first checks worth doing:

1. Confirm the current Kubernetes context is `minikube`.
2. Confirm `minikube` is running.
3. Confirm ArgoCD is installed and reachable.
4. Confirm the `it-dashboard` namespace exists.
5. Confirm the required secret `it-dashboard-secrets` exists.
6. Confirm ArgoCD sees the app and reports it as synced or at least healthy enough to debug.
7. Confirm the deployed image tag matches the tag in `values.yaml`.

---

## Short Mental Model

If you forget everything else, remember this:

- `IT-Dashboard` builds the software
- `it-dashboard-gitops` declares what version should run
- ArgoCD watches this repo and applies it to Kubernetes
- `values.yaml` is the most important day-to-day config file in this repo
- secrets are not fully represented here and must exist in the cluster separately

That is the core of the setup.
