---
layout: post
title: "Kubeadm Cluster Setup"
date: 2026-06-4
categories: [k8s]
---

## Introduction

**kubeadm** is a K8s tool that bootstraps a production‑grade cluster by installing and configuring the control plane components (API server, controller manager, scheduler, etc.) and joining worker nodes.

> **kubeadm** the “official” K8s installer for building clusters on any Linux machines.
---
> Without **kubeadm**, you would build K8s by hand — component by component, config by config. **kubeadm** automates all of that so you can focus on using the cluster, not assembling it.

## Kubeadm Cluster Prerequisites

All the K8s cluster setup is defined in our **settings.yaml** file.

```bash
box_name: "bento/ubuntu-22.04"
vm:
- name: "controlplane"
  ip: "192.168.201.10"
  memory: "2048"
  cpus: "2"
- name: "node01"
  ip: "192.168.201.11"
  memory: "2048"
  cpus: "1"
- name: "node02"
  ip: "192.168.201.12"
  memory: "2048"
  cpus: "1"
```

### Provision Underlying Infrastructure to Deploy a K8s Cluster

**Important Note:** Performed the following setup on all nodes (both control plane and worker nodes).

Open PowerShell or Windows Terminal and login to each virtual machine using **vagrant ssh**.

```bash
C:\Users\ZAFAR\Desktop\cka-certification-guide\lab-setup\windows> vagrant ssh controlplane

vagrant@controlplane:~$ sudo su
root@controlplane:/home/vagrant#
```

Create a file with **common.sh** name in the home directory or each node (controlplane & worker nodes).

If your system does not have **nano**, install it using the following command.

```bash
sudo apt update -y
sudo apt install nano
```

```bash
nano common.sh
```

Enter the follwoing script into the **common.sh** file. It contains all the necessary commands to install kubeadm, Containerd, kubelet, and kubectl.

- Linux kernel preparation
- Networking preparation
- Container runtime installation (containerd)
- CRI tooling installation (crictl)
- Kubernetes components installation (kubelet, kubeadm, kubectl)
- Node-specific kubelet configuration

### The Big Picture

The script inside the **common.sh** does the following:

```bash
Layer 1: Linux Kernel
--------------------------------
swapoff
overlay
br_netfilter
sysctl

Layer 2: Container Runtime
--------------------------------
containerd
runc
config.toml
SystemdCgroup

Layer 3: CRI Tools
--------------------------------
crictl
containerd.sock

Layer 4: Kubernetes
--------------------------------
kubeadm
kubelet
kubectl
```

