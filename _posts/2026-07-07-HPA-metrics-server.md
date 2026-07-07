---
layout: post
title: "HPA Metrics Server Examples"
date: 2026-07-07
categories: [k8s]
---

## Example Scenarios

Here are some HPA examples.

### Deploying a Microservice with HPA

A new microservice named **sre-web-app**, which experiences fluctuating CPU usage. To ensure optimal performance and resource utilization, you need to:

- Deploy a microservice named **sre-web-app** with the following specifications:
  - Image: **nginx**
  - Replicas: **1**
  - Resource Requests: **100m CPU**
  - Resource Limits: **200m CPU**
- Expose the deployment as a **ClusterIP** service:
  - Name: **sre-web-app-service**
  - Port: **80**
- Configure a Horizontal Pod Autoscaler (**HPA**) for **sre-web-app**:
  - Target 65% average CPU utilization
  - Allow scaling between 1 and 5 replicas
- Simulate high CPU load to trigger autoscaling. You can use **load-testing-job.yaml** file.
- Verify that the HPA scales the **sre-web-app** deployment.

Here is the **load-testing-job.yaml** manifest file.

```bash
apiVersion: batch/v1
kind: Job
metadata:
  name: load-generator-job
spec:
  completions: 5  # Number of load generators
  parallelism: 5  # Number of concurrent requests
  template:
    spec:
      containers:
      - name: load-generator
        image: busybox
        command: ["/bin/sh", "-c", "while true; do wget -q -O- http://sre-web-app-service > /dev/null; done"]
      restartPolicy: Never
```

```bash
# Generate the Deployment manifest.
vagrant@controlplane:~$ kubectl create deploy web-app --image=nginx --replicas=1 --dry-run=client -o yaml > sre-web-app-deployment.yaml

# Now modify the sre-web-app-deployment.yaml according to the requirement.
vagrant@controlplane:~$ nano sre-web-app-deployment.yaml

# The final deployment manifest file
# ----------------------------------
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: sre-web-app
  name: sre-web-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: sre-web-app
  strategy: {}
  template:
    metadata:
      labels:
        app: sre-web-app
    spec:
      containers:
      - image: nginx
        name: nginx
        resources:
         requests:
          cpu: "100m"
         limits:
          cpu: "200m"
# ----------------------------------

# Apply the deployment:
vagrant@controlplane:~$ kubectl apply -f sre-web-app-deployment.yaml
deployment.apps/sre-web-app created

# Expose the Deployment as a Service
vagrant@controlplane:~$ kubectl expose deploy sre-web-app --type=ClusterIP --port=80 --name=sre-web-app-service
service/sre-web-app-service exposed

# Verify the deployments and services.
vagrant@controlplane:~$ kubectl get deploy,svc
NAME                                            READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/sre-web-app                     1/1     1            1           108s

NAME                                    TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
service/sre-web-app-service             ClusterIP   10.108.239.50   <none>        80/TCP    47s

# Configure Horizontal Pod Autoscaler (HPA)
kubectl autoscale deploy sre-web-app --cpu=65% --min=1 --max=5 --name=sre-web-app-hpa --dry-run=client -o yaml > sre-web-hpa.yaml

# Apply HPA
vagrant@controlplane:~$ kubectl apply -f sre-web-hpa.yaml
horizontalpodautoscaler.autoscaling/sre-web-app-hpa created

# Check the HPA status
vagrant@controlplane:~$ kubectl get hpa
NAME              REFERENCE                TARGETS          MINPODS   MAXPODS   REPLICAS   AGE
sre-web-app-hpa   Deployment/sre-web-app   cpu: 0%/65%      1         5         1          22s

# Simulate High CPU Load using the load-testing-job.yaml file. Apply the manifest.
vagrant@controlplane:~$ kubectl apply -f load-testing-job.yaml
job.batch/load-generator-job created

# Monitor HPA Scaling
vagrant@controlplane:~$ kubectl get hpa
NAME              REFERENCE                TARGETS         MINPODS   MAXPODS   REPLICAS   AGE
sre-web-app-hpa   Deployment/sre-web-app   cpu: 117%/65%   1         5         1          3m12s

vagrant@controlplane:~$ kubectl get pods
NAME                                            READY   STATUS    RESTARTS        AGE
load-generator-job-4vvmp                        1/1     Running   0               87s
load-generator-job-6xpk9                        1/1     Running   0               87s
load-generator-job-8zd6h                        1/1     Running   0               87s
load-generator-job-mhzpz                        1/1     Running   0               87s
load-generator-job-rbzsc                        1/1     Running   0               87s

sre-web-app-5777c8c975-8fbdh                    1/1     Running   0               41s
sre-web-app-5777c8c975-qq7cz                    1/1     Running   0               9m41s

# Delete the job after the testing
vagrant@controlplane:~$ kubectl delete job load-generator-job
job.batch "load-generator-job" deleted from default namespace
```

