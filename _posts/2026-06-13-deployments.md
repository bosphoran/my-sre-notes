---
layout: post
title: "Deployments"
date: 2026-06-13
categories: [k8s]
---

## Introduction

A **Deployment** runs and manages applications in K8s — it handles scaling, upgrades, and rollbacks automatically, while **ReplicaSets** just ensure Pod count.

```bash
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app-deployment
  labels:
    app: frontend
spec:
  replicas: 3
  selector:
    matchLabels:
      app: frontend
  template:
    metadata:
      labels:
        app: frontend
    spec:
      containers:
      - name: nginx-container
        image: nginx:1.25.4
```

### How Does Deployment Work?

The **Deployment** object doesn't manage Pods directly. Instead, it uses **ReplicaSets** to create and manage Pods, ensuring the desired state is reached.

> Deployments use **labels** to identify and manage Pods (through ReplicaSets).

![Deployment]({{ site.baseurl }}/assets/img/k8s-course/deployment.jpg)

> The **selector** in the **spec** uses **matchLabels** so the **Deployment** can find the right **ReplicaSets** for the Pods it should run.

Each pod within a deployment has a unique label named **pod-template-hash** with a unique hash appended to it. It is added by the deployment controller during its creation.

So even if multiple pods use the same general labels (like app: myapp), the **pod-template-hash** keeps them clearly grouped by version.

![Deployment 2]({{ site.baseurl }}/assets/img/k8s-course/deployment-2.png)

### Updating & Scaling Deployment

K8s **Deployments** use a **rolling update strategy** by default.

> A rolling update strategy means updating a Deployment gradually, replacing old Pods with new ones in batches so the application stays available and downtime is minimized.

A K8s **Deployment update** is like downloading a new version of the app — the system installs it gradually in the background so you can keep using it without interruption, and if something goes wrong, it can roll back to the previous version.

```bash
kubectl set image deployment/web-app-deployment nginx=nginx:1.16.1
```

The imperative command above does not change the yaml file. If you want the changes to be permanent even after Deployment is reapplied, recreated, or restarted, then, you have to edit the yaml file too.

For **scaling** a Deployment, you adjust the **replicas** field in the Deployment **spec**.

```bash
kubectl scale deployment/web-app-deployment --replicas=5
# To make the change permanet, yaml file must be updated too.
```

> For automatic scaling, we need to use a **Horizontal Pod Autoscaler** (HPA) or **Vertical Pod Autoscaler** (VPA). 

---

### Create Deployment Using Imperative Commands

Let's create a deployment named **webserver** using nginx image with **3 replicas**.

```bash
# Create deployment with nginx image and 3 replicas
vagrant@controlplane:~$ kubectl create deployment webserver-deploy --image=nginx --replicas=3
deployment.apps/webserver-deploy created
```

With the above command, the Deployment definition is stored in the cluster’s API server, not in a file.

If you want to keep it as a file for re‑use, you can redirect the output:

```bash
vagrant@controlplane:~$ kubectl get deployment webserver-deploy -o yaml > webserver-deploy.yaml
```

Now, lets verify the deployment we have just created.

```bash
# Verify deployment
vagrant@controlplane:~$ kubectl get deployment
NAME               READY   UP-TO-DATE   AVAILABLE   AGE
webserver-deploy   3/3     3            3           12s

# Verify replicaset
vagrant@controlplane:~$ kubectl get replicaset
NAME                         DESIRED   CURRENT   READY   AGE
webserver-deploy-58ddb6bdc   3         3         3       18s

# The deployment created a ReplicaSet, which in turn created 3 pods.
vagrant@controlplane:~$ kubectl get pods
NAME                               READY   STATUS    RESTARTS   AGE
webserver-deploy-58ddb6bdc-67jvp   1/1     Running   0          23s
webserver-deploy-58ddb6bdc-jwdqn   1/1     Running   0          23s
webserver-deploy-58ddb6bdc-wjhcr   1/1     Running   0          23s
```

You get the deployment yaml file with the following command. It will output the whole Deployment yaml file with all the default fields and their values.

