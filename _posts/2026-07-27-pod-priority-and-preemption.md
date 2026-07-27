---
layout: post
title: "Pod Priority & Preemption"
date: 2026-07-27
categories: [k8s]
---

## Introduction

Pod Priority is a Kubernetes scheduling feature that allows Kubernetes to make scheduling decisions by comparing pods based on priority numbers.

Since K8s cluster is a distributed system, pods sometimes need to move between nodes due to issues like node failures, upgrades, or maintenance.

In these situations, it's important to ensure that mission critical applications continue running. Less critical applications may need to be removed (evicted) temporarily to free resources.

When resources are limited:

- Which pod gets scheduled first?
- Which pods should be evicted?

**Pod Priority** helps manage this effectively. It tells the scheduler how important a pod is compared to others. This helps K8s schedule the right pods and evict less critical ones when needed.

---

### What is Pod priority & Priority Class

Let's look at the two main concepts in pod priority.

1. **Pod Preemption:** If a high-priority Pod cannot be scheduled due to lack of resources, K8s may evict (preempt) lower-priority Pods to free up resources.
2. **Pod Priority Class:** A cluster scoped object that allows you to set a priority for a pod using a value. It comes under scheduling.k8s.io/v1 API.

```bash
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: high-priority-apps     # Name of the PriorityClass
value: 100000                  # Numerical priority (higher = more important)
globalDefault: false           # Not the default for all Pods
description: "Priority class for backend"
```

> The **globalDefault** field in a PriorityClass determines whether that PriorityClass should be used as the default priority for pods that don't explicitly specify a PriorityClassName.

- When you create a pod without a PriorityClassName, it will automatically get the default PriorityClass assigned
- Only one PriorityClass in the entire cluster can have globalDefault: true at any time.

Here's a simple example pod spec showing an nginx pod assigned the priority class **high-priority-apps**.

```bash
apiVersion: v1
kind: Pod
metadata:
  name: nginx                  # Pod name
  labels:
    env: dev                   # Label to mark environment as development
spec:
  containers:
  - name: web                  # Container name
    image: nginx:latest        # Nginx image
    imagePullPolicy: IfNotPresent  # Pull image only if not already present
  priorityClassName: high-priority-apps  # Pod gets its priority from this PriorityClass
```

### Default High-Priority Classes

How do you safeguard **system-critical pods** (CoreDNS, CNI controllers etc) from preemption?

1. **system-node-critical:** This class has a value of 2000001000. Static pods like etcd, kube-apiserver, kube-scheduler, and controller-manager use this priority class.
2. **system-cluster-critical:** This class has a value of 2000000000. Addon pods like CoreDNS, Calico controller, metrics-server, etc., use this priority class.

```bash
# the following command lists the default PriorityClasses
vagrant@controlplane:~$ kubectl get priorityclass
NAME                      VALUE        GLOBAL-DEFAULT   AGE   PREEMPTIONPOLICY
system-cluster-critical   2000000000   false            53d   PreemptLowerPriority
system-node-critical      2000001000   false            53d   PreemptLowerPriority
```

---

### Pod Priority & Preemption Workflow

![Pod Priority & Preemption]({{ site.baseurl }}/assets/img/k8s-course/pod-priority.png)

1. When a pod is deployed with a **priorityClassName**, the priority admission controller assigns a numerical priority value based on that class.
2. The scheduler then organizes pods in the scheduling queue based on their priorities, placing higher-priority pods ahead of lower-priority ones.
3. If no nodes have enough resources to schedule a high-priority pod, K8s uses preemption. This means it removes (evicts) lower-priority pods from nodes to make space for the higher-priority pod.
4. Evicted pods get a default termination grace period of 30 seconds. If pods specify their own terminationGracePeriodSeconds, that value overrides the default.
5. If a higher priority pod cannot be scheduled (even after potential preemption attempts), the scheduler will continue and try to schedule other lower priority pods.

---

### Working With PriorityClasses

Lets say you're running an e-commerce platform with these components:

- Payment processing service (critical)
- Product catalog service (high importance)
- Recommendation engine (medium importance)
- Analytics collector (low importance)

Let's create **PriorityClasses** to represent different priority levels.

1. Critical - 1000000
2. High importance - 100000
3. Medium importance - 10000
4. Low importance - 1000

