# registry

A Helm chart for deploying [Docker Distribution](https://hub.docker.com/_/registry) (`registry:2`) on Kubernetes.

## Features

- Deployment of the Docker Registry (image: `registry:2`)
- Optional Ingress (disabled by default)
- PVC for registry storage (always enabled)
- Optional second PVC for image sources (shared with Buildah)
- Optional Buildah StatefulSet for building container images
- Fully configurable registry via `values.yaml` (mounted as ConfigMap at `/etc/docker/registry/config.yml`)

## Quick Start

```bash
helm install my-registry .
```

## Configuration

### Important Parameters

| Parameter | Default | Description |
|-----------|---------|-------------|
| `image.tag` | `2` | Registry image tag |
| `service.type` | `ClusterIP` | Service type |
| `service.port` | `5000` | Service port |
| `ingress.enabled` | `false` | Enable ingress |
| `persistence.registry.size` | `10Gi` | Registry PVC size |
| `persistence.sources.enabled` | `false` | Enable second PVC for image sources |
| `persistence.sources.mountPath` | `/var/lib/registry/sources` | Mount path inside the registry container |
| `buildah.enabled` | `false` | Enable Buildah StatefulSet |

## Persistence

### Registry PVC (always enabled)

```yaml
persistence:
  registry:
    enabled: true
    size: 10Gi
    storageClass: ""
    accessMode: ReadWriteOnce
    existingClaim: ""
```

### Sources PVC (optional)

For image sources shared between the registry and Buildah:

```yaml
persistence:
  sources:
    enabled: true
    size: 10Gi
    storageClass: ""
    accessMode: ReadWriteOnce
    existingClaim: ""
    mountPath: /var/lib/registry/sources
```

When `sources.enabled: true` is set, the registry mounts the volume at `mountPath`.
When `buildah.enabled: true` is set, the PVC is created automatically (even without `sources.enabled`).

## Buildah

The Buildah StatefulSet is a separate container for building OCI images.
It runs with `sleep infinity` and expects you to exec in and run buildah commands.

### Enabling

```yaml
buildah:
  enabled: true
```

### What Happens

1. The sources PVC is mounted at `/sources`
2. A dedicated PVC (`volumeClaimTemplate`, 5Gi default) is mounted at `/var/lib/containers` (Buildah storage)
3. The container runs with `privileged: true` (required by buildah for overlay filesystem)
4. Images can be built from `/sources` and pushed to the registry via `http://<release-name>:5000`

## Workflow: Building and Using a Custom Image

This section walks through the full process of building an image with Buildah,
pushing it to the in-cluster registry, and running a pod with that image.

### Prerequisites

- Helm release installed with `buildah.enabled=true` and `persistence.sources.enabled=true`
- The Buildah pod `registry-registry-buildah-0` is running

### Step 1: Create a Containerfile and Build the Image

```bash
kubectl exec -n registry registry-registry-buildah-0 -- sh -c '
mkdir -p /sources/my-alpine
cat > /sources/my-alpine/Containerfile << EOF
FROM docker.io/alpine:latest
RUN apk add --no-cache curl ca-certificates
CMD ["sleep", "infinity"]
EOF
buildah bud -t registry-registry:5000/my-alpine:latest /sources/my-alpine
'
```

This creates a Containerfile, places it under `/sources`, and builds the image
tagged for the local registry.

### Step 2: Push the Image to the Registry

Since the registry runs without TLS, pass `--tls-verify=false`:

```bash
kubectl exec -n registry registry-registry-buildah-0 -- \
  buildah push --tls-verify=false registry-registry:5000/my-alpine:latest
```

### Step 3: Verify the Image is in the Registry

```bash
kubectl exec -n registry registry-registry-buildah-0 -- sh -c '
curl -s http://registry-registry:5000/v2/_catalog
'
```

Expected output: `{"repositories":["my-alpine"]}`

### Step 4: Configure the Container Runtime for the Insecure Registry

> **Note:** This is a one-time cluster-wide setup. The registry runs without TLS
> (plain HTTP), so the container runtime must be configured to allow insecure
> connections to it.

#### For CRI-O

Add the registry to `/etc/containers/registries.conf` on each node:

```ini
[[registry]]
location = "registry-registry.registry.svc.cluster.local:5000"
insecure = true
```

Then add a host entry and restart CRI-O:

```bash
echo "<cluster-ip> registry-registry.registry.svc.cluster.local" >> /etc/hosts
systemctl restart crio
```

#### For containerd

Add the registry to `/etc/containerd/config.toml` under `[plugins."io.containerd.grpc.v1.cri".registry.configs]`:

```toml
[plugins."io.containerd.grpc.v1.cri".registry.configs]
  [plugins."io.containerd.grpc.v1.cri".registry.configs."registry-registry.registry.svc.cluster.local:5000"]
    [plugins."io.containerd.grpc.v1.cri".registry.configs."registry-registry.registry.svc.cluster.local:5000".tls]
      insecure_skip_verify = true
```

### Step 5: Create a Pod Using the Image

```bash
kubectl run my-alpine-test \
  --image=registry-registry.registry.svc.cluster.local:5000/my-alpine:latest \
  --restart=Never \
  -- sleep infinity
```

Verify the pod is running:

```bash
kubectl get pod my-alpine-test
kubectl describe pod my-alpine-test | grep -i image
```

Expected output shows the image pulled from your in-cluster registry.

### Step 6: Clean Up

```bash
kubectl delete pod my-alpine-test
```

### Example

```bash
# Exec into the Buildah container
kubectl exec -n registry -it registry-registry-buildah-0 -- bash

buildah bud -t my-release-registry:5000/myimage:latest /sources/myimage
buildah push --tls-verify=false my-release-registry:5000/myimage:latest
```

The registry service name follows the pattern `{{ include "registry.fullname" . }}`
(e.g., `my-release-registry`).

### Parameters

```yaml
buildah:
  image:
    repository: quay.io/buildah/stable
    tag: latest
  persistence:
    size: 5Gi       # Size of the containers PVC (volumeClaimTemplate)
    storageClass: ""
    accessMode: ReadWriteOnce
```

## Registry Configuration

The registry configuration is defined as YAML under `config` in `values.yaml` and
served as a ConfigMap:

```yaml
config:
  http:
    addr: :5000
    debug:
      addr: :5001
  storage:
    filesystem:
      rootdirectory: /var/lib/registry
    delete:
      enabled: true
```

All fields are passed verbatim into the registry's `config.yml`.
See the [Docker Registry Configuration](https://docs.docker.com/registry/configuration/)
for all available options.

## Ingress

Optional, disabled by default:

```yaml
ingress:
  enabled: true
  className: nginx
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt
  hosts:
    - host: registry.example.com
      paths:
        - path: /
          pathType: Prefix
  tls:
    - hosts:
        - registry.example.com
      secretName: registry-tls
```

## Deployment

```bash
# Registry only
helm upgrade --install my-registry .

# Registry + Buildah
helm upgrade --install my-registry . \
  --set persistence.sources.enabled=true \
  --set buildah.enabled=true

# With existing PVCs
helm upgrade --install my-registry . \
  --set persistence.registry.existingClaim=my-registry-pvc \
  --set persistence.sources.existingClaim=my-sources-pvc \
  --set buildah.enabled=true
```
