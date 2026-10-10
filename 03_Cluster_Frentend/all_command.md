# Docker commands
```cmd
# 1. Check Docker version
docker --version

# 2. Check whether Docker service is running
sudo systemctl status docker

# 3. Start Docker if it is stopped
sudo systemctl start docker

# 4. Enable Docker to start after reboot
sudo systemctl enable docker

# 5. Display running containers
docker ps

# 6. Display all containers, including stopped containers
docker ps -a

# 7. Display container images
docker images

# 8. Display Docker resource usage
docker stats --no-stream

# 9. Show logs from the Kind control-plane container
docker logs babu-cluster-control-plane --tail 50

# 10. Show host port mappings
docker port babu-cluster-control-plane

# 11. Check which process is listening on host port 8080
sudo ss -lntp | grep ':8080'
```
# Kind commands
```cmd
# 1. Check the Kind version
kind --version

# 2. List all Kind clusters
kind get clusters

# 3. Check the node containers belonging to this cluster
kind get nodes --name babu-cluster

# 4. Check cluster details
kubectl cluster-info

# 5. Check the current Kubernetes context
kubectl config current-context

# 6. List all Kubernetes contexts
kubectl config get-contexts

# 7. Select the correct Kind cluster context
kubectl config use-context kind-babu-cluster

# 8. Check Kubernetes client and server versions
kubectl version

# 9. Export the cluster configuration for inspection
kind export kubeconfig --name babu-cluster
```
# Node commands
```cmd
# 1. List all nodes
kubectl get nodes

# 2. Display nodes with IP addresses and runtime details
kubectl get nodes -o wide

# 3. Inspect the control-plane node
kubectl describe node babu-cluster-control-plane

# 4. Inspect worker node 1
kubectl describe node babu-cluster-worker

# 5. Inspect worker node 2
kubectl describe node babu-cluster-worker2

# 6. Show resource usage if Metrics Server is installed
kubectl top nodes

# 7. Check node-related events across namespaces
kubectl get events -A --sort-by=.metadata.creationTimestamp
```
# Namespace commands
```cmd
# 1. List all namespaces
kubectl get namespaces

# 2. Check only the DevBoard namespace
kubectl get namespace devboard

# 3. Inspect namespace configuration and status
kubectl describe namespace devboard

# 4. List all pods inside devboard
kubectl get pods -n devboard

# 5. List pods across all namespaces
kubectl get pods -A

# 6. Show all common resources in devboard
kubectl get all -n devboard

# 7. Watch DevBoard resources as they change
kubectl get pods -n devboard -w
```

## Namespace.yml

```yml
# Create a separate logical space for DevBoard
apiVersion: v1
kind: Namespace
metadata:
  name: devboard
```
## Apply
```cmd
# Create the namespace or update its declared configuration
kubectl apply -f namespace.yml

# Verify that it exists
kubectl get namespace devboard
```
# Pod commands

