---
layout: post
title: "Roles & ClusterRoles"
date: 2026-07-18
categories: [k8s]
---

## Introduction

**Roles & ClusterRoles** are the key ways to manage authorization in K8s, following the **principle of least privilege**.

> The principle of least privilege states that users and applications should have only the minimum permissions necessary to perform their intended functions.

Let's take a real-world example where you are running K8s clusters in production. You will have users like **developers, DevOps engineers performance engineers**, and support teams accessing these clusters. Also, external applications like **CI/CD and monitoring tools** need access to the cluster.

Using **Roles and ClusterRoles**, you can assign granular permissions to users and applications, specifying what they can and cannot access in the cluster.

### Roles & RoleBinding

Roles and RoleBindings are namespace-scoped, meaning they are created within specific namespaces.

Let's consider a scenario where a developer needs read access to pods and deployments within a specific namespace. 

Lets see how can the permissions defined in a Role be applied to the user?

This is where **RoleBinding** comes into the picture. As the name suggests, it binds the Role to a specific user, group, or service account.

![Roles & ClusterRoles]({{ site.baseurl }}/assets/img/k8s-course/roles-clusterroles.gif)

> Roles and RoleBindings have a one-to-many relationship, which means that a single Role can be used with multiple RoleBindings.

 A Role is tied to a specific namespace and cannot be referenced or used in other namespaces. A Role created in namespace A be used in namespace B?

### ClusterRole and ClusterRoleBindings

To give access to users, groups, or any service account across the entire cluster, we use **clusterRoles** and **clusterRoleBindings**.

**ClusterRole** is similar to a Role except for the **cluster-wide scope**. When you add read access to the pod in a ClusterRole, it means read access to all the pods in all the namespaces on the cluster.

> **ClusterRoleBinding** binds a ClusterRole to a specific user, group, or service account, granting the permissions defined in the ClusterRole.

---

### Understanding Role & RoleBinding YAML Specification

### Role YAML

Here is a simple Role specification named developer-role bound to default namespace.

```bash
apiVersion: rbac.authorization.k8s.io/v1   # RBAC API group and version
kind: Role                                 # Defines a Role (namespaced permissions)
metadata:
  namespace: default                       # Role applies only within the "default" namespace
  name: developer-role                     # Name of the Role
rules:                                     # Each rule has three sections: apiGroups, resources, and verbs.
- apiGroups: [""]                          # "" = core API group (Pods, Services, ConfigMaps, etc.)
  resources: ["pods"]                      # This Role applies to the "pods" resource
  verbs: ["list"]                          # Allowed action: "list" (view all Pods in the namespace)
```

This Role lets whoever it’s bound to see the list of Pods in the default namespace, but nothing more.

### Understanding apiGroups

Assume you want to create a role for a developer that gives read access only to Deployments in a namespace. For this, you should know which API group the Deployment belongs to.

There are two categories of apiGroups groups in Kubernetes.

**1. Core API Groups:** The Empty apiGroups **[""]** refers to the core API group. It contains the fundamental resource types that are essential for Kubernetes operations. For example, pods, nodes, namespaces, PersistentVolumes etc.

You can list all the core API group resources using the following command.

```bash
vagrant@controlplane:~$ kubectl api-resources --api-group=""
NAME                     SHORTNAMES   APIVERSION   NAMESPACED   KIND
bindings                              v1           true         Binding
componentstatuses        cs           v1           false        ComponentStatus
configmaps               cm           v1           true         ConfigMap
endpoints                ep           v1           true         Endpoints
events                   ev           v1           true         Event
limitranges              limits       v1           true         LimitRange
namespaces               ns           v1           false        Namespace
nodes                    no           v1           false        Node
persistentvolumeclaims   pvc          v1           true         PersistentVolumeClaim
persistentvolumes        pv           v1           false        PersistentVolume
pods                     po           v1           true         Pod
podtemplates                          v1           true         PodTemplate
replicationcontrollers   rc           v1           true         ReplicationController
resourcequotas           quota        v1           true         ResourceQuota
secrets                               v1           true         Secret
serviceaccounts          sa           v1           true         ServiceAccount
services                 svc          v1           true         Service
```

