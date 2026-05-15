# Multi-Service Voting App on Kubernetes KIND with Argo CD 🚀

This guide explains how to deploy a multi-service voting application on a **Kubernetes KIND cluster** running inside an **AWS EC2 Ubuntu instance**. The app is managed using **Argo CD** and can be accessed through port forwarding.

> **KIND** means **Kubernetes IN Docker**. It creates Kubernetes cluster nodes as Docker containers, which makes it great for learning and practice.

![Kubernetes KIND Voting App Architecture](./k8s-kind-voting-app.png)

## What You Will Build 🧱

By the end of this setup, you will have:

- An Ubuntu EC2 instance
- Docker installed on the EC2 instance
- A 3-node Kubernetes cluster using KIND
- `kubectl` configured to manage the cluster
- Argo CD installed for GitOps deployment
- A voting app deployed from GitHub
- Vote and result web pages accessible from your browser
- Optional Kubernetes Dashboard access
- Optional Prometheus and Grafana monitoring

## Prerequisites ✅

Before starting, make sure you have:

- An AWS account
- Basic knowledge of EC2 and SSH
- PuTTY, MobaXterm, or any SSH client
- A key pair to connect to your EC2 instance
- A GitHub repository that contains Kubernetes manifests for the voting app

Example project repository:

```text
https://github.com/agravi987/k8s-kind-voting-app
```

## Credit 🙏

This project repository was forked from `LondheShubham153/k8s-kind-voting-app` for learning and practice purposes.

Original repository:

```text
https://github.com/LondheShubham153/k8s-kind-voting-app
```

## Important Security Note 🔐

This setup is for **learning and practice**.

For a real project, do not open all ports to everyone. In the EC2 security group, allow only the required ports and restrict access to your own IP address whenever possible.

Common ports used in this guide:

| Port | Purpose |
| --- | --- |
| `22` | SSH connection |
| `8443` | Argo CD UI |
| `5000` | Voting app UI |
| `5001` | Result app UI |
| `8080` | Kubernetes Dashboard, optional |
| `3000` | Grafana UI, optional monitoring |
| `9090` | Prometheus UI, optional monitoring |
| `9093` | Alertmanager UI, optional monitoring |

## Step 1: Create an EC2 Instance ☁️

Create an EC2 instance with the following recommended settings:

| Setting | Value |
| --- | --- |
| Name | `Kubernetes-cluster` |
| AMI | Ubuntu |
| Instance type | `t2.medium` or higher |
| Storage | At least `15 GB` |
| Key pair | Create or select an existing key pair |
| Security group | Allow required ports only |

After the instance is running, copy its **public IP address**. You will use it to connect through SSH and open the web apps.

## Step 2: Connect to the EC2 Instance 🔌

Connect to the EC2 instance using PuTTY, MobaXterm, or SSH.

Example SSH command:

```bash
ssh -i your-key.pem ubuntu@<EC2_PUBLIC_IP>
```

Replace `<EC2_PUBLIC_IP>` with your actual EC2 public IP address.

## Step 3: Install Docker 🐳

KIND needs Docker because it runs Kubernetes nodes as Docker containers.

Run these commands on the EC2 instance:

```bash
sudo apt-get update
sudo apt-get install -y docker.io
sudo systemctl enable --now docker
sudo usermod -aG docker $USER
newgrp docker
docker ps
```

If `docker ps` shows an empty list without an error, Docker is working correctly.

> If you still get a permission error, log out of the EC2 instance and log in again.

## Step 4: Create a Working Directory 📁

Create one folder to keep all installation files:

```bash
mkdir -p ~/k8s-install
cd ~/k8s-install
```

## Step 5: Install KIND ⚙️

Create the KIND install script:

```bash
nano install_kind.sh
```

Paste this content:

```bash
#!/bin/bash
set -e

# Install KIND for AMD64 / x86_64 Linux
curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.31.0/kind-linux-amd64
chmod +x ./kind
sudo mv ./kind /usr/local/bin/kind
```

