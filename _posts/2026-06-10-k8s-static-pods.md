---
layout: post
title: "Static Pods"
date: 2026-06-10
categories: [k8s]
---

## Introduction

**Static Pods** are special K8s Pods that are managed directly by the local **kubelet** daemon on a node rather than by the K8s **API server** on controlplane.

The **kubelet** service periodically scans the directory at **/etc/kubernetes/manifest** for the manifest files and creates static pods based on these manifest files.

![static pod]({{ site.baseurl }}/assets/img/k8s-course/static-pod.png)

- **Node-Bound**: Static pods are permanently tied to the specific node where they reside. If the node crashes, the Pods cannot be rescheduled to a different node.
- **Mirror Pods**: Although the **API server** doesn't control static Pods, the **kubelet** creates a read-only **mirror pod** on the API server so you can view the Pod's status using standard **kubectl** command.

> **Static Pods** are mainly used for bootstrapping the K8s control plane. For example, **kubeadm** uses static pods to bring up **etcd**, **kube-apiserver**, **kube-controller-manager**, and **kube-scheduler** before the full cluster exists.


### Creating Static Pod

First, we need to find the **staticPodPath** parameter configured with Kubelet.

> The **staticPodPath** is a configuration setting for the **kubelet** that defines the local directory on a node where it continuously scans for Pod definition files.

The default **staticPodPath** location is **/etc/kubernetes/manifests**. You can verify it by checking the kubelet config.

```bash
vagrant@node01:~$ ps -aux | grep kubelet | grep config.yaml
root        1416  4.7  4.1 1904260 83672 ?       Ssl  18:20   0:20 /usr/bin/kubelet --bootstrap-kubeconfig=/etc/kubernetes/bootstrap-kubelet.conf --kubeconfig=/etc/kubernetes/kubelet.conf --config=/var/lib/kubelet/config.yaml --pod-infra-container-image=registry.k8s.io/pause:3.10.1 --node-ip=192.168.201.11
```

The **kubelet** configuration file location is at **/var/lib/kubelet/config.yaml**. Lets check inside the file to find the static pod location.

```bash
vagrant@node01:~$ cat /var/lib/kubelet/config.yaml | grep "staticPodPath"
staticPodPath: /etc/kubernetes/manifests
```

Here, we will create a simple **nginx** static pod. Lets, navigate to the static pod manifest directory in **node01**.

```bash
vagrant@node01:~$ cd /etc/kubernetes/manifests/
vagrant@node01:/etc/kubernetes/manifests$ sudo nano nginx.yaml

vagrant@node01:/etc/kubernetes/manifests$ cat nginx.yaml
apiVersion: v1
kind: Pod
metadata:
 name: webserver
spec:
 containers:
  - name: webserver-container
    image: nginx
```

Once the manifest file has been created, **kubelet** regularly scans the location and creates the static pod based on the existing manifest files.

Since, we can not access **kubectl** in the worker nodes, we use **crictl**.

> **crictl** stands for **Container Runtime Interface**. It is a command-line interface tool used to interact with CRI-compatible container runtimes.

```bash
vagrant@node01:/etc/kubernetes/manifests$ sudo crictl pods
POD ID              CREATED             STATE               NAME                               NAMESPACE           ATTEMPT             RUNTIME
340daaa791cdc       6 minutes ago       Ready               webserver-node01                   default             0                   (default)
75b7187054560       22 minutes ago      Ready               csi-node-driver-vscfx              calico-system       8                   (default)
a12f1a53fe776       22 minutes ago      Ready               webserver                          default             1                   (default)
```

You can also view the static pod mirror in the controlplane node.

```bash
vagrant@controlplane:~$ kubectl get pods
NAME                      READY   STATUS      RESTARTS      AGE
busybox-multi-container   0/3     Completed   0             10h
multi-container-pod       2/2     Running     2 (24m ago)   11h
webserver                 1/1     Running     1 (26m ago)   11h
webserver-node01          1/1     Running     0             8m21s
```

As you may notice, **kubelet** has appended the node name in the container name. This way, API-Server can differentiate between api-server managed pods and static pods.

Since, static pods are managed by **kubelet**, therefore, **kubectl** can not edit or delete static pods.

```bash
vagrant@controlplane:~$ kubectl edit pod -n webserver-node01
error: edit cancelled, no objects found
```

To make changes to a static pod, we need to modify its **.yaml** file in the **/etc/kubernetes/manifests**.

```bash
vagrant@node01:~$ ls /etc/kubernetes/manifests/
nginx.yaml
```

To **delete** a static pod, we just need to delete its manifest file.

---

While listing K8s objects, pods, services, etc., we can use custom columns:

```bash
vagrant@controlplane:/etc/kubernetes/manifests$ kubectl get pods -o custom-columns=NAME:.metadata.name,NODE:.spec.nodeName,STATUS:.status.phase
NAME                      NODE     STATUS
busybox-multi-container   node02   Succeeded
multi-container-pod       node02   Running
webserver                 node01   Running
webserver-node01          node01   Running
```
