---
layout: post
title: "Network Policy"
date: 2026-08-08
categories: [k8s]
---

## Introduction

In networking, "**east-west**" and "**north-south**" traffic refer to the directions data moves within a network.

1. **North-South Traffic:** This is the data that travels between external networks (like the internet) and an internal data center or network. For example, a request coming in from the internet to a private K8s cluster.
2. **East-West Traffic:** This refers to the data that moves between devices or servers within the same internal network or data center. For example, traffic between pods in a K8s cluster.

When it comes to network security, both east-west and north-south traffic need to be secured. To achieve this, the **Zero Trust Model** is a recommended approach.

> The **Zero Trust Principle** is a security approach that assumes no one is trusted by default, whether inside or outside the network. Everyone and everything must be verified before access is granted.

### Traffic Management In Kubernetes

K8s cluster have a dedicated Pod Network that handles all pod-to-pod (East-West Traffic) communication within the cluster.

- Each pod is assigned a unique IP address
- Pods can be deployed across different Namespaces.
- By default, all pods can communicate with each other freely

---

### Kubernetes Network Policy

K8s **NetworkPolicy** is a native namespace-scoped resource type used to control the traffic flow between pods within the cluster.

There are two types of rules we can define in a network policy:

1. **Ingress**: Ingress defines the rules for inbound communications i.e. the incoming traffic to the pod.
2. **Egress**: Egress defines the rules for outbound communications i.e. the outgoing traffic from the pod.

Using network policy we can define rules to allow or deny communication based on various criteria such as pod labels, namespaces, IP addresses etc. Essentially, they act as a **virtual firewall** between pods.

### Network Policy & CNI Plugins

When a network policy is applied, it's the **CNI** plugin's responsibility to enforce that policy at the network level. The **CNI plugin translates the K8s network policy into low-level network rules** that nodes can enforce.

![Network Policy 2]({{ site.baseurl }}/assets/img/k8s-course/network-policy-2.jpg)

---

### Network Policy Object

Let’s say you have an application running as a deployment in the backend namespace. In terms of the traffic it receives (ingress) and requests it initiates (egress), the following are the requirements:

- The application should receive traffic (ingress) only from pods with the label app: frontend from the frontend namespace. Other sources should be denied.
- It should accept traffic only on port 80.
- The egress traffic, meaning the requests initiated from the backend pods, is allowed only to pods with the label app: payment-api in the payment namespace.
- Additionally, the backend application needs to communicate with services outside the cluster in the IP range 10.0.5.0/24. So, egress should be allowed to this range as well.

![Network Policy 3]({{ site.baseurl }}/assets/img/k8s-course/network-policy-3.jpg)

1. The **podSelector parameter under spec** specifies which pods (target pods) the NetworkPolicy applies to (i.e., which pods are selected by the policy). In our example, the podSelector targets pods labeled app: backend belongs to backend namespace. Which means the policy specifically manages traffic for these pods. Other pods remain unaffected.
2. The **podSelector under rules** (e.g., ingress or egress): Defines the pods that are allowed to send or receive traffic to/from the target pods (backend pods)

You need to specify the policyTypes, which can be either **Ingress**, **Egress**, or both.

### Ingress and Egress Rules

- **Ingress rules** define which pods or external sources can send traffic to your selected pods. In our example, ingress traffic is allowed only from pods labeled **app: frontend** in the frontend namespace, and only on **port 80**.
- **Egress rules** control which destinations your pods are allowed to communicate with. For the backend pods, egress is allowed to payment-api pods in the payment namespace and to the external IP range **10.0.5.0/24**.

---

### Deploy Demo Application

Let's deploy an application which includes three NGINX-based deployments in separate namespaces—frontend, backend, and payment.

1. frontend, backend, and payment namespaces.
2. ConfigMaps containing custom HTML pages, which will be served by NGINX to differentiate each service.
3. Nginx based deployments (webserver, backend-api and payment-api) in each namespace, with their respective ConfigMaps mounted to serve custom HTML content with unique messages.
4. Each NGINX deployment will be exposed via a ClusterIP service, listening on port 80 for internal cluster communication.

**Namespace labels:**

- name: <namspace-name>
- env: production
- project: ecommerce-platform
- version: "1.0.0"