```bash
vagrant@controlplane:~$ kubectl get deployment webserver-deploy -o yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  annotations:
    deployment.kubernetes.io/revision: "1" # Every time you update the Deployment, K8s increments this revision number.
  creationTimestamp: "2026-06-12T22:50:15Z"
  generation: 1
  labels:
    app: webserver-deploy
  name: webserver-deploy
  namespace: default
  resourceVersion: "376266"
  uid: e3d888a6-4664-4f07-8c9a-c9a943c5d80f
spec:
  progressDeadlineSeconds: 600 # Maximum time (in seconds) K8s will wait for the Deployment to successfully roll out before marking it as failed
  replicas: 3
  revisionHistoryLimit: 10 # Number of old ReplicaSets for this Deployment that K8s will keep; 
                           # Useful for rollbacks, since you can revert to one of the last 10 versions if needed.
  selector:
    matchLabels:
      app: webserver-deploy # Defines which Pods belong to this Deployment (must match template labels)
  strategy:
    rollingUpdate: # Controls how updates are applied to Pods
      maxSurge: 25% # Allows up to 25% extra Pods temporarily during an update
      maxUnavailable: 25% # Allows up to 25% of Pods to be unavailable during an update
    type: RollingUpdate
  template:
    metadata:
      labels:
        app: webserver-deploy # Labels applied to Pods; must match selector above
    spec:
      containers:
      - image: nginx
        imagePullPolicy: Always
        name: nginx
        resources: {}
        terminationMessagePath: /dev/termination-log
        terminationMessagePolicy: File
      dnsPolicy: ClusterFirst
      restartPolicy: Always
      schedulerName: default-scheduler
      securityContext: {}
      terminationGracePeriodSeconds: 30 # Time (in seconds) K8s gives a Pod to shut down gracefully before forcefully killing it; 
status:
  availableReplicas: 3
  conditions:
  - lastTransitionTime: "2026-06-12T22:50:23Z"
    lastUpdateTime: "2026-06-12T22:50:23Z"
    message: Deployment has minimum availability.
    reason: MinimumReplicasAvailable
    status: "True"
    type: Available
  - lastTransitionTime: "2026-06-12T22:50:15Z"
    lastUpdateTime: "2026-06-12T22:50:23Z"
    message: ReplicaSet "webserver-deploy-58ddb6bdc" has successfully progressed.
    reason: NewReplicaSetAvailable
    status: "True"
    type: Progressing
  observedGeneration: 1
  readyReplicas: 3
  replicas: 3
  updatedReplicas: 3
```

---

### Deployment Creation Using Declarative Method

We can create a **Deployment** manifest file using **kubectl create** command.

```bash
vagrant@controlplane:~$ kubectl create deployment web-deployment --image=nginx --dry-run=client -o yaml > deployment.yaml --replicas=4
```

K8s creates the following yaml file. If we do not sepcify the replicas, it will set it to 1.

```bash
vagrant@controlplane:~$ cat deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: web-deployment
  name: web-deployment
spec:
  replicas: 4
  selector:
    matchLabels:
      app: web-deployment
  strategy: {}
  template:
    metadata:
      labels:
        app: web-deployment
    spec:
      containers:
      - image: nginx
        name: nginx
        resources: {}
status: {}
```

Lets apply the **deployment.yaml** Deployment manifest file.

```bash
vagrant@controlplane:~$ kubectl apply -f deployment.yaml
deployment.apps/web-deployment created

# Verify Deployment
vagrant@controlplane:~$ kubectl get deployment
NAME               READY   UP-TO-DATE   AVAILABLE   AGE
web-deployment     4/4     4            4           17s
```

Lets list all our resource in the cluster in the default namespace.

```bash
vagrant@controlplane:~$ kubectl get all
NAME                                   READY   STATUS    RESTARTS   AGE
pod/web-deployment-6b7c96bccb-4vntq    1/1     Running   0          65s
pod/web-deployment-6b7c96bccb-n5rv4    1/1     Running   0          65s
pod/web-deployment-6b7c96bccb-pbmgd    1/1     Running   0          65s
pod/web-deployment-6b7c96bccb-xbnqz    1/1     Running   0          65s
pod/webserver-deploy-58ddb6bdc-67jvp   1/1     Running   0          34m
pod/webserver-deploy-58ddb6bdc-jwdqn   1/1     Running   0          34m
pod/webserver-deploy-58ddb6bdc-wjhcr   1/1     Running   0          34m

NAME                 TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
service/kubernetes   ClusterIP   10.96.0.1    <none>        443/TCP   9d

NAME                               READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/web-deployment     4/4     4            4           65s
deployment.apps/webserver-deploy   3/3     3            3           34m

NAME                                         DESIRED   CURRENT   READY   AGE
replicaset.apps/web-deployment-6b7c96bccb    4         4         4       65s
replicaset.apps/webserver-deploy-58ddb6bdc   3         3         3       34m
```

As you may notice, we have total **7** pods from **2** **Deployments**. Also, We have 2 **replicasets**.

