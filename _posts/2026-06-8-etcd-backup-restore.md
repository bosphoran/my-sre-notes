---
layout: post
title: "etcd Backup & Restore"
date: 2026-06-08
categories: [k8s]
---

## etcd and etcdctl

etcd stores all the cluster objects and their state as a key-value store.

All K8s objects are stored under the **/registry** key in etcd.

For example, a Pod named nginx in the default namespace is stored at: **/registry/pods/default/nginx**

![etcd]({{ site.baseurl }}/assets/img/k8s-course/etcd.png)

- Use **etcdctl** to take backups (snapshots).
- Use **etcdutl** to restore etcd data when the control plane components are stopped.

### etcd Backup Using etcdctl

**etcd** can natively create backups of its entire **key‑value** store without needing external tools. (built-in snapshot mechanism)

**etcdctl** is the command-line tool used to interact with a running etcd cluster.

Here is the etcdctl workflow:

![etcdctl]({{ site.baseurl }}/assets/img/k8s-course/etcdctl.png)

---

### etcd Restore Using etcdutl

Unlike etcdctl, **etcdutl** does not communicate with a running etcd server. It is used to restore snapshots safely when etcd is stopped.

![etcdutl]({{ site.baseurl }}/assets/img/k8s-course/etcdutl.png)

> In real-world projects, tools like **Velero** are used for cluster backup, recovery, and disaster recovery purposes.

---

### Install etcdctl and etcdutl

First, we install **etcdctl** and **etcdutl**. For this, use the following script.

```bash
# Detect the system architecture
ARCH=$(uname -m)
if [ "$ARCH" = "x86_64" ]; then
    ARCH_TYPE="amd64"       # Map x86_64 to amd64
elif [ "$ARCH" = "aarch64" ]; then
    ARCH_TYPE="arm64"       # Map aarch64 to arm64
else
    echo "Unsupported architecture: $ARCH"
    exit 1                  # Exit if architecture is not supported
fi

# Define etcd version to install
ETCD_VER=v3.6.7

# Define download URL and file name based on version and architecture
DOWNLOAD_URL=https://storage.googleapis.com/etcd
DOWNLOAD_FILE=etcd-${ETCD_VER}-linux-${ARCH_TYPE}.tar.gz

# Create a temporary directory for download
mkdir -p /tmp/etcd-download

# Download the etcd tarball from Google Cloud Storage
curl -L ${DOWNLOAD_URL}/${ETCD_VER}/${DOWNLOAD_FILE} -o /tmp/${DOWNLOAD_FILE}

# Extract the tarball into the temporary directory
# --strip-components=1 removes the top-level folder inside the tarball
tar xzvf /tmp/${DOWNLOAD_FILE} -C /tmp/etcd-download --strip-components=1

# Move the binaries (etcd, etcdctl, etcdutl) into /usr/local/bin for system-wide use
sudo mv /tmp/etcd-download/etcd /tmp/etcd-download/etcdctl /tmp/etcd-download/etcdutl /usr/local/bin/

# Clean up: remove the downloaded tarball and temporary directory
rm -f /tmp/${DOWNLOAD_FILE}
rm -rf /tmp/etcd-download
```

Run the script. Then verify the installation.

```bash
vagrant@controlplane:~$ sh etcdctl-etcdutl.sh
```

```bash
vagrant@controlplane:~$ etcdctl version
etcdctl version: 3.6.7
API version: 3.6
```

```bash
vagrant@controlplane:~$ etcdutl version
etcdutl version: 3.6.7
API version: 3.6
```

### Backup ETCD with etcdctl

Now, we will backup etcd using etcdctl.

To perform a backup, we need to provide the following four pieces of information to etcdctl as command-line arguments.

- etcd endpoint (–endpoints)
  - advertise-client-urls
- ca certificate (–cacert)
  - trusted-ca-file
- server certificate (–cert)
  - cert-file
- server key (–key)
  - key-file

We can get the above details from the etcd static pod manifest file located at **/etc/kubernetes/manifests/etcd.yaml**. Or, by describing the etcd pod running in the **kube-system** namespace.

```bash
vagrant@controlplane:~$ sudo cat /etc/kubernetes/manifests/etcd.yaml
apiVersion: v1
kind: Pod
metadata:
  annotations:
    kubeadm.kubernetes.io/etcd.advertise-client-urls: https://192.168.201.10:2379
  labels:
    component: etcd
    tier: control-plane
  name: etcd
  namespace: kube-system
spec:
  containers:
  - command:
    - etcd
    - --advertise-client-urls=https://192.168.201.10:2379
    - --cert-file=/etc/kubernetes/pki/etcd/server.crt
    - --client-cert-auth=true
    - --data-dir=/var/lib/etcd
    - --feature-gates=InitialCorruptCheck=true
    - --initial-advertise-peer-urls=https://192.168.201.10:2380
    - --initial-cluster=controlplane=https://192.168.201.10:2380
    - --key-file=/etc/kubernetes/pki/etcd/server.key
    - --listen-client-urls=https://127.0.0.1:2379,https://192.168.201.10:2379
    - --listen-metrics-urls=http://127.0.0.1:2381
    - --listen-peer-urls=https://192.168.201.10:2380
    - --name=controlplane
    - --peer-cert-file=/etc/kubernetes/pki/etcd/peer.crt
    - --peer-client-cert-auth=true
    - --peer-key-file=/etc/kubernetes/pki/etcd/peer.key
    - --peer-trusted-ca-file=/etc/kubernetes/pki/etcd/ca.crt
    - --snapshot-count=10000
    - --trusted-ca-file=/etc/kubernetes/pki/etcd/ca.crt
    - --watch-progress-notify-interval=5s
    image: registry.k8s.io/etcd:3.6.5-0
# ...
# ...
```

