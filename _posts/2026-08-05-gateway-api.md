---
layout: post
title: "Gateway API"
date: 2026-08-05
categories: [k8s]
---

## Introduction

The **Gateway API** is a K8s feature that helps create gateways for external traffic entering your cluster.

K8s Ingress has been widely used to handle all incoming traffic to the cluster. As K8s usage increased, the need for more advanced traffic routing also grew.

However, Ingress has the following limitations.

- Supports only host and path-based routing for HTTP/HTTPS (Layer 7).
- Does not support advanced routing features like header-based routing, traffic splitting for canary releases, or A/B testing.
- Cross-namespace routing is complex and not straightforward.
- Relies heavily on annotations for anything beyond basic routing. These annotations differ between various ingress controllers.

Gateway API solves many of these issues. It provides:

- Flexible routing for HTTP, HTTPS, gRPC, TCP, and UDP. (Layer L4 and L7)
- Ability to route traffic based on HTTP headers.
- Cross-namespace traffic routing.
- Reduced reliance on vendor-specific controller annotations, making configurations more portable across environments.
- Advanced routing features for AI/ML workloads and more.

> In summary, the Gateway API is an improved version of K8s Ingress that offers more powerful and flexible traffic management.

---

### Gateway API Controller

Gateway API provides several API resources like Gateway and HTTPRoute to define how traffic should flow. The controller is what implements these core Gateway APIs, and routing is also done by the controller implementation.

### NGINX Gateway Fabric Controller

The image below illustrates the high-level architecture of how the NGINX Gateway Fabric implementation works.

![Gateway API]({{ site.baseurl }}/assets/img/k8s-course/gateway-api.png)

NGINX Gateway Fabric has two main parts: 

1. **The control plane:** The control plane runs as a pod called the fabric controller. It's a K8s controller that watches Gateway API resources. 
2. **The data plane:** The data plane handles the actual traffic. It's a separate NGINX proxy pod that gets created automatically when you create a Gateway resource. This pod runs NGINX along with an NGINX agent.

A K8s Service is also created for the NGINX pod. This service is exposed to the outside world using a LoadBalancer or NodePort, so client requests can reach your app.

Whenever routing rules change, the fabric controller sends the updated NGINX config to the agent using gRPC. The agent then applies the new config to the NGINX pod to route traffic correctly.

### Traffic Flow

Here’s how external traffic reaches the application pods inside the cluster in NGINX Gateway Fabric:

1. The traffic first hits the cloud provider’s LoadBalancer or NodePort, which is exposed by the NGINX data plane pod’s service.
2. The traffic is then forwarded directly to the NGINX pod, which is part of the data plane.
3. Inside this pod, the NGINX agent has already applied the routing rules received from the control plane (fabric controller), so NGINX can route the traffic to the correct application service or backend pod based on the configured rules.

> The fabric controller (control plane) only watches and processes routing rules, but it does not handle any client traffic.

---

### Gateway API Resources

To expose applications using the Gateway API, there are three key resource types:

- GatewayClass
- Gateway
- HTTPRoute
- ReferenceGrant

The image below illustrates the relationship between the resources and the controller.

![Gateway API 2]({{ site.baseurl }}/assets/img/k8s-course/gateway-api-2.png)

---

### GatewayClass

In a cluster, you might have more than one **gateway controller**. For example, one could handle traffic for your applications, while another manages traffic for tools like Argo or Prometheus.

To make sure the right controller handles the right traffic, we use something called a **GatewayClass**. It's like a tag that tells each controller which routing configs it should manage.

![Gateway API 3]({{ site.baseurl }}/assets/img/k8s-course/gateway-api-3.png)

So, when you create routing rules (Gateway), you link them to a specific **GatewayClass**. This way, the rules get picked up by the correct controller.

Here’s an example of a GatewayClass resource that links to an NGINX controller using the ID **gateway.nginx.org/nginx-gateway-controller**. This ID is assigned to the controller during its deployment.

![Gateway API 4]({{ site.baseurl }}/assets/img/k8s-course/gateway-api-4.png)

### Behind the Scenes: Watching & Patching GatewayClass

The controller running inside the cluster has all the built-in logic needed to interact with the K8s API server.

Here is what really happens at the backend.

1. The controller uses the K8s API server’s watch feature to keep an eye on GatewayClass resources.
2. When a GatewayClass is created or modified, the API server notifies the controller of the change (via the watch).
3. It then processes the GatewayClass. If it meets all criteria (correct controllerName, valid references, etc.), the controller updates the status of the GatewayClass, typically setting conditions like "Accepted" or reporting errors/warnings if there are issue.
4. To update the status, the controller sends a PATCH request to the status subresource of the GatewayClass via the API server.
5. The K8s API server then saves this updated status in etcd, which acts as the cluster’s storage. This change becomes visible to anyone querying the GatewayClass, whether through kubectl, API etc.

---

### Gateway

As the name indicates, the Gateway resource defines how external traffic enters the cluster. Think of it as the main entry point for requests coming from outside the cluster.

Each Gateway is associated with a single GatewayClass, which tells K8s which controller should handle its traffic.

![Gateway API 5]({{ site.baseurl }}/assets/img/k8s-course/gateway-api-5.png)

The Gateway controls key configs such as:

- Which port (like 80 or 443) the traffic should come in on.
- Which hostnames (like app.example.com) it accepts.
- TLS configuration and certificates.
- Which GatewayClass/controller will handle the traffic.

For example, a Gateway could be created in the frontend namespace, configured to listen on port 443 for HTTPS traffic for app.example.com, using a specific GatewayClass.

![Gateway API 6]({{ site.baseurl }}/assets/img/k8s-course/gateway-api-6.png)

When you create a Gateway using the NGINX Gateway Fabric controller, here is what happens.

- A new NGINX data plane pod is created in the same namespace.
- A K8s Service (usually of type LoadBalancer or NodePort) is also created to expose that pod to external traffic.

![Gateway API 7]({{ site.baseurl }}/assets/img/k8s-course/gateway-api-7.png)

Next, we need to define routing rules to tell the controller what to do with traffic arriving at the Gateway.
These rules are created using the **HTTPRoute** resource. We will learn about it in the next lesson.

---

### HTTPRoute

The **HTTPRoute** resource defines how traffic should be routed to applications inside the cluster once it reaches the Gateway.

- **/** can route to the frontend service
- **/api** can route to the API service
- **/payment** can route to the payment service

