# GitLab Runner

GitLab Runner is used to execute the CI/CD pipeline for the Online Boutique project.

## Setup

Runner is deployed with Docker Compose using the Docker executor:

```yaml
services:
  gitlab-runner:
    image: gitlab/gitlab-runner:alpine
    container_name: gitlab-runner
    restart: unless-stopped
    extra_hosts:
      - "gitlab.local:host-gateway"
    volumes:
      - ./config:/etc/gitlab-runner
      - /var/run/docker.sock:/var/run/docker.sock
      - ./certs:/etc/gitlab-runner/certs:ro
```

Runner was registered against the local GitLab instance:

```bash
docker exec gitlab-runner gitlab-runner register \
  --non-interactive \
  --url "https://gitlab.local" \
  --token "<RUNNER_TOKEN>" \
  --executor "docker" \
  --docker-image "docker:29.8.0"
```

## Runner Configuration

Important `config.toml` settings:

```toml
[[runners]]
  name = "online-boutique-runner"
  url = "https://gitlab.local"
  executor = "docker"

  [runners.docker]
    image = "docker:29.8.0"
    volumes = ["/cache"]
    network_mode = "host"
    extra_hosts = ["gitlab.local:host-gateway"]
```

### Why these settings?

* `docker:29.8.0` — default image for CI jobs.
* Docker socket — allows CI jobs to build and push images using the host Docker daemon.
* `extra_hosts` — allows CI job containers to resolve `gitlab.local`.
* `network_mode = "host"` — allows CI jobs to reach the Kubernetes API on the libvirt network (`192.168.122.0/24`).

## HTTPS

GitLab uses a local `mkcert` certificate.

The CA is mounted into the Runner:

```text
./certs:/etc/gitlab-runner/certs:ro
```

This allows the Runner to trust:

```text
https://gitlab.local
```

## Kubernetes Access

A dedicated ServiceAccount is used for CI deployments:

```text
ServiceAccount: gitlab-ci
Namespace: online-boutique
```

It is bound to a namespace-scoped Role with permissions for:

```text
deployments: get, list, watch, patch, update
pods:        get, list, watch
```

Authentication uses a short-lived TokenRequest token:

```bash
kubectl create token gitlab-ci \
  -n online-boutique \
  --duration=24h
```

The following values are stored as GitLab CI/CD variables:

```text
KUBE_SERVER
KUBE_CA
KUBE_TOKEN
```

No Kubernetes admin credentials are exposed to the pipeline.

## CI/CD Flow

```text
Test
  ↓
Build
  ↓
Push → GitLab Container Registry
  ↓
Trivy Scan
  ↓
Deploy → Kubernetes
```

Images are tagged using the Git commit SHA:

```text
$CI_REGISTRY_IMAGE/frontend:$CI_COMMIT_SHORT_SHA
```

## Verification

The Runner successfully authenticated to Kubernetes and verified its permissions:

```text
kubectl auth can-i get deployments
yes

kubectl auth can-i patch deployments
yes

kubectl auth can-i get pods
yes
```

The CI job can also query the `online-boutique` namespace:

```bash
kubectl get deployments -n online-boutique
```
