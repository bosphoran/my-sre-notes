---
layout: post
title: "Environment Variables"
date: 2026-07-24
categories: [k8s]
---

## Introduction

**Environment variables** let you configure applications without hardcoding values, keeping code portable and easier to manage across environments. 

> **Environment variables** separate configuration from code and provide a simple way for K8s to pass dynamic values into containers. or example, a frontend app can get its backend API URL or a backend service can access its database endpoint through environment variables.

| Type | Example |
| :---: | :---: |
| Database Configuration | DB_HOST=mongo.database.svc.cluster.local <br>DB_PORT=3306 <br>DB_NAME=dev_database |
| API Keys and Credentials | API_KEY=dev_api_key <br> API_SECRET=dev_api_secret |
| Proxy Configuration | HTTP_PROXY=http://proxy.example.com:8080 <br>HTTPS_PROXY=http://proxy.example.com:8080 |

### Default Environment Variables

Every container running inside the pod has a list of default environment variables assigned by K8s.

The following command spins up a temporary BusyBox Pod, prints its environment variables, and then cleans itself up automatically.

```bash
# Lets test this by executing printenv command on a busybox container
vagrant@controlplane:~$ kubectl run busybox --image=busybox:latest --rm -i --restart=Never -- printenv
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
HOSTNAME=busybox
CUSTOM_SRE_WEB_SERVICE_HOST=10.111.98.178
CUSTOM_SRE_WEB_PORT=tcp://10.111.98.178:80
CUSTOM_SRE_WEB_PORT_80_TCP=tcp://10.111.98.178:80
CUSTOM_SRE_WEB_PORT_80_TCP_ADDR=10.111.98.178
KUBERNETES_SERVICE_PORT_HTTPS=443
KUBERNETES_PORT_443_TCP=tcp://10.96.0.1:443
KUBERNETES_PORT_443_TCP_ADDR=10.96.0.1
CUSTOM_SRE_WEB_PORT_80_TCP_PROTO=tcp
MY_RELEASE_PROMETHEUS_ADAPTER_SERVICE_HOST=10.102.119.32
MY_RELEASE_PROMETHEUS_ADAPTER_SERVICE_PORT=443
MY_RELEASE_PROMETHEUS_ADAPTER_PORT_443_TCP_PORT=443
MY_RELEASE_PROMETHEUS_ADAPTER_PORT_443_TCP_ADDR=10.102.119.32
KUBERNETES_SERVICE_HOST=10.96.0.1
KUBERNETES_SERVICE_PORT=443
CUSTOM_SRE_WEB_SERVICE_PORT=80
CUSTOM_SRE_WEB_PORT_80_TCP_PORT=80
MY_RELEASE_PROMETHEUS_ADAPTER_PORT=tcp://10.102.119.32:443
MY_RELEASE_PROMETHEUS_ADAPTER_PORT_443_TCP=tcp://10.102.119.32:443
KUBERNETES_PORT_443_TCP_PROTO=tcp
MY_RELEASE_PROMETHEUS_ADAPTER_SERVICE_PORT_HTTPS=443
MY_RELEASE_PROMETHEUS_ADAPTER_PORT_443_TCP_PROTO=tcp
KUBERNETES_PORT=tcp://10.96.0.1:443
KUBERNETES_PORT_443_TCP_PORT=443
HOME=/root
pod "busybox" deleted from default namespace
```

The **environment variables** prefixed with **KUBERNETES_** and **HOSTNAME** are automatically injected by K8s and provide information about the Kubernetes environment, such as the API server details and service information.

### Working With Custom Environment Variables

To define custom environment variables in K8s, you need to specify them in the pod template using the **env** property under the container **spec**.

```bash
# Create an env-pod.yaml file and define three environment variables named APP_ENV, DB_HOST, and API_KEY
apiVersion: v1
kind: Pod
metadata:
  name: env-pod
spec:
  containers:
  - name: env-container
    image: nginx:latest
    env:
    - name: APP_ENV
      value: "production"
    - name: DB_HOST
      value: "mongo.example.com"
    - name: API_KEY
      value: "sd234sdfsdfgwe5r"
```

- The **env** field is used to specify the custom environment variables for the container.
- Each environment variable is defined as a separate entry under **env**.
- The **name** field specifies the name of the environment variable.
- The **value** field specifies the value assigned to the environment variable.

