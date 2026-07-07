
# overview

using Renovate for automated image updates. as soon as a app image has an update from developers, renovate-bot creates a pull request which i can then review.

the basic idea: wether i am away at work or on vacay or something, Renovate will
notify me with short description and when an image is updated, which i can review 
from github e.g. from my phone. If approved the changes are applied on my cluster.

---
# install

of course this is applied through gitops, as it should be.

### 1st.

create a Github PAT for renovate because it needs access to your repo. then we will 
create k8s secret with the PAT passed to it, which is outputted to file.

```bash
kubectl create secret generic renovate-container-env \
--from-literal=RENOVATE_TOKEN=<secret> \
--dry-run=client \
-o yaml > renovate-container-env.yaml
```

then encrypt:

```bash
sops --age=$AGE_PUBLIC \
--encrypt --encrypted-regex '^(data|stringData)$' --in-place renovate-container-env.yaml
```

### 2nd.

configure renovate. I used 4 files:

#### namespace.yaml
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: renovate
```

#### cronjob.yaml
```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: renovate
  namespace: renovate
spec:
  schedule: "@hourly"
  concurrencyPolicy: Forbid
  jobTemplate:
    spec:
      template:
        spec:
          containers:
            - name: renovate
              image: renovate/renovate:latest
              args:
                - angel-n-chavez/homelab

              envFrom:
                - secretRef:
                    name: renovate-container-env
                - configMapRef:
                    name: renovate-configmap

          restartPolicy: Never
```

#### configmap.yaml

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: renovate-configmap
  namespace: renovate
data:
  RENOVATE_AUTODISCOVER: "false"
  RENOVATE_GIT_AUTHOR: "Renovate Bot <bot@renovateapp.com>"
  RENOVATE_PLATFORM: "github"
```

and of course a kustomization file because thats how I manage this gitops repo:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - renovate-container-env.yaml
  - namespace.yaml
  - configmap.yaml
  - cronjob.yaml
```

### 3rd.

add `renovate.json` file to the root of git repo:

```json
{
  "$schema": "https://docs.renovatebot.com/renovate-schema.json",
  "kubernetes": {
    "fileMatch": [
      "\\.yaml$"
    ]
  }
}
```

### 4th. & final

add `infrastructure` to the Flux-System. In
`/clusters/staging/infrastructure.yaml` add:

```yaml
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: infrastructure-controllers
  namespace: flux-system
spec:
  interval: 1m0s
  retryInterval: 1m
  timeout: 5m
  sourceRef:
    kind: GitRepository
    name: flux-system
  path: ./infrastructure/controllers/staging
  prune: true
  decryption:
    provider: sops
    secretRef:
      name: sops-age
```

commit and push.

---

# renovate in action

this is dope; I can now update(or not) my applications/infra from github by 
approving a PR and merging a branch all of which Renovate created. Also the review
that Renovate generates is very in depth and links to the official release. this saves
me so much overhead-headache!

The best part is if something breaks for whatever
reason; then i can just revert back to the previous commit and its as if it never
happened!!!'

this is dope.