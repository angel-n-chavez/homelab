
namespace.yaml

```
apiVersion: v1
kind: Namespace
metadata:
  name: monitoring
```

repository.yaml

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

release.yaml

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
      adminPassword: mischa
```

kustomization.yaml

```
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - namespace.yaml
  - repository.yaml
  - release.yaml
```