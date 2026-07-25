---
layout: post
title: "ConfigMaps"
date: 2026-07-25
categories: [k8s]
---

## Introduction

A **ConfigMap** is a K8s object used to store non-sensitive configuration data (like key-value pairs, environment variables, or entire config files).
It allows you to separate configuration from application code, making your workloads more flexible and portable.

- You define a ConfigMap with key-value data.
- Pods can consume this data in different ways:
  - As environment variables inside containers.
  - As command-line arguments.
  - Mounted as configuration files inside a Pod.

> **ConfigMaps** let you manage configuration separately from your application code, making K8s workloads easier to update and maintain across environments.

```bash
apiVersion: v1
kind: ConfigMap
metadata:
  name: db-config
data:
  DB_HOST: "database.zacademy.org"
  DB_PORT: "5432"
  DB_USER: "admin"
  DB_NAME: "zacdatabase"
```

> ConfigMap is an API object used to store non-sensitive data. It should not be used for storing passwords, API keys, or other sensitive information.

### ConfigMap Data

Here is an example of a **ConfigMap** that contains an nginx.conf file along with other key-value pairs:

```bash
apiVersion: v1                 # API version for Kubernetes core objects
kind: ConfigMap                # Declares this object as a ConfigMap
metadata:
  name: nginx-config           # Name of the ConfigMap (used to reference it in Pods)
data:                          # Holds the configuration data as key-value pairs
  client_max_body_size: "10m"  # Example key-value pair: sets max body size for requests
  keepalive_requests: "100"    # Another key-value pair: limits number of keepalive requests
  nginx.conf: |                # Multi-line value (YAML block scalar) for full config file
    events {
      worker_connections 1024; # NGINX events block: defines max worker connections
    }

    http {
      server {
        listen 80;             # Server listens on port 80
        server_name example.com; # Defines the server name (domain)

        location / {
          root /usr/share/nginx/html; # Root directory for serving files
          index index.html;           # Default file served at /
        }
      }
    }
```

### Injecting ConfigMap Data into Pods

Pods can consume the ConfigMap data in the following ways.

- **Environment Variables:** You can expose ConfigMap data as environment variables in your container.
- **Volume Mounts:** You can mount a ConfigMap as a volume, which allows you to use the data as files within your container. This is useful for larger configurations or files like certificates.

### ConfigMap - Pod Relationship

A single ConfigMap can be used by multiple pods in the same namespace, establishing a one-to-many relationship. Also, a single pod can use multiple ConfigMaps.

![ConfigMaps]({{ site.baseurl }}/assets/img/k8s-course/configmaps.jpg)

