---
layout: post
title: "Docker Course Notes"
date: 2026-03-19
categories: [docker]
---

## Introduction

Containers use operating system kernel, which is the core part of the OS that manages resources like CPU, memory, and networking.

### User Space and Kernel Space

### User Space

- The environment where user facing applications run. For example, Web servers, Chrome, Text editors, command utilities etc.
- It acts like a restricted zone because applications cannot directly access hardware or manage system resources on their own.
- That is why if one application crashes, it usually does not crash the whole system.
- A program in user space does not have a direct access to system memory, cpu and hardware resources.

### Kernel Space

- This is where the core of the operating system, known as the kernel, runs.
- The kernel has full access to the system’s memory and hardware resources.
- Programs in userspace must go through the kernel. This interaction happens through system calls (syscalls).
- The kernel acts as a middleman. It ensures that the apps use system resources safely without affecting the whole system.

![User space vs Kernel space]({{ site.baseurl }}/assets/img/docker-course/user-space-vs-kernel-space.gif)

> 💡 System calls are part of the kernel. They are implemented as functions in the kernel space, and user programs can access them using specific instructions or libraries (like the C library, glibc).

### Real World Example: The **ls** Command

- **ls command** is a userspace utility that lists directory contents.
- Every time you type ls to list files, your system is secretly making dozens of syscalls to communicate with the Kernel.

When you run the **ls command** to list files, the following happens:

- The ls program starts running in user space with limited privileges.
- Since ls cannot directly access the filesystem, it requests help from the kernel through system calls like **opendir()** and **readdir()**.
- The kernel receives these requests and performs the privileged operations like reading the actual directory structure from the disk, checking your permissions, and gathering file metadata.
- The kernel packages the directory information and safely returns it to the **ls program**.
- The **ls program** formats and displays the results in your terminal.

![User space vs Kernel space ls example]({{ site.baseurl }}/assets/img/docker-course/user-space-vs-kernel-space-example.gif)

---

## Understanding Linux Process

A process is simply a running program.
When a process runs, it often needs to divide its work into smaller tasks. it creates child processes to manage different parts of the job.
The **parent process** (master) loads configuration, manages workers, and keeps the server running.
The **child processes** (workers) handle the actual web requests coming from users. It runs as a normal user (like nginx or **www-data**) to reduce risk if they are hacked.

> The www-data user in Linux is a special system user account under which web server processes (like Apache or Nginx) run by default. Its purpose is to run the web server with restricted privileges for security reasons.

### The Linux Process Model

All processes on a Linux system are child processes of a common parent: the **init** process, which is executed by the kernel at boot time and starts other processes. Because all processes descend from a single parent, the Linux process model is a single hierarchy, or **tree**.

### How Isolated Is a Process?

- Every process gets its own PID (process ID) so the system can track it.
- Each process has its own memory space, so it cannot directly touch another process’s memory.
- **Control groups** limit and track resources like CPU, memory, and network. This way one process (or group of processes) can’t take over all the system resources.

> Linux Control Groups (cgroups) are a Linux kernel feature that organizes processes into hierarchical groups to limit, account for, and isolate resource usage (CPU, memory, disk I/O, network).

---

### Isolating Processess

Imagine if we could take this idea of isolation of processes further.

- What if we could give a process only the files, libraries, and configurations it needs?
- What if we could package it in such a way that it doesn’t interfere with other processes on the system?
- And what if we could take this package and run it on any computer, and it would still work exactly the same way?

That is the idea of a **container**.

A container is like putting your process (and its child processes) inside a box. The box holds everything the process needs. Nothing more, nothing less.

> In short, a container is a standardized execution environment like a restricted box for running one or more processes.

By using containers, we gain three powerful benefits:

- **Security**: The process is separated from others, reducing risk.
- **Consistency**: The process runs the same way, no matter which system you put it on.
- **Portability**: You can move the container across laptops, servers, or clouds, and it will still behave the same.

![Visual Idea of a Container]({{ site.baseurl }}/assets/img/docker-course/container.png)

---

### How Containers Stay Portable Across Systems?

All Linux distributions (Red Hat, Ubuntu, Debian, etc.) share the same Linux kernel at the core. What makes them different are the user-space tools and libraries (things like /bin, /lib, and package managers).

As long as the host system has a Linux kernel, the container can run.

---

### How does Containers Actually Stay Isolated from one another on the same Machine?

The answer lies in the following two powerful features of the Linux kernel.

- Namespaces
- Control Groups (cgroups)

A good way to think about this is by imagining an apartment building.

An apartment building (Linux Machine) consists of many apartments (containers), they all share the same building infrastructure such as foundations, electricity, plumbing, etc, (Linux Kernel). However, each apartment has its own walls, door (file system, libraries, CPU, Memory etc) which isolate each single apartment from other apartments.

This isloation is made possible by Linux namespaces which give each container its own processes, network, mounts etc.
On the other hand, cgroups control how much CPU, memory, and other resources each container can use.

![Namespaces & cgroups]({{ site.baseurl }}/assets/img/docker-course/namespace-cgroups.png)

---

### Linux Namespaces

It makes process isolation possible, so that multiple applications could run securely and independently without affecting each other.

Namespaces are the building blocks for container technology. Think of namespaces as drawing boundaries around containers. Namespaces make sure that processes inside a container can only see and use their onw resources.

### Key Linux Namespaces

The most common Linux namespaces are as follow:

- PID namespace (PID): Isolates process IDs. Each container can have its own PID 1.
- Network namespace (NET): Gives each container its own network stack (interfaces, routing tables, IP addresses).
- IPC namespace (IPC): This manages access to interprocess communication (IPC) resources.
- Mount namespace (MNT): Provides each container with its own filesystem view.
- UTS namespace (UTS): Lets containers have their own hostname and domain name.
- User namespace (USR): Maps user IDs inside a container to different IDs on the host (improving security).
- Control Group namespace (cgroup): This namespace isolates control group information from the container process.

Each container gets its own namespaces, and using these namespaces, a container can have its own network interfaces, IP address, and more.

![Container Namespaces]({{ site.baseurl }}/assets/img/docker-course/container-namespaces.png)

---

##  Linux Network Namespaces

A network namespace is like a separate networking world inside your host. Each namespace has its own interfaces, routing table, and ARP table.

```bash
sudo ip netns add ns1 # create a network namespace ns1
sudo ip netns add ns2 # create a network namespace ns2

ip netns list # list the created namesapces
```

## Create a Virtual Ethernet (veth) Pair

A veth pair works like a cable. Anything sent into one end comes out of the other. We will attach each end to a different namespace.

> A veth pair is essentially a virtual Ethernet cable: two ends that are always connected to each other. 

```bash
sudo ip link add veth1 type veth peer name veth2 # creates two interfaces (veth1 and veth2) that are directly connected.
ip link show veth1 # ifconfig -a veth1
ip link show veth2 # ifconfig -a veth2
```

### Why we need veth pair

- **Connecting namespaces**: Each end of the veth pair can be placed into a different network namespace. That way, processes running in ns1 can talk to processes in ns2 through this virtual link.
- **Building network topologies**: Veth pairs are the building blocks for simulating routers, bridges, or containers. For example, you can connect a namespace to a Linux bridge, then connect that bridge to another namespace or to the host.
- **Container Networking**: Docker and K8s rely on veth pairs under the hood. One end stays in the container’s namespace, the other in the host namespace, often attached to a bridge (like docker0).

### Assign veth Interfaces to Namespaces

Now we have to move our network interfaces (veth1 & veth2) into the desired namespaces.

```bash
sudo ip link set veth1 netns ns1 # moves the network interface veth1 into the network namespace ns1.
sudo ip link set veth2 netns ns2 # moves the network interface veth2 into the network namespace ns2.
```

Each namespace has its own "network card."

```bash
sudo ip netns exec ns1 ip link #  lets you inspect what’s inside that namespace ns1.
sudo ip netns exec ns2 ip link # lets you inspect what’s inside that namespace ns2.
```

### Assign IP Addresses

Assigning IP addresses lets the namespaces talk over Layer 3 (IP). Both are placed in the same subnet 10.0.0.0/24 .

```bash
sudo ip netns exec ns1 ip addr add 10.0.0.1/24 dev veth1
sudo ip netns exec ns2 ip addr add 10.0.0.2/24 dev veth2
```

- **ip netns exec ns1** → runs the command inside the namespace ns1.
- **ip addr add 10.0.0.1/24 dev veth1** → assigns the IP address **10.0.0.1** with a subnet mask **/24 (255.255.255.0)** to the interface veth1.

### Bring Interfaces Up

Interfaces are down by default. Bringing them up is necessary for communication. Loopback (lo ) is also enabled because many tools expect it.

```bash
sudo ip netns exec ns1 ip link set lo up
sudo ip netns exec ns1 ip link set veth1 up # 

sudo ip netns exec ns2 ip link set lo up
sudo ip netns exec ns2 ip link set veth2 up
```

At this point, veth1 in ns1 can communicate with its peer (veth2 in ns2) once you assign an IP and bring that one up too.

### Verify Routes

The kernel automatically installs a route for each subnet when IPs are assigned. This shows both namespaces know how to reach each other.

```bash
sudo ip netns exec ns1 ip route # verifies what routes exist inside ns1 
sudo ip netns exec ns2 ip route # verifies what routes exist inside ns2
```

### Test Connectivity

The following command checks whether the interface in ns1 (with IP 10.0.0.1) can reach the peer interface in ns2 (with IP 10.0.0.2) through the veth pair.

```bash
sudo ip netns exec ns1 ping -c 3 10.0.0.2
```

- **ip netns exec ns1** → runs the command inside the namespace ns1.
- **ping -c 3 10.0.0.2** → sends 3 ICMP echo requests to the IP address 10.0.0.2.

---

## Linux Control Groups

When a service starts, the Kernel does the following:

- Allocate memory and CPU time as needed.
- Share resources fairly between processes (with priorities, scheduling, etc.).

How do we make sure one service or container doesn’t take over all the CPU & Memory resources? 

The answer is Control Groups (**cgroups**).

> Linux cgroups (control groups) are a kernel feature that ensure services don’t consume more than their fair share.

### What cgroups does?

- **Organize processes hierarchically**: Processes are grouped into “control groups” that can be managed together.
- **Apply resource limits**: You can restrict how much CPU time, memory, or bandwidth a group of processes can use.
- **Prioritize workloads**: Critical services can be guaranteed resources, while background tasks get less.

### cgroups in Containers

Cgroups are a core building block of container technology. With cgroups, you can limit how much CPU, memory, network, or disk I/O a container can use.

![cgroups]({{ site.baseurl }}/assets/img/docker-course/cgroups.png)

---

### LAB: Understanding Linux Control Groups

In this lab, we will learn how Linux control groups (cgroups) work by creating and applying resource limits to a process.

- Create a new cgroup (labgroup)
- Set CPU and memory limits for the cgroup
- Run a Python process inside the cgroup
- Observe how the process behaves when it hits CPU throttling or memory OOM limits

![cgroup example]({{ site.baseurl }}/assets/img/docker-course/cgroup-example.png)

> cgroups form the foundation for container resource management in systems like Docker and K8s.

### IStep 1: Install Required Tools

```bash
sudo apt update
sudo apt install -y cgroup-tools
```

cgroup-tools (libcgroup) provides commands like **cgcreate , cgexec , and cgset** for managing cgroups.
**stress-ng** is useful for simulating CPU and memory load.

```bash
sudo apt update
sudo apt install -y stress-ng
```

### Step 2: Create a Cgroup

```bash
sudo cgcreate -g cpu,memory:/labgroup # 
```

- **cgcreate** → creates a new control group (cgroup).
- **-g cpu,memory:/labgroup** → specifies the controllers (cpu and memory) and the path (/labgroup).

1. We have created a cgroup named **labgroup** that can manage CPU and memory resources for any processes you assign to it.

