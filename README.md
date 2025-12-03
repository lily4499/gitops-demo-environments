# gitops-demo-environments

GitOps environment configuration for the **gitops-demo-app** project.

This repo contains:

- A Helm chart under `charts/gitops-demo-app/`
- Environment folders:
  - `dev/` (auto-sync, 1 replica, NodePort 30081)
  - `prod/` (intended manual sync, 3 replicas, NodePort 30082)
- An ArgoCD `ApplicationSet` to automatically create one ArgoCD Application per environment folder.

Update the following before applying to a real cluster:

- `image.repository` and `image.tag` in `values.yaml` / `dev/values.yaml` / `prod/values.yaml`
- `repoURL` fields in `applicationset/appset.yaml` to point to your own GitHub repository.

---
---
---

````markdown
# GitOps Demo – Helm + ArgoCD ApplicationSet (Dev & Prod)

> This README contains **only CLI commands**, ordered by step, to build the full project from scratch.

---

## Step 1 – Clone Repositories

```bash
# Clone application repository
git clone https://github.com/<github-user>/gitops-demo-app.git

# Clone GitOps environments repository
git clone https://github.com/<github-user>/gitops-demo-environments.git

# Go into GitOps repo (most commands below assume this as base)
cd gitops-demo-environments
````

---

## Step 2 – Start Kubernetes Cluster (Example: Minikube)

```bash
# Start Minikube
minikube start --memory=4096 --cpus=4

# Verify cluster
kubectl get nodes
```

---

## Step 3 – Install ArgoCD

```bash
# Create ArgoCD namespace
kubectl create namespace argocd

# Install ArgoCD
kubectl apply -n argoccd \
  -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Check ArgoCD pods
kubectl get pods -n argocd

# Port-forward ArgoCD UI
kubectl port-forward svc/argocd-server -n argocd 8080:443

# Get initial admin password
kubectl get secret argocd-initial-admin-secret -n argocd \
  -o jsonpath="{.data.password}" | base64 -d

# Login to ArgoCD from CLI (optional)
argocd login localhost:8080 --username admin --password <ARGOCD_PASSWORD> --insecure
```

---

## Step 4 – Build & Push Initial Application Image

```bash
# Go to application repo
cd ../gitops-demo-app

# (Optional) Check status
git status

# Build image
docker build -t <dockerhub-username>/gitops-demo-app:v1 .

# Login to Docker registry (if needed)
docker login

# Push image
docker push <dockerhub-username>/gitops-demo-app:v1
```

---

## Step 5 – Create Helm Chart in GitOps Repo

```bash
# Go back to GitOps repo
cd ../gitops-demo-environments

# Create Helm chart (in current directory)
helm create gitops-demo-app

# Create charts folder if you want to nest it
mkdir -p charts

# Move chart under charts/ (optional but common pattern)
mv gitops-demo-app charts/

# Verify chart structure
ls charts/gitops-demo-app
ls charts/gitops-demo-app/templates

# Stage and commit chart
git add charts/
git status
git commit -m "Add Helm chart for gitops-demo-app"
git push
```

---

## Step 6 – Create Dev & Prod Environment Folders

```bash
# Make environment folders
mkdir dev prod

# Create placeholder files (contents edited later)
touch dev/values.yaml
touch dev/patch-sync.yaml
touch prod/values.yaml
touch prod/patch-sync.yaml

# Verify structure
ls dev
ls prod

# Stage and commit
git add dev prod
git status
git commit -m "Add dev and prod environment folders"
git push
```

---

## Step 7 – Create ApplicationSet

>What this does
  >Base template
      >All apps (dev, prod, staging…)  
      >Use the same Helm chart at charts/gitops-demo-app
      >Use ../../<env>/values.yaml for that environment
      >Deploy to namespace gitops-demo-<env>
      >Have manual sync by default, with CreateNamespace=true.
  >templatePatch logic
    >If folder name is dev:
        >Override syncPolicy → turn auto-sync ON (prune + self-heal).
    >If folder name is prod:
        >Add ignoreDifferences for /spec/replicas so Prod doesn’t constantly show drift if you scale manually/HPA.

```bash
# Create folder for ApplicationSet
mkdir applicationset

# Create ApplicationSet file (edit contents separately)
touch applicationset/appset.yaml

# Stage and commit
git add applicationset/appset.yaml
git status
git commit -m "Add ArgoCD ApplicationSet definition"
git push

# Apply ApplicationSet in ArgoCD namespace
kubectl apply -f applicationset/appset.yaml -n argocd

# List ArgoCD applications
argocd app list
```

---

## Step 8 – Verify Initial Deployments (Dev & Prod)

```bash
# Check namespaces created by ArgoCD (CreateNamespace option)
kubectl get ns | grep gitops-demo

