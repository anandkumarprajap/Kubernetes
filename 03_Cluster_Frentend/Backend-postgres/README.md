# DevBoard Three-Tier Application on Kubernetes

Beginner-friendly notes based on the terminal session in this project. This guide explains the commands, Kubernetes YAML manifests, errors encountered, deployment checks, and important fixes.

> **Project:** DevBoard  
> **Namespace:** `devboard`  
> **Application tiers:** Frontend, Backend API, PostgreSQL  
> **Container registry image shown in the session:** `anandkumarprajap/devboard-backend:latest`

---

## 1. Project overview

The goal is to run a three-tier application in Kubernetes.

```text
                         User / Browser
                               |
                               v
                  Frontend Service (NodePort)
                       port 30080 on node
                               |
                               v
                    Frontend Deployment
                         Frontend Pod
                               |
                               | API requests
                               v
                     Backend Service
                       ClusterIP :8080
                               |
                               v
                 Backend Deployment (2 replicas)
                    +---------------------+
                    | Backend Pod 1       |
                    | Backend Pod 2       |
                    +---------------------+
                               |
                               | PostgreSQL connection
                               v
                     PostgreSQL Service
                       ClusterIP :5432
                               |
                               v
                   PostgreSQL Deployment
                       PostgreSQL Pod
```

- **Frontend:** the user interface.
- **Backend:** handles API requests and application logic.
- **PostgreSQL:** stores application data.
- **Deployment:** keeps the desired number of Pods running.
- **Service:** gives a stable network endpoint to selected Pods.
- **ConfigMap:** stores non-sensitive configuration and SQL scripts.
- **Secret:** stores sensitive configuration such as a database password.
- **Namespace:** logically groups related Kubernetes resources.

The final `kubectl get all -n devboard` output in the session showed two backend Pods, a frontend Pod managed by a Deployment, a separate frontend Pod, and one PostgreSQL Pod running. This confirms the displayed workloads were running at that moment; it does **not** by itself prove that database tables were initialized or that the application can successfully query the database.

## 2. Go to the project directory

```bash
ls
cd GitHub-Actions-Revision/
cd k8s/
ls
```

- `ls` lists files and directories.
- `cd GitHub-Actions-Revision/` enters the repository.
- `cd k8s/` enters the Kubernetes manifests directory.

The session renamed and copied manifests to make the frontend and backend files separate:

```bash
mv deployment.yml frontend-deployment.yml
mv service.yml frontend-service.yml

cp frontend-deployment.yml backend-deployment.yml
cp frontend-service.yml backend-service.yml
```

**Commands explained:**

- `mv old-name new-name` renames or moves a file.
- `cp source destination` copies a file.

Copying a manifest only creates a starting point. Edit the copied file so its resource name, labels, selectors, image, ports, and settings match the new component. Otherwise, Kubernetes may create the wrong resource or route traffic incorrectly.

## 3. Kubernetes manifest files

The session worked with these files:

| File | Purpose |
|---|---|
| `namespace.yml` | Defines the `devboard` namespace |
| `frontend-deployment.yml` | Defines the frontend Deployment |
| `frontend-service.yml` | Exposes the frontend through NodePort |
| `backend-deployment.yml` | Runs two backend replicas |
| `backend-service.yml` | Provides an internal endpoint for the backend |
| `postgres-deployment.yml` | Runs PostgreSQL |
| `postgres-service.yml` | Provides an internal endpoint for PostgreSQL |
| `configMap.yml` | Holds non-sensitive settings |
| `secrets.yml` | Holds database credentials |
| `postgres-init.yml` | In the session, copied SQL scripts into a ConfigMap manifest |

A Kubernetes YAML file describes the desired state. `kubectl apply -f filename.yml` asks the Kubernetes API server to create or update the resource described in that file.

## 4. Namespace

A namespace keeps related objects grouped together. The session's resources use:

```yaml
metadata:
  namespace: devboard
```

Check that the namespace exists:

```bash
kubectl get namespaces
kubectl get all -n devboard
```

If `devboard` has not been created yet, apply the namespace manifest:

```bash
kubectl apply -f namespace.yml
```

Create the namespace **before** applying resources that refer to it.

