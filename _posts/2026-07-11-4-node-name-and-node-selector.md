---
layout: post
title: "Node Name & Node Selector"
date: 2026-07-11
categories: [k8s]
---

## Introduction

You can schedule a pod on nodes based on their name and labels using **nodeName** and **nodeSelector**.

Assume a K8s cluster with twenty nodes. By default, K8s schedules a pod based on a **scoring algorithm**. It scores each node based on certain criteria, like how busy the node is, what resources it has available, and how well it can meet the needs of the pod.

What if I want to deploy a pod on a specific node, let's say **node02**?

### Node Name

You can use the **nodeName field** to manually assign the pod to that specific node. This allows for direct scheduling of pods on specific nodes, bypassing the default scheduling process.

![Node Name]({{ site.baseurl }}/assets/img/k8s-course/node-name.jpg)

### Node Selector

In production K8s clusters, you will run both **GPU** and **non-GPU** nodes in the cluster. Some applications require GPU power while others wont.

Let's say out of 20 nodes in the cluster, 5 nodes have GPUs.

If you want to deploy applications that need the GPU on nodes equipped with GPUs? This is where **nodeSelector** comes into play.

![Node Selector]({{ site.baseurl }}/assets/img/k8s-course/node-selector.jpg)

- **Labeling Nodes:** First, the nodes in the cluster are labeled with key-value pairs that describe their capabilities. In our example, nodes with GPUs would have a label like **gpu=true**.
- **Pod Targeting with NodeSelector:** When deploying pods that require GPUs, a nodeSelector section is added to the pod specification. This section specifies **key-value pairs** that must match the node labels. For GPU pods, the specification would be **nodeSelector: { gpu: "true" }**
- **Scheduling Based on Labels:** The K8s scheduler will only schedule pods onto nodes that have matching labels defined in the **nodeSelector** field. This ensures your GPU pods run exclusively on nodes with GPUs.

---

### Working With NodeName

Create a nodename-pod.yaml file.

```bash
vagrant@controlplane:~$ kubectl run webserver --image=nginx --dry-run=client -o yaml > nodename-pod.yaml
```

Open the **nodename-pod.yaml** file and add the nodeName parameter with the value **random-node**. Our cluster does not have any node named random-node. However, this is to test the behavior of the parameter.

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
  nodeName: random-node
  dnsPolicy: ClusterFirst
  restartPolicy: Always
status: {}
```

Lets apply the manifest file.

```bash
vagrant@controlplane:~$ kubectl apply -f nodename-pod.yaml
pod/webserver created

# The pod will likely remain in a Pending state as the scheduler cannot find a node named "random-node".
vagrant@controlplane:~$ kubectl get pods
NAME                                            READY   STATUS    RESTARTS      AGE
webserver                                       0/1     Pending   0             7s
```

After a certain amount of time, K8s recognizes the **node non-existent** issue and marks the pod as failed. The pod then disappears from the kubectl get pods output.

Lets, update **nodename-pod.yaml** by replacing "random-node" with **node01**.

```bash
vagrant@controlplane:~$ kubectl apply -f nodename-pod.yaml
pod/webserver created

# Check pod status
vagrant@controlplane:~$ kubectl get pods -o wide
NAME                                            READY   STATUS    RESTARTS      AGE   IP               NODE
webserver                                       1/1     Running   0             46s   10.244.196.153   node01  
```

### Working with Node Selectors

K8s (k8s) nodes do come with a set of default labels pre-populated by the kubelet. These labels provide information about the node's characteristics like operating system, architecture, and hostname.

- **kubernetes.io/hostname** - Node's hostname
- **kubernetes.io/os** - Operating system running on the node
- **kubernetes.io/arch** - Architecture of the node (e.g., amd64, arm64)

You can view these labels using the **--show-labels** flag.

```bash
vagrant@controlplane:~$ kubectl get nodes --show-labels
NAME           STATUS   ROLES           AGE   VERSION   LABELS
controlplane   Ready    control-plane   38d   v1.34.8   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/arch=amd64,kubernetes.io/hostname=controlplane,kubernetes.io/os=linux,node-role.kubernetes.io/control-plane=,node.kubernetes.io/exclude-from-external-load-balancers=
node01         Ready    worker          38d   v1.34.8   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/arch=amd64,kubernetes.io/hostname=node01,kubernetes.io/os=linux,node-role.kubernetes.io/worker=worker
node02         Ready    worker          38d   v1.34.8   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/arch=amd64,kubernetes.io/hostname=node02,kubernetes.io/os=linux,node-role.kubernetes.io/worker=worker
```

You can leverage existing labels or define your own labels for NodeSelectors in K8s.

- If your nodes already have labels that describe their capabilities (eg, **kubernetes.io/arch=arm64**), you can directly use those labels in your NodeSelectors.
- If your nodes don't have relevant labels, or you need more specific categorization, you can define custom labels to target pods to specific node types. (e.g., **gpu=true** for nodes with GPUs, **storage=ssd** for nodes with solid-state drives or location-specific labels like **region=us-west-2**, **zone=us-west-2a**, etc)

**To label a node, there are 2 ways:**

1. Directly edit the node and add the label in the manifest. Run **kubectl edit node <node-name>** command, it will open the manifest of the node. Add the desired labels in the metadata.labels section using key-value pairs (e.g., **gpu=true**)
2. Use an imperative way to label the node. **kubectl label node <node-name> <key>=<value>**.

Let's see how we can use a node selector to constrain a Pod to nodes with a specific label. Let us label the node named node01 with a custom label **gpu=true**.

```bash
vagrant@controlplane:~$ kubectl label node node01 gpu=true
node/node01 labeled
```

To check the labels, we can run the following commands:

```bash
vagrant@controlplane:~$ kubectl describe no node01 | grep Labels -A 5
Labels:             beta.kubernetes.io/arch=amd64
                    beta.kubernetes.io/os=linux
                    gpu=true
                    kubernetes.io/arch=amd64
                    kubernetes.io/hostname=node01
                    kubernetes.io/os=linux

