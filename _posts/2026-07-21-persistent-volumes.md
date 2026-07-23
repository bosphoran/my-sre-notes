---
layout: post
title: "Persistent Volumes"
date: 2026-07-21
categories: [k8s]
---

## Introduction

Suppose, running databases like MySQL, PostgreSQL, or MongoDB requires persistent storage to ensure data is not lost when pods are restarted or rescheduled. **Persistent Volumes** (PVs) provide a solution to this problem.

> **Persistent Volumes** represent a cluster-wide pool of storage resources that are independent of any specific node or Pod. By decoupling storage management from the Pod lifecycle, they ensure that data remains available even after individual Pods or nodes are terminated.

### Persistent Volume Storage

The obvious question that comes to mind is: when we create a persistent volume, which storage does it use?

For a **persistent volume**, storage can come from various sources, such as **block storage**, **network-attached storage** (NAS), or **cloud-based storage** solutions. The type of storage depends on whether the Kubernetes cluster is deployed on the cloud or on-premises.

![Persistent Volume]({{ site.baseurl }}/assets/img/k8s-course/persistent-volume.jpg)

### Persistent Volume Claim

In K8s, a pod cannot directly use a Persistent Volume (PV). Instead, it needs to go through a **PersistentVolumeClaim** (PVC) to access the persistent storage.

> **PersistentVolumeClaim** (PVC)- is used by a pod to claim or request to use the storage defined by a **PersistentVolume**.

You can compare a **PVC** to a Linux **mount** command. Just as attaching a disk to a Linux server is not enough—you must mount it to use it. Having a **PV** in K8s is not sufficient on its own. You need to create a **PVC** for the pod to access the PV.

> Think of the PVC as a middleman between the pod and the storage infrastructure.

- A pod refers to a PVC for its storage needs.
- The PVC then refers to a PV for storage.
- The PV connects to the actual storage backend (like AWS EBS or NFS).

![Persistent Volume 2]({{ site.baseurl }}/assets/img/k8s-course/persistent-volume-2.jpg)

### Volume Access Modes

For pods to access and use persistent volumes, the backend storage system needs to be properly configured and mounted (actual linux mount) on the worker nodes where the pods are scheduled to run.

> mounting is the process of attaching a storage device or a file system to a specific directory in the Linux directory hierarchy, making it accessible to the operating system and its users.

The accessModes in persistentVolume defines how a volume should be mounted. There are 4 access modes available:

1. ReadWriteOnce (RWO)
2. ReadWriteMany (RWX)
3. ReadOnlyMany (ROX)
4. ReadWriteOncePod

---

### Working With Persistent Volumes & Claims

**MySQL** uses the data directory **/var/lib/mysql** to store its data. We will create a **Persistent Volume** (PV) to hold the MySQL data and a **Persistent Volume Claim** (PVC) to make the PV available for the MySQL pod.

- Create a Persistent Volume (PV): Define a 2Gi storage volume to hold the MySQL data using hostpath /mnt/data/mysql
- Create a Persistent Volume Claim (PVC): Request to use the 2Gi storage volume.
- Deploy the MySQL Pod: Use the PVC to mount the /var/lib/mysql directory.

First let's create a PersistentVolume object of **2Gi**. Note: For demonstration purposes, we will use the **hostPath** directory as a persistent volume backend.

```bash
# Create mysql-pv.yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: mysql-pv
  labels:
    type: mysql-storage
spec:
  capacity:
    storage: 2Gi
  accessModes:
  - ReadWriteOnce
  hostPath:
    path: /mnt/data/mysql
# ---------------------------------------------------

#  create the pv
vagrant@controlplane:~$ kubectl apply -f mysql-pv.yaml
persistentvolume/mysql-pv created

# get the pv details
vagrant@controlplane:~$ kubectl get pv
NAME       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM   STORAGECLASS   VOLUMEATTRIBUTESCLASS   REASON   AGE
mysql-pv   2Gi        RWO            Retain           Available                          <unset>                          33s
# status of the PV is marked as Available because it is not claimed by any Persistent Volume Claim (PVC) yet.
```

The **reclaim policy** determines what happens to the storage when the Persistent Volume Claim (PVC) is deleted or the Pod using the PVC is terminated. 

- **Retain** - It does not delete the storage automatically when the PVC is deleted. If no reclaim policy is specified, it defaults to Retain.
- **Recycle** - Using this we can restore the data later. It's deprecated now and will only be used in nfs and hostPath volume types.
- **Delete** - The associated storage will get deleted.