2. A directory **/sys/fs/cgroup/labgroup/** will now exist to track and control resources.

Lets list the contents of the directory **/sys/fs/cgroup/labgroup/**

```bash
zafar@zserver:~$ ls /sys/fs/cgroup/labgroup/
cgroup.controllers               cpuset.cpus.partition     hugetlb.2MB.numa_stat     memory.stat
cgroup.events                    cpuset.mems               hugetlb.2MB.rsvd.current  memory.swap.current
cgroup.freeze                    cpuset.mems.effective     hugetlb.2MB.rsvd.max      memory.swap.events
cgroup.kill                      cpu.stat                  io.max                    memory.swap.high
cgroup.max.depth                 cpu.stat.local            io.pressure               memory.swap.max
cgroup.max.descendants           cpu.uclamp.max            io.prio.class             memory.swap.peak
cgroup.pressure                  cpu.uclamp.min            io.stat                   memory.zswap.current
cgroup.procs                     cpu.weight                io.weight                 memory.zswap.max
cgroup.stat                      cpu.weight.nice           memory.current            memory.zswap.writeback
cgroup.subtree_control           hugetlb.1GB.current       memory.events             misc.current
cgroup.threads                   hugetlb.1GB.events        memory.events.local       misc.events
cgroup.type                      hugetlb.1GB.events.local  memory.high               misc.max
cpu.idle                         hugetlb.1GB.max           memory.low                pids.current
cpu.max                          hugetlb.1GB.numa_stat     memory.max                pids.events
cpu.max.burst                    hugetlb.1GB.rsvd.current  memory.min                pids.max
cpu.pressure                     hugetlb.1GB.rsvd.max      memory.numa_stat          pids.peak
cpuset.cpus                      hugetlb.2MB.current       memory.oom.group          rdma.current
cpuset.cpus.effective            hugetlb.2MB.events        memory.peak               rdma.max
cpuset.cpus.exclusive            hugetlb.2MB.events.local  memory.pressure
cpuset.cpus.exclusive.effective  hugetlb.2MB.max           memory.reclaim
```

Now we can do the following with our cgroup, labgroup:

- Add processes to the group (e.g., **sudo cgclassify -g cpu,memory:/labgroup <PID>**).
- Set limits (e.g., **sudo cgset -r memory.limit_in_bytes=256M labgroup**).
- Monitor usage (e.g., **sudo cgget -g cpu,memory:/labgroup**).

### Step 3: Apply CPU Limit

```bash
sudo cgset -r cpu.max="20000 100000" labgroup # PERIOD = 100000 µs (100 ms) and QUOTA = 20000 µs (20 ms)
```

This means processes inside labgroup can only run for 20 ms every 100 ms, which is ~20% of a single CPU core.

If a process tries to use more, the kernel throttles it until the next scheduling window.

### Step 4: Apply Memory Limit

```bash
sudo cgset -r memory.max=100M labgroup # restricts processes in labgroup to 100 MB of RAM.
```

If memory usage goes beyond this, the kernel’s OOM (Out-of-Memory) killer terminates the process.

Lets verify the limits:

```bash
cat /sys/fs/cgroup/labgroup/memory.max
cat /sys/fs/cgroup/labgroup/cpu.max
```

### Step 5: Run a Test Process

Let’s run a memory-hungry Python script inside the cgroup:

```python
sudo cgexec -g cpu,memory:labgroup python3 - <<'PY'
import time
chunks = [bytearray(1024*1024) for _ in range(200)]  # Allocate ~200 MB
time.sleep(20)
PY
```

Since we limited memory to **100 MB**, this script (200 MB allocation) should trigger the **OOM killer**.

You’ll likely see:

```bash
Killed
```

### Step 6: Observe Memory Usage

Check current memory consumption and events:

```bash
cat /sys/fs/cgroup/labgroup/memory.current
cat /sys/fs/cgroup/labgroup/memory.events
```

### Step 7: Test CPU Throttling

Run a CPU-intensive process inside the cgroup:

```bash
sudo cgexec -g cpu,memory:labgroup bash -c 'timeout 10s sh -c "while :; do :; done"'
```

This infinite loop tries to consume 100% CPU for 10 seconds.
Because of the CPU limit (20% of one core ), it will be throttled.

Check the stats:

```bash
zafar@zserver:~$ cat /sys/fs/cgroup/labgroup/cpu.stat
usage_usec 2172010
user_usec 2044186
system_usec 127823
core_sched.force_idle_usec 0
nr_periods 119
nr_throttled 107
throttled_usec 8411896
nr_bursts 0
burst_usec 0
```

- **nr_throttled** : How many times processes were throttled.
- **throttled_usec** : Total time processes spent waiting due to CPU limits.

---

## Why Are Containers Better Than VMs?

### 1. 💰 Resource Utilization & Cost

Each VM runs a full operating system, which means more CPU and memory usage. Also, resizing a VM in production is not always smooth.

Containers are lightweight. They only package what is needed. The process and its dependencies. This means,

- You can run many containers inside a single VM.
- Scaling a container takes just seconds.

### 2. 🛠️ Provisioning & Deployment

Launching a VM, installing software, and configuring it can take minutes or even hours. Rolling back is also slow.

Containers start in seconds. Rolling back is as simple as pulling the previous image.

### 3. ⚙️ Drift Management

VMs often face configuration drift. Meaning, differences between dev, test, and prod environments.

Containers solve this in a simple manner. Just update the container image with changes.

---

## Docker Basics

Docker allows to pack your application along with everything it needs (libraries, configurations, and dependencies). This “container” can then run on any system that has Docker installed.

> Docker is “an open-source project to pack, ship and run any application as a lightweight container.”

Docker solves real problems for developers and system admins:

- **Isolation**: Applications run in their own containers. They do not interfere with each other, even if they use different libraries or dependencies.
- **Speed**: Moving from development to production is much faster. You can start, stop, and update applications quickly.
- **Consistency**: No matter where you run the container, it behaves the same way. This removes the classic “it worked on my machine” problem.

### Docker’s Modular Architecture

Docker is built on four main parts. These parts work together to manage and run containers.

### 1. dockerd

- dockerd is the Docker Daemon. It is the main background service that runs whenever Docker is installed on your system.
- Think of it as the brain of Docker. It listens for requests, makes decisions, and manages the whole lifecycle of containers.
- The logic inside dockerd decides what to do with those requests. When required, dockerd passes tasks down to other components like containerd and runc to actually create and run containers.

### 2. containerd

- A container runtime is the software responsible for actually running and managing containers, containerd is such a runtime.
- It is a lightweight background service (daemon) that is responsible for the lifecycle of containers. dockerd is the manager and containerd is the worker that does the actual heavy lifting.

When dockerd receives the API request to create containers, dockerd understands the request and prepares the plan. dockerd then asks containerd to handle the container lifecycle by passing the container configuration and metadata.

![containerd architecture]({{ site.baseurl }}/assets/img/docker-course/containerd.png)

### 3. runc

So far, we have seen that,

- dockerd is the manager that takes your commands.
- containerd is the worker that manages the container lifecycle.

> runc actually creates the container process inside Linux?

- runc is a low-level container runtime. It is a lightweight command-line tool, not a background service.
- Its only job is to create the isolated environment for a single container process.

When container-shim asks it to start a container, here is what runc does.

- It sets up Linux namespaces to isolate processes, networking, and filesystems.
- Applies cgroups to control CPU, memory, and other resources.
- Starts the actual container process inside this isolated environment.
- Finally, it exits.

### 4. containerd-shim

When runc starts a container, it immediately exits. But the container process should keep running. So who takes care of it?

That is the job of **containerd-shim**.

> A shim is a small lightweight process that stays alive for each running container after runc exits.

Overall here is what **shim** does,

- It keeps the container alive even though runc has exited.
- It handles I/O streams (stdin, stdout, stderr) so you can see logs or attach to the container.
- It reports status of the container back to containerd.
- It cleans up resources when the container stops, so there are no zombie processes.

## What is Docker Engine?

It is the combination of dockerd + containerd + runc + containerd-shim working together.

![containerd architecture]({{ site.baseurl }}/assets/img/docker-course/docker-architecture.png)

---

## Docker Workflow & Docker Components

To build a developer-friendly workflow, Docker gives us a set of core components.

- **Docker Daemon (dockerd)**: The background service that manages containers and images.
- **Docker Client**: The CLI we use to interact with the daemon (docker run, docker ps, etc.).
- **Docker Images**: Immutable templates (like a VM image) that define what goes into a container: OS, app code, dependencies. Every container starts from an image.
- **Docker Registries**: External systems that store and distribute images.
- **Docker Containers**: The running instances created from images.

![Docker Components]({{ site.baseurl }}/assets/img/docker-course/docker-components.png)

> Write Code --> Build Image (with app code & dependencies) --> Push to Registry --> Pull Image to Docker Host --> Run as Container.

### Docker Daemon (dockerd)

The Docker Daemon (dockerd) acts as the server. It is responsible for everything related to containers, like building images, running containers, managing networks, and more.

The Docker Client (the CLI or any tool using the REST API) sends requests to the daemon.

### Docker Image

> A Docker image is a read-only template that contains everything needed to run an application.

A docker image contains the following:

- Application code
- Runtime environment (Node.js, Python, Java, PHP, MySQL, etc.)
- System libraries and dependencies
- Configuration files
- Environment variables

Think of an image as a snapshot or blueprint. Every time you run an image, Docker creates a container, a running instance based on that blueprint.

### Docker Registry

If you build an image on your laptop, how do you share it with other systems or teammates? Copying it manually is messy and doesn’t scale.

That is where registries come in.

> A registry is a centralized system for storing and distributing Docker images.

![Docker Registry System]({{ site.baseurl }}/assets/img/docker-course/docker-registry.png)

The most well-known docker registry is **Docker Hub**, provided by Docker Inc.

With Docker Hub, you can upload (push) your images and download (pull) them from anywhere. This works using the concept of image repositories.

Think of **DockerHub** as **GitHub**:

- GitHub stores your source code repositories.
- Docker Hub stores your image repositories.

If your repository is public, anyone on Docker Hub can pull your images. If it is private, access is restricted.

In enterprise environments, teams often don’t rely on the public Docker Hub. Instead, they set up **private registries** for security and performance.

Cloud providers also offer their own managed registries, such as:

- Amazon Elastic Container Registry (ECR)
- Google Artifact Registry
- Azure Container Registry (ACR)

### Docker Container

From a single image, you can spin up any number of identical containers.

Containers have a writable layer on top of the image. The base image itself is immutable (read only), but the container’s writable layer allows temporary changes while it is running.

However, in real world projects, if want to change the container a container, we rebuild the image with the changes, and re-deploy it.

---

## Docker File (Dockerfile)

In Docker, that configuration file is called a Dockerfile.

> A Dockerfile is just a plain text file that contains a list of **instructions** and **arguments**.

- **Instruction** : The actual command that Docker understands (like FROM, COPY, RUN, CMD).
- **Argument**: The value or parameters that follow an instruction, telling Docker what exactly to do.

Docker reads the Dockerfile and automatically builds the image according to those instructions.

![Dockerfile]({{ site.baseurl }}/assets/img/docker-course/dockerfile.png)

###  How a Dockerfile Works

A Docker image is organized in a layered fashion. Each instruction in a **Dockerfile** adds a new layer to the image being built.

> **Each instruction = One layer**. When Docker processes a Dockerfile, instructions like **FROM, RUN, COPY,** etc., each creates a new read-only layer.

The final layer is the topmost **writable layer**, which is used by the running container. The other layers in the image are **read-only**.

![Dockerfile Layers]({{ site.baseurl }}/assets/img/docker-course/dockerfile-layers.png)

---

### Core Docker Image Basics

Docker uses **base image** to launch containers or to build new container images.

For example, you might start with a lightweight base image like Alpine or a full OS image like Ubuntu. On top of that, you add your application code, dependencies, and configurations. Once the image is ready, you can use it to launch as many containers as you want.

![Docker Base Image]({{ site.baseurl }}/assets/img/docker-course/docker-base-image.png)

### Building Custom Docker Image

Create your Dockerfile and save it as **Dockerfile**.

This Dockerfile takes a clean Ubuntu base image and turns it into a ready-to-run Nginx web server image.

```bash
FROM ubuntu:22.04 # - Sets the base image to Ubuntu 22.04
RUN apt-get update && apt-get install -y nginx # Runs commands inside the image during build
EXPOSE 80 # - Declares that the container will listen on port 80
CMD ["nginx", "-g", "daemon off;"] # - Defines the default command to run when the container starts
```
**In short**: this Dockerfile builds an Ubuntu‑based image, installs Nginx, exposes port 80, and ensures Nginx runs properly inside the container.

Let's build our Docker image.

```bash
docker build -t mynginx:1.0 . # -t mynginx:1.0 gives the image a name (mynginx) and a tag (1.0).
```

When you run the above command, here is what happens.

- Docker will read the Dockerfile.
- Execute each instruction in order.
- Create image layers for each instruction.
- Store the final image locally under the name mynginx:1.0.

Now we have an **Nginx image** ready. Let's create a container using that image and check if we can access the Nginx webpage locally in your browser.

```bash
docker run -d -p 8090:80 mynginx:1.0 # 
```

- **docker run** creates and starts a new container.
- **-d** runs it in detached mode, so it runs in the background.
- **-p 8090:80** maps port 8080 on your host to port 80 inside the container.
- **mynginx:1.0** the image we built earlier.

When the docker container is up and running, we can access our nginx page.

```yaml
http://192.168.77.50:8090
```

![Nginx Page]({{ site.baseurl }}/assets/img/docker-course/mynginx-page.png)

---

### Docker Image Tags

When you build or pull images, Docker needs to know which version you want.
Tags solve this by linking human-readable names to image IDs.

A tag is just a label used to identify a specific version of a Docker image.

Every Docker image has two main parts as given below.

```html
<repository>:<tag>
```

For example, nginx:1.27.0

- **nginx** is the repository name
- **1.27.0** is the tag

---

### Docker Image Layers

Docker uses a Union File System (UnionFS). Each instruction in that file (like FROM, RUN, COPY, ENV, etc.) adds a new layer to the image.

Every layer holds a specific part of your app’s environment, such as

- **Base OS**: Rarely changes
- **System libraries**: Essential tools and packages
- **Runtime**: Language layer (Python, Node.js, Java…)
- **Dependencies**: App packages (pip, npm, etc.)
- **App code**: Your actual app
- **Config layer**: ENV vars, ports, entrypoints (changes often)

![Docker Image Layers]({{ site.baseurl }}/assets/img/docker-course/docker-image-layers.png)

### Layer Sharing

If every new image had to include the full operating system and all dependencies from scratch, it would take a huge amount of space and time.

The layered filesystem fixes that.

> Instead of duplicating everything, Docker reuses existing layers wherever possible. Meaning, these layers can be shared between multiple images and different containers.

### Layer Sharing Example

Docker stores layers independently in **Docker’s local cache**, so they can be shared among different images.

Let's look at an example of team building a microservices architecture with the following three services.

- API Service (Python Flask) - 147MB (complete image)
- Worker Service (Python with Celery) - 145MB (complete image)
- Admin Dashboard (Node.js React app) - 260MB (complete image)

Each image contains duplicate copies of Ubuntu, package managers, and common dependencies.

Now, if these images share the layers that are identical, it would look like the following.

![Docker Layer Sharing]({{ site.baseurl }}/assets/img/docker-course/docker-layer-sharing.png)

---

### Layer Storage & Unification

Docker uses a Union File System (UFS) to combine all these read-only layers into one complete filesystem.

Let's say you have an image with three layers:

- **Layer 1 (base)**: Ubuntu OS with /bin, /lib, /etc
- **Layer 2**: Adds Python at /usr/bin/python3
- **Layer 3**: Adds your app at /app/myapp.py

When a container runs, it sees the following.

```text
/
├── bin/        (from layer 1)
├── lib/        (from layer 1)
├── etc/        (from layer 1)
├── usr/
│   └── bin/
│       └── python3  (from layer 2)
└── app/
    └── myapp.py     (from layer 3)
```

The container sees this as one complete filesystem at /, even though the files physically come from different layers stored separately on disk.

---

## Lab 04: Understanding Docker Image Layers

In this lab, you will learn how to analyze and explore the layers of a container image using Docker built-in command.

- Use **docker history** to see how each image layers are built
- Use **docker inspect** to check the image metadata, such as environment variables, entrypoints, and exposed ports

```bash
zafar@zserver:~$ docker images
IMAGE                                 ID             DISK USAGE   CONTENT SIZE   EXTRA
graylog/graylog:5.2                   3f4aa0885cd3        869MB          325MB    U
jawedzafar/php-mysqli:8.2             252e9efc9497        707MB          176MB
mongo:6.0                             03cda579c8ca       1.06GB          273MB    U
my-php-mysql:latest                   20ed0280b468        708MB          176MB
mynginx:1.0                           a1b0fa8beb13        324MB          103MB    U
opensearchproject/opensearch:2.11.0   2f49c399988d       2.12GB          883MB    U
php-demo:latest                       09d577b0ac1f        707MB          176MB
php-mysql-demo:latest                 8f41fbbeeb6c        707MB          176MB
php:8.2-apache                        1628cad804f8        714MB          183MB    U
zafar@zserver:~$
```

To see all the layers that make up an image, use:

```bash
zafar@zserver:~$ docker history php-demo:latest
IMAGE          CREATED       CREATED BY                                      SIZE      COMMENT
09d577b0ac1f   3 weeks ago   EXPOSE [80/tcp]                                 0B        buildkit.dockerfile.v0
<missing>      3 weeks ago   COPY index.php /var/www/html # buildkit         20.5kB    buildkit.dockerfile.v0
<missing>      3 weeks ago   WORKDIR /var/www/html                           4.1kB     buildkit.dockerfile.v0
<missing>      3 weeks ago   CMD ["apache2-foreground"]                      0B        buildkit.dockerfile.v0
<missing>      3 weeks ago   EXPOSE map[80/tcp:{}]                           0B        buildkit.dockerfile.v0
<missing>      3 weeks ago   WORKDIR /var/www/html                           4.1kB     buildkit.dockerfile.v0
<missing>      3 weeks ago   COPY apache2-foreground /usr/local/bin/ # bu…   20.5kB    buildkit.dockerfile.v0
<missing>      3 weeks ago   STOPSIGNAL SIGWINCH                             0B        buildkit.dockerfile.v0
<missing>      3 weeks ago   ENTRYPOINT ["docker-php-entrypoint"]            0B        buildkit.dockerfile.v0
<missing>      3 weeks ago   RUN /bin/sh -c docker-php-ext-enable sodium …   28.7kB    buildkit.dockerfile.v0
<missing>      3 weeks ago   RUN /bin/sh -c docker-php-ext-enable opcache…   28.7kB    buildkit.dockerfile.v0
<missing>      3 weeks ago   COPY docker-php-ext-* docker-php-entrypoint …   32.8kB    buildkit.dockerfile.v0
<missing>      3 weeks ago   RUN /bin/sh -c set -eux;   savedAptMark="$(a…   51MB      buildkit.dockerfile.v0
<missing>      3 weeks ago   COPY docker-php-source /usr/local/bin/ # bui…   20.5kB    buildkit.dockerfile.v0
<missing>      3 weeks ago   RUN /bin/sh -c set -eux;   savedAptMark="$(a…   13MB      buildkit.dockerfile.v0
<missing>      3 weeks ago   ENV PHP_SHA256=bc90523e17af4db46157e75d0c9ef…   0B        buildkit.dockerfile.v0
<missing>      3 weeks ago   ENV PHP_URL=https://www.php.net/distribution…   0B        buildkit.dockerfile.v0
<missing>      3 weeks ago   ENV PHP_VERSION=8.2.30                          0B        buildkit.dockerfile.v0
<missing>      3 weeks ago   ENV GPG_KEYS=39B641343D8C104B2B146DC3F9C39DC…   0B        buildkit.dockerfile.v0
<missing>      3 weeks ago   ENV PHP_LDFLAGS=-Wl,-O1 -pie                    0B        buildkit.dockerfile.v0
<missing>      3 weeks ago   ENV PHP_CPPFLAGS=-fstack-protector-strong -f…   0B        buildkit.dockerfile.v0
<missing>      3 weeks ago   ENV PHP_CFLAGS=-fstack-protector-strong -fpi…   0B        buildkit.dockerfile.v0
<missing>      3 weeks ago   RUN /bin/sh -c {   echo '<FilesMatch \.php$>…   45.1kB    buildkit.dockerfile.v0
<missing>      3 weeks ago   RUN /bin/sh -c a2dismod mpm_event && a2enmod…   45.1kB    buildkit.dockerfile.v0
<missing>      3 weeks ago   RUN /bin/sh -c set -eux;  apt-get update;  a…   14.2MB    buildkit.dockerfile.v0
<missing>      3 weeks ago   ENV APACHE_ENVVARS=/etc/apache2/envvars         0B        buildkit.dockerfile.v0
<missing>      3 weeks ago   ENV APACHE_CONFDIR=/etc/apache2                 0B        buildkit.dockerfile.v0
<missing>      3 weeks ago   RUN /bin/sh -c set -eux;  mkdir -p "$PHP_INI…   36.9kB    buildkit.dockerfile.v0
<missing>      3 weeks ago   ENV PHP_INI_DIR=/usr/local/etc/php              0B        buildkit.dockerfile.v0
<missing>      3 weeks ago   RUN /bin/sh -c set -eux;  apt-get update;  a…   366MB     buildkit.dockerfile.v0
<missing>      3 weeks ago   ENV PHPIZE_DEPS=autoconf   dpkg-dev   file  …   0B        buildkit.dockerfile.v0
<missing>      3 weeks ago   RUN /bin/sh -c set -eux;  {   echo 'Package:…   20.5kB    buildkit.dockerfile.v0
<missing>      3 weeks ago   # debian.sh --arch 'amd64' out/ 'trixie' '@1…   87.4MB    debuerreotype 0.17
```

- Layers are listed in **reverse order** - newest at the top, base layer at the bottom.
- **<missing>** entries - These are intermediate build cache layers. Only the final complete image is stored locally.
- SIZE column - Shows disk space each layer adds:
  - Large layers (73.2MB, 78.6MB): Base OS and Nginx installation.
  - Small layers (4.62kB, 3.02kB): Configuration scripts.
  - Zero-size layers (0B): Metadata like ENV, EXPOSE, CMD.
- CREATED BY column - Shows the Dockerfile instruction that created the layer.

### Inspect Image Details

Get detailed JSON information about the image:

```bash
docker inspect php-demo:latest
```

```json
zafar@zserver:~$ docker inspect php-demo:latest
[
    {
        "Id": "sha256:09d577b0ac1fe9c0b945bf39dff6c30a4e70c75334d5baf7977e37d9a6353aff",
        "RepoTags": [
            "php-demo:latest"
        ],
        "RepoDigests": [
            "php-demo@sha256:09d577b0ac1fe9c0b945bf39dff6c30a4e70c75334d5baf7977e37d9a6353aff"
        ],
        "Comment": "buildkit.dockerfile.v0",
        "Created": "2026-02-27T12:46:21.978527557Z",
        "Config": {
            "ExposedPorts": {
                "80/tcp": {}
            },
            "Env": [
                "PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin",
                "PHPIZE_DEPS=autoconf \t\tdpkg-dev \t\tfile \t\tg++ \t\tgcc \t\tlibc-dev \t\tmake \t\tpkg-config \t\tre2c",
                "PHP_INI_DIR=/usr/local/etc/php",
                "APACHE_CONFDIR=/etc/apache2",
                "APACHE_ENVVARS=/etc/apache2/envvars",
                "PHP_CFLAGS=-fstack-protector-strong -fpic -fpie -O2 -D_LARGEFILE_SOURCE -D_FILE_OFFSET_BITS=64",
                "PHP_CPPFLAGS=-fstack-protector-strong -fpic -fpie -O2 -D_LARGEFILE_SOURCE -D_FILE_OFFSET_BITS=64",
                "PHP_LDFLAGS=-Wl,-O1 -pie",
                "GPG_KEYS=39B641343D8C104B2B146DC3F9C39DC0B9698544 E60913E4DF209907D8E30D96659A97C9CF2A795A 1198C0117593497A5EC5C199286AF1F9897469DC",
                "PHP_VERSION=8.2.30",
                "PHP_URL=https://www.php.net/distributions/php-8.2.30.tar.xz",
                "PHP_ASC_URL=https://www.php.net/distributions/php-8.2.30.tar.xz.asc",
                "PHP_SHA256=bc90523e17af4db46157e75d0c9ef0b9d0030b0514e62c26ba7b513b8c4eb015"
            ],
            "Entrypoint": [
                "docker-php-entrypoint"
            ],
            "Cmd": [
                "apache2-foreground"
            ],
            "WorkingDir": "/var/www/html",
            "StopSignal": "SIGWINCH"
        },
        "Architecture": "amd64",
        "Os": "linux",
        "Size": 175646819,
        "RootFS": {
            "Type": "layers",
            "Layers": [
                "sha256:a257f20c716c7b5d0474b1b5998f4d1090b573d90b82f764b8f1f1faeb4b524f",
                "sha256:49f2428bea5de3ebf0c4a63e9b0d1dd6cd9bb96ebb8124d952d35d1f69402219",
                "sha256:a866d44f8bcc3c534acd261b5cbc319de70f89bead22a328326ee20634373a78",
                "sha256:f245bf2f689609bd0864d1ec1837742c20068672dace29d16aaddda05552a458",
                "sha256:3fe2e3f048e5ee54abe98c8c5f4d0e04cfa81919f4ceaee053443efb819fd834",
                "sha256:3cff72515b3adfad3a683fd0a62dd6b714268bce6ba5ff65994cf3892faef7c5",
                "sha256:9a4022a55faf0b36c14a7e359e92bee88dc7dd9753ab8af34d47eb89f815980a",
                "sha256:7d5dc74ac64409267284aa447cb4ec7ff1303b6c8cba4d805f8a9bd3c71abae8",
                "sha256:490c9769fa48a2e103f77a2410f44483e89596668e3b00d0ed9afba8507806dc",
                "sha256:cc3c175bdf62f23523d137fae670424f4feeca2390e2912f90da3198fa108ccf",
                "sha256:bfa07296c80e05167a58ada50d03ba10210c040b73954fe3b2774b27a975a1d6",
                "sha256:5535e0eb15603ad8b235144617f6cbe2d763df897ca02ffc4c15fb9115e5b720",
                "sha256:ff3f7828ebe7b756bf74d437db48e1f75fd07597388178d87acfdc5e760807c6",
                "sha256:c5329e9d26d98f42af45ee2c97029e34c7534042f998a0f5b6333a0c74e030b0",
                "sha256:5f70bf18a086007016e948b04aed3b82103a36bea41755b6cddfaf10ace3c6ef",
                "sha256:5f70bf18a086007016e948b04aed3b82103a36bea41755b6cddfaf10ace3c6ef",
                "sha256:2c07522d700e85e1889b6ecb8cdc371c1575dbfd0512affe72494abce391e054"
            ]
        },
        "Metadata": {
            "LastTagTime": "2026-02-27T12:46:22.064854964Z"
        },
        "Descriptor": {
            "mediaType": "application/vnd.oci.image.index.v1+json",
            "digest": "sha256:09d577b0ac1fe9c0b945bf39dff6c30a4e70c75334d5baf7977e37d9a6353aff",
            "size": 856
        },
        "Identity": {
            "Build": [
                {
                    "Ref": "xiembpurvxjjmlyimbir5jknu",
                    "CreatedAt": "2026-02-27T12:46:22.084918769Z"
                }
            ]
        }
    }
]
```

### Explore Image Layers on Disk

Docker stores all image layers locally at:

```bash
/var/lib/docker/containers # /var/lib/docker/overlayfs
```

```bash
zafar@zserver:~$ sudo ls -l /var/lib/docker/containers
total 24
drwx--x--- 4 root root 4096 Mar 22 22:25 2065a5fc114f43a75dee190a18493c4940839089fd4f1b6604a7ed71417621ab
drwx--x--- 4 root root 4096 Mar 22 22:25 a040a8ab434262616e15d62e445eff70a78d3d04d4098c82802e6a26c7a9e910
drwx--x--- 4 root root 4096 Mar 22 22:24 b1802f137eef796ff3701960daf2c2f4cce7f4b65ecc08bd5bee6c6e25798bdb
drwx--x--- 4 root root 4096 Mar 22 22:26 b9d16f4316b4bc18a14fa3e3a3bc6342d3839df51532b67a61573415d571c508
drwx--x--- 4 root root 4096 Mar 22 22:25 bf18cbfbeae3161337dfd37bedb28e1907627f676639edc22b105137887e6c31
drwx--x--- 4 root root 4096 Mar 22 22:25 e64eb26e99be8d6cc538b2c91836d747091e0d3a004d4c0e5a43de62fe6355b8
```

Each directory represents one layer of an image on your system.

---

## Build Push and Run Docker Image

We will build a web server image from scratch and run it.

1. **Prepare Code Files**: We will start by creating the basic HTML files for our web app.
2. **Create a Dockerfile**: Next, we will write a Dockerfile that defines how the web server image should be built, including the base image and setup commands.
3. **Build and Tag the Docker Image**: We will use Docker to build the image and assign it a tag, such as nginx:1.0.0.
4. **Create a Container from the Image**: We will then run a container using this image to bring our web server to life.
5. **Validate the Running Container**: After that, we will open the web app in a browser to confirm it’s running correctly.
6. **Push the Image to a Registry**: Finally, we will push our verified image to Docker Hub so it can be shared or used in other environments.

![Docker image build workflow]({{ site.baseurl }}/assets/img/docker-course/image-build-workflow.png)

Create a directory in the following order:

```text
nginx-demo2
├── Dockerfile
└── files
    ├── default
    └── index.html
```

The Dockerfile will look as follows:

```yaml
# Base image
FROM ubuntu:latest

# Use LABEL for metadata as recommended
LABEL maintainer="contact@zacademy.com"

# Install nginx
RUN apt-get update -y && apt-get install -y nginx

# Copy configuration files and content
COPY files/default /etc/nginx/sites-available/default
COPY files/index.html /usr/share/nginx/html/index.html

# Expose port 80
EXPOSE 80

# Start nginx
CMD ["/usr/sbin/nginx", "-g", "daemon off;"]
```

Here’s the explanation of each step:

- **FROM** instruction will pull the Ubuntu latest version Image from the Docker hub.
- With **LABEL** instruction, we are adding metadata about the maintainer. It is not a mandatory instruction.
- Then we’re installing Nginx using the **RUN** instruction
- Then we’re **copying the Nginx default config file** from the local files directory to the target image directory.
- Next, we’re **copying our index.html** file from the local files directory into the target image directory. It will **overwrite the default index.html** file created during Nginx installation.
- We’re **exposing port 80** as the Nginx service listens on port 80.
- At last, we’re **running the Nginx server using CMD instruction** when the Docker image launches.

---

The **default** file contains our nginx web server configurations.

```bash
server {
    listen 80 default_server;
    listen [::]:80 default_server;
    
    root /usr/share/nginx/html;
    index index.html index.htm;
    server_name _;
    location / {
        try_files $uri $uri/ =404;
    }
}
```

Finally, the **index.html** file contains our code or web server front page. You may change it as you wish.

```html
<!DOCTYPE html>
  <head>
    <title>Nginx WebServer</title>
  </head>
  <body>
    <div class="container">
      <h1>My Nginx App WebServer</h1>
      <h2>This is my Nginx WebServer</h2>
      <p>Hello everyone, This Nginx Web Server is running via Docker container</p>
    </div>
  </body>
</html>
```

### Building the Web Server Image

```bash
docker build -t nginx-demo2:1.0 . # -t is for tagging the image.
# nginx is the name of the image.
# 1.0 is the tag name.
```

Now you can see your images in Docker. Notice the image name and tag.

```bash
zafar@zserver:~/docker-projects/nginx-demo2$ docker images
IMAGE                                 ID             DISK USAGE   CONTENT SIZE   EXTRA
nginx-demo2:1.0                       0faa4213404e        229MB         73.9MB
php-demo:latest                       09d577b0ac1f        707MB          176MB
php-mysql-demo:latest                 8f41fbbeeb6c        707MB          176MB
php:8.2-apache                        1628cad804f8        714MB          183MB    U
```

### Create a Container From the Image

Now that we have built our webserver image, let’s run it as a container.

```bash
docker run -d -p 9090:80 --name webserver nginx-demo2:1.0
# -d runs the container in detached mode, so it runs in the background
# -p maps ports between your host and the container. Format: host-port:container-port.
# Here, 9090 on your machine maps to port 80 inside the container.
# --name assigns a custom name to your container. In this example, it’s webserver.
```

Now, open your browser and visit:

```bash
http://192.168.77.50:9090
```

![Nginx Demo2]({{ site.baseurl }}/assets/img/docker-course/nginx-demo2.png)

---

## Push Docker Image To Docker Hub

This is the shipping phase of Docker. You are essentially packaging your app and **pushing** it to **DockerHub**, so others can **pull** and run it.

> In enterprise environments, this is usually done using private registries such as AWS ECR, Google Container Registry (GCR), or Azure Container Registry (ACR).

To push a Docker image into the registry, you need to log in to the Docker Hub using the **Docker CLI**.

```bash
zafar@zserver:~/$ docker login -u jawedzafar
# then enter you password
```

Now your local Docker CLI is now authenticated with Docker Hub.

Next, you need to **tag your image** with your Docker Hub username and repository name before pushing it.

```bash
docker tag nginx-demo2:1.0 jawedzafar/nginx-demo2-webserver:1.0
```

> By tagging it as **jawedzafar/nginx-demo2-webserver:1.0**, we are telling Docker to push it under the **jawedzafar namespace** in Docker Hub.
Without this tag, Docker won’t know which repository to upload it to.

```bash
zafar@zserver:~$ docker images
IMAGE                                  ID             DISK USAGE   CONTENT SIZE   EXTRA
jawedzafar/nginx-demo2-webserver:1.0   0faa4213404e        229MB         73.9MB    U
```

### Push the Image

Now, push the image to Docker Hub using the following command.

```bash
docker push jawedzafar/nginx-demo2-webserver:1.0
```

Docker will start uploading the image layers. Once the push completes, you will see a confirmation as given below.

```golang
zafar@zserver:~$ docker push jawedzafar/nginx-demo2-webserver:1.0
The push refers to repository [docker.io/jawedzafar/nginx-demo2-webserver]
ac9dbd7a384a: Pushed
2909f5caf3b2: Pushed
817807f3c64e: Pushed
4f2078622cca: Pushed
874964188da0: Pushed
1.0: digest: sha256:0faa4213404e3752bcd740cb5cf7b011643d07c1a78c44f9812b899748a3c773 size: 856
```

Head to your Docker Hub dashboard to see the image under your repositories.

![DockerHub New Repository Push]({{ site.baseurl }}/assets/img/docker-course/docker-hub.png)

We’ve successfully built an image, verified it locally, and pushed it to a registry.

---

### Useful Docker Commands

```bash
docker stop $(docker ps -q) # stop all running containers at once
docker system prune # remove unused Docker objects like images, containers, volumes, and networks
```

> Be careful when running the **docker system prune** command as it will permanently delete all unused objects, including any data or configurations that may be stored in them.

```bash
docker system prune -a --volumes # aggressive cleanup, removes everything unused, 
# including volumes and all images not tied to a container.
```

```bash
docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' <CONTAINER_ID_OR_NAME>
# returns the container IP address.
```

```bash
zafar@zserver:~$ docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' cfee36bed256
172.17.0.5
```

---

## Advance Docker Image Build Practices

### Docker Multistage Build

The multistage build pattern involves using intermediate images, or build stages, to compile code, install dependencies, and package files. The aim is to **eliminate unnecessary layers** in the image.

After this, only the required files are copied into the final image, making it smaller and more efficient to run.

To understand this better, let’s create a simple **Node.js application** and see how multistage builds help optimize image size.

```bash
git clone https://github.com/techiescamp/docker-course.git
cd docker-course/multi-stage-builds
```

You will see the following folder structure.

```text
├── Dockerfile1
├── Dockerfile2
├── env
├── index.js
└── package.json
```

This is a simple Node.js app that which returns a simple message.

Now, let’s look at the first Dockerfile, **Dockerfile1**.

```bash
FROM node:22
COPY . .
RUN npm install
EXPOSE 3000
CMD [ "index.js" ]
```

This is a single stage Dockerfile example, which uses node:22 as base image and copies all files into the image.

Then it install dependencies with npm install, exposes in port 3000 and uses index.js command to run the application.

Lets build this image with **Dockerfile1**.

```bash
zafar@zserver:~/docker-projects$ docker build -t node-app:1.0 --no-cache -f Dockerfile1 .
```

Now, if we see the size of our image, it is 1.65GB:

```bash
zafar@zserver:~/docker-projects$ docker images
IMAGE                                  ID             DISK USAGE   CONTENT SIZE   EXTRA
jawedzafar/php-mysqli:8.2              252e9efc9497        707MB          176MB
my-php-mysql:latest                    20ed0280b468        708MB          176MB
mynginx:1.0                            a1b0fa8beb13        324MB          103MB    U
node-app:1.0                           b42c0c41312b       1.65GB          412MB
```

### Let’s create a multistage build

We will use node:22 as the base image, i.e., the image for all the dependencies & modules installation, after that, we will move the contents into a minimal and lighter alpine based image.

![Multistage Builds]({{ site.baseurl }}/assets/img/docker-course/multistage-builds.png)

Lets, check our new Dockerfile, **Dockerfile2**.

```bash
FROM node:22 AS build

WORKDIR /app
COPY package.json index.js env ./
RUN npm install

FROM node:alpine AS main

COPY --from=build /app /
EXPOSE 3000
CMD ["index.js"]
```

In **Dockerfile2** we have Two stages:

- The first stage is **build stage** which uses node:22 as base image, copies only the required files into the working directory, and installs the dependencies using npm install command.
- The second stage is the **main stage**, which uses node:alpine as base image and copies the files and dependencies from the /app directory to the root.

Now build the image using **Dockerfile2**.

```bash
zafar@zserver:~/docker-projects$ docker build -t node-app:2.0 --no-cache -f Dockerfile2 .
zafar@zserver:~/docker-projects$ docker images
IMAGE                                  ID             DISK USAGE   CONTENT SIZE   EXTRA
jawedzafar/php-mysqli:8.2              252e9efc9497        707MB          176MB
my-php-mysql:latest                    20ed0280b468        708MB          176MB
mynginx:1.0                            a1b0fa8beb13        324MB          103MB    U
node-app:1.0                           b42c0c41312b       1.65GB          412MB
node-app:2.0                           b9f93b91285f        243MB         60.4MB
```

The new **reduced image size is 243MB** as compared to the image with all dependencies.
That’s an **optimization of over 80%!**.

---

### Using heredoc With Dockerfil

The Dockerfile supports **heredoc syntax**, which is useful for **running multiple commands** or creating files directly within the Dockerfile.

If you have multiple RUN commands, you can use heredoc syntax to group them together.

### Example 1: Running Multiple Commands

```html
RUN <<EOF
apt-get update
apt-get upgrade -y
apt-get install -y nginx
EOF
```

### Example 2: Executing a Python Script

Let’s say you want to execute a Python script from a Dockerfile, you can use the following syntax.

```html
RUN python3 <<EOF
with open("/hello", "w") as f:
    print("Hello", file=f)
    print("World", file=f)
EOF
```

### Example 3: Creating a File

You can use heredoc syntax to create a file directly in the Dockerfile. Here’s an example with Nginx.

```html
FROM nginx
COPY <<EOF /usr/share/nginx/html/index.html
<html>
  <head>
    <title>Dockerfile</title>
  </head>
  <body>
    <div class="container">
      <h1>My App</h1>
      <h2>This is my first app</h2>
      <p>Hello everyone, This is running via Docker container</p>
    </div>
  </body>
</html>
EOF
```

Benefits of Using Heredoc Syntax

- **Simplifies Dockerfile**: Reduces the number of RUN commands, making the Dockerfile cleaner and easier to read.
- **Efficient Layering**: Combining commands in a single RUN instruction reduces the number of layers in the image.
- **Inline Scripts and Files**: Allows you to write scripts and create files directly within the Dockerfile, avoiding the need for separate script files.

---

### ENTRYPOINT vs CMD

In a Dockerfile, **ENTRYPOINT** and **CMD** are two different instructions that are used to define how a container should run.

**ENTRYPOINT** is used to specify the main command that will always run when the container starts. The default ENTRYPOINT command is **/bin/sh -c**.