## 5. Backend Deployment

The session's backend manifest defined a Deployment with two replicas and the image `anandkumarprajap/devboard-backend:latest`.

Simplified structure:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: devboard-backend-deployment
  namespace: devboard
  labels:
    app: devboard-backend
spec:
  replicas: 2
  selector:
    matchLabels:
      app: devboard-backend
  template:
    metadata:
      labels:
        app: devboard-backend
    spec:
      containers:
        - name: devboard-backend
          image: anandkumarprajap/devboard-backend:latest
          ports:
            - containerPort: 8080
```

### Important fields

- `apiVersion: apps/v1`: API version used for Deployments.
- `kind: Deployment`: asks Kubernetes to manage Pods through a Deployment.
- `metadata.name`: name of the Deployment.
- `metadata.namespace`: namespace where it belongs.
- `spec.replicas: 2`: desired number of backend Pods.
- `selector.matchLabels`: tells the Deployment which Pods it manages.
- `template.metadata.labels`: labels assigned to Pods; they must match the selector.
- `containers.image`: image Kubernetes should run.
- `containerPort: 8080`: documents the port the containerized application listens on. It does not publish the port outside the cluster.

Apply and inspect it:

```bash
kubectl apply -f backend-deployment.yml
kubectl get deployments -n devboard
kubectl get pods -n devboard
kubectl describe deployment devboard-backend-deployment -n devboard
```

## 6. Backend Service

The session created a ClusterIP Service named `backend`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: backend
  namespace: devboard
spec:
  selector:
    app: devboard-backend
  ports:
    - protocol: TCP
      port: 8080
      targetPort: 8080
```

- `kind: Service`: creates a stable network endpoint.
- `type` is omitted, so Kubernetes defaults to `ClusterIP`.
- `selector.app: devboard-backend`: routes traffic to Pods with this label.
- `port: 8080`: port exposed by the Service inside the cluster.
- `targetPort: 8080`: port on the selected Pods.

Other Pods in the same namespace can normally reach the backend at `http://backend:8080`. The Service is internal; it is not directly exposed to the public internet.

```bash
kubectl apply -f backend-service.yml
kubectl get service -n devboard
kubectl get endpoints backend -n devboard
```

If the Service has no endpoints, check that the Service selector matches the backend Pod labels and that the Pods are Ready.

## 7. ConfigMap: non-sensitive configuration

The session used `configMap.yml` for the database username and backend port:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: devboard-configmap
  namespace: devboard
data:
  POSTGRES_USER: devboard
  BACKEND_PORT: "8080"
```

ConfigMaps are for **non-sensitive** values. A port written in `data` should be a string, for example `"8080"`.

Apply and inspect:

```bash
kubectl apply -f configMap.yml
kubectl get configmap -n devboard
kubectl describe configmap devboard-configmap -n devboard
```

### SQL in a ConfigMap

The session also placed `01_schema.sql` and `02_seed.sql` in a ConfigMap manifest. The schema SQL creates `projects` and `tasks` tables, indexes, and an update trigger. The seed SQL inserts example projects and tasks.

A ConfigMap only stores these SQL files; it does **not** execute them by itself. The PostgreSQL container must mount the SQL files into `/docker-entrypoint-initdb.d/` for the official PostgreSQL image to run them automatically on the first initialization of an empty data directory.

Also, the session later reduced `configMap.yml` to only `POSTGRES_USER` and copied the larger manifest to `postgres-init.yml`. Both manifests showed the name `devboard-configmap` in the transcript, so they refer to the **same ConfigMap object** in the same namespace. Applying one can overwrite the data supplied by the other. Prefer distinct names, such as `devboard-config` and `postgres-init-sql`.

## 8. Secret: database credentials

The session generated Base64 text with:

```bash
echo -n "devboard" | base64
```

Output:

```text
ZGV2Ym9hcmQ=
```

The Secret manifest used those values:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: devboard-secrets
  namespace: devboard
type: Opaque
data:
  POSTGRES_PASSWORD: ZGV2Ym9hcmQ=
  POSTGRES_DB: ZGV2Ym9hcmQ=
```

Apply and inspect:

```bash
kubectl apply -f secrets.yml
kubectl get secrets -n devboard
```

**Security note:** Base64 is encoding, not encryption. Do not commit real passwords to a public Git repository. Use appropriate secret-management controls for production. The example password `devboard` is suitable only as a disposable lab value, not a production credential.

## 9. PostgreSQL Deployment

The session created a PostgreSQL Deployment using `postgres:16-alpine`, port `5432`, one replica, and environment variables sourced from the ConfigMap and Secret.

A simplified, corrected example is:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: postgres-deployment
  namespace: devboard
spec:
  replicas: 1
  selector:
    matchLabels:
      app: devboard-postgres
  template:
    metadata:
      labels:
        app: devboard-postgres
    spec:
      containers:
        - name: postgres
          image: postgres:16-alpine
          ports:
            - containerPort: 5432
          env:
            - name: POSTGRES_USER
              valueFrom:
                configMapKeyRef:
                  name: devboard-configmap
                  key: POSTGRES_USER
            - name: POSTGRES_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: devboard-secrets
                  key: POSTGRES_PASSWORD
            - name: POSTGRES_DB
              valueFrom:
                secretKeyRef:
                  name: devboard-secrets
                  key: POSTGRES_DB
```

Apply and check:

```bash
kubectl apply -f postgres-deployment.yml
kubectl get pods -n devboard
kubectl logs deployment/postgres-deployment -n devboard
```

### Important: persistence

The session's PostgreSQL Deployment does not show a PersistentVolumeClaim mounted for database storage. Without persistent storage, database files in the container's writable layer can be lost when a replacement Pod is created. For anything beyond a disposable lab, configure a PersistentVolumeClaim and mount it at `/var/lib/postgresql/data`.

For production, a managed database or an appropriately operated PostgreSQL StatefulSet may be more suitable than a simple Deployment.

## 10. PostgreSQL Service

The session's PostgreSQL Service is named `postgres`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: postgres
  namespace: devboard
spec:
  selector:
    app: devboard-postgres
  ports:
    - protocol: TCP
      port: 5432
      targetPort: 5432
```

Apply it:

```bash
kubectl apply -f postgres-service.yml
kubectl get service postgres -n devboard
kubectl get endpoints postgres -n devboard
```

Because the Service is named `postgres`, the backend connection string in the same namespace can use the hostname `postgres`:

```text
postgres://USERNAME:PASSWORD@postgres:5432/DATABASE?sslmode=disable
```

The actual username, password, and database name should come from the configured environment variables. The backend container must also include a PostgreSQL driver/client library and the application must actually read the expected connection variable.

## 11. Connect the backend to PostgreSQL

The session used this connection string in the backend Deployment:

```yaml
- name: POSTGRES_URL
  value: postgres://$(POSTGRES_USER):$(POSTGRES_PASSWORD)@postgres:5432/$(POSTGRES_DB)?sslmode=disable
- name: PORT
  value: "8080"
```

Kubernetes environment-variable references use `$(VARIABLE_NAME)` syntax in a container's `env[].value`. Ensure the referenced variables are declared earlier in the same container's `env` list, as they are in the session.

The port value should be quoted:

```yaml
value: "8080"
```

YAML may parse an unquoted `8080` as a number, while Kubernetes expects `env[].value` to be a string.

## 12. PostgreSQL initialization SQL: what must be fixed

The session created a ConfigMap containing:

- `01_schema.sql`: tables, indexes, and a trigger.
- `02_seed.sql`: example projects and tasks.

However, the shown PostgreSQL Deployment ended with a `volumes` entry pointing to `postgres-init`, but the transcript does not show a matching volume mount on the container. The volume was also initially placed at the wrong YAML level, which caused Kubernetes validation errors. The Pod eventually ran after that structural issue was edited, but the displayed final manifest still did not show a `volumeMounts` entry.

To use a ConfigMap for PostgreSQL initialization, give the SQL ConfigMap a unique name and mount it into the PostgreSQL container. For example:

```yaml
# Inside the PostgreSQL container specification:
volumeMounts:
  - name: init-sql
    mountPath: /docker-entrypoint-initdb.d
    readOnly: true

# At spec.template.spec level (same level as containers):
volumes:
  - name: init-sql
    configMap:
      name: postgres-init-sql
```

