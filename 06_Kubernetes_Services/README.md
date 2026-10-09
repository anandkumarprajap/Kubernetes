# Kubernetes Services — Easy Notes

## 1. What is a Kubernetes Service?

A **Kubernetes Service** provides a stable **single access point** for a group of Pods.

Pods are temporary. Their IP addresses can change when Pods are recreated.

Instead of users connecting directly to changing Pod IPs:

```text
User
  |
  v
Kubernetes Service
  |
  +---- Pod
  +---- Pod
  +---- Pod
```

The Service finds the correct Pods using **labels/selectors** and distributes traffic between them.

### Simple definition

> **Deployment manages Pods, while Service provides networking/access to those Pods.**

---

# 2. Why do we need a Service?

Suppose a Deployment has three frontend Pods:

```text
frontend Deployment
       |
       +---- Pod 1 → 10.0.1.10
       +---- Pod 2 → 10.0.2.20
       +---- Pod 3 → 10.0.3.30
```

Pod IPs can change.

A Service gives the application a stable address:

```text
frontend-service
ClusterIP: 10.96.10.50
```

Now clients use the Service instead of individual Pod IPs.

```text
Frontend Client
      |
      v
frontend-service
      |
      +---- Pod 1
      +---- Pod 2
      +---- Pod 3
```

---

# 3. Important Kubernetes Service Types

Common Service types:

1. **ClusterIP**
2. **NodePort**
3. **LoadBalancer**
4. **ExternalName**
5. **Headless Service**

`External IP` is also commonly discussed with Services, but it is not a separate `type:` value like `ClusterIP` or `NodePort`.

---

# 4. ClusterIP Service

## What is ClusterIP?

**ClusterIP is the default Kubernetes Service type.**

It provides an internal virtual IP that is reachable **inside the Kubernetes cluster**.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: frontend-service
  namespace: dev
spec:
  type: ClusterIP
  selector:
    app: frontend
  ports:
    - port: 80
      targetPort: 4173
      protocol: TCP
```

### Traffic flow

```text
Backend Pod
    |
    | frontend-service:80
    v
ClusterIP
    |
    v
Frontend Pods
    |
    +--- :4173
    +--- :4173
```

### Important

ClusterIP is normally **not directly accessible from the public Internet**.

It is useful for communication such as:

```text
Frontend → Backend Service
Backend → Database Service
```

### Service name

Inside Kubernetes, applications can normally use the Service DNS name:

```text
frontend-service
```

or, when using another namespace:

```text
frontend-service.dev
```

Fully qualified form:

```text
frontend-service.dev.svc.cluster.local
```

---

# 5. NodePort Service

## What is NodePort?

**NodePort exposes a Service through a port on every Kubernetes Node.**

Typical NodePort range:

```text
30000 - 32767
```

Example:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: frontend-service
  namespace: dev
spec:
  type: NodePort
  selector:
    app: frontend
  ports:
    - protocol: TCP
      port: 80
      targetPort: 4173
      nodePort: 30080
```

## Port mapping

```text
User
 |
 | :30080
 v
Node IP
 |
 | Service :80
 v
Pod :4173
```

So:

```text
NodePort 30080
      ↓
Service port 80
      ↓
Pod targetPort 4173
```

### Example

If an EC2 instance running a Kubernetes node has:

```text
EC2 / Node IP = 3.10.20.30
NodePort       = 30080
```

A client can access:

```text
http://3.10.20.30:30080
```

The request is then routed to the Service and finally to a matching frontend Pod on port `4173`.

> **Important:** `30080` is the NodePort, `80` is the Service port, and `4173` is the application's container/target port.

---

# 6. Understanding port, targetPort and nodePort

This is one of the most important Kubernetes Service concepts.

Example:

```yaml
ports:
  - protocol: TCP
    port: 80
    targetPort: 4173
    nodePort: 30080
```

## 1. port

```text
port: 80
```

