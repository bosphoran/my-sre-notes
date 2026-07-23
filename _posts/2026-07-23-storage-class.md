---
layout: post
title: "Storage Class"
date: 2026-07-23
categories: [k8s]
---

## Introduction

A StorageClass defines how storage is provisioned in the cluster.

- It acts like a template for creating Persistent Volumes (PVs).
- Specifies details such as the provisioner (e.g., AWS EBS, GCE PD, Ceph, NFS), parameters (like disk type, replication), and reclaim policy.
- When a Pod requests storage via a PersistentVolumeClaim (PVC), Kubernetes looks at the StorageClass to decide how to create or bind the volume.

Consider a scenario where you have a K8s cluster running in a cloud environment, and you have multiple applications with different storage requirements.

- A database application that requires fast, reliable storage with SSD-based persistent volumes.
- A data processing application that needs large amounts of storage but can tolerate slower performance, so it can use HDD-based persistent volumes.
- A web application that requires shared storage (NFS) accessible from multiple pods.

Without **Storage Classes**, you would need to manually provision the required persistent volumes for each application and manage them separately. This can be **time-consuming**, especially as the number of applications and storage requirements grows.

However, with **Storage Classes**, you can define different classes of storage based on your application's needs.

- Create a **db-storage** Storage Class that provisions SSD-based persistent volumes with high performance.
- Create a **shared-storage** Storage Class that provisions persistent volumes with ReadWriteMany access mode, allowing multiple pods to access the same storage simultaneously.

```bash
# Here is an example db-storage StorageClass object
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: db-storage
provisioner: ebs.csi.aws.com
parameters:
  type: gp2
  fsType: ext4
```

Now, when deploying your applications, you can simply create a **PersistentVolumeClaim** specifying the desired Storage Class, and K8s will automatically provision the appropriate persistent volume based on the defined class.

![Storage Class]({{ site.baseurl }}/assets/img/k8s-course/storage-class.jpg)

### Static Volume Provisioning

n our persistent volume examples, we used static provisioning. This means we manually set up the storage (PersistentVolume or PV). Then we used the PVC to make use of the Volume.

**static provisioning** is particularly useful in an **on-premises** environment where a K8s administrator sets up storage that is shared across the entire cluster.

For example, the administrator might provision a **2TB NFS volume**. Developers can then use this persistent volume through **PersistentVolumeClaims** (PVCs) for different applications.

- Admins manually create PersistentVolumes (PVs) ahead of time.
- Each PV points to a specific piece of storage (disk, NFS share, etc.).
- PVCs are then matched to these pre-created PVs if they fit the request.

### Dynamic Volume Provisioning

Primarily in cloud environments, Persistent Volumes (PVs) are typically dynamically provisioned. This means that **creating a Persistent Volume Claim (PVC) automatically triggers the creation of a Persistent Volume**.

- Admins define a StorageClass.
- When a PVC is created, Kubernetes automatically provisions a new PV using the StorageClass.
- No need to pre-create volumes manually.
- **Use case:** Cloud environments (AWS, GCP, Azure) where storage can be provisioned on demand.

### Storage Class Provisioners

The **StorageClass** defines which provisioner to use and the parameters needed for dynamic provisioning of storage resources. Each cloud provider has its own provisioner that integrates with their storage services.

The provisioner is responsible for creating the actual storage volumes based on the parameters defined in the StorageClass.

> A **StorageClass** in K8s defines how PersistentVolumes are provisioned, and the “provisioner” field specifies which storage backend or driver is responsible for creating those volumes. **Static provisioning** requires admins to manually create PVs, while **dynamic provisioning** uses StorageClasses and provisioners to automatically create PVs when PVCs are requested.

### Container Storage Interface Drivers (CSI)

The **Container Storage Interface** (CSI) is a standardized framework that enables storage vendors (such as AWS EBS) to create plugins, known as **CSI drivers**, for K8s. These drivers manage the entire lifecycle of storage volumes, including creation, attachment, mounting, and deletion.

![Storage Class 2]({{ site.baseurl }}/assets/img/k8s-course/storage-class-2.jpg)

