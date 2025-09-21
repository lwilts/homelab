# Homelab K3s Setup

Personal homelab running on single K3s node with GitOps via Flux.

## Stack
- **K3s** - Lightweight Kubernetes
- **Flux CD** - GitOps deployment
- **MetalLB** - LoadBalancer for bare metal (192.168.68.20-30)
- **nginx-gateway-fabric** - Gateway API implementation
- **cert-manager** - Let's Encrypt wildcard certs for `*.lab.lkwt.dev`
- **Cloudflare DNS** - DNS-01 challenge for cert validation

## Apps
- **Jellyfin** - Media server (NFS media from nas.lab)
- **Home Assistant** - Home automation
- **Hello** - Test nginx app
- **Dashboard** - K8s web UI

## Structure
```
flux/
├── clusters/home1/     # Bootstrap config
├── infra/              # Infrastructure (prereqs → controllers → configs)
└── apps/home1/         # Applications
```

## Key Challenges & Solutions

### Gateway API Issues
**Problem**: nginx-gateway-fabric needed specific GatewayClass and dependency handling.
**Solution**:
- Removed invalid HelmRelease dependency on metallb-system (it's a Kustomization)
- Fixed chart version from 2.1.1 to 1.2.0 (2.1.1 doesn't exist in OCI registry)

### Home Assistant Proxy Errors
**Problem**: "reverse proxy not set-up" errors behind nginx-gateway.
**Solution**: Init container adds HTTP config to configuration.yaml:
```yaml
http:
  use_x_forwarded_for: true
  trusted_proxies:
    - 10.42.0.0/16  # Pod network
    - 10.43.0.0/16  # Service network
```

### MetalLB IP Changes
**Problem**: MetalLB kept reassigning IPs during config changes.
**Solution**: Specify interface in metallb config to avoid conflicts.

### NFS Permissions
**Problem**: Alpine backup containers couldn't write to NFS share.
**Solution**: Configure maproot user/group on TrueNAS NFS export.

## Useful Commands

```bash
# Force reconcile everything
flux reconcile source git flux-system
flux reconcile kustomization infra-prereqs infra-controllers infra-configs apps

# Check application status
kubectl get httproute,gateway,certificates -A

# Debug networking
kubectl run debug --image=alpine:3.18 --rm -it -- sh

# Manual backup
kubectl create job backup-$(date +%Y%m%d) --from=cronjob/daily-backup -n backup-system
```

## Notes
- Daily backups to nas.lab:/mnt/red6/k8s-backup (Home Assistant + Jellyfin configs)
- K3s storage at /var/lib/rancher/k3s/storage with UUID-based PVC names
- Wildcard DNS: `*.lab.lkwt.dev → 192.168.68.22`
- All traffic HTTPS-only, certs auto-renewed
- hostNetwork needed for Home Assistant device discovery