vagrant@controlplane:~$ kubectl get node node01  --show-labels
NAME     STATUS   ROLES    AGE   VERSION   LABELS
node01   Ready    worker   38d   v1.34.8   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,gpu=true,kubernetes.io/arch=amd64,kubernetes.io/hostname=node01,kubernetes.io/os=linux,node-role.kubernetes.io/worker=worker
```

### Deploy Pod with nodeSelector

Let's create a gpu-webserver pod that uses nodeSelector with gpu=true label.

```bash
vagrant@controlplane:~$ kubectl run webserver --image=nginx --dry-run=client -o yaml > gpu-webserver.yaml
```

Below is the **gpu-webserver.yaml**:

```bash
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    run: gpu-webserver
  name: gpu-webserver
spec:
  containers:
  - image: nginx
    name: gpu-webserver
    resources: {}
  nodeSelector: 
    gpu: "true"
  dnsPolicy: ClusterFirst
  restartPolicy: Always
status: {}
```

We mentioned the same **key-value** pair in the **nodeSelector** field that we added as a label to the node. Now, when we apply this manifest, the pod will be scheduled on **node01**.

```bash
# Apply manifest file
vagrant@controlplane:~$ kubectl apply -f gpu-webserver.yaml
pod/webserver created

# Verify pod status
vagrant@controlplane:~$ kubectl get pods gpu-webserver -o wide
NAME            READY   STATUS    RESTARTS   AGE   IP               NODE     NOMINATED NODE   READINESS GATES
gpu-webserver   1/1     Running   0          12s   10.244.196.155   node01   <none>           <none>
```

To remove a label from a node, use "**-**" with the label key as shown below.

```bash
vagrant@controlplane:~$ kubectl label no node01 gpu-
node/node01 unlabeled
```

When you remove the label from the node, K8s does not automatically evict the pod from the node. This is because **nodeSelector** is designed for initial scheduling, not for ongoing pod placement management.

---

### Scenario: Assign Pod to Node Using Node Selector

The zacademy.org team is managing a K8s cluster that hosts a variety of applications, including machine learning services that require specific hardware configurations.

To ensure optimal performance and resource allocation, the team decide to utilize node selectors to schedule these services onto nodes with the appropriate hardware characteristics.

- First, label the nodes that have GPU accelerators installed with a custom label (node01), such as **type=ml-app**, to distinguish them from other nodes in the cluster.
- Then, write a Pod specification for a ml-app Pod that should be scheduled only on nodes labeled with **type=ml-app**. Check whether it is running on the specific nodes or not.

First, we need to label a node.

```bash
vagrant@controlplane:~$ kubectl label node node01 type=ml-app
node/node01 labeled

# Now let's generate a pod manifest:
vagrant@controlplane:~$ kubectl run ml-app --image=nginx --dry-run=client -o yaml > ml-app.yaml

# edit the ml-app.yaml and add the nodeSelector
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    run: ml-app
  name: ml-app
spec:
  containers:
  - image: nginx
    name: ml-app
    resources: {}
  nodeSelector:
    type: ml-app
  dnsPolicy: ClusterFirst
  restartPolicy: Always
status: {}

# Apply this manifest
vagrant@controlplane:~$ kubectl apply -f ml-app.yaml
pod/ml-app created

# Check pod status
vagrant@controlplane:~$ kubectl get pods -o wide
NAME                                            READY   STATUS    RESTARTS      AGE   IP               NODE
ml-app                                          1/1     Running   0             15s   10.244.196.157   node01
```
