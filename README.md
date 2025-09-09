<table style="border-collapse: collapse; border: none;">
  <tr style="border-collapse: collapse; border: none;">
    <td style="border-collapse: collapse; border: none;">
      <a href="http://www.openairinterface.org/">
         <img src="https://gitlab.eurecom.fr/uploads/-/system/user/avatar/716/avatar.png?width=800" alt="" border=3 height=50 width=50>
         </img>
      </a>
    </td>
    <td style="border-collapse: collapse; border: none; vertical-align: center;">
      <b><font size = "5">FluxCD + Kubernetes Multitenancy Lab</font></b>
    </td>
  </tr>
</table>

# Author
**Luis Felipe Ariza Vesga** 
emails: lfarizav@gmail.com, lfarizav@unal.edu.co

# 🚀 Description

This repository contains step-by-step instructions to set up a **FluxCD GitOps environment** on a local **Kubernetes cluster using Kind**.  
The lab is designed for teaching **multitenancy** in the virtual event of the Fundación Hispana de Cloud Native el 11 de septiembre de 2025 a las 5 pm, where each team (tenant) has its own namespace, RBAC rules, and GitOps configuration.
In this lab we use Facebooc and Instavote applications running on facebooc and instavote namespaces.

---

## 📑 Table of Contents

1. [Introduction](#-introduction)  
2. [Prerequisites](#-prerequisites)  
3. [Install Tools](#-install-tools)  
   - [Install Docker](#install-docker)  
   - [Install Kind](#install-kind)
   - [Install Kubectl](#install-kubectl)
   - [Install Helm](#install-helm)
   - [Install FluxCD](#install-fluxcd)
4. [Create kind cluster](#create-kind-cluster)
5. [Alias bash completion](#alias-bash-completion)
6. [Install Multus](#-install-multus)  
7. [Bootstrap Flux](#-bootstrap-flux)
8. [Check Flux resources](#-check-flux-resources)
9. [Repo Structure](#-repo-structure)  
10. [Create a new tenant](#-create-a-new-tenant)
11. [Configure alerts with Slack](#-configure-alerts-with-slack)
    - [Create an Incoming Webhook for Slack](#create-an-incoming-webhook-for-slack)
    - [Create a secret for slacks incomming webhook url](#create-a-secret-for-slacks-incomming-webhook-url)
    - [Add a Provider to Connect to Slack from Flux](#add-a-provider-to-connect-to-slack-from-flux)
    - [Set Up an Alert to Send Notifications to Slack](#set-up-an-alert-to-send-notifications-to-slack)
    - [Update facebooc-deploy, instavote-deploy or both folders](#update-faceboocdeploy-instavotedeploy-or-both-folders)
12. [Results](#-results)
13. [Next Steps](#-next-steps)

---

## 📖 Introduction

This lab demonstrates how to use [FluxCD](https://fluxcd.io) for **multi-tenant GitOps** on Kubernetes.  
We will:  
- Install core tools (Docker, Kind, helm, kustomize, kubectl, Multus, and FluxCD).  
- Create a Kubernetes cluster.  
- Enable multi-networking with Multus CNI.  
- Bootstrap FluxCD with Git repositories.  
- Deploy Facebooc and Instavote applications in **isolated tenant namespaces**.  

---
## ✅ Install Requirements

### When using Ubuntu

#### Setup sudo with no password or your current user (optional)

Skip the next command if you already have that ability or if you want to enter a password each time you open a new terminal session for the current user.

```bash
echo "$USER ALL=(ALL) NOPASSWD: ALL" | sudo tee /etc/sudoers.d/$USER >/dev/null
```
Optionally, you can use the sudo visdo and add the line $USER ALL=(ALL) NOPASSWD: ALL
```bash
sudo visudo
```
---
## ✅ Prerequisites

- Ubuntu 22.04 (Intel architecture).  
- 4 CPUs, 8 GB RAM (minimum).  
- GitHub account with 3 repositories:
  - `facebooc` → Facebooc application source code.
  - `Instavote` → Instavote application source code.  
  - `facebook-deploy` → Facebooc tenant deployment configs.
  - `instavote-deploy` → Instavote tenant deployment configs.  
  - `flux-k8s-fleet-lab` → Fleet/platform configurations.  

---
#### Ensure Packages

Ensure you are up-to-date with all the required packages.
If you are using ubuntu as host operating system to run the lab,
which is the base where the whole environment has been tested, ensure you have the following packages installed:

```bash
sudo apt update
sudo apt upgrade
sudo apt -y install curl
sudo apt -y install git
sudo apt -y install net-tools
sudo apt -y install bash-completion
```
## 🛠 Install Tools
The best practice is to go to https://docs.docker.com/engine/install/ubuntu/ and follow the steps. The following are the steps that works in 4/9/2025.

### Install Docker
```bash
# Add Docker's official GPG key:
sudo apt-get update
sudo apt-get install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

# Add the repository to Apt sources:
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update
```
Install docker packages
```bash
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```
Add user to Docker group:
```bash
sudo usermod -aG docker $USER
newgrp docker
```
And reboot
```bash
reboot
```
### Prerequisites

Ensure you have the following tools installed on your computer.

- kind: version v0.30.0 or later
- kubectl: version v1.34.0 or later
- Kustomize: Version: v5.7.1 or later
- helm: version v3.18.6 or later
- FluxCD: version v2.6.4 or later

### Installation

The whole setup will be operational executing the next steps.

### Install Kind

To install kind you can use the next code snippet:

```bash
# For AMD64 / x86_64
[ $(uname -m) = x86_64 ] && curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.30.0/kind-linux-amd64
chmod +x ./kind
sudo mv ./kind /usr/local/bin/kind
kind version
```
It is recommended always go to https://kind.sigs.k8s.io/docs/user/quick-start/#installation to see updated steps.
---
### Install Kubectl
Go to the webpage https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/ and follow the steps
```bash
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl.sha256"
echo "$(cat kubectl.sha256)  kubectl" | sha256sum --check
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
kubectl version --client
```
---
### Install Helm
Go to the webpage https://helm.sh/docs/intro/install/ and follow the steps
```bash
sudo apt-get install curl gpg apt-transport-https --yes
curl -fsSL https://packages.buildkite.com/helm-linux/helm-debian/gpgkey | gpg --dearmor | sudo tee /usr/share/keyrings/helm.gpg > /dev/null
echo "deb [signed-by=/usr/share/keyrings/helm.gpg] https://packages.buildkite.com/helm-linux/helm-debian/any/ any main" | sudo tee /etc/apt/sources.list.d/helm-stable-debian.list
sudo apt-get update
sudo apt-get install helm
helm version
```
---
### Install FluxCD

Go to the webpage https://fluxcd.io/flux/installation/ and follow the steps
```bash
curl -s https://fluxcd.io/install.sh | sudo bash
```
---

## Create kind cluster

```bash
git clone https://github.com/lfarizav/flux-k8s-setup-lab.git
cd flux-k8s-setup-lab
kind create cluster --config kind-two-nodes-cluster.yaml --name dev
kubectl get nodes -owide
```
---
## Alias bash completion

As an optional step, add the completion executing the next snippets

```bash
echo 'alias ks=kubectl' >>~/.bashrc
echo 'complete -F __start_kubectl ks' >>~/.bashrc
```
---
## 🌐 Install Multus

Multus adds support for multiple network interfaces per pod.
```bash
# Apply Multus
kubectl apply -f https://raw.githubusercontent.com/k8snetworkplumbingwg/multus-cni/master/deployments/multus-daemonset.yml
# Control-plane bridge configuration
docker exec dev-control-plane ip link add br-net1 type bridge || true
docker exec dev-control-plane ip addr add 192.168.1.1/24 dev br-net1 || true
docker exec dev-control-plane ip link set br-net1 up
docker exec dev-control-plane sysctl -w net.ipv4.ip_forward=1
docker exec dev-control-plane iptables -t nat -A POSTROUTING -s 192.168.1.0/24 ! -o br-net1 -j MASQUERADE

# Download and install CNI plugins in control-plane
docker exec dev-control-plane sh -c "curl -LO https://github.com/containernetworking/plugins/releases/download/v1.7.1/cni-plugins-linux-amd64-v1.7.1.tgz && \
tar -xzf cni-plugins-linux-amd64-v1.7.1.tgz -C /opt/cni/bin && rm cni-plugins-linux-amd64-v1.7.1.tgz"

# Worker bridge configuration
docker exec dev-worker ip link add br-net1 type bridge || true
docker exec dev-worker ip addr add 192.168.1.1/24 dev br-net1 || true
docker exec dev-worker ip link set br-net1 up
docker exec dev-worker sysctl -w net.ipv4.ip_forward=1
docker exec dev-worker iptables -t nat -A POSTROUTING -s 192.168.1.0/24 ! -o br-net1 -j MASQUERADE

# Download and install CNI plugins in worker
docker exec dev-worker sh -c "curl -LO https://github.com/containernetworking/plugins/releases/download/v1.7.1/cni-plugins-linux-amd64-v1.7.1.tgz && \
tar -xzf cni-plugins-linux-amd64-v1.7.1.tgz -C /opt/cni/bin && rm cni-plugins-linux-amd64-v1.7.1.tgz"
```
---
## 🔥 Check Flux pre-requisites once the KIND cluster is ready and add github credentials
```bash
flux check --pre
flux check
#After installing fluxcd is important to set GITHUB_TOKEN and GITHUB_USER environment variables. The following are the env for helm_open5gs github repository
export GITHUB_TOKEN=<github-token>
export GITHUB_USER=<github-user>
echo $GITHUB_TOKEN
echo $GITHUB_USER
```
---

## 🔥 Bootstrap Flux
Once the cluster is ready, bootstrap Flux with your fleet repo
```bash
#Create a new blank repostitory called flux-k8s-fleet-lab in Github then you can bootstrap with Flux
#Then, you need to bootstrap flux with your github project.
flux bootstrap github --owner=$GITHUB_USER --repository=flux-k8s-fleet-lab --branch=main --path=./clusters/dev --personal --log-level=debug --network-policy=false --components-extra=image-reflector-controller,image-automation-controller
#Bootstrapping creates a gitrepository and a kustomization to check them use the following command
flux get all -A
#Once the fleet repository is bootstrapped you will find a new push made by flux in your new repository, a new directory clusters/dev/flux-system and a deploy key
#In that folder we are going to find gotk(gitops toolkit)-synch.yaml manifest composed of gitrepository and kustomization - API-Resources (flux components) in the flux-system namespace - Kustomization that join together resources gotk-components.yaml and gotk-sync.yaml.

#Only creates 1 bootstrap with Flux then if you want 1 cluster with different environments or projects using namespaces and give freedom to developers
```
---
## 🔥 Check Flux Resources
Try after bootstraping task is completed, the new namespace called flux-system and all pods must be running. If k is not recognized, you need to check the alias kubectl=k
```bash
k get ns
k get all -n flux-system
```
---

## 🗂 Repo Structure

- **flux-k8s-fleet-lab** → Cluster + tenants.  
- **instavote-deploy** → Instavote infrastructure.  
- **facebooc-deploy** → Facebooc infrastructure.  
- **instavote** → Instavote code.  
- **facebooc** → Facebooc code.  
---

## 👩‍💻 Create a new tenant
This laboratory has two tenants. They are called Facebooc and Instavote. 
```bash
flux create tenant facebooc --with-namespace=facebooc --export > flux-k8s-fleet-lab/projects/base/facebooc/rbac.yaml
flux create tenant instavote --with-namespace=instavote  --export > flux-k8s-fleet-lab/projects/base/instavote/rbac.yaml
```
However, if you need to add a new tenant to the dev cluster just use FluxCD as follows:
### Create the tenant
```bash
flux create tenant <new-tenant-name> --with-namespace=<new-tenant-name> --export > flux-k8s-fleet-lab/projects/base/<new-tenant-name>/rbac.yaml
```
### Add a source tip git for the new repository.
```bash
flux create source git <new-tenant-name>-deploy \
--namespace=<new-tenant-name> \
--url=https://github.com/dopsdemo/<new-tenant-name>-deploy \
--branch=main \
--export >
flux-k8s-fleet-lab/projects/base/instavote/<new-tenant-name>-deploy-gitrepository.yaml
```
### Add a kustomization for the that tenant.
```bash
flux create kustomization <new-tenant-name>-deploy \
--namespace=<new-tenant-name> \
--service-account=<new-tenant-name>\
--source=GitRepository/<new-tenant-name>-deploy \
--path="./flux" \
--export >
flux-k8s-fleet-lab/projects/base/<new-tenant-name>/<new-tenant-name>-deploy-kustomization.yaml
```
### Auto-generate kustomization.yaml. 
```bash
cd flux-k8s-fleet-lab/projects/base/<new-tenant-name>/
kustomize create --autodetect
```
### Auto-generate kustomization.yaml in dev cluster for the new <new-tenant-name> tenant. 
```bash
cd cd flux-k8s-fleet-lab/
cat << EOF tee ./projects/dev/<new-tenant-name>/<new-tenant-name>-deploy-kustomization.yaml
apiVersion: kustomize.toolkit.fluxcd.io/v1beta1
kind: Kustomization
metadata:
  name: <new-tenant-name>-deploy
  namespace: <new-tenant-name>
spec:
  path: ./flux/dev
EOF
```
---
## 🚀 Configure alerts with Slack
To configure alerts using Slack we need to follow these steps:
1. Create an Incoming Webhook for Slack
2. Create a secret for slack's incomming webhook url
3. Add a Provider to Connect to Slack from Flux
4. Set Up an Alert to Send Notifications to Slack
5. Update facebooc-deploy, instavote-deploy or both folders
---
### Create an Incoming Webhook for Slack

If you do have the appropriate access, create a new channel or use an existing one and browse to the configuration on the top right corner of the channel. Go to Slack marketplace and search for Incomming Webhooks. Then hit the button "add to slack". Then in "Post to Channel" select the channel you are already created. Finally, hit the button "Add Incoming Webhooks Integration" to get the url.

### Create a secret for slack's incomming webhook url
```bash
kubectl create secret -n facebooc generic slack-url --from-literal=address=https://hooks.slack.com/services/xxx/yyy/zzz
kubectl create secret -n instavote generic slack-url --from-literal=address=https://hooks.slack.com/services/xxx/yyy/zzz
kubectl get secrets -n facebooc
kubectl get secrets -n instavote
```
### Add a Provider to Connect to Slack from Flux
Remember xxxxx must be the same as your Slack channel
```bash
flux create alert-provider slack \
--type=slack \
--channel= xxxxx \
--secret-ref=slack-url
flux get alert-providers
```
### Set Up an Alert to Send Notifications to Slack
Now, create an Alert to send notifications to Slack. For this task you will use the provider created above to track the
changes for the following resources:
1.  Kustomization/*
2.  GitRepository/*
3.  HelmRelease/*
```bash
flux create alert slack-notif \
--provider-ref=slack \
--event-source=GitRepository/* \
--event-source=Kustomization/* \
--event-source=HelmRelease/* \
--event-severity=info \
flux get alert
```
### Update faceboocdeploy, instavotedeploy or both folders
Export alert and alert-provider to yaml files. Open slack-notif-alert.yaml and slack-provider.yaml and add the namespace correspondly to facebooc or instavote applications. Then move them to ~/instavote-deploy/flux/base or ~/facebook-deploy/flux/base

```bash
flux export alert slack-notif > slack-notif-alert.yaml
flux export alert-provider slack > slack-provider.yaml
```
---
## 🔥 Results
At the end, you will the applications inside facebooc and instavote namespaces as follows:
```bash
luis@lfarizav:~/facebooc-deploy$ k get ns
NAME                 STATUS   AGE
default              Active   2d
facebooc             Active   18m
flux-system          Active   18m
instavote            Active   18m
kube-node-lease      Active   2d
kube-public          Active   2d
kube-system          Active   2d
local-path-storage   Active   2d
luis@lfarizav:~/facebooc-deploy$ k get all -n instavote
NAME                                     READY   STATUS    RESTARTS   AGE
pod/instavote-db-postgres-0              1/1     Running   0          18m
pod/instavote-result-f9b57c5cf-wlcwq     1/1     Running   0          18m
pod/redis-85b9d4ffb8-c5sgg               1/1     Running   0          18m
pod/vote-8fc66c9f9-nc27z                 1/1     Running   0          18m
pod/worker-deployment-6d5dc55fbb-b7m4n   1/1     Running   0          18m

NAME                       TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
service/db                 ClusterIP   10.96.62.100    <none>        5432/TCP       18m
service/instavote-result   NodePort    10.96.137.200   <none>        80:30804/TCP   18m
service/redis              ClusterIP   10.96.36.33     <none>        6379/TCP       18m
service/vote               NodePort    10.96.23.29     <none>        80:30200/TCP   18m

NAME                                READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/instavote-result    1/1     1            1           18m
deployment.apps/redis               1/1     1            1           18m
deployment.apps/vote                1/1     1            1           18m
deployment.apps/worker-deployment   1/1     1            1           18m

NAME                                           DESIRED   CURRENT   READY   AGE
replicaset.apps/instavote-result-f9b57c5cf     1         1         1       18m
replicaset.apps/redis-85b9d4ffb8               1         1         1       18m
replicaset.apps/vote-8fc66c9f9                 1         1         1       18m
replicaset.apps/worker-deployment-6d5dc55fbb   1         1         1       18m

NAME                                     READY   AGE
statefulset.apps/instavote-db-postgres   1/1     18m

luis@lfarizav:~/facebooc-deploy$ k get all -n facebooc
NAME                            READY   STATUS    RESTARTS   AGE
pod/facebooc-648478b86d-vnnlf   1/1     Running   0          18m

NAME               TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)           AGE
service/facebooc   NodePort   10.96.150.142   <none>        16000:32134/TCP   18m

NAME                       READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/facebooc   1/1     1            1           18m

NAME                                  DESIRED   CURRENT   READY   AGE
replicaset.apps/facebooc-648478b86d   1         1         1       18m
```
## 🚀 Next Steps 
- Play with the applications and use the repositories as templates for your next FluxCD automatic deployment
- Connect Tekton CI with Flux CD so one your push or merge is authorized from devops team, it is automatically reconciled with kubernetes
  1.  Install Tekton pipelines in the Kubernetes cluster
  ```bash
  kubectl apply --filename https://storage.googleapis.com/tekton-releases/pipeline/latest/release.yaml
  ```
  2.  Install Tekton CLI
  ```bash
  curl -LO https://github.com/tektoncd/cli/releases/download/v0.42.0/tektoncd-cli-0.42.0_Linux-64bit.deb
  sudo dpkg -i ./tektoncd-cli-0.42.0_Linux-64bit.deb
  ```
  3.  Create a Pipeline and a PipelineRun with task fetch-repo with git-clone tekton application, build-image using kaniko application and finnaly publish that image to a repository like Dockerhub.
---