**Deployment labels:**

- app: <deployment-name>
- env: production
- project: ecommerce-platform
- version: "1.0.0"

We will use the following YAML manifest to deploy the services. This YAML sets up the ConfigMaps, Deployments, and Services for our application.

```bash
# deploy the demo application
vagrant@controlplane:~$ kubectl apply -f https://raw.githubusercontent.com/techiescamp/cka-certification-guide/main/network-policy/deployments.yaml
namespace/frontend created
namespace/backend created
namespace/payment created
configmap/nginx-config created
configmap/nginx-config created
configmap/nginx-config created
deployment.apps/frontend-deployment created
service/frontend-service created
deployment.apps/backend-deployment created
service/backend-service created
deployment.apps/payment-deployment created
service/payment-service created
```

```bash
# Validate Deployments and Services
vagrant@controlplane:~$ kubectl get deploy,services -n frontend
NAME                                  READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/frontend-deployment   1/1     1            1           65s

NAME                       TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
service/frontend-service   ClusterIP   10.100.60.125   <none>        80/TCP    65s

# 
vagrant@controlplane:~$ kubectl get deploy,services -n backend
NAME                                 READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/backend-deployment   1/1     1            1           93s

NAME                      TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
service/backend-service   ClusterIP   10.106.124.51   <none>        80/TCP    92s

# 
vagrant@controlplane:~$ kubectl get deploy,services -n payment
NAME                                 READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/payment-deployment   1/1     1            1           2m3s

NAME                      TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)   AGE
service/payment-service   ClusterIP   10.106.170.173   <none>        80/TCP    2m2s
```

### Verify Service URLs

Since these services are ClusterIP, we will use curl commands from within the cluster to verify the service URLs.

```bash
vagrant@controlplane:~$ FRONTEND_IP=$(kubectl get svc frontend-service -n frontend -o jsonpath='{.spec.clusterIP}')
vagrant@controlplane:~$ curl http://$FRONTEND_IP
<!DOCTYPE html>
<html>
<body>
<h1>Hey! I am the website frontend</h1>
</body>
</html>

# 
vagrant@controlplane:~$ BACKEND_IP=$(kubectl get svc backend-service -n backend -o jsonpath='{.spec.clusterIP}')
vagrant@controlplane:~$ curl http://$BACKEND_IP
<!DOCTYPE html>
<html>
<body>
<h1>Hey I am the backend server</h1>
</body>
</html>

# 
vagrant@controlplane:~$ PAYMENT_IP=$(kubectl get svc payment-service -n payment -o jsonpath='{.spec.clusterIP}')
vagrant@controlplane:~$ curl http://$PAYMENT_IP
<!DOCTYPE html>
<html>
<body>
<h1>Hey I am the payment API</h1>
</body>
</html>
```

### Test Connectivity between Services

Let's check the connectivity between services: frontend -> backend & payment, backend -> frontend & payment, payment -> frontend and backend.

To check the connectivity between deployment, we will exec into the respective pod and curl the relevant service DNS endpoints.

In the **exec** command, we get the relevant pod name using labels and jsonpath

```bash
# frontend -> backend & payment
vagrant@controlplane:~$ kubectl exec -it $(k get pods --selector=app=webserver -o jsonpath='{.items[0].metadata.name}' -n frontend) -n frontend -- /bin/bash
root@frontend-deployment-7ff968f54d-mnhzr:/# curl backend-service.backend.svc.cluster.local
<!DOCTYPE html>
<html>
<body>
<h1>Hey I am the backend server</h1>
</body>
</html>
root@frontend-deployment-7ff968f54d-mnhzr:/# curl payment-service.payment.svc.cluster.local
<!DOCTYPE html>
<html>
<body>
<h1>Hey I am the payment API</h1>
</body>
</html>
root@frontend-deployment-7ff968f54d-mnhzr:/# exit
exit
```

```bash
# backend -> frontend & payment
vagrant@controlplane:~$ kubectl exec -it $(k get pods --selector=app=backend-api -o jsonpath='{.items[0].metadata.name}' -n backend) -n backend -- /bin/bash
root@backend-deployment-69d96d7b58-2db5m:/# curl frontend-service.frontend.svc.cluster.local
<!DOCTYPE html>
<html>
<body>
<h1>Hey! I am the website frontend</h1>
</body>
</html>
root@backend-deployment-69d96d7b58-2db5m:/# curl payment-service.payment.svc.cluster.local
<!DOCTYPE html>
<html>
<body>
<h1>Hey I am the payment API</h1>
</body>
</html>
root@backend-deployment-69d96d7b58-2db5m:/# exit
exit
```

