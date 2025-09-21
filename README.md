# Homelab Infrastructure

This repository contains the GitOps configuration for my homelab infrastructure using Flux CD.

## Overview

This homelab uses:
- **Kubernetes**: Container orchestration platform
- **Flux CD**: GitOps continuous delivery for Kubernetes
- **MetalLB**: Load balancer implementation for bare metal Kubernetes clusters
- **nginx-gateway-fabric**: Kubernetes Gateway API implementation
- **cert-manager**: Automatic TLS certificate management

## Architecture

```
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│  Applications   │    │  Infrastructure │    │  Prerequisites  │
│                 │    │                 │    │                 │
│ • Hello App     │◄───│ • nginx-gateway │◄───│ • Helm Repos    │
│ • Dashboard     │    │ • cert-manager  │    │ • CRDs          │
└─────────────────┘    │ • MetalLB       │    └─────────────────┘
                       └─────────────────┘
```

## Directory Structure

```
flux/
├── clusters/home1/          # Cluster-specific configuration
│   ├── flux-system/         # Flux CD bootstrap configuration
│   ├── infra.yaml          # Infrastructure Kustomizations
│   └── apps.yaml           # Application Kustomizations
├── infra/                  # Infrastructure components
│   ├── prereqs/            # Prerequisites (Helm repos, CRDs)
│   ├── controllers/        # Infrastructure controllers
│   └── configs/            # Infrastructure configurations
└── apps/home1/             # Applications for home1 cluster
    ├── hello.yml           # Sample nginx application
    └── dashboard.yaml      # Kubernetes dashboard
```

## Bootstrap Process

The infrastructure is deployed in a specific order to handle dependencies:

1. **Prerequisites** (`infra-prereqs`): Helm repositories and CRDs
2. **Controllers** (`infra-controllers`): Core infrastructure components
3. **Configurations** (`infra-configs`): Infrastructure settings and policies
4. **Applications** (`apps`): User applications

## Components

### MetalLB
- Provides LoadBalancer services on bare metal
- Configured with Layer 2 mode
- IP pool: `192.168.68.20-192.168.68.30`

### nginx-gateway-fabric
- Implements Kubernetes Gateway API
- Provides ingress capabilities with HTTPS termination
- LoadBalancer IP: `192.168.68.22`

### cert-manager
- Automatic TLS certificate provisioning
- Let's Encrypt integration for wildcard certificates
- Certificates for `*.lab.lkwt.dev`

### Gateway Configuration
- HTTPS-only listener on port 443
- TLS termination with wildcard certificates
- HTTPRoutes accepted from all namespaces

## Deployment

### Initial Setup

1. Bootstrap Flux on your cluster:
```bash
flux bootstrap github \
  --owner=<your-github-username> \
  --repository=homelab \
  --branch=main \
  --path=./flux/clusters/home1
```

2. Flux will automatically deploy all components in the correct order.

### Monitoring Deployment

```bash
# Check all Flux resources
flux get all

# Check specific Kustomizations
flux get kustomizations

# Check Helm releases
flux get helmreleases -A
```

### Troubleshooting

```bash
# Force reconciliation
flux reconcile source git flux-system
flux reconcile kustomization infra-prereqs
flux reconcile kustomization infra-controllers
flux reconcile kustomization infra-configs
flux reconcile kustomization apps

# Check resource status
kubectl get gateway -A
kubectl get httproute -A
kubectl get certificates -A
```

## Accessing Applications

### Hello App
- URL: `https://hello.lab.lkwt.dev`
- Simple nginx welcome page for testing

### Kubernetes Dashboard
- URL: `https://dashboard.lab.lkwt.dev`
- Web-based Kubernetes management interface

## Development

### Adding New Applications

1. Create application manifests in `flux/apps/home1/`
2. Include HTTPRoute for ingress access
3. Commit and push - Flux will deploy automatically

### Modifying Infrastructure

1. Update configurations in `flux/infra/`
2. Test changes in a development environment first
3. Commit and push for automatic deployment

## DNS Configuration

This setup uses a wildcard DNS record for simplicity. Ensure your DNS resolver has:

```bash
# Wildcard DNS entry pointing to MetalLB IP
*.lab.lkwt.dev A 192.168.68.22
```

All applications will be accessible via `https://<app-name>.lab.lkwt.dev`

## Security

- All HTTP traffic is redirected to HTTPS
- TLS certificates are automatically renewed
- No HTTP listeners exposed on LoadBalancer
- Applications isolated in separate namespaces

## Maintenance

### Certificate Renewal
Certificates are automatically renewed by cert-manager. Monitor with:
```bash
kubectl get certificates -A
kubectl describe certificate -n nginx-gateway wildcard-lab-lkwt-dev
```

### Updates
Component versions are pinned in the Helm charts. Update by modifying version numbers in the HelmRelease manifests.

## Contributing

1. Create feature branch
2. Make changes
3. Test in development environment
4. Submit pull request

## License

This homelab configuration is for personal use and learning purposes.