**CMD**, on the other hand, is used to specify the default command and arguments that should be executed when a container is started.

If both **ENTRYPOINT** and **CMD** are specified in a **Dockerfile**, the command specified in **CMD will be appended to the ENTRYPOINT** command. It acts as an argument for ENTRYPOINT. The resulting command will be executed when the container is started.

The workflow diagram of ENTRYPOINT and CMD is given below

![Entrypoint vs CMD]({{ site.baseurl }}/assets/img/docker-course/entrypoint-cmd.gif)

### ENTRYPOINT

- Specifies the main command that will always run when the container starts.
- It makes the container behave like a specific executable.
- **Example**: ENTRYPOINT ["python3", "app.py"] ensures the container always runs that Python script.
- **Example**: ENTRYPOINT ["php", "app.php"] - ensures that whenever the container starts, it will run php app.php.

### CMD

- Provides default arguments to the ENTRYPOINT or defines a command if no ENTRYPOINT is set.
- If ENTRYPOINT is not set, then, CMD will act as ENTRYPOINT.

```bash
FROM php:8.2-cli
COPY app.php /usr/src/app/app.php
WORKDIR /usr/src/app
# No ENTRYPOINT defined
CMD ["php", "app.php"]
```

- Running **docker run myphp-app** → executes php app.php.
- Running **docker run myphp-app php other.php** → completely **replaces the CMD**, so it runs **php other.php**.

