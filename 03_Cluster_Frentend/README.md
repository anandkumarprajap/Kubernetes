# Frontend-Devboard
![Image 4](4.png)
![Image 5](5.png)
![Image 6](6.png)
![Image 7](7.png)
![Image 8](8.png)
![Image 9](9.png)
![Image 10](10.png)
![Image 11](11.png)
![Image 12](12.png)
![Image 13](13.png)
![Image 14](14.png)
![Image 15](15.png)
![Image 16](16.png)
![Image 17](17.png)
![Image 18](18.png)
![Image 19](19.png)
![Image 20](20.png)
![Image 21](21.png)
![Image 22](22.png)
![Image 23](23.png)
![Image 24](24.png)
![Image 1.24](1.24.png)
![Image 2.24](2.24.png)
![Image 3.24](3.24.png)
![Image 25](25.png)
![Image 26](26.png)
![Image 26.a](26.a.png)
![Image 27](27.png)
![Image 28](28.png)
![Image 29](29.png)


# 🚀 DevBoard Frontend Deployment on EC2 using Kind Kubernetes

This project demonstrates how to deploy the **DevBoard Frontend Docker image** on a **Kubernetes cluster created with Kind** inside an **AWS EC2 Ubuntu server**.

The deployment covers:

* AWS EC2
* Ubuntu Linux
* Docker
* Kind Kubernetes
* kubectl
* Kubernetes Namespace
* Pod
* Deployment
* ReplicaSet
* Service
* NodePort
* Docker Hub
* Scaling
* Port Forwarding
* Kubernetes troubleshooting

---

# 📌 1. Project Architecture

```text
                         👤 User / Browser
                               |
                               |
                         AWS EC2 Public IP
                               |
                         NodePort :30088
                               |
                     ┌─────────┴─────────┐
                     │   EC2 Ubuntu      │
                     │                   │
                     │      Docker       │
                     │        │          │
                     │        ▼          │
                     │   Kind Cluster    │
                     │                   │
                     │ ┌───────────────┐ │
                     │ │ Control Plane │ │
                     │ └───────────────┘ │
                     │        │          │
                     │   ┌────┴────┐     │
                     │   ▼         ▼     │
                     │ Worker  Worker2   │
                     │   │         │     │
                     │   └────┬────┘     │
                     │        │          │
                     │     Service       │
                     │    NodePort       │
                     │       :30088      │
                     │        │          │
                     │        ▼          │
                     │ DevBoard Frontend │
                     │      Pods × 5     │
                     └───────────────────┘
```

---

# 🎯 2. Project Goal

The goal is to deploy the DevBoard frontend Docker image:

```text
anandkumarprajap/devboard-frontend:latest
```

into a Kubernetes cluster running inside an AWS EC2 instance.

We use:

```text
EC2
  ↓
Ubuntu
  ↓
Docker
  ↓
Kind
  ↓
Kubernetes Cluster
  ↓
Namespace
  ↓
Deployment
  ↓
Pods
  ↓
Service
  ↓
NodePort
  ↓
Browser
```

---

# 👥 3. Who Uses What?

| User/Component        | Responsibility                          |
| --------------------- | --------------------------------------- |
| Developer             | Creates application and Docker image    |
| Docker Hub            | Stores Docker image                     |
| DevOps Engineer       | Creates infrastructure and deployment   |
| EC2                   | Provides the server                     |
| Docker                | Runs Kind containers                    |
| Kind                  | Creates Kubernetes cluster              |
| kubectl               | Communicates with Kubernetes            |
| Kubernetes API Server | Receives Kubernetes commands            |
| Deployment            | Manages application replicas            |
| ReplicaSet            | Maintains desired number of Pods        |
| Pod                   | Runs frontend container                 |
| Service               | Provides stable access to Pods          |
| NodePort              | Exposes application outside the cluster |
| End User              | Opens frontend in browser               |

---

# 🖥️ 4. EC2 Environment

The Kubernetes cluster is running inside an AWS EC2 Ubuntu server.

Example:

