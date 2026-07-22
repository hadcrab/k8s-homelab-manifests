# Kubernetes Manifests (GitOps)

Pure YAML manifests for ArgoCD. All application-level components and workloads are defined here, not in the Ansible playbook.

## How It Works

ArgoCD root Application (`argocd/root.yml`) watches the `argocd/` directory recursively. Any Application resource placed there is automatically picked up and synced.

The root Application is bootstrapped by the Ansible playbook on first run. After that, all changes are driven by pushing to this repository.

## Platform Components

MetalLB - metallb-system namespace, L2 mode, IP pool 192.168.122.200-192.168.122.220.

cert-manager - cert-manager namespace, self-signed local CA chain.

Traefik - traefik-system namespace, ingress controller.

Longhorn - longhorn namespace, 2 replicas, 5% storage reservation.

kube-prometheus-stack - monitoring namespace, Prometheus 2d retention, Grafana, no alertmanager.

Prometheus Operator CRDs - monitoring namespace, separate Application to avoid ArgoCD annotation overflow.

## TLS

cert-manager is configured with a self-signed local CA chain:

1. `selfsigned-issuer` - root self-signed ClusterIssuer
2. `local-ca` - CA certificate signed by the self-signed issuer
3. `local-ca-issuer` - ClusterIssuer using the local CA

All TLS certificates (Grafana, Longhorn, ArgoCD) are issued by `local-ca-issuer`.

## Adding a New Platform Component

1. Create `platform/<name>/` directory with raw Kubernetes resources
2. Create `argocd/platform/<name>.yml` with an Application resource pointing to the directory
3. If the component has a Helm chart, use a multi-source Application with the chart as one source and your custom manifests as the other

## Adding a New Workload

1. Create `<workload>/` directory with Deployments, Services, IngressRoutes, etc.
2. Create `argocd/<workload>.yml` with an Application resource pointing to the directory
