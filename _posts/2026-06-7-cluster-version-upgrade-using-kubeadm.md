---
layout: post
title: "Cluster Version Upgrade Using kubeadm"
date: 2026-06-7
categories: [k8s]
---

## Introduction

To upgrade a Kubernetes cluster, two key concepts are involved.

- Node Drain
- Node Uncordon

**Node draining** is the method of safely removing all pods from a node, getting it ready for maintenance tasks like upgrades or shutdowns.

When you initiate a node drain, Kubernetes automatically evicts and reschedules the regular application pods running on that node to other available nodes in the cluster.

> Node drain operation, does not evict the control plane pods running on the node (also known as system pods).

**Uncordoning** is the process of marking a node as schedulable after it was previously drained.

### Cluster Upgrade Using Kubeadm

- **Control Plane**: Upgrade kubeadm, control plane components, kubelet, and kubectl
- **Worker Nodes**: Upgrade kubeadm and kubelet.

![Cluster Upgrade Using Kubeadm]({{ site.baseurl }}/assets/img/k8s-course/cluster-upgrade-kubeadm.png)

### Perform a Version upgrade on Control Plane

We will upgrade the following on the control plane node:

- Control plane components (kube-apiserver, etcd, kube-scheduler, kube-controller-manager)
- kubelet
- kubectl

The upgrade process, on **control-plane** components, involves the following steps:

- Update your linux system
- **Unhold** kubeadm package.
- Check for the latest available version for kubeadm.
- Install the latest **kubeadm** version.
- Verify the upgraded version.
- Check if you can upgrade the cluster to the latest version, using kubeadm upgrade plan.
- Apply kubeadm upgrade.
- **Hold** the kubeadm package.
- Verify the upgrade.

- Upgrade **kubelet** & **kubectl**.
- **Drain** the control-plane node.
- **Unhold** kubelet & kubectl.
- Update Linux system.
- Install the same version used for kubeadm.
- **Hold** kubelet & kubeadm.
- Restart kubelet **systemd** service.
- **Uncordon the control-plane node so it can become schedulable.

Now, lets apply each step:

```bash
vagrant@controlplane:~$ sudo apt update -y
```

Unhold kubeadm package.

```bash
vagrant@controlplane:~$ sudo apt-mark unhold kubeadm
Canceled hold on kubeadm.
```

> A hold is a status that prevents the package from being automatically installed, upgraded, or removed.

Check for latest available versions:

```bash
vagrant@controlplane:~$ sudo apt-cache madison kubeadm | tac
   kubeadm | 1.34.0-1.1 | https://pkgs.k8s.io/core:/stable:/v1.34/deb  Packages
   kubeadm | 1.34.1-1.1 | https://pkgs.k8s.io/core:/stable:/v1.34/deb  Packages
   kubeadm | 1.34.2-1.1 | https://pkgs.k8s.io/core:/stable:/v1.34/deb  Packages
   kubeadm | 1.34.3-1.1 | https://pkgs.k8s.io/core:/stable:/v1.34/deb  Packages
   kubeadm | 1.34.4-1.1 | https://pkgs.k8s.io/core:/stable:/v1.34/deb  Packages
   kubeadm | 1.34.5-1.1 | https://pkgs.k8s.io/core:/stable:/v1.34/deb  Packages
   kubeadm | 1.34.6-1.1 | https://pkgs.k8s.io/core:/stable:/v1.34/deb  Packages
   kubeadm | 1.34.7-1.1 | https://pkgs.k8s.io/core:/stable:/v1.34/deb  Packages
   kubeadm | 1.34.8-1.1 | https://pkgs.k8s.io/core:/stable:/v1.34/deb  Packages
```

Next, install the latest version which is **1.34.8-1.1**.

```bash
vagrant@controlplane:~$ sudo apt install -y kubeadm=1.34.8-1.1
```

Once, finished verify the upgrade:

```bash
vagrant@controlplane:~$ kubeadm version -o json
{
  "clientVersion": {
    "major": "1",
    "minor": "34",
    "gitVersion": "v1.34.8",
    "gitCommit": "1f328c5e9dd683d0c5e69f3d7d58f8371278dec2",
    "gitTreeState": "clean",
    "buildDate": "2026-05-12T09:52:10Z",
    "goVersion": "go1.25.9",
    "compiler": "gc",
    "platform": "linux/amd64"
  }
}
```

Next, check the upgrade plane.