**2. Named API Groups:** A named API group is a way to organize and categorize related API resources.
Some commonly used named API groups in Kubernetes include:

- apps: Contains resources related to workload management, such as Deployments, Daemonsets, ReplicaSets, and StatefulSets.
- batch: Contains resources related to batch operations, such as Jobs and CronJobs.
- storage.k8s.io: Contains resources related to storage, such as StorageClass, PersistentVolume, and PersistentVolumeClaim.
- rbac.authorization.k8s.io: Contains resources related to role-based access control, such as Roles, RoleBindings, ClusterRoles, and ClusterRoleBindings.
- networking.k8s.io: Contains resources related to networking, such as Ingress and NetworkPolicy.

You can list all the named API group resources using the following commands.

```bash
vagrant@controlplane:~$ kubectl api-resources --api-group="apps"
NAME                  SHORTNAMES   APIVERSION   NAMESPACED   KIND
controllerrevisions                apps/v1      true         ControllerRevision
daemonsets            ds           apps/v1      true         DaemonSet
deployments           deploy       apps/v1      true         Deployment
replicasets           rs           apps/v1      true         ReplicaSet
statefulsets          sts          apps/v1      true         StatefulSet

# 
vagrant@controlplane:~$ kubectl api-resources --api-group="batch"
NAME       SHORTNAMES   APIVERSION   NAMESPACED   KIND
cronjobs   cj           batch/v1     true         CronJob
jobs                    batch/v1     true         Job

# 
vagrant@controlplane:~$ kubectl api-resources --api-group="storage.k8s.io"
NAME                      SHORTNAMES   APIVERSION          NAMESPACED   KIND
csidrivers                             storage.k8s.io/v1   false        CSIDriver
csinodes                               storage.k8s.io/v1   false        CSINode
csistoragecapacities                   storage.k8s.io/v1   true         CSIStorageCapacity
storageclasses            sc           storage.k8s.io/v1   false        StorageClass
volumeattachments                      storage.k8s.io/v1   false        VolumeAttachment
volumeattributesclasses   vac          storage.k8s.io/v1   false        VolumeAttributesClass

# 
vagrant@controlplane:~$ kubectl api-resources --api-group="rbac.authorization.k8s.io"
NAME                  SHORTNAMES   APIVERSION                     NAMESPACED   KIND
clusterrolebindings                rbac.authorization.k8s.io/v1   false        ClusterRoleBinding
clusterroles                       rbac.authorization.k8s.io/v1   false        ClusterRole
rolebindings                       rbac.authorization.k8s.io/v1   true         RoleBinding
roles                              rbac.authorization.k8s.io/v1   true         Role

# 
vagrant@controlplane:~$ kubectl api-resources --api-group="networking.k8s.io"
NAME              SHORTNAMES   APIVERSION             NAMESPACED   KIND
ingressclasses                 networking.k8s.io/v1   false        IngressClass
ingresses         ing          networking.k8s.io/v1   true         Ingress
ipaddresses       ip           networking.k8s.io/v1   false        IPAddress
networkpolicies   netpol       networking.k8s.io/v1   true         NetworkPolicy
servicecidrs                   networking.k8s.io/v1   false        ServiceCIDR
```

---

### RoleBinding YAML

Here is a **RoleBinding YAML** specification that binds the Developer role to a user, group, and a serviceaccount

```bash
apiVersion: rbac.authorization.k8s.io/v1   # RBAC API group and version
kind: RoleBinding                          # Binds a Role to subjects (users, groups, service accounts)
metadata:
  name: developer-role-binding             # Name of the RoleBinding
  namespace: default                       # Scope: applies only within the "default" namespace
subjects:                                  # Entities that will get the Role's permissions
- kind: User                               # A user identity
  name: sre                                # Username "sre" will inherit the Role's permissions
  apiGroup: rbac.authorization.k8s.io      # API group for RBAC subjects
- kind: Group                              # A group identity
  name: devops                             # Group "devops" will inherit the Role's permissions
  apiGroup: rbac.authorization.k8s.io
- kind: ServiceAccount                     # A service account identity
  name: app-service-account                # ServiceAccount "app-service-account" in "default" namespace
  namespace: default
roleRef:                                   # Reference to the Role being bound
  kind: Role                               # Binding to a Role (namespaced)
  name: developer-role                     # The Role name (must exist in the same namespace)
  apiGroup: rbac.authorization.k8s.io      # RBAC API group
```