> That **kubernetes** Service is a **built‑in system service** for **API** access. It is automatically created when the cluster is initialized. All Pods inside the cluster use it to talk to the API server. 
> The IP **10.96.0.1** is the default service **CIDR** address reserved for the API server endpoint.

---

### Rolling Update and Rollbacks

A K8s **deployment rollout** involves updating your application pods gradually while minimizing downtime and ensuring stability.

When you update the deployment specification with a new container image, environment variables, etc, the rollout gets initiated.

**Deployment Controller** is responsible for orchestrating the entire process.

- **Analyze Updates:** Continuously monitors for changes in the deployment specification, like updates to the pod template.
- **Triggers Rollout:** Upon detecting a change, it initiates the rollout process by creating a new replica set with the updated configuration.
- **Manages Replicas:** Gradually **scales up** the new set and **scales down** the old set during the rollout.
- **Performs Health Checks:** Triggers health checks and readiness probes defined in the deployment.
- **Monitors Status:** Continually monitors the rollout progress, tracking the health and status of all pods involved.
- **Rolls Back if Needed:** Keeps track of old replica sets. It allows for **rollbacks** to previous versions if issues arise during the update.

![Deployment 3]({{ site.baseurl }}/assets/img/k8s-course/deployment-3.jpg)

---

### Update Strategies

1. **Rolling Update Strategy (Default):** This strategy is designed to minimize downtime during the update.

   - **New ReplicaSet:** A new replica set is created with the updated pod template.
   - **Gradual Scaling:** The new replica set gradually scales up, creating new pods with the updated configuration.
   - **Health Checks:** Each new pod undergoes health checks defined in the deployment.
   - **Old ReplicaSet Scaling Down:** As new pods become healthy, the old replica set with outdated pods is scaled down.
   - **Deletion (Optional):** By default, the old replica set is not immediately deleted after reaching zero replicas. You can configure its deletion after a certain grace period using the revisionHistoryLimit field in the deployment spec. By default, 10 old replicasets are retained. 

2. **Recreate Strategy:** This strategy abruptly replaces all pods with the updated version. It's faster but can lead to downtime.

   - **New ReplicaSet:** A new replica set is created with the updated pod template.
   - **Old ReplicaSet Deletion:** The existing replica set with outdated pods is immediately deleted.
   - **New Pods:** New pods with the updated configuration are created in the new replica set.
   - **No Downtime Guarantee:** While new pods are starting up, there will be a brief period of unavailability.

---

### Perform a Rolling Update and Rollback of a Deployment

Lets perform a **rolling update** of a Deployment and then **roll it back**.

First, create a deployment named **frontend-app** with the **nginx:1.24.0** image and 3 replicas. Then, check the status of the related **ReplicaSet** and pods that come under the deployment.

```bash
vagrant@controlplane:~$ kubectl create deployment frontend-app --image=nginx:1.24.0 --replicas=3
deployment.apps/frontend-app created

# Check the deployment status
vagrant@controlplane:~$ kubectl get deploy
NAME               READY   UP-TO-DATE   AVAILABLE   AGE
frontend-app       3/3     3            3           101s
web-deployment     4/4     4            4           19h
webserver-deploy   3/3     3            3           20h

# Check ReplicaSet status
vagrant@controlplane:~$ kubectl get replicaset
NAME                         DESIRED   CURRENT   READY   AGE
frontend-app-78d66c8757      3         3         3       2m24s
web-deployment-6b7c96bccb    4         4         4       19h
webserver-deploy-58ddb6bdc   3         3         3       20h

# Check pod status
vagrant@controlplane:~$ kubectl get pods
NAME                               READY   STATUS    RESTARTS   AGE
frontend-app-78d66c8757-llb7j      1/1     Running   0          3m5s
frontend-app-78d66c8757-mb2zg      1/1     Running   0          3m5s
frontend-app-78d66c8757-xv854      1/1     Running   0          3m5s
web-deployment-6b7c96bccb-4vntq    1/1     Running   0          19h
web-deployment-6b7c96bccb-n5rv4    1/1     Running   0          19h
web-deployment-6b7c96bccb-pbmgd    1/1     Running   0          19h
web-deployment-6b7c96bccb-xbnqz    1/1     Running   0          19h
webserver-deploy-58ddb6bdc-67jvp   1/1     Running   0          20h
webserver-deploy-58ddb6bdc-jwdqn   1/1     Running   0          20h
webserver-deploy-58ddb6bdc-wjhcr   1/1     Running   0          20h
```

If we check the **Deployment Strategy** we will see that the **rolling update** is configured:

```bash
vagrant@controlplane:~$ kubectl describe deployment frontend-app | grep -n "Strategy"
8:StrategyType:           RollingUpdate
10:RollingUpdateStrategy:  25% max unavailable, 25% max surge
```

There are some other parameters that can be configured in the **RollingUpdate strategy** type.

- **minReadySeconds**: This parameter defines a minimum waiting time (in seconds) for the K8s scheduler after a new pod is created. 
- **maxSurge**: Defines how many extra Pods (above the desired replica count) K8s is allowed to create temporarily during a rolling update, ensuring new Pods can come online before old ones are terminated.
  - If a Deployment wants 5 Pods and maxSurge is set to 1 (or 20%), K8s can temporarily run 6 Pods during the update — one extra Pod beyond the desired count.
- **maxUnavailable**: It specifies the percentage or maximum number of unavailable pods during the update.
  - For instance, with maxUnavailable set to 1 (or 20%), the deployment with 5 desired pods can have a maximum of 1 pod unavailable at any given time during the update process.

Now lets update our deployment and set the image to **nginx:1.25.4**.

```bash
vagrant@controlplane:~$ kubectl edit deployment frontend-app
deployment.apps/frontend-app edited
```

After updating, you'll see that the older pods will get terminated in a rollout manner once the new pods are created.

```bash
vagrant@controlplane:~$ kubectl get pods
NAME                               READY   STATUS              RESTARTS   AGE
frontend-app-78d66c8757-llb7j      1/1     Running             0          34m
frontend-app-78d66c8757-mb2zg      1/1     Running             0          34m
frontend-app-78d66c8757-xv854      1/1     Running             0          34m
frontend-app-85694c658d-4lb4d      0/1     ContainerCreating   0          4s

# After update has been completed
vagrant@controlplane:~$ kubectl get pods
NAME                               READY   STATUS    RESTARTS   AGE
frontend-app-85694c658d-4lb4d      1/1     Running   0          2m7s
frontend-app-85694c658d-7tvv7      1/1     Running   0          80s
frontend-app-85694c658d-vqdhg      1/1     Running   0          23s
```

> As you have noticed, during the rollout, we see an extra pod, which is the new pod being created. Once, the new pod is created, an old pod is removed.

Lets verify the new image in the pod:

```bash
vagrant@controlplane:~$ kubectl describe pod frontend-app-85694c658d-4lb4d | grep "image"
  Normal  Pulling    6m33s  kubelet            Pulling image "nginx:1.25.0"
  Normal  Pulled     5m51s  kubelet            Successfully pulled image "nginx:1.25.0" in 42.267s (42.267s including waiting). Image size: 57205272 bytes.
```

---

### Rollout Status, Record, and History

To understand **rollout status**, **record**, and **history** functionalities, let's update our deployment image to the **latest** Nginx image.

> A **rollout** in K8s means the process of gradually updating Pods in a Deployment.

```bash
vagrant@controlplane:~$ kubectl set image deploy frontend-app nginx=nginx:latest --record
Flag --record has been deprecated, --record will be removed in the future
deployment.apps/frontend-app image updated

# Lets verify the image
vagrant@controlplane:~$ kubectl describe pod frontend-app-65f5ff48bf-4nbqh | grep Image
    Image:          nginx:latest
    Image ID:       docker.io/library/nginx@sha256:1df1a963b5b8a3315c81c9d3c6fbd3324479a16c5e266bc6642c080be636145f
```

- **What a Rollout Does:**
  - Creates a new **ReplicaSet** for the updated Pod template.
  - Gradually replaces old Pods with new ones, following the rollout strategy (RollingUpdate by default).
  - Ensures availability by respecting **maxSurge** (extra Pods allowed) and **maxUnavailable** (Pods that can be offline).
  - **Monitors health**: new Pods must pass readiness checks before old ones are terminated.
  - **Tracks history**: each rollout increments the Deployment’s revision, so you can roll back if needed.

We can check the status and history of the deployment using the following command:

```bash
vagrant@controlplane:~$ kubectl rollout status deploy frontend-app
deployment "frontend-app" successfully rolled out
```

K8s automatically maintains a history of all deployments.

```bash
vagrant@controlplane:~$ kubectl rollout history deploy frontend-app
deployment.apps/frontend-app
REVISION  CHANGE-CAUSE
1         <none>
2         <none>
3         <none>
4         kubectl set image deploy frontend-app nginx=nginx:latest --record=true
```

Here, we can see there are a total of 4 revisions. The second column refers to the cause of that change. 