However, with ENTRYPOINT we can not replace the command, we just append to it:

```bash
FROM php:8.2-cli
COPY app.php /usr/src/app/app.php
WORKDIR /usr/src/app
# No ENTRYPOINT defined
ENTRYPOINT ["php", "app.php"]
```

- Running **docker run myphp-app** → executes php app.php.
- Running **docker run myphp-app other.php** → executes **php app.php other.php** (because arguments are appended to ENTRYPOINT).

When you use both **ENTRYPOINT** and **CMD** in a Dockerfile, they work together:

- **ENTRYPOINT** defines the fixed program that will always run.
- **CMD** provides default arguments to that program, which can be overridden at runtime.

```bash
FROM php:8.2-cli
COPY app.php /usr/src/app/app.php
WORKDIR /usr/src/app

ENTRYPOINT ["php", "app.php"]
CMD ["--mode=debug"]
```

Running **docker run myphp-app** → executes

```bash
php app.php --mode=debug
```

Running **docker run myphp-app --mode=prod** → executes

```bash
php app.php --mode=prod
```

If your PHP app expects a **.txt** file as input, you can design your Dockerfile so that ENTRYPOINT fixes the program (php app.php) and CMD provides a default file argument.

```bash
FROM php:8.2-cli
COPY app.php /usr/src/app/app.php
COPY default.txt /usr/src/app/default.txt
WORKDIR /usr/src/app

ENTRYPOINT ["php", "app.php"]
CMD ["data.txt"]
```

Running docker run myphp-app → executes

```bash
php app.php data.txt
```

Running **docker run myphp-app input.txt** → executes

```bash
php app.php input.txt
```

---

### Dockerfile Best Practices

Some of the Dockerfile practices which we should follow:

- Use a **.dockerignore file** to exclude unnecessary files and directories to increase the build’s performance.
- Use **trusted base images** only and keep updating the images periodically.
- Each instruction in the Dockerfile adds an extra layer to the Docker image. **Minimize the number of layers** by consolidating the instructions to increase the build’s performance and time.
- **Run as a Non-Root User** to avoid security breaches.
- Reduce the image size for faster deployment and **avoid installing unnecessary tools** in your image. Use minimal images to reduce the attack surface.
- **Use specific tags** over the latest tag for the image to avoid breaking changes over time.
- **Avoid using multiple RUN commands** as it creates multiple cacheable layers which will affect the efficiency of the build process.
- Never share or copy the application credentials or any sensitive information in the Dockerfile. If you use it, add it to **.dockerignore**.
- Use **EXPOSE** and **ENV** commands as late as possible in Dockerfile.

---

## Dockerizing Applications

Let’s look at a few real-world examples like Dockerizing Prometheus, Grafana, and similar tools.

### Dockerize Java Application

Using multistage build, you can build the java application and dockerize the application using a single **Dockerfile**.

```bash
# Build stage: use official Maven + JDK image
FROM maven:3.9.6-eclipse-temurin-17 AS build

# Copy source code into container
COPY . /usr/src/app
WORKDIR /usr/src/app

# Build Java application (skip tests for faster build)
RUN mvn clean install -DskipTests

# Runtime stage: use official JRE image
FROM eclipse-temurin:17-jre
WORKDIR /app

# Copy the JAR file from the build stage
COPY --from=build /usr/src/app/target/*.jar ./java.jar

# Expose the port
EXPOSE 8080

# Run the JAR file
CMD ["java", "-jar", "java.jar"]
```

You can see in the above Dockerfile that it has **two stages**, the first stage **builds the application** and the second stage **runs the application**.

- The first stage used a **maven:3.9.6-eclipse-temurin-17** as its base image.
- Then it copies the Java application source code and pastes it in /usr/src/, make sure to specify the path of your Java application source code.
- Finally, it starts building the application using the mvn clean install command.
- The second stage uses **eclipse-temurin:17-jre** as its base image and it copies the JAR file from the build stage directory /usr/src/target/ to the current working directory /app as java.jar.
- Then it **exposes port 8080** and runs the JAR file using the **java -jar** command.

Now lets build our image.

```bash
docker build -t java-app:1.0 . # 
```

```bash
docker tag java-app:latest java-app:1.0 # you may change the image tag if you forgot during the build
```

After build process has been completed, you see your images:

```bash
zafar@zserver:~$ docker images
IMAGE                                  ID             DISK USAGE   CONTENT SIZE   EXTRA
java-app:1.0                           1f0f136e3ca8        487MB          148MB
jawedzafar/php-mysqli:8.2              252e9efc9497        707MB          176MB
mynginx:1.0                            a1b0fa8beb13        324MB          103MB    U
node-app:2.0                           b9f93b91285f        243MB         60.4MB    U
```

Now, run the following command to run your Java application in a container.

```bash
docker run -d -p 8092:8080 java-app:1.0 # maps local port 8092 to the container port 8080
```

Now you can access you application at:

```bash
http://192.168.77.50:8092
```

---

## Dockerize Python Flask Application

Given below is an overview workflow of what we are going to do in this lesson.

![Dockerize Python Flask App]({{ site.baseurl }}/assets/img/docker-course/python-flask-app.png)

We will create the following directory structure:

```text
python-flask
    ├── Dockerfile
    ├── app.py
    ├── requirements.txt
    └── templates
        └── index.html
```

We will use the following **Dockerfile** to build our image:

```bash
ARG PYTHON_VERSION=3.12
FROM python:${PYTHON_VERSION}-slim

ENV PYTHONDONTWRITEBYTECODE=1
ENV PYTHONUNBUFFERED=1

WORKDIR /app

ARG UID=10001
RUN adduser \
    --disabled-password \
    --gecos "" \
    --home "/nonexistent" \
    --shell "/sbin/nologin" \
    --no-create-home \
    --uid "${UID}" \
    appuser

RUN --mount=type=cache,target=/root/.cache/pip \
    --mount=type=bind,source=requirements.txt,target=requirements.txt \
    python -m pip install -r requirements.txt

USER appuser

COPY . .

EXPOSE 8000

CMD ["gunicorn", "--bind", "0.0.0.0:8000", "app:app"]
```

- The dockerfile uses **Python 3.12-slim** as the **base image** and sets /app as the working directory.
- The next step sets up a **non-root user** named **appuser** for security best practices and sses a cache mount to speed up installs by avoiding unnecessary downloads.
- Then it **mounts the requirements.txt** file from the host filesystem into the container. It allows the pip install command to access the requirements.txt file without copying it into the container’s filesystem. It ensures that the latest version of the requirements.txt file is always used during the build.
- Copies the application code to the working directory and **exposes on port 8000**.
- Finally, uses **Gunicorn** to run the application and **bind it to 0.0.0.0**.

```bash
zafar@zserver:~/docker-projects/python-flask-app$ docker build -t python-flask-app:1.0 .
```

```bash
zafar@zserver:~/docker-projects/python-flask-app$ docker images
IMAGE                                  ID             DISK USAGE   CONTENT SIZE   EXTRA
java-app:1.0                           1f0f136e3ca8        487MB          148MB    U
my-php-mysql:latest                    20ed0280b468        708MB          176MB
mynginx:1.0                            a1b0fa8beb13        324MB          103MB    U
node-app:1.0                           b42c0c41312b       1.65GB          412MB
node-app:2.0                           b9f93b91285f        243MB         60.4MB    U
python-flask-app:1.0                   4a0ecf521aa8        188MB         45.5MB
```

Once the Docker image build is finished, run the Docker image using the command below:

```bash
zafar@zserver:~/docker-projects$ docker run -d -p 8000:8000 python-flask-app:1.0
```
Now, you can access your Python Flask app at ***http://192.168.77.50:8000***

![Python Flask App Page]({{ site.baseurl }}/assets/img/docker-course/python-flask-app-page.png)

---

## Dockerize Node.js Application

Given below is an overview workflow of what we are going to do in this lesson.

![Dockerize NodeJS App]({{ site.baseurl }}/assets/img/docker-course/dockerize-nodejs-app.png)

We will have the following Node.js project structure.

```text
node.js
   ├── Dockerfile
   ├── package-lock.json
   ├── package.json
   ├── public
   ├── src
   └── .gitignore
```

The content of the Dockerfile is given below.

```bash
FROM node:20.12.2-alpine

# Work directory for all steps
WORKDIR /app

# Copy files from local to the work directory
COPY /public ./public
COPY /src ./src
COPY /package*.json ./

# Install all dependencies
RUN npm install

# Expose port
EXPOSE 3000

# Command to run the application
CMD ["npm", "start"]
```

- The Dockerfile uses the **node:20.12.2-alpine** as its **base image** and set **/app** as the working directory.
- The next step **copies the public and src folders** into the working directory and also copies the **package.json** and **package-lock.json** files into the working directory.
- Then it runs the **npm install** command in the working directory to **install all the dependencies** given in the **package.json** file which are required to run the application.
- Finally, it exposes the Docker image on **port 3000** because the application uses port 3000 and it runs the **npm start command** to run the application.

```bash
zafar@zserver:~/docker-projects/nodejs-app$ docker build -t nodejs-app:3.0 .
```

```bash
zafar@zserver:~/docker-projects/nodejs-app$ docker images
IMAGE                                  ID             DISK USAGE   CONTENT SIZE   EXTRA
java-app:1.0                           1f0f136e3ca8        487MB          148MB    U
mynginx:1.0                            a1b0fa8beb13        324MB          103MB    U
node-app:2.0                           b9f93b91285f        243MB         60.4MB    U
nodejs-app:3.0                         62685f43fe6e        910MB          179MB
python-flask-app:1.0                   4a0ecf521aa8        188MB         45.5MB    U
```

Once the Docker image build is finished, run the Docker image using the command below:

```bash
zafar@zserver:~/docker-projects$ docker run -d -p 3003:3000 nodejs-app:3.0
```

Now, you can access your NodeJS app at ***http://192.168.77.50:3003***

![NodeJS App Page]({{ site.baseurl }}/assets/img/docker-course/nodejs-app-page.png)

---

## Dockerize Prometheus

In this lesson, you are going to learn about dockerizing Prometheus.

![Dockerize Prometheus]({{ site.baseurl }}/assets/img/docker-course/dockerize-prometheus.png)

### What is Prometheus

Prometheus is an open-source monitoring and alerting toolkit. Its core role is to collect, store, and query time-series metrics from systems and applications.

- **Metrics collection**: It scrapes metrics from targets (like nodes, pods, or apps) via HTTP endpoints, usually /metrics.
- **Time-series database**: Stores metrics with labels (e.g., cpu_usage{instance="node1"}).
- **Query language (PromQL)**: Lets you slice, dice, and aggregate metrics for dashboards or alerts.
- **Alerting**: Works with Alertmanager to send notifications when conditions are met.
- **Cloud-native design**: Built for dynamic environments like Kubernetes, where services come and go.


```text
prometheus
     .
     ├── Dockerfile
     └── prometheus.yml
```

To dockerize the Prometheus, we are going to use the following Dockerfile:

```bash
# Use Alpine Linux as the base image (very small footprint)
FROM alpine:3.19

# Install required tools: wget, curl, tar
# apk is Alpine's package manager; --no-cache avoids storing index files
RUN apk add --no-cache wget curl tar

# Download Prometheus release tarball, extract it, move to /usr/local/prometheus
# Then clean up the tarball to keep the image small
RUN wget https://github.com/prometheus/prometheus/releases/download/v3.5.0/prometheus-3.5.0.linux-amd64.tar.gz \
    && tar xvfz prometheus-3.5.0.linux-amd64.tar.gz \
    && mv prometheus-3.5.0.linux-amd64 /usr/local/prometheus \
    && rm prometheus-3.5.0.linux-amd64.tar.gz

# Copy your Prometheus configuration file into the container
COPY prometheus.yml /usr/local/prometheus/prometheus.yml

# Expose Prometheus default port
EXPOSE 9090

# Set the default command to run Prometheus with the config file
CMD ["/usr/local/prometheus/prometheus", "--config.file=/usr/local/prometheus/prometheus.yml"]
```

- Use **Ubuntu 24.04** as the base image.
- Set an **environment variable** to avoid installation prompts.
- **Update and install** required packages.
- **Clean up package** lists to save space.
- Download and **extract Prometheus 3.5.0**.
- Move Prometheus to **/usr/local/prometheus** and remove the **tar.gz** file.
- Copy **prometheus.yml** to the container and expose Prometheus on **port 9090**.
- Set the command to **start Prometheus** with the given configuration file.

The configuration file **prometheus.yml** is already pushed with the Dockerfile inside the repository.

```bash
# Global settings that apply to all scrape jobs
global:
  # How often Prometheus scrapes targets (every 15 seconds here)
  scrape_interval: 15s

# Define the list of scrape jobs (each job is a set of targets)
scrape_configs:
  # Name of this scrape job (used for labeling and identification)
  - job_name: 'prometheus'
    # Static list of targets to scrape (IP:PORT format)
    static_configs:
      # Prometheus will scrape metrics from this endpoint
      # In this case, your Prometheus server itself, running on 192.168.77.50:9092
      - targets: ['192.168.77.50:9092']
```

To scrape metrics of different services you can add another **job_name** and **target URLs** under the **scrape_configs**.

However, Prometheus does not directly scrap application metrics, for this, we need **exporters**.

> Exporters are small programs that translate app stats into Prometheus metrics.

Prometheus doesn’t scrape arbitrary application traffic — it only scrapes endpoints that expose metrics in a specific format (plain text, key=value pairs, with timestamps).

For example:

- **MySQL Exporter** (mysqld_exporter) → exposes DB metrics (queries/sec, connections, cache hits) on a port like 9104.
- **PHP-FPM Exporter** or Apache/Nginx Exporter → exposes web server/PHP metrics (requests, latency, errors).
- Prometheus scrapes the exporter’s **IP:PORT**, not the app directly.

```bash
scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['192.168.77.50:9092']   # Prometheus itself

  - job_name: 'php-app'
    static_configs:
      - targets: ['192.168.77.50:9100']   # Example exporter port
```

Once, we add new endpoints to our config file, we need to restart our container.

Now, lets build our Prometheus image:

```bash
zafar@zserver:~/docker-projects/prometheus$ docker build -t prometheus:2.0 .
```

We can view our newly built images:

```bash
zafar@zserver:~/docker-projects/prometheus$ docker images
IMAGE                                  ID             DISK USAGE   CONTENT SIZE   EXTRA
java-app:1.0                           1f0f136e3ca8        487MB          148MB    U
node-app:2.0                           b9f93b91285f        243MB         60.4MB    U
nodejs-app:3.0                         62685f43fe6e        910MB          179MB    U
prometheus:2.0                         191eb3bb906a        454MB          129MB
python-flask-app:1.0                   4a0ecf521aa8        188MB         45.5MB    U
```

Lets run our image to create our Prometheus container:

```bash
zafar@zserver:~/docker-projects$ docker run -d -p 9092:9090 prometheus:2.0
```

Now, you can access your Prometheus Web UI at ***http://192.168.77.50:9092***

![Prometheus Web UI]({{ site.baseurl }}/assets/img/docker-course/prometheus-webui.png)

Prometheus uses **PromQL** query language. Also, in order to get system metrics we need NodeExporter. Otherwise, we can only query Prometheus's own metrics.

---

## Dockerize Grafana

```text
grafana
     .
     ├── Dockerfile
     └── grafana.ini
```

we will use the following **Dockerfile** for Grafana containerization:

```bash
# Use Alpine Linux as the base image (lightweight, ~5 MB)
FROM alpine:3.19

# Install required tools and dependencies
# apk is Alpine's package manager; --no-cache avoids storing index files
RUN apk add --no-cache wget curl bash libc6-compat

# Download and install Grafana OSS (binary tarball)
RUN wget https://dl.grafana.com/oss/release/grafana-11.0.0.linux-amd64.tar.gz \
    && tar -zxvf grafana-11.0.0.linux-amd64.tar.gz \
    && mv grafana-v11.0.0 /usr/share/grafana \
    && rm grafana-11.0.0.linux-amd64.tar.gz

# Copy your Grafana configuration file into the container
COPY grafana.ini /etc/grafana/grafana.ini

# Expose Grafana’s default port (3000)
EXPOSE 3000

# Start Grafana server with the provided config and homepath
CMD ["/usr/share/grafana/bin/grafana-server", "--config=/etc/grafana/grafana.ini", "--homepath=/usr/share/grafana"]
```

Our Grafana configuration file **grafana.ini** is as follows:

```bash
# -------------------------------
# Grafana server configuration
# -------------------------------
[server]
# The port Grafana will listen on inside the container
http_port = 3000

# -------------------------------
# Security settings
# -------------------------------
[security]
# Default admin username
admin_user = admin
# Default admin password (change this in production!)
admin_password = admin

# -------------------------------
# User management
# -------------------------------
[users]
# Disable public sign-up; only admins can create users
allow_sign_up = false
```

Now, run the following command in the same directory where the Dockerfile is located.

```bash
zafar@zserver:~/docker-projects/grafana$ docker build -t grafana:2.0 .
```

```bash
zafar@zserver:~/docker-projects/grafana$ docker images
IMAGE                                  ID             DISK USAGE   CONTENT SIZE   EXTRA
grafana:2.0                            31464ae866f5        627MB          129MB
java-app:1.0                           1f0f136e3ca8        487MB          148MB    U
nodejs-app:3.0                         62685f43fe6e        910MB          179MB    U
prometheus:2.0                         191eb3bb906a        454MB          129MB    U
python-flask-app:1.0                   4a0ecf521aa8        188MB         45.5MB    U
```

Lets run our Grafana container:

```bash
zafar@zserver:~/docker-projects$ docker run -d -p 3002:3000 grafana:2.0
```

Now, you can access your Grafana dashboard at ***http://192.168.77.50:3002***

---

## Optimize Docker Images

Docker image optimization means making your container images smaller, faster, and more secure by reducing unnecessary layers, dependencies, and files.

### Use Minimal Base Images

Alpine images mostly have a size of less than 10MB.

### Use Multistage builds

Multistage help to reduce size by allowing you to separate the build process into multiple stages, where each stage can use a different Docker image with a specific set of tools and dependencies.

This enables you to build only the necessary components and discard the rest, resulting in a smaller final image.

### Minimize the Number of Layers

Docker images work in the following way: 

Each **RUN, COPY, FROM** instructions in Dockerfile, add a new layer & each layer adds to the build execution time & increases the storage requirements of the image.

Let’s create a Ubuntu image with updated & upgraded libraries, along with some necessary packages installed such as vim, net-tools, dnsutils.

We have prepared two Docker files, **Dockerfile1** and **Dockerfile2**.

```bash
FROM ubuntu:latest
ENV DEBIAN_FRONTEND=noninteractive 
RUN apt-get update -y 
RUN apt-get upgrade -y 
RUN apt-get install vim -y 
RUN apt-get install net-tools -y 
RUN apt-get install dnsutils -y
```

```bash
FROM ubuntu:latest
ENV DEBIAN_FRONTEND=noninteractive
RUN apt-get update -y && \
    apt-get upgrade -y && \
    apt-get install --no-install-recommends vim net-tools dnsutils -y
```

As you can see, we have decreased the number of layers in Dockerfile2, by combining the **RUN commands** into a single layer.

Also, we have used **--no-install-recommends** flag to disable recommended packages. It is recommended whenever you use **install** in your Dockerfiles.

### Docker Image Caching and Build Optimization

When you build a Docker image, Docker caches each layer of the image. When you rebuild the image, Docker only rebuilds the layers that have changed, and it reuses the cached layers that haven't changed.

This can significantly speed up the build process, but it requires some careful planning to ensure that Docker can effectively use the cache.

Lets compares **Dockerfile1** and **Dockerfile2**.

```bash
FROM ubuntu:latest 
ENV DEBIAN_FRONTEND=noninteractive 
RUN apt-get update && apt-get upgrade -y && apt-get install -y vim net-tools dnsutils 
COPY . .
```

```bash
FROM ubuntu:latest 
COPY . . 
ENV DEBIAN_FRONTEND=noninteractive 
RUN apt-get update && apt-get upgrade -y && apt-get install -y vim net-tools dnsutils
```

Docker would be able to use the cache functionality better with **Dockerfile1** than **Dockerfile2** due to the better placement of the **COPY command**.

> Put frequently changing commands, such as COPY or ADD, at the end of the Dockerfile.

This is because Docker caches all previous layers, so if a layer changes, all subsequent layers will also need to be rebuilt.

### Use Dockerignore

Dockerignore is a mechanism in Docker that enables you to exclude specific files and directories from being copied to the Docker image during the build process.

For example, you may have log files, cache files, or test files that are not needed in the final image.

### Dockerfile

```bash
FROM node:14-alpine 
WORKDIR /app 
COPY . . 
RUN npm install --production 
CMD ["npm", "start"]
```

### Dockerignore

```bash
node_modules 
npm-debug.log
```

In this example, we're building a Node.js application and installing its dependencies. We want to exclude the node_modules directory and npm-debug.log file from copying into the Docker image.

> Dockerignore can be a simple but effective way to reduce Docker image size by excluding unnecessary files and directories from the image during the build process.

---

### Keep Application Data Elsewhere

Storing application data in the image will unnecessarily increase the size of the images.

It’s highly recommended to use the volume feature of the container runtimes to keep the image separate from the data.

---

## Image Optimization With Tools

There are also tools that help reduce image size and improve performance. You can integrate these tools into your image build pipelines to further optimize your Docker images.

### Inspecting Layers With Dive

Dive is a simple command-line tool that shows what is actually inside a container image layer.

Dive helps visualize how the image layers were formed during a build, which also helps to identify unnecessary files and packages that are not required in this image.

### Installing Dive on Ubuntu

```bash
sudo apt install dive
# or
sudo DIVE_VERSION=$(curl -sL "https://api.github.com/repos/wagoodman/dive/releases/latest" | grep '"tag_name":' | sed -E 's/.*"v([^"]+)".*/\1/')
curl -fOL "https://github.com/wagoodman/dive/releases/download/v${DIVE_VERSION}/dive_${DIVE_VERSION}_linux_amd64.deb"
sudo apt install ./dive_${DIVE_VERSION}_linux_amd64.deb
```

To analyze an image using Dive, use the following command.

```bash
dive php:8.2-apache
```

When executing the command, Dive opens an interactive terminal that displays each layer created based on the instructions provided in the Dockerfile.

![Dive Image Details]({{ site.baseurl }}/assets/img/docker-course/dive.png)


### Understanding Dive Interface

When Dive opens, you will see the terminal as two sections.

In the left panel, the dive displays all layers built during image creation, based on the Dockerfile instructions.

In the right panel, it displays the files in a tree structure and highlights changes in files with different colors.

- Green - Added new files
- Orange - Modified files
- Red - Removed files

At the bottom, Dive displays the Image efficiency score, total wasted space, and layer details.

### Image Efficiency Score

Dive assigns a score to images ranging from 0 to 100 based on duplication and optimization within layers.

A high score, which is close to 100, indicates that there is minimal duplication and the layers are well optimized.

In the real world, we can utilize Dive with CI/CD pipelines to ensure the efficiency of images in an automated manner.

### SlimToolkit

SlimToolkit is a tool used to optimize Docker images by removing unnecessary files and layers that do not affect the application’s functionality.

It analyzes your Docker image, identifies unused or redundant components, and then creates a new, optimized image that’s smaller, faster, and more secure.

![SlimToolkit Diagram]({{ site.baseurl }}/assets/img/docker-course/SlimToolkit.gif)

- Before optimization, the Docker image size is, for example, 500MB.
- The slim process starts with the slim build command.
- The tool analyzes the Docker image to find unnecessary layers and files.
- It ignores the files and directories specified in an ignore.txt file.
- After identifying unnecessary files and layers, it removes them and creates a new optimized image.
- Along with the new image, it generates a JSON report named slim.report.json detailing the slimming process.

> SlimToolkit does not modify the original Docker image. Instead, it creates a new optimized Docker image.

### Installing SlimToolkit

```bash
sudo curl -sL https://raw.githubusercontent.com/slimtoolkit/slim/master/scripts/install-slim.sh | sudo -E bash -
```

Use SlimToolkit with the following command:

```bash
slim build --target <image-name:tag> --tag <new-image-name:tag>
```

You can specify the files and directories you want to exclude during the slimming process using the ignore.txt file.

### Optimizing Python Flask application using SlimToolkit

First, create an ignore.txt file and specify the file names or directory paths that should not be removed during the slimming process. For example:

```bash
/app
/usr/local/bin/flask
```

In this example, the paths specified are where the application code and essential files needed to run the Flask application are located.

Now, slim the Docker image using the following command.

```bash
sudo slim build --preserve-path-file ignore.txt --target python-flask-app:1.0  --tag python-flask-app:slimmed
```

This command will create a new optimized Docker image with the specified tag.

```bash
zafar@zserver:~/docker-projects/python-flask-app$ docker images
IMAGE                                  ID             DISK USAGE   CONTENT SIZE   EXTRA
php:8.2-apache                         1628cad804f8        714MB          183MB    U
python-flask-app:1.0                   4a0ecf521aa8        188MB         45.5MB    U
python-flask-app:slimmed               133730f76dd5       45.5MB         12.5MB
```

You can see the size difference between the original and the slimmed Docker images. 

---

### Docker Init

docker init is a command-line utility that automatically creates a Dockerfile and related configuration files.

It detects the programming language or framework of your project and generates all necessary files to containerize it using best practices.