This RoleBinding grants read-only Pod listing rights in the default namespace to a specific user, a group, and a service account.

A role binding has two important sections.

1. **subjects:** The subjects section defines the users, groups, and service accounts that will be granted the permissions defined in the role.
2. **roleRef:** It establishes the link between the subjects and the role itself. It acts as the glue that binds the specified subjects to the permissions defined in the role.

---

### Working With Roles and Role Bindings

Let's look at how we can add roles to grant permission to the **SRE** user to the cluster resources.

### Create a Role

Let's create a Role named **kube-system-reader** that gives read access (get, list, watch) to pods, Deployments, Daemonsets, Jobs, and ClusterRoles in the kube-system namespace.

Since Role is namespace-scoped, we must create the Role in the kube-system namespace to grant access to resources within the kube-system namespace.

Create a file named **sre-role.yaml** and copy the following contents.

```bash
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: kube-system-reader
  namespace: kube-system
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list", "watch"]
- apiGroups: ["apps"]
  resources: ["deployments", "daemonsets"]
  verbs: ["get", "list", "watch"]
- apiGroups: ["batch"]
  resources: ["jobs"]
  verbs: ["get", "list", "watch"]
- apiGroups: ["rbac.authorization.k8s.io"]
  resources: ["clusterroles"]
  verbs: ["get", "list", "watch"]
```

Now, let's create this Role.

```bash
vagrant@controlplane:~/sre-user$ kubectl apply -f sre-role.yaml
role.rbac.authorization.k8s.io/kube-system-reader created

# Let's validate the role.
vagrant@controlplane:~/sre-user$ kubectl get roles -n kube-system | grep system-reader
kube-system-reader                               2026-07-18T08:35:29Z
```

Now, if you describe the role, you can see the **apiGroups** and **permissions** we added to the role.

```bash
vagrant@controlplane:~/sre-user$ kubectl describe role kube-system-reader -n kube-system
Name:         kube-system-reader
Labels:       <none>
Annotations:  <none>
PolicyRule:
  Resources                               Non-Resource URLs  Resource Names  Verbs
  ---------                               -----------------  --------------  -----
  pods                                    []                 []              [get list watch]
  daemonsets.apps                         []                 []              [get list watch]
  deployments.apps                        []                 []              [get list watch]
  jobs.batch                              []                 []              [get list watch]
  clusterroles.rbac.authorization.k8s.io  []                 []              [get list watch]
```

### Create RoleBinding

Now that we have a role, the next step is to bind the **SRE user** to the kube-system-reader role.

For this, we will create another K8s object called **RoleBinding**. RoleBinding object binds the user to a specific role.

Create a file named **sre-role-binding.yaml** and copy the following contents.

```bash
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: kube-system-reader-role-binding
  namespace: kube-system
subjects:
- kind: User
  name: sre
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role 
  name: kube-system-reader
  apiGroup: rbac.authorization.k8s.io
```

Let's create the RoleBinding

```bash
vagrant@controlplane:~/sre-user$ kubectl apply -f sre-role-binding.yaml
rolebinding.rbac.authorization.k8s.io/kube-system-reader-role-binding created

# Validate the RoleBinding. It lists the RoleBinding and associated role.
vagrant@controlplane:~/sre-user$ kubectl get rolebinding -n kube-system | grep kube-system-reader
kube-system-reader-role-binding           Role/kube-system-reader        35s
```

Describe the RoleBinding. It will show the role and associated user details.

