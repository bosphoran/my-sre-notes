---
layout: post
title: "K8s Cluster Configurations"
date: 2026-06-5
categories: [k8s]
---

## Cluster Configurations

When it comes to CKA certification, you will get scenarios to rectify issues in the cluster.

We will look at the key configurations related to control plane components.

### Static Pod Manifests

All the control plane components are started by the kubelet from the static pod manifests present in the **/etc/kubernetes/manifests** directory.

```bash
vagrant@controlplane:~$ tree /etc/kubernetes/manifests/
/etc/kubernetes/manifests/
├── etcd.yaml
├── kube-apiserver.yaml
├── kube-controller-manager.yaml
└── kube-scheduler.yaml

0 directories, 4 files
```

From a **CKA** perspective, if you're tasked with troubleshooting a control plane component and need to modify it, you should edit the respective static pod manifest file to ensure the changes are permanent.

### API Server Configurations

If you want to troubleshoot or verify the cluster components configurations, first you should look at the static API server pod manifest configurations.

```bash
vagrant@controlplane:/etc/kubernetes/manifests$ sudo cat kube-apiserver.yaml
apiVersion: v1
kind: Pod
metadata:
  annotations:
    kubeadm.kubernetes.io/kube-apiserver.advertise-address.endpoint: 192.168.201.10:6443
  labels:
    component: kube-apiserver
    tier: control-plane
  name: kube-apiserver
  namespace: kube-system
spec:
  containers:
  - command:
    - kube-apiserver
    - --advertise-address=192.168.201.10
    - --allow-privileged=true
    - --audit-log-path=/var/log/kubernetes/audit.log
    - --authorization-mode=Node,RBAC
    - --client-ca-file=/etc/kubernetes/pki/ca.crt
    - --enable-admission-plugins=NodeRestriction
    - --enable-bootstrap-token-auth=true
    - --etcd-certfile=/etc/kubernetes/pki/apiserver-etcd-client.crt
    - --etcd-keyfile=/etc/kubernetes/pki/apiserver-etcd-client.key
    .
    .
    .
```

### ETCD Configurations

If you want to **backup etcd** you need to know the **etcd service endpoint** and related **certificates** to authenticate against **etcd** and create a backup.

```bash
vagrant@controlplane:/etc/kubernetes/manifests$ sudo cat etcd.yaml
apiVersion: v1
kind: Pod
metadata:
  annotations:
    kubeadm.kubernetes.io/etcd.advertise-client-urls: https://192.168.201.10:2379
  labels:
    component: etcd
    tier: control-plane
  name: etcd
  namespace: kube-system
spec:
  containers:
  - command:
    - etcd
    - --advertise-client-urls=https://192.168.201.10:2379
    - --cert-file=/etc/kubernetes/pki/etcd/server.crt
    - --client-cert-auth=true
    - --data-dir=/var/lib/etcd
    - --feature-gates=InitialCorruptCheck=true
    - --initial-advertise-peer-urls=https://192.168.201.10:2380
    - --initial-cluster=controlplane=https://192.168.201.10:2380
    - --key-file=/etc/kubernetes/pki/etcd/server.key
    - --listen-client-urls=https://127.0.0.1:2379,https://192.168.201.10:2379
    - --listen-metrics-urls=http://127.0.0.1:2381
    - --listen-peer-urls=https://192.168.201.10:2380
    - --name=controlplane
    - --peer-cert-file=/etc/kubernetes/pki/etcd/peer.crt
    - --peer-client-cert-auth=true
    .
    .
    .
```

### TLS Certificates

In K8s, all the components talk to each other over **mTLS**.

All the **TLS certificates** and **keys** are under the **pki** folder. K8s control-plane components use these certificates to authenticate and communicate with each other.

Also, there is an **etcd** subdirectory that contains the etcd-specific certificates and private keys used to secure communication between **etcd nodes** and between the **API server** and **etcd nodes**.

```bash
vagrant@controlplane:/etc/kubernetes/pki$ tree
.
├── apiserver.crt
├── apiserver-etcd-client.crt
├── apiserver-etcd-client.key
├── apiserver.key
├── apiserver-kubelet-client.crt
├── apiserver-kubelet-client.key
├── ca.crt
├── ca.key
├── etcd
│   ├── ca.crt
│   ├── ca.key
│   ├── healthcheck-client.crt
│   ├── healthcheck-client.key
│   ├── peer.crt
│   ├── peer.key
│   ├── server.crt
│   └── server.key
├── front-proxy-ca.crt
├── front-proxy-ca.key
├── front-proxy-client.crt
├── front-proxy-client.key
├── sa.key
└── sa.pub

1 directory, 22 files
```

If you are setting up a **self-hosted** cluster for production use, these certificates have to be requested from the organization’s network or security team. They will generate these certificates signed by the organization’s internal Certificate authority and provide them to you.

![TLS Certificates & PKI]({{ site.baseurl }}/assets/img/k8s-course/pki.png)

### Kubeconfig Files

Any components that need to authenticate to the API server need the **kubeconfig** file.