```bash
# We can define the reclaim policy under the spec section
persistentVolumeReclaimPolicy: <reclaim-policy-name>

# describe the pv to get all the PV details
vagrant@controlplane:~$ kubectl describe pv mysql-pv
Name:            mysql-pv
Labels:          type=mysql-storage
Annotations:     <none>
Finalizers:      [kubernetes.io/pv-protection]
StorageClass:
Status:          Available
Claim:
Reclaim Policy:  Retain
Access Modes:    RWO
VolumeMode:      Filesystem
Capacity:        2Gi
Node Affinity:   <none>
Message:
Source:
    Type:          HostPath (bare host directory volume)
    Path:          /mnt/data/mysql
    HostPathType:
Events:            <none>
```

### Create Persistent Volume Claim

Let's create a **PVC** so that we can use it with the MySQL deployment.

```bash
# create the mysql-pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mysql-pvc
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 2Gi
  selector:
    matchLabels:
      type: mysql-storage
```

You can see that **there are no details about the Persistent Volume**. This is because, when Persistent Volume Claim (PVC) is created, it is matched with an available Persistent Volume (PV) that matches its criteria.

- The PVC requests 2Gi of storage, so K8s looks for PVs with at least 2Gi capacity.
- The PVC requests ReadWriteOnce access mode, so K8s looks for PVs that support this access mode.
- The PVC mysql-pvc has a selector label type: mysql-storage. So K8s will look for a PV that has the label type: mysql-storage

Once a match is found, K8s binds the PVC to that PV, making the storage available for use by the pod that requested it.

```bash
vagrant@controlplane:~$ kubectl apply -f mysql-pvc.yaml
persistentvolumeclaim/mysql-pvc created

# Get the status of the PVC
vagrant@controlplane:~$ kubectl get pvc
NAME        STATUS   VOLUME     CAPACITY   ACCESS MODES   STORAGECLASS   VOLUMEATTRIBUTESCLASS   AGE
mysql-pvc   Bound    mysql-pv   2Gi        RWO                           <unset>                 24s
```

From the output, you can see that the PVC is bound to mysql-pv as it meets all the criteria mentioned in the PVC.

```bash
# describe the PVC to get all the details
vagrant@controlplane:~$ kubectl describe pvc mysql-pvc
Name:          mysql-pvc
Namespace:     default
StorageClass:
Status:        Bound
Volume:        mysql-pv
Labels:        <none>
Annotations:   pv.kubernetes.io/bind-completed: yes
               pv.kubernetes.io/bound-by-controller: yes
Finalizers:    [kubernetes.io/pvc-protection]
Capacity:      2Gi
Access Modes:  RWO
VolumeMode:    Filesystem
Used By:       <none>
Events:        <none>
```

### Use PVC With MySQL Deployment

Now that we have the PVC ready, we can use it with the MySQL pod to mount the **/var/lib/mysql** data folder to the persistent volume.

```bash
# Create mysql.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mysql
spec:
  selector:
    matchLabels:
      app: mysql
  replicas: 1
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
      - name: mysql
        image: mysql
        ports:
        - containerPort: 3306
        env:
        - name: MYSQL_ROOT_PASSWORD
          value: mys3cr3t 
        volumeMounts:
        - name: mysql-storage
          mountPath: /var/lib/mysql
      volumes:
      - name: mysql-storage
        persistentVolumeClaim:
          claimName: mysql-pvc
```

```bash
# deploy MySQL deployment
vagrant@controlplane:~$ kubectl apply -f mysql.yaml
deployment.apps/mysql created

# get deployment status
vagrant@controlplane:~$ kubectl get deployment
NAME                            READY   UP-TO-DATE   AVAILABLE   AGE
mysql                           1/1     1            1           16m

# Get the pod status
vagrant@controlplane:~$ kubectl get pod
NAME                                            READY   STATUS    RESTARTS      AGE
mysql-759d95555b-gb58n                          1/1     Running   0             11m

# check the volume details by describing the pod
vagrant@controlplane:~$ kubectl describe po mysql-759d95555b-gb58n | grep volume -B6
Volumes:
  mysql-storage:
    Type:       PersistentVolumeClaim (a reference to a PersistentVolumeClaim in the same namespace)
    ClaimName:  mysql-pvc
    ReadOnly:   false
  kube-api-access-t4d9h:
    Type:                    Projected (a volume that contains injected data from multiple sources)
```

