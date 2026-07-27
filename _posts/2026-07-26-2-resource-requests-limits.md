---
layout: post
title: "Resource Requests & Limits"
date: 2026-07-26
categories: [k8s]
---

## Introduction

K8s uses **requests and limits** parameters to schedule pods and allocate resources efficiently.

**Memory and CPU** resource specifications fall under resource requests and limits, which are important for cluster resource management and capacity planning.

We can define resource requests and limits in the spec section of a pod manifest.

```bash
apiVersion: v1
kind: Pod
metadata:
  name: webserver-pod              # Pod name
spec:
  containers:
  - name: webserver-container      # Container name
    image: nginx:latest            # Using Nginx as the webserver
    resources:
      requests:                    # Minimum guaranteed resources
        cpu: "100m"                # 0.1 CPU core reserved
        memory: "128Mi"            # 128 MB memory reserved
      limits:                      # Maximum allowed resources
        cpu: "500m"                # 0.5 CPU core max
        memory: "256Mi"            # 256 MB memory max
```

- **requests:** Ensures the Pod always gets at least 0.1 CPU and 128 MB memory.
- **limits:** Prevents the Pod from using more than 0.5 CPU and 256 MB memory.

### Requests (Guaranteed Resources)

Requests are the minimum compute resources (CPU and memory) that K8s guarantees for a container. The K8s scheduler ensures a node has at least this much resource available before placing a pod on a worker node.

> If a Pod's total requested resources are unavailable on any of the worker nodes, the Pod will remain in a Pending state until these resources become available.

### Limits (Maximum Resources)

Limits are the maximum compute resources that a container is allowed to consume. A container can never consume more than the specified limits. If a container exceeds its limits, it will be throttled or killed.

Even though the worker node might have more memory and CPU available, the application cannot consume more than its limits.

> OOM killed stands for Out of Memory killed. This occurs when a container tries to use more memory than its specified limit.

![Resource Request & Limit]({{ site.baseurl }}/assets/img/k8s-course/request-limit.jpg)

### Pod Without Resource Specifications

What happens if you deploy a pod without the specified CPU and memory resources?

1. The pod could end up using all the available CPU and memory on the node, which could lead to resource starvation for other pods on the same node that require CPU and memory.
2. Pods may be scheduled on nodes that don't have enough capacity to run them effectively.

If you deploy a pod without specifying resources, K8s will apply the default values from the **LimitRange** as the CPU and memory for the container.

> **LimitRange** is a K8s object that lets you set the minimum, maximum, default, and default request values for CPU and memory that containers can use.

---

### How Kubernetes uses Cgroups and CFS to Manage Resources

> K8s uses **cgroups** to manage and enforce resource limits and quotas at the container level within the Pods. Cgroups (Control Groups) are a Linux kernel feature that lets you control and isolate system resources like CPU and memory.
> K8s uses the **Completely Fair Scheduler** (CFS) quota mechanism, a Linux kernel feature, to make sure containers within Pods don’t use more CPU time than they’re supposed to.

---

### Pod Quality of Service (QoS)

It is important to understand how resource requests and limits affect pod scheduling and quality of service.

**Without Specifying Resources:**
When you deploy a pod without specifying any resource requests or limits, K8s will not reserve any specific amount of CPU or memory for the pod. This can lead to overcommitment on the node.

**Specifying Requests and Limits:**
When you specify resource requests and limits in the pod configuration, K8s uses the request value to ensure the node has sufficient resources to run the pod. However, if multiple pods try to utilize all their limit ranges, the node can become **overcommitted** (Resource Pressure).

> **Overcommitment** = sum of resource limits > node capacity

To handle **overcommitment** situation, K8s needs a way to determine which pod to kill first (**eviction**).

> **Eviction** refers to the process of forcefully terminating one or more pods running on a node.

To manage resource overcommitment and prioritize pod eviction, K8s uses a concept called **Pod Quality of Service** (QoS) classes.

When deploying a pod, K8s assigns one of the following **QoS** classes based on the requests and limit parameters:

