# Craft Microservices - Kubernetes Deployment on minikube with public image
# build image and push to dockerhub public repo
# Argocd deployment
# minikube should be installed on linux machines for this to work



This guide explains how to deploy and test the Craft Origami application on Kubernetes.

## Prerequisites

- Minikube installed and running
- kubectl configured
- Docker for building images

## Deployment Process

# Create github codespace or use a linux machine

```bash
# install kubctl
https://docs.aws.amazon.com/eks/latest/userguide/install-kubectl.html
curl -O https://s3.us-west-2.amazonaws.com/amazon-eks/1.29.0/2024-01-04/bin/linux/amd64/kubectl

Install minikube
curl -LO https://github.com/kubernetes/minikube/releases/latest/download/minikube-linux-amd64

sudo install minikube-linux-amd64 /usr/local/bin/minikube && rm minikube-linux-amd64

start minikube cluster
minikube start

minikube status
```

### 1. Start Minikube

Start Minikube and enable the ingress addon:

```bash
minikube start
minikube addons enable ingress
```

### 2. use the Service Images from dockerhub
# 
✅ Catalogue: underdogdevops/microservices:catalogue-latest
✅ Frontend: underdogdevops/microservices:frontend-latest
✅ Recommendation: underdogdevops/microservices:recommendation-latest
✅ Voting: underdogdevops/microservices:voting-latest

### 3. Create AWS Credentials Secret

```bash
# Create AWS credentials secret (replace with your actual credentials)
kubectl create secret generic aws-credentials \
  --from-literal=aws_access_key_id=aws_access_key \
  --from-literal=aws_secret_access_key=aws_secret_key
```

### 4. Deploy Services

Deploy all services in order:

```bash
# Apply deployment files
cd k8s-minikube-2/
kubectl apply -f catalogue-service.yaml
kubectl apply -f recommendation-service.yaml
kubectl apply -f voting-service.yaml
kubectl apply -f frontend-service.yaml
```

### 5. Deploy Ingress

```bash
kubectl apply -f craft-ingress.yaml
```

### 6. Testing Options

#### Option A: Test with Port Forwarding

```bash
# Port-forward each service for testing
kubectl port-forward svc/catalogue 5000:5000 &
kubectl port-forward svc/recco 8081:8080 &
kubectl port-forward svc/voting 8082:8080 &
kubectl port-forward svc/frontend 3000:80 &

# Access services at:
# Catalogue: http://localhost:5000
# Recommendation: http://localhost:8081
# Voting: http://localhost:8082
# Frontend: http://localhost:3000

# Stop all port-forwarding processes when done
pkill -f "kubectl port-forward"
```

#### Option B: Test with Ingress

```bash
# Get Minikube IP
minikube ip

# Add to /etc/hosts
echo "$(minikube ip) craft.local" | sudo tee -a /etc/hosts

# Access the application at:
# http://craft.local
```

#### Option C: Use Minikube Tunnel (Alternative if Ingress isn't working)

```bash
# In a separate terminal
minikube tunnel

# Access the application at:
# http://craft.local
```

## Verification

```bash
# Check pod status
kubectl get pods

# Check logs for a specific service
kubectl logs -l app=catalogue-service
kubectl logs -l app=recommendation-service
kubectl logs -l app=voting-service
kubectl logs -l app=frontend-service

# Check service status
kubectl get svc

# Check ingress status
kubectl get ingress
```

## What's Happening Behind the Scenes

- **Catalogue Service**: Python service connecting to PostgreSQL, using AWS Secrets Manager for credentials
- **Recommendation Service**: Go service providing daily origami suggestions
- **Voting Service**: Java Spring Boot service for voting functionality with H2 in-memory database
- **Frontend Service**: Node.js UI that integrates with all backend services

The deployment leverages Kubernetes features like:
- ConfigMaps for configuration
- Secrets for sensitive data
- Services for internal communication
- Ingress for external access

## Cleanup

```bash
kubectl delete -f craft-ingress.yaml
kubectl delete -f frontend-service.yaml
kubectl delete -f voting-service.yaml
kubectl delete -f recommendation-service.yaml
kubectl delete -f catalogue-service.yaml
kubectl delete secret aws-credentials
```


## Argo deploy
```bash
# Clone your existing repository
git clone https://github.com/akhileshmishrabiz/kubernetes-zero-to-hero.git
cd kubernetes-zero-to-hero

# Create a new branch for ArgoCD deployments
git checkout -b argo-deploy-dev
git push origin argo-deploy-dev

# Create ArgoCD namespace
kubectl create namespace argocd

# Apply the ArgoCD installation manifest
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Wait for ArgoCD pods to be ready
kubectl wait --for=condition=Ready pods --all -n argocd --timeout=300s

# Port-forward the ArgoCD server
kubectl port-forward svc/argocd-server -n argocd 8080:443 &

# Get the initial admin password
ARGO_PWD=$(kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d)
echo "ArgoCD Initial Password: $ARGO_PWD"

# Access ArgoCD UI at: https://localhost:8080
# Username: admin
# Password: use the output from above command

```