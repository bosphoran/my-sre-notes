---
layout: post
title: "Horizontal Pod Autoscaler"
date: 2026-06-15
categories: [k8s]
---

## Introduction

Scaling in K8s means adjusting the amount of **computing resources** your application uses so it can handle changes in demand.

There are two types resource scalling in K8s:

- **Horizontal Pod Scaling** (adding more servers).
- **Vertical Pod Scaling** (adding more power to a server)

![HPA]({{ site.baseurl }}/assets/img/k8s-course/hpa.jpg)

### Horizontal Pod Autoscaler (HPA)

Automatically **increases** or **decreases** the number of **Pod replicas** in a Deployment, ReplicaSet, or StatefulSet.

**How HPA Works:**

- **Monitors** metrics like **CPU/memory** usage (via Metrics Server).
- If usage exceeds a threshold (e.g., 70% CPU), it adds more Pods.
- If usage drops, it scales down Pods.

> If your application needs more power to handle increased load, **HPA** will add more pods (**scale up**); if the load drops, HPA will remove pods (**scale down**) to save resources​.

### How Does HPA Work?

HPA is a K8s resource managed by **HPA controller**.

If you want to scale a Deployment based on load, you create an HPA object that targets that Deployment.

```bash
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: my-app-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: my-app   # reference only
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
```

The **HPA controller** keeps checking the workload’s (pods) metrics and adjusts the number of replicas in the target based on the load.

![HPA Controller]({{ site.baseurl }}/assets/img/k8s-course/hpa-controller.gif)

---

**Metric Collection:**

For HPA to work, it needs metrics like **CPU** and **memory** usage of the pods.

These metrics are provided by the **Metrics Server**. If Metrics Server is not installed or providing data, HPA will not be able to retrieve metrics and will not scale your pods.

> Metrics Server is an add-on component in the cluster. When you install the metrics server, it registers the **metrics.k8s.io/v1beta1** API. The **HPA controller** uses the Metrics API to check the current resource usage of the pods.

**Desired Replica Calculation:**

**HPA** compares the current metrics to the target threshold you set.

For example, you might set a target of **50% CPU utilization**. If the current average CPU use across pods is higher than 50%, HPA will decide to **scale up** pods.

The following **formula** to decide the number of replicas:

> **desiredReplicas** = **ceil**(currentReplicas **×** currentMetricValue **/** desiredMetricValue)

For example, if you have 3 pods, the current CPU usage is 80%, and the target (desired) is 50%:

**desiredReplicas** = **ceil**(3 **×** 80 **/** 50) **→** **ceil**(4.8) **=** 5.

So the **HPA controller** updates the Deployment’s **spec.replicas to 5**.

---

**When CPU or Memory Usage is Below the Target:**

**Scaling down** starts when the average CPU or memory usage drops below the target value set in the HPA.

For example, if your target CPU usage is 50%, and the actual usage goes down to 30%, HPA will start to reduce the number of pods.

- he HPA controller runs in a loop, regularly checking the resource usage of pods.
- **Scale-up** happens until it reaches the **maxReplicas**.
- **Scale down** happens until it reaches the **minReplicas** value.
- HPA controller checks metrics **every 15 seconds** by default. It waits at least **5 minutes** before scaling down.

---

### How Does Metrics Server Work

**Metrics Server** supplies the live **CPU & Memory** data, collected from each node’s **kubelet**, that K8s needs for autoscaling and quick resource checks.

- The **Metrics Server** continuously scrapes resource usage data (**CPU-Memory usage**) from individual Kubelets running on each node in the cluster.
- **Kubelet** directly interacts with the underlying operating system and hardware monitoring tools to gather various metrics.
- Inside the Kubelet, a tool called **cAdvisor** collects container-level metrics by talking to the **container runtime** (like Containerd or CRI-O).
- The collected data is then aggregated by the Metrics Server and exposed through the K8s **API server** using the Metrics API.
- K8s controllers like HPA & VPA use the Metrics API to get the current CPU/memory usage and scale workloads up or down as needed.

> **Metrics Server** scrapes kubelets **→** aggregates data **→** exposes it via the Metrics API, enabling tools like HPA and kubectl top to access real‑time CPU/memory usage.

![Metrics Server]({{ site.baseurl }}/assets/img/k8s-course/metrics-server.gif)

---

### Working With HPA

To install Metrics Server on a Linux Kubernetes cluster, the standard command is:

```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

Once installed, you should be able to see node or pod resource usage.

```bash
vagrant@controlplane:~$ kubectl top nodes
NAME           CPU(cores)   CPU(%)   MEMORY(bytes)   MEMORY(%)
controlplane   647m         32%      1401Mi          75%
node01         57m          5%       1085Mi          58%
node02         28m          2%       1018Mi          54%
```

**Autoscaling an NGINX Deployment:**

Lets create an **sre-webserver.yaml** deployment YAML with **nginx** image.

```bash
apiVersion: apps/v1
kind: Deployment
metadata:
  name: sre-webserver
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        ports:
        - containerPort: 80
        resources:
          requests:
            cpu: "100m"
            memory: "128Mi"
          limits:
            cpu: "250m"
            memory: "256Mi"
```

```bash
vagrant@controlplane:~$ kubectl apply -f sre-webserver.yaml
deployment.apps/sre-webserver created

#
vagrant@controlplane:~$ kubectl get deploy
NAME            READY   UP-TO-DATE   AVAILABLE   AGE
sre-webserver   1/1     1            1           40s

# 
vagrant@controlplane:~$ kubectl get pods -l app=nginx
NAME                             READY   STATUS    RESTARTS   AGE
sre-webserver-7554bc5d7b-kzd74   1/1     Running   0          48s
```

For our test, let’s also expose this deployment as a service so that we can easily send traffic to it.

```bash
vagrant@controlplane:~$ kubectl expose deploy sre-webserver --port=80 --target-port=80
service/sre-webserver exposed
```

Now we will create a **HorizontalPodAutoscaler** that targets the **sre-webserver** deployment.

> In this example, we are using **apiVersion: autoscaling/v1**, which only supports **CPU-based autoscaling**. If you want to scale based on memory or other custom metrics, you’ll need to use **apiVersion: autoscaling/v2**. We’ll cover that in the next lesson.

We want to scale the deployment between **1 and 5 replicas**, aiming for an average CPU utilization of **50%**.

Create **sre-webserver-hpa.yaml**.

```bash
apiVersion: autoscaling/v1
kind: HorizontalPodAutoscaler
metadata:
  name: sre-webserver-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: sre-webserver
  minReplicas: 1
  maxReplicas: 5
  targetCPUUtilizationPercentage: 50
```

```bash
# Apply the HPA
vagrant@controlplane:~$ kubectl apply -f sre-webserver-hpa.yaml
horizontalpodautoscaler.autoscaling/sre-webserver-hpa created
```

Alternatively, you can accomplish the same in one line with the kubectl autoscale command (imperative approach).

```bash
vagrant@controlplane:~$ kubectl autoscale deploy sre-webserver --min=1 --max=6 --cpu=50% --name=sre-webserver-hpa
horizontalpodautoscaler.autoscaling/sre-webserver-hpa autoscaled
```

Verify HPA

```bash
vagrant@controlplane:~$ kubectl get hpa
NAME                REFERENCE                  TARGETS       MINPODS   MAXPODS   REPLICAS   AGE
sre-webserver-hpa   Deployment/sre-webserver   cpu: 0%/50%   1         6         1          22s
```

**Lets generate load to test the HPA**.

We’ll run a simple load generator using **hey** utility that continuously makes **HTTP requests** to the NGINX service, causing CPU usage to rise.

```bash
vagrant@controlplane:~$ kubectl run load-generator --image=techiescamp/hey:latest --restart=Never -- -c 100 -q 50 -z 60m http://sre-webserver.default.svc.cluster.local
pod/load-generator created
```

Once the **load generator** is running, it will start hitting the NGINX pod constantly. Give it a little time (30 seconds to a minute) to generate sustained CPU load.

```bash
vagrant@controlplane:~$ kubectl get hpa
NAME                REFERENCE                  TARGETS         MINPODS   MAXPODS   REPLICAS   AGE
sre-webserver-hpa   Deployment/sre-webserver   cpu: 238%/50%   1         6         4          5m35s
```

As you can see, the CPU usage has reached 238%.

Lets check our pod status and observe how HPA has created 5 extra pods.

```bash
vagrant@controlplane:~$ kubectl get pods -l app=nginx
NAME                             READY   STATUS    RESTARTS   AGE
sre-webserver-7554bc5d7b-bsf9q   1/1     Running   0          2m5s
sre-webserver-7554bc5d7b-jr2x2   1/1     Running   0          2m5s
sre-webserver-7554bc5d7b-jwg8f   1/1     Running   0          110s
sre-webserver-7554bc5d7b-kzd74   1/1     Running   0          15m
sre-webserver-7554bc5d7b-n5rrs   1/1     Running   0          33s
sre-webserver-7554bc5d7b-r7jnz   1/1     Running   0          2m5s
```

We can monitor this in **real time** using the following command.

```bash
vagrant@controlplane:~$ kubectl get hpa --watch
NAME                REFERENCE                  TARGETS        MINPODS   MAXPODS   REPLICAS   AGE
sre-webserver-hpa   Deployment/sre-webserver   cpu: 60%/50%   1         6         6          13m
sre-webserver-hpa   Deployment/sre-webserver   cpu: 59%/50%   1         6         6          13m
sre-webserver-hpa   Deployment/sre-webserver   cpu: 55%/50%   1         6         6          13m
sre-webserver-hpa   Deployment/sre-webserver   cpu: 70%/50%   1         6         6          13m
```

Now stop the load by deleting the **load-generator** pod.

```bash
vagrant@controlplane:~$ kubectl delete pod load-generator
pod "load-generator" deleted from default namespace
```

After the load stops, CPU usage will fall.

> By default, scale-down is more cautious. HPA waits ~5 mins and ensure metrics stay low before removing pods.

Now, delete the **sre-webserver-hpa**.

```bash
vagrant@controlplane:~$ kubectl delete hpa sre-webserver-hpa
horizontalpodautoscaler.autoscaling "sre-webserver-hpa" deleted from default namespace
```

---

### Memory Based Scaling

For memory metrics, you need to use the updated API version **autoscaling/v2**.

In **autoscaling/v2**, you can define multiple metrics for scaling. That’s why the metrics section is written as a list in the YAML file, with each item having the type Resource.

Create the **webserver-hpa-v2.yaml** yaml file.

```bash
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: webserver-hpa-v2
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: sre-webserver
  minReplicas: 1
  maxReplicas: 5
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 50
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 50
```

Lets apply the **HPA**.

```bash
# Create HPA-V2
vagrant@controlplane:~$ kubectl apply -f webserver-hpa-v2.yaml
horizontalpodautoscaler.autoscaling/webserver-hpa-v2 created