> This is similar to the path-based routing rules we define in an Ingress resource.

Each **HTTPRoute** must be attached to a Gateway using **parentRefs**. This binding is essential for the routing rules to be applied.

![Gateway API 8]({{ site.baseurl }}/assets/img/k8s-course/gateway-api-8.png)

Here is an example HTTPRoute resource.

![Gateway API 9]({{ site.baseurl }}/assets/img/k8s-course/gateway-api-9.png)

In practical terms, the **NGINX fabric controller** takes these paths, **headers, rewrites, backend service names**, etc., and converts them into real NGINX directives like **proxy_set_header, rewrite, and proxy_pass** to build the actual routing configuration. It then sends the configuration to the NGINX Gateway pod, which updates its settings to handle the actual traffic routing.

---

### ReferenceGrant

In Ingress, one of the issues was **cross-namespace routing** - sending traffic from an Ingress object created in one namespace to services in other namespaces.

The Gateway API solves this using the **ReferenceGrant** resource.

> A ReferenceGrant is like a permission slip. It says: "**I allow resources from another namespace to access this resource in my namespace.**"

Suppose, you have a Gateway in the frontend namespace. An **HTTPRoute**, also in the frontend namespace, needs to send traffic to services in the api and payment namespaces.

To allow this, you must create a **ReferenceGrant** in both the api and payment namespaces. This grant gives permission for the **HTTPRoute** in the frontend namespace to access their services as shown in the image below.

![Gateway API 10]({{ site.baseurl }}/assets/img/k8s-course/gateway-api-10.png)

> Without ReferenceGrant, cross-namespace routing will be blocked for security reasons.

---

### Seperation of Concerns

You might be wondering, why do we need a **Gateway** and an **HTTPRoute**? Wouldn’t it be simpler to just have one object that handles everything?

By breaking things into distinct pieces, K8s allows each team to focus on what they do best, without interfering with others. And that’s key for scaling smoothly in large, collaborative environments.

In most enterprise K8s setups, you're not just running a single app managed by one team. You're working in a shared environment where multiple teams need to collaborate, but without stepping on each other’s toes.

Each team brings different tools, responsibilities, and access levels. This is where the Gateway API's **separation of objects** becomes essential.

The separation isn’t just about clean design, it’s what makes collaboration at scale possible.

- Platform team: “We ensure the Gateway is secure, scalable, and stable.”
- App team: “We define how our traffic flows to services.”

It might feel like extra complexity at first. But it brings structure, security, and clarity to how you manage traffic across your Kubernetes cluster.

---

### Deploy Gateway API CRDs

Gateway API objects aren't built into K8s by default. To use them, you need to install the Gateway API **Custom Resource Definitions** (CRDs).

```bash
vagrant@controlplane:~$ kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.4.1/standard-install.yaml
customresourcedefinition.apiextensions.k8s.io/backendtlspolicies.gateway.networking.k8s.io created
customresourcedefinition.apiextensions.k8s.io/gatewayclasses.gateway.networking.k8s.io created
customresourcedefinition.apiextensions.k8s.io/gateways.gateway.networking.k8s.io created
customresourcedefinition.apiextensions.k8s.io/grpcroutes.gateway.networking.k8s.io created
customresourcedefinition.apiextensions.k8s.io/httproutes.gateway.networking.k8s.io created
customresourcedefinition.apiextensions.k8s.io/referencegrants.gateway.networking.k8s.io created

# validate the installed CRDs
vagrant@controlplane:~$ kubectl get crds | grep gateway
backendtlspolicies.gateway.networking.k8s.io            2026-08-06T11:44:21Z
gatewayapis.operator.tigera.io                          2026-06-03T21:53:51Z
gatewayclasses.gateway.networking.k8s.io                2026-08-06T11:44:22Z
gateways.gateway.networking.k8s.io                      2026-08-06T11:44:22Z
grpcroutes.gateway.networking.k8s.io                    2026-08-06T11:44:23Z
httproutes.gateway.networking.k8s.io                    2026-08-06T11:44:24Z
referencegrants.gateway.networking.k8s.io               2026-08-06T11:44:27Z

# check the API resources to view detailed information about the Gateway API CRDs
vagrant@controlplane:~$ kubectl api-resources --api-group=gateway.networking.k8s.io
NAME                 SHORTNAMES   APIVERSION                          NAMESPACED   KIND
backendtlspolicies   btlspolicy   gateway.networking.k8s.io/v1        true         BackendTLSPolicy
gatewayclasses       gc           gateway.networking.k8s.io/v1        false        GatewayClass
gateways             gtw          gateway.networking.k8s.io/v1        true         Gateway
grpcroutes                        gateway.networking.k8s.io/v1        true         GRPCRoute
httproutes                        gateway.networking.k8s.io/v1        true         HTTPRoute
referencegrants      refgrant     gateway.networking.k8s.io/v1beta1   true         ReferenceGrant
```

From the output, you can see that all Gateway API resources are namespace-scoped, except for GatewayClass, which is cluster-scoped.

---

### Setting Up Nginx Fabric Controller

We'll use Helm to deploy the Gateway API controller.

```bash
vagrant@controlplane:~$ git clone https://github.com/techiescamp/cka-certification-guide.git
Cloning into 'cka-certification-guide'...
remote: Enumerating objects: 1181, done.
remote: Counting objects: 100% (441/441), done.
remote: Compressing objects: 100% (196/196), done.
remote: Total 1181 (delta 304), reused 280 (delta 238), pack-reused 740 (from 1)
Receiving objects: 100% (1181/1181), 36.55 MiB | 1.56 MiB/s, done.
Resolving deltas: 100% (573/573), done.

vagrant@controlplane:~$ cd cka-certification-guide/helm-charts/nginx-gateway-fabric/
vagrant@controlplane:~/cka-certification-guide/helm-charts/nginx-gateway-fabric$
```

All configurable options are found in the values.yaml file. You can adjust the settings there to suit your needs.

Below is a snapshot of the directory structure for the NGINX Gateway Fabric Controller Helm chart.

![Gateway API 11]({{ site.baseurl }}/assets/img/k8s-course/gateway-api-11.png)

The Fabric controller is operator-based, so you’ll see NGINX custom resource definitions (CRDs) included in the Helm chart. These CRDs are installed along with the controller components.

The following are the container images used by the NGINX Gateway Fabric Helm chart:

- ghcr.io/nginxinc/nginx-gateway-fabric This is the image used in Fabric controller control plane.
- ghcr.io/nginxinc/nginx-gateway-fabric/nginx This is the NGINX image used in the data plane pods, which are created automatically when you create Gateway resources.

### Exposing NodePorts

Since we're setting this up on a self-hosted cluster, we need to configure the Gateway to use a NodePort service with the following port mappings:

- 32000 for port 80
- 32443 for port 443
- 32500 for port 8080

This setup helps us simulate external traffic using curl. It allows us to test and validate routing through the Gateway API for both HTTPS and non-HTTPS connections, similar to how we did it with Ingress.

![Gateway API 12]({{ site.baseurl }}/assets/img/k8s-course/gateway-api-12.png)

The **custom-values.yaml** file used to configure these port mappings is included in the repo. These values are automatically picked up by the Fabric controller when the Gateway service is created. The file is shown below just for reference.

```bash
nginx: 
  service: 
    type: NodePort
    nodePorts:
    - port: 32000
      listenerPort: 80
    - port: 32500
      listenerPort: 8080
    - port: 32443
      listenerPort: 443
```

### Install Nginx NGINX Gateway Fabric

The following command installs the Helm release named nga in the nginx-gateway namespace, along with all the related NGINX custom resource definitions (CRDs).

```bash
vagrant@controlplane:~/cka-certification-guide/helm-charts/nginx-gateway-fabric$ helm install ngf . -n nginx-gateway --create-namespace -f custom-values.yaml
NAME: ngf
LAST DEPLOYED: Thu Aug  6 12:00:51 2026
NAMESPACE: nginx-gateway
STATUS: deployed
REVISION: 1
TEST SUITE: None
```

Once the installation is complete, make sure all the controller components are running correctly by using the following command.

```bash
vagrant@controlplane:~$ kubectl -n nginx-gateway get all
NAME                                            READY   STATUS    RESTARTS   AGE
pod/ngf-nginx-gateway-fabric-76b65d677d-r84lz   1/1     Running   0          106s

NAME                               TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE
service/ngf-nginx-gateway-fabric   ClusterIP   10.105.253.5   <none>        443/TCP   107s

NAME                                       READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/ngf-nginx-gateway-fabric   1/1     1            1           107s

NAME                                                  DESIRED   CURRENT   READY   AGE
replicaset.apps/ngf-nginx-gateway-fabric-76b65d677d   1         1         1       106s
```

```bash
# let's validate all the installed NGINX CRDs
vagrant@controlplane:~$ kubectl get crds | grep nginx
clientsettingspolicies.gateway.nginx.org                2026-08-06T12:00:48Z
nginxgateways.gateway.nginx.org                         2026-08-06T12:00:48Z
nginxproxies.gateway.nginx.org                          2026-08-06T12:00:48Z
observabilitypolicies.gateway.nginx.org                 2026-08-06T12:00:48Z
snippetsfilters.gateway.nginx.org                       2026-08-06T12:00:49Z
upstreamsettingspolicies.gateway.nginx.org              2026-08-06T12:00:49Z
```

The data plane NGINX pod configuration and the **NodePort** mappings come from the **ngf-proxy-config** custom resource created in the nginx-gateway namespace.

The following command describes this custom resource. In the output, you will see the image URL and the custom NodePort mappings we added in the **custom-values.yaml** file.

```bash
vagrant@controlplane:~$ kubectl describe nginxproxies ngf-proxy-config -n nginx-gateway
Name:         ngf-proxy-config
Namespace:    nginx-gateway
Labels:       app.kubernetes.io/instance=ngf
              app.kubernetes.io/managed-by=Helm
              app.kubernetes.io/name=nginx-gateway-fabric
              app.kubernetes.io/version=2.0.1
              helm.sh/chart=nginx-gateway-fabric-2.0.1
Annotations:  meta.helm.sh/release-name: ngf
              meta.helm.sh/release-namespace: nginx-gateway
API Version:  gateway.nginx.org/v1alpha2
Kind:         NginxProxy
Metadata:
  Creation Timestamp:  2026-08-06T12:01:25Z
  Generation:          1
  Resource Version:    1715492
  UID:                 dbec3849-7606-4711-8de5-f0578fa5632f
Spec:
  Ip Family:  dual
  Kubernetes:
    Deployment:
      Container:
        Image:
          Pull Policy:  IfNotPresent
          Repository:   ghcr.io/nginx/nginx-gateway-fabric/nginx
          Tag:          2.0.1
      Replicas:         1
    Service:
      External Traffic Policy:  Local
      Node Ports:
        Listener Port:  80
        Port:           32000
        Listener Port:  8080
        Port:           32500
        Listener Port:  443
        Port:           32443
      Type:             NodePort
Events:
```

---

### Create GatewayClass

As we’ve learned, the **GatewayClass** is similar to the **IngressClass**. It tells K8s which gateway controller should handle routing traffic to the backend services.

```bash
# create a GatewayClass manifest nginx-gateway-class.yaml
apiVersion: gateway.networking.k8s.io/v1
kind: GatewayClass
metadata:
  name: nginx-gateway-class
spec:
  controllerName: gateway.nginx.org/nginx-gateway-controller
  parametersRef:
    group: gateway.nginx.org
    kind: NginxProxy
    name: ngf-proxy-config
    namespace: nginx-gateway
```

In the manifest above, **gateway.nginx.org/nginx-gateway-controller** is the name of the Gateway controller we deployed.

The **parametersRef** block points to a **NginxProxy** custom resource, which can be customized. This resource defines how the NGINX proxy pods (data plane) should be deployed and configured as part of the Gateway setup.

> The name **nginx-gateway-class** should match the **gatewayClassName** value in the Helm **values.yaml** file (it is the same in our YAML). This is important because the NGINX Gateway controller uses that name internally.