```cmd
# 1. List DevBoard pods
kubectl get pods -n devboard

# 2. Show pod IPs and the nodes hosting the pods
kubectl get pods -o wide -n devboard

# 3. Continuously watch pod status
kubectl get pods -n devboard -w

# 4. Inspect the standalone frontend pod
kubectl describe pod devboard-frontend -n devboard

# 5. Display logs from the standalone frontend pod
kubectl logs devboard-frontend -n devboard

# 6. Display logs from a specific Deployment pod
kubectl logs devboard-frontend-deployment-6c959767b5-lt67n -n devboard

# 7. Display recent events for a pod
kubectl get events -n devboard --sort-by=.metadata.creationTimestamp

# 8. Open a shell inside a running frontend container
kubectl exec -it devboard-frontend -n devboard -- sh
```
## To apply your standalone pod manifest:
```cmd
kubectl get pods -n devboard

# Create or update the standalone frontend pod
kubectl apply -f pod.yml

# Check its current status
kubectl get pod devboard-frontend -n devboard
```
# Deployment commands
```yml
# 1. List Deployments in devboard
kubectl get deployments -n devboard

# 2. Show detailed Deployment status
kubectl describe deployment devboard-frontend-deployment -n devboard

# 3. Wait for the Deployment rollout to complete
kubectl rollout status deployment/devboard-frontend-deployment -n devboard

# 4. Display ReplicaSets
kubectl get replicasets -n devboard

# 5. Show the Deployment's YAML configuration
kubectl get deployment devboard-frontend-deployment -n devboard -o yaml

# 6. Scale the Deployment to three replicas
kubectl scale deployment devboard-frontend-deployment --replicas=3 -n devboard

# 7. Verify the replica count
kubectl get deployment devboard-frontend-deployment -n devboard

# 8. Inspect rollout history
kubectl rollout history deployment/devboard-frontend-deployment -n devboard
```
## Deployment.yml
```yml
spec:
  replicas: 5  # Desired number of application pods
  selector:
    matchLabels:
      app: devboard-frontend  # Deployment manages pods with this label
  template:
    metadata:
      labels:
        app: devboard-frontend  # Must match the selector and Service
    spec:
      containers:
        - name: devboard-frontend
          image: anandkumarprajap/devboard-frontend:latest
          ports:
            - containerPort: 4173  # Frontend application port
```
# Service commands
```cmd
# 1. List Services in devboard
kubectl get svc -n devboard

# 2. Show detailed information about the frontend Service
kubectl describe svc devboard-frontend-service -n devboard

# 3. Show the Service IP and exposed ports
kubectl get svc devboard-frontend-service -n devboard -o wide

# 4. Show the Service configuration in YAML
kubectl get svc devboard-frontend-service -n devboard -o yaml

# 5. Check the endpoints selected by the Service
kubectl get endpoints devboard-frontend-service -n devboard

# 6. Inspect modern EndpointSlice resources
kubectl get endpointslices -n devboard

# 7. Check which pods match the Service selector
kubectl get pods -n devboard -l app=devboard-frontend

# 8. Show all Services in every namespace
kubectl get svc -A
```
## Service.yml

```yml
apiVersion: v1
kind: Service
metadata:
  name: devboard-frontend-service
  namespace: devboard
spec:
  type: NodePort
  selector:
    app: devboard-frontend  # Finds pods carrying this label
  ports:
    - protocol: TCP
      port: 8080       # Service port inside Kubernetes
      targetPort: 4173 # Port where the frontend listens
      nodePort: 30080  # Port exposed on Kubernetes nodes
```
```cmd
# Apply the Service manifest
kubectl apply -f service.yml

# Verify Service port configuration
kubectl get svc devboard-frontend-service -n devboard

# Verify that at least one ready application endpoint is available
kubectl get endpoints devboard-frontend-service -n devboard
```
# Check NodePort and HTTP access
```yml
# kind-config.yml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
name: babu-cluster

nodes:
  - role: control-plane
    image: kindest/node:v1.35.0
    extraPortMappings:
      - containerPort: 30080  # NodePort inside the Kind node container
        hostPort: 8080         # Port exposed on the EC2 host
        protocol: TCP

  - role: worker
    image: kindest/node:v1.35.0

  - role: worker
    image: kindest/node:v1.35.0
```

```cmd
# 1. Verify the Service's NodePort
kubectl get svc devboard-frontend-service -n devboard

# 2. Inspect the Kind control-plane port mapping
docker port babu-cluster-control-plane

# 3. Test HTTP from the EC2 host itself
curl -v http://127.0.0.1:8080/

# 4. Test the application from inside the Kubernetes cluster
kubectl run curl-test -n devboard \
  --image=curlimages/curl:latest \
  --restart=Never \
  --command -- curl -sS -m 10 \
  http://devboard-frontend-service:8080/

# 5. Check the temporary test pod
kubectl get pod curl-test -n devboard

# 6. Read its output
kubectl logs curl-test -n devboard

# 7. Remove the temporary test pod
kubectl delete pod curl-test -n devboard
```

