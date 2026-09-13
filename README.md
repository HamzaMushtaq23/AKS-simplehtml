# AKS Simple HTML CI/CD Pipeline

This project demonstrates a complete **CI/CD pipeline** for deploying a simple HTML website to **Azure Kubernetes Service (AKS)**.

The application is containerized using Docker and Nginx, stored in **Azure Container Registry (ACR)**, and automatically deployed to AKS through **Azure DevOps Pipelines** whenever changes are pushed to the GitHub `main` branch.

---

# Architecture

```text
                         GitHub
                           │
                           │ Push Code
                           ▼
                    Azure DevOps
                     CI/CD Pipeline
                           │
                    Docker Build
                           │
                           ▼
              Azure Container Registry
                   aksacr11.azurecr.io
                           │
                           │ Pull Image
                           ▼
              Azure Kubernetes Service
                         (AKS)
                           │
                    Kubernetes
                    Deployment
                           │
                  ┌────────┴────────┐
                  │                 │
                Pod 1             Pod 2
                  │                 │
                  └────────┬────────┘
                           │
                           ▼
                  LoadBalancer Service
                           │
                           ▼
                     External IP
                           │
                           ▼
                      Web Browser
                           │
                           ▼
                       index.html
```

---

# Technologies Used

* GitHub
* Azure DevOps
* Azure Container Registry (ACR)
* Azure Kubernetes Service (AKS)
* Docker
* Kubernetes
* Nginx
* Azure CLI
* kubectl

---

# Project Structure

```text
AKS-simplehtml/
│
├── index.html
├── Dockerfile
├── deployment.yaml
└── azure-pipelines.yml
```

---

# 1. Create the GitHub Repository

Create a GitHub repository named:

```text
AKS-simplehtml
```

The repository contains the application code and Kubernetes configuration.

---

# 2. Create the HTML Application

Create `index.html`:

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>AKS Simple HTML</title>
</head>
<body>

    <h1>WOWWWWWWWWW??!! 101-2 Pakistan</h1>
    <h1>Awais 45</h1>
    <h1>Shan Masood 51</h1>

</body>
</html>
```

This is the simple website that will be deployed to AKS.

---

# 3. Create the Dockerfile

Create a file named:

```text
Dockerfile
```

Add:

```dockerfile
FROM nginx:alpine

COPY index.html /usr/share/nginx/html/index.html

EXPOSE 80
```

### What this does

* Uses the lightweight `nginx:alpine` image.
* Copies `index.html` into Nginx's default web directory.
* Exposes port `80`.
* Nginx serves the HTML page.

---

# 4. Create Azure Container Registry

Create an Azure Container Registry.

Example configuration:

```text
Name: aksacr11
SKU: Basic
```

The registry login server is:

```text
aksacr11.azurecr.io
```

ACR is used to store the Docker images created by the Azure DevOps pipeline.

---

# 5. Create Azure Kubernetes Service

Create an AKS cluster.

Example configuration:

```text
Resource Group: aks-demo-rg
Cluster Name: aks-demo
```

Use a small VM size suitable for development/testing.

For this project, one node is sufficient.

---

# 6. Connect ACR with AKS

AKS needs permission to pull the Docker image from ACR.

Run:

```bash
az aks update \
  --resource-group aks-demo-rg \
  --name aks-demo \
  --attach-acr aksacr11
```

Verify AKS:

```bash
kubectl get nodes
```

---

# 7. Connect to AKS from Azure Cloud Shell

First login:

```bash
az login
```

Get AKS credentials:

```bash
az aks get-credentials \
  --resource-group aks-demo-rg \
  --name aks-demo \
  --overwrite-existing
```

Check the connection:

```bash
kubectl get nodes
```

Check all resources:

```bash
kubectl get pods -A
```

---

# 8. Create Kubernetes Deployment

Create:

```text
deployment.yaml
```

Add:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: simplehtml-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: simplehtml-app
  template:
    metadata:
      labels:
        app: simplehtml-app
    spec:
      containers:
        - name: simplehtml-app
          image: aksacr11.azurecr.io/aks-simplehtml:latest
          ports:
            - containerPort: 80

---

apiVersion: v1
kind: Service
metadata:
  name: simplehtml-service
spec:
  type: LoadBalancer
  selector:
    app: simplehtml-app
  ports:
    - port: 80
      targetPort: 80
```

### Deployment

The Deployment creates two replicas:

```text
simplehtml-app
     │
     ├── Pod 1
     │
     └── Pod 2
```

### Service

The Kubernetes Service uses:

```text
type: LoadBalancer
```

This provides an external IP that can be accessed from a web browser.

No Ingress is required for this simple project.

---

# 9. Create Azure DevOps Project

Create a project in Azure DevOps.

Connect the project to the GitHub repository:

```text
AKS-simplehtml
```

The pipeline will use the GitHub repository as its source code.

---

# 10. Create Azure Service Connection

In Azure DevOps, create a service connection:

```text
Azure Resource Manager
```

Use:

```text
Authentication: Workload identity federation
```

Name it:

```text
Azure-Service-Connection
```

This connection allows Azure DevOps to communicate with Azure resources such as AKS.

---

# 11. Create ACR Service Connection

Create another service connection for Azure Container Registry.

Use:

```text
Docker Registry
```

Select:

```text
Azure Container Registry
```

Name it:

```text
ACR-Service-Connection
```

This connection allows the pipeline to build and push Docker images to ACR.

---

# 12. Create Azure DevOps Pipeline

Create a pipeline using the GitHub repository.

Create:

```text
azure-pipelines.yml
```

Use the following pipeline:

```yaml
trigger:
- main

pool:
  vmImage: ubuntu-latest

variables:
  imageRepository: 'aks-simplehtml'
  containerRegistry: 'aksacr11.azurecr.io'
  dockerRegistryServiceConnection: 'ACR-Service-Connection'

  azureServiceConnection: 'Azure-Service-Connection'
  aksResourceGroup: 'aks-demo-rg'
  aksClusterName: 'aks-demo'

stages:

# ==========================================
# Stage 1 - Build and Push Docker Image
# ==========================================

- stage: Build
  displayName: 'Build and Push'

  jobs:
  - job: BuildAndPush
    displayName: 'Build Docker Image'

    steps:

    - checkout: self

    - task: Docker@2
      displayName: 'Build and Push Docker Image'
      inputs:
        command: buildAndPush
        repository: $(imageRepository)
        dockerfile: Dockerfile
        containerRegistry: $(dockerRegistryServiceConnection)
        tags: |
          latest
          $(Build.BuildId)


# ==========================================
# Stage 2 - Deploy to AKS
# ==========================================

- stage: Deploy
  displayName: 'Deploy to AKS'

  dependsOn: Build
  condition: succeeded()

  jobs:
  - job: DeployToAKS
    displayName: 'Deploy Application'

    steps:

    - checkout: self

    - task: AzureCLI@2
      displayName: 'Deploy to AKS'
      inputs:
        azureSubscription: $(azureServiceConnection)
        scriptType: bash
        scriptLocation: inlineScript

        inlineScript: |

          echo "Getting AKS credentials..."

          az aks get-credentials \
            --resource-group $(aksResourceGroup) \
            --name $(aksClusterName) \
            --overwrite-existing

          echo "Applying Kubernetes configuration..."

          kubectl apply -f deployment.yaml

          echo "Updating deployment with new image..."

          kubectl set image deployment/simplehtml-app \
            simplehtml-app=$(containerRegistry)/$(imageRepository):$(Build.BuildId)

          echo "Waiting for deployment rollout..."

          kubectl rollout status deployment/simplehtml-app

          echo "Checking pods..."

          kubectl get pods

          echo "Checking service..."

          kubectl get service simplehtml-service
```

---

# 13. How the Pipeline Works

The pipeline contains two stages.

## Stage 1 — Build

When code is pushed to `main`:

```text
GitHub
   ↓
Azure DevOps Pipeline
   ↓
Docker Build
   ↓
Docker Image
   ↓
ACR
```

The image is pushed with two tags:

```text
aksacr11.azurecr.io/aks-simplehtml:latest
```

and:

```text
aksacr11.azurecr.io/aks-simplehtml:<Build ID>
```

For example:

```text
aksacr11.azurecr.io/aks-simplehtml:125
```

---

# 14. Stage 2 — Deploy

After the Build stage succeeds, the Deploy stage starts.

First, the pipeline connects to AKS:

```bash
az aks get-credentials
```

Then it applies the Kubernetes configuration:

```bash
kubectl apply -f deployment.yaml
```

After that, it updates the Deployment to use the **current Build ID image**:

```bash
kubectl set image deployment/simplehtml-app \
  simplehtml-app=aksacr11.azurecr.io/aks-simplehtml:$(Build.BuildId)
```

For example:

```text
Build 125
    ↓
aksacr11.azurecr.io/aks-simplehtml:125
    ↓
AKS
```

This is important because every pipeline run uses a new image version.

---

# 15. Automatic Rolling Update

When a new image is deployed, Kubernetes automatically performs a rolling update.

For example:

```text
Old Version
    │
    ├── Pod 1
    └── Pod 2
          ↓
      New Version
          ↓
    ├── New Pod 1
    └── New Pod 2
```

You do **not** need to manually run:

```bash
kubectl rollout restart
```

The pipeline updates the image and Kubernetes handles the rollout.

---

# 16. Push Changes to GitHub

After changing your HTML code:

```text
index.html
```

commit and push the changes to:

```text
main
```

Because the pipeline contains:

```yaml
trigger:
- main
```

the pipeline automatically starts.

---

# 17. Check the Pipeline

Azure DevOps will show:

```text
Build
  ↓
Build Docker Image
  ↓
Push Image to ACR
  ↓
Deploy
  ↓
Update AKS Deployment
  ↓
Rollout
```

Both stages should complete successfully.

---

# 18. Check AKS Pods

From Cloud Shell:

```bash
kubectl get pods
```

Expected:

```text
NAME                              READY   STATUS
simplehtml-app-xxxxxxxxxx        1/1     Running
simplehtml-app-yyyyyyyyyy        1/1     Running
```

There should be two pods because:

```yaml
replicas: 2
```

---

# 19. Check Deployment

Run:

```bash
kubectl get deployment
```

Expected:

```text
NAME             READY   UP-TO-DATE   AVAILABLE
simplehtml-app   2/2     2            2
```

---

# 20. Check Kubernetes Service

Run:

```bash
kubectl get svc
```

Expected:

```text
NAME                 TYPE           EXTERNAL-IP
simplehtml-service   LoadBalancer   xx.xx.xx.xx
```

The `EXTERNAL-IP` is the public IP assigned to the LoadBalancer.

---

# 21. Open the Website

Copy the `EXTERNAL-IP`:

```text
xx.xx.xx.xx
```

Open it in your browser:

```text
http://xx.xx.xx.xx
```

The HTML application should now be accessible.

---

# 22. Verify New Changes

Change something in:

```text
index.html
```

For example:

```html
<h1>Version 2 - AKS CI/CD</h1>
```

Push the change to GitHub:

```text
GitHub
   ↓
Azure DevOps Pipeline
   ↓
New Docker Image
   ↓
ACR
   ↓
AKS
   ↓
Automatic Rolling Update
```

After the pipeline completes, refresh the same LoadBalancer IP.

The updated website should be displayed.

---