```bash
ubuntu@ip-172-31-23-191:~$
```

Check OS:

```bash
cat /etc/os-release
```

Check CPU architecture:

```bash
uname -m
```

Example:

```text
x86_64
```

Check system resources:

```bash
free -h
df -h
nproc
```

---

# 🐳 5. Install Docker

Docker is required because Kind creates Kubernetes nodes as Docker containers.

Check Docker:

```bash
docker --version
```

Example:

```text
Docker version 29.1.3
```

Install Docker on Ubuntu:

```bash
sudo apt-get update -y
sudo apt-get install -y docker.io
```

Start Docker:

```bash
sudo systemctl enable --now docker
```

Add the current user to Docker group:

```bash
sudo usermod -aG docker $USER
```

Refresh the group:

```bash
newgrp docker
```

Test:

```bash
docker ps
```

### Common problem

If you see:

```text
permission denied while trying to connect to the Docker daemon
```

run:

```bash
sudo usermod -aG docker $USER
newgrp docker
```

Then:

```bash
docker ps
```

---

# ☸️ 6. Install Kind

Kind means:

> Kubernetes IN Docker

Kind is useful for creating Kubernetes clusters for:

* Learning
* Development
* Testing
* CI/CD
* Kubernetes practice

Check:

```bash
kind version
```

Example:

```text
kind v0.29.0
```

For an EC2 x86_64 machine:

```bash
curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.29.0/kind-linux-amd64
```

Make it executable:

```bash
chmod +x ./kind
```

Move it:

```bash
sudo mv ./kind /usr/local/bin/kind
```

Verify:

```bash
kind version
```

---

# ☸️ 7. Install kubectl

`kubectl` is the command-line tool used to communicate with Kubernetes.

Check:

```bash
kubectl version --client
```

Install:

```bash
ARCH=$(uname -m)

VERSION=$(curl -Ls https://dl.k8s.io/release/stable.txt)

curl -LO "https://dl.k8s.io/release/${VERSION}/bin/linux/amd64/kubectl"

chmod +x kubectl

sudo mv kubectl /usr/local/bin/
```

Verify:

```bash
kubectl version --client
```

---

# 🧩 8. Create Kind Cluster

Example Kind configuration:

```yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
name: tws-cluster

nodes:
- role: control-plane
  image: kindest/node:v1.35.0
  extraPortMappings:
    - containerPort: 30080
      hostPort: 8080
      protocol: TCP

- role: worker
  image: kindest/node:v1.35.0

- role: worker
  image: kindest/node:v1.35.0

```
0r
```yml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4

name: babu-cluster

nodes:

- role: control-plane

- role: worker

- role: worker
```

Save as:

```text
kind-config.yml
```

Create the cluster:

```bash
kind create cluster --config kind-config.yml
```

Check clusters:

```bash
kind get clusters
```

Expected:

```text
babu-cluster
```

---

# 🔍 9. Check Kubernetes Nodes

Run:

```bash
kubectl get nodes
```

Example:

```text
NAME                         STATUS   ROLES
babu-cluster-control-plane   Ready    control-plane
babu-cluster-worker          Ready    <none>
babu-cluster-worker2         Ready    <none>
```

This means:

```text
Control Plane
      |
      +---- Worker
      |
      +---- Worker
```

---

# 🔎 10. Check Kubernetes System Pods

```bash
kubectl get pods -n kube-system
```

You should see components such as:

```text
coredns
etcd
kube-apiserver
kube-controller-manager
kube-scheduler
kube-proxy
kindnet
```

These are Kubernetes system components.

---

# 📦 11. Docker Containers Created by Kind

Run:

```bash
docker ps
```

You will see containers similar to:

```text
babu-cluster-control-plane
babu-cluster-worker
babu-cluster-worker2
```

Important:

```text
Kind Kubernetes Node
        =
Docker Container
```

Kind uses Docker containers as Kubernetes nodes.

---

# 🏷️ 12. Create Kubernetes Namespace

A namespace logically separates resources.

Create:

```yaml
apiVersion: v1
kind: Namespace

metadata:
  name: devboard
```

Save:

```text
namespace.yml
```

Apply:

```bash
kubectl apply -f namespace.yml
```

Check:

```bash
kubectl get namespaces
```

You should see:

```text
devboard
```

---

# 📦 13. Create a Simple Pod

Example:

```yaml
apiVersion: v1
kind: Pod

metadata:
  name: devboard-frontend
  namespace: devboard

spec:
  containers:
    - name: devboard-frontend
      image: anandkumarprajap/devboard-frontend:latest

      ports:
        - containerPort: 4173
```

Save as:

```text
pod.yml
```

Apply:

```bash
kubectl apply -f pod.yml
```

Check:

```bash
kubectl get pods -n devboard
```

Expected:

```text
devboard-frontend   1/1   Running
```

---

# 🧠 14. Why Do We Need a Deployment?

A Pod is not normally used directly for production-style application management.

A Deployment provides:

* Replica management
* Self-healing
* Rolling updates
* Scaling
* ReplicaSet management

Instead of manually creating five Pods:

```text
Pod
Pod
Pod
Pod
Pod
```

we tell Kubernetes:

```yaml
replicas: 5
```

Kubernetes creates and manages them automatically.

---

# 🚀 15. Create Frontend Deployment

Example:

```yaml
apiVersion: apps/v1
kind: Deployment

metadata:
  name: devboard-frontend-deployment
  namespace: devboard

  labels:
    app: devboard-frontend

spec:
  replicas: 5

  selector:
    matchLabels:
      app: devboard-frontend

  template:
    metadata:
      labels:
        app: devboard-frontend

    spec:
      containers:
        - name: devboard-frontend

          image: anandkumarprajap/devboard-frontend:latest

          ports:
            - containerPort: 4173
```

Save:

```text
deployment.yml
```

Apply:

```bash
kubectl apply -f deployment.yml
```

---

# 🔍 16. Check Deployment

```bash
kubectl get deployment -n devboard
```

Expected:

```text
NAME                         READY   UP-TO-DATE   AVAILABLE
devboard-frontend-deployment 5/5     5            5
```

This means:

```text
Desired Pods = 5
Current Pods = 5
Ready Pods   = 5
```

---

# 📦 17. Check Pods

```bash
kubectl get pods -n devboard
```

Example:

```text
devboard-frontend-deployment-xxxxx   1/1   Running
devboard-frontend-deployment-xxxxx   1/1   Running
devboard-frontend-deployment-xxxxx   1/1   Running
devboard-frontend-deployment-xxxxx   1/1   Running
devboard-frontend-deployment-xxxxx   1/1   Running
```

---

# 🔄 18. Deployment → ReplicaSet → Pod

The Kubernetes relationship is:

```text
Deployment
     |
     ▼
ReplicaSet
     |
     ├── Pod
     ├── Pod
     ├── Pod
     ├── Pod
     └── Pod
```

The Deployment does not directly manage individual Pods.

It manages a ReplicaSet.

The ReplicaSet maintains the desired number of Pods.

---

# 📈 19. Scale the Application

Scale from 5 to 10 Pods:

```bash
kubectl scale deployment devboard-frontend-deployment \
  --replicas=10 \
  -n devboard
```

Check:

```bash
kubectl get pods -n devboard
```

You should now have:

```text
10 Pods
```

Scale back:

```bash
kubectl scale deployment devboard-frontend-deployment \
  --replicas=1 \
  -n devboard
```

Check:

```bash
kubectl get deployment -n devboard
```

---

# ❤️ 20. Kubernetes Self-Healing

Suppose you have:

```text
replicas: 5
```

and one Pod crashes or is deleted.

Example:

```bash
kubectl delete pod <pod-name> -n devboard
```

Kubernetes notices:

```text
Desired = 5
Current = 4
```

So it creates a new Pod:

```text
Desired = 5
Current = 5
```

This is one of the major benefits of using a Deployment.

---

# 🌐 21. Create Kubernetes Service