1. **Best Effort:** They are considered the lowest priority pods.
2. **Guaranteed:** It is considered the highest priority pod. These pods are the last to be terminated if the node runs out of resources.
3. **Burstable:** They have a higher priority than Best Effort pods but lower than Guaranteed pods.

---

### Working With Requests & Limits

Let's create an Nginx web server multi container pod with CPU and memory resources.

```bash
# Create webserver.yaml
apiVersion: v1
kind: Pod
metadata:
  name: webserver
  labels:
    type: web
spec:
  containers:
  - name: nginx-webserver
    image: nginx
    resources:
      requests:
        memory: "64Mi"
        cpu: "250m"
      limits:
        memory: "128Mi"
        cpu: "500m"
  - name: sidecar-container
    image: busybox
    command: ["sleep", "infinity"]
    resources:
      requests:
        memory: "32Mi"
        cpu: "100m"
      limits:
        memory: "64Mi"
        cpu: "200m"
```

```bash
# deploy the pod and ensure it is in running state.
vagrant@controlplane:~$ kubectl apply -f webserver-request-limit.yaml
pod/webserver created

#
vagrant@controlplane:~$ kubectl get pods
NAME                                            READY   STATUS    RESTARTS      AGE
webserver                                       2/2     Running   0             12s

# describe the pod, to view the CPU and memory information
vagrant@controlplane:~$ kubectl describe pod webserver | grep "Limits" -A5
    Limits:
      cpu:     500m
      memory:  128Mi
    Requests:
      cpu:        250m
      memory:     64Mi
--
    Limits:
      cpu:     200m
      memory:  64Mi
    Requests:
      cpu:        100m
      memory:     32Mi

# You can also view the assigned pod QoS Class
vagrant@controlplane:~$ kubectl describe pod webserver | grep "QoS Class"
QoS Class:                   Burstable
```

### Top Command

The **kubectl top** command provides real-time CPU and memory metrics of the pods and nodes.

> To use the kubectl top command, you need to have the **metrics-server** running in your cluster.

The **metrics-server** collects resource usage metrics from the kubelet on each node and provides this data to the K8s API.

![Resource Request & Limit 2]({{ site.baseurl }}/assets/img/k8s-course/request-limit-2.jpg)

```bash
# Lets check the webserver pod metrics using the top command
vagrant@controlplane:~$ kubectl top pod webserver
NAME        CPU(cores)   MEMORY(bytes)
webserver   0m           5Mi
# CPU(cores): The amount of CPU currently being used by the Pod, measured in millicores (m).
# MEMORY(bytes): The amount of memory currently being used by the Pod, measured in bytes.
```

Let's check the CPU and memory consumption of all the pods in the **kube-system** namespace:

```bash
vagrant@controlplane:~$ kubectl top pods -n kube-system
NAME                                   CPU(cores)   MEMORY(bytes)
coredns-66bc5c9577-qh56x               5m           17Mi
coredns-66bc5c9577-tf5wj               5m           16Mi
etcd-controlplane                      119m         65Mi
kube-apiserver-controlplane            198m         482Mi
kube-controller-manager-controlplane   93m          63Mi
kube-proxy-hwzhf                       1m           30Mi
kube-proxy-jjv2v                       1m           30Mi
kube-proxy-qnrl8                       146m         29Mi
kube-scheduler-controlplane            24m          25Mi
metrics-server-6cbd6b9dd4-jrgwl        8m           20Mi

# To list pods from all the namespaces use the --all-namespaces or -A flag
```

### Filtering & Sorting

You can filter the pods using labels. We deployed the webserver pod with the label **app=web**.

```bash
# try the top command using the labels
vagrant@controlplane:~$ kubectl top pod -l type=web
NAME        CPU(cores)   MEMORY(bytes)
webserver   0m           5Mi

# Using the --containers flag, you can check the resource usage of individual containers inside the pod
vagrant@controlplane:~$ kubectl top pod -l type=web --containers
POD         NAME                CPU(cores)   MEMORY(bytes)
webserver   nginx-webserver     0m           5Mi
webserver   sidecar-container   0m           0Mi
```

You can also **sort** the list by CPU & Memory usage in descending order.