```bash
#!/bin/bash

###############################################################################
# Kubernetes Node Preparation Script
#
# This script prepares an Ubuntu machine to become either:
#   - A Kubernetes Control Plane node
#   - A Kubernetes Worker node
#
# It performs the following tasks:
#   1. Disables swap
#   2. Configures Linux kernel modules and networking
#   3. Installs containerd runtime
#   4. Installs crictl (CRI troubleshooting tool)
#   5. Installs kubeadm, kubelet, and kubectl
#   6. Configures kubelet with the correct node IP
###############################################################################

# Exit immediately if:
# -e : a command fails
# -u : an undefined variable is used
# -x : print commands as they execute
# -o pipefail : any command failure inside a pipe causes failure
set -euxo pipefail

###############################################################################
# Version Variables
###############################################################################

# Kubernetes repository version
KUBERNETES_VERSION="v1.34"

# crictl version
CRICTL_VERSION="v1.34.0"

# Exact package version for kubeadm, kubelet and kubectl
KUBERNETES_INSTALL_VERSION="1.34.0-1.1"

###############################################################################
# Disable Swap
###############################################################################

# Kubernetes requires swap to be disabled because kubelet needs accurate
# memory accounting and resource management.
sudo swapoff -a

# Ensure swap remains disabled after reboot.
# This adds a cron job that runs during every system startup.
(crontab -l 2>/dev/null; echo "@reboot /sbin/swapoff -a") | crontab - || true

sudo apt-get update -y

###############################################################################
# Load Required Kernel Modules
###############################################################################

# Create a configuration file so these modules load automatically on boot.
#
# overlay:
#   Required by container runtimes for layered container filesystems.
#
# br_netfilter:
#   Allows bridged network traffic to pass through iptables rules.
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

# Load the modules immediately without rebooting.
sudo modprobe overlay
sudo modprobe br_netfilter

###############################################################################
# Configure Kubernetes Networking Requirements
###############################################################################

# Create kernel networking settings.
#
# bridge-nf-call-iptables:
#   Ensures bridge traffic is processed by iptables.
#
# bridge-nf-call-ip6tables:
#   Same behavior for IPv6 traffic.
#
# ip_forward:
#   Allows Linux to route packets between interfaces.
#   Required for pod-to-pod communication.
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

# Apply all sysctl settings immediately.
sudo sysctl --system

###############################################################################
# Install Basic Utilities
###############################################################################

sudo apt-get update -y

# apt-transport-https:
#   Allows APT to use HTTPS repositories.
#
# ca-certificates:
#   Trusted certificate authorities.
#
# curl:
#   Download files from the internet.
#
# gpg:
#   Verify package signatures.
sudo apt-get install -y \
    apt-transport-https \
    ca-certificates \
    curl \
    gpg

###############################################################################
# Install Container Runtime (containerd)
###############################################################################

sudo apt-get update -y

# Additional package dependencies.
sudo apt-get install -y \
    software-properties-common \
    curl \
    apt-transport-https \
    ca-certificates

# Create directory for repository signing keys.
sudo install -m 0755 -d /etc/apt/keyrings

###############################################################################
# Add Docker Repository
#
# We install containerd from Docker's official repository.
###############################################################################

# Download Docker repository GPG key.
curl -fsSL https://download.docker.com/linux/ubuntu/gpg \
| sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg

# Make key readable by apt.
sudo chmod a+r /etc/apt/keyrings/docker.gpg

# Add Docker repository to apt sources.
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo "$VERSION_CODENAME") stable" \
| sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt-get update

###############################################################################
# Install Containerd
###############################################################################

# Containerd is the container runtime used by Kubernetes.
#
# kubelet
#   -> containerd
#       -> runc
#           -> Linux kernel
sudo apt-get install -y containerd.io

# Reload systemd configuration.
sudo systemctl daemon-reload

# Enable containerd at boot and start it immediately.
sudo systemctl enable containerd --now

# Ensure service is running.
sudo systemctl start containerd.service

echo "Containerd runtime installed successfully"

###############################################################################
# Configure Containerd
###############################################################################

# Generate default configuration file.
sudo containerd config default \
| sudo tee /etc/containerd/config.toml

# Kubernetes and containerd should use the same cgroup driver.
#
# Modern Kubernetes recommends:
#   systemd cgroup driver
#
# Change:
#   SystemdCgroup = false
#
# To:
#   SystemdCgroup = true
sudo sed -i \
's/SystemdCgroup = false/SystemdCgroup = true/g' \
/etc/containerd/config.toml

# Restart containerd to apply configuration.
sudo systemctl restart containerd

###############################################################################
# Detect CPU Architecture
###############################################################################

ARCH="$(dpkg --print-architecture)"

case "$ARCH" in
  amd64)
    CRICTL_ARCH="amd64"
    ;;
  arm64)
    CRICTL_ARCH="arm64"
    ;;
  *)
    echo "Unsupported architecture: $ARCH"
    exit 1
    ;;
esac

###############################################################################
# Install crictl
#
# crictl is a troubleshooting utility for CRI-compatible runtimes.
#
# Examples:
#   crictl ps
#   crictl images
#   crictl logs
###############################################################################

curl -LO \
"https://github.com/kubernetes-sigs/cri-tools/releases/download/${CRICTL_VERSION}/crictl-${CRICTL_VERSION}-linux-${CRICTL_ARCH}.tar.gz"

sudo tar zxvf \
"crictl-${CRICTL_VERSION}-linux-${CRICTL_ARCH}.tar.gz" \
-C /usr/local/bin

rm -f \
"crictl-${CRICTL_VERSION}-linux-${CRICTL_ARCH}.tar.gz"

###############################################################################
# Configure crictl
###############################################################################

# Point crictl to the containerd socket.
#
# containerd listens on:
#   /run/containerd/containerd.sock
#
# crictl communicates with containerd through this socket.
cat <<EOF | sudo tee /etc/crictl.yaml
runtime-endpoint: unix:///run/containerd/containerd.sock
image-endpoint: unix:///run/containerd/containerd.sock
timeout: 10
debug: false
EOF

echo "crictl installed and configured successfully"

###############################################################################
# Install Kubernetes Components
###############################################################################

# Download Kubernetes repository signing key.
curl -fsSL \
https://pkgs.k8s.io/core:/stable:/$KUBERNETES_VERSION/deb/Release.key \
| gpg --dearmor \
-o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

# Add Kubernetes repository.
echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/$KUBERNETES_VERSION/deb/ /" \
| tee /etc/apt/sources.list.d/kubernetes.list

sudo apt-get update -y

###############################################################################
# Install kubeadm, kubelet and kubectl
#
# kubeadm  -> cluster bootstrap tool
# kubelet  -> node agent
# kubectl  -> cluster command-line client
###############################################################################

sudo apt-get install -y \
    kubelet="$KUBERNETES_INSTALL_VERSION" \
    kubectl="$KUBERNETES_INSTALL_VERSION" \
    kubeadm="$KUBERNETES_INSTALL_VERSION"

###############################################################################
# Prevent Automatic Upgrades
###############################################################################

# Avoid accidental version mismatches between nodes.
sudo apt-mark hold kubelet kubeadm kubectl

sudo apt-get update -y

###############################################################################
# Install jq
#
# jq is a JSON parsing utility used below to extract the node IP.
###############################################################################

sudo apt-get install -y jq

###############################################################################
# Determine Node IP Address
###############################################################################

# In most Vagrant Kubernetes labs:
#
# eth0 = NAT interface
# eth1 = Private cluster network
#
# We want kubelet to advertise the cluster network IP.
local_ip="$(
ip --json addr show eth1 \
| jq -r '.[0].addr_info[] | select(.family == "inet") | .local'
)"

###############################################################################
# Configure Kubelet Node IP
###############################################################################

# Force kubelet to use the desired node IP.
#
# Without this, kubelet may choose the wrong interface,
# causing node communication issues.
cat > /etc/default/kubelet << EOF
KUBELET_EXTRA_ARGS=--node-ip=$local_ip
EOF

###############################################################################
# End Result
#
# This node is now prepared for Kubernetes.
#
# Control Plane:
#   kubeadm init ...
#
# Worker Node:
#   kubeadm join ...
#
###############################################################################
```