This is the **Service port**.

Clients inside the cluster can access:

```text
frontend-service:80
```

## 2. targetPort

```text
targetPort: 4173
```

This is the port where the application is listening inside the Pod.

Example:

```text
Pod
 |
 +--- Container
       |
       +--- Application :4173
```

## 3. nodePort

```text
nodePort: 30080
```

This exposes the Service through the Node.

```text
NodeIP:30080
```

### Complete mapping

```text
User
 |
 | NodeIP:30080
 v
+-------------------+
| Kubernetes Node   |
|                   |
| NodePort :30080   |
+---------+---------+
          |
          | Service :80
          v
+-------------------+
| Service           |
| ClusterIP         |
| Port :80          |
+---------+---------+
          |
          | targetPort :4173
          v
+-------------------+
| Frontend Pod      |
| App :4173         |
+-------------------+
```

---

# 7. LoadBalancer Service

## What is LoadBalancer?

A **LoadBalancer Service** exposes an application outside the Kubernetes cluster through an external/cloud load balancer.

On cloud platforms such as AWS, Kubernetes can integrate with AWS load-balancing infrastructure.

Conceptually:

```text
Internet User
      |
      | HTTP/HTTPS :80/:443
      v
AWS Load Balancer
      |
      v
Kubernetes Service
      |
      v
Frontend Pods
```

In Amazon EKS, the exact AWS load balancer created depends on the Kubernetes Service configuration and AWS load-balancing integration/controller.

---

# 8. LoadBalancer request flow in EKS

A simplified EKS architecture:

```text
                         Internet
                            |
                            | HTTP/HTTPS
                            v
                 +----------------------+
                 | AWS Load Balancer    |
                 | DNS Name             |
                 +----------+-----------+
                            |
                            v
                 +----------------------+
                 | EKS Service          |
                 | LoadBalancer         |
                 +----------+-----------+
                            |
                 +----------+----------+
                 |                     |
                 v                     v
            +---------+            +---------+
            | Node 1  |            | Node 2  |
            +----+----+            +----+----+
                 |                      |
                 v                      v
             Frontend Pod          Frontend Pod
                :4173                  :4173

                            |
                            v
                       Backend Service
                            |
                            v
                       Backend Pods
                            |
                            v
                       Database Service
                            |
                            v
                       Database Pods
```

---

# 9. DNS and Load Balancer

Users normally do not remember an IP address.

Instead, a DNS name points to the load balancer.

Example concept:

```text
www.example.com
       |
       | DNS
       v
AWS Load Balancer
       |
       v
EKS Service
       |
       v
Frontend Pods
```

The exact DNS record depends on how the domain and AWS load balancer are configured.

### Important idea

```text
DNS
 ↓
Load Balancer
 ↓
Kubernetes Service
 ↓
Pods
```

---

# 10. EKS + Frontend + Backend + Database

A typical three-tier application can look like this:

```text
                         USERS
                           |
                           | HTTPS
                           v
                    +-------------+
                    | DNS         |
                    +------+------+
                           |
                           v
                    +-------------+
                    | AWS ELB     |
                    +------+------+
                           |
                           v
                +---------------------+
                | Frontend Service    |
                | LoadBalancer        |
                +----------+----------+
                           |
                 +---------+---------+
                 |         |         |
                 v         v         v
              Frontend  Frontend  Frontend
                Pod       Pod       Pod
               :4173     :4173     :4173
                           |
                           | API request
                           v
                +---------------------+
                | Backend Service     |
                | ClusterIP           |
                +----------+----------+
                           |
                 +---------+---------+
                 |         |         |
                 v         v         v
              Backend   Backend   Backend
                Pod       Pod       Pod
               :8080     :8080     :8080
                           |
                           | DB request
                           v
                +---------------------+
                | Database Service    |
                | ClusterIP           |
                +----------+----------+
                           |
                           v
                     Database Pods
```

### Why use ClusterIP for Backend and Database?