```bash
# sort by cpu usage
vagrant@controlplane:~$ kubectl top pods --sort-by=cpu -n kube-system
NAME                                   CPU(cores)   MEMORY(bytes)
kube-apiserver-controlplane            127m         511Mi
etcd-controlplane                      70m          66Mi
kube-controller-manager-controlplane   47m          62Mi
kube-proxy-qnrl8                       34m          32Mi
kube-proxy-jjv2v                       29m          34Mi
kube-scheduler-controlplane            17m          24Mi
metrics-server-6cbd6b9dd4-jrgwl        6m           20Mi
coredns-66bc5c9577-qh56x               3m           17Mi
coredns-66bc5c9577-tf5wj               3m           16Mi
kube-proxy-hwzhf                       1m           32Mi

# sort by memory usage
vagrant@controlplane:~$ kubectl top pods --sort-by=memory -n kube-system
NAME                                   CPU(cores)   MEMORY(bytes)
kube-apiserver-controlplane            121m         511Mi
etcd-controlplane                      64m          66Mi
kube-controller-manager-controlplane   46m          62Mi
kube-proxy-jjv2v                       25m          31Mi
kube-proxy-hwzhf                       1m           30Mi
kube-proxy-qnrl8                       52m          30Mi
kube-scheduler-controlplane            15m          24Mi
metrics-server-6cbd6b9dd4-jrgwl        5m           20Mi
coredns-66bc5c9577-qh56x               3m           17Mi
coredns-66bc5c9577-tf5wj               3m           16Mi
```

### Checking Nodes Resource Usage

You can use the kubectl top command to check the CPU and memory usage of all nodes in your K8s cluster.

```bash
vagrant@controlplane:~$ kubectl top nodes
NAME           CPU(cores)   CPU(%)   MEMORY(bytes)   MEMORY(%)
controlplane   310m         15%      1474Mi          79%
node01         29m          2%       1372Mi          73%
node02         43m          4%       894Mi           48%

# check the resource usage of a specific node
vagrant@controlplane:~$ kubectl top node controlplane
NAME           CPU(cores)   CPU(%)   MEMORY(bytes)   MEMORY(%)
controlplane   273m         13%      1475Mi          79%
```

---

### Resource Quota

In a real-world project, a K8s cluster can be used by a single team or multiple teams. To ensure that the end users or teams do not use up all the resources in the cluster, K8s offers a namespace-scoped object called **ResourceQuota**.

Using **ResourceQuota** we set hard limits on the resources a namespace can use, like CPU and memory. This ensures that each team only uses their fair share of resources and the cluster remains stable.

### Creating Resource Quota

```bash
# lets first create a namespace
vagrant@controlplane:~$ kubectl create ns backend-apps
namespace/backend-apps created
```

Using **ResourceQuota** we limit various resources on the backend-apps namespace.

```bash
# Create backend-apps-quota.yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: backend-apps-quota
  namespace: backend-apps
spec:
  hard:
    requests.cpu: "4"
    requests.memory: "8Gi"
    limits.cpu: "8"
    limits.memory: "16Gi"
    pods: "10"
    persistentvolumeclaims: "5"
    requests.storage: "100Gi"
    services: "10"
    configmaps: "20"
    secrets: "20"
    replicationcontrollers: "10"
    resourcequotas: "5"
# hard: Defines the hard limits for the resources
```

```bash
# create the Resource Quota
vagrant@controlplane:~$ kubectl apply -f backend-app-quota.yaml
resourcequota/backend-apps-quota created

# The short-form for Resource Quota object is quota
# List the Resource Quotas in backend-apps-namespace
vagrant@controlplane:~$ kubectl get quota -n backend-apps
NAME                 REQUEST                                                                                                                                                                                                             LIMIT                                    AGE
backend-apps-quota   configmaps: 1/20, persistentvolumeclaims: 0/5, pods: 0/10, replicationcontrollers: 0/10, requests.cpu: 0/4, requests.memory: 0/8Gi, requests.storage: 0/100Gi, resourcequotas: 1/5, secrets: 0/20, services: 0/10   limits.cpu: 0/8, limits.memory: 0/16Gi   77s
```

