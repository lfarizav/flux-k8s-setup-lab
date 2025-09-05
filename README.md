GitOps on Kubernetes with Flux: A Multitenant Lab for Ubuntu 24.04This project provides a hands-on lab to build a multitenant Kubernetes cluster managed entirely through GitOps principles with Flux CD. Using local tools like KinD on an Ubuntu 24.04 system, you will create a complete, reproducible environment that separates platform (fleet) management from application deployment.This repository, flux-k8s-fleet-lab, acts as the central platform or fleet repository. It defines the overall state of the cluster, including tenant namespaces, permissions, and configurations.The Lab's ArchitectureThis lab uses three distinct repositories to simulate a real-world GitOps environment:🚢 flux-k8s-fleet-lab (This Repo): The "admin" or "platform" repo. It's the single source of truth for the cluster's configuration. Flux is bootstrapped to watch this repository.📦 flux-k8s-deploy-lab: A "tenant" or "application" repo. It contains the Kubernetes manifests for applications deployed by a specific team or tenant. It is watched by Flux based on instructions from the fleet repo.💻 flux-k8s-code-lab: The application's source code repo. This is where developers work. A CI pipeline (e.g., GitHub Actions) in this repo would build container images and update the manifests in flux-k8s-deploy-lab.Table of ContentsPrerequisites: Installing Your Toolchain on Ubuntu 24.041. Docker Engine2. KinD (Kubernetes in Docker)3. kubectl4. Flux CD CLILab Setup: Building the ClusterStep 1: Fork and Clone the RepositoriesStep 2: Create the KinD ClusterStep 3: Install Multus CNIStep 4: Bootstrap FluxYour First Tenant: Onboarding a Team1. Define the Tenant2. Commit and Watch Flux WorkDeploying an Application (as a Tenant)1. Add Application Manifests2. Commit, Push, and VerifyCleanupPrerequisites: Installing Your Toolchain on Ubuntu 24.04Before starting the lab, you must install the following tools on your Ubuntu 24.04 machine.1. Docker EngineKinD uses Docker containers as Kubernetes nodes. These steps will install the Docker Engine from Docker's official repository.Set up Docker's apt repository:# Add Docker's official GPG key:
sudo apt-get update
sudo apt-get install -y ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL [https://download.docker.com/linux/ubuntu/gpg](https://download.docker.com/linux/ubuntu/gpg) -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

# Add the repository to Apt sources:
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] [https://download.docker.com/linux/ubuntu](https://download.docker.com/linux/ubuntu) \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update
Install the latest Docker packages:sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
Add your user to the docker group to run Docker commands without sudo (requires logout/login to take effect):sudo usermod -aG docker $USER
newgrp docker # Activate the changes for the current terminal session
Verify the installation:docker run hello-world
2. KinD (Kubernetes in Docker)KinD is a tool for running local Kubernetes clusters using Docker.# For AMD64 / x86_64
[ $(uname -m) = x86_64 ] && curl -Lo ./kind [https://kind.sigs.k8s.io/dl/v0.20.0/kind-linux-amd64](https://kind.sigs.k8s.io/dl/v0.20.0/kind-linux-amd64)
# For ARM64
[ $(uname -m) = aarch64 ] && curl -Lo ./kind [https://kind.sigs.k8s.io/dl/v0.20.0/kind-linux-arm64](https://kind.sigs.k8s.io/dl/v0.20.0/kind-linux-arm64)
chmod +x ./kind
sudo mv ./kind /usr/local/bin/kind
Verify the installation:kind --version
3. kubectlkubectl is the command-line tool for interacting with the Kubernetes API.sudo apt-get update
# apt-transport-https may be a dummy package; if so, you can skip that package
sudo apt-get install -y apt-transport-https ca-certificates curl gpg

# If the folder `/etc/apt/keyrings` does not exist, it should be created before the curl command, read the note below.
# sudo mkdir -p -m 755 /etc/apt/keyrings
curl -fsSL [https://pkgs.k8s.io/core:/stable:/v1.28/deb/Release.key](https://pkgs.k8s.io/core:/stable:/v1.28/deb/Release.key) | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] [https://pkgs.k8s.io/core:/stable:/v1.28/deb/](https://pkgs.k8s.io/core:/stable:/v1.28/deb/) /' | sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt-get update
sudo apt-get install -y kubectl
Verify the installation:kubectl version --client
4. Flux CD CLIThe Flux CLI is used to bootstrap the cluster and interact with the Flux components.curl -s [https://fluxcd.io/install.sh](https://fluxcd.io/install.sh) | sudo bash
Verify the installation:flux --version
Lab Setup: Building the ClusterStep 1: Fork and Clone the RepositoriesFor this lab to work, you need your own copies of the three repositories.Navigate to each repository on GitHub and click the "Fork" button:https://github.com/<your-username>/flux-k8s-fleet-labhttps://github.com/<your-username>/flux-k8s-deploy-labhttps://github.com/<your-username>/flux-k8s-code-labClone your forked fleet repository to your local machine:git clone [https://github.com/](https://github.com/)<your-username>/flux-k8s-fleet-lab.git
cd flux-k8s-fleet-lab
Replace <your-username> with your GitHub username.Step 2: Create the KinD ClusterWe will use a configuration file to create a cluster with one control-plane node and one worker node.Create a file named kind-config.yaml in the root of your cloned flux-k8s-fleet-lab directory:kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
name: flux-lab
nodes:
- role: control-plane
  kubeadmConfigPatches:
  - |
    kind: InitConfiguration
    nodeRegistration:
      kubeletExtraArgs:
        node-labels: "ingress-ready=true"
  extraPortMappings:
  - containerPort: 80
    hostPort: 80
    protocol: TCP
  - containerPort: 443
    hostPort: 443
    protocol: TCP
- role: worker
Create the cluster using this configuration:kind create cluster --config=kind-config.yaml
This will take a few minutes. Once complete, your kubectl context will be automatically set to the new cluster.Step 3: Install Multus CNIMultus is a CNI plugin that enables attaching multiple network interfaces to pods. We install it for future networking labs.kubectl apply -f [https://raw.githubusercontent.com/k8snetworkplumbingwg/multus-cni/master/deployments/multus-daemonset-thick.yml](https://raw.githubusercontent.com/k8snetworkplumbingwg/multus-cni/master/deployments/multus-daemonset-thick.yml)
Step 4: Bootstrap FluxThis critical step installs Flux onto your cluster and configures it to watch your flux-k8s-fleet-lab repository.Create a GitHub Personal Access Token (PAT):Go to GitHub > Settings > Developer settings > Personal access tokens > Tokens (classic).Click "Generate new token".Give it a name (e.g., flux-lab-token).Set an expiration date.Select the repo scope.Click "Generate token" and copy the token immediately.Export Environment Variables:export GITHUB_TOKEN="<Your-PAT-Token>"
export GITHUB_USER="<Your-GitHub-Username>"
Run the Bootstrap Command:This command tells Flux to manage itself from a clusters/flux-lab path within your repository.flux bootstrap github \
  --owner=$GITHUB_USER \
  --repository=flux-k8s-fleet-lab \
  --branch=main \
  --path=./clusters/flux-lab \
  --personal
Verify:Flux will commit its manifests to your repository and then sync them to the cluster. Check that the components are running:kubectl get pods -n flux-system
You should see several pods running after a minute or two.Your First Tenant: Onboarding a TeamNow, as the platform admin, you will onboard a new tenant named app-team.1. Define the TenantIn your local flux-k8s-fleet-lab repository, create the following file structure and content. This manifest tells Flux to watch the tenant's application repository.File: tenants/app-team.yamlapiVersion: v1
kind: Namespace
metadata:
  name: app-team
---
apiVersion: source.toolkit.fluxcd.io/v1
kind: GitRepository
metadata:
  name: app-team-apps
  namespace: flux-system
spec:
  interval: 1m
  url: [https://github.com/](https://github.com/)<your-username>/flux-k8s-deploy-lab # <-- IMPORTANT: Change this!
  ref:
    branch: main
---
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: app-team-deploy
  namespace: flux-system
spec:
  interval: 5m
  sourceRef:
    kind: GitRepository
    name: app-team-apps
  path: "./"
  prune: true
  targetNamespace: app-team
Remember to replace <your-username> with your actual GitHub username.2. Commit and Watch Flux WorkCommit the new tenant manifest to your flux-k8s-fleet-lab repo:git add tenants/app-team.yaml
git commit -m "feat: Onboard app-team tenant"
git push
Verify:Flux will soon detect the change. You can force a reconciliation or wait.# Check that Flux sees the new instructions
flux get kustomizations --watch

# Check that the tenant's namespace has been created
kubectl get ns | grep app-team
Deploying an Application (as a Tenant)Now, switch hats. You are a developer on app-team. You only have access to the flux-k8s-deploy-lab repository.1. Add Application ManifestsClone your forked flux-k8s-deploy-lab repository in a separate directory:git clone [https://github.com/](https://github.com/)<your-username>/flux-k8s-deploy-lab.git
cd flux-k8s-deploy-lab
Create a simple Nginx deployment file named nginx-deployment.yaml:apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-web-server
  namespace: app-team # This is important
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.21.6
        ports:
        - containerPort: 80
2. Commit, Push, and VerifyCommit the new application manifest to the flux-k8s-deploy-lab repo:git add nginx-deployment.yaml
git commit -m "feat: Deploy nginx web server"
git push
Verify on the cluster:Flux will pick up this change and apply the manifest.# Wait a minute, then check for the pods in the tenant namespace
kubectl get pods -n app-team
# NAME                                READY   STATUS    RESTARTS   AGE
# nginx-web-server-75d7c4d49-abcde    1/1     Running   0          30s
# nginx-web-server-75d7c4d49-fghij    1/1     Running   0          30s
Congratulations! You have successfully deployed an application using a multi-repo GitOps workflow.CleanupTo tear down the entire lab environment, simply delete the KinD cluster:kind delete cluster --name flux-lab
