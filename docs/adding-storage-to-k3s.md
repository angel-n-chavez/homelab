
# overview

- creating persistent storage for linkding deployment with a PVC via Flux Gitops.
- observe the security risk when running containers as `root` user, and mitigating that risk.
- explore k9s for cluster mgmt.

---
# local testing for when using ssh-machine/jumpbox

I ssh into a ssh-server runnng Debian named "Larry". From Larry i run `kubectl, k9s, kubens, etc` none of that is installed on my main fedora machine. so when i portforward linkding on larry it is only reachable at 127.0.0.1 on Larry's loopback interface, my browser from fedora can not reach it. this hinders the testing process significantly.

#### 1st solution

curl directly from Larry's cli:
```bash
curl http://localhost:8080
curl -I http://localhost:8080
```

this works but is limited, and lame.

#### better solution

use ssh port-forwarding.

on main machine(fedora):
```bash
ssh -N -L 8080:localhost:PORT <ssh-server>
```

this creates a ssh tunnel, which tunnels my laptops local traffic directly to Larry's loopback int. allowing me to test linkding on my browser

#### future solution

work exclusively from a dev-container, with tailored networking configs for local testing, and of course all the tools, storage-mounts and dependecies needed in my daily workflow i.e. kubectl, kubens, PVCs, k9s, etc.

working on it.

---
# persistence with PVC

#### 1st
if you describe the linkding pod you will see only 1 mount, the default one. and you will see that it is running as root.

so first things first - create a PVC named `linkding-data-pvc` in a file called storage.yaml:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: linkding-data-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
```

then, add a volume/volumeMount to the linkding container in `deployment.yaml`; referencing my newly created PVC.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: linkding
spec:
  replicas: 1
  selector:
    matchLabels:
      app: linkding
  template:
    metadata:
      labels:
        app: linkding
    spec:
      containers:
        - name: linkding
          image: sissbruecker/linkding:1.31.0
          ports:
            - containerPort: 9090

          volumeMounts:
            - name: linkding-data
              mountPath: /etc/linkding/data
      volumes:
        - name: linkding-data
          persistentVolumeClaim:
            claimName: linkding-data-pvc
```

port-forward again:
```bash
k port-forward linkding 8080:9090
```

#### 2nd

we need to create a linkding user using the cmd: 
```bash
k exec -it linkding-pod-name -- python manage.py createsuperuser --username=user --email=user@email.com
```

now no matter how many times you delete the pod or even the deployment, the newly created pods will attach to the PVC storage and my linkding data persiste e.g. my users/bookmarks

---

# observing security attack-vectors

if we shell into our linkding pod/container and look around we will find a couple of things.

1. all the neccesary files/dependencies to run linkding
2. our `/etc/data` mount
3. we are `root` which is not great - we will address this in the next section

if a threat-actor gained access to this container, depending on the sophistication of their skills, they can do anything within the container. they can escape to the node(s) itself, and they can map out or even gain access to the network. this is a big no no.

---

# securing linkding

before we expose this app to the internet we must secure this application (and all future ones for that matter). 

changes to deployment:

- adding security context
- specifying a FS group for volume permissions
- setting run as user and run as group
- disable privilege escalation

it will no longer be able to perform package updates, escalate to root, and will be limited to preinstalled functionality.

- we determine the appropriate user by looking at the `/etc/passwd` file INSIDE the container. shell into the pod using `k9s`
- www-data-user (ID 33) is the ID of the intended user for the application

add securityContext to `deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: linkding
spec:
  replicas: 1
  selector:
    matchLabels:
      app: linkding
  template:
    metadata:
      labels:
        app: linkding
    spec:
      **securityContext**:
        fsGroup: 33 # www-data group ID
        runAsUser: 33 # www=data user ID
        runAsGroup: 33

      containers:
        - name: linkding
          image: sissbruecker/linkding:1.31.0
          ports:
            - containerPort: 9090
            
          **securityContext**:
            allowPrivilegeEscalation: false

          volumeMounts:
            - name: linkding-data
              mountPath: /etc/linkding/data
      volumes:
        - name: linkding-data
          persistentVolumeClaim:
            claimName: linkding-data-pvc

```

now this pod/container is running as www-data user which has no user login or access to a shell and privilege escalation is turned off. this is crucial before exposing to the internet which i will doing with "Cloudflare Tunnels" in future docs/guides.