Pods are temporary.

Their IP addresses can change.

Therefore, users should not connect directly to Pod IPs.

A Service provides stable networking.

Example:

```yaml
apiVersion: v1
kind: Service

metadata:
  name: devboard-frontend-service
  namespace: devboard

spec:
  type: NodePort

  selector:
    app: devboard-frontend

  ports:
    - protocol: TCP
      port: 8880
      targetPort: 4173
```

Save:

```text
service.yml
```

Apply:

```bash
kubectl apply -f service.yml
```

---

# 🔌 22. NodePort

NodePort exposes a Kubernetes Service through a port on the Kubernetes node.

The traffic flow is:

```text
Browser
   |
   | EC2-IP:NodePort
   ▼
EC2
   |
   ▼
Kind Node
   |
   ▼
NodePort Service
   |
   ▼
Service
   |
   ▼
Frontend Pod
   |
   ▼
Container :4173
```

Important distinction:

```text
Service port       = 8880
Container port     = 4173
NodePort            = Kubernetes-assigned port
```

If you explicitly want a NodePort, you can configure:

```yaml
ports:
  - protocol: TCP
    port: 8880
    targetPort: 4173
    nodePort: 30088
```

Then access:

```text
http://<EC2-PUBLIC-IP>:30088
```

The NodePort must normally be in the Kubernetes NodePort range:

```text
30000-32767
```

---

# ⚠️ 23. Important Kind + EC2 Networking Issue

This is important when running Kind inside EC2.

Your architecture is:

```text
Internet
   |
   ▼
EC2 Public IP
   |
   ▼
EC2 Network Interface
   |
   ▼
Docker
   |
   ▼
Kind Node Container
   |
   ▼
Kubernetes NodePort
```

The EC2 Security Group must allow the required port.

For example, if using:

```text
NodePort = 30088
```

add an inbound rule:

```text
Type: Custom TCP
Port: 30088
Source: Your IP
```

For temporary testing:

```text
0.0.0.0/0
```

is possible, but it is **not recommended for production**.

Prefer your own public IP/CIDR when possible.

---

# 🔁 24. Port Forwarding

You can also access the Service without exposing NodePort externally.

Example:

```bash
kubectl port-forward \
  svc/devboard-frontend-service \
  8001:8880 \
  -n devboard \
  --address 0.0.0.0
```

Then:

```text
EC2-IP:8001
```

can forward to the Kubernetes Service.

However, port-forwarding is mainly useful for:

* Debugging
* Development
* Temporary testing

It is not normally the preferred production exposure method.

---

# 🔍 25. Check All DevBoard Resources

```bash
kubectl get all -n devboard
```

This is one of the most useful commands.

Example:

```text
Pods
Services
Deployments
ReplicaSets
```

---

# 🔍 26. Check Service

```bash
kubectl get svc -n devboard
```

Example:

```text
NAME                       TYPE       CLUSTER-IP      PORT(S)
devboard-frontend-service  NodePort   10.96.x.x      8880:30088/TCP
```

Here:

```text
8880  = Service Port
30088 = NodePort
```

---

# 🔎 27. Describe Resources

Deployment:

```bash
kubectl describe deployment \
  devboard-frontend-deployment \
  -n devboard
```

Pod:

```bash
kubectl describe pod <pod-name> -n devboard
```

Service:

```bash
kubectl describe service \
  devboard-frontend-service \
  -n devboard
```

`describe` is extremely useful for troubleshooting.

---

# 📜 28. Check Application Logs

Correct command:

```bash
kubectl logs <pod-name> -n devboard
```

For example:

```bash
kubectl logs \
  devboard-frontend-deployment-xxxxx \
  -n devboard
```

Do **not** use:

```bash
kubectl log
```

The correct command is:

```bash
kubectl logs
```

Also, don't include the word `pod` as the resource name unless using the proper syntax.

Correct:

```bash
kubectl logs pod/<pod-name> -n devboard
```

or simply:

```bash
kubectl logs <pod-name> -n devboard
```

---