---

### Deployment with Dual-Metric HPA

Here we deploy the **api-server** application. The app needs to **autoscale** based on both **CPU and memory utilization**.

- Create a deployment named api-server in the default namespace using the image **nginx:1.21**:
  - Start with 3 replicas
  - Expose port 80
  - Set resource requests to:
    - CPU: 200m
    - Memory: 256Mi
- Create a Horizontal Pod Autoscaler that:
  - Scales between 2 and 5 replicas
  - Scales when CPU utilization exceeds 50% or Memory utilization exceeds 60%

```bash
# Generate the Deployment manifest.
vagrant@controlplane:~$ kubectl create deploy api-server --image=nginx:1.21 --replicas=3 --dry-run=client -o yaml > api-server-deployment.yaml

# Modify the api-server-deployment.yaml according to the question.
# ----------------------------
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-server
  namespace: default
spec:
  replicas: 3
  selector:
    matchLabels:
      app: api-server
  template:
    metadata:
      labels:
        app: api-server
    spec:
      containers:
      - name: api-server
        image: nginx:1.21
        ports:
        - containerPort: 80
        resources:
          requests:
            cpu: "200m"
            memory: "256Mi"
# ----------------------------

# Apply the deployment:
vagrant@controlplane:~$ kubectl apply -f api-server-deployment.yaml
deployment.apps/api-server created

# Verify the deployments and pods.
vagrant@controlplane:~$ kubectl get deploy
NAME                            READY   UP-TO-DATE   AVAILABLE   AGE
api-server                      3/3     3            3           49s

vagrant@controlplane:~$ kubectl get pods
NAME                                            READY   STATUS    RESTARTS   AGE
api-server-6c58d4dc65-hrzfz                     1/1     Running   0          47s
api-server-6c58d4dc65-pvn7j                     1/1     Running   0          47s
api-server-6c58d4dc65-rrm85                     1/1     Running   0          47s

# Configure Horizontal Pod Autoscaler (HPA):
vagrant@controlplane:~$ kubectl autoscale deploy api-server --cpu=50% --min=2 --max=5 --dry-run=client -o yaml > api-server-hpa.yaml

# Now edit the api-server-hpa.yaml and add the memory part as well.
# -----------------------------
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: api-server
  namespace: default
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: api-server
  minReplicas: 2
  maxReplicas: 5
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 50
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 60
# -----------------------------

# Now apply the manifest
vagrant@controlplane:~$ kubectl apply -f api-server-hpa.yaml
horizontalpodautoscaler.autoscaling/api-server created

# Verify HPA
vagrant@controlplane:~$ kubectl get hpa api-server
NAME         REFERENCE               TARGETS                       MINPODS   MAXPODS   REPLICAS   AGE
api-server   Deployment/api-server   cpu: 0%/50%, memory: 0%/60%   2         5         3          66s
```

---

### Corner-Case Q&A

- **Question 1:** If my Deployment yaml says replicas: 1, but the HPA that targets the Deployment has minReplicas: 2. How many Pods will I actually see?
  - K8s first applies whatever the Deployment says (ie, one replica). 
  - Then the HPA controller notices that the current replica count (1) is below its minimum (2) and immediately modifies the Deployment to 2 replicas. Very quickly, the Deployment brings a second Pod online. So you will end up with 2 running Pods, not 1.

- **Question 2:** Now, if the deployment says replicas: 2 and HPA says minReplicas: 1 what happens?
  - When you apply the manifest, Kubernetes spins up the two Pods you asked for (replicas: 2). 
  - If your metrics are below the HPA target threshold, then eventually the HPA would scale down to 1 Pod. The scale-down doesn't happen immediately. It respects the scale-down stabilization window (default: 5 minutes).

- Question 3: What happens when the Deployment requests 2 replicas, the HPA’s minReplicas is 1, and the Pod Disruption Budget (PDB) is set to 2?
  - In this case, the cluster starts with 2 Pods because the Deployment specifies 2 replicas. 
  - When traffic is quiet, the HPA triggers a scale-down activity to reduce to 1 pod. 
  - Contrary to what might be expected, the HPA can successfully scale down the Deployment to 1 replica, even though the PDB specifies a minimum of 2 available pods.

- Question 4: What happens when two HPAs target the same deployment?
  - When you create two HPAs targeting the same deployment, it gets created. However, none of the HPAs will be active.
  - Because the HPA controller runs an ambiguous-selector check. When it sees more than one HPA selecting the same set of Pods, it turns both HPAs off.