```bash
# deploy the manifest
vagrant@controlplane:~$ kubectl apply -f nginx-gateway-class.yaml
gatewayclass.gateway.networking.k8s.io/nginx-gateway-class created

# validate the GatewayClass
vagrant@controlplane:~$ kubectl get gc
NAME                  CONTROLLER                                   ACCEPTED   AGE
nginx-gateway-class   gateway.nginx.org/nginx-gateway-controller   True       32s
# It should show True under ACCEPTED column as shown below

# if you describe the  GatewayClass, you'll see details like its status, controller name, etc.
vagrant@controlplane:~$ kubectl describe gc nginx-gateway-class
Name:         nginx-gateway-class
Namespace:
Labels:       <none>
Annotations:  <none>
API Version:  gateway.networking.k8s.io/v1
Kind:         GatewayClass
Metadata:
  Creation Timestamp:  2026-08-06T12:13:39Z
  Generation:          1
  Resource Version:    1717776
  UID:                 1a653061-f2d6-4456-857f-2b1faf739ca4
Spec:
  Controller Name:  gateway.nginx.org/nginx-gateway-controller
  Parameters Ref:
    Group:      gateway.nginx.org
    Kind:       NginxProxy
    Name:       ngf-proxy-config
    Namespace:  nginx-gateway
Status:
  Conditions:
    Last Transition Time:  2026-08-06T12:13:39Z
    Message:               GatewayClass is accepted
    Observed Generation:   1
    Reason:                Accepted
    Status:                True
    Type:                  Accepted
    Last Transition Time:  2026-08-06T12:13:39Z
    Message:               Gateway API CRD versions are not recommended. Recommended version is v1.3.0
    Observed Generation:   1
    Reason:                UnsupportedVersion
    Status:                False
    Type:                  SupportedVersion
    Last Transition Time:  2026-08-06T12:13:39Z
    Message:               ParametersRef resource is resolved
    Observed Generation:   1
    Reason:                ResolvedRefs
    Status:                True
    Type:                  ResolvedRefs
Events:                    <none>
```

As highlighted in the following output, the GatewayClass object should be in the Accepted state.

![Gateway API 13]({{ site.baseurl }}/assets/img/k8s-course/gateway-api-13.png)

Now that we have a GatewayClass attached to the gateway controllers, we can use it to expose applications outside the cluster.

---

### Deploy a Demo Application

To test Gateway API routing, we'll deploy the same application we used when testing the Ingress setup.

1. 🖥️ **Frontend App**
   1. shop.techiescamp.com: Routes traffic to the Frontend App, allowing users to browse products and access the main website.
   2. shop.techiescamp.com/cart: Routes traffic to the Frontend App to display items users have added to their cart.
   3. shop.techiescamp.com/orders: Routes traffic to the Frontend App to list users' previous orders.
2. ⚙️ **API Service**
   1. shop.techiescamp.com/api: Routes traffic to the API Service to handle backend operations, such as order management and product updates.
   2. shop.techiescamp.com/api/inventory: Specifically routes traffic to the Inventory API (a part of the API Service) to manage stock levels and check product availability.
3. 💳 **Payment Service**
   1. shop.techiescamp.com/payment: Routes traffic to the Payment Service to securely process user transactions.

```bash
# Execute the following command to deploy the demo application
vagrant@controlplane:~$ kubectl apply -f https://raw.githubusercontent.com/techiescamp/cka-certification-guide/main/ingress/deployments.yaml
namespace/frontend unchanged
namespace/api unchanged
namespace/payment unchanged
configmap/nginx-config unchanged
configmap/nginx-config unchanged
configmap/nginx-config unchanged
deployment.apps/frontend-deployment unchanged
service/frontend-service unchanged
deployment.apps/api-deployment unchanged
service/api-service unchanged
deployment.apps/payment-deployment unchanged
service/payment-service unchanged
```

Validate all the Services

```bash
vagrant@controlplane:~$ kubectl get pods -n frontend
NAME                                  READY   STATUS    RESTARTS       AGE
frontend-deployment-d9848f7b8-rswtv   1/1     Running   1 (109m ago)   44h
#
vagrant@controlplane:~$ kubectl get pods -n api
NAME                             READY   STATUS    RESTARTS       AGE
api-deployment-d9848f7b8-b6gbr   1/1     Running   1 (109m ago)   44h
#
vagrant@controlplane:~$ kubectl get pods -n payment
NAME                                 READY   STATUS    RESTARTS       AGE
payment-deployment-c949bcf48-hj8nv   1/1     Running   1 (111m ago)   44h
```

---

### Create Gateway

Next step is to create a gateway. It is like creating the traffic entry point for your application from the controller. Also it is a namespace-scoped resource.

```bash
# Create frontend-gateway.yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: frontend-gateway
  namespace: frontend
spec:
  gatewayClassName: nginx-gateway-class
  listeners:
  - name: http
    protocol: HTTP
    port: 80
    hostname: "shop.techiescamp.com"
```

- The gatewayClassName: nginx-gateway-class tells Kubernetes which controller will run the Gateway.
- hostname: "shop.techiescamp.com" to only accept traffic sent to shop.techiescamp.com

```bash
# deploy the gateway
vagrant@controlplane:~$ kubectl apply -f frontend-gateway.yaml
gateway.gateway.networking.k8s.io/frontend-gateway created

# verify the created Gateway
vagrant@controlplane:~$ kubectl get gateway -n frontend
NAME               CLASS                 ADDRESS   PROGRAMMED   AGE
frontend-gateway   nginx-gateway-class             True         24s
```

In the above output, **True** means the controller has picked it up and processed it. Meaning the gateway is valid.

```bash
# Describe the gateway to get more information about the it
vagrant@controlplane:~$ kubectl describe gateway frontend-gateway -n frontend
Name:         frontend-gateway
Namespace:    frontend

# ...

Spec:
  Gateway Class Name:  nginx-gateway-class
  Listeners:
    Allowed Routes:
      Namespaces:
        From:  Same
    Hostname:  shop.techiescamp.com
    Name:      http
    Port:      80
    Protocol:  HTTP

# ...
```

Also, if you check the frontend namespace, you’ll see a Gateway pod deployed by the Fabric controller.

```bash
vagrant@controlplane:~$ kubectl get pods -n frontend
NAME                                  READY   STATUS    RESTARTS       AGE
frontend-deployment-d9848f7b8-rswtv   1/1     Running   1 (116m ago)   44h
```

This is an **NGINX proxy pod** that handles the actual traffic routing. It’s the entry point for all traffic managed by this Gateway.

---

### Create HTTPRoute

**HTTPRoute** is where we define how incoming traffic should be routed inside the cluster. For example, which service should handle requests to specific paths or hostnames.

```bash
# Create frontend-httproute.yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: frontend-httproute
  namespace: frontend
spec:
  parentRefs:
  - name: frontend-gateway
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /
    backendRefs:
    - name: frontend-service
      port: 80
  - matches:
    - path:
        type: PathPrefix
        value: /cart
    backendRefs:
    - name: frontend-service
      port: 80
  - matches:
    - path:
        type: PathPrefix
        value: /orders
    backendRefs:
    - name: frontend-service
      port: 80
```

