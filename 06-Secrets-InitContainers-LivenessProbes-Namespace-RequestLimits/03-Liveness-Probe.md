# Deep-Dive: Kubernetes Liveness Probes 🩺❤️💫

In Kubernetes, it's not enough for a container to just be "running." It needs to be **healthy**. **Liveness Probes** are the mechanism Kubernetes uses to detect when a container has entered a "dead" state (hung, deadlocked, etc.) and automatically restart it.

---

## 1. What are Liveness Probes? (Self-Healing) 🪄

A **Liveness Probe** is a periodic check performed by the Kubelet on a container. 
- **Success**: The container stays running.
- **Failure**: The Kubelet kills the container, and it is subjected to its **Restart Policy**.

### Why do we need them? (The "Zombie" Problem) 🧟
Your application might be running (process is alive in `ps aux`), but it is deadlocked or unable to serve traffic. Without a Liveness Probe, Kubernetes would think everything is fine. With a probe, Kubernetes "pings" the app, notices it's not responding, and performs a fresh restart.

---

## 2. The Three Check Methods (The "How") 🛠️

Kubernetes supports three ways to check a container's health:

### 2.1 HTTP `httpGet` (Most Common)
Kubernetes sends an HTTP GET request to a specific path and port.
- Any code `>= 200` and `< 400` is considered **Success**.

```yaml
livenessProbe:
  httpGet:
    path: /health
    port: 8080
```

### 2.2 Command `exec`
Kubernetes runs a command inside the container.
- Exit code `0` is **Success**.
- **When to use**: For simple scripts or database CLI checks (e.g., checking if a file exists or if a Redis server is alive).

```yaml
livenessProbe:
  exec:
    command:
    - cat
    - /tmp/healthy
```

### 2.3 TCP `tcpSocket`
Kubernetes attempts to open a TCP socket on a specific port.
- Successful connection is **Success**.
- **When to use**: For applications that don't have an HTTP endpoint (e.g., a simple proxy, a cache, or a database).

```yaml
livenessProbe:
  tcpSocket:
    port: 3306
```

---

## 3. Tuning the Probe (The "Breakdown") ⚙️

You must configure the probe carefully to avoid "Kill Loops" (where a slow-starting app is killed before it can get ready).

| Parameter | Meaning | Recommended Start |
| :--- | :--- | :--- |
| `initialDelaySeconds` | How long to wait after the container starts before the first probe. | 30-60s (for Java) |
| `periodSeconds` | How often to perform the probe. | 10s |
| `timeoutSeconds` | How long to wait for a response before considered failed. | 1-5s |
| `failureThreshold` | Number of consecutive failures before restarting. | 3 |
| `successThreshold` | Number of consecutive successes to count as healthy. | 1 |

---

## 4. Implementation Example 📄🐳

In [10-usermanagement-microservice-with-liveness-probe.yaml](file:///home/user/Skills/EKS/06-Secrets-InitContainers-LivenessProbes-Namespace-RequestLimits/kube-manifests/10-usermanagement-microservice-with-liveness-probe.yaml), we target our Spring Boot health endpoint:

```yaml
livenessProbe:
  httpGet:
    path: /usermgmt/health-status
    port: 8095
  initialDelaySeconds: 60  # Give Spring Boot time to start!
  periodSeconds: 10
```

---

## 5. When to use them? (Scenario Guide) 🕰️

*   **USE WHEN**: Your app can hang or crash without the process actually exiting.
*   **DO NOT USE**: To check for external dependencies (like a DB). Use **Readiness Probes** or **InitContainers** for that! If you use a Liveness probe to check a DB, and the DB goes down for maintenance, Kubernetes will kill **all** your app pods, making a bad situation worse.

---

## 6. Where to put them? (Manifest Location) 📍

They live inside the `containers` list in your Deployment manifest:

```yaml
spec:
  template:
    spec:
      containers:
        - name: my-app
          image: my-app:1.0
          livenessProbe: # <--- Right here!
            httpGet:
              path: /health
              port: 8080
```

---

### Victory! 🏆
You have now added **Self-Healing** to your EKS arsenal. Your applications are now mission-critical ready, able to recover from "frozen" states without human intervention.
