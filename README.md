# Kubernetes Manifests (GitOps)

All platform components and workloads for the homelab cluster. Managed by ArgoCD.

## How It Works

ArgoCD root Application (`argocd/root.yml`) watches the `argocd/` directory recursively. Any Application resource placed there is automatically picked up and synced.

The root Application is bootstrapped by the Ansible playbook on first run. After that, all changes are driven by pushing to this repository.

## Platform Components

Each platform component is a Helm chart in `charts/<name>/` with:

- `Chart.yaml` - dependency on the upstream Helm chart
- `values.yaml` - default configuration (domainSuffix, resource limits, etc.)
- `templates/` - custom resources (certificates, ingressroutes, issuers)

ApplicationSet (`argocd/platform-set.yml`) uses a merge generator:
- git generator discovers directories under `charts/*` automatically
- list generator provides `namespace` and `skipCrds` for each component
- merge combines them by `name` key

Components:

- MetalLB - metallb-system namespace, L2 mode, IP pool 192.168.122.200-192.168.122.220
- cert-manager - cert-manager namespace, self-signed local CA chain
- Traefik - traefik-system namespace, ingress controller
- Longhorn - longhorn namespace, 2 replicas, 5% storage reservation
- kube-prometheus-stack - monitoring namespace, Prometheus 2d retention, Grafana, no alertmanager
- Prometheus Operator CRDs - monitoring namespace, separate chart to avoid ArgoCD annotation overflow

## TLS

cert-manager is configured with a self-signed local CA chain:

1. `selfsigned-issuer` - root self-signed ClusterIssuer
2. `local-ca` - CA certificate signed by the self-signed issuer
3. `local-ca-issuer` - ClusterIssuer using the local CA

All TLS certificates (Grafana, Longhorn, ArgoCD) are issued by `local-ca-issuer`.

Domain names use `domainSuffix` from chart `values.yaml` (default: `kube`).

## Adding a New Platform Component

1. Create `charts/<name>/Chart.yaml` with dependency on the upstream chart
2. Create `charts/<name>/values.yaml` with default values
3. If TLS is needed, add `templates/certificate.yml` and `templates/ingressroute.yml`
4. Add entry to list generator in `argocd/platform-set.yml`:

```yaml
- name: <name>
  namespace: <namespace>
  skipCrds: "false"
```

5. Push. ArgoCD picks it up automatically.

## Adding a New Workload

1. Create `<workload>/` directory with Deployments, Services, IngressRoutes, etc.
2. Create `argocd/<workload>.yml` with an Application resource pointing to the directory
3. Push