```bash
vagrant@controlplane:~/sre-user$ kubectl describe rolebindings kube-system-reader-role-binding -n kube-system
Name:         kube-system-reader-role-binding
Labels:       <none>
Annotations:  <none>
Role:
  Kind:  Role
  Name:  kube-system-reader
Subjects:
  Kind  Name  Namespace
  ----  ----  ---------
  User  sre
```

Now that the Role and RoleBinding are created, we can test user access to these resources.

```bash
vagrant@controlplane:~/sre-user$ kubectl --kubeconfig=sre-config get pods -n kube-system
NAME                                   READY   STATUS    RESTARTS          AGE
coredns-66bc5c9577-qh56x               1/1     Running   14 (6h29m ago)    39d
coredns-66bc5c9577-tf5wj               1/1     Running   14 (6h29m ago)    39d
etcd-controlplane                      1/1     Running   16 (6h29m ago)    39d
kube-apiserver-controlplane            1/1     Running   22 (6h29m ago)    40d
kube-controller-manager-controlplane   1/1     Running   121 (5h51m ago)   40d
kube-proxy-hwzhf                       1/1     Running   16 (4h50m ago)    40d
kube-proxy-jjv2v                       1/1     Running   16 (5h50m ago)    40d
kube-proxy-qnrl8                       1/1     Running   16 (6h29m ago)    40d
kube-scheduler-controlplane            1/1     Running   121 (6h29m ago)   40d
metrics-server-6cbd6b9dd4-jrgwl        1/1     Running   0                 5h45m
```

The **user SRE** is able to list the pods in the kube-system namespace

We have granted read access to specific objects only in the kube-system namespace. The **user SRE** should not have any other access to the clusters. To verify this, we will use the kubeconfig named **sre-config** we have previously created.

```bash
vagrant@controlplane:~/sre-user$ kubectl --kubeconfig=sre-config get pods
Error from server (Forbidden): pods is forbidden: User "sre" cannot list resource "pods" in API group "" in the namespace "default"
```

As you can see, the user **SRE** does not have permission in the default namespace.

### Kubectl Auth Command

You can also use the **kubectl auth** command to check if a user has specific permissions.

If the **SRE** user has access, the output will be **yes**; otherwise, it will be **no**.

```bash
vagrant@controlplane:~/sre-user$ kubectl auth can-i get pods -n kube-system --as sre
yes

# 
vagrant@controlplane:~/sre-user$ kubectl auth can-i delete pods -n kube-system --as sre
no

# 
vagrant@controlplane:~/sre-user$ kubectl auth can-i get deploy -n kube-system --as sre
yes

# 
vagrant@controlplane:~/sre-user$ kubectl auth can-i delete deploy -n kube-system --as sre
no
```

Now, we see that the user **SRE** can list the pods and deployments in the **kube-system** namespace but cannot delete them.

In other namespaces, the user **SRE** will not have any access. Let's test this for the default namespace.

```bash
vagrant@controlplane:~/sre-user$ kubectl auth can-i get pods --as sre
no
```

---

### Imperative Method (Recommended for CKA)

When it comes to the CKA exam, it is recommended to use imperative commands to create roles and role bindings and then edit the object based on the exam requirements.

Lets create the namespace named **backend-apps**.

```bash
vagrant@controlplane:~/sre-user$ kubectl create ns backend-apps
namespace/backend-apps created
```

Create the role named **backend-role** with create access for Pod, Deployment, StatefulSet, and DaemonSet resources.

```bash
vagrant@controlplane:~/sre-user$ kubectl create role backend-role --verb=create --resource=pods,deployments,statefulsets,daemonsets -n backend-apps
role.rbac.authorization.k8s.io/backend-role created
```

Create the **role binding** to assign the role to the user **SRE** in the **backend-apps** namespace.

```bash
vagrant@controlplane:~/sre-user$ kubectl create rolebinding backend-rolebinding --role=backend-role --user=sre -n backend-apps
rolebinding.rbac.authorization.k8s.io/backend-rolebinding created
```

We have just created permissions for Pod, Deployment, StatefulSet, and DaemonSet resources within the **backend-apps** namespace for the user **SRE**.