Save the file, then run:

```bash
chmod +x install_kind.sh
./install_kind.sh
kind --version
```

If the version is displayed, KIND is installed successfully.

## Step 6: Install kubectl 🧰

`kubectl` is the command-line tool used to control Kubernetes.

This guide uses Kubernetes `v1.30.0` in the KIND cluster, so install a matching `kubectl` version:

```bash
curl -LO https://dl.k8s.io/release/v1.30.0/bin/linux/amd64/kubectl
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
kubectl version --client
```

> Tip: Try to keep your `kubectl` version within one minor version of your Kubernetes cluster version.

## Step 7: Create a 3-Node KIND Cluster 🧩

Create the KIND cluster configuration file:

```bash
nano config.yml
```

Paste this YAML:

```yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4

nodes:
  - role: control-plane
    image: kindest/node:v1.30.0
  - role: worker
    image: kindest/node:v1.30.0
  - role: worker
    image: kindest/node:v1.30.0
```

Create the cluster:

```bash
kind create cluster --config=config.yml
```

Verify the cluster:

```bash
kubectl cluster-info --context kind-kind
kubectl get nodes
kind get clusters
```

You should see one control-plane node and two worker nodes.

## Step 8: Install Argo CD 🔄

Argo CD will be installed inside a namespace called `argocd`.

Create the namespace:

```bash
kubectl create namespace argocd
```

Install Argo CD:

```bash
kubectl apply -n argocd --server-side --force-conflicts -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

Check the Argo CD pods and services:

```bash
kubectl get pods -n argocd
kubectl get svc -n argocd
```

Wait until the pods are in the `Running` state.

## Step 9: Access the Argo CD UI 🌐

Forward the Argo CD service to port `8443`:

```bash
kubectl port-forward -n argocd service/argocd-server 8443:443 --address=0.0.0.0 &
```

Open Argo CD in your browser:

```text
https://<EC2_PUBLIC_IP>:8443
```

Your browser may show a certificate warning because Argo CD uses a self-signed certificate by default. For this learning setup, you can continue to the site.

Login details:

| Field | Value |
| --- | --- |
| Username | `admin` |
| Password | Use the command below |

Get the initial admin password:

```bash
kubectl get secret -n argocd argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d && echo
```

> After logging in, change the default password from the Argo CD UI.

## Step 10: Create the Voting App in Argo CD 🗳️

In the Argo CD UI, click **New App** and use the following values.

### General

| Field | Value |
| --- | --- |
| Application Name | `voting-app` |
| Project Name | `default` |
| Sync Policy | `Automatic` |

Enable these options:

- `Prune Resources`
- `Self Heal`

### Source

| Field | Value |
| --- | --- |
| Repository URL | `https://github.com/agravi987/k8s-kind-voting-app` |
| Revision | `main` |
| Path | `k8s_specification` |

### Destination

| Field | Value |
| --- | --- |
| Cluster URL | `https://kubernetes.default.svc` |
| Namespace | `default` |

Click **Create**.

Argo CD will now sync the Kubernetes manifests from GitHub into your KIND cluster.

## Step 11: Verify the Application ✅

Run these commands to check whether the application is running:

```bash
kubectl get pods
kubectl get deployments
kubectl get services
```

You should see services such as `vote`, `result`, `redis`, `db`, and `worker`, depending on the manifests in your repository.

## Step 12: Access the Voting App 🧪

Forward the voting service:

```bash
kubectl port-forward service/vote 5000:5000 --address=0.0.0.0 &
```

Open the voting page:

```text
http://<EC2_PUBLIC_IP>:5000
```

Forward the result service:

```bash
kubectl port-forward service/result 5001:5001 --address=0.0.0.0 &
```

Open the result page:

```text
http://<EC2_PUBLIC_IP>:5001
```