Backend and database usually do not need to be directly exposed to the Internet.

Therefore:

```text
Internet
   |
   v
LoadBalancer
   |
   v
Frontend
   |
   v
Backend Service (ClusterIP)
   |
   v
Database Service (ClusterIP)
```

This reduces unnecessary external exposure.

---

# 11. Headless Service

A **Headless Service** is created with:

```yaml
clusterIP: None
```

Example:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: database
spec:
  clusterIP: None
  selector:
    app: database
  ports:
    - port: 5432
      targetPort: 5432
```

A Headless Service does not provide a normal virtual ClusterIP.

Instead, DNS can return the IP addresses of the matching Pods.

Conceptually:

```text
database.default.svc.cluster.local
              |
      +-------+-------+
      |       |       |
      v       v       v
    Pod 1   Pod 2   Pod 3
```

Headless Services are useful for applications that need to discover individual Pods, such as some StatefulSet/database systems.

---

# 12. External IP

A Service can also be associated with an external IP in appropriate Kubernetes networking configurations.

Example:

```yaml
spec:
  externalIPs:
    - 203.0.113.10
```

Conceptually:

```text
External User
      |
      v
External IP
      |
      v
Kubernetes Service
      |
      v
Pods
```

> `externalIPs` is different from `type: LoadBalancer`.

A LoadBalancer Service asks the cloud/platform integration to provide external load-balancing infrastructure, while `externalIPs` tells Kubernetes about an externally reachable IP that routes to the Service.

---

# 13. Service and Deployment relationship

A **Deployment** creates and manages Pods.

A **Service** provides stable networking to those Pods.

```text
Deployment
    |
    +---- ReplicaSet
            |
            +---- Pod
            +---- Pod
            +---- Pod

Service
    |
    +---- selects Pods using labels
```

Example Deployment labels:

```yaml
metadata:
  labels:
    app: frontend
```

Service selector:

```yaml
selector:
  app: frontend
```

Because both use:

```text
app: frontend
```

the Service can send traffic to those Pods.

---

# 14. One Service = Single Access Point

A useful way to remember Services:

```text
                  Single Access Point
                         |
                         v
                  +-------------+
                  |   Service   |
                  +------+------+
                         |
            +------------+------------+
            |            |            |
            v            v            v
          Pod 1        Pod 2        Pod 3
```

The client does not need to know every Pod IP.

It only needs the Service address/name.

---

# 15. Kubernetes Service Types — Quick Comparison

| Service Type | Main Purpose | Internet Access | Common Use |
|---|---|---|---|
| ClusterIP | Internal access | No, normally | Backend, database |
| NodePort | Expose through Node IP + port | Yes, if network/firewall allows | Labs, simple external access |
| LoadBalancer | Cloud external load balancer | Yes | Production frontend/API |
| Headless | Pod-level DNS discovery | Usually internal | Stateful apps |
| ExternalName | DNS alias to an external service | Depends on external service | External dependencies |
| externalIPs | Route through an existing external IP | Depends on network | Special networking setups |

---

# 16. Commands

## List Services

```bash
kubectl get svc -n dev
```

Example:

```text
NAME               TYPE           CLUSTER-IP      EXTERNAL-IP   PORT(S)
frontend-service   NodePort       10.96.10.50     <none>        80:30080/TCP
backend-service    ClusterIP      10.96.20.50     <none>        8080/TCP
```

For a NodePort:

```text
80:30080/TCP
```

means:

```text
Service Port : NodePort
80            : 30080
```

---

## Get all resources

```bash
kubectl get all -n dev
```

---

## Get Pods with detailed node information

```bash
kubectl get pods -n dev -o wide
```

This can show:

- Pod IP
- Node
- Pod status
- Restart count
- Age

---

## Describe a Service

```bash
kubectl describe svc frontend-service -n dev
```

Useful for checking:

- Selector
- ClusterIP
- Ports
- TargetPort
- NodePort
- Endpoints

---

## Get Endpoints

```bash
kubectl get endpoints -n dev
```

Or, on newer Kubernetes versions:

```bash
kubectl get endpointslices -n dev
```

---

# 17. Example NodePort Service

```yaml
apiVersion: v1
kind: Service

