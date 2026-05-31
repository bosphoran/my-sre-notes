---
layout: post
title: "Prometheus + Grafana Setup"
date: 2026-03-13
categories: [monitoring]
---

- ***User: admin***
- ***Pass: x0P6gwS1DAO0leYpco24PhMUWXI1zbo1VkJz4jj7***

---

## Prometheus – the data collector

- Prometheus is a time-series database + monitoring system.
- It scrapes metrics from endpoints in your cluster at regular intervals (default 15s).
- Metrics can come from:
  - Node exporters → CPU, RAM, disk, load of your server nodes
  - kube-state-metrics → Kubernetes object states (pods, deployments, replicas)
  - Applications → any app exposing Prometheus metrics via /metrics endpoint
- Prometheus stores this data in memory and/or disk (depending on configuration).

## Exporters / Agents – how data is collected

- Prometheus itself doesn’t actively run on every node; instead it scrapes metrics from exporters:
  - node-exporter → installed on each node to expose system metrics
  - kube-state-metrics → runs inside the cluster to expose Kubernetes object status
- These exporters are HTTP endpoints that Prometheus pulls data from.
- Think of them as sensors: they expose data, Prometheus collects it.

## Grafana – the visualization layer

- Grafana is not collecting metrics itself, it only reads from data sources (like Prometheus).
- It allows you to:
  - Create dashboards
  - Build charts, tables, gauges
  - Filter by namespace, node, pod, or time
- Grafana queries Prometheus via PromQL, a query language for metrics.

> node-exporter + kube-state-metrics → expose metrics → Prometheus scrapes metrics → Grafana queries Prometheus → dashboards show live data

---

## Installing Prometheus + Grafana

### Step 0 – Update system and install prerequisites

```bash
- sudo apt update && sudo apt upgrade -y # Update system packages
- sudo apt install -y curl wget tar vim # Install curl, wget, tar, editor
```

### Step 1 – Install Helm

```bash	
- curl -LO https://get.helm.sh/helm-v3.14.0-linux-amd64.tar.gz # Download Helm
- tar -zxvf helm-v3.14.0-linux-amd64.tar.gz # Extract the archive
- sudo mv linux-amd64/helm /usr/local/bin/helm # Move Helm to PATH
- helm version # Verify Helm installed
```

### Step 2 – Add Helm repositories

```bash
- helm repo add prometheus-community https://prometheus-community.github.io/helm-charts # Prometheus charts
- helm repo add grafana https://grafana.github.io/helm-charts # Grafana charts
- helm repo update # Update repos
- helm repo list # Verify repos
```

### Step 3 – Create monitoring namespace

```bash
- kubectl create namespace monitoring # Create namespace for Prometheus and Grafana
- kubectl get namespaces # Verify namespace exists
```

### Step 4 – Install Prometheus (no persistent storage, no Alertmanager)

```bash
- helm install prometheus prometheus-community/prometheus --namespace monitoring --set server.persistentVolume.enabled=false \
  --set alertmanager.enabled=false # Install Prometheus server and node-exporter, in-memory only
- kubectl get pods -n monitoring # Verify Prometheus pods are running
```

### Step 5 – Install Grafana

```bash	
- helm install grafana grafana/grafana --namespace monitoring # Install Grafana in the same namespace
- kubectl get pods -n monitoring # Verify Grafana pod is running
```

### Step 6 – Get Grafana admin password

```bash
- kubectl get secret grafana -n monitoring -o jsonpath="{.data.admin-password}" | base64 --decode
```

***Username: admin***
***Password: (from above command)***

### Step 7 – Expose Grafana via NodePort (so it can be accessed from other devices in LAN)

```bash
- kubectl patch svc grafana -n monitoring -p '{"spec": {"type": "NodePort"}}'
- kubectl get svc -n monitoring grafana    # Note the NodePort number (e.g., 31234)
```

### Access Grafana from any device in the network:

- <http://SERVER-IP:NodePort>

### Step 8 – Add Prometheus as Grafana Data Source

> Login to Grafana → Settings → Data Sources → Add → Prometheus
> URL: <http://prometheus-server.monitoring.svc.cluster.local>
> Save & Test → should succeed

### Step 9 – Import Dashboards (optional)

> Dashboards → Import → Use IDs:

| Purpose | Dashboard ID |
| :---: | :---: |
| Cluster overview | 6417 |
| Nodes | 1860 |
| Pods | 6417 |
| Kube-state | 13332 |

---

## Update System & Install Prerequisites

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y curl wget tar vim
```

### Step 1: Install Helm

```bash
curl -LO https://get.helm.sh/helm-v3.14.0-linux-amd64.tar.gz
tar -zxvf helm-v3.14.0-linux-amd64.tar.gz
sudo mv linux-amd64/helm /usr/local/bin/helm
helm version
```

### Step 2: Add Helm repos

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update
helm repo list
```

### Step 3: Create monitoring namespace

```bash
kubectl create namespace monitoring
kubectl get namespaces
```

### Step 4: Install Prometheus (in-memory, no Alertmanager)

```bash
helm install prometheus prometheus-community/prometheus \
  --namespace monitoring \
  --set server.persistentVolume.enabled=false \
  --set alertmanager.enabled=false

kubectl get pods -n monitoring
```

### Step 5: Install Grafana

```bash
helm install grafana grafana/grafana --namespace monitoring
kubectl get pods -n monitoring
```

### Step 6: Get Grafana admin password

- echo "Grafana login credentials:"
- echo "Username: admin"
- echo "Password:"

```bash
kubectl get secret grafana -n monitoring -o jsonpath="{.data.admin-password}" | base64 --decode
echo ""
```

### Step 7: Expose Grafana via NodePort (LAN access)

```bash
kubectl patch svc grafana -n monitoring -p '{"spec": {"type": "NodePort"}}'
kubectl get svc -n monitoring grafana
```

```php
echo "Access Grafana from any device in LAN: http://<server-ip>:<NodePort>"
```

### Step 8: Notes for next steps

- echo "1) Login to Grafana with above credentials"
- echo "2) Add Prometheus data source: URL=http://prometheus-server.monitoring.svc.cluster.local"
- echo "3) Import dashboards: 6417, 1860, 13332 etc."

---

### ✅ What This Does

- Installs Helm
- Adds Prometheus & Grafana Helm charts
- Creates a namespace monitoring
- Installs Prometheus server + node-exporter + kube-state-metrics (in-memory)
- Installs Grafana and exposes it on a NodePort
- Prints Grafana admin password
- Gives instructions for adding data source and importing dashboards
