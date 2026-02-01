# Deep-Dive: NodePort Service YAML Manifests 🌐

A **Service** in Kubernetes is an abstraction that defines a logical set of Pods and a policy by which to access them. Use a **NodePort** service when you want to expose your application to the outside world by opening a specific port on every Worker Node.

---

## 1. The Core Structure 🧱

Like all Kubernetes objects, a Service manifest starts with the standard headers:

1.  **`apiVersion`**: `v1`
2.  **`kind`**: `Service`
3.  **`metadata`**: Name and labels for the service itself.
4.  **`spec`**: The definition of how the service behaves.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-nodeport-service
spec:
  type: NodePort  # Critical: Tells K8s to open a port on the nodes
  ...
```

---

## 2. Port Mapping: The "Three Ports" Logic 🔌

This is the most confused part of Kubernetes services. You must understand the relationship between these three:

| Port Field | Level | Description |
| :--- | :--- | :--- |
| **`port`** | **Service Level** | The port the service is reachable on *inside* the cluster. |
| **`targetPort`** | **Pod Level** | The port the application is actually listening on inside the container. |
| **`nodePort`** | **Node Level** | The port opened on every Worker Node for *external* access (Range: 30000-32767). |

> [!TIP]
> If you don't specify a `nodePort`, Kubernetes will automatically assign a random one between 30000 and 32767.

---

## 3. Selector: The Connection Logic 🔗

The **`selector`** is the "glue" that connects the Service to the Pods. The Service looks for any Pod in the cluster that has labels matching the selector's key-value pairs.

```yaml
spec:
  selector:
    app: my-web-app  # Must match the labels in your Pod metadata!
```

---

## 4. The "Pro-Version" Commented Manifest 🌟

This is how a professional DevOps engineer defines a NodePort service to expose an Nginx application.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-service-nodeport
  labels:
    tier: frontend
spec:
  type: NodePort
  
  # How the Service finds the Pods
  selector:
    app: nginx-app
    tier: frontend
    
  ports:
    - name: http
      protocol: TCP
      
      # Internal Cluster Port (ClusterIP level)
      port: 80
      
      # Port inside the Container
      targetPort: 80
      
      # External Port on Worker Nodes
      # If omitted, K8s picks one in 30000-32767
      nodePort: 32000 
```

---

## 5. Verification & Troubleshooting 🔍

Once you apply your YAML, use these commands to verify the setup:

```bash
# 1. Check basic status
kubectl get svc web-service-nodeport

# 2. Detailed Inspection (Check if 'Endpoints' are populated!)
# Enpoint population means the selector successfully found Pods.
kubectl describe svc web-service-nodeport

# 3. Test access (External)
# Get your Node IP and hit the NodePort
curl <NODE_IP>:32000

# 4. Get Endpoints directly
kubectl get endpoints web-service-nodeport
```

> [!IMPORTANT]
> **Common Pitfall:** If the `Endpoints` field in `kubectl describe svc` is `<none>`, it means your **selectors** do not match any Pod **labels**. Check your Pod YAML!
