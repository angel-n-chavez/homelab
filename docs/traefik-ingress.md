
# overview

create an ingress using traefik which ships w/k3s by default. this will allow me to
create ingress routes and enable local domain routing. traefik also enables 
automated service discovery.

for now I will disable my cloudflare tunnel by commenting out cloudflared under 
`resources`
in `/apps/staging/linkding/kustomization.yaml

---
# how to

start with creating `ingress.yaml` in `./apps/staging/linkding`

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: linkding
  namespace: linkding
spec:
  ingressClassName: traefik
  rules:
    - host: ld.angelchavez.net
      http:
        paths:
          - backend:
              service:
                name: linkding
                port:
                  number: 9090
            path: /
            pathType: Prefix
```

then optionally you can add an entry to you `/etc/hosts` file but I just added a record 
to my cloudflare dns and it worked rather managing a bunch of hosts files.

when doing this i ran into some problems with my att router blocking local loops
this will need to be addressed in the future.
So i ended up just adding an entry into `/etc/hosts` afterall and this will be my
solution for now until I either get a new router or I can use "xip.io" or "sslip.io" in my
`ingress.yaml`. These free magic loopback domain/DNS servers bypass the routers
hairpin restrictions. see below:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: linkding
  namespace: linkding
spec:
  ingressClassName: traefik
  rules:
    - host: ld.angelchavez.net.sslip.io
      http:
        paths:
          - backend:
              service:
                name: linkding
                port:
                  number: 9090
            path: /
            pathType: Prefix
```

---
# helm example

using the "kube-prometheus-stack" helm chart to create a ingress on the cluster vs 
pure k8s-manifest, as well as create a self signed TLS cert. The self signed cert will
be served on the ingress as well.

#### 1st

add helm repo and update

```bash
helm repo add prometheus-community https://prometheus-
community.github.io/helm-charts

helm repo update
```

#### 2nd

show the default values of the kube-prometheus-stack chart I just added and pipe
the output to a file to browse/search w/vim:

```bash
helm show values prometheus-community/kube-prometheus-stack  > values.yaml
```

inside of `monitoring/controllers/base/kube-prometheus-stack/release.yaml` under values, on the same level as `adminPassword:` add `ingress:`

```yaml
  values:
    grafana:
      adminPassword: <pass>
      ingress:
        enabled: true
        ingressClassName: traefik

        hosts:
          - gfs.angelchavez.net

        tls:
          - secretName: grafana-tls-secret
            hosts:
              - gfs.angelchavez.net
```

#### 3rd

generate tls cert and tls cert k8s-secret, then use sops w/your age key to encrypt the k8s-secret:

```bash
# Generate the private key and certificate
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout ./tls.key \
  -out ./tls.crt \
  -subj "/C=US/ST=Texas/L=San Antonio/O=Home Lab Heroes Inc./OU=Department of Monitoring/CN=gfs.angelchavez.net" \
  -addext "subjectAltName=DNS:gfs.angelchavez.net"

# Create TLS secret
kubectl create secret tls grafana-tls-secret \
  --cert=tls.crt \
  --key=tls.key \
  --namespace=monitoring \
  --dry-run=client \
  -o yaml > grafana-tls-secret.yaml

sops --age=$AGE_PUBLIC \
--encrypt --encrypted-regex '^(data|stringData)$' --in-place grafana-tls-secret.yaml
```

#### 4th

add the secret to flux in `monitoring/configs/staging/kube-prometheus-stack/` 
directory, and add kustomization file pointing to `grafana-tls-secret.yaml`

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - grafana-tls-secret.yaml
```

#### 5th

make sure Flux knows about this directory. In `clusters/staging/monitoring.yaml`
add the second block for monitoring configs:

```yaml
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

---

apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: monitoring-configs
  namespace: flux-system
spec:
  interval: 1m0s
  retryInterval: 1m
  timeout: 5m
  sourceRef:
    kind: GitRepository
    name: flux-system
  path: ./monitoring/configs/staging
  prune: true
  decryption:
    provider: sops
    secretRef:
      name: sops-age
```

#### 6th

update `monitoring/controllers/base/kube-prometheus-stack/release.yaml` to
configure the ingress to use this secret

```yaml
tls:
  - secretName: grafana-tls-secret
    hosts:
      - gfs.angelchavez.net

```