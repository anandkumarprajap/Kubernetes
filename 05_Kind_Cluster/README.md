# Kubernetes Local Practice — Kind + Docker + kubectl

Beginner-friendly hands-on Kubernetes practice using **Docker Desktop, Kind, and kubectl on macOS**.

---

## 📚 Table of Contents

1. [Goal](#1-goal)
2. [Architecture](#2-architecture)
3. [Prerequisites](#3-prerequisites)
4. [Check Docker](#4-check-docker)
5. [Check Mac Architecture](#5-check-mac-architecture)
6. [Install Kind](#6-install-kind)
7. [Verify Kind](#7-verify-kind)
8. [Create Kind Configuration](#8-create-kind-configuration)
9. [Understand Kind Configuration](#9-understand-kind-configuration)
10. [Create Kubernetes Cluster](#10-create-kubernetes-cluster)
11. [Check Docker Containers](#11-check-docker-containers)
12. [Install kubectl](#12-install-kubectl)
13. [Verify kubectl](#13-verify-kubectl)
14. [Check Kubernetes Context](#14-check-kubernetes-context)
15. [Check Cluster](#15-check-cluster)
16. [Check Nodes](#16-check-nodes)
17. [Understand Kubernetes Nodes](#17-understand-kubernetes-nodes)
18. [Enter a Kind Node](#18-enter-a-kind-node)
19. [Create First Pod](#19-create-first-pod)
20. [Check Pods](#20-check-pods)
21. [Check Pod Details](#21-check-pod-details)
22. [Create Devboard Pod](#22-create-devboard-pod)
23. [Port Forwarding](#23-port-forwarding)
24. [Port Mapping vs Port Forwarding](#24-port-mapping-vs-port-forwarding)
25. [Pod Networking](#25-pod-networking)
26. [Useful Commands](#26-useful-commands)
27. [Troubleshooting](#27-troubleshooting)
28. [Important Kubernetes Concept](#28-important-kubernetes-concept)
29. [Final Architecture](#29-final-architecture)

---

# 1. Goal

The goal of this practice is to create a **local Kubernetes cluster on macOS** using:

* Docker Desktop
* Kind
* Kubernetes
* kubectl

The lab creates:

```text
Docker Desktop
      |
      v
     Kind
      |
      v
Kubernetes Cluster
      |
      +-------------------+
      |                   |
      v                   v
Control Plane         Worker Nodes
                          |
                          v
                         Pods
                          |
                          v
                      Containers
```

---

# 2. Architecture

Kind means:

> **Kubernetes IN Docker**

Kind runs Kubernetes nodes as Docker containers.

```text
                         macOS
                           |
                    Docker Desktop
                           |
                          Kind
                           |
              +------------+------------+
              |            |            |
              v            v            v
        Control Plane   Worker 1     Worker 2
        Docker Node     Docker Node  Docker Node
              |            |            |
              |            v            v
              |        Devboard        Nginx
              |          Pod            Pod
              |       10.244.1.2     10.244.2.2
              |
              v
        Kubernetes API Server
              ^
              |
           kubectl
```

---

# 3. Prerequisites

Before starting, make sure you have:

* macOS
* Docker Desktop
* Internet connection
* Terminal
* Apple Silicon Mac or Intel Mac

For Apple Silicon:

```bash
uname -m
```

Expected:

```text
arm64
```

---

# 4. Check Docker

## Command

```bash
docker --version
```

Example:

```text
Docker version 29.5.3, build d1c06ef
```

## What does it do?

It verifies that Docker is installed and available.

Docker will be used by Kind to create Kubernetes nodes.

```text
Docker
   |
   v
Kind Kubernetes Nodes
```

---

# 5. Check Mac Architecture

## Command

```bash
uname -m
```

For Apple Silicon Macs:

```text
arm64
```

Examples:

* MacBook M1
* MacBook M2
* MacBook M3
* MacBook M4

These use ARM64 architecture.

---

# 6. Install Kind

## Step 1: Download Kind

For Apple Silicon:

```bash
curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.32.0/kind-darwin-arm64
```

### Explanation

```text
curl
```

Downloads a file.

```text
-L
```

Follows redirects.

```text
-o ./kind
```

Saves the downloaded file as:

```text
./kind
```

---

## Step 2: Give Execute Permission

```bash
chmod +x ./kind
```

Check:

```bash
ls -l kind
```

You should see execute permission:

```text
-rwxr-xr-x
```

The `x` means executable.

---

## Step 3: Move Kind to PATH

```bash
sudo mv ./kind /usr/local/bin/kind
```

### Why `sudo`?

`/usr/local/bin` may require administrator permission.

Without `sudo`, you may get:

```text
Permission denied
```

---

# 7. Verify Kind

Run:

```bash
kind version
```

Example:

```text
kind v0.32.0 go1.26.3 darwin/arm64
```

This confirms that Kind is installed.

---

# 8. Create Kind Configuration

Create a file:

```bash
vim kind-config.yml
```

Use the following configuration:

```yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4

name: udaan-cluster

nodes:
  - role: control-plane
    image: kindest/node:v1.35.0
    extraPortMappings:
      - containerPort: 30080
        hostPort: 8086
        protocol: TCP

  - role: worker
    image: kindest/node:v1.35.0

  - role: worker
    image: kindest/node:v1.35.0
```

Check the file:

```bash
cat kind-config.yml
```

---

# 9. Understand Kind Configuration

## `kind`

```yaml
kind: Cluster
```

This tells Kind that we are creating a Kubernetes cluster.

---

## `apiVersion`

```yaml
apiVersion: kind.x-k8s.io/v1alpha4
```

Specifies the Kind configuration API version.

---

## Cluster Name

```yaml
name: udaan-cluster
```

Our cluster name is:

```text
udaan-cluster
```

---

## Nodes

```yaml
nodes:
  - role: control-plane
  - role: worker
  - role: worker
```

We are creating:

```text
1 Control Plane
+
2 Worker Nodes
=
3 Nodes
```

Architecture:

```text
                 udaan-cluster
                       |
          +------------+------------+
          |            |            |
          v            v            v
     Control Plane  Worker 1     Worker 2
```

---

## Kubernetes Node Image

```yaml
image: kindest/node:v1.35.0
```

This specifies the Kubernetes node image.

Therefore, the Kubernetes server version is:

```text
v1.35.0
```

---

# 10. Create Kubernetes Cluster

Run:

```bash
kind create cluster --config kind-config.yml
```

Kind performs approximately:

```text
Read configuration
       |
       v
Download node image
       |
       v
Create Docker containers
       |
       v
Create Control Plane
       |
       v
Create Worker Nodes
       |
       v
Install CNI
       |
       v
Install StorageClass
       |
       v
Join Worker Nodes
       |
       v
Configure kubectl context
```

Expected output:

```text
Creating cluster "udaan-cluster" ...

Ensuring node image (kindest/node:v1.35.0)

Preparing nodes
Writing configuration
Starting control-plane
Installing CNI
Installing StorageClass
Joining worker nodes

Set kubectl context to "kind-udaan-cluster"

You can now use your cluster with:

kubectl cluster-info --context kind-udaan-cluster
```

---

# 11. Check Docker Containers

Run:

```bash
docker ps
```

You should see containers similar to:

```text
udaan-cluster-control-plane
udaan-cluster-worker
udaan-cluster-worker2
```

All these containers are Kubernetes nodes created by Kind.

## Important

Kind does **not** create normal virtual machines for these nodes.

Instead:

```text
Docker Container
       |
       v
Kubernetes Node
```

Therefore:

```text
Kind Node = Docker Container
```

---

# 12. Install kubectl

`kubectl` is the Kubernetes command-line tool.

It communicates with the Kubernetes API Server.

For Apple Silicon:

```bash
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/darwin/arm64/kubectl"
```

---

## Download SHA256 Checksum

```bash
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/darwin/arm64/kubectl.sha256"
```

Verify the downloaded binary:

```bash
echo "$(cat kubectl.sha256) kubectl" | shasum -a 256 --check
```

Expected:

```text
kubectl: OK
```

This verifies the integrity of the downloaded binary.

---

## Make kubectl Executable

```bash
chmod +x ./kubectl
```

Move it to PATH:

```bash
sudo mv ./kubectl /usr/local/bin/kubectl
```

---

# 13. Verify kubectl

Run:

```bash
kubectl version
```

Example:

```text
Client Version: v1.36.2
Kustomize Version: v5.8.1
Server Version: v1.35.0
```

There are two important versions:

```text
kubectl Client
      |
      v
v1.36.2

Kubernetes Server
      |
      v
v1.35.0
```

### Client

`kubectl` installed on your Mac.

### Server

Kubernetes running inside the Kind cluster.

---

# 14. Check Kubernetes Context

Kind automatically configures a kubectl context.

Check current context:

```bash
kubectl config current-context
```

Expected:

```text
kind-udaan-cluster
```

List all contexts:

```bash
kubectl config get-contexts
```

---

# 15. Check Cluster

Run:

```bash
kubectl cluster-info
```

Or:

```bash
kubectl cluster-info --context kind-udaan-cluster
```

This verifies:

```text
kubectl
   |
   v
Kubernetes API Server
   |
   v
udaan-cluster
```

---

# 16. Check Nodes

Run:

```bash
kubectl get nodes
```

Example:

```text
NAME                           STATUS   ROLES
udaan-cluster-control-plane    Ready    control-plane
udaan-cluster-worker           Ready    <none>
udaan-cluster-worker2          Ready    <none>
```

This confirms:

```text
3 Nodes
 |
 +-- 1 Control Plane
 |
 +-- 2 Workers
```

All nodes should normally show:

```text
Ready
```

---

# 17. Understand Kubernetes Nodes

A Kubernetes cluster contains nodes.

```text
Cluster
   |
   +---- Control Plane
   |
   +---- Worker Node
   |
   +---- Worker Node
```

### Control Plane

The control plane manages the cluster.

Important components include:

```text
API Server
Scheduler
Controller Manager
etcd
```

### Worker Node

Worker nodes run application Pods.

```text
Worker Node
     |
     +---- Pod
     |
     +---- Pod
```

---

# 18. Enter a Kind Node

First find the container:

```bash
docker ps
```

Then:

```bash
docker exec -it <container-id> bash
```

Example:

```bash
docker exec -it 39b1fd83b7d9 bash
```

You will see:

```text
root@udaan-cluster-control-plane:/#
```

You are now inside the Kind control-plane container.

Check the filesystem:

```bash
ls
```

You will see directories such as:

```text
bin
boot
dev
etc
home
lib
media
mnt
opt
proc
root
run
sbin
tmp
usr
var
```

---

## Important: Docker Inside Kind Node

If you run:

```bash
docker ps
```

inside the Kind node, you may get:

```text
bash: docker: command not found
```

This is normal.

The Kubernetes node uses:

```text
containerd
```

as its container runtime.

Check:

```bash
kubectl get nodes -o wide
```

You can see:

```text
CONTAINER-RUNTIME
containerd://2.2.0
```

Therefore:

```text
Mac
 |
Docker Desktop
 |
Kind Node
 |
containerd
 |
Pods
 |
Containers
```

---

# 19. Create First Pod

Create a file:

```bash
vim pod.yml
```

Add:

```yaml
apiVersion: v1
kind: Pod

metadata:
  name: nginx

spec:
  containers:
    - name: nginx
      image: nginx:1.14.2
      ports:
        - containerPort: 80
```

---

## Understand the YAML

### API Version

```yaml
apiVersion: v1
```

The Pod uses Kubernetes core API version `v1`.

### Kind

```yaml
kind: Pod
```

We are creating a Pod.

### Name

```yaml
metadata:
  name: nginx
```

Pod name:

```text
nginx
```

### Container

```yaml
containers:
  - name: nginx
```

Container name:

```text
nginx
```

### Image

```yaml
image: nginx:1.14.2
```

The container uses the Nginx image.

### Container Port

```yaml
containerPort: 80
```

Nginx listens on port `80`.

---

# 20. Check Pods Before Creation

Run:

```bash
kubectl get pods
```

You may see:

```text
No resources found in default namespace.
```

This means there are currently no Pods in the `default` namespace.

---

## Create the Pod

Run:

```bash
kubectl apply -f pod.yml
```

Output:

```text
pod/nginx created
```

---

## Check the Pod

```bash
kubectl get pods
```

Example:

```text
NAME    READY   STATUS    RESTARTS   AGE
nginx   1/1     Running   0          35s
```

### What does `1/1` mean?

```text
1 container expected
1 container running
```

---

# 21. Check Pod Details

Run:

```bash
kubectl get pods -o wide
```

Example:

```text
NAME    READY   STATUS    IP           NODE
nginx   1/1     Running   10.244.2.2   udaan-cluster-worker2
```

Important information:

```text
Pod Name
    |
    v
nginx

Pod IP
    |
    v
10.244.2.2

Node
    |
    v
udaan-cluster-worker2
```

This means Kubernetes scheduled the Nginx Pod on:

```text
udaan-cluster-worker2
```

---

# 22. Create Devboard Pod

Create:

```bash
vim pod2.yml
```

Use:

```yaml
apiVersion: v1
kind: Pod

metadata:
  name: devboard

spec:
  containers:
    - name: devboard-frontend
      image: trainwithshubham/devboard-frontend
      ports:
        - containerPort: 4173
```

This creates:

```text
devboard Pod
      |
      v
devboard-frontend container
      |
      v
trainwithshubham/devboard-frontend
      |
      v
Port 4173
```

---

## Apply Devboard Pod

```bash
kubectl apply -f pod2.yml
```

Output:

```text
pod/devboard created
```

Initially you may see:

```text
devboard   0/1   ContainerCreating
```

This means Kubernetes is creating the container.

After a few seconds:

```text
devboard   1/1   Running
```

---

## Check Both Pods

```bash
kubectl get pods
```

Example:

```text
NAME       READY   STATUS
devboard   1/1     Running
nginx      1/1     Running
```

---

## Check Pod Locations

```bash
kubectl get pods -o wide
```

Example:

```text
NAME       IP           NODE
devboard   10.244.1.2   udaan-cluster-worker
nginx      10.244.2.2   udaan-cluster-worker2
```

Kubernetes has distributed the Pods across different worker nodes.

```text
                    Cluster
                       |
             +---------+---------+
             |                   |
             v                   v
         Worker 1            Worker 2
             |                   |
             v                   v
         devboard              nginx
       10.244.1.2            10.244.2.2
```

---

# 23. Port Forwarding

Port forwarding allows you to access a Pod from your local machine.

## Nginx

Nginx listens on:

```text
80
```

Run:

```bash
kubectl port-forward pod/nginx 8086:80
```

Meaning:

```text
localhost:8086
       |
       | port-forward
       v
nginx Pod:80
```

Open:

```text
http://localhost:8086
```

You should see the Nginx page.

---

## Devboard

Devboard listens on:

```text
4173
```

Run:

```bash
kubectl port-forward pod/devboard 6767:4173
```

Meaning:

```text
localhost:6767
       |
       | port-forward
       v
devboard Pod:4173
```

Open:

```text
http://localhost:6767
```

---

# 24. Port Mapping vs Port Forwarding

These are different concepts.

## Kind `extraPortMappings`

Configuration:

```yaml
extraPortMappings:
  - containerPort: 30080
    hostPort: 8086
```

This maps:

```text
Mac
 |
 | 8086
 v
Kind Node
 |
 | 30080
 v
Kubernetes
```

---

## kubectl port-forward

Example:

```bash
kubectl port-forward pod/nginx 8086:80
```

This maps:

```text
Mac
 |
 | 8086
 v
Nginx Pod
 |
 | 80
 v
Nginx
```

### Remember

```text
extraPortMappings
    |
    v
Host -> Kind Node

port-forward
    |
    v
Host -> Pod
```

---

# 25. Pod Networking

Check Pod IP:

```bash
kubectl get pods -o wide
```

Example:

```text
nginx
10.244.2.2
```

Pod IPs are part of the Kubernetes cluster network.

For example:

```text
Worker 2
   |
   +---- nginx Pod
            |
            +---- 10.244.2.2
```

The Pod IP is normally not how you access the application directly from your Mac.

For local testing:

```bash
kubectl port-forward
```

is convenient.

In production Kubernetes, applications are normally exposed using:

```text
Pod
 |
 v
Service
 |
 v
Ingress / LoadBalancer / NodePort
 |
 v
User
```

---

# 26. Useful Commands

## Docker

Check Docker:

```bash
docker --version
```

List running containers:

```bash
docker ps
```

List images:

```bash
docker images
```

---

## Kind

Check version:

```bash
kind version
```

Create cluster:

```bash
kind create cluster --config kind-config.yml
```

List Kind clusters:

```bash
kind get clusters
```

Delete cluster:

```bash
kind delete cluster --name udaan-cluster
```

---

## kubectl

Check version:

```bash
kubectl version
```

Check current context:

```bash
kubectl config current-context
```

List contexts:

```bash
kubectl config get-contexts
```

Cluster information:

```bash
kubectl cluster-info
```

Get nodes:

```bash
kubectl get nodes
```

Detailed node information:

```bash
kubectl get nodes -o wide
```

Get Pods:

```bash
kubectl get pods
```

Detailed Pod information:

```bash
kubectl get pods -o wide
```

Get Services:

```bash
kubectl get svc
```

---

# 27. Troubleshooting

## Check Pod Status

```bash
kubectl get pods
```

---

## Describe Pod

```bash
kubectl describe pod nginx
```

Useful for finding:

* Scheduling problems
* Image pull errors
* Container errors
* Events
* Networking information

---

## Check Pod Logs

```bash
kubectl logs nginx
```

---

## Check Cluster Events

```bash
kubectl get events
```

---

## Execute Command Inside Pod

```bash
kubectl exec -it nginx -- bash
```

---

## Delete Pod

```bash
kubectl delete pod nginx
```

---

## Delete Using YAML

```bash
kubectl delete -f pod.yml
```

---

# 28. Common Mistakes

## Mistake 1: Typing `kibectl`

Incorrect:

```bash
kibectl version
```

Correct:

```bash
kubectl version
```

---

## Mistake 2: Wrong Port Forward

Incorrect:

```bash
kubectl port-forward pod/nginx 8086:8086
```

Nginx is listening on port `80`, not `8086`.

Correct:

```bash
kubectl port-forward pod/nginx 8086:80
```

---

## Mistake 3: Wrong Pod Name

If Kubernetes says:

```text
pods "devoboard" not found
```

check the actual Pod name:

```bash
kubectl get pods
```

Then use the exact name.

For example:

```bash
kubectl port-forward pod/devboard 6767:4173
```

---

## Mistake 4: Running Docker Inside Kind Node

Inside the Kind node:

```bash
docker ps
```

may return:

```text
docker: command not found
```

This is normal.

The Kubernetes node uses:

```text
containerd
```

as its container runtime.

---

# 29. Important Kubernetes Concept

One of the most important concepts to remember:

> **A Pod runs one or more containers. A Node runs Pods. A Cluster contains Nodes.**

Do **not** think:

```text
Cluster
   |
   v
Container
   |
   v
Pod
```

The correct mental model is:

```text
Kubernetes Cluster
       |
       +---- Control Plane
       |
       +---- Worker Node
                |
                +---- Pod
                       |
                       +---- Container
```

For this lab:

```text
Kubernetes Cluster
       |
       +---- Control Plane Node
       |
       +---- Worker Node
       |       |
       |       +---- Devboard Pod
       |                |
       |                +---- Devboard Container
       |
       +---- Worker Node
               |
               +---- Nginx Pod
                        |
                        +---- Nginx Container
```

### Remember

```text
Cluster
   ↓
Nodes
   ↓
Pods
   ↓
Containers
```

---

# 30. Final Architecture

Your complete local Kubernetes lab looks like this:

```text
                         macOS
                           |
                           v
                    Docker Desktop
                           |
                           v
                          Kind
                           |
             +-------------+-------------+
             |             |             |
             v             v             v
       Control Plane    Worker 1      Worker 2
       Docker Node      Docker Node   Docker Node
             |             |             |
             |             |             |
             |             v             v
             |         Devboard        Nginx
             |           Pod            Pod
             |        10.244.1.2     10.244.2.2
             |             |             |
             |             v             v
             |        Container       Container
             |             |             |
             |          :4173           :80
             |
             v
       Kubernetes API Server
             ^
             |
          kubectl
             ^
             |
          Terminal
```

---

# 🎯 Final Learning Flow

The complete learning flow from this practice is:

```text
Docker
  ↓
Kind
  ↓
Create Kubernetes Cluster
  ↓
Control Plane + Worker Nodes
  ↓
kubectl
  ↓
Kubernetes API Server
  ↓
Create Pod
  ↓
Container
  ↓
Pod IP
  ↓
Port Forward
  ↓
Access Application
```

## Quick Revision

```text
Docker
→ Container platform

Kind
→ Creates local Kubernetes cluster using Docker

Kubernetes
→ Container orchestration platform

kubectl
→ CLI to communicate with Kubernetes

Cluster
→ Collection of Kubernetes nodes

Node
→ Machine/environment that runs Pods

Pod
→ Smallest deployable Kubernetes unit

Container
→ Application process running inside a Pod

Service
→ Stable network endpoint for Pods

Port Forward
→ Local machine → Pod connection
```

## 🧠 One-Line Memory Trick

```text
Cluster → Node → Pod → Container
```

And for communication:

```text
kubectl → API Server → Kubernetes Cluster
```

And for local access:

```text
localhost → port-forward → Pod → Container → Application
```