```bash
# validate the access using kubectl auth command
vagrant@controlplane:~/sre-user$ kubectl auth can-i create pods -n backend-apps --as sre
yes

# 
vagrant@controlplane:~/sre-user$ kubectl auth can-i create daemonsets -n backend-apps --as sre
yes
```

---

### Working With Cluster Roles and ClusterRole Bindings

**ClusterRole** is a **cluster-scoped** object. It's used to grant permissions across the entire cluster to users or service accounts.

Let's consider a scenario where we need to give only read access to a specific user, **SRE**, for pods and nodes across the entire cluster.

**1. Create ClusterRole:** Create a file named **cluster-role.yaml** with the following contents.

```bash
# Grants read-only access (get, watch, list) to Pods and Nodes across the entire cluster.
apiVersion: rbac.authorization.k8s.io/v1   # RBAC API group and version
kind: ClusterRole                          # Defines a cluster-wide role (not limited to a namespace)
metadata:
  name: dev-cluster-role                   # Name of the ClusterRole
rules:                                     # List of permission rules
- apiGroups: [""]                          # "" = core API group (Pods, Nodes, Services, etc.)
  resources: ["pods","nodes"]              # Applies to Pods and Nodes resources
  verbs: ["get", "watch", "list"]          # Allowed actions: read-only access (view, stream, list)
- nonResourceURLs:                         # Permissions for non-resource endpoints (not tied to objects)
  - /metrics                               # Access to cluster metrics endpoint
  - /logs                                  # Access to cluster logs endpoint
  - /healthz                               # Access to health check endpoint
  verbs:
  - get                                    # Allowed action: read (GET requests only)
```

> Cluster roles are not tied to a specific namespace. They are cluster-scoped resources and are stored at the cluster level rather than within a particular namespace.

Now let’s create the cluster role using the kubectl apply command and list down the cluster roles.

```bash
vagrant@controlplane:~$ kubectl apply -f cluster-role.yaml
clusterrole.rbac.authorization.k8s.io/dev-cluster-role created

# 
vagrant@controlplane:~$ kubectl get clusterroles | grep dev-cluster-role
dev-cluster-role                                                       2026-07-18T12:13:49Z

# 
vagrant@controlplane:~$ kubectl describe clusterrole dev-cluster-role
Name:         dev-cluster-role
Labels:       <none>
Annotations:  <none>
PolicyRule:
  Resources  Non-Resource URLs  Resource Names  Verbs
  ---------  -----------------  --------------  -----
  nodes      []                 []              [get watch list]
  pods       []                 []              [get watch list]
             [/healthz]         []              [get]
             [/logs]            []              [get]
             [/metrics]         []              [get]
```

In a K8s cluster, there are several **default cluster roles** provided out of the box. These default cluster roles are created to fulfill common administration and user requirements.

```bash
# shows cluster roles available in the cluster
vagrant@controlplane:~$ kubectl get clusterroles | wc -l
91
```

**2. Create ClusterRoleBinding:** Now that the cluster role is created, the next step is to link the user **SRE** to this cluster role. For this, we will create a **ClusterRoleBinding**.

Create a **cluster-role-binding.yaml** file with the following contents.

```bash
# It links the dev-cluster-role (which grants read-only access to Pods, Nodes, and certain non-resource URLs) to the user sre.
apiVersion: rbac.authorization.k8s.io/v1   # RBAC API group and version
kind: ClusterRoleBinding                   # Binds a ClusterRole to subjects cluster-wide
metadata:
  name: dev-cluster-rolebinding            # Name of the ClusterRoleBinding
subjects:                                  # Entities that will receive the ClusterRole's permissions
- kind: User                               # Binding to a user identity
  name: sre                                # The username "sre" will inherit the ClusterRole's permissions
  apiGroup: rbac.authorization.k8s.io      # API group for RBAC subjects
roleRef:                                   # Reference to the ClusterRole being bound
  kind: ClusterRole                        # Binding to a ClusterRole (cluster-wide scope)
  name: dev-cluster-role                   # The ClusterRole name (must exist in the cluster)
  apiGroup: rbac.authorization.k8s.io      # RBAC API group
```