# Verify HPA status
vagrant@controlplane:~$ kubectl get hpa
NAME               REFERENCE                  TARGETS                       MINPODS   MAXPODS   REPLICAS   AGE
webserver-hpa-v2   Deployment/sre-webserver   cpu: 0%/50%, memory: 4%/50%   1         5         5          48s
```

In the output, you can see the **HPA** has both **CPU** and **memory** as targets.

To avoid unnecessary and frequent scale up or scale down **("thrashing" or "flapping")**, **HPA v2** supports a configurable parameter called **stabilizationWindowSeconds**.

> **stabilizationWindowSeconds** defines a time window during which the autoscaler will consider historical metrics before making scaling decisions.

The **default values** for **stabilizationWindowSeconds** are:

- For **scaling up**: Shorter window (0 seconds by default)
- For **scaling down**: Longer window (300 seconds/5 minutes by default)

**For example**, if your workload experiences a brief **30-second** spike in CPU usage, but your downscale stabilization window is set to 300 seconds, HPA won't immediately scale down when the spike ends. It will wait to ensure the reduced load persists before removing pods.


Lets, remove the resources from the cluster.

```bash
# Remove webserver-hpa-v2
vagrant@controlplane:~$ kubectl delete hpa webserver-hpa-v2
horizontalpodautoscaler.autoscaling "webserver-hpa-v2" deleted from default namespace

# Remove Deployment sre-webserver
vagrant@controlplane:~$ kubectl delete deployment sre-webserver
deployment.apps "sre-webserver" deleted from default namespace
```

---

### Custom Metrics Scaling

In real-world deployments running scalable systems, CPU/memory metrics don't always correlate with actual application load. Therefore, it makes sense to scale the apps based on what matters most to your specific application.

- Request count per second
- Queue length
- Number of active sessions
- Business-specific metrics (like the number of orders placed)

> **Custom metrics** let K8s autoscale based on application‑level signals (like queue length) instead of just CPU/memory, making scaling smarter and more aligned with real workload needs.

Let's say you have a **payment-processing** service running on K8s. You already added an **HPA** that should add more pods when CPU usage climbs above **80%**.

A pod might handle 20 transactions per second at 50-60% CPU utilization. When transaction volume increases to 100/second, the application doesn't consume more CPU percentage - it simply processes at its maximum rate and queues the rest.

**The pod's CPU remains at 50-60% while the queue grows**.

Since CPU never crosses the 80% threshold, the HPA doesn’t scale up, leading to customer delays.

**Solution:**

- Define a custom metric (e.g., **transaction_queue_length**).
- Expose it via a monitoring system (Prometheus, custom adapter).
- Configure HPA to use this metric instead of CPU.
- Pods scale up when the queue grows, and scale down when it shrinks.

### Implementing Custom Metrics

To autoscale on custom application metrics (like **transaction_queue_length** or **request latency**), K8s needs an additional API endpoint: **custom.metrics.k8s.io***

- This endpoint is not built in.
- It is usually provided by a Custom Metrics Adapter (for example, **Prometheus Adapter**).
- Once installed, the HPA can query those custom metrics through the K8s API, just like it queries CPU/memory from Metrics Server.

Fresh cluster only has **metrics.k8s.io** (CPU/memory).

```bash
vagrant@controlplane:~$ kubectl api-versions | grep metrics
metrics.k8s.io/v1beta1
```

Let’s see how you’d actually enable custom metrics in K8s using **Prometheus Adapter**:

```bash
# Install Helm
vagrant@controlplane:~$ curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

