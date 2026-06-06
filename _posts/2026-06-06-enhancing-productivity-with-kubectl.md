---
layout: post
title: "Enhancing Productivity with kubectl"
date: 2026-06-06
categories: [k8s]
---

## Kubectl Shortnames

We can create an alias for **kubectl** and use object short names for all the imperative commands.

The following command shows only object names that have shortnames.

```bash
vagrant@controlplane:~$ kubectl api-resources --no-headers | awk '$2 != "" && $2 !~ /^[a-z]+\.[a-z]+/ && $2 != "v1" {print $1 "\t" $2}' | column -t
componentstatuses                cs
configmaps                       cm
endpoints                        ep
events                           ev
limitranges                      limits
namespaces                       ns
nodes                            no
persistentvolumeclaims           pvc
persistentvolumes                pv
pods                             po
replicationcontrollers           rc
resourcequotas                   quota
serviceaccounts                  sa
services                         svc
customresourcedefinitions        crd,crds
controllerrevisions              apps/v1
daemonsets                       ds
deployments                      deploy
replicasets                      rs
statefulsets                     sts
horizontalpodautoscalers         hpa
cronjobs                         cj
jobs                             batch/v1
certificatesigningrequests       csr
events                           ev
ingresses                        ing
ipaddresses                      ip
networkpolicies                  netpol
poddisruptionbudgets             pdb
adminnetworkpolicies             anp
baselineadminnetworkpolicies     banp
bgpconfigurations                bgpconfig,bgpconfigs
blockaffinities                  blockaffinity,affinity,affinities
caliconodestatuses               caliconodestatus
clusterinformations              clusterinfo
felixconfigurations              felixconfig,felixconfigs
globalnetworkpolicies            gnp,cgnp,calicoglobalnetworkpolicies
hostendpoints                    hep,heps
ipamconfigurations               ipamconfig
kubecontrollersconfigurations    kcconfig
networkpolicies                  cnp,caliconetworkpolicy,caliconetworkpolicies
networksets                      netsets
stagedglobalnetworkpolicies      sgnp
stagedkubernetesnetworkpolicies  sknp
stagednetworkpolicies            snp
priorityclasses                  pc
storageclasses                   sc
volumeattributesclasses          vac
```

### Creating Kubectl Aliases

You can add the following alias to your **.bashrc** or **.zshrc** file, depending on which shell you are using.

**For bash shell users**: Open your **.bashrc** file by running.

```bash
vagrant@controlplane:~$ nano ~/.bashrc
```

Then, add the following line at the end of the **.bashrc** file.

```bash
alias k='kubectl'
```

Apply the changes to the **.bashrc**:

```bash
vagrant@controlplane:~$ source ~/.bashrc
```

Now you can use you **kubectl** alias:

```bash
vagrant@controlplane:~$ k get nodes
NAME           STATUS   ROLES           AGE    VERSION
controlplane   Ready    control-plane   2d3h   v1.34.0
node01         Ready    worker          2d2h   v1.34.0
node02         Ready    worker          2d2h   v1.34.0
```

To not make things complicated, don't create other aliases. However, you can create like such:

```bash
alias k='kubectl'
alias kgp='kubectl get pods'
alias kdr='kubectl --dry-run=client'
alias kgs='kubectl get svc'
alias kdesc='kubectl describe'
alias kga='kubectl get all'
```

### kubectl explain

**kubectl explain** is a built‑in documentation command in K8s.

- Explains the fields, types, and usage of resources.
- Helps you understand what each field in a manifest means.
- Acts like a manual page for Kubernetes resources.

```bash
vagrant@controlplane:~$ kubectl explain pod
KIND:       Pod
VERSION:    v1

DESCRIPTION:
    Pod is a collection of containers that can run on a host. This resource is
    created by clients and scheduled onto hosts.

FIELDS:
  apiVersion    <string>
    APIVersion defines the versioned schema of this representation of an object.
    Servers should convert recognized schemas to the latest internal value, and
    may reject unrecognized values. More info:
    https://git.k8s.io/community/contributors/devel/sig-architecture/api-conventions.md#resources

  kind  <string>
    Kind is a string value representing the REST resource this object
    represents. Servers may infer this from the endpoint the client submits
    requests to. Cannot be updated. In CamelCase. More info:
    https://git.k8s.io/community/contributors/devel/sig-architecture/api-conventions.md#types-kinds

  metadata      <ObjectMeta>
    Standard object's metadata. More info:
    https://git.k8s.io/community/contributors/devel/sig-architecture/api-conventions.md#metadata

  spec  <PodSpec>
    Specification of the desired behavior of the pod. More info:
    https://git.k8s.io/community/contributors/devel/sig-architecture/api-conventions.md#spec-and-status

  status        <PodStatus>
    Most recently observed status of the pod. This data may not be up to date.
    Populated by the system. Read-only. More info:
    https://git.k8s.io/community/contributors/devel/sig-architecture/api-conventions.md#spec-and-status
```

The following commands list all the key parameters that comes under each fields and sub-fields.

```bash
kubectl explain pods.spec
kubectl explain pods.spec.volumes
k explain pods.spec.containers.resources
```

If you forgot how to configure a container's resources, use:

```bash
vagrant@controlplane:~$ kubectl explain pods.spec.containers.resources
KIND:       Pod
VERSION:    v1

FIELD: resources <ResourceRequirements>

DESCRIPTION:
    Compute Resources required by this container. Cannot be updated. More info:
    https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/
    ResourceRequirements describes the compute resource requirements.

FIELDS:
  claims        <[]ResourceClaim>
    Claims lists the names of resources, defined in spec.resourceClaims, that
    are used by this container.

    This field depends on the DynamicResourceAllocation feature gate.

    This field is immutable. It can only be set for containers.

  limits        <map[string]Quantity>
    Limits describes the maximum amount of compute resources allowed. More info:
    https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/

  requests      <map[string]Quantity>
    Requests describes the minimum amount of compute resources required. If
    Requests is omitted for a container, it defaults to Limits if that is
    explicitly specified, otherwise to an implementation-defined value. Requests
    cannot exceed Limits. More info:
    https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/
```

> Use **--recursive** combined with **grep** to find a field name when you only remember part of it.

For example, if you want to find where **hostPath** goes in a Pod.

```bash
vagrant@controlplane:~$ k explain pods --recursive | grep -i "hostPath"
      hostPath  <HostPathVolumeSource>
```

**This shows clearly**: **hostPath** is a field, and **HostPathVolumeSource** is its type definition.

```bash
apiVersion: v1
kind: Pod
metadata:
  name: hostpath-demo
spec:
  containers:
  - name: test-container
    image: nginx:alpine
    volumeMounts:
    - name: test-volume
      mountPath: /usr/share/nginx/html
  volumes:
  - name: test-volume
    hostPath:
      path: /data/html       # <-- required field
      type: DirectoryOrCreate # <-- optional field
```

- volumes[].hostPath → the field name in the Pod spec.
- HostPathVolumeSource → the object type that defines what goes inside hostPath.
  - path: the directory/file on the host.
  - type: optional, specifies behavior (e.g., Directory, File, DirectoryOrCreate).

The **--recursive** flag only flattens the schema into field names + types, so you lose the hierarchy.

### Finding Indentation Levels

The **less** flag allows you to scroll and see the indentation levels clearly.

```bash
kubectl explain pods --recursive | less
```

Once inside the pods documentation, you can search for any field to see the hierachy.

```bash
:/volumes
:/podAffinity
```
