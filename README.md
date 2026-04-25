# restic-ops Helm Chart

## Usage

1. **Create a Kubernetes secret** with your cloud credentials and restic password (see "Secrets Setup" below).
2. **Prepare your custom values file** (see example in "Quick Start").
3. **Add the Helm repo and install the chart:**
  ```bash
  helm repo add restic-ops https://ashwinijindal10.github.io/restic-ops-helm-repo/charts
  helm repo update
  helm upgrade --install restic-ops restic-ops/restic-ops \
    -f my-values.yaml \
    -n backup --create-namespace
  ```

For detailed configuration and troubleshooting, see the sections below in the README.

A production-ready Helm chart for scheduled Kubernetes volume backups using restic, with an optional Backrest UI for management and monitoring.

## Features

- ✅ **Parameterized CronJob** for restic backups with dynamic scheduling
- ✅ **Multiple backup targets** - Support unlimited PVCs to backup
- ✅ **Configurable retention** - Daily, weekly, monthly snapshot retention
- ✅ **Backrest UI** - Optional web UI for backup monitoring and management
- ✅ **Secure secrets** - All credentials managed via Kubernetes secrets
- ✅ **Node selectors** - Pin UI and jobs to specific nodes
- ✅ **Production-ready** - Resource limits, job history, TTL, concurrency control
- ✅ **Highly customizable** - Override any value via custom values.yaml

## Requirements

- Kubernetes 1.21+
- Helm 3.0+
- Restic backend (S3, Azure, GCS, OCI, etc.)
- Pre-created Kubernetes secret with cloud credentials
- PVCs already created for volumes to backup

## Quick Start

### 1. Create the backup secret

```bash
kubectl create namespace backup

kubectl create secret generic restic-secret \
  --from-literal=AWS_ACCESS_KEY_ID=your-access-key \
  --from-literal=AWS_SECRET_ACCESS_KEY=your-secret-key \
  --from-literal=BUCKET_ENDPOINT=s3:https://s3.example.com/backups \
  --from-literal=RESTIC_PASSWORD=your-secure-password \
  -n backup
```

### 2. Create your custom values

```bash
cat > my-values.yaml << EOF
secretName: restic-secret

backup:
  schedule: "0 2 * * *"  # Daily at 2 AM UTC
  retention:
    daily: 7
    weekly: 4
    monthly: 3
  targets:
    - name: app-data
      mountPath: /data
      pvc: pvc-app-data
      storageClass: standard
      size: 50Gi

enableBackrestUI: true
ui:
  ingress:
    enabled: true
    host: "backrest.example.com"
    tls: true
EOF
```



## Configuration

### Core Parameters

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `secretName` | string | `restic-secret` | Secret containing cloud credentials and password |
| `enableBackrestUI` | bool | `true` | Deploy Backrest UI dashboard |
| `createBackupTargetPVCs` | bool | `false` | Auto-create backup target PVCs |

### Backup Configuration

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `backup.schedule` | string | `"0 2 * * *"` | Cron schedule (UTC) |
| `backup.region` | string | unset | Optional cloud region for backup storage; restic defaults to `us-east-1` when omitted |
| `backup.retention.daily` | int | `7` | Keep daily snapshots |
| `backup.retention.weekly` | int | `4` | Keep weekly snapshots |
| `backup.retention.monthly` | int | `3` | Keep monthly snapshots |
| `backup.concurrencyPolicy` | string | `Forbid` | Allow/Forbid/Replace parallel jobs |
| `backup.successHistory` | int | `2` | Keep last N successful jobs |
| `backup.failHistory` | int | `2` | Keep last N failed jobs |
| `backup.backoffLimit` | int | `2` | Max retries per job |
| `backup.ttlSecondsAfterFinished` | int | `36000` | Delete finished jobs after (seconds) |
| `backup.nodeSelector` | map | `{}` | Node labels for job pods |
| `backup.resources` | map | See defaults | CPU/memory requests and limits |