```bash
vagrant@controlplane:~$ sudo kubeadm upgrade plan
[preflight] Running pre-flight checks.
[upgrade/config] Reading configuration from the "kubeadm-config" ConfigMap in namespace "kube-system"...
[upgrade/config] Use 'kubeadm init phase upload-config kubeadm --config your-config-file' to re-upload it.
[upgrade] Running cluster health checks
[upgrade] Fetching available versions to upgrade to
[upgrade/versions] Cluster version: 1.34.0
[upgrade/versions] kubeadm version: v1.34.8
I0607 22:24:09.824509   16418 version.go:260] remote version is much newer: v1.36.1; falling back to: stable-1.34
[upgrade/versions] Target version: v1.34.8
[upgrade/versions] Latest version in the v1.34 series: v1.34.8

Components that must be upgraded manually after you have upgraded the control plane with 'kubeadm upgrade apply':
COMPONENT   NODE           CURRENT   TARGET
kubelet     controlplane   v1.34.0   v1.34.8
kubelet     node01         v1.34.0   v1.34.8
kubelet     node02         v1.34.0   v1.34.8

Upgrade to the latest version in the v1.34 series:

COMPONENT                 NODE           CURRENT   TARGET
kube-apiserver            controlplane   v1.34.0   v1.34.8
kube-controller-manager   controlplane   v1.34.0   v1.34.8
kube-scheduler            controlplane   v1.34.0   v1.34.8
kube-proxy                               1.34.0    v1.34.8
CoreDNS                                  v1.12.1   v1.12.1
etcd                      controlplane   3.6.4-0   3.6.5-0

You can now apply the upgrade by executing the following command:

        kubeadm upgrade apply v1.34.8

_____________________________________________________________________


The table below shows the current state of component configs as understood by this version of kubeadm.
Configs that have a "yes" mark in the "MANUAL UPGRADE REQUIRED" column require manual config upgrade or
resetting to kubeadm defaults before a successful upgrade can be performed. The version to manually
upgrade to is denoted in the "PREFERRED VERSION" column.

API GROUP                 CURRENT VERSION   PREFERRED VERSION   MANUAL UPGRADE REQUIRED
kubeproxy.config.k8s.io   v1alpha1          v1alpha1            no
kubelet.config.k8s.io     v1beta1           v1beta1             no
_____________________________________________________________________
```

> **upgrade plane** checks whether your K8s cluster can be upgraded and lists all stable versions available for your next upgrade

As you may notice, the command & version of the kubeadm to be applied on the cluster is given.

Now, lets apply **kubeadm** upgrade.

```bash
vagrant@controlplane:~$ sudo kubeadm upgrade apply v1.34.8

# [upgrade] SUCCESS! A control plane node of your cluster was upgraded to "v1.34.8".

# [upgrade] Now please proceed with upgrading the rest of the nodes by following the right order.
```

