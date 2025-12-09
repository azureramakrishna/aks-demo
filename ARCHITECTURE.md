# Architecture Overview

## System Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                          GitHub Repository                           │
│                     (azureramakrishna/aks-demo)                     │
└────────────────────────────┬────────────────────────────────────────┘
                             │
                             │ Push to main branch
                             ▼
┌─────────────────────────────────────────────────────────────────────┐
│                       GitHub Actions Workflow                        │
│                                                                       │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                        BUILD JOB                             │   │
│  │  1. Checkout code                                            │   │
│  │  2. Azure Login (Service Principal)                          │   │
│  │  3. Build Docker image                                       │   │
│  │  4. Push to ACR (saanvikit.azurecr.io)                      │   │
│  │     - Tag: commit SHA                                        │   │
│  │     - Tag: latest                                            │   │
│  └──────────────────────────┬───────────────────────────────────┘   │
│                             │                                         │
│                             │ Outputs: image-tag                      │
│                             ▼                                         │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                      RELEASE JOB                             │   │
│  │  Environment: production (Manual Approval Required)          │   │
│  │                                                               │   │
│  │  1. Checkout code                                            │   │
│  │  2. Azure Login                                              │   │
│  │  3. Set AKS context (AAD authentication)                     │   │
│  │  4. Update deployment manifest                               │   │
│  │  5. Apply Kubernetes resources                               │   │
│  │  6. Update container image                                   │   │
│  │  7. Rollout restart                                          │   │
│  │  8. Monitor rollout status                                   │   │
│  └──────────────────────────┬───────────────────────────────────┘   │
└─────────────────────────────┼────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────┐
│                    Azure Container Registry (ACR)                    │
│                      saanvikit.azurecr.io                           │
│                                                                       │
│  Repository: aks-demo-app                                            │
│  Images: aks-demo-app:latest, aks-demo-app:<commit-sha>            │
└────────────────────────────┬────────────────────────────────────────┘
                             │
                             │ Pull image
                             ▼
┌─────────────────────────────────────────────────────────────────────┐
│              Azure Kubernetes Service (AKS)                          │
│                    Cluster: saanvikit-aks                           │
│                    Resource Group: k8s-group                        │
│                                                                       │
│  ┌───────────────────────────────────────────────────────────────┐ │
│  │              Namespace: saanvikit                              │ │
│  │                                                                 │ │
│  │  ┌─────────────────────────────────────────────────────────┐  │ │
│  │  │           Deployment: aks-demo-app                       │  │ │
│  │  │  - Replicas: 2                                           │  │ │
│  │  │  - Container: aks-demo-app                               │  │ │
│  │  │  - Image: saanvikit.azurecr.io/aks-demo-app:<tag>      │  │ │
│  │  │  - Port: 3000                                            │  │ │
│  │  │  - Health checks: /health endpoint                       │  │ │
│  │  │  - Resources: 128Mi-256Mi RAM, 100m-200m CPU            │  │ │
│  │  └─────────────────────────────────────────────────────────┘  │ │
│  │                                                                 │ │
│  │  ┌─────────────────────────────────────────────────────────┐  │ │
│  │  │           Service: aks-demo-app                          │  │ │
│  │  │  - Type: LoadBalancer                                    │  │ │
│  │  │  - External Port: 80                                     │  │ │
│  │  │  - Target Port: 3000                                     │  │ │
│  │  │  - Selector: app=aks-demo-app                           │  │ │
│  │  └─────────────────────────────────────────────────────────┘  │ │
│  └───────────────────────────────────────────────────────────────┘ │
└────────────────────────────┬────────────────────────────────────────┘
                             │
                             │ External IP
                             ▼
                    ┌─────────────────┐
                    │   End Users     │
                    │  (HTTP Port 80) │
                    └─────────────────┘
```

## Component Details

### Application Layer
- **Technology**: Node.js with Express
- **Endpoints**:
  - `GET /` - Main endpoint returning JSON response
  - `GET /health` - Health check endpoint for Kubernetes probes
- **Port**: 3000

### Container Layer
- **Base Image**: node:18-alpine
- **Build Tool**: Docker
- **Registry**: Azure Container Registry (ACR)
- **Tagging Strategy**: 
  - Commit SHA for versioning
  - Latest for current production

### Orchestration Layer
- **Platform**: Azure Kubernetes Service (AKS)
- **Namespace**: saanvikit
- **Deployment Strategy**: Rolling update
- **Replicas**: 2 pods for high availability
- **Service Type**: LoadBalancer (public access)

### CI/CD Pipeline
- **Platform**: GitHub Actions
- **Trigger**: Push to main branch
- **Jobs**:
  1. **Build**: Compile and push Docker image to ACR
  2. **Release**: Deploy to AKS (requires manual approval)
- **Authentication**: Azure Service Principal
- **Approval Gate**: Production environment protection

## Security Features

1. **Azure AD Integration**: AKS cluster uses AAD authentication
2. **Service Principal**: GitHub Actions authenticates via service principal
3. **RBAC**: Role-based access control on AKS
4. **Private Registry**: Images stored in private ACR
5. **Namespace Isolation**: Application deployed in dedicated namespace

## High Availability

1. **Multiple Replicas**: 2 pod replicas for redundancy
2. **Health Probes**: 
   - Liveness probe: Restarts unhealthy pods
   - Readiness probe: Routes traffic only to ready pods
3. **Rolling Updates**: Zero-downtime deployments
4. **Resource Limits**: Prevents resource exhaustion

## Monitoring & Validation

1. **Rollout Status**: Automated deployment verification
2. **Health Checks**: Continuous pod health monitoring
3. **Deployment Logs**: GitHub Actions provides detailed logs
4. **Emoji Status**: Visual feedback during deployment stages

## Network Flow

```
User Request (HTTP:80)
    ↓
LoadBalancer Service
    ↓
Pod Selector (app=aks-demo-app)
    ↓
Pod (Container Port 3000)
    ↓
Node.js Application
    ↓
JSON Response
```

## Deployment Flow

```
Code Push → Build Image → Push to ACR → Manual Approval → 
Deploy to AKS → Update Image → Rollout Restart → Verify Status → Complete
```