The ConfigMap must contain keys named `01_schema.sql` and `02_seed.sql`. Mounting SQL scripts at this location causes the official PostgreSQL image to run them when it initializes an **empty** database data directory. It will not automatically rerun them on every Pod restart or retrofit an already-initialized database. If the database already exists, run migrations or SQL deliberately and safely.

A corrected initialization ConfigMap must use a distinct resource name, for example:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: postgres-init-sql
  namespace: devboard
data:
  01_schema.sql: |
    CREATE TABLE IF NOT EXISTS projects (
      id SERIAL PRIMARY KEY,
      name VARCHAR(200) NOT NULL,
      description TEXT,
      owner_id INTEGER,
      created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
    );
  02_seed.sql: |
    INSERT INTO projects (id, name, description, owner_id)
    VALUES
      (1, 'DevBoard MVP', 'Ship the v1 task tracker', 1),
      (2, 'Marketing Site', 'Landing page + launch blog', 1)
    ON CONFLICT (id) DO NOTHING;
```

This is a **short illustrative example**, not the full schema and seed script from the session. Preserve the complete SQL from your original manifest when you implement it.

## 13. Frontend Service and accessing the app

The final output showed:

```text
service/devboard-frontend-service   NodePort   ...   8080:30080/TCP
```

That means the Service port is `8080`, and its NodePort is `30080`. In a typical NodePort setup, you access the app through:

```text
http://<KUBERNETES-NODE-IP>:30080
```

Use a reachable node IP. For an EC2-hosted cluster, the instance's security group and host/network firewall must allow the intended traffic. Do not expose database port `5432` publicly.

Check the frontend Service manifest and ensure its selector matches the frontend Pod labels.

## 14. Apply manifests in a sensible order

First, check that the namespace exists, then apply configuration and credentials before workloads:

```bash
kubectl apply -f namespace.yml
kubectl apply -f configMap.yml
kubectl apply -f secrets.yml
kubectl apply -f postgres-init.yml
kubectl apply -f postgres-deployment.yml
kubectl apply -f postgres-service.yml
kubectl apply -f backend-deployment.yml
kubectl apply -f backend-service.yml
kubectl apply -f frontend-deployment.yml
kubectl apply -f frontend-service.yml
```

Before applying the initialization ConfigMap, fix the resource-name collision described above. If `postgres-init.yml` still declares `metadata.name: devboard-configmap`, it is not a separate ConfigMap.

You can validate manifests without changing cluster state:

```bash
kubectl apply --dry-run=client -f backend-deployment.yml
kubectl apply --dry-run=client -f postgres-deployment.yml
```

`--dry-run` alone is deprecated in the shown Kubernetes version; use `--dry-run=client` for client-side validation. Client-side validation does not prove that the application works at runtime.

## 15. Errors encountered and their explanations

### Error 1: `kubectl exex`

Incorrect command:

```bash
kubectl exex ...
```

Correct command:

```bash
kubectl exec -it <pod-name> -n devboard -- /bin/bash
```

`exec` runs a command inside a running container. The `--` separates kubectl options from the command to run in the container.

### Error 2: `unknown field "spec.volumes"`

A Deployment's Pod-level volumes belong under `spec.template.spec.volumes`, not the Deployment's top-level `spec.volumes`.

Correct hierarchy:

```yaml
spec:
  template:
    spec:
      containers:
        - name: postgres
          image: postgres:16-alpine
          volumeMounts:
            - name: init-sql
              mountPath: /docker-entrypoint-initdb.d
      volumes:
        - name: init-sql
          configMap:
            name: postgres-init-sql
```

The `volumes` and `containers` entries are siblings under `spec.template.spec`. `volumeMounts` belongs inside the relevant container.

### Error 3: `unknown field "spec.template.volumes"`

The `volumes` key was one level too high. Move it under `spec.template.spec`, at the same indentation level as `containers`.

### Error 4: `configMap.yml.yml does not exist`

The actual file is named `configMap.yml`, not `configMap.yml.yml`.

Correct command:

```bash
kubectl apply -f configMap.yml
```

Use `ls` to confirm exact filenames.

### Error 5: `cannot unmarshal number ... EnvVar ... value of type string`

Incorrect:

```yaml
- name: PORT
  value: 8080
