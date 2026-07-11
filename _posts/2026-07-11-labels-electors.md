---
layout: post
title: "Labels and Selectors"
date: 2026-07-11
categories: [k8s]
---

## Introduction

**Labels** are **key–value pairs** attached to K8s objects (Pods, Deployments, Services, etc.).
They describe or categorize objects.

```bash
labels:
  app: frontend
  tier: web
  env: prod
```

**Selectors** are queries that match labels. They allow K8s controllers to find the right Pods.

```bash
selector:
  matchLabels:
    app: frontend
```

- **Labels:** tags on objects
- **Selectors:** rules that pick objects based on those tags

> Objects such as **Replicasets**, **Deployments**, **Services**, **StatefulSets**, etc., make use of selectors to identify the pods that match the labels specified in the selector.

![Labels & Selectors]({{ site.baseurl }}/assets/img/k8s-course/labels-selectors.jpg)

### Working With Labels & Selectors

Let's create three pods (frontend, backend, and database) with different labels. We will use four labels: **name, env, tier, and component**, with different values.

To add labels to pods, we will use the **-l flag** followed by a set of labels (key-value pairs).

```bash
vagrant@controlplane:~$ kubectl run frontend --image=nginx -l name=nginx,env=prod,tier=frontend,component=webserver
pod/frontend created

vagrant@controlplane:~$ kubectl run backend --image=nginx -l name=java,env=prod,tier=backend,component=appserver
pod/backend created

vagrant@controlplane:~$ kubectl run database --image=nginx -l name=mysql,env=prod,tier=database,component=dbserver
pod/database created
```

Lets check the labels.

```bash
vagrant@controlplane:~$ kubectl get pods
NAME                                            READY   STATUS    RESTARTS        AGE
backend                                         1/1     Running   0               15m
database                                        1/1     Running   0               15m
frontend                                        1/1     Running   0               15m
```

### Querying Using Labels & Selectors

```bash
vagrant@controlplane:~$ kubectl get pods -l env=prod
NAME       READY   STATUS    RESTARTS   AGE
backend    1/1     Running   0          16m
database   1/1     Running   0          16m
frontend   1/1     Running   0          16m

# Use the selector flag and get the same output
vagrant@controlplane:~$ kubectl get po --selector env=prod
NAME       READY   STATUS    RESTARTS   AGE
backend    1/1     Running   0          17m
database   1/1     Running   0          17m
frontend   1/1     Running   0          18m
```

You can use both the label and selector flags to list the pods as given below. The **-L tier** displays the value of the tier.

```bash
vagrant@controlplane:~$ kubectl get po --selector 'tier in (frontend, database)' -L tier
NAME       READY   STATUS    RESTARTS   AGE   TIER
database   1/1     Running   0          20m   database
frontend   1/1     Running   0          20m   frontend

vagrant@controlplane:~$ kubectl get po -l 'tier in (frontend,database)' -L tier
NAME       READY   STATUS    RESTARTS   AGE   TIER
database   1/1     Running   0          22m   database
frontend   1/1     Running   0          22m   frontend
```

### Modify Labels

Let's add a label named **version=1.0.2** to the frontend pod.

```bash
vagrant@controlplane:~$ kubectl label pod frontend version=1.0.2
pod/frontend labeled

# Let's verify the label.
vagrant@controlplane:~$ kubectl get pod -l version -L version
NAME       READY   STATUS    RESTARTS   AGE   VERSION
frontend   1/1     Running   0          24m   1.0.2
```

What if we want to add a release version to the frontend, backend, and database pods? First we **filters** all the pods with the **env=prod** label and then adds a label named **release=v1.0**.

```bash
vagrant@controlplane:~$ kubectl label pod -l env=prod release=v1.0
pod/backend labeled
pod/database labeled
pod/frontend labeled

# Verify release
vagrant@controlplane:~$ kubectl get pod -l release -L  release
NAME       READY   STATUS    RESTARTS   AGE   RELEASE
backend    1/1     Running   0          27m   v1.0
database   1/1     Running   0          27m   v1.0
frontend   1/1     Running   0          27m   v1.0

# Update release
vagrant@controlplane:~$ kubectl label pod -l env=prod release=v2.0 --overwrite
pod/backend labeled
pod/database labeled
pod/frontend labeled

# Verify updated release
vagrant@controlplane:~$ kubectl get pod -l release -L  release
NAME       READY   STATUS    RESTARTS   AGE   RELEASE
backend    1/1     Running   0          28m   v2.0
database   1/1     Running   0          27m   v2.0
frontend   1/1     Running   0          28m   v2.0
```

### Deleting Labels

You can delete the labels from objects by adding a **hyphen (-)** to the label name

```bash
# Delete the release label
vagrant@controlplane:~$ kubectl label po frontend release-
pod/frontend unlabeled
```

Now let's filter the backend and database tiers and delete the release label.

```bash
vagrant@controlplane:~$ kubectl label  pod -l 'tier in (backend, database)' release-
pod/backend unlabeled
pod/database unlabeled
```