### Backup Targets

Define multiple PVCs to backup:

```yaml
backup:
  targets:
    - name: app-data
      mountPath: /data
      pvc: pvc-app-data
      storageClass: standard
      size: 50Gi
    - name: database
      mountPath: /db
      pvc: pvc-database
      storageClass: nfs
      size: 100Gi
```

### Backrest UI Configuration

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `enableBackrestUI` | bool | `true` | Enable/disable UI |
| `replicas` | int | `1` | UI deployment replicas |
| `image` | string | `ghcr.io/garethgeorge/backrest:latest` | UI container image |
| `ui.storageClass` | string | `standard` | Storage class for Backrest UI data PVC |
| `ui.storageSize` | string | `1Gi` | Storage size for Backrest UI data PVC |
| `ui.nodeSelector` | map | `{}` | Node labels for UI pods |
| `ui.service.type` | string | `ClusterIP` | Service type (ClusterIP/LoadBalancer) |
| `ui.ingress.enabled` | bool | `false` | Enable Ingress |
| `ui.ingress.host` | string | `backrest.example.com` | Ingress hostname |
| `ui.ingress.tls` | bool | `false` | Enable TLS |
| `ui.traefik.ingressRoute.enabled` | bool | `false` | Enable Traefik IngressRoute instead of standard Ingress |
| `ui.traefik.ingressRoute.host` | string | `backrest.example.com` | Traefik IngressRoute hostname |
| `ui.traefik.ingressRoute.entryPoints` | list | `[websecure]` | Traefik entry points |
| `ui.traefik.ingressRoute.auth.enabled` | bool | `false` | Create and attach a Traefik basicAuth middleware |
| `ui.traefik.ingressRoute.auth.secretName` | string | `ext-secrets` | Secret used by Traefik basicAuth middleware |
| `ui.traefik.ingressRoute.headers.enabled` | bool | `true` | Create and attach a Traefik headers middleware |

## Secrets Setup

### Required Secret Keys

The Kubernetes secret (referenced by `secretName`) must contain:

**For All Backends:**
```yaml
RESTIC_PASSWORD: <strong-encryption-password>  # Restic repository encryption password
BUCKET_ENDPOINT: <restic-repository-base-url>   # Example: s3:https://s3.example.com/backups
```

**For S3/AWS:**
```yaml
AWS_ACCESS_KEY_ID: <your-key>
AWS_SECRET_ACCESS_KEY: <your-secret>
# AWS_DEFAULT_REGION is optional; set backup.region only if your provider requires it
```

**For OCI:**
```yaml
BUCKET_ENDPOINT: s3:https://objectstorage.us-phoenix-1.oraclecloud.com/<namespace>/<bucket>
AWS_ACCESS_KEY_ID: <your-customer-key>
AWS_SECRET_ACCESS_KEY: <your-customer-secret>
# AWS_DEFAULT_REGION is optional; set backup.region only if your provider requires it
```

**For Azure:**
```yaml
AZURE_ACCOUNT_NAME: <account-name>
AZURE_ACCOUNT_KEY: <account-key>
```

**Example: Create S3 secret**
```bash
kubectl create secret generic restic-secret \
  --from-literal=AWS_ACCESS_KEY_ID=your-key \
  --from-literal=AWS_SECRET_ACCESS_KEY=your-secret \
  --from-literal=BUCKET_ENDPOINT=s3:https://s3.example.com/backups \
  --from-literal=RESTIC_PASSWORD=your-strong-password \
  -n backup  # Use your namespace
```

