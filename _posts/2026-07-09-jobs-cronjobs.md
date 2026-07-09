---
layout: post
title: "Jobs & CronJobs"
date: 2026-07-09
categories: [k8s]
---

## Introduction

- Job = one-off task until completion
- CronJob = scheduled Job that repeats at defined times.

### Job

Job object deploys a pod and runs tasks to completion and then stops.

> Jobs run until the tasks specified in the job are completed and if the pods give an exit code 0, the Job is considered finished.

The task could be a shell script execution, an API call, or a Java Python execution that does a data transformation and uploads it to cloud storage.

![Job]({{ site.baseurl }}/assets/img/k8s-course/job.gif)

### Cronjobs

What if you want to run a batch job on specific schedules, for example, every 2 hours? You can create a K8s **CronJob** with a **cron expression**.

> **schedule: "0,15,30,45 * * * *"**

![CronJob]({{ site.baseurl }}/assets/img/k8s-course/cronjob.png)

### Creating a Job

Lets create a **job.yaml** using imperative command.

```bash
vagrant@controlplane:~$ kubectl create job my-job --image=busybox --dry-run=client -o yaml -- echo "Hello from the Zacademy.org" > job.yaml
```

The output YAML should look like the following.

```bash
vagrant@controlplane:~$ cat job.yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: my-job
spec:
  template:
    metadata: {}
    spec:
      containers:
      - command:
        - echo
        - Hello from the Zacademy.org
        image: busybox
        name: my-job
        resources: {}
      restartPolicy: Never
status: {}
```

- Once the command echo Hello from the Zacademy.org is executed, the container will print the message and then exit.
- If the command executes successfully, the container will exit with a status code of 0, indicating success.

Now, lets apply the job manifest file.

```bash
vagrant@controlplane:~$ kubectl apply -f job.yaml
job.batch/my-job created

# check the status of the job.
vagrant@controlplane:~$ kubectl get job
NAME     STATUS     COMPLETIONS   DURATION   AGE
my-job   Complete   1/1           7s         33s

# check the status of the pod.
vagrant@controlplane:~$ kubectl get pods
NAME                                            READY   STATUS      RESTARTS      AGE
my-job-qthqz                                    0/1     Completed   0             38s

# Check the logs of the pod to see if the job completed its task correctly or not.
vagrant@controlplane:~$ kubectl logs my-job-qthqz
Hello from the Zacademy.org
```

When a job is deployed, you can make it run on multiple pods with parallelism. It is particularly useful for batch jobs.

If you want to run **6 pods** and have **2 pods** running in **parallel**, you need to add the following two parameters to your job manifest.

```bash
completions: 6
parallelism: 2
```

The Job will run 2 pods in parallel to achieve 6 completions. But the overall completion of the Job will happen in a sequence of batches ( 2 + 2 + 2).

![Job in Batch]({{ site.baseurl }}/assets/img/k8s-course/job-batch.jpg)

Lets, create the following job manifest file.

```bash
vagrant@controlplane:~$ kubectl create job parallel-job --image=busybox --dry-run=client -o yaml > job-parallel.yaml
```

Now, edit the **job-parallel.yaml** and add the **completions, parallelism, backoffLimit**, and custom command as shown below.

```bash
vagrant@controlplane:~$ cat job-parallel.yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: parallel-job
spec:
  completions: 6
  parallelism: 2
  backoffLimit: 2
  template:
    metadata:
      name: parallel-job
    spec:
      containers:
      - name: parallel-container
        image: busybox
        command: ["sh", "-c", "echo Hello from pod && sleep 5"]
      restartPolicy: OnFailure
```

Now, apply the job manifest file.

```bash
vagrant@controlplane:~$ kubectl apply -f job-parallel.yaml
job.batch/parallel-job created

# First batch
vagrant@controlplane:~$ kubectl get job
NAME           STATUS     COMPLETIONS   DURATION   AGE
my-job         Complete   1/1           7s         25m
parallel-job   Running    2/6           14s        14s

# Second batch
vagrant@controlplane:~$ kubectl get job
NAME           STATUS     COMPLETIONS   DURATION   AGE
my-job         Complete   1/1           7s         25m
parallel-job   Running    4/6           30s        30s

# Third batch
vagrant@controlplane:~$ kubectl get job
NAME           STATUS     COMPLETIONS   DURATION   AGE
my-job         Complete   1/1           7s         25m
parallel-job   Complete   6/6           33s        43s
```

You can see that 2 pods are running at the same time, and at **6/6** job completes.

Lets check the log for the pods.

