## Architecture Overview

- **Infrastructure**: AWS EKS Cluster with managed node groups
- **CI/CD**: GitHub Actions for building and pushing Docker images
- **GitOps**: ArgoCD for continuous deployment
- **Security**: tfsec , trivy and sealed secret.
- **Tools**: Terraform, Helm, kubectl, eksctl

## Step 1: Prepare Ubuntu VM

Start with a fresh Ubuntu VM (recommended: Ubuntu 20.04 LTS or later) and ensure you have sudo access.

```bash
# Update the system
sudo apt update && sudo apt upgrade -y
```

## Step 2: Install Required Tools

### Install Dependencies and AWS CLI

```bash
# Install unzip utility
sudo apt install unzip -y

# Install AWS CLI v2
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install

# Verify installation
aws --version
```

### Install eksctl

```bash
# Set architecture (change to arm64, armv6, or armv7 for ARM systems)
ARCH=amd64
PLATFORM=$(uname -s)_$ARCH

# Download eksctl
curl -sLO "https://github.com/eksctl-io/eksctl/releases/latest/download/eksctl_$PLATFORM.tar.gz"

# Optional: Verify checksum
curl -sL "https://github.com/eksctl-io/eksctl/releases/latest/download/eksctl_checksums.txt" | grep $PLATFORM | sha256sum --check

# Extract and install
tar -xzf eksctl_$PLATFORM.tar.gz -C /tmp && rm eksctl_$PLATFORM.tar.gz
sudo install -m 0755 /tmp/eksctl /usr/local/bin && rm /tmp/eksctl

# Verify installation
eksctl version
```

### Install kubectl

```bash
# Install kubectl via snap
sudo snap install kubectl --classic

# Verify installation
kubectl version --client
```

### Install Terraform

```bash
# Download and install Terraform
curl -O https://releases.hashicorp.com/terraform/1.5.6/terraform_1.5.6_linux_amd64.zip
unzip terraform_1.5.6_linux_amd64.zip
sudo mv terraform /usr/local/bin/

# Verify installation
terraform -version
```

### Install Helm

```bash
# Download and install Helm
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
chmod 700 get_helm.sh
./get_helm.sh

# Verify installation
helm version
```

## Step 3: Configure AWS Credentials

Configure AWS CLI with your credentials:

```bash
aws configure
```

Provide the following information:

- **AWS Access Key ID**: Your AWS access key
- **AWS Secret Access Key**: Your AWS secret key
- **Default region name**: us-east-1 (or your preferred region)
- **Default output format**: table

## Step 4: Create EKS Cluster

### Create the EKS Cluster

```bash
eksctl create cluster \
  --name react-eks \
  --region us-east-1 \
  --nodegroup-name react-nodes \
  --node-type t3.medium \
  --nodes 2 \
  --nodes-min 1 \
  --nodes-max 3 \
  --managed
```

**Note**: This process takes 15-20 minutes. While waiting, proceed with the GitHub repository setup.

### Verify Cluster Creation

```bash
# Verify cluster is running
kubectl get nodes

# Get cluster info
kubectl cluster-info
```

## Step 5: Setup GitHub Repository

### Create Repository Structure

Create a new repository similar to: `https://github.com/subham-behera/devops-project`

Your repository should include:

```
devops-project/
├── k8s/
│   ├── deployment.yaml
│   ├── service.yaml
├── terraform/
│   ├── main.tf
│   ├── variables.tf
│   ├── terraform.tfvars
│   └── outputs.tf
├── .github/
│   └── workflows/
│       └── main.yaml
├── Dockerfile
├── package.json
├── nginx.conf
└── src/
    └── (your React app files)
```

Under terraform directory we just need a simple script to test. So it can even work with main.tf 

### Configure GitHub Secrets

In your GitHub repository, go to Settings → Secrets and variables → Actions, and add:

- `DOCKER_USERNAME`: Your Docker Hub username
- `DOCKER_PASSWORD`: Your Docker Hub password or access token