```bash
# deploy the pod manifest
vagrant@controlplane:~$ kubectl apply -f env-pod-2.yaml
pod/env-pod created

# if you describe the pod, you can see the variables we specified in the container spec
vagrant@controlplane:~$ kubectl describe pod env-pod | grep Env -A5
    Environment:
      APP_ENV:  production
      DB_HOST:  mongo.example.com
      API_KEY:  sd234sdfsdfgwe5r
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-qk95x (ro)

# Run the printenv command inside the running env-pod pod. It will list all the available environment variables.
vagrant@controlplane:~$ kubectl exec env-pod -- printenv
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
HOSTNAME=env-pod
NGINX_VERSION=1.31.3
NJS_VERSION=1.0.0
NJS_RELEASE=1~trixie
ACME_VERSION=0.4.1
PKG_RELEASE=1~trixie
DYNPKG_RELEASE=1~trixie
APP_ENV=production
DB_HOST=mongo.example.com
API_KEY=sd234sdfsdfgwe5r
MY_RELEASE_PROMETHEUS_ADAPTER_SERVICE_PORT=443
MY_RELEASE_PROMETHEUS_ADAPTER_PORT=tcp://10.102.119.32:443
MY_RELEASE_PROMETHEUS_ADAPTER_PORT_443_TCP_PORT=443
MY_RELEASE_PROMETHEUS_ADAPTER_PORT_443_TCP_ADDR=10.102.119.32
KUBERNETES_SERVICE_PORT_HTTPS=443
KUBERNETES_PORT=tcp://10.96.0.1:443
KUBERNETES_PORT_443_TCP_PORT=443
CUSTOM_SRE_WEB_SERVICE_HOST=10.111.98.178
MY_RELEASE_PROMETHEUS_ADAPTER_PORT_443_TCP=tcp://10.102.119.32:443
KUBERNETES_SERVICE_HOST=10.96.0.1
KUBERNETES_SERVICE_PORT=443
KUBERNETES_PORT_443_TCP_ADDR=10.96.0.1
CUSTOM_SRE_WEB_SERVICE_PORT=80
CUSTOM_SRE_WEB_PORT=tcp://10.111.98.178:80
CUSTOM_SRE_WEB_PORT_80_TCP=tcp://10.111.98.178:80
CUSTOM_SRE_WEB_PORT_80_TCP_ADDR=10.111.98.178
MY_RELEASE_PROMETHEUS_ADAPTER_SERVICE_HOST=10.102.119.32
MY_RELEASE_PROMETHEUS_ADAPTER_SERVICE_PORT_HTTPS=443
MY_RELEASE_PROMETHEUS_ADAPTER_PORT_443_TCP_PROTO=tcp
CUSTOM_SRE_WEB_PORT_80_TCP_PROTO=tcp
CUSTOM_SRE_WEB_PORT_80_TCP_PORT=80
KUBERNETES_PORT_443_TCP=tcp://10.96.0.1:443
KUBERNETES_PORT_443_TCP_PROTO=tcp
HOME=/root

# Clean up the pod
vagrant@controlplane:~$ kubectl delete pod env-pod
pod "env-pod" deleted from default namespace
```

### Imperative Method (For CKA)

From a **CKA perspective**, you can create pod YAML with environment variables using the imperative method.

> You can use the **--env flag** followed by the key and value to set the environment variable. For example, **--env=APP_ENV=dev**.

Let's create a nginx pod with two environment variables **APP_ENV=dev** and **DB_HOST=mongo.zacademy.org**.

```bash
vagrant@controlplane:~$ kubectl run env-pod --image=nginx:latest --env=APP_ENV=dev --env=DB_HOST=mongo.zacademy.org --dry-run=client -o yaml > env-pod-3.yaml

# deploy the pod
vagrant@controlplane:~$ kubectl apply -f env-pod-3.yaml
pod/env-pod created

# verify the variables using the describe command
vagrant@controlplane:~$ kubectl describe pod env-pod | grep Env -A5
    Environment:
      APP_ENV:  dev
      DB_HOST:  mongo.zacademy.org
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-rfspd (ro)
Conditions:

# Clean up the pod
vagrant@controlplane:~$ kubectl delete pod env-pod
pod "env-pod" deleted from default namespace
```

### Environment Variables Mutability