> Since we used the --record flag, the **CHANGE-CAUSE** for the **third rollout** shows the command we used for the rollout.

We can also manually update or insert our custom message in the **CHANGE-CAUSE** of the latest rollout using the **annotate** command.

```bash
vagrant@controlplane:~$ kubectl annotate deploy frontend-app kubernetes.io/change-cause="Updated the deployment with latest Nginx image"
deployment.apps/frontend-app annotated

# Check the rollout history
vagrant@controlplane:~$ kubectl rollout history deploy frontend-app
deployment.apps/frontend-app
REVISION  CHANGE-CAUSE
1         <none>
2         <none>
3         <none>
4         Updated the deployment with latest Nginx image
```

> Besides set image, you can scale replicas, set env vars, adjust resources, change service accounts, or tweak secrets imperatively.


- **Rollout** → the overall process of applying changes to a Deployment. 
- **Rolling update** → one specific strategy for performing a rollout.

### Rollback the Deployment

To **rollback** the deployment we use the **rollout undo** command:

```bash
vagrant@controlplane:~$ kubectl rollout undo deploy frontend-app
deployment.apps/frontend-app rolled back
```

The deployment should have been **rolled back** from **nginx:latest** to the image version **nginx:1.26.3**.

```bash
vagrant@controlplane:~$ kubectl describe pod frontend-app-668978d548-tncs5 | grep Image
    Image:          nginx:1.26.3
    Image ID:       docker.io/library/nginx@sha256:41b194461e4bae16f9b25d68b0976ed4735b89ca625c89aad88e1c1c3b7e8860
```

If we need to **roll back** to a specific revision we can achieve this using the **--to-revision** flag.

```bash
vagrant@controlplane:~$ kubectl rollout undo deploy frontend-app --to-revision=1
deployment.apps/frontend-app rolled back

# Check pod status
vagrant@controlplane:~$ kubectl get pods
NAME                               READY   STATUS    RESTARTS   AGE
frontend-app-78d66c8757-2btnk      1/1     Running   0          20s
frontend-app-78d66c8757-dpvvs      1/1     Running   0          17s
frontend-app-78d66c8757-nsnzf      1/1     Running   0          22s

# Verify rollback: The deployment has been rolled back to the initial image nginx:1.24.0
vagrant@controlplane:~$ kubectl describe pod frontend-app-78d66c8757-2btnk | grep Image
    Image:          nginx:1.24.0
    Image ID:       docker.io/library/nginx@sha256:f6daac2445b0ce70e64d77442ccf62839f3f1b4c24bf6746a857eff014e798c8
```

---

### Pause & Resume Rollouts

By pausing a deployment, you can control the rollout process.

> Normally, any changes to a deployment trigger a new rollout. **Pausing** prevents this, allowing you to make multiple changes to the live deployment configuration without unintended rollouts. Once you're ready, simply resume the deployment to initiate the rollout with the updated configuration.

Let's pause the **frontend-app** deployment rollout.

```bash
vagrant@controlplane:~$ kubectl rollout pause deploy frontend-app
deployment.apps/frontend-app paused
```

Now, let's edit the deployment and change the image to **nginx:latest**.

```bash
vagrant@controlplane:~$ kubectl edit deploy frontend-app
deployment.apps/frontend-app edited

# Verify rollout pause
vagrant@controlplane:~$ kubectl describe pod frontend-app-78d66c8757-2btnk | grep Image
    Image:          nginx:1.24.0
    Image ID:       docker.io/library/nginx@sha256:f6daac2445b0ce70e64d77442ccf62839f3f1b4c24bf6746a857eff014e798c8
```

As you can see, the rollout did not take effect, since, we have **paused** the **rollout**.

Now let's **resume** the **rollout** and see if the changes are getting rolled out.

```bash
vagrant@controlplane:~$ kubectl rollout resume deploy frontend-app
deployment.apps/frontend-app resumed

# Check pod status
vagrant@controlplane:~$ kubectl get pods
NAME                               READY   STATUS    RESTARTS   AGE
frontend-app-65f5ff48bf-ccsj5      1/1     Running   0          5s
frontend-app-65f5ff48bf-vkwv8      1/1     Running   0          7s
frontend-app-65f5ff48bf-vkznq      1/1     Running   0          9s

# Verify rollout
vagrant@controlplane:~$ kubectl describe pod frontend-app-65f5ff48bf-ccsj5 | grep Image
    Image:          nginx:latest
    Image ID:       docker.io/library/nginx@sha256:1df1a963b5b8a3315c81c9d3c6fbd3324479a16c5e266bc6642c080be636145f
```