# Commands to describe and debug everythin
```cmd
# Namespace status and configuration
kubectl describe namespace devboard

# Node health and scheduling information
kubectl get nodes -o wide
kubectl describe nodes

# Deployment status and events
kubectl describe deployment devboard-frontend-deployment -n devboard

# Service selector, ports and endpoints
kubectl describe svc devboard-frontend-service -n devboard

# Pod state, image pull errors and scheduling events
kubectl get pods -o wide -n devboard
kubectl describe pods -n devboard

# Application logs
kubectl logs -l app=devboard-frontend -n devboard --all-containers=true

# Recent events in the DevBoard namespace
kubectl get events -n devboard --sort-by=.metadata.creationTimestamp

# All common DevBoard resources
kubectl get all -n devboard

# Kubernetes system pods
kubectl get pods -n kube-system

# Kind container status
docker ps -a

# Kind control-plane logs
docker logs babu-cluster-control-plane --tail 100
```



# DevBoard Kubernetes Deployment on Kind

Step-by-step notes for checking Docker and Kind, inspecting Kubernetes
resources, and exposing the DevBoard frontend through a NodePort
service.

## 1. Lab Architecture

``` text
Browser
  |
  | http://EC2-PUBLIC-IP:8080
  v
EC2 Host :8080
  |
  | Docker port mapping
  v
Kind Control-Plane Container :30080
  |
  | Kubernetes NodePort Service :30080
  v
Service: devboard-frontend-service
  |  Service port: 8080
  |  targetPort: 4173
  v
Frontend Pod :4173
```

### Important details

-   Kind cluster: `babu-cluster`
-   Kubernetes namespace: `devboard`
-   Frontend Deployment: `devboard-frontend-deployment`
-   Frontend Service: `devboard-frontend-service`
-   Service type: `NodePort`
-   Service port: `8080`
-   Container target port: `4173`
-   NodePort: `30080`
-   Host port: `8080`

The request path is: **Browser → EC2 port 8080 → Docker/Kind port
mapping → NodePort 30080 → Service port 8080 → Pod port 4173**.

> The EC2 security group must allow inbound TCP traffic on port `8080`
> from the intended client IP. The application must also be listening on
> port `4173` inside the frontend container.

------------------------------------------------------------------------

## 2. Move to the Kubernetes Project Directory

``` bash
cd ~/GitHub-Actions-Revision/k8s
pwd
ls -la
```

-   `cd` changes the current directory.
-   `pwd` prints the current directory.
-   `ls -la` lists files, including hidden files.

Check that the Kubernetes manifest files are present, for example
`namespace.yml`, `pod.yml`, `deployment.yml`, and `service.yml`.

------------------------------------------------------------------------

## 3. Check Docker

### 3.1 Docker version

``` bash
docker --version
```

Displays the installed Docker client version.

### 3.2 Docker service status

``` bash
sudo systemctl status docker
```

Checks whether the Docker service is running on the Linux host.

### 3.3 List running containers

``` bash
docker ps
```

Shows running containers. For a Kind cluster, you should see containers
such as `babu-cluster-control-plane`, `babu-cluster-worker`, and
`babu-cluster-worker2`.

### 3.4 List all containers

``` bash
docker ps -a
```

Shows running and stopped containers.

### 3.5 List local images

``` bash
docker images
```

Lists container images available on the host.

### 3.6 Check container resource usage

``` bash
docker stats --no-stream
```

Shows a one-time snapshot of CPU, memory, network, and block I/O usage.

### 3.7 Check Kind control-plane logs

``` bash
docker logs babu-cluster-control-plane --tail 50
```

Displays the last 50 log lines from the control-plane container.

### 3.8 Inspect published ports

``` bash
docker port babu-cluster-control-plane
```

Shows host-to-container port mappings. The lab expects host port `8080`
to map to container port `30080`.

### 3.9 Check whether host port 8080 is listening

``` bash
sudo ss -lntp | grep ':8080'
```

Checks for a process listening on TCP port `8080`. No output can mean no
matching listening socket was found.

------------------------------------------------------------------------

## 4. Check Kind

### 4.1 Kind version

``` bash
kind --version
```

Prints the installed Kind version.