We will set **globalDefault: true** for the **Low importance** PriorityClass.

```bash
# Create a priorityclass.yaml file
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: critical-priority
value: 1000000
globalDefault: false
description: "Critical services that must not be disrupted"

---
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: high-priority
value: 100000
globalDefault: false
description: "High priority services - important for business operations"

---
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: medium-priority
value: 10000
globalDefault: false
description: "Medium priority services - enhancements to core functionality"

---
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: low-priority
value: 1000
globalDefault: true
description: "Low priority services - batch processing, reporting, etc."
```

```bash
# Deploy the manifest
vagrant@controlplane:~$ kubectl apply -f priorityclass.yaml
priorityclass.scheduling.k8s.io/critical-priority created
priorityclass.scheduling.k8s.io/high-priority created
priorityclass.scheduling.k8s.io/medium-priority created
priorityclass.scheduling.k8s.io/low-priority created

# List the PriorityClasses
# pc is the short-form for PriorityClass object
vagrant@controlplane:~$ kubectl get pc
NAME                      VALUE        GLOBAL-DEFAULT   AGE   PREEMPTIONPOLICY
critical-priority         1000000      false            36s   PreemptLowerPriority
high-priority             100000       false            36s   PreemptLowerPriority
low-priority              1000         true             36s   PreemptLowerPriority
medium-priority           10000        false            36s   PreemptLowerPriority
system-cluster-critical   2000000000   false            53d   PreemptLowerPriority
system-node-critical      2000001000   false            53d   PreemptLowerPriority
# you can see that the low-priority class is set as the global default
```

### Deploy a Pod With Critical-Priority

Here's a simple **payment-pod.yaml** file that creates an nginx pod with the PriorityClass set to **critical-priority**.

```bash
apiVersion: v1
kind: Pod
metadata:
  name: payment-pod
spec:
  priorityClassName: critical-priority
  containers:
  - name: payment-api
    image: nginx

# Deploy the pod
vagrant@controlplane:~$ kubectl apply -f payment-pod.yaml
pod/payment-pod created

# describe the pod, to see the critical priority assigned to the pod
vagrant@controlplane:~$ kubectl describe pod payment-pod | grep priority
Priority Class Name:  critical-priority
```

### Validating Default Priority Class

Now, let's deploy a sample pod without a priority class and see if it gets the default priority class.

```bash
# Deploy a webserver pod and verify it
vagrant@controlplane:~$ kubectl run webserver --image=nginx
pod/webserver created

# verify pod status
vagrant@controlplane:~$ kubectl get pods
NAME                                            READY   STATUS    RESTARTS       AGE
webserver                                       1/1     Running   0              17s

# 
vagrant@controlplane:~$ kubectl describe pod webserver | grep priority
Priority Class Name:  low-priority
```

```bash
# Clean up the pods 
vagrant@controlplane:~$ kubectl delete pod payment-pod webserver
pod "payment-pod" deleted from default namespace
pod "webserver" deleted from default namespace

# Clean up PriorityClasses
vagrant@controlplane:~$ kubectl delete pc critical-priority high-priority low-priority medium-priority
priorityclass.scheduling.k8s.io "critical-priority" deleted
priorityclass.scheduling.k8s.io "high-priority" deleted
priorityclass.scheduling.k8s.io "low-priority" deleted
priorityclass.scheduling.k8s.io "medium-priority" deleted
```

---

### How is Pod QoS related to Pod Priority & Preemption?n

Pod priority based preemption happens only when high-priority pods are waiting in the scheduling queue. In this case, the scheduler removes lower-priority pods without considering their QoS class.

- **QoS-based eviction:** Handled by kubelet during resource shortages (no scheduling queue involved).
- **Priority-based preemption:** Handled by the scheduler when higher-priority pods wait in the scheduling queue (QoS is ignored).

### What is the effect of PodDisruptionBudget on Preemption?

**PodDisruptionBudget** (PDB) limits how many pods K8s can evict (remove) at the same time during planned actions like upgrades or maintenance.

When the scheduler considers pods for preemption, it respects PDB constraints where possible. It tries not to violate a PDB by ensuring the minimum number of pods specified in the PDB remains running.

However, if a very high-priority pod needs to be scheduled, K8s will still remove lower-priority pods, even if it violates the PDB rules.