- **parentRefs.name**: frontend-gateway indicates this route is attached to the Gateway named frontend-gateway
- **PathPrefix** - Match any path that starts with the given value (as seen in Ingress)
We have 3 rules, and each rule listens for a specific path prefix:
  - **/** – Matches all paths and routes to **frontend-service:80**
  - **/cart** – Would match /cart and subpaths like **/cart/items**
  - **/orders** – Would match /orders and subpaths like **/orders/123**

> when someone visits **http://shop.techiescamp.com**, **/cart** and **/orders**, this route tells the Gateway to forward the request to the frontend-service on port 80 inside your cluster.

```bash
# Lets deploy it
vagrant@controlplane:~$ kubectl apply -f frontend-httproute.yaml
httproute.gateway.networking.k8s.io/frontend-httproute created

# Validate the httproute
vagrant@controlplane:~$ kubectl get httproute -n frontend
NAME                 HOSTNAMES   AGE
frontend-httproute               22s

# describe the frontend-httproute and get all the details
vagrant@controlplane:~$ kubectl describe httproute frontend-httproute -n frontend
Name:         frontend-httproute
Namespace:    frontend
Labels:       <none>
Annotations:  <none>
API Version:  gateway.networking.k8s.io/v1
Kind:         HTTPRoute
Metadata:
  Creation Timestamp:  2026-08-06T13:21:29Z
  Generation:          1
  Resource Version:    1722840
  UID:                 78729e45-4cc4-4f1a-a6ec-06bf9b32ea1e
Spec:
  Parent Refs:
    Group:  gateway.networking.k8s.io
    Kind:   Gateway
    Name:   frontend-gateway
  Rules:
    Backend Refs:
      Group:
      Kind:    Service
      Name:    frontend-service
      Port:    80
      Weight:  1
    Matches:
      Path:
        Type:   PathPrefix
        Value:  /
    Backend Refs:
      Group:
      Kind:    Service
      Name:    frontend-service
      Port:    80
      Weight:  1
    Matches:
      Path:
        Type:   PathPrefix
        Value:  /cart
    Backend Refs:
      Group:
      Kind:    Service
      Name:    frontend-service
      Port:    80
      Weight:  1
    Matches:
      Path:
        Type:   PathPrefix
        Value:  /orders
# ...
```

### Verify The DNS Access

Now that we have deployed a valid HTTPRoute, let’s verify if the routing works as expected.

As discussed earlier, we don’t have a real DNS name mapped to a load balancer yet. So instead of calling the domain directly, we’ll **simulate the DNS call using curl** with the **NodePort (32000)** exposed by the Gateway Nginx Pod running in the frontend namespace.

We’ll use the Host header with curl to mimic a real DNS request. This simulates what would happen if a real DNS pointed to your cluster.

```bash
vagrant@controlplane:~$ curl -H "Host: shop.techiescamp.com" http://192.168.201.12:32000
<html>
<head><title>308 Permanent Redirect</title></head>
<body>
<center><h1>308 Permanent Redirect</h1></center>
<hr><center>nginx</center>
</body>
</html>
```

- **shop.techiescamp.com** is the hostname defined in the Gateway listener.
- Replace **192.168.201.12** with your worker node’s IP.

There are two ways to create **HTTPRoutes** for these services:

Weh have covered how to create separate Gateways and Routes inside each namespace (api, payment).

---

### Cross Namespace Routing

Now, let's use a **shared Gateway** (like the one in the frontend namespace) and create **cross-namespace HTTPRoutes** that point to services in api and payment.

To enable cross namespace routing using a single gateway, we need to create a Referencegrant object.

This is like giving permission to two external teams (api, payment) to plug into your shared Gateway (frontend-gateway), even though they live in different namespaces.

Without this, their **HTTPRoute** objects would be ignored by the Gateway, even if the Gateway allows cross-namespace routes.

So we need to create **ReferenceGrant** allows **HTTPRoutes** from two different namespaces (api and payment) to attach to a Gateway named frontend-gateway that lives in the frontend namespace.

```bash
# Create referencegrants.yaml
apiVersion: gateway.networking.k8s.io/v1beta1
kind: ReferenceGrant
metadata:
  name: allow-api-ns
  namespace: api
spec:
  from:
  - group: gateway.networking.k8s.io
    kind: HTTPRoute
    namespace: frontend
  to:
  - group: ""
    kind: Service
---
apiVersion: gateway.networking.k8s.io/v1beta1
kind: ReferenceGrant
metadata:
  name: allow-payment-ns
  namespace: payment
spec:
  from:
  - group: gateway.networking.k8s.io
    kind: HTTPRoute
    namespace: frontend
  to:
  - group: ""
    kind: Service
```

The above YAML defines two **ReferenceGrant objects**. One in the api namespace and one in the payment namespace. These grants give permission for an HTTPRoute in the frontend namespace to access Services in those namespaces.

- **from block** – Defines who is asking for access (for example, an HTTPRoute in the frontend namespace).
- **to block** – defines what resource is being accessed (for example, a Service in the api or payment namespace where the ReferenceGrant is created).

```bash
# lets deploy it
vagrant@controlplane:~$ kubectl apply -f referencegrants.yaml
referencegrant.gateway.networking.k8s.io/allow-api-ns created
referencegrant.gateway.networking.k8s.io/allow-payment-ns created

# Verify the ReferenceGrants
vagrant@controlplane:~$ kubectl get refgrant -n api
NAME           AGE
allow-api-ns   27s
#
vagrant@controlplane:~$ kubectl get refgrant -n payment
NAME               AGE
allow-payment-ns   32s
```

Describe them to view more details.

```bash
vagrant@controlplane:~$ kubectl describe refgrant allow-api-ns -n api
Name:         allow-api-ns
Namespace:    api
Labels:       <none>
Annotations:  <none>
API Version:  gateway.networking.k8s.io/v1beta1
Kind:         ReferenceGrant
Metadata:
  Creation Timestamp:  2026-08-06T15:20:23Z
  Generation:          1
  Resource Version:    1741576
  UID:                 cb0f3404-de15-499e-9d34-f072a6685b33
Spec:
  From:
    Group:      gateway.networking.k8s.io
    Kind:       HTTPRoute
    Namespace:  frontend
  To:
    Group:
    Kind:   Service
Events:     <none>
```

