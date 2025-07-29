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
# Basic installation
helm install my-postgres postgresql/postgresql

# With custom namespace
helm install my-postgres postgresql/postgresql -n database --create-namespace

# With custom values
helm install my-postgres postgresql/postgresql -f my-values.yaml
```

### Uninstall
```bash
helm uninstall my-postgres
```

## Features

- **StatefulSet** deployment with persistent storage
- **Security** contexts and secrets management
- **Health checks** with liveness and readiness probes
- **Auto scaling** with HPA support
- **Flexible configuration** via ConfigMaps and Secrets
- **Service discovery** ready
- **Backup scheduling** via CronJobs

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

Secrets:
  POSTGRES_PASSWORD: "your-secure-password"
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

## Access PostgreSQL

### Get Password
```bash
export POSTGRES_PASSWORD=$(kubectl get secret my-postgres-secret -o jsonpath="{.data.POSTGRES_PASSWORD}" | base64 -d)
echo $POSTGRES_PASSWORD
```

### Port Forward
```bash
kubectl port-forward service/my-postgres-service 5432:5432
```

### Connect
```bash
psql -h localhost -U myuser -d mydb
```

## Backup & Restore

### Manual Backup
```bash
kubectl exec -it my-postgres-0 -- pg_dump -U myuser mydb > backup.sql
```

### Restore
```bash
kubectl exec -i my-postgres-0 -- psql -U myuser -d mydb < backup.sql
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
        pg_dump -h my-postgres-service -U $POSTGRES_USER $POSTGRES_DB > /backup/backup-$(date +%Y%m%d).sql
    env:
      - name: POSTGRES_USER
        valueFrom:
          secretKeyRef:
            name: my-postgres-secret
            key: POSTGRES_USER
    # Add your backup storage volume here
```

## Configuration Reference

| Parameter | Description | Default |
|-----------|-------------|---------|
| `statefulset` | Deploy as StatefulSet instead of Deployment | `true` |
| `replicaCount` | Number of replicas | `1` |
| `image.repository` | PostgreSQL image repository | `postgres` |
| `image.tag` | PostgreSQL image tag | `17-alpine` |
| `persistence.enabled` | Enable persistent storage | `true` |
| `Secrets.POSTGRES_PASSWORD` | PostgreSQL password | `SecurePassword123!` |
| `Secrets.POSTGRES_USER` | PostgreSQL user | `konga` |
| `Secrets.POSTGRES_DB` | PostgreSQL database | `konga` |
| `resources.requests.memory` | Memory request | `256Mi` |
| `resources.requests.cpu` | CPU request | `100m` |
| `autoscaling.enabled` | Enable Horizontal Pod Autoscaler | `false` |
| `service.type` | Kubernetes service type | `ClusterIP` |

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

### Connection refused
Make sure the service is running and accessible:
```bash
kubectl get svc
kubectl describe pod my-postgres-0
```

### Password authentication failed
Verify the secret was created correctly:
```bash
kubectl get secret my-postgres-secret -o yaml
```

## Development

### Local Testing
```bash
# Install locally
helm install test-postgres . -f values.yaml

# Test template rendering
helm template test-postgres . -f values.yaml

# Debug
helm install test-postgres . --dry-run --debug
```

### Contributing
Feel free to open issues or submit pull requests. This is a personal project but contributions are welcome.

## License

This chart is provided as-is under the MIT License. Use at your own risk.

---

**Note**: This is a personal Helm chart. While it's designed to be production-ready, please review and test thoroughly before using in critical environments.