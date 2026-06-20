# k8s-homelab

A self-hosted homelab managed entirely from Git. A Talos Kubernetes cluster is
reconciled by Flux, and a set of Docker hosts is reconciled by Ansible. Pushing to
`main` is the only deploy step.

## Architecture

Two control planes, one repo:

- **Kubernetes** runs on a 3-node Talos Linux cluster. Flux watches this repo and
  applies the desired state on an interval, with pruning, so the cluster always matches
  Git.
- **Docker** runs on a Synology NAS and an Ubuntu VM. A self-hosted GitHub Actions
  runner triggers Ansible whenever anything under `ansible/` changes, which renders and
  brings up the Compose stacks on each host.

```mermaid
flowchart LR
    push[git push to main] --> repo[(this repo)]
    repo -->|Flux, reconciled hourly| k8s[Talos Kubernetes]
    repo -->|GitHub Actions + Ansible| docker[Docker hosts]
```

## How it works

### Kubernetes (Flux)

Flux is bootstrapped from `clusters/homelab-prod/flux-system`, which defines the Git
source. Two Flux Kustomizations reconcile the rest of the repo every hour with
`prune: true`:

| Kustomization         | Path              | Contents                                    |
| --------------------- | ----------------- | ------------------------------------------- |
| `infrastructure-sync` | `./infrastructure`| Helm repositories, generic-device-plugin    |
| `apps-sync`           | `./apps`          | All application workloads                   |

Applications are deployed as Flux `HelmRelease` resources pinned to versioned Helm
repositories, so every upgrade is an explicit, reviewable change in Git.

### Docker (Ansible)

`.github/workflows/ansible-deploy.yaml` runs on a self-hosted runner and executes
`ansible/deploy-homelab.yml`. The playbook copies each stack's `docker-compose.yaml`
to the target host and runs `docker compose up`, idempotently, for two host groups
defined in `ansible/inventory.ini`.

## What's running

### On Kubernetes

| Component              | Role                                                          |
| ---------------------- | ------------------------------------------------------------- |
| Traefik                | Ingress controller and reverse proxy                          |
| cert-manager           | Automated TLS, issued through the Cloudflare DNS-01 challenge |
| Authentik              | SSO and identity, enforced in front of services via Traefik   |
| Grafana                | Dashboards                                                     |
| Jellyfin               | Media server, GPU transcoding via the device plugin below     |
| Sonarr / Prowlarr      | Media automation                                              |
| generic-device-plugin  | Exposes `/dev/dri` to the cluster for hardware transcoding     |

### On Docker

| Host             | Stacks                                            |
| ---------------- | ------------------------------------------------- |
| Ubuntu VM        | BIND (DNS), Nextcloud, Nginx Proxy Manager, Traefik |
| Synology NAS     | MariaDB, qBittorrent, Uptime Kuma                  |

## Secrets

Secrets are committed to Git as **Sealed Secrets**. The values are encrypted with the
cluster controller's public key and can only be decrypted in-cluster, so the repo is
safe to keep public. No plaintext credentials live in the tree or its history.

## Storage

Persistent volumes for the cluster are provisioned over iSCSI by the Synology CSI
driver (`synology-nas/syno-sc.yaml`).

## Updates

Renovate runs continuously and opens pull requests to bump Helm charts, container
images, and GitHub Actions across the repo, so dependency upgrades stay small and
reviewable.

## Layout

```
.
├── clusters/homelab-prod/   # Flux bootstrap (Git source + sync)
├── infrastructure/          # cluster-wide infra, reconciled first
├── apps/                    # Helm-based application workloads
├── synology-nas/            # cluster storage class (Synology CSI)
├── ansible/                 # Docker Compose stacks + playbook
│   ├── docker/              # per-host, per-stack compose files
│   ├── inventory.ini
│   └── deploy-homelab.yml
├── flux-apps.yaml           # Flux Kustomization: apps
├── infrastructure-sync.yaml # Flux Kustomization: infrastructure
└── renovate.json
```

## Roadmap

- Full observability: Prometheus for metrics and Loki for logs, feeding the existing
  Grafana, with Alertmanager for alerting.
- Backup and disaster recovery for persistent volumes and application data.