### 4.2 List Kind clusters

``` bash
kind get clusters
```

Confirms which Kind clusters exist. This lab uses `babu-cluster`.

### 4.3 List nodes belonging to the cluster

``` bash
kind get nodes --name babu-cluster
```

Lists the Docker containers that serve as Kubernetes nodes in this Kind
cluster.

### 4.4 Export the kubeconfig for this cluster

``` bash
kind export kubeconfig --name babu-cluster
```

Updates the local kubeconfig so `kubectl` can connect to the cluster.

------------------------------------------------------------------------

## 5. Check kubectl and Cluster Context

### 5.1 kubectl client version

``` bash
kubectl version --client
```

Shows the installed `kubectl` client version.

### 5.2 Cluster information

``` bash
kubectl cluster-info
```

Displays the Kubernetes control-plane endpoint and cluster services.

### 5.3 Current context

``` bash
kubectl config current-context
```

Shows which cluster context `kubectl` currently uses.

### 5.4 Available contexts

``` bash
kubectl config get-contexts
```

Lists configured contexts.

### 5.5 Select the Kind context

``` bash
kubectl config use-context kind-babu-cluster
```

Selects the context for `babu-cluster`.

### 5.6 Client and server versions

``` bash
kubectl version
```

Displays the client and Kubernetes API server versions when the cluster
is reachable.

------------------------------------------------------------------------

## 6. Check Kubernetes Nodes

``` bash
kubectl get nodes
```

Lists cluster nodes and their status.

``` bash
kubectl get nodes -o wide
```

Adds details such as node IP, OS image, and runtime.

``` bash
kubectl describe node babu-cluster-control-plane
kubectl describe node babu-cluster-worker
kubectl describe node babu-cluster-worker2
```

Shows detailed status, labels, capacity, allocated resources, and events
for each node. Adjust node names if your cluster uses different names.

``` bash
kubectl top nodes
```

Shows node CPU and memory usage if the Metrics Server is installed and
working. If it is unavailable, this command may report an error; that
does not by itself mean the nodes are unhealthy.

``` bash
kubectl get events -A --sort-by=.metadata.creationTimestamp
```

Lists cluster events ordered by creation time.

------------------------------------------------------------------------

## 7. Create and Inspect the Namespace

A namespace groups Kubernetes resources and helps separate workloads
within a cluster.

### 7.1 List namespaces

``` bash
kubectl get namespaces
```

### 7.2 Check the DevBoard namespace

``` bash
kubectl get namespace devboard
kubectl describe namespace devboard
```

If the namespace does not exist, create it.

### 7.3 Example `namespace.yml`

``` yaml
apiVersion: v1
kind: Namespace
metadata:
  name: devboard
```

### 7.4 Apply the namespace manifest

``` bash
kubectl apply -f namespace.yml
```

`kubectl apply` creates the namespace if it does not exist or updates it
if the manifest changes.

### 7.5 List pods in the namespace

``` bash
kubectl get pods -n devboard
```

### 7.6 List pods in all namespaces

``` bash
kubectl get pods -A
```

### 7.7 List common resources in DevBoard

``` bash
kubectl get all -n devboard
```

This shows common resources such as pods, services, Deployments, and
ReplicaSets. It does not show every possible Kubernetes resource type.

------------------------------------------------------------------------

## 8. Inspect Pods

A **Pod** is the smallest deployable Kubernetes unit. It contains one or
more containers that share network and storage resources.

### 8.1 List pods

``` bash
kubectl get pods -n devboard
```

### 8.2 Show pod IP and node placement

``` bash
kubectl get pods -o wide -n devboard
```

Useful columns include pod IP and the node on which each pod runs.

### 8.3 Watch pod changes

``` bash
kubectl get pods -n devboard -w
```

Keeps the command running and updates the output as pod states change.
Press `Ctrl+C` to stop watching.

### 8.4 Describe a pod

``` bash
kubectl describe pod devboard-frontend -n devboard
```

Shows events, container state, image, IP, node, and other details. Use
the exact pod name from `kubectl get pods` because generated names may
change.

