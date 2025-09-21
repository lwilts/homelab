# Backup & Restore Procedures

## Overview
Automated daily backups of Home Assistant and Jellyfin configurations to NAS storage.

**Backup Location**: `nas.lab:/mnt/red6/k8s-backup`
**Schedule**: Daily at 2:00 AM
**Retention**: 30 days

## What Gets Backed Up
- **Home Assistant**: Configuration, database, automations, user data (5Gi)
- **Jellyfin**: Configuration, user settings, metadata (1Gi)
- **Media Files**: Not backed up (already stored on NAS)

## Backup Structure
```
/mnt/red6/k8s-backup/
├── 20250921-020000/
│   ├── home-assistant-20250921-020000.tar.gz
│   ├── jellyfin-20250921-020000.tar.gz
│   └── manifest.yaml
├── 20250922-020000/
├── latest -> 20250922-020000  # Symlink to latest backup
```

## Manual Backup

### Trigger Manual Backup
```bash
# Copy the template and create a new job
kubectl get job manual-backup-template -n backup-system -o yaml | \
  sed 's/manual-backup-template/manual-backup-'$(date +%Y%m%d-%H%M%S)'/' | \
  kubectl apply -f -

# Monitor backup progress
kubectl logs -n backup-system -l job-name=manual-backup-<timestamp> -f
```

### Check Backup Status
```bash
# View recent backups
kubectl logs -n backup-system -l app=cronjob-daily-backup --tail=50

# Check backup storage
kubectl exec -n backup-system deploy/backup-browser -- ls -la /backups/
```

## Disaster Recovery

### Scenario 1: Single Application Data Loss

If only one application loses data (e.g., Home Assistant config corruption):

```bash
# 1. Scale down the affected application
kubectl scale deployment home-assistant -n home-assistant --replicas=0

# 2. Delete the PVC (this will delete the data)
kubectl delete pvc home-assistant-config-pvc -n home-assistant

# 3. Recreate the PVC (it will be recreated automatically by the deployment)
kubectl scale deployment home-assistant -n home-assistant --replicas=1

# 4. Wait for new pod to create empty volume, then scale down again
kubectl scale deployment home-assistant -n home-assistant --replicas=0

# 5. Run restore job
kubectl apply -f - <<EOF
apiVersion: batch/v1
kind: Job
metadata:
  name: restore-home-assistant-$(date +%Y%m%d-%H%M%S)
  namespace: backup-system
spec:
  template:
    spec:
      restartPolicy: Never
      containers:
      - name: restore
        image: alpine:3.18
        command: ["/bin/sh", "/scripts/restore.sh"]
        args: ["latest"]  # or specific backup date like "20250921-020000"
        volumeMounts:
        - name: backup-storage
          mountPath: /backups
        - name: target-data
          mountPath: /target
        - name: backup-scripts
          mountPath: /scripts
      volumes:
      - name: backup-storage
        persistentVolumeClaim:
          claimName: backup-storage-pvc
      - name: target-data
        hostPath:
          path: /var/lib/rancher/k3s/storage
      - name: backup-scripts
        configMap:
          name: backup-scripts
          defaultMode: 0755
EOF

# 6. Monitor restore
kubectl logs -n backup-system -l job-name=restore-home-assistant-<timestamp> -f

# 7. Scale application back up
kubectl scale deployment home-assistant -n home-assistant --replicas=1
```

### Scenario 2: Complete Cluster Rebuild

If you need to rebuild the entire cluster:

#### Prerequisites
1. **Fresh K3s installation** on new/rebuilt node
2. **Flux CD bootstrapped** and all applications deployed
3. **NFS share accessible** from new cluster

#### Restore Process

