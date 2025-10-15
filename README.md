# PostgreSQL Helm Chart

A simple, production-ready Helm chart for deploying PostgreSQL on Kubernetes.

![Version: 0.1.0](https://img.shields.io/badge/Version-0.1.0-informational?style=flat-square) ![Type: application](https://img.shields.io/badge/Type-application-informational?style=flat-square) ![AppVersion: 17-alpine](https://img.shields.io/badge/AppVersion-17--alpine-informational?style=flat-square)

## Quick Start

### Add Helm Repository
```bash
helm repo add postgresql https://leozw.github.io/postgresql/
helm repo update
```

### Install
```bash
# Basic installation with custom values
helm install my-postgres postgresql/postgresql \
  --set Secrets.POSTGRES_PASSWORD=YourSecurePassword123! \
  --set Secrets.POSTGRES_USER=myuser \
  --set Secrets.POSTGRES_DB=mydb

# With custom namespace
helm install my-postgres postgresql/postgresql \
  -n database --create-namespace \
  --set Secrets.POSTGRES_PASSWORD=YourSecurePassword123!

# With custom values file
helm install my-postgres postgresql/postgresql -f my-values.yaml
```

### Uninstall
```bash
helm uninstall my-postgres
```

## Features

- **StatefulSet** deployment with persistent storage
- **Enhanced Security** with Pod Security Standards, NetworkPolicy, and secure contexts
- **High Availability** with anti-affinity rules and topology spread constraints
- **Health checks** with liveness and readiness probes
- **Auto scaling** with HPA v2 support and advanced scaling behaviors
- **Flexible configuration** via ConfigMaps and Secrets
- **Service discovery** ready
- **Automated backup** scheduling via CronJobs with retention policies
- **Monitoring** ready with Prometheus ServiceMonitor support
- **PodDisruptionBudget** for availability during updates
- **Production-ready** configurations with security best practices

## Configuration

### Basic Example
```yaml
# values.yaml
statefulset: true
replicaCount: 1

image:
  repository: postgres
  tag: "17-alpine"

persistence:
  enabled: true
  volumeClaimTemplates:
    - name: postgres-storage
      resources:
        requests:
          storage: 10Gi

# IMPORTANT: Set strong passwords in production!
Secrets:
  POSTGRES_PASSWORD: "YourSecurePassword123!"
  POSTGRES_USER: "myuser"
  POSTGRES_DB: "mydb"
```

### Production Example
```yaml
# production-values.yaml
replicaCount: 3
statefulset: true

resources:
  requests:
    memory: 2Gi
    cpu: 1000m
  limits:
    memory: 4Gi
    cpu: 2000m

persistence:
  enabled: true
  volumeClaimTemplates:
    - name: postgres-storage
      storageClassName: "fast-ssd"
      resources:
        requests:
          storage: 100Gi

# Use Kubernetes secrets management in production
Secrets:
  POSTGRES_PASSWORD: ""  # Set via --set or external secret manager
  POSTGRES_USER: "produser"
  POSTGRES_DB: "proddb"

autoscaling:
  enabled: true
  minReplicas: 2
  maxReplicas: 10
  targetCPUUtilizationPercentage: 70

nodeSelector:
  node-type: database

tolerations:
  - key: "database"
    operator: "Equal"
    value: "true"
    effect: "NoSchedule"
```

## Security Best Practices

### Enhanced Security Features

This chart includes several security enhancements inspired by Bitnami's best practices:

- **Pod Security Standards**: Enforces restricted security policies
- **NetworkPolicy**: Controls network traffic to/from PostgreSQL pods
- **Secure Contexts**: Runs containers as non-root with minimal capabilities
- **Service Account**: Dedicated service account with controlled permissions
- **Seccomp Profiles**: Runtime security profiles for additional protection

### Using External Secrets

Instead of hardcoding passwords in values.yaml, you can:

1. **Set via command line:**
```bash
helm install my-postgres postgresql/postgresql \
  --set Secrets.POSTGRES_PASSWORD="$(openssl rand -base64 32)"
```

2. **Use existing Kubernetes secret:**
```yaml
# Create secret manually first
kubectl create secret generic postgres-credentials \
  --from-literal=POSTGRES_PASSWORD=your-password \
  --from-literal=POSTGRES_USER=myuser \
  --from-literal=POSTGRES_DB=mydb

# Reference in values.yaml
secretName: "postgres-credentials"
Secrets: {}  # Leave empty when using existing secret
```

3. **Use external secret managers** (e.g., AWS Secrets Manager, HashiCorp Vault):
```yaml
envFrom:
  - secretRef:
      name: postgres-external-secret
```

## Access PostgreSQL

### Get Password
```bash
export POSTGRES_PASSWORD=$(kubectl get secret my-postgres-secret \
  -o jsonpath="{.data.POSTGRES_PASSWORD}" | base64 -d)
echo $POSTGRES_PASSWORD
```

### Port Forward
```bash
kubectl port-forward service/my-postgres-service 5432:5432
```

### Connect
```bash
# Using psql
psql -h localhost -U myuser -d mydb

# Or with password in command
PGPASSWORD=$POSTGRES_PASSWORD psql -h localhost -U myuser -d mydb
```

## Backup & Restore

### Manual Backup
```bash
kubectl exec -it my-postgres-0 -- \
  pg_dump -U myuser mydb > backup-$(date +%Y%m%d).sql
```

### Restore
```bash
kubectl exec -i my-postgres-0 -- \
  psql -U myuser -d mydb < backup-20240101.sql
```

### Automated Backup (CronJob)
```yaml
CronJobs:
  - name: postgres-backup
    schedule: "0 2 * * *"  # Every day at 2 AM
    image:
      repository: postgres
      tag: "17-alpine"
    command: ["sh", "-c"]
    args:
      - |
        BACKUP_FILE="/backup/backup-$(date +%Y%m%d-%H%M%S).sql"
        pg_dump -h my-postgres-service -U $POSTGRES_USER $POSTGRES_DB > $BACKUP_FILE
        echo "Backup created: $BACKUP_FILE"
    env:
      - name: POSTGRES_USER
        valueFrom:
          secretKeyRef:
            name: my-postgres-secret
            key: POSTGRES_USER
      - name: POSTGRES_PASSWORD
        valueFrom:
          secretKeyRef:
            name: my-postgres-secret
            key: POSTGRES_PASSWORD
      - name: POSTGRES_DB
        valueFrom:
          secretKeyRef:
            name: my-postgres-secret
            key: POSTGRES_DB
    volumeMounts:
      - name: backup-storage
        mountPath: /backup
    volumes:
      - name: backup-storage
        persistentVolumeClaim:
          claimName: postgres-backup-pvc
    restartPolicy: OnFailure
    concurrencyPolicy: Forbid
    successfulJobsHistoryLimit: 3
    failedJobsHistoryLimit: 1
```

## High Availability & Production

### Production Deployment

For production environments, use the provided production values:

```bash
# Deploy with production configuration
helm install my-postgres postgresql/postgresql \
  -f values-production.yaml \
  --set Secrets.POSTGRES_PASSWORD="$(openssl rand -base64 32)" \
  --namespace database --create-namespace
```

### High Availability Features

- **Anti-Affinity Rules**: Ensures pods are distributed across different nodes
- **Topology Spread Constraints**: Controls pod distribution for optimal availability
- **PodDisruptionBudget**: Maintains availability during cluster updates
- **Multi-Replica Support**: Run multiple PostgreSQL instances for redundancy

### Monitoring & Observability

Enable monitoring with Prometheus:

```yaml
monitoring:
  enabled: true
  serviceMonitor:
    enabled: true
    labels:
      prometheus: kube-prometheus
```

### Automated Backups

Configure automated backups with retention:

```yaml
backup:
  enabled: true
  schedule: "0 2 * * *"  # Daily at 2 AM
  retention: "30d"       # Keep for 30 days
  storage:
    size: 50Gi
    storageClass: "fast-ssd"
```

## Configuration Reference

| Parameter | Description | Default |
|-----------|-------------|---------|
| `statefulset` | Deploy as StatefulSet instead of Deployment | `true` |
| `replicaCount` | Number of replicas | `1` |
| `image.repository` | PostgreSQL image repository | `postgres` |
| `image.tag` | PostgreSQL image tag | `17-alpine` |
| `persistence.enabled` | Enable persistent storage | `true` |
| `Secrets.POSTGRES_PASSWORD` | PostgreSQL password (REQUIRED) | `""` |
| `Secrets.POSTGRES_USER` | PostgreSQL user | `postgres` |
| `Secrets.POSTGRES_DB` | PostgreSQL database | `postgres` |
| `resources.requests.memory` | Memory request | `256Mi` |
| `resources.requests.cpu` | CPU request | `100m` |
| `resources.limits.memory` | Memory limit | `1Gi` |
| `resources.limits.cpu` | CPU limit | `1000m` |
| `autoscaling.enabled` | Enable Horizontal Pod Autoscaler | `false` |
| `service.type` | Kubernetes service type | `ClusterIP` |
| `secretName` | Custom secret name | `<release-name>-secret` |
| `networkPolicy.enabled` | Enable NetworkPolicy | `false` |
| `podDisruptionBudget.enabled` | Enable PodDisruptionBudget | `false` |
| `highAvailability.enabled` | Enable HA features | `false` |
| `backup.enabled` | Enable automated backups | `false` |
| `monitoring.enabled` | Enable monitoring | `false` |

See [values.yaml](values.yaml) for all available options.

## Requirements

- Kubernetes 1.19+
- Helm 3.0+
- PV provisioner support in the underlying infrastructure (for persistence)

## Common Issues

### Pod stuck in Pending
Check if you have a storage class configured:
```bash
kubectl get storageclass
```

If no default storage class exists, specify one in values.yaml:
```yaml
persistence:
  volumeClaimTemplates:
    - storageClassName: "standard"  # or your storage class name
```

### Connection refused
Make sure the service is running and accessible:
```bash
kubectl get svc
kubectl get pods
kubectl describe pod my-postgres-0
kubectl logs my-postgres-0
```

### Password authentication failed
Verify the secret was created correctly:
```bash
kubectl get secret my-postgres-secret -o yaml

# Check password value
kubectl get secret my-postgres-secret \
  -o jsonpath="{.data.POSTGRES_PASSWORD}" | base64 -d
```

### Empty password error
The chart requires a password to be set. Either:
- Set it via `--set Secrets.POSTGRES_PASSWORD=yourpassword`
- Provide it in your values.yaml file
- Use an existing secret with `secretName`

## Development

### Local Testing
```bash
# Install locally with password
helm install test-postgres . \
  --set Secrets.POSTGRES_PASSWORD=testpass123

# Test template rendering
helm template test-postgres . -f values.yaml \
  --set Secrets.POSTGRES_PASSWORD=testpass123

# Debug
helm install test-postgres . --dry-run --debug \
  --set Secrets.POSTGRES_PASSWORD=testpass123
```

### Contributing
Feel free to open issues or submit pull requests. This is a personal project but contributions are welcome.

## License

This chart is provided as-is under the MIT License. Use at your own risk.

---

**Note**: This is a personal Helm chart. While it's designed to be production-ready, please review and test thoroughly before using in critical environments. Always use strong passwords and proper secrets management in production.