It creates a Dockerfile and a compose.yaml file for multi-container setups, and a .dockerignore file to exclude unnecessary files from the image build context.

> Don't confuse **docker init** with **docker-init**. Since, **docker-init** is a process that acts as the init process (PID 1) within a Docker container.

The following diagram shows the docker init workflow.

![Docker Init Diagram]({{ site.baseurl }}/assets/img/docker-course/docker_init.gif)

- Developer runs the docker init command inside the project directory.
- docker init scans your project code files.
- Prompts you with configuration questions based on the detected language/framework.
- Once you have answered all the prompts, docker init creates the Dockerfile, compose.yaml, and .dockerignore files.

> Docker Init works with Docker Desktop. On Ubuntu Server, we have to manually define dockerfiles.

```bash
git clone https://github.com/techiescamp/docker-course.git
cd docker-course/docker-init
```

Inside the **docker-init** folder, you’ll see two files, **app.py** and **requirements.txt**.

### app.py

```bash
# app.py
from flask import Flask

app = Flask(__name__)

@app.route('/')
def hello_docker():
    return '<h1> hello world </h1>'

if __name__ == '__main__':
    app.run(debug=True, host='0.0.0.0')
```

### requirements.txt

```bash
flask
gunicorn
```

**requirements.txt** lists the dependencies needed to run the Flask application.
Docker init uses the dependency file to detect the language of your application automatically.

Lets, run the following command in the same directory to start the process.

```bash
docker init
```

It will then prompt you for the Python version, listening port, and the command to run the app, suggesting default values.

Once you have answered all questions, docker init will create three files: .dockerignore, compose.yaml, and Dockerfile.

---

## Linting Dockerfiles

Linting is the process of checking code for errors, bad practices, or inconsistencies using a linter utility.

To lint Dockerfiles, we have an open source utility called **Hadolint**.

### What is Hadolint?

Hadolint is an open-source command-line tool that analyzes Dockerfiles for errors, security risks, and inefficiencies. It’s written in Haskell and is widely used in DevOps pipelines to enforce Dockerfile best practices automatically. 

Hadolint works by reading and parsing your Dockerfile into an Abstract Syntax Tree (AST). It then checks each instruction and argument against a set of predefined rules covering syntax, security, and performance best practices.

![HadoLint Diagram]({{ site.baseurl }}/assets/img/docker-course/HadoLint.gif)

When Hadolint finds issues, it reports them along with severity levels to help you prioritize fixes.

The severity of the issues is categorized as follows:

- **Info** – You get suggestions for improvement in info. It is considered less severe.
- **Style** – Related to formatting or structure of the Dockerfile like using indentation or using long single lines, etc.
- **Warning** – Less critical issues and minor security concerns that need improvement.
- **Error** – These issues are severe and could potentially relate to security vulnerabilities or major best practice violations.

### Installing Hadolint on Linux

Download the latest Linux Hadolint package:

```bash
wget -O hadolint https://github.com/hadolint/hadolint/releases/download/v2.12.0/hadolint-Linux-x86_64
```

Move it to the /usr/local/bin directory.

```bash
sudo mv hadolint /usr/local/bin/hadolint
```

Give execution permission to Hadolint using the command given below.

```bash
sudo chmod +x /usr/local/bin/hadolint

hadolint --version # check if hadolint is installed on the system
hadolint --help # see the available commands for hadolint
```

### Lint Dockerfiles Using Hadolint

Lets lint the following Dockerfile with the HadoLint:

```bash
FROM ubuntu:latest
RUN apt-get update
RUN apt-get install -y curl
RUN apt-get update && apt-get install -y curl
RUN echo "hello world" | grep "world" | wc -l
CMD ["echo", "Hello, world!"]
```

Run Hadolint against the above Dockerfile. Ensure you have the Dockerfile in the same directory you are running the hadolint command.

```bash
hadolint Dockerfile
```

You will get the following output with a Warning and Info message with the rule numbers and remediation info as shown in the image below.

![HadoLint Dockerfile]({{ site.baseurl }}/assets/img/docker-course/hadolint-dockerfile.png)

When working with CI/CD systems, we need to standardize all the configuration and rules for linting. Here is where hadolint.yaml configuration files come in to picture.

### hadolint.yaml

handolint.yaml is the configuration file for Hadolint. You can customize how Hadolint handles the output and recommendations using the configuration file.

This is particularly useful when you want standardization and customization for linting Dockerfile across organizations or projects. This template can be shared with developers or DevOps engineers who develop Dockerfiles.

Let’s take a look at the key parameters.

- failure-threshold: To specify at what level the lint should fail or give a non-zero exit code.
- ignored: If you want to ignore specific rules.
- override: This parameter is specifically useful if you want to override the severity of specific rules.
- trustedRegistries: If you want to allow only images from specific container registries.

Given below is an example of .hadolint.yaml file.

```bash
failure-threshold: warning
format: tty
ignored:
- DL3007
override:
  error:
  - DL3015
  warning:
  - DL3015
  info:
  - DL3008
  style:
  - DL3015
trustedRegistries:
- docker.io
- techiescamp.com:5000
- "*.gcr.io"
- quay.io
```

You can use the config file with Hadolint as shown below.

```bash
hadolint --config .hadolint.yaml Dockerfile
```

### Hadolint in Docker Build Pipelines

As a standard practice, Developers can run Hadolint while creating Dockerfiles before pushing it to the repository. However, the key use of Hadolint comes in the Docker Image build pipeline.

If the Dockerfile does not adhere to the Hadolint rules, the build fails and developers can use the Hadolint feedback to rectify the issues with Dockerfile.

You can customize the rules and configs using the hadolint.yaml file as per your organization’s standards and project requirements.

---

## Docker Image Security

Docker images can also be a security risk if they contain vulnerabilities. It could be issues in libraries, vulnerabilities in application dependencies, container misconfigurations etc.

One way to ensure the security of Docker images in the software supply chain is by scanning for vulnerabilities every time the images are built. This practice can begin on the developers laptop and has to be extended to CI/CD pipelines, where Docker images are built for deployment.

### Trivy Security Scanner

Trivy is an **open-source security scanner** that checks for vulnerabilities in containers and other artifacts. It uses an internal database called **trivy-db**, which holds details about various vulnerabilities. AquaSecurity created and maintains Trivy.

This tool can access a broad range of vulnerability information from various databases, including the **National Vulnerability Database** (NVD), **Red Hat Security Data**, and **Alpine SecDB**. It uses this data to identify security issues.

During a scan, Trivy compares the software packages and libraries in a directory or container image with the data in its vulnerability database. If it finds a match, it means there's a vulnerability in the package or library.

Here is an image that shows the Trivy scanning workflow.

![Trivy Workflow]({{ site.baseurl }}/assets/img/docker-course/trivy.gif)

### Install Trivy

```bash
sudo apt-get install wget apt-transport-https gnupg lsb-release
wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key | gpg --dearmor | sudo tee /usr/share/keyrings/trivy.gpg > /dev/null
echo "deb [signed-by=/usr/share/keyrings/trivy.gpg] https://aquasecurity.github.io/trivy-repo/deb $(lsb_release -sc) main" | sudo tee -a /etc/apt/sources.list.d/trivy.list
sudo apt update -y
sudo apt install trivy
```

To verify the installation and understand all the available options

```bash
trivy -v
trivy -h
```

### Scan Docker Images

Scanning Docker Images using Trivy is very easy.

```bash
trivy image <IMAGE_NAME>
```

![Trivy Scan]({{ site.baseurl }}/assets/img/docker-course/trivy-scan.png)

Trivy can scan for vulnerabilities of a specific severity. To do this, use the --severity <severity> flag to specify the vulnerability severity.

```bash
trivy image --severity CRITICAL php-mysql-demo:latest
```

Trivy can also give output in JSON format. To do this, use the --format json flag, it will display the scanned results in JSON format.

```bash
trivy image --format json php-demo:latest
```

### Trivy SBOM

A Software Bill of Materials (SBOM) is a complete list of all components used in a software application, such as a library, framework, and module, including their versions.

Trivy uses **SPDX** and **CycloneDX** formats to generate SBOM for Docker images.

In Trivy, SPDX is primarily used for license agreement and copyright tracking, while CycloneDX focuses more on security and supply chain risk management.

Use the command below to generate SBOM in CycloneDX or SPDX format.

```bash
trivy image --format spdx-json --output result.json jawedzafar/php-mysqli:8.2 # creates result.json file
trivy image --format cyclonedx --output result.json php-demo:latest
```

Trivy has the ability to use the SBOM report to perform offline vulnerability scans.

```bash
trivy sbom result.json
```

### Integrating Trivy Scans into image Build Pipelines

Organizations use trivy to scan for vulnerabilities in CI/CD pipeline to ensure a secure image is getting deployed in production.

When using in CI/CD pipeline, the pipeline job should fail if there is any vulnerability in the image. The severity depends on the organizations security compliance. For example, some organization may have strict guidelines to fail the build for both HIGH and CRITICAL severities.

```bash
trivy image --severity HIGH,CRITICAL  --exit-code 1 jawedzafar/php-mysqli:8.2
```

A recommended approach is to use a Trivy config file to set the defaults for a scan.

Here is an example of **trivy.yaml** file

```bash
timeout: 10m
format: json
dependency-tree: true
list-all-pkgs: true
exit-code: 1
output: result.json
severity:
  - HIGH
  - CRITICAL
scan:
  skip-dirs:
    - /lib64
    - /lib
    - /usr/lib
    - /usr/include

  security-checks:
    - vuln
    - secret
vulnerability:
  type:
    - os
    - library
  ignore-unfixed: true
db:
  skip-update: false
```

Here is the syntax to use the config file

```bash
trivy image --config path/to/trivy.yaml php-demo:latest
```

---

## Docker Scout

Docker Scout is a native Docker tool to scan, analyze, and secure container images.

It’s built to give developers quick insights into image vulnerabilities, dependencies, and supply chain risks right from the Docker CLI or Docker Desktop.

You can also use Docker Scout as a standalone binary without Docker Desktop or Docker CLI.

```bash
curl -fsSL https://raw.githubusercontent.com/docker/scout-cli/main/install.sh -o install-scout.sh
sh install-scout.sh
```

The below command scans your Docker image and gives you an overview of every vulnerability in the image.

```bash
docker scout quickview <image-name>
```

![Docker Scout Quickview]({{ site.baseurl }}/assets/img/docker-course/scout-quickview.png)

To get a detailed report of Docker images, run the following command.

```bash
docker scout cves <image-name>  # detailed report on image and scan for vulnerability
docker scout recommendations <image-name>   # gives recommendations on you image
docker scout cves <image-name> --only-severity high # report of a specific vulnerability.
docker scout compare --to <image-01> <image-02> # compares two images.
```

### Enable Docker Scout for an Image Registry

You can integrate **Scout** with your Docker images registry.

Login to your docker image registry:

```bash
docker login
# or
docker login -u <username> -p <password>
```

After login, enroll your Docker account with Docker Scout using the enroll command.

```bash
docker scout enroll <account-name>
```

The final step is to enable Docker Scout for your image repository.

```bash
docker scout repo enable --org <account-name> <account-name>/<image-repository>
# free plan allows only for one image
```

```bash
https://scout.docker.com/
```

![Docker Scout Result]({{ site.baseurl }}/assets/img/docker-course/scout-docker-registry.png)

---

## Container Image Signing & Verification

Docker image signing is a security practice that adds a signature to the image for the trust and integrity of container images.

When you build a Docker image, it becomes just another file. Anyone can copy it, rename it, or push it to a registry.

Now imagine someone replaces your image with a modified version that has a hidden script or malware. Without checks, your system will still pull and run it.

Image signing helps verify that the image came from a trusted source and that it has not changed since it was built. It is like putting a digital signature on your Docker image.

This signature basically proves two things.

- Who created it (authenticity)
- That it hasn not been changed since it was signed (integrity)

### Tools for Image Signing

- **Key-based signing** (traditional method): The private key is used to sign the image. The public key is used by others (or automation) to verify that signature.
- **Keyless signing** (modern method): No need to create or store keys manually.
Cosign uses your identity (like your GitHub, Google, or corporate SSO account) to sign automatically.

### Cosign

Cosign is an open-source tool which is created to make container signing, verification, and storage of signatures simple and secure.

**Cosign** integrates with most container registries and supports both keyless and key-based signing.

![Cosign]({{ site.baseurl }}/assets/img/docker-course/cosign.gif)

Once the Docker image is pushed to a container registry, the developer signs the image using tools like Cosign using a private key.

- This is to prove who built the image and that it has not been modified by anyone after signing.
- In K8s deployments, before pulling or running an image, a verification step is performed.
- An admission controller like **Kyverno** checks whether the image is signed and trusted.
- If the image is correctly signed and verified, then it will be deployed.
- If it’s not signed, or the signature is invalid/trusted key is missing, the deployment is blocked.

### Sign Docker Image

To sign a Docker image, make sure Cosign is installed in your system, and the Docker image should be in a **container registry**.

First, lets install cosgin:

```bash
# binary
curl -O -L "https://github.com/sigstore/cosign/releases/latest/download/cosign-linux-amd64"
sudo mv cosign-linux-amd64 /usr/local/bin/cosign
sudo chmod +x /usr/local/bin/cosign
```

Verify:

```bash
zafar@zserver:~$ cosign version
  ______   ______        _______. __    _______ .__   __.
 /      | /  __  \      /       ||  |  /  _____||  \ |  |
|  ,----'|  |  |  |    |   (----`|  | |  |  __  |   \|  |
|  |     |  |  |  |     \   \    |  | |  | |_ | |  . `  |
|  `----.|  `--'  | .----)   |   |  | |  |__| | |  |\   |
 \______| \______/  |_______/    |__|  \______| |__| \__|
cosign: A tool for Container Signing, Verification and Storage in an OCI registry.

GitVersion:    v3.0.6
GitCommit:     f1ad3ee952313be5d74a49d67ba0aa8d0d5e351f
GitTreeState:  clean
BuildDate:     2026-04-06T21:39:58Z
GoVersion:     go1.25.7
Compiler:      gc
Platform:      linux/amd64
```

Next step is to create a new key pair for signing, run the following command to create a key pair.

```bash
zafar@zserver:~$ cosign generate-key-pair
Enter password for private key:
Enter password for private key again:
Private key written to cosign.key
Public key written to cosign.pub
```

Once you specify the password, it creates two key files **cosign.key** and **cosign.pub**.

After creating the key pair, run the following command to sign the Docker image using the private key.

```bash
cosign sign --key cosign.key <registry-name>/<image-name>:<tag>
```

The command will ask you to enter the password for the private key. Cosign will then sign the image and store the signature in the registry alongside the image as a SHA file.

To verify if an image is signed, run the following command:

```bash
cosign verify --key cosign.pub <registry-name>/<image-name>:<tag>
```

```bash
zafar@zserver:~$ cosign verify --key cosign.pub jawedzafar/php-mysqli:8.2

Verification for index.docker.io/jawedzafar/php-mysqli:8.2 --
The following checks were performed on each of these signatures:
  - The cosign claims were validated
  - Existence of the claims in the transparency log was verified offline
  - The signatures were verified against the specified public key

[{"critical":{"identity":{"docker-reference":"index.docker.io/jawedzafar/php-mysqli:8.2"},
"image":{"docker-manifest-digest":"sha256:252e9efc949748531b41c4843f59b0516e91e99884f03822351409a7e7616d56"},
"type":"https://sigstore.dev/cosign/sign/v1"},"optional":{}}]
```

If you modify the image after signing it and try to verify it again, the verification will fail as shown below

---

## Common Real World Docker Tasks

### Running Custom Shell Scripts In Docker

In many real-world projects, you’ll need to run a custom shell script inside a container, often passing different arguments to control how the script behaves.

Let’s see how you can do that easily using **ENTRYPOINT** and **CMD** in Docker.

Start by creating a simple shell script, **script.sh**, that accepts arguments.