```

Correct:

```yaml
- name: PORT
  value: "8080"
```

Environment variable values are strings.

### Error 6: PostgreSQL reports `Did not find any relations`

The session connected to PostgreSQL and ran `\d`, but no relations were found. That means no tables were visible in the current database/schema at that time. Merely storing SQL in a ConfigMap does not execute it. Verify the ConfigMap mount, initialization logs, database name, and whether the data directory had already been initialized.

In the `psql` prompt:

```sql
\l
\c devboard
\dt
```

- `\l` lists databases.
- `\c devboard` connects to the `devboard` database.
- `\dt` lists tables in the current schema.
- `\q` exits `psql`.

The correct PostgreSQL command is `\d` or `\dt`, not `/d`.

## 16. Troubleshooting commands

```bash
# View all resources in the namespace
kubectl get all -n devboard

# Check Deployments
kubectl get deployments -n devboard

# Check Pods and their status
kubectl get pods -n devboard -o wide

# Follow logs from a Deployment
kubectl logs deployment/devboard-backend-deployment -n devboard

# Inspect a Pod
kubectl describe pod <pod-name> -n devboard

# Check Services and endpoints
kubectl get services -n devboard
kubectl get endpoints -n devboard

# Inspect configuration and secrets (avoid printing real secrets into shared logs)
kubectl describe configmap devboard-configmap -n devboard
kubectl describe secret devboard-secrets -n devboard

# Check recent namespace events
kubectl get events -n devboard --sort-by=.metadata.creationTimestamp
```

If a Pod is not running, inspect `kubectl describe pod` and its logs. Common causes include image-pull errors, bad environment variables, failing startup, missing configuration, and database connectivity problems.

## 17. Final status shown in the terminal session

The last `kubectl get all -n devboard` output showed:

- Backend Deployment: **2/2 replicas available**
- Frontend Deployment: **1/1 replica available**
- PostgreSQL Deployment: **1/1 replica available**
- Backend Service: `ClusterIP`, port `8080`
- PostgreSQL Service: `ClusterIP`, port `5432`
- Frontend Service: `NodePort`, port mapping `8080:30080`

These are the values recorded at the end of the uploaded terminal session. Re-run the commands to check the current state.

**Important follow-up:** a `Running` Pod does not guarantee database initialization, persistent storage, or end-to-end application functionality. Verify tables and test a real API request before considering the deployment complete.

## 18. Quick revision: interview answers

**What is a Deployment?**  
A Deployment manages application Pods and maintains the desired replica count.

**Why use two backend replicas?**  
To run more than one backend Pod, which can improve availability and distribute requests when a Service selects both healthy Pods.

**What is a ClusterIP Service?**  
An internal Kubernetes Service that provides a stable cluster-network endpoint.

**What is a NodePort Service?**  
A Service that exposes an application on a port on Kubernetes nodes. Network and firewall rules still control whether users can reach it.

**What is a ConfigMap?**  
A Kubernetes object for non-sensitive configuration data.

**What is a Secret?**  
A Kubernetes object intended for sensitive configuration. Base64-encoded Secret data is not automatically encrypted just because it is Base64.

**Why use a Service for PostgreSQL?**  
The backend can connect to the stable hostname `postgres` rather than depending on the changing IP address of a PostgreSQL Pod.

**Why did the PostgreSQL tables not appear?**  
The SQL was stored in a ConfigMap, but the session's displayed manifest did not show the required SQL `volumeMounts`. The SQL needs to be mounted and executed during initialization, or applied through a migration process.

---

## 19. Recommended next steps

1. Give the general application ConfigMap and SQL initialization ConfigMap different names.
2. Add the SQL `volumeMounts` and correctly placed `volumes` entry.
3. Configure persistent storage for PostgreSQL.
4. Verify the tables with `kubectl exec` and `psql`.
5. Confirm the backend can connect to `postgres:5432`.
6. Test the frontend through the NodePort and verify an API request succeeds.
7. Keep real credentials out of source control.