Lets delete the deployment **frontend-app**.

```bash
vagrant@controlplane:~$ kubectl delete deploy frontend-app
deployment.apps "frontend-app" deleted from default namespace
```

---

### Example Scenario 01: Create a Deployment

Create a Deployment named sre-camp-deploy. Use image httpd:2.4-alpine. The number of replicas should be 3. Make sure all the pods are running. Then, delete the deployment.

```bash
# Create deployment
vagrant@controlplane:~$ kubectl create deploy sre-camp-deploy --image=httpd:2.4-alpine --replicas=3
deployment.apps/sre-camp-deploy created

# Verify deployment
vagrant@controlplane:~$ kubectl get deploy
NAME              READY   UP-TO-DATE   AVAILABLE   AGE
sre-camp-deploy   3/3     3            3           44s

# Check pods
vagrant@controlplane:~$ kubectl get pods
NAME                               READY   STATUS    RESTARTS   AGE
sre-camp-deploy-86c5df9c8d-26tlh   1/1     Running   0          55s
sre-camp-deploy-86c5df9c8d-c699m   1/1     Running   0          55s
sre-camp-deploy-86c5df9c8d-cklqw   1/1     Running   0          55s

# Delete deployment
vagrant@controlplane:~$ kubectl delete deploy sre-camp-deploy
deployment.apps "sre-camp-deploy" deleted from default namespace
```

---

### Example Scenario 02: Edit a Deployment

Create a Deployment named **sre-deploy**. Use image **nginx**. The number of **replicas** should be **3**.

- Change the strategy type to **Recreate** and change the image to **nginx:1.19.9**. Make sure all the pods are running.
- Next, change the image to **nginx:latest**. Wait for the pods to start.
- Finally, **scale down** the replicas to **2**.

```bash
# Create sre-deploy deployment with nginx image and 3 replicas
vagrant@controlplane:~$ kubectl create deploy sre-deploy --image=nginx --replicas=3
deployment.apps/sre-deploy created

# Check deployment status
vagrant@controlplane:~$ kubectl get deploy
NAME         READY   UP-TO-DATE   AVAILABLE   AGE
sre-deploy   3/3     3            3           26s

# Check pods status
vagrant@controlplane:~$ kubectl get pods
NAME                          READY   STATUS    RESTARTS   AGE
sre-deploy-5c8b4c84d6-4zvbf   1/1     Running   0          39s
sre-deploy-5c8b4c84d6-b6ff7   1/1     Running   0          39s
sre-deploy-5c8b4c84d6-pkvj7   1/1     Running   0          39s

# Rollout deployment image to nginx:1.26.3
vagrant@controlplane:~$ kubectl edit deploy sre-deploy
deployment.apps/sre-deploy edited

# Check pods status after rollout
vagrant@controlplane:~$ kubectl get pods
NAME                          READY   STATUS    RESTARTS   AGE
sre-deploy-568f9748cb-hd7gm   1/1     Running   0          13s
sre-deploy-568f9748cb-hslrk   1/1     Running   0          13s
sre-deploy-568f9748cb-mgphj   1/1     Running   0          13s

# Verify image version after rollout
vagrant@controlplane:~$ kubectl describe pod sre-deploy-568f9748cb-hd7gm | grep Image
    Image:          nginx:1.26.3
    Image ID:       docker.io/library/nginx@sha256:41b194461e4bae16f9b25d68b0976ed4735b89ca625c89aad88e1c1c3b7e8860

# Update image back to nginx:latest
vagrant@controlplane:~$ kubectl edit deploy sre-deploy
deployment.apps/sre-deploy edited

# Verify rollout
vagrant@controlplane:~$ kubectl describe pod sre-deploy-7f85bd9477-cb4jg | grep Image
    Image:          nginx:latest
    Image ID:       docker.io/library/nginx@sha256:1df1a963b5b8a3315c81c9d3c6fbd3324479a16c5e266bc6642c080be636145f

# Scale down deployment to 2 replicas
vagrant@controlplane:~$ kubectl scale deploy sre-deploy --replicas=2
deployment.apps/sre-deploy scaled

# Verify scale down
vagrant@controlplane:~$ kubectl get pods
NAME                          READY   STATUS    RESTARTS   AGE
sre-deploy-7f85bd9477-cmtjm   1/1     Running   0          2m31s
sre-deploy-7f85bd9477-jbjsm   1/1     Running   0          2m31s
```

---

### Example Scenario 03: Update a Deployment using Rolling Update Strategy

