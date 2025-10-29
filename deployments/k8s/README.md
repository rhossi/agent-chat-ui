# Podman + Kubernetes Deployment

Complete workflow: Build with Podman → Push to OCIR → Deploy with kubectl

## Prerequisites

- OCI OKE Cluster
- kubectl configured
- Podman installed
- OCIR access

## Deployment Steps

### 1. Build Image

```bash
podman build -t agent-chat-ui:latest .
```

### 2. Push to OCIR

```bash
# Login to OCIR
podman login <region>.ocir.io

# Tag for OCIR
podman tag agent-chat-ui:latest <region>.ocir.io/<tenancy-namespace>/agent-chat-ui:latest

# Push to OCIR
podman push <region>.ocir.io/<tenancy-namespace>/agent-chat-ui:latest
```

### 3. Deploy to Kubernetes

```bash
kubectl apply -f k8s/
```

## Verify Deployment

```bash
kubectl get pods -n agent-chat-ui
kubectl get svc -n agent-chat-ui
```

## Configuration

Before deploying, copy and configure these files:

```bash
# Copy example files
cp 02-secret.example.yaml 02-secret.yaml
cp 03-configmap.example.yaml 03-configmap.yaml
```

Then update with your actual values:

- `02-secret.yaml` - OAuth credentials and secrets
- `03-configmap.yaml` - Non-sensitive configuration  
- `04-deployment.yaml` - Update image field with your OCIR path

## Quick Reference

```bash
# Complete workflow
podman build -t agent-chat-ui:latest .
podman login <region>.ocir.io
podman tag agent-chat-ui:latest <region>.ocir.io/<tenancy>/agent-chat-ui:latest
podman push <region>.ocir.io/<tenancy>/agent-chat-ui:latest
kubectl apply -f k8s/
```