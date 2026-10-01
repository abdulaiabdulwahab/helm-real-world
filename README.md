# helm-real-world

# Real-World Helm Deployment on Azure AKS

## Overview

This project demonstrates how to use **Helm** to deploy and manage applications on **Azure Kubernetes Service (AKS)**.

The project focuses on creating reusable Helm charts, managing environment-specific values, deploying releases, performing upgrades and rollbacks, and storing packaged Helm charts in **Azure Container Registry (ACR)** as OCI artifacts.

A dedicated Azure Resource Group is used to organize and manage all Azure resources created for the project.

---

## Project Architecture

```text
Azure Subscription
        │
        ▼
Resource Group
rg-helm-project
        │
        ├───────────────┐
        ▼               ▼
       ACR             AKS
        │               │
        │               ▼
        │          Kubernetes
        │               │
        └──────────► Helm Releases
```

---

## Technologies Used

- Helm
- Kubernetes
- Azure Kubernetes Service (AKS)
- Azure Container Registry (ACR)
- Azure CLI
- kubectl
- NGINX

---

## Skills Demonstrated

This project covers:

- Azure Resource Group creation and management
- AKS cluster creation
- ACR creation
- Connecting `kubectl` to AKS
- Helm chart creation
- Helm templates
- Helm values
- Environment-specific configuration
- Helm chart validation
- Helm release installation
- Helm upgrades
- Helm release history
- Helm rollback
- Helm chart packaging
- OCI Helm charts
- Pushing Helm charts to ACR
- Installing Helm charts from ACR
- Kubernetes and Helm troubleshooting
- Azure resource cleanup

---

## Project Structure

```text
helm-production-project/
│
├── charts/
│   └── devops-web/
│       ├── Chart.yaml
│       ├── values.yaml
│       ├── values-dev.yaml
│       ├── values-staging.yaml
│       ├── values-prod.yaml
│       │
│       └── templates/
│           ├── deployment.yaml
│           └── service.yaml
│
└── dist/
    ├── devops-web-0.1.0.tgz
    └── devops-web-0.2.0.tgz
```

---

## Azure Resource Group

The project uses a dedicated Resource Group:

```bash
RG="rg-helm-project"
LOCATION="canadacentral"
```

Create the Resource Group:

```bash
az group create \
  --name "$RG" \
  --location "$LOCATION"
```

The Resource Group contains the Azure resources used by the project, including:

```text
rg-helm-project
│
├── Azure Container Registry
└── Azure Kubernetes Service
```

---

## Create Azure Container Registry

Create the registry:

```bash
az acr create \
  --resource-group "$RG" \
  --name "$ACR_NAME" \
  --location "$LOCATION" \
  --sku Basic
```

Get the registry login server:

```bash
ACR_LOGIN_SERVER=$(az acr show \
  --resource-group "$RG" \
  --name "$ACR_NAME" \
  --query loginServer \
  --output tsv)
```

---

## Create the AKS Cluster

```bash
az aks create \
  --resource-group "$RG" \
  --name "$AKS_NAME" \
  --location "$LOCATION" \
  --node-count 1 \
  --generate-ssh-keys
```

Connect `kubectl` to the cluster:

```bash
az aks get-credentials \
  --resource-group "$RG" \
  --name "$AKS_NAME" \
  --overwrite-existing
```

Verify:

```bash
kubectl get nodes
```

---

## Helm Environments

The same Helm chart is used across multiple environments.

```text
devops-web
     │
     ├── values-dev.yaml
     │
     ├── values-staging.yaml
     │
     └── values-prod.yaml
```

Example production configuration:

```yaml
replicaCount: 3

image:
  repository: nginx
  tag: alpine

resources:
  requests:
    cpu: 50m
    memory: 64Mi

  limits:
    cpu: 200m
    memory: 128Mi
```

This allows the same Kubernetes templates to be reused with different configurations.

---

## Validate the Helm Chart

Lint the chart:

```bash
helm lint ./charts/devops-web
```

Validate production values:

```bash
helm lint ./charts/devops-web \
  -f ./charts/devops-web/values-prod.yaml
```

Render the Kubernetes YAML:

```bash
helm template devops-web \
  ./charts/devops-web \
  -f ./charts/devops-web/values-prod.yaml
```

Perform a dry run:

```bash
helm install devops-web-dev \
  ./charts/devops-web \
  --namespace dev \
  --create-namespace \
  -f ./charts/devops-web/values-dev.yaml \
  --dry-run \
  --debug
```

---

## Deploy the Development Environment

```bash
helm upgrade --install devops-web-dev \
  ./charts/devops-web \
  --namespace dev \
  --create-namespace \
  -f ./charts/devops-web/values-dev.yaml \
  --wait
```

Verify:

```bash
helm list -n dev
```

```bash
kubectl get all -n dev
```

---

## Deploy Production

```bash
helm upgrade --install devops-web-prod \
  ./charts/devops-web \
  --namespace production \
  --create-namespace \
  -f ./charts/devops-web/values-prod.yaml \
  --wait
```

Verify:

```bash
helm list -n production
```

```bash
kubectl get pods -n production
```

---

## Upgrade a Helm Release

After modifying configuration such as:

```yaml
replicaCount: 2
```

upgrade the release:

```bash
helm upgrade devops-web-dev \
  ./charts/devops-web \
  --namespace dev \
  -f ./charts/devops-web/values-dev.yaml \
  --wait
```

---

## View Release History

```bash
helm history devops-web-dev \
  --namespace dev
```

Example:

```text
REVISION    STATUS
1           superseded
2           deployed
```

---

## Roll Back a Deployment

If an upgrade introduces a problem:

```bash
helm rollback devops-web-dev 1 \
  --namespace dev
```

Verify:

```bash
helm status devops-web-dev \
  --namespace dev
```

---

## Package the Helm Chart

Package the chart:

```bash
helm package ./charts/devops-web \
  --destination ./dist
```

Example output:

```text
dist/devops-web-0.1.0.tgz
```

---

## Store the Helm Chart in ACR

Authenticate to ACR:

```bash
az acr login \
  --name "$ACR_NAME"
```

Push the Helm chart as an OCI artifact:

```bash
helm push \
  ./dist/devops-web-0.1.0.tgz \
  oci://$ACR_LOGIN_SERVER/helm
```

Verify:

```bash
az acr repository list \
  --name "$ACR_NAME" \
  --output table
```

Check chart versions:

```bash
az acr repository show-tags \
  --name "$ACR_NAME" \
  --repository helm/devops-web \
  --output table
```

---

## Install a Helm Chart Directly From ACR

```bash
helm install devops-web-prod \
  oci://$ACR_LOGIN_SERVER/helm/devops-web \
  --version 0.1.0 \
  --namespace production \
  --create-namespace \
  -f ./charts/devops-web/values-prod.yaml \
  --wait
```

This creates the workflow:

```text
Helm Chart
    ↓
helm package
    ↓
Helm OCI Package
    ↓
Azure Container Registry
    ↓
helm install
    ↓
AKS
```

---

## Troubleshooting

### Kubernetes Cluster Unreachable

Check:

```bash
kubectl get nodes
```

Refresh AKS credentials:

```bash
az aks get-credentials \
  --resource-group "$RG" \
  --name "$AKS_NAME" \
  --overwrite-existing
```

---

### Helm Template Errors

```bash
helm lint ./charts/devops-web
```

```bash
helm template devops-web \
  ./charts/devops-web \
  --debug
```

---

### Check Helm Release Status

```bash
helm status devops-web-prod \
  --namespace production
```

---

### Check Release Values

```bash
helm get values devops-web-prod \
  --namespace production \
  --all
```

---

### Inspect Deployed Manifests

```bash
helm get manifest devops-web-prod \
  --namespace production
```

---

### Troubleshoot Kubernetes Pods

```bash
kubectl get pods -n production
```

```bash
kubectl describe pod <POD_NAME> \
  -n production
```

```bash
kubectl logs <POD_NAME> \
  -n production
```

A useful troubleshooting workflow is:

```text
helm lint
    ↓
helm template
    ↓
helm status
    ↓
helm history
    ↓
helm get values
    ↓
helm