**Task Instructions:**

- Create a Deployment named **sre-camp-app** with container image **nginx:1.18.0** and **3** replicas.
- Add container **port 80**.
- Put **max surge 1** and **max unavailable 2** in the strategy.
- **Verify Pods**: Check that all 3 pods are Running.
- **Update the App**: Update the deployment with the new image **nginx:1.19.10**.
- Verify that the new pods are running and they have the new image.
- Check the **rollout status** and **history**.
- **Rollback the App**: Undo the deployment to the previous **version 1.18.0**.
- Verify the pods’s status and image.
- Again check the **rollout history**.

```bash
vagrant@controlplane:~$ kubectl create deployment sre-app --image=nginx:1.18.0 --replicas=3
deployment.apps/sre-app created

# Apply the changes to the sre-app deployment
vagrant@controlplane:~$ kubectl edit deploy sre-app
deployment.apps/sre-app edited

# Check pod status
vagrant@controlplane:~$ kubectl get pods
NAME                       READY   STATUS    RESTARTS   AGE
sre-app-6b6898579f-gt7g7   1/1     Running   0          5m44s
sre-app-6b6898579f-ktgm8   1/1     Running   0          5m45s
sre-app-6b6898579f-scw5z   1/1     Running   0          5m43s

# Verify container Image
vagrant@controlplane:~$ kubectl describe pod sre-app-6b6898579f-gt7g7 | grep Image
    Image:          nginx:1.18.0
    Image ID:       docker.io/library/nginx@sha256:e90ac5331fe095cea01b121a3627174b2e33e06e83720e9a934c7b8ccc9c55a0

# Update container image to nginx:1.19.10
vagrant@controlplane:~$ kubectl edit deploy sre-app
deployment.apps/sre-app edited

# Check pod status after Image update
vagrant@controlplane:~$ kubectl get pods
NAME                       READY   STATUS    RESTARTS   AGE
sre-app-57bd67cf8c-l2xn6   1/1     Running   0          77s
sre-app-57bd67cf8c-pkc42   1/1     Running   0          78s
sre-app-57bd67cf8c-tlxxl   1/1     Running   0          77s

# Verify container image after update
vagrant@controlplane:~$ kubectl describe pod sre-app-57bd67cf8c-l2xn6 | grep Image
    Image:          nginx:1.19.10
    Image ID:       docker.io/library/nginx@sha256:df13abe416e37eb3db4722840dd479b00ba193ac6606e7902331dcea50f4f1f2

# Check the rollout status and history.
vagrant@controlplane:~$ kubectl rollout status deploy sre-app
deployment "sre-app" successfully rolled out

vagrant@controlplane:~$ kubectl rollout history deploy sre-app
deployment.apps/sre-app
REVISION  CHANGE-CAUSE
1         <none>
2         <none>
3         <none>

# Rollback the App to Image version 1.18.0
vagrant@controlplane:~$ kubectl rollout undo deploy sre-app
deployment.apps/sre-app rolled back

# Verify the pods’s status and image.
vagrant@controlplane:~$ kubectl get pods
NAME                       READY   STATUS    RESTARTS   AGE
sre-app-6b6898579f-l265j   1/1     Running   0          21s
sre-app-6b6898579f-l5qcf   1/1     Running   0          20s
sre-app-6b6898579f-m6jhd   1/1     Running   0          21s

vagrant@controlplane:~$ kubectl describe pod sre-app-6b6898579f-l265j | grep Image
    Image:          nginx:1.18.0
    Image ID:       docker.io/library/nginx@sha256:e90ac5331fe095cea01b121a3627174b2e33e06e83720e9a934c7b8ccc9c55a0

# Again check the rollout history.
vagrant@controlplane:~$ kubectl rollout history deploy sre-app
deployment.apps/sre-app
REVISION  CHANGE-CAUSE
1         <none>
3         <none>
4         <none>
```

---

### Example Scenario 04: Update a Deployment using Recreate Strategy

**Task Instructions:**

- Create a Deployment named sre-web with container image **nginx:1.18.0** and **3 replicas**.
- **Verify Pods**: Check that all 3 pods are Running.
- Update the deployment with the new image **nginx:1.19.9**. Do not edit the deployment directly, only set the new image name for the existing deployment. Also, record the cause of change.
- Verify that the new pods are running and they have the new image.
- Check the **rollout status** and **history**. Save the change reason to the **cause.txt** file.
- Change the strategy to **Recreate** and **rollback**.
- Again use the new image **nginx:1.24.0** and record the cause.
- Verify the pods's status and image.
- **Rollback** the deployment and check the pod status and image.

