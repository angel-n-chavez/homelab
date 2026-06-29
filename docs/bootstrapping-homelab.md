
# pre-reqs

- debian 13 Trixie lite iso
- ssh keys
- kubectl installed on local dev/work machine
- github personal access token

---
# provision VM

when provisioning VM in Proxmox hypervisor follow these steps:
1. check "start at boot"
2. select `virtIO SCSI single` for SCSI Controller + check Qemu Agent
3. under Disks check `SSD emulation` and `Discard`
4. under CPU change the type from `default` to `host` 
5. under Memory uncheck `Ballooning Device`  
6. under Network enable `Multiqueue` by setting the value to the same amount as your vCPU count. (e.g. 2 if you assigned 2 vCPUs)

once completed, boot up and set up your server, hostname, user/passwd, etc. Debian does not come with sudo, curl and other packages you may need; so make sure you install them.

**Note: always update packages on a new system e.g. 

```bash
sudo apt update -y && sudo apt upgrade -y
```

then set up ssh or however you are going to access this VM(in my case im using SSH)

Lastly install qemu guest agent and enable it; then you need to disable swap. 
See the cmds below:

```bash
sudo apt install qemu-guest-agent -y
sudo systemctl enable --now qemu-guest-agent
sudo swapoff -a
```
 to make the swap disable permanent, edit `/etc/fstab` and comment out any lines containing `swap`

**enabling cgroup memory in the bootloader is no longer need in Debian 13 Trixie**

---
# install k3s

```bash
# install k3s as root with helm controller disabled
sudo su -
curl -sfL https://get.k3s.io | INSTALL_K3S_EXEC="--disable=helm-controller" sh
systemctl status k3s.service
```

### kube config so were not sshing into our server to run kubectl
```bash
# cp the kube config file to your user dir on the server NOT root
cd
sudo cp /etc/rancher/k3s/k3s.yaml .
```
you may need to change the ownership of the file as it will still be owned by root, so we can scp the file to our main system.

### copy kube config to main machine 
```bash
# on main system
scp user@<ip-of-server>:/home/user/k3s.yaml .
vi k3s.yaml # edit the IP so it points to the k3s server

mkdir ~/.kube
mv k3s.yaml .kube/config
```

---

# installing kubectl on Linux

I recommend going through this guide and install using native package management

[Install and Set Up kubectl on Linux](https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/)
[Install using native package management](https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/#install-using-native-package-management)

---
# kubectl auto completion and alias

[Enable kubectl autocompletion](https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/#enable-kubectl-autocompletion)

---
# install FluxCD

highly recommend at least skimming through Flux's docs, they are really good IMO and it is what I used to bootstrap FluxCD on my k3s cluster.
[fluxcd.io](https://fluxcd.io/flux/get-started/) 

### install Flux CLI

first export you github personal access token and username see [github](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/managing-your-personal-access-tokens#creating-a-personal-access-token-classic):
```bash
export GITHUB_TOKEN=<your-token>
export GITHUB_USER=<your-username>
```

then install the cli for me using 
```bash
curl -s https://fluxcd.io/install.sh | sudo bash

# check your k8s cluster
flux check --pre
```

bootstrap command
```bash
flux bootstrap github \
  --owner=$GITHUB_USER \
  --repository=fleet-infra \
  --branch=main \
  --path=./clusters/my-cluster \
  --personal
```

ensure that you edit the values to match your needs

once this command is complete and all components are healthy you can check your github repo to see if it succeeded.

---