```bash
# payment -> frontend and backend
vagrant@controlplane:~$ kubectl exec -it $(k get pods --selector=app=payment-api -o jsonpath='{.items[0].metadata.name}' -n payment) -n payment -- /bin/bash
root@payment-deployment-5df467d4d6-h8d6v:/# curl frontend-service.frontend.svc.cluster.local
<!DOCTYPE html>
<html>
<body>
<h1>Hey! I am the website frontend</h1>
</body>
</html>
root@payment-deployment-5df467d4d6-h8d6v:/# curl backend-service.backend.svc.cluster.local
<!DOCTYPE html>
<html>
<body>
<h1>Hey I am the backend server</h1>
</body>
</html>
root@payment-deployment-5df467d4d6-h8d6v:/# exit
exit
```

Now we have a fully working demo application without any traffic restrictions. This means all the pods in different namespaces can communicate with each other via IP and DNS without any issues.

---

### Deny all Ingress and Egress Traffic

We will block all ingress and egress traffic across the three namespaces by applying a default deny-all network policy, with the exception of DNS traffic. This approach ensures that no traffic is allowed by default, and only explicitly permitted traffic is allowed, enhancing security and control.

To enable DNS resolution, we will allow egress traffic to the kube-system namespace on port 53, permitting pods to communicate with **CoreDNS**. This ensures DNS resolution remains functional, as it would otherwise be blocked by the default deny policy.

By applying the deny-all policy, we restrict pod communication as intended, allowing us to test and confirm that network policies are being correctly enforced within each namespace.

```bash
# create a YAML file named deny-all-np.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
  egress:
  - to:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: kube-system
    ports:
    - protocol: UDP
      port: 53
    - protocol: TCP
      port: 53
```

This K8s **NetworkPolicy** does the following:

- **The podSelector:** {} means the policy applies to all pods in the namespace where it's created.
- Because no other specific ingress or egress rules are defined, all traffic is blocked except for the exception allowing communication with the kube-system namespace for DNS on port 53 (UDP and TCP) (identified by the label kubernetes.io/metadata.name: kube-system).

```bash
# create the deny-all network policy in all three namespaces
vagrant@controlplane:~$ kubectl apply -f deny-all-np.yaml -n frontend
networkpolicy.networking.k8s.io/default-deny-all created

#
vagrant@controlplane:~$ kubectl apply -f deny-all-np.yaml -n backend
networkpolicy.networking.k8s.io/default-deny-all created

#
vagrant@controlplane:~$ kubectl apply -f deny-all-np.yaml -n payment
networkpolicy.networking.k8s.io/default-deny-all created
```

Make sure that all the network policies are created in their respective namespaces.

```bash
vagrant@controlplane:~$ kubectl get netpol -n frontend
NAME               POD-SELECTOR   AGE
default-deny-all   <none>         103s

#
vagrant@controlplane:~$ kubectl get netpol -n backend
NAME               POD-SELECTOR   AGE
default-deny-all   <none>         97s

#
vagrant@controlplane:~$ kubectl get netpol -n payment
NAME               POD-SELECTOR   AGE
default-deny-all   <none>         92s
```

Now, validate Deny Policies by attempting to access the backend and payment service endpoints from the frontend pod.