Add execute permissions to the script. Then, execute the script.

```bash
vagrant@controlplane:~$ chmod +x common.sh
```

```bash
vagrant@controlplane:~$ ./common.sh
```

### Create Kubeadm Config

Now that we have the nodes ready with all the utilities for kubernetes, we will initialize the control plane. We will be using **K8s version v1.34**.

> To access the controlplane from you local machine, give the controlplane a static IP, otherwise, the public IP will change on VM restart.

To set a static IP, open the **kubeadm.config** file.

```bash
vagrant@controlplane:~$ nano kubeadm.config
```

Replace the **advertiseAddress** and **controlPlaneEndpoint** with the controlplane's public IP. If you don't want public access, replace it with the private IP of the controlplane.

```bash
apiVersion: kubeadm.k8s.io/v1beta4
kind: InitConfiguration
localAPIEndpoint:
  advertiseAddress: "192.168.201.10"    # Replace with the controlplane IP
  bindPort: 6443
nodeRegistration:
  name: "controlplane"

---
apiVersion: kubeadm.k8s.io/v1beta4
kind: ClusterConfiguration
kubernetesVersion: "v1.34.0"
controlPlaneEndpoint: "192.168.201.10:6443" # Replace with the controlplane IP
```

### Initialize Kubeadm Cluser

To initialize the cluster run the following command.

```bash
sudo kubeadm init --config=kubeadm.config
```

Upon successful execution, it will provide three command section.

- Command to copy the admin kubeconfig file.
- Command to join other control plane nodes (not required for this guide).
- Command to join the worker nodes

```text
Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:
```

```bash
# Run the following command on the control plane node to copy the admin kubeconfig file to the home directory. This enables you to use the **kubectl** command to interact with the cluster.

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

```text
You can now join any number of control-plane nodes by copying certificate authorities
and service account keys on each node and then running the following as root:
```

```bash
    kubeadm join 192.168.201.10:6443 --token 9rr2xa.w9zlny1bj4un976k \
        --discovery-token-ca-cert-hash sha256:3c49eb334cb25011e63a1d339ae8e7a0d579bd928d4ef30b9fb20439a5fa5dbb \
        --control-plane
