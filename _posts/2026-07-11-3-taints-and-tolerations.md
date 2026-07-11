---
layout: post
title: "Taints and Tolerations"
date: 2026-07-11
categories: [k8s]
---

## Introduction

A **Taint** is a kind of "repellent" applied to a K8s node. It tells the K8s scheduler not to schedule any pods on that node.

In a pod's configuration, you can define toleration parameters. A pod with tolerations can run on tainted nodes.

![Taint & Toleration]({{ site.baseurl }}/assets/img/k8s-course/taint-toleration.jpg)

### Taints

Taints are the labels we apply to a node to repel pods from being scheduled on it.

> Taints mark a node with a condition that repels Pods unless those Pods explicitly tolerate it.

The following command is used to apply a taint to a specific node in a **K8s** cluster.

```bash
kubectl taint nodes <node-name> <key>=<value>:<effect>

# Taint node01
vagrant@controlplane:~$ kubectl taint no node01 type=web:NoSchedule
node/node01 tainted

# Remove taint on node01
vagrant@controlplane:~$ kubectl taint nodes node01 type=web:NoSchedule-
node/node01 untainted
```

### The Three Taint Effects

![Taint & Toleration 2]({{ site.baseurl }}/assets/img/k8s-course/taint-toleration-2.jpg)

- **NoSchedule:** Pods that do not tolerate the taint will not be scheduled on the node. Pods already running on the node are not affected.
- **PreferNoSchedule:** It's a softer version of NoSchedule. The scheduler will try to avoid scheduling non-tolerant pods on the node, but it's not guaranteed. For instance, if the cluster nodes are already overloaded the scheduler might resort to scheduling the non-tolerant pod on the PreferNoSchedule node as a last resort.
- **NoExecute:** When a taint with the NoExecute effect is added to a node, any pods that do not tolerate the taint will be immediately evicted from the node.

---

### Tolerations

A **taint** on a node repels pods. A **toleration** on a pod is like an exception. It tells the scheduler, "**this pod can run on a node with this taint**."

> Taints live on nodes. Tolerations live on pods

![Taint & Toleration 3]({{ site.baseurl }}/assets/img/k8s-course/tain-toleration-3.png)

We add tolerations under the **spec** section of the Pod. For example,

```bash
apiVersion: v1
kind: Pod
metadata:
  name: web-pod
spec:
  tolerations:
    - key: "type"
      operator: "Equal"
      value: "web"
      effect: "NoSchedule"
  containers:
    - name: nginx
      image: nginx
```

This pod can now be scheduled on a node tainted with **type=web:NoSchedule**.

### Working With Taints & Tolerations

```bash
vagrant@controlplane:~$ kubectl get nodes
NAME           STATUS   ROLES           AGE   VERSION
controlplane   Ready    control-plane   37d   v1.34.8
node01         Ready    worker          37d   v1.34.8
node02         Ready    worker          37d   v1.34.8
```

Let's taint worker nodes node01 and node02 with NoSchedule effect.

```bash
vagrant@controlplane:~$ kubectl taint node node01 type=web:NoSchedule
node/node01 tainted

vagrant@controlplane:~$ kubectl taint node node02 type=web:NoSchedule
node/node02 tainted
```

You can describe the node and check the taints section of the node.

```bash
vagrant@controlplane:~$ kubectl describe no node01 | egrep "Name:|Taints:"
Name:               node01
Taints:             type=web:NoSchedule
```

Now, let's deploy an nginx pod, get the pod status, and see what happens.

```bash
vagrant@controlplane:~$ kubectl run webserver --image=nginx
pod/webserver created

# Check pod status
vagrant@controlplane:~$ kubectl get pods
NAME                                            READY   STATUS    RESTARTS      AGE
webserver                                       0/1     Pending   0             30s
```

Lets **describe** the pod, to find out from the events why it is in a **pending** state.

```bash
vagrant@controlplane:~$ kubectl describe po webserver | grep Events -A 10
Events:
  Type     Reason            Age   From               Message
  ----     ------            ----  ----               -------
  Warning  FailedScheduling  102s  default-scheduler  0/3 nodes are available: 3 node(s) had untolerated taint(s). no new claims to deallocate, preemption: 0/3 nodes are available: 3 Preemption is not helpful for scheduling.
```

We can see that the scheduler is unable to schedule the pod because both nodes have some taints defined. (The controlplane node has a taint by default.)