```bash
# describe the backend-apps-quota
vagrant@controlplane:~$ kubectl describe quota backend-apps-quota -n backend-apps
Name:                   backend-apps-quota
Namespace:              backend-apps
Resource                Used  Hard
--------                ----  ----
configmaps              1     20
limits.cpu              0     8
limits.memory           0     16Gi
persistentvolumeclaims  0     5
pods                    0     10
replicationcontrollers  0     10
requests.cpu            0     4
requests.memory         0     8Gi
requests.storage        0     100Gi
resourcequotas          1     5
secrets                 0     20
services                0     10
```

What happens if you try to create an object that exceeds the quota limit?

```bash
# Create quota-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: quota-pod
  namespace: backend-apps
spec:
  containers:
  - name: busybox
    image: busybox
    command: ["sh", "-c", "sleep infinity"]
    resources:
      requests:
        cpu: "5" # This exceeds the requests.cpu quota of "4"
        memory: "9Gi" # This exceeds the requests.memory quota of "8Gi"
      limits:
        cpu: "10" # This exceeds the limits.cpu quota of "8"
        memory: "17Gi" # This exceeds the limits.memory quota of "16Gi"
```

```bash
# Deploy the pod
vagrant@controlplane:~$ kubectl apply -f quota-pod.yaml
Error from server (Forbidden): error when creating "quota-pod.yaml": pods "quota-pod" is forbidden: exceeded quota: backend-apps-quota, requested: limits.cpu=10,limits.memory=17Gi,requests.cpu=5,requests.memory=9Gi, used: limits.cpu=0,limits.memory=0,requests.cpu=0,requests.memory=0, limited: limits.cpu=8,limits.memory=16Gi,requests.cpu=4,requests.memory=8Gi
```

As you can see, it throws an exceeded quota error.

### Resource Quota Scopes

Let's say you have a use case where you don't want to allow any pods with the **BestEffort QoS** class in a namespace. In this scenario, you can use **ResourceQuota** scopes to limit the **BestEffort pod** deployment.

ResourceQuota supports the following scopes:

- **Terminating:** Applies to pods that have an activeDeadlineSeconds in its specification. For example, jobs, cronjobs etc.
- **NotTerminating:** Applies to pods that do not have an active deadline. For example, regular pods without a specified termination period.
- **NotBestEffort:** Applies to pods that have resource requests or limits specified. For example, pods that are not classified under the BestEffort QoS class.
- **PriorityClass:** Applies to pods with a specific priority class. For example, pods with the PriorityClass class parameter.

```bash
# cleanup the created resources
vagrant@controlplane:~$ kubectl delete quota backend-apps-quota -n backend-apps
resourcequota "backend-apps-quota" deleted from backend-apps namespace
```

---

### Limit Ranges

**LimitRanges** are namespace-scoped objects that set the minimum and maximum resources, such as CPU and memory, that a container or pod can request or use within a namespace.

You can also set namespace's minimum and maximum storage requests for each **PersistentVolumeClaim**.

### Creating a LimitRange

Let's create a LimitRange named **dev-apps-limits** in the **backend-apps namespace**.

```bash
# Create limit-ranges.yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: dev-apps-limits
  namespace: backend-apps
spec:
  limits:
  - max:
      cpu: "2"
      memory: "2Gi"
    min:
      cpu: "200m"
      memory: "256Mi"
    default:
      cpu: "500m"
      memory: "512Mi"
    defaultRequest:
      cpu: "300m"
      memory: "300Mi"
    maxLimitRequestRatio:
      cpu: "4"
      memory: "4"
    type: Container
```

```bash
# Create the LimitRange
vagrant@controlplane:~$ kubectl apply -f limit-ranges.yaml
limitrange/dev-apps-limits created

# Verify the LimitRange
vagrant@controlplane:~$ kubectl get limitrange -n backend-apps
NAME              CREATED AT
dev-apps-limits   2026-07-27T03:35:30Z

# describe the LimitRange to view the limits
vagrant@controlplane:~$ kubectl describe limitrange dev-apps-limits -n backend-apps
Name:       dev-apps-limits
Namespace:  backend-apps
Type        Resource  Min    Max  Default Request  Default Limit  Max Limit/Request Ratio
----        --------  ---    ---  ---------------  -------------  -----------------------
Container   memory    256Mi  2Gi  300Mi            512Mi          4
Container   cpu       200m   2    300m             500m           4
```