```

```text
Then you can join any number of worker nodes by running the following on each as root:
```

```bash
kubeadm join 192.168.201.10:6443 --token 9rr2xa.w9zlny1bj4un976k \
        --discovery-token-ca-cert-hash sha256:3c49eb334cb25011e63a1d339ae8e7a0d579bd928d4ef30b9fb20439a5fa5dbb
```

Use the following command to validate the cluster control plane.

```bash
vagrant@controlplane:~$ kubectl get pods --all-namespaces
```

You should see an output similar to the one below, where all control plane pods are in a **Ready** and **Running** state, except for the **coredns** pods.

The **coredns** pods are in a **Pending** state because the cluster needs a Network Plugin (**CNI**) to enable communication between pods.

> Remember that **kubectl** is configured to be used from the **control plane** node for the user under which you copied the admin kubeconfig file. If you copied the file using the root user, kubectl will only work for root.
---
> If you want to use it as a normal user, you need to copy the kubeconfig file to that user's home directory.
> Always run kubectl commands from the control plane node. Running them on other nodes will result in a "connection refused" error as shown below.

### Join Worker Nodes

Run the following command inside the worker nodes to join the cluster.

```bash
vagrant@node01:~$ sudo kubeadm join 192.168.201.10:6443 --token 9rr2xa.w9zlny1bj4un976k --discovery-token-ca-cert-hash sha256:3c49eb334cb25011e63a1d339ae8e7a0d579bd928d4ef30b9fb20439a5fa5dbb
```

If you dont have the join token, then, run the following commands in the controlplane to list or re-create it.

```bash
vagrant@controlplane:~$ kubeadm token list # lists existing tokens

kubeadm token create --print-join-command # creates a new join token
```

Once the nodes have successfully joined the cluster, you verify it by the following command:

```bash
vagrant@controlplane:~$ kubectl get nodes
NAME           STATUS      ROLES           AGE   VERSION
controlplane   NotReady    control-plane   15m   v1.34.0
node01         NotReady    <none>          3m    v1.34.0
node02         NotReady    <none>          45s   v1.34.0
```

You can see that the nodes are in a NotReady state because a **Network Plugin (CNI)** has not been installed in the cluster yet.

> Kubernetes requires the CNI for pod networking. Without it, the nodes cannot communicate properly, which is why they are in a NotReady state.

### Label Worker Nodes

You can add label to the worker nodes using the following command:

```bash
kubectl label node node01  node-role.kubernetes.io/worker=worker
```

### Install the Network Plugin

To enable pod networking, we need to install a **CNI** such as **Calico**. 

First install **Calico Tigera Operator** which is responsible for managing the installation and lifecycle of Calico plugin used for pod networking and network policies.

Execute the following commands:

```bash
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.31.3/manifests/operator-crds.yaml
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.31.3/manifests/tigera-operator.yaml
```

Next, use the following command to download the calico custom resource.

```bash
curl -O https://raw.githubusercontent.com/projectcalico/calico/v3.31.3/manifests/custom-resources.yaml
```

After downloading **custom-resources.yaml** file, we need to adjust the IP pools to match our Pod network settings that we used in the Kubeadm config.

To get the cluster CIDR range, run the following command to get the cluster CIDR range.

```bash
vagrant@controlplane:~$ kubectl -n kube-system get pod -l component=kube-controller-manager -o yaml | grep -i cluster-cidr
      - --cluster-cidr=10.244.0.0/16
```

As you may notice, our current CIDR range is **10.244.0.0/16**, because, this value was used in the **kubeadm.config** during cluster initialization.

Open the **custom-resources.yaml** file and change the default CIDR from **192.168.0.0/16** to **10.244.0.0/16**, which is the value specified in your kubeadm configuration.

```yaml
apiVersion: operator.tigera.io/v1
kind: Installation
metadata:
  name: default
spec:
  # Configures Calico networking.
  calicoNetwork:
    ipPools:
      - name: default-ipv4-ippool
        blockSize: 26
        cidr: 10.244.0.0/16 # Here we have replaced 192.168.0.0/16** with **10.244.0.0/16
        encapsulation: VXLANCrossSubnet
        natOutgoing: Enabled
        nodeSelector: all()
