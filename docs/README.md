# docs

Markdown documentation for this homelab. Each file covers a specific setup or operational topic.

| Document                                            | Description                                                                 | ID  |
| --------------------------------------------------- | --------------------------------------------------------------------------- | --- |
| [bootstrapping-homelab](bootstrapping-homelab.md)   | Steps to bootstrap the k3s cluster and install Flux.                        | 1   |
| [k3s-storage](k3s-storage.md)                       | How to configure persistent storage on k3s.                                 | 2   |
| [gitops-deployment](gitops-deployment.md)           | How to deploy and manage apps through the GitOps workflow.                  | 3   |
| [k3s-cloudflare-tunnels](k3s-cloudflare-tunnels.md) | How to deploy cloudflare tunnels w/encrypted secrets using SOPs/AGE.        | 4   |
| [helm-monitoring-stack](helm-monitoring-stack.md)   | How to deploy kube-prometheus-stack using Helm chart.                       | 5   |
| [traefik-ingress](traefik-ingress.md)               | How to deploy ingress using Traefik w/both Helm charts & raw k8s manifests. | 6   |
| [automated-image-PR](automated-image-PR.md)         | How to deploy Renovate for auto image updates.                              | 7   |
