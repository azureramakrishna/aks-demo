# AKS Demo Application

Sample Node.js application with CI/CD pipeline to Azure Kubernetes Service (AKS).

## Prerequisites

1. Azure subscription
2. Azure Container Registry (ACR)
3. Azure Kubernetes Service (AKS) cluster
4. GitHub repository

## Setup Instructions

### 1. Create Azure Resources

```bash
# Set variables
RESOURCE_GROUP="aks-demo-rg"
LOCATION="eastus"
ACR_NAME="aksdemoregistry"
AKS_CLUSTER="aks-demo-cluster"

# Create resource group
az group create --name $RESOURCE_GROUP --location $LOCATION

# Create ACR
az acr create --resource-group $RESOURCE_GROUP \
  --name $ACR_NAME --sku Basic

# Create AKS cluster
az aks create --resource-group $RESOURCE_GROUP \
  --name $AKS_CLUSTER \
  --node-count 2 \
  --attach-acr $ACR_NAME \
  --generate-ssh-keys
```

### 2. Configure GitHub Secrets

Create a service principal and add to GitHub secrets:

```bash
az ad sp create-for-rbac --name "github-actions-sp" \
  --role contributor \
  --scopes /subscriptions/<SUBSCRIPTION_ID>/resourceGroups/$RESOURCE_GROUP \
  --sdk-auth
```

Add the output JSON as `AZURE_CREDENTIALS` secret in GitHub repository settings.

### 3. Update Workflow File

Edit `.github/workflows/deploy.yml` and replace:
- `<YOUR_ACR_NAME>` with your ACR name
- `<YOUR_AKS_CLUSTER_NAME>` with your AKS cluster name
- `<YOUR_RESOURCE_GROUP>` with your resource group name

### 4. Deploy

Push code to the `main` branch to trigger the workflow:

```bash
git add .
git commit -m "Initial commit"
git push origin main
```

### 5. Access Application

Get the external IP:

```bash
kubectl get service aks-demo-app
```

Access the application at `http://<EXTERNAL-IP>`

## Local Testing

Build and run locally:

```bash
docker build -t aks-demo-app .
docker run -p 3000:3000 aks-demo-app
```

Visit `http://localhost:3000`