### Testing LimitRange

Let's see what happens if we create a container that exceeds the LimitRange.

```bash
# Create limit-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: limit-test-pod
  namespace: backend-apps
spec:
  containers:
  - name: busybox
    image: busybox
    command: ["sh", "-c", "sleep infinity"]
    resources:
      requests:
        cpu: "3"
        memory: "3Gi"
      limits:
        cpu: "3"
        memory: "3Gi"
```

Apply the configuration and you will see that the pod creation fails because it exceeds the limit range.

```bash
vagrant@controlplane:~$ kubectl apply -f limit-pod.yaml
Error from server (Forbidden): error when creating "limit-pod.yaml": pods "limit-test-pod" is forbidden: [maximum memory usage per Container is 2Gi, but limit is 3Gi, maximum cpu usage per Container is 2, but limit is 3]
```

At a higher level, the Limit Range for resources works like this:

- If a Pod is created in a namespace without resource requests and limits, the default values of resources defined in a LimitRange created in the same namespace will automatically be assigned to the Pod.
- If a Pod has smaller resource requests and limits than the minimum values defined in a LimitRange, the Pod will not be created.
- If a Pod has higher resource requests and limits than the maximum values defined in a Limit Range, the Pod will not be created.
- If a Pod is already running and we make changes to the LimitRange configuration, there will be no effect on that Pod.


### Key Difference Between Resource Quota and Limit Range

![Resource Request & Limit 3]({{ site.baseurl }}/assets/img/k8s-course/request-limit-3.jpg)

---

### Capacity Planning Real World Example

When you work on a real-world project, cluster capacity planning is an important part where you need to determine the required cluster capacity to host the applications.

If you don't plan your cluster capacity requirements, you will end up wasting cluster resources, which will translate into higher costs in managing the clusters.

### Example

Let’s say you are managing a K8s cluster that runs an e-commerce application.

This application has multiple microservices such as a **front-end service**, a **product catalog service**, a **user authentication service**, and a **payment processing service**. Each of these services has different resource requirements.

### Understand Resource Requirements

Before applications are deployed into the production cluster, they go through performance testing. In this test, the team performs CPU benchmarking and gathers pod memory metrics to identify the pod running requirements in production.

Based on the performance test, teams determine the total resource requirement for the clusters, leaving enough buffer resources for immediate pod scaling requirements.

### Choosing Node Types

Once you know the resource requirements of your applications, you can select node types that balance performance, cost, and scalability.

You need to choose an instance type based on your requirements. For example:

- **Machine Learning Workloads:** Require high CPU performance, large amounts of memory, and possibly GPU support.
- **Web Servers:** Typically need a balance of CPU and memory.
- **High-Performance Computing:** Requires high CPU performance and low latency. Compute-optimized instances.
- **In-Memory Databases:** Demand large amounts of memory and fast access speeds.

You can check the node's allocatable resources using the following command:

```bash
vagrant@controlplane:~$ kubectl get node -ojson | jq '.items[].status.allocatable'
{
  "cpu": "2",
  "ephemeral-storage": "29317393565",
  "hugepages-2Mi": "0",
  "memory": "1908584Ki",
  "pods": "110"
}
{
  "cpu": "1",
  "ephemeral-storage": "29317393565",
  "hugepages-2Mi": "0",
  "memory": "1908840Ki",
  "pods": "110"
}
{
  "cpu": "1",
  "ephemeral-storage": "29317393565",
  "hugepages-2Mi": "0",
  "memory": "1908840Ki",
  "pods": "110"
}
```

### Monitor and Adjust

Use K8s monitoring tools like **Prometheus** and **Grafana** to monitor resource usage. Adjust the resource requests and limits based on the actual usage and performance metrics.

Tools like **Kubecost** provides real-time cost visibility and insights for teams using K8s.