```bash
# Creat the deployment manifest file
vagrant@controlplane:~$ kubectl create deployment sre-web --image=nginx:1.18.0 --replicas=3 --dry-run=client -o yaml > sre-web.yaml

# Apply the deployment
vagrant@controlplane:~$ kubectl apply -f sre-web.yaml
deployment.apps/sre-web created

# Verify deployment
vagrant@controlplane:~$ kubectl get deploy
NAME      READY   UP-TO-DATE   AVAILABLE   AGE
sre-web   3/3     3            3           46s

# Check pod status
vagrant@controlplane:~$ kubectl get pods
NAME                      READY   STATUS    RESTARTS   AGE
sre-web-f87dcfd9c-gc6xx   1/1     Running   0          64s
sre-web-f87dcfd9c-tzn8v   1/1     Running   0          64s
sre-web-f87dcfd9c-vrqmd   1/1     Running   0          64s

# Set image to nginx:1.19.9
vagrant@controlplane:~$ kubectl set image deployment/sre-web nginx=nginx:1.19.9 --record
Flag --record has been deprecated, --record will be removed in the future
deployment.apps/sre-web image updated

# Add custom rollout message
vagrant@controlplane:~$ kubectl annotate deployment sre-web kubernetes.io/change-cause="Updated nginx:1.18.0 imge to 1.1
9.9"
deployment.apps/sre-web annotated

# Verify new image in the container
vagrant@controlplane:~$ kubectl describe pod sre-web-698cc5c985-5fvh5 | grep Image
    Image:          nginx:1.19.9
    Image ID:       docker.io/library/nginx@sha256:6b5f5eec0ac03442f3b186d552ce895dce2a54be6cb834358040404a242fd476

# Check the rollout status.
vagrant@controlplane:~$ kubectl rollout status deploy sre-web
deployment "sre-web" successfully rolled out

# Check rollout history
vagrant@controlplane:~$ kubectl rollout history deploy sre-web
deployment.apps/sre-web
REVISION  CHANGE-CAUSE
1         <none>
2         Updated nginx:1.18.0 image to 1.19.9

# Change the strategy to Recreate.
strategy:
  - type: Recreate

# Use the new image nginx:1.24.0 and record the cause.
vagrant@controlplane:~$ kubectl set image deployment/sre-web nginx=nginx:1.24.0 --record
Flag --record has been deprecated, --record will be removed in the future
deployment.apps/sre-web image updated

# Write a custom message to the rollout
vagrant@controlplane:~$ kubectl annotate deployment sre-web kubernetes.io/change-cause="Updated image from nginx:1.18.0
to 1.24.0"
deployment.apps/sre-web annotated

# Verify pod status
vagrant@controlplane:~$ kubectl get pods
NAME                      READY   STATUS    RESTARTS   AGE
sre-web-78c687848-45mhq   1/1     Running   0          65s
sre-web-78c687848-9s2lt   1/1     Running   0          61s
sre-web-78c687848-zj6zf   1/1     Running   0          69s

# Verify new container image
vagrant@controlplane:~$ kubectl describe pod sre-web-78c687848-45mhq | grep Image
    Image:          nginx:1.24.0
    Image ID:       docker.io/library/nginx@sha256:f6daac2445b0ce70e64d77442ccf62839f3f1b4c24bf6746a857eff014e798c8

# Check rollout history
vagrant@controlplane:~$ kubectl rollout history deployment sre-web
deployment.apps/sre-web
REVISION  CHANGE-CAUSE
1         <none>
2         Updated nginx:1.18.0 image to 1.19.9
3         Updated image from nginx:1.18.0 to 1.24.0

# Rollback the deployment and check the pod status and image.
vagrant@controlplane:~$ kubectl rollout undo deploy sre-web
deployment.apps/sre-web rolled back

# Check pod status and verify rollback
vagrant@controlplane:~$ kubectl get pods
NAME                       READY   STATUS    RESTARTS   AGE
sre-web-698cc5c985-2qlpb   1/1     Running   0          21s
sre-web-698cc5c985-dxhrw   1/1     Running   0          18s
sre-web-698cc5c985-gb58g   1/1     Running   0          13s

vagrant@controlplane:~$ kubectl describe pod sre-web-698cc5c985-2qlpb | grep Image
    Image:          nginx:1.19.9
    Image ID:       docker.io/library/nginx@sha256:6b5f5eec0ac03442f3b186d552ce895dce2a54be6cb834358040404a242fd476
```

---
