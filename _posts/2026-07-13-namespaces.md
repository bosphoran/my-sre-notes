---
layout: post
title: "Namespaces"
date: 2026-07-13
categories: [k8s]
---

## Introduction

A namespace in a K8s cluster is a logical partition that lets you divide cluster resources into isolated, manageable groups.

- **Namespace-scoped objects:** These are K8s resources that belong to a specific namespace and are isolated within that namespace. For example, pods, ReplicaSets, Deployment, etc.
- **Cluster-scoped objects:** These are K8s resources that are not confined to any specific namespace and have a global scope across the entire cluster, allowing them to be accessed and managed from any namespace. For example, ClusterRoleBinding, ClusterRole, IngressClass, StorageClass, etc.

```bash
vagrant@controlplane:~$ kubectl api-resources --namespaced=true
NAME                              SHORTNAMES       APIVERSION        NAMESPACED   KIND
bindings                                           v1                true         Binding
configmaps                        cm               v1                true         ConfigMap
endpoints                         ep               v1                true         Endpoints
events                            ev               v1                true         Event
limitranges                       limits           v1                true         LimitRange
persistentvolumeclaims            pvc              v1                true         PersistentVolumeClaim
pods                              po               v1                true         Pod
podtemplates                                       v1                true         PodTemplate
replicationcontrollers            rc               v1                true         ReplicationController
resourcequotas                    quota            v1                true         ResourceQuota
secrets                                            v1                true         Secret
serviceaccounts                   sa               v1                true         ServiceAccount
services                          svc              v1                true         Service
# ...
# 
vagrant@controlplane:~$ kubectl api-resources --namespaced=false
NAME                                SHORTNAMES       APIVERSION                          NAMESPACED   KIND
componentstatuses                   cs               v1                                  false        ComponentStatus
namespaces                          ns               v1                                  false        Namespace
nodes                               no               v1                                  false        Node
persistentvolumes                   pv               v1                                  false        PersistentVolume
mutatingwebhookconfigurations                        admissionregistration.k8s.io/v1     false        MutatingWebhookConfiguration
validatingadmissionpolicies                          admissionregistration.k8s.io/v1     false        ValidatingAdmissionPolicy
validatingadmissionpolicybindings                    admissionregistration.k8s.io/v1     false        ValidatingAdmissionPolicyBinding
validatingwebhookconfigurations                      admissionregistration.k8s.io/v1     false        ValidatingWebhookConfiguration
customresourcedefinitions           crd,crds         apiextensions.k8s.io/v1             false        CustomResourceDefinition
apiservices                                          apiregistration.k8s.io/v1           false        APIService
# ...
```

Let's look at the four key features of the namespace object.

1. Isolation and Logical Grouping:
   - Namespaces enable logical grouping of related resources based on different criteria, such as application components, environments, or teams.
2. Resource Management:
   - Namespaces allow efficient resource allocation and control by setting resource quotas (CPU & Memory) and limits for each namespace.
3. Security:
   - Namespaces enhance security by providing an additional layer of access control and allowing different security policies to be applied to each namespace.
4. Authorization and Access Control:
   - Namespaces allow fine-grained authorization and role-based access control (RBAC) rules to be defined at the namespace level.

### Default Namespace

Every cluster has a built-in default namespace. When you don't specify a namespace, the resources are deployed in the default namespace.

```bash
vagrant@controlplane:~$ kubectl get ns
NAME              STATUS   AGE
calico-system     Active   43d
default           Active   43d
keda              Active   10d
kube-node-lease   Active   43d
kube-public       Active   43d
kube-system       Active   43d
monitoring        Active   6d23h
tigera-operator   Active   43d
```

You can list all resources associated with a specific namespace.

```bash
vagrant@controlplane:~$ kubectl get pods -n calico-system
NAME                                      READY   STATUS    RESTARTS      AGE
calico-apiserver-986dd8459-8mjhx          1/1     Running   13 (8h ago)   38d
calico-apiserver-986dd8459-wz7xc          1/1     Running   13 (8h ago)   39d
calico-kube-controllers-58cc4895f-dwtwn   1/1     Running   15 (8h ago)   39d
calico-node-28vlb                         1/1     Running   30 (8h ago)   43d
calico-node-j5j48                         1/1     Running   34 (8h ago)   43d
calico-node-q5z82                         1/1     Running   20 (8h ago)   43d
calico-typha-6c8c7956fd-n72m4             1/1     Running   14 (8h ago)   39d
calico-typha-6c8c7956fd-xmpl9             1/1     Running   15 (8h ago)   38d
csi-node-driver-4cmnl                     2/2     Running   39 (8h ago)   43d
csi-node-driver-h96wh                     2/2     Running   38 (8h ago)   43d
csi-node-driver-vscfx                     2/2     Running   37 (8h ago)   43d
goldmane-8b794fbc6-2jb4w                  1/1     Running   33 (8h ago)   39d
whisker-d6f6bb5ff-d7fd7                   2/2     Running   26 (8h ago)   39d
```

### Working With Namespaces

Let's create a namespace called **dev-env**.

```bash
vagrant@controlplane:~$ kubectl create ns dev-env
namespace/dev-env created

# Create namespace using declarative command
apiVersion: v1
kind: Namespace
metadata:
  name: dev-env
```

To associate an object with a specific namespace, you must use the **-n <namespace-name>** option.

```bash
vagrant@controlplane:~$ kubectl run webserver --image=nginx -n dev-env
pod/webserver created

# 
vagrant@controlplane:~$ kubectl get pods -n dev-env
NAME        READY   STATUS    RESTARTS   AGE
webserver   1/1     Running   0          68s
```

You can also specify the namespace parameter in the metadata section of the object's YAML file.

```bash
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  namespace: dev-env
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
```

You can setup any namespace as the current one using the **kubectl** command. Once set, you won't need to explicitly include the **-n <namespace>** option with every command.

```bash
vagrant@controlplane:~$ kubectl config set-context --current --namespace=dev-env
Context "kubernetes-admin@kubernetes" modified.

# dont need to specify namespace, since, dev-env is set as default namespace
vagrant@controlplane:~$ kubectl get pods
NAME        READY   STATUS    RESTARTS   AGE
webserver   1/1     Running   0          3m37s

# switch back to the default namespace
vagrant@controlplane:~$ kubectl config set-context --current --namespace=default
Context "kubernetes-admin@kubernetes" modified.
```

You can edit the namespace using the following command.

```bash
vagrant@controlplane:~$ kubectl edit ns dev-env
```

When you **delete** a namespace, it deletes all the resources associated with the namespace.

```bash
vagrant@controlplane:~$ kubectl delete ns dev-env
namespace "dev-env" deleted
```

> You cannot delete the **default namespace**, since, it is a protected namespace that is essential for the functioning of the K8s cluster.