# Check Dev pods
kubectl get pods -n gitops-demo-dev

# Check Prod pods
kubectl get pods -n gitops-demo-prod

# Check Dev services
kubectl get svc -n gitops-demo-dev

# Check Prod services
kubectl get svc -n gitops-demo-prod

# If using Minikube, get Dev URL
minikube service -n gitops-demo-dev dev-gitops-demo-gitops-demo-app --url

# If using Minikube, get Prod URL
minikube service -n gitops-demo-prod prod-gitops-demo-gitops-demo-app --url
```

---

## Step 9 – Deploy New Version to Dev (Auto-Sync)

```bash
# Go to app repo
cd ../gitops-demo-app

# (Optional) Check current files
git status

# Edit application source to represent "v2" (done in editor, not CLI)

# Build new image
docker build -t <dockerhub-username>/gitops-demo-app:v2 .

# Push new image
docker push <dockerhub-username>/gitops-demo-app:v2

# Go back to GitOps repo
cd ../gitops-demo-environments

# Edit dev/values.yaml to set image.tag: v2 (done in editor, not CLI)

# Stage and commit Dev change
git add dev/values.yaml
git status
git commit -m "Dev: upgrade image tag to v2"
git push

# Check Dev application in ArgoCD
argocd app get dev-gitops-demo

# (Optional) Force refresh
argocd app sync dev-gitops-demo

argocd app history dev-gitops-demo

# Verify Dev pods
kubectl get pods -n gitops-demo-dev

# Verify Dev URL again
minikube service -n gitops-demo-dev gitops-demo-app --url
```

---

## Step 10 – Promote New Version to Prod (Manual Sync)

```bash
# Ensure you are in GitOps repo
cd gitops-demo-environments

# Create feature branch for Prod change (optional but recommended)
git checkout -b feature/prod-v2

# Edit prod/values.yaml to set image.tag: v2 (done in editor, not CLI)

# Stage and commit
git add prod/values.yaml
git status
git commit -m "Prod: prepare upgrade to v2"

# Push feature branch
git push --set-upstream origin feature/prod-v2

# (In GitHub UI: Create PR from feature/prod-v2 → main, review & merge)

# Back on local: switch to main and pull latest
git checkout main
git pull

# Check Prod app status
argocd app get prod-gitops-demo

# Prod should show OutOfSync. Manually sync:
argocd app sync prod-gitops-demo

# Verify Prod pods
kubectl get pods -n gitops-demo-prod

# Verify Prod URL
minikube service -n gitops-demo-prod gitops-demo-app --url
```

---

## Step 11 – Add a New Environment (Staging Example)

```bash
# Ensure you are in GitOps repo
cd gitops-demo-environments

# Create staging folder
mkdir staging

# Copy Dev config as a starting point
cp dev/values.yaml staging/values.yaml
cp dev/patch-sync.yaml staging/patch-sync.yaml

# (Optionally edit staging/values.yaml and staging/patch-sync.yaml in editor)

# Stage and commit
git add staging
git status
git commit -m "Add staging environment"
git push

# Check ArgoCD apps (staging-gitops-demo should appear)
argocd app list

# Check namespaces
kubectl get ns | grep gitops-demo

# Check staging pods
kubectl get pods -n gitops-demo-staging

# Check staging services
kubectl get svc -n gitops-demo-staging

# If using Minikube, get staging URL
minikube service -n gitops-demo-staging gitops-demo-app --url
```

---

## Step 12 – Useful Debug Commands

```bash
# Get detailed info about an ArgoCD app
argocd app get dev-gitops-demo
argocd app get prod-gitops-demo
argocd app get staging-gitops-demo

# Show ArgoCD app history
argocd app history dev-gitops-demo
argocd app history prod-gitops-demo

# Re-sync an app (if needed)
argocd app sync dev-gitops-demo
argocd app sync prod-gitops-demo
argocd app sync staging-gitops-demo

# Delete an app (if needed)
argocd app delete dev-gitops-demo
argocd app delete prod-gitops-demo
argocd app delete staging-gitops-demo

# General Kubernetes checks
kubectl get pods -A
kubectl get svc -A
kubectl describe pod <POD_NAME> -n <NAMESPACE>
kubectl logs <POD_NAME> -n <NAMESPACE>
```

---


### ❓“What was the most challenging part?”

> “The most challenging part was getting ArgoCD ApplicationSet working exactly how I wanted for multiple environments.
>
> I had to understand how the git **directories generator** works — how it maps each folder to an Application — and then make sure the template pointed to the Helm chart path while still loading env-specific `values.yaml` files.
> I also needed to think about **sync policy**: Dev should auto-sync, but Prod must be manual. That forced me to think like production: where do we allow automation and where do we require human approval.”

---

---









