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
   - [Docker](#docker)  
   - [kubectl](#kubectl)  
   - [Kind](#kind)  
   - [Multus CNI](#multus-cni)  
   - [FluxCD CLI](#fluxcd-cli)  
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
## Install Requirements

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

- Ubuntu 24.04 (Intel architecture).  
- 2 CPUs, 4 GB RAM (minimum).  
- GitHub account with 3 repositories:
  - `flux-k8s-code-lab` → Application source code.  
  - `flux-k8s-deploy-lab` → Tenant deployment configs.  
  - `flux-k8s-fleet-lab` → Fleet/platform configurations.  

---

## 🛠 Install Tools

### Docker
```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y ca-certificates curl gnupg lsb-release apt-transport-https software-properties-common
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
sudo systemctl enable docker --now
```

Add user to Docker group:
```bash
sudo usermod -aG docker $USER
newgrp docker
```

---

### kubectl
```bash
curl -LO "https://dl.k8s.io/release/$(curl -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x kubectl
sudo mv kubectl /usr/local/bin/
kubectl version --client
```

---

### Kind
```bash
curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.24.0/kind-linux-amd64
chmod +x ./kind
sudo mv ./kind /usr/local/bin/kind
kind version
```

---

### Multus CNI
```bash
kubectl apply -f https://raw.githubusercontent.com/k8snetworkplumbingwg/multus-cni/master/deployments/multus-daemonset-thick.yml
kubectl -n kube-system get pods | grep multus
```

---

### FluxCD CLI
```bash
curl -s https://fluxcd.io/install.sh | sudo bash
flux --version
```

---

## ⚙️ Create Kind Cluster
```bash
kind create cluster --name flux-lab
kubectl cluster-info --context kind-flux-lab
```

---

## 🌐 Install Multus
Multus adds support for multiple network interfaces per pod.
```bash
kubectl apply -f https://raw.githubusercontent.com/k8snetworkplumbingwg/multus-cni/master/deployments/multus-daemonset-thick.yml
kubectl -n kube-system get pods | grep multus
```

---

## 🔥 Bootstrap Flux
Once the cluster is ready, bootstrap Flux with your fleet repo:
```bash
flux bootstrap github   --owner=<your-github-username>   --repository=flux-k8s-fleet-lab   --branch=main   --path=clusters/cluster01
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
