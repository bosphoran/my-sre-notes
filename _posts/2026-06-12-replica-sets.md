---
layout: post
title: "ReplicaSets in K8s"
date: 2026-06-12
categories: [k8s]
---

## Introduction

A ReplicaSet is a Kubernetes object that ensures a specified number of identical pods are always running.

If a pod fails or is evicted, the ReplicaSet automatically creates a new one to maintain the desired state.

ReplicaSet uses a selector to track the pods it is responsible for. A selector is a set of key-value pairs.

> The ReplicaSet object is managed by the **ReplicaSet controller**. It is a background process in K8s that constantly monitors **ReplicaSets**. Meaning, it queries the **API server** for the number of pods whose labels match the ReplicaSet selector.

![ReplicaSets]({{ site.baseurl }}/assets/img/k8s-course/replicasets.jpg)

### Replicaset vs. Pod

- **Pod**: Represents a single instance of an application container, with its own dedicated IP and resources.
- **Replicaset**: Manages a set of identical Pods, ensuring a desired number are always running.

Lets create a **replicaset** that deploys three replicas (instances) of a **nginx** webserver pod.

```bash
# replicaset.yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: webserver-replicaset
spec:
  replicas: 3
  selector:
    matchLabels:
      app: webserver-3
  template:
    metadata:
      labels:
        app: webserver-3
    spec:
      containers:
      - name: webserver-container-3
        image: nginx
```

For **replicaset**, the **apiVersion** is **apps/v1** and the **kind** is **ReplicaSet**. It has majorly three sections under the spec field.

- **Replicas**: It defines the desired number of pods.
- **Selector**: It is used to identify which pods the ReplicaSet should manage. The **matchLabels** under the selector field should be the same as the pod labels.
- **Template**: It has the pod template. The pod template defines the underlying Pod configuration.

Let's apply this manifest file.

```bash
# Apply replicaset yaml file
vagrant@controlplane:~$ kubectl apply -f replicaset.yaml
replicaset.apps/webserver-replicaset created

# View ReplicaSets
vagrant@controlplane:~$ kubectl get replicaset
NAME                   DESIRED   CURRENT   READY   AGE
webserver-replicaset   3         3         3       21s

# View pods
vagrant@controlplane:~$ kubectl get pods
NAME                         READY   STATUS      RESTARTS         AGE
webserver-replicaset-khlrw   1/1     Running     0                38s
webserver-replicaset-ljn9s   1/1     Running     0                38s
webserver-replicaset-qmbc7   1/1     Running     0                38s
```

If you delete a pod, replicaset will create a new pod with the same configuration.

```bash
vagrant@controlplane:~$ kubectl delete pod webserver-replicaset-qmbc7
pod "webserver-replicaset-qmbc7" deleted from default namespace

# ReplicaSet created the pod webserver-replicaset-7hh26
vagrant@controlplane:~$ kubectl get pods
NAME                         READY   STATUS      RESTARTS         AGE
webserver-replicaset-7hh26   1/1     Running     0                11s
webserver-replicaset-khlrw   1/1     Running     0                34m
webserver-replicaset-ljn9s   1/1     Running     0                34m
```

We can **scale up** or **scale down** a replicaSet using **kubectl scale** command:

```bash
# Scale up replicaSet to 5 replicas
vagrant@controlplane:~$ kubectl scale replicaset webserver-replicaset --replicas=5
replicaset.apps/webserver-replicaset scaled

# Verify number of replicaSets
vagrant@controlplane:~$ kubectl get replicaset
NAME                   DESIRED   CURRENT   READY   AGE
webserver-replicaset   5         5         5       37m

# Verify number of pod replicas
vagrant@controlplane:~$ kubectl get pods
NAME                         READY   STATUS      RESTARTS       AGE
webserver-replicaset-7hh26   1/1     Running     0              3m53s
webserver-replicaset-khlrw   1/1     Running     0              37m
webserver-replicaset-ljn9s   1/1     Running     0              37m
webserver-replicaset-thhjc   1/1     Running     0              19s
webserver-replicaset-wx4ln   1/1     Running     0              19s
```

You can delete a ReplicaSet either by deleting the yaml file or using the **kubectl delete** command.

```bash
vagrant@controlplane:~$ kubectl delete replicaset webserver-replicaset
replicaset.apps "webserver-replicaset" deleted from default namespace
```

> In real-world projects, **ReplicaSets** are managed as part of a Deployment object.