```bash
#!/bin/bash

arg1=${1}
arg2=${2}
arg3=${3}

# Run the script in an infinite loop
while true; do
    echo "Argument 1: $arg1"
    echo "Argument 2: $arg2"
    echo "Argument 3: $arg3"
    sleep 1
done
```

In this example, we have a shell script that accepts three command-line arguments ($1, $2 & $3).

The **while true loop** then runs indefinitely, printing the values of arg1, arg2, and arg3 in each iteration with a one-second delay between each iteration.

Create the Dockerfile with the following contents. Here we are copying the shell script in the`root location.

```bash
FROM ubuntu:latest
LABEL maintainer="AJ ZAFAR <bosphoran@gmail.com>"
RUN apt-get update && apt-get install -y \
    # Add any necessary packages here
    && rm -rf /var/lib/apt/lists/*
# Copy the script to the container
COPY ./script.sh /
RUN chmod +x /script.sh
# Set the entrypoint to the script with CMD arguments
ENTRYPOINT ["/script.sh"]
CMD ["hulk", "batman", "superman"]
```

This **entrypoint** of the container is set to **/script.sh**, which means that this script will be run whenever the container is started.

The **CMD** line specifies default arguments for the script if none are provided when the container is started. In this case, the default arguments are hulk, batman, and superman.

Now, build the Docker image **script-demo**, using the following command.

```bash
docker build -t script-demo .
```

Now run the container image.

```bash
docker run --name script-demo -d script-demo
```

Now, lets check the container logs.

```bash
docker logs demo-script -f
```

You can now pass the **CMD** arguments at the end of docker run command. It will override the arguments passed in the Dockerfile. For example,

```bash
docker run --name demo-script2 demo-script.sh:latest false spiderman hulk
```

Here "false spiderman hulk" will override "true", "batman", "superman" present in the docker image

> Note: If you don’t want to pass arguments to the shell script, you can ignore the CMD arguments and execute the shell script directly using Entrypoint.

---

### Running Containers as Non-Root User

Running containers as a non-root user is one of the key best practices in container security. It helps prevent attackers from gaining host-level control if a container is ever compromised.

A non-root user is simply a regular system user who does not have administrative (root) privileges. They can perform basic tasks but cannot modify system files or affect other users’ data unless explicitly granted permission.

> Understand that host root and container root are not the same in terms of how much control they have over system resources.

The root user on the host system has complete control over all resources and processes on that system. The host root can access any files, run any commands, install or uninstall software, and change system configurations.

The root user inside a container has similar privileges within the container.

> Container root (UID 0) is the same as host UID 0 unless user namespaces are explicitly configured.

This means that while it has root privileges within the container, it does not have direct access to the host system's resources due to Namespace, Control groups and security profiles like seccomp.

Meaning, the container runtime removes some special permissions (called capabilities) and blocks certain system calls (using seccomp) to make the container environment more secure, even though the user ID (UID) remains the same inside the container and on the host.

### Non-Root User Image Considerations

Most applications assume they’re running as root and try to access system directories or privileged ports, so you must adjust these configurations to make them work safely as a regular user.

### Creating a Non-Root User

The groupadd and useradd commands create a new user ( eg: flaskuser) and group (eg: flaskgroup) with a fixed UID and GID.

```bash
RUN groupadd -g 1001 flaskgroup && useradd -u 1001 -g flaskgroup -m flaskuser
```

> UIDs 1000 and above are generally assigned to non-root users (regular users). Many Linux distributions assign UIDs starting from 1000 or 1001 to non-root users by default. The UID 0 is reserved for the root user.

### File Permissions

When you run applications, they often expect to write to specific directories. It could be a log file or a runtime configuration file.

You need to ensure proper ownership and permissions are set during the image build process.

**For example**, let's say your app wants to write logs to /app/logs. In that case, your non-root user configuration would look like this:

```bash
RUN mkdir -p /app/logs && chown flaskuser:flaskgroup /app/logs && chmod 755 /app/logs
```

If these permissions aren’t configured, your application might crash with errors like Permission denied when it tries to write to a directory it doesn’t own.

### Port Binding

Non-root users can't bind to privileged ports (ports < 1024).

**For example**, if you run an Nginx or PHP container and check its processes, you can see that the master process is running as root.

```bash
zafar@zserver:~$ docker top php-demo-container | awk '{print $1, $8, $9}'
UID CMD
root apache2 -DFOREGROUND
www-data apache2 -DFOREGROUND
www-data apache2 -DFOREGROUND
www-data apache2 -DFOREGROUND
www-data apache2 -DFOREGROUND
www-data apache2 -DFOREGROUND
```

Now, if you want to run apps like these as non-root, you need to change the default port to another port , let's say 8080, during the build time.

### Service Dependencies

Let’s say you want to containerize an application that assumes it can write logs to a system directory like /var/log/ (which requires root).

Additionally, it communicates with another process through /var/run/ for inter-process communication (IPC), which also requires root.

In these cases, you might want to configure the application or refactor it to write to different locations. For example:

- Logs are written to /app/logs instead of /var/log.
- IPC files are created in /app/run instead of /var/run.

In the Dockerfile, the application runs as appuser, a non-root user created during the build process. Ownership of /app/logs and /app/run is granted to appuser.

```bash
ENV APP_USER=appuser
ENV LOG_DIR=/app/logs
ENV RUN_DIR=/app/run

# Create a non-root user
RUN useradd -ms /bin/bash $APP_USER

# Create alternative directories for logs and runtime files
RUN mkdir -p $LOG_DIR $RUN_DIR && chown -R $APP_USER:$APP_USER $LOG_DIR $RUN_DIR
```

### Configuration Files & Runtime Directories

There are applications that often assume root access and expect to create or modify directories in locations requiring root privileges.

For such applications, you might need to:

- Allow non-root user access to those directories or files.
- Pre-create directories with the correct permissions.
- Configure alternative paths for files and directories

---

### Build non-root User Container Image

In this example, you will learn how to create a Docker image with a non-root user.

For this demonstration, let's use Nginx as the application, and we'll configure a non-root user who can only manage the Nginx web server. This user will have no additional privileges, such as creating directories or modifying files outside the Nginx web server.

Below is the Dockerfile to create the Nginx Docker image with a non-root user:

```bash
FROM nginx:alpine

RUN addgroup -g 1001 -S nginxgroup && adduser -u 1001 -S -G nginxgroup nginxuser

RUN mkdir -p /var/run/nginx /var/cache/nginx/client_temp /var/cache/nginx/proxy_temp /var/cache/nginx/fastcgi_temp /var/cache/nginx/scgi_temp /var/cache/nginx/uwsgi_temp \
    && chown -R nginxuser:nginxgroup /run /var/cache/nginx /var/run/nginx /var/log/nginx /etc/nginx /usr/share/nginx/html

RUN sed -i 's/listen       80;/listen       8096;/g' /etc/nginx/conf.d/default.conf
RUN sed -i 's/\/var\/run\/nginx.pid/\/var\/run\/nginx\/nginx.pid/g' /etc/nginx/nginx.conf

USER nginxuser

EXPOSE 8096

CMD ["nginx", "-g", "daemon off;"]
```

- This Dockerfile uses nginx as the base image.
- It creates a group nginxgroup with GID 1001 and a user nginxuser with UID 1001 and gives the non-root user ownership of Nginx directories. Creating user and group with IDs is particularly useful when you run this image on Kubernetes. By explicitly setting the UID and GID in the Docker image, you can align these IDs with the securityContext settings in Kubernetes.
- Then, it creates the required temporary directories for Nginx and assigns them to the non-root user so that Nginx can write its data on the required directories.
- For example, the /var/cache/nginx/client_temp directory stores temporary HTTP request data sent and received by Nginx. /var/run/nginx will hold the PID created by Nginx during its startup (explained in step 6).
- The SED command changes Nginx to listen on port 8096 because the ports below 1024 are privileged ports, which means only users with root permission can use them by default.
- During startup, Nginx writes its PID to /var/run directory by default. The second SED command changes the PID file location to /var/run/nginx because the non-root user has no permission on the /var/run directory. By creating and changing /var/run/nginx directory permission to a non-root user, Nginx can save the PID file in it.
- Then, it switches the user to a non-root user nginxuser because if we don’t switch to a non-root user, it will run as a root user by default.
- And it exposes a port 8096 for external access.
- Finally, the CMD command tells Docker to start Nginx in the foreground. It daemon off keeps Nginx running and makes sure it doesn’t go into the background, which would cause the container to stop.

Run the following command to build the docker image.

```bash
docker build -t nginx-non-root:1.0.0 .
```

> Assigning a specific ID to user and group is typically helpful if you want to run this container in Kubernetes as non root user using securityContext. Also, if you don’t specify an ID, it will assign a random ID to user and group which will cause error when deploying in Kubernetes if the correct ID is not specified.

Now that the image is built, we will run the container and test it to see if it is running as a non-user user.

```bash
docker run -d -p 8096:8096 nginx-non-root:1.0.0
```

If your container is running, check the current user using the docker inspect command.

```bash
zafar@zserver:~/docker-projects$ docker inspect -f '{{.Config.User}}' b860c65e00a6
nginxuser
zafar@zserver:~/docker-projects$
```

Or

```bash
zafar@zserver:~/docker-projects$ docker exec -it b860c65e00a6 sh
/ $ whoami
nginxuser
/ $ exit
zafar@zserver:~/docker-projects$
```

You can see the container is running as a non-root user nginxuser.

Now, If you want to run the non-root container as a root user, run the following command.

```bash
docker run -u 0 -it nginx-non-root:1.0.0 sh
```

The **-u 0** option tells the container to run as the root user, **0** is the **UID** of the root user.

```bash
zafar@zserver:~/docker-projects$ docker run -u 0 -it nginx-non-root:1.0.0 sh
/ # whoami
root
/ # exit
zafar@zserver:~/docker-projects$
```

> UID 0 (User ID 0) is reserved for the root user, who is the superuser or administrator with full access to all commands and files on the system.

### Testing Non-Root User Behavior

Now, let's check what the non-root user can and cannot do. Make sure container is runnning.

```bash
docker run -it nginx-non-root:1.0.0 sh
```

### Check Directory Permissions

Let's check the permission of non-root user in directories, for that, we will try to create a file on /var and /usr/share/nginx/html directory.

```bash
zafar@zserver:~/docker-projects$ docker exec -it b860c65e00a6 sh
/ $ cd /var
/var $ touch test
touch: test: Permission denied
/var $ exit
zafar@zserver:~/docker-projects$
```

You'll receive a Permission denied error as shown above.

Now, let's try to create a file in /usr/share/nginx/html directory.

```bash
zafar@zserver:~/docker-projects$ docker exec -it b860c65e00a6 sh
/ $ cd /usr/share/nginx/html
/usr/share/nginx/html $ touch test
/usr/share/nginx/html $
/usr/share/nginx/html $ exit
zafar@zserver:~/docker-projects$
```

This time, the command will succeed because the non-root user nginxuser has the necessary permissions for this directory

### Try Installing a Package

Now, let's try to install a tool using the non-root user, run the following command to install **curl**.

```bash
zafar@zserver:~/docker-projects$ docker exec -it b860c65e00a6 sh
/ $ apk add curl
ERROR: Unable to open log: Permission denied
/ $
/ $ exit
zafar@zserver:~/docker-projects$
```

You got this error because the non-root user doesn’t have permission to install software.

---

## Image Tags Vs Digests

When you move from development to production, one question always comes up,
Should I use image tags or image digests?

### Tags vs Digests

A tag is a readable label like **nginx:1.10.0**. A digest is the unique fingerprint of an image like **nginx@sha256:a4f8c7e...**.

While both point to an image, tags can change, digests cannot.

Most CI/CD systems (GitLab, ArgoCD, Flux, Jenkins, etc.) resolve tags to digests automatically behind the scenes.

Overall, here is why digests are used in real environments.

- Tags can be retagged or replaced. Digests are permanent.
- Prevents supply-chain attacks where a tag is re-pushed with malicious content.
- Easier to prove exactly which binary ran in production.
- Ensures rollback points always map to the same image.

**For local dev / staging**, use tags (v1.2.0, latest, etc.) for speed and readability.
**For production deployments**, lock images by digest for guaranteed reproducibility.

### Keep Docker Container Running for Debugging

In this lesson, we will look at how to keep a Docker container running for testing, debugging, and troubleshooting purposes.

To keep the container running, a foreground process needs to be added to the Docker Entrypoint.

The official Nginx image includes the Nginx foreground process in the Dockerfile. However, the base Ubuntu image does not have an Entrypoint for the foreground process, which causes the container to exit immediately after running the docker run command.

### Ways to Keep the Container Running

Here is a basic Dockerfile with an ENTRYPOINT that will keep the container running without getting terminated:

```bash
FROM ubuntu:latest
ENTRYPOINT ["tail", "-f", "/dev/null"]
```

Additionally, here are three methods to keep the container running with the docker run command:

**Method 1**: Use the -t (pseudo-tty) docker parameter to keep the container running.

```bash
docker run -d -t ubuntu
```

**Method 2**: Run the container directly and pass the tail command via arguments.

```bash
docker run -d ubuntu tail -f /dev/null
```

**Method 3**: Method 3: Execute a sleep command for infinity:

```bash
docker run -d ubuntu sleep infinity
```

Once you have a running container, you can attach the container to the terminal session using the exec parameter as shown below, where 0ab99d8ab11c is the container ID:

```bash
docker exec -it 0ab99d8ab11c /bin/bash
```

This will attach the container to the terminal session, allowing you to interact with it.

---

### Run Docker in Docker Container

In this topic, we’ll see how to run Docker inside a Docker container, often called Docker in Docker or **DinD**.

When you plan to use Docker-based dynamic CI/CD agents, Docker in Docker comes as a must-have functionality.

There are two ways to achieve Docker in Docker

- Run Docker by mounting docker.sock (DooD Method)
- Run a full Docker daemon inside a container using the docker:dind image.

### Method 1: Docker in Docker Using [/var/run/docker.sock]

Given below is an example diagram of how Docker in Docker by mounting the Docker socket (/var/run/docker.sock) works.

![Docker in Docker]({{ site.baseurl }}/assets/img/docker-course/dind.png)

First, understand what **/var/run/docker.sock** is.

It’s the default Unix socket used by the Docker daemon to accept API calls.

Docker daemon by default listens to docker.sock. If you are on the same host where Docker daemon is running, you can use the /var/run/docker.sock to manage containers.

For example, if you run the following command, it would return the version of docker engine.

```bash
curl --unix-socket /var/run/docker.sock http://localhost/version
```

Let’s see how to run Docker in Docker using docker.sock.

To test this setup, start a Docker container in interactive mode, mounting the Docker.sock as a volume. We will use the official Docker image.

```bash
docker run -v /var/run/docker.sock:/var/run/docker.sock -ti docker
```

> Just a word of caution: If your container gets access to docker.sock, it means it has more privileges over your docker daemon. So when used in real projects, understand the security risks, and use it.

Now, from within the container, you should be able to execute Docker commands for building and pushing images to the registry.

Here, the actual Docker operations happen on the VM host running your base Docker container rather than from within the container.

Meaning, even though you are executing the Docker commands from within the container, you are instructing the Docker client to connect to the VM host Docker Engine through docker.sock.

Once you are inside the container, execute the following Docker commands to pull an Ubuntu image and list the images.

```bash
docker pull ubuntu
docker images
```

To test it, create a Dockerfile inside the container:

```bash
mkdir test && cd test
vi Dockerfile
```

Copy the following Dockerfile contents to test the image build from within the container.

```bash
FROM ubuntu:18.04
LABEL maintainer="Bibin Wilson <contact@techiescamp.com>"
RUN apt-get update && \
    apt-get -qy full-upgrade && \
    apt-get install -qy curl && \
    curl -sSL https://get.docker.com/ | sh
```

Now build the image.

```bash
docker build -t test-image .
```

