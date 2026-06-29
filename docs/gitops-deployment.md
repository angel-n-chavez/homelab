
# deploying first app via gitops/flux

my first app im deploying is one that ive already done with docker compose; Linkding

the structure for this GitOps repo is a mono repo which i got straight from the FluxCD docs.

from: [Flux Docs](https://fluxcd.io/flux/guides/repository-structure/)
```console
├── apps
│   ├── base
│   ├── production 
│   └── staging
├── infrastructure
│   ├── base
│   ├── production 
│   └── staging
└── clusters
    ├── production
    └── staging
```

Flux is always going to look for a `kustomization.yaml` so the flow kinda goes like this, from my understanding.

it starts with `/clusters/staging/flux-system/kustomization.yaml` then goes to --> `/apps/staging/linkding/kustomization.yaml` which then goes to --> `/apps/base/linkding/kustomization.yaml`

I know thats alot of `kustomization.yaml` but im about to break it down

- **`/clusters/staging/flux-system/kustomization.yaml`** - is where it all begins. when you bootstrap fluxCD on a cluster it creates this one for you. inside the file the main thing to look for `resources: ` this objects points to custom resource definitions. this is true for all `kustomization.yaml`

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - gotk-components.yaml
  - gotk-sync.yaml
```

-  **`/apps/staging/linkding/kustomization.yaml`** - is referenced in `gotk-sync.yaml` and points to ../../base/linkding

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namespace: linkding
resources:
  - ../../base/linkding
```

- **`/apps/base/linkding/kustomization.yaml`** - references the actual deployment, namespace, and other resources needed to run the app.

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - deployment.yaml
  - namespace.yaml
```

- **`clusters/staging/apps.yaml`** - tells Flux to look at the apps/staging directory

```yaml
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: apps
  namespace: flux-system
spec:
  interval: 1m0s
  retryInterval: 1m
  timeout: 5m
  sourceRef:
    kind: GitRepository
    name: flux-system
  path: ./apps/staging
  prune: true

```

# verify/check work

```bash
flux get kustomizations
```

when i ran this cmd i got an error:
```bash
kustomize build failed: accumulating resources: accumulation err="'accumulating resources from './linkding': read /tmp/kustomization-750200480/apps/staging/linkding: is a directory': couldn't make target for path '/tmp/kustomization-750200480/apps/staging/linkding': Failed to read kustomization file under /tmp/kustomization-750200480/apps/staging/linkding:
                                                                apiVersion for Kustomization should be kustomize.config.k8s.io/v1beta1"
```

it looks like alot but it actually tells me what i did wrong in this case. it could not read the `kustomizaion.yaml` in /app/staging/linkding because i used the wrong `apiVersion: `.

Did you catch it? **`apiVersion for Kustomization should be "kustomize.config.k8s.io/v1beta1`**" currently it is "`kustomize.toolkit.fluxcd.io/v1`"

after the fix the command worked and should output something similar:
```bash
> flux get kustomizations 
NAME            REVISION                SUSPENDED       READY   MESSAGE                              
apps            main@sha1:0d49ae59      False           True    Applied revision: main@sha1:0d49ae59
flux-system     main@sha1:0d49ae59      False           True    Applied revision: main@sha1:0d49ae59
```

run `kubectl get ns` to see if the linkding namespace was created

```bash
k get ns
```

port forward the linkding pod to see if you can connect:
```bash
kubectl port-forward pod/linkding 8080:9090
```

this specifically did not work for me; because i ssh into a jump-box called "Larry", from my main machine. When i run the cmd above on Larry, the port `8080` is only listening on `larry`'s local loopback interface (`127.0.0.1`). My Fedora laptop(main) cannot see it yet.

its okay though i will fix that with a loadbalancer service and ingress w/Traefik in the future.

Success! I deployed a application without `kubectl -f apply`, purely with GitOps using FluxCD.