```bash
vagrant@controlplane:~$ kubectl logs -l job-name=parallel-job --prefix
[pod/parallel-job-5pkmf/parallel-container] Hello from pod
[pod/parallel-job-7rtgr/parallel-container] Hello from pod
[pod/parallel-job-882kz/parallel-container] Hello from pod
[pod/parallel-job-dlzlj/parallel-container] Hello from pod
[pod/parallel-job-z867m/parallel-container] Hello from pod
[pod/parallel-job-5dlq2/parallel-container] Hello from pod
```

---

### Suspend a Job

You can **suspend** a running job or create a job in suspended mode using the **suspend: true** parameter.

```bash
vagrant@controlplane:~$ kubectl create job suspend-job --image=busybox --dry-run=client -o yaml -- sleep infinity > suspend-job.yaml
```

Here is the output YAML

```bash
vagrant@controlplane:~$ cat suspend-job.yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: suspend-job
spec:
  template:
    metadata: {}
    spec:
      containers:
      - command:
        - sleep
        - infinity
        image: busybox
        name: suspend-job
        resources: {}
      restartPolicy: Never
status: {}
```

Now, lets apply the job.

```bash
vagrant@controlplane:~$ kubectl apply -f suspend-job.yaml
job.batch/suspend-job created

# Check the pods, you will have a running pod. It will run indefinitely due to the sleep command.
vagrant@controlplane:~$ kubectl get pods
NAME                                            READY   STATUS    RESTARTS      AGE
suspend-job-d6fbs                               1/1     Running   0             29s
```

Now, let's patch this job with the **suspend: true** spec using the following command.

```bash
vagrant@controlplane:~$ kubectl patch job suspend-job -p '{"spec":{"suspend": true}}'
job.batch/suspend-job patched
```

When you suspend a job, the **job controller** will stop all currently running Pods and will continue to wait without any time limit until the **flag** is switched from **true** to **false**.

```bash
vagrant@controlplane:~$ kubectl get pods
NAME                                            READY   STATUS        RESTARTS      AGE
suspend-job-d6fbs                               1/1     Terminating   0             2m49s
```

Now, let's patch the job with **suspend: false** and see what happens.

```bash
vagrant@controlplane:~$ kubectl patch job suspend-job -p '{"spec":{"suspend":false}}'
job.batch/suspend-job patched

# Verify pod status
vagrant@controlplane:~$ kubectl get pods
NAME                                            READY   STATUS    RESTARTS      AGE
suspend-job-r9flr                               1/1     Running   0             20s
```

If you **describe** the job you will see the **suspend** and **resume** events under **events**.

```bash
vagrant@controlplane:~$ kubectl describe job suspend-job | grep Events: -A10
Events:
  Type    Reason            Age    From            Message
  ----    ------            ----   ----            -------
  Normal  SuccessfulCreate  6m2s   job-controller  Created pod: suspend-job-d6fbs
  Normal  SuccessfulDelete  3m34s  job-controller  Deleted pod: suspend-job-d6fbs
  Normal  Suspended         3m34s  job-controller  Job suspended
  Normal  SuccessfulCreate  96s    job-controller  Created pod: suspend-job-r9flr
  Normal  Resumed           96s    job-controller  Job resumed
  ```

---

### Creating CronJobs

Create a **cronjob.yaml** using the imperative command. 

```bash
vagrant@controlplane:~$ kubectl create cj my-cronjob --schedule="*/2 * * * *" --image=busybox --dry-run=client -o yaml -- /bin/sh -c 'echo "Hello From Zacademy.org"' > cronjob.yaml
```

The output YAML should look like the following. It will run every two minutes. Each job will print "Hello from Zacademy.org"

```bash
vagrant@controlplane:~$ cat cronjob.yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: my-cronjob
spec:
  jobTemplate:
    metadata:
      name: my-cronjob
    spec:
      template:
        metadata: {}
        spec:
          containers:
          - command:
            - /bin/sh
            - -c
            - echo "Hello From Zacademy.org"
            image: busybox
            name: my-cronjob
            resources: {}
          restartPolicy: OnFailure
  schedule: '*/2 * * * *'
status: {}
```

Deploy this cronjob using kubectl apply command.

```bash
vagrant@controlplane:~$ kubectl apply -f cronjob.yaml
cronjob.batch/my-cronjob created

# Now let's check the cronjob status.
vagrant@controlplane:~$ kubectl get cronjob
NAME         SCHEDULE      TIMEZONE   SUSPEND   ACTIVE   LAST SCHEDULE   AGE
my-cronjob   */2 * * * *   <none>     False     0        <none>          18s

# Now let's list the pods
vagrant@controlplane:~$ kubectl get pods
NAME                                            READY   STATUS      RESTARTS      AGE
my-cronjob-29726374-sxhb6                       0/1     Completed   0             22s
```

