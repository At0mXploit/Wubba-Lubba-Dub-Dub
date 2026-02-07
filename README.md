# Wubba Lubba Dub Dub

A K8 demo that proves it can handle your dreams and nightmares... 

## Overview

It’s designed as a hands-on learning project for Kubernetes, Helm, and DevOps workflows.

```bash
React Frontend --> Node API / Go API --> PostgreSQL
        ^                         |
        |                         v
  Python Load Generator --------> API endpoints
```

```
.
├── applcation-deploy/ # Kubernetes manifests & Taskfiles
├── demo-application/ # Source code for APIs, React frontend, load generator
```

- React frontend queries APIs and displays data.
- Node.js & Go APIs serve / and /ping endpoints and query PostgreSQL.
- PostgreSQL stores timestamps and request counts.
- Python load generator simulates traffic to the APIs.
- Kubernetes resources: Deployments, Services, ConfigMaps, Secrets, Jobs, IngressRoutes  

## Prerequisites

- Linux or macOS  
- Docker  
- [Kind](https://kind.sigs.k8s.io/)  
- [kubectl](https://kubernetes.io/docs/tasks/tools/)  
- [Task](https://taskfile.dev/#/installation)  
- Node.js & Go installed locally for builds  
- Optional: `kubens` to easily switch namespaces  

## Setup & Configurations

Clone the Repo.

```bash
docker --version
kubectl version --client
kind version
task --version
```
### Create a local Kubernetes cluster using Kind (Kubernetes in Docker)

```bash
sudo kind create cluster --name demo-cluster
sudo kubectl cluster-info --context kind-demo-cluster
```
### Add `/etc/hosts` entry for Traefik routing

```bash
sudo nano /etc/hosts
# Add the line:
127.0.0.1 kubernetes-course.devopsdirective.com
```

### Build Docker Images of Applications

```bash
cd demo-application

# Go API
cd api-golang
task install
ko build --bare --tags=latest --platform=all .

# Node API
cd ../api-node
task install
docker build -t localhost:5000/node-api:latest .

# React Frontend
cd ../client-react
task install
docker build -t localhost:5000/react-client:latest .

# Python Load Generator
cd ../load-generator-python
task install
docker build -t localhost:5000/load-generator:latest .

# Load images into Kind
kind load docker-image localhost:5000/node-api:latest --name demo-cluster
kind load docker-image localhost:5000/react-client:latest --name demo-cluster
kind load docker-image localhost:5000/load-generator:latest --name demo-cluster
```
### Deploy application on Kubernetes

```bash
cd ../../application-deploy

# Create namespace
task common:apply-namespace

# Deploy Traefik Ingress controller
task common:deploy-traefik
task common:apply-traefik-middleware

# Deploy PostgreSQL
task postgresql:install-postgres
task postgresql:apply-initial-db-migration-job

# Deploy application services
task api-golang:apply
task api-node:apply
task client-react:apply

# Deploy load generator
task load-generator-python:create-image-pull-secret
task load-generator-python:apply
```

Or to run everything at once:

```bash
task apply-all
```

```bash
➜  applcation-deploy git:(main) ✗ kubectl get pods -n demo-app
NAME                                     READY   STATUS              RESTARTS      AGE
api-golang-687595bcc-thhj4               0/1     ContainerCreating   0             2m41s
api-node-66ccb654bd-h2sdr                0/1     ContainerCreating   0             2m40s
client-react-nginx-5f7bc48ccc-mgxvp      0/1     ContainerCreating   0             2m39s
db-migrator-4vlck                        0/1     CrashLoopBackOff    4 (15s ago)   2m46s
load-generator-python-64bbfcbc4b-26p7h   0/1     ContainerCreating   0             6s

➜  applcation-deploy git:(main) ✗ kubectl get svc -n demo-app
NAME                 TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
api-golang           ClusterIP   10.96.180.153   <none>        8000/TCP   3m10s
api-node             ClusterIP   10.96.113.140   <none>        3000/TCP   3m9s
client-react-nginx   ClusterIP   10.96.154.180   <none>        8080/TCP   3m8s
```

## Testing

### Check Pods

```bash
kubectl get pods -n demo-app
```

### Port-forward Traefik for local access

```bash
kubectl port-forward svc/traefik 8080:80 -n traefik
```

### Test APIs

```bash
curl http://kubernetes-course.devopsdirective.com:8080/api/golang/ping
curl http://kubernetes-course.devopsdirective.com:8080/api/golang
curl http://kubernetes-course.devopsdirective.com:8080/api/node/ping
curl http://kubernetes-course.devopsdirective.com:8080/api/node
```

Go here in browser for frontend:

```bash
http://kubernetes-course.devopsdirective.com:8080/
```
### Check Load Generator

```bash
kubectl logs -f deploy/load-generator-python -n demo-app
```

## Scaling & Debugging

### Scale API pods:

```bash
kubectl scale deployment api-golang --replicas=3 -n demo-app
```

### Delete a pod to simulate failure:

```bash
kubectl delete pod <pod-name> -n demo-app
```

### Check logs:

```bash
kubectl logs -f deploy/api-golang -n demo-app
kubectl logs -f deploy/api-node -n demo-app
kubectl logs -f deploy/client-react-nginx -n demo-app
```

### Traefik dashboard:

```bash
kubectl port-forward svc/traefik 9000:80 -n traefik
```

Open in browser: `http://localhost:9000`

### Metrics:

```bash
kubectl top pods -n demo-app
```

## Common Issues & Fixes:

```bash
# go: executable file not found
Install Go: sudo apt install golang-go (or via your OS package manager)
# ko: command not found
Install Ko: go install github.com/google/ko@latest
# Traefik IngressRoutes not found
# Apply CRDs manually:
kubectl apply -f https://raw.githubusercontent.com/traefik/traefik/v2.11/docs/content/reference/dynamic-configuration/kubernetes-crd-definition-v1.yml
# kubens: executable file not found
# Install using Webi:
curl -sS https://webi.sh/kubens | sh
source ~/.config/envman/PATH.env
# Load generator private image
# Must set Docker credentials in environment variables.
```

 
This project was task from [DevOps Directive](https://courses.devopsdirective.com/kubernetes-beginner-to-pro/lessons/00-introduction/01-main). 

---