Now let’s create the cluster role binding and validate it.

```bash
vagrant@controlplane:~$ kubectl apply -f cluster-role-binding.yaml
clusterrolebinding.rbac.authorization.k8s.io/dev-cluster-rolebinding created

#
vagrant@controlplane:~$ kubectl get clusterrolebinding | grep dev-cluster-rolebinding
dev-cluster-rolebinding          ClusterRole/dev-cluster-role            26s

#
vagrant@controlplane:~$ kubectl describe clusterrolebinding dev-cluster-rolebinding
Name:         dev-cluster-rolebinding
Labels:       <none>
Annotations:  <none>
Role:
  Kind:  ClusterRole
  Name:  dev-cluster-role
Subjects:
  Kind  Name  Namespace
  ----  ----  ---------
  User  sre
```

### Validate ClusterRole

Run **kubectl auth** commands as user **SRE** to verify the given access. We will use the **--all-namespaces** flag to check if the permission is available across all namespaces cluster-wide.

```bash
vagrant@controlplane:~$ kubectl auth can-i get pods --all-namespaces --as sre
yes
vagrant@controlplane:~$ kubectl auth can-i get /metrics --as sre
yes
vagrant@controlplane:~$ kubectl auth can-i delete pods --all-namespaces --as sre
no
vagrant@controlplane:~$ kubectl auth can-i get nodes --as sre
Warning: resource 'nodes' is not namespace scoped

yes
vagrant@controlplane:~$ kubectl auth can-i delete nodes --as sre
Warning: resource 'nodes' is not namespace scoped

no
```

### Imperative Method (For CKA)

Now, let's create cluster roles and cluster role bindings using imperative commands.

Here is what we need: User **sre** should have cluster-wide read and edit access to Pods, Deployments, and DaemonSets.

```bash
# Create a custom cluster role
vagrant@controlplane:~$ kubectl create clusterrole sre-cluster-role --verb=get,list,watch,create,update,patch,delete --r
esource=pods,deployments,daemonsets
clusterrole.rbac.authorization.k8s.io/sre-cluster-role created

# Create a ClusterRoleBinding to assign the cluster role to user sre
vagrant@controlplane:~$ kubectl create clusterrolebinding sre-cluster-rolebinding --clusterrole=sre-cluster-role --user=sre
clusterrolebinding.rbac.authorization.k8s.io/sre-cluster-rolebinding created

# validate the access using kubectl auth
vagrant@controlplane:~$ kubectl auth can-i delete deployments --all-namespaces --as sre
yes
```

### Other Useful Commands

```bash
# add cluster roles to a specific group
vagrant@controlplane:~$ kubectl create clusterrolebinding group-rolebinding --clusterrole=group-cluster-role --group=devops
clusterrolebinding.rbac.authorization.k8s.io/group-rolebinding created
```

You can create a **ClusterRoleBinding** that binds the cluster role to multiple subjects (users and groups).

```bash
vagrant@controlplane:~$ kubectl create clusterrolebinding example-cluster-rolebinding --clusterrole=example-cluster-role --user=user1 --user=user2 --group=group1 --group=group2
clusterrolebinding.rbac.authorization.k8s.io/example-cluster-rolebinding created
```

---

### Adding Roles and ClusterRoles to Service Accounts

Let's say an application running in Namespace B(e.g., backend-apps) needs to access resources like Pods or Services in Namespace A (e.g., frontend-apps). To achieve this, you would follow these steps:

- In **Namespace A** (frontend-apps), create a Role that defines the permissions you want to grant, such as read access to Pods and Services.
- In **Namespace A** (frontend-apps), create a RoleBinding that binds the Service Account from **Namespace B** (backend-apps) to the Role you just created.

![Roles & ClusterRoles 2]({{ site.baseurl }}/assets/img/k8s-course/roles-clusterroles-2.jpg)

### Practical Example

Let's look at an example of giving permissions using Roles to a service account. Create the service account named backend-sa in the backend-apps namespace.