### Method 2: Docker in Docker Using dind

![Docker in Docker 2]({{ site.baseurl }}/assets/img/docker-course/dind2.png)

This method actually creates a child container inside a container. Use this method only if you really want to have the containers and images inside the container. Otherwise, I would suggest you use the first approach.

For this, you just need to use the official docker image with dind tag. The dind image is baked with required utilities for Docker to run inside a docker container.

> Note: This requires your container to be run in privileged mode.

Start with creating a container named dind-test with docker:dind image.

```bash
docker run --privileged -d --name dind-test docker:dind
```

This runs the container in privileged mode, which allows it to create inner containers.

Now, log in to the container using exec.

```bash
docker exec -it dind-test /bin/sh
```

Inside this container, you can run Docker commands just like on a regular host.

```bash
docker pull ubuntu
docker images
```

### Key Considerations

- Only use Docker in Docker if it is necessary. Before migrating any workflow to the Docker-in-Docker method, conduct proof of concepts (POCs) and sufficient testing.
- When using containers in privileged mode, ensure that you receive necessary approvals from enterprise security teams regarding your plans.
- There are certain challenges when using Docker in Docker with Kubernetes pods.
- If you plan to use Nestybox (Sysbox), ensure that it is tested and approved by enterprise architects and security teams.

---

## Build Images for Multiple Architecture

In a real-world project environment, you might find virtual machines with different architectures. For example, there are servers with x86 CPU architecture and others based on ARM.

A Docker image made for x86 CPUs won't work on ARM machines. This means you might need to create Docker images for each architecture separately.

But, Docker offers a feature called multi-arch build. This lets you make a Docker image that works on various system architectures.

Let’s see how you can build your own multi-arch image.

- Docker Desktop because Buildx comes pre-installed.
- A valid Dockerfile.
- A container registry to push your image.

First, create a separate environment to build a multi-arch image.

On Docker Desktop, this is often preconfigured, so you can skip it. You can use the following command to list the builders.

```bash
docker buildx ls
```

If you want to create a builder, run the following commands.

```bash
docker buildx create --name mymultiarchbuilder --driver docker-container --use
```

This creates a new builder instance named mymultiarchbuilder that can be used to build multiple architectures.

- **--name mymultiarchbuilder** - Builders name.
- **--driver docker-container** - Runs the builds in an isolated container.
- **--use** - Makes this builder the active one.

Now, use the same command **docker buildx ls** to list the builders. You will get an output similar to this.

You can see a new builder is created with a new buildkit version and an additional supported platform.

Now, run the following command to start the builder and make sure it is ready to build images that support multiple architectures.

```bash
docker buildx inspect --bootstrap
```

Now you can build your image for multiple architectures and push it to a registry.

```bash
docker buildx build --platform linux/amd64,linux/arm64,linux/arm/v7 -t yourusername/yourimagename:tag --push .
```

- **--platform** → Specify the architectures.
- **-t** → tags your image.
- **--push** → Uploads the image to the container registry.

After the build finishes, anyone can run the image on their system. Docker will automatically fetch the correct version for their system.

---

## Enterprise Image Management

In this section, we will look at how all these pieces come together in an enterprise environment.

When organizations use containers at scale, things work a little differently.

You can’t just pull random images or build everything manually. Every image must follow strict standards, security checks, and CI workflows.

Here are some of the key questions we will explore.

- Where do you get certified base images from?
- How is a base image in an enterprise setup different from one in a public registry?
- What extra configurations do enterprise images need?
- How are platform tool images (like Prometheus or Grafana) built?
- What does a production-grade image build workflow look like?
- What does the image build workflow look like in CI/CD?
- What security features are mandatory in enterprise environments?
- How do we patch and update container images safely?
- Who owns container images in an enterprise? (Platform team vs Dev teams)
- What is image promotion across environments? (Dev, Staging, Prod)
- Should we use single registry or multiple registries per environment?
- What happens when a critical CVE is discovered? (Incident response)

### Base Image Management

A security-focused company will usually have a central platform or security team responsible for creating and maintaining all base images.

These base images go through security scans, patching, and compliance checks before being published.

The platform team maintains proper documentation listing all available base images. For example, base Ubuntu, Java, Python, Node.js, or Nginx images.

As a DevOps engineer or a developer, your job is to,

- Pull the approved base image from the internal registry.
- Add your application code, dependencies, and configs on top of it.
- Push the final image to your project-specific registry.

For Example, if you are building a Java app, you will start with something like.

```bash
FROM internal-registry.company.com/java-base:17
COPY app.jar /opt/app.jar
CMD ["java", "-jar", "/opt/app.jar"]
```

The base images managed by platform/security teams are typically refreshed monthly or when critical CVEs are found.

### New Base Image Approval Process

Sometimes, you might need a base image that is not in the approved catalog.

In that case, most organizations have a request and approval process. You cant just pull it from Docker Hub or build it on your own.

### Option 1: Platform Team Builds It

You raise a ticket or service request with a business justification stating why this new base image is required and what it will be used for.

The platform or security team reviews it, runs internal scans, and if approved, they build and publish it to the internal registry.

This process can take 1–2 weeks, depending on the complexity and approval flow.

---

## Production Image Promotion Workflow

Learning Docker is the easy part. The real challenge is understanding how containers are used in actual projects, and that is what most courses skip.

The process explained in this section will give you a solid foundation on how image builds, tagging, and promotions usually happen from dev to stage to prod in an enterprise environment.

### How Many Environments Do Real Projects Have?

- **Small projects** typically have three environments. For example, dev, stage, and prod. This is the minimum you need for a proper promotion flow.
- **Medium projects** add a dedicated QA (for functional testing, separate from dev) and a Performance environment for load testing. The stage environment acts as a pre-prod environment.
- **Enterprise projects** (finance, healthcare, government) often have many environments. For example, dev, SIT (system integration testing), QA, performance, staging, UAT (user acceptance testing), and production. It can be more.

> In this lesson, we will use a dev, stage and prod environments as example. It can scale to any number of environments. You just need to add more promotion steps in between.

In most enterprise projects, each environment means, a dedicated cloud account ( or subscription/project depending on the cloud provider). Each account has its own VPC, IAM roles, Security, etc.

![Docker in Docker]({{ site.baseurl }}/assets/img/docker-course/enterprise-env-architecture.png)

The key reason for this is, **blast radius isolation**. Meaning, how much of the environment is affected when something happens to the environment. So if a developer deletes something in dev, only dev is affected.

> In the production account, no one will have access to do anything manually (zero standing access in production). All the changes happen through CI/CD systems.

### Build Once, Deploy Everywhere

Image promotion means taking a verified container image and moving it across environments without rebuilding it. In practice, this usually means,

- Retagging the same image
- Or copying the image to another registry using tools like crane or skopeo.

![Docker in Docker]({{ site.baseurl }}/assets/img/docker-course/image-promotion.png)

### Container Registry Patterns

How many container registries should we have for a project?

The answer, as always, depends on the project and its requirements. For simplicity, let's assume we have dev, test, stage, and prod environments.

Here are the two most commonly used registry patterns in enterprises.

### Pattern 1: Per-Account registries

In this pattern, each environment gets its own dedicated container registry.

![Docker in Docker]({{ site.baseurl }}/assets/img/docker-course/container-registry-pattern-1.png)

Why separate registries?

- If the dev account is compromised, they can't touch the prod registry or its images. The accounts are completely separate.
- Each account has different permissions. For example, Dev teams may get full access to the dev registry for productivity. However, only the CI pipeline job can push to prod. No one can push or modify images in prod manually.
- It also helps comply with SOC2, HIPAA, and PCI-DSS for strong security controls.

> Note: This setup comes with more setup and CI/CD complexity (cross-account roles, replication, permissions).

### Pattern 2: One registry, env by tags

In this pattern, you have one container registry for multiple environments. All images are stored in the same repository.

Each environment for the app is represented using tags, not separate repositories.

![Docker in Docker]({{ site.baseurl }}/assets/img/docker-course/container-registry-pattern-2.png)

The CI/CD pipeline promotes the same image digest through environments by re-tagging. No rebuilds or cross-registry copies.

For example, in **Dockerhub**, you have to use naming conventions like **myorg/dev-payment-svc**, **myorg/stage-payment-svc**.

The key issue in this pattern is access control and no isolation between non-prod environments. Anyone with push access can push to both qa-payment-svc and stage-payment-svc.

```html
registry.company.com/
├── dev/
│   ├── payment-svc
│   ├── auth-svc
│   └── order-svc
├── stage/
├── release/
└── prod/
```

### Pattern 3: Two registries: nonprod and prod

In this pattern, the goal is to keep non-production and production environments in separate registries for stronger isolation and cleaner control.

Here is how it works.

- The CI pipeline builds and pushes the image to the nonprod registry.
- Automated tests, vulnerability scans, and reviews are run against this image. \
  After validation, one of two approaches is followed:
  - Promote the same digest to the prod registry (copy or retag, no rebuild).
  - Rebuild on main branch and push to the prod registry.

### Registry Storage and Garbage Collection

Think of 100+ services pushing images daily. What happens to registry storage? That is where garbage collection policies come in.

You can configure policies to automatically clean up old images based on certain criteria.

- Delete after a certain number of days or
- After a set number of images.

### Image Tagging Strategy

One of the key things that helps in how we track, promote, and trace images across environments is image tags.

Understanding the difference between tag types is important for production workflows.

There are two types of tags.

- **Immutable tag**: It is a tag that never changes and always points to the same image.
  - For example, tag based on the GitHub commit SHA ID (**registry/service:sha-abc1234**). It is a one to one mapping between the image and the exact source code that built it.
- **Mutable tag**: It is a tag that can be updated to point to a different image.
  - For example, the latest tag. It always points to the last pushed image. It is actually useful for quick testing.

> Note: Production deployments should always use immutable tags so we can always track exact version of the application. It also helps in easy rollback.

### Image Promotion Pipeline Architecture

Our pipeline is split into four stages. Each stage get triggered at a different point in the development lifecycle.

1. PR Check Workflow
2. Image Build workflow
3. Image promotion from dev to stage environment
4. Image promotion from stage to prod environment.

![Docker in Docker]({{ site.baseurl }}/assets/img/docker-course/image-build-promotion-pipeline.png)

### Branching Strategy

Every workflow in this pipeline is triggered by a branch event.

The branching strategy has a direct impact on how your Docker image CI works and how images move through environments.

Our pipeline follows a Git-flow style branching model. The key branches develop, release, and main drive the pipeline through dev, stage, and prod environments.

> There are other branching strategies like trunk based where all developers commit directly to a single main branch with short-lived feature branches.

The following image illustrates the git branching strategy.

![Docker in Docker]({{ site.baseurl }}/assets/img/docker-course/branching-strategy.png)

Here is the development flow.

- **feature/*:** Developers create feature branches from develop. When the feature is ready, they raise a PR. This triggers the PR check workflow.
**develop**: Merging a PR triggers the image build followed by deployment to the dev environment. This happens many times.
**release/*:** When a version is finalized for production deployment, the team cuts a release branch from develop. This triggers the stage promotion. QA and integration testing happen here. Also, all the bugfixes go directly on this branch.
**main**: Merging the release branch into main triggers the production promotion.

Now, let's say the app is running in production, and what if we want to fix something in production?

This is where we will create a **hotfix branch**.

For **emergency production fixes**, a **hotfix branch** is created from **main**, tested quickly, merged back to both main (prod deploy) and develop (so the fix is not lost).

![Docker in Docker]({{ site.baseurl }}/assets/img/docker-course/branching-strategy-hotfix.png)

### PR Check Workflow

When you raise a Pull Request (PR) to the develop branch, this workflow kicks in automatically.
![Docker in Docker]({{ site.baseurl }}/assets/img/docker-course/pr-check-workflow.png)

Here is how it works.

- The PR first checks out the files and folders on the GitHub repository.
- Then, it uses Hadolint to lint the Dockerfile that catches common issues (like missing version pinning or running as root).
- Then it uses Buildx to build the Docker image using the Dockerfile to check if the image builds without any issues.
- For the built image, it generates an SBOM (Software Bill of Materials) report using Trivy and uploads it to GitHub Actions artifacts.
- Then, it scans the Docker image using Trivy for vulnerabilities.
- Finally, it uses the container structure test to verify if it meets your compliance rules.

If the PR check passes, it means the Dockerfile has no issues, the image builds as expected, there are no known vulnerabilities in the image, and the container structure is compliant. Only then we will merge the PR.

### Building Multi-Arch Images for Dev

Once the PR is merged into the develop branch, the image build workflow triggers automatically.

![Docker in Docker]({{ site.baseurl }}/assets/img/docker-course/multi-arch-image-build.png)

Here is how it works.

- The build workflow starts with checking out the files and folders on the GitHub repository.
- Then it installs QEMU and Buildx to build multi architecture Docker image. In the workflow, we specify to build linux/amd64 and linux/arm64 architecture.
- Logs in to DockerHub using credentials stored in GitHub Secrets.
- Builds and pushes the image to the dev container registry with two tags. An immutable git commit SHA (e.g., sha-2b2e927) based tag and a mutable tag (latest). The SHA tag is what we will use later to promote the exact image to stage and prod.
- Deploys the image to the dev environment (Kubernetes, ECS, or whatever your team uses).

Now, the built image is running in dev. The Dev team uses this deployment to test their changes for the latest code.

> Important Note: Developers keep pushing changes through PR's and the pipeline builds new images deploys it to to dev environment until a version is finalized for production. Once the code is stable and ready for deployment, the specific image version gets promoted to the stage environment.

### Promote Docker Image from Dev to Stage Registry

This is where the "build once, deploy everywhere" principle starts.

The dev team picks a specific application version/commit that has been tested, reviewed, and agreed upon for production deployment. That is the image that gets promoted to the stage environment.

For that, we need to create a release branch from the specific commit on develop the branch. The image that was built from that commit is the one we want to promote to stage environment. 

For example, lets say **2b2e927** is the SHA of the commit development team chose for production. The following command creates a release branch named **release/v1.2.0-sha-2b2e927** from that exact commit **2b2e927**.

```bash
$ git checkout develop
$ git checkout -b release/v1.2.0-sha-2b2e927 2b2e927
```

> The branch naming convention depends on your team. Common patterns include **release/sprint-42**, **release/2025-03-10**, or **release/v1.2.0** if you follow semantic versioning. You can pick whatever works for your project.

The stage promotion workflow is manual. We need to pass the Docker image tagged with SHA in the pipeline as input.

![Docker in Docker]({{ site.baseurl }}/assets/img/docker-course/image-build-promotion04.png)

Here is what happens in this promotion pipeline.

- It runs a Trivy vulnerability scan on the dev image to catch issues before staging.
- Once it passes the vulnerability scan, it uses Crane to copy (promote) the image from the dev registry to the stage registry.
- Finally, the image gets deployed to the stage environment for integration and QA testing.

### Promote Docker Image to Production with Cosign Signing

This is the final gate in the pipeline. After stage environment tests pass, we push the changes to the main branch and manually trigger the production promotion.

![Docker in Docker]({{ site.baseurl }}/assets/img/docker-course/image-build-promotion05.png)

Now, you might ask, **don't we need to rebuild the image from main?**

**No**. The whole point of the promotion model is that we should never rebuild between environments. The exact same image (same SHA digest) that was tested in stage gets copied to the prod registry via **Crane**.

**Why?** Because even with the same source code, an image rebuild can pull different base image layers, get updated code dependencies, etc. So the code merge from release to main is just for keeping the Git history in sync.

Here is what this pipeline does.

- It runs one last Trivy vulnerability scan on the staged image. This is called admission-time scanning. It is sort of a safety net to catch any new vulnerabilities that appeared between staging and now.
- It then uses Crane to copy the image from the stage registry to the production registry.
- Finally, the image is digitally signed using Cosign. This way, during deployment, we can verify if it was created by our image pipeline and has not been modified after it was pushed to the registry.
- The signed image is deployed to the production environment.