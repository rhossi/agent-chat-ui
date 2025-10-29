# Kubernetes Deployment Guide for OCI OKE

This directory contains all the Kubernetes manifests needed to deploy the Agent Chat UI application to Oracle Cloud Infrastructure (OCI) Kubernetes Engine (OKE).

## Prerequisites

Before deploying, ensure you have:

1. **OCI OKE Cluster** - A running Kubernetes cluster in OCI
2. **kubectl** - Configured to connect to your OKE cluster
3. **Podman** - Installed and configured
4. **OCI Container Registry (OCIR)** - Access to push images
5. **OCI Identity Domain** - OAuth application configured

## Architecture Overview

- **Application**: Next.js 15 with NextAuth
- **Authentication**: OCI Identity Domain OAuth
- **Port**: 3000
- **Replicas**: 2 (configurable)
- **Service Type**: LoadBalancer (OCI Load Balancer)

## Quick Start

### Step 1: Build and Push Container Image

```bash
# Build the container image
podman build -t agent-chat-ui:latest .

# Tag for OCIR (replace with your details)
podman tag agent-chat-ui:latest <region>.ocir.io/<tenancy-namespace>/agent-chat-ui:latest

# Login to OCIR
podman login <region>.ocir.io
# Username: <tenancy-namespace>/<oci-username>
# Password: <auth-token>

# Push to OCIR
podman push <region>.ocir.io/<tenancy-namespace>/agent-chat-ui:latest
```

### Step 2: Configure Secrets and ConfigMaps

Edit the following files with your actual values:

#### k8s/02-secret.yaml
```yaml
NEXTAUTH_SECRET: "Generate with: openssl rand -base64 32"
OCI_ISSUER: "https://idcs-xxxxxxxx.identity.oraclecloud.com"
OCI_CLIENT_ID: "your-client-id"
OCI_CLIENT_SECRET: "your-client-secret"
```

#### k8s/03-configmap.yaml
```yaml
NEXTAUTH_URL: "https://your-domain.com"
```

#### k8s/04-deployment.yaml
Update the image field:
```yaml
image: <region>.ocir.io/<tenancy-namespace>/agent-chat-ui:latest
```

### Step 3: Create Image Pull Secret (if using OCIR)

```bash
kubectl create secret docker-registry ocir-secret \
  --docker-server=<region>.ocir.io \
  --docker-username='<tenancy-namespace>/<oci-username>' \
  --docker-password='<auth-token>' \
  --docker-email='<email>' \
  -n agent-chat-ui
```

Note: Despite using Podman, kubectl still uses `docker-registry` as the secret type for compatibility.

Then uncomment the `imagePullSecrets` section in `04-deployment.yaml`.

### Step 4: Deploy to Kubernetes

```bash
# Apply in order (or just apply all at once)
kubectl apply -f k8s/01-namespace.yaml
kubectl apply -f k8s/02-secret.yaml
kubectl apply -f k8s/03-configmap.yaml
kubectl apply -f k8s/04-deployment.yaml
kubectl apply -f k8s/05-service.yaml

# Or apply all at once (files are numbered in correct order)
kubectl apply -f k8s/
```

### Step 5: Verify Deployment

```bash
# Check pods
kubectl get pods -n agent-chat-ui

# Check service and get Load Balancer IP
kubectl get svc -n agent-chat-ui

# Check logs
kubectl logs -f -n agent-chat-ui -l app=agent-chat-ui

# Describe deployment
kubectl describe deployment agent-chat-ui -n agent-chat-ui
```

## Configuration Files

| File | Description |
|------|-------------|
| `01-namespace.yaml` | Creates the agent-chat-ui namespace |
| `02-secret.yaml` | Stores sensitive configuration (OAuth credentials) |
| `03-configmap.yaml` | Non-sensitive configuration |
| `04-deployment.yaml` | Main application deployment with 2 replicas |
| `05-service.yaml` | LoadBalancer service exposing the app |

**Note**: Files are numbered in deployment order. You can apply them individually in sequence or all at once with `kubectl apply -f k8s/`

## Environment Variables

### Required Variables

These must be configured in `secret.yaml` and `configmap.yaml`:

| Variable | Location | Description |
|----------|----------|-------------|
| `NEXTAUTH_URL` | ConfigMap | Canonical URL of your site |
| `NEXTAUTH_SECRET` | Secret | JWT encryption secret |
| `OCI_ISSUER` | Secret | OCI Identity Domain issuer URL |
| `OCI_CLIENT_ID` | Secret | OAuth client ID |
| `OCI_CLIENT_SECRET` | Secret | OAuth client secret |

### Optional Variables (LangGraph)

| Variable | Location | Description |
|----------|----------|-------------|
| `NEXT_PUBLIC_API_URL` | ConfigMap | LangGraph API endpoint |
| `NEXT_PUBLIC_ASSISTANT_ID` | ConfigMap | Assistant/Graph ID |
| `LANGGRAPH_API_URL` | ConfigMap | Backend LangGraph URL |
| `LANGSMITH_API_KEY` | Secret | LangSmith API key |

## OCI-Specific Configurations

### Load Balancer

The service uses OCI Load Balancer with these annotations:

```yaml
service.beta.kubernetes.io/oci-load-balancer-shape: "flexible"
service.beta.kubernetes.io/oci-load-balancer-shape-flex-min: "10"
service.beta.kubernetes.io/oci-load-balancer-shape-flex-max: "100"
```