```

After updating the **custom-resources.yaml** file with the correct pod **CIDR**, apply it to the cluster to complete the **Calico** setup.

```bash
kubectl apply -f custom-resources.yaml
```

Now, when you check the pod status, you should see all the pods, including **Calico** and **CoreDNS**, in a Running state.

```bash
vagrant@controlplane:~$ kubectl get pods -A
NAMESPACE         NAME                                      READY   STATUS    RESTARTS      AGE
calico-system     calico-apiserver-986dd8459-f92sq          1/1     Running   1 (71m ago)   22h
calico-system     calico-apiserver-986dd8459-lgpdk          1/1     Running   1 (71m ago)   22h
calico-system     calico-kube-controllers-58cc4895f-5f8z9   1/1     Running   1 (71m ago)   22h
calico-system     calico-node-28vlb                         1/1     Running   1 (68m ago)   22h
calico-system     calico-node-j5j48                         1/1     Running   1 (71m ago)   22h
calico-system     calico-node-q5z82                         1/1     Running   1 (69m ago)   22h
calico-system     calico-typha-6c8c7956fd-k6fsg             1/1     Running   1 (69m ago)   22h
calico-system     calico-typha-6c8c7956fd-lwsqf             1/1     Running   1 (68m ago)   22h
calico-system     csi-node-driver-4cmnl                     2/2     Running   2 (71m ago)   22h
calico-system     csi-node-driver-h96wh                     2/2     Running   2 (68m ago)   22h
calico-system     csi-node-driver-vscfx                     2/2     Running   2 (69m ago)   22h
calico-system     goldmane-8b794fbc6-kqbxq                  1/1     Running   1 (71m ago)   22h
calico-system     whisker-d6f6bb5ff-qgzz4                   2/2     Running   2 (71m ago)   22h
kube-system       coredns-66bc5c9577-6vcdk                  1/1     Running   1 (71m ago)   24h
kube-system       coredns-66bc5c9577-rm42p                  1/1     Running   1 (71m ago)   24h
kube-system       etcd-controlplane                         1/1     Running   1 (71m ago)   24h
kube-system       kube-apiserver-controlplane               1/1     Running   1 (71m ago)   24h
kube-system       kube-controller-manager-controlplane      1/1     Running   4 (71m ago)   24h
kube-system       kube-proxy-2x8jc                          1/1     Running   1 (68m ago)   23h
kube-system       kube-proxy-4z5q5                          1/1     Running   1 (69m ago)   23h
kube-system       kube-proxy-bwlj7                          1/1     Running   1 (71m ago)   24h
kube-system       kube-scheduler-controlplane               1/1     Running   4 (71m ago)   24h
kube-system       metrics-server-6cbd6b9dd4-n57ng           1/1     Running   3 (68m ago)   9h
tigera-operator   tigera-operator-85979684d8-xc5nj          1/1     Running   4 (69m ago)   23h
```

Also, if you check the node status, you will see that all the nodes are now in a Ready state.

```bash
vagrant@controlplane:~$ kubectl get nodes
NAME           STATUS   ROLES           AGE   VERSION
controlplane   Ready    control-plane   24h   v1.34.0
node01         Ready    worker          23h   v1.34.0
node02         Ready    worker          23h   v1.34.0
```

### Deploy Metrics Server

The K8s **Metrics Server** is a lightweight component that collects **CPU** and **memory** usage from each node. It is not a full monitoring solution like **Prometheus**.

- Collects resource metrics (CPU, memory) from kubelets via the /metrics/resource endpoint.
- Aggregates and exposes metrics through the Kubernetes API server.
- Provides quick debugging with:
  - kubectl top nodes
  - kubectl top pods

Deploy the metrics server using the following manifest. It will be installed in the **kube-system** namespace:

```bash
kubectl apply -f https://raw.githubusercontent.com/techiescamp/cka-certification-guide/refs/heads/main/lab-setup/manifests/metrics-server/metrics-server.yaml
```

Validate the metrics server deployment:

```bash
vagrant@controlplane:~$ kubectl get deployment metrics-server -n kube-system
NAME             READY   UP-TO-DATE   AVAILABLE   AGE
metrics-server   1/1     1            1           22h
```

Once the metric-server is up and running, you may check the CPU and memory usage of the nodes.

```bash
vagrant@controlplane:~$ kubectl top nodes
NAME           CPU(cores)   CPU(%)   MEMORY(bytes)   MEMORY(%)
controlplane   248m         12%      1375Mi          73%
node01         28m          2%       914Mi           49%
node02         22m          2%       798Mi           42%
```

```bash
vagrant@controlplane:~$ kubectl top pod -n kube-system
NAME                                   CPU(cores)   MEMORY(bytes)
coredns-66bc5c9577-6vcdk               3m           17Mi
coredns-66bc5c9577-rm42p               2m           18Mi
etcd-controlplane                      48m          58Mi
kube-apiserver-controlplane            86m          374Mi
kube-controller-manager-controlplane   30m          69Mi
kube-proxy-2x8jc                       1m           15Mi
kube-proxy-4z5q5                       1m           16Mi
kube-proxy-bwlj7                       1m           20Mi
kube-scheduler-controlplane            13m          28Mi
metrics-server-6cbd6b9dd4-n57ng        4m           18Mi
```

### Validate the Cluster

Lets look at few methods to validate our cluster.

- Run the following command to verify access to the API server:

```bash
vagrant@controlplane:~$ kubectl cluster-info
Kubernetes control plane is running at https://192.168.201.10:6443
CoreDNS is running at https://192.168.201.10:6443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy
```

- Get the service endpoint details:

```bash
vagrant@controlplane:~$ kubectl get svc -o wide
NAME         TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE   SELECTOR
kubernetes   ClusterIP   10.96.0.1    <none>        443/TCP   25h   <none>
```

- Validate health endpoints: The following command lists individual health checks (e.g., etcd, scheduler, controller-manager) and their status. Response is **ok** and **healthz check passed** output.

```bash
vagrant@controlplane:~$ kubectl get --raw='/healthz?verbose'
[+]ping ok
[+]log ok
[+]etcd ok
[+]poststarthook/start-apiserver-admission-initializer ok
[+]poststarthook/generic-apiserver-start-informers ok
[+]poststarthook/priority-and-fairness-config-consumer ok
[+]poststarthook/priority-and-fairness-filter ok
[+]poststarthook/storage-object-count-tracker-hook ok
[+]poststarthook/start-apiextensions-informers ok
[+]poststarthook/start-apiextensions-controllers ok
[+]poststarthook/crd-informer-synced ok
[+]poststarthook/start-system-namespaces-controller ok
[+]poststarthook/start-cluster-authentication-info-controller ok
[+]poststarthook/start-kube-apiserver-identity-lease-controller ok
[+]poststarthook/start-kube-apiserver-identity-lease-garbage-collector ok
[+]poststarthook/start-legacy-token-tracking-controller ok
[+]poststarthook/start-service-ip-repair-controllers ok
[+]poststarthook/rbac/bootstrap-roles ok
[+]poststarthook/scheduling/bootstrap-system-priority-classes ok
[+]poststarthook/priority-and-fairness-config-producer ok
[+]poststarthook/bootstrap-controller ok
[+]poststarthook/start-kubernetes-service-cidr-controller ok
[+]poststarthook/aggregator-reload-proxy-client-cert ok
[+]poststarthook/start-kube-aggregator-informers ok
[+]poststarthook/apiservice-status-local-available-controller ok
[+]poststarthook/apiservice-status-remote-available-controller ok
[+]poststarthook/apiservice-registration-controller ok
[+]poststarthook/apiservice-discovery-controller ok
[+]poststarthook/kube-apiserver-autoregistration ok
[+]autoregister-completion ok
[+]poststarthook/apiservice-openapi-controller ok
[+]poststarthook/apiservice-openapiv3-controller ok
healthz check passed
```

- Test CoreDNS DNS resolution: To check if **CoreDNS** can resolve internal K8s services, lets deploy a **dnsutils** pod.

```bash
kubectl apply  -f https://raw.githubusercontent.com/techiescamp/cka-certification-guide/refs/heads/main/lab-setup/manifests/utilities/dnsutils.yaml