```bash
vagrant@controlplane:~$ kubectl exec -it $(k get pods --selector=app=webserver -o jsonpath='{.items[0].metadata.name}' -n frontend) -n frontend -- /bin/bash
root@frontend-deployment-7ff968f54d-mnhzr:/# curl backend-service.backend.svc.cluster.local -v
* Host backend-service.backend.svc.cluster.local:80 was resolved.
* IPv6: (none)
* IPv4: 10.106.124.51
*   Trying 10.106.124.51:80...
* connect to 10.106.124.51 port 80 from 10.244.196.141 port 39620 failed: Connection timed out
* Failed to connect to backend-service.backend.svc.cluster.local port 80 after 129903 ms: Could not connect to server
* closing connection #0
curl: (28) Failed to connect to backend-service.backend.svc.cluster.local port 80 after 129903 ms: Could not connect to server
root@frontend-deployment-7ff968f54d-mnhzr:/#
root@frontend-deployment-7ff968f54d-mnhzr:/# curl payment-service.payment.svc.cluster.local -v
* Host payment-service.payment.svc.cluster.local:80 was resolved.
* IPv6: (none)
* IPv4: 10.106.170.173
*   Trying 10.106.170.173:80...
* connect to 10.106.170.173 port 80 from 10.244.196.141 port 54318 failed: Connection timed out
* Failed to connect to payment-service.payment.svc.cluster.local port 80 after 129847 ms: Could not connect to server
* closing connection #0
curl: (28) Failed to connect to payment-service.payment.svc.cluster.local port 80 after 129847 ms: Could not connect to server
root@frontend-deployment-7ff968f54d-mnhzr:/#
root@frontend-deployment-7ff968f54d-mnhzr:/# exit
exit
command terminated with exit code 28
```

As shown in the output, the connection will remain stuck in the "**Trying**" state for some time and it will throw the error "**Connection timed out**", as it is unable to reach the services due to the default deny policies.

### Bidirectional Connectivity

If a NetworkPolicy allows incoming traffic to a pod on port 80, the pod can automatically send a response back to the source without needing an explicit egress rule.

This works because K8s tracks connections (conntrack). When a pod (like a frontend pod) starts a connection (e.g., to a backend pod), K8s remembers it. Once that connection is established, the backend pod can respond to the frontend pod without needing a specific egress rule, as the response is considered part of the same session.

---

### Implement Network Policies for Secure Communication Between Services

Following are the connections to be allowed.

- Frontend Connectivity: Allow all ingress connections to frontend pods.
- Frontend to Backend Connectivity: Backend pods will accept only ingress connections from frontend pods.
- Backend to Payment Connectivity: Payment pods will accept ingress connections only from backend pods.

Following are the connections to be blocked.

- Frontend to Payment Pods: Frontend pods are blocked from connecting to payment pods, meaning ingress from frontend pods is denied.
- All Egress from Payment Pods: All egress connectivity from egress pods should be blocked.

![Network Policy 4]({{ site.baseurl }}/assets/img/k8s-course/network-policy-4.jpg)

### Frontend Network Policy

For the frontend, we need to allow all incoming (ingress) connections. Additionally, we must enable outgoing (egress) connectivity to the backend pods, as the frontend needs to communicate with the backend.

```bash
# Create a YAML file frontend-np.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: frontend-network-policy
  namespace: frontend
spec:
  podSelector:
    matchLabels:
      app: webserver
      tier: frontend
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - ports:
    - protocol: TCP
      port: 80
  egress:
  - to:
    - namespaceSelector:
        matchLabels:
          name: backend
      podSelector:
        matchLabels:
          app: backend-api
    ports:
    - protocol: TCP
      port: 80
```

```bash
# create a network policy for frontend
vagrant@controlplane:~$ kubectl apply -f frontend-np.yaml
networkpolicy.networking.k8s.io/frontend-network-policy created
```

To test the policy, get the cluster IP of the frontend service. Since all ingress is allowed, you should be able to access the frontend service from the control plane node.

```bash
vagrant@controlplane:~$ FRONTEND_IP=$(kubectl get svc frontend-service -n frontend -o jsonpath='{.spec.clusterIP}')
vagrant@controlplane:~$ curl http://$FRONTEND_IP
<!DOCTYPE html>
<html>
<body>
<h1>Hey! I am the website frontend</h1>
</body>
</html>
```

We have also added egress rules to the backend pods.

Let's test the egress connectivity from the frontend pod by exec'ing into a frontend pod and using curl to access the backend service via its service DNS.