If you check **after 2 minutes**, another job will be executed.

```bash
vagrant@controlplane:~$ kubectl get pods
NAME                                            READY   STATUS      RESTARTS      AGE
my-cronjob-29726374-sxhb6                       0/1     Completed   0             2m7s
my-cronjob-29726376-4bl9s                       0/1     Completed   0             7s
```

Now, let's check the logs of one of the pods to see if it has executed successfully.

```bash
vagrant@controlplane:~$ kubectl logs my-cronjob-29726374-sxhb6
Hello From Zacademy.org
```

---

### Controlling Duration of a Job Run

**activeDeadlineSeconds** is a field used to specify a maximum lifetime for a Pod or Job. It's essentially a timeout value set in seconds.

```bash
spec:
   activeDeadlineSeconds: 15
```

Assume we have a **CronJob** that runs daily at **5 pm**. It deploys pods that run an **ETL Job**.

> **ETL** stands for Extract, Transform, Load, and it refers to a type of data integration process commonly used in data warehousing and data management.

The **ETL job** extracts data from a database, transforms it, and loads it into a data warehouse. This job typically runs smoothly and takes around 2 hours to complete. However, there might be unexpected situations that cause the job to hang.

- **Data source issue:** The database might be overloaded or experiencing technical problems, slowing down data extraction significantly.
- **Transformation error:** A bug in the transformation logic could lead to an infinite loop or processing bottleneck.

**Solution:** We can configure the **CronJob** with **activeDeadlineSeconds** set to a reasonable value, say 2.5 hours (slightly higher than the usual runtime).

- **Normal execution:** If the job finishes within 2 hours, everything functions normally.
- **Job Stalled:** If the **ETL job** gets stuck due to unforeseen issues exceeding the deadline (2.5 hours), K8s will automatically terminate the Pod(s) running the job.
- **Alerts for investigation:** The job failure due to the deadline being reached triggers alerts using monitoring systems.

---

### Cronjob Real World Example

Imagine there is an e-commerce website that receives hundreds of files daily with updates on products, like price changes or stock levels. These files are stored in cloud storage. (like Google Drive or Amazon S3).

To keep the website updated with the latest product information, the files stored in cloud storage must be processed. Processing involves reading the files, interpreting the updates, and then adding these updates to the website's database, where all product information is kept.

![CronJob Example]({{ site.baseurl }}/assets/img/k8s-course/cronjob-example.gif)

- **CronJob:** A scheduled task is configured to run automatically at a specific time, such as 8 AM daily, to process these updates during off-peak hours.
- **Deployment and Initial Setup:** When the job deploys the pod, an init container first retrieves the database connection secrets from a secret management system. These secrets are then saved onto a volume, ensuring secure access to the database.
- **File Processing:** The main container then takes over, fetching the files from cloud storage and saving them onto the volume. It processes these files to extract the necessary updates for the products.
- **Database Update:** Using the database password obtained by the init container, the main container connects to the database. It then updates the database with the new product information extracted from the files.
- **Completion and Status Check:** After all the files have been processed, the script concludes its execution. If the process was successful and all files were correctly processed, the script exits with a 0 exit code, indicating a successful job completion. If there were any issues, it exits with a non-zero exit code, signaling that the job was not completed as expected.

---

### Scenario 01: Create a Job

Creat a K8s Job named **zacademy-job** that uses the image **alpine:latest**.

The job's primary function is to **print** the **date** using date command. Then forward the logs to **/opt/job.log** file.

```bash
vagrant@controlplane:~$ kubectl create job zacademy-job --image=alpine:latest \--dry-run=client -o yaml -- /bin/sh -c "d
ate" > date-job.yaml

# Apply the yaml file
vagrant@controlplane:~$ kubectl apply -f date-job.yaml
job.batch/zacademy-job created

# Verify the yaml and deploy the job
vagrant@controlplane:~$ kubectl get jobs
NAME           STATUS     COMPLETIONS   DURATION   AGE
zacademy-job   Complete   1/1           12s        18s

# Verify pod
vagrant@controlplane:~$ kubectl get pods
NAME                                            READY   STATUS      RESTARTS      AGE
zacademy-job-8hmpx                              0/1     Completed   0             50s

# Verify logs
vagrant@controlplane:~$ kubectl logs zacademy-job-8hmpx
Thu Jul  9 09:18:08 UTC 2026

# Forward the logs
vagrant@controlplane:~$ kubectl logs zacademy-job-8hmpx > /opt/job.log
```

