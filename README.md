# homelab

A personal homelab GitOps monorepo that mirrors real-world production environments. Application and infrastructure state is defined as code, with Flux CD continuously reconciling the cluster to match what is committed here.

## Structure
```
.
├── apps/
│   ├── base/
│   │   ├── audiobookshelf/
│   │   ├── kustomization.yaml
│   │   └── linkding/
│   └── staging/
│       ├── audiobookshelf/
│       ├── kustomization.yaml
│       └── linkding/
├── clusters/
│   └── staging/
│       ├── apps.yaml
│       ├── flux-system/
│       ├── infrastructure.yaml
│       └── monitoring.yaml
├── docs/
├── infrastructure/
│   └── controllers/
│       ├── base/
│       │   ├── kustomization.yaml
│       │   └── renovate/
│       └── staging/
│           ├── kustomization.yaml
│           └── renovate/
├── monitoring/
│   ├── configs/
│   │   └── staging/
│   │       ├── kube-prometheus-stack/
│   │       └── kustomization.yaml
│   └── controllers/
│       ├── base/
│       │   └── kube-prometheus-stack/
│       └── staging/
│           ├── kube-prometheus-stack/
│           └── kustomization.yaml
├── README.md
├── renovate.json
```

### `apps/`

Holds all application manifests. `base/` contains the canonical Kubernetes resources for each app. `staging/` contains Kustomize overlays that extend or patch the base for the staging environment.

| App                                                  | Description                   |
| ---------------------------------------------------- | ----------------------------- |
| [linkding](https://github.com/sissbruecker/linkding) | Self-hosted bookmark manager  |
| [audiobookshelf](https://audiobookshelf.org/)        | Self-hosted audiobook library |

### `clusters/`

The Flux bootstrap entry point for each cluster. Flux is pointed at `clusters/staging/` and uses it to discover and reconcile everything else in the repo. The `flux-system/` subfolder contains the Flux controller manifests generated during bootstrap and is managed by Flux itself.

| Environment   | Description                                                                                                   |
| ------------- | ------------------------------------------------------------------------------------------------------------- |
| `staging/`    | staging cluster; used to test new software, updates, and configs before releasing to users. (dress-rehearsal) |
| `production/` | prod cluster; is the live env used my actual users.                                                           |

### `docs/`

Markdown documentation covering setup and operational procedures. See [`docs/README.md`](docs/README.md).

### `infrastructure/`

Holds all infra manifests. Follows same organization as `apps/` and `monitoring/`

| App                                       | Description             |
| ----------------------------------------- | ----------------------- |
| [Renovate](https://docs.renovatebot.com/) | Automated image updates |

### `monitoring/`

Holds all monitoring manifests. Follows same org as `apps/` and `infrastructure/`

| App                                                                                                                 | Description                                          |
| ------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------- |
| [kube-prometheus-stack](https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack) | Monitoring stack (Prometheus, Grafana, AlertManager) |
## How It Works

Changes to manifests in `apps/`, `infrastructure/`, `monitoring/` are picked up,
automatically by Flux on its reconciliation interval. Committing to `main` is the 
mechanism for deploying to the cluster, there is no manual `kubectl apply` step.

The reconciliation order is defined in `clusters/staging/`: `flux-system` installs the 
Flux controllers, and `apps.yaml` creates the Kustomization that watches 
`apps/staging/`.