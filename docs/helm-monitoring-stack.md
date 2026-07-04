
# overview

install helm chart using flux through code, this method can be applied to any helm chart that we want. all you need to do is:

- add the helm repo to the cluster
- create release for specified repo
- add values
- viola

ensure helm is installed on your local/dev environment/computer see [helm docs](https://helm.sh/docs/intro/install)

---
# how to

create these in `./monitoring/controllers/base/kube-prometheus-stack/` directory.
see [flux docs](https://fluxcd.io/flux/guides/repository-structure/)
#### namespace.yaml

```
apiVersion: v1
kind: Namespace
metadata:
  name: monitoring
```

#### repository.yaml

```
apiVersion: source.toolkit.fluxcd.io/v1
kind: HelmRepository
metadata:
  name: kube-prometheus-stack
  namespace: monitoring
spec:
  interval: 24h
  url: https://prometheus-community.github.io/helm-charts
```

#### release.yaml

```
apiVersion: helm.toolkit.fluxcd.io/v2
kind: HelmRelease
metadata:
  name: kube-prometheus-stack
  namespace: monitoring
spec:
  interval: 30m
  chart:
    spec:
      chart: kube-prometheus-stack
      version: "66.2.2"

      # version: "58.x"

      sourceRef:
        kind: HelmRepository
        name: kube-prometheus-stack
        namespace: monitoring
      interval: 12h
  install:
    crds: Create
  upgrade:
    crds: CreateReplace
  driftDetection:
    mode: enabled
    ignore:
      # Ignore "validated" annotation which is not inserted during install
      - paths: ["/metadata/annotations/prometheus-operator-validated"]
        target:
          kind: PrometheusRule
  values:
    grafana:
      adminPassword: <pass>
```

#### kustomization.yaml

```
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - namespace.yaml
  - repository.yaml
  - release.yaml
```

#### adding to the cluster

in `./clusters/staging/` create `monitoring.yaml` and add the following:

```
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: monitoring-controllers
  namespace: flux-system
spec:
  interval: 1m0s
  retryInterval: 1m
  timeout: 5m
  sourceRef:
    kind: GitRepository
    name: flux-system
  path: ./monitoring/controllers/staging
  prune: true
```

in `./monitoring/controllers/staging/kube-prometheus-stack/` add:

```
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namespace: monitoring
resources:
  - ../../base/kube-prometheus-stack
```

then add the following to `monitoring/controllers/staging/kustomization.yaml`:

```
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - kube-prometheus-stack
```