```bash
vagrant@controlplane:~$ kubectl get pods -n kube-system
NAME                                   READY   STATUS        RESTARTS       AGE
coredns-66bc5c9577-qh56x               1/1     Running       1 (162m ago)   10h
coredns-66bc5c9577-tf5wj               1/1     Running       1 (162m ago)   3h9m
etcd-controlplane                      1/1     Running       1 (162m ago)   22h
```

```bash
vagrant@controlplane:~$ kubectl describe pod etcd-controlplane -n kube-system
```

Before, applying the backup command, we need to create a directory, where we save our snapshot.

```bash
vagrant@controlplane:~$ sudo mkdir -p /opt/backup/
```

Finally, we can apply the backup command to take a snapshot of etcd.

```bash
vagrant@controlplane:~$ sudo etcdctl 
    --endpoints=https://192.168.201.10:2379 
    --cacert=/etc/kubernetes/pki/etcd/ca.crt 
    --cert=/etc/kubernetes/pki/etcd/server.crt 
    --key=/etc/kubernetes/pki/etcd/server.key 
    snapshot save /opt/backup/etcd.db

Snapshot saved at /opt/backup/etcd.db
Server version 3.6.0
```

Now, lets verify the snapshot status using the **etcdutl** command.

```bash
vagrant@controlplane:~$ sudo etcdutl --write-out=table snapshot status /opt/backup/etcd.db
+----------+----------+------------+------------+---------+
|   HASH   | REVISION | TOTAL KEYS | TOTAL SIZE | VERSION |
+----------+----------+------------+------------+---------+
| 53131229 |   161305 |        569 |     8.4 MB |   3.6.0 |
+----------+----------+------------+------------+---------+
```

---

### Restore etcd With etcdutl

Now, lets restore etcd using etcdutl.

> Before starting the restore, you must stop the kube-apiserver and ETCD pods.

To fully stop the api-server and etcd pods, we need to move their manifest files out of the static pod directory to **/tmp** directory.

```bash
vagrant@controlplane:~$ sudo mv /etc/kubernetes/manifests/kube-apiserver.yaml /tmp/
vagrant@controlplane:~$ sudo mv /etc/kubernetes/manifests/etcd.yaml /tmp/
```

Once the manifests are moved, the kube-apiserver and ETCD pods will be terminated within a few seconds.

```bash
vagrant@controlplane:~$ kubectl get pods -n kube-system
The connection to the server 192.168.201.10:6443 was refused - did you specify the right host or port?
```

Since we have the backup file at **/opt/backup/etcd.db** location, we will use this snapshot to restore etcd to the **/var/lib/etcd-from-backup** location.

```bash
vagrant@controlplane:~$ ls /opt/backup/
etcd.db
```

```bash
vagrant@controlplane:~$ sudo etcdutl --data-dir /var/lib/etcd-from-backup snapshot restore /opt/backup/etcd.db
```

After restoring the etcd snapshot to a new directory, we need to update **etcd.yaml** because it still references the old data path.

For this, you should change only the **hostPath** in the etcd.yaml file.

```bash
vagrant@controlplane:~$ sudo nano /tmp/etcd.yaml
# ...
# ...

volumes:
  - hostPath:
      path: /etc/kubernetes/pki/etcd
      type: DirectoryOrCreate
    name: etcd-certs
  - hostPath:
      path: /var/lib/etcd-from-backup # the new restored directory
      type: DirectoryOrCreate
    name: etcd-data
```

Now, move the manifest files back to the original static pod directory. This will redeploy the kube-apiserver and ETCD pods.

```bash
vagrant@controlplane:~$ sudo mv /tmp/kube-apiserver.yaml /etc/kubernetes/manifests/
vagrant@controlplane:~$ sudo mv /tmp/etcd.yaml /etc/kubernetes/manifests/
```

After a few seconds, api-server and etcd pods will be active and you may verify them:

```bash
vagrant@controlplane:~$ kubectl get pods -n kube-system
NAME                                   READY   STATUS    RESTARTS         AGE
coredns-66bc5c9577-qh56x               1/1     Running   0                12h
coredns-66bc5c9577-tf5wj               1/1     Running   0                5h6m
etcd-controlplane                      1/1     Running   1 (2m21s ago)    119s
kube-apiserver-controlplane            1/1     Running   2 (2m21s ago)    24h
kube-controller-manager-controlplane   1/1     Running   15 (2m21s ago)   24h
kube-proxy-hwzhf                       1/1     Running   1 (9m4s ago)     24h
kube-proxy-jjv2v                       1/1     Running   1 (9m11s ago)    24h
kube-proxy-qnrl8                       1/1     Running   2 (2m21s ago)    24h
kube-scheduler-controlplane            1/1     Running   16 (2m21s ago)   24h
metrics-server-6cbd6b9dd4-hmtvv        1/1     Running   0                5h6m
```

You can check the etcd status using the following command:

```bash
vagrant@controlplane:~$ sudo etcdctl 
    --endpoints=https://192.168.201.10:2379 
    --cacert=/etc/kubernetes/pki/etcd/ca.crt 
    --cert=/etc/kubernetes/pki/etcd/server.crt 
    --key=/etc/kubernetes/pki/etcd/server.key 
    endpoint health

# https://192.168.201.10:2379 is healthy: successfully committed proposal: took = 26.990158ms
```