> Make sure ports `5000` and `5001` are allowed in the EC2 security group.

## Optional: Install Kubernetes Dashboard 📊

Kubernetes Dashboard is optional. The official Kubernetes documentation now marks it as deprecated and unmaintained, so use it only for learning.

Install Helm first:

```bash
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
chmod 700 get_helm.sh
./get_helm.sh
helm version
```

Install Kubernetes Dashboard using Helm:

```bash
helm repo remove kubernetes-dashboard 2>/dev/null || true
helm repo add kubernetes-dashboard https://kubernetes-retired.github.io/dashboard/
helm repo update
helm upgrade --install kubernetes-dashboard kubernetes-dashboard/kubernetes-dashboard --create-namespace --namespace kubernetes-dashboard
```

Create an admin user manifest:

```bash
nano dashboard-adminuser.yml
```

Paste this content:

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: admin-user
  namespace: kubernetes-dashboard
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: admin-user
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
  - kind: ServiceAccount
    name: admin-user
    namespace: kubernetes-dashboard
```

Apply the manifest:

```bash
kubectl apply -f dashboard-adminuser.yml
```

Create a login token:

```bash
kubectl -n kubernetes-dashboard create token admin-user
```

Forward the Dashboard service:

```bash
kubectl -n kubernetes-dashboard port-forward svc/kubernetes-dashboard-kong-proxy 8080:443 --address=0.0.0.0 &
```

Open Dashboard in your browser:

```text
https://<EC2_PUBLIC_IP>:8080
```

Paste the generated token on the Dashboard login page.

> The `admin-user` account has full cluster-admin access. Use it only for practice.

## Optional: Install Prometheus and Grafana

Prometheus collects metrics from the cluster, and Grafana gives you dashboards to view those metrics. The easiest learning setup is the `kube-prometheus-stack` Helm chart, which installs Prometheus Operator, Prometheus, Alertmanager, Grafana, node-exporter, kube-state-metrics, default alerts, and Kubernetes dashboards.

> This stack uses more CPU and memory than the voting app. A `t2.medium` can work for practice, but if pods stay pending or restart often, use a larger EC2 instance such as `t3.medium` or higher.

Add the Prometheus Community Helm repository:

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
```

Install the monitoring stack into a namespace called `monitoring`:

```bash
helm upgrade --install kube-prometheus-stack prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --create-namespace
```

Check the monitoring pods and services:

```bash
kubectl get pods -n monitoring
kubectl get svc -n monitoring
```

Wait until the main pods are `Running`. You should see Grafana, Prometheus, Alertmanager, kube-state-metrics, node-exporter, and Prometheus Operator resources.

### Access Grafana

Get the Grafana admin password:

```bash
kubectl get secret -n monitoring kube-prometheus-stack-grafana \
  -o jsonpath="{.data.admin-password}" | base64 -d && echo
```

Forward Grafana to port `3000`:

```bash
kubectl -n monitoring port-forward svc/kube-prometheus-stack-grafana 3000:80 --address=0.0.0.0 &
```

Open Grafana in your browser:

```text
http://<EC2_PUBLIC_IP>:3000
```

Login details:

| Field | Value |
| --- | --- |
| Username | `admin` |
| Password | Use the password from the secret command |

After login, go to **Dashboards** and browse the prebuilt Kubernetes dashboards. Useful dashboards include node, namespace, pod, workload, and API server views.

### Access Prometheus

Forward Prometheus to port `9090`:

```bash
kubectl -n monitoring port-forward svc/kube-prometheus-stack-prometheus 9090:9090 --address=0.0.0.0 &
```

Open Prometheus in your browser:

```text
http://<EC2_PUBLIC_IP>:9090
```

Useful places to check:

- **Status > Targets** shows what Prometheus is scraping.
- **Graph** lets you run PromQL queries.

Example PromQL queries:

```text
up
kube_pod_info
container_cpu_usage_seconds_total
```

