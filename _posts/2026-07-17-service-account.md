---
layout: post
title: "Service Accounts"
date: 2026-07-17
categories: [k8s]
---

## Introduction

Service account is a K8s resource that gives pods access to API Server.

Service account acts as an identity for processes running inside pods, providing them with the necessary credentials and permissions to authenticate and access specific resources within the cluster.

- When you, as a user, access a K8s cluster, you typically authenticate through your user account defined in the kubeconfig file.
- An applications running inside K8s pods, need a secure mechanism to authenticate and interact with the K8s **API server**. This is where K8s service accounts come into play.

> By associating a pod with a K8s service account, you provide the application running inside the pod with the necessary credentials and permissions to authenticate and access specific resources in the API server.

When a service account is created, K8s automatically generates a **token secret** associated with that service account.
The **token secret** contains a **JWT** (JSON Web Token) that is used to authenticate the service account to the K8s API server. The **token secret** is automatically mounted into pods that use the service account.

![Service Account]({{ site.baseurl }}/assets/img/k8s-course/service-account.png)

Let's say you have a **Python application** running inside a pod that needs to retrieve the list of pods running in the **kube-system namespace**. To grant the application access to the **API server**, you need to attach a service account to the pod that has access to list pods in the kube-system namespace.

Another scenario could involve an external **monitoring application** that requires cluster API access to **fetch cluster metrics**. In this case, the external application can make use of the **token** in the service account and make **HTTP calls** to the **API server** using the token as an authentication parameter.

![Service Account 2]({{ site.baseurl }}/assets/img/k8s-course/service-account-2.jpg)

---

### The Default Service Account

If you don't associate a **service account** with a pod, Kubernetes automatically assigns the **default service account** to it. The default service account is created by K8s itself for each namespace.

```bash
# To check the service accounts
vagrant@controlplane:~$ kubectl get sa
NAME                            SECRETS   AGE
default                         0         43d
my-release-prometheus-adapter   0         10d
```

Let's create a pod and check the service account.

```bash
vagrant@controlplane:~$ kubectl run test-pod --image=busybox --command sleep infinity
pod/test-pod created

# default ServiceAccount is added to this pod.
vagrant@controlplane:~$ kubectl describe po test-pod | grep -i "service account"
Service Account:  default
```

When the service account is attached to a pod, a **service account token** gets mounted as a file inside the pod's file system. To check this, you can describe the pod:

```bash
vagrant@controlplane:~$ kubectl describe po test-pod | grep -i -A1 "mounts"
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-qwh5w (ro)
```

We can see that the token is mounted to the **/var/run/secrets/kubernetes.io/serviceaccount** directory from a volume named **kube-api-access-jwvgd**.

The name **kube-api-access-jwvgd** is an automatically generated name for the volume that holds the service account token and credentials. The name is unique for each pod and is used to identify the specific volume.

Let's list the files under the **/var/run/secrets/kubernetes.io/serviceaccount** directory.

```bash
vagrant@controlplane:~$ kubectl exec test-pod -- ls /var/run/secrets/kubernetes.io/serviceaccount
ca.crt
namespace
token
```

You can view the toke contents:

```bash
vagrant@controlplane:~$ kubectl exec test-pod -- cat /var/run/secrets/kubernetes.io/serviceaccount/token
eyJhbGciOiJSUzI1NiIsImtpZCI6IllBbHhNRFJIZWRORDEteU5ORkplcHZlNTQzcUNTMER1QkloZEFmS2FMVW8ifQ.
# ...
```

The default service account doesn't have any permissions. It is created due to the way k8s is designed.

---

### Creating Service Account

We must create custom service accounts specifically for pods.

```bash
vagrant@controlplane:~$ kubectl create sa app-sa
serviceaccount/app-sa created

# We can describe it or get it in YAML format.
vagrant@controlplane:~$ kubectl get sa app-sa -o yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  creationTimestamp: "2026-07-17T12:11:21Z"
  name: app-sa
  namespace: default
  resourceVersion: "911537"
  uid: 9c3a6e00-450e-4a88-9952-b57dae05c4c8
```

This service account will have the same access and permissions as the default service account (no access). To grant permissions to service accounts, we need to use **Roles** and **ClusterRoles**.


To add a service account to the Pods, the s**erviceAccountName** parameter should be added under the pod **spec** section.

Let's create a **Redis** pod using redis image and attach the app-sa service account we previously created.

```bash
vagrant@controlplane:~$ kubectl run redis --image=redis --overrides='{"spec":{ "serviceAccountName":"app-sa" }}' --dry-run=client -o yaml > redis.yaml
```

> The **--overrides** flag enables you to supply a JSON or YAML string to modify fields in the generated resource definition. Parameters not supported by **dry-run** can be utilized through --overrides. For the **CKA exam**, you can generate the YAML and then edit it directly.

After modification, our **redis.yaml** file should look like the following:

```bash
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    run: redis
  name: redis
spec:
  containers:
  - image: redis
    name: redis
    resources: {}
  serviceAccountName: app-sa
  dnsPolicy: ClusterFirst
  restartPolicy: Always
status: {}
```

Let's deploy the pod and validate the service account details.

```bash
vagrant@controlplane:~$ kubectl apply -f redis.yaml
pod/redis created

# Verify pod
vagrant@controlplane:~$ kubectl get pods
NAME                                            READY   STATUS    RESTARTS     AGE
redis                                           1/1     Running   0            51s

# Check service account
vagrant@controlplane:~$ kubectl describe po redis | grep -i "service account"
Service Account:  app-sa
```

```bash
vagrant@controlplane:~$ kubectl delete pod redis
pod "redis" deleted from default namespace
kubevagrant@controlplane:~$ kubectl delete sa app-sa
serviceaccount "app-sa" deleted from default namespace
```