Now, clean up the deployment, PVC and PV.

```bash
vagrant@controlplane:~$ kubectl delete deploy mysql
deployment.apps "mysql" deleted from default namespace

vagrant@controlplane:~$ kubectl delete pvc mysql-pvc
persistentvolumeclaim "mysql-pvc" deleted from default namespace

vagrant@controlplane:~$ kubectl delete pv mysql-pv
persistentvolume "mysql-pv" deleted
```

---

### Volume Subpath

**subPath** in volumes allows you to mount a subdirectory of a volume into your pod instead of the whole volume.

It is primarily useful to isolate container access to specific subdirectories. This helps prevent containers in the same pod from interfering with each other's data.

- The default path where Nginx stores its public HTML files is **/usr/share/nginx/html**. We will use the subPath feature to mount this path to a folder named html_folder.
- Additionally, we will mount the default Nginx log directory **/var/log/nginx** to a subpath named log_dir in the Persistent Volume

![Persistent Volume 3]({{ site.baseurl }}/assets/img/k8s-course/persistent-volume-3.jpg)

```bash
# Create a volume.yaml manifest
apiVersion: v1
kind: PersistentVolume
metadata:
  name: nginx-pv
spec:
  capacity:
    storage: 2Gi
  accessModes:
  - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  hostPath:
    path: /data/nginx
    type: DirectoryOrCreate
---

apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: nginx-pvc
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 2Gi
```

```bash
# Create the volumes
vagrant@controlplane:~$ kubectl apply -f volume.yaml
persistentvolume/nginx-pv created
persistentvolumeclaim/nginx-pvc created

# Validate the PV and PVC
vagrant@controlplane:~$ kubectl get pv,pvc
NAME                        CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM               STORAGECLASS   VOLUMEATTRIBUTESCLASS   REASON   AGE
persistentvolume/nginx-pv   2Gi        RWO            Retain           Bound    default/nginx-pvc                  <unset>                          67s

NAME                              STATUS   VOLUME     CAPACITY   ACCESS MODES   STORAGECLASS   VOLUMEATTRIBUTESCLASS   AGE
persistentvolumeclaim/nginx-pvc   Bound    nginx-pv   2Gi        RWO                           <unset>                 67s
```

```bash
# create a webserver.yaml manifest
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
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
      - name: nginx-server
        image: nginx
        volumeMounts:
        - name: nginx-volume
          mountPath: /usr/share/nginx/html
          subPath: html_folder
        - name: nginx-volume
          mountPath: /var/log/nginx
          subPath: log_dir
      volumes:
      - name: nginx-volume
        persistentVolumeClaim:
          claimName: nginx-pvc
```

In this deployment manifest, you can see the **volumeMounts** using **subPath html_folder** and **log_dir**.

```bash
vagrant@controlplane:~$ kubectl apply -f webserver-pvc.yaml
deployment.apps/nginx-deployment created

# get pod status
vagrant@controlplane:~$ kubectl get pods
NAME                                            READY   STATUS    RESTARTS       AGE
my-release-prometheus-adapter-8756f6f6c-mpdtm   1/1     Running   8 (101m ago)   5d3h
nginx-deployment-7c9f58cccd-ckp5n               1/1     Running   0              41s

# describe the pod, you will see the subPath under Mounts
vagrant@controlplane:~$ kubectl describe po nginx-deployment-7c9f58cccd-ckp5n | grep volume -B3
    Restart Count:  0
    Environment:    <none>
    Mounts:
      /usr/share/nginx/html from nginx-volume (rw,path="html_folder")
      /var/log/nginx from nginx-volume (rw,path="log_dir")
--
  ContainersReady             True
  PodScheduled                True
Volumes:
  nginx-volume:
--
    ClaimName:  nginx-pvc
    ReadOnly:   false
  kube-api-access-h2dsh:
    Type:                    Projected (a volume that contains injected data from multiple sources)
```

Lets, clean up the deployment and volumes.

```bash
vagrant@controlplane:~$ kubectl delete deployment nginx-deployment
deployment.apps "nginx-deployment" deleted from default namespace

# 
vagrant@controlplane:~$ kubectl delete pvc nginx-pvc
persistentvolumeclaim "nginx-pvc" deleted from default namespace

#
vagrant@controlplane:~$ kubectl delete pv nginx-pv
persistentvolume "nginx-pv" deleted
```
