---
layout: post
title: "Init Containers"
date: 2026-06-11
categories: [k8s]
---

## Introduction

An **Init Container** is a specialized container that runs and completes its tasks before the main application containers start inside a Pod. They are primarily used to perform setup utilities, fetch configuration files, or block the application from starting until external dependencies (like a database) are ready.

![Init Container]({{ site.baseurl }}/assets/img/k8s-course/init-container.png)

Suppose we have an application that requires a **secret** to connect to an **API**. Here, we can use an **init container** to retrieve the **secret** from a secret management service like **Vault** or **AWS** Secrets Manager and place it in a location within the pod where the application container can access it.

![Init Container 2]({{ site.baseurl }}/assets/img/k8s-course/init-container-2.png)

---

### Creating Init Containers

**Init containers** are defined in the **spec.initContainers** field of a Pod's manifest, similar to a regular **spec.containers** definition.

Lets create a pod with the following requirements:

- One **init container**, named **write-ip**, gets the pod IP using the **MY_POD_IP** environment variable and writes it to an **ip.txt** file inside the **/web-content** volume attached to the pod.
- The second **init container**, named **create-html**, reads the pod IP from the **/web-content/ip.txt** file, which contains the pod IP created by the first init container, and writes it to the **/web-content/index.html** file.
- Now, the main **Nginx container** (web-container) mounts the default **/usr/share/nginx/html** to the **/web-content** volume, where we have the **index.html** file.

![Init Container 3]({{ site.baseurl }}/assets/img/k8s-course/init-container-3.png)

Here is the full Pod YAML file:

```bash
vagrant@controlplane:~$ cat init-container.yaml
# ---
apiVersion: v1
kind: Pod
metadata:
  name: web-server-pod
spec:
  initContainers:
  - name: write-ip
    image: busybox
    command: ["sh", "-c", "echo $MY_POD_IP > /web-content/ip.txt; echo 'Wrote the Pod IP to ip.txt'"]
    env:
    - name: MY_POD_IP
      valueFrom:
        fieldRef:
          fieldPath: status.podIP
    volumeMounts:
    - name: web-content
      mountPath: /web-content
  - name: create-html
    image: busybox
    command: ["sh", "-c", "echo 'Hello, World! Your Pod IP is: ' > /web-content/index.html; cat /web-content/ip.txt >> /web-content/index.html; echo 'Created index.html with the Pod IP'"]
    volumeMounts:
    - name: web-content
      mountPath: /web-content
  containers:
  - name: web-container
    image: nginx
    volumeMounts:
    - name: web-content
      mountPath: /usr/share/nginx/html
  volumes:
  - name: web-content
    emptyDir: {}
```

Now, lets apply the manifest file to create our pod.

```bash
vagrant@controlplane:~$ kubectl apply -f init-container.yaml
pod/web-server-pod created
```

Lets go through the pod creation process.

```bash
# Init container one is starting
vagrant@controlplane:~$ kubectl get pods
NAME                      READY   STATUS      RESTARTS         AGE
web-server-pod            0/1     Init:0/2    0                8s

# Init container two is starting
vagrant@controlplane:~$ kubectl get pods
NAME                      READY   STATUS      RESTARTS         AGE
web-server-pod            0/1     Init:1/2    0                15s

# Main container and pod is initializing
vagrant@controlplane:~$ kubectl get pods
NAME                      READY   STATUS            RESTARTS         AGE
web-server-pod            0/1     PodInitializing   0                31s

# Pod is in running state
vagrant@controlplane:~$ kubectl get pods
NAME                      READY   STATUS      RESTARTS         AGE
web-server-pod            1/1     Running     0                57s
```

Let's check the init container logs and see if they have executed successfully.

```bash
vagrant@controlplane:~$ kubectl logs web-server-pod -c write-ip
Wrote the Pod IP to ip.txt

vagrant@controlplane:~$ kubectl logs web-server-pod -c create-html
Created index.html with the Pod IP
```

Finally, to verify if the **Nginx** pod is using the custom HTML, we **port forward** the local port **8080** to the Pod’s port **80**.

```bash
vagrant@controlplane:~$ kubectl port-forward pod/web-server-pod 8080:80

Forwarding from 127.0.0.1:8080 -> 80
Forwarding from [::1]:8080 -> 80
```

Now, open another terminal session into the controlplane node and type the following command to access the custom nginx page.

```bash
vagrant@controlplane:~$ curl http://localhost:8080
Hello, World! Your Pod IP is:
10.244.196.173
```

> **kubectl port-forward** is a foreground process that keeps a live tunnel open. It lets you open a tunnel from your local machine to a Pod or Service inside the cluster.

