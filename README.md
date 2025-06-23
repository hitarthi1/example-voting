# 🗳️ Example Voting App – CI/CD with GitHub Actions & Azure DevOps

This project demonstrates a complete CI/CD pipeline for a microservices-based app using **GitHub Actions** for Docker image builds and **Azure DevOps** for structured pipeline deployments

## 📁 Microservice Structure

This app consists of:
- `vote/` – Voting frontend (Python Flask)
- `result/` – Results backend (Node.js)
- `worker/` – Background processor (Redis consumer)

## ⚙️ What I Learned

- How to import external GitHub repositories into Azure DevOps
- How to build Docker images for individual microservices
- How to create and connect a self-hosted Azure DevOps agent
- How to set up a working container registry
- Why Microsoft-hosted agents aren’t available on the free trial
- End-to-end deployment flow using Infrastructure as Code (IaC)

## 🪜 Step-by-Step Setup Guide

### Step 1: Create New Project in Azure DevOps

```bash
Go to https://dev.azure.com
Login with your Microsoft account
Click on "New Project"
Project Name: example-voting-devops
Visibility: Private
Click "Create"
```

### Step 2: Import the Repository

```bash
Navigate to Repos → Import repository
Paste the GitHub URL: https://github.com/dockersamples/example-voting-app.git
Click "Import"
Set main as the default branch
```

### Step 3: Set Up Azure Container Registry

```bash
# Login to Azure Portal (if using CLI)
az login

# Create Resource Group
az group create --name example-voting-rg --location eastus

# Create Container Registry
az acr create \
  --resource-group example-voting-rg \
  --name votingregistry \
  --sku Basic \
  --admin-enabled true
```

Or using Azure Portal:

1. Go to [https://portal.azure.com](https://portal.azure.com)  
2. Click **Create a resource**  
3. Search for **Container Registry** and click **Create**  
4. Fill in the details:  
   - **Resource Group**: example-voting-rg  
   - **Registry Name**: votingregistry (must be globally unique)  
   - **SKU**: Basic or Standard  
   - **Enable Admin Access**: Yes  
5. Click **Review + create** → **Create**

### Step 4: Prepare Azure Pipelines (Separate for Each Microservice)

#### azure-pipelines-result.yml

```yaml
trigger:
  paths:
    include:
      - result/*

pool:
  name: 'azureagent'

steps:
  - script: echo "Build Docker image for result service"
  - script: docker build -t votingregistry.azurecr.io/result ./result
  - script: az acr login --name votingregistry
  - script: docker push votingregistry.azurecr.io/result
```

#### azure-pipelines-vote.yml

```yaml
trigger:
  paths:
    include:
      - vote/*

pool:
  name: 'azureagent'

steps:
  - script: echo "Build Docker image for vote service"
  - script: docker build -t votingregistry.azurecr.io/vote ./vote
  - script: az acr login --name votingregistry
  - script: docker push votingregistry.azurecr.io/vote
```

#### azure-pipelines-worker.yml

```yaml
trigger:
  paths:
    include:
      - worker/*

pool:
  name: 'azureagent'

steps:
  - script: echo "Build Docker image for worker service"
  - script: docker build -t votingregistry.azurecr.io/worker ./worker
  - script: az acr login --name votingregistry
  - script: docker push votingregistry.azurecr.io/worker
```

### Step 5: Set Up a Self-Hosted Azure DevOps Agent

#### Connect to VM

```bash
ssh -i azureagent_key.pem azureuser@<your-vm-public-ip>
```

#### Install Azure DevOps Agent

```bash
mkdir myagent && cd myagent
wget https://vstsagentpackage.azureedge.net/agent/3.236.0/vsts-agent-linux-x64-3.236.0.tar.gz
tar zxvf vsts-agent-linux-x64-3.236.0.tar.gz
./config.sh
```

During config:

- Azure DevOps URL: `https://dev.azure.com/YOUR_ORG`
- Use a Personal Access Token (PAT)
- Agent pool name: `azureagent`

#### Install Docker

```bash
sudo apt update
sudo apt install docker.io -y
sudo usermod -aG docker azureuser
sudo systemctl restart docker
```

#### Start the Agent

```bash
./run.sh
```

## 🐙 GitHub Actions Workflow (Optional)

### .github/workflows/docker-build.yml

```yaml
name: Build and Push Docker Image

on:
  push:
    paths:
      - 'result/**'

jobs:
  build:
    runs-on: ubuntu-latest

    steps:
    - name: Checkout code
      uses: actions/checkout@v3

    - name: Log in to Azure Container Registry
      uses: azure/docker-login@v1
      with:
        login-server: votingregistry.azurecr.io
        username: ${{ secrets.ACR_USERNAME }}
        password: ${{ secrets.ACR_PASSWORD }}

    - name: Build and push Docker image
      run: |
        docker build -t votingregistry.azurecr.io/result ./result
        docker push votingregistry.azurecr.io/result
```

## 🌐 Environment Variables and Secrets

Use GitHub or Azure DevOps secrets for:

```text
ACR_USERNAME
ACR_PASSWORD
DOCKER_REGISTRY_URL
```

## 🏗️ Infrastructure as Code (IaC)

Use Bicep, Terraform, or ARM templates to:

```text
- Automate creation of resource group, ACR, and agent VM
- Configure roles and network settings
- Reuse setup across environments
```

## ✅ Final Thoughts

```text
This guide helps you set up CI/CD for each microservice using GitHub Actions and Azure DevOps with self-hosted agents, while keeping everything flexible and infrastructure-as-code friendly.
```

## 📚 References

```text
- https://www.youtube.com/playlist?list=PLLasX02E8BPDT2Z5mK7Yp3w0tLQnOY0-Q
- https://learn.microsoft.com/en-us/azure/devops/pipelines/agents/agents?view=azure-devops&tabs=yaml%2Cbrowser
```