```bash
vagrant@controlplane:~$ kubectl describe refgrant allow-payment-ns -n payment
Name:         allow-payment-ns
Namespace:    payment
Labels:       <none>
Annotations:  <none>
API Version:  gateway.networking.k8s.io/v1beta1
Kind:         ReferenceGrant
Metadata:
  Creation Timestamp:  2026-08-06T15:20:23Z
  Generation:          1
  Resource Version:    1741577
  UID:                 2eadc117-4cf7-44a7-85ca-104069f72df3
Spec:
  From:
    Group:      gateway.networking.k8s.io
    Kind:       HTTPRoute
    Namespace:  frontend
  To:
    Group:
    Kind:   Service
Events:     <none>
```

### Modify HTTProute

In the current **frontend-httproute.yaml**, we’ve only defined routes for the frontend-service within the frontend namespace.

Now that we’ve created the necessary **ReferenceGrants** in the api and payment namespaces, we can go ahead and add cross-namespace routes for the **api-service** and **payment-service**.

Below is the **fully updated HTTPRoute**. Replace the existing content of **frontend-httproute.yaml** with the following:

```bash
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: frontend-httproute
  namespace: frontend
spec:
  parentRefs:
  - name: frontend-gateway
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /
    backendRefs:
    - name: frontend-service
      port: 80
  - matches:
    - path:
        type: PathPrefix
        value: /cart
    backendRefs:
    - name: frontend-service
      port: 80
  - matches:
    - path:
        type: PathPrefix
        value: /orders
    backendRefs:
    - name: frontend-service
      port: 80
  # API paths (namespace: api) 
  - matches:
    - path:
        type: PathPrefix
        value: /api/inventory
    backendRefs:
    - name: api-service
      namespace: api
      port: 80

  - matches:
    - path:
        type: PathPrefix
        value: /api
    backendRefs:
    - name: api-service
      namespace: api
      port: 80

  # Payment path (namespace: payment)
  - matches:
    - path:
        type: PathPrefix
        value: /payment
    filters:
    - type: URLRewrite
      urlRewrite:
        path:
          type: ReplacePrefixMatch
          replacePrefixMatch: /
    backendRefs:
    - name: payment-service
      namespace: payment
      port: 80
```

Lets update the routes.

```bash
vagrant@controlplane:~$ kubectl apply -f frontend-httproute.yaml
httproute.gateway.networking.k8s.io/frontend-httproute configured
```

Now, let’s verify if we can access the API and Payment endpoints and check if the responses are coming from the correct backend services.

```bash
# Replace 192.168.201.12 with your worker node IP
vagrant@controlplane:~$ curl -H "Host: shop.techiescamp.com" http://192.168.201.12:32000/api
<html>
<head><title>308 Permanent Redirect</title></head>
<body>
<center><h1>308 Permanent Redirect</h1></center>
<hr><center>nginx</center>
</body>
</html>

# 
vagrant@controlplane:~$ curl -H "Host: shop.techiescamp.com" http://192.168.201.12:32000/api/inventory
<html>
<head><title>308 Permanent Redirect</title></head>
<body>
<center><h1>308 Permanent Redirect</h1></center>
<hr><center>nginx</center>
</body>
</html>
```

### Payment URL Rewriting

