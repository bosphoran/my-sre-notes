---
layout: post
title: "Node Affinity"
date: 2026-07-12
categories: [k8s]
---

## Introduction

NodeAffinity is similar to NodeSelector but provides more granular control over the selection of nodes. It's a feature that allows us to schedule pods based on multiple properties of a node.

Using NodeAffinity we can set complex criteria allowing both **hard requirements** (must meet, similar to nodeSelector) and **soft requirements** (prefer to meet but not mandatory).

> Scheduling focuses on resource allocation and planning (where to run a pod), while execution focuses on running and maintaining the containers. The scheduler is responsible for scheduling, and the kubelet is responsible for execution.

There are two types of node affinity:

1. **requiredDuringSchedulingIgnoredDuringExecution** (Hard rule): This type of node affinity ensures that a pod will be scheduled only on nodes with labels matching the defined criteria. The pod remains unscheduled if no nodes fulfill the requirement. It is also known as required node affinity.
2. **preferredDuringSchedulingIgnoredDuringExecution** (Soft rule): This type of node affinity expresses a preference for node selection. If no preferred nodes are available, the pod is scheduled on any other suitable node in the cluster. There's no scheduling failure. It is also known as preferred node affinity.

> **IgnoredDuringExecution** means that if the node labels change after Kubernetes schedules the Pod, it does not evict the pod. It will continue running on the same node.

![Node Affinity]({{ site.baseurl }}/assets/img/k8s-course/node-affinity.jpg)

We can specify node affinities using the **affinity.nodeAffinity** field in the **spec** section of the pod manifest. And the **matchExpressions** acts as a filter for labels. Also, we can use operators like **In**, **NotIn**, **Exists**, and **DoesNotExist**.

![Node Affinity 2]({{ site.baseurl }}/assets/img/k8s-course/node-affinity-2.jpg)

**preferredDuringSchedulingIgnoredDuringExecution** supports a parameter called **weight**. It is a way to say how important this particular rule is compared to other rules. **Weight** can be between **1 and 100**, where a higher number means the rule is more important.

In this example, if a node has the label **diskType=ssd**, it will get a score of **5**. If a node has the label **nodeType=high-memory**, it will get a score of **10**.

If a node has both labels, it will get a score of **15** (5 + 10 - cumulative weight). The scheduler will prefer to schedule the pod on the node with the highest score (in this case, a node with both labels).

```bash
nodeAffinity:
  preferredDuringSchedulingIgnoredDuringExecution:
  - weight: 5
    preference:
      matchExpressions:
      - key: diskType
        operator: In
        values:
        - ssd
  - weight: 10
    preference:
      matchExpressions:
      - key: nodeType
        operator: In
        values:
        - high-memory
```

![Node Affinity 3]({{ site.baseurl }}/assets/img/k8s-course/node-affinity-3.jpg)

---

### Working With Node Affinity

Imagine a cluster with **20 worker nodes**. Four nodes are dedicated to running database workloads with **SSD hard disks** for better performance, and we want to deploy our pods only on those nodes.

```bash
vagrant@controlplane:~$ kubectl run database --image=nginx --dry-run=client -o yaml > node-affinity-database.yaml
```

Now, open the **node-affinity-database.yaml** file and modify it by adding nodeAffinity parameters to schedule the deployment on nodes with the specific label **diskType=ssd**.

```bash
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    run: database
  name: database
spec:
  containers:
  - image: nginx
    name: database
    resources: {}
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: diskType
            operator: In
            values:
            - ssd
  dnsPolicy: ClusterFirst
  restartPolicy: Always
status: {}
```

Now apply this manifest.

```bash
vagrant@controlplane:~$ kubectl apply -f node-affinity-database.yaml
pod/database created

# Check pod status
vagrant@controlplane:~$ kubectl get pods
NAME                                            READY   STATUS    RESTARTS       AGE
database                                        0/1     Pending   0              31s
```

As you notice, the pod is in a **pending** state. This is because we used the Required Node Affinity, and the scheduler couldn't find any node with the **diskType=ssd** label.

```bash
vagrant@controlplane:~$ kubectl describe pod database | grep Events -A10
Events:
  Type     Reason            Age   From               Message
  ----     ------            ----  ----               -------
  Warning  FailedScheduling  2m    default-scheduler  0/3 nodes are available: 1 node(s) had untolerated taint(s), 2 node(s) didn't match Pod's node affinity/selector. no new claims to deallocate, preemption: 0/3 nodes are available: 3 Preemption is not helpful for scheduling.
```

Let's label a node with **diskType=ssd**.

```bash
vagrant@controlplane:~$ kubectl label node node01 diskType=ssd
node/node01 labeled
```

Once you label **node01** with **diskType=ssd**, the pending pod gets scheduled on node01 and starts running.

```bash
vagrant@controlplane:~$ kubectl get pods -o wide
NAME                                            READY   STATUS    RESTARTS       AGE    IP               NODE
database                                        1/1     Running   0              4m4s   10.244.196.149   node01
```

---

### Preferred Affinity With Weights

This method is of type **preferredDuringSchedulingIgnoredDuringExecution** with weight.

**Preferred affinities** are soft rules with weights. The scheduler scores nodes based on how many preferred rules they satisfy and how heavily those rules are weighted.