vagrant@controlplane:~$ kubectl get pods dnsutils
NAME       READY   STATUS    RESTARTS   AGE
dnsutils   1/1     Running   0          63s
```

Execute the following **nslookup** command. Expected output: Resolves to the cluster **IP (10.96.0.1)**.

```bash
vagrant@controlplane:~$ kubectl exec -i -t dnsutils -- nslookup kubernetes.default
Server:         10.96.0.10
Address:        10.96.0.10#53

Name:   kubernetes.default.svc.cluster.local
Address: 10.96.0.1
```

To check if the cluster can resolve external domains, execute:

```bash
vagrant@controlplane:~$ kubectl exec -i -t dnsutils -- nslookup google.com
Server:         10.96.0.10
Address:        10.96.0.10#53

Non-authoritative answer:
Name:   google.com
Address: 192.178.24.46
Name:   google.com
Address: 2a00:1450:4017:820::200e
```

Cleanup dnutils pod.

```bash
vagrant@controlplane:~$ kubectl delete -f https://raw.githubusercontent.com/techiescamp/cka-certification-guide/refs/heads/main/lab-setup/manifests/utilities/dnsutils.yaml

pod "dnsutils" deleted from default namespace
```

### Deploy a Sample Application

Now, lets deploy a sample Nginx application and access it through the browser.

Create the following file in the **home** directory of the controlplane node and name it **nginx.yaml**.

```bash
#nginx.yaml

apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  selector:
    matchLabels:
      app: nginx
  replicas: 2
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: docker.io/library/nginx:latest
        ports:
        - containerPort: 80

---

apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  selector:
    app: nginx
  type: NodePort
  ports:
    - port: 80
      targetPort: 80
      nodePort: 32000
```

Next, apply the **nginx.yaml** file.

```bash
vagrant@controlplane:~$ kubectl apply -f my-apps/nginx.yaml

deployment.apps/nginx-deployment created
service/nginx-service created
```

Lets list the services and get the service endpoint details:

```bash
vagrant@controlplane:~$ kubectl get svc
NAME            TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
kubernetes      ClusterIP   10.96.0.1       <none>        443/TCP        25h
nginx-service   NodePort    10.108.64.109   <none>        80:32000/TCP   7s
```

Get the pod objects and its details:

```bash
vagrant@controlplane:~$ kubectl get pods
NAME                                READY   STATUS    RESTARTS   AGE
nginx-deployment-6694858967-67t6s   1/1     Running   0          3m23s
nginx-deployment-6694858967-798qw   1/1     Running   0          3m23s
```

From the above output, extract the ClusterIP of the Nginx server (**10.108.64.109**) and use **curl** to retrieve the Nginx home page HTML. This will confirm that the ClusterIP is accessible from within the control plane.

```bash
vagrant@controlplane:~$ curl 10.108.64.109
```

```html
<!DOCTYPE html>
<html>
  <head>
    <title>Welcome to nginx!</title>
    <style>
      html { color-scheme: light dark; }
      body { width: 35em; margin: 0 auto;
      font-family: Tahoma, Verdana, Arial, sans-serif; }
    </style>
  </head>
  <body>

    <h1>Welcome to nginx!</h1>
    <p>If you see this page, nginx is successfully installed and working.
    Further configuration is required for the web server, reverse proxy,
    API gateway, load balancer, content cache, or other features.</p>

    <p>For online documentation and support please refer to
    <a href="https://nginx.org/">nginx.org</a>.<br/>
    To engage with the community please visit
    <a href="https://community.nginx.org/">community.nginx.org</a>.<br/>
    For enterprise grade support, professional services, additional
    security features and capabilities please refer to
    <a href="https://f5.com/nginx">f5.com/nginx</a>.</p>

    <p><em>Thank you for using nginx.</em></p>
  </body>
</html>
```

Now you can access the Nginx homepage in your browser using the IP of any worker node on port **32000**.

```bash
http://192.168.201.11:32000/
http://192.168.201.12:32000/
```

- Exec in to Nginx Pod:
  - Now, let's exec into an nginx pod and perform a curl operation internally to verify webpage accessibility.

```bash
vagrant@controlplane:~$ kubectl exec -it $(kubectl get pod -l app=nginx -o jsonpath="{.items[0].metadata.name}") -- sh
```

The command above, allows us to enter inside the container. Once inside, apply the **curl -I http://localhost** command.

```bash
# curl -I http://localhost
HTTP/1.1 200 OK
Server: nginx/1.31.1
Date: Thu, 04 Jun 2026 22:24:14 GMT
Content-Type: text/html
Content-Length: 896
Last-Modified: Fri, 22 May 2026 12:50:47 GMT
Connection: keep-alive
ETag: "6a105127-380"
Accept-Ranges: bytes
```

As you can see, **HTTP** return **200 OK** response.

You may delete **nginx-deployment** and **nginx-service** using the following commands:

```bash
vagrant@controlplane:~$ kubectl delete deployment nginx-deployment
vagrant@controlplane:~$ kubectl delete svc nginx-service
```


