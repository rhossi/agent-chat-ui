# Podman Specific Guide

This guide provides Podman-specific instructions and best practices for building and deploying the Agent Chat UI application.

## Building with Podman

### Basic Build

```bash
# Build from Dockerfile (Podman can read Dockerfiles)
podman build -t agent-chat-ui:latest .

# Or use Containerfile (Podman-native)
podman build -f Containerfile -t agent-chat-ui:latest .
```

### Platform-Specific Builds

If you're building on Apple Silicon (M1/M2) but deploying to x86_64:

```bash
# Build for specific architecture
podman build --platform linux/amd64 -t agent-chat-ui:latest .

# Multi-architecture build
podman build --platform linux/amd64,linux/arm64 -t agent-chat-ui:latest .
```

### Build Options

```bash
# Build with no cache
podman build --no-cache -t agent-chat-ui:latest .

# Build with build args
podman build --build-arg NODE_VERSION=20 -t agent-chat-ui:latest .

# Quiet build (less output)
podman build -q -t agent-chat-ui:latest .
```

## Working with OCIR

### Login to OCI Container Registry

```bash
# Interactive login
podman login <region>.ocir.io

# Non-interactive login
echo $OCI_AUTH_TOKEN | podman login <region>.ocir.io \
  -u <tenancy-namespace>/<oci-username> \
  --password-stdin

# Verify login
podman login --get-login <region>.ocir.io
```

### Push Images

```bash
# Tag for OCIR
podman tag agent-chat-ui:latest \
  us-ashburn-1.ocir.io/mytenancy/agent-chat-ui:latest

# Push to OCIR
podman push us-ashburn-1.ocir.io/mytenancy/agent-chat-ui:latest

# Push with progress bar
podman push --log-level=info us-ashburn-1.ocir.io/mytenancy/agent-chat-ui:latest
```

### Pull Images

```bash
# Pull from OCIR
podman pull us-ashburn-1.ocir.io/mytenancy/agent-chat-ui:latest

# Pull with specific platform
podman pull --platform linux/amd64 us-ashburn-1.ocir.io/mytenancy/agent-chat-ui:latest
```

## Local Testing

### Run Container Locally

```bash
# Run interactively
podman run -it --rm -p 3000:3000 \
  -e NEXTAUTH_URL=http://localhost:3000 \
  -e NEXTAUTH_SECRET=test-secret \
  -e OCI_ISSUER=https://idcs-xxx.identity.oraclecloud.com \
  -e OCI_CLIENT_ID=your-client-id \
  -e OCI_CLIENT_SECRET=your-client-secret \
  agent-chat-ui:latest

# Run in background (detached)
podman run -d --name agent-chat-ui -p 3000:3000 \
  --env-file .env.local \
  agent-chat-ui:latest

# View logs
podman logs -f agent-chat-ui

# Stop container
podman stop agent-chat-ui

# Remove container
podman rm agent-chat-ui
```

### Use Environment File

Create `.env.local` with your variables, then:

```bash
podman run -d -p 3000:3000 \
  --env-file .env.local \
  --name agent-chat-ui \
  agent-chat-ui:latest
```

## Podman Compose (Optional)

If you prefer docker-compose-like experience:

### Install podman-compose

```bash
# Python pip
pip3 install podman-compose

# Or via package manager
brew install podman-compose  # macOS
sudo dnf install podman-compose  # Fedora/RHEL
```

### Create compose.yaml

```yaml
version: '3.8'

services:
  agent-chat-ui:
    build: .
    ports:
      - "3000:3000"
    environment:
      - NEXTAUTH_URL=http://localhost:3000
      - NEXTAUTH_SECRET=${NEXTAUTH_SECRET}
      - OCI_ISSUER=${OCI_ISSUER}
      - OCI_CLIENT_ID=${OCI_CLIENT_ID}
      - OCI_CLIENT_SECRET=${OCI_CLIENT_SECRET}
    restart: unless-stopped
```

### Run with podman-compose

```bash
# Start services
podman-compose up -d

# View logs
podman-compose logs -f

# Stop services
podman-compose down
```

## Rootless Containers

Podman's rootless mode provides enhanced security:

```bash
# Check if running rootless
podman info | grep rootless

# Run container as non-root user (default in Podman)
podman run -d -p 3000:3000 agent-chat-ui:latest

# For ports < 1024, use port mapping
podman run -d -p 8080:3000 agent-chat-ui:latest
```

## Image Management

### List Images

```bash
# List all images
podman images

# List with filters
podman images --filter reference=agent-chat-ui

# Show image details
podman inspect agent-chat-ui:latest
```

### Remove Images

```bash
# Remove specific image
podman rmi agent-chat-ui:latest

# Remove all unused images
podman image prune

# Remove all images (careful!)
podman rmi -a
```

