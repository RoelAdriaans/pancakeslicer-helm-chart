# PancakeSlicer Helm chart

> **Note:** This chart is experimental and is not officially supported by the original author of PancakeSlicer. Use at your own risk.

Deploys PancakeSlicer (web UI, service and slicing engine in one container) on Kubernetes. The app listens on port 6868 and stores its data in `/data`.

## Install

### From the Helm repository

```sh
helm repo add pancakeslicer https://roeladriaans.github.io/pancakeslicer-helm-chart/
helm repo update
helm install pancakeslicer pancakeslicer/pancakeslicer -n pancakeslicer
```

### From a local checkout

```sh
helm install pancakeslicer ./pancakeslicer \
  --set ingress.enabled=true \
  --set ingress.hosts[0].host=pancake.example.com \
  --set ingress.hosts[0].paths[0].path=/ \
  -n pancakeslicer
```

The chart creates the `pancakeslicer` namespace and installs all resources into it, so do not pass `--create-namespace`. Note that `helm uninstall` removes the namespace, including the data volume.

## Values

### Namespace

| Value | Default | Description |
|---|---|---|
| `namespace` | `pancakeslicer` | Namespace the chart creates and installs into. Pass the same name to `helm -n`. |

### Image

| Value | Default | Description |
|---|---|---|
| `image.repository` | `zimengxiong/pancakeslicer` | Container image. |
| `image.tag` | `0.5.4` | Image tag. |
| `image.pullPolicy` | `IfNotPresent` | Kubernetes image pull policy. |
| `replicaCount` | `1` | Number of replicas. Keep at 1: state lives on a single `ReadWriteOnce` volume and the Deployment uses the `Recreate` strategy. |

### Application config

| Value | Default | Environment variable | Description |
|---|---|---|---|
| `config.maxJobs` | `"2"` | `ORCA_WEB_MAX_JOBS` | Slices running at once; more are queued. |
| `config.printerNetwork` | `lan` | `ORCA_WEB_PRINTER_NETWORK` | `lan`, `off`, or comma-separated CIDRs/host names the app may reach for printers. |
| `config.stageCacheMaxMb` | `"1024"` | `PANCAKE_STAGE_CACHE_MAX_MB` | Cap in MB for the engine stage cache (disposable). |

### Authentication

| Value | Default | Description |
|---|---|---|
| `bearerToken` | `""` | If set, the chart creates a Secret and sets `ORCA_WEB_BEARER_TOKEN`, requiring a token for changes. Recommended when exposed. |
| `existingSecret` | `""` | Name of an existing Secret containing the key `ORCA_WEB_BEARER_TOKEN`. Takes precedence over `bearerToken` (no Secret is created). |

### Service

| Value | Default | Description |
|---|---|---|
| `service.type` | `ClusterIP` | Service type. |
| `service.port` | `6868` | Service port (the container port is always 6868). |

### Ingress

| Value | Default | Description |
|---|---|---|
| `ingress.enabled` | `false` | Create an Ingress. |
| `ingress.className` | `""` | `ingressClassName` (e.g. `nginx`). |
| `ingress.annotations` | `{}` | Ingress annotations, e.g. `nginx.ingress.kubernetes.io/proxy-body-size: 512m` for large model uploads. |
| `ingress.hosts` | one example host | List of `{host, paths: [{path, pathType}]}`. `pathType` defaults to `Prefix`. |
| `ingress.tls` | `[]` | Standard Ingress TLS list: `{secretName, hosts}`. |

#### Example: basic authentication (ingress-nginx)

> **Warning:** PancakeSlicer has no built-in user authentication. If you expose it through an Ingress, **anyone who can reach it, including the whole internet, can use it**. Always put authentication in front of it (and use TLS) before exposing it publicly.

Generate an htpasswd entry (the Secret key must be named `auth`):

```sh
htpasswd -nb admin 'my-password'   # prints admin:$apr1$...
```

Then define the Secret through `extraObjects` so it is deployed with the chart, in one release:

```yaml
extraObjects:
  - apiVersion: v1
    kind: Secret
    metadata:
      name: pancakeslicer-basic-auth
      namespace: pancakeslicer
    type: Opaque
    stringData:
      auth: "admin:$apr1$REPLACE_WITH_HTPASSWD_HASH"

ingress:
  enabled: true
  className: nginx
  annotations:
    nginx.ingress.kubernetes.io/proxy-body-size: 512m
    nginx.ingress.kubernetes.io/auth-type: basic
    nginx.ingress.kubernetes.io/auth-secret: pancakeslicer-basic-auth
    nginx.ingress.kubernetes.io/auth-realm: "Authentication Required"
  hosts:
    - host: pancake.example.com
      paths:
        - path: /
  tls:
    - secretName: pancakeslicer-tls
      hosts:
        - pancake.example.com
```

Other ingress controllers use different annotations.

> **Recommendation:** Don't store the password hash in plain text in your values files or version control. Encrypt the values with [SOPS](https://github.com/getsops/sops) (e.g. via `helm secrets`), or use an external secret provider such as [External Secrets Operator](https://external-secrets.io/) (add an `ExternalSecret` through `extraObjects` instead of the `Secret`).

### Extra objects

| Value | Default | Description |
|---|---|---|
| `extraObjects` | `[]` | List of additional Kubernetes objects (YAML maps or strings) rendered with the chart. Values are passed through `tpl`. Set `metadata.namespace` to the chart namespace. |

### Persistence (volume `pancakeslicer-data`)

Holds projects, profiles, printers, preferences and jobs. Mounted at `/data`.

| Value | Default | Description |
|---|---|---|
| `persistence.enabled` | `true` | Use a PVC. If `false`, an `emptyDir` is used and data is lost when the pod restarts. |
| `persistence.name` | `pancakeslicer-data` | Name of the PVC (and volume). |
| `persistence.existingClaim` | `""` | Use an existing PVC instead of creating one. |
| `persistence.storageClass` | `""` | Storage class; empty uses the cluster default. |
| `persistence.accessModes` | `[ReadWriteOnce]` | PVC access modes. |
| `persistence.size` | `10Gi` | Requested size. |

### Pod settings

| Value | Default | Description |
|---|---|---|
| `resources` | `limits.memory: 8Gi` | Container resources (mirrors `mem_limit: 8g` from the compose file). |
| `securityContext` | no privilege escalation, drop `ALL` capabilities | Container security context. |
| `podSecurityContext` | `{}` | Pod-level security context (e.g. `fsGroup` if the volume is not writable). |
| `nodeSelector` | `{}` | Node selector. |
| `tolerations` | `[]` | Tolerations. |
| `affinity` | `{}` | Affinity rules. |