**Step 1:** Create the service account named **backend-sa** in the **backend-apps** namespace.

```bash
vagrant@controlplane:~$ kubectl create sa backend-sa -n backend-apps
serviceaccount/backend-sa created
```

**Step 2:** Create the role that lets the service account read, edit, and delete access to pods, deployments, and daemonsets in the backend-apps namespace.

```bash
# create a role named backend-sa-role with a list of allowed access.
vagrant@controlplane:~$ kubectl create role backend-sa-role --verb=get,list,watch,create,update,patch,delete --resource=pods,deployments,daemonsets -n backend-apps
role.rbac.authorization.k8s.io/backend-sa-role created

# Use the describe command to check whether it has been created as expected.
vagrant@controlplane:~$ kubectl describe role backend-sa-role -n backend-apps
Name:         backend-sa-role
Labels:       <none>
Annotations:  <none>
PolicyRule:
  Resources         Non-Resource URLs  Resource Names  Verbs
  ---------         -----------------  --------------  -----
  pods              []                 []              [get list watch create update patch delete]
  daemonsets.apps   []                 []              [get list watch create update patch delete]
  deployments.apps  []                 []              [get list watch create update patch delete]
```

**Step 3:** Create the role binding named **backend-sa-rolebinding** to bind the role to the backend-sa service account.

```bash
vagrant@controlplane:~$ kubectl create rolebinding backend-sa-rolebinding --role=backend-sa-role --serviceaccount=backend-apps:backend-sa -n backend-apps
rolebinding.rbac.authorization.k8s.io/backend-sa-rolebinding created
```

The syntax **--serviceaccount=backend-apps:backend-sa** consists of two parts separated by a colon (**:**). Because service accounts are **namespace-scoped** objects in K8s.

- backend-apps represents the namespace
- backend-sa - name of the service account

Describe it to check whether it is created correctly or not.

```bash
vagrant@controlplane:~$ kubectl describe rolebinding backend-sa-rolebinding -n backend-apps
Name:         backend-sa-rolebinding
Labels:       <none>
Annotations:  <none>
Role:
  Kind:  Role
  Name:  backend-sa-role
Subjects:
  Kind            Name        Namespace
  ----            ----        ---------
  ServiceAccount  backend-sa  backend-apps
```

**Step 4:** Validate Service Account Access

```bash
vagrant@controlplane:~$ kubectl auth can-i get deploy -n backend-apps --as=system:serviceaccount:backend-apps:backend-sa
yes

vagrant@controlplane:~$ kubectl auth can-i get po --all-namespaces --as=system:serviceaccount:backend-apps:backend-sa
no
```

---

### Creating API token for a ServiceAccount

This section is particularly useful in real-world scenarios where you want to give access to the cluster API for external applications. For example, to provide read access to the cluster metrics API to a monitoring system.

If you want to make API calls from outside the cluster, you cannot use the default short-lived token. You need to create a secret of type kubernetes.io/service-account-token to have a long-lived service account token.

Create a file named **sa-token.yaml** with the following contents. It will create a secret in the **backend-apps** namespace and create a **token** for the **backend-sa** service account.

```bash
apiVersion: v1
kind: Secret
metadata:
  name: sa-token-secret
  namespace: backend-apps
  annotations:
    kubernetes.io/service-account.name: backend-sa
type: kubernetes.io/service-account-token

# Create the secret
vagrant@controlplane:~$ kubectl apply -f sa-token.yaml
secret/sa-token-secret created

# List the token
vagrant@controlplane:~$ kubectl get secrets -n backend-apps
NAME              TYPE                                  DATA   AGE
sa-token-secret   kubernetes.io/service-account-token   3      27s

# oken details using the CA certificate. The token will be in base64 encoded format.
vagrant@controlplane:~$ kubectl get secret/sa-token-secret -n backend-apps -o yaml
apiVersion: v1
data:
  ca.crt: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSURCV
  # ...
```

To test the API call, first, let's deploy a web server pod in the backend-apps namespace.