### 8.5 Read pod logs

``` bash
kubectl logs devboard-frontend -n devboard
```

For a Deployment-managed pod, use the exact current pod name:

``` bash
kubectl get pods -n devboard
kubectl logs <deployment-pod-name> -n devboard
```

Follow logs continuously:

``` bash
kubectl logs -f <pod-name> -n devboard
```

### 8.6 Open a shell in a container

``` bash
kubectl exec -it <pod-name> -n devboard -- sh
```

Starts a shell if the container image includes `sh`. Use `bash` only if
it is installed in the image.

------------------------------------------------------------------------

## 9. Inspect the Deployment and ReplicaSet

A **Deployment** declares the desired state for an application,
including how many pod replicas should run. Kubernetes creates and
manages ReplicaSets to maintain that state.

### 9.1 List Deployments

``` bash
kubectl get deployments -n devboard
```

### 9.2 Describe the Deployment

``` bash
kubectl describe deployment devboard-frontend-deployment -n devboard
```

Shows desired and available replicas, pod template, selector, rollout
information, and events.

### 9.3 Check rollout status

``` bash
kubectl rollout status deployment/devboard-frontend-deployment -n devboard
```

Waits for the Deployment rollout to complete.

### 9.4 List ReplicaSets

``` bash
kubectl get replicasets -n devboard
```

A ReplicaSet maintains the requested number of matching pods for a
Deployment revision.

### 9.5 View the Deployment YAML

``` bash
kubectl get deployment devboard-frontend-deployment -n devboard -o yaml
```

Displays the live Deployment configuration.

### 9.6 Scale replicas

``` bash
kubectl scale deployment/devboard-frontend-deployment --replicas=3 -n devboard
kubectl get pods -o wide -n devboard
```

Changes the desired replica count to three, then lists the pods. If the
manifest is later reapplied with a different replica count, that
manifest can change the desired count again.

### 9.7 Check rollout history

``` bash
kubectl rollout history deployment/devboard-frontend-deployment -n devboard
```

Lists rollout revisions.

### Important: standalone Pod vs Deployment

If `pod.yml` creates a standalone pod named `devboard-frontend`, and
`deployment.yml` creates `devboard-frontend-deployment`, these are
separate workloads. The standalone pod is not managed by the Deployment.
For a typical application deployment, use the Deployment to manage
replicas and updates rather than maintaining an extra standalone pod
unless it is needed for practice.

------------------------------------------------------------------------

## 10. Inspect the Service

A **Service** gives a stable network endpoint for a set of pods. Its
selector must match the labels on the intended pods.

### 10.1 List Services

``` bash
kubectl get services -n devboard
```

Short form:

``` bash
kubectl get svc -n devboard
```

### 10.2 Describe the frontend Service

``` bash
kubectl describe service devboard-frontend-service -n devboard
```

Check the Service type, selector, Service port, target port, NodePort,
and endpoints.

### 10.3 View the Service YAML

``` bash
kubectl get service devboard-frontend-service -n devboard -o yaml
```

### 10.4 Check endpoints

``` bash
kubectl get endpoints devboard-frontend-service -n devboard
```

The endpoint list should include ready pod IPs and the target port. If
it shows no endpoints, check that the Service selector matches pod
labels and that the selected pods are Ready.

### 10.5 Check EndpointSlices

``` bash
kubectl get endpointslices -n devboard
```

EndpointSlices contain endpoint information used by Services.

### 10.6 Check pod labels

``` bash
kubectl get pods --show-labels -n devboard
```

For this lab, the Service selector is expected to match:

``` yaml
selector:
  app: devboard-frontend
```

------------------------------------------------------------------------

## 11. Service YAML: NodePort

Example `service.yml`:

``` yaml
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
      port: 8080
      targetPort: 4173
      nodePort: 30080
```

### Line-by-line explanation

-   `apiVersion: v1`: Uses the core Kubernetes API.
-   `kind: Service`: Creates a Service resource.
-   `metadata.name`: Sets the Service name.
-   `metadata.namespace`: Places the Service in the `devboard`
    namespace.