> CSI drivers are the standardized bridge between K8s and external storage systems. They replaced older built-in plugins, making storage integration modular, flexible, and vendor-driven. In practice, whenever you define a StorageClass provisioner, you’re pointing to a CSI driver that knows how to create and manage volumes on your chosen backend.

---

### Working With Storage Class

For learning purposes, we can use the **Rancher Local Path Provisioner**, which utilizes the local storage on each node for dynamic provisioning.

In a real-world setup, instead of the Local Path Provisioner, you would typically use **cloud-specific storage provisioners**, such as those provided by AWS, GCP, or Azure.

### Deploy the Local Path Provisioner

o deploy the Rancher Local Path Provisioner, use the following command: It gets deployed in the **local-path-storage** namespace,

```bash
# deploy the Rancher Local Path Provisioner
vagrant@controlplane:~$ kubectl apply -f https://raw.githubusercontent.com/techiescamp/cka-certification-guide/main/storage-provisioner/local-path-storage.yaml
namespace/local-path-storage created
serviceaccount/local-path-provisioner-service-account created
role.rbac.authorization.k8s.io/local-path-provisioner-role created
clusterrole.rbac.authorization.k8s.io/local-path-provisioner-role created
rolebinding.rbac.authorization.k8s.io/local-path-provisioner-bind created
clusterrolebinding.rbac.authorization.k8s.io/local-path-provisioner-bind created
deployment.apps/local-path-provisioner created
configmap/local-path-config created

# check the deployment status
vagrant@controlplane:~$ kubectl get deploy -n local-path-storage
NAME                     READY   UP-TO-DATE   AVAILABLE   AGE
local-path-provisioner   1/1     1            1           95s
```

Now, we have a storage provisioner deployment that watches for **PersistentVolumeClaims** (PVCs) that match its StorageClass. The provisioner name for the Local Path Provisioner is **rancher.io/local-path**.

### Create Storage Class

Next, we need to create a StorageClass that uses rancher.io/local-path as the provisioner.

```bash
# create a local-storage Storage class, sc.yaml, that utilizes the rancher.io/local-path provisioner.
# For sc (storage class), apiVersion is storage.k8s.io/v1 and the kind is StorageClass.
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: local-storage
  annotations: # storageclass.kubernetes.io/is-default-class: "true" marks this StorageClass as the default StorageClass for the cluster. 
    storageclass.kubernetes.io/is-default-class: "true"
provisioner: rancher.io/local-path # the plugin used to provision the storage volume. In our case, it's the rancher.io/local-path local provisioner
volumeBindingMode: WaitForFirstConsumer # Determines when the PersistentVolume (PV) should be bound to a PersistentVolumeClaim (PVC).
# There are 2 modes available, Immediate and WaitForFirstConsumer.
reclaimPolicy: Delete # Specifies what happens to the PersistentVolume (PV) when the PersistentVolumeClaim (PVC) is deleted.
```

```bash
# apply the manifest and create the storage class
vagrant@controlplane:~$ kubectl apply -f sc.yaml
storageclass.storage.k8s.io/local-storage created

# verify storage class
vagrant@controlplane:~$ kubectl get sc
NAME                      PROVISIONER             RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION   AGE
local-storage (default)   rancher.io/local-path   Delete          WaitForFirstConsumer   false                  32s

# To know all the details of the storage class, describe it
vagrant@controlplane:~$ kubectl describe sc local-storage
Name:            local-storage
IsDefaultClass:  Yes
Annotations:     kubectl.kubernetes.io/last-applied-configuration={"apiVersion":"storage.k8s.io/v1","kind":"StorageClass","metadata":{"annotations":{"storageclass.kubernetes.io/is-default-class":"true"},"name":"local-storage"},"provisioner":"rancher.io/local-path","reclaimPolicy":"Delete","volumeBindingMode":"WaitForFirstConsumer"}
,storageclass.kubernetes.io/is-default-class=true
Provisioner:           rancher.io/local-path
Parameters:            <none>
AllowVolumeExpansion:  <unset>
MountOptions:          <none>
ReclaimPolicy:         Delete
VolumeBindingMode:     WaitForFirstConsumer
Events:                <none>
```