### How to Handle Evictions Gracefully?

When a pod is being removed (preemption or eviction), we must make sure the application shuts down smoothly.

- **Add graceful shutdown handlers to application code:** These catch the **SIGTERM** signal and do things like:
  - Close database connections
  - Finish ongoing requests or tasks
  - Save data in memory or clear caches
- **Set a proper shutdown time:** In the Pod settings, set a **terminationGracePeriodSeconds**. The default is 30 seconds, but you can change it based on what your app needs.
- **Use preStop hooks (Container Lifecycle Hooks):** This lets you run custom cleanup steps before the pod shuts down.
- **Add readiness probes:** These help make sure traffic doesn't go to a pod that’s shutting down or not ready.
- **Test your setup:** Try eviction and shutdown cases in a test environment to see how your app behaves.

---

### Real-World Case Study

Lets explore a real-world outage happened at **Grafana Cloud** and understand how K8s Pod Priority and Preemption play an important role in ensuring cluster stability.

### Incident Background

On July 19, 2019, Grafana Cloud experienced a ~30-minute outage in their Hosted multi-tenant Prometheus service.

It is because of misconfigured Pod Priorities during the deployment of a new cluster led to the eviction of critical production pods.

It impacted production Ingester pods (responsible for processing metrics) because it got preempted, causing a temporary cascading failure in data ingestion.

### Cortex Cluster

Grafana uses **Cortex**, an open-source system to run **multi-tenant** Prometheus at scale. Cortex has components called Ingesters, which temporarily hold metrics data before saving it to long-term storage.

To perform zero-downtime Cortex cluster upgrades, they need to temporarily run extra Ingester pods (~25% extra resources in the cluster).

### The Intended Solution: Pod Priority

To make space for these temporary extra Ingesters without permanently over-provisioning cluster resources, the team decided to use K8s Pod Priority and Preemption.

1. **Goal:** Designate Ingesters as high priority.
2. **Mechanism:** If an extra high priority Ingester needs scheduling, K8s should be able to preempt (evict) less important, medium or low priority pods.
3. **Expectation:** The preempted pods would reschedule elsewhere on the cluster where smaller pockets of resources exist, creating a large enough space for the high-priority Ingester.

### The Configuration Steps & The Mistake

To implement Pod Priority, the team defined critical, high, medium, and low PriorityClasses cluster-wide.

However, they made two critical mistakes.

1. **Critical Mistake 1:** They set medium as the default priority for any pod that didn't explicitly define one.
2. **Critical Mistake 2:** They rolled out these new PriorityClasses before updating their existing, critical production Ingester deployments to use the high priority setting. The production Ingesters effectively had no specific priority set, making them lower than the new medium default.

Meaning, any pod deployed in the cluster automatically gets medium priority, which is higher than the Ingester pods.

### The Incident Trigger

With these configurations, for a new large customer, Grafana team deployed a new dedicated Cortex cluster in the production.

This new deployment also didn't have any pod priorities specified in its configuration yet. Therefore, its pods (including its own Ingesters) defaulted to medium priority.

The cluster didn't have enough free resources to schedule these new medium priority Ingesters without impacting existing pods.

### How Things Went Wrong (The Cascade)

The Ku8s scheduler saw the incoming new medium priority Ingesters. To make space, it looked for lower-priority pods to preempt. It found the existing production Ingesters (which had no priority set, thus lower than medium).

A production Ingester was preempted. Its ReplicaSet immediately tried to create a replacement pod to maintain the desired count.

This newly created replacement production Ingester pod also didn't have an explicit priority set (because the production deployment config hadn't been updated). It therefore defaulted to medium priority.

This new medium priority production Ingester pod now potentially needed to preempt another existing, lower-priority (no priority set) production Ingester to get scheduled.

This cycle repeated, causing a cascading failure where the production Ingesters that preempted each other until all were disrupted.

Since the Ingesters were stateful and required a quorum to function, losing them all resulted in a ~30-minute outage for the monitoring service.

### Key Takeaways on Pod Priority

Setting a cluster-wide default PriorityClass is powerful but risky. Ensure you understand the implications before deploying it, especially regarding existing workloads.

When implementing priorities, update your most critical workloads to use the high or critical priorities first, before changing defaults or deploying lower-priority workloads that might compete for resources.
