# frps Helm Chart

[frp](https://github.com/fatedier/frp) is a fast reverse proxy that lets you expose a local server behind a NAT or firewall to the internet. This chart deploys **frps** (the server component).

## Introduction

This chart deploys frps configured for HTTP vhost subdomain routing. TLS termination is handled by the ingress controller (e.g. Traefik + cert-manager), and frps routes plain HTTP traffic to frpc tunnels based on the Host header.

## Prerequisites

- Kubernetes 1.19+
- Helm 3.2.0+
- An ingress controller (e.g. Traefik, nginx)
- cert-manager (optional, for automatic TLS certificates)

## Installing the Chart

```bash
helm repo add mvanduijker https://mvanduijker.github.io/mvanduijker-helm-charts
helm repo update

helm install my-frps mvanduijker/frps
```

## Uninstalling the Chart

```bash
helm uninstall my-frps
```

## Configuration

| Parameter | Description | Default |
|-----------|-------------|---------|
| `replicaCount` | Number of replicas | `1` |
| `image.repository` | Image repository | `fatedier/frps` |
| `image.tag` | Image tag (defaults to appVersion) | `""` |
| `image.pullPolicy` | Image pull policy | `IfNotPresent` |
| `serviceAccount.create` | Create service account | `true` |
| `serviceAccount.automount` | Automount service account token | `false` |
| `serviceAccount.name` | Service account name | `""` |
| `serviceAccount.annotations` | Service account annotations | `{}` |
| `podAnnotations` | Pod annotations | `{}` |
| `podLabels` | Pod labels | `{}` |
| `podSecurityContext` | Pod security context | See values.yaml |
| `securityContext` | Container security context | See values.yaml |
| `deploymentAnnotations` | Deployment annotations | `{}` |
| `auth.token` | Auth token for frpc clients (stored in a Secret) | `""` |
| `auth.existingSecret` | Name of an existing Secret containing key `auth-token` | `""` |
| `service.type` | HTTP vhost service type | `ClusterIP` |
| `service.port` | HTTP vhost service port | `8080` |
| `service.annotations` | HTTP vhost service annotations | `{}` |
| `bind.service.type` | Bind service type (frpc connects here) | `LoadBalancer` |
| `bind.service.port` | Bind service port | `7000` |
| `bind.service.annotations` | Bind service annotations | `{}` |
| `ingress.enabled` | Enable ingress for vhost traffic | `false` |
| `ingress.className` | Ingress class name | `""` |
| `ingress.annotations` | Ingress annotations | `{}` |
| `ingress.hosts` | Ingress host rules | `*.example.com` |
| `ingress.tls` | Ingress TLS configuration | `[]` |
| `config` | frps TOML configuration | See values.yaml |
| `resources` | Resource requests/limits | `{}` |
| `livenessProbe` | Liveness probe configuration | tcpSocket on port http |
| `readinessProbe` | Readiness probe configuration | tcpSocket on port http |
| `networkPolicy.enabled` | Enable NetworkPolicy | `false` |
| `nodeSelector` | Node selector | `{}` |
| `tolerations` | Tolerations | `[]` |
| `affinity` | Affinity rules | `{}` |
| `extraObjects` | Additional Kubernetes objects | `[]` |

## Authentication

Set `auth.token` to require frpc clients to authenticate with a shared secret. The token is stored in a Kubernetes Secret and injected via environment variable — it never appears in the ConfigMap.

```yaml
auth:
  token: "my-secret-token"
```

Alternatively, if you manage secrets externally (e.g. with external-secrets or sealed-secrets), point to an existing Secret:

```yaml
auth:
  existingSecret: "my-frps-secret"
```

The existing Secret must contain the key `auth-token`.

On the frpc client side, set the matching token:

```toml
auth.token = "my-secret-token"
```

## Ingress and TLS

TLS is terminated at the ingress controller; frps receives plain HTTP.

### Example with wildcard TLS

```yaml
ingress:
  enabled: true
  className: "traefik"
  annotations:
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
  hosts:
    - host: "*.example.com"
      paths:
        - path: /
          pathType: Prefix
  tls:
    - secretName: frps-wildcard-tls
      hosts:
        - "*.example.com"
```

## Connecting with frpc

frpc connects to frps on the **bind port** (default 7000). The chart creates a dedicated `LoadBalancer` service for this so frpc clients can reach it from outside the cluster.

Get the external IP of the bind service:

```bash
kubectl get svc my-frps-bind
```

### Example frpc.toml

Assuming frps is deployed with:
- bind service at `frps.example.com:7000`
- wildcard ingress for `*.example.com`
- auth token set

```toml
serverAddr = "frps.example.com"
serverPort = 7000
auth.token = "my-secret-token"

[[proxies]]
name = "my-web-app"
type = "http"
localPort = 3000
customDomains = ["myapp.example.com"]
```

This exposes your local port 3000 at `https://myapp.example.com` (TLS handled by the ingress controller).

## Custom Configuration

The `config` value is written directly to `frps.toml`. Auth settings are appended automatically when enabled. See the [full example config](https://github.com/fatedier/frp/blob/dev/conf/frps_full_example.toml) for all options.

**Important:** The ports in your config must match the service port values:
- `bindPort` must match `bind.service.port` (default: 7000)
- `vhostHTTPPort` must match `service.port` (default: 8080)

```yaml
config: |
  bindPort = 7000
  vhostHTTPPort = 8080
  transport.maxPoolCount = 10
```