- Node01 has diskType=ssd.
- Node02 has both diskType=ssd and nodeType=high-memory.

The scheduler adds up the weights. Node02’s score will be higher because it satisfies more preferred rules.

### Required & Preferred Affinity Combination

Consider a backend API application that processes sensitive credit card information. Due to strict security compliance regulations, the application must be deployed on nodes that have the label **apptype=pci-compliant** (Required Affinity). This is a non-negotiable requirement.

> PCI compliance refers to the Payment Card Industry Data Security Standard (PCI DSS), which is a set of security standards that were developed to protect credit card information and prevent credit card fraud. PCI compliance is required for any organization that processes, stores or transmits credit card data.

Create the **affinity-webserver.yaml** and add the Required Affinity rule where it says schedule pod on the node with labels appType=pci-compliant as given below.

```bash
apiVersion: v1
kind: Pod
metadata:
  name: webserver-affinity
spec:
  containers:
  - name: nginx-container
    image: nginx
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: appType
            operator: In
            values:
            - pci-compliant
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 5
        preference:
          matchExpressions:
          - key: diskType
            operator: In
            values:
            - ssd
      - weight: 10
        preference:
          matchExpressions:
          - key: nodeType
            operator: In
            values:
            - high-memory
```

Let's deploy this YAML and check the pod status. It should go to the pending state as no nodes have the label **apptype=pci-compliant** and it is part of the Required Affinity.

```bash
vagrant@controlplane:~$ kubectl apply -f affinity-webserver.yaml
pod/webserver-affinity created

# Check pod status
vagrant@controlplane:~$ kubectl get pods
NAME                                            READY   STATUS    RESTARTS      AGE
webserver-affinity                              0/1     Pending   0             19s

# Check the reason for pending status
vagrant@controlplane:~$ kubectl describe pod webserver-affinity | grep Events -A10
Events:
  Type     Reason            Age   From               Message
  ----     ------            ----  ----               -------
  Warning  FailedScheduling  51s   default-scheduler  0/3 nodes are available: 1 node(s) had untolerated taint(s), 2 node(s) didn't match Pod's node affinity/selector. no new claims to deallocate, preemption: 0/3 nodes are available: 3 Preemption is not helpful for scheduling.
```

Let's label node01 with **appType=pci-compliant**. The pod automatically gets scheduled on node01.

```bash
vagrant@controlplane:~$ kubectl label node node01 appType=pci-compliant
node/node01 labeled

# Check pod status
vagrant@controlplane:~$ kubectl get pods -o wide
NAME                                            READY   STATUS    RESTARTS      AGE     IP               NODE
webserver-affinity                              1/1     Running   0             4m54s   10.244.196.158   node01
```

Clean up the created labels using the following commands.

```bash
vagrant@controlplane:~$ kubectl label node node01 node02 appType- diskType- nodeType-
node/node01 unlabeled
node/node01 unlabeled
label "appType" not found.
label "diskType" not found.
label "nodeType" not found.
node/node02 not labeled
```

> While nodeSelector and nodeAffinity overlap in functionality for simple exact matches, nodeAffinity offers added flexibility. 

---

###

- Label **node01** with key-value **payment-gateway=enabled**.
- Create a Deployment named **zacademy-payment-gw** using the image nginx and replicas 3. Use affinity **requiredDuringSchedulingIgnoredDuringExecution** to schedule pods on node1.
- Ensure the pods are in running state on **node01**.
- Finally, delete the deployment and remove the node labels.

```bash
# Label node01
vagrant@controlplane:~$ kubectl label node node01 payment-gateway=enabled
node/node01 labeled

# Create the deployment yaml file
vagrant@controlplane:~$ kubectl create deploy zacademy-payment-gw --image=nginx --replicas=3 --dry-run=client -o yaml > zacademy-payment-deploy.yaml
```

Now let's edit the **zacademy-payment-deploy.yaml** file and add the Required Affinity:

```bash
apiVersion: apps/v1
kind: Deployment
metadata:
  creationTimestamp: null
  labels:
    app: zacademy-payment-gw
  name: zacademy-payment-gw
spec:
  replicas: 3
  selector:
    matchLabels:
      app: zacademy-payment-gw
  strategy: {}
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: zacademy-payment-gw
    spec:
      containers:
      - image: nginx
        name: nginx
        resources: {}
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: payment-gateway
                operator: In
                values:
                - enabled
status: {}
```

Apply this manifest and check the status.

```bash
# Apply the manifest file
vagrant@controlplane:~$ kubectl apply -f zacademy-payment-deploy.yaml
deployment.apps/zacademy-payment-gw created

# Check pod status
vagrant@controlplane:~$ kubectl get pods -o wide
NAME                                            READY   STATUS    RESTARTS      AGE     IP               NODE
zacademy-payment-gw-6487475775-xqwcm            1/1     Running   0             32s     10.244.196.148   node01
```

Unlable the node and delete deployment.

```bash
# Unlabel node01
vagrant@controlplane:~$ kubectl label node node01 payment-gateway-
node/node01 unlabeled

# Delete zacademy-payment-gw deployment
vagrant@controlplane:~$ kubectl delete deploy zacademy-payment-gw
deployment.apps "zacademy-payment-gw" deleted from default namespace
```