### Save and Load Images

```bash
# Save image to tar file
podman save -o agent-chat-ui.tar agent-chat-ui:latest

# Compress while saving
podman save agent-chat-ui:latest | gzip > agent-chat-ui.tar.gz

# Load image from tar
podman load -i agent-chat-ui.tar

# Load from compressed tar
gunzip -c agent-chat-ui.tar.gz | podman load
```

## Debugging

### Inspect Running Container

```bash
# Execute shell in running container
podman exec -it agent-chat-ui /bin/sh

# Check environment variables
podman exec agent-chat-ui env

# View container processes
podman top agent-chat-ui

# View resource usage
podman stats agent-chat-ui
```

### Build Debugging

```bash
# Build with verbose output
podman build --log-level=debug -t agent-chat-ui:latest .

# Stop at specific build stage
podman build --target builder -t agent-chat-ui:builder .

# Run intermediate stage
podman run -it agent-chat-ui:builder /bin/sh
```

## Podman vs Docker Differences

### Command Differences

| Task | Docker | Podman |
|------|--------|--------|
| Run daemon | `dockerd` | Not needed |
| Build | `docker build` | `podman build` |
| Run | `docker run` | `podman run` |
| Push | `docker push` | `podman push` |
| List | `docker ps` | `podman ps` |

### Configuration Locations

- **Docker**: `~/.docker/config.json`
- **Podman**: `~/.config/containers/auth.json` (Linux) or `~/.config/containers/auth.json` (macOS)

### Network Differences

```bash
# Docker uses bridge network by default
# Podman uses slirp4netns for rootless containers

# Create network (same syntax)
podman network create mynetwork

# List networks
podman network ls

# Inspect network
podman network inspect mynetwork
```

## CI/CD with Podman

### GitHub Actions Example

```yaml
name: Build and Push with Podman

on:
  push:
    branches: [ main ]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v3
    
    - name: Install Podman
      run: |
        sudo apt-get update
        sudo apt-get -y install podman
    
    - name: Build image
      run: podman build -t agent-chat-ui:${{ github.sha }} .
    
    - name: Login to OCIR
      run: |
        echo "${{ secrets.OCIR_TOKEN }}" | podman login \
          ${{ secrets.OCIR_REGION }}.ocir.io \
          -u ${{ secrets.OCIR_USERNAME }} \
          --password-stdin
    
    - name: Push to OCIR
      run: |
        podman tag agent-chat-ui:${{ github.sha }} \
          ${{ secrets.OCIR_REGION }}.ocir.io/${{ secrets.OCIR_NAMESPACE }}/agent-chat-ui:latest
        podman push ${{ secrets.OCIR_REGION }}.ocir.io/${{ secrets.OCIR_NAMESPACE }}/agent-chat-ui:latest
```

## Troubleshooting

### Podman Machine Issues (macOS/Windows)

```bash
# Check machine status
podman machine ls

# Stop machine
podman machine stop

# Remove and recreate machine
podman machine rm
podman machine init --cpus 4 --memory 8192 --disk-size 50
podman machine start
```

### Permission Errors

```bash
# If you get permission errors on macOS
podman machine ssh
sudo chown -R core:core /var/home/core

# Or restart the machine
podman machine stop
podman machine start
```

### Network Issues

```bash
# Reset network
podman network prune

# Check connectivity
podman run --rm alpine ping -c 3 google.com
```

### Build Cache Issues

```bash
# Clear build cache
podman system prune -a

# Remove all build cache
podman system reset
```

## Best Practices

1. **Use .containerignore**: Same as .dockerignore, tells Podman what to exclude
2. **Multi-stage builds**: Reduce final image size
3. **Specific tags**: Don't rely on `latest` tag in production
4. **Rootless when possible**: Enhanced security
5. **Health checks**: Include HEALTHCHECK in Containerfile
6. **Minimal base images**: Use Alpine or distroless images
7. **Scan images**: Use `podman scan` (if available) or external tools

## Additional Resources

- [Podman Official Documentation](https://docs.podman.io/)
- [Podman Desktop](https://podman-desktop.io/) - GUI for Podman
- [Podman vs Docker](https://docs.podman.io/en/latest/markdown/podman.1.html#differences-from-docker)
- [OCI Image Specification](https://github.com/opencontainers/image-spec)

## Quick Reference

```bash
# Build
podman build -t agent-chat-ui:latest .

# Run locally
podman run -d -p 3000:3000 --env-file .env.local agent-chat-ui:latest

# Push to OCIR
podman login us-ashburn-1.ocir.io
podman tag agent-chat-ui:latest us-ashburn-1.ocir.io/mytenancy/agent-chat-ui:latest
podman push us-ashburn-1.ocir.io/mytenancy/agent-chat-ui:latest

# Cleanup
podman system prune -a
```