### Create PVC

```bash
# Create redis-pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: redis-pvc
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
  storageClassName: local-storage
```

Here, we have specified storageClassName: local-storage. This means we are requesting storage supported by the local-storage storage class.

```bash
# create the PVC
vagrant@controlplane:~$ kubectl apply -f redis-pvc.yaml
persistentvolumeclaim/redis-pvc created

# verify pvc
vagrant@controlplane:~$ kubectl get pvc
NAME        STATUS    VOLUME   CAPACITY   ACCESS MODES   STORAGECLASS    VOLUMEATTRIBUTESCLASS   AGE
redis-pvc   Pending                                      local-storage   <unset>                 29s

# describe the PVC and check its status
vagrant@controlplane:~$ kubectl describe pvc redis-pvc
Name:          redis-pvc
Namespace:     default
StorageClass:  local-storage
Status:        Pending
Volume:
Labels:        <none>
Annotations:   <none>
Finalizers:    [kubernetes.io/pvc-protection]
Capacity:
Access Modes:
VolumeMode:    Filesystem
Used By:       <none>
Events:
  Type    Reason                Age               From                         Message
  ----    ------                ----              ----                         -------
  Normal  WaitForFirstConsumer  2s (x5 over 60s)  persistentvolume-controller  waiting for first consumer to be created before binding
```

If you check the output, you will see the status as **Pending**, and the Events will show the reason as **WaitForFirstConsumer**. This means that the PVC will only be bound to a volume when a pod references it.

### Deploy Redis With PVC

Let's deploy a Redis pod that uses the PVC to mount in data.

```bash
# Create a file redis.yaml
apiVersion: v1
kind: Pod
metadata:
  name: redis
spec:
  containers:
  - name: redis
    image: redis:latest
    volumeMounts:
    - mountPath: /data
      name: redis-storage
  volumes:
  - name: redis-storage
    persistentVolumeClaim:
      claimName: redis-pvc
```

```bash
# deploy the pod
vagrant@controlplane:~$ kubectl apply -f redis-sc.yaml
pod/redis created

# verify pod status
vagrant@controlplane:~$ kubectl get pods
NAME                                            READY   STATUS    RESTARTS        AGE
redis                                           1/1     Running   0               34s

# check the PVC, it shows the status as Bound
vagrant@controlplane:~$ kubectl get pvc redis-pvc
NAME        STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS    VOLUMEATTRIBUTESCLASS   AGE
redis-pvc   Bound    pvc-ca04327a-6187-4b15-8f1e-d4bf6e395f5e   5Gi        RWO            local-storage   <unset>                 6m50s
```

Additionally, if you list the **PV**s, you will find a dynamically created PV through the local-storage storage class when we deployed the pod.

```bash
vagrant@controlplane:~$ kubectl get pv
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM               STORAGECLASS    VOLUMEATTRIBUTESCLASS   REASON   AGE
pvc-ca04327a-6187-4b15-8f1e-d4bf6e395f5e   5Gi        RWO            Delete           Bound    default/redis-pvc   local-storage   <unset>                          2m17s
```

Since we are using the **Local Path Provisioner**, the volume folder will be created on the node where the pod is deployed. For example, if the pod is deployed on node01, you can find the volume folder at the **/opt/local-path-provisioner** location on that node.

```bash
vagrant@node02:~$ ls /opt/local-path-provisioner
pvc-ca04327a-6187-4b15-8f1e-d4bf6e395f5e_default_redis-pvc
```

```bash
# Now, clean up the pod and volumes
vagrant@controlplane:~$ kubectl delete pod redis
pod "redis" deleted from default namespace

#
vagrant@controlplane:~$ kubectl delete pvc redis-pvc
persistentvolumeclaim "redis-pvc" deleted from default namespace
```

When you delete the PVC, the dynamically created PV will also be deleted because we set the reclaimPolicy to Delete in the StorageClass.
