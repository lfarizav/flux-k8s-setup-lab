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
The lab is designed for teaching **multitenancy**, where each team (tenant) has its own namespace, RBAC rules, and GitOps configuration.

---

## 📑 Table of Contents

1. [Introduction](#-introduction)  
2. [Prerequisites](#-prerequisites)  
3. [Install Tools](#-install-tools)  
   - [Install Docker](#docker)  
   - [Install Kind](#kind)
   - [Install Kubectl](#kubectl)
   - [Install Helm](#helm)
   - [Install FluxCD](#fluxcd)
   - [Installl Multus CNI](#multus-cni)  
4. [Create Kind Cluster](#-create-kind-cluster)  
5. [Install Multus](#-install-multus)  
6. [Bootstrap Flux](#-bootstrap-flux)  
7. [Repo Structure](#-repo-structure)  
8. [First Tenant Deployment](#-first-tenant-deployment)  
9. [Next Steps](#-next-steps)

---

## 📖 Introduction

This lab demonstrates how to use [FluxCD](https://fluxcd.io) for **multi-tenant GitOps** on Kubernetes.  
We will:  
- Install core tools (Docker, Kind, kubectl, FluxCD).  
- Create a Kubernetes cluster.  
- Enable multi-networking with Multus CNI.  
- Bootstrap FluxCD with Git repositories.  
- Deploy applications in **isolated tenant namespaces**.  

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
## ✅ Prerequisites

- Ubuntu 22.04 (Intel architecture).  
- 4 CPUs, 8 GB RAM (minimum).  
- GitHub account with 3 repositories:
  - `flux-k8s-code-lab` → Application source code.  
  - `flux-k8s-deploy-lab` → Tenant deployment configs.  
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

## ⚙️ Create Kind Cluster
```bash
git clone https://github.com/lfarizav/flux-k8s-setup-lab.git
cd flux-k8s-setup-lab
kind create cluster --config kind-two-nodes-cluster.yaml --name dev
kubectl get nodes -owide
```
---
## ⚙️ Create kubectl alias k and bash completion
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
GITHUB_TOKEN=<github-token>
GITHUB_USER=lfarizav
```
---

## 🔥 Bootstrap Flux
Once the cluster is ready, bootstrap Flux with your fleet repo:
```bash
#Then, you need to bootstrap flux with your github project.
flux bootstrap github   --owner=$GITHUB_USER   --repository=open5gs   --branch=main   --path=./clusters/dev   --personal   --token-auth --log-level=debug --network-policy=false 
```
---

## 🗂 Repo Structure

- **flux-k8s-fleet-lab** → Cluster + tenants.  
- **flux-k8s-deploy-lab** → Tenant manifests.  
- **flux-k8s-code-lab** → Application code.  

---

## 👩‍💻 First Tenant Deployment

1. Create a namespace for **team-a**.  
2. Add a `Kustomization` in `flux-k8s-deploy-lab/tenants/team-a/`.  
3. Flux will reconcile and deploy the app automatically.  

---

## 🚀 Next Steps

- Add multiple tenants.  
- Apply RBAC for tenant isolation.  
- Enforce policies with Kyverno or OPA.  
- Explore HelmReleases for shared services.  

---