When you update a ConfigMap, all pods using it can potentially receive the updated data (though some may require a restart to pick up changes, depending on how they're consuming the ConfigMap).

---

### Working With ConfigMap

When you deploy an Nginx pod, it displays the default welcome page using the default **index.html** page.

We will use a **ConfigMap** object to replace the default page with a custom HTML page at runtime. This way, you will understand how data from a ConfigMap is injected into a pod.

First, let's deploy the **Nginx web server** and check the default web page.

```bash
# Create webserver.yaml
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
      - name: nginx
        image: nginx:latest
```

Deploy it and ensure the pod is running

```bash
# deploy the yaml file
vagrant@controlplane:~$ kubectl apply -f webserver-configmaps.yaml
deployment.apps/nginx-deployment created

# verify pod status
vagrant@controlplane:~$ kubectl get pods
NAME                                            READY   STATUS    RESTARTS      AGE
nginx-deployment-7c5d8bf9f7-kl6j7               1/1     Running   0             15s

# get the pod IP in the POD_IP environment variable using JSONPath
vagrant@controlplane:~$ POD_IP=$(kubectl get pods -l app=nginx -o jsonpath='{.items[0].status.podIP}')
```

Let's curl the POD IP to check the default Nginx home page. 

```bash
vagrant@controlplane:~$ curl http://$POD_IP
```

```html
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
html { color-scheme: light dark; }
body { width: 35em; margin: 0 auto;
font-family: Tahoma, Verdana, Arial, sans-serif; }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, nginx is successfully installed and working.
Further configuration is required for the web server, reverse proxy,
API gateway, load balancer, content cache, or other features.</p>

<p>For online documentation and support please refer to
<a href="https://nginx.org/">nginx.org</a>.<br/>
To engage with the community please visit
<a href="https://community.nginx.org/">community.nginx.org</a>.<br/>
For enterprise grade support, professional services, additional
security features and capabilities please refer to
<a href="https://f5.com/nginx">f5.com/nginx</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
```

Next, let's see how we can replace this default page with a custom HTML page using ConfigMap.

Create a file named **configmap.yaml**. It will create a ConfigMap named **custom-index-html**.

```bash
apiVersion: v1
kind: ConfigMap
metadata:
  name: custom-index-html
data:
  backend_api: "api.techiescamp.com"
  vault_url: "vault.techiescamp.com/secrets" 
  index.html: |
    <!DOCTYPE html>
    <html>
    <head>
        <title>Custom HTML Page</title>
    </head>
    <body>
        <h1>Heloo CKA Aspirants</h1>
        <p>This is a custom HTML page served by Nginx.</p>
    </body>
    </html>
```

n the above YAML, under the data section, you can see two single-line key-value pairs. We will use this data for the pod to consume as environment variables.

```GOLANG
backend_api: "api.techiescamp.com"
vault_url: "vault.techiescamp.com/secrets"
```

Also, you can see **index.html** as the key and the custom HTML script added as the value using multi-line YAML content. It displays the message **Hello CKA Aspirants** We will mount this key-value pair as a volume to the pod.

```bash
vagrant@controlplane:~$ kubectl apply -f configmap.yaml
configmap/custom-index-html created

# describe the custom-index-html you can see all the key and value. cm is the short name of configmap.
vagrant@controlplane:~$ kubectl describe cm custom-index-html
Name:         custom-index-html
Namespace:    default
Labels:       <none>
Annotations:  <none>

Data
====
backend_api:
----
api.techiescamp.com

index.html:
----
<!DOCTYPE html>
<html>
<head>
    <title>Custom HTML Page</title>
</head>
<body>
    <h1>Heloo CKA Aspirants</h1>
    <p>This is a custom HTML page served by Nginx.</p>
</body>
</html>


vault_url:
----
vault.techiescamp.com/secrets


BinaryData
====

Events:  <none>
```

### Add ConfigMap to Deployment

We are trying to inject the custom-index-html ConfigMap data into the pod as a file (index.html). To achieve this, we need to use volumes and specify the ConfigMap as the volume type.

In the above YAML, we consume the two single-line key-value pairs as environment variables (**BACKEND_API** and **VAULT_URL**) using the following block. We utilize valueFrom and **configMapKeyRef** to assign the values from the ConfigMap to the respective environment variables.

```bash
env:
  - name: BACKEND_API
    valueFrom:
      configMapKeyRef:
        name: custom-index-html
        key: backend_api
  - name: VAULT_URL
    valueFrom:
      configMapKeyRef:
        name: custom-index-html
        key: vault_url
```

We are also mounting the ConfigMap as a volume to the deployment. It mounts the custom-index-html ConfigMap into the Nginx container under the **/usr/share/nginx/html** directory.

```bash
volumes:
  - name: html-volume
    configMap:
      name: custom-index-html
```

Let's apply this configuration again. Since we used a Deployment, it will automatically roll out the updated pod to reflect the changes made in the manifest.

```bash
vagrant@controlplane:~$ kubectl apply -f cm-deploy.yaml
deployment.apps/nginx-deployment configured

# verify deployment
vagrant@controlplane:~$ kubectl get deploy
NAME                            READY   UP-TO-DATE   AVAILABLE   AGE
nginx-deployment                1/1     1            1           51m

# verify pod status
vagrant@controlplane:~$ kubectl get pods
NAME                                            READY   STATUS    RESTARTS      AGE
nginx-deployment-7c978dc764-zvknl               1/1     Running   0             71s
```

### Validate Configmap Environment Variables

Now let's verify if the ConfigMap has injected the key-value pairs as environment variables inside the nginx pod.

```bash
vagrant@controlplane:~$ kubectl exec -it nginx-deployment-7c978dc764-zvknl -- printenv
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
HOSTNAME=nginx-deployment-7c978dc764-zvknl
NGINX_VERSION=1.31.3
NJS_VERSION=1.0.0
NJS_RELEASE=1~trixie
ACME_VERSION=0.4.1
PKG_RELEASE=1~trixie
DYNPKG_RELEASE=1~trixie
# the two environment variables injected from the ConfigMap
BACKEND_API=api.techiescamp.com
VAULT_URL=vault.techiescamp.com/secrets
# 
MY_RELEASE_PROMETHEUS_ADAPTER_PORT=tcp://10.102.119.32:443
MY_RELEASE_PROMETHEUS_ADAPTER_PORT_443_TCP_ADDR=10.102.119.32
CUSTOM_SRE_WEB_PORT=tcp://10.111.98.178:80
KUBERNETES_PORT_443_TCP=tcp://10.96.0.1:443
MY_RELEASE_PROMETHEUS_ADAPTER_PORT_443_TCP_PROTO=tcp
CUSTOM_SRE_WEB_PORT_80_TCP=tcp://10.111.98.178:80
CUSTOM_SRE_WEB_PORT_80_TCP_PROTO=tcp
KUBERNETES_SERVICE_HOST=10.96.0.1
KUBERNETES_SERVICE_PORT_HTTPS=443
KUBERNETES_PORT=tcp://10.96.0.1:443
KUBERNETES_PORT_443_TCP_PORT=443
MY_RELEASE_PROMETHEUS_ADAPTER_SERVICE_HOST=10.102.119.32
MY_RELEASE_PROMETHEUS_ADAPTER_PORT_443_TCP_PORT=443
CUSTOM_SRE_WEB_PORT_80_TCP_PORT=80
CUSTOM_SRE_WEB_PORT_80_TCP_ADDR=10.111.98.178
KUBERNETES_PORT_443_TCP_PROTO=tcp
KUBERNETES_PORT_443_TCP_ADDR=10.96.0.1
MY_RELEASE_PROMETHEUS_ADAPTER_SERVICE_PORT=443
MY_RELEASE_PROMETHEUS_ADAPTER_PORT_443_TCP=tcp://10.102.119.32:443
CUSTOM_SRE_WEB_SERVICE_HOST=10.111.98.178
CUSTOM_SRE_WEB_SERVICE_PORT=80
KUBERNETES_SERVICE_PORT=443
MY_RELEASE_PROMETHEUS_ADAPTER_SERVICE_PORT_HTTPS=443
TERM=xterm
HOME=/root
```

### Validate ConfigMap Volume

Now, let's validate if the ConfigMap volume has mounted all the key-value pairs as files inside the **/usr/share/nginx/html** directory.

```bash
# describe the Deployment, to see if the ConfigMap has mounted as a volume
vagrant@controlplane:~$ kubectl describe deploy nginx-deployment | grep volume -A4
      /usr/share/nginx/html from html-volume (rw)
  Volumes:
   html-volume:
    Type:          ConfigMap (a volume populated by a ConfigMap)
    Name:          custom-index-html
    Optional:      false
  Node-Selectors:  <none>
  Tolerations:     <none>

# check if the ConfigMap data is injected into the /usr/share/nginx/html path
vagrant@controlplane:~$ kubectl exec -it nginx-deployment-7c978dc764-zvknl -- ls /usr/share/nginx/html
backend_api  index.html  vault_url
# As you can see, it has created three files with the key names as the filenames and the values as the file contents.

vagrant@controlplane:~$ kubectl exec -it nginx-deployment-7c978dc764-zvknl -- cat  /usr/share/nginx/html/backend_api
api.techiescamp.com

#
vagrant@controlplane:~$ kubectl exec -it nginx-deployment-7c978dc764-zvknl -- cat  /usr/share/nginx/html/vault_url
vault.techiescamp.com/secrets

# 
vagrant@controlplane:~$ kubectl exec -it nginx-deployment-7c978dc764-zvknl -- cat  /usr/share/nginx/html/index.html
<!DOCTYPE html>
<html>
<head>
    <title>Custom HTML Page</title>
</head>
<body>
    <h1>Heloo CKA Aspirants</h1>
    <p>This is a custom HTML page served by Nginx.</p>
</body>
</html>
```

Now, let's get the pod IP and curl it to see if it serves our custom HTML page (index.html) injected via the ConfigMap.

```bash
vagrant@controlplane:~$ POD_IP=$(kubectl get pods -l app=nginx -o jsonpath='{.items[0].status.podIP}')

# 
vagrant@controlplane:~$ echo http://$POD_IP
http://10.244.196.137

# 
vagrant@controlplane:~$ curl http://$POD_IP
<!DOCTYPE html>
<html>
<head>
    <title>Custom HTML Page</title>
</head>
<body>
    <h1>Heloo CKA Aspirants</h1>
    <p>This is a custom HTML page served by Nginx.</p>
</body>
</html>
```

### Mounting Single File Using subPath

There are scenarios where you may only need to mount a single file from the ConfigMap data, rather than all the key-value pairs. In this case, you can use the **subPath**.

In the **volumeMounts** section, specify the subPath and the absolute mount path of the file. In the volumes section, define the file name and path under items that need to be mounted.

```bash
volumeMounts:
  - name: html-volume
    mountPath: /usr/share/nginx/html/index.html
    subPath: index.html
volumes:
- name: html-volume
  configMap:
    name: custom-index-html
    items:
    - key: index.html
      path: index.html
```

---

### Updating ConfigMaps

ConfigMaps can normally be changed after creation. But if you make a ConfigMap immutable, it cannot be edited. To update it, you must delete and recreate it.

Let's add some changes to the key-value pairs in the **custom-index-html** ConfigMap as shown below.

```bash
apiVersion: v1
kind: ConfigMap
metadata:
  name: custom-index-html
data:
  backend_api: "api.techiescamp.com/v1/apps"
  vault_url: "vault.techiescamp.com/secrets/webserver"
  index.html: |
    <!DOCTYPE html>
    <html>
    <head>
        <title>Modified HTML Page</title>
    </head>
    <body>
        <h1>Heloo CKA Aspirants! This is the updated ConfigMap</h1>
        <p>This is the modified HTML page served by Nginx.</p>
    </body>
    </html>
```

In the above YAML, we updated **backend_api** and **vault_url** with additional URL parameters.

```bash
## Existing Values
backend_api: "api.techiescamp.com"
vault_url: "vault.techiescamp.com/secrets" 

## Updated
backend_api: "api.techiescamp.com/v1/apps"
vault_url: "vault.techiescamp.com/secrets/webserver"

## We have also made changes to the index.html content in the title and H1 tags.
```

Now, let's update the configmap and see how the updates are reflected in the nginx pod.

```bash
vagrant@controlplane:~$ kubectl apply -f configmap.yaml
configmap/custom-index-html configured
```

> When you use a subPath for mounting ConfigMap volumes in Kubernetes, the ConfigMap will not get updated automatically in the Pod.

### Configmap Volume Updates

Let's check if the pod has updated the index.html file mounted via configmap volume.

```bash
#
vagrant@controlplane:~$ kubectl get pods -l app=nginx
NAME                                READY   STATUS    RESTARTS        AGE
nginx-deployment-7c978dc764-zvknl   1/1     Running   1 (4m34s ago)   30h

# 
vagrant@controlplane:~$ kubectl exec -it nginx-deployment-7c978dc764-zvknl -- cat /usr/share/nginx/html/index.html
<!DOCTYPE html>
<html>
<head>
    <title>Modified HTML Page</title>
</head>
<body>
    <h1>Heloo CKA Aspirants! This is the updated ConfigMap</h1>
    <p>This is a modified HTML page served by Nginx.</p>
</body>
</html>
```

Let's check if the pod has updated the configmap environment variables since we updated to the new values.

```bash
vagrant@controlplane:~$ kubectl exec -it nginx-deployment-7c978dc764-zvknl -- printenv
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
HOSTNAME=nginx-deployment-7c978dc764-zvknl
NGINX_VERSION=1.31.3
BACKEND_API=api.techiescamp.com   # not updated
VAULT_URL=vault.techiescamp.com/secrets   # not updated
KUBERNETES_PORT_443_TCP_PORT=443
CUSTOM_SRE_WEB_SERVICE_HOST=10.111.98.178
CUSTOM_SRE_WEB_SERVICE_PORT=80
```

You can see that the environment variables are not updated even though the ConfigMap values have been changed. This happens because the **environment variables** in the pod are **immutable**.

```bash
# Delete the deployment.
vagrant@controlplane:~$ kubectl delete deploy nginx-deployment
deployment.apps "nginx-deployment" deleted from default namespace

# delete configmap
vagrant@controlplane:~$ kubectl delete cm custom-index-html
configmap "custom-index-html" deleted from default namespace
```

---

### ConfigMap Update Delays

The **Kubelet** periodically synchronizes the ConfigMap data with the pod's filesystem based on the **syncFrequency**. This ensures that the **ConfigMap** updates are reflected in the pod over time.

During each sync cycle (default: **every 1 minute**), the Kubelet checks its local cache to determine the current state of the ConfigMap.

---

### Real-world Scenario

How can we automate the pod rollouts during configmap changes?

- By default, K8s does not restart Pods when a ConfigMap or Secret changes.
- This can cause applications to keep running with outdated configs (e.g., old API URLs, expired credentials).

Reloader is a K8s controller that watches for changes in ConfigMaps and Secrets. When it detects an update, it automatically triggers a rollout of workloads such as Deployments, StatefulSets, or DaemonSets that reference those resources.

> **Reloader** keeps your workloads automatically updated whenever ConfigMaps or Secrets change, eliminating stale configurations.

### Hot Reloading

**Hot reloading** is a feature in software development that allows applications to update their configuration or code in real-time without needing to restart the entire application. This speeds up the development process, as developers can see the configuration changes immediately.

> Hot reloading is a development feature that lets you update an application’s code or configuration while it’s running, without restarting the whole process.

---

### Immutable Configmap

In production environments where application stability is critical, configuration changes should only occur through a controlled deployment process (e.g., CI/CD pipeline).

To create an immutable ConfigMap, simply add the **immutable: true** parameter to the ConfigMap object.

```bash
# create an immutable configmap named api-endpoints.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: api-endpoints
data:
  user-service: "https://api.example.com/user"
  order-service: "https://api.example.com/order"
  product-service: "https://api.example.com/product"
immutable: true

# create the configmap.
vagrant@controlplane:~$ kubectl apply -f api-endpoints.yaml
configmap/api-endpoints created

# verify configmap status
vagrant@controlplane:~$ kubectl get cm
NAME                            DATA   AGE
api-endpoints                   3      100s

# If you change any data in the ConfigMap and try to update it, you will get the forbidden error.
vagrant@controlplane:~$ kubectl apply -f api-endpoints.yaml
The ConfigMap "api-endpoints" is invalid: data: Forbidden: field is immutable when `immutable` is set

# Delete the immutable configmap
vagrant@controlplane:~$ kubectl delete cm api-endpoints
configmap "api-endpoints" deleted from default namespace
```

---

### ConfigMap Imperative Commands

For the CKA exam, you need to use **imperative commands** to create and modify Configmaps.

```bash
# Create Configmap
vagrant@controlplane:~$ kubectl create cm myconfigmap
configmap/myconfigmap created

# edit the configmap and add values
vagrant@controlplane:~$ kubectl edit cm myconfigmap

#
apiVersion: v1
kind: ConfigMap
metadata:
  creationTimestamp: "2026-07-25T16:34:10Z"
  name: myconfigmap
  namespace: default
  resourceVersion: "1265535"
  uid: 8f66fb4a-9ab0-4128-ae7d-c3a46bfff981

# use the --dry-run flag to create the configmap YAML
vagrant@controlplane:~$ kubectl create cm myconfig --dry-run=client -o yaml > mycm.yaml

# create a configmap named api-config with a key-value pair named api=api.techiescamp.com
vagrant@controlplane:~$ kubectl create cm api-config --from-literal=api=api.techiescamp.com
configmap/api-config created

# for more than one key-value pair, use the flag multiple times
vagrant@controlplane:~$ kubectl create cm webserver-config --from-literal=api=api.techiescamp.com --from-literal=secrets=vault.techiescamp.com --from-literal=discovery=consul.techiescamp.com
configmap/webserver-config created
```

### Create Configmap From File

You can place all the key-value pairs in a file and use the **--from-file** option to create the ConfigMap.

```bash
# Create a file named app.properties
database_url=jdbc:mysql://db.techiescamp.com:3306/mydatabase
database_user=dbuser
database_password=secretpassword
redis_host=redis.techiescamp.com
redis_port=6379
api_key=abcdef1234567890
log_level=INFO
max_connections=100
min_connections=10
timeout=5000

# create a ConfigMap named app-config from the app.properties file
vagrant@controlplane:~$ kubectl create cm app-config --from-file=app.properties
configmap/app-config created
```

When you use the **--from-file** option, the entire content of the file is used as the value, and the file name becomes the key.

```bash
# describe the app-config configmap and verify the key and value.
vagrant@controlplane:~$ kubectl describe cm app-config
Name:         app-config
Namespace:    default
Labels:       <none>
Annotations:  <none>

Data
====
app.properties:
----
database_url=jdbc:mysql://db.techiescamp.com:3306/mydatabase
database_user=dbuser
database_password=secretpassword
redis_host=redis.techiescamp.com
redis_port=6379
api_key=abcdef1234567890
log_level=INFO
max_connections=100
min_connections=10
timeout=5000

BinaryData
====

Events:  <none>
```

To create a configmap from more than one file, you can pass the **--from-file** parameter multiple times.

```bash
kubectl create cm app-config --from-file=app.properties --from-file=nginx.conf --from-file=server.conf
```

### Define Custom key

If you want to use a custom key name instead of the file name, you can specify the key name with the --from-file parameter.

```bash
# if you want the key name to be app.configs
kubectl create cm app-config --from-file=app.configs=app.properties
```

### Create Configmap From Env File

If you want each line of the file to be a separate key-value pair in the ConfigMap, you can use the **--from-env-file** parameter. This requires you to place all the key-value pairs in a file.

```bash
# Create a file named backend.properties
database_url=jdbc:mysql://db.techiescamp.com:3306/mydatabase
database_user=dbuser
database_password=secretpassword
redis_host=redis.example.com
redis_port=6379

# create the configmap named backend-config using --from-env-file.
vagrant@controlplane:~$ kubectl create cm backend-config --from-env-file=backend.properties
configmap/backend-config created

# describe the configmap, you can see each line from the file is added as a separate key-value pair in the configmap.
vagrant@controlplane:~$ kubectl describe cm backend-config
Name:         backend-config
Namespace:    default
Labels:       <none>
Annotations:  <none>

Data
====
database_password:
----
secretpassword

database_url:
----
jdbc:mysql://db.techiescamp.com:3306/mydatabase

database_user:
----
dbuser

redis_host:
----
redis.example.com

redis_port:
----
6379


BinaryData
====

Events:  <none>
```

You can also view the output in YAML format.

```bash
vagrant@controlplane:~$ kubectl get cm backend-config -o yaml
apiVersion: v1
data:
  database_password: secretpassword
  database_url: jdbc:mysql://db.techiescamp.com:3306/mydatabase
  database_user: dbuser
  redis_host: redis.example.com
  redis_port: "6379"
kind: ConfigMap
metadata:
  creationTimestamp: "2026-07-25T17:16:31Z"
  name: backend-config
  namespace: default
  resourceVersion: "1268661"
  uid: cd8ae963-b8f5-47c8-9ee8-83129fb99e32
```