# 📝 29. Example Frontend Logs

Your Vite frontend can show:

```text
Local:   http://localhost:4173/
Network: http://10.244.x.x:4173/
```

This confirms that the Vite server is running inside the container.

The Pod IP:

```text
10.244.x.x
```

should not normally be given directly to end users.

Use a Kubernetes Service instead.

---

# 📊 30. Check Kubernetes Events

When something goes wrong:

```bash
kubectl get events -n devboard
```

For a specific Pod:

```bash
kubectl describe pod <pod-name> -n devboard
```

Look at the bottom under:

```text
Events:
```

Typical events include:

```text
Scheduled
Pulling
Pulled
Created
Started
Killing
Failed
```

---

# 🚨 31. Troubleshooting: ImagePullBackOff

If you see:

```text
ImagePullBackOff
```

or:

```text
ErrImagePull
```

check:

```bash
kubectl describe pod <pod-name> -n devboard
```

Look for:

```text
Failed to pull image
```

Common causes:

1. Incorrect Docker Hub username
2. Incorrect image name
3. Incorrect tag
4. Image does not exist
5. Docker Hub repository is private
6. Typographical error

Verify the image:

```bash
docker pull anandkumarprajap/devboard-frontend:latest
```

Then check:

```bash
kubectl describe pod <pod-name> -n devboard
```

---

# 🚨 32. Troubleshooting: CrashLoopBackOff

If:

```text
STATUS = CrashLoopBackOff
```

check logs:

```bash
kubectl logs <pod-name> -n devboard
```

Also:

```bash
kubectl describe pod <pod-name> -n devboard
```

Possible reasons:

* Application crashes
* Wrong startup command
* Missing environment variable
* Incorrect configuration
* Port/configuration problem

---

# 🚨 33. Troubleshooting: Pod Not Found

If you run:

```bash
kubectl logs <pod-name>
```

and receive:

```text
NotFound
```

first check:

```bash
kubectl get pods -n devboard
```

Remember that Pod names generated by Deployments contain a random suffix.

For example:

```text
devboard-frontend-deployment-6c959767b5-djb68
```

Use the current Pod name.

---

# 🚨 34. Troubleshooting: Deployment Not Found

If:

```bash
kubectl get deployment
```

returns:

```text
No resources found in default namespace.
```

but your application exists in `devboard`, that's normal.

You created:

```yaml
namespace: devboard
```

Therefore use:

```bash
kubectl get deployment -n devboard
```

or:

```bash
kubectl get all -n devboard
```

### Important

Kubernetes commands without `-n` normally use:

```text
default
```

Example:

```bash
kubectl get pods
```

means:

```text
default namespace
```

For DevBoard:

```bash
kubectl get pods -n devboard
```

means:

```text
devboard namespace
```

---

# 🚨 35. Troubleshooting: Service Has No Endpoints

If the Service cannot reach Pods, check:

```bash
kubectl get endpoints -n devboard
```

Also:

```bash
kubectl describe service devboard-frontend-service -n devboard
```

Check the Service selector:

```yaml
selector:
  app: devboard-frontend
```

The Pod labels must match:

```yaml
labels:
  app: devboard-frontend
```

Therefore:

```text
Service selector
        |
        | matches
        ▼
Pod label
```

If they don't match, the Service won't route traffic to the Pods.

---

# 🔍 36. Useful Kubernetes Commands

## Cluster

```bash
kind get clusters
```

```bash
kubectl cluster-info
```

```bash
kubectl get nodes
```

---

## Namespace

```bash
kubectl get ns
```

```bash
kubectl get all -n devboard
```

---

## Pods

```bash
kubectl get pods -n devboard
```

```bash
kubectl get pods -o wide -n devboard
```

```bash
kubectl describe pod <pod-name> -n devboard
```

```bash
kubectl logs <pod-name> -n devboard
```

---

## Deployment

```bash
kubectl get deployment -n devboard
```

```bash
kubectl describe deployment devboard-frontend-deployment -n devboard
```

