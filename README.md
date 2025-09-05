FLUXCD GITOP WORKSHOPTable of ContentsForewordClone sourcesInstall RequirementsUsing UbuntuSetup sudo with no password (optional)Ensure packagesEnsure docker installationPrerequisites SummaryInstallationInstall Docker EngineInstall KindInstall kubectl CLIInstall FluxCD CLISetupCreate kind clusterInstall Multus CNIBootstrap FluxThe Experiment DemoOnboard Your First TenantDeploy an ApplicationCleanupForewordThis repository contains a full demo workshop to start using FluxCD for multitenant GitOps on a local Kubernetes cluster. This project provides a hands-on lab to build a multitenant cluster managed entirely through GitOps principles. Using local tools like KinD on an Ubuntu 24.04 system, you will create a complete, reproducible environment that separates platform (fleet) management from application deployment.This repository, flux-k8s-fleet-lab, acts as the central platform or fleet repository. It defines the overall state of the cluster, including tenant namespaces, permissions, and configurations.TOCClone the following repositoriesFor this lab to work, you need your own forked copies of the three repositories.Navigate to each repository on GitHub and click the "Fork" button:https://github.com/<your-username>/flux-k8s-fleet-labhttps://github.com/<your-username>/flux-k8s-deploy-labhttps://github.com/<your-username>/flux-k8s-code-labClone your forked fleet repository to your local machine:git clone [https://github.com/](https://github.com/)<your-username>/flux-k8s-fleet-lab.git
cd flux-k8s-fleet-lab
Replace <your-username> with your GitHub username.TOCInstall RequirementsWhen using UbuntuSetup sudo with no password or your current user (optional)Skip the next command if you already have that ability or if you want to enter a password each time.echo "$USER ALL=(ALL) NOPASSWD: ALL" | sudo tee /etc/sudoers.d/$USER >/dev/null
TOCEnsure PackagesEnsure you are up-to-date with all the required packages.sudo apt update
sudo apt -y install ca-certificates curl git
TOCEnsure docker installationCheck if the docker daemon is running correctly.systemctl status docker
Ensure the docker access is granted to the current user.sudo usermod -aG docker $USER
newgrp docker
You may need to log out and log back in for the group change to take full effect.TOCPrerequisites SummaryEnsure you have the following tools installed on your computer.Docker: Latest versionkind: version 0.20.0 or laterkubectl: version 1.28 or laterflux: version 2.x or laterInstallationThe whole setup will be operational after executing the next steps.1. Docker EngineKinD uses Docker containers as Kubernetes nodes. These steps will install the Docker Engine from Docker's official repository.Set up Docker's apt repository:# Add Docker's official GPG key:
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
Verify the installation:docker run hello-world
TOC2. KinD (Kubernetes in Docker)To install kind you can use the next code snippet for your Intel/AMD (x86_64) architecture.curl -Lo ./kind [https://kind.sigs.k8s.io/dl/v0.20.0/kind-linux-amd64](https://kind.sigs.k8s.io/dl/v0.20.0/kind-linux-amd64)
chmod +x ./kind
sudo mv ./kind /usr/local/bin/kind
Verify the installation:kind --version
TOC3. kubectlTo install the kubectl latest version, run:sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl gpg

curl -fsSL [https://pkgs.k8s.io/core:/stable:/v1.28/deb/Release.key](https://pkgs.k8s.io/core:/stable:/v1.28/deb/Release.key) | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] [https://pkgs.k8s.io/core:/stable:/v1.28/deb/](https://pkgs.k8s.io/core:/stable:/v1.28/deb/) /' | sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt-get update
sudo apt-get install -y kubectl
Verify the installation:kubectl version --client
TOC4. Flux CD CLIUse the following commands to install the Flux CLI:curl -s [https://fluxcd.io/install.sh](https://fluxcd.io/install.sh) | sudo bash
Verify the installation:flux --version
TOCSetupStep 1: Create the KinD ClusterNow it's time to create the kind cluster, using a proper configuration file.Create a file named kind-config.yaml in the root of your cloned flux-k8s-fleet-lab directory:kind: Cluster
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
This will take a few minutes. Once complete, your kubectl context will be automatically set to the new cluster.TOCStep 2: Install Multus CNIMultus is a CNI plugin that enables attaching multiple network interfaces to pods. We install it for future networking labs.kubectl apply -f [https://raw.githubusercontent.com/k8snetworkplumbingwg/multus-cni/master/deployments/multus-daemonset-thick.yml](https://raw.githubusercontent.com/k8snetworkplumbingwg/multus-cni/master/deployments/multus-daemonset-thick.yml)
TOCStep 3: Bootstrap FluxThis critical step installs Flux onto your cluster and configures it to watch your flux-k8s-fleet-lab repository.Create a GitHub Personal Access Token (PAT):Go to GitHub > Settings > Developer settings > Personal access tokens > Tokens (classic).Click "Generate new token".Give it a name (e.g., flux-lab-token) and set an expiration date.Select the repo scope.Click "Generate token" and copy the token immediately.Export Environment Variables:export GITHUB_TOKEN="<Your-PAT-Token>"
export GITHUB_USER="<Your-GitHub-Username>"
Run the Bootstrap Command:This command tells Flux to manage itself from a clusters/flux-lab path within your repository.flux bootstrap github \
  --owner=$GITHUB_USER \
  --repository=flux-k8s-fleet-lab \
  --branch=main \
  --path=./clusters/flux-lab \
  --personal
Verify:Flux will commit its manifests to your repository and then sync them to the cluster. Check that the components are running:kubectl get pods -n flux-system
You should see several pods running after a minute or two.TOCThe Experiment DemoThe next steps demonstrate the core GitOps workflow.1. Onboard Your First TenantNow, as the platform admin, you will onboard a new tenant named app-team.Define the TenantIn your local flux-k8s-fleet-lab repository, create the file tenants/app-team.yaml. This manifest tells Flux to watch the tenant's application repository.apiVersion: v1
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
Remember to replace <your-username> with your actual GitHub username.Commit and Watch Flux WorkCommit the new tenant manifest to your flux-k8s-fleet-lab repo:git add tenants/app-team.yaml
git commit -m "feat: Onboard app-team tenant"
git push
Flux will soon detect the change. You can verify this:# Check that Flux sees the new instructions
flux get kustomizations --watch

# Check that the tenant's namespace has been created
kubectl get ns | grep app-team
TOC2. Deploy an Application (as a Tenant)Now, switch hats. You are a developer on app-team. You only have access to the flux-k8s-deploy-lab repository.Add Application ManifestsClone your forked flux-k8s-deploy-lab repository in a separate directory:git clone [https://github.com/](https://github.com/)<your-username>/flux-k8s-deploy-lab.git
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
Commit, Push, and VerifyCommit the new application manifest to the flux-k8s-deploy-lab repo:git add nginx-deployment.yaml
git commit -m "feat: Deploy nginx web server"
git push
Flux will pick up this change and apply the manifest. Verify on the cluster:# Wait a minute, then check for the pods in the tenant namespace
kubectl get pods -n app-team
# NAME                                READY   STATUS    RESTARTS   AGE
# nginx-web-server-75d7c4d49-abcde    1/1     Running   0          30s
# nginx-web-server-75d7c4d49-fghij    1/1     Running   0          30s
Congratulations! You have successfully deployed an application using a multi-repo GitOps workflow.TOCCleanupTo tear down the entire lab environment, simply delete the KinD cluster:kind delete cluster --name flux-lab
TOC