See [restic documentation](https://restic.readthedocs.io/en/latest/030_preparing_a_new_repo.html) for your specific backend.

## Cron Schedule Examples

```
"0 2 * * *"        # Daily at 2 AM UTC
"0 2,14 * * *"     # Twice daily (2 AM and 2 PM UTC)
"0 3 * * 0"        # Weekly on Sunday at 3 AM UTC
"0 4 1 * *"        # Monthly on 1st at 4 AM UTC
"*/6 * * * *"      # Every 6 hours
```

## Monitoring & Troubleshooting

### View backup jobs

```bash
kubectl get cronjobs -n backup
kubectl get jobs -n backup
kubectl logs -n backup -l app=restic --tail=100
```

### Access Backrest UI

```bash
# Port forward (if ingress not enabled)
kubectl port-forward -n backup svc/backrest 8080:80

# Then open: http://localhost:8080
```

### Check backup status

```bash
# List recent backup snapshots
kubectl exec -it -n backup <pod-name> -- \
  sh -c 'RESTIC_REPOSITORY=s3:... restic snapshots'

# Verify backup integrity
kubectl exec -it -n backup <pod-name> -- \
  sh -c 'RESTIC_REPOSITORY=s3:... restic check'
```

## Production Best Practices

1. **Secure credentials**: Store all secrets (RESTIC_PASSWORD, AWS keys, OCI credentials) in Kubernetes secrets or external secret managers - never in values.yaml
2. **Use ExternalSecrets**: For production, use ExternalSecrets Operator to sync secrets from OCI Vault, AWS Secrets Manager, or similar
3. **Secure secrets**: Use RBAC, encryption at rest, and audit logging
4. **Test restores**: Regularly test restore procedures to ensure backups are usable
5. **Monitor jobs**: Set up alerts for failed backup jobs
6. **Retention policy**: Adjust retention to match your requirements and disk space
7. **Resource limits**: Adjust CPU/memory based on data size and cluster capacity
8. **Node affinity**: Use nodeSelectors to run backups on stable nodes with reliable storage access
9. **Network**: Ensure reliable connectivity to backup backend with retry logic
10. **Versioning**: Pin image versions instead of using `latest` tags
11. **Backup frequency**: Balance backup frequency with resource usage and recovery needs
12. **Separate secrets per environment**: Use different Kubernetes secrets for dev/staging/prod

## Common Issues

### CronJob not running
- Check if namespace exists: `kubectl get ns`
- Verify CronJob: `kubectl describe cronjob -n backup`
- Check controller logs: `kubectl logs -n kube-system -l component=cronjob-controller`

### Backup fails with "no such file"
- Verify PVC is mounted correctly
- Check mountPath in values matches actual mount point
- Verify PVC is in same namespace

### UI cannot connect to backups
- Ensure AWS_ACCESS_KEY_ID, AWS_SECRET_ACCESS_KEY are in secret
- Verify BUCKET_ENDPOINT repository base URL is correct and accessible
- Check network policies allow outbound connections

### Out of memory/CPU errors
- Increase `backup.resources.limits` for larger datasets
- Reduce `replicas` or spread backups across time

## Helm Commands

```bash
# Lint the chart
helm lint .

# Template (dry run)
helm template restic-ops . -f my-values.yaml -n backup

# Install
helm upgrade --install restic-ops restic-ops/restic-ops \
  -f my-values.yaml \
  -n backup --create-namespace

# Upgrade
helm upgrade restic-ops restic-ops/restic-ops \
  -f my-values.yaml \
  -n backup

# Uninstall
helm uninstall restic-ops -n backup

# Get values
helm get values restic-ops -n backup
```

## Development & Contributing

- Chart version: 0.1.0
- Restic version: Latest
- Backrest version: Latest

## Rebuild and publish chart (after changes)

Run these steps when Chart.yaml, templates/, or chart behavior changes.

Bump version in Chart.yaml for each release.
Package chart into docs/charts.
Rebuild chart index with the same repository base URL.
Commit and push.
# from repo root

helm lint .
helm package . --destination docs/charts
helm repo index docs/charts \
  --url https://ashwinijindal10.github.io/restic-ops-helm-repo/charts

git add .
git commit -m "chore(release): package chart and update index"
git push origin main

## License

MIT

## Support

For issues, questions, or contributions, please open an issue on GitHub.