# Verify Helm
vagrant@controlplane:~$ helm version
version.BuildInfo{Version:"v3.21.2"

# Add Prometheus Community Repo
vagrant@controlplane:~$ helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
"prometheus-community" has been added to your repositories

# Update Helm Repositories
vagrant@controlplane:~$ helm repo update
Hang tight while we grab the latest from your chart repositories...
...Successfully got an update from the "prometheus-community" chart repository
Update Complete. ⎈Happy Helming!⎈

# Install Prometheus Adapter
vagrant@controlplane:~$ helm install my-release prometheus-community/prometheus-adapter
my-release-prometheus-adapter has been deployed.

# Verify Metrics APIs
vagrant@controlplane:~$ kubectl api-versions | grep metrics
custom.metrics.k8s.io/v1beta1
metrics.k8s.io/v1beta1
```

From this moment on, the **HPA** can read any custom metric the adapter exposes.

![Custom Metrics]({{ site.baseurl }}/assets/img/k8s-course/custom-metrics.png)

- **Application instrumentation** → Your app exposes metrics at /metrics using a Prometheus client library.
- **Prometheus scraping** → Prometheus collects and stores those metrics.
- **Prometheus Adapter** → Runs in the cluster, queries Prometheus with PromQL, and exposes results via the custom.metrics.k8s.io API.
- **HPA integration** → The Horizontal Pod Autoscaler (using autoscaling/v2) can reference these custom metrics to scale Pods based on application‑level signals instead of just CPU/memory.

> The /metrics endpoint isn’t native to Pods — it’s created when you add a Prometheus client library to your application. Prometheus then scrapes that endpoint to collect custom metrics.

Here is an example HPA config that uses http_requests custom metrics from prometheus. If each pod is handling more than 0.5 http_requests, the HPA will try to scale up.

```bash
kind: HorizontalPodAutoscaler
apiVersion: autoscaling/v2
metadata:
  name: sre-web
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: sre-web
  minReplicas: 1
  maxReplicas: 10
  metrics:
  - type: Pods
    pods:
      metric:
        name: http_requests
      target:
        type: Value
        averageValue: 500m
```

Now, lets apply the custom-metrics.yaml HPA.

```bash
vagrant@controlplane:~$ kubectl apply -f custom-metrics.yaml
horizontalpodautoscaler.autoscaling/sre-web created

# Verify HPA
vagrant@controlplane:~$ kubectl get hpa
NAME      REFERENCE            TARGETS          MINPODS   MAXPODS   REPLICAS   AGE
sre-web   Deployment/sre-web   <unknown>/500m   1         10        3          18m
```

> **<unknown>** means the HPA can’t fetch the metric value. You need to ensure your app exposes http_requests, Prometheus scrapes it, and the Adapter maps it correctly into **custom.metrics.k8s.io**.

Adding a **Prometheus client library** to your application code is what makes **/metrics** available.

---

### Scaling With External Metrics

The External Metrics API in K8s provides a way to access metrics from systems outside of the K8s cluster for making autoscaling decisions. For this, we can make use of **KEDA**.

> **KEDA** (Kubernetes Event-driven Autoscaling) is an open-source tool for K8s that provides event-driven autoscaling. **KEDA** doesn't replace **HPA**. It builds upon and extends the standard K8s Horizontal Pod Autoscaler (HPA) v2 with **type: External**.

**KEDA** scales based on external signals or events.

- The number of messages in a message queue (like SQS, Kafka, Azure Service Bus, etc)
- The length of a list in a Redis cache
- Metrics from systems like Prometheus, Datadog, etc.

**Enabling External Metrics API:**

When you install **KEDA**, it automatically registers the **external.metrics.k8s.io** API group with the API server.

```bash
# Add KEDA repo
vagrant@controlplane:~$ helm repo add kedacore https://kedacore.github.io/charts
"kedacore" has been added to your repositories

# Update Helm repository
vagrant@controlplane:~$ helm repo update

# Install KEDA helm chart
vagrant@controlplane:~$ helm install keda kedacore/keda --namespace keda --create-namespace

# Verify external metrics server
vagrant@controlplane:~$ kubectl api-versions | grep metrics
custom.metrics.k8s.io/v1beta1
external.metrics.k8s.io/v1beta1
metrics.k8s.io/v1beta1
```
