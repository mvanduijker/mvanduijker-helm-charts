# GoatCounter Helm Chart

[GoatCounter](https://github.com/arp242/goatcounter) is an open source web analytics platform available as a self-hosted app. It aims to offer easy to use and meaningful privacy-friendly web analytics as an alternative to Google Analytics or Matomo.

## Introduction

This chart deploys GoatCounter on Kubernetes with SQLite as the default database. TLS termination is handled by the Traefik ingress controller.

## Prerequisites

- Kubernetes 1.19+
- Helm 3.2.0+
- Traefik ingress controller (configured with `kubernetes.io/ingress.class: traefik`)
- cert-manager (optional, for automatic TLS certificates)

## Installing the Chart

```bash
helm repo add mvanduijker https://mvanduijker.github.io/mvanduijker-helm-charts
helm repo update

helm install my-goatcounter mvanduijker/goatcounter
```

## Uninstalling the Chart

```bash
helm uninstall my-goatcounter
```

The command removes all the Kubernetes components associated with the chart and deletes the release.

## Configuration

The following table lists the configurable parameters of the GoatCounter chart and their default values.

| Parameter | Description | Default |
|-----------|-------------|---------|
| `replicaCount` | Number of replicas | `1` |
| `image.repository` | Image repository | `arp242/goatcounter` |
| `image.tag` | Image tag | `latest` |
| `image.pullPolicy` | Image pull policy | `IfNotPresent` |
| `serviceAccount.create` | Create service account | `true` |
| `serviceAccount.name` | Service account name | `` |
| `podAnnotations` | Pod annotations | `{}` |
| `podLabels` | Pod labels | `{}` |
| `podSecurityContext` | Pod security context | `{}` |
| `securityContext` | Container security context | `{}` |
| `service.type` | Service type | `ClusterIP` |
| `service.port` | Service port | `8080` |
| `ingress.enabled` | Enable ingress | `true` |
| `ingress.className` | Ingress class name | `traefik` |
| `ingress.annotations` | Ingress annotations | `{}` |
| `ingress.hosts` | Host configurations | `chart-example.local` |
| `ingress.tls` | TLS configurations | `[]` |
| `resources` | Resource requests/limits | `{}` |
| `livenessProbe` | Liveness probe configuration | TCP socket on port 8080 |
| `readinessProbe` | Readiness probe configuration | TCP socket on port 8080 |
| `autoscaling.enabled` | Enable HPA | `false` |
| `nodeSelector` | Node selector | `{}` |
| `tolerations` | Tolerations | `[]` |
| `affinity` | Affinity rules | `{}` |
| `storage.enabled` | Enable PVC | `true` |
| `storage.existingClaim` | Existing PVC name | `""` |
| `storage.size` | PVC size | `1Gi` |
| `storage.className` | Storage class | `""` |
| `storage.accessMode` | Access mode | `ReadWriteOnce` |
| `extraEnv` | Additional environment variables | `[]` |
| `extraVolumeMounts` | Additional volume mounts | `[]` |
| `extraVolumes` | Additional volumes | `[]` |
| `extraObjects` | Additional K8s objects | `[]` |

## Database Configuration

By default, GoatCounter uses SQLite stored in a PersistentVolume. The database is located at `/home/goatcounter/goatcounter-data/goatcounter.sqlite3`.

### Using an Existing PVC

```yaml
storage:
  existingClaim: my-existing-pvc
```

### PostgreSQL (Optional)

To use PostgreSQL instead of SQLite, disable storage and set the database connection via environment variable:

```yaml
storage:
  enabled: false

extraEnv:
  - name: GOATCOUNTER_DB
    value: "postgresql+postgresql://user:password@hostname:5432/dbname?sslmode=disable"
```

## Ingress and TLS

The chart enables ingress with Traefik by default. TLS is handled by the ingress controller (Traefik) with cert-manager for automatic certificate provisioning.

### Example with TLS

```yaml
ingress:
  enabled: true
  className: "traefik"
  annotations:
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
  hosts:
    - host: stats.example.com
      paths:
        - path: /
          pathType: Prefix
  tls:
    - secretName: goatcounter-tls
      hosts:
        - stats.example.com
```

## Additional Environment Variables

GoatCounter supports various environment variables for configuration. Common ones include:

```yaml
extraEnv:
  - name: GOATCOUNTER_EMAIL_FROM
    value: "admin@example.com"
```

GoatCounter reads environment variables based on CLI flag names with the `GOATCOUNTER_` prefix. For example, `-email-from` becomes `GOATCOUNTER_EMAIL_FROM`. See the [GoatCounter documentation](https://github.com/arp242/goatcounter/blob/master/README.md#running) for all available options.

## Post-Installation Setup

After installing the chart, you need to create the first site manually:

1. Wait for the pod to be ready:

   ```bash
   kubectl wait --for=condition=ready pod -l app.kubernetes.io/name=goatcounter --timeout=120s
   ```

2. Create the first site:

   ```bash
   kubectl exec -it deploy/my-goatcounter -- goatcounter db create site -vhost=stats.example.com -user.email=you@example.com
   ```

3. Follow the prompts to set a password.

4. Access GoatCounter at your configured domain.

## Upgrading

### Upgrading the Image

```bash
helm upgrade my-goatcounter mvanduijker/goatcounter --set image.tag=<new-version>
```

### Database Migrations

GoatCounter automatically runs migrations on startup when `GOATCOUNTER_AUTOMIGRATE` is set to `true` (default).