metadata:
  name: frontend-service
  namespace: dev

spec:
  type: NodePort

  selector:
    app: frontend

  ports:
    - protocol: TCP
      port: 80
      targetPort: 4173
      nodePort: 30080
```

Apply:

```bash
kubectl apply -f frontend-service.yaml
```

Check:

```bash
kubectl get svc -n dev
```

---

# 18. Complete NodePort Traffic Flow

For this configuration:

```text
port:       80
targetPort: 4173
nodePort:   30080
```

Traffic flows like:

```text
User
 |
 | NodeIP:30080
 v
+----------------------+
| Kubernetes Node/EC2  |
| NodePort :30080      |
+----------+-----------+
           |
           | Service :80
           v
+----------------------+
| frontend-service     |
| ClusterIP            |
| Service Port :80     |
+----------+-----------+
           |
           | targetPort :4173
           v
+----------------------+
| Frontend Pod         |
| Application :4173    |
+----------------------+
```

---

# 19. Important correction: EC2 and EKS

In EKS, worker nodes may be EC2 instances when using EC2-based node groups.

Conceptually:

```text
AWS
 |
 +--- EKS Cluster
       |
       +--- EC2 Node 1
       |      |
       |      +--- Pods
       |
       +--- EC2 Node 2
       |      |
       |      +--- Pods
       |
       +--- EC2 Node 3
              |
              +--- Pods
```

EKS manages the Kubernetes control-plane side, while your workloads can run on managed/self-managed EC2 worker nodes or other supported compute options.

---

# 20. Easy Memory Trick

Remember:

```text
Deployment → creates/manages Pods

Pod → runs the application

Service → gives Pods a stable access point

ClusterIP → internal access

NodePort → NodeIP:30000-32767

LoadBalancer → external/cloud load balancer

Headless → DNS discovers individual Pod IPs
```

And remember the three important ports:

```text
NodePort
   ↓
Service Port
   ↓
TargetPort
```

Example:

```text
30080
  ↓
 80
  ↓
4173
```

So:

```text
NodeIP:30080
      ↓
Service:80
      ↓
Pod:4173
```

---

# 21. Final Architecture

```text
                    INTERNET USERS
                           |
                           v
                    +-------------+
                    |    DNS      |
                    +------+------+
                           |
                           v
                    +-------------+
                    | AWS ELB/LB  |
                    +------+------+
                           |
                           v
                 +--------------------+
                 | Frontend Service   |
                 | LoadBalancer       |
                 +---------+----------+
                           |
              +------------+------------+
              |            |            |
              v            v            v
          Frontend      Frontend     Frontend
             Pod           Pod          Pod
            :4173         :4173        :4173
              |
              | API request
              v
       +--------------------+
       | Backend Service    |
       | ClusterIP           |
       +---------+----------+
                 |
          +------+------+
          |      |      |
          v      v      v
       Backend Backend Backend
         Pod      Pod      Pod
        :8080    :8080    :8080
                 |
                 | DB request
                 v
       +--------------------+
       | Database Service   |
       | ClusterIP          |
       +---------+----------+
                 |
                 v
           Database Pods


Alternative simple lab access:

User
 |
 | NodeIP:30080
 v
EC2/Kubernetes Node
 |
 | Service :80
 v
Frontend Pod :4173
```

---

# 22. One-Line Interview Answer

> **A Kubernetes Service provides a stable network endpoint for a group of Pods. ClusterIP is used for internal communication, NodePort exposes the Service through a port on each Node, LoadBalancer exposes it through a cloud load balancer, and Headless Services provide DNS-based discovery of individual Pod IPs.**