```bash
vagrant@controlplane:~$ kubectl exec -it $(k get pods --selector=app=webserver -o jsonpath='{.items[0].metadata.name}' -n frontend) -n frontend -- /bin/bash
root@frontend-deployment-7ff968f54d-mnhzr:/# curl backend-service.backend.svc.cluster.local -v
* Host backend-service.backend.svc.cluster.local:80 was resolved.
* IPv6: (none)
* IPv4: 10.106.124.51
*   Trying 10.106.124.51:80...
* connect to 10.106.124.51 port 80 from 10.244.196.141 port 38272 failed: Connection timed out
* Failed to connect to backend-service.backend.svc.cluster.local port 80 after 130958 ms: Could not connect to server
* closing connection #0
curl: (28) Failed to connect to backend-service.backend.svc.cluster.local port 80 after 130958 ms: Could not connect to server
root@frontend-deployment-7ff968f54d-mnhzr:/# exit
exit
command terminated with exit code 28
```

As you can see, you won't be able to connect because, although we have added an egress rule to the frontend pod, the backend pod doesn't have any ingress rule applied. The default deny policy we set is blocking the connection.

### Backend Network Policy

For the backend application, we need to allow ingress connections from the frontend-api pods in the frontend namespace.

Additionally, we need to allow egress connections to the payment pods, so the backend can connect to the payment-api pods.

```bash
# Create a YAML file named backend-np.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: backend-network-policy
  namespace: backend
spec:
  podSelector:
    matchLabels:
      app: backend-api
      tier: backend
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          name: frontend
      podSelector:
        matchLabels:
          app: webserver
          tier: frontend
    ports:
    - protocol: TCP
      port: 80
  egress:
  - to:
    - namespaceSelector:
        matchLabels:
          name: payment
      podSelector:
        matchLabels:
          app: payment-api
          tier: payment
    ports:
    - protocol: TCP
      port: 80
```

```bash
# create a network policy for backend
vagrant@controlplane:~$ kubectl apply -f backend-np.yaml
networkpolicy.networking.k8s.io/backend-network-policy created
```

Let's test the ingress connectivity from the frontend pod by exec'ing into a frontend pod and using curl to access the backend service via its service DNS.

```bash
vagrant@controlplane:~$ kubectl exec -it $(k get pods --selector=app=webserver -o jsonpath='{.items[0].metadata.name}' -n frontend) -n frontend -- /bin/bash
root@frontend-deployment-7ff968f54d-mnhzr:/# curl backend-service.backend.svc.cluster.local -v
* Host backend-service.backend.svc.cluster.local:80 was resolved.
* IPv6: (none)
* IPv4: 10.106.124.51
*   Trying 10.106.124.51:80...
* Connected to backend-service.backend.svc.cluster.local (10.106.124.51) port 80
* using HTTP/1.x
> GET / HTTP/1.1
> Host: backend-service.backend.svc.cluster.local
> User-Agent: curl/8.14.1
> Accept: */*
>
* Request completely sent off
< HTTP/1.1 200 OK
< Server: nginx/1.31.3
< Date: Sat, 08 Aug 2026 14:43:53 GMT
< Content-Type: text/html
< Content-Length: 83
< Last-Modified: Sat, 08 Aug 2026 12:31:57 GMT
< Connection: keep-alive
< ETag: "6a7721bd-53"
< Accept-Ranges: bytes
<
<!DOCTYPE html>
<html>
<body>
<h1>Hey I am the backend server</h1>
</body>
</html>
* Connection #0 to host backend-service.backend.svc.cluster.local left intact
root@frontend-deployment-7ff968f54d-mnhzr:/#
root@frontend-deployment-7ff968f54d-mnhzr:/# exit
exit
```

As you can see, the webserver pod in the frontend namespace can now connect to the backend-api pods.
However, if you try to connect to the payment pods from the backend-api pods, the connection will fail.

Let’s add the relevant network policy to the payment-api pods in the payment namespace to allow connections from the backend-api pods.

### Payment Network Policy

For the payment-api pods, we need to allow only ingress connection from the backend-api pods.

```bash
# Create a YAML file named payment-np.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: payment-network-policy
  namespace: payment
spec:
  podSelector:
    matchLabels:
      app: payment-api
      tier: payment
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          name: backend
      podSelector:
        matchLabels:
          app: backend-api
          tier: backend
```

```bash
# create a network policy for payment
vagrant@controlplane:~$ kubectl apply -f payment-np.yaml
networkpolicy.networking.k8s.io/payment-network-policy created
```

Let's test the ingress connectivity from the backend-api pod by exec'ing into a payment-api pod and using curl to access the payment service via its service DNS.

