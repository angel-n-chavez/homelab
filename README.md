# homelab

A personal homelab GitOps monorepo that mirrors real-world production environments. Application and infrastructure state is defined as code, with Flux CD continuously reconciling the cluster to match what is committed here.

## Structure

```
.
├── apps
│   ├── base
│   │   └── linkding
│   │       ├── deployment.yaml
│   │       ├── kustomization.yaml
│   │       ├── namespace.yaml
│   │       └── storage.yaml
│   └── staging
│       └── linkding
│           └── kustomization.yaml
├── clusters
│   └── staging
│       ├── apps.yaml
│       └── flux-system
│           ├── gotk-components.yaml
│           ├── gotk-sync.yaml
│           └── kustomization.yaml
└── docs
    ├── adding-storage-to-k3s.md
    ├── bootstrapping-homelab.md
    └── gitops-deployment.md
```

### `apps/`

Holds all application manifests. `base/` contains the canonical Kubernetes resources for each app. `staging/` contains Kustomize overlays that extend or patch the base for the staging environment.

**Apps:**

| App | Description |
|---|---|
| [linkding](https://github.com/sissbruecker/linkding) | Self-hosted bookmark manager |

### `clusters/`

The Flux bootstrap entry point for each cluster. Flux is pointed at `clusters/staging/` and uses it to discover and reconcile everything else in the repo. The `flux-system/` subfolder contains the Flux controller manifests generated during bootstrap and is managed by Flux itself.

### `docs/`

Markdown documentation covering setup and operational procedures. See [`docs/README.md`](docs/README.md).

## How It Works

Changes to manifests in `apps/` are picked up automatically by Flux on its reconciliation interval. Committing to `main` is the mechanism for deploying to the cluster, there is no manual `kubectl apply` step.

The reconciliation order is defined in `clusters/staging/`: `flux-system` installs the Flux controllers, and `apps.yaml` creates the Kustomization that watches `apps/staging/`.