When you apply the **upgrade**, kubeadm checks the role of the node (whether it's a control plane node or a worker node). If the node is a controlplane node, it upgrades the necessary control plane components like kube-apiserver, kube-controller-manager, and kube-scheduler to the new version.

Once the upgrade is completed, we **hold** the kubeadm package to prevent upgrades.

```bash
vagrant@controlplane:~$ sudo apt-mark hold kubeadm
kubeadm set on hold.
```

Now, lets verify the upgrade.

```bash
vagrant@controlplane:~$ kubeadm version -o json
{
  "clientVersion": {
    "major": "1",
    "minor": "34",
    "gitVersion": "v1.34.8",
    "gitCommit": "1f328c5e9dd683d0c5e69f3d7d58f8371278dec2",
    "gitTreeState": "clean",
    "buildDate": "2026-05-12T09:52:10Z",
    "goVersion": "go1.25.9",
    "compiler": "gc",
    "platform": "linux/amd64"
  }
}
```

---

### Upgrade Kubelet and Kubectl

```bash
vagrant@controlplane:~$ kubectl get nodes
NAME           STATUS     ROLES           AGE    VERSION
controlplane   Ready      control-plane   4d2h   v1.34.0
node01         Ready      worker          4d1h   v1.34.0
node02         NotReady   worker          4d1h   v1.34.0
```

**Drain** the **control plane** node using the node name and make it unschedulable.

```bash
vagrant@controlplane:~$ kubectl drain controlplane --ignore-daemonsets
node/controlplane already cordoned
Warning: ignoring DaemonSet-managed Pods: calico-system/calico-node-j5j48, calico-system/csi-node-driver-4cmnl, kube-system/kube-proxy-qnrl8
evicting pod calico-system/calico-apiserver-986dd8459-lgpdk
pod/calico-apiserver-986dd8459-lgpdk evicted
node/controlplane drained
```

---

If you face an error while draining the controlplane like **PodDisruptionBudget** (PDB) protection, you have to first **delete** PDB first.

> PDB is a K8s resource that requires a minimum number of replicas to stay online.

```bash
vagrant@controlplane:~$ kubectl get pdb -A
NAMESPACE       NAME               MIN AVAILABLE   MAX UNAVAILABLE   ALLOWED DISRUPTIONS   AGE
calico-system   calico-apiserver   N/A             1                 1                     4d11h
calico-system   calico-typha       N/A             1                 1                     4d11h
```

Lets delete calico-apiserver & calico-typha **PDB**s.

```bash
vagrant@controlplane:~$ kubectl delete pdb calico-apiserver -n calico-system
poddisruptionbudget.policy "calico-apiserver" deleted from calico-system namespace
```

---

Next, we unhold the **kubelet** and **kubectl**.

```bash
vagrant@controlplane:~$ sudo apt-mark unhold kubelet kubectl
Canceled hold on kubelet.
Canceled hold on kubectl.
```

Update the system and install **kubelet** and **kubectl** using the same version you used for **kubeadm**.

```bash
vagrant@controlplane:~$ sudo apt install kubelet=1.34.8-1.1 kubectl=1.34.8-1.1
```

**Hold** the kubelet and kubectl packages.

```bash
vagrant@controlplane:~$ sudo apt-mark hold kubelet kubectl
kubelet set on hold.
kubectl set on hold.
```

Restart kubelet **systemd** service using the following commands.

```bash
vagrant@controlplane:~$ sudo systemctl daemon-reload
vagrant@controlplane:~$ sudo systemctl restart kubelet
```

Finally, **uncordon** the node so that the control plane becomes schedulable.

```bash
vagrant@controlplane:~$ kubectl uncordon controlplane
node/controlplane uncordoned
```

---

### Perform a Version upgrade on Worker Nodes

Now lets upgrade the kubeadm and kubelet on worker nodes.

> Run all the **kubectl** commands from the control-plane node. Other commands have to be run on the worker nodes.

```bash
vagrant@controlplane:~$ kubectl get nodes
NAME           STATUS   ROLES           AGE     VERSION
controlplane   Ready    control-plane   4d13h   v1.34.8
node01         Ready    worker          4d12h   v1.34.0
node02         Ready    worker          4d12h   v1.34.0
```

As you may notice, **node01** and **node02** worker nodes version is **1.34.0**.

We pretty much apply the same process for worker node upgrade as we did for the controlplane node.

```bash
vagrant@node01:~$ sudo apt update -y
vagrant@node01:~$ sudo apt-mark unhold kubeadm
Canceled hold on kubeadm.
```

Next, upgrade kubeadm on the node

```bash
vagrant@node01:~$ sudo kubeadm upgrade node

# [upgrade/kubelet-config] The kubelet configuration for this node was successfully upgraded!
# [upgrade/addon] Skipping the addon/coredns phase. Not a control plane node.
# [upgrade/addon] Skipping the addon/kube-proxy phase. Not a control plane node.
```

Next, we will upgrade **kubelet*** on worker node. First, drain the worker node.

```bash
vagrant@controlplane:~$ kubectl drain node01 --ignore-daemonsets --delete-emptydir-data

# node/node01 drained
```

> During **CKA** exam, you dont have to use the **--delete-emptydir-data** flag.

Next, unhold kubelet on the worker node.

```bash
vagrant@node01:~$ sudo apt-mark unhold kubelet
Canceled hold on kubelet.
```

Next, update the system and install **kubelet** and **hold** the packages. Replace **1.34.8-1.1** with the required version.

```bash
vagrant@node01:~$ sudo apt update -y
vagrant@node01:~$ sudo apt install kubelet=1.34.8-1.1
vagrant@node01:~$ sudo apt-mark hold kubelet
# kubelet set on hold.
```

Next, restart the **kubelet** service.

```bash
vagrant@node01:~$ sudo systemctl daemon-reload
vagrant@node01:~$ sudo systemctl restart kubelet
```

Finally, **uncordon** node01 for scheduling.

```bash
vagrant@controlplane:~$ kubectl uncordon node01
# node/node01 uncordoned
```

Now, lets verify the upgrade.

```bash
vagrant@controlplane:~$ kubectl get nodes
NAME           STATUS   ROLES           AGE     VERSION
controlplane   Ready    control-plane   4d14h   v1.34.8
node01         Ready    worker          4d12h   v1.34.8
node02         Ready    worker          4d12h   v1.34.0
```

As you may notice, node01 has been upgrade to **v1.34.8**.

You have to apply the upgrade to all worker nodes separately. Follow the same steps applied to the **node01**.

---