All the cluster Kubeconfig files (**.conf files**) are present in the **/etc/kubernetes** folder.

```bash
vagrant@controlplane:/etc/kubernetes$ tree | grep -n ".conf"
2:├── admin.conf
3:├── controller-manager.conf
4:├── kubelet.conf
34:├── scheduler.conf
35:└── super-admin.conf
```

The **admin.conf**, file, used by end users to access the API server to manage the clusters. You can use this file to connect the cluster from a remote workstation.

For example, if you check the **Controller Manager** static pod manifest file (**/etc/kubernetes/manifests/**), you can see the **controller-manager.conf** added as the authentication and authorization parameter.

```bash
vagrant@controlplane:/etc/kubernetes/manifests$ sudo cat kube-controller-manager.yaml
...
spec:
  containers:
  - command:
    - kube-controller-manager
    - --allocate-node-cidrs=true
    - --authentication-kubeconfig=/etc/kubernetes/controller-manager.conf
    - --authorization-kubeconfig=/etc/kubernetes/controller-manager.conf
```

### Kubelet Configurations

Kubelet service runs as a **systemd** service on all the cluster nodes.

> **Systemd** is the modern default system and service manager for almost all Linux distributions. Running as the very first process during boot (Process ID 1), it initializes the system, mounts file systems, and starts the background services.
---
> **Kubelet** is the primary node agent that runs on every machine (node) within a K8s cluster to manage container execution. It acts as the execution arm of the control plane, ensuring that all containers described in your PodSpecs are running and perfectly healthy.

**kubelet** systemd services can be viewed under **/etc/systemd/system**:

```bash
vagrant@controlplane:/$ ls /etc/systemd/system
apt-daily.service                           multipath-tools.service         snap.lxd.user-daemon.service
apt-daily-upgrade.service                   multi-user.target.wants         snap.lxd.user-daemon.unix.socket
cloud-final.service.wants                   network-online.target.wants     snap-snapd-19457.mount
cloud-init.target.wants                     open-vm-tools.service.requires  snap-snapd-26865.mount
dbus-org.freedesktop.ModemManager1.service  paths.target.wants              sockets.target.wants
dbus-org.freedesktop.resolve1.service       rescue.target.wants             sshd-keygen@.service.d
dbus-org.freedesktop.thermald.service       sleep.target.wants              sshd.service
dbus-org.freedesktop.timesync1.service      snap-core20-1974.mount          sudo.service
emergency.target.wants                      snapd.mounts.target.wants       sysinit.target.wants
final.target.wants                          snap-lxd-24322.mount            syslog.service
getty.target.wants                          snap-lxd-38800.mount            timers.target.wants
graphical.target.wants                      snap.lxd.activate.service       vmtoolsd.service
iscsi.service                               snap.lxd.daemon.service
mdmonitor.service.wants                     snap.lxd.daemon.unix.socket
```

The exact location of the **kubelet.service** is found by the following command:

```bash
vagrant@controlplane:/$ sudo find -name "kubelet.service"
./usr/lib/systemd/system/kubelet.service
./sys/fs/cgroup/system.slice/kubelet.service
./etc/systemd/system/multi-user.target.wants/kubelet.service
./run/calico/cgroup/system.slice/kubelet.service
```

The **/var/lib/kubelet/config.yaml** contains all the kubelet-related configurations.

**/var/lib/kubelet/kubeadm-flags.env** file contains the container runtime environment Linux socket and the infra container (**pause container**) image.

> The pause container is a hidden infrastructure container that K8s automatically creates inside every single Pod to hold the network namespace and keep the Pod alive. By having the lightweight pause container start first, the network remains active and unchanged even if your application container crashes, restarts, or updates.

You can view the **pause container** by the following command:

```bash
vagrant@controlplane:~$ sudo crictl images
IMAGE                                     TAG                 IMAGE ID            SIZE
quay.io/calico/apiserver                  v3.31.3             ac46eecb3d7f8       50MB
quay.io/calico/cni                        v3.31.3             f2520fbaa2761       72.1MB
quay.io/calico/csi                        v3.31.3             6f60b868a2970       10.3MB
quay.io/calico/goldmane                   v3.31.3             6eaae458d5f11       55.6MB
quay.io/calico/kube-controllers           v3.31.3             95bc8e4bc61e7       54MB
```

### CoreDNS Configurations

If you list the **Configmaps** in the **kube-system** namespace, you can see the **CoreDNS** configmap.

```bash
vagrant@controlplane:~$ kubectl get configmaps --namespace=kube-system
NAME                                                   DATA   AGE
coredns                                                1      2d1h
extension-apiserver-authentication                     6      2d1h
kube-apiserver-legacy-service-account-token-tracking   1      2d1h
kube-proxy                                             2      2d1h
kube-root-ca.crt                                       1      2d1h
kubeadm-config                                         1      2d1h
kubelet-config                                         1      2d1h
```

To view the **CoreDNS** configmap contents, use the following command:

```bash
vagrant@controlplane:~$ kubectl edit configmap coredns --namespace=kube-system
```