-   `type: NodePort`: Exposes the Service through a port on each
    Kubernetes node.
-   `selector`: Selects pods with the label `app: devboard-frontend`.
-   `protocol: TCP`: Uses TCP networking.
-   `port: 8080`: The port clients use when addressing the Service
    inside the cluster.
-   `targetPort: 4173`: The port on the selected frontend pod/container
    that receives the traffic.
-   `nodePort: 30080`: The port exposed on the Kubernetes nodes.

Apply and verify it:

``` bash
kubectl apply -f service.yml
kubectl get svc devboard-frontend-service -n devboard
kubectl describe svc devboard-frontend-service -n devboard
kubectl get endpoints devboard-frontend-service -n devboard
```

------------------------------------------------------------------------

## 12. Understand the Kind Port Mapping

The Kind configuration for this lab maps host port `8080` to node
container port `30080`.

Example `~/kind-config.yml`:

``` yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
name: babu-cluster

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

### What the mapping means

-   `hostPort: 8080`: Port exposed on the EC2 host.
-   `containerPort: 30080`: Port on the Kind node container.
-   Kubernetes Service `nodePort: 30080`: The NodePort receiving
    external traffic inside the Kind node.

Check the actual mapping:

``` bash
docker port babu-cluster-control-plane
```

Expected output should include a mapping from container port `30080` to
host port `8080`, such as `30080/tcp -> 0.0.0.0:8080`.

**Important:** `extraPortMappings` are applied when the Kind node
containers are created. Editing the config file later does not
automatically change an existing cluster's port mappings. Recreating a
cluster may be necessary to apply a new mapping, and deleting a cluster
removes the Kubernetes resources stored in that cluster. Back up or
recreate the manifests before doing so.

------------------------------------------------------------------------

## 13. Test the Application

### 13.1 Confirm the Deployment is ready

``` bash
kubectl rollout status deployment/devboard-frontend-deployment -n devboard
kubectl get pods -o wide -n devboard
```

### 13.2 Confirm the Service and endpoints

``` bash
kubectl get svc devboard-frontend-service -n devboard
kubectl get endpoints devboard-frontend-service -n devboard
```

### 13.3 Test from the EC2 host

``` bash
curl -v http://127.0.0.1:8080/
```

-   `curl` sends an HTTP request.
-   `-v` prints connection and HTTP details.
-   A successful response indicates that the request reached an HTTP
    server through the configured port mapping. The returned page
    depends on the frontend application.

### 13.4 Open from a browser

``` text
http://EC2-PUBLIC-IP:8080
```

Replace `EC2-PUBLIC-IP` with the public IPv4 address of your EC2
instance.

Before testing externally:

1.  Confirm Docker shows host port `8080` mapped to container port
    `30080`.
2.  Confirm the Kubernetes Service uses NodePort `30080`.
3.  Confirm the Service has ready endpoints.
4.  Add an inbound EC2 Security Group rule for TCP port `8080`, ideally
    restricted to your own public IP.
5.  Confirm the frontend server listens on the expected container port
    `4173`.

Do not open the application to `0.0.0.0/0` unless public access is
intentional and appropriate for your lab.

------------------------------------------------------------------------

## 14. Troubleshooting Commands

### Pods are not Running

``` bash
kubectl get pods -n devboard
kubectl describe pod <pod-name> -n devboard
kubectl logs <pod-name> -n devboard
kubectl get events -n devboard --sort-by=.metadata.creationTimestamp
```

Look for image pull errors, failed probes, crashes, or scheduling
problems.

### Service has no endpoints

``` bash
kubectl describe svc devboard-frontend-service -n devboard
kubectl get pods --show-labels -n devboard
kubectl get endpoints devboard-frontend-service -n devboard
```

Confirm the selector and pod labels match and that the selected pod is
Ready.

### Browser cannot connect

``` bash
docker ps
docker port babu-cluster-control-plane
sudo ss -lntp | grep ':8080'
kubectl get svc devboard-frontend-service -n devboard
kubectl get endpoints devboard-frontend-service -n devboard
curl -v http://127.0.0.1:8080/
```

Then check the EC2 Security Group inbound rule and the frontend's
listening port.

### Check all resources in the namespace

``` bash
kubectl get all -n devboard
```

### Watch resources while troubleshooting

``` bash
kubectl get pods,svc,deploy,rs -n devboard -w
```

Press `Ctrl+C` to stop watching.

------------------------------------------------------------------------

## 15. Complete Step-by-Step Deployment Workflow

Run the commands in this order from the EC2 host.

### Step 1: Enter the project directory

``` bash
cd ~/GitHub-Actions-Revision/k8s
```

### Step 2: Verify Docker and Kind

``` bash
docker --version
docker ps
kind --version
kind get clusters
kind get nodes --name babu-cluster
```

### Step 3: Connect kubectl to the correct cluster

``` bash
kind export kubeconfig --name babu-cluster
kubectl config current-context
kubectl get nodes -o wide
```

If needed, select the context:

``` bash
kubectl config use-context kind-babu-cluster
```

### Step 4: Create the namespace

``` bash
kubectl apply -f namespace.yml
kubectl get namespace devboard
```

### Step 5: Apply the application Deployment

``` bash
kubectl apply -f deployment.yml
kubectl rollout status deployment/devboard-frontend-deployment -n devboard
kubectl get deployment -n devboard
kubectl get pods -o wide -n devboard
```

### Step 6: Apply the Service

``` bash
kubectl apply -f service.yml
kubectl get svc -n devboard
kubectl describe svc devboard-frontend-service -n devboard
kubectl get endpoints devboard-frontend-service -n devboard
```

### Step 7: Verify the Kind-to-host port mapping

``` bash
docker port babu-cluster-control-plane
```

### Step 8: Test the application

``` bash
curl -v http://127.0.0.1:8080/
```

Then open `http://EC2-PUBLIC-IP:8080` in a browser after confirming the
EC2 Security Group permits inbound TCP `8080`.

