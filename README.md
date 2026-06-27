# registry

Helm Chart fur Docker Distribution (`registry:2`) im Kubernetes Cluster.

## Features

- Deployment der Docker Registry (Image: `registry:2`)
- Optionales Ingress (default: deaktiviert)
- PVC fur Registry-Storage (immer aktiv)
- Optionales zweites PVC fur Image-Sources (geteilt mit Buildah)
- Optionales Buildah-StatefulSet zum Bauen von Container-Images
- Vollstandig uber `values.yaml` konfigurierbare Registry (wird als ConfigMap `/etc/docker/registry/config.yml` gemountet)

## Quick Start

```bash
helm install my-registry .
```

## Konfiguration

### Wichtige Parameter

| Parameter | Default | Beschreibung |
|-----------|---------|--------------|
| `image.tag` | `2` | Registry-Image-Tag |
| `service.type` | `ClusterIP` | Service-Typ |
| `service.port` | `5000` | Service-Port |
| `ingress.enabled` | `false` | Ingress aktivieren |
| `persistence.registry.size` | `10Gi` | Grosse des Registry-PVC |
| `persistence.sources.enabled` | `false` | Zweites PVC fur Image-Sources |
| `persistence.sources.mountPath` | `/var/lib/registry/sources` | Mount-Pfad der Sources im Registry-Container |
| `buildah.enabled` | `false` | Buildah-StatefulSet aktivieren |

## Persistence

### Registry-PVC (immer aktiv)

```yaml
persistence:
  registry:
    enabled: true
    size: 10Gi
    storageClass: ""
    accessMode: ReadWriteOnce
    existingClaim: ""
```

### Sources-PVC (optional)

Fur Image-Sources, die sowohl von der Registry als auch von Buildah verwendet werden konnen:

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

Wird `sources.enabled: true` gesetzt, mounted die Registry das Volume unter `mountPath`.
Wird `buildah.enabled: true` gesetzt, wird das PVC automatisch angelegt (auch ohne `sources.enabled`).

## Buildah

Das Buildah-StatefulSet ist ein separater Container zum Bauen von OCI-Images.
Es lauft mit `sleep infinity` und erwartet, dass man via `kubectl exec` hineingeht und buildah-Kommandos ausfuhrt.

### Aktivieren

```yaml
buildah:
  enabled: true
```

### Was passiert

1. Sources-PVC wird unter `/sources` gemountet
2. Ein eigener PVC (`volumeClaimTemplate`, 5Gi default) wird unter `/var/lib/containers` gemountet (Buildah-Storage)
3. Der Container lauft mit `privileged: true` (benotigt buildah fur Overlay-FS)
4. Images konnen aus `/sources` gebaut und via `http://<release-name>:5000` in die Registry gepusht werden

### Beispiel

```bash
# Im Buildah-Container
kubectl exec -it <pod-name> -- bash

buildah bud -t my-release-registry:5000/myimage:latest /sources/myimage
buildah push my-release-registry:5000/myimage:latest
```

Der Service-Name der Registry entspricht `{{ include "registry.fullname" . }}` (z.B. `my-release-registry`).

### Parameter

```yaml
buildah:
  image:
    repository: quay.io/buildah/stable
    tag: latest
  persistence:
    size: 5Gi       # Grosse des containers-PVC (volumeClaimTemplate)
    storageClass: ""
    accessMode: ReadWriteOnce
```

## Registry-Konfiguration

Die Registry-Konfiguration wird als YAML unter `config` in der `values.yaml` definiert und als ConfigMap bereitgestellt:

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

Alle Felder werden 1:1 in die `config.yml` der Registry ubernommen.
Siehe [Docker Registry Configuration](https://docs.docker.com/registry/configuration/) fur alle Optionen.

## Ingress

Optional, standardmassig deaktiviert:

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
# Nur Registry
helm upgrade --install my-registry .

# Registry + Buildah
helm upgrade --install my-registry . \
  --set persistence.sources.enabled=true \
  --set buildah.enabled=true

# Mit bestehenden PVCs
helm upgrade --install my-registry . \
  --set persistence.registry.existingClaim=my-registry-pvc \
  --set persistence.sources.existingClaim=my-sources-pvc \
  --set buildah.enabled=true
```