> Environment variables in a Pod are fixed once the Pod is created. To change them, you must recreate the Pod. 

In higher-level resources like Deployments, updating the Pod template triggers a rolling update, replacing old Pods with new ones that have the updated environment variables.

If you need to provide dynamic or configurable values for environment variables, you can use variables through **ConfigMaps** or **Secrets**.

---

### Environment Variables With Downward API

Let's consider a scenario where an application running inside a container needs information about the pod it is running in. For example, details about the pod name, namespace, pod IP address, and pod annotations.

One way is to make **API calls** to the **API server** using a service account to get the details. However, K8s provides an efficient way to get all the pod information using the **Downward API**.

> The Downward API allows containers running in the pod to access pod and environment information, retrieving runtime metadata and configuration details without directly communicating with the K8s **API server**.

1. Environment variables
2. Volume files

### Exposing Downward API Values via Environment Variables

We can obtain the values using the env block with **valueFrom**, **fieldRef**, and **fieldPath** parameters as shown below.

```bash
env:
- name: POD_NAME
  valueFrom:
    fieldRef:
      fieldPath: metadata.name
- name: POD_IP
  valueFrom:
    fieldRef:
      fieldPath: status.podIP
```

Here **POD_NAME** & **POD_IP** are the environment variable key and its values would be derived from the downward API **metadata.name** and **status.podIP** values.

> **metadata** field provides static information about the resource. **status** field provides dynamic information about the current state of the resource.

Let's deploy a pod to access the pod name, namespace and IP details.

```bash
# Create a api-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: api-pod
  labels:
    app: api-app
spec:
  containers:
  - name: api-container
    image: nginx:latest
    env:
    - name: POD_NAME
      valueFrom:
        fieldRef:
          fieldPath: metadata.name
    - name: POD_NAMESPACE
      valueFrom:
        fieldRef:
          fieldPath: metadata.namespace
    - name: POD_IP
      valueFrom:
        fieldRef:
          fieldPath: status.podIP

# Deploy the pod
vagrant@controlplane:~$ kubectl apply -f api-pod.yaml
pod/api-pod created

# validate the variables using printenv command
vagrant@controlplane:~$ kubectl exec -it api-pod -- printenv
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
HOSTNAME=api-pod
NGINX_VERSION=1.31.3
NJS_VERSION=1.0.0
NJS_RELEASE=1~trixie
ACME_VERSION=0.4.1
PKG_RELEASE=1~trixie
DYNPKG_RELEASE=1~trixie
POD_IP=10.244.140.80
POD_NAME=api-pod
POD_NAMESPACE=default
KUBERNETES_PORT_443_TCP=tcp://10.96.0.1:443
CUSTOM_SRE_WEB_PORT_80_TCP_PROTO=tcp
MY_RELEASE_PROMETHEUS_ADAPTER_SERVICE_HOST=10.102.119.32
MY_RELEASE_PROMETHEUS_ADAPTER_SERVICE_PORT_HTTPS=443
MY_RELEASE_PROMETHEUS_ADAPTER_PORT_443_TCP=tcp://10.102.119.32:443
MY_RELEASE_PROMETHEUS_ADAPTER_PORT_443_TCP_PORT=443
KUBERNETES_SERVICE_HOST=10.96.0.1
KUBERNETES_SERVICE_PORT=443
KUBERNETES_PORT=tcp://10.96.0.1:443
KUBERNETES_PORT_443_TCP_PROTO=tcp
MY_RELEASE_PROMETHEUS_ADAPTER_PORT_443_TCP_PROTO=tcp
KUBERNETES_SERVICE_PORT_HTTPS=443
KUBERNETES_PORT_443_TCP_PORT=443
KUBERNETES_PORT_443_TCP_ADDR=10.96.0.1
CUSTOM_SRE_WEB_SERVICE_HOST=10.111.98.178
CUSTOM_SRE_WEB_SERVICE_PORT=80
CUSTOM_SRE_WEB_PORT_80_TCP_PORT=80
CUSTOM_SRE_WEB_PORT_80_TCP_ADDR=10.111.98.178
MY_RELEASE_PROMETHEUS_ADAPTER_SERVICE_PORT=443
MY_RELEASE_PROMETHEUS_ADAPTER_PORT=tcp://10.102.119.32:443
MY_RELEASE_PROMETHEUS_ADAPTER_PORT_443_TCP_ADDR=10.102.119.32
CUSTOM_SRE_WEB_PORT=tcp://10.111.98.178:80
CUSTOM_SRE_WEB_PORT_80_TCP=tcp://10.111.98.178:80
TERM=xterm
HOME=/root

# Clean up the pod
vagrant@controlplane:~$ kubectl delete pod api-pod
pod "api-pod" deleted from default namespace
```