```bash
# 1. Ensure backup system is deployed first
kubectl apply -f flux/apps/home1/backup-system.yaml

# 2. Wait for backup-system to be ready
kubectl wait --for=condition=available -n backup-system deployment/backup-browser

# 3. Scale down all applications
kubectl scale deployment home-assistant -n home-assistant --replicas=0
kubectl scale deployment jellyfin -n jellyfin --replicas=0

# 4. Delete existing PVCs to start fresh
kubectl delete pvc home-assistant-config-pvc -n home-assistant
kubectl delete pvc jellyfin-config-pvc -n jellyfin

# 5. Scale applications back up to recreate PVCs
kubectl scale deployment home-assistant -n home-assistant --replicas=1
kubectl scale deployment jellyfin -n jellyfin --replicas=1

# 6. Wait for pods to create empty volumes, then scale down
sleep 30
kubectl scale deployment home-assistant -n home-assistant --replicas=0
kubectl scale deployment jellyfin -n jellyfin --replicas=0

# 7. Run full restore
kubectl apply -f - <<EOF
apiVersion: batch/v1
kind: Job
metadata:
  name: full-restore-$(date +%Y%m%d-%H%M%S)
  namespace: backup-system
spec:
  template:
    spec:
      restartPolicy: Never
      containers:
      - name: restore
        image: alpine:3.18
        command: ["/bin/sh", "/scripts/restore.sh"]
        args: ["latest"]  # or specific backup date
        volumeMounts:
        - name: backup-storage
          mountPath: /backups
        - name: target-data
          mountPath: /target
        - name: backup-scripts
          mountPath: /scripts
      volumes:
      - name: backup-storage
        persistentVolumeClaim:
          claimName: backup-storage-pvc
      - name: target-data
        hostPath:
          path: /var/lib/rancher/k3s/storage
      - name: backup-scripts
        configMap:
          name: backup-scripts
          defaultMode: 0755
EOF

# 8. Monitor restore progress
kubectl logs -n backup-system -l job-name=full-restore-<timestamp> -f

# 9. Scale applications back up
kubectl scale deployment home-assistant -n home-assistant --replicas=1
kubectl scale deployment jellyfin -n jellyfin --replicas=1

# 10. Verify applications are working
kubectl get pods -n home-assistant
kubectl get pods -n jellyfin
```

## Monitoring & Maintenance

### Check Backup Health
```bash
# View backup CronJob status
kubectl get cronjob -n backup-system

# Check recent backup logs
kubectl logs -n backup-system -l app=cronjob-daily-backup --tail=100

# List available backups
kubectl exec -n backup-system -c backup $(kubectl get pod -n backup-system -l app=backup-browser -o name) -- ls -la /backups/
```

### Troubleshooting

#### Backup Fails
```bash
# Check CronJob status
kubectl describe cronjob daily-backup -n backup-system

# Check failed job logs
kubectl logs -n backup-system -l app=cronjob-daily-backup --previous

# Manual test backup
kubectl create job test-backup-$(date +%Y%m%d) --from=cronjob/daily-backup -n backup-system
```

#### NFS Mount Issues
```bash
# Test NFS connectivity from cluster
kubectl run nfs-test --image=alpine:3.18 --rm -it -- sh
# Inside pod: mount -t nfs nas.lab:/mnt/red6/k8s-backup /mnt

# Check NFS server exports
showmount -e nas.lab
```

#### Storage Issues
```bash
# Check PVC status
kubectl get pvc -n backup-system

# Check available space
kubectl exec -n backup-system -c backup $(kubectl get pod -n backup-system -l app=backup-browser -o name) -- df -h /backups
```

## Security Notes

- Backups contain sensitive data (Home Assistant tokens, Jellyfin user data)
- NFS share should have appropriate access controls
- Consider encrypting backups for additional security
- Regularly test restore procedures

## Recovery Time Objectives

- **RTO (Recovery Time Objective)**: ~15 minutes for single app, ~30 minutes for full cluster
- **RPO (Recovery Point Objective)**: Up to 24 hours (daily backups)
- **Manual backup**: Available for immediate backup before major changes