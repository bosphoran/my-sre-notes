---
layout: post
title: "Command & Arguments in the Pod Manifest File"
date: 2026-06-11
categories: [k8s]
---

## Introduction

The **command** and **args** fields are similar to the **ENTRYPOINT** and **CMD** instructions in a Dockerfile.

![command & args]({{ site.baseurl }}/assets/img/k8s-course/command-args.jpg)

Lets create a simple pod manifest file and add the **command** and **args** fields.

```bash
vagrant@controlplane:~$ kubectl run args-pod --image=busybox --dry-run=client -o yaml > args-pod.yaml

# Edit the yaml file
vagrant@controlplane:~$ cat args-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    run: args-pod
  name: args-pod
spec:
  containers:
  - image: busybox
    name: args-pod
    command: ["/bin/sh"]
    args: ["-c", "date -u; sleep infinity;"]
    resources: {}
  dnsPolicy: ClusterFirst
  restartPolicy: Always
status: {}
```

Now, lets apply our manifest file:

```bash
vagrant@controlplane:~$ kubectl apply -f args-pod.yaml
pod/args-pod created

# Check the pod status
vagrant@controlplane:~$ kubectl get pods
NAME                      READY   STATUS      RESTARTS      AGE
args-pod                  1/1     Running     0             83s
```

Now, lets check the pod logs to see if the argument passed to the container really works:

```bash
vagrant@controlplane:~$ kubectl logs args-pod
Wed Jun 10 22:38:30 UTC 2026
```

For the **CKA** exam, you can directly use the imperative command by including the command field (**--command**).

```bash
vagrant@controlplane:~$ kubectl run my-pod 
                          --image=busybox 
                          --dry-run=client -o yaml 
                          --command -- /bin/sh -c 'echo "Hello from K8s"; sleep 3600' > busybox.yaml
```

Now lets check the **busybox.yaml** file.

```bash
vagrant@controlplane:~$ cat busybox.yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    run: my-pod
  name: my-pod
spec:
  containers:
  - command:
    - /bin/sh
    - -c
    - echo "Hello from K8s"; sleep 3600
    image: busybox
    name: my-pod
    resources: {}
  dnsPolicy: ClusterFirst
  restartPolicy: Always
status: {}
```

---

### Executing Shell Scripts From Args

Lets run a complex shell script with loop and function in args.

For this, we need to use the **pipe operator "\|"** with arguments. It allows you to write a command or script that spans multiple lines.

```bash
vagrant@controlplane:~$ cat script-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: script-pod
spec:
  containers:
  - name: script-container
    image: busybox
    command: ["/bin/sh", "-c"]
    args:
      - |
        print_numbers() {
          for i in $(seq 1 10); do
            echo $i
          done
        }
        print_numbers
        sleep infinity
```

Lets apply the manifest file.

```bash
vagrant@controlplane:~$ kubectl apply -f script-pod.yaml
pod/script-pod created

# Verify pod status
vagrant@controlplane:~$ kubectl get pods
NAME                      READY   STATUS      RESTARTS      AGE
busybox-multi-container   0/3     Completed   0             15h
multi-container-pod       2/2     Running     6 (44m ago)   17h
script-pod                1/1     Running     0             12s
```

Next, check the container logs.

```bash
vagrant@controlplane:~$ kubectl logs script-pod
1
2
3
4
5
6
7
8
9
10
```

---

### Practice

Create a pod using the **busybox** image. 

- Pod Name: env-pod.
- Pod should use ["/bin/sh", "-c"] as command.
- Custom Arguments:First the Pod should execute a custom command that prints a greeting message to the standard output. - Message is "Hello, Kubernetes world"
- Then, it should execute the printenv command.
- Then the pod should execute sleep infinity to keep the pod in running state.
- Verify the Output:After the Pod is created, verify that it executed the command successfully by checking its logs.
- Write the log output to a file called /tmp/env.log

```bash
vagrant@controlplane:~$ kubectl run env-pod --image=busybox --dry-run=client -o yaml > env-pod.yaml

# Open the manifest file
vagrant@controlplane:~$ nano env-pod.yaml
# ---
apiVersion: v1
kind: Pod
metadata:
  labels:
    run: env-pod
  name: env-pod
spec:
  containers:
  - image: busybox
    name: env-pod
    command: ["/bin/sh"]
    args: ["-c", "echo Hello, Kubernetes World!; printenv; sleep infinity;"]
    resources: {}
  dnsPolicy: ClusterFirst
  restartPolicy: Always
status: {}
# ---
# Apply the manifest file to create the pod
vagrant@controlplane:~$ kubectl apply -f env-pod.yaml
pod/env-pod created

# Verify the pod status
vagrant@controlplane:~$ kubectl get pods
NAME                      READY   STATUS      RESTARTS       AGE
env-pod                   1/1     Running     0              43s
```

Now, lets verify the pod output:

```bash
vagrant@controlplane:~$ kubectl logs env-pod
Hello, Kubernetes World!
KUBERNETES_SERVICE_PORT=443
KUBERNETES_PORT=tcp://10.96.0.1:443
HOSTNAME=env-pod
SHLVL=1
HOME=/root
KUBERNETES_PORT_443_TCP_ADDR=10.96.0.1
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
KUBERNETES_PORT_443_TCP_PORT=443
KUBERNETES_PORT_443_TCP_PROTO=tcp
KUBERNETES_SERVICE_PORT_HTTPS=443
KUBERNETES_PORT_443_TCP=tcp://10.96.0.1:443
KUBERNETES_SERVICE_HOST=10.96.0.1
PWD=/
```

Finally, lets write the pod output to a file.

```bash
vagrant@controlplane:~$ kubectl logs env-pod > /tmp/env.log
```
