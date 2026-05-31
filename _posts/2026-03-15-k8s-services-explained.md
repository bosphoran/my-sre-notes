---
layout: post
title: "K8s Services Explained"
date: 2026-03-15
categories: [k8s]
---

## Introduction

In K8s, if you want to reach a pod, you reach it through a service. Since, service is the gate keep or the load balancer.
Pod IPs are ephemural/temporary while service IP is static. So, whenever we want to reach a pod with 3 replicas, we reach the service, and the service redirects to the available pod.

### Deployment (The Manager)

Its job is to ensure that a specific number of Pods are running and healthy. It handles the Lifecycle. If you want 3 PHP Pods, the Deployment makes it so. If one crashes, it replaces it.

### Service (The Endpoint)

Its job is to provide a single, unchanging IP address and DNS name to reach those Pods. It handles the Networking.

## Why "3 Deployment Pods" and "1 Service"?

- **3 Deployment Pods**: You run 3 replicas for High Availability and Scaling. If Node A goes down, your app survives on Node B and C. If traffic spikes, you scale these to 5 or 10.

- **1 Service** You only need one "Front Door." You don't want your users (or your other internal apps) to have to track 3 different IP addresses that change every time a Pod restarts. The Service acts as a Load Balancer that sits in front of all 3 Pods.

- **Crucial SRE Detail**: There is no such thing as a "Service Pod." A Service is just a set of rules in the cluster's network configuration (programmed by **kube-proxy**). It doesn't use a container; it's a virtual resource.

## Do I need a Service to talk to a Deployment?

- Technically: No.
- Architecturally: Yes.

### The "No" (Direct Access)

Since K8s uses a flat network, every Pod can ping every other Pod IP, even across different nodes. If you happen to know the IP of php-pod-xyz is 10.244.1.5, a pod on another node can reach it directly.

### The "Yes" (The Service Purpose)

As an SRE, you never want to rely on direct Pod-to-Pod communication for three reasons:

- **Volatility**: Pods are ephemeral. If your PHP Deployment does a "Rolling Update" to a new version, all 3 old Pods are killed and 3 new ones are created with completely different IPs. Your direct connection would break immediately.
- **Load Balancing**: If you talk to a Pod IP directly, you are only hitting one instance. If that instance is busy, the other 2 replicas sit idle. A Service automatically distributes traffic across all healthy replicas.
- **Service Discovery**: You don't want to hardcode IPs in your code. With a Service, you just tell your app to connect to <http://php-service>. K8s DNS handles the rest.

## The "Big Picture" Flow

- Deployment creates 3 PHP Pods (e.g., 10.1.1.2, 10.1.1.3, 10.1.1.4).
- Service is created with IP 10.96.0.10 and a Label Selector (e.g., app: php).
- K8s Control Plane looks for any Pods with the label app: php and adds their IPs to the Service's Endpoints list.
- Traffic hits 10.96.0.10, and kube-proxy redirects it to one of the 3 Pod IPs.

## ClusterIP vs NodePort

A NodePort service is a ClusterIP service with an extra door attached.
When you define a Service as type: NodePort, Kubernetes automatically creates a ClusterIP for it first.

### The Traffic Flow Logic

Think of it like a building with two entrances:

- **The Internal Entrance (ClusterIP)**: This is the "employees only" door inside the lobby. Pods inside the cluster use this. It’s efficient and stable.
- **The External Entrance (NodePort)**: This is a specific door on the outside of the building (a port on the physical host machine). Traffic from the outside world hits this door, and the **kube-proxy** immediately redirects it to the Internal Entrance (ClusterIP), which then sends it to the Pods.

### What happens if you use both?

Nothing conflicts. If a PHP Pod talks to the MySQL Service via ClusterIP, the traffic stays entirely within the virtual network. If you hit the NodePort from your browser, kube-proxy performs a NAT (Network Address Translation) to bring that traffic into the same virtual path.
