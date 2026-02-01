# Deep-Dive: Writing Pod YAML Manifests 🏗️

A **Pod** is the smallest deployable unit in Kubernetes. To create one declaratively, you need a YAML manifest. This guide breaks down every field you need to know for a professional-grade Pod definition.

---

## 1. The "Big Four" Foundation 🧱

Every Pod manifest begins with these four top-level fields:

1.  **`apiVersion`**: The version of the Kubernetes API (Pod uses `v1`).
2.  **`kind`**: The type of object (always `Pod` for this file).
3.  **`metadata`**: Data that identifies the Pod (name, labels, namespace).
4.  **`spec`**: The desired state—what containers to run and how.

---

## 2. Metadata: Tags and Identification 🏷️

Metadata helps you organize and find your Pods.

-   **`name`**: A unique name for the Pod within the namespace.
-   **`labels`**: Key-value pairs used for selection (e.g., `app: nginx`). These are vital for connecting Services to Pods.
-   **`annotations`**: Non-identifying metadata (e.g., "created-by: saikiran"). Used for tools or documentation.
-   **`namespace`**: The logical folder where the Pod lives (default is `default`).

```yaml
metadata:
  name: my-pro-pod
  namespace: production
  labels:
    app: web-server
    env: prod
  annotations:
    description: "Main frontend pod"
```

---

## 3. The Spec: The Container Engine ⚙️

The `spec` section is where the real work happens. It defines the containers inside the Pod.

### 3.1 Container Arrays
A Pod can have multiple containers, though usually, it's one (Sidecar pattern uses more).

-   **`name`**: Name of the container.
-   **`image`**: The container image (e.g., `nginx:1.21`).
-   **`imagePullPolicy`**: 
    - `Always`: Always pull from registry.
    - `IfNotPresent`: Pull only if not on the node.
    - `Never`: Only use local images.

### 3.2 Environments & Commands
```yaml
spec:
  containers:
    - name: my-app
      image: my-app:v1
      env:
        - name: DB_URL
          value: "postgres://db:5432"
      ports:
        - containerPort: 8080
```

---

## 4. Resource Management: Limits & Requests ⚖️

In a production environment, you **must** specify resources to prevent one Pod from crashing the whole node.

-   **`requests`**: The minimum resources the Pod needs to start (Kubernetes uses this for scheduling).
-   **`limits`**: The maximum resources the Pod is allowed to consume (throttles or kills the Pod if exceeded).

```yaml
resources:
  requests:
    memory: "64Mi"
    cpu: "250m" # 0.25 of a core
  limits:
    memory: "128Mi"
    cpu: "500m"
```

---

## 5. The "Pro-Version" Commented Manifest 🌟

Here is what a professional DevOps engineer's Pod YAML looks like:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: production-web
  labels:
    tier: frontend
    version: v1
spec:
  containers:
    - name: nginx-web
      image: nginx:1.21
      imagePullPolicy: IfNotPresent
      
      # Define how the container starts
      ports:
        - name: http
          containerPort: 80
      
      # Resource constraints for stability
      resources:
        requests:
          cpu: "100m"
          memory: "128Mi"
        limits:
          cpu: "200m"
          memory: "256Mi"
          
      # Environment variables
      env:
        - name: APP_MODE
          value: "production"
```

---

## 6. Verification Commands 🔍

After writing your YAML, use these commands to ensure it works!

```bash
# 1. Validate without creating
kubectl apply -f pod.yaml --dry-run=client

# 2. Create the Pod
kubectl apply -f pod.yaml

# 3. See the full status (including default fields K8s added)
kubectl get pod production-web -o yaml

# 4. Filter by labels
kubectl get pods -l tier=frontend
```