The **payment deployment** serves its content from the root path (/). So if you access the service directly using its ClusterIP, you’ll see the payment HTML page at **/**.

However, in our **HTTPRoute**, we added a URL rewrite filter so the page can be accessed from **/payment** instead of **/**, as shown below.

```bash
filters:
- type: URLRewrite
  urlRewrite:
    path:
      type: ReplacePrefixMatch
      replacePrefixMatch: /
```

This demonstrates how **URL rewriting** works. We saw a similar setup earlier with Ingress, but the key difference is:

1. In **Gateway API**, URL rewriting is supported natively using filters.
2. In **Ingress**, URL rewriting depends on controller-specific annotations (like NGINX’s rewrite-target).

Lets try accessing the payment url. Replace 10.139.0.6 with your worker node IP.

```bash
vagrant@controlplane:~$ curl -H "Host: shop.techiescamp.com" http://192.168.201.12:32000/payment
<html>
<head><title>308 Permanent Redirect</title></head>
<body>
<center><h1>308 Permanent Redirect</h1></center>
<hr><center>nginx</center>
</body>
</html>
```

---

### Canary Deployments Using HTTPRoute

Gateway API lets you control how traffic is split between services using HTTPRoute weighted routing.

This is commonly used for **canary deployments**, where traffic is gradually shifted from the old version of an app to a new version.

For example, you can start by sending 10% of the traffic to the new version and the remaining 90% to the old one. If everything looks good, the traffic is increased step by step, 25%, 50%, and finally 100% to the new version (Progressive Traffic Increase).

![Gateway API 14]({{ site.baseurl }}/assets/img/k8s-course/gateway-api-14.png)

---

### GatewayAPI TLS

The Gateway API offers several options for configuring TLS to secure traffic to K8s services, much like the Ingress.

In the Gateway API, gateway controllers manage TLS certificates in a way that's similar to how Ingress controllers handle TLS.

There are two main TLS modes in Gateway API.

1. **Terminate:** In this mode, the TLS traffic is decrypted at the Gateway. This allows the Gateway to read headers, paths, and query parameters, so it can route the traffic correctly.
2. **Passthrough:** In this mode, the Gateway does not decrypt the traffic. It simply passes it along to the backend service, which will handle the decryption.

When you add TLS to a Gateway , the **TLS termination** happens at the Gateway controller layer.

> TLS termination refers to the process of decrypting incoming encrypted traffic at a certain point in the network, typically at the load balancer or proxy layer, before it reaches the backend servers (like application servers or pods in K8s).

Just like Ingress, Gateway also uses TLS certificates stored as K8s secrets of type **kubernetes.io/tls**.

![Gateway API 15]({{ site.baseurl }}/assets/img/k8s-course/gateway-api-15.png)

### Full SSL (End-to-End Encryption) to the Pod

If you want full SSL encryption all the way to the pod, you need to use mode: **Passthrough** in the Gateway TLS configuration.

![Gateway API 16]({{ site.baseurl }}/assets/img/k8s-course/gateway-api-16.png)

> A Gateway can have multiple listeners, and each listener can have its own TLS configuration.

---

### Create Gateway TLS

Let's create **TLS** for a gateway API in a practical way using **self-signed TLS** certificates generated with **OpenSSL**.

> We will be using ClusterIP and NodePort along with curl's DNS simulation to mimic how DNS resolution and TLS would function in a production setup.
> In a real-world scenario, this process would be managed through a valid DNS record that points to the IP or DNS name of the load balancer connected to the controller's Kubernetes service.

```bash
# Generate a self-signed key and certificate for shop.techiescamp.com. 
# The command creates two files: tls.crt for the certificate and tls.key for the private key
openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout tls.key -out tls.crt -subj "/CN=shop.techiescamp.com/O=DevOps"

#
vagrant@controlplane:~$ openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout tls.key -out tls.crt -subj "/CN=shop.techiescamp.com/O=DevOps"
...+...........+...+.+..+....+........+..........+...+..+......+++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++*......+..+....+...+.....+..........+..+.+........+......+.+..+.+.........+...+..+............+....+.....+.......+.........+........+................+..+....+......+........+......+.+............+...+............+......+.....+++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++*..........+.......+..+....+.....+............+...+.....................+.........+....+.....+.+...........+...+......+.+...........+..................+.+......+...+..+........................+....+..+.......+++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++
..+...+........................+.+..+...............+..................+.+...+..+.........+.+......+...+..+....+...+........+.............+..+.......+...............+....................+.+.....+++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++*.....+...+......+......+.......+...+...+...........+...+.............+..+.+............+..+++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++*......+.+.........+.........+...+..+...+.............+++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++
-----
```

Now, using the certificate and private key, we'll create a Kubernetes TLS secret named frontend-tls in the frontend namespace with the following command.

```bash
vagrant@controlplane:~$ kubectl create secret tls frontend-tls --cert=tls.crt --key=tls.key -n frontend
error: failed to create secret secrets "frontend-tls" already exists
```

To make the TLS configuration work, the protocol should be set to HTTPS and use port 443. You'll also need to reference the TLS secret in the TLS section as shown below.

```bash
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: frontend-gateway
  namespace: frontend
spec:
  gatewayClassName: nginx-gateway-class
  listeners:
  - name: http
    protocol: HTTPS
    port: 443
    hostname: "shop.techiescamp.com"
    tls:
      mode: Terminate
      certificateRefs:
      - kind: Secret
        group: ""
        name: frontend-tls
```

The above YAML has the following key TLS configurations under the tls spec.

- Protocol: HTTPS on port 443
- Hostname: Only accepts traffic for shop.techiescamp.com
- TLS Mode: Terminate - the gateway will decrypt TLS traffic

```bash
# Now that the configuration is ready, lets update the existing Gateway
vagrant@controlplane:~$ kubectl apply -f frontend-gateway.yaml
gateway.gateway.networking.k8s.io/frontend-gateway configured
```

Describe the Gateway resource to confirm that the TLS configurations have been correctly applied. You should see output similar to the example below, which shows the TLS section of the spec.

```bash
vagrant@controlplane:~$ kubectl describe gateway frontend-gateway -n frontend
Name:         frontend-gateway
Namespace:    frontend
# ...
Spec:
  Gateway Class Name:  nginx-gateway-class
  Listeners:
    Allowed Routes:
      Namespaces:
        From:  Same
    Hostname:  shop.techiescamp.com
    Name:      http
    Port:      443
    Protocol:  HTTPS
    Tls:
      Certificate Refs:
        Group:
        Kind:   Secret
        Name:   frontend-tls
      Mode:     Terminate
# ...
```

### Validating TLS With ClusterIP

Run the following command to retrieve the ClusterIP of the gateway controller service:

```bash
vagrant@controlplane:~$ kubectl get svc ngf-nginx-gateway-fabric  -n nginx-gateway -o jsonpath='{.spec.clusterIP}{"\n"}'
10.105.253.5

# or
vagrant@controlplane:~$ kubectl get svc -n nginx-gateway
NAME                       TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE
ngf-nginx-gateway-fabric   ClusterIP   10.105.253.5   <none>        443/TCP   7h1m
```

Now we’ll simulate TLS traffic using curl and the ClusterIP with the following command. Replace **10.105.253.5** with your ClusterIP.

```bash
vagrant@controlplane:~$ curl -v -k --resolve shop.techiescamp.com:443:10.105.253.5 https://shop.techiescamp.com
* Added shop.techiescamp.com:443:10.105.253.5 to DNS cache
* Hostname shop.techiescamp.com was found in DNS cache
*   Trying 10.105.253.5:443...
* Connected to shop.techiescamp.com (10.105.253.5) port 443 (#0)
* ALPN, offering h2
* ALPN, offering http/1.1
* TLSv1.0 (OUT), TLS header, Certificate Status (22):
* TLSv1.3 (OUT), TLS handshake, Client hello (1):
* TLSv1.2 (IN), TLS header, Certificate Status (22):
* TLSv1.3 (IN), TLS handshake, Server hello (2):
* TLSv1.2 (IN), TLS header, Finished (20):
* TLSv1.2 (IN), TLS header, Supplemental data (23):
* TLSv1.3 (IN), TLS handshake, Encrypted Extensions (8):
* TLSv1.2 (IN), TLS header, Supplemental data (23):
* TLSv1.3 (IN), TLS handshake, Request CERT (13):
* TLSv1.2 (IN), TLS header, Supplemental data (23):
* TLSv1.3 (IN), TLS handshake, Certificate (11):
* TLSv1.2 (IN), TLS header, Supplemental data (23):
* TLSv1.3 (IN), TLS handshake, CERT verify (15):
* TLSv1.2 (IN), TLS header, Supplemental data (23):
* TLSv1.3 (IN), TLS handshake, Finished (20):
* TLSv1.2 (OUT), TLS header, Finished (20):
* TLSv1.3 (OUT), TLS change cipher, Change cipher spec (1):
* TLSv1.2 (OUT), TLS header, Supplemental data (23):
* TLSv1.3 (OUT), TLS handshake, Certificate (11):
* TLSv1.2 (OUT), TLS header, Supplemental data (23):
* TLSv1.3 (OUT), TLS handshake, Finished (20):
* SSL connection using TLSv1.3 / TLS_AES_128_GCM_SHA256
* ALPN, server accepted to use h2
* Server certificate:
*  subject: C=US; L=SEA; O=F5; OU=NGINX; CN=nginx-gateway
*  start date: Aug  6 12:01:21 2026 GMT
*  expire date: Aug  5 12:01:21 2029 GMT
*  issuer: C=US; L=SEA; O=F5; OU=NGINX; CN=nginx-gateway
*  SSL certificate verify result: self-signed certificate (18), continuing anyway.
* Using HTTP2, server supports multiplexing
* Connection state changed (HTTP/2 confirmed)
* Copying HTTP/2 data in stream buffer to connection buffer after upgrade: len=0
* TLSv1.2 (OUT), TLS header, Supplemental data (23):
* TLSv1.2 (OUT), TLS header, Supplemental data (23):
* TLSv1.2 (OUT), TLS header, Supplemental data (23):
* Using Stream ID: 1 (easy handle 0x56089d536e90)
* TLSv1.2 (OUT), TLS header, Supplemental data (23):
> GET / HTTP/2
> Host: shop.techiescamp.com
> user-agent: curl/7.81.0
> accept: */*
>
* TLSv1.2 (IN), TLS header, Supplemental data (23):
* TLSv1.3 (IN), TLS alert, unknown (628):
* OpenSSL SSL_read: error:0A00045C:SSL routines::tlsv13 alert certificate required, errno 0
* Failed receiving HTTP2 data
* OpenSSL SSL_write: SSL_ERROR_ZERO_RETURN, errno 0
* Failed sending HTTP2 data
* Connection #0 to host shop.techiescamp.com left intact
curl: (56) OpenSSL SSL_read: error:0A00045C:SSL routines::tlsv13 alert certificate required, errno 0
```

### Validating TLS With NodePort

When we deployed the gateway controller, we mapped port 80 to NodePort 32000 and port 443 to NodePort 32443. So, we can also use port 32443 to validate TLS traffic via NodePort.

```bash
vagrant@controlplane:~$ curl -v -k --resolve shop.techiescamp.com:32443:192.168.201.12 https://shop.techiescamp.com:3244
3
* Added shop.techiescamp.com:32443:192.168.201.12 to DNS cache
* Hostname shop.techiescamp.com was found in DNS cache
*   Trying 192.168.201.12:32443...
* Connected to shop.techiescamp.com (192.168.201.12) port 32443 (#0)
* ALPN, offering h2
* ALPN, offering http/1.1
* TLSv1.0 (OUT), TLS header, Certificate Status (22):
* TLSv1.3 (OUT), TLS handshake, Client hello (1):
* TLSv1.2 (IN), TLS header, Certificate Status (22):
* TLSv1.3 (IN), TLS handshake, Server hello (2):
* TLSv1.2 (IN), TLS header, Finished (20):
* TLSv1.2 (IN), TLS header, Supplemental data (23):
* TLSv1.3 (IN), TLS handshake, Encrypted Extensions (8):
* TLSv1.2 (IN), TLS header, Supplemental data (23):
* TLSv1.3 (IN), TLS handshake, Certificate (11):
* TLSv1.2 (IN), TLS header, Supplemental data (23):
* TLSv1.3 (IN), TLS handshake, CERT verify (15):
* TLSv1.2 (IN), TLS header, Supplemental data (23):
* TLSv1.3 (IN), TLS handshake, Finished (20):
* TLSv1.2 (OUT), TLS header, Finished (20):
* TLSv1.3 (OUT), TLS change cipher, Change cipher spec (1):
* TLSv1.2 (OUT), TLS header, Supplemental data (23):
* TLSv1.3 (OUT), TLS handshake, Finished (20):
* SSL connection using TLSv1.3 / TLS_AES_256_GCM_SHA384
* ALPN, server accepted to use h2
* Server certificate:
*  subject: CN=shop.techiescamp.com; O=techiescamp
*  start date: Aug  5 05:20:06 2026 GMT
*  expire date: Aug  5 05:20:06 2027 GMT
*  issuer: CN=shop.techiescamp.com; O=techiescamp
*  SSL certificate verify result: self-signed certificate (18), continuing anyway.
* Using HTTP2, server supports multiplexing
* Connection state changed (HTTP/2 confirmed)
* Copying HTTP/2 data in stream buffer to connection buffer after upgrade: len=0
* TLSv1.2 (OUT), TLS header, Supplemental data (23):
* TLSv1.2 (OUT), TLS header, Supplemental data (23):
* TLSv1.2 (OUT), TLS header, Supplemental data (23):
* Using Stream ID: 1 (easy handle 0x555bb6488e90)
* TLSv1.2 (OUT), TLS header, Supplemental data (23):
> GET / HTTP/2
> Host: shop.techiescamp.com:32443
> user-agent: curl/7.81.0
> accept: */*
>
* TLSv1.2 (IN), TLS header, Supplemental data (23):
* TLSv1.3 (IN), TLS handshake, Newsession Ticket (4):
* TLSv1.2 (IN), TLS header, Supplemental data (23):
* TLSv1.3 (IN), TLS handshake, Newsession Ticket (4):
* old SSL session ID is stale, removing
* TLSv1.2 (IN), TLS header, Supplemental data (23):
* Connection state changed (MAX_CONCURRENT_STREAMS == 128)!
* TLSv1.2 (OUT), TLS header, Supplemental data (23):
* TLSv1.2 (IN), TLS header, Supplemental data (23):
< HTTP/2 200
< server: nginx
< date: Thu, 06 Aug 2026 19:06:32 GMT
< content-type: text/html
< content-length: 86
< last-modified: Tue, 04 Aug 2026 16:12:48 GMT
< etag: "6a720f80-56"
< accept-ranges: bytes
<
<!DOCTYPE html>
<html>
<body>
<h1>Hey! I am the website frontend</h1>
</body>
</html>
* Connection #0 to host shop.techiescamp.com left intact
```

---

### Cleanup

Now, let's clean up some of the resources to tidy up the cluster and make sure you can run the scenarios without any issues.

If you don't clean them up, the gateways in the scenarios will conflict with the ones already running.

```bash
# delete the demo apps, their namespace, and the app-ns namespace
vagrant@controlplane:~$ kubectl delete -f https://raw.githubusercontent.com/techiescamp/cka-certification-guide/main/ingress/deployments.yaml
namespace "frontend" deleted
namespace "api" deleted
namespace "payment" deleted
configmap "nginx-config" deleted from frontend namespace
configmap "nginx-config" deleted from api namespace
configmap "nginx-config" deleted from payment namespace
deployment.apps "frontend-deployment" deleted from frontend namespace
service "frontend-service" deleted from frontend namespace
deployment.apps "api-deployment" deleted from api namespace
service "api-service" deleted from api namespace
deployment.apps "payment-deployment" deleted from payment namespace
service "payment-service" deleted from payment namespace
```