```bash
kubectl scale deployment devboard-frontend-deployment \
  --replicas=10 \
  -n devboard
```

---

## Service

```bash
kubectl get svc -n devboard
```

```bash
kubectl describe svc devboard-frontend-service -n devboard
```

```bash
kubectl get endpoints -n devboard
```

---

## Events

```bash
kubectl get events -n devboard
```

---

## Everything

```bash
kubectl get all -n devboard
```

---

# 🧹 37. Delete Application

Delete Deployment:

```bash
kubectl delete deployment \
  devboard-frontend-deployment \
  -n devboard
```

Delete Service:

```bash
kubectl delete service \
  devboard-frontend-service \
  -n devboard
```

Delete Pod:

```bash
kubectl delete pod <pod-name> -n devboard
```

Delete complete namespace:

```bash
kubectl delete namespace devboard
```

Deleting the namespace deletes resources inside it.

---

# 🔄 38. Complete Deployment Flow

The complete process is:

```text
1. Create AWS EC2
        ↓
2. Install Ubuntu dependencies
        ↓
3. Install Docker
        ↓
4. Install Kind
        ↓
5. Install kubectl
        ↓
6. Create Kind cluster
        ↓
7. Verify Kubernetes nodes
        ↓
8. Create devboard namespace
        ↓
9. Pull DevBoard image from Docker Hub
        ↓
10. Create Deployment
        ↓
11. Deployment creates ReplicaSet
        ↓
12. ReplicaSet creates Pods
        ↓
13. Create Service
        ↓
14. Service selects Pods
        ↓
15. NodePort exposes Service
        ↓
16. EC2 Security Group allows traffic
        ↓
17. User opens EC2-IP:NodePort
        ↓
18. DevBoard Frontend is displayed
```

---

# 🧠 39. Important Kubernetes Concepts

## Pod

Smallest deployable Kubernetes unit.

```text
Pod
 └── Container
      └── DevBoard Frontend
```

---

## Deployment

Manages application Pods.

```text
Deployment
     ↓
ReplicaSet
     ↓
Pods
```

---

## ReplicaSet

Ensures the required number of Pods are running.

Example:

```yaml
replicas: 5
```

means Kubernetes tries to maintain five Pods.

---

## Service

Provides stable networking for Pods.

```text
Service
   ↓
Pod
Pod
Pod
Pod
Pod
```

---

## NodePort

Exposes a Service outside the Kubernetes cluster through a node port.

```text
EC2 IP
  +
NodePort
```

Example:

```text
EC2_PUBLIC_IP:30088
```

---

# 🏗️ 40. Final Architecture

```text
                         Internet
                            |
                            ▼
                    👤 End User
                            |
                            |
                  EC2 Public IP
                      :30088
                            |
                            ▼
                  ┌─────────────────┐
                  │    AWS EC2      │
                  │    Ubuntu       │
                  └────────┬────────┘
                           |
                        Docker
                           |
                           ▼
                ┌─────────────────────┐
                │    Kind Cluster     │
                │                     │
                │ Control Plane       │
                │       │             │
                │       ├── Worker    │
                │       │             │
                │       └── Worker2   │
                │                     │
                └──────────┬──────────┘
                           |
                           ▼
                    Namespace
                     devboard
                           |
                           ▼
                    Deployment
                           |
                           ▼
                     ReplicaSet
                           |
            ┌──────────────┼──────────────┐
            ▼              ▼              ▼
          Pod            Pod            Pod
            │              │              │
            └──────────────┼──────────────┘
                           |
                           ▼
                       Service
                      NodePort
                        :30088
                           |
                           ▼
                  DevBoard Frontend
                         :4173
```

---

# 📝 41. Key Interview Explanation

### Question: How did you deploy your frontend on Kubernetes?

**Answer:**