### Scenario 02: Run Parallel Jobs

Create a Job named **parallel-job** using the image busybox that should print "**Data Processed**" and sleep for 30 seconds. This job should run a maximum of 6 times and only 3 jobs can run parallel.

```bash
vagrant@controlplane:~$ kubectl create job parallel-job --image=busybox --dry-run=client -o yaml -- /bin/sh -c 'echo "Data Processed"; sleep 30;' > parallel-job.yaml
```

Now edit the **parallel-job.yaml** and completions: 6 and parallelism: 2 parameters under spec.

```bash
apiVersion: batch/v1
kind: Job
metadata:
  name: parallel-job
spec:
  completions: 6
  parallelism: 3
  template:
    metadata: {}
    spec:
      containers:
      - command:
        - /bin/sh
        - -c
        - echo "Data Processed"; sleep 30;
        image: busybox
        name: parallel-job
        resources: {}
      restartPolicy: Never
status: {}
```

Now apply this manifest:

```bash
vagrant@controlplane:~$ kubectl apply -f parallel-job.yaml
job.batch/parallel-job created

# Verify job
vagrant@controlplane:~$ kubectl get jobs
NAME           STATUS     COMPLETIONS   DURATION   AGE
parallel-job   Running    0/6           20s        20s

# First batch of pods are running
vagrant@controlplane:~$ kubectl get pods
NAME                                            READY   STATUS      RESTARTS      AGE
parallel-job-kbksr                              1/1     Running     0             36s
parallel-job-md822                              1/1     Running     0             36s
parallel-job-s67bt                              1/1     Running     0             36s

# First batch completed and second batch of pods are running
vagrant@controlplane:~$ kubectl get pods
NAME                                            READY   STATUS      RESTARTS      AGE
parallel-job-2g8qk                              1/1     Running     0             10s
parallel-job-btxts                              1/1     Running     0             13s
parallel-job-fj2jj                              1/1     Running     0             15s
parallel-job-kbksr                              0/1     Completed   0             59s
parallel-job-md822                              0/1     Completed   0             59s
parallel-job-s67bt                              0/1     Completed   0             59s

# Job is completed
vagrant@controlplane:~$ kubectl get pods
NAME                                            READY   STATUS      RESTARTS      AGE
parallel-job-2g8qk                              0/1     Completed   0             79s
parallel-job-btxts                              0/1     Completed   0             82s
parallel-job-fj2jj                              0/1     Completed   0             84s
parallel-job-kbksr                              0/1     Completed   0             2m8s
parallel-job-md822                              0/1     Completed   0             2m8s
parallel-job-s67bt                              0/1     Completed   0             2m8s

# Verify job by viewing logs
vagrant@controlplane:~$ kubectl logs parallel-job-2g8qk
Data Processed
```

### Scenario 03: Create a CronJob

Create a CronJob named **zacademy-cj** using the image **busybox**. This **cronjob** should run **every 2 minutes** and print "**Welcome to the CKA course**". Each job should run for **15 seconds**. If a job exceeds 15 seconds, it should be terminated.

```bash
vagrant@controlplane:~$ kubectl create cj zacademy-cj --schedule="*/2 * * * *" --image=busybox --dry-run=client -o yaml -- /bin/sh -c 'echo Welcome to the CKA course' > batch-job.yaml

vagrant@controlplane:~$ cat batch-job.yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: zacademy-cj
spec:
  jobTemplate:
    metadata:
      name: zacademy-cj
    spec:
      activeDeadlineSeconds: 15
      template:
        metadata:
         creationTimestamp: null
        spec:
          containers:
          - command:
            - /bin/sh
            - -c
            - echo Welcome to the CKA course
            image: busybox
            name: zacademy-cj
            resources: {}
          restartPolicy: OnFailure
  schedule: '*/2 * * * *'
status: {}

# Apply the cronjob manifest
vagrant@controlplane:~$ kubectl apply -f batch-job.yaml
cronjob.batch/zacademy-cj created

# Verify CronJob
vagrant@controlplane:~$ kubectl get cj
NAME          SCHEDULE      TIMEZONE   SUSPEND   ACTIVE   LAST SCHEDULE   AGE
zacademy-cj   */2 * * * *   <none>     False     0        <none>          63s

# Verify pod
vagrant@controlplane:~$ kubectl get pods
NAME                                            READY   STATUS      RESTARTS      AGE
zacademy-cj-29726618-sd6zc                      0/1     Completed   0             15s
```