```bash
vagrant@controlplane:~$ kubectl run webserver --image=nginx -n backend-apps
pod/webserver created
```

Now, let's decode the token and try to make an API call to the cluster using the token.

```bash
# fetch the secret and store it in the token variable.
vagrant@controlplane:~$ token=$(kubectl get secret sa-token-secret -n backend-apps -o jsonpath="{.data.token}" | base64 -d)

# 
vagrant@controlplane:~$ echo $token
eyJhbGciOiJSUzI1NiIsImtpZCI6IllBbHhNRFJIZWRORDEteU5ORkplcH
# ...
```

Here is the curl command to make the API request with the token to list pods in the backend-apps namespace.

```bash
# Replace 192.168.201.10 with your cluster IP. 
vagrant@controlplane:~$ curl -X GET -k -H "Authorization: Bearer $token" https://10.96.0.1:443/api/v1/namespaces/backend-apps/pods
{
  "kind": "PodList",
  "apiVersion": "v1",
  "metadata": {
    "resourceVersion": "994754"
  },
  "items": [
    {
      "metadata": {
        "name": "webserver",
        "namespace": "backend-apps",
        "uid": "b2cb90ef-725e-4071-8af8-3bea0435bab8",
        "resourceVersion": "993739",
        "generation": 1,
        "creationTimestamp": "2026-07-19T03:52:44Z",
        "labels": {
          "run": "webserver"
        },
# ...
```

In the output, you should see the webserver pod details in JSON format.

---

**Clean up** the backend-apps namespace. It will delete all the objects associated with the namespace.

```bash
vagrant@controlplane:~$ kubectl delete ns backend-apps
namespace "backend-apps" deleted
```

---

### Scenario 01: Create ClusterRole To Provide Access

You need to create a service account for a specific application, define a cluster role that grants permissions to perform certain actions within the cluster, and bind the cluster role to the service account to enable the application to access the required resources.

- Create a service account named techiescamp-sa for the application requiring access to certain cluster resources. - Ensure that the service account techiescamp-sa has been created successfully.
- Define a cluster role named techiescamp-clusterrole that grants permissions to list pods and deployments in all namespaces. Ensure that the cluster role techiescamp-clusterrole has been created successfully.
- Now create a cluster role binding named techiescamp-clusterrolebinding to bind the cluster role techiescamp-clusterrole to the techiescamp-sa service account. Ensure that the cluster role binding techiescamp-clusterrolebinding has been created successfully.
- Verify that the service account can list down pods and deployments.
- Now, we want to give the creation permission only for pods. Edit the cluster role to add create permission to pods.
- Verify again whether the service account can create the pod or not.

### Scenario 02: Create Role To Provide Access

You are a K8s administrator responsible for managing access control within a K8s cluster. Techiescamp's development team requires controlled access to specific resources in the cluster for deploying applications. You have been tasked with setting up Role-Based Access Control (RBAC) to fulfill this requirement.

- Create a service account named jarvis-sa for the development team to use when deploying applications.
- Next, define roles that specify the permissions required for deploying applications. Create a role named jarvis-role with permissions to create, read, update, and delete deployments, services, and pods within the default namespace.
- Now bind the jarvis-role to the jarvis-sa service account using a role binding named jarvis-role-binding.
- Verify that the RBAC setup is working correctly.

### Scenario 03: Provide Access to Non Resource URLs

You are a K8s administrator for an organization. You need to ensure that a specific service account has access to certain non-resource URLs for monitoring and debugging purposes.

- Create a namespace called monitoring-apps.
- Create a service account named monitoring-sa in the monitoring-apps namespace.
- Create a ClusterRole named monitoring-nonresource-access that allows access to the non-resource URLs /metrics and /healthz with the get verb.
- Bind the created ClusterRole to the monitoring-sa service account using a ClusterRoleBinding named monitoring-nonresource-access-binding
- Deploy an nginx:latest pod in the monitoring-apps namespace, using the monitoring-sa service account.
- Test the access to the non-resource URLs from within the nginx pod using curl and internal cluster URL.
