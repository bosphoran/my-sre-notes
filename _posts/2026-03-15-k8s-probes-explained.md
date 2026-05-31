---
layout: post
title: "K8s Probes Explained"
date: 2026-03-15
categories: [k8s]
---

## Introduction

Probes are not "special pods" or separate containers; they are internal diagnostic tasks executed by the Kubelet (the agent on each node).

When you define a probe in your YAML, you are essentially giving the Kubelet a "health check script" and a schedule. The Kubelet then reaches into your container to perform that check.

As the developer/SRE of the application, you are responsible for providing the logic that the probes will check.

### How K8s Handles Probes?

The Kubelet performs these checks using three main handlers:

- **HTTPGet**: The Kubelet acts like a web browser and sends a request (e.g., GET /health) to your container's IP.
- **TCPSocket**: The Kubelet tries to open a TCP connection on a specific port (e.g., 3306 for MySQL). If the "handshake" succeeds, the probe passes.
- **Exec**: The Kubelet runs a specific command (like cat /tmp/ready) inside your container's namespaces. If the command returns exit code 0, it's a success.

### The PHP Implementation

If you choose the HTTPGet method (the most common for web apps), you must create those files or routes in your code.

**The Liveness Script (/healthz.php)**: This should be very "cheap" and fast. It simply proves the PHP engine is still processing requests.

```php
<?php
    // A very simple checks
    http_response_code(200);
    echo "OK";
?>
```

**The Readiness Script (/ready.php)**: This is more "expensive." It checks the dependencies your app needs to actually do work.

```php
<?php
    // Check if we can connect to MySQL
    $conn = @new mysqli("mysql-service", "user", "pass", "db");

    if ($conn->connect_error) {
        // If DB is down, return 500
        http_response_code(500);
        echo "Database Offline";
    } else {
        // If DB is up, return 200
        http_response_code(200);
        echo "Ready";
    }
?>
```

## Scenario: The PHP + MySQL Web App

### A. Startup Probe: The "Patience" Phase

Imagine your MySQL pod needs to perform a heavy database migration or index rebuild when it first starts. This takes 2 minutes.

- Problem: If you only had a Liveness probe, it would see the MySQL process isn't responding yet and kill the container every 30 seconds. The database would be stuck in a "Restart Loop."
- Solution: You set a Startup Probe. Kubernetes will disable all other probes until this one passes. It gives the database the 2 minutes it needs to "wake up" without being killed.

### B. Readiness Probe: The "Traffic Cop"

Your PHP pod is up, but it needs to verify it can actually reach the MySQL database before it starts taking user requests.

- **Use Case**: You create an endpoint in your PHP code called /ready.php. This script tries to mysqli_connect() to the database.
- **The Probe**: Kubelet hits /ready.php every 5 seconds.
- **Outcome**: If the database is down, /ready.php returns a 500 error. The ***Readiness Probe fails***. K8s immediately removes this specific PHP pod from the ***Service*** Endpoints. Users won't get a "Database Connection Error" page; they will be routed to a different PHP pod that is ready.

### C. Liveness Probe: The "Resuscitator"

Your PHP pod is running, but due to a memory leak or a deadlock, the process gets stuck. It's technically "running" (the process exists), but it's frozen.

**Use Case**: You have a /healthz endpoint that just returns 200 OK.
**The Probe**: Kubelet hits /healthz every 10 seconds.
**Outcome**: When the app deadlocks, it stops responding to the Kubelet. After 3 failed attempts, the ***Liveness Probe fails***. The Kubelet ***kills the container and starts a fresh one***. This is "Self-Healing."

### Don't let Liveness check the Database

As an aspiring SRE, remember this rule:

- A Liveness probe should only check the app itself. If your PHP Liveness probe checks if MySQL is up, and MySQL goes down, all your PHP pods will restart in a loop. They won't be able to fix the problem by restarting.
- Instead, use a Readiness probe for the database check—this way, the pods stay alive but just stop taking traffic until the database returns.

## The SRE "Best Practice"

For a professional PHP + MySQL setup, the HTTPGet method is the gold standard.

- Liveness: Use a simple healthz.php. If it fails, the container is "brain dead"—restart it.
- Readiness: Use a ready.php that checks the MySQL connection. If it fails, the container is "busy/unreachable"—stop sending users to it, but don't kill it (restarting won't fix a database that is down).

## How the YAML looks

Once you have written your ready.php, you tell Kubernetes where to find it:

```yaml
spec:
  containers:
  - name: php-app
    image: my-php-app:v1
    readinessProbe:
      httpGet:
        path: /ready.php
        port: 80
      initialDelaySeconds: 5  # Wait 5 seconds after start before checking
      periodSeconds: 10       # Check every 10 seconds
```