> I first created an Ubuntu EC2 instance and installed Docker, Kind, and kubectl. I used Kind to create a multi-node Kubernetes cluster with one control-plane node and two worker nodes. I created a dedicated `devboard` namespace and deployed the DevBoard frontend Docker image from Docker Hub using a Kubernetes Deployment. The Deployment created a ReplicaSet, which maintained multiple frontend Pods. I then created a NodePort Service to provide stable access to the Pods and exposed the application through the EC2 instance. I also configured the EC2 Security Group to allow the required port and used Kubernetes commands such as `kubectl get`, `describe`, `logs`, `events`, and `scale` for monitoring and troubleshooting.

---

# 🎯 42. Why Kubernetes Instead of Running Docker Directly?

Docker alone:

```text
Docker Container
      ↓
Application
```

Kubernetes:

```text
Deployment
    ↓
ReplicaSet
    ↓
Multiple Pods
    ↓
Service
    ↓
Networking
```

Kubernetes provides:

* Self-healing
* Scaling
* Service discovery
* Load distribution
* Rolling updates
* Declarative configuration
* Container orchestration

---

# 🔐 43. Security Notes

For real environments:

* Don't expose unnecessary EC2 ports.
* Avoid `0.0.0.0/0` where possible.
* Use restricted Security Group rules.
* Don't store Docker Hub passwords in YAML.
* Use Kubernetes Secrets for sensitive values.
* Use private Docker repositories where appropriate.
* Use HTTPS for production applications.
* Use an Ingress or cloud load balancer instead of relying on NodePort for production.
* Use least-privilege IAM permissions.

---

# ⚠️ 44. Important Corrections From the Practice

Some commands/configurations in the practice contained typing mistakes. The correct forms are:

### Correct

```bash
kubectl logs <pod-name> -n devboard
```

Not:

```bash
kubectl log
```

### Correct

```bash
kubectl get pods -n devboard
```

Not:

```bash
kubectl get pods devboard
```

### Correct

```bash
kubectl delete deployment devboard-frontend-deployment -n devboard
```

Not:

```bash
kubectl delete devboard-frontend-deployment
```

### Correct

```bash
kubectl scale deployment devboard-frontend-deployment \
  --replicas=10 \
  -n devboard
```

### Correct Kind API version

```yaml
apiVersion: kind.x-k8s.io/v1alpha4
```

### Correct Kubernetes API version for Namespace

```yaml
apiVersion: v1
```

### Correct Deployment API version

```yaml
apiVersion: apps/v1
```

---

# 📚 45. One-Minute Revision

Remember this sequence:

```text
EC2
 ↓
Docker
 ↓
Kind
 ↓
Kubernetes Cluster
 ↓
Namespace
 ↓
Deployment
 ↓
ReplicaSet
 ↓
Pods
 ↓
Service
 ↓
NodePort
 ↓
EC2 Public IP
 ↓
Browser
```

### Most important commands

```bash
docker ps

kind get clusters

kubectl get nodes

kubectl get ns

kubectl get pods -n devboard

kubectl get deployment -n devboard

kubectl get svc -n devboard

kubectl get all -n devboard

kubectl logs <pod-name> -n devboard

kubectl describe pod <pod-name> -n devboard

kubectl get events -n devboard

kubectl scale deployment devboard-frontend-deployment \
  --replicas=10 \
  -n devboard
```

---

# ✅ Final Result

The DevBoard frontend Docker image is successfully deployed on Kubernetes running through Kind on an AWS EC2 Ubuntu server.

The final architecture is:

```text
AWS EC2
   ↓
Docker
   ↓
Kind Kubernetes
   ↓
Namespace: devboard
   ↓
Deployment
   ↓
ReplicaSet
   ↓
5 Frontend Pods
   ↓
NodePort Service
   ↓
EC2 Network
   ↓
Browser
```

This setup provides a practical foundation for moving from a simple Docker deployment toward a complete **DevOps/Kubernetes CI/CD workflow**.





# Backend-Devboard

![Image 30](30.png)
![Image 31](31.png)
![Image 32](32.png)
![Image 33](33.png)
![Image 34](34.png)
![Image 35](35.png)
![Image 36](36.png)
![Image 37](37.png)
![Image 38](38.png)
![Image 39](39.png)
![Image 40](40.png)