### Step 9: Final verification

``` bash
kubectl get nodes
kubectl get all -n devboard
kubectl get pods -o wide -n devboard
kubectl get svc -n devboard
kubectl get endpoints devboard-frontend-service -n devboard
docker ps
```

------------------------------------------------------------------------

## 16. Quick Command Reference

  ------------------------------------------------------------------------------------------------------------------
  Goal                                Command
  ----------------------------------- ------------------------------------------------------------------------------
  Show cluster nodes                  `kubectl get nodes -o wide`

  List namespaces                     `kubectl get ns`

  List DevBoard resources             `kubectl get all -n devboard`

  List pods with node/IP              `kubectl get pods -o wide -n devboard`

  Describe a pod                      `kubectl describe pod <pod-name> -n devboard`

  Read pod logs                       `kubectl logs <pod-name> -n devboard`

  List Deployments                    `kubectl get deploy -n devboard`

  Check rollout                       `kubectl rollout status deployment/devboard-frontend-deployment -n devboard`

  List Services                       `kubectl get svc -n devboard`

  Check Service endpoints             `kubectl get endpoints devboard-frontend-service -n devboard`

  Check Kind host mapping             `docker port babu-cluster-control-plane`

  Test locally                        `curl -v http://127.0.0.1:8080/`
  ------------------------------------------------------------------------------------------------------------------

## Key Takeaways

-   **Docker** runs the Kind node containers on the EC2 host.
-   **Kind** runs a local Kubernetes cluster using Docker containers as
    nodes.
-   **Namespace** groups resources such as the DevBoard workload.
-   **Pod** runs the application container(s).
-   **Deployment** manages replicas and application updates.
-   **ReplicaSet** helps keep the desired number of pods running.
-   **Service** provides a stable endpoint and selects pods using
    labels.
-   **NodePort `30080`** exposes the Service on a Kubernetes node.
-   **Kind port mapping `8080:30080`** publishes that node port on the
    EC2 host.
-   **EC2 Security Group TCP `8080`** allows the browser to reach the
    host port.