---

### Resources for Init Containers

You can add CPU and memory resources to init containers. However, if the init container has higher resource requests, the **scheduler** will reserve that amount for the Pod even after the **init container** has finished even though the app does not need it.

> Keep init container requests ≤ app container requests unless the init task genuinely needs more (like running a DB migration).

---

### Native Sidecars Using Init Containers

A **sidecar container** is a secondary container that runs alongside the main application container in the same Pod, sharing the Pod’s lifecycle, networking, and storage volumes.

To make an **init container** a **sidecar**, we need to add the **restartPolicy: Always** attribute to its **spec**. Otherwise, the container will act as a regular **init container**.

Let’s understand the use case with the following scenario.

- An **nginx webserver** pod with an nginx main container that writes logs to **/var/log/nginx** volume mount.
- We need a native sidecar **fluentd logging agent** container that reads all the nginx logs from **/var/log/nginx**.

The **restartPolicy: Always** is added to the logging-agent **init container** to make it behave like a **sidecar** container.

Create a file named **sidecar.yaml** and copy the following YAML.

```bash
apiVersion: v1
kind: Pod
metadata:
  name: webserver-pod
spec:
  # Sidecar container that starts before the main container
  initContainers:
  - name: logging-agent
    image: fluentd:latest
    # Makes this initContainer behave like a sidecar.
    restartPolicy: Always # It remains running and is restarted if it crashes.
    volumeMounts:
    - name: nginx-logs
      # Shared directory where nginx writes logs
      # and fluentd reads them.
      mountPath: /var/log/nginx
  containers:
  - name: nginx
    image: nginx:1.14.2
    ports:
    - containerPort: 80
    volumeMounts:
    - name: nginx-logs
      # Same shared volume mounted into nginx.
      # Any logs written here can be accessed by fluentd.
      mountPath: /var/log/nginx
  volumes:
  - name: nginx-logs
    # Temporary shared storage available to all containers
    # in the Pod. Created when the Pod starts and deleted
    # when the Pod is removed.
    emptyDir: {} # emptyDir creates an empty directory when the Pod starts.
```

Now, lets apply the yaml file to create our pod.

```bash
vagrant@controlplane:~$ kubectl apply -f sidecar.yaml
pod/webserver-pod created
```

If you check the **pod status**, you can see **2/2 containers** that are in **running status**. One is the native **sidecar init container** and the other one is the **main Nginx container**.

```bash
vagrant@controlplane:~$ kubectl get pods
NAME                      READY   STATUS      RESTARTS       AGE
webserver-pod             2/2     Running     0              98s
```

---

### How Do Init Containers Work?

- **kubelet** runs the init containers in the order they appear in the Pod’s spec ensuring that each container completes its task before starting the next (**Sequential Execution**). Meaning only one init container runs at a time.
- Init Containers run before the main application containers start.
- If the Pod is restarted, all its init containers will run again.

![Init Container 4]({{ site.baseurl }}/assets/img/k8s-course/init-container-4.gif)

---

### Example Scenario 01: Modify a Pod to Include Init Containers

Create an **init.yaml** manifest file and perform the given tasks.

- **Modify the Pod Manifest**: Incorporate **two init containers** into the Pod's spec, utilizing the **busybox image** for both. Assign them names that describe their initialization roles, such as **prepare-config** and **setup-dependencies**.
- **Init Tasks**: These containers should perform simple tasks. Use **command and args**, add **sleep** for **30 seconds**, and **echo "hello <container-name>"**.
- **Deploy the Pod**: Apply the modified manifest to your K8s cluster.
- **Check Init Containers Execution & Pod Status**: Confirm the execution of both init containers. Ensure the Pod reaches the **Running** state after the init containers have finished their tasks.
- Write init container logs to **/tmp/init.log** file and verify the logs.

```bash
apiVersion: v1
kind: Pod
metadata:
  name: zacademy-app
spec:
  initContainers:
  - image: busybox
    name: prepare-env
    command: ["/bin/sh"]
    args: ["-c", "echo Hello $(printenv HOSTNAME) from prepare-env container; sleep 30;"]
  - image: busybox
    name: print-hostname
    command: ["/bin/sh"]
    args: ["-c", "echo Hello $(printenv HOSTNAME) from print-hostname container;"]
  containers:
  - image: busybox
    name: zacademy-app-container
    command: ["/bin/sh"]
    args: ["-c", "echo Welcome to the CKA course; sleep 1000;"]
```

Lets apply the yaml file to create our containers.

```bash
vagrant@controlplane:~$ kubectl apply -f zacademy-init.yaml
pod/zacademy-app created
```

Verify init container initialization and pod status.