### Exposing Downward API Values via Volumes

K8s mounts the requested metadata into the pod’s filesystem, making it available to the containers as files. This can be useful for applications that need to read configuration data from files rather than environment variables.

> Labels and annotations are mounted as volumes inside the container at /etc/podinfo using volumeMounts.

```bash
# Create downward-api-volume.yaml
vagrant@controlplane:~$ kubectl apply -f downward-api-volume.yaml
pod/downward-api-volume created

# 
vagrant@controlplane:~$ kubectl get pods
NAME                                            READY   STATUS    RESTARTS      AGE
downward-api-volume                             1/1     Running   0             21s

# list the files in the /etc/podinfo mount path
vagrant@controlplane:~$ kubectl exec -it downward-api-volume -- ls -laR /etc/podinfo
/etc/podinfo:
total 4
drwxrwxrwt    3 root     root           120 Jul 24 06:10 .
drwxr-xr-x    1 root     root          4096 Jul 24 06:10 ..
drwxr-xr-x    2 root     root            80 Jul 24 06:10 ..2026_07_24_06_10_57.1309207126
lrwxrwxrwx    1 root     root            32 Jul 24 06:10 ..data -> ..2026_07_24_06_10_57.1309207126
lrwxrwxrwx    1 root     root            18 Jul 24 06:10 annotations -> ..data/annotations
lrwxrwxrwx    1 root     root            13 Jul 24 06:10 labels -> ..data/labels

/etc/podinfo/..2026_07_24_06_10_57.1309207126:
total 8
drwxr-xr-x    2 root     root            80 Jul 24 06:10 .
drwxrwxrwt    3 root     root           120 Jul 24 06:10 ..
-rw-r--r--    1 root     root           939 Jul 24 06:10 annotations
-rw-r--r--    1 root     root            17 Jul 24 06:10 labels
```

From the output, you can see that the files are actually symbolic links pointing to the **..data** folder, which itself is a symbolic link to a timestamped directory containing the actual annotations and label files.

This structure ensures that if there is a change in the pod metadata, such as a label update or an annotation update, K8s can automatically switch the **..data** link to point to a new timestamped directory with the updated metadata.

```bash
# You can also view the contents of the individual files
vagrant@controlplane:~$ kubectl exec -it downward-api-volume -- cat /etc/podinfo/annotations
cni.projectcalico.org/containerID="52123d8b833cd20b169a1a108bb576f6270df40d9d7f29c0378f2ddb1508ab5a"
cni.projectcalico.org/podIP="10.244.140.81/32"
cni.projectcalico.org/podIPs="10.244.140.81/32"
kubectl.kubernetes.io/last-applied-configuration="{\"apiVersion\":\"v1\",\"kind\":\"Pod\",\"metadata\":{\"annotations\":{},\"labels\":{\"environment\":\"dev\"},\"name\":\"downward-api-volume\",\"namespace\":\"default\"},\"spec\":{\"containers\":[{\"command\":[\"sh\",\"-c\",\"sleep infinity\"],\"image\":\"busybox\",\"name\":\"downward-api-container\",\"volumeMounts\":[{\"mountPath\":\"/etc/podinfo\",\"name\":\"podinfo\"}]}],\"volumes\":[{\"downwardAPI\":{\"items\":[{\"fieldRef\":{\"fieldPath\":\"metadata.labels\"},\"path\":\"labels\"},{\"fieldRef\":{\"fieldPath\":\"metadata.annotations\"},\"path\":\"annotations\"}]},\"name\":\"podinfo\"}]}}\n"
kubernetes.io/config.seen="2026-07-24T06:10:54.164165626Z"
kubernetes.io/config.source="api"

# 
vagrant@controlplane:~$ kubectl exec -it downward-api-volume -- cat /etc/podinfo/labels
environment="dev"

# Clean up the pod.
vagrant@controlplane:~$ kubectl delete pod downward-api-volume
pod "downward-api-volume" deleted from default namespace
```