```bash
vagrant@controlplane:~$ kubectl exec -it $(k get pods --selector=tier=backend -o jsonpath='{.items[0].metadata.name}' -n backend) -n backend -- /bin/bash
root@backend-deployment-69d96d7b58-2db5m:/# curl payment-service.payment.svc.cluster.local -v
* Host payment-service.payment.svc.cluster.local:80 was resolved.
* IPv6: (none)
* IPv4: 10.106.170.173
*   Trying 10.106.170.173:80...
* Connected to payment-service.payment.svc.cluster.local (10.106.170.173) port 80
* using HTTP/1.x
> GET / HTTP/1.1
> Host: payment-service.payment.svc.cluster.local
> User-Agent: curl/8.14.1
> Accept: */*
>
< HTTP/1.1 200 OK
< Server: nginx/1.31.3
< Date: Sat, 08 Aug 2026 14:48:06 GMT
< Content-Type: text/html
< Content-Length: 80
< Last-Modified: Sat, 08 Aug 2026 12:31:57 GMT
< Connection: keep-alive
< ETag: "6a7721bd-50"
< Accept-Ranges: bytes
<
<!DOCTYPE html>
<html>
<body>
<h1>Hey I am the payment API</h1>
</body>
</html>
* Connection #0 to host payment-service.payment.svc.cluster.local left intact
root@backend-deployment-69d96d7b58-2db5m:/#
root@backend-deployment-69d96d7b58-2db5m:/# exit
exit
```

As you can see, the backend application is now able to access the payment service.

---

### Default Deny Network Policies

In K8s, by default all pods can communicate with each other without restrictions. There are no rules in place to control or limit network traffic between pods by default.

### Deny all ingress traffic

A default ingress deny policy is a good starting point for creating a secure environment, as you can then create additional policies to allow only the necessary traffic.

```bash
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
spec:
  podSelector: {}
  policyTypes:
  - Ingress
```

In the given manifest,

- The podSelector: {} selects all pods in the namespace.
- The policyTypes: - Ingress means this policy only controls incoming (ingress) traffic.
- Since the Ingress rule section is not defined (there are no specific from rules), it means that all ingress traffic to the pods in this namespace is denied by default.

### Deny all egress traffic

This policy will block all outgoing traffic from the pods in the namespace, preventing them from making any outbound connections.

To enable egress for any pod, you must create an additional network policy with an egress rule and apply it to the required pods.

```bash
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-egress
spec:
  podSelector: {}
  policyTypes:
  - Egress
```

### Deny all ingress and all egress traffic

This policy will block all incoming and outgoing traffic to and from the pods in the namespace, making it the most restrictive network policy.

```bash
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
```

---

### Common Pitfalls and Key Concepts

The K8s **Network Policy API** is simple but can be confusing. When implemented incorrectly or accidental misconfigurations could lead to security gaps or communication issues within your K8s cluster.

1. To fully isolate a pod, you need to apply both ingress and egress policies. Missing one means that traffic might still flow in or out.
2. Network policies apply only to pods that match the specified podSelector. Other pods remain unaffected.
3. Network policies work within a specific namespace. If you want to apply rules across multiple namespaces, you need additional non-native features, like global policies offered by CNI tools like Calico Operators.
4. Network policies use a whitelist model. This means they specify which traffic is allowed but don’t directly block traffic unless you set rules to allow only specific types of traffic.
5. A pod can have multiple policies applied, and they all combine to determine what traffic is allowed.
6. Network policies do not have a priority order. All applicable policies are evaluated together, and traffic is allowed if any one of the policies permits it.
7. You can control network traffic based on particular ports and protocols (like TCP or UDP). Missing a required port or protocol in your policy could lead to unintended traffic being blocked.
8. Pods need egress access to DNS servers (CoreDNS) on port 53 for name resolution. If you’re restricting egress, make sure to allow DNS traffic, or your pod could have trouble reaching other services.
9. When restricting access to external systems, use specific CIDR blocks (IP ranges) to control which external IPs can be reached. Misconfigurations in CIDR can easily lead to blocked or unexpected traffic.
10. Network policies apply to pods, not services. Always target the pods that back a service, rather than targeting the service IP directly.