We can get the secrets from Docker Hub account.
### Sample GitHub Actions Workflow

Create `.github/workflows/main.yaml`:

Get the file at [devops-project/.github/workflows/main.yml at main · subham-behera/devops-project](https://github.com/subham-behera/devops-project/blob/main/.github/workflows/main.yml)
## Step 6: Install Sealed Secrets

### Install Sealed Secrets Controller

```bash
# Install the Sealed Secrets controller
kubectl apply -f https://github.com/bitnami-labs/sealed-secrets/releases/latest/download/controller.yaml

# Verify installation
kubectl get pods -n kube-system | grep sealed-secrets
```

### Create and Seal Secrets

```bash
# Create a regular secret (don't apply this)
kubectl create secret generic my-app-secret \
  --from-literal=REACT_APP_API_KEY=super-secret-value \
  --dry-run=client -o yaml > my-app-secret.yaml

# Install kubeseal (Sealed Secrets CLI)
wget https://github.com/bitnami-labs/sealed-secrets/releases/download/v0.24.0/kubeseal-0.24.0-linux-amd64.tar.gz
tar -xzf kubeseal-0.24.0-linux-amd64.tar.gz
sudo install -m 755 kubeseal /usr/local/bin/kubeseal

# Create sealed secret
kubeseal -f my-app-secret.yaml -w sealed-secret.yaml

# Apply the sealed secret
kubectl apply -f sealed-secret.yaml
```

## Step 7: Install and Configure ArgoCD

### Install ArgoCD

```bash
# Create ArgoCD namespace
kubectl create namespace argocd

# Install ArgoCD
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

Wait till process gets completed.
### Get ArgoCD Admin Password

```bash
# Get the initial admin password
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 --decode ; echo
```

**Important**: Copy this password as you'll need it to log into ArgoCD.

### Access ArgoCD UI

```bash
# Forward port to access ArgoCD UI
kubectl port-forward svc/argocd-server -n argocd 8080:443 --address=0.0.0.0
```

**Note**: Make sure port 8080 is allowed in your security group/firewall.

Access ArgoCD at: `https://your-vm-ip:8080`

- Username: `admin`
- Password: (the password you copied earlier)

### Configure Repository Connection

In ArgoCD UI:

1. Go to **Settings** → **Repositories**
2. Click **Connect Repo**
3. Enter your GitHub repository URL
4. Provide authentication if repository is private

Before moving make sure to make manifest files like deployment.yml and service.yml inside k8s directory.
### Create Application

In ArgoCD UI:

1. Click **New App**
    
2. Fill in the following fields:
    
    - **Application Name**: react-app
    - **Project**: default
    - **Sync Policy**: Automatic
    - **Repository URL**: Your GitHub repository URL
    - **Path**: k8s (path to your Kubernetes manifests)
    - **Destination Cluster**: https://kubernetes.default.svc
    - **Namespace**: default
3. Click **Create**
    

## Step 8: Verify Deployment

### Check Application Status

```bash
# Check pods
kubectl get pods

# Check services
kubectl get svc

# Get external IP (if using LoadBalancer service)
kubectl get svc your-service-name
```

### Access Your Application

If using a LoadBalancer service, get the external IP and access your application.

## Step 9: Test the Complete Pipeline

### Make a Code Change

1. Make a simple change to your React application
2. Commit and push to the main branch
3. GitHub Actions will automatically:
    - Build the Docker image
    - Push it to Docker Hub
    - Update the Kubernetes manifests
4. ArgoCD will automatically:
    - Detect the changes
    - Deploy the updated application

### Monitor the Deployment

- Check GitHub Actions for build status
- Check ArgoCD UI for deployment status
- Verify the changes are reflected in your running application

### Note : After completion check if you can see and download the artifact in github actions.

## Cleanup

To avoid AWS charges, cleanup resources when done:

```bash
# Delete EKS cluster
eksctl delete cluster --name react-eks --region us-east-1

# This will delete all associated resources
```