```bash
vagrant@controlplane:~$ kubectl get pods
NAME                      READY   STATUS      RESTARTS       AGE
zacademy-app              0/1     Init:0/2    0              24s

vagrant@controlplane:~$ kubectl get pods
NAME                      READY   STATUS      RESTARTS       AGE
zacademy-app              0/1     Init:1/2    0              35s

vagrant@controlplane:~$ kubectl get pods
NAME                      READY   STATUS      RESTARTS       AGE
zacademy-app              1/1     Running     0                46s
```

Next, lets check the container logs:

```bash
# zacademy-app-container
vagrant@controlplane:~$ kubectl logs zacademy-app -c zacademy-app-container
Welcome to the CKA course by zacademy-app-container

# prepare-env & print-hostname containers
vagrant@controlplane:~$ kubectl logs zacademy-app -c prepare-env
Hello zacademy-app from prepare-env container

vagrant@controlplane:~$ kubectl logs zacademy-app -c print-hostname
Hello zacademy-app from print-hostname container
```

Finally, save the logs to **/tmp/init.log**.

```bash
vagrant@controlplane:~$ kubectl logs zacademy-app -c prepare-env >> /tmp/init.log
vagrant@controlplane:~$ kubectl logs zacademy-app -c print-hostname >> /tmp/init.log

# Verify the log file
vagrant@controlplane:~$ cat /tmp/init.log
Hello zacademy-app from prepare-env container
Hello zacademy-app from print-hostname container
```

---

### Example Scenario 02: Add Init Container with Resources to an Existing Pod

Complete the following tasks:

- Pod name: **zacademy-web**
- Create the **web-init.yaml** manifest file.
- Modify the manifest to include an **init container** with the following specifications:
  - Name: init-permissions
  - Image: busybox
  - Command: Echo "Setting up permissions" and sleep for 5 seconds
  - Resources: Request 50m CPU, 64Mi memory | Limit 100m CPU, 128Mi memory
- Configure the nginx container with resources:
  - Request 100m CPU, 128Mi memory
  - Limit 200m CPU, 256Mi memory
- Once configured redeployed the pod and confirm the init container runs successfully before the main container starts.

Here is the manifest file.

```bash
apiVersion: v1
kind: Pod
metadata:
  labels:
    run: zacademy-web
  name: zacademy-web
spec:
  initContainers:
  - image: busybox
    name: init-permissions
    command: ["/bin/sh"]
    args: ["-c", "echo Setting up permissions; sleep 5;"]
    resources:
     requests:
      cpu: "100m"
      memory: "128Mi"
  containers:
  - image: nginx
    name: zacademy-web
    ports:
    - containerPort: 80
    resources:
     requests:
      cpu: "100m"
      memory: "128Mi"
     limits:
      cpu: "200m"
      memory: "256Mi"
  dnsPolicy: ClusterFirst
  restartPolicy: Always
status: {}
```

Lets apply the yaml file.

```bash
vagrant@controlplane:~$ kubectl apply -f zacademy-web.yaml
pod/zacademy-web created
```

```bash
# init container initialization phase
vagrant@controlplane:~$ kubectl get pods
NAME                      READY   STATUS      RESTARTS        AGE
zacademy-web              0/1     Init:0/1    0               5s

# main container status
vagrant@controlplane:~$ kubectl get pods
NAME                      READY   STATUS      RESTARTS        AGE
zacademy-web              1/1     Running     0               44s
```

Check the container logs:

```bash
vagrant@controlplane:~$ kubectl logs zacademy-web -c init-permissions
Setting up permissions
```

Finally, check resource allocation for all containers in the **pod description**.

```bash
vagrant@controlplane:~$ kubectl describe pod zacademy-web
Name:             zacademy-web
Namespace:        default
Priority:         0
Service Account:  default
Node:             node02/192.168.201.12
Start Time:       Thu, 11 Jun 2026 23:45:05 +0000
Labels:           run=zacademy-web
# ...
# ...
Containers:
  zacademy-web:
    Container ID:   containerd://094a1bfb0b1642494570bf1fc7b0477a2665fee7e306a0a7706538e84f0726f6
    Image:          nginx
    Image ID:       docker.io/library/nginx@sha256:1df1a963b5b8a3315c81c9d3c6fbd3324479a16c5e266bc6642c080be636145f
    Port:           80/TCP
    Host Port:      0/TCP
    State:          Running
      Started:      Thu, 11 Jun 2026 23:45:19 +0000
    Ready:          True
    Restart Count:  0
    Limits:
      cpu:     200m
      memory:  256Mi
    Requests:
      cpu:        100m
      memory:     128Mi
    Environment:  <none>
```

---