### SSL/TLS Configuration

For HTTPS access via OCI Load Balancer, configure SSL certificates in the service annotations:

```yaml
service.beta.kubernetes.io/oci-load-balancer-ssl-ports: "443"
service.beta.kubernetes.io/oci-load-balancer-tls-secret: "agent-chat-ui-tls"
```

## Setting Up OCI Identity Domain OAuth

1. **Navigate to OCI Console** → Identity & Security → Domains
2. **Select your domain** → Applications → Add application
3. **Configure OAuth application**:
   - Application type: Confidential Application
   - Allowed Grant Types: Authorization Code
   - Redirect URL: `https://your-domain.com/api/auth/callback/oci`
   - Post Logout Redirect URL: `https://your-domain.com`
4. **Copy credentials** and update `secret.yaml`:
   - Client ID
   - Client Secret
   - Issuer URL (from domain overview)

## Health Checks

The application includes health check endpoints at `/api/health`:

- **Liveness Probe**: Checks if the container is alive
- **Readiness Probe**: Checks if the container can serve traffic
- **Startup Probe**: Handles slow application startup

## Resource Management

### Default Resources
- **Requests**: 200m CPU, 256Mi memory
- **Limits**: 500m CPU, 512Mi memory

Adjust these values in `04-deployment.yaml` based on your workload.

## Monitoring and Troubleshooting

### View Logs
```bash
# All pods
kubectl logs -n agent-chat-ui -l app=agent-chat-ui --tail=100 -f

# Specific pod
kubectl logs -n agent-chat-ui <pod-name> -f
```

### Check Pod Status
```bash
kubectl get pods -n agent-chat-ui
kubectl describe pod -n agent-chat-ui <pod-name>
```

### Check Service and Load Balancer
```bash
kubectl get svc -n agent-chat-ui
kubectl describe svc agent-chat-ui -n agent-chat-ui
```

### Common Issues

#### Pods not starting
- Check if image is accessible: `kubectl describe pod <pod-name> -n agent-chat-ui`
- Verify image pull secrets are configured
- Check logs: `kubectl logs <pod-name> -n agent-chat-ui`

#### Health checks failing
- Ensure `/api/health` endpoint is accessible
- Increase `initialDelaySeconds` if app takes longer to start
- Check application logs for errors

#### OAuth errors
- Verify `OCI_ISSUER`, `OCI_CLIENT_ID`, and `OCI_CLIENT_SECRET`
- Ensure redirect URL in OCI Identity Domain matches `NEXTAUTH_URL/api/auth/callback/oci`
- Check that `NEXTAUTH_SECRET` is properly set

#### Load Balancer not provisioning
- Check service events: `kubectl describe svc agent-chat-ui -n agent-chat-ui`
- Verify OCI quotas and limits
- Ensure OKE cluster has proper IAM policies

## Security Best Practices

1. **Never commit secrets** - Use sealed secrets or external secret managers
2. **Use non-root containers** - Already configured in deployment
3. **Enable Pod Security Standards** - Configure PSS for the namespace
4. **Rotate secrets regularly** - Especially `NEXTAUTH_SECRET` and OAuth credentials
5. **Use Network Policies** - Restrict pod-to-pod communication
6. **Enable RBAC** - Create service accounts with minimal permissions

## Scaling

### Manual Scaling
```bash
kubectl scale deployment agent-chat-ui -n agent-chat-ui --replicas=5
```

## Updating the Application

```bash
# Build and push new image
podman build -t agent-chat-ui:v2 .
podman tag agent-chat-ui:v2 <region>.ocir.io/<tenancy-namespace>/agent-chat-ui:v2
podman push <region>.ocir.io/<tenancy-namespace>/agent-chat-ui:v2

# Update deployment
kubectl set image deployment/agent-chat-ui agent-chat-ui=<region>.ocir.io/<tenancy-namespace>/agent-chat-ui:v2 -n agent-chat-ui

# Or update the deployment.yaml and apply
kubectl apply -f k8s/deployment.yaml

# Watch rollout
kubectl rollout status deployment/agent-chat-ui -n agent-chat-ui
```

## Rollback

```bash
# View rollout history
kubectl rollout history deployment/agent-chat-ui -n agent-chat-ui

# Rollback to previous version
kubectl rollout undo deployment/agent-chat-ui -n agent-chat-ui

# Rollback to specific revision
kubectl rollout undo deployment/agent-chat-ui -n agent-chat-ui --to-revision=2
```

## Clean Up

```bash
# Delete all resources
kubectl delete -f k8s/

# Or using namespace
kubectl delete namespace agent-chat-ui
```

## Quick Start Guide

For a streamlined build and deploy workflow, see **`BUILD-DEPLOY.md`** in the project root.

## Additional Resources

- [OCI OKE Documentation](https://docs.oracle.com/en-us/iaas/Content/ContEng/home.htm)
- [OCI Load Balancer Annotations](https://docs.oracle.com/en-us/iaas/Content/ContEng/Tasks/contengcreatingloadbalancer.htm)
- [Next.js Container Documentation](https://nextjs.org/docs/deployment#docker-image)
- [NextAuth.js Documentation](https://next-auth.js.org/)

## Support

For issues or questions:
1. Check application logs
2. Review OCI OKE cluster events
3. Consult the main README.md in the project root
4. Check the [Agent Chat UI repository](https://github.com/langchain-ai/agent-chat-ui)