### Access Alertmanager

Alertmanager is optional for this project, but you can open it if you want to view alert status.

```bash
kubectl -n monitoring port-forward svc/kube-prometheus-stack-alertmanager 9093:9093 --address=0.0.0.0 &
```

Open:

```text
http://<EC2_PUBLIC_IP>:9093
```

### Monitor the Voting App

After the voting app is deployed, check that Prometheus can see Kubernetes objects from the `default` namespace:

```bash
kubectl get pods -n default
kubectl get svc -n default
```

In Grafana, open the Kubernetes dashboards and filter by namespace `default`. You can watch pod health, CPU usage, memory usage, restarts, and workload status for the voting app.

### Remove Prometheus and Grafana

If you want to remove the monitoring stack:

```bash
helm uninstall kube-prometheus-stack -n monitoring
kubectl delete namespace monitoring
```

The chart installs monitoring CRDs. For a learning cluster, deleting the KIND cluster also removes everything:

```bash
kind delete cluster
```

## Useful Troubleshooting Commands 🛠️

Check all pods:

```bash
kubectl get pods -A
```

Describe a failing pod:

```bash
kubectl describe pod <POD_NAME>
```

Check pod logs:

```bash
kubectl logs <POD_NAME>
```

Check Argo CD application resources:

```bash
kubectl get applications -n argocd
```

Check running port-forward processes:

```bash
ps aux | grep port-forward
```

Stop a port-forward process:

```bash
kill <PROCESS_ID>
```

Delete the KIND cluster when you are done:

```bash
kind delete cluster
```

## Common Errors and Fixes 💡

| Problem | Reason | Fix |
| --- | --- | --- |
| `docker ps` gives permission error | User is not active in Docker group | Run `newgrp docker` or log out and log in again |
| Argo CD page does not open | Port forwarding or security group issue | Check port `8443` and keep the port-forward command running |
| Voting app page does not open | Port `5000` is blocked | Allow port `5000` in the EC2 security group |
| Result page does not open | Port `5001` is blocked | Allow port `5001` in the EC2 security group |
| Argo CD app is not syncing | Wrong repo URL, branch, or manifest path | Recheck repository URL, revision, and path |
| Dashboard Helm repo gives `404 Not Found` | Old Dashboard chart repo was moved after retirement | Use `https://kubernetes-retired.github.io/dashboard/` |
| Dashboard token does not work | Dashboard needs HTTPS token login | Open the Dashboard URL with `https://` |
| Grafana page does not open | Port forwarding or security group issue | Check port `3000` and keep the port-forward command running |
| Prometheus page does not open | Port forwarding or security group issue | Check port `9090` and keep the port-forward command running |
| Monitoring pods stay pending | EC2 instance does not have enough resources | Use a larger instance or delete unused workloads |

## Reference Links 🔗

- KIND documentation: <https://kind.sigs.k8s.io/docs/user/quick-start/>
- kubectl installation: <https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/>
- Argo CD getting started: <https://argo-cd.readthedocs.io/en/stable/getting_started/>
- Kubernetes Dashboard: <https://kubernetes.io/docs/tasks/access-application-cluster/web-ui-dashboard/>
- Helm installation: <https://helm.sh/docs/intro/install/>
- Prometheus Community Helm Charts: <https://github.com/prometheus-community/helm-charts>
- kube-prometheus-stack chart: <https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack>

## Final Result 🎉

You now have a complete GitOps workflow:

1. Code and Kubernetes manifests are stored in GitHub.
2. Argo CD watches the repository.
3. Argo CD deploys the voting app into the KIND Kubernetes cluster.
4. You access the vote and result services from your browser.
5. Prometheus and Grafana can monitor the cluster and voting app.

This is a strong beginner-friendly DevOps project because it covers EC2, Docker, Kubernetes, KIND, Argo CD, GitOps, services, deployments, monitoring, dashboards, and port forwarding in one practical setup.