Now to fix this let's add **tolerations** in our pod **manifest**.

```bash
# First, delete the existing pod.
vagrant@controlplane:~$ kubectl delete pod webserver
pod "webserver" deleted from default namespace
```

Let's create a pod YAML and add the **toleration** parameters for the **webserver pod** to get scheduled on the tainted **node01**.

```bash
vagrant@controlplane:~$ kubectl run webserver --image nginx --dry-run=client -o yaml > toleration-webserver.yaml
```

Open the **toleration-webserver.yaml** and add the toleration parameters under spec as given below.

```bash
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    run: webserver
  name: webserver
spec:
  containers:
  - image: nginx
    name: webserver
    resources: {}
  tolerations:
  - key: "type"
    operator: "Equal"
    value: "web"
    effect: "NoSchedule"
  dnsPolicy: ClusterFirst
  restartPolicy: Always
status: {}
```

Let's apply this manifest and check the status:

```bash
vagrant@controlplane:~$ kubectl apply -f toleration-webserver.yaml
pod/webserver created

# Check pod status
vagrant@controlplane:~$ kubectl get pods -o wide
NAME                                            READY   STATUS    RESTARTS      AGE    IP               NODE   
webserver                                       1/1     Running   0             17s    10.244.140.93    node02
```

Now, lets remove the **Taint** from the worker nodes.

```bash
# Node01
vagrant@controlplane:~$ kubectl taint no node02  type=web:NoSchedule-
node/node02 untainted

# Node02
vagrant@controlplane:~$ kubectl taint no node01  type=web:NoSchedule-
node/node01 untainted
```

### Evict Pods From Node01

Consider a scenario where it's necessary to remove all pods from a particular node. For example, to upgrade a node.

For this, we can use the **NoExecute** taint effect.

First, let's set up an **Nginx deployment** with **two replicas**.

```bash
vagrant@controlplane:~$ kubectl create deploy webserver --image=nginx --replicas=2
deployment.apps/webserver created

# Check status
vagrant@controlplane:~$ kubectl get pods -o wide
NAME                                            READY   STATUS    RESTARTS      AGE    IP               NODE
webserver-cc6f5d7cc-jns4m                       1/1     Running   0             32s    10.244.196.147   node01
webserver-cc6f5d7cc-sdxn5                       1/1     Running   0             32s    10.244.140.95    node02
```

Now let's taint the node01 with NoExecute effect.

```bash
vagrant@controlplane:~$ kubectl taint node node01 type=upgrade:NoExecute
node/node01 tainted
```

Now if you check the pods, it will be in **pending** state.

```bash
vagrant@controlplane:~$ kubectl get pods -o wide
NAME                                            READY   STATUS              RESTARTS   AGE     IP              NODE
webserver-cc6f5d7cc-7lt4d                       1/1     Running             0          92s     10.244.140.97   node02
webserver-cc6f5d7cc-sdxn5                       1/1     Running             0          2m58s   10.244.140.95   node02
```

As you may notice, the pod has been rescheduled to an untainted node, node02.

Now, lets untaint node01.

```bash
vagrant@controlplane:~$ kubectl taint node node01 type=upgrade:NoExecute-
node/node01 untainted
```

---

### Real World Example

We will look at two key use cases of Taints & Tolerations.

1. Node Maintenance:

To safely carry out maintenance tasks on nodes, such as upgrading or rebooting, we need to ensure that no pods are running on the node and that it is made unschedulable.

During the upgrade task, we used a command called **kubectl drain** to make the node unschedulable for the upgrade process.

When you run the **kubectl drain** command, it automatically applies a taint to the node being drained. This taint has the key **node.kubernetes.io/unschedulable** and the effect **NoSchedule**. It ensures that no new pods are scheduled on the node while it is being drained.

Once the kubeadm upgrade was completed, we used the kubectl **uncordon** command to make the node schedulable again.

2. Dedicated GPU Node:

Taints and tolerations are very important in **GPU-based workflows**.

**GPU nodes** are expensive and limited. You taint them so only **GPU workloads** (pods with the matching toleration) get scheduled there. This prevents regular CPU workloads from accidentally consuming GPU node capacity.

This pattern is standard practice in **ML/AI** clusters where GPU nodes run alongside regular CPU nodes. Teams use it to reserve GPU nodes exclusively for training jobs, inference servers, or any workload that actually